# Database (Drizzle ORM)

Last verified: 2026-07-21

**Purpose:** Store the app's data. Drizzle is the ORM in both branches; only the driver and connection differ. Follow exactly one branch: **SQLite** (local/prototype, zero setup, data lives in a file in the project) or **Postgres** (production-ready, runs in Docker locally and points at a hosted database in production via the same environment variable).

## Install

Both branches:

```bash
pnpm add drizzle-orm
pnpm add -D drizzle-kit
```

**SQLite branch:**

```bash
pnpm add better-sqlite3
pnpm add -D @types/better-sqlite3
```

**Postgres branch:**

```bash
pnpm add pg
pnpm add -D @types/pg
```

## Configure

Schema lives at `src/lib/db/schema.ts` — define tables from the user's interview nouns (plus auth tables later if sign-in is chosen).

**SQLite branch** — `drizzle.config.ts` at project root:

```ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./src/lib/db/schema.ts",
  out: "./drizzle",
  dialect: "sqlite",
  dbCredentials: { url: "./data/app.db" },
});
```

`src/lib/db/index.ts`:

```ts
import { drizzle } from "drizzle-orm/better-sqlite3";
import Database from "better-sqlite3";
import * as schema from "./schema";

const sqlite = new Database("./data/app.db");
export const db = drizzle(sqlite, { schema });
```

Create the folder and ignore the data file: `mkdir -p data` and add `data/` to `.gitignore`.

**Postgres branch** — two ways to run it while you build. Both end with a standard connection string in `POSTGRES_URL` and share identical application code, so this is a question about the user's machine, not about the app.

Ask at most one question here. Most users should hear "I'll set up a free hosted database" or "it'll run in Docker on your machine" and nothing more.

### Option A — Neon through the Vercel marketplace

Explain it as: *"Your database lives online on a free plan. While you build, you get your own private copy of it, so nothing you do here can touch the real thing — and it's the exact same database when you deploy."*

Worth preferring when the user has a Vercel account and any intention of deploying: nothing is installed, nothing has to be running, and the production and preview variables are already set when they eventually go live.

**Check before promising anything:**

```bash
vercel whoami
```

