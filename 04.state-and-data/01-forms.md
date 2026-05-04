# Topic 07 — Forms: Controlled vs Uncontrolled, react-hook-form 7 vs Formik 2

> **What / Why / How** — controlled forms are the default tutorial answer and the wrong default for real apps.

---

## 1. What Is a "Controlled" Component?

### What

A **controlled** input is one where React state is the single source of truth — the input's `value` comes from state, and changes flow through a state setter:

```tsx
function Controlled() {
  const [email, setEmail] = useState('');
  return <input value={email} onChange={(e) => setEmail(e.target.value)} />;
}
```

An **uncontrolled** input lets the DOM hold its own value; you read it on submit:

```tsx
function Uncontrolled() {
  const ref = useRef<HTMLInputElement>(null);
  function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    console.log(ref.current!.value);
  }
  return <input ref={ref} defaultValue="" />;
}
```

### Why controlled is the React tutorial default

Controlled inputs make:
- Real-time validation easy (`if (email.length > 100) showError()`).
- Conditional rendering of dependent UI easy (`{email.includes('@') && <Tip>` ).
- State serialization (e.g., for autosave, undo/redo) trivial.

### Why controlled is the wrong default for real apps

Every keystroke causes a re-render of the entire form component. For a 30-field form, that's 30 components re-rendering on every key press — measurable jank on mid-range mobile.

Real measurement: typing into one input on a 30-field controlled form re-renders all 30 components. The same form with `react-hook-form@7.51` re-renders only the changed input.

---

## 2. The Three Practical Approaches

| Approach | Re-render scope | Validation | Best for |
|----------|-----------------|------------|----------|
| Controlled with `useState` | Whole form per keystroke | Manual | < 5 fields, simple cases |
| Uncontrolled + FormData | None during typing | On submit only | Server Actions, simple submit-only forms |
| `react-hook-form@7.51` | Only the changed field | Built-in + Zod resolver | Real production forms |

---

## 3. Approach 1: Controlled with `useState` (when it's fine)

```tsx
function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState<string | null>(null);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError(null);
    try {
      await api.login(email, password);
    } catch (err) {
      setError('Invalid credentials');
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <input type="email" value={email} onChange={(e) => setEmail(e.target.value)} required />
      <input type="password" value={password} onChange={(e) => setPassword(e.target.value)} required />
      {error && <p className="error">{error}</p>}
      <button type="submit">Log in</button>
    </form>
  );
}
```

Two fields, no real-time validation needed beyond HTML5 `required`. Don't reach for `react-hook-form` here.

---

## 4. Approach 2: Uncontrolled + FormData (Server Actions style)

Best fit when the form has many fields and you don't need per-keystroke logic. Native HTML, minimal React state.

```tsx
function ProfileForm({ defaultUser }: { defaultUser: User }) {
  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const data = Object.fromEntries(new FormData(e.currentTarget));
    api.updateProfile(data);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input name="name"  defaultValue={defaultUser.name} />
      <input name="email" defaultValue={defaultUser.email} />
      <textarea name="bio" defaultValue={defaultUser.bio} />
      <button type="submit">Save</button>
    </form>
  );
}
```

### With Next.js 14 Server Actions, this gets even cleaner

```tsx
// app/profile/actions.ts
'use server';
export async function updateProfile(formData: FormData) {
  const data = Object.fromEntries(formData);
  await db.update(users).set(data).where(eq(users.id, currentUser.id));
}

// app/profile/page.tsx
import { updateProfile } from './actions';

export default function ProfilePage({ user }) {
  return (
    <form action={updateProfile}>
      <input name="name"  defaultValue={user.name} />
      <input name="email" defaultValue={user.email} />
      <button type="submit">Save</button>
    </form>
  );
}
```

No `useState`, no `onChange`, no `e.preventDefault()`. Combine with `useActionState` (React 19) for pending state and validation results.

---

## 5. Approach 3: `react-hook-form@7.51` — Production Default

### Why this beats Formik 2

| Feature | Formik 2.4 | react-hook-form 7.51 |
|---------|-----------|----------------------|
| Re-render strategy | Re-renders entire form on keystroke | Subscribes per field — only changed field re-renders |
| Bundle size (gzipped) | ~13 KB | ~8.5 KB |
| Uncontrolled (refs) by default | ❌ Controlled | ✅ Uncontrolled (refs) — that's the perf win |
| TypeScript support | Patchy generics | Excellent — typed by `register('field')` |
| Last meaningful release | June 2022 (stale) | Active (monthly releases) |
| Zod / Yup / Joi resolvers | Yup only via Formik bindings | First-class resolver packages |

