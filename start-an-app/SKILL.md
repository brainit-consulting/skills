---
name: start-an-app
description: Interview the user about what they want to build, then scaffold a working full-stack web app around it. Use when the user wants to start a new app, website, prototype, or SaaS; when they don't know what tech stack to pick; or when they want a working starting point fast. Covers requirements discovery, project setup, database (SQLite, or Postgres hosted, in Docker, or local), sign-in, email, file uploads, payments, AI features, background jobs, optional agent access over MCP so tools like Claude can use the app, a DESIGN.md matched to the user's existing site or brand, a real landing page and dashboard, an in-app help guide, optional public help pages, account settings with system logs, the legal pages and cookie consent the app owes, whether search engines and AI crawlers should find it, a closing pass that proves the result builds, serves and does what was agreed, and offers to put it on GitHub, deploy it, or add a demo mode.
---

# Start an App

Turn an idea into a running web app. Understand the idea properly first, then build. The result is the user's actual app from the first commit — their name, their pages, their data model, only the infrastructure they need. It should never feel like a template.

**Understanding comes before scaffolding.** The interview is the most valuable part of this skill, not a formality to get through. Ten minutes of good questions produces an app the user recognises; skipping them produces a generic CRUD shell they have to rewrite. Do not run a single command until Step 3 is agreed.

## Ground rules

**Talking to the user**

- Explain every choice like you would to a smart friend who doesn't code. Say "a place to store your data" before saying "database". Introduce each technical term once, briefly, then use it normally.
- **Use their word for the thing.** If they call it a site, a system, or "the booking thing", call it that too. Saying "app" to someone who means a website — or who hears *phone app from the App Store* — quietly tells them this wasn't built for them.
- **Dig until it's clear.** Follow up on vague answers rather than filling the gap with an assumption. "A site for my club" is not yet a spec — what does a member *do* there?
- Ask about one topic at a time. During discovery, follow the conversation rather than reading from a list; for the technical choices, one question at a time with a recommended default so the user can just say "whatever you recommend".
- Surface gaps as suggestions, not interrogation. "Most apps like this need a way to edit an entry after posting it — want that in the first version?" is better than a checklist, and it's where the user learns what they actually want.
- Recommend, then respect. If the user picks the non-recommended option, go with it without relitigating.
- **If the user has standing rules about how their work should look (in a CLAUDE.md, a memory, or said in the conversation), those rules are the design brief.** Write them into `DESIGN.md` rather than asking as if the page were blank.

**What gets done, and who does it**

- **Set up what the app needs to run; offer everything beyond that.** The database is needed — wire it up yourself and don't make it a conversation. Pushing to GitHub and deploying are not needed for a working app, so they are offered at hand-off and never done mid-build. This skill ends with something that runs on their machine, not with their business live on the internet.
- **Do it if you can, guide if you can't, and always say which is happening.** Browser sign-ins can't be automated — `vercel login`, the Google Cloud console, a Polar or Stripe account. Say so plainly and offer to walk through it together, exactly as question 2 does for "Sign in with Google". Never pretend a step was done when the user has to do it.
- **Run commands freely, describe them in outcomes.** *"I've set up your database — it's free, it's yours, and it's already connected to where the app will live"*, never *"running `vercel integration add neon` to provision a Postgres instance"*. Good test: if you can't say what you just did in one plain sentence, don't do it unasked.
- **Four things are never done on your own initiative**, however easy the command is: creating an account or completing a login; anything that costs money, including leaving a free tier or buying a domain; anything that makes something public — the first deploy, a public repository; and attaching a domain, because their live business site is on the other end of that DNS record. Ask first, every time.
- Prefer the `vercel` CLI for Vercel work. Use a Vercel MCP server if this agent has one, but never depend on it — availability differs per user and per tool. Never use raw API calls with a hand-pasted token: it puts a credential in the conversation to do what the CLI already has a session for.
- **A check that wasn't run is named, never claimed.** Saying the app does something because you wrote the code that should make it do it is recall, not verification. Run the check where you can; where you can't — no browser, no key, no domain yet — say which one you couldn't do and what it would need. The user reads silence as success.
- **When several agents build at once, give each its own files, install every package and component before they start, and let only one of them run the app.** Two agents editing one file, two installs racing on the lockfile, or two dev servers fighting over a port each cost more than the parallel work saved.

**How the app is built**

