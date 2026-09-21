# Transactional email (Resend)

Last verified: 2026-09-21

**Purpose:** Send the emails the app owes people — verify your address, reset your password, here's your receipt, someone invited you. Real delivery in production, and a readable log locally without sending anything to anyone.

> **Hard rule: nothing sends until it's logged.** Every message goes through `src/lib/email.ts`, which writes an `email_log` row first and sends second. That row is what `references/ops.md` renders, what tells the user *why* an email never arrived, and what survives Resend's 30-day retention. Never call `resend.emails.send()` directly from a page, action, or route.
>
> If Resend's documentation and this file disagree on *how to integrate*, this file wins. If they disagree on a DNS record, dashboard path, or API detail, Resend's docs win — update this file afterwards.

**Sign-in is not required.** A contact form or a notification-only app can send email with no accounts at all — see *A contact form* below. But if `references/auth.md` ran, this file also wires up email verification and password reset — Better Auth cannot do either without a sender, so those features simply do not exist until this step runs.

## Install

```bash
pnpm add resend react-email
pnpm add -D @react-email/ui
```

> React Email merged everything into the single `react-email` package. Import components, `render`, and `toPlainText` from `"react-email"` — **not** `@react-email/components`, and not the per-component `@react-email/button` packages. Those still install and still resolve, which is exactly why the mistake is easy to make and hard to spot. `@react-email/ui` is the local preview server only, hence `-D`.

Append to `.env`:

```
RESEND_API_KEY=
EMAIL_FROM="TrailLog <hello@send.example.com>"
```

Leave `RESEND_API_KEY` **empty for now**. An empty key is the local development mode — see *How sending is switched* below — so the app is fully working before the user has an account or a domain.

Add the preview server to `package.json`:

```json
"email:dev": "email dev --dir src/emails --port 3333"
```

Port 3333, not 3001: when 3000 is busy Next takes 3001 for the app itself, and the preview server would then be fighting the app for it.

## Set up the Resend account

Do this when the user is ready to send to somebody other than themselves. Say that plainly first, because it decides whether they need a domain today:

> Resend gives you a test address, `onboarding@resend.dev`, that works instantly — but it will **only** deliver to the email address you signed up with. The moment you want to email anyone else, you need to prove you own a domain. That's a DNS change, about ten minutes, and it's free.

Then walk them through it, one step at a time:

1. Open https://resend.com and sign up.
2. **Domains → Add Domain.** Enter a **subdomain**, not the bare domain — `send.theirdomain.com` or `notifications.theirdomain.com`. Explain why in one line: if something ever goes wrong with deliverability, it damages the subdomain's reputation and leaves their main domain, and their normal email, untouched. Pick the region closest to them.
3. Resend shows three DNS records. Add all three at their domain registrar, exactly as shown:
   - an **MX** record on `send` (priority 10) — this is how bounces come back
   - a **TXT** record on `send` — SPF, which says Resend is allowed to send as them
   - a **TXT** record on `resend._domainkey` — DKIM, which signs each message
   Paste the names exactly as Resend gives them (`send`, not `send.theirdomain.com`) — most registrars add the domain themselves. On Cloudflare, set these to **DNS only**, with the orange proxy cloud off.
4. Click **Verify**. It usually completes in under fifteen minutes. If it stalls, the records are almost always right but not yet visible — wait and press it again rather than editing them.
5. Once verified, add one more **TXT** record on `_dmarc` with the value `v=DMARC1; p=none;`. It isn't needed for verification, but Gmail and Outlook increasingly expect it.
6. **API Keys → Create API Key.** Set permission to **Sending access** and restrict it to the domain just added. Copy it — it is shown once.

Fill in `.env`:

```
RESEND_API_KEY=re_<from step 6>
EMAIL_FROM="<App name> <hello@send.theirdomain.com>"
```

Any address at the verified domain works — there is no separate "sender" to create.

**Free tier, so there are no surprises:** 100 emails a day, 3,000 a month, one domain. That is generous for a new app and worth saying out loud.

