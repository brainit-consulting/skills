# System visibility

Last verified: 2026-09-21

**Purpose:** Give the app an inside view of itself — what's configured, what happened, what's running in the background, and what email went out. Every app gets this, scaled to what it actually has.

**This step is not optional.** Everything else in this skill is a branch; this one always runs. An app whose owner cannot see why an email didn't arrive, or that a job has been retrying for an hour, is an app they can only debug by reading logs on a hosting dashboard — which is exactly the moment a new project stops being fun. The panels below appear only if the app has the thing they describe, but the page itself always exists.

> **Hard rule: never render a secret, or part of one.** This page reports whether something is *configured*, never what it is configured with. No API keys, no connection strings, no "sk-...abc4" tails, not even in a tooltip or a copy button. A masked key is still a key with fewer characters to guess, and this page exists to be looked at.

## Who can see it

**If the app has sign-in:** the page lives at `/settings/system`, is linked in the settings nav only for admins, and is guarded on the server by the helpers in `src/lib/auth-guards.ts`: the page calls `requireAdmin()`, which answers a non-admin with a 404, and every server action and route handler behind it calls `requireAdminAction()`, which throws. Hiding the link is presentation; the guard is the security boundary. On a one-owner app the owner is the admin. Where the app has more than two roles, `references/auth.md` names the one role that may see this page, and `requireAdmin()` checks for that role.

**If the app has no accounts:** there is nobody to be an admin, so what happens depends on where the app runs. The build sheet already says which.

- **Deployed where strangers can reach it** (a public site, a shared tool): the page renders in development only and answers 404 in production.
- **A tool that only ever runs on the person's own machine:** the page stays available in production mode too (`pnpm build && pnpm start`). The only person who can reach it is the person it is for, and removing it would take away their one view of what went wrong. Leave the `notFound()` line out. If the app is ever deployed, that line goes back in first; say so at hand-off.

```tsx
// src/app/settings/system/page.tsx — no-accounts apps only
import { notFound } from "next/navigation";

export default async function SystemPage() {
  // Deployed public app: keep this line. Tool that stays on the person's machine: remove it.
  if (process.env.NODE_ENV === "production") notFound();
  // ...panels
}
```

**Where it is linked, with no accounts:** there is no settings nav, so put a small "System" link in the footer of `/`. On a deployed public app render that link in development only, the same condition as the page, so strangers are not shown a link that leads to a 404.

Say why to the user in one line: without accounts there's no way to tell them apart from anyone else on the internet, so a page listing what's wired up doesn't belong on a public site. It's still there while they're building, which is when they need it.

## Health and configuration — always

One card listing every integration the app could have, and whether it's ready. Presence of the environment variable only.

```ts
// src/lib/system-status.ts
import "server-only";
import { sql } from "drizzle-orm";
import { db } from "@/lib/db";
import { usingBlobStorage } from "@/lib/storage"; // only if references/storage.md ran

export type Check = { name: string; ready: boolean; hint: string };

export async function getSystemStatus() {
  const integrations: Check[] = [
    // Include only the ones this app actually set up.
    { name: "Email (Resend)", ready: Boolean(process.env.RESEND_API_KEY), hint: "RESEND_API_KEY" },
    { name: "Background jobs (Inngest)", ready: Boolean(process.env.INNGEST_EVENT_KEY) || process.env.INNGEST_DEV === "1", hint: "INNGEST_EVENT_KEY" },
    { name: "File uploads", ready: usingBlobStorage, hint: "Blob store not connected — local folder in use until it is" },
    { name: "Payments", ready: Boolean(process.env.POLAR_ACCESS_TOKEN), hint: "POLAR_ACCESS_TOKEN" },
    { name: "AI", ready: Boolean(process.env.OPENROUTER_API_KEY), hint: "OPENROUTER_API_KEY" },
    { name: "Agent access (MCP)", ready: Boolean(process.env.BETTER_AUTH_URL), hint: "BETTER_AUTH_URL — must be the app's public URL" },
    { name: "Canonical URL", ready: Boolean(process.env.APP_URL || process.env.BETTER_AUTH_URL), hint: "APP_URL — the app's public address" },
  ];

  let database = false;
  try {
    await db.execute(sql`select 1`); // Postgres. On SQLite: db.run(sql`select 1`)
    database = true;
  } catch {
    database = false;
  }

  return { integrations, database };
}
```

**On the SQLite branch the database check is ``db.run(sql`select 1`)``.** Measured: `db.execute` does not exist on the SQLite driver and the file fails to compile with TS2339.

