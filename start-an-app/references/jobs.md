# Background processing (Inngest)

Last verified: 2026-09-21

**Purpose:** Run work that shouldn't happen inside a web request — anything slow, anything that has to retry itself, anything on a schedule, anything that must survive the server restarting mid-way. And make all of it visible from inside the app.

> **Hard rule: the app keeps its own record of every job.** A row goes into the `jobs` table *before* the event is sent, and the function updates it as it goes. (Scheduled work has nobody to send the event, so there the function opens its own row as its first step — see *Work on a schedule*.) Inngest is the engine and the live detail view; the `jobs` table is the record. This is not belt-and-braces — Inngest's free plan keeps 24 hours of history, and its API is account-wide rather than scoped to one of the app's users, so it can be neither the archive nor the thing a user is allowed to query. `references/ops.md` renders this table.
>
> If Inngest's documentation and this file disagree on *how to structure this*, this file wins. If they disagree on an API shape or dashboard path, their docs win — update this file afterwards.

**Only open this file if the interview genuinely called for it.** Sending one email, resizing one image, or writing one row does not need a job queue; a server action does that fine. This earns its place when work must survive a restart, retry on failure, run on a schedule, fan out over many items, or wait for something that takes minutes to days. If none of that came up, close this file — an unused job system is pure overhead.

> Inngest's SDK changed enough at its last major that older examples are actively wrong: triggers moved *inside* the config object, `EventSchemas` was replaced, and the SDK now assumes production unless told otherwise. Almost every tutorial and blog post online predates that. Use what's below; if something doesn't compile, check Inngest's current docs rather than a search result.

## Install

```bash
pnpm add inngest
pnpm add -D inngest-cli concurrently
```

Append to `.env`:

```
INNGEST_DEV=1
```

That one line is the whole local setup. `INNGEST_DEV=1` tells the SDK to talk to the local dev server and skip signature checks — without it the SDK assumes production and demands a signing key. Production keys come later, at deploy time.

Change the `dev` script in `package.json` so both processes start together:

```json
"dev": "concurrently -n next,inngest -c cyan,magenta \"next dev\" \"inngest-cli dev -u http://localhost:3000/api/inngest\"",
```

`concurrently` rather than `&` or `&&`: those behave differently across shells and don't work on Windows at all.

The `localhost:3000` in that script is hard-coded, and it must match the port the app actually runs on. When 3000 is busy Next moves itself to another port without asking, and the Inngest dev server then looks for the app at an address nothing is listening on — it starts, and lists no functions. Either free port 3000, or pin the port on both sides (`next dev -p <port>` and the same port in the `-u` address).

## Configure

### The client

`src/lib/inngest/client.ts`:

```ts
import { Inngest } from "inngest";

export const inngest = new Inngest({
  id: "my-app", // kebab-case app name; this id is what production syncs against
  checkpointing: { maxRuntime: "200s" },
});
```

`maxRuntime` must sit comfortably below the hosting platform's function timeout. On Vercel that's 300 seconds, so 200 is a safe margin — the SDK wraps up and hands back before the platform kills the request.

### The endpoint

`src/app/api/inngest/route.ts`:

```ts
import { serve } from "inngest/next";
import { inngest } from "@/lib/inngest/client";
import { functions } from "@/lib/inngest/functions";

export const maxDuration = 300;

export const { GET, POST, PUT } = serve({ client: inngest, functions });
```

All three verbs are required: `PUT` registers the app, `POST` runs functions, `GET` is introspection. Miss one and the dev server finds the app but can't do anything with it.

### A function

`src/lib/inngest/functions.ts`. Name the function after the real work from the interview — `generate-monthly-report`, `import-csv`, `send-digest` — not `process-task`.

```ts
import { NonRetriableError } from "inngest";
import { eq } from "drizzle-orm";
import { inngest } from "./client";
import { db } from "@/lib/db";
import { jobs } from "@/lib/db/schema";

export const generateReport = inngest.createFunction(
  {
    id: "generate-report",
    triggers: { event: "app/report.requested" },
    retries: 4,
    onFailure: async ({ event, error }) => {
      await db
        .update(jobs)
        .set({ status: "failed", error: error.message, finishedAt: new Date() })
        .where(eq(jobs.id, event.data.event.data.jobId));
    },
  },
  async ({ event, step, runId }) => {
    const { jobId } = event.data;

    await step.run("mark-running", async () => {
      // Store the run id: it is what ops.md's detail view and the cancel button call the provider with.
      await db.update(jobs).set({ status: "running", runId }).where(eq(jobs.id, jobId));
    });

    const rows = await step.run("gather-data", async () => {
      // Throwing here retries. Throwing NonRetriableError gives up immediately.
      return gatherRowsFor(event.data.userId);
    });

    const file = await step.run("render-pdf", async () => renderPdf(rows));

    await step.run("finish", async () => {
      await db
        .update(jobs)
        .set({ status: "completed", result: { file }, finishedAt: new Date() })
        .where(eq(jobs.id, jobId));
    });
  },
);

export const functions = [generateReport];
```