## Configure

### The send helper

`src/lib/email.ts` is the only file that talks to Resend. It switches on **whether the key is present**, not on a mode flag or `NODE_ENV` — same rule as `references/storage.md`, so nothing has to be remembered at deploy time.

```ts
import type { ReactElement } from "react";
import { Resend } from "resend";
import { render, toPlainText } from "react-email";
import { eq } from "drizzle-orm";
import { db } from "@/lib/db";
import { emailLog } from "@/lib/db/schema";

const apiKey = process.env.RESEND_API_KEY;
export const emailConfigured = Boolean(apiKey);

const resend = apiKey ? new Resend(apiKey) : null;
// `||`, not `??`: an EMAIL_FROM that is present but blank must fall back too.
const from = process.env.EMAIL_FROM || "TrailLog <onboarding@resend.dev>"; // the app's real name, never a placeholder

type SendArgs = {
  to: string;
  subject: string;
  react: ReactElement;
  template: string;
  replyTo?: string; // the contact form sets this to the visitor
};

export async function sendEmail({ to, subject, react, template, replyTo }: SendArgs) {
  // Render here, once. The stored HTML is what the system page shows, and the
  // same HTML is what gets sent, so the log can never disagree with the message.
  const html = await render(react);
  const text = toPlainText(html);

  const [row] = await db
    .insert(emailLog)
    .values({ to, subject, template, status: "pending", html, text })
    .returning();

  if (!resend) {
    console.info(
      `\n[email] ${subject}\n[email] to: ${to}\n[email] not sent — RESEND_API_KEY is empty. Logged as ${row.id}.\n`,
    );
    await db
      .update(emailLog)
      .set({ status: "logged" })
      .where(eq(emailLog.id, row.id));
    return { id: row.id };
  }

  const { data, error } = await resend.emails.send(
    { from, to, subject, html, text, ...(replyTo ? { replyTo } : {}) },
    { idempotencyKey: `${template}/${row.id}` },
  );

  await db
    .update(emailLog)
    .set(
      error
        ? { status: "failed", error: error.message }
        : { status: "sent", providerId: data.id },
    )
    .where(eq(emailLog.id, row.id));

  return { id: row.id };
}
```

**No `import "server-only"` in this file.** `src/lib/auth.ts` imports it, and the Better Auth CLI and any `tsx` seed script load `auth.ts` outside Next.js, where that import fails. The same goes for the templates.

Passing `html` and `text` instead of `react` also removes a hidden dependency: Resend renders a `react` field through an optional peer package that may not resolve under pnpm.

Three things in there are not optional, and all three are easy to get wrong:

- **`resend.emails.send()` does not throw on an API error.** It returns `{ data, error }`. A `try`/`catch` around it catches network failures only and silently swallows every rejected send — wrong domain, unverified sender, over quota. Check `error`.
- **`idempotencyKey` is the second argument**, not a field in the payload. Put it in the payload and it is ignored, and a retried request sends twice.
- **Parameters are camelCase in the Node SDK** — `replyTo`, `scheduledAt`. The REST API uses snake_case; the SDK does not, and it fails silently rather than erroring.

Anything that sends must run on the Node runtime. In a route handler, say so explicitly:

```ts
export const runtime = "nodejs";
```

### The log table

Add to `src/lib/db/schema.ts` — Postgres branch shown; on SQLite use a `text` id with `$defaultFn(() => crypto.randomUUID())` and `integer` timestamps, exactly as `references/database.md` describes.

```ts
export const emailLog = pgTable("email_log", {
  id: uuid("id").primaryKey().defaultRandom(),
  to: text("to").notNull(),
  subject: text("subject").notNull(),
  template: text("template").notNull(),
  status: text("status").notNull(), // pending | logged | sent | delivered | bounced | complained | failed
  html: text("html"),
  text: text("text"),
  providerId: text("provider_id"),
  error: text("error"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});
```

```bash
pnpm db:generate
pnpm db:migrate
```

### Templates

