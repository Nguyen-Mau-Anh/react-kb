# Capabilities — 02. File Uploads: Signed URLs to S3 / R2 / Vercel Blob, react-dropzone, tus Resumable

> **What / Why / How** — never proxy uploads through your Next.js server. Use **direct-to-storage with signed URLs** for files < 100 MB, **multipart** for 100 MB–5 GB, **tus resumable** for "must survive network drops." The browser uploads to the storage bucket; your server only signs the request.

---

## 1. The Single Most Important Rule

**Never upload files through your Node/Edge function.** Every uploaded byte counts against:
- Vercel function memory (1 GB max).
- Vercel function execution time (10–300 seconds, costs money).
- Bandwidth (egress + ingress).
- The serverless cold-start problem under load.

The pattern that scales: the **browser uploads directly to object storage**, and your server only **signs the upload URL**. Your server stays small, fast, and stateless; the storage provider handles the actual bytes.

This file covers four real upload strategies in order of complexity:

| Strategy | When to use | npm |
|----------|-------------|-----|
| **`@vercel/blob` direct upload** | On Vercel; file < 5 GB; want zero infra | `@vercel/blob@0.24` |
| **Direct-to-S3 / R2 with presigned URL** | Single file, single PUT, file < ~100 MB | `@aws-sdk/client-s3@3`, `@aws-sdk/s3-request-presigner@3` |
| **S3 / R2 multipart upload** | Big files (100 MB – 5 GB) | Same SDK + `CreateMultipartUpload` |
| **tus resumable** | Must survive network drops, mobile uploads | `tus-js-client@4`, `@uppy/tus@4`, server: `@tus/server@1` |

Pair any of these with **`react-dropzone@14`** for the drag-drop UI, or **Uppy 4** for a complete end-to-end UI library.

---

## 2. Storage Provider Pick

| Provider | Pricing (mid-2026) | Egress fees | Best for |
|----------|---------------------|-------------|----------|
| **Vercel Blob** | $0.15/GB stored, $0.30/GB egress | Egress is charged | On Vercel; want zero AWS knowledge |
| **Cloudflare R2** | $0.015/GB stored, **$0 egress** | **No egress fees** — the killer feature | Public-served files (images, video, downloads) |
| **AWS S3** | $0.023/GB stored, $0.09/GB egress | Egress charged (high) | Already on AWS, deep ecosystem needs |
| **Backblaze B2** | $0.006/GB stored, $0.01/GB egress (free with Cloudflare CDN) | Cheap | Backups, archive, cost-sensitive |
| **Supabase Storage** | $0.021/GB stored | Egress charged | Already on Supabase |
| **UploadThing** | Pay-per-upload | Bundled CDN | Indie SaaS, "let me ship faster" |

**The 80% pick in 2026**: **Cloudflare R2** with R2's zero-egress + free Cloudflare CDN. Used by every "we serve user-generated images" SaaS that's done the math.

R2 implements the S3 API exactly, so any S3 SDK code works with R2 by changing the endpoint. The patterns below are written for the S3 API and run on AWS S3, R2, Backblaze B2, MinIO, Wasabi — all interchangeable.

---

## 3. Pattern A — Vercel Blob Direct Upload (Easiest)

The fastest path on Vercel. The browser uploads straight to Vercel's blob storage; your server hands out a signed token.

### Install

```bash
npm i @vercel/blob@0.24
```

```bash
# .env
BLOB_READ_WRITE_TOKEN=vercel_blob_rw_...
```

(Generated automatically when you create a Blob store in the Vercel dashboard.)

### Server — issue an upload token

