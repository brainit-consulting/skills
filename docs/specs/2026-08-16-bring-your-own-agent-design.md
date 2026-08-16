# bring-your-own-agent — design

**Status:** agreed 2026-08-16. Not yet implemented.

A skill that gives an app somebody already has the thing Ratchet & Rail has: an
assistant that can do its real work. It adds one folder and edits nothing.

## Why it is a separate skill

`start-an-app` builds agent access as question 7 of its interview, and can do
that easily because it built everything underneath — it knows there is Better
Auth to hang an OAuth provider on, and its own functions to wrap. None of that
holds for an app that already exists.

It also already **halts** when it finds a real project in the target folder. That
stop is deliberate: the upstream skill worked around an existing `package.json`
by scaffolding to a temp directory and moving the result up, which silently
splices a fresh Next.js app into somebody's repo when the skill is fired in the
wrong place. The fork made it a stop instead.

That stop is a dead end today. It becomes the front door: *this looks like an
existing project — want agent access added to it instead?* One line changes in a
skill people have already installed, and `bring-your-own-agent` still stands
alone for anyone who knows what they want.

## The five decisions this design rests on

Each was a real fork, and the rejected options are recorded because the reasons
outlive the decision.

### 1. Any stack — Rails, Django, Express, FastAPI, Laravel, Go

Not "apps this skill built", which would be a handful of people, and not "any
Next.js app", which is a narrower version of the same problem.

This is only tractable because of decision 2. The skill must **read** an
unfamiliar codebase well enough to find its routes; it never has to **write** in
that language. Reading unfamiliar code is what an agent is genuinely good at.

### 2. It sits alongside and changes nothing

The MCP server is a new folder in their repo. Not one existing file is edited.

Rejected: putting an MCP endpoint inside their app so it calls their internal
functions and inherits their permission checks. That is the property that makes
R&R's version trustworthy, and it is the better design where you can have it —
but it needs a token scheme most apps lack, and it must be written again for
every framework. Also rejected: the full R&R treatment — OAuth provider, consent
screen, scopes. Correct and large. For most existing apps it means adding an
OAuth server to something that has never had one, which cannot be promised
generically across stacks.

**The cost, which the skill must state plainly:** the agent acts as one identity.
There is no consent screen, no per-user scoping, no per-app revoke. Whatever that
credential reaches, Claude reaches.

### 3. Local by default, deployed as a deliberate second step

Over stdio, launched by Claude Code itself: no port, no network, no token to
leak. It does not work from Claude on a phone, from claude.ai, or for anyone
else — which is how most MCP servers work today, and worth saying rather than
discovering.

Deploying is offered at hand-off, never done mid-build — the same shape
`start-an-app` uses for GitHub and deployment.

### 4. API first, read-only database fallback, writes never touch SQL

The rule that makes R&R's agent access trustworthy is *the tools call the same
functions the buttons call*: one query path, one permission check, so the tools
cannot drift from what the UI enforces. A server outside the app honours that
only by going through the app's own API.

- **Reads** — the API where one exists; the database directly where none does.
- **Writes** — the API. Always. No SQL write, no exception, no flag.

An agent writing SQL skips validation, permissions, hooks and audit logging. The
app ends up in a state its own code believes impossible, and it surfaces later,
somewhere unrelated.

### 5. Discovery by reading the code, confirmed by the user

Point it at the folder; it finds `routes.rb`, `urls.py`, `app.get(...)`, FastAPI
decorators. It then shows what it found in plain words and asks which matters.
**The user edits a list rather than writing one** — they correct, and are
reminded of capabilities they would not have thought to ask for.

Rejected: OpenAPI-only, since most small apps have none and the fallback is
interviewing somebody about their own endpoints. Rejected: asking outright, which
misses whatever they did not think to mention.

## Shape

```
their repo (every existing file untouched)
└── agent-access/
    ├── server.ts        the MCP server
    ├── tools.ts         one tool per agreed capability
    └── .env             the credential, gitignored

   server → HTTP → their app's API            reads and ALL writes
   server → SQL  → their database, read-only  reads only, when no API
```

TypeScript regardless of their stack, because it only ever speaks HTTP and SQL.