Templates live in `src/emails/`. One file per message, plain and short — an email is not a landing page.

```tsx
// src/emails/verify-email.tsx
import { Body, Button, Container, Head, Html, Preview, Text } from "react-email";

export default function VerifyEmail({ url, name }: { url: string; name?: string }) {
  return (
    <Html>
      <Head />
      <Preview>Confirm your email address</Preview>
      <Body style={{ fontFamily: "sans-serif", backgroundColor: "#fff" }}>
        <Container style={{ maxWidth: 480, padding: "32px 0" }}>
          <Text>Hi{name ? ` ${name}` : ""},</Text>
          <Text>Confirm your email address to finish setting up your account.</Text>
          <Button href={url} style={{ background: "#000", color: "#fff", padding: "12px 20px", borderRadius: 6 }}>
            Confirm email
          </Button>
          <Text style={{ color: "#666", fontSize: 13 }}>
            If you didn&apos;t create an account, you can ignore this email.
          </Text>
        </Container>
      </Body>
    </Html>
  );
}
```

`pnpm email:dev` opens a preview at http://localhost:3333 (its default port is the same as the app's, hence the flag) with a mobile toggle, the plain-text version, and a spam check. It's the fastest way to iterate, and it sends nothing.

Write the copy for *their* app, in their voice. "Confirm your email to start logging hikes" beats "Verify your account".

### Wiring Better Auth (only if sign-in was chosen)

Extend `src/lib/auth.ts`. This adds to the existing config; it replaces nothing:

```ts
import { sendEmail } from "@/lib/email";
import VerifyEmail from "@/emails/verify-email";
import ResetPassword from "@/emails/reset-password";

export const auth = betterAuth({
  // ...existing config
  emailAndPassword: {
    enabled: true,
    sendResetPassword: ({ user, url }) => {
      void sendEmail({
        to: user.email,
        subject: "Reset your password",
        react: ResetPassword({ url, name: user.name }),
        template: "reset-password",
      });
    },
  },
  emailVerification: {
    sendOnSignUp: true,
    autoSignInAfterVerification: true,
    sendVerificationEmail: ({ user, url }) => {
      void sendEmail({
        to: user.email,
        subject: "Confirm your email address",
        react: VerifyEmail({ url, name: user.name }),
        template: "verify-email",
      });
    },
  },
});
```

> **`void`, not `await`.** This is Better Auth's own instruction and it is a security rule, not a style preference: awaiting the send makes the response measurably slower when the account exists than when it doesn't, which tells an attacker who has an account. Fire it and return.

**Leave `requireEmailVerification` off.** Blocking sign-in on verification turns one mistyped address into a support request the user cannot answer. `references/settings.md` builds the banner-and-resend pattern instead, which nags without locking anyone out.

Adding these hooks does not change the schema, so there is no Better Auth CLI regeneration here.

### A contact form (no accounts, or a public site with one author)

Not yet built with this skill: confirm the form and action APIs in the check-what's-current step and prove it with the contact-form items in Verify below.

A visitor writes, the owner gets an email, and pressing Reply in their mail program answers the visitor. Four parts.

**Where it goes.** Append to `.env`:

```
CONTACT_TO_EMAIL=
```

The owner's own address. It is never shown on the page and never taken from the form. With it blank the form says it isn't set up yet; it does not crash and it does not guess.

**The template**, `src/emails/contact-message.tsx`, is the plainest one in the app: who wrote, their address, and the message with its line breaks kept. No button, no branding to speak of. It is mail to the owner, not to a customer.

**The action**, `src/app/contact/actions.ts`:

```ts
"use server";

import { and, count, eq, gte } from "drizzle-orm";
import { db } from "@/lib/db";
import { emailLog } from "@/lib/db/schema";
import { sendEmail } from "@/lib/email";
import ContactMessage from "@/emails/contact-message";

const HOURLY_LIMIT = 20; // across all visitors

export type ContactState = { ok: boolean; message: string } | null;

const THANKS = { ok: true, message: "Thanks. Your message has been sent." };

export async function sendContactMessage(_prev: ContactState, form: FormData): Promise<ContactState> {
  const field = (key: string) => String(form.get(key) ?? "").trim();

  // Honeypot: a field people never see and bots fill in. Answer as if it
  // worked, so the bot learns nothing, and write nothing.
  if (field("website")) return THANKS;

  const name = field("name").replace(/[\r\n]+/g, " ").slice(0, 200);
  const email = field("email").slice(0, 320);
  const message = field("message").slice(0, 5000);
  if (!name || !message || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
    return { ok: false, message: "Add your name, a working email address and a message." };
  }

  const to = process.env.CONTACT_TO_EMAIL;
  if (!to) return { ok: false, message: "The contact form isn't set up yet." };

  // Counted in the database, so it holds across serverless instances.
  const since = new Date(Date.now() - 60 * 60 * 1000);
  const [{ sent }] = await db
    .select({ sent: count() })
    .from(emailLog)
    .where(and(eq(emailLog.template, "contact-message"), gte(emailLog.createdAt, since)));
  if (sent >= HOURLY_LIMIT) {
    return { ok: false, message: "Too many messages just now. Please try again in an hour." };
  }

  await sendEmail({
    to,
    replyTo: email,
    subject: `Message from ${name}`,
    react: ContactMessage({ name, email, message }),
    template: "contact-message",
  });
  return THANKS;
}
```

**The form** is a client component using `useActionState`, with labelled name, email and message fields, plus the honeypot: an input named `website`, with `tabIndex={-1}`, `autoComplete="off"` and `aria-hidden`, moved off screen with CSS. Not `type="hidden"`, which bots skip, and not `display: none` on the input itself, which some of them check.

Why both guards: this is a public POST that anyone on the internet can call. Every accepted call writes a row and spends one of the day's sends. The honeypot stops the dumb bots for free; the hourly cap is what keeps a determined one from using up the quota and filling the table. The cap is across all visitors on purpose. A per-visitor limit needs the visitor's IP address stored, which is more personal data than a contact form should hold.

**`from` stays the app's own verified address.** Putting the visitor's address in `from` fails SPF and lands in spam; `replyTo` is what makes Reply go to them.

**The message is stored once, in `email_log`**, as the rendered email. There is no separate messages table in a first version. That row holds a stranger's name, address and words, which `references/legal.md` has to disclose. With `RESEND_API_KEY` empty the message is still logged, so the form works locally; say at hand-off that until the key is set, messages reach the log and not the inbox.

### Delivery status (optional, worth it)

Resend can tell the app what happened after the send. Add `src/app/api/webhooks/resend/route.ts`:

```ts
import { NextRequest, NextResponse } from "next/server";
import { Resend } from "resend";
import { db } from "@/lib/db";
import { emailLog } from "@/lib/db/schema";
import { eq } from "drizzle-orm";

export const runtime = "nodejs";

export async function POST(req: NextRequest) {
  // Built here, and only when both values are present. A client made at module
  // scope with an empty key throws while the route is being loaded.
  const apiKey = process.env.RESEND_API_KEY;
  const webhookSecret = process.env.RESEND_WEBHOOK_SECRET;
  if (!apiKey || !webhookSecret) {
    return new NextResponse("Email webhooks are not set up", { status: 503 });
  }

  const id = req.headers.get("svix-id");
  const timestamp = req.headers.get("svix-timestamp");
  const signature = req.headers.get("svix-signature");
  if (!id || !timestamp || !signature) {
    return new NextResponse("Missing signature headers", { status: 400 });
  }

  const payload = await req.text(); // raw body — parsing it breaks the signature
  const resend = new Resend(apiKey);

  let event;
  try {
    event = resend.webhooks.verify({
      payload,
      headers: { id, timestamp, signature },
      webhookSecret,
    });
  } catch {
    return new NextResponse("Invalid signature", { status: 400 });
  }

  const status = event.type.replace("email.", "");
  await db
    .update(emailLog)
    .set({ status, updatedAt: new Date() })
    .where(eq(emailLog.providerId, event.data.email_id));

  return new NextResponse(null, { status: 200 });
}
```