```ts
// app/api/upload/route.ts
import { handleUpload, type HandleUploadBody } from '@vercel/blob/client';
import { NextResponse } from 'next/server';
import { auth } from '@/auth';

export async function POST(req: Request) {
  const body = (await req.json()) as HandleUploadBody;

  try {
    const result = await handleUpload({
      body,
      request: req,
      onBeforeGenerateToken: async (pathname) => {
        const session = await auth();
        if (!session?.user) throw new Error('Unauthorized');
        return {
          allowedContentTypes: ['image/jpeg', 'image/png', 'image/webp'],
          maximumSizeInBytes: 10 * 1024 * 1024,        // 10 MB
          tokenPayload: JSON.stringify({ userId: session.user.id }),
        };
      },
      onUploadCompleted: async ({ blob, tokenPayload }) => {
        const { userId } = JSON.parse(tokenPayload!);
        await db.upload.create({ data: { userId, url: blob.url, pathname: blob.pathname } });
      },
    });
    return NextResponse.json(result);
  } catch (err) {
    return NextResponse.json({ error: (err as Error).message }, { status: 400 });
  }
}
```

### Client — direct upload

```tsx
'use client';
import { upload } from '@vercel/blob/client';
import { useState } from 'react';

export function AvatarUpload() {
  const [progress, setProgress] = useState(0);
  const [url, setUrl] = useState<string>();

  async function handleFile(file: File) {
    const blob = await upload(file.name, file, {
      access: 'public',
      handleUploadUrl: '/api/upload',
      onUploadProgress: ({ percentage }) => setProgress(percentage),
    });
    setUrl(blob.url);
  }

  return (
    <>
      <input type="file" onChange={(e) => e.target.files?.[0] && handleFile(e.target.files[0])} />
      {progress > 0 && <progress value={progress} max={100} />}
      {url && <img src={url} className="size-32 rounded-full" />}
    </>
  );
}
```

### What happens

1. Browser POSTs to `/api/upload` with file metadata (NOT the file).
2. Server validates auth and content type, returns a one-time signed token.
3. Browser uses the token to PUT the file directly to Vercel Blob's edge endpoint.
4. Once complete, Vercel calls back to `onUploadCompleted` with the final URL.

The actual file bytes never touch your server. Your server's CPU cost is constant regardless of file size.

### Why Vercel Blob wins for Vercel-hosted apps

- **Zero infra.** No bucket setup, no IAM policies, no CORS config.
- **CDN built in** — files served from Vercel's edge.
- **Auth integrates naturally** with Auth.js v5 (covered in `07.nextjs/04-auth.md`).
- **Type-safe upload SDK** with progress out of the box.

### Why Vercel Blob loses

- **Egress fees** — $0.30/GB is high for image-heavy apps. Cloudflare R2 is $0.
- **Vendor lock-in** to Vercel's blob format.
- **Less mature** than S3 (released 2023). Some operations not yet exposed.

For pure image hosting at scale: **Cloudflare R2 + a Cloudflare CDN** is dramatically cheaper. For app uploads tied to Vercel features (preview comments, internal tooling): Vercel Blob's DX wins.

---

## 4. Pattern B — Direct-to-S3 / R2 with Presigned URL

The S3 protocol pattern. Works identically on AWS S3, **Cloudflare R2**, Backblaze B2, MinIO, Wasabi, DigitalOcean Spaces.

### Install

```bash
npm i @aws-sdk/client-s3@3 @aws-sdk/s3-request-presigner@3
```

### Configure for R2 (zero-egress, the 2026 default)

```ts
// lib/s3.ts
import { S3Client } from '@aws-sdk/client-s3';

export const s3 = new S3Client({
  region: 'auto',                                   // R2 ignores region
  endpoint: process.env.R2_ENDPOINT!,                // https://<account>.r2.cloudflarestorage.com
  credentials: {
    accessKeyId: process.env.R2_ACCESS_KEY_ID!,
    secretAccessKey: process.env.R2_SECRET_ACCESS_KEY!,
  },
});
```

For AWS S3: omit `endpoint` and set `region` to a real AWS region.

### Server — issue a presigned PUT URL

