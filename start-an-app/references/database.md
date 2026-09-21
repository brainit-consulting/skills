# Database (Drizzle ORM)

Last verified: 2026-09-21

**Purpose:** Store the app's data. Drizzle is the ORM in both branches; only the driver and connection differ. Follow exactly one branch: **SQLite** (local/prototype, zero setup, data lives in a file in the project) or **Postgres** (production-ready — the same database engine while you build and after you deploy, selected by one environment variable).

## Install

Both branches:

```bash
pnpm add drizzle-orm
pnpm add -D drizzle-kit dotenv
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

**Install what the registry's `latest` tag points at, and read the docs for that.** Drizzle's documentation site may describe a release candidate by default, with a different relations API and different install tags. The main skill's check-what's-current research settles which release is current before anything here is installed.

## Configure

Schema lives at `src/lib/db/schema.ts` — define tables from the user's interview nouns.

**If the app has accounts, this step sets up the connection, the scripts and the migration path, and stops there. Tables that point at a user are defined after `references/auth.md` has generated `auth-schema.ts`**, so the owner column and its foreign key are part of the table's first `CREATE TABLE`. "Who owns a row" below has the reason, which was measured. Tables with no column pointing at a person (a lookup list, a settings row) can be defined now. An app with no accounts defines all of its tables now.

### SQLite branch

`drizzle.config.ts` at project root:

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
export const db = drizzle({ client: sqlite, schema });
```

Create the folder and ignore the data file: `mkdir -p data` and add `data/` to `.gitignore`.

### Postgres branch

Two steps: choose **where the database runs while you build**, then wire it up. The wiring is the same whichever you choose — that is the whole point of this branch.

#### Step 1 — where the database runs while you build

Four options. **Recommend A unless something rules it out.** A, B and C all end with a standard Postgres connection string in `DATABASE_URL` and share identical application code; D is the only one that changes a source file, and that is why it is last.

Ask at most one question here, and only if A is ruled out. Most users should hear "I'll set up a free hosted database" and nothing more.

---

**A. Neon on the Vercel marketplace — the default**

Explain it as: *"Your database lives online on a free plan. While you build, you get your own private copy of it, so nothing you do here can touch the real thing — and it's the exact same database when you deploy."*

**Check before promising anything:**

```bash
vercel whoami
```

