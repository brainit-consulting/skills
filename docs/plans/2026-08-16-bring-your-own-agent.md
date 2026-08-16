# bring-your-own-agent Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a skill that adds agent access over MCP to an app somebody already has, on Rails, Django, Express or FastAPI, by generating one folder and editing nothing they wrote.

**Architecture:** A `SKILL.md` interview plus four reference files. The skill reads an unfamiliar codebase to find its routes, agrees a capability list with the user, and generates a TypeScript MCP server in `agent-access/` that talks to the app over HTTP (all writes, and reads where an API exists) and read-only SQL (reads only, where none does). It runs locally over stdio by default; deploying is a separate offered step.

**Tech Stack:** Markdown skill files. The *generated* artefact is TypeScript on Node with `@modelcontextprotocol/sdk`. Fixture apps in Ruby, Python and Node for verification.

**Spec:** `H:\skills\docs\specs\2026-08-16-bring-your-own-agent-design.md` (commit `6c40292`). Read it before Task 1 and keep it open — this plan argues from it and does not restate its reasoning.

## Global Constraints

- **Repo:** `H:\skills` (`brainit-consulting/skills`). Skill lives at `H:\skills\bring-your-own-agent\`.
- **Skill name:** `bring-your-own-agent`. Never `byoa` or `add-agent-access` in any user-facing string.
- **Voice:** as `start-an-app` — explain every choice to a smart friend who doesn't code. Say "a place to store your data" before "database". Use the user's word for their thing.
- **Never edit a file the user already had.** The only exception in this whole plan is appending to their `.gitignore`, and that is asked for first.
- **Writes go through the app's API. No SQL write, ever, no flag to enable one.**
- **Reads via direct SQL require a database user the database itself restricts to SELECT.** Not a promise in code.
- **No delete tool**, in any generated server.
- **Every list tool takes `limit` with a hard maximum, enforced in the schema and in the query.**
- **Two disclosures, in plain words, at the moment they become true:** one identity; direct reads can see rows the app would hide.
- **Never present a partial discovery as complete.** State what was searched, what was found, what could not be determined.
- **A stack ships only when its implicit routing, its auth, and its no-API case are all written down.**
- All commands and code live in `references/`, never in `SKILL.md`.

---

## File Structure

| Path | Responsibility |
|---|---|
| `bring-your-own-agent/SKILL.md` | The interview and ground rules only. No commands, no code. |
| `bring-your-own-agent/references/discovery.md` | Finding routes, auth and the no-API case, per stack. |
| `bring-your-own-agent/references/credentials.md` | Obtaining a credential per stack; the CSRF trap; the read-only database user. |
| `bring-your-own-agent/references/server.md` | Specification for the generated server, with worked examples. |
| `bring-your-own-agent/references/deploy.md` | stdio → remote, and the bearer check that becomes necessary. |
| `start-an-app/SKILL.md` | One added line at the existing-project halt. |
| `start-an-app/references/mcp.md` | One added cross-reference paragraph. |
| `README.md` | Skill list entry. |
| `fixtures/` (gitignored, local only) | The four apps used to verify discovery. |

---

### Task 1: Fixture apps, so discovery can be verified rather than asserted

Without these, `discovery.md` is a document nobody has run. Generated apps are fine — they exercise the framework's *implicit* routing, which is the whole difficulty.

**Files:**
- Create: `H:\skills\fixtures\` (four apps)
- Modify: `H:\skills\.gitignore`

**Interfaces:**
- Produces: four app directories, each with at least one resourceful route set and one auth mechanism, referenced by path in Tasks 2 and 3.

- [ ] **Step 1: Check what is installed**

```bash
ruby -v; rails -v; python --version; node -v
```

Record which are missing. A stack whose toolchain is absent must be built via Docker in Step 3 or explicitly deferred — **do not** write its discovery section from documentation alone. That is the corner this plan exists to prevent.

- [ ] **Step 2: Ignore the fixtures**

```bash
cd /h/skills && printf '\n# Local fixture apps for verifying discovery. Not shipped.\nfixtures/\n' >> .gitignore
```

- [ ] **Step 3: Generate the four apps**

```bash
mkdir -p /h/skills/fixtures && cd /h/skills/fixtures

