# Account settings

Last verified: 2026-09-21

**Purpose:** The place a signed-in person manages themselves — their profile, whether their email is confirmed, their password, the devices they're signed in on, and how to leave. Every app with accounts owes people this, and an app without it feels unfinished the first time someone asks "how do I change my password?"

**Prerequisite: sign-in must exist.** If the app has no accounts, skip this file entirely and go straight to `references/ops.md`, which builds the system page on its own.

> **Hard rule: every panel here is built on Better Auth's own API.** Do not write a route that updates the `user` table directly, hashes a password by hand, or deletes a user row with Drizzle. Better Auth owns anything that belongs to a user — the same rule that governs payments — and going around it means sessions that don't get revoked, tokens that don't get cleaned up, and a password hash format that quietly diverges from the one sign-in checks against. The one exception is an admin changing somebody's `role`, which is the app's own field; `references/auth.md` has it and the reason.
>
> If Better Auth's documentation and this file disagree on *how to structure the page*, this file wins. If they disagree on a method name or option, their docs win — update this file afterwards, because this is the fastest-moving dependency in the stack.

## What changes in `src/lib/auth.ts` first

**The `role` field and the first-account hook already exist.** `references/auth.md` added both when sign-in was built, and the `role` column was in the first auth migration. Do not add them again here. If `src/lib/auth.ts` somehow has neither, go back to that file's *The role field and the first account* and put in the version that matches this app's access shape before building anything below.

This step adds four things to the same config: `session.freshAge`, `changeEmail`, `deleteUser` and the rate limits. Skipping them causes bugs that only appear in production.

```ts
export const auth = betterAuth({
  // ...existing config

  session: {
    freshAge: 0,
  },

  user: {
    // additionalFields.role is already here, from references/auth.md. Leave it as it is.
    changeEmail: { enabled: true },
    deleteUser: { enabled: true },
  },

  // databaseHooks: the first-account hook is already here too. Leave it as it is.

  rateLimit: {
    enabled: true,
    storage: "database",
    customRules: {
      "/send-verification-email": { window: 60, max: 2 },
      "/change-email": { window: 60, max: 3 },
      "/change-password": { window: 60, max: 5 },
      "/delete-user": { window: 60, max: 3 },
    },
  },
});
```

**`freshAge: 0`** — by default Better Auth treats a session that was *created* more than 24 hours ago as "not fresh" and refuses to list sessions. It is measured from when the session started, not from when it was last used, so being active does not keep it fresh. The devices panel would then work for a day and start returning 403 to every returning user, while the revoke buttons kept working, which reads as a bug in the page rather than a policy. Turning the gate off and requiring a password on the genuinely dangerous action (deletion) is the clearer trade. Say it in a comment so nobody "fixes" it back.

**`input: false` on `role`**, which `references/auth.md` set, is the security control, not a formality. Without it the role is an ordinary profile field and a user can set their own to `admin` through the normal update call. Check it is still there.

Better Auth already ships its own tight limits for sign-in, sign-up, change-password, change-email, send-verification-email and request-password-reset. A custom rule *replaces* the built-in one for that path, so the rules above are only worth keeping where they are stricter than the default; check the current defaults in Step 2 and drop the ones that are looser.

`rateLimit` is disabled in development and defaults to in-memory storage, which does nothing across serverless instances — `storage: "database"` is what makes the resend cooldown real. If `references/mcp.md` already added a `rateLimit` block, merge these rules into its `customRules` rather than replacing the object.

`rateLimit` with `storage: "database"` adds a table (and so does any profile field added under *Profile* below), so regenerate and migrate:

```bash
pnpm dlx auth@latest generate --config src/lib/auth.ts --output src/lib/db/auth-schema.ts -y
pnpm db:generate
pnpm db:migrate
```

## The admin guards

The first account is already the admin: the hook in `references/auth.md` did that, in the version that fits this app's access shape (open sign-up, one owner, or invited people only). What is missing is the check that uses it.

`references/pages.md` already wrote `src/lib/auth-guards.ts` with a cached `getSession()`, `requireUser()` (pages: redirects) and `requireUserAction()` (server actions and route handlers: throws). Add the two admin guards to that file rather than starting a second one:

```ts
import { notFound } from "next/navigation"; // add beside the existing redirect import

/** For pages: a signed-in non-admin sees a 404. A throw here would be a 500 page. */
export async function requireAdmin() {
  const session = await requireUser();
  if (session.user.role !== "admin") notFound();
  return session;
}

/** For server actions and route handlers: throws instead. */
export async function requireAdminAction() {
  const session = await requireUserAction();
  if (session.user.role !== "admin") throw new Error("Not allowed");
  return session;
}
```

On an app with its own role names, `"admin"` here is `ADMIN_ROLE` from `src/lib/roles.ts`, and `requireRole(...roles)` sits in the same file. Both are described in `references/auth.md`.

If `src/lib/auth-guards.ts` is somehow missing, write it first exactly as `references/pages.md` defines it, then add the two guards above. The definitions are repeated here only so this file stands alone; they must stay identical to that file's:

```ts
// src/lib/auth-guards.ts
import "server-only";
import { cache } from "react";
import { headers } from "next/headers";
import { redirect } from "next/navigation";
import { auth } from "@/lib/auth";

// cache() means a page, its layout and its data functions share one lookup per request.
export const getSession = cache(async () => auth.api.getSession({ headers: await headers() }));

/** For pages: sends a signed-out visitor to sign-in. */
export async function requireUser() {
  const session = await getSession();
  if (!session) redirect("/sign-in");
  return session;
}

/** For server actions and route handlers: throws instead of redirecting. */
export async function requireUserAction() {
  const session = await getSession();
  if (!session) throw new Error("Not signed in");
  return session;
}
```

## The shell

`src/app/(dashboard)/settings/layout.tsx` — inside the existing dashboard route group, so it inherits the sign-in check `references/pages.md` already wrote.

```tsx
import Link from "next/link";
import { requireUser } from "@/lib/auth-guards";

export default async function SettingsLayout({ children }: { children: React.ReactNode }) {
  const session = await requireUser();
  const isAdmin = session.user.role === "admin";

  return (
    <div className="flex flex-col gap-8 md:flex-row">
      <nav className="flex gap-1 overflow-x-auto md:w-48 md:flex-col">
        <Link href="/settings">Profile</Link>
        <Link href="/settings/account">Account</Link>
        <Link href="/settings/security">Security</Link>
        {/* Notifications — only if email was set up */}
        {/* Connected apps — only if agent access was set up */}
        {/* Billing — only if payments were set up */}
        {/* Cookie preferences — only if a consent banner was built */}
        {isAdmin && <Link href="/settings/system">System</Link>}
      </nav>
      <div className="flex-1 space-y-6">{children}</div>
    </div>
  );
}
```

The dashboard layout's `main` already sets the content width and the padding, so this shell adds neither: a second `mx-auto max-w-* px-* py-*` wrapper doubles the padding, and a second `main` inside the first is invalid markup.

One route per section, one shadcn `Card` per concern, each card with its own save button. Never one giant form with a single Save — the user has no idea what they're about to change, and one validation error blocks everything.

**Build only the sections this app has.** A personal tool with no payments has no Billing tab; an app that never chose email has no Notifications tab. An empty section is worse than a missing one.

## Profile — `/settings`

Name and avatar. `authClient.updateUser({ name, image })`. If the app took uploads (`references/storage.md`), the avatar goes through the existing upload route rather than a second one.

If the interview turned up profile fields that belong to *their* app — a display handle, a default currency, a home trail — add them as `additionalFields` and put them here. This is the section that stops the settings area feeling generic.

## Account — `/settings/account`

**Apps without email: show the address and nothing else.** With no sender, nothing can ever confirm an address, so `emailVerified` stays `false` for ever. Do not render the badge, the resend control or the banner at the end of this file: they would tell the person to click a link that was never sent. The card below is for apps where `references/email.md` ran.

**Email and verification status.** Show the address with a badge, and the resend control only when it's needed:

```tsx
"use client";
import { useEffect, useState } from "react";
import { authClient } from "@/lib/auth-client";

export function VerifyEmailCard({ email, verified }: { email: string; verified: boolean }) {
  const [cooldown, setCooldown] = useState(0);

  useEffect(() => {
    if (!cooldown) return;
    const t = setTimeout(() => setCooldown((c) => c - 1), 1000);
    return () => clearTimeout(t);
  }, [cooldown]);

  if (verified) return <Badge variant="secondary">Email confirmed</Badge>;

  return (
    <div>
      <p>We sent a confirmation link to {email}.</p>
      <Button
        disabled={cooldown > 0}
        onClick={() =>
          authClient.sendVerificationEmail(
            { email, callbackURL: "/settings/account" },
            {
              onSuccess: () => setCooldown(60),
              onError: (ctx) => {
                const retry = Number(ctx.response.headers.get("X-Retry-After"));
                setCooldown(retry || 60);
              },
            },
          )
        }
      >
        {cooldown > 0 ? `Resend in ${cooldown}s` : "Resend confirmation"}
      </Button>
    </div>
  );
}
```

