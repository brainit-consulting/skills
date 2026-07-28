# skills

Agent Skills — a fork of [leonvanzyl/skills](https://github.com/leonvanzyl/skills), maintained by Emile du Toit, BrainIT Consulting.

Upstream is kept as a git remote so his updates can be pulled in:

```bash
git fetch upstream && git merge upstream/main
```

## Skills

### `start-an-app`

Interviews the user about what they actually want to build, then scaffolds a working full-stack Next.js app around it — database, sign-in, uploads, payments, AI, landing page and dashboard. The interview is the valuable part; the scaffold is meant to look like *their* app from the first commit, not a template.

### `security-scanner`

OWASP Top 10:2025 audit of any codebase, in any language — eleven reference files of CWEs and detection patterns, severity scoring, and a dated markdown report. Imported from Leon's [agentic-coding-starter-kit](https://github.com/leonvanzyl/agentic-coding-starter-kit) with his permission.

It's here because **`start-an-app` checks that what it built works, never that it is safe** — and it's routinely used to build apps holding a small business's customer records. Run it once the app is real: most of what A01 and A07 look for doesn't exist until sign-in works and there's data in the database.

## What this fork changes

### The dev database no longer leads with Docker

Upstream's Postgres branch runs the local development database in Docker. This fork keeps that option but no longer leads with it.

**Default: Neon through the Vercel marketplace.** Free hosted Postgres, with the integration's development branch enabled so localhost gets its own copy-on-write clone (`vercel-dev`) and never touches production data. Preview deployments get a branch each, production uses `main`, and deploying needs no extra setup because the integration already wrote the production variables.

**All four ways to run the dev database are documented**, so nothing upstream offered was removed:

| | Needs | Deploys unchanged |
|---|---|---|
| **A. Neon via Vercel** *(default)* | Vercel account, internet | yes — already wired |
| **B. Docker** | Docker Desktop | yes |
| **C. `embedded-postgres`** | nothing (real Postgres binaries via npm) | yes |
| **D. PGlite** | nothing (Postgres in WASM, offline) | **no** — one file must be swapped |

Supporting changes that follow from it:

- `POSTGRES_URL` → `DATABASE_URL`, matching what the Neon integration injects. Migrations use `DATABASE_URL_UNPOOLED` where a pooler exists, via `DATABASE_URL_UNPOOLED ?? DATABASE_URL` so one config serves every option.
- `.env.local` (pulled by `vercel env pull`, overwritten wholesale) is kept separate from `.env` (hand-written keys), with the trap called out explicitly.
- The plain `pg` driver is specified over provider serverless/HTTP drivers — HTTP mode has no interactive transactions, which the auth and payments steps depend on, and Vercel's default runtime is full Node.js anyway.

### An optional design system, extracted from a real site

New interview question, asked once and easy to decline: *"Do you already have a website? Paste the address and I'll match your colours and fonts."*

- **A URL** → the agent loads the home page and one interior page in a real browser, tallies computed styles (colour roles, font families and weights, radius, shadow, container width), reads density and layout temperament off screenshots, and writes `DESIGN.md` at the project root.
- **A vibe instead** ("like Linear", "expensive and quiet") → same file, no extraction.
- **Nothing** → a short `DESIGN.md` derived from what the app is. No pressure applied.

`DESIGN.md` is then *enforced*, which is the part that usually gets skipped: its token table is applied to `globals.css` (light and dark) and its font to `layout.tsx` **before any page exists**, `references/pages.md` defers to it, and both Verify checklists fail if a component sets a colour outside the tokens.

Guardrails, because "copy that site" is a request with sharp edges: feel is extracted, assets never are — no logo, images, copy or stylesheet. Commercial webfonts are substituted with the closest open equivalent and both are recorded. Provenance is a required section. And where the brand and the anti-slop rules disagree, **brand facts win on identity, the rules win on craft** — the user's own font stays even if a rule would ban it; their centred-hero-with-three-cards layout does not.

### The interview reworded for non-technical users

The skill's stated audience is "a smart friend who doesn't code", and most of it already reads that way — but a few questions were written from the builder's side of the table. This fork points them at small business owners **without lengthening the interview**. It is one question shorter than upstream.

- **The duplicate is gone.** "Whose data is it?" (Step 1b) and "Who's going to use it?" (Step 1c) were the same question to a non-technical ear, asked in two sections for two different internal reasons. They're merged into one, asked early, and Step 1c now confirms rather than re-asks — being asked twice reads as *you weren't listening*.
- **"When someone lands on the app signed out…"** → *"When someone who's never used this before arrives, what should they see first?"* Signed-out is a state; people think in people.
- **"Who's going to use it?"** → *"Your customers, your staff, or just you?"* Those are three different apps, and upstream's wording flattened the first two into "other people".
- Smaller passes: *their own login* over "sign in"; *will customers pay you through this* over "will people pay for anything"; AI framed as what it does rather than as "AI features".
- **New ground rule: use their word for the thing.** Site, system, "the booking thing" — mirror it. Saying "app" to someone who hears *phone app from the App Store* quietly signals this wasn't built for them.

Two questions added, one swapped out:

- **"What are you doing about this today — a spreadsheet, a notebook, WhatsApp, nothing?"** replaces the weaker "is there anything like this you already use?". A working business already has the process, just badly: their spreadsheet columns *are* the schema. Its follow-up — *do you want what's already in there brought across?* — decides whether day one produces a tool or a demo.
- **"Anything sensitive in here?"** (health details, children's information, card numbers) as a gap-check raised only when the subject matter suggests it. Free on a hiking journal; decisive for a vet clinic or a school.

Step 1b's soft "two or three usually matter" is now a hard cap of three, because nine fair questions in a row is still an interrogation.

**Scaffolding into an existing project is now a stop, not a merge.** Upstream listed "an existing `package.json`" among the cases to work around by scaffolding to a temp dir and moving the result up — which silently splices a fresh Next.js app into someone's repo if the skill is fired in the wrong folder. Stray files still merge; a real project halts. The build sheet also states the absolute target path and what's in it, so the user confirms a location instead of inheriting whichever directory the session opened in.

### An in-app help guide, written from the interview

Apps built for customers or staff get a `?` in the navbar that opens a guide explaining the app in the owner's own words.

The reason it belongs in this skill rather than in a component library: Step 1a already produced a description of the app in the owner's words, its nouns, its verbs, and a walkthrough of a first visit — **a user guide with the labels changed**, thrown away today the moment the build sheet is agreed. The first-visit walkthrough becomes *Getting started*, each verb becomes a chapter, the ownership answer becomes *Who sees what*.

It matters most for small businesses, where the person who commissioned the app isn't the person using it in six months and "ask the owner how it works" doesn't survive staff turnover.

**No new question.** Question 1 already says customers / staff / just you — the first two get a guide, the third doesn't. It appears on the build sheet as something to decline.

`references/help.md` pins down where this gets built badly: build on shadcn's `Dialog` so the focus trap, Esc and `aria-modal` come free; 80% of the viewport is the *starting* size, not a fixed one; persist position and size but **clamp on every open**, or a box saved on a 32-inch monitor opens off-screen on a laptop the next morning; full-screen sheet with no dragging below `md`, because touch-drag fights scrolling and scrolling should win; and nothing may require dragging to reach.

### Rules for when the agent may use real tooling

Upstream's scaffold path touched nothing but `npx` and `pnpm`. Putting Neon behind the Vercel CLI crosses that line, so this fork names the rule instead of leaving it implicit. The test is **does the app run without it?**

- **Needed to run** — the database. The agent uses the `vercel` CLI directly and doesn't make it a conversation. Doing it for someone is *less* technical than explaining it to them.
- **Not needed to run** — GitHub and deployment. Offered at hand-off, one line each, never done mid-build. The skill ends with something running on their machine, not with their business live on the internet.
- **Can't be automated** — `vercel login`, Google Cloud, Stripe/Polar signup. Guide, don't drive, and say which is happening. Upstream already did this correctly for Google sign-in; it's now a general rule rather than a one-off.
- **Never on the agent's own initiative:** creating accounts or logins, anything that costs money, anything that makes something public, attaching a domain.
- **Tool preference:** `vercel` CLI first; a Vercel MCP server if the agent happens to have one, but never as a dependency; never raw API calls with a hand-pasted token, which puts a credential in the conversation to do what the CLI already holds a session for.
- **Narrate outcomes, not commands.** *"I've set up your database — it's free and it's yours"*, not *"running `vercel integration add neon`"*.

Two concrete consequences: `references/database.md` runs `vercel whoami` **before** promising a free hosted database, so the fallback to Docker/local happens before expectations are set rather than after a failed command; and the GitHub offer at hand-off is **private by default with the visibility said out loud**, because a business owner accidentally publishing their source is a real harm rather than an untidiness.

## Install

**Normal use** — from the published copy, which works in Claude Code, Codex, Cursor, Gemini CLI, Copilot and others:

```bash
npx skills add brainit-consulting/DreamForgeSoftwareAgentSkills --skill start-an-app
```

```bash
npx skills add brainit-consulting/DreamForgeSoftwareAgentSkills --skill security-scanner
```

**Working on the skill itself** — symlink this repo so edits take effect immediately (PowerShell, needs Developer Mode or an elevated shell):

```powershell
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.claude\skills\start-an-app" -Target "H:\skills\start-an-app"
```

A plain copy works too, but then edits here don't reach the installed copy.

## Using `start-an-app`

**Open the folder you want the app to live in, and make sure it's empty.** The app is created in the current working directory — the folder you open is the folder it lands in. There's no "where should this go?" question, because an agent can't reliably write outside where it started. If the folder already holds a project, the skill stops rather than scaffolding over it.

Then just say what you want:

```
/start-an-app
```

or simply *"I want to build a booking system for my salon"* — the skill triggers on the intent.

### What it asks

An open conversation about the idea first — including *"what are you doing about this today, a spreadsheet, a notebook, WhatsApp?"*, which usually hands over the data model. Then at most three gap-checks, then seven technical questions, each with a recommended default so **"whatever you recommend" is a complete answer**:

1. Who's it for — customers, staff, or just you?
2. Do people need their own login?
3. Will people upload anything?
4. Will customers pay you through this?
5. Should it do anything with AI?
6. What should a first-time visitor see?
7. Do you already have a website to match the look of?

It then reads the whole plan back — data model, what you can do, where the folder is, and an explicit *not in version one* list — and waits for a yes before running a single command.

### What you need first

**Nothing, to start.** Two things unlock more if you have them:

- **A Vercel account** (free) — gets you hosted Postgres with your own copy-on-write branch for local work. Without one it falls back to Docker, a local Postgres server, or offline PGlite; you're never blocked.
- **An existing website** — paste the address and it matches your colours, typefaces and shape.

Anything needing a signup (Polar or Stripe for payments, Google sign-in, OpenRouter for AI) is only touched if you ask for that feature, and the skill walks you through it rather than doing it for you.

### What you get

A running app on your machine, its schema in real migrations, a `DESIGN.md` it actually obeys, and — for apps used by customers or staff — a `?` in the corner opening a guide written from your own answers. Putting it on GitHub and deploying are **offered at the end, never done for you**.

## Using `security-scanner`

Run it in the repo you want audited, once the app is real:

```
/security-scanner
```

It sweeps all ten OWASP 2025 categories and writes a dated report to `audit/` with file, line, evidence and a fix for every finding. Worth knowing before you read it: **a finding is not a breach.** A fresh scaffold produces mostly Low and Info — missing headers, console-only logging. Look at the Critical and High counts first.

## Credits

Licensed [MIT](LICENSE). Contributions upstream are made as Emile du Toit, BrainIT Consulting ([github.com/brainit-consulting](https://github.com/brainit-consulting)).

- [leonvanzyl/skills](https://github.com/leonvanzyl/skills) — the upstream skill this forks.
- [leonvanzyl/agentic-coding-starter-kit](https://github.com/leonvanzyl/agentic-coding-starter-kit) — `security-scanner` is Leon's, imported with his permission and unchanged apart from an attribution note and a section on pairing it with `start-an-app`.
- [Leonxlnx/taste-skill](https://github.com/Leonxlnx/taste-skill) (MIT, © 2026 Leonxlnx) — `references/design.md` adapts its `stitch-skill` `DESIGN.md` export format, its three dials, and its anti-pattern list. `design.md` works standalone; taste-skill is an optional install for deeper front-door work, and the two divide as **`DESIGN.md` owns the facts, taste-skill owns the craft**.
