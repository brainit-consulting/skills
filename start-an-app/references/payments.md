# Payments (Polar or Stripe)

Last verified: 2026-09-21

**Purpose:** Take money — a subscription or a one-off purchase — and know which users have paid.

> **Hard rule: payments go through the Better Auth plugin. Always.** Better Auth ships first-class plugins for both providers (`@polar-sh/better-auth`, `@better-auth/stripe`), and this stack uses them without exception. Do **not** install the standalone provider SDK and wire checkout, customer creation, or webhooks by hand — not "just for a quick one-off payment", not because a provider's own quickstart shows the standalone route. Going around Better Auth means a second source of truth for who the customer is, a hand-rolled webhook route with hand-rolled signature verification, and a customer id that has to be reconciled with the session forever. The plugin does all of it and keeps "who is signed in" and "what have they paid for" as one object.
>
> If a provider's documentation and this file disagree on *how to integrate*, this file wins. If they disagree on a product ID, dashboard path, or API detail, the provider's docs win — update this file afterwards.

**Prerequisite: sign-in must exist.** A payment has to attach to somebody, and the plugin lives inside the auth config, so there is nowhere to put it otherwise. If the user asked for payments but said no to accounts, go back and set up `references/auth.md` first — explain it in one sentence ("we need accounts so the app knows *whose* subscription it is") rather than treating it as a blocker.

Follow exactly one branch: **Polar** (recommended) or **Stripe**.

> Which to recommend, in plain words: Polar is the *merchant of record* — they sell to your customer, so sales tax and VAT in every country are their legal problem, not the user's. That is the single biggest hidden cost of selling software internationally, and it is why Polar is the default here. Stripe leaves the user as the seller: more control, lower fees at scale, more paperwork. Recommend Polar; if the user already has a Stripe account or needs Stripe-specific features, take the Stripe branch without argument.

Everything below is **test/sandbox mode**. Nobody is charged. Say that out loud — users get nervous when a card form appears.

---

## Polar branch (recommended)

### Install

```bash
pnpm add @polar-sh/better-auth @polar-sh/sdk
```

### Set up the Polar account

Walk the user through it, one step at a time:

1. Open https://sandbox.polar.sh and sign up — this is the sandbox, entirely separate from real money.
2. Create an organization (the app's name is fine).
3. **Products → New Product.** Name it after what they're actually selling ("Pro", "Lifetime"), set a price, save. Copy the **product ID**.
4. **Settings → Developers → New Token.** Copy the token — it is shown once.
5. **Settings → Webhooks → Add Endpoint**, URL `https://<their-domain>/api/auth/polar/webhooks`, format **Raw**. Copy the signing secret. (Until the app is deployed there's no public URL for this — see *Testing webhooks locally* below.)

Append to `.env`:

```
POLAR_ACCESS_TOKEN=<from step 4>
POLAR_WEBHOOK_SECRET=<from step 5>
POLAR_SERVER=sandbox
POLAR_PRODUCT_ID=<from step 3>
```

`POLAR_SERVER` becomes `production` on launch day — that one word is the whole go-live switch.

### Configure

Extend `src/lib/auth.ts`. What follows is a **fragment to add to the file `references/auth.md` wrote**, not a replacement for it: the role field, the first-account hook, the email wiring and every plugin already there stay exactly as they are.

```ts
// src/lib/auth.ts — add these imports beside the existing ones
import { polar, checkout, portal, webhooks } from "@polar-sh/better-auth";
import { Polar } from "@polar-sh/sdk";
import { recordPaidState } from "@/lib/billing"; // written below, under "Both branches"

// Add above the betterAuth({ ... }) call
const polarToken = process.env.POLAR_ACCESS_TOKEN;
const polarProductId = process.env.POLAR_PRODUCT_ID;
const polarWebhookSecret = process.env.POLAR_WEBHOOK_SECRET;

// Empty when the keys are missing, so the app starts without them.
const paymentsPlugins =
  polarToken && polarProductId
    ? [
        polar({
          client: new Polar({
            accessToken: polarToken,
            server: process.env.POLAR_SERVER === "production" ? "production" : "sandbox",
          }),
          createCustomerOnSignUp: true,
          use: [
            checkout({
              products: [{ productId: polarProductId, slug: "pro" }],
              successUrl: "/thanks?checkout_id={CHECKOUT_ID}",
              authenticatedUsersOnly: true,
            }),
            portal(),
            ...(polarWebhookSecret
              ? [
                  webhooks({
                    secret: polarWebhookSecret,
                    // Fires on every subscription change — the reliable source of truth.
                    onCustomerStateChanged: async (payload) => {
                      await recordPaidState(payload);
                    },
                    // One-off purchases arrive here.
                    onOrderPaid: async (payload) => {
                      await recordPaidState(payload);
                    },
                  }),
                ]
              : []),
          ],
        }),
      ]
    : [];

export const paymentsConfigured = paymentsPlugins.length > 0;
```

Then, inside the existing `betterAuth({ ... })` call, add one line to the existing `plugins` array:

```ts
  plugins: [
    // ...every plugin already here stays...
    ...paymentsPlugins,
    nextCookies(), // must stay last
  ],
```

Why it is shaped this way: a missing key must degrade, never crash. There is no `!` on any env value, the Polar client is only constructed when its token is present, and `createCustomerOnSignUp` only exists when the plugin does — so with no keys, sign-up still works and nothing calls Polar. `paymentsConfigured` is derived from the same array the auth config uses, so the pricing page cannot disagree with the server about whether payments are on.

With the plugin present, `createCustomerOnSignUp: true` means every new account gets a Polar customer automatically, so there is never a "customer not found" branch to write.

The conditional plugin list is not yet built with this skill: if TypeScript loses the plugin's endpoint types on `auth.api` because of the conditional, keep the condition and cast the array rather than going back to an unconditional plugin, and prove the missing-keys case with the Verify item below.

Client plugin in `src/lib/auth-client.ts` — again a fragment. **Add** `polarClient()` to the existing `plugins` list and add the two new names to the existing export line; keep every plugin and export already there:

```ts
import { polarClient } from "@polar-sh/better-auth/client";

// inside the existing createAuthClient({ plugins: [ ... ] })
//   polarClient(),

// add to the existing named exports
//   checkout, customer
```

The client plugin is safe to include unconditionally — it only adds method names. What must be conditional is the button: see *Both branches* below.

Adding the plugin can add columns and tables, so regenerate the schema and migrate:

```bash
pnpm dlx auth@latest generate --config src/lib/auth.ts --output src/lib/db/auth-schema.ts -y
pnpm db:generate
pnpm db:migrate
```

Wire up two buttons:

- **Upgrade** → `authClient.checkout({ slug: "pro" })`
- **Manage billing** → `authClient.customer.portal()` (Polar hosts the cancel/invoice/payment-method screens, so there is nothing to build)

### Testing webhooks locally

Polar can only call a public URL, so on `localhost` webhooks stay silent — checkout itself still works end to end. Either deploy first and test webhooks there, or expose the dev server with a tunnel (`pnpm dlx untun@latest tunnel http://localhost:3000`, or ngrok) and use that URL as the webhook endpoint. The port in that command has to be the one the app is actually running on. Say which one you did.

Be plain about what this means: without a tunnel, no webhook reaches `localhost`, so the app's paid state never changes locally however many test payments succeed. Checkout can be proven on a laptop; the paid state can only be proven through a tunnel or on the deployed app.

Sandbox test card: `4242 4242 4242 4242`, any future expiry, any CVC.

---

## Stripe branch

### Install

```bash
pnpm add @better-auth/stripe stripe
```

### Set up the Stripe account

1. Open https://dashboard.stripe.com and make sure the **Test mode** toggle is on.
2. **Product catalogue → Add product.** Name and price it, save, copy the **price ID** (`price_...`, not the product ID).
3. **Developers → API keys.** Copy the **secret key** (`sk_test_...`).
4. Get a webhook secret by running the Stripe CLI (below) — it prints one.

Append to `.env`:

```
STRIPE_SECRET_KEY=sk_test_<from step 3>
STRIPE_WEBHOOK_SECRET=whsec_<from the CLI>
STRIPE_PRICE_ID=price_<from step 2>
```

### Configure

Extend `src/lib/auth.ts`. This is a **fragment to add to the file `references/auth.md` wrote**, not a replacement for it: the role field, the first-account hook, the email wiring and every plugin already there stay exactly as they are.

```ts
// src/lib/auth.ts — add these imports beside the existing ones
import { stripe } from "@better-auth/stripe";
import Stripe from "stripe";
import { recordPaidState } from "@/lib/billing"; // written below, under "Both branches"

// Add above the betterAuth({ ... }) call
const stripeKey = process.env.STRIPE_SECRET_KEY;
const stripeWebhookSecret = process.env.STRIPE_WEBHOOK_SECRET;
const stripePriceId = process.env.STRIPE_PRICE_ID;

// Empty when the keys are missing, so the app starts without them.
const paymentsPlugins =
  stripeKey && stripeWebhookSecret && stripePriceId
    ? [
        stripe({
          stripeClient: new Stripe(stripeKey),
          stripeWebhookSecret,
          createCustomerOnSignUp: true,
          subscription: {
            enabled: true,
            plans: [{ name: "pro", priceId: stripePriceId }],
            // The plugin's subscription hooks are where the app's own paid state is written.
            // Confirm their current names in the check-what's-current step; each one calls
            // recordPaidState(...) with the user id the subscription refers to.
          },
        }),
      ]
    : [];

export const paymentsConfigured = paymentsPlugins.length > 0;
```

Then, inside the existing `betterAuth({ ... })` call, add one line to the existing `plugins` array:

```ts
  plugins: [
    // ...every plugin already here stays...
    ...paymentsPlugins,
    nextCookies(), // must stay last
  ],
```

Why it is shaped this way: a missing key must degrade, never crash. There is no `!` on any env value, the Stripe client is only constructed when its key is present, and `createCustomerOnSignUp` only exists when the plugin does — so with no keys, sign-up still works and nothing calls Stripe. `paymentsConfigured` is derived from the same array the auth config uses, so the pricing page cannot disagree with the server about whether payments are on.

The conditional plugin list is not yet built with this skill: if TypeScript loses the plugin's endpoint types on `auth.api` because of the conditional, keep the condition and cast the array rather than going back to an unconditional plugin, and prove the missing-keys case with the Verify item below.

If TypeScript complains about a missing or mismatched `apiVersion`, pass the exact version string the installed `stripe` package's types name — don't guess one.

Client plugin in `src/lib/auth-client.ts` — again a fragment. **Add** `stripeClient(...)` to the existing `plugins` list and add the new name to the existing export line; keep every plugin and export already there:

```ts
import { stripeClient } from "@better-auth/stripe/client";

// inside the existing createAuthClient({ plugins: [ ... ] })
//   stripeClient({ subscription: true }),

// add to the existing named exports
//   subscription
```

Regenerate the schema — this plugin definitely adds a `subscription` table and a customer id on `user`:

```bash
pnpm dlx auth@latest generate --config src/lib/auth.ts --output src/lib/db/auth-schema.ts -y
pnpm db:generate
pnpm db:migrate
```

Better Auth's own docs mention `npx auth migrate` — that is for its built-in Kysely adapter. This stack is Drizzle, so migrations always go through `db:generate` + `db:migrate`.

Wire up the upgrade button:

```ts
await authClient.subscription.upgrade({
  plan: "pro",
  successUrl: "/thanks",
  cancelUrl: "/pricing",
});
```

The plugin serves its own webhook at `/api/auth/stripe/webhook` — do not write a webhook route by hand.

### Testing webhooks locally

Install the Stripe CLI, then in a second terminal:

```bash
stripe login
stripe listen --forward-to localhost:3000/api/auth/stripe/webhook
```

It prints a `whsec_...` — that is `STRIPE_WEBHOOK_SECRET` for local development. The deployed app needs a *different* secret, created under **Developers → Webhooks** with the real URL. The port in the `--forward-to` address has to be the one the app is actually running on.

Without `stripe listen` running, no webhook reaches `localhost`, so the app's paid state never changes locally however many test payments succeed. Checkout can be proven without it; the paid state cannot.

Test card: `4242 4242 4242 4242`, any future expiry, any CVC.

---

## Both branches — gating the app

Paying for something has to change something. The app keeps its own record of who has paid, the webhook handlers write it, and the server reads it.

Not yet built with this skill: the paid-state design below was desk-checked, never run. Confirm the webhook payload fields and the plugin's hook names in the check-what's-current step, and prove it with the Verify items below.

### The paid state

One app-owned table (a column on an existing per-user table is fine for a single plan). Postgres branch shown; on SQLite use `integer` timestamps, per `references/database.md`. It goes in `src/lib/db/schema.ts`, which is safe here because `auth-schema.ts` already exists — after `pnpm db:generate`, read the SQL and check the `REFERENCES` line carries `ON DELETE cascade`.

```ts
export const billing = pgTable("billing", {
  userId: text("user_id")
    .primaryKey()
    .references(() => user.id, { onDelete: "cascade" }),
  plan: text("plan").notNull().default("free"), // free | pro — use the app's real plan names
  status: text("status"), // the provider's own word: active, canceled, past_due...
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});
```

`src/lib/billing.ts`:

```ts
import "server-only";
import { eq } from "drizzle-orm";
import { db } from "@/lib/db";
import { billing } from "@/lib/db/schema";

export type Plan = "free" | "pro";

export async function getPlan(userId: string): Promise<Plan> {
  const [row] = await db.select().from(billing).where(eq(billing.userId, userId));
  return row?.plan === "pro" ? "pro" : "free";
}

export async function setPlan(userId: string, plan: Plan, status?: string) {
  await db
    .insert(billing)
    .values({ userId, plan, status })
    .onConflictDoUpdate({
      target: billing.userId,
      set: { plan, status, updatedAt: new Date() },
    });
}

// Called by the webhook handlers in src/lib/auth.ts.
export async function recordPaidState(payload: unknown) {
  // 1. Find the Better Auth user id. Polar: the customer's external id, which the plugin
  //    sets to the user id at sign-up. Stripe: the subscription's reference id.
  // 2. Decide paid or not from the payload (an active subscription, or a paid order
  //    for the one-off product).
  // 3. await setPlan(userId, paid ? "pro" : "free", status);
  // A payload with no user id is logged and ignored, never thrown — a throw makes the
  // provider retry the same webhook for days.
}
```

No row means free, so nothing has to be written at sign-up and an app whose payment keys are missing treats everyone as free.

- Read the plan on the server with `getPlan(session.user.id)`, never from a client-side flag a user can flip in devtools.
- Show a real upgrade prompt on the gated page, not a blank screen.
- Keep the free tier usable — the app should still make sense to someone who never pays.

### Limits live in the shared function, not the page

A plan limit — 5 plants on the free plan — is enforced inside the shared `src/lib/<domain>` function that does the write, because the page is not the only caller. If `references/mcp.md` runs, an agent's tool call goes through the same function, and a limit that only the page checks is a limit an agent walks past.

```ts
// src/lib/plants.ts
import { getPlan } from "@/lib/billing";

export class PlanLimitError extends Error {}

export async function createPlant(userId: string, input: NewPlant) {
  if ((await getPlan(userId)) === "free") {
    const count = await countPlants(userId);
    if (count >= 5) throw new PlanLimitError("The free plan holds 5 plants. Upgrade to add more.");
  }
  // ...insert...
}
```

The server action catches `PlanLimitError` and returns the message with a link to the pricing page; the agent tool returns the same sentence. The page may also hide the Add button at the limit — as a courtesy, not as the check.

### The pricing page, and the page people come back to

- **Pricing page.** A server component reads `paymentsConfigured` from `@/lib/auth`. When it is false the page still shows the plans, and where the Upgrade button would be it says "Payments are not set up yet." — it never renders a button that would call an endpoint that does not exist.
- **`/thanks`.** Both branches point `successUrl` at it, so create it: `src/app/(dashboard)/thanks/page.tsx`, behind `requireUser()` from `src/lib/auth-guards.ts`. It reads `getPlan()` and says one of two true things — "You're on Pro." or "Payment received. Your plan updates within a minute; refresh this page." The webhook usually lands after the redirect, so the page must not claim the upgrade before the server knows about it.

Pages are written by `references/pages.md`, which runs after this file. If it has not run yet, write the auth fragment, the table and `src/lib/billing.ts` now, and build the pricing page, `/thanks`, the buttons and the gate when the pages exist.

If account deletion is built, `references/settings.md` cancels the subscription in its `beforeDelete` hook, before the rows cascade — a deleted account must not keep being charged.

## Going to production

At hand-off, tell the user the go-live steps in order: switch the provider out of sandbox/test mode, create the product again in live mode (test and live catalogues are separate — this surprises everyone), swap the keys in the host's environment variables, and register the webhook against the real domain.

## Verify

Do not create an account to run these. The first account belongs to the real person and is made in Step 6; on a one-owner app a test account would take the owner's seat.

Checked now, with no account:

- `pnpm exec tsc --noEmit` passes, and `grep -rn "process\.env\.[A-Z_]*!" src` finds nothing.
- The schema was regenerated and migrated, and the `billing` table's generated SQL carries `ON DELETE cascade`.
- With the payment env vars blank or missing, `pnpm build` passes, the app starts, the sign-in and sign-up pages still load, and — once the pricing page exists — it says payments are not set up yet instead of showing a dead button.
- Deferred until `references/pages.md` has run: `/thanks` sends a signed-out visitor to sign-in rather than answering 500.

Needs an account, so not run here:

- Deferred to Step 6, after the user has signed up: clicking Upgrade reaches the provider's hosted checkout with the right product and price.
- Deferred to Step 6, after the user has signed up: paying with the test card returns to `/thanks` inside the app.
- Deferred to Step 6, after the user has signed up, and only with a tunnel (or `stripe listen`) running or on the deployed app: the webhook arrives, the `billing` row changes, and the gated feature unlocks. If there was no tunnel, say in the hand-off that the paid state has not been proven yet.
- Deferred to Step 6, after the user has signed up: the plan limit refuses the write from the shared function — at the limit, the server action returns the upgrade message, and so does the agent tool if agent access was built.
- Deferred to Step 6, after the user has signed up: the billing portal opens for a paying user.