Hide the button entirely once verified — the endpoint returns a 400 for an already-verified user, and surfacing that error where a success message belongs is confusing.

**Changing email** needs a sender. `changeEmail.enabled: true` on its own returns a 400 unless `emailVerification.sendVerificationEmail` is also configured, which only happens if `references/email.md` ran.

- **Email was set up:** `authClient.changeEmail({ newEmail, callbackURL: "/settings/account" })`. A link goes to the *new* address and the change applies when it's clicked. Word the confirmation as "check your new inbox", never "email changed" — Better Auth deliberately returns success even when the address already belongs to somebody else, so that the page can't be used to discover who has an account.
- **Email was not set up:** render the field disabled with one honest line — "changing your email needs email sending set up first" — rather than a button that 400s.

**Linked sign-in methods** — only if Google sign-in was chosen. `authClient.listAccounts()` to show what's connected, `linkSocial` and `unlinkAccount({ accountId })` for the buttons (it takes the account row's `id`). Keep `account.accountLinking.allowUnlinkingAll` at its default `false` so nobody can remove their last way in. A row with `providerId: "credential"` means they have a password.

## Security — `/settings/security`

**Change password:**

```ts
await authClient.changePassword({
  currentPassword,
  newPassword,
  revokeOtherSessions: true,
});
```

`currentPassword` is required. `revokeOtherSessions: true` should be checked by default — changing a password usually means "someone might have it".

**Set a password**, for accounts created through Google that have never had one. `auth.api.setPassword` is server-only, so it goes in a server action. It throws if a password already exists — use `listAccounts()` to decide which card to render rather than calling it and catching.

**Active sessions and devices.** `authClient.listSessions()` returns each session with `ipAddress`, `userAgent`, `createdAt` and `token`. Render one row each: a readable device line, the IP, when it was last active, and a **Revoke** button calling `authClient.revokeSession({ token })`. Mark the row whose token matches the current session as **This device** and don't offer to revoke it. Add "Sign out everywhere else" wired to `revokeOtherSessions()`.

Parse the user-agent into something human — "Chrome on Windows" — rather than printing the raw string. A small lookup is fine; a dependency is fine too.

## Notifications — `/settings/notifications`

Only if `references/email.md` ran. Otherwise there is nothing to opt out of and the tab should not exist.

A small table, following the id conventions in `references/database.md`:

```ts
export const notificationPreference = pgTable("notification_preference", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: text("user_id").notNull().references(() => user.id, { onDelete: "cascade" }),
  category: text("category").notNull(),
  enabled: boolean("enabled").notNull().default(true),
});
```

The categories are the ones this app actually sends — the emails named in the interview, not a generic list.

> **Transactional email ignores these preferences.** Password resets, email confirmation, receipts, and account-security notices always send, and never carry an unsubscribe link. Only optional mail — digests, product updates, activity summaries — checks the table. Putting an unsubscribe link on a password reset is how people lock themselves out.

## Connected apps — `/settings/connections`

Only if `references/mcp.md` ran. This is where somebody sees which AI agents can act as them, and takes it back.

One row per granted consent, listed the way `references/mcp.md` lists them: on the server, with `auth.api.getOAuthConsents({ headers: await headers() })`, in the page itself. That file's *Connected apps* section is the source for the call and for the revoke action; if a name there differs from one here, that file wins.

- **The client's name** as it registered itself, with a quiet line saying that name was chosen by whoever connected. With dynamic registration anything can call itself anything, and a user comparing "Claude" against "Claude Desktop " should be able to tell they are different rows.
- **What it can do**, in the app's words — "Read your hikes", "Add and edit hikes" — not the scope strings.
- **When it was granted**, and **when it was last used**, from `mcp_call_log`. Last-used is the one that matters: a connection nobody has touched in three months is the one to revoke.
- **Revoke**, a form button that calls the revoke server action `references/mcp.md` defines. **Do not write a second one here, and do not wire the button to a bare "delete the consent" call.** Revoke is three deletes (the consent, the access tokens, the refresh tokens), then the client row when nobody else uses it, and that file holds the one copy.

