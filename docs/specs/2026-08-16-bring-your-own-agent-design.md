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