Inside their repo rather than beside it: the server is coupled to their API's
shape, so it should move when the API moves and be reviewed in the same pull
request. A separate repo keeps the project pristine and drifts the first time an
endpoint changes.

## The fork the interview must offer

**Some apps can do this properly, and those users must be told so — by name,
early, and in plain words — even though it means this skill steps aside.**

Everything in decision 2 is a trade: no per-user scoping, no consent screen, one
identity, in exchange for touching nothing. That trade is worth making when the
app cannot support the alternative. It is a bad trade when it can, and the user
is the only one who can weigh it — but they can only weigh what they are told.

**When to raise it.** After the stack is detected, before any capability is
discussed:

- **Next.js with Better Auth** — the strongest case. `start-an-app/references/mcp.md`
  is a proven playbook for exactly this, and the result is better in every
  dimension that matters: the tools call the same functions the buttons call,
  each person approves their own access, each can revoke it, and the agent can
  only ever do what that person could do.
- **Any stack with real user accounts and a permission model**, where in-app is
  better in principle but this skill has no playbook for it. Say exactly that —
  *the better version exists but I cannot build it for you here* — rather than
  quietly recommending the one thing on offer.
- **Sensitive data** — money, health, anyone else's records, multi-tenant.
  "Whatever that credential reaches, Claude reaches" is a far heavier sentence
  for an accounting app than a to-do list. Raise it as a question about their
  data, not as a technical footnote.

**How to put it**, in the skill's own voice — no jargon, no scare tactics, and no
pretending the recommended path is free:

> Your app already has proper sign-in. That means it can do this the good way:
> the assistant asks *you* for permission, acts only as you, and can only do what
> you could do — and you can take that away again whenever you like.
>
> What I can build here is simpler and quicker. It sits beside your app and works
> as one fixed login, so anything that login can see or change, the assistant can
> too. There is no permission screen, and no way to give one person less access
> than another.
>
> The good way is more work and touches your app's code. Which would you rather?

**Then respect the answer.** If they choose this skill anyway — for speed, to try
the idea, because they do not want their app touched — build it, well, without
relitigating. Record the choice in the generated README so whoever reads it later
knows the trade was made deliberately. The recommendation is given once.

**Never make this decision for them.** Do not silently refuse, and do not quietly
hand off to another skill. A skill that decides an app is "too important" for it
and stops has taken a judgement that belongs to the person who owns the app.

## The interview

0. **Could your app do this properly?** — the fork above, once the stack is
   known, before capabilities are discussed.
1. **Which app?** Folder or repo. Confirm it is real before touching anything.
2. **What is in it?** Detect the stack, read the routes, list them in plain words.
3. **What may Claude do?** Edit the found list. Reads and writes decided
   separately — agreeing to reads is not agreeing to writes.
4. **How does it get in?** Credentials, below.
5. **Build, connect, verify.** Real calls, observed. Never "this should work".
6. **Deploy?** Offered, explained, not assumed.

## What makes it safe rather than merely useful

**Writes only ever go through the app.** Stated once here so it is not
re-litigated per app.

**The read-only fallback uses a genuinely read-only database user**, created
during setup — a permission the database enforces, not a promise in code that a
later edit quietly breaks. Where one cannot be created, say so and fall back to
API-only reads.

**Every list tool takes a `limit` with a hard maximum, enforced in the schema and
in the query.** An uncapped list either truncates mid-JSON into something
unparseable or hands one call the entire account.

**No delete tool.** Throwing work away is a decision for a person, in the app,
where they can see what they are throwing away.

**Two disclosures, in plain words, at the moment they become true:**

- *The agent acts as one identity* — when the credential is chosen.
- *Direct database reads can see rows the app would hide.* If the app filters by
  tenant or owner in application code, SQL does not know that. Said when the
  fallback is chosen, not in a footnote afterwards.

## The hard cases, which are the normal cases

"Read the routes" is trivially true for Express and Next.js, where a route is a
file or a literal `app.get(...)`. That is the shape this design was drafted
around and it is the **least** representative one. Every item below must be
handled by `discovery.md` or the skill is only honest about JavaScript.

**Routes that are never written down.** Rails `resources :orders` silently
becomes seven routes across two controllers. Django REST Framework routers do the
same from a `ViewSet`. Laravel `Route::resource` likewise. A skill that greps for
literal paths finds *none of them* and reports an app with no API. Each stack
needs its expansion rules, and where the framework can list them itself
(`rails routes`, `manage.py show_urls`, `php artisan route:list`) running it beats
inferring — with the inference documented as the fallback when the app will not
boot.