Signed in → carry on, and don't mention any of this; it's plumbing. Not signed in, or no CLI → **decide now, not after a failed command.** Either offer to walk them through `vercel login` (it opens a browser; they sign in, you can't do it for them), or move to option B or C without making it sound like a downgrade. What you must not do is announce a free hosted database and then discover halfway through that there's no account.

**Ask which account before linking.** Many people belong to more than one Vercel team, and one of them is often a client's:

```bash
vercel teams list
```

One team → link and don't mention it. **More than one → ask, and never guess.** `vercel link --yes` refuses to choose anyway and errors demanding `--scope`, so guessing means picking wrong on purpose. Putting someone's side project into a client's team is the sort of mistake that costs a phone call.

```bash
vercel link --yes --scope <team-slug>
vercel integration add neon --scope <team-slug>
```

`vercel link` writes `.env.local` and gitignores it on its own. `integration add` provisions the database, connects it to all three environments and runs `env pull` for you — no terms prompt, no browser step, and it works non-interactively under an agent. It also drops `.agents/skills/neon`, `.claude/` and `skills-lock.json` into the project. **Do not wave these through as harmless** — they are third-party *agent instructions* that arrived uninvited, they can carry scripts, and they will shape how this and every later agent behaves in the project. Tell the user they appeared, read the diff before anything loads them, and let the user decide whether they get committed.

If the CLI flow has changed, do it in the Vercel dashboard instead: **Storage → Neon → Install**, then link it to this project.

##### The development branch does not create itself

**`integration add` points production, preview *and* development at the same branch: `main`.** Nothing about it makes a dev branch, so the first `db:migrate` from someone's laptop lands on production. This is the one step that has to be done deliberately — and the Verify section below is what catches it when it's missed.

Either enable **"Create a branch for your development environment"** in the Neon integration's settings — which creates a persistent `vercel-dev` branch and rewrites the Development-scope variables — or do the same explicitly: branch `main` in the Neon console, then repoint only the development scope:

```bash
vercel env rm DATABASE_URL development --yes
printf '%s' "<pooled url for the dev branch>" | vercel env add DATABASE_URL development
vercel env rm DATABASE_URL_UNPOOLED development --yes
printf '%s' "<direct url for the dev branch>" | vercel env add DATABASE_URL_UNPOOLED development
vercel env pull .env.local
```

Branching is copy-on-write, so the dev branch arrives with whatever schema and data `main` already had, in about a second.

**If neither is done, say so plainly at hand-off** — *"your local app is writing to the same database the live site will use"* — rather than leaving the user with a claim of isolation that isn't true.

`vercel env pull .env.local` refreshes the file at any time. Two values matter:

| Variable | Connection | Used by |
|---|---|---|
| `DATABASE_URL` | pooled | the app at runtime |
| `DATABASE_URL_UNPOOLED` | direct | `drizzle-kit generate` / `migrate` / `studio` |

**Migrations must use the unpooled URL.** Schema changes take locks that a connection pooler handles badly; the config in Step 2 already prefers it.

**Never hand-edit `.env.local`** — `vercel env pull` overwrites the whole file. Everything this skill adds later (auth secret, API keys) goes in `.env`, which is never overwritten. Both are loaded, and `.env.local` wins where they overlap. Say this to the user, because it is the one way to lose keys later.

Which branch each environment gets, for free, once this is set up:

| Where | Neon branch |
|---|---|
| Production | `main` |
| Preview deployment | `preview/<git-branch>`, created per deployment |
| Local development | `vercel-dev` |

**Going to production: check, don't assume.** The integration sets *its own* variables — `POSTGRES_URL`, `PGHOST`, `PGUSER` and the rest — in all three environments, and that is the reason A is the default. But the app reads **`DATABASE_URL`**, and that one can end up scoped to development only: it is what the development-branch step above creates, and a project can reach a first deploy with no production `DATABASE_URL` at all. Without the guard in Step 2's client, the app would then start against the integration's `PG*` fallbacks and fail on the first query.

One command settles it, and it costs nothing to run:

```bash
vercel env ls production
```

`references/deploy.md` covers the rest of what a first deploy needs. The short version: "the integration wired it up" is true of Neon and not necessarily true of your app.

---

**B. Postgres in Docker**

For users who already run Docker Desktop and want the database on their own machine. They install nothing else, and nothing is left running once the container is stopped. `docker-compose.yml` at project root:

```yaml
services:
  db:
    image: postgres:alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: app
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql
volumes:
  pgdata:
```

**The volume is mounted at `/var/lib/postgresql`, not `/var/lib/postgresql/data`.** Current Postgres images keep their data in a versioned folder under that path; mounting the old `data` path means the container refuses to start or the data does not survive a restart. The check-what's-current research confirms the path for the image the tag currently points at.

**Check what is already running before choosing the port.** `docker ps -a` is read-only and shows every container's ports. If 5432 is taken, or the user has other projects' databases on the machine, pick a free host port, use it in both `docker-compose.yml` and `DATABASE_URL`, and give the compose file a top-level `name:` so the container and volume are named after this app. **Only ever run `docker compose` commands from the project folder.** Never `docker system prune`, `docker volume prune` or anything else that acts on every container: the user's other work lives in the same Docker.

The tag carries no version on purpose, so a fresh project gets the current stable Postgres. Say one thing about it to the user if they ever ask why: a Postgres data directory belongs to the major version that created it, so once the app has real data in it, an image that moves to a new major will refuse to start against the old volume — the fix is a dump and restore, not a flag. That is the moment to pin the major, not before.

Append to `.env`:

```
DATABASE_URL=postgresql://app:app@localhost:5432/app
```

Start it with `pnpm db:up` (see the scripts below). Docker Desktop must be running — if it isn't, `docker compose` fails with a daemon connection error; tell the user to start Docker Desktop rather than debugging the app.

**Going to production:** nothing in the code changes. The compose file is a local convenience only — a deployed app points `DATABASE_URL` at a hosted Postgres set in the host's environment variables. Say this at hand-off so the user doesn't think they need to deploy a container.

---

**C. `embedded-postgres` — a real Postgres server, no Docker, no account**

Real Postgres binaries downloaded as an npm package and run as a local process. Use this when the user won't install Docker *and* wants to work offline with a genuine Postgres server.

```bash
CI=true pnpm add -D embedded-postgres
pnpm approve-builds embedded-postgres
```

Approving the build is required — the package installs binaries in a postinstall script, and pnpm blocks those by default, so without it the package installs but cannot start. **Name the package explicitly.** Bare `pnpm approve-builds` opens an interactive checklist, which under an agent has no TTY to answer it — the same no-TTY problem `CI=true` solves everywhere else in this file.

`scripts/db-local.ts`:

```ts
import EmbeddedPostgres from "embedded-postgres";
import { existsSync } from "node:fs";

const databaseDir = "./data/pg";
const pg = new EmbeddedPostgres({
  databaseDir,
  user: "app",
  password: "app",
  port: 5432,
  persistent: true,
});

const firstRun = !existsSync(databaseDir);
if (firstRun) await pg.initialise();
await pg.start();
if (firstRun) await pg.createDatabase("app");
console.log("Postgres running on localhost:5432 — leave this window open.");
```

Append to `.env`:

```
DATABASE_URL=postgresql://app:app@localhost:5432/app
```

If something on the machine already holds 5432 — another project's database, in Docker or not — pick a free port and use it in both the script and `DATABASE_URL`.

Add `data/` to `.gitignore`. It runs in the foreground, so it needs its own terminal alongside `pnpm dev` — tell the user that plainly, because a closed window looks like a broken app.

---

**D. PGlite — offline, nothing to install, one code change**

Postgres compiled to WebAssembly and run inside the app's own process, persisting to a folder. Nothing to install, nothing to sign up for, no separate terminal.

**Offer this last, and say the tradeoff out loud:** it is the only option that does not survive deployment unchanged. `src/lib/db/index.ts` has to be swapped to the `node-postgres` version below before the app can go live. Every other option deploys as-is.

```bash
pnpm add @electric-sql/pglite
```

`drizzle.config.ts` gets a driver line the other options don't need:

```ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./src/lib/db/schema.ts",
  out: "./drizzle",
  dialect: "postgresql",
  driver: "pglite",
  dbCredentials: { url: "./data/pgdata" },
});
```

`src/lib/db/index.ts`:

```ts
import { drizzle } from "drizzle-orm/pglite";
import * as schema from "./schema";

export const db = drizzle({ connection: { dataDir: "./data/pgdata" }, schema });
```

Add `data/` to `.gitignore`.

Two limits to know before choosing it:

- **One process at a time owns the data directory.** Stop `pnpm dev` before running `db:migrate`, `db:studio` or `pnpm build` (which migrates first — see the scripts below), or they will fail to open the database.
- **Confirm `drizzle-kit migrate` works before building on it.** Support has been unreliable in past releases. Run `pnpm db:generate && pnpm db:migrate` on the very first table; if it errors, move the user to A, B or C rather than applying SQL by hand — this skill never leaves a project without a working migration path.

---

**Anything else hosted** — Supabase, Railway, RDS, an existing company database. Nothing above is special: put its connection string in `DATABASE_URL` and follow Step 2 unchanged. If it offers a separate direct/non-pooled string, put that in `DATABASE_URL_UNPOOLED`.

#### Step 2 — wire it up

Identical for A, B and C. For D, use the config and client shown in D instead, then rejoin here at the scripts.

`drizzle.config.ts` at project root:

```ts
import { config } from "dotenv";
import { defineConfig } from "drizzle-kit";

config({ path: ".env.local", override: true });

export default defineConfig({
  schema: "./src/lib/db/schema.ts",
  out: "./drizzle",
  dialect: "postgresql",
  // || not ??: a variable that exists but is empty must not win.
  dbCredentials: { url: process.env.DATABASE_URL_UNPOOLED || process.env.DATABASE_URL || "" },
});
```

**`drizzle-kit` loads `.env` by itself, before it reads this file. It does not read `.env.local`.** That is the only reason the dotenv line is there: option A keeps its connection strings in `.env.local`, and without the line `db:migrate` cannot see them. `override: true` makes `.env.local` win where both files set the same name, which matches what Next does at runtime — without it, the `.env` value drizzle-kit already loaded stays in place, and migrations can run against a different database from the one the app uses. Where there is no `.env.local` (options B and C, or a build on the host) the line does nothing.

The `??` is what lets one config serve every option: hosted Postgres with a pooler supplies both variables and migrations correctly use the direct one; a local database supplies only `DATABASE_URL` and it falls through.

`src/lib/db/index.ts`:

```ts
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import * as schema from "./schema";

if (!process.env.DATABASE_URL) {
  throw new Error(
    "The database isn't connected yet: DATABASE_URL is not set. " +
      "With the hosted database, run `vercel env pull .env.local`. Otherwise add it to .env."
  );
}

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
export const db = drizzle({ client: pool, schema });
```

**Pass one object: `drizzle({ client: pool, schema })`.** The object form is the one that survives Drizzle's next major release, which drops the positional `drizzle(pool, …)` form.

**The guard is not defensive decoration, and it is not optional.** `new Pool({ connectionString: undefined })` does not fail — `pg` falls back to libpq's `PGHOST`, `PGUSER`, `PGPASSWORD` and `PGDATABASE`, every one of which the Neon integration sets. So an app missing `DATABASE_URL` doesn't refuse to start; it quietly connects to *a* database, which is not the one the migrations ran on, and throws `relation "..." does not exist` on the first query instead. That sends the user reading their schema when the actual fault is a missing environment variable one layer away. The main skill file requires this behaviour under "the database is not optional, so it does not degrade" — this is where it gets built.

**Use the plain `pg` driver, not a provider's serverless HTTP driver.** Vercel's default runtime is full Node.js and reuses warm instances, so a pooled `pg` connection is fine there; HTTP drivers, meanwhile, don't support interactive transactions, which the auth and payments steps rely on. One driver, every environment, transactions intact.

### Both branches — scripts

Add to `package.json`:

```json
"db:generate": "drizzle-kit generate",
"db:migrate": "drizzle-kit migrate",
"db:studio": "drizzle-kit studio"
```

Also change the existing `build` script so migrations run on every deploy:

```json
"build": "pnpm db:migrate && next build"
```

(If the project was scaffolded with npm rather than pnpm, use `npm run db:migrate && next build`.)

Without this, a deploy ships new code against an old database schema: the migration files are committed but nothing ever applies them on the host, and the first request touching a new column fails at runtime. Hooking `db:migrate` into `build` means the host applies pending migrations as part of the deploy, in the same step that produces the build. It is a no-op locally when there is nothing pending, so `pnpm build` stays safe to run any time.

This needs the deployed environment to have the database connection variable the migration uses (`DATABASE_URL`, plus `DATABASE_URL_UNPOOLED` where the host offers a direct string; or the SQLite file path) set at *build* time, not just at runtime — say so at hand-off, because a build that can't reach the database fails the whole deploy. On option A a preview deployment migrates its own preview branch and production migrates `main`, which is what you want.

**Docker (option B)** — also add:

```json
"db:up": "docker compose up -d",
"db:down": "docker compose down"
```

**`embedded-postgres` (option C)** — also add:

```json
"db:local": "tsx scripts/db-local.ts"
```

(`pnpm add -D tsx` if it isn't already present.)

**Never use `drizzle-kit push`.** Not for the first schema, not for a "quick" column, not while prototyping — and `db:push` is deliberately absent from the scripts above so it isn't within reach. `push` diffs the schema straight onto the database with no artefact left behind, which means the project has no migration history, teammates and production have no way to reproduce the schema, and the first destructive diff silently drops a column with real data in it. Migrations are the whole point of using an ORM with a migration tool.

**The schema workflow, every single time:**

```bash
pnpm db:generate   # writes a reviewable SQL file into ./drizzle
pnpm db:migrate    # applies pending migrations
```

Read what `db:generate` produced before applying it. Drizzle cannot always tell a rename from a drop-plus-add, and the generated SQL is where that shows up — a `DROP COLUMN` you didn't intend is obvious in the file and invisible if you skip it. Look for the opposite fault too: a `REFERENCES` with no `ON DELETE` after it, where the schema declared one ("Who owns a row" below has the measured case). Because `build` migrates first, generate and read the SQL before building too: a build reached with an ungenerated schema edit outstanding is the wrong moment to find out.

Commit the `drizzle/` folder. It is source code, not build output.

## Rules the database has to enforce itself

Some rules cannot be left to application code, because two requests can pass the same check at the same instant. The common one: **two bookings, reservations or shifts for the same person or room must never overlap.** Postgres enforces that with an exclusion constraint, and Drizzle cannot express one, so it goes in a hand-written migration:

```bash
pnpm exec drizzle-kit generate --custom --name=no_overlap
```

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;
--> statement-breakpoint
ALTER TABLE "bookings" ADD CONSTRAINT "bookings_no_overlap"
  EXCLUDE USING gist (
    "stylist_id" WITH =,
    tstzrange("starts_at", "ends_at", '[)') WITH &&
  ) WHERE ("status" = 'booked');
```

`'[)'` leaves the end instant out, so back-to-back slots are allowed. The `WHERE` is what frees a slot when a row is cancelled. In the app, catch SQLSTATE `23P01` and turn it into a sentence ("someone just took that time"); with Drizzle the code sits on the error's `cause`. Later `db:generate` runs leave the constraint alone, because drizzle-kit does not track it. Prove it once with a script that inserts an overlapping row directly, going round the app.

This is a Postgres feature. SQLite has no exclusion constraints, so an app whose core promise is "no double bookings" belongs on the Postgres branch. On option D, confirm the `btree_gist` extension loads before relying on it; if it doesn't, move to A, B or C.

**Anything that is a moment in time gets `withTimezone: true`.** A plain `timestamp` is stored without a zone and comes back interpreted as UTC, so an appointment booked for 9:30am renders as 5:30am — wrong by exactly the user's offset, on every row, in a way they notice on day one and never trust again:

```ts
startsAt: timestamp("starts_at", { withTimezone: true }).notNull(),
```

The `tstzrange` in the constraint above needs these columns to be `timestamptz` as well.

**One app time zone, decided in one place.** If the app has a "today" or opening hours, decide the app's time zone in one config file and convert through it everywhere. A server in production runs on UTC, and "today" worked out from the server's clock is wrong for part of every day.

## Schema conventions

**Better Auth's tables are not yours to design.** `src/lib/db/auth-schema.ts` is written by the Better Auth CLI (see `references/auth.md`) and is left exactly as generated — same column names, same types, same `text` ids. Editing it breaks the adapter, and the next CLI run overwrites the edit anyway.

**Every table you define gets a randomly generated UUID primary key.** Not an auto-incrementing integer. Sequential ids leak information the app never meant to publish — `/invoices/1042` tells any customer roughly how many invoices exist, and lets them walk the whole table by counting down — and they collide the moment data is merged or seeded from more than one place. A UUID is unguessable, and the row can be given its id before it ever reaches the database.

The id fills itself in, so application code never passes one on insert.

**Postgres branch:**

```ts
import { pgTable, uuid, text, timestamp } from "drizzle-orm/pg-core";
import { user } from "./auth-schema";

export const hikes = pgTable("hikes", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: text("user_id")
    .notNull()
    .references(() => user.id, { onDelete: "cascade" }),
  trail: text("trail").notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});
```

`defaultRandom()` is a real database-level default (`gen_random_uuid()`), so rows inserted by anything other than the app get an id too.

**SQLite branch** — SQLite has no `uuid` column type, so the id is `text` filled in by Drizzle at insert time:

```ts
import { sqliteTable, text, integer } from "drizzle-orm/sqlite-core";
import { user } from "./auth-schema";

export const hikes = sqliteTable("hikes", {
  id: text("id")
    .primaryKey()
    .$defaultFn(() => crypto.randomUUID()),
  userId: text("user_id")
    .notNull()
    .references(() => user.id, { onDelete: "cascade" }),
  trail: text("trail").notNull(),
  createdAt: integer("created_at", { mode: "timestamp" })
    .notNull()
    .$defaultFn(() => new Date()),
});
```

`$defaultFn` runs in the app, not the database — fine here, because every insert goes through Drizzle. `crypto.randomUUID()` is a global in the Node and edge runtimes; nothing to import.

**The one trap: a column that points at a user stays `text`.** Better Auth's `user.id` is `text` in both branches, so `userId` above is `text` even in the Postgres example where the table's own id is `uuid`. Declaring it `uuid` looks consistent and fails — the foreign key won't create, and `db:migrate` stops with a type mismatch. The same applies to any other column referencing a generated auth table.

Tables that reference *your* tables use the matching type: `uuid` on Postgres, `text` on SQLite.

### Who owns a row: three cases

The `hikes` examples above show the first case. It is not the only one, and the interview settles which applies before any table is written. `references/pages.md` applies the same three cases to the queries.

| Case | Column | Every query is scoped by | `onDelete` on the column pointing at a person |
|---|---|---|---|
| **Private to each user** (my hikes, my recipes) | `userId`, `notNull` | the session user's id | `"cascade"`: the rows go when the account goes |
| **Shared by a team** (a company's jobs, customers, stock) | no owner column. Columns such as `assignedTo` or `createdBy` say who did what | **role**: a plumber sees the jobs assigned to them, the office sees all of them | `"set null"` or `"restrict"`, **never `"cascade"`** |
| **No accounts** | no `userId` column at all | nothing | not applicable |

**Shared data and `cascade` do not mix.** With `cascade` on `assignedTo`, deleting the account of someone who has left the company deletes every job they ever worked on, and the team's history goes with them. `"set null"` keeps the row and leaves the column empty, so the column must be nullable. `"restrict"` refuses the deletion until the rows are reassigned; choose it where a row without a person makes no sense, and have the app say why the deletion was refused. `references/settings.md` builds account deletion and depends on this choice being right.

The shared-team case has not been built with this skill; the rule comes from how foreign keys behave, not from a finished app. Prove it with the role probe in `references/verify.md`.

**Define user-owned tables after auth has generated `auth-schema.ts`, not before.** Measured on the SQLite branch: a table created without its owner column, then given one in a later migration, got `ALTER TABLE ... ADD ... REFERENCES user(id)` with **no `ON DELETE` clause**. The migration reported success. `PRAGMA foreign_key_list` on the table showed `NO ACTION`. Deleting an account later either fails on the foreign key or leaves rows with no owner, depending on whether foreign keys are enforced on that connection. When the column is in the first `CREATE TABLE`, the clause is written correctly. The same order is used on Postgres; the fault was not measured there.

If a table already exists without the column, do not trust the `ALTER`. On SQLite the dependable fix is a new table with the right foreign key, the rows copied across, the old table dropped and the new one renamed, all in one hand-written migration (`drizzle-kit generate --custom`). Then check `foreign_key_list` again.

**Every time a migration adds a foreign key, search the generated SQL for `REFERENCES` and check each one carries the `ON DELETE` you declared.** A missing clause is silent: nothing errors until someone deletes an account.

**Run one-off scripts with `tsx`.** `pnpm add -D tsx`, then `pnpm exec tsx scripts/check.ts`. Measured: plain `node` fails on this project's TypeScript imports, which have no file extension. On the Postgres branch the script also needs the connection string, and nothing loads `.env` or `.env.local` for it the way Next does. Load them before the database module is imported: imports run before any other line in the file, so a `config()` call written below `import { db }` runs too late and the guard in `src/lib/db/index.ts` throws. Passing the files on the command line avoids the ordering problem (`pnpm exec tsx --env-file=.env --env-file=.env.local scripts/check.ts`, and leave out a file that does not exist). That command line was not run during the measured build, which was on SQLite and needs no variables; confirm it on the first script.

## Verify

- `pnpm db:generate` produces a migration file in `drizzle/`, and `pnpm db:migrate` applies it without errors. No schema was ever pushed. Where every table waits for auth, there is nothing to generate yet, and `db:migrate` may complain that it has no migrations folder to read. That is expected; this item and the `pnpm build` item below are ticked after `references/auth.md` has generated the first migration.
- Every table you defined has a UUID primary key that fills itself in — inserting a row without passing an `id` works — and, once auth has run, `auth-schema.ts` is untouched from what the Better Auth CLI generated.
- `package.json` has `"build": "pnpm db:migrate && next build"`, and `pnpm build` completes — running migrations first, then the Next.js build.
- Inserting and reading one row through `db` works, from a script run with `tsx`. Use a table that has no owner column. **If every table in the app belongs to a user, none of them exists yet at this step**: prove the connection with `select 1` through `db` instead (`db.run` on SQLite, `db.execute` on Postgres), and either defer the row test until the user-owned tables exist, or use a throwaway table that the same script creates and drops with plain SQL. Keep a throwaway table out of `schema.ts` and out of the migrations, where it would stay in the history. Do not create an account to own a test row either: the first account belongs to the real person, in Step 6.
- After auth has run and the user-owned tables are migrated: every `REFERENCES` in the generated SQL carries the `ON DELETE` the schema declared. On SQLite, `PRAGMA foreign_key_list(<table>)` shows `CASCADE` for private data and `SET NULL` or `RESTRICT` for shared data, never `NO ACTION`.
- Deferred to Step 6, after the user has signed up: a row created through the app carries the right owner (private data), or is visible to the right roles (shared data).
- Where the app has a no-overlap rule: a script that inserts an overlapping row directly is refused with `23P01`, and the app shows a sentence for it rather than a stack trace.
- `pnpm db:studio` opens and shows the tables (optional, good demo for the user).
- **Option A only:** `.env.local` contains both `DATABASE_URL` and `DATABASE_URL_UNPOOLED`, and the Neon dashboard shows the migration landed on the `vercel-dev` branch — not on `main`. If it landed on `main`, the development branch was never enabled; fix that before any real data exists.
- **Option A only:** the host in `DATABASE_URL` is the same host as in `POSTGRES_URL`, in the same `.env.local`. Two different hosts means two different Neon projects — the app is reading one and the integration provisioned the other, so the migrations, the data and the deployment are not all in the same place. Compare them by eye; the endpoint id (`ep-...`) is the part that has to match.
- **Option B only:** `docker ps` shows this app's container under the compose project's own name, on the port `DATABASE_URL` uses, and no other project's container was stopped, removed or pruned along the way.