```ts
// app/api/upload-url/route.ts
import { NextResponse } from 'next/server';
import { PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { z } from 'zod';
import { auth } from '@/auth';
import { s3 } from '@/lib/s3';
import { randomUUID } from 'crypto';

const Schema = z.object({
  filename: z.string().min(1),
  contentType: z.enum(['image/jpeg', 'image/png', 'image/webp']),
  size: z.number().int().positive().max(20 * 1024 * 1024), // 20 MB max
});

export async function POST(req: Request) {
  const session = await auth();
  if (!session?.user) return new Response('Unauthorized', { status: 401 });

  const parsed = Schema.safeParse(await req.json());
  if (!parsed.success) return NextResponse.json(parsed.error.flatten(), { status: 400 });

  const key = `uploads/${session.user.id}/${randomUUID()}-${parsed.data.filename}`;

  const url = await getSignedUrl(
    s3,
    new PutObjectCommand({
      Bucket: process.env.R2_BUCKET!,
      Key: key,
      ContentType: parsed.data.contentType,
      ContentLength: parsed.data.size,
    }),
    { expiresIn: 60 } // URL valid for 60 seconds
  );

  return NextResponse.json({
    uploadUrl: url,
    key,
    publicUrl: `https://cdn.example.com/${key}`,    // your custom-domain CDN URL
  });
}
```

### Client — PUT the file directly

```tsx
'use client';
import { useState } from 'react';

async function uploadToR2(file: File) {
  // 1. Get a presigned URL from our server
  const r1 = await fetch('/api/upload-url', {
    method: 'POST',
    body: JSON.stringify({
      filename: file.name,
      contentType: file.type,
      size: file.size,
    }),
  });
  const { uploadUrl, publicUrl } = await r1.json();

  // 2. PUT the file straight to R2
  await fetch(uploadUrl, {
    method: 'PUT',
    body: file,
    headers: { 'Content-Type': file.type },
  });

  return publicUrl;
}

export function ImageUpload() {
  const [url, setUrl] = useState<string>();
  return (
    <input
      type="file"
      onChange={async (e) => {
        const file = e.target.files?.[0];
        if (file) setUrl(await uploadToR2(file));
      }}
    />
  );
}
```

### Critical: validate on the server AFTER upload

Presigned URLs include `ContentType` and `ContentLength` — but a malicious client can lie about the file's actual content. Always run validation **after** the upload completes:

```ts
// app/api/uploads/finalize/route.ts
import { GetObjectCommand, HeadObjectCommand, DeleteObjectCommand } from '@aws-sdk/client-s3';
import { fileTypeFromStream } from 'file-type';
import sharp from 'sharp';

export async function POST(req: Request) {
  const { key } = await req.json();

  // Get object metadata
  const head = await s3.send(new HeadObjectCommand({ Bucket: BUCKET, Key: key }));

  // Optional: re-validate the actual MIME type by sniffing magic bytes
  const obj = await s3.send(new GetObjectCommand({ Bucket: BUCKET, Key: key }));
  const sniffed = await fileTypeFromStream(obj.Body as any);

  if (!sniffed || !['image/jpeg', 'image/png', 'image/webp'].includes(sniffed.mime)) {
    await s3.send(new DeleteObjectCommand({ Bucket: BUCKET, Key: key }));
    return new Response('Invalid file type', { status: 400 });
  }

  // Optional: re-encode with sharp to strip EXIF, normalize, generate thumbnails
  const buffer = await s3.send(new GetObjectCommand({ Bucket: BUCKET, Key: key }));
  const optimized = await sharp(await streamToBuffer(buffer.Body as any))
    .rotate()         // honor EXIF orientation
    .resize({ width: 2000, withoutEnlargement: true })
    .webp({ quality: 85 })
    .toBuffer();
  // ... re-upload optimized version

  return NextResponse.json({ ok: true });
}
```

`file-type@19` is the standard library for magic-byte MIME sniffing. **Don't trust `file.type` from the browser** — it's user-supplied.

### Bucket CORS — the gotcha that always bites you

The browser cannot PUT to R2/S3 unless CORS is configured on the bucket:

```json
[
  {
    "AllowedOrigins": ["https://your-app.com", "http://localhost:3000"],
    "AllowedMethods": ["PUT", "GET"],
    "AllowedHeaders": ["Content-Type"],
    "ExposeHeaders": ["ETag"],
    "MaxAgeSeconds": 3600
  }
]
```

For R2: configure via the Cloudflare dashboard → R2 bucket → Settings → CORS. For S3: bucket → Permissions → CORS configuration. **The first time you set this up, you will spend 30 minutes on a CORS error.** Plan for it.

### Custom domain + CDN

Don't serve files from `https://<account>.r2.cloudflarestorage.com` — slow and ugly URL. Configure a custom domain (`cdn.example.com`) attached to your R2 bucket. Cloudflare's CDN caches automatically. AWS S3 + CloudFront is the equivalent on AWS.