The uploads row imports `usingBlobStorage` from `references/storage.md`'s module instead of checking one variable, because that module accepts either of two credentials and the row has to agree with what the upload code will do. The canonical URL row uses `||`, not `??`: a variable that is present but empty is still not configured.

Render each as a row with a green/grey state and the plain-language hint — the *name* of the variable to set, never its value. "Not configured yet" is a normal state here, not an error: it is exactly what a half-built app looks like, and showing it as a warning trains the user to ignore warnings.

The canonical URL row is the odd one out and earns its place anyway: it gates no feature, so nothing breaks without it — the sitemap and every canonical link just quietly point at `localhost` in production, which nobody notices until a search engine has already read them. `references/seo.md` sets it up; this row is where its absence becomes visible.

## Activity log — always

The one panel that exists even in the smallest app. A record of things that happened, so the user can answer "when did that change?"

```ts
export const activityLog = pgTable("activity_log", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: text("user_id").references(() => user.id, { onDelete: "set null" }),
  action: text("action").notNull(),
  detail: jsonb("detail"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});
```

Postgres branch shown. On SQLite use `text` ids and `integer` timestamps per `references/database.md`, and `detail: text("detail", { mode: "json" })`, because SQLite has no `jsonb` column. Drop `userId` if there are no accounts.

Every timestamp in a log table (`activity_log`, `email_log`, `jobs`, `mcp_call_log`) uses `withTimezone: true` on Postgres, per `references/database.md`'s rule. A log whose times are wrong by the owner's offset cannot answer "when did that happen?". If one of those tables was written earlier without it, correct the column in a new migration.

`onDelete: "set null"` rather than `cascade` — the log outlives the account so "who deleted this?" is still answerable afterwards, with only an anonymous row left behind.

One helper, **called from the shared `src/lib/<domain>` functions, not from the server actions.** A server action and an agent tool call both go through the same domain function, so logging there records both. Logging in the action would miss everything an agent does.

```ts
// src/lib/activity.ts
import "server-only";
import { db } from "@/lib/db";
import { activityLog } from "@/lib/db/schema";

export async function logActivity(action: string, detail?: unknown, userId?: string) {
  await db.insert(activityLog).values({ action, detail, userId });
}
```

**Record who the caller was inside `detail`, as `via: "owner" | "client" | "agent"`.** No new column. The domain function takes the caller from whoever called it: the server action passes `"owner"` (or `"client"` where the app has a second kind of person, such as a customer booking through a public page), and the MCP tool in `references/mcp.md` passes `"agent"`.

```ts
await logActivity("hike.created", { id: hike.id, via }, userId);
```

Log the app's real verbs, in the app's own words — `hike.created`, `invoice.sent`, `member.invited` — not `POST /api/x`. Log writes, not reads. Never put a password, token, or full payload in `detail`; the ids and the changed fields are enough.

Render newest-first with the user's name where there is one, the `via` value beside it, and a date filter. Keep it to the last few hundred rows in the page query — this table grows.

## Background jobs — only if `references/jobs.md` ran

The point of this panel is that background work stops being invisible. It reads the app's own `jobs` table, which is the record, and reaches out to Inngest only for live detail on a single run.

```tsx
const rows = await db
  .select()
  .from(jobs)
  .orderBy(desc(jobs.createdAt))
  .limit(50);
```

Show: what it was, who it was for, status, when it started, how long it took, and the error if it failed. Group or filter by status so a wall of completed jobs doesn't bury the one that broke.

Each row opens a detail view that calls `getRun` and `getTrace` from `src/lib/inngest/admin.ts` to show the step-by-step timeline — which step failed, how many attempts, what each returned. **Those calls take the provider's run id, so the run id has to be stored on the `jobs` row**; `references/jobs.md` stores it. A row with no run id shows what the table knows and says live detail is not available for it. Two buttons:

- **Cancel** → `cancelRun`. Label the result "cancelling", not "cancelled": it takes effect at the next step boundary, not instantly.
- **Re-run** → `rerunRun`. This starts a *new* run; show the new id rather than pretending the old one restarted.

Both go through a server action that calls `requireAdminAction()` first. The Inngest API key is account-wide and never reaches the browser.

Cancel and re-run may not be possible against the local dev server, as `references/jobs.md` notes. Where they are not, the buttons say so instead of failing. For scheduled work, this panel also shows the pause switch `references/jobs.md` defines.

If `INNGEST_API_KEY` isn't set, the list still works from the `jobs` table and the detail view says live detail needs the key — degraded, not broken.

