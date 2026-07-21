# Base project

Last verified: 2026-07-21

**Purpose:** Create the Next.js project that everything else builds on: TypeScript, Tailwind, App Router, shadcn/ui components.

## Install

Use pnpm if available, otherwise npm. Replace `my-app` with the kebab-case version of the user's app name.

```bash
npx create-next-app@latest my-app --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --turbopack --use-pnpm
cd my-app
```

If a prompt still appears for an option not covered by flags, accept the default.

Initialize shadcn/ui with defaults:

```bash
pnpm dlx shadcn@latest init -d
```

Add components on demand as pages need them (start small):

```bash
pnpm dlx shadcn@latest add button card input label
```

## Configure

Project layout to follow for everything added later:

```
src/
├── app/           # routes: page.tsx, layout.tsx, api/
├── components/    # shared React components (shadcn/ui lands in components/ui)
└── lib/           # db, auth, utilities
```

Create `.env` at the project root now (empty is fine) and confirm `.env*` is in `.gitignore` — later steps append to it.

## Verify

- `pnpm dev` starts without errors and http://localhost:3000 renders.
- A shadcn `Button` imported into `src/app/page.tsx` renders styled.
