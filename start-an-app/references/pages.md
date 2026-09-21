# Landing page and dashboard

Last verified: 2026-09-21

**Purpose:** Give the app a real front door and, when there are accounts, a real place to land after signing in. This is the step that decides whether the result looks like *their app* or like a scaffold with the boxes ticked.

## First: what is the front door?

Do not reflexively build a marketing landing page. Pick from what the interview established:

| The app is… | Front door | Dashboard |
| --- | --- | --- |
| A product other people will sign up for (SaaS, marketplace, community) | Full landing page: what it is, who it's for, how it works, call to action | Yes — everything real lives behind sign-in |
| A personal tool with no accounts (it only ever runs on their own machine) | **No marketing page.** `/` *is* the app — the list, the board, the log | Not separately; `/` is already it |
| A personal tool **with** sign-in (one owner) | **No marketing page.** `/` is the app, and it lives inside the protected route group. A signed-out visitor is sent to `/sign-in` | `/` is it. There is no `/dashboard` |
| An internal or team tool | A short signed-out screen: name, one line, sign-in button | Yes |
| A public site where content is the point (blog, directory, portfolio) | The content itself, on `/` | Yes, for the author: someone has to publish, so this is a **one owner** app with a public front. The author's side sits behind sign-in, with sign-up closed after the first account |

A hiking journal for one person does not need a hero section and a pricing table. Building one is the single fastest way to make the result feel like a template. If the interview said "just me", skip straight to the app.

## Styling: `DESIGN.md` decides, these pages obey

`references/design.md` has already run, always: it comes right after the base project, so a `DESIGN.md` sits at the project root and its tokens and fonts are already in `src/app/globals.css` and `src/app/layout.tsx`. **Read it and follow it.** Where it and this section overlap, it wins — it has the actual palette, type and dials for this specific app, and it was written with the user in the room.

The direction was chosen there, once. A developer tool, a children's reading tracker, and an invoicing app should not look alike, and `DESIGN.md` is where that was decided. This file does not pick colours or fonts again.

The look lives in **one place** — the CSS variables in `src/app/globals.css` — never in one-off colours scattered through components:

- `--primary` (and `--primary-foreground`): the app's colour. This one variable does most of the work.
- The neutral base (`--background`, `--foreground`, `--muted`, `--border`): warm neutrals read friendly and editorial; cool greys read technical; near-black reads premium.
- `--radius`: `0.3rem` is precise and serious, `0.625rem` is the default, `1rem` is soft and approachable.

If a page needs a value `DESIGN.md` does not have, add it to `DESIGN.md` and to `globals.css`, then use it. Do not set it on the page.

A theme colour for the phone's browser bar goes in its own `export const viewport`, not in `metadata`.

**Dark mode is off unless something switches it on.** shadcn writes a `.dark` block into `globals.css`, but those tokens do nothing until the `dark` class is set on `<html>`, and nothing in this skill sets it (checked on the real build). The default is to build light only, and to say so at hand-off. If `DESIGN.md` calls for dark mode, add a `prefers-color-scheme` switch or a toggle, and only then check every page in both — a primary colour that's legible on white and invisible on near-black is a bug users will hit.

Rules that hold regardless of direction:

- Use shadcn components rather than hand-rolling buttons and inputs. `references/stack.md` added the likely set up front, before any page existed. Nothing runs `shadcn add` while pages are being written: it installs packages, and two installs at once damage the lockfile. If a page turns out to need a component that is missing, stop page work, add it once from a single process, and carry on.
- **`Button` has no `asChild`.** Current shadcn generates Base UI components, not Radix, so the reflex `<Button asChild><Link /></Button>` fails type-checking outright. Put the exported `buttonVariants` on the link instead — which is also one element fewer — but **wrap it in `cn()`**:

  ```tsx
  // shadcn's own components import cn from "cn"; src/lib/utils.ts re-exports it.
  // Either import works (measured on the real build). Pick one and keep to it.
  import { cn } from "@/lib/utils";

  <Link className={cn(buttonVariants({ variant: "outline" }))}>Sign in</Link>
  ```

  Without the wrap the link renders with **no border at all**. `cva` concatenates the base class string with the variant's, so `border-transparent` from the base and `border-border` from the variant both survive into the class list, and the one that wins is whichever Tailwind emitted last — not the one you asked for. `cn()` runs tailwind-merge, which drops the loser. The `Button` component never shows this because it merges internally; the bug appears only on links, which is to say on the landing page, which is to say on the first thing a stranger sees.

  It is also close to invisible: a missing 1px border on a dark surface reads as "that's just the style". Check it by measuring, not by looking — `getComputedStyle(el).borderTopColor` returning `rgba(0, 0, 0, 0)` is the tell.