---

## 5. Pattern C — Multipart Upload for Large Files

S3 (and R2) cap a single PUT at **5 GB**. For files between 100 MB and 5 GB, use multipart upload — splits the file into 5–100 MB chunks, uploads in parallel, recombines on the server.

### Why multipart wins for big files

- **Parallel uploads** — 4–8 chunks at once is much faster than one stream.
- **Chunk-level retry** — only re-upload the failed chunk, not the whole file.
- **Progress per chunk** — report per-chunk progress for big-file UIs.
- **5 TB max upload size** (10,000 chunks × 5 GB max each).

### Server — issue presigned URLs for each part

```ts
// app/api/multipart/start/route.ts
import { CreateMultipartUploadCommand, UploadPartCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

export async function POST(req: Request) {
  const { filename, contentType, partCount } = await req.json();

  const key = `uploads/${randomUUID()}-${filename}`;
  const create = await s3.send(
    new CreateMultipartUploadCommand({
      Bucket: BUCKET,
      Key: key,
      ContentType: contentType,
    })
  );
  const uploadId = create.UploadId!;

  // Pre-sign URLs for each part
  const partUrls = await Promise.all(
    Array.from({ length: partCount }, async (_, i) => {
      const url = await getSignedUrl(
        s3,
        new UploadPartCommand({ Bucket: BUCKET, Key: key, UploadId: uploadId, PartNumber: i + 1 }),
        { expiresIn: 3600 }
      );
      return { partNumber: i + 1, url };
    })
  );

  return NextResponse.json({ key, uploadId, partUrls });
}

// app/api/multipart/complete/route.ts
import { CompleteMultipartUploadCommand } from '@aws-sdk/client-s3';

export async function POST(req: Request) {
  const { key, uploadId, parts } = await req.json();
  // parts: [{ partNumber: 1, etag: '...' }, ...]
  await s3.send(
    new CompleteMultipartUploadCommand({
      Bucket: BUCKET,
      Key: key,
      UploadId: uploadId,
      MultipartUpload: { Parts: parts.map((p: any) => ({ PartNumber: p.partNumber, ETag: p.etag })) },
    })
  );
  return NextResponse.json({ ok: true });
}
```

### Client — chunk the file and upload in parallel

```ts
const CHUNK_SIZE = 10 * 1024 * 1024; // 10 MB

async function uploadMultipart(file: File) {
  const partCount = Math.ceil(file.size / CHUNK_SIZE);

  // 1. Get presigned URLs for each part
  const r1 = await fetch('/api/multipart/start', {
    method: 'POST',
    body: JSON.stringify({ filename: file.name, contentType: file.type, partCount }),
  });
  const { key, uploadId, partUrls } = await r1.json();

  // 2. Upload each chunk in parallel
  const parts = await Promise.all(
    partUrls.map(async ({ partNumber, url }: any) => {
      const start = (partNumber - 1) * CHUNK_SIZE;
      const chunk = file.slice(start, start + CHUNK_SIZE);
      const res = await fetch(url, { method: 'PUT', body: chunk });
      return { partNumber, etag: res.headers.get('etag')! };
    })
  );

  // 3. Tell S3 we're done
  await fetch('/api/multipart/complete', {
    method: 'POST',
    body: JSON.stringify({ key, uploadId, parts }),
  });
}
```

