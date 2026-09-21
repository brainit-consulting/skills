# Base project

Last verified: 2026-09-21

**Purpose:** Create the Next.js project that everything else builds on: TypeScript, Tailwind, App Router, shadcn/ui components.

## Install

**The app is created in the current working directory, never in a subfolder.** The user is already standing in the folder they want the app to live in, so `package.json`, `src/`, and `next.config.ts` belong at its top level. Passing `.` as the project name does this — and there is no `cd` afterwards.

**Use pnpm if available, otherwise npm** — and if it's npm, drop `--use-pnpm` from the commands below and read every later `pnpm x` in this skill as `npm run x` (`pnpm dlx` becomes `npx`, `pnpm install` becomes `npm install`). The commands are written for pnpm because it's the recommended path; they are not a requirement, and a Verify step that checks `pnpm dev` on an npm project is checking the wrong thing.

**pnpm under an agent needs `CI=true`.** With no TTY, `pnpm install` aborts with `ERR_PNPM_ABORTED_REMOVE_MODULES_DIR_NO_TTY` rather than prompting, and `pnpm add` can silently do nothing. Prefix pnpm commands with `CI=true` for the whole build, and don't debug the package that "didn't install" — check for that error first.

### Check the folder before running anything

Two things decide which command to run, and both are cheaper to check than to discover.

**The folder name.** `.` makes `create-next-app` derive the package name from the folder, and npm rejects names with **capital letters**, spaces, or a leading dot — so in `H:\GroomRoom` or `~/MyApp` the command fails outright with *"name can no longer contain capital letters"* before anything is created. Folders named after the app are exactly the common case.

**What is already in it.** The installer refuses a folder that isn't empty, and a folder that already holds this skill's own files, a `DESIGN.md`, a `README.md`, a `.gitignore` or notes counts as not empty. This is the normal case, not the exception.

If the folder already holds a **project** — a `package.json`, a `src/`, a `.git` with commits — **stop.** Do not scaffold over it, and do not work around it. Name the folder, say what you found, and ask the user to open an empty folder and start again. Merging a fresh Next.js app into someone's existing repo is the one mistake in this skill that can't be undone.

### Empty folder with a lowercase, npm-legal name

```bash
npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --use-pnpm --no-react-compiler --agents-md --disable-git
```

### Any other folder: scaffold into a temporary folder and move it up

Use this when the folder name is not npm-legal, or when the folder holds stray files that do not amount to a project. Do *not* fall back to a subfolder. The folder keeps its name and the package gets a legal one.

```bash
npx create-next-app@latest scaffold-tmp --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --use-pnpm --no-react-compiler --agents-md --disable-git
```

The temporary folder must be `scaffold-tmp`, not `.scaffold-tmp`. The name cannot start with a dot: the installer treats it as a package name, npm rejects a leading dot, and the command refuses.

Then move everything up **except `node_modules` and `.next`**. Moving a pnpm `node_modules` on Windows is slow and can break its links; reinstalling from the lockfile takes seconds.

**Look before you move.** `mv` silently overwrites, so decide about collisions while both files still exist — not afterwards, when the user's version is already gone. This loop moves what does not collide and names what does:

```bash
shopt -s dotglob
for f in scaffold-tmp/*; do
  name="$(basename "$f")"
  case "$name" in node_modules|.next) continue ;; esac
  if [ -e "./$name" ]; then echo "COLLIDES: $name"; else mv "$f" .; fi
done
```

Handle each collision deliberately: keep the user's file where it has content worth keeping (their `README.md`), take the scaffold's otherwise, and merge `.gitignore` by hand if both exist. Only then delete the temporary folder and install at the root:

```bash
rm -rf scaffold-tmp && CI=true pnpm install
```

### What the flags are for

Passing any flag makes the installer skip its questions and take its defaults for the rest, so nothing prompts. If a prompt ever does appear for an option the flags don't cover, accept the default.

Three of the flags are there to make a choice visible instead of inherited:

- `--no-react-compiler`.
- `--agents-md`, which writes an `AGENTS.md` and a `CLAUDE.md` pointing at the framework docs bundled in `node_modules`, so every agent that later works in the project reads current docs instead of remembering old ones.
- `--disable-git`, because you run `git init` yourself at the project root and a scaffold made in a temporary folder should not bring a nested repository with it.