Revoking has to break the *next* tool call, not just remove a row from this page. **Deleting the consent does not do that by itself.** Access tokens are signed notes checked without asking the database, so one already issued keeps working until it expires, up to an hour after the user believed they had cut it off. The rest of the fix lives at the endpoint, not on this page: `references/mcp.md` has the route check that looks the consent up on every call. Confirm it against a live connection rather than assuming, and do not write "access stops immediately" on this page until that test has been run.

Put the connector URL at the top of the page with a copy button — the app's own address plus `/mcp`, per `references/mcp.md` — and one line saying where it goes in Claude. It is not a secret, it is the thing somebody needs in order to connect at all, and there is nowhere else they would think to look for it.

Show the person their own agent activity underneath, the same `mcp_call_log` rows `references/ops.md` renders for admins, scoped to `userId`. Seeing that a connection read forty rows an hour ago is what makes the revoke button meaningful.

Link the page from the app's account menu as well as the settings nav. Someone who has just realised an agent has their data will look for it in the obvious place first.

## Billing — `/settings/billing`

Only if payments were set up. There is nothing to build: show the current plan from the server-side subscription state, then link to the provider's hosted portal that `references/payments.md` already wired (`authClient.customer.portal()` for Polar, the Stripe equivalent otherwise). Do not rebuild cancel, invoice, or card-change screens.

## Cookie preferences — `/settings/cookies`

Only if `references/legal.md` built a consent banner. Most apps here have no banner, so most have no tab — and an app that tracks nobody must not grow a page implying it might.

There is almost nothing to build: read the consent cookie, show what it currently says in the app's own words ("Analytics: off"), and reopen the same dialog the banner uses. One component, two entry points — never a second preferences screen that drifts from the first.

**Withdrawing has to take effect, not just be recorded.** Turning a category off clears what the app can clear and stops that script being rendered on the next request, which is the same mechanism `references/legal.md` describes and the reason the choice lives in a cookie the server reads. A preferences page that writes a value while the tag keeps firing is a worse lie than no page at all.

Link it from the footer too. Someone looking for it is signed out as often as not.

## Danger zone — bottom of `/settings/account`

A separate card, visually distinct, with real whitespace between it and anything harmless above it. Deletion is immediate and permanent — there is no grace period and no undo, so the confirmation has to carry the weight.

Configure it in `src/lib/auth.ts`. This is a fragment: it goes inside the existing `user` block, beside `additionalFields` and `changeEmail`. Keep only the parts whose branch ran.

```ts
import { eq } from "drizzle-orm";
import { deleteFile } from "@/lib/storage"; // uploads branch only
import { photo } from "./db/schema"; // uploads branch only: whichever table holds this app's pathnames

// Uploads branch only. beforeDelete fills it, afterDelete empties it, in the same request.
const filesToRemove = new Map<string, string[]>();

// inside betterAuth({ user: { ... } })
deleteUser: {
  enabled: true,

  // Email branch only. Leave these two out when the app has no email: see below.
  deleteTokenExpiresIn: 60 * 60,
  sendDeleteAccountVerification: ({ user, url }) => {
    void sendEmail({
      to: user.email,
      subject: "Confirm deleting your account",
      react: ConfirmDeleteEmail({ url }),
      template: "confirm-delete",
    });
  },

  beforeDelete: async (user) => {
    // 1. Uploads branch: collect the pathnames NOW. Once the user row goes, the rows that
    //    hold them cascade away and the files can never be found again.
    const rows = await db
      .select({ pathname: photo.pathname })
      .from(photo)
      .where(eq(photo.uploadedBy, user.id));
    filesToRemove.set(user.id, rows.map((r) => r.pathname));

    // 2. Payments branch: cancel the subscription here, BEFORE the row goes, through the
    //    provider call references/payments.md set up. Let a failure throw: it stops the
    //    delete, which is better than a live subscription with no account behind it.

    // 3. Agent access branch: remove this user's consents, access tokens and refresh tokens
    //    with the revoke function references/mcp.md defines, once per consent they hold.
  },

  afterDelete: async (user) => {
    // Uploads branch: the rows are gone; now remove the files they pointed at.
    const pathnames = filesToRemove.get(user.id) ?? [];
    filesToRemove.delete(user.id);
    await Promise.allSettled(pathnames.map((p) => deleteFile(p)));
  },
},
```