- The app is scaffolded **in the current working directory** — that folder is the project root. Never create a subfolder for it and never `cd` into one; the user already chose where the app goes by being there.
- The stack is fixed: Next.js, TypeScript, Tailwind, shadcn/ui, Drizzle, Better Auth. The interview chooses *within* it (which database, what kind of sign-in, email, uploads, payments, AI, background jobs, the look, help and documentation, whether the app is meant to be found) — it never swaps out these pieces, and it never bolts a second framework alongside them. Documentation is pages in this app, not a docs platform beside it.
- **Never `drizzle-kit push`.** Schema changes always go through `db:generate` then `db:migrate`, every time, from the very first table.
- **Ids are randomly generated UUIDs — except in Better Auth's tables.** Every table you define gets one. The tables Better Auth's CLI generates stay exactly as generated, which also means any column pointing at a user stays `text`, not `uuid`. `references/database.md` has both branches.
- **Better Auth owns anything that belongs to a user.** Where Better Auth has a plugin for an integration — payments above all — use the plugin, never the provider's standalone SDK wired in beside it. One source of truth for the user, one place customer ids and webhooks live.
- **A tool is another caller, never a second way in.** If the app is opened up to AI agents, every tool goes through the same functions, the same ownership checks and the same log as the buttons do, and takes the user from the token rather than from anything the model passed. `references/mcp.md`.
- Prefer choices that survive deployment. Where a feature works differently in production (uploads, Postgres), the local setup and the deployed setup must be the same code switched by an environment variable — never a second code path the user has to remember to change.
- **Every app gets a `DESIGN.md`, and every page obeys it.** Long when the user gave a site or a brand to match, short when they said "you decide" — but written down once, before the first page exists, so no component invents a colour. `references/design.md`.
- **Every app gets a settings area.** Not as a finishing touch — from the first commit, scaled to what the app has. Accounts mean a profile, verification status, password, devices, and a way to leave. Every app, accounts or not, gets a system view: what's configured, what happened, what's running. `references/settings.md` and `references/ops.md`.
- **What the app owes its users legally is worked out, never asked.** Whether it needs a privacy policy, terms, or a cookie banner follows from what it is and what it loads — a personal journal owes none of them, a public product people sign up for owes the first two, and a banner is owed only where something non-essential actually loads, which the session cookie is not. Decide it, build exactly that, and put the call on the build sheet in one line so the user reads a decision rather than an oversight. `references/legal.md`.
- **Anything the app does out of sight is visible and controllable from inside it.** If the app sends an email, runs work in the background, or acts on a schedule, the user can see it happened, read why it failed, stop it, and try it again — in the app, not by reading logs on a hosting dashboard. Building something the user cannot watch is not finished.
- **Whether the app should be found is asked, and both answers are built.** Only where it could plausibly be found — a public product or a content site. A personal or internal tool is never asked and never gets a sitemap; it gets a real title in the browser tab and a deliberate *keep me out of search results*, which is a deliverable rather than an omission. `references/seo.md`.
- **Help is two different things, and neither is built on spec.** The **in-app guide** (`references/help.md`) is for people already using the app — staff, customers — and is decided from who the app is for, then offered on the build sheet. **Public help pages** (`references/docs.md`) are for strangers who haven't signed up, and are asked about only for a public product. Both are written only for what exists: four honest pages beat twenty, and a page describing a feature the app doesn't have is worse than no page at all — it sends someone looking for a button that isn't there.

**Keeping this skill honest**

- **Never write or accept a version number.** Not in an install command, not in a `package.json` snippet, not in prose, not a Docker image tag. No file in this skill pins one, and none should ever gain one. Every install takes the **current stable** release, and Step 2 is what establishes what that is. A version written into a skill file is a lie with a timestamp on it: it goes stale in silence and builds the app against last year's API.
- **Nothing deprecated, ever.** If the current release deprecates, renames, or supersedes something a reference file uses, use the replacement — not the old path that "still works". Still working is what deprecated means; it is a removal notice with a delay on it, and shipping onto one hands the user a rewrite they didn't ask for.
- All commands, package names, and config live in the reference files, never in this file. Load only the references for the branches the user chose.
- If a reference command fails because a tool changed (renamed flag, different init flow), check that tool's official docs, use the current equivalent, finish the job, and tell the user at the end that this skill's reference file needs a refresh.

## Step 1a — Understand the idea

Start here and stay here until the picture is sharp.

> **"What are you building? Describe it like you'd describe it to a friend."**

Then *follow up*. Listen for the **nouns** (the things the app keeps track of) and the **verbs** (what people do with them) — those become the database tables and the pages. Keep pulling until both are concrete:

- "Walk me through it — someone opens the app for the first time. What do they do?"
- "And then what? What brings them back the next day?"
- "When you say *[their vague word]* — what does that actually look like on screen?"
- "Is there anything like this you already use, that this is better than?"
- **"What are you doing about this today — a spreadsheet, a notebook, WhatsApp, nothing?"**

That last one earns its place. Anyone running a business already has the process, just badly — and their spreadsheet columns *are* the data model, their WhatsApp group *is* the notification requirement. Ask to see it if they'll show you. You stop guessing at a schema and start reading one.

It has a follow-up that decides whether the result is useful on day one: **"do you want what's already in there brought across?"** A business with four hundred customers opening an empty app has been handed a demo, not a tool. If the answer is yes, say plainly that importing is its own job and offer it as the first thing after the app works.

**Then say the data model back to them in plain words** and let them correct it. This is the highest-value question in the whole skill, because people who can't design a schema can absolutely tell you what's wrong with one:

> So the app keeps a list of **hikes** — each with a date, a trail name, distance, how it felt, and some photos. They're all yours; nobody else sees them. Have I got that right, or is there something else it needs to remember?