Signed in → carry on, and don't mention any of this; it's plumbing. Not signed in, or no CLI → **decide now, not after a failed command.** Either offer to walk them through `vercel login` (it opens a browser; they sign in, you can't do it for them) or move to Docker without making it sound like a downgrade. What you must not do is announce a free hosted database and then discover halfway through that there's no account.

**Ask which account before linking.** Many people belong to more than one Vercel team, and one of them is often a client's:

```bash
vercel teams list
```

One team → link and don't mention it. **More than one → ask, and never guess.** `vercel link --yes` refuses to choose anyway and errors demanding `--scope`, so guessing means picking wrong on purpose.

```bash
vercel link --yes --scope <team-slug>
vercel integration add neon --scope <team-slug>
```

`vercel link` writes `.env.local` and gitignores it on its own. `integration add` provisions the database, connects it to all three environments and runs `env pull` for you — no terms prompt, no browser step, and it works non-interactively under an agent. It also drops `.agents/skills/neon`, `.claude/` and `skills-lock.json` into the project. **Do not wave these through as harmless** — they are third-party *agent instructions* that arrived uninvited, they can carry scripts, and they will shape how this and every later agent behaves in the project. Tell the user they appeared, read the diff before anything loads them, and let the user decide whether they get committed.

Two of the variables it writes matter here:

| Variable | Connection | Used by |
|---|---|---|
| `POSTGRES_URL` | pooled | the app at runtime |
| `POSTGRES_URL_NON_POOLING` | direct | `drizzle-kit generate` / `migrate` / `studio` |

**Migrations must use the non-pooling URL.** Schema changes take locks that a connection pooler handles badly; the config below prefers it automatically.

**The development branch does not create itself.** `integration add` points production, preview *and* development at the same branch, `main` — so the first `db:migrate` from someone's laptop lands on production. Either enable **"Create a branch for your development environment"** in the Neon integration's settings, or branch `main` in the Neon console and repoint the development scope by hand:

```bash
vercel env rm POSTGRES_URL development --yes
printf '%s' "<pooled url for the dev branch>" | vercel env add POSTGRES_URL development
vercel env pull .env.local
```

Branching is copy-on-write, so the dev branch arrives with whatever schema and data `main` already had, in about a second. **If this isn't done, say so plainly at hand-off** — *"your local app is writing to the same database the live site will use"* — rather than leaving the user with a claim of isolation that isn't true.

**Never hand-edit `.env.local`** — `vercel env pull` overwrites the whole file. Everything added later (auth secret, API keys) goes in `.env`, which is never overwritten. Both are loaded, and `.env.local` wins where they overlap. Say this to the user, because it is the one way to lose keys later.

**Anything else hosted** — Supabase, Railway, RDS, an existing company database — is not special: put its connection string in `POSTGRES_URL` and carry on. If it offers a separate direct/non-pooled string, put that in `POSTGRES_URL_NON_POOLING`.

### Option B — Postgres in Docker

Run the database locally so the user installs nothing but Docker Desktop, and nothing is left running on their machine afterwards. `docker-compose.yml` at project root:

```yaml
services:
  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:
```

Append to `.env`:

```
POSTGRES_URL=postgresql://app:app@localhost:5432/app
```

`drizzle.config.ts`:

```ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./src/lib/db/schema.ts",
  out: "./drizzle",
  dialect: "postgresql",
  dbCredentials: {
    url: process.env.POSTGRES_URL_NON_POOLING ?? process.env.POSTGRES_URL!,
  },
});
```

The `??` is what lets one config serve both options: a hosted database with a pooler supplies both variables and migrations correctly use the direct one, while Docker supplies only `POSTGRES_URL` and it falls through.

**Use the plain `pg` driver, not a provider's serverless HTTP driver.** HTTP drivers don't support interactive transactions, which the auth and payments steps rely on, and Node.js runtimes on every major host handle a pooled `pg` connection fine. One driver, every environment, transactions intact.

`src/lib/db/index.ts`:

```ts
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import * as schema from "./schema";

if (!process.env.POSTGRES_URL) {
  throw new Error("The database isn't connected yet: POSTGRES_URL is not set.");
}

const pool = new Pool({ connectionString: process.env.POSTGRES_URL });
export const db = drizzle(pool, { schema });
```

**The guard matters more than it looks.** `new Pool({ connectionString: undefined })` does not fail — `pg` falls back to libpq's `PGHOST`, `PGUSER`, `PGPASSWORD` and `PGDATABASE`, which are set in more places than people expect: any machine with Postgres tooling configured, and several hosting platforms, including anything provisioned by a Postgres integration. So an app with no `POSTGRES_URL` doesn't refuse to start. It quietly connects to *a* database, which is not the one the migrations ran on, and throws `relation "..." does not exist` on the first query instead — sending the user to read their schema for a fault that is one layer away, in a missing environment variable.

The database is not an optional feature and shouldn't degrade like one. Fail immediately, and name the variable.

Start it with `pnpm db:up` (see the scripts below). Docker Desktop must be running — if it isn't, `docker compose` fails with a daemon connection error; tell the user to start Docker Desktop rather than debugging the app.

**If Docker isn't available and the user doesn't want to install it,** don't force it. Either fall back to the SQLite branch, or point `POSTGRES_URL` at a free hosted Postgres (Neon, Supabase) — the rest of this file is identical either way, because only the connection string changes.

**Going to production:** nothing in the code changes. Docker Compose is a local convenience only; a deployed app points `POSTGRES_URL` at a hosted Postgres set in the host's environment variables. Say this at hand-off so the user doesn't think they need to deploy a container.

**Both branches** — add scripts to `package.json`:

```json
"db:generate": "drizzle-kit generate",
"db:migrate": "drizzle-kit migrate",
"db:studio": "drizzle-kit studio"
```

**Postgres branch** — also add:

```json
"db:up": "docker compose up -d",
"db:down": "docker compose down"
```

**Never use `drizzle-kit push`.** Not for the first schema, not for a "quick" column, not while prototyping — and `db:push` is deliberately absent from the scripts above so it isn't within reach. `push` diffs the schema straight onto the database with no artefact left behind, which means the project has no migration history, teammates and production have no way to reproduce the schema, and the first destructive diff silently drops a column with real data in it. Migrations are the whole point of using an ORM with a migration tool.

**The schema workflow, every single time:**

```bash
pnpm db:generate   # writes a reviewable SQL file into ./drizzle
pnpm db:migrate    # applies pending migrations
```

Read what `db:generate` produced before applying it. Drizzle cannot always tell a rename from a drop-plus-add, and the generated SQL is where that shows up — a `DROP COLUMN` you didn't intend is obvious in the file and invisible if you skip it.

Commit the `drizzle/` folder. It is source code, not build output.

## Verify

- `pnpm db:generate` produces a migration file in `drizzle/`, and `pnpm db:migrate` applies it without errors.
- Inserting and reading one row through `db` works (a quick script or the first page using a table is fine).
- `pnpm db:studio` opens and shows the tables (optional, good demo for the user).