In production, throttle concurrent chunks (e.g., 4 at a time) to avoid network saturation. Use `p-limit@5` or a small custom queue.

---

## 6. Pattern D — tus Resumable Upload (Network-Drop-Tolerant)

The four-pattern hierarchy:

| Need | Pattern |
|------|---------|
| Fast & simple | Single PUT (Pattern B) |
| Big files | Multipart (Pattern C) |
| **Mobile users + flaky networks + must-survive-drops** | **tus (Pattern D)** |

`tus` (Tus Resumable Upload Protocol) is an open standard that lets a client resume an interrupted upload from the byte where it stopped. Built by the team at Vimeo (where dropped uploads on mobile = serious revenue loss).

### When tus wins over multipart

- **Mobile uploads** that pause/resume across network changes (cell → wifi → cell).
- **Long uploads** (15+ minutes) where you can't ask the user to start over.
- **Flaky connections** where a single network blip kills a multipart attempt.

### Install

```bash
# Client
npm i tus-js-client@4

# If you want a UI
npm i @uppy/core@4 @uppy/tus@4 @uppy/dashboard@4 @uppy/react@4
```

### Server — Tus is a separate process or service

You have two real options:

#### Option A — `@tus/server@1` on a Node host (Railway / Fly.io)

```ts
// tus-server.ts (separate service, not on Vercel)
import { Server } from '@tus/server';
import { S3Store } from '@tus/s3-store';

const server = new Server({
  path: '/files',
  datastore: new S3Store({
    s3ClientConfig: {
      bucket: process.env.R2_BUCKET!,
      region: 'auto',
      endpoint: process.env.R2_ENDPOINT,
      credentials: {
        accessKeyId: process.env.R2_ACCESS_KEY_ID!,
        secretAccessKey: process.env.R2_SECRET_ACCESS_KEY!,
      },
    },
  }),
});

const httpServer = require('http').createServer(server.handle.bind(server));
httpServer.listen(8080);
```

**Tus servers are long-running; they don't fit on Vercel's serverless functions.** Run on Railway, Fly.io, or any Node host (covered in `07.nextjs/05-deployment.md`).

#### Option B — Hosted tus service

- **Transloadit** — managed tus + image processing.
- **Uploadcare** — hosted upload service (proprietary protocol, not tus, but similar resumable feel).

For most teams: a small Railway service running `@tus/server@1` writing to R2 is the cheapest production-grade tus setup.

### Client — `tus-js-client@4`

```tsx
'use client';
import * as tus from 'tus-js-client';
import { useState } from 'react';

export function ResumableUpload() {
  const [progress, setProgress] = useState(0);
  const [url, setUrl] = useState<string>();

  async function handleFile(file: File) {
    const upload = new tus.Upload(file, {
      endpoint: 'https://uploads.example.com/files',  // your tus server
      retryDelays: [0, 3000, 5000, 10000, 20000],
      metadata: {
        filename: file.name,
        filetype: file.type,
      },
      onError(err) { console.error('Failed:', err); },
      onProgress(uploaded, total) {
        setProgress(Math.round((uploaded / total) * 100));
      },
      onSuccess() {
        setUrl(upload.url!);
      },
    });

    // Check for previous uploads to continue
    const previous = await upload.findPreviousUploads();
    if (previous.length) upload.resumeFromPreviousUpload(previous[0]);

    upload.start();
  }

  return (
    <>
      <input type="file" onChange={(e) => e.target.files?.[0] && handleFile(e.target.files[0])} />
      <progress value={progress} max={100} />
      {url && <a href={url}>Done</a>}
    </>
  );
}
```

The killer detail: `findPreviousUploads()` checks localStorage for an interrupted upload of the same file. **If the user closes the tab mid-upload, reopens it, and selects the same file, the upload resumes from where it stopped.** That's tus's whole point.

### Uppy 4 — full-featured upload UI

If you want the polished UI Vimeo and Wix-style apps use:

```tsx
'use client';
import Uppy from '@uppy/core';
import Tus from '@uppy/tus';
import { Dashboard } from '@uppy/react';
import '@uppy/core/dist/style.css';
import '@uppy/dashboard/dist/style.css';

const uppy = new Uppy({ restrictions: { maxFileSize: 5_000_000_000, maxNumberOfFiles: 10 } })
  .use(Tus, { endpoint: 'https://uploads.example.com/files' });

export function UploadDashboard() {
  return <Dashboard uppy={uppy} proudlyDisplayPoweredByUppy={false} />;
}
```

Get drag-drop, multi-file, progress per file, retries, webcam capture, screen capture, Google Drive / Dropbox / Instagram pickers — all in one component. ~50 KB extra bundle for the Dashboard.

---

## 7. The Drop-Zone UI — `react-dropzone@14`

For pure single-file or multi-file drop UI without Uppy's complexity:

```bash
npm i react-dropzone@14
```

```tsx
'use client';
import { useDropzone } from 'react-dropzone';
import { useCallback } from 'react';

export function FileDrop({ onFile }: { onFile: (f: File) => void }) {
  const onDrop = useCallback((accepted: File[]) => {
    accepted.forEach(onFile);
  }, [onFile]);

  const { getRootProps, getInputProps, isDragActive } = useDropzone({
    onDrop,
    accept: {
      'image/*': ['.jpg', '.jpeg', '.png', '.webp'],
      'application/pdf': ['.pdf'],
    },
    maxSize: 10 * 1024 * 1024,
    maxFiles: 5,
  });

  return (
    <div
      {...getRootProps()}
      className={`rounded-lg border-2 border-dashed p-8 text-center
        ${isDragActive ? 'border-blue-500 bg-blue-50' : 'border-gray-300 bg-gray-50'}`}
    >
      <input {...getInputProps()} />
      <p>{isDragActive ? 'Drop files here' : 'Drag files here or click to browse'}</p>
    </div>
  );
}
```

Pair with any of the upload patterns above:

```tsx
<FileDrop onFile={(file) => uploadToR2(file)} />
```

`react-dropzone@14`:
- Handles drag-enter, drag-leave, drag-over correctly across browsers.
- Filters by MIME type and extension.
- Validates size and count.
- Provides `fileRejections` for showing errors.
- Bundle: ~5 KB gzipped.

---

## 8. Image Optimization — Process Server-Side

Don't trust the browser to send a properly-sized image. The user's iPhone takes 12 MP photos; your UI shows a 200×200 avatar.

### Standard pipeline

```
Browser uploads original → R2 (raw bucket)
   ↓
Server runs sharp:
  - Strip EXIF (privacy + smaller files)
  - Auto-rotate based on orientation
  - Resize to max 2000px
  - Re-encode as WebP at quality 85
  - Generate thumbnails: 64×64, 256×256, 1024×1024
   ↓
Save processed versions → R2 (cdn bucket)
   ↓
Serve via Cloudflare CDN
```

### `sharp@0.33` — the Node image library

```ts
import sharp from 'sharp';

const processed = await sharp(buffer)
  .rotate()                           // honor EXIF orientation
  .resize({
    width: 2000,
    height: 2000,
    fit: 'inside',
    withoutEnlargement: true,         // don't upscale small images
  })
  .webp({ quality: 85 })
  .toBuffer();

// Thumbnails
const thumb = await sharp(buffer)
  .resize(256, 256, { fit: 'cover' })
  .webp({ quality: 80 })
  .toBuffer();
```

### Run in a queue, not the request

`sharp` on a 4K photo takes 200–500ms — too long for an HTTP handler. Run via:

- **Inngest** (`inngest@3`) — event-driven background functions, free tier.
- **Trigger.dev** (`@trigger.dev/sdk@3`) — same model, also free tier.
- **A dedicated worker** on Railway / Fly.io polling a queue.
- **Vercel Cron** for scheduled batch processing (acceptable for non-urgent thumbnails).