Non-admin users should still see *their own* jobs somewhere in the app if the app's work is job-shaped ("your export is being prepared"). That belongs on the relevant feature page, scoped by `userId`, not here.

## Agent activity — only if `references/mcp.md` ran

Reads `mcp_call_log` newest-first: which tool, which client, which user, worked or failed, how long it took, and how many rows went out.

```tsx
const rows = await db
  .select()
  .from(mcpCallLog)
  .orderBy(desc(mcpCallLog.createdAt))
  .limit(100);
```

This panel shows **reads as well as writes**, unlike the activity log above. An agent's writes appear in both places: here as a tool call, and in the activity log with `via: "agent"`, because the shared domain function logged them. When a person reads their own data it is noise; when an agent does, it is the event — a read is how data leaves the app, and the row count is the number that tells you how much. Sort or filter by it and an agent pulling an entire account stands out from one answering a question.

Surface the error text on failed rows. "Claude said it couldn't find anything" is the support question this panel exists to answer, and the answer is usually a scope the user never approved or a tool that threw on an argument the model guessed.

Show the client name beside the tool. One person can have Claude Code and the Claude connector both attached, and "which one did that?" is otherwise unanswerable.

Non-admin users should be able to see their *own* agent activity from `/settings/connections` — the same rows scoped by `userId`. Admins see everyone's here.

## Email log — only if `references/email.md` ran

Reads `email_log`, newest first: recipient, subject, template, status, and when. Status comes from the send itself and then from the Resend webhook if it was set up, so `sent` becoming `delivered` or `bounced` is visible here.

This is the panel that answers the most common support question a small app gets — "I never got the email" — with an actual answer: it was never sent because the key is missing, it bounced, or it was delivered and is in their spam folder.

Offer a **Resend** action on a failed row, going through the same `sendEmail` helper so the retry is logged too. The action calls `requireAdminAction()` first.

Show the recipient address, since an admin needs it to help. **The list shows no message bodies.** The owner may open one stored message on demand, from its row: `references/email.md` stores the HTML for exactly that, so "what did the email actually say?" has an answer. Load the body only when the row is opened, behind the same admin guard, and render it in a sandboxed `<iframe srcDoc>` so nothing in a stored message runs inside the system page. Opening a stored message has not yet been built with this skill: prove it with the Verify item below. Where the app has a contact form, these bodies include what visitors wrote; `references/legal.md` discloses that.

## Verify

**No account is created to run these checks.** The first account belongs to the real person and is made in Step 6; on a one-owner app a test account would take the owner's seat. What needs a signed-in admin is deferred and marked so.

Checked now:

- Types compile. On SQLite that includes the `db.run` health check.
- Apps with accounts: signed out, `/settings/system` answers with a redirect to sign-in or a 404, never a 200, and the response contains none of the panels.
- No-accounts app, deployed and public: the page renders with `pnpm dev` and returns 404 after `pnpm build && pnpm start`, and the "System" link in the footer of `/` is absent in production. `references/verify.md`'s route sweep counts that 404 as the pass.
- No-accounts tool that stays on the person's machine: the page renders in both modes and the footer link is present in both.
- No-accounts apps only, where the page can be opened without an account: the health card lists every integration this app set up, correctly reflects which are configured, and shows no key, token, or connection string anywhere — check the rendered HTML, not just the screen. Doing the app's main action writes an activity row that reads in plain language.
- Every `logActivity` call sits in a `src/lib/<domain>` function, and none in a server action or a tool handler: search for the calls and read where they are.

Deferred to Step 6, after the user has signed up:

- The system page is linked in settings navigation for the admin. On an app where a second, non-admin account can exist, it is not linked for that account, and visiting `/settings/system` directly as that account answers 404 from the server, not a 500. On a one-owner app this probe is not applicable: only one account can exist.
- The health card check above, on apps with accounts.
- Doing the app's main action writes an activity row that reads in plain language, with `via: "owner"` in its detail.
- Jobs branch: a running job appears with its steps, cancelling it moves it out of running, and re-running it starts a new run. Where cancel and re-run are not possible against the local dev server, the page says so.
- Email branch: an email sent from the app appears in the log with the right status, the list shows no bodies, opening a row shows that one message, and a failed send can be retried from the page.
- Agent access branch: a tool called from Claude appears with its client and row count, a read shows up as well as a write, a write also appears in the activity log with `via: "agent"`, and a failed call shows the reason it failed.

Checked in `references/verify.md`, not here:

- With the optional keys blanked, the page still renders, shows those integrations as not configured, and nothing crashes. The database URL and `BETTER_AUTH_SECRET` are required and are never blanked.