**How the durability actually works, so the code is written correctly:** each `step.run` result is saved. If the function fails or the server restarts, it runs again from the top — but every step that already finished returns its saved value instead of re-executing. That has two consequences worth stating to the user:

- Code *outside* a step re-runs every time. Anything with a side effect belongs inside a `step.run`.
- Step ids must be stable and unique within the function. Renaming one mid-flight orphans its saved result.

Throw `NonRetriableError` when retrying cannot help — a missing record, invalid input. Plain errors retry with backoff.

### The jobs table

Postgres branch shown; on SQLite use a `text` id with `$defaultFn(() => crypto.randomUUID())`, `integer` timestamps, and `text("input", { mode: "json" })` in place of `jsonb` (SQLite has no `jsonb` column), per `references/database.md`.

```ts
export const jobs = pgTable("jobs", {
  id: uuid("id").primaryKey().defaultRandom(),
  kind: text("kind").notNull(),
  userId: text("user_id").references(() => user.id, { onDelete: "cascade" }),
  status: text("status").notNull().default("queued"), // queued | running | completed | failed | cancelled
  eventId: text("event_id"),
  runId: text("run_id"),
  input: jsonb("input"),
  result: jsonb("result"),
  error: text("error"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  finishedAt: timestamp("finished_at", { withTimezone: true }),
});
```

`userId` is `text`, not `uuid`, because it points at a Better Auth table — the trap `references/database.md` warns about. Drop the column entirely if the app has no accounts. Where the data is shared by a team rather than private to each user, use `onDelete: "set null"` instead of `cascade`, so a member who leaves does not take the team's job history with them. A scheduled job (below) has no user and leaves the column empty.

`runId` is filled by the function itself as its first step. `references/ops.md`'s detail view needs it — without a stored run id there is nothing to ask the provider about.

```bash
pnpm db:generate
pnpm db:migrate
```

### Starting a job

One helper, so the row always exists before the event fires:

```ts
// src/lib/inngest/enqueue.ts
import "server-only";
import { inngest } from "./client";
import { db } from "@/lib/db";
import { jobs } from "@/lib/db/schema";
import { eq } from "drizzle-orm";

export async function enqueue(
  kind: string,
  eventName: string,
  input: Record<string, unknown>,
  userId?: string,
) {
  const [job] = await db
    .insert(jobs)
    .values({ kind, userId, input, status: "queued" })
    .returning();

  const { ids } = await inngest.send({
    name: eventName,
    data: { ...input, jobId: job.id, userId },
  });

  await db.update(jobs).set({ eventId: ids[0] }).where(eq(jobs.id, job.id));
  return job;
}
```

Storing the returned event id is what later lets the app ask Inngest what happened to this exact run.

## Work on a schedule

The interview offers this path by name — a nightly tidy-up, a weekly digest. Skip the section if nothing in the app runs on a clock.

Not yet built with this skill: confirm the cron trigger shape, `step.sendEvent` and the fields `onFailure` receives in the check-what's-current step, and prove it with the Verify items below.

Three things differ from a job somebody started:

- **Nobody calls `enqueue()`.** The schedule fires the function directly, so the function inserts its own `jobs` row as its first step and updates it at the end. The hard rule still holds: every run has a row.
- **It fans out per user.** The scheduled function decides *who*; a second, event-triggered function does the work for *one* person. One slow or failing user then retries alone instead of re-sending the digest to everybody.
- **"Stop it" means a pause switch, not a cancel button.** Cancelling one run does nothing about next week's. The switch is stored in the app, the function checks it, and the system page shows it.

The pause switch, in `src/lib/db/schema.ts` (Postgres shown; on SQLite `integer("paused", { mode: "boolean" })` and an `integer` timestamp):

```ts
export const jobSwitch = pgTable("job_switch", {
  kind: text("kind").primaryKey(), // "weekly-digest"
  paused: boolean("paused").notNull().default(false),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});
```

No row means running. `references/ops.md` lists each scheduled kind on the system page with its state, its last run from the `jobs` table, and a Pause / Resume button. The button is a server action behind `requireAdminAction()` from `src/lib/auth-guards.ts` that upserts the row. That guard does not exist until `references/settings.md` has run, and the button lives on the system page `references/ops.md` builds, so write the pause table and the check inside the function now, and the button when ops runs.

The scheduled function, added to `src/lib/inngest/functions.ts`:

```ts
export const weeklyDigest = inngest.createFunction(
  {
    id: "weekly-digest",
    // Put the person's timezone in the expression, or "Monday 7am" means 7am UTC.
    triggers: { cron: "TZ=Europe/London 0 7 * * 1" },
    retries: 2,
    onFailure: async ({ event, error }) => {
      // No jobId travelled with a cron event, so find the row by the run that failed.
      await db
        .update(jobs)
        .set({ status: "failed", error: error.message, finishedAt: new Date() })
        .where(eq(jobs.runId, event.data.run_id));
    },
  },
  async ({ step, runId }) => {
    const jobId = await step.run("open-job-row", async () => {
      const [job] = await db
        .insert(jobs)
        .values({ kind: "weekly-digest", status: "running", runId })
        .returning();
      return job.id;
    });

    const paused = await step.run("check-pause", async () => {
      const [row] = await db.select().from(jobSwitch).where(eq(jobSwitch.kind, "weekly-digest"));
      return row?.paused ?? false;
    });

    if (paused) {
      await step.run("finish-paused", async () => {
        await db
          .update(jobs)
          .set({ status: "cancelled", result: { skipped: "paused" }, finishedAt: new Date() })
          .where(eq(jobs.id, jobId));
      });
      return;
    }

    const userIds = await step.run("list-recipients", async () => listDigestRecipients());

    if (userIds.length > 0) {
      await step.sendEvent(
        "fan-out",
        userIds.map((userId) => ({ name: "app/digest.requested", data: { userId, parentJobId: jobId } })),
      );
    }

    await step.run("finish", async () => {
      await db
        .update(jobs)
        .set({ status: "completed", result: { sent: userIds.length }, finishedAt: new Date() })
        .where(eq(jobs.id, jobId));
    });
  },
);
```

The per-user function is an ordinary event-triggered one, shaped like `generateReport` above with `triggers: { event: "app/digest.requested" }`: gather that user's week, skip them if there is nothing to say, send one email through `sendEmail` from `references/email.md`. Add both functions to the exported `functions` array. The parent row records how many were handed out; give each person their own `jobs` row only if the app shows people their own digest history — otherwise `email_log` already records every send.

An app with no accounts has no users to fan out over: the scheduled function does the work itself in its own steps, and the rest of this section still applies.

### Optional mail asks first, and can be refused signed out

A digest is optional mail, so the per-user function does two things a password reset never does:

- **It checks the person's preference.** `references/settings.md` builds the `notification_preference` table with one category per kind of optional mail. `listDigestRecipients()` leaves out anyone whose row for the digest category is disabled; no row means enabled. That table arrives later in Step 4 than this file, so write the scheduled function now and add the preference check when the table exists — the Verify item for it is deferred for the same reason.
- **It carries an unsubscribe link that works signed out.** People unsubscribe from their inbox on a phone that has never signed in to the app. The link is `${APP_URL}/unsubscribe?u=<user id>&c=<category>&sig=<signature>`, where the signature is an HMAC of the user id and category made with a server-side secret, so nobody can unsubscribe somebody else by editing the address. The page at `src/app/unsubscribe/page.tsx` checks the signature and shows one button; pressing it posts to a server action that checks the signature again and sets that one category to disabled. The write happens on the button, not on opening the link, because mail scanners open every link in a message. The page needs no session and says what it did in a sentence, with a link to `/settings/notifications` for people who want to turn it back on.

Build `APP_URL` into the link with `||`, never `??` — an empty value has to fall through to the next one, and `??` keeps the empty string.

## Watching it work locally

`pnpm dev` starts Next.js and the Inngest dev server together. The dev server's UI is at **http://localhost:8288** — no Docker, no account, no signup.

It shows every event, every run, and every run's steps with timings, inputs, outputs, and retry attempts. Show it to the user: for most people this is the first time background work has been anything other than invisible.

Trigger something from the app and watch it appear. Then make a step throw on purpose and watch it retry — that demonstration is worth more than any explanation of what "durable" means.

## Going to production

1. Sign up at https://app.inngest.com.
2. **Vercel**: install the Inngest integration from the Vercel marketplace. It sets both keys and re-syncs the app on every deploy, which is the part people otherwise forget.
3. **Anywhere else** (Railway, Fly, Render, a container): copy the **Event Key** and **Signing Key** from the environment's settings into the host's environment variables as `INNGEST_EVENT_KEY` and `INNGEST_SIGNING_KEY`, deploy, then use **Apps → Sync New App** in the Inngest dashboard with the URL `https://<their-domain>/api/inngest`. Re-sync after any deploy that adds or changes a function.