Turbopack is the default now and needs no flag.

**Leave the scaffold's TypeScript and ESLint pins alone.** The template pins both below the newest release on purpose, because the lint config it installs does not support the newest yet. "Upgrade everything to latest" after scaffolding breaks linting. *Latest stable* here means what the current scaffold gives you, not what the registry's newest tag says.

### Name the package

Set the package name either way, because the derived one is the folder (or `scaffold-tmp`), which is rarely the app's name:

```bash
npm pkg set name=my-app   # kebab-case version of the user's app name
```

### Components

Initialize shadcn/ui with defaults:

```bash
pnpm dlx shadcn@latest init -d
```

**Know what the default gives you.** `-d` now installs components built on Base UI, not Radix. Both are maintained; take the default unless the user has a reason not to (`-b radix` selects the other). It changes how components are written, so say it once to anything that writes UI afterwards: Base UI uses a `render` prop where Radix used `asChild`, and the generated components import the class helper as `import { cn } from "cn"`. That is a small package from the shadcn team which the CLI installs, not a typo. `src/lib/utils.ts` re-exports it, so `import { cn } from "@/lib/utils"` works as well (both measured on the real build). Use either; they are the same function.

**The stock components break most design systems in three places. Fix them once, here, before any page uses them**, or every page works round them separately:

- `dialog.tsx` and `alert-dialog.tsx` put a backdrop blur on the overlay, and `AlertDialogHeader` centres its text on phones.
- `button.tsx`, `input.tsx` and the select trigger default to 32px high, which is too small for a thumb. Raise the default to 40px or more.
- The select menu and the active tab carry a shadow.

Add components before pages are written, not during. If several agents build pages at once, none of them may run `shadcn add`: it installs packages, and two installs racing each other damage the lockfile. Add the likely set up front:

```bash
pnpm dlx shadcn@latest add button card input label select textarea table dialog alert-dialog badge checkbox separator tabs switch sheet
```

`sheet` is there for the phone version of the help guide in `references/help.md`; leave it out only if the build sheet has no built-in help and no page wants a slide-over panel.

## Configure

Project layout to follow for everything added later — all paths are relative to the current working directory, which *is* the project root:

```
src/
├── app/           # routes: page.tsx, layout.tsx, api/
├── components/    # shared React components (shadcn/ui lands in components/ui)
└── lib/           # db, auth, utilities
```

Create `.env` at the project root now (empty is fine) and confirm `.env*` is in `.gitignore` — later steps append to it.

### Start the history

The scaffold was made with `--disable-git`, so there is no repository yet. Make one now, at the project root, once the `.gitignore` check above has passed:

```bash
git init -b main
git add -A
git commit -m "Base project"
```

Then **commit after every Step 4 step**, once that step's Verify has passed — one commit per reference file, with a message that names it ("Database", "Sign-in"). This is local history only; nothing is pushed.

It is not housekeeping. `references/verify.md` reads the current commit with `git rev-parse HEAD` and checks for schema drift with `git status --porcelain drizzle`. Measured on the real build with no commits: `HEAD` is an "unknown revision", and the drift check prints `?? drizzle/` whether or not anything changed, so it proves nothing.

If `git commit` refuses because no name and email are set, ask the user for them. Do not invent an identity and do not change their global git settings without being asked.

## What runs next

`references/design.md`, straight away. The design step comes right after the base project and before the database, so `globals.css` and `layout.tsx` carry the app's own tokens and fonts before the first real page is written, and every page built afterwards inherits them. The three component fixes above are made against stock values here; the design step may adjust them again once `DESIGN.md` exists.

## Verify

- `package.json` sits in the current working directory — there is no nested project folder, and no `scaffold-tmp` left behind.
- `AGENTS.md` and `CLAUDE.md` from the installer sit at the project root.
- `pnpm dev` (`npm run dev` on an npm project) starts without errors and http://localhost:3000 renders. If the dev server prints a different port because 3000 was taken, use that port here and in every later check.
- A shadcn `Button` imported into `src/app/page.tsx` renders styled.
- `git log --oneline` shows the first commit on `main`, and `git ls-files` lists no `.env` file.
