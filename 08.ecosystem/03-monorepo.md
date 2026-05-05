# Ecosystem — 03. Monorepo: Turborepo 2 vs Nx 19 for Shared React Component Libraries

> **What / Why / How** — pick `turbo@2` for "two Next.js apps + a shared UI package". Pick `nx@19` for "we have 80 services and need code generators." Don't reach for Yarn workspaces alone in 2026.

---

## 1. The Decision Up Front

Two real tools, both production-proven:

| | **Turborepo 2** (`turbo@2.1`) | **Nx 19** (`nx@19`) |
|--|------------------------------|---------------------|
| Made by | Vercel (acquired 2021) | Nrwl (acquired by ServiceNow 2024) |
| Mental model | "npm scripts with caching + dependency-aware ordering" | "First-class workspace tooling with code gen, plugins, generators" |
| Setup time for first app | ~5 minutes | ~15 minutes |
| Best at | Two Next.js apps + 3 shared packages | 50+ packages with consistent tooling, polyglot stacks |
| Plugin ecosystem | Lean | Huge (`@nx/react`, `@nx/next`, `@nx/expo`, `@nx/storybook`, etc.) |
| Code generators | None | `nx g @nx/react:lib my-feature` scaffolds everything |
| Distributed remote cache | Vercel Remote Cache (free with Vercel account, or self-host with `@ducktors/turborepo-remote-cache`) | Nx Cloud (paid SaaS or self-host) |
| Used by | Vercel.com, Cal.com, Adobe, Disney+ | Microsoft (vscode-eslint), Capital One, Storybook |

**The 80% rule for new React projects in 2026: pick `turbo@2`.** It's the lighter tool, plays beautifully with Next.js (same vendor), and won't lock you into a specific package layout. Reach for Nx when you need code generators, plugin-based tooling, or a polyglot graph (Java + React + Python).

---

## 2. Why a Monorepo at All?

Real reasons (every one of these is something you'd otherwise solve worse):

| Problem | Monorepo solution |
|---------|-------------------|
| Two Next.js apps share a `<Button>` component | One `packages/ui` workspace consumed by both |
| Frontend types must match backend types | One `packages/shared` exports Zod schemas + TS types used by tRPC server **and** React forms |
| API contract changes break consumers silently | Workspace-aware TypeScript checking flags the breakage at PR time |
| Different repos drift on `eslint` / `prettier` / `tsconfig` | Single source of truth at the root |
| Coordinating a release across 5 packages | Versioning tooling (`changesets@2`) handles the cascade |

When **not** to use a monorepo:
- Single app, no shared internal libraries → just a regular repo.
- Teams that aren't ready to share code (Conway's Law: monorepo without coordination = pain).
- Heavy CI compute constraints with no remote cache — first builds without cache get expensive.

---

## 3. The Standard Monorepo Layout (Both Tools Use It)

```
my-monorepo/
├── apps/
│   ├── web/                    ← Next.js 14 marketing site
│   ├── dashboard/              ← Next.js 14 logged-in app
│   └── docs/                   ← Next.js 14 with @next/mdx
├── packages/
│   ├── ui/                     ← shared shadcn/ui-style components
│   ├── eslint-config/          ← shared eslint config
│   ├── tsconfig/               ← shared base tsconfig.json
│   ├── shared/                 ← Zod schemas, types
│   └── db/                     ← Prisma schema + client
├── package.json                ← root, with workspaces field
├── turbo.json                  ← Turborepo config (or nx.json + project.json for Nx)
└── pnpm-workspace.yaml         ← if using pnpm (recommended)
```

`apps/*` are deployable. `packages/*` are reusable libraries. This split is unofficial-standard; both Turborepo and Nx examples follow it.

---

## 4. Package Manager — pnpm Is the 2026 Default

