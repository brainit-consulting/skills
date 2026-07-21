# Database (Drizzle ORM)

Last verified: 2026-07-21

**Purpose:** Store the app's data. Drizzle is the ORM in both branches; only the driver and connection differ. Follow exactly one branch: **SQLite** (local/prototype, zero setup) or **Postgres** (production-ready, runs in Docker locally).

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

**Postgres branch** — `docker-compose.yml` at project root:

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
  dbCredentials: { url: process.env.POSTGRES_URL! },
});
```

`src/lib/db/index.ts`:

```ts
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import * as schema from "./schema";

const pool = new Pool({ connectionString: process.env.POSTGRES_URL });
export const db = drizzle(pool, { schema });
```

Start it with `docker compose up -d` (tell the user Docker Desktop must be running).

**Both branches** — add scripts to `package.json`:

```json
"db:push": "drizzle-kit push",
"db:generate": "drizzle-kit generate",
"db:migrate": "drizzle-kit migrate",
"db:studio": "drizzle-kit studio"
```

While building the first version, `db:push` is the fast path (syncs schema straight to the database). Switch to `db:generate` + `db:migrate` once the app has real data to protect.

## Verify

- `pnpm db:push` completes without errors.
- Inserting and reading one row through `db` works (a quick script or the first page using a table is fine).
- `pnpm db:studio` opens and shows the tables (optional, good demo for the user).