## Step 1b — Find the gaps

The user has told you the happy path. Your job is the rest. Run through these silently, and **raise only the ones that genuinely apply** — as a suggestion with a recommendation, not a quiz:

- **Who is this for, and whose data is it?** Their customers, their staff, or just them — and is what's stored private to each person, shared across the team, or public? Those three audiences are wildly different apps, and this is the one people forget to say. **Ask this one nearly always**: it decides every query in the app, and it also answers question 1 in Step 1c, so asking it well here means not asking it twice.
- **Can things be changed?** Most descriptions only cover creating. Editing and deleting are usually wanted and almost never mentioned.
- **Is anyone special?** An admin, a moderator, an owner who sees more than everyone else.
- **What does day one look like?** The app opens with zero data. What should be on that screen?
- **Anything time-based?** Due dates, reminders, recurring items, "this week" views.
- **Does anyone need telling?** Email on signup, on invite, when something happens. If yes, that's the email question in Step 1c — carry the answer forward rather than asking twice.
- **Does anything take a while?** Work that shouldn't happen while someone waits — importing a file, generating a report, calling a slow service, anything on a schedule. Most apps have none; the ones that do usually mention it here rather than in the happy path.
- **Phone or desktop?** Changes layout decisions early and is cheap to ask.
- **Anything sensitive in here?** Health details, children's information, card numbers. Only raise it when the subject matter suggests it — but when it applies it changes real decisions: card numbers are never stored (the payment provider holds them), sign-in defaults get stricter, and a record of who-changed-what starts being worth having in version one.
- **What is deliberately *not* in version one?** Ask directly. Naming what's out is what keeps a first version shippable, and it gives you permission to leave things out instead of guessing.

**Raise at most three.** Not "two or three usually" — three, hard, and the first and last on that list are normally two of them, which leaves you one free choice. Every one of these is a fair question and that is exactly the trap: ten fair questions in a row is an interrogation, and it's where someone who doesn't do this for a living decides the whole thing is too much like hard work. Pick the one that changes what you build; a gap found after the app exists is cheap to fill, and by then they'll be enjoying it.

## Step 1c — Technical choices

Now the branches. One at a time, each with a recommendation. **Don't ask what they've already told you** — if the description made an answer obvious ("a paid newsletter", "a photo journal"), confirm it in passing instead: *"Sounds like people will be paying for this — I'll set that up."*

1. **"Who's this for — your customers, your staff, or just you?"**
   → **Usually already answered** by the first gap-check in Step 1b. If it is, don't ask again — confirm it in passing (*"since this is for your customers, I'll set up the sturdier kind of database"*) and move to question 2. Asking a second time is how someone decides you weren't listening.
   → Just me / trying an idea: recommend **SQLite** ("your data lives in a simple file inside the project — nothing extra to install or run").
   → Other people / production ambitions: recommend **Postgres** ("the database most real apps use — I'll set it up on a free hosted plan, where you get your own private copy to build against, so nothing you do here can touch the real thing").
   → Default to **Neon through the Vercel marketplace**: free, nothing installed, and production and preview deployments are already wired up when you deploy. It needs a Vercel account and an internet connection — check that before promising it.
   → If either is a problem, `references/database.md` has three alternatives that need neither: **Docker** ("one command to start, nothing installed permanently, and the same database you'll use in production"), a **local Postgres server** installed as a package, or **PGlite** (Postgres inside the project, offline, nothing to install). Docker needs Docker Desktop installed and running — check before promising that too. Offer whichever fits their objection; don't make them choose from a menu.

2. **"Do people need their own login?"**
   → No accounts: skip auth entirely.
   → Yes: recommend **email + password** as the default ("works immediately, nothing to configure").
   → If they want "Sign in with Google": say yes, and set expectations — it needs a free Google Cloud setup with a few copy-paste steps; offer to walk through it together or add it later.

3. **"Does it need to send any email — confirming an address, resetting a password, telling someone something happened?"**
   → No: skip email entirely. Sign-in still works; there's just no verification or password reset until it's added.
   → Yes: recommend **Resend**. Set expectations honestly and early, because this is the one component that needs something they may not have: "It works straight away for sending to yourself. To email anyone else you'll need a domain name, and a few DNS records — about ten minutes, and free." If they don't have a domain, take it anyway and say the sending step waits for one — everything else works in the meantime, with emails printed to the terminal.
   → If they said no to sign-in but yes to email, that's fine — a contact form or a notification doesn't need accounts.

4. **"Will people upload anything — photos, documents, a profile picture?"**
   → No: skip file storage entirely.
   → Yes: no decision to make, so don't offer one. Say what happens: "While you're building, uploads save into a folder in the project. When you deploy, they'll go to proper cloud storage automatically — same code, you just connect a store." Only mention Vercel Blob by name if they ask.