**Apps with no API at all.** A server-rendered Rails or Django app returns HTML
and nothing else. Under decision 4 this means **reads work and writes are
impossible** — and that is a limit, not a bug. It must be said at the start of
the interview, not discovered at step 5 after somebody has invested an hour.

The tempting workaround — posting to HTML form endpoints the way a browser
does — is **rejected**. It means scraping a CSRF token out of markup, replaying
a form encoding, and interpreting a 302 as success; it breaks the first time
somebody edits a template, and it fails silently by re-rendering the form with
errors that look like a 200. If an app needs agent *writes* and has no API, the
honest answers are: add one endpoint, or build agent access inside it.

**Auth that is nothing like a bearer token.** Devise, Django sessions, Laravel
Sanctum, hand-rolled cookies, an API key in a header nobody documented. This is
where the CSRF trap lives, and it needs a section per stack rather than a
paragraph.

**Layouts that do not match the tutorial.** Monorepos, nested modules, several
apps in one repo, generated code, vendored dependencies. FlatBooks already has
`modules/*/src/app/api/**` — real apps are messier than that. The skill must ask
where the app actually lives when it is not obvious, rather than picking the
first thing it recognises.

**Verifying without the conveniences.** The dev server may not run on this
machine, the database may need seeding, the app may need credentials the skill
does not have. Verification cannot assume any of it. Where a real call is
impossible, say the tools are **unverified** and name what is missing — never
report success because the code looks right.

**The rule that ties these together:** the skill must never present a partial
discovery as a complete one. "I found three endpoints" when a framework generated
forty is worse than finding nothing, because the user accepts a list believing it
is the list. Where discovery is uncertain, say what was searched, what was found,
and what could not be determined.

## Reference files

| File | Holds |
|---|---|
| `discovery.md` | Finding routes and auth per stack |
| `server.md` | The server template, tool shapes, caps, logging |
| `credentials.md` | Getting a token out of common auth systems; creating the read-only database user |
| `deploy.md` | stdio → remote, and the bearer check that then becomes necessary |

Cross-referenced both ways with `start-an-app/references/mcp.md`: that one is for
building agent access *into* a Next.js app you control, this one for bolting it
*beside* an app you would rather not touch. Anyone who lands on the wrong one
should be told within a paragraph.

## The first trap to write down

Most existing apps authenticate with **session cookies plus CSRF tokens**, not
bearer tokens. A server posting to a Rails or Django endpoint with a valid
session cookie and no CSRF token gets a 403 that explains nothing. Reads work,
writes fail, and the obvious fix — exempting those routes from CSRF protection —
is a real security regression in their app, arrived at by accident.

`credentials.md` must cover: obtaining the CSRF token the way their framework
expects, per-framework header names, and why exempting the route is the wrong
answer.

## v1 stacks

**Rails, Django, Express, FastAPI.** Laravel and Go are the likely next two; each
costs a discovery section and ships only when earned.

A stack ships only when its *implicit* routing, its auth, and its no-API case are
all written down. A stack listed on the strength of a route grep is a promise the
skill cannot keep, and the user finds out at step 5 — which is the worst possible
moment, because by then they have chosen this path over the better one.

## Generated per app, from a documented shape

Not a template copied in. The tools are named for that app's verbs, take that
app's parameters, and return that app's shapes — the same principle
`start-an-app` holds to, that the result is the user's actual app rather than a
template they have to rewrite. A generic `call_endpoint(path, method, body)` tool
is a template wearing a disguise: it pushes every decision onto the model at
runtime and gives the user nothing to review.

The cost is that `server.md` must be a **specification with worked examples**
rather than a file to copy — the caps, the error handling, the logging, the tool
shapes — precise enough that two runs of this skill produce servers that differ
only where the apps differ.

## What "verify" means here

The skill did not build this app and cannot know the right answer to
`list_orders`. So verification is a collaboration, and it is the user who
confirms — but the skill does the work and never asks them to take anything on
trust.