Do **not** set `INNGEST_DEV` in production — the SDK already defaults to cloud mode, and setting it to `0` is unnecessary noise.

Two things that waste an afternoon if unmentioned:

- On a custom domain, set `INNGEST_SERVE_ORIGIN` to it. Otherwise Inngest syncs the `*.vercel.app` deployment URL and calls the wrong host.
- Vercel's **Deployment Protection** blocks Inngest from reaching the endpoint, so syncs fail with an authentication error that looks like a key problem. Either disable it or configure a protection bypass.

**Free tier:** 50,000 function executions a month, 5 running at once, and 24 hours of history. The execution count is per *step*, not per job — a five-step function burns five. Say this at hand-off so nobody is surprised.

## Controlling jobs from inside the app

`references/ops.md` builds the page. This file provides what it calls. Add `src/lib/inngest/admin.ts`:

```ts
import "server-only";

const API = "https://api.inngest.com/v2";
const key = process.env.INNGEST_API_KEY;

async function call(path: string, init?: RequestInit) {
  if (!key) return null;
  const res = await fetch(`${API}${path}`, {
    ...init,
    headers: { Authorization: `Bearer ${key}`, "Content-Type": "application/json" },
    cache: "no-store",
  });
  return res.ok ? res.json() : null;
}

export const getRun = (runId: string) => call(`/runs/${runId}?includeOutput=true`);
export const getTrace = (runId: string) => call(`/runs/${runId}/trace`);
export const cancelRun = (runId: string) => call(`/runs/${runId}/cancel`, { method: "POST" });
export const rerunRun = (runId: string) => call(`/runs/${runId}/rerun`, { method: "POST" });
export const runsForEvent = (eventId: string) => call(`/events/${eventId}/runs`);
```

`INNGEST_API_KEY` is a separate key, created under the account menu → **API Keys** (it starts `sk-inn-api-`), not the event or signing key.

**This module is server-only and must stay that way.** The key is account-wide and not scoped to a user — anything it can read, it can read for every customer. The app's own pages query the `jobs` table (scoped by `userId`), and only reach for these functions to show live detail on a job the current user is already allowed to see.

Cancelling takes effect at the next step boundary, not mid-step. Say that in the UI — "cancelling" is an honest label, "cancelled" the instant the button is clicked is not.

These functions call Inngest's cloud API. A run started against the local dev server lives only in that dev server, so cancel, re-run and the live trace may not be possible locally at all: with no `INNGEST_API_KEY` every call returns `null`, and with one, the cloud has never heard of a local run id. Treat them as provable on the deployed app only, make the buttons say so when the call returns `null`, and do not report them as working from a local test. Locally, cancel and re-run from the dev server's own UI at http://localhost:8288.

For scheduled work, the control is the pause switch from *Work on a schedule*, not `cancelRun`.

## Verify

Do not create an account to run these. The first account belongs to the real person and is made in Step 6. The feature that starts the job, the settings area and the system page do not exist yet either, so this step checks only what can be checked now.

Checked now:

- `pnpm exec tsc --noEmit` passes and the `jobs` migration has run (and `job_switch`, if anything is scheduled).
- `pnpm dev` starts both processes and the dev server at http://localhost:8288 lists the app and its functions. If it lists none, check the port in the `dev` script against the port the app is on.
- Sending the event by hand from the dev server's UI (or, for a schedule, pressing its trigger button there) produces a run with each step and its timing. For a scheduled function, a `jobs` row appears with its `runId` filled.
- A step made to throw retries, is visible retrying, and the `jobs` row ends as `failed` with the error recorded. Remove the deliberate throw afterwards.
- With all Inngest env vars absent, `pnpm build` passes and the app still starts.

Deferred, because they need the finished app or an account:

- Deferred to Step 5, once the feature exists: triggering it from the app's own UI creates a `jobs` row before the event is sent, the row reaches `completed`, and the app shows that state without the user refreshing something obscure. With Inngest env vars absent, the feature shows a plain "background jobs aren't set up yet" notice.
- Deferred to Step 6, after the user has signed up: job rows are scoped to the signed-in user in every query. Which isolation check applies depends on the access shape, and `references/verify.md` says which — on a one-owner app a second account cannot exist, so the check is reading the queries.
- Deferred until `references/settings.md` has run: someone who turned the digest off is left out of the recipients, and the unsubscribe link works in a signed-out browser, refuses a changed signature, and only writes when the button is pressed.
- Deferred until `references/ops.md` has run: the system page shows the pause switch, pausing makes the next scheduled run finish as skipped, and with `INNGEST_API_KEY` absent the detail view falls back to what the `jobs` table knows instead of crashing.
- Deferred to the deployed app: cancel, re-run and the live trace. If the app was not deployed, the hand-off says these were not proven.