5. **"Will customers pay you through this — a subscription, or a one-off purchase?"**
   → No: skip payments entirely.
   → Yes: recommend **Polar** ("they handle sales tax and VAT worldwide for you, which is the part that usually bites"), with **Stripe** as the option if they already use it or need it.
   → Payments need accounts. If they said no to sign-in, say so plainly and add it: "we'll need accounts too, so the app knows whose subscription is whose."
   → Set expectations: everything is set up in test mode, no real money, and going live is a key swap later.

6. **"Should it do anything with AI — like answering questions, or writing text for you?"**
   → Only include AI plumbing if yes. If yes, mention they'll need an OpenRouter API key (free to create) and you'll show them where to get it — one key, many models.

7. **"Does anything need to keep running on its own — work that carries on after you close the tab, or happens on a schedule?"**
   → Default is **no**, and most apps should stay there. A server action handles saving a record, sending one email, or resizing one image perfectly well; adding a job system for that is overhead with a dashboard attached.
   → Yes when work must survive a restart, retry itself after a failure, run on a schedule, fan out over many items, or wait minutes to days for something. Importing a spreadsheet, generating a report, calling a slow external service, a nightly digest.
   → If yes: recommend **Inngest** ("it runs the work outside the app, picks up where it left off if something crashes, retries on its own, and you can watch every step of it happen while you build"). It's free to start and needs no account at all during development.

8. **"Do you want Claude to be able to work in this app for you — reading and updating it the way you would?"**
   → A genuine either/or, so ask it that way and don't lean. No means the app is used by people in a browser, which is a perfectly good answer, and this can be added later without changing anything built before it. If no, nothing about agent access is scaffolded.
   → Yes: say what it actually means, because the AI question above was something else. "You connect Claude to the app once, and from then on you can ask it to do the real work — add something, move something, tell you what's on there. It signs in as you, and it can only do what you can do." There is no API key to mind: it goes through the app's own sign-in and a consent screen. It works from Claude Code straight away, and from Claude.ai or ChatGPT once the app is deployed somewhere public.
   → Say the two things that surprise people: it acts **as them**, so it inherits their permissions and nothing more; and every call it makes is written down, reads included, on a page they can see.
   → Worth saying if they're unsure: the tools end up being the same handful of things the app already does, so the cost is mostly the sign-in plumbing, and there's a page listing every agent that has access with a button to cut it off.
   → `references/mcp.md`. Needs accounts, the same way payments do. If they said no to question 2, say it in one sentence — "an agent has to sign in as you, so the app knows whose data it's touching" — and add sign-in, rather than treating it as a blocker.

9. **"When someone who's never used this before arrives, what should they see first — a page explaining what it is, or straight into the thing itself?"**
   → Decides the front door: a real landing page for something other people will sign up for, or straight into the app for a personal tool. Don't assume a marketing page — `references/pages.md` has the call.

10. **"Do you already have a website? Paste the address and I'll match your colours and fonts. If not — is there a site whose look you like?"** *(optional — one ask, then move on)*
    → **If the user already has standing rules about how their work should look** — in a CLAUDE.md, a memory, or said earlier in the conversation — those are the brief. Don't ask as if the page were blank: say in one line that you'll follow them, ask only whether there's a site to match on top, and write the rules into `DESIGN.md`.
    → A URL: extract the palette, type, shape and density from it and write them into a `DESIGN.md` the rest of the build obeys. `references/design.md`.
    → A vibe instead ("like Linear", "warm and friendly", "expensive and quiet"): equally good input — same file, skip the extraction.
    → Nothing, or "you decide": don't push. Write a short `DESIGN.md` from what the app *is*, which is what `references/pages.md` would have made you decide anyway.
    → If they name someone else's site, say once that you'll take the feel and not the identity — no logo, images, copy or stylesheet — and carry on. Don't turn it into a lecture.

11. **"Should it come with a few help pages people can read without signing in?"**
    → These are **public** pages for strangers deciding whether to sign up, or stuck before they have. They are not the in-app guide, which is for people already inside the app and is never asked about — it goes on the build sheet in Step 3.
    → **Ask only if the answer to 9 was a real landing page** — a product strangers sign up for, most of all one that takes money. A personal tool has one user who already knows how it works, and an internal tool's documentation is usually a message to three colleagues; asking there invites a yes to something nobody will read. Don't ask, don't build, don't mention it.
    → Default is **no**. Say what a yes actually costs: four to six short pages that have to stay true every time a screen changes. If they want it, name the pages you'd write from what they've already told you — "getting started, how it works, plans and billing, connecting Claude" — so they're agreeing to something concrete rather than to the idea of documentation. `references/docs.md`.

12. **"Should search engines — and AI assistants — be able to find this?"**
    → **Ask only where the app is public**: a product people sign up for, or a site whose content is the point. For a personal or internal tool, don't ask. Say what you're doing instead, in one line: "nobody's meant to find this, so I'll give it a proper name in the browser tab and keep it out of search results" — that's the deliverable, not the absence of one.
    → Where it applies, default is **yes**, and it's cheap: a sitemap, a `robots.txt`, an `llms.txt`, and a preview card for when the link gets shared. If the help pages question above was a yes, mention those pages get indexed too — for most products that's the half people actually search for.
    → One sub-question, and only where the app's *content* is the product (a blog, a directory): whether AI crawlers may train on it. Search and citation crawlers are a different thing and worth allowing — that's how an assistant recommends the app with a link. `references/seo.md` splits the two.