No non-null assertions on env values or headers anywhere in this route: `references/verify.md` greps for them, and each one is a crash waiting for a blank variable.

Register it under **Webhooks → Add Endpoint** at `https://<their-domain>/api/webhooks/resend`, and put the signing secret in `.env` as `RESEND_WEBHOOK_SECRET`. Signature verification needs the **raw** body — reading it as JSON and re-serialising changes the bytes and every request fails.

Resend keeps 30 days of history. `email_log` is the app's own record and keeps whatever the user wants, which is the point of writing it first.

## Testing locally

With `RESEND_API_KEY` empty, nothing leaves the machine: the message is logged to the terminal with the link clickable, and a row lands in `email_log`. That is the normal way to develop, and it means signup and password reset work end to end on day one.

To test real delivery, set the key and send to Resend's simulation addresses — these are real, documented, and safe:

| Address | What it simulates |
| --- | --- |
| `delivered@resend.dev` | a successful delivery |
| `bounced@resend.dev` | a hard bounce |
| `complained@resend.dev` | the recipient marking it as spam |
| `suppressed@resend.dev` | a previously-bounced address |

The first three accept a label, so `delivered+alice@resend.dev` and `delivered+bob@resend.dev` are distinct test users. `suppressed@` does not.

Do **not** test with `@example.com` or `@test.com` — Resend rejects them with a 422 rather than letting them damage the account's bounce rate.

## Going to production

Nothing in the code changes. Set `RESEND_API_KEY` and `EMAIL_FROM` (plus `CONTACT_TO_EMAIL` where there is a contact form, and `RESEND_WEBHOOK_SECRET` where the webhook was added) in the host's environment variables and the same code sends for real. Tell the user that at hand-off, along with the two limits that actually bite: 100 emails a day on the free plan, and 10 API requests per second.

If they ever add marketing email — a newsletter, product updates — send it from a **different** subdomain than this one. Campaign complaints must never be able to affect whether password resets arrive.

## Verify

**No account is created in this step.** The first account belongs to the real person and is made in Step 6; on a one-owner app a test sign-up here would take the owner's seat. So nothing below signs anyone up. To put a row in the log now, call `sendEmail()` from a throwaway `tsx` script with one of the templates (plain `node` cannot load the TypeScript imports), then delete the script.

- `pnpm email:dev` opens the preview on port 3333 and each template renders, on desktop and mobile widths.
- With `RESEND_API_KEY` empty: the script's send prints to the terminal and a row appears in `email_log` with status `logged`.
- With a real key: sending to `delivered@resend.dev` returns success and the row reaches status `sent`, visible in `pnpm db:studio`.
- Sign-in branch. Deferred to Step 6, after the user has signed up: their sign-up produced a verification email, and the link in it (from the terminal when the key is empty) marks the account verified; "forgot password" sends a reset link that actually resets the password.
- Contact-form check, for a no-accounts app and equally for a one-owner public site with a contact form (there it runs as well as the sign-in item above). On a no-accounts app there are no auth emails, so the item above does not apply. Submit the contact form (or trigger the app's one notification) instead: a row lands in `email_log` with template `contact-message`, and with a real key the owner's copy arrives with Reply going to the visitor's address.
- Contact form: a POST with the `website` field filled returns the thanks message and writes **no** row. With `CONTACT_TO_EMAIL` blank the form says it isn't set up and the page still renders. Lower `HOURLY_LIMIT` to 2 for a minute, submit three times, see the third refused, and put it back.
- Webhook route, where added: a POST with no signature headers answers `400`, and with `RESEND_WEBHOOK_SECRET` blank it answers `503`. Neither is a `500`.
- `grep -rn 'process\.env\.[A-Z_]*!' src` finds nothing in the email files.
- Every email's wording is about *their* app — no "My App", no placeholder addresses.
- With `.env` values absent, the app still starts and email degrades to the log instead of crashing.