- Give the app real spacing. Cramped, full-width, edge-to-edge content is the clearest tell of a scaffold: constrain the content width and let it breathe.
- Every list needs an **empty state** — the first thing the user sees is zero rows, and "nothing here yet, add your first hike" is the difference between working and broken-looking.
- Responsive from the start. Check one narrow viewport before calling it done.

## Landing page

Write it about *their* product, from the interview. Structure that works for almost any app:

1. **Header** — name, and either Sign in / Get started or nothing at all if there are no accounts.
2. **Hero** — what it is in one sentence a stranger understands, one supporting line, one primary action. Their words from the interview, not "Welcome to MyApp".
3. **What you can do** — three or four real capabilities, named concretely ("Log a hike with photos and notes"), not adjectives ("Powerful. Fast. Simple.").
4. **Pricing** — only if payments were set up, showing the real product name and price from `references/payments.md`.
5. **Footer** — name, year, and only links that exist. This is where later steps add theirs: `references/docs.md` puts a **Docs** link here if the app got documentation, and `references/legal.md` runs after both and decides whether this app owes a privacy policy or terms at all. Leave the footer somewhere they can add to it, and add nothing on spec. Where legal also built a consent banner, the footer carries the **Cookie preferences** control that reopens it.

**Page titles are not set here.** `references/seo.md` owns everything in `<head>` — the title, the template every page's title slots into, the description, and the preview card — because it runs last and is the only step that sees every public page. Leave `src/app/layout.tsx`'s metadata alone; it gets fixed there, in one place, rather than in two that drift.

**Never fabricate credibility.** No invented testimonials, customer quotes, company logos, star ratings, user counts, "trusted by 10,000 teams", or press mentions. The app has no users yet and everyone reading the page knows it. If a section would need social proof to work, leave the section out — an honest page with three real features beats a fake one, and fake reviews are the kind of thing that gets a real product in real trouble later.

Same rule for screenshots: show the actual UI or show nothing. No stock mockups.

## Dashboard

Only when sign-in was chosen. This is what proves auth actually works end to end, so build it even if it starts small.

Put it in a route group so the shell and the signed-out redirect are written once, in `src/app/(dashboard)/layout.tsx`:

```tsx
import { headers } from "next/headers";
import { redirect } from "next/navigation";
import { auth } from "@/lib/auth";

export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session) redirect("/sign-in");

  return (
    <div className="min-h-[100dvh]">
      {/* nav + signed-in user + sign-out */}
      <main className="mx-auto max-w-5xl px-4 py-8">{children}</main>
    </div>
  );
}
```

`min-h-[100dvh]`, not `min-h-screen`: on a phone the address bar makes the two differ, and `DESIGN.md`'s banned list says the same.

That `main` sets the content width and the padding for everything inside the group. A shell nested in it — the settings shell from `references/settings.md` is the usual one — adds its own sidebar or tabs but **no second `mx-auto max-w-* px-* py-*` wrapper**, or the padding doubles.

**One owner, personal tool:** the app's main page is `src/app/(dashboard)/page.tsx`, which serves `/`. Delete the scaffold's `src/app/page.tsx`, because two files that both resolve to `/` stop the build. There is no `/dashboard` route; wherever this file says "the dashboard", read `/`.

**Check the session on the server, in every page and every action, not only in the layout.** The layout redirect above is for tidiness: a layout does not re-run on client-side navigation and does not stop a child page from rendering, so on its own it protects nothing. Put the check in one cached helper and call it at the top of each page, each server action and each route handler behind sign-in.