## Step 2 — Check what's current

The branches are chosen, so now find out what building them actually involves *today*. Nothing in this skill names a version, deliberately — this step is where the versions come from. It costs one round of parallel subagents and prevents the expensive failure: an app built confidently against an API that moved.

**Dispatch one subagent per chosen branch, all in a single message so they run at once.** Only the branches the interview selected — there is no sense researching payments for an app that takes no money. The base project, the database, the pages step and discoverability always count as branches here.

Each gets the same brief with its own packages filled in:

> Find the current stable release of `<packages>`. Report: the latest stable version of each; anything deprecated, renamed, moved to a different package, or removed within the last two majors; the current import paths and function signatures for `<the specific things this reference file uses>`; **any capability added since that would replace hand-written code in `references/<file>.md`**; and any migration note that would break what's in there. Prefer the package's own docs and changelog over blog posts or search summaries, and check what is actually published on the registry rather than what a docs page claims. Say plainly what you verified against a primary source and what you inferred.

**The agent-access branch gets one extra sentence in its brief**, because packages are not the only thing that moves under it: *establish the current revision of the Model Context Protocol specification, and check `references/mcp.md`'s assumptions against that revision's changelog and its registry of deprecated features.* A protocol revision can deprecate something the file relies on without any package changing its name or its signature, and the brief above would sail straight past it. No other branch sits on a spec that versions independently of its libraries.

**The discoverability branch installs no packages and researches conventions instead**, so give it its own brief: *confirm Next's current `MetadataRoute` file conventions for `sitemap`, `robots` and `opengraph-image`; and — only where the app is meant to be found — establish the current user-agent tokens for AI crawlers, split into training, search/citation, and user-initiated, and whether `llms.txt` has moved past a proposal toward anything a named crawler documents reading.* Crawler names change without notice, and a wrong one in `robots.ts` is not an error, it is a rule that silently matches nothing.

**Commands move as fast as packages do.** The host's CLI is the clearest case: `references/database.md` uses it to set up a hosted database, and `references/deploy.md` is almost nothing but its commands, flags and prompts. Where the database is hosted, that branch's brief also confirms the CLI's current commands for adding the database and pulling environment variables. `references/deploy.md` and `references/demo.md` are not part of this round, because deploying is only offered at hand-off — but if the user accepts that offer, run the same check on `references/deploy.md` before the first deploy command, not after it fails.

Each reference file carries a `Last verified` date at the top. Use it to size the effort: a file verified recently needs a confirmation pass, one verified a year ago needs the assumption that something has moved.

Then reconcile, before installing anything:

- **Latest stable only.** Not release candidates, not betas, not `next` or `canary` tags — unless the user asks for one specifically and knows why.
- **Take the new capability when there is one.** Reference files sometimes hand-roll something because the library couldn't do it yet. If it can now, use the built-in and delete the workaround — `references/mcp.md` says exactly where this is likely.
- **On API detail the research wins; on how the pieces fit together this skill wins.** Names, signatures, import paths, options, flags: take what the research found. Which piece owns what, and how it wires into the rest of the app: the reference file. Most reference files restate this split at the top for their own dependency.
- **If a reference file's approach is now deprecated, take the replacement** and finish the job with it. Don't split the difference.
- **Say something to the user only when something changed.** One line, plain: "Better Auth moved that into a separate package since this was written — I'm using the new one." Never narrate research that found everything was fine; it reads as filler.
- **Write down what's stale.** Anything the research contradicted goes in the hand-off at the end, so this skill can be corrected.

## Step 3 — Build sheet

Restate the plan in plain words before touching anything. Example shape:

> Here's what I'll set up: **"TrailLog"** — a hiking journal, just for you.
>
> **Where it goes:** `H:\TrailLog` — that folder is empty, so the app will be created there.
>
> **What it remembers:** hikes — date, trail, distance, how it felt, and photos.
> **What you can do:** log a hike, edit it later, delete one, see them newest-first.
> **Signing in:** email and password, so it's yours alone.
> **Photos:** saved in the project while you build; they move to cloud storage when you deploy.
> **From Claude:** you'll be able to log a hike or ask about past ones from Claude itself, without opening the app — and see and revoke that access from inside it.
> **Look:** warm and quiet, built from the colours and type on your own site — written down in `DESIGN.md` so it stays consistent.
> **Also included:** a settings page where you can change your password and delete your account, and a system page showing what's set up and what's happened.
> **Legal:** nothing needed — it's just you, and nothing here tracks anyone, so no privacy policy, no terms, no cookie banner.
> **Being found:** nothing to index — it's just you, so I'll give it a proper name in the browser tab and keep it out of search results.
> **Not in version one:** sharing hikes with friends, maps, and the stats page — easy to add once the basics feel right.
>
> Sound right?

Include the data model and the explicit **not in version one** list — those two lines are what stop a rewrite later.