rails new rails-shop --api --skip-test --skip-system-test
cd rails-shop && bin/rails generate scaffold Order customer:string total:decimal && cd ..

python -m venv .venv
django-admin startproject django_shop && cd django_shop \
  && python manage.py startapp orders && cd ..

npx --yes express-generator express-shop

mkdir fastapi-shop
```

- [ ] **Step 4: Give Django a DRF ViewSet and FastAPI a router**

These are the implicit-routing cases. A grep finds neither.

`fixtures/django_shop/orders/views.py`:

```python
from rest_framework import viewsets
from rest_framework.response import Response

class OrderViewSet(viewsets.ViewSet):
    def list(self, request):
        return Response([{"id": 1, "customer": "Ada", "total": "9.99"}])

    def retrieve(self, request, pk=None):
        return Response({"id": pk, "customer": "Ada", "total": "9.99"})

    def create(self, request):
        return Response(request.data, status=201)
```

`fixtures/django_shop/django_shop/urls.py`:

```python
from django.urls import include, path
from rest_framework.routers import DefaultRouter
from orders.views import OrderViewSet

router = DefaultRouter()
router.register(r"orders", OrderViewSet, basename="order")

urlpatterns = [path("api/", include(router.urls))]
```

`fixtures/fastapi-shop/main.py`:

```python
from fastapi import APIRouter, FastAPI

app = FastAPI()
router = APIRouter(prefix="/api/orders", tags=["orders"])

@router.get("")
def list_orders(limit: int = 50):
    return [{"id": 1, "customer": "Ada", "total": "9.99"}][:limit]

@router.get("/{order_id}")
def get_order(order_id: int):
    return {"id": order_id, "customer": "Ada", "total": "9.99"}

@router.post("")
def create_order(customer: str, total: float):
    return {"id": 2, "customer": customer, "total": total}

app.include_router(router)
```

- [ ] **Step 5: Add a template-rendered app — the no-API case**

The hardest case in the spec, and the one most likely to be skipped. Rails scaffolding without `--api` produces HTML.

```bash
cd /h/skills/fixtures && rails new rails-html --skip-test --skip-system-test
cd rails-html && bin/rails generate scaffold Invoice number:string amount:decimal
```

- [ ] **Step 6: Record what each fixture is for**

Create `fixtures/README.md`:

```markdown
# Fixtures

Not shipped. These exist so `discovery.md` is verified rather than asserted.

| App | Verifies |
|---|---|
| `rails-shop` | `resources :orders` expanding to 7 routes a grep cannot see |
| `rails-html` | The no-API case: HTML only, so reads work and writes are impossible |
| `django_shop` | A DRF router generating routes from a ViewSet |
| `express-shop` | The easy case, kept honest as a control |
| `fastapi-shop` | An `APIRouter` with a prefix, mounted via `include_router` |
```

- [ ] **Step 7: Commit**

```bash
cd /h/skills && git add .gitignore && git commit -m "chore: ignore local fixture apps used to verify discovery"
```

---

### Task 2: `references/discovery.md`

**Files:**
- Create: `H:\skills\bring-your-own-agent\references\discovery.md`

**Interfaces:**
- Consumes: fixture apps from Task 1.
- Produces: for each of the four stacks — a **detect** command (proves which stack), an **enumerate** command (lists routes including implicit ones), a **fallback** for when the app will not boot, and an **auth-shape** section naming what `credentials.md` must then handle. `SKILL.md` step 2 calls into this by stack name.

- [ ] **Step 1: Establish the ground truth for each fixture**

Run each and paste the real output into the reference. Do not paraphrase from memory.

```bash
cd /h/skills/fixtures/rails-shop && bin/rails routes | head -30
cd ../django_shop && python manage.py show_urls 2>/dev/null || echo "django-extensions absent — fallback needed"
cd ../fastapi-shop && python -c "
from main import app
for r in app.routes:
    print(getattr(r,'methods',None), r.path)"
cd ../express-shop && grep -rn "router\.\|app\." routes/ app.js
```

- [ ] **Step 2: Write the framework-asks-itself rule first**

Open the file with the principle, because it is the whole difference between this working and not:

```markdown
## Ask the framework, do not guess