**Reads.** One real call per tool, with the result shown as it came back, not
summarised. The question is *"is this your data, and does it look complete?"* —
that catches the two failures the skill cannot see for itself: pointing at the
wrong environment, and a query returning a slice while claiming to be the whole.

**Writes.** One real write, then two confirmations: the skill reads it back
through a different tool, and the user finds it in the app's own interface. The
second is not redundant. It is the only check that the write went through the
app's own rules rather than round them, and it is the one that catches a
successful-looking call the app itself does not consider valid.

**Undo what you wrote.** A verification write is a real row in a real system. Say
what it will create before creating it, use something obviously disposable, and
remove it afterwards — through the app, so the removal is as legitimate as the
write.

**When a real call is impossible** — no dev server on this machine, no
credentials, an empty database — the tools are reported **unverified**, with what
was missing named. Never "this should work". A tool that has never been run is
not a working tool, and saying so is the difference between a hand-off and a
hand-wave.

## Validating it

`H:\flatbooks` is available as a test target and is the **smoke test, not the
proof**. It is Next.js 16 + Better Auth + Drizzle/Neon — the same stack as
Ratchet & Rail — so it exercises the mechanics (discovery, the server, tools
returning sane data) and says nothing about decision 1, which is the whole risk.
The real test needs a stack this skill has never seen: Rails, Django or Laravel.
Worth having one before implementation rather than during.

FlatBooks is also the worked example for *the fork the interview must offer*. It
has Better Auth, so step 0 should recognise it, name the better path, and put the
choice to the user — and it is accounting software with a bank feed and Stripe,
so the one-identity trade is exactly the case where that conversation earns its
place. Whichever way the answer goes, it is the user's to give.

Running the skill against FlatBooks is therefore two tests, not one: whether step
0 fires and reads honestly, and whether the build still works when somebody hears
the recommendation and chooses this path anyway.

## Not in scope

Per-user scoping, consent screens, OAuth. Those need the app's cooperation, which
is decision 2 traded away on purpose. An app that wants them should build agent
access *inside* itself — which is `start-an-app/references/mcp.md`, and the
cross-reference should say so.

## What running it found

Run on 2026-08-16 against `H:\flatbooks`, following `SKILL.md` as written rather
than from memory of building it. Every status code, header and message below was
observed on a running app; nothing here is inferred.

Two things did not happen and are said first, because a validation section that
buries them is worse than none.

**No verified read.** The server was built, connected, and made a real call to
the running app — but no credential could be obtained, so neither tool was ever
run against real data. Both are reported **unverified**, and the reason is a
finding in itself; it is *Getting in* below.