Some lines go in every time, as statements rather than questions:

- The **look** line says where the design comes from — their site, the vibe they named, their own standing rules, or your call from what the app is — and that it is written down in `DESIGN.md`.
- The **legal** line: this example says nothing is needed and why, and a public product would name the pages it gets instead. `references/legal.md` makes the call.
- The **being found** line: this example shows the harder half — the app that is deliberately kept out of search still gets a line, because "no SEO" read as silence looks like something forgotten. A public product names what it gets instead: sitemap, `robots.txt`, `llms.txt`, and a preview card for shared links.

Two lines about help appear only sometimes, and they are different things:

- **Built-in help** — the in-app guide, for people already using the app. The example above is a personal tool, so it has none. **When the app is for customers or staff, add the line** — *"a `?` in the corner opens a short guide, written from what you've just told me, so new staff can work it out without asking you"* — as something they can decline rather than something you ask about. `references/help.md` explains why it isn't a question.
- **Help pages** — public documentation for strangers, outside sign-in. The line appears only where question 11 was asked and answered yes, and it names the pages rather than promising documentation.

An app can have either, both, or neither. Never let one stand in for the other: a public "how it works" page does not help a receptionist mid-booking, and an in-app guide is invisible to someone who hasn't signed up.

Also mention anything that needs something from them before it can work (a Vercel account, Docker running, an API key, a domain for email, a provider account), so there are no surprises mid-build.

**Say the full path, and say what is already in it.** Don't ask them to choose a folder — the app is created where the session already is, and an agent can't reliably write outside it. But they can't confirm a location they were never shown, and "wherever you happened to open the terminal" is not the same as a decision. One line, as above.

If the folder already contains a project — a `package.json`, a `src/`, a `.git` with history — **stop and say so.** Do not scaffold over it. Ask them to open a new, empty folder and start again there. A few stray files (a `README.md`, notes, a `.gitignore`) are fine and `references/stack.md` handles them; an existing app is not, and merging into one is unrecoverable in a way nothing else in this skill is. Offer them the other door: the `bring-your-own-agent` skill can give an assistant like Claude access to this existing app instead — without scaffolding anything or touching a line they wrote — if that's closer to what they actually want.

Get a clear go-ahead. Adjust anything they push back on. If the answer reopens what the app *is* rather than tweaking a detail, go back to Step 1a — that's cheaper now than after the schema exists.

## Step 4 — Scaffold

Work through these in order. Each reference has a **Verify** section — complete it before moving on. Those are your own check as you go; Step 6 is the one that has to survive a command. Every path in them is relative to the current working directory.

1. Base project → `references/stack.md`
2. Database (SQLite or Postgres branch) → `references/database.md`
3. Sign-in, if chosen (email+password, optionally Google) → `references/auth.md`
4. Email, if chosen → `references/email.md` (also wires verification and password reset, if sign-in ran)
5. File uploads, if chosen → `references/storage.md`
6. Payments, if chosen → `references/payments.md` (requires sign-in)
7. AI features, if chosen → `references/ai.md`
8. Background jobs, if chosen → `references/jobs.md`
9. Design system → `references/design.md` (always; a short `DESIGN.md` even when the user gave no direction)
10. Landing page and dashboard → `references/pages.md`
11. Agent access, if chosen → `references/mcp.md` (requires sign-in)
12. In-app help guide, if the app is for customers or staff → `references/help.md` (offered on the build sheet, never asked)
13. Public documentation, if chosen → `references/docs.md` (rarely; only a public product that was asked and said yes)
14. Legal pages and cookie consent, as much as this app owes → `references/legal.md` (decided, never asked; often nothing)
15. Account settings → `references/settings.md` (requires sign-in; skip only if there is no sign-in)
16. System visibility → `references/ops.md` (always)
17. Discoverability → `references/seo.md` (always, but for most apps this means a real title and staying out of search)

The order matters: payments, uploads and agent access all extend what sign-in built. The design system lands before any page exists, so everything built afterwards inherits its tokens rather than being repainted later. The pages step needs the lot in place, and settings and system visibility hang off the navigation it creates. Agent access sits after the pages step because its consent screen has to look like the rest of the app, and before the rest because each grows a section only if it ran. The in-app guide comes next: it documents what the pages step actually built rather than what was planned, and it has to describe the app as it ended up, agent access included where that was chosen (add a short "Working with an AI assistant" chapter pointing at Connected apps in settings). Public documentation follows because it too can only describe branches that exist, and before legal so that legal's pass over the footer sees the docs link already there. Legal comes after every feature branch for the same reason in reverse — the privacy page has to describe all of them — and before settings, which grows a cookie-preferences section only if a banner was built. Anything that changes `src/lib/auth.ts` means regenerating the Better Auth schema and running `db:generate` + `db:migrate` again — the reference files say where.

**Discoverability is last because it is the only step that has to know every public page.** It writes the sitemap and `llms.txt` from one list, and legal and documentation both add pages to it — a sitemap written before them is wrong the moment they run.