Every stack here generates routes that appear in no source file. Rails
`resources :orders` becomes seven routes across two controllers; a DRF router
does the same from a `ViewSet`; FastAPI's `include_router` rewrites every path
with a prefix. **A grep for literal paths finds none of them and reports an app
with no API** — which is not a small error, because the user then chooses a
read-only integration, or this skill over a better one, on the strength of it.

So: run the command that makes the framework list its own routes. Inference is
the fallback for an app that will not boot, and it is always announced as such.
```

- [ ] **Step 3: Write the four stack sections**

Each must contain, with real observed output:

```markdown
### Rails

**Detect:** `Gemfile` with `gem "rails"`, plus `config/routes.rb`.

**Enumerate:** `bin/rails routes` (Rails 5+). Columns: prefix, verb, URI pattern,
controller#action. Needs a bootable app — it loads the environment, so a missing
database or absent env var makes it fail. Actual output from a scaffolded app:

    orders     GET    /orders(.:format)          orders#index
    ...

**Fallback when it will not boot:** read `config/routes.rb` and expand by rule —
`resources :x` yields index, create, new, edit, show, update, destroy; `only:`
and `except:` narrow it; nesting prefixes the parent path. Say the list was
inferred and may be wrong where `member`/`collection` blocks are used.

**Is there an API at all?** `rails new --api` omits `ActionView`. Look for
`config.api_only = true` in `config/application.rb`, controllers inheriting
`ActionController::API`, or `jbuilder`/serializer files. Where actions render
HTML templates only, this is the no-API case — see below.

**Auth shape:** Devise (`gem "devise"`) is session + cookie; `has_secure_token`
or a custom header suggests a token. Hand off to `credentials.md`.
```

Repeat in full for Django, Express, FastAPI. Do not write "similar to Rails".

- [ ] **Step 4: Write the no-API section**

```markdown
## When there is no API

A template-rendered app returns HTML. Under the write rule this means **reads
work and writes are impossible**, and that must be said at step 1 of the
interview, not discovered at step 5.

**Do not** post to HTML form endpoints as a browser would. It requires scraping a
CSRF token out of markup, replaying a form encoding, and reading a 302 as
success. It breaks the first time somebody edits a template, and it fails
silently: a re-rendered form full of validation errors is still HTTP 200.

The honest options, offered in this order: add one JSON endpoint to the app; or
build agent access inside it (`start-an-app/references/mcp.md`); or accept a
read-only integration, which is often genuinely useful.
```

- [ ] **Step 5: Verify against every fixture**

For each of the five fixture apps, follow only what the reference says and record whether the route list matches ground truth from Step 1.

Expected: `rails-shop` yields 7 order routes; `django_shop` yields the router's set; `fastapi-shop` shows `/api/orders` with the prefix applied; `express-shop` matches its route files; `rails-html` is correctly identified as no-API.

Any mismatch is a defect in the reference. Fix it before committing.

- [ ] **Step 6: Commit**

```bash
cd /h/skills && git add bring-your-own-agent/references/discovery.md
git commit -m "feat(byoa): route discovery for rails, django, express and fastapi

Each stack asks the framework to list its own routes, because every one of
them generates routes that appear in no source file. Verified against
fixture apps rather than written from documentation."
```

---

### Task 3: `references/credentials.md`

**Files:**
- Create: `H:\skills\bring-your-own-agent\references\credentials.md`

**Interfaces:**
- Consumes: the auth-shape section of `discovery.md`.
- Produces: per stack, how to obtain a working credential and what the server must send; the CSRF section; `createReadOnlyUser` SQL for Postgres and MySQL. `server.md` relies on the header names defined here.

- [ ] **Step 1: Write the CSRF trap first, because it is the one that bites**

```markdown
## The trap: session cookies and CSRF

Most existing apps authenticate with a session cookie plus a CSRF token, not a
bearer token. A server posting to a Rails or Django endpoint with a valid session
cookie and no CSRF token gets a 403 that explains nothing. **Reads work and
writes fail**, which points suspicion at the write code rather than at auth.

The obvious fix — exempting those routes from CSRF protection — is a real
security regression in somebody's app, arrived at by accident while trying to
make a demo work. **Never do it, and never suggest it.**

Do this instead:

| Stack | Fetch the token | Send it as |
|---|---|---|
| Rails | `GET /` and read `<meta name="csrf-token">` | `X-CSRF-Token` |
| Django | `GET` any page, read the `csrftoken` cookie | `X-CSRFToken` + same cookie |
| Express (csurf) | app-specific; usually a `GET` that sets `_csrf` | `X-CSRF-Token` |
| FastAPI | rarely present; usually token auth | — |

If the app has a token-auth path (Rails `has_secure_token`, Django REST
`TokenAuthentication`, Sanctum), **prefer it** — it sidesteps all of this and is
what the app's own authors intended for non-browser clients.
```

- [ ] **Step 2: Write the per-stack credential sections**

For each stack: where a token is issued, how to create one for this purpose, and how to test it. Rails/Devise, Django (session and DRF token), Express (varies — say so), FastAPI (OAuth2 password flow / API key).

- [ ] **Step 3: Write the read-only database user section**

```markdown
## The read-only database user

The fallback reads SQL directly. That must be enforced by the database, not by
the server's good intentions — a later edit can change code, and cannot grant
itself a permission it does not have.

**Postgres:**

    CREATE USER agent_readonly WITH PASSWORD '<generated>';
    GRANT CONNECT ON DATABASE <db> TO agent_readonly;
    GRANT USAGE ON SCHEMA public TO agent_readonly;
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO agent_readonly;
    ALTER DEFAULT PRIVILEGES IN SCHEMA public
      GRANT SELECT ON TABLES TO agent_readonly;

**MySQL:**

    CREATE USER 'agent_readonly'@'%' IDENTIFIED BY '<generated>';
    GRANT SELECT ON <db>.* TO 'agent_readonly'@'%';

**Prove it, do not assume it.** Connect as that user and attempt a write:

    INSERT INTO <any table> DEFAULT VALUES;

Expected: a permission error. **If that INSERT succeeds, stop.** The fallback is
not safe to enable; use API-only reads and tell the user why.

Where you cannot create a user — a managed database, no admin access — say so
plainly and fall back to API-only reads. Do not use the app's own credentials.
```

- [ ] **Step 4: Write the disclosure text**

The exact words for the one-identity disclosure and the row-visibility disclosure, so the implementation cannot dilute them into a footnote:

```markdown
## What the user must be told, in these words or closer

**When the credential is chosen:**

> This assistant will act as one fixed login. Anything that login can see or
> change, the assistant can see or change too. There is no per-person permission
> screen, and no way to give one person less access than another.

**When the database fallback is chosen:**

> Reading your database directly means the assistant can see rows your app would
> normally hide — anything your app filters in its own code, like "only show a
> customer their own orders", the database does not know about.
```

- [ ] **Step 5: Verify the read-only user against a real database**

Create the user on the `rails-shop` fixture's database, run the INSERT, confirm it is refused.

- [ ] **Step 6: Commit**

```bash
cd /h/skills && git add bring-your-own-agent/references/credentials.md
git commit -m "feat(byoa): credentials, the CSRF trap, and the read-only database user

The CSRF section leads because it is the failure whose obvious fix is a
security regression in somebody's working app. The read-only user is proved
with a refused INSERT rather than trusted."
```

---

### Task 4: `references/server.md`

**Files:**
- Create: `H:\skills\bring-your-own-agent\references\server.md`

**Interfaces:**
- Consumes: header names from `credentials.md`; the capability list agreed at interview step 3.
- Produces: the generated file layout (`agent-access/{server.ts,tools.ts,.env,README.md,package.json}`), the tool-shape rules, and worked examples that Tasks 6 and 8 generate from.

- [ ] **Step 1: State the generated-not-templated rule**

```markdown
## Generated, not copied

The tools carry this app's verbs, parameters and shapes. A single
`call_endpoint(path, method, body)` tool looks generated and is a template in
disguise: it moves every decision to runtime and leaves the user nothing to
review. If you find yourself writing one, the discovery step did not finish.