The Formik project has effectively gone dormant. `react-hook-form` is the React community's de-facto answer for production forms.

### Real example with Zod 3 validation

```bash
npm i react-hook-form@7.51 zod@3.23 @hookform/resolvers@3.4
```

```tsx
'use client';
import { useForm } from 'react-hook-form';
import { z } from 'zod';
import { zodResolver } from '@hookform/resolvers/zod';

const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8, 'At least 8 characters'),
  age: z.coerce.number().min(18, 'Must be 18+'),
});

type FormData = z.infer<typeof schema>;

function SignupForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<FormData>({ resolver: zodResolver(schema) });

  async function onSubmit(data: FormData) {
    await api.signup(data); // data is typed AND validated
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email')} />
      {errors.email && <p>{errors.email.message}</p>}

      <input type="password" {...register('password')} />
      {errors.password && <p>{errors.password.message}</p>}

      <input type="number" {...register('age')} />
      {errors.age && <p>{errors.age.message}</p>}

      <button disabled={isSubmitting}>Sign up</button>
    </form>
  );
}
```

What you get for free:
- Each field is uncontrolled — no re-renders during typing.
- Submit is blocked until validation passes.
- TypeScript infers the entire `FormData` from the Zod schema.
- The same `schema` works on the server (Next.js Server Action) for end-to-end validation.

### Watching values when you actually need them

```tsx
const password = useWatch({ control, name: 'password' });
// Only the component using useWatch re-renders when password changes,
// not the whole form.
```

---

## 6. Wiring Up Custom UI Components — `Controller`

Some component libraries (Radix UI's Select, react-datepicker, MUI's Autocomplete) don't accept native `ref` registration. Use `<Controller>`:

```tsx
import { Controller, useForm } from 'react-hook-form';
import { Select } from '@radix-ui/react-select';

function MyForm() {
  const { control, handleSubmit } = useForm<{ country: string }>();

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <Controller
        name="country"
        control={control}
        render={({ field }) => (
          <Select value={field.value} onValueChange={field.onChange}>
            ...
          </Select>
        )}
      />
    </form>
  );
}
```

---

## 7. Server-Side Validation — Don't Skip It

Real principle: **client validation is for UX; server validation is for security**. Run the same Zod schema on both sides:

```ts
// schemas/signup.ts — shared
import { z } from 'zod';
export const signupSchema = z.object({ ... });

// app/api/signup/route.ts (Next.js Route Handler)
export async function POST(req: Request) {
  const result = signupSchema.safeParse(await req.json());
  if (!result.success) {
    return Response.json({ errors: result.error.flatten() }, { status: 400 });
  }
  // result.data is now typed and trustworthy
}
```

This pattern is used by `t3-stack` (`create-t3-app`) and is one of the strongest reasons to pick the Next.js + tRPC + Zod combo (covered in Topic 24 and 29).

---

## 8. File Uploads with Forms

```tsx
const { register, handleSubmit } = useForm<{ avatar: FileList }>();

<form onSubmit={handleSubmit(async ({ avatar }) => {
  const file = avatar[0];
  const formData = new FormData();
  formData.append('file', file);
  await fetch('/api/upload', { method: 'POST', body: formData });
})}>
  <input type="file" accept="image/*" {...register('avatar')} />
  <button>Upload</button>
</form>
```

For resumable / chunked uploads: `tus-js-client@4` (used by Vimeo) or `uppy@4` (used by Transloadit). Topic 30+ later if needed.

---

## Summary

| Form scenario | Recommended approach |
|---------------|----------------------|
| 1–3 fields, simple | Controlled `useState` |
| Many fields, no per-keystroke logic, Next.js Server Actions | Uncontrolled + FormData |
| Real production form with validation | `react-hook-form@7.51` + `zod@3.23` |
| Custom UI components (Select, DatePicker) | `react-hook-form`'s `<Controller>` |

| Rule | Why |
|------|-----|
| Don't pick Formik in 2026 | Stale, larger, slower than `react-hook-form` |
| Use the same Zod schema on client and server | Single source of truth, type-safe |
| Server validation is mandatory | Client checks are for UX, not security |

---

## Further reading

- [react-hook-form docs](https://react-hook-form.com/get-started)
- [Zod docs](https://zod.dev/)
- [Next.js — Server Actions and Forms](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)
- [react.dev — Sharing State Between Components](https://react.dev/learn/sharing-state-between-components)