The last three are not a polish pass to drop if time is short. Two of them turn a scaffold into something the user can operate, and the third decides whether anyone will ever find it.

## Step 5 — Make it theirs

This is not a polish pass; it is most of the value. The scaffold in Step 4 is infrastructure — here the app becomes recognisably theirs.

- Name the project after their idea (package name, page titles, visible branding).
- The schema tables are the **nouns** from Step 1a, each with a UUID primary key, and the ownership rule from Step 1b applied — a `userId` column (`text`, matching Better Auth) and every query scoped to it if data is private.
- Build the real pages: the front door and dashboard from `references/pages.md`, real navigation, and the **verbs** from Step 1a wired up — including editing and deleting if the gap-check said so. Everything visual comes from `DESIGN.md`; no page invents a colour.
- Seed nothing generic: every visible string should make sense for *their* app. No "Item", no "Welcome to Next.js", no lorem ipsum. This includes the settings area, the emails, the legal pages, the help guide, any documentation, and **the browser tab** — a section called "Notifications" listing categories the app never sends, an email signed "My App", a privacy policy about "user-generated items" in an app whose every other screen says "hikes", or a tab still reading "Create Next App", are all the same failure as a page of lorem ipsum.
- Build only the settings sections this app has. An empty Billing tab or a Notifications tab for an app that sends no email is worse than a missing one.
- If agent access was chosen, the tools are named for the **verbs** too — `log_hike`, not `create_item` — and they are the handful of things someone would actually ask for, not one per table.
- If the help guide was built, its chapters are written here too — from the interview, in their words. Done when every **verb** from Step 1a has one.
- Done when: someone opening the app would know what it is without being told, and the user can do the main thing the app exists for, end to end.

## Step 6 — Prove it

The app is built. Nothing has established that it works. Every Verify section you just completed was confirmed by the same agent that wrote the code it checks, and recall is not evidence — an app can satisfy every one of them while failing to compile.

`references/verify.md` has the commands: types, schema drift, the build, lint, the app served in production mode, every route answering, two accounts against each other, and the app with its keys taken away. Read the output of each. Having written the code a command tests is the reason to run it, not a reason to skip it.

Two points of order matter enough to say here, because getting either wrong does damage:

- **Schema before build.** `build` runs `db:migrate` first, so reaching it with an ungenerated schema edit outstanding applies SQL nobody read — the one thing `references/database.md` says never to do, performed by the step meant to catch it.
- **The user signs up before any probe account exists.** The first account created becomes the admin. A fixture that takes that place, and is then deleted, locks the user out of their own system page.

And four things the commands have to show, not just run:

- **Lint is its own check.** Next no longer runs ESLint as part of `next build`, so a build that goes green is not evidence of a clean project — a broken hook rule sails straight through it. If anything is left failing, name it at hand-off rather than leaving it to be found later.
- **Missing keys degrade, never crash — for *optional* features.** With `.env` values absent, the app still starts and the affected feature (payments, AI, cloud storage, Google sign-in, email, background jobs) shows a friendly "not configured yet" notice.
- **The database is not optional, so it does not degrade.** With `DATABASE_URL` missing the app fails immediately and says what's wrong in one plain sentence — on the hosted branch, *"the database isn't connected yet: run `vercel env pull .env.local`"*. An app that starts, looks fine, and then errors on the first click is worse than one that refuses to start, because it sends the user looking in the wrong place.
- **The real thing, end to end.** `drizzle/` contains generated migration files, no schema was ever pushed, and creating one real record works. With sign-in: signing up, signing out and signing back in works, and `/dashboard` redirects when signed out and shows their own data when signed in. With uploads: a file uploaded through the app's own UI renders after a refresh. With payments: the test-mode checkout completes and the paid state is visible server-side. `DESIGN.md` exists, `globals.css` matches it in both light and dark, and no component sets a colour outside the tokens. With the help guide: `?` opens it, every verb from Step 1a has a chapter, and it reopens where the user left it.

Where a check needs a browser, a provider, or a person, `references/verify.md` lists it. Name what you couldn't run.

## Step 7 — Fresh eyes

The gate proves the app builds, serves and answers. It cannot tell whether the app *does* anything — an empty project passes every command in it, because nothing leaks when there is nothing to leak. So the last check is the one the builder cannot perform on itself.

**Dispatch the critics in a single message so they run at once**, the same way Step 2 does: promise-keeping and looks-like-theirs always, ownership wherever there is sign-in, operability scaled to the branches that ran. The briefs are in `references/verify.md`.

**They check the app against the Step 3 build sheet, and against nothing else.** That sheet is the only bar here, because it is the one thing the user actually approved — and they approved a description of an app without being able to read the code they were handed. Closing exactly that gap is the whole job. Critics report where the app diverges from the sheet; they never propose a different sheet. Anything that would change what the app *is* goes to the user at hand-off, the same as it would have gone to them at Step 3. The looks-like-theirs critic reads `DESIGN.md` as part of the sheet: the look line is a promise like any other.