| | npm 10 | yarn 4 | pnpm 9 | bun 1.x |
|--|--------|--------|--------|---------|
| Workspace support | Yes | Yes (Berry) | Yes (best in class) | Yes |
| Disk usage for monorepo | High (duplicates) | Medium | Low (content-addressable store) | Low |
| Strict dependency resolution | No | Optional | Yes (default) | Partial |
| Default in Vercel deploys | npm | npm | pnpm (if pnpm-lock detected) | bun |
| Used by | Many | Yarn-history projects | Vercel monorepos, Vue, Vite, Astro, Prisma | Bun-native projects |

**Pick `pnpm@9`** for any monorepo unless you have specific reason not to. It's ~2× faster than npm, uses ~3× less disk, and its strict resolution catches "accidentally working because of hoisting" bugs that npm/yarn miss.

```bash
npm i -g pnpm@9
pnpm init
echo "packages:\n  - 'apps/*'\n  - 'packages/*'" > pnpm-workspace.yaml
```

---

## 5. Turborepo 2 — The Default Pick

### Setup with `create-turbo`

```bash
pnpm dlx create-turbo@latest my-monorepo
cd my-monorepo
```

The starter scaffolds:
- `apps/web` and `apps/docs` — two Next.js 14 apps
- `packages/ui` — shared React components
- `packages/eslint-config` and `packages/tsconfig`
- `turbo.json` at the root

### `turbo.json` — the brain of the build pipeline

```jsonc
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],            // build dependencies first
      "outputs": [".next/**", "!.next/cache/**", "dist/**"],
      "env": ["NODE_ENV", "DATABASE_URL"]
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    },
    "lint": {
      "dependsOn": ["^lint"],
      "outputs": []
    },
    "dev": {
      "cache": false,
      "persistent": true                  // long-running task, never cache
    },
    "type-check": {
      "dependsOn": ["^type-check"],
      "outputs": []
    }
  }
}
```

The `^` prefix means "run this task in dependencies first." So `build` for `apps/web` will run `build` in `packages/ui` first.

### Run tasks across the workspace

```bash
turbo build              # build everything
turbo build --filter=web # build only apps/web (and its deps)
turbo dev                # start dev for everything that has a `dev` script
turbo test --filter=...packages/ui  # test only the ui package
turbo type-check
```

Every command is **content-hashed** — Turborepo computes a hash of inputs (source files, dep versions, env vars listed in `env`) and skips the task if the hash matches a previous run. Cached output is written to `.turbo/`.

### Real example: `packages/ui` consumed by `apps/web`

```jsonc
// packages/ui/package.json
{
  "name": "@my/ui",
  "version": "0.0.0",
  "private": true,
  "main": "./src/index.ts",     // direct TS export — no build step needed in dev
  "types": "./src/index.ts",
  "exports": {
    ".": "./src/index.ts",
    "./button": "./src/button.tsx",
    "./styles.css": "./src/styles.css"
  },
  "scripts": {
    "lint": "eslint .",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "react": "^18.3.0",
    "@radix-ui/react-slot": "^1.1.0"
  }
}
```

```jsonc
// apps/web/package.json
{
  "name": "web",
  "dependencies": {
    "@my/ui": "workspace:*",   // pnpm workspace protocol — always uses local source
    "next": "^14.2.0",
    "react": "^18.3.0",
    "react-dom": "^18.3.0"
  }
}
```

```tsx
// apps/web/app/page.tsx
import { Button } from '@my/ui/button';

export default function Page() {
  return <Button>Hello</Button>;
}
```

`workspace:*` is the pnpm/yarn protocol that says "use the workspace package, never npm." When the version is bumped for publishing, tooling rewrites it to a real version.

### Source-mode vs build-mode for shared packages

Two real strategies:

| Strategy | When | Cost |
|----------|------|------|
| **Source mode** (`main: src/index.ts`) | All consumers are TS Next.js apps; you transpile via `transpilePackages` in `next.config.js` | Zero — Next.js compiles your TS package along with the app |
| **Build mode** (`main: dist/index.js`) | Mixed-language consumers; published to npm | Need a build step (`tsup@8`, `vite@5` lib mode) and `dependsOn: ["^build"]` |