```ts
// Server Action that creates the upload row, then enqueues processing
'use server';
import { inngest } from '@/inngest/client';

export async function finalizeUpload(key: string) {
  const upload = await db.upload.create({ data: { key, status: 'pending' } });
  await inngest.send({ name: 'image.process', data: { uploadId: upload.id, key } });
  return upload.id;
}
```

The user sees the upload immediately; thumbnails appear seconds later.

### Skip the pipeline for some cases

- **Cloudflare Images** ($5/mo for 100K images) — automatic resizing on the fly via URL params. No sharp pipeline needed.
- **Vercel Image Optimization** — `next/image` does on-the-fly resize from any URL.
- **`imgproxy@3`** — self-hosted image-proxy service.

For a small SaaS: **`next/image` over R2 URLs** is the path of least resistance. For high volume: Cloudflare Images.

---

## 9. Security — The Real Threats

| Threat | Mitigation |
|--------|------------|
| User uploads `.exe` named `image.png` | Sniff MIME with `file-type@19` after upload |
| User uploads 100 GB file (DoS via storage cost) | Set `ContentLength` cap in presigned URL; reject unknown sizes |
| User PUTs to a URL signed for someone else | Include user ID in the storage key (`uploads/${userId}/...`) and verify on access |
| EXIF geotag leak | Strip with `sharp().rotate()` — implicitly drops EXIF |
| Stored XSS via SVG | Reject `image/svg+xml` content type, OR sanitize with `dompurify@3` if rendering inline |
| Polyglot files (PDF that's also valid HTML) | `file-type@19` sniffs by magic bytes; reject ambiguous results |
| Malware in uploaded files | Scan with `clamav` (self-host) or VirusTotal API for high-trust scenarios |
| Hot-linking driving up egress | Use signed URLs with short TTL for private files; tie public URLs to your own domain so you can rate-limit at the CDN |
| Oversized image DoS the image processor | Pre-check `ContentLength` before queuing for sharp |

### Signed read URLs for private files

For private files (paid course videos, sensitive documents), generate **signed GET URLs** with short TTLs:

```ts
import { GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const url = await getSignedUrl(
  s3,
  new GetObjectCommand({ Bucket: BUCKET, Key: key }),
  { expiresIn: 300 }   // 5 minutes
);
```

The user gets a URL valid for 5 minutes. After that, the link is dead. Prevents URL sharing and link rot.

---

## 10. Storing the Upload — Your DB Schema

```prisma
model Upload {
  id          String       @id @default(cuid())
  userId      String
  key         String       @unique          // S3/R2 key
  publicUrl   String?
  filename    String
  contentType String
  sizeBytes   Int
  status      UploadStatus @default(pending) // pending | processed | failed
  variants    Json?                          // { thumb: 'key.webp', medium: 'key.webp', original: 'key.jpg' }
  createdAt   DateTime     @default(now())
  user        User         @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
}

enum UploadStatus {
  pending
  processed
  failed
}
```

`variants` as `Json` lets you add new sizes (`'sq': 'key-square.webp'`) without schema migrations. Read it with a Zod schema in your code so it stays type-safe.

---

## 11. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Upload works locally, fails in prod with CORS error | Configure CORS on the bucket — see §4 |
| File size limit hits at 4.5 MB on Vercel | You're proxying through Next.js; switch to direct-to-R2/Blob |
| `Content-Type` mismatch between presigned URL and PUT | The browser sometimes sets `Content-Type` itself; pin it in the `fetch` headers |
| Vercel function times out on big upload | You're proxying. Always direct upload. |
| EXIF data leaks user GPS | `sharp().rotate()` strips it; or use `exifr@7` to inspect first |
| User uploads malicious SVG | Reject SVG, OR sanitize with `dompurify@3` if you must accept |
| File appears uploaded but DB row missing | Your `onUploadCompleted` callback failed silently. Add observability (covered in `10.production/04-observability.md`) |
| Multipart works but ETags missing | Bucket CORS doesn't expose `ETag` header. Add `"ExposeHeaders": ["ETag"]` |
| tus server can't run on Vercel | Tus is long-running; deploy to Railway/Fly (covered in `07.nextjs/05-deployment.md`) |
| Sharp install fails on Vercel build | Vercel handles `sharp` natively; if errors, use `--platform=linux --arch=x64` install flags |
| Egress bills are huge | Migrate to Cloudflare R2 ($0 egress) |
| Public URLs expose AWS account ID | Use a custom domain (`cdn.example.com`) attached to the bucket |

---

## 12. Decision Tree

```
What's the upload pattern?
│
├── Single image / avatar / small file (< 100 MB)
│   ├── On Vercel, want zero infra → @vercel/blob client upload (Pattern A)
│   └── Want zero egress fees → S3 SDK + R2 + presigned PUT (Pattern B)
│
├── Big file (100 MB – 5 GB) — videos, datasets
│   → Multipart upload (Pattern C) with parallel chunk PUTs
│
├── Mobile users on flaky networks / long uploads
│   → tus protocol (Pattern D) with @tus/server@1 on Railway + tus-js-client@4
│
└── Want a polished multi-file UI with cloud picker integrations
   → Uppy 4 + Tus or S3 plugin

What storage provider?
│
├── On Vercel, simple → @vercel/blob
├── Public-served files (images, video, downloads) → Cloudflare R2 + Cloudflare CDN
├── Already on AWS → S3 + CloudFront
├── Backups / cold storage → Backblaze B2 + Cloudflare CDN (free egress)
└── Indie SaaS shipping fast → UploadThing

What about image processing?
│
├── Low volume, simple needs → next/image on top of stored originals
├── Real product → server pipeline with sharp@0.33 in a queue (Inngest / Trigger.dev)
└── Massive volume → Cloudflare Images ($5/mo for 100K) or imgproxy
```

---

## 13. What This Topic Connects To

- **`07.nextjs/06-api-routes.md`** — Route Handlers issuing presigned URLs.
- **`07.nextjs/04-auth.md`** — Authenticating presigned-URL requests.
- **`07.nextjs/05-deployment.md`** — Where to host the long-running tus server.
- **`10.production/04-observability.md`** — Tracking upload success / failure rates.
- **`08.ecosystem/04-real-project.md`** — Adding image attachments to the Task Manager is a Pattern B job.

---

## Summary

| Pattern | Pick when |
|---------|-----------|
| `@vercel/blob` direct upload | On Vercel, want zero infra |
| Direct-to-R2/S3 with presigned URL | Single PUT, < 100 MB, want $0 egress |
| Multipart upload | Files 100 MB – 5 GB |
| tus + `@tus/server@1` + `tus-js-client@4` | Network-drop-tolerant; mobile uploads |
| Uppy 4 + Tus / S3 plugin | Want polished multi-file UI with cloud pickers |
| `react-dropzone@14` | Tiny drag-drop UI without Uppy's overhead |

| Rule | Why |
|------|-----|
| Never upload through your Next.js function | Bandwidth, memory, time costs scale with file size |
| Always validate MIME with `file-type@19` after upload | Browser-supplied content type is user input |
| Configure bucket CORS once before launch | First-attempt CORS error wastes 30 minutes |
| Use Cloudflare R2 for public files | $0 egress beats S3's $0.09/GB |
| Process images in a queue, not the request | `sharp` on big images is too slow for HTTP |
| Use signed read URLs for private content | Prevents link sharing + URL theft |

---

## Further reading

- [Vercel Blob docs](https://vercel.com/docs/storage/vercel-blob)
- [AWS S3 — Presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html)
- [Cloudflare R2 docs](https://developers.cloudflare.com/r2/)
- [tus protocol](https://tus.io/)
- [@tus/server](https://github.com/tus/tus-node-server)
- [tus-js-client](https://github.com/tus/tus-js-client)
- [Uppy docs](https://uppy.io/docs/)
- [react-dropzone docs](https://react-dropzone.js.org/)
- [sharp docs](https://sharp.pixelplumbing.com/)
- [file-type docs](https://github.com/sindresorhus/file-type)