They read evidence, not the running app. Four agents cannot share a port or a browser between them, so capture everything once during Step 6 and hand it over. Findings come back as `broken`, `missing` or `worth knowing`; only the first two are fixed now, and no fix widens the gate or changes scope.

**Two rounds, then stop and report what's left.** A third round is where an agent starts editing code it doesn't understand to make a report go away. Tell the user this step is happening, in one line — otherwise they are watching a terminal do nothing.

## Step 8 — Hand off

- Every check that could not be run is named, with what it would need — a browser, a domain, a payment key — as a short list they can work through in a minute once the app is open. Findings left unfixed after Step 7, and any finding you disagreed with, go here too, one line each with the reason.
- Anything Step 2's research contradicted, or any reference command that had to be changed on the fly, is named at the end — which file, what was wrong — so this skill can be corrected. Say it to the user in one line; they may be the person who fixes it.
- Legal branch: say once, plainly, that the privacy and terms pages are a first draft assembled from what the app actually does rather than legal advice. Then list the fields in `src/lib/legal.ts` that only they can fill — usually a contact address and whose law governs the terms — as a short thing they can clear in a minute. Where there is no cookie banner, give the one-line reason and what would change it: "nothing here tracks anyone, so there's nothing to consent to — add analytics later and it'll need one."
- Discoverability, kept out of search: name the two places the switch lives — `robots: { index: false }` in `src/app/layout.tsx` and `src/app/robots.ts` — as the thing to change if the app ever goes public. Left in place on a launched product it costs them every visitor they were expecting, and it is invisible.
- Discoverability, public: say plainly that a sitemap is an invitation and not a ranking, that `llms.txt` is a proposed convention no major AI crawler has committed to reading, and that what a crawler is actually *permitted* to do lives in `robots.txt` alone. Point them at the preview card once — it is what a shared link looks like, and it is the first thing they'll see the app judged by.
- Docs branch: say how many pages there are and that they are true today, which makes them the first thing to go stale. If writing any page was hard because the flow needed explaining, say which one — that is a finding about the app, not about the page. The same goes for the in-app guide: a chapter that was hard to write points at a screen that is hard to use.
- Close with a plain-language summary: how to start the app (including `pnpm db:up` or `pnpm db:local` if the database runs on their machine), what each entry in `.env` is for, and two or three sensible next steps.
- Show them the system page and say what it's for. It is the answer to "why didn't that email arrive?" and "is that still running?", and they will not find it on their own.
- Where local and production differ, spell out the one-time switch: connect a Blob store for uploads, point `DATABASE_URL` at a hosted database if it isn't already, swap payment keys out of test mode, add the Resend key once the domain is verified, add the Inngest keys and sync the app, point `BETTER_AUTH_URL` at the real domain so agent tokens are issued for it, and set `APP_URL` to the same real domain so the sitemap, canonical links and preview card aren't full of `localhost`. Each is a setting on the host, not a code change — say that, because it's the part people expect to be hard. A host's database integration sets its own `POSTGRES_URL` as well; the app reads `DATABASE_URL`, so that is the one to check exists for production. The two that also need an action outside the host are verifying the email domain in DNS and syncing the app with Inngest after the first deploy; call those out by name.
- **Then offer the things you deliberately didn't do**, one line each, and do none of them without a clear yes:
  - *"Want me to put this on GitHub, so there's a backup and a history of every change?"* → **private by default**, and say the visibility out loud before creating it: *"I'll make it private — only you can see it."* `gh repo create <name> --private --source . --push`. If they want it public, that's their call, but it should be a decision they made rather than a default they inherited.
    **Before the first push, check what is about to leave the machine.** Scan the tracked files for the value of the auth secret and for key-shaped strings — provider key prefixes, long random tokens, a connection string with a password in it — and confirm `.env` and `.env.local` are ignored and were never committed. A private repository is still a copy of the secret on someone else's server, and a repository made public later carries its whole history with it. If anything turns up, remove it and rotate the key before pushing, and say so.
  - *"Want me to put it online so you can look at it on your phone?"* → `references/deploy.md`. Say what a preview URL is in one sentence: a real web address, live now, that anyone with the link can open. That last part matters to a business owner and takes four words.
    **Don't deploy from memory, however familiar the command looks.** Everything the app reads from `.env` — the auth secret, the API keys, often `DATABASE_URL` itself — exists only on the user's machine, and a deploy that doesn't carry them across produces a site that builds, goes green, loads, and then fails on the first click. That reference is a checklist for exactly that, and it verifies against the live URL rather than the one that already worked. Its commands move like any other's, so give it the Step 2 check first.
  - If the reason for putting it online is to **show** the app to someone — a client, a prospect, a colleague — rather than to use it, that is a different job: `references/demo.md` adds a way in that doesn't hand over the user's own data. Load it *before* `deploy.md`, because it introduces a build-time variable that has to be set before the build runs. Offer it only when the user says the deployment is for showing; a demo nobody asked for is scaffolding to delete later.
- Deploying is where this skill stops. A real launch — a custom domain, going live with payments, search engines finding it — is its own conversation, and saying so is more useful than half-doing it.