The two hooks with real content in them are not yet built with this skill: the build that measured deletion had no uploads, payments or agent access. Confirm the hook signatures in the check-what's-current step, and prove them with the Verify items below (the file is gone from storage, the subscription is cancelled, the agent's next call is refused). `deleteFile` comes from `references/storage.md`'s module, and `photo` stands for whichever files table that step made (it has `pathname` and `uploadedBy` columns). Nothing `auth.ts` imports may import `"server-only"`, for the reason `references/auth.md` gives.

**Email or no email decides how deletion is confirmed.**

- **Email was set up:** `sendDeleteAccountVerification` is configured, so `authClient.deleteUser(...)` sends a link and the account goes when it is clicked.
- **Email was not set up:** leave out `sendDeleteAccountVerification` and `deleteTokenExpiresIn`, and do not import `sendEmail`. Deletion then happens on the API call itself, so the typed email address and the password in the dialog are the whole confirmation. Measured on a real build with no email: immediate deletion behind the typed email address and the password worked.

The dialog, in order:

1. Say exactly what disappears, counted from their real data: "This deletes your account and all 47 hikes. This cannot be undone."
2. If they're paying, say so: "Your Pro subscription will be cancelled."
3. **One-owner apps** add a sentence only they need: "You are the only account. Deleting it reopens sign-up, and the next person to sign up becomes the owner." That is what the hook in `references/auth.md` does once the user count is back to zero, so the person has to hear it before, not after.
4. Offer **Download my data** first.
5. Require them to **type their email address** to enable the button. Typing their own address beats typing "DELETE" — it's harder to do by reflex and it restates whose account this is.
6. Require their password.
7. `authClient.deleteUser({ password, callbackURL: "/goodbye" })`. With email, they get a message and clicking the link deletes the account, clears every session, and lands on `/goodbye`. Without email, the account is deleted there and then, and the page sends them to `/goodbye` itself.

**Apps where a team shares the records: a member leaving must not delete the team's history.** `references/database.md` has the rule: a column pointing at a person on a shared table uses `onDelete: "set null"` or `"restrict"`, never `"cascade"`. So on these apps the dialog does not promise "and all your jobs": it says the account goes and the team's records stay, with the person's name no longer attached. Step 1 of `beforeDelete` collects only files that were private to that person, never photos attached to the team's records. With `"restrict"`, the delete fails while rows still point at the person; catch that and say what has to be reassigned first. And the last account holding the admin role cannot delete itself on a team app: refuse in `beforeDelete` with a sentence telling them to make someone else the admin first. Not yet built with this skill: prove it with the Verify item below.

**Download my data** is a route handler, not a server action and not a job. Measured on a real build: written as a server action that returns `new Response(...)`, it compiles and then fails at runtime with "Only plain objects... can be passed", because an action's return value is sent to the browser as data and a `Response` is not data. A route handler returns the file; a plain `<a>` links to it.

`src/app/(dashboard)/settings/export/route.ts`:

```ts
import { requireUserAction } from "@/lib/auth-guards";

export async function GET() {
  let session;
  try {
    session = await requireUserAction();
  } catch {
    return new Response("Not signed in", { status: 401 });
  }

  const data = {
    exportedAt: new Date().toISOString(),
    user: { name: session.user.name, email: session.user.email, createdAt: session.user.createdAt },
    // ...their rows from this app's tables, scoped by session.user.id
  };
  return new Response(JSON.stringify(data, null, 2), {
    headers: {
      "Content-Type": "application/json",
      "Content-Disposition": 'attachment; filename="my-data.json"',
    },
  });
}
```

```tsx
<a href="/settings/export" download>Download my data</a>
```

It uses `requireUserAction()`, not `requireUser()`: a layout does not wrap a route handler, so the dashboard's redirect never runs for this address, and a signed-out request has to get a `401` rather than a redirect to a sign-in page saved as `my-data.json`.

Include the things they created. Never include password hashes, OAuth tokens, or anything belonging to another user. On a shared-team app that means their own profile and the records assigned to or created by them, not the whole team's data.

## The unverified-email banner