Next's request interceptor is now a file called `proxy.ts` (it replaced `middleware.ts`, which is deprecated). It is optional. It is fine as an optimistic redirect that only looks at whether a cookie exists, and it is not the security boundary: a check that lives only in the browser or only in `proxy.ts` can be bypassed.

Request data is asynchronous throughout: `await params`, `await searchParams`, `await headers()`, `await cookies()`.

Write that helper now, in `src/lib/auth-guards.ts`, because this is the first step that needs it. `references/settings.md` adds `requireAdmin()` (pages) and `requireAdminAction()` (actions and route handlers) to the same file later and does not redefine these. On an app with more than two roles, the `requireRole(...roles)` guard described in `references/auth.md` goes in this file too, beside `requireUser`:

```ts
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

Leave room in the navigation for **Settings** — `references/settings.md` builds it later, after agent access, help, docs and legal have run, and hangs it off whatever nav you write here. It needs a place to live, not a placeholder page.

The dashboard page itself shows **their real data** — the nouns from the interview, with the primary action ("Log a hike") in reach. A page that only says "Welcome back, user@example.com" proves auth works and nothing else; go one step further and render the actual thing the app is for.

**Who may see which rows is not always "the signed-in user's own".** The interview's ownership answer settles one of three cases, and `references/database.md` built the tables to match:

| The data is… | Every query is scoped by | In the table |
| --- | --- | --- |
| Private to each user | The session user: `where(eq(table.userId, session.user.id))`, on every read and every write | A `userId` column, `onDelete: "cascade"` |
| Shared by a team | **Role**, not user. A plumber sees the jobs assigned to them; the office sees all of them. Take the role from the session and branch on it, or use `requireRole(...roles)` | No per-user scoping. A column that points at a person uses `onDelete: "set null"` or `"restrict"`, never cascade, or a member who leaves deletes the team's history |
| No accounts | Nothing: there is no session | No `userId` column at all |

On a private-data app, a query that forgets the `where` shows everyone each other's data — check each one. On a team app the same mistake is a role that sees too much — check each query against the "who sees what" answer from the interview. Either way the user or role comes from the session on the server, never from the request.

Put the rule in the shared `src/lib/<domain>` function that the page and the server action both call, not in the page. Then an agent tool call or a second page cannot get round it.

## Verify

- Signed out, `/` renders the right front door for this app, and every visible string is about their product.
- No invented testimonials, logos, ratings, or user numbers anywhere.
- Every footer link resolves, and there is no link to a legal page this app was never going to have.
- Signed out, visiting the protected page redirects to sign-in. That is `/dashboard` on most apps and `/` on a one-owner personal tool, where no `/dashboard` exists. Follow the redirect and see the sign-in page answer 200.
- **No account is created here.** The first account has to be the real person's, in Step 6; on a one-owner app a test account takes the owner's seat and locks them out. The three checks marked "Deferred" below wait until then.
- Deferred to Step 6, after the user has signed up: signing in lands on the dashboard (or `/`), which shows the signed-in user and their own data.
- Deferred to Step 6, after the user has signed up: sign out returns to the signed-out state, and the protected page redirects again.
- Every page, server action and route handler behind sign-in calls the session helper itself. Open one and look; the layout redirect does not count.
- There is no `middleware.ts` in the project. If a request interceptor exists, it is `proxy.ts`.
- Every query follows the ownership case above. Open each `src/lib/<domain>` function and read it: private data is filtered by the session user, team data by role, and nothing takes a user id from the request.
- Deferred to Step 6, after the user has signed up: one account cannot see what it should not. `references/verify.md` has the probe for each access shape — two accounts on open sign-up, role against role on an invited team, and on a one-owner app the two-account probe does not apply because only one account can exist.
- Empty states read as intentional, not broken. Signed-out pages can be checked now; the signed-in ones are looked at in Step 6.
- The app looks right at a narrow viewport. It is checked in light only, unless `DESIGN.md` calls for dark mode and a switch was built, in which case check both.
- No page or nested shell repeats the dashboard `main`'s width and padding.
- Any link styled with `buttonVariants` has a **visible border** where the variant says it should. Measure one: `getComputedStyle(link).borderTopColor` must not be `rgba(0, 0, 0, 0)`.
- Nothing on these pages contradicts `DESIGN.md`, and no component sets a colour outside its tokens.
