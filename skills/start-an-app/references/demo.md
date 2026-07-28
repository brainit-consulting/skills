# Demo mode

Last verified: 2026-07-28

**Purpose:** Let a stranger try the app without being handed the owner's data. A shared account, believable sample content, and a badge saying plainly what this is. **Loaded on request only** — when the user wants to *show* the app to someone (a client, a prospect, a colleague, a visitor to the landing page) rather than use it.

Not for a personal tool. If the app is one person's hiking journal, the demo is a screenshot.

## Set the flag before the build, not after

The flag below is a `NEXT_PUBLIC_` variable, which Next.js inlines at **build** time. On a deployment it has to exist in the host's environment before the build runs, or the app is built with demo mode off and redeploying the same artefact will not turn it on. If this is going online, set it alongside the other variables in `references/deploy.md` Step 1.

## The flag

`src/lib/demo.ts`:

```ts
export const DEMO_MODE = process.env.NEXT_PUBLIC_DEMO_MODE === "true";

export const DEMO_EMAIL = "demo@<app>.app";
export const DEMO_PASSWORD = "<something typeable>";
export const DEMO_NAME = "Demo <role>";
```

Three things this shape gets right:

- **One flag, checked everywhere, defaulting to off.** A real deployment for a real business leaves `NEXT_PUBLIC_DEMO_MODE` unset and the app is an ordinary tool again, with no demo scaffolding to strip out later. Never write the demo UI in unconditionally.
- **The credentials are in the source on purpose.** They are printed on the landing page for anyone to read; hiding them in an environment variable would be ceremony, not security. Say this out loud rather than letting it look like an oversight.
- **The password is meant to be typed by a human**, from a phone, possibly read out on a call. Not a generated string. Eight characters minimum — Better Auth rejects shorter.

**Demo mode must never point at real data.** It is a shared account with published credentials; anyone who reads the page can sign in. If the app already holds real customer records, the demo needs its own database, not just its own flag.

## What the flag turns on

**A badge, on the front door and in the app's header.** Both, not one: someone sent a deep link lands inside the app without ever seeing the landing page, and a person who doesn't know they're in a demo will report the sample data as a bug.

**A card on the front door** with the email and password as plain, selectable text — labelled, one per line, in a monospace face so `l` and `1` are distinguishable. This is the thing being asked for; make it the most legible object on the page.

**A button that signs the visitor straight in.** The credentials are printed directly above it for anyone who prefers the normal form, so this is a shortcut rather than the only way in:

```tsx
// shape, not gospel
const { error } = await signIn.email({
  email: DEMO_EMAIL,
  password: DEMO_PASSWORD,
});
if (!error) { router.push("/dashboard"); router.refresh(); }
```

**One honest sentence, in small text, under the card.** Whatever is actually true: that everyone trying it works from the same data, that anything they add is there for the next person, that the sample records are made up. One line, and it is the difference between a visitor who understands what they're looking at and one who thinks they've found someone's real customer list.

**Watch the primary action count.** The demo button becomes the page's one primary action, so whatever was primary before — usually "Sign in" — steps down to a secondary style. Two competing filled buttons is how a considered page starts looking generated.

Consider whether public sign-up should still be open. A demo where strangers can create their own accounts on the owner's database is usually not what was meant; gating `/sign-up` behind the flag is a two-line change. Raise it, don't decide it.

## The seed script

An empty demo is worse than no demo — it shows the app's empty states, which is exactly the screen the user does *not* want to lead with.

`scripts/seed-demo.ts`, run with `pnpm demo:seed` (`pnpm add -D tsx` if it isn't already there):

```json
"demo:seed": "tsx scripts/seed-demo.ts"
```

Two traps, both of which fail in ways that don't point at the cause:

**Top-level `await` fails.** Without `"type": "module"` in `package.json`, `tsx` compiles to CommonJS and every top-level `await` in the file is a build error — `Top-level await is currently not supported with the "cjs" output format`, reported against your line numbers as though the syntax were wrong. Wrap the body in `async function main()` and call it at the bottom.

**`dotenv` has to run before the database module is imported.** `src/lib/db/index.ts` builds its pool at import time, and ES imports are hoisted — so `config()` written after the import statements executes *after* the pool was constructed with an undefined connection string. Put the loading in its own file and import it first, because import order is preserved:

```ts
// scripts/load-env.ts — imported for its side effect, and always first
import { config } from "dotenv";

config({ path: [".env.local", ".env"] });

if (!process.env.POSTGRES_URL) {
  throw new Error("POSTGRES_URL is not set — check .env.");
}
```

```ts
// scripts/seed-demo.ts
import "./load-env";

import { db } from "../src/lib/db";
import { auth } from "../src/lib/auth";
// ...
```

### What the script does

**Create the account through Better Auth, never by inserting a row.** `auth.api.signUpEmail({ body: { email, password, name } })` — password hashing is Better Auth's format, and a hand-written row produces an account that exists and cannot sign in. Check whether the account is already there so a re-run doesn't try twice.

**Make it re-runnable, and say so at hand-off.** Visitors edit and delete things; a demo drifts. The script should clear the sample records and lay them down again — deleting the top of the ownership chain and letting `onDelete: "cascade"` do the rest is usually one statement. Leave the demo *account* alone.

**Build the dates from the day it runs**, not from fixed timestamps. Anything with a calendar in it has a "today", and a demo whose newest entry is three weeks old reads as abandoned. Offsets from midnight on the current day: something yesterday, several today, a couple in the days after.

**Write sample data someone could believe.** This is where a demo is won or lost:

- Real-shaped names, varied in length and origin. Not `Test User`, not `John Smith` five times.
- **No round numbers.** `$34.50` and `$58.00`, never `$50.00` across the board.
- Phone numbers from a reserved-for-fiction range — `555-01xx` in the US, `07700 900xxx` in the UK — so nobody's actual phone rings.
- The awkward records too: the one marked done, the one with a note attached, the customer with two of something. A list where every row is identical demonstrates a table, not an app.
- Enough rows to show grouping and scrolling, few enough to read. Eight to a dozen is usually right.
- The same rule `references/pages.md` applies to the landing page applies here: nothing fabricated that a reader could mistake for real proof.

**Seed the database the deployment actually reads.** A demo seeded into the local database and then deployed against a hosted one is a demo with no account in it — the sign-in button fails on the live site while working perfectly on localhost.

## Verify

- With `NEXT_PUBLIC_DEMO_MODE` unset, the app has **no** badge, no credentials card and no demo button anywhere. Check this properly; it is what makes the feature removable.
- With it set to `true`, the badge appears on both the front door and the in-app header.
- The credentials on the card are the ones that work. Type them into the normal sign-in form — not just the shortcut button — and land inside the app.
- The one-click button signs in and lands on the app's main page, on the deployed site, not only locally.
- `pnpm demo:seed` runs twice in a row without error, and the second run leaves the same data as the first.
- The seeded list shows something on **today**, and no visible number is a round fake.
- The sentence under the card is true of what the demo actually does.