**Only in apps where `references/email.md` ran.** Without a sender `emailVerified` is `false` for every account and always will be, so the banner would sit on every page for ever, promising a link nobody sent. An app with no email has no banner and no resend control.

In the dashboard layout, above the content, whenever `session.user.emailVerified` is false:

> **Confirm your email.** We sent a link to you@example.com. [Resend] [Change email]

Do not lock the app. Let people look around, and gate only the things that would embarrass them or the app if done from an unconfirmed address — inviting others, sending outbound email, publishing something public, paying. That list comes from the interview, not from a default.

## Verify

**No account is created to run these.** The first account on the database becomes the admin, and on a one-owner app it takes the only seat, so it has to be the real person's, made in Step 6. What can be checked now is checked now; the rest is listed as deferred and run then. `references/verify.md` has the account rules for each access shape, including which probes do not apply when only one account can exist.

Checked now, signed out:

- Types compile and the build passes with the new pages, guards and route handler in it.
- `src/lib/auth.ts` still has the `role` field with `input: false` and one first-account hook, not two.
- Every `/settings/...` page sends a signed-out visitor to `/sign-in`.
- `GET /settings/export` signed out answers `401`, not a redirect and not a file.
- "Download my data" is a route handler linked with a plain `<a>`; there is no server action that returns a `Response`.
- Every section that exists corresponds to something this app actually has — no empty Billing tab, no Notifications tab without email, no Cookie preferences tab in an app with no banner. Read the nav in the code.
- No-email apps: there is no unverified banner, no verification badge and no resend control anywhere in the code.
- Consent branch: from the footer link, signed out, the dialog the banner uses reopens, and turning a category off actually stops that script rendering on the next request.
- With `.env` values absent, the app still builds and starts, and signed-out pages render.

Needs an account, so not now:

- Deferred to Step 6, after the user has signed up: `/settings` is reachable from the app's navigation, not just by typing the URL, and its padding matches the rest of the dashboard rather than doubling it.
- Deferred to Step 6, after the user has signed up: changing a name saves and the new name shows in the header without a hard refresh.
- Deferred to Step 6, after the user has signed up: email branch only — a brand-new account shows the unverified banner; confirming the email clears it and the badge flips; the resend button disables for 60 seconds and disappears entirely once verified.
- Deferred to Step 6, after the user has signed up: changing a password works, and with "sign out other devices" checked, a second browser's session is actually ended.
- Deferred to Step 6, after the user has signed up: the devices list shows the current session marked as this device, and revoking another session signs it out — confirm in a second browser. Whether the list still loads for an account signed in more than a day ago cannot be seen on the day; `freshAge: 0` is the reason it should, so confirm the setting is in place.
- Deferred to Step 6, after the user has signed up: "Download my data" saves a JSON file holding their own rows and nobody else's.
- Deferred to Step 6, after the user has signed up: the system page opens for them, and the admin guards answer a non-admin with a `404` on the page and a refusal from the actions. On a one-owner app no second account can exist, so the non-admin half is not applicable; `references/verify.md` says how that is reported.
- Deferred to Step 6, after the user has signed up: open sign-up apps only — a second account cannot see the first account's data anywhere in the settings area, and `/settings/system` is not in its navigation. `references/verify.md` creates and removes the probe accounts.
- Deferred to Step 6, after the user has signed up: deletion. On an open sign-up app, delete a probe account: with email the confirmation message arrives and the link removes the account and its data; without email it goes at once behind the typed address and password; signing in with it afterwards fails. Uploads branch: its files are gone from storage. Payments branch: its subscription is cancelled. **On a one-owner app do not run this on the owner's account**: read the dialog for the sentence about sign-up reopening, and leave the delete itself to the person.
- Deferred to Step 6, after the user has signed up: shared-team apps — removing an invited fixture member leaves the team's records in place with the person's name detached, and the last admin cannot delete themselves.
- Deferred to Step 6, after the user has signed up: consent branch — `/settings/cookies` shows the current choice and reopens that same dialog.
- Deferred to Step 6, after the user has signed up: agent access branch — Connected apps lists a real connection with a last-used time, and revoking it makes the next tool call fail rather than only clearing the row.
- Deferred to Step 6, after the user has signed up: with optional `.env` values blank, the signed-in settings pages still render and the affected controls show a friendly "not configured yet" note instead of crashing.