For internal-only Next.js monorepos in 2026: **source mode**. Add to `apps/web/next.config.js`:

```js
module.exports = {
  transpilePackages: ['@my/ui', '@my/shared'],
};
```

No build step in `packages/ui`, no stale-cache issues, instant updates during dev.

### Filtering — the killer feature

```bash
# Affected since main
turbo build --filter='[main]'

# Specific app and its deps
turbo build --filter=web

# A package and everything that depends on it
turbo build --filter=@my/ui...

# Everything except an app
turbo build --filter='!docs'
```

Combined with CI: only run tests/builds for what changed. Cuts CI time dramatically on real repos.

### Remote cache — Vercel Remote Cache

```bash
turbo login
turbo link
```

Now any cached result from one machine (your laptop, CI, another dev) is shared via the Vercel Remote Cache. Free for any team using Vercel; fully self-hostable via [`@ducktors/turborepo-remote-cache`](https://github.com/ducktors/turborepo-remote-cache).

The CI win: a colleague's PR build that hashes to `abc123` becomes a cache hit on your CI run for the same commit. Cuts cold-build CI from ~5 min to ~30 sec on real repos.

---

## 6. Nx 19 — When You Need More Than Caching

### What Nx adds beyond Turborepo

- **Code generators** (`nx g @nx/react:lib my-feature`) — scaffold a complete library with tests, lint config, tsconfig, project.json in one command.
- **Plugins per language/framework** — `@nx/next@19`, `@nx/react@19`, `@nx/expo@19`, `@nx/storybook@19`, `@nx/playwright@19`, `@nx/cypress@19`, `@nx/jest@19`, `@nx/vite@19`. Each plugin understands its tool and integrates the build cache.
- **Project graph visualizer** — `nx graph` opens a browser visualization of the dependency graph. For 100-package repos this is genuinely useful.
- **Affected commands** — `nx affected -t test` runs tests only for projects affected by changes since the base branch (Turborepo has the same with `--filter='[main]'`).
- **Module boundaries** — enforce "the `web` app cannot import from `mobile`" at lint time via the `@nx/enforce-module-boundaries` ESLint rule.
- **Migrations** (`nx migrate latest`) — automatic codemods when a plugin version bumps.

### Setup with `create-nx-workspace`

```bash
pnpm dlx create-nx-workspace@19 my-workspace --preset=next
```

Choose preset: `next`, `react`, `react-monorepo`, `ts`, etc. Each scaffolds with sensible defaults.

### `nx.json` — root config

```jsonc
{
  "namedInputs": {
    "default": ["{projectRoot}/**/*", "sharedGlobals"],
    "production": [
      "default",
      "!{projectRoot}/.eslintrc.json",
      "!{projectRoot}/**/?(*.)+(spec|test).[jt]s?(x)",
      "!{projectRoot}/tsconfig.spec.json"
    ]
  },
  "targetDefaults": {
    "build": {
      "cache": true,
      "dependsOn": ["^build"],
      "inputs": ["production", "^production"]
    },
    "test": {
      "cache": true,
      "inputs": ["default", "^production"]
    }
  }
}
```

Each project also has its own `project.json` describing how it builds, tests, and lints.

### Generators in action

```bash
# Generate a new React library
nx g @nx/react:library --name=ui --directory=libs/ui --bundler=vite

# Generate a Next.js app
nx g @nx/next:application --name=dashboard --directory=apps/dashboard

# Generate a component inside the ui library
nx g @nx/react:component --name=button --project=ui --export
```

This is the headline Nx feature. Every generator creates a consistent, lint-passing, test-ready scaffold. For a 50-developer team where consistency matters, this is huge.

### Affected runs

```bash
nx affected -t lint                   # lint only changed projects
nx affected -t test --base=main       # test what changed since main
nx affected -t build --parallel=4
```

### When Nx is the right call

- **Polyglot stack**: React + Node + Python + Go all in one repo. Nx has plugins for many ecosystems.
- **50+ projects**: the generator and plugin system pays off heavily at scale.
- **Strict module boundaries** matter: the `enforce-module-boundaries` rule plus tags lets you encode "feature libraries can't import from each other, only from shared."
- **You want managed remote caching as a paid SaaS**: Nx Cloud is more polished than self-hosting Turbo's remote cache.

### When Nx is overkill

- Two Next.js apps, three shared packages, five developers → use Turborepo. Nx's overhead pays for itself only at scale.

---

## 7. Versioning & Publishing — `changesets@2`

For monorepos with packages you publish to npm (or even just for tracking internal "what changed" history), `@changesets/cli@2` is the standard tool. Works equally well with Turborepo and Nx.

```bash
pnpm dlx @changesets/cli init
```

Workflow:

```bash
# Developer adds a "changeset" to their PR
pnpm changeset
# Choose which packages changed, pick semver bump (patch/minor/major), write a summary

# When merging, CI runs:
pnpm changeset version    # rewrites package.json versions + CHANGELOGs
pnpm changeset publish    # publishes to npm
```

The [Changesets GitHub Action](https://github.com/changesets/action) automates this: opens a "Version Packages" PR that, when merged, publishes to npm.

This is what Vercel's, Astro's, Stitches', Storybook's, and most other prominent JS monorepos use.

---

## 8. CI/CD — The Real Recipe

### Turborepo + GitHub Actions + Vercel Remote Cache

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request: {}
env:
  TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
  TURBO_TEAM: ${{ vars.TURBO_TEAM }}

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 2 }
      - uses: pnpm/action-setup@v4
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo lint type-check test build --filter='[origin/main]'
```

`--filter='[origin/main]'` runs only what's changed since main. Combined with remote cache: most PR builds finish in seconds.

### Nx + GitHub Actions

```yaml
- run: pnpm dlx nx-cloud start-ci-run --distribute-on="3 linux-medium-js"
- run: pnpm exec nx affected -t lint test build --parallel=3
- run: pnpm dlx nx-cloud stop-all-agents
```

Nx Cloud distributes the affected tasks across agents — true parallel CI without writing matrix boilerplate.

---

## 9. Shared `tsconfig`, `eslint`, `prettier`

The pattern both ecosystems use:

```jsonc
// packages/tsconfig/base.json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "esModuleInterop": true
  }
}
```

```jsonc
// apps/web/tsconfig.json
{
  "extends": "@my/tsconfig/base.json",
  "compilerOptions": {
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"]
}
```

Same idea for ESLint:

```js
// packages/eslint-config/next.js
module.exports = {
  extends: ['next/core-web-vitals', 'eslint:recommended', '@my/eslint-config/base'],
  // ...
};
```

`apps/web/.eslintrc.js`:
```js
module.exports = { extends: ['@my/eslint-config/next'] };
```

ESLint v9's flat config (`eslint.config.js`) is recommended for new projects but the older `.eslintrc` style still works. As of mid-2026, both Turborepo's and Nx's starters have moved to flat config.

---

## 10. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Two `react` versions resolved (one in `apps/web`, one in `packages/ui`) | Hoist `react` to the root with `pnpm`'s `peerDependencies` declaration in `packages/ui` |
| `Next.js Module not found: Can't resolve '@my/ui'` | Add `transpilePackages: ['@my/ui']` in `next.config.js` |
| Stale build because environment variable wasn't tracked | Add it to `env` array in `turbo.json` for the relevant task |
| Cache hits in CI but content changed | Wrong `inputs` config — globs need to actually cover the changed files |
| `workspace:*` versions ending up in published `package.json` | The publish tool (changesets, npm publish) usually rewrites these; verify before tagging |
| `nx affected` says nothing affected when something obviously changed | Check `--base` and `--head` flags; defaults differ between local and CI |
| Long-running `dev` task hangs Turborepo | Set `"persistent": true, "cache": false` for dev tasks |
| `tsc` complains about phantom imports between packages | Add `"composite": true` and TypeScript project references, OR set up path aliases consistently |

---

## 11. Decision Tree

```
Is this a single Next.js app with no shared internal libraries?
│   YES → Don't use a monorepo. Just one repo.
│   NO  ↓
Are you building 2–10 packages, mostly Next.js / React / TypeScript?
│   YES → turbo@2 + pnpm@9 + transpilePackages source mode
│   NO  ↓
50+ packages, polyglot stack, code-generation needs, strict module boundaries?
│   YES → nx@19 + Nx Cloud
│   NO  ↓
Just want yarn-workspaces-without-the-cache?
│   YES → pnpm@9 workspaces alone is fine for very small monorepos.
│         Add turbo@2 the moment task ordering or caching matters.
```

---

## 12. Migration Notes

### From multiple repos → Turborepo

1. `pnpm init` at a new repo root, add `pnpm-workspace.yaml`.
2. `git subtree add --prefix=apps/web ../web main` (preserves history) — repeat per existing repo.
3. Move shared code to `packages/*` workspaces.
4. Add `turbo.json` with `build`, `lint`, `test` tasks.
5. Update CI to one consolidated workflow.

This is well-documented at [turbo.build/repo/docs/guides/from-multi-to-monorepo](https://turbo.build/repo/docs/guides/from-multi-to-monorepo).

### From Lerna → Turborepo

Lerna is effectively maintenance-only. Migration is straightforward:
1. Replace `lerna run build` with `turbo run build`.
2. Move `lerna.json` task config into `turbo.json`'s `tasks`.
3. Keep `changesets@2` for versioning (Lerna's versioning still works but is deprecated).

### From Turborepo → Nx (rare, but happens at scale)

Usually triggered by needing code generators, polyglot support, or stricter boundary rules. Nx provides `nx init` that adds Nx on top of an existing pnpm workspace without removing Turborepo first — you can run them side-by-side during transition.

---

## Summary

| Tool | Pick when |
|------|-----------|
| `turbo@2.1` + `pnpm@9` | Default for any new React/Next.js monorepo with 2–10 packages |
| `nx@19` + Nx Cloud | 50+ packages, polyglot stack, generators matter, paid distributed cache welcome |
| `pnpm@9` workspaces alone | Tiny monorepo (2 packages, no CI cache yet) |
| `@changesets/cli@2` | Versioning + npm publishing for any monorepo |

| Rule | Why |
|------|-----|
| Default to pnpm | Faster, smaller disk, strict resolution catches bugs |
| Use `transpilePackages` for shared TS packages in Next.js | No build step for internal libs in dev |
| Pin shared `tsconfig` and `eslint-config` to `packages/*` | Single source of truth across apps |
| Track env vars in `turbo.json`'s `env` field | Otherwise builds cache incorrectly |
| Use `[main]` filter in CI | Run only what changed |
| Add Vercel Remote Cache from day one | CI builds become near-instant |
| Adopt `changesets@2` before publishing v1 | Manual versioning doesn't scale |

---

## Further reading

- [Turborepo docs](https://turbo.build/repo/docs)
- [`create-turbo` starter](https://github.com/vercel/turborepo/tree/main/examples/basic)
- [Nx docs](https://nx.dev/)
- [Nx React tutorial](https://nx.dev/getting-started/tutorials/react-monorepo-tutorial)
- [Changesets docs](https://github.com/changesets/changesets)
- [pnpm workspaces](https://pnpm.io/workspaces)
- [Vercel Remote Cache](https://vercel.com/docs/monorepos/remote-caching)
- [Nx Cloud](https://nx.app/)