This file is a specification with worked examples, not a file to copy. Two runs
of this skill should produce servers that differ only where the apps differ.
```

- [ ] **Step 2: Write the worked example, in full**

A complete `agent-access/tools.ts` for the `rails-shop` fixture — `list_orders` with a capped `limit`, `get_order`, `create_order` going through the API with the CSRF header — plus the `server.ts` that registers them over stdio. Real code, runnable.

- [ ] **Step 3: Write the rules that apply to every generated server**

Caps enforced in schema *and* query; no delete tool; every call logged with tool name, arguments and outcome to `agent-access/calls.log`; secrets never returned from a tool (destructure, never spread); read tools marked `readOnlyHint`; writes `destructiveHint: false`.

- [ ] **Step 4: Write the generated README template**

It must record which path each tool uses, that the agent acts as one identity, whether the database fallback is in play, and — per the spec — **that the user was offered the in-app alternative and chose this**, so a later reader knows the trade was deliberate.

- [ ] **Step 5: Verify by generating and running against `rails-shop`**

Boot the fixture, generate the server per this reference, connect with `claude mcp add`, call `list_orders`, confirm real rows. Then `create_order` and confirm the row in the app's own UI.

- [ ] **Step 6: Commit**

```bash
cd /h/skills && git add bring-your-own-agent/references/server.md
git commit -m "feat(byoa): the generated server specification, with a worked example"
```

---

### Task 5: `references/deploy.md`

**Files:**
- Create: `H:\skills\bring-your-own-agent\references\deploy.md`

**Interfaces:**
- Consumes: the stdio server from `server.md`.
- Produces: the HTTP transport variant and its bearer check.

- [ ] **Step 1: Write what changes and what it costs**

Local stdio has no auth because there is no network. The moment it is reachable, a single bearer token grants everything, from anywhere, to whoever holds it. Say that before the instructions, not after.

- [ ] **Step 2: Write the HTTP transport and bearer check**

Real code: constant-time comparison, `POST` only with `GET`/`DELETE` returning 405, no wildcard CORS, the token from an environment variable and never a default.

- [ ] **Step 3: Write the verification block**

`curl` with no token → 401. With a wrong token → 401. With the right one → the tool list. Run them; do not reason about them.

- [ ] **Step 4: Commit**

```bash
cd /h/skills && git add bring-your-own-agent/references/deploy.md
git commit -m "feat(byoa): deploying the server, and the bearer check that then matters"
```

---

### Task 6: `SKILL.md`

Written after the references, so it can point at things that exist.

**Files:**
- Create: `H:\skills\bring-your-own-agent\SKILL.md`

**Interfaces:**
- Consumes: all four references.
- Produces: the skill's frontmatter `description`, which is how any agent finds it.

- [ ] **Step 1: Frontmatter**

```yaml
---
name: bring-your-own-agent
description: Add agent access over MCP to an app that already exists, so tools like Claude can do its real work. Use when someone wants Claude to work with an app they already have, in any stack — Rails, Django, Express, FastAPI — rather than building a new one. Reads the app to find what it can do, agrees a capability list, and generates an MCP server that talks to the app's own API. Adds one folder and edits nothing they wrote.
---
```

- [ ] **Step 2: Ground rules**

Copy the Global Constraints of this plan into the skill's own voice. Include: never edit a file they already had; writes through the API only; no delete tool; never present a partial discovery as complete.

- [ ] **Step 3: Step 0 — the fork**

Verbatim from the spec's *The fork the interview must offer*, including the three triggers and the plain-words script. Plus the three rules: recommend once, record the choice, never decide for them.

- [ ] **Step 4: Steps 1–6**

Which app; what is in it (→ `discovery.md`); what may Claude do (reads and writes agreed separately); how it gets in (→ `credentials.md`); build and verify (→ `server.md`); deploy? (→ `deploy.md`).

- [ ] **Step 5: The verification section**

From the spec: one real read shown raw; one real write confirmed twice — read back through a different tool *and* found by the user in the app's own interface; verification writes cleaned up through the app; tools reported **unverified** with the reason named where no real call is possible.

- [ ] **Step 6: Read it end to end as a user would**

Check: is any step's outcome unclear? Does any sentence assume the reader knows what MCP is before it is explained? Is a technical term used before being introduced once?

- [ ] **Step 7: Commit**

```bash
cd /h/skills && git add bring-your-own-agent/SKILL.md
git commit -m "feat(byoa): the interview, opening with the fork to the better path"
```

---

### Task 7: The front door

**Files:**
- Modify: `H:\skills\start-an-app\SKILL.md` (the existing-project halt)
- Modify: `H:\skills\start-an-app\references\mcp.md` (add a cross-reference)

- [ ] **Step 1: Find the halt**

```bash
cd /h/skills && grep -n "existing\|halt\|stop\|package.json" start-an-app/SKILL.md | head -20
```

- [ ] **Step 2: Add one sentence there**

Not a redirect — an offer. It must name what the other skill does in plain words, so somebody who has never heard of MCP can tell whether they want it.

- [ ] **Step 3: Cross-reference `mcp.md`**

A paragraph near the top: this file is for building agent access *into* a Next.js app you control; `bring-your-own-agent` is for bolting it *beside* an app you would rather not touch. Anyone on the wrong page should know within a paragraph.

- [ ] **Step 4: Commit**

```bash
cd /h/skills && git add start-an-app/SKILL.md start-an-app/references/mcp.md
git commit -m "feat(start-an-app): offer agent access when the folder already holds a project"
```

---

### Task 8: End-to-end, on an app nobody wrote for this

The point of the whole plan. **Task 2's fixtures do not count** — they were built alongside the reference and share its assumptions.

**Files:** none. This task produces findings.

- [ ] **Step 1: Run the skill against `H:\flatbooks`**

Confirm step 0 fires: it is Next.js + Better Auth, so the skill must name the in-app alternative and put the choice to the user. If step 0 does not fire, that is a defect — fix and re-run.

- [ ] **Step 2: Choose this path anyway, and finish the build**

The spec requires the build still works when somebody hears the recommendation and picks this path. Run it through to a connected server and a verified read.

- [ ] **Step 3: Run it against a foreign app**

A real Rails or Django app that was not generated for this. If none is to hand, say so and mark the stack **unverified in the wild** — do not quietly count the fixture as proof.

- [ ] **Step 4: Write findings into the spec**

Append a *What running it found* section. Anything that needed a workaround belongs in a reference file, not in the operator's memory.

- [ ] **Step 5: Commit**

```bash
cd /h/skills && git add docs/specs/2026-08-16-bring-your-own-agent-design.md
git commit -m "docs(byoa): what running the skill against real apps found"
```

---

### Task 9: Publish

**Files:**
- Modify: `H:\skills\README.md`

- [ ] **Step 1: Add the skill to the list**

Match the existing entries' shape, including the `npx skills add brainit-consulting/skills --skill bring-your-own-agent` line.

- [ ] **Step 2: Say what it does not do**

The README is where an honest limitation belongs: one identity, no consent screen, and an app with no API gets reads only. Someone should be able to decide against this skill without installing it.

- [ ] **Step 3: Commit and push**

```bash
cd /h/skills && git add README.md
git commit -m "docs: add bring-your-own-agent to the skill list"
git push origin main
```

---

## Self-Review

**Spec coverage.** Five decisions → Tasks 2 (any stack, discovery), 4 (sits alongside, generated), 5 (local default), 3+4 (API first, read-only fallback), 2 (reads code, confirms). The fork → Task 6 Step 3, tested in Task 8 Step 1. Hard cases → Task 2 Steps 2/4 (implicit routing, no-API), Task 3 Step 1 (auth, CSRF), Task 2 Step 3 (layouts, via detect). Safety rules → Task 4 Step 3 and the Global Constraints. Verification → Task 6 Step 5. Validation → Task 8. Reference files → Tasks 2–5. **No gaps found.**

**Placeholders.** Task 3 Step 2 and Task 4 Steps 2/4 describe content rather than showing it in full — deliberate, because that content is long and app-specific, and each names exactly what must appear and carries a verification step that fails without it. Every other step contains its actual content. No "TBD", no "similar to Task N".

**Naming consistency.** `agent-access/` as the generated folder throughout; `bring-your-own-agent` as the skill everywhere; `list_orders` / `get_order` / `create_order` consistent between Tasks 4 and 8; `agent_readonly` in Task 3 only.

**One risk worth stating.** Task 1 Step 1 may find Ruby or Python absent. The plan's answer is Docker or an explicitly deferred stack — **never** a discovery section written from documentation. A stack shipped that way is exactly the promise the spec says the skill cannot keep.