**Django is unverified in the wild.** There is no Django app on this machine
outside the skill's own fixtures. `dir /s /b H:\manage.py` — unlimited depth,
the whole drive — returns four files: the three under `H:\skills\fixtures\`, and
`H:\python\Lib\site-packages\pinecone\manage.py`, which is a library module for
managing Pinecone indexes with no `settings.py` beside it and no Django in it. A
`find` to depth eight, run separately, agrees, and so does a third check from
the other end: of the 34 `requirements*.txt`, `pyproject.toml` and `settings.py`
files on the drive, **not one names Django**. So decision 1 — *any stack* — is
still tested only against apps written for this skill, and the one foreign app
available was Next.js. **The fixtures are not evidence and this section does not
treat them as any.**

### Step 0 fired, on two triggers at once

It fired, and it fired at the right moment — the stack is legible from
`package.json` while Step 1 is still confirming which app this is, and Step 0
says to raise it the moment the stack is recognised. FlatBooks matched two of
the three triggers: Better Auth 1.6.11 is in its dependencies, and it is
accounting software with a bank feed, invoices and Stripe.

The text the skill produces is `SKILL.md`'s own, with the app's words in it:

> Your app already has proper sign-in — Better Auth. That means FlatBooks can do
> this the good way: the assistant asks *you* for permission, acts only as you,
> and can only do what you could do — and you can take that away again whenever
> you like. That is `start-an-app/references/mcp.md`, and it is a path that has
> been built before.
>
> What I can build here is simpler and quicker. It sits beside FlatBooks and
> works as one fixed login, so anything that login can see or change, the
> assistant can too. There is no permission screen, and no way to give one
> person less access than another. This is your books — a bank feed, your
> clients' names and what they owe — so that sentence is heavier here than it
> would be for a to-do list. What is actually in there, and who would mind if it
> were read?
>
> The good way is more work and touches your app's code. Which would you rather?

No defect. It named the better path by file, put the choice in plain words, did
not decide, and did not hand off. **This is the text for the appendix.**

One thing it does not yet say, and should, is in the next finding.

### The stack Step 0 recommends is the stack discovery cannot read

`discovery.md` covers Django, Express and FastAPI, and its *Not yet supported*
section names Rails, Laravel and Go. **Next.js appears in neither list.** Its
`Verify` says *the stack is one of Django, Express or FastAPI; if not, the owner
was told it is not covered* — so the skill, run as written on FlatBooks, must
tell the owner their stack is uncovered one paragraph after Step 0 told them
their app was special *because* of that stack.

That is not dishonest, but it is a bad minute, and the design is the reason: it
asserts that reading routes is *"trivially true for Express and Next.js, where a
route is a file"*. That turned out to be true and useless — see below.

**Gap → `discovery.md` needs a Next.js App Router section.** It is not written
anywhere today. What it needs was measured here and is in the three findings
that follow.

### The routes are not the app

FlatBooks has 18 files under `src/app/api/**/route.ts`. Sorted by what an
assistant could use:

| What they are | How many |
|---|---|
| Webhooks, signature-verified (Stripe, PayPal, bank feed) | 5 |
| Cron, gated by a shared `CRON_SECRET` | 3 |
| Better Auth's own catch-all | 1 |
| Session-gated flows that only make sense from the UI (bank connect, Stripe checkout/portal) | 6 |
| **Actually useful to an assistant** | **3** |

The three are `GET /api/exports/ledger` and `GET /api/exports/pack`, both
returning files to a signed-in browser, and `POST /api/integrations/ingest`,
which takes its own bearer key.

Meanwhile the app's real work — create a client, create an invoice, send it,
mark it paid, add a transaction, import a CSV, save settings — lives in **22
exported Server Actions** across four files marked `'use server'`. Next.js
compiles each into a build-time id, and the framework will list them:
`.next/server/server-reference-manifest.json` holds 25 entries, each naming its
source file and exported name.

**They are enumerable and they are not callable.** The id is a hash that changes
when the build changes, and the calling convention — POST to a page URL with a
`Next-Action` header — is internal protocol with no compatibility promise.
Driving it is the same bargain `discovery.md` already refuses for HTML form
posts: replaying an undocumented encoding that breaks silently the next time
somebody rebuilds.

So the honest capability list for FlatBooks is **two tools out of roughly
twenty-four things the app can do**, and that is the finding decision 1 most
needed. An app can have an API, pass every no-API check, and still keep nine
tenths of itself out of reach. Nothing in `discovery.md` today would say so: it
asks *is there an API?* and FlatBooks answers yes.

**Gap → `discovery.md`.** The Next.js section has to make the Server Action
count part of the discovery report, next to the route count, and say plainly
that those capabilities are unavailable to a server sitting beside the app. It
is also the strongest argument Step 0 has, and Step 0 does not currently make
it: *the good way would reach the other twenty-two.*

### Next.js does generate a route list, and the file tree lies about it

The framework writes `.next/app-path-routes-manifest.json` — source path to URL,
for every route — on a build, and it is there after `next dev` too. That is the
Next.js answer to `manage.py shell` and `app.openapi()`, and it matters for the
usual reason:

```
/(app)/invoices/page        -> /invoices
/(auth)/login/page          -> /login
/(app)/invoices/[id]/page   -> /invoices/[id]
```

**Route groups are stripped from the URL.** A path built from the file tree
gives `/(app)/invoices`, which is a 404. FlatBooks has two of them, `(app)` and
`(auth)`, wrapping 20 of its 49 app paths. So "a route is a file" is true only
if you know which parts of the path are not part of the path.

The manifest does not carry HTTP methods — those come from reading which of
`GET`/`POST`/… each `route.ts` exports, the same way Django's `-` verbs column
means *go and read the view*.

**Gap → `discovery.md`**, in the same new section.

### A route that exists answers `405`, not `404`

Measured, with no credentials:

```
GET /api/integrations/ingest   405
GET /api/banksync/sync         405
GET /api/exports/ledger        307  -> /login
GET /api/cron/remind           401  text/plain
```

`discovery.md`'s probe rule is *send one request to each route and keep the ones
that answer*, and it never mentions 405. A 405 means the opposite of what
dropping the route would imply: the route is there and the verb is wrong. The
example given for a route to discard is a DRF format-suffix 500, which is a
genuine casualty — so the rule as written invites someone to throw away every
POST-only endpoint in the app, which on FlatBooks is most of them.

**Gap → `discovery.md`'s *Then probe what you found* section.** Stack-neutral;
it belongs above the per-stack parts.

### Getting in — where the build actually stopped

`credentials.md` opens with a check it calls measurable rather than a guess:
send one unauthenticated request and read the `WWW-Authenticate` header, because
the server names its own scheme there. Its first `Verify` item makes that
mandatory.

**FlatBooks emits no `WWW-Authenticate` header anywhere.** Zero occurrences
across the 401 from `ingest`, the 401 from `cron`, and the 307 from `ledger`.
Nothing in Next.js emits one and no hand-written route handler here does either.
The scheme word had to be read out of the 401's body —
`{"error":"Missing Authorization: Bearer <key>"}` — and confirmed in the source.

Then the rule that actually stopped the build. `SKILL.md` Step 4 says to *create
a login for the assistant, rather than borrowing a person's, wherever the app
allows it*. On FlatBooks that path dead-ends, in the worst shape:

```
POST /api/auth/sign-up/email   200
{"token":null,"user":{"name":"Agent Access","emailVerified":false,...}}
```

**HTTP 200, a complete-looking user object, `"token": null`, and no `Set-Cookie`
at all** — the cookie jar came back empty, and the ledger with that jar still
returned 307 to `/login`. Every check an operator would naturally run says the
sign-up worked. There is no credential. Then:

```
POST /api/auth/sign-in/email   403
{"message":"Email not verified","code":"EMAIL_NOT_VERIFIED"}
```

The account exists and cannot sign in until a verification email is delivered to
a real inbox. `credentials.md`'s Django recipe mints a token from
`manage.py shell` — a command-line path into the app's own data — and the
dedicated-login rule quietly assumes every app has one. Most modern SaaS does
not, and on those the rule produces an account that cannot be used. This is the
same failure `credentials.md` already documents for Django's login form — *a 200
from that POST is a failure, not a success* — arrived at from a different
direction, on a stack with no section.

And there is a second reason the rule would not have helped even if the sign-up
had worked: **on a per-tenant app a fresh agent login sees nothing.**
`/api/exports/ledger` reads `bankFeed(ctx.ownerId)`, scoped to the signed-in
user, so a brand-new account has empty books. The dedicated identity that makes
revocation clean also makes the agent useless.

FlatBooks has its own answer, and it is a better one than either horn: an
accountant invite grants read-only access to somebody else's books, and
`resolveBooksContext` returns `canWrite: false` for it — enforced by the app,
per-owner, revocable. **An existing app may already have a read-only delegate
role, and where it does, using it beats both a fresh login and borrowing the
owner's.** Nothing in the skill says to go looking for one.

**Gaps → `credentials.md`**, three of them: a stated fallback for when no
`WWW-Authenticate` header exists (and a softened `Verify` item, since today it
demands something most apps do not emit); a section on token-less, verification-
gated sign-up, with the 200-and-no-cookie shape written down; and a companion to
the dedicated-login rule — look for the app's own delegate role first, and say
what to do when creating an account is not possible.

### `redirect: "error"` is wrong, and the default is worse

`server.md`'s worked example sets `redirect: "error"` on its fetch. Against an
app that bounces an unauthenticated request to a login page — which is most apps
with sessions — all three modes were measured:

```
redirect:"error"    -> THREW TypeError: fetch failed | cause: unexpected redirect
redirect:"manual"   -> returned 307 /login | res.ok = false
redirect:"follow"   -> returned 200        | res.ok = true
```

`"error"` throws, so the handler's `if (!res.ok)` branch never runs and an
expired session reaches the client as `TypeError: fetch failed` — a message
naming the network, with auth nowhere in it.

**`"follow"` is the one to write down, because it is the default.** Drop the
option and an unauthenticated request returns **200, `res.ok` true**, carrying
the login page. The tool then parses HTML as if it were data and hands the
assistant a result with no error anywhere in the chain. That is the failure this
whole skill keeps circling — a rejection that looks exactly like a success — and
here it is reached by leaving one line out.

The generated server uses `"manual"` plus an explicit 3xx branch, and the tool
call returned the right thing:

> The app redirected to /login instead of answering. That means the session
> cookie in agent-access/.env is missing or expired.

**Gap → `server.md`.** The worked example's `redirect: "error"` should become
`"manual"` with a 3xx branch, and the three-way measurement above belongs beside
it.

### Two smaller shapes `server.md` has not met

**Two tools, two credentials, two schemes.** The worked example carries one
`APP_API_TOKEN`. FlatBooks needs a Better Auth session cookie for the read and
an `fbk_` bearer key for the write, and no single credential reaches both. Not a
defect — but the one-identity disclosure is written for one identity, and here
there are two, revoked in two different places.

**A list endpoint that returns CSV and has no `limit`.** `/api/exports/ledger`
returns `text/csv` and every row on the account. `server.md` calls the cap
applied to returned rows a second cap; here it is the **only** cap, because
there is no parameter to send. The *copy fields out one at a time* rule still
applies, to CSV columns rather than JSON keys.

**Gap → `server.md`:** a short note on both, so neither is re-derived per app.

### What the promise is worth: measured

The central claim held, and it was checked rather than asserted.

```
$ git status --porcelain          # in H:\flatbooks, after the whole build
?? agent-access/
```

One untracked folder. **Not one existing file modified**, and `.gitignore` was
never touched — which was a constraint of this run, since there was no owner to
ask. It was not needed:

```
$ git check-ignore -v agent-access/.env
agent-access/.gitignore:1:.env	agent-access/.env
```

The nested `.gitignore` is honoured by the parent repository, so the credential
is protected without editing a file the owner wrote. `SKILL.md` already says
this; it is now measured.

The folder was removed afterwards and `git status --porcelain` returned to
empty.

### What was verified, and what was not

Verified, observed:

- `npx tsc --noEmit` clean, under the `nodenext` config `server.md` specifies.
- The server connected over stdio **from a working directory that was not the
  folder** (`cwd: C:/`), which is the case the absolute-path `.env` loading
  exists for.
- Both tools advertised, with the right annotations — `readOnlyHint: true` on
  the read, `destructiveHint: false` on the write.
- The schema cap enforced: `limit: 500` was rejected before the handler ran, with
  `Too big: expected number to be <=50 at limit`.
- One real HTTP call from a tool to the running app, answered, and logged:
  `{"tool":"list_transactions","args":{"limit":5},"outcome":"error redirect_307"}`.
- No credential in `calls.log`, no `console.log` in either source file, no spread
  of a response into a result.

**Not verified:** either tool against real data. No credential existed, for the
reasons above, and inventing one would have been the hand-wave this skill spends
a whole section forbidding.

Two further limits on this run, both worth knowing. The server was driven over
stdio by a script rather than registered with `claude mcp add`, so as not to
write to the operator's own client configuration — it proves the same protocol
exchange, but `server.md`'s registration line is still unrun outside its own
fixture. And the run left one artefact it could not clear: an account created
through FlatBooks' own sign-up API while the dev server ran against the
project's `test` database branch. Database access was unavailable in the
session, so which branch the row landed in could not be confirmed, and FlatBooks
exposes no delete-user endpoint to remove it through the app. It is a row, not a
file; `git status` is clean either way. It is recorded here rather than left for
somebody to find.

### The one that surprised us

Not the redirect, and not the missing header. It was that **FlatBooks passes
every honesty check in `discovery.md` and still cannot be honestly served.** It
has an API. The API returns JSON. Routes enumerate, probe, and answer. Every
question the skill knows how to ask gets the reassuring answer — and the app
still only offers two of its twenty-four capabilities, because the rest were
written in a shape that has no web address at all.

`discovery.md`'s rule is *never present a partial discovery as a complete one*.
This run found the way to break that rule while following every instruction in
it: report the routes truthfully, and let the owner infer that the routes are
the app. On the stack where Step 0 has its strongest recommendation, the
strongest argument for taking that recommendation is a number the skill does not
currently count.
