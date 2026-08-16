# Route discovery

Last verified: 2026-08-16

**Purpose:** Work out what an existing app can actually be asked to do over HTTP — which URLs exist, which of them return data rather than a web page, and how the app decides who is allowed in — without editing a single file the owner wrote.

Everything here was run against real apps. Every block of output below is pasted from a terminal, not paraphrased.

> **Shell:** the commands assume a POSIX shell — `test -f`, `grep -E`, and single-quoted `curl -w` format strings. On Windows run them in Git Bash or WSL rather than PowerShell. The one long exception is Django's `manage.py shell -c` block below, which was confirmed to run unchanged in PowerShell 7.

## Ask the framework, do not guess

Every stack here generates routes that appear in no source file.

A Django REST Framework router turns one line — `router.register(r"orders", OrderViewSet)` — into **six** URL patterns. FastAPI's `include_router` rewrites every path in the router with a prefix, so an endpoint whose source says `@router.get("")` is served at `/api/orders` and **nothing** answers at `/orders`. Django's `path('admin/', admin.site.urls)` is one line that becomes twenty-three routes. Not one of those URLs appears as a string anywhere in the project.

**A grep for literal paths finds none of them and reports an app with no API.** That is not a small error. The owner then chooses a read-only integration, or chooses this skill over a better one, on the strength of a list they believe is complete. "I found three endpoints" when the framework generated forty is worse than finding nothing, because nobody goes looking for the other thirty-seven.

So: run the command that makes the framework list its own routes. Inference is the fallback for an app that will not boot, and when you fall back you **say the list was inferred** — in the same breath as giving it, not in a footnote.

## Then probe what you found

Enumerating is half the job. The framework tells you what it *registered*, not what *works*, and the gap between the two is real:

- **A registered route can be broken.** DRF's router generates format-suffix routes (`/api/orders.json`) whether or not the view accepts a `format` argument. On the Django fixture the router enumerated **six** routes and only **four** answer: `/api/orders.json` and `/api/orders/1.json` both return **HTTP 500** with a valid token — `TypeError: OrderViewSet.list() got an unexpected keyword argument 'format'` — while `/api/orders/` and `/api/orders/1/` return 200. All six were in the enumerated list. Hand all six to an agent and a third of its tools fail every time they are used, with a stack trace that blames the app's own code and gives no hint that the route was auto-generated and never worked.
- **A route that exists can answer `405`, and dropping it is the expensive mistake.** A probe is a request, and a request has a verb — so the natural probe, a `GET`, asks a POST-only endpoint a question it does not answer. Measured on `fastapi-shop`, with no credentials:

  ```
  $ curl -s -o /dev/null -w '%{http_code}\n' -X DELETE http://127.0.0.1:8001/api/orders
  405
  $ curl -s -o /dev/null -w '%{http_code}\n' -X DELETE http://127.0.0.1:8001/api/nonexistent
  404
  $ curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8001/token
  405
  ```

  **404 means there is nothing there. 405 means the route is there and your verb is wrong** — the opposite of a reason to discard it. That third line is the one to look at: `/token` is this app's login endpoint, the route `credentials.md` needs before anything else works, and a `GET` probe reports it as a failure. On the Next.js app this skill was validated against, **eleven of eighteen** route handlers export `POST` and nothing else; a rule that drops 405s drops all eleven, and the owner never learns they existed.

  Two things measured alongside it, both of which mislead:

  - **The `Allow` header is a hint, not the verb list.** The 405 above carried `allow: GET`, while `POST /api/orders` on the same path answers perfectly well (401 without credentials, 422 with a token and an empty body). Starlette answered from the first route registered on that path. Read it, then confirm against the enumeration.
  - **Some stacks check auth before the method, so you never see the 405.** `DELETE /api/orders/` on `django_api` returns **401** with no credentials and **405** — `Allow: GET, POST, HEAD, OPTIONS`, complete this time — with a valid token. On DRF the verb question is answered only after the credential question, so an unauthenticated probe leaves it open. Re-probe once you have a credential, and do not read the earlier 401 as the final word on what the route does.

  So record the status per route rather than a yes/no. Only a 404, and a route that is genuinely broken, are grounds for dropping something from the list — and "broken" does not oblige you to try every verb: the DRF format-suffix case above 500s on the verb its own enumeration advertised, which settles it.

- **A trailing slash is not cosmetic, and the stacks disagree in opposite directions.** Measured on the fixtures:

  | Request | Django + DRF (:8000) | FastAPI (:8001) | Express (:3001) |
  |---|---|---|---|
  | `/api/orders/` (with slash) | **200** | 307 redirect | — |
  | `/api/orders` (no slash) | 301 redirect | **200** | — |
  | `/users` / `/users/` | — | — | **200** both (with the key; 401 both without) |

  A client that does not follow redirects sees a 301 or a 307 and reports failure. A client that *does* follow them may drop the body on the way, so a POST silently becomes a GET.

So after enumerating, send one request to each route and **write down the status you got**, per route, rather than a yes or a no. A route list is a claim; a status code is evidence — and each code says something different, which is the point of keeping it.

---

## The routes may not be the app

Everything above answers *which URLs exist and work*. It does not answer the question the owner is actually asking, which is **how much of what their app does can be reached over HTTP at all**. Those are not the same question, and on some apps the answer to the second is "almost none" while every check in this file returns the reassuring answer.

This is not hypothetical and it is not a Next.js problem, though that is where it was met. The validated case: a Next.js accounting app with **18** route handlers, of which three were of any use to an assistant — the rest being webhooks with signature checks, cron endpoints behind a shared secret, and browser-only flows. The app's actual work — create a client, raise an invoice, send it, mark it paid, add a transaction, import a statement, change settings — lives in **25 Server Actions**, which have no URL a program can call. The honest capability list was **two tools, against twenty-five Server Actions and three usable routes** — and the app passed every honesty check here on the way to that answer: it has an API, the API returns JSON, the routes enumerate, and they answer.

**So the rule *never present a partial discovery as a complete one* can be broken while following every instruction in this file** — by reporting the routes truthfully and letting the owner infer that the routes are the app.

**How to notice it, rather than hope to.** Three moves, none of which takes long:

1. **Get the list of capabilities from the owner before you look at the code.** Ask what they *do* in the app in a normal week, in their words — "raise an invoice", "chase a late payer", "reconcile the bank" — and write the list down before discovery can bias it. Then map each item to a route you probed. **The unmapped ones are the finding.** Done in the other order, the routes suggest the capabilities and the missing ones are invisible, because nothing on the screen is where they would have been.
2. **Count the other shape, whatever it is on this stack.** Work that has no URL still has to live somewhere, and every stack has a place for it:

   | Stack | Where work hides from HTTP | How to count it |
   |---|---|---|
   | Next.js App Router | Server Actions | `.next/server/server-reference-manifest.json`, or files marked `'use server'` |
   | Django / Rails / Laravel | form-post controllers that render HTML | routes whose response is `text/html`, from the probe above |
   | Any stack | a single GraphQL endpoint | one route, many operations — count the schema's mutations, not the route |
   | Any stack | admin-only interfaces, management commands, background jobs | they are in the repo and not in the route list |

3. **Report both numbers side by side**, in the discovery report, in the owner's words: *"Your app does about twenty-eight things. Three of them have a web address a program can call, and two are worth a tool."* One number is a route count and reads as progress; two numbers are a ratio and read as a decision.

That ratio is also the strongest argument the skill has for the better path. "Building it inside your app would reach the twenty-five this cannot" is a sentence with a number in it, and it is the difference between an owner choosing this path knowingly and choosing it because nobody counted.

**Where the ratio is bad, say so at interview step 2**, in the same breath as the no-API case and for the same reason: at step 5 the owner has already chosen this path over a better one, on the strength of a list that was true and incomplete.

---

## Django

**Detect:** a `manage.py` file at the project root. It is the one file `django-admin startproject` always writes and nothing else uses the name. Confirm with the import:

```bash
test -f manage.py && python -c "import django; print(django.get_version())"
```

Observed on the fixture: `6.1`.

**Enumerate:** the documented tool is `manage.py show_urls`, and it will usually not be there. It comes from **`django-extensions`**, a third-party package that most projects do not install. On the fixture:

```
$ python manage.py show_urls
Unknown command: 'show_urls'
Type 'manage.py help' for usage.
```

That is the common case, not the exception, so the answer cannot be "install django-extensions" — the whole point of this skill is that you change nothing in someone else's app. Django will list its own URLconf without any extra package, because the resolver is a public API. Walk it:

```bash
python manage.py shell -c "
from django.urls import get_resolver
from django.urls.resolvers import URLResolver

def walk(patterns, prefix=''):
    for p in patterns:
        if isinstance(p, URLResolver):
            walk(p.url_patterns, prefix + str(p.pattern))
        else:
            actions = getattr(p.callback, 'actions', None)
            verbs = ','.join(sorted(m.upper() for m in actions)) if actions else '-'
            print(verbs.ljust(20), '/' + prefix + str(p.pattern), p.name)

walk(get_resolver().url_patterns)
"
```

Real output from `django_shop`:

```
14 objects imported automatically (use -v 2 for details).

GET,POST             /api/^orders/$ order-list
GET,POST             /api/^orders\.(?P<format>[a-z0-9]+)/?$ order-list
GET                  /api/^orders/(?P<pk>[^/.]+)/$ order-detail
GET                  /api/^orders/(?P<pk>[^/.]+)\.(?P<format>[a-z0-9]+)/?$ order-detail
-                    /api/ api-root
-                    /api/<drf_format_suffix:format> api-root
```

Six routes from a `urls.py` whose only route entry is `urlpatterns = [path("api/", include(router.urls))]`. Not one of the six URLs appears as a string in that file, or anywhere else in the project. Reading it would have found none of them.

Three things about the output, each of which has misled someone:

- **`^` and `$` are regex anchors, not part of the URL.** DRF registers its routes with `re_path`, so the pattern prints raw. The first line means `/api/orders/` — with the leading `^` stripped and the trailing `$` stripped.
- **`-` in the verbs column means Django does not know.** It is not "no methods". The column is populated from `callback.actions`, which only DRF viewsets set; a plain Django view accepts whatever its body checks with `if request.method ==`. So `-` means *read the view*, not *skip this route*.
- **The `14 objects imported automatically` line is Django's shell banner**, printed on stdout above the results. Ignore it; do not let a script parse it as a route.

**A second route, if DRF is in use and the app is running: ask the router.** A `DefaultRouter` serves an API root listing every collection registered on it. Against the running fixture:

```bash
$ curl -s -H "Accept: application/json" http://127.0.0.1:8000/api/
{"orders":"http://127.0.0.1:8000/api/orders/"}
```

That is a cross-check on the enumeration and a useful sanity test that you found the right mount prefix. Two limits: it lists **collections, not routes** — one line for `orders`, not the six URLs the router generated — so it never replaces the walk. And it is a `DefaultRouter` feature; a `SimpleRouter` does not serve one, so a 404 there is not evidence of no API.

Worth flagging to the owner: on this fixture `/api/` returns **200 with no credentials** while `/api/orders/` returns 401. The list of collections is public even though the data is not. That is DRF's default and they may not know it.

**Fallback when it will not boot:** the walk loads the settings module and every app in `INSTALLED_APPS`, so a missing dependency or a required environment variable stops it. You will see a Python traceback instead of a route list — `ModuleNotFoundError: No module named 'rest_framework'` and its kind — rather than a message about routes.

Then read `ROOT_URLCONF` (named in `settings.py`) and expand by rule. Verified against the fixture, a `DefaultRouter` registration of `r"<prefix>"` yields six patterns plus the router root:

| Pattern | Verbs |
|---|---|
| `/<prefix>/` | GET → `list`, POST → `create` |
| `/<prefix>.<format>` | the same verbs |
| `/<prefix>/<pk>/` | GET → `retrieve`, PUT → `update`, PATCH → `partial_update`, DELETE → `destroy` |
| `/<prefix>/<pk>.<format>` | the same verbs |
| the router's own root | GET |
| the root's format suffix | GET |

**A verb only exists if the class defines the matching method.** The fixture's `OrderViewSet` defines `list`, `retrieve` and `create` and nothing else, and the enumeration shows exactly `GET,POST` on the collection and `GET` alone on the detail route — no PUT, no DELETE. Do not assume seven CRUD routes because the class is called a ViewSet.

`include(...)` prefixes everything below it, so an inferred path is the concatenation of every `path()` above it. The rule above does not cover `@action`-decorated methods, which add further routes. Say the list was **inferred**, say it is likely to be incomplete, and probe every entry before promising it.

**Is there an API at all?** One measurable check, no server required:

```bash
grep -n "rest_framework" <project>/settings.py
```

**Find `settings.py` rather than guessing at it.** On an unfamiliar app the project package is not the repo name and there may be several `settings*.py` files. `manage.py` names the real one:

```bash
grep -n "DJANGO_SETTINGS_MODULE" manage.py
```

Observed on the fixture: `os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'django_shop.settings')` — so the file is `django_shop/settings.py`. Dots are directories; the last segment is the filename.

`django_shop` matches (`'rest_framework'` is in `INSTALLED_APPS`); `django_html` returns nothing. No DRF, no Ninja, no Tastypie in `INSTALLED_APPS` and no view returning `JsonResponse` means the app renders templates, which is the no-API case below.

With the app running, the decisive check is the response's content type — ask for JSON and see what comes back:

```bash
curl -s -o /dev/null -w '%{http_code} %{content_type}\n' -H "Accept: application/json" <url>
```

`django_shop`'s `/api/orders/` answers `200 application/json`. `django_html`'s `/invoices/`, requested with a valid session cookie and the same `Accept` header, answers `200 text/html; charset=utf-8` and a full HTML document. The header is a request, not a command; a template app ignores it.

**Auth shape:** what you can tell before asking the owner anything.

- `rest_framework.authtoken` in `INSTALLED_APPS`, or `TokenAuthentication` in a view's `authentication_classes`, means a token in an `Authorization` header — and DRF's own scheme word is **`Token`**, not `Bearer`. Sending `Bearer` against `TokenAuthentication` fails as though the token were wrong.
- `SessionAuthentication`, `@login_required`, or `LoginView` in the URLconf means a cookie obtained by posting a login form — and then Django's CSRF middleware guards every unsafe method as well.
- `permission_classes = [IsAuthenticated]` on a view tells you the route is gated even when the enumeration does not.

Measure it rather than reading it: request the route with no credentials. `django_shop`'s `/api/orders/` returns **401** unauthenticated and **200** with `Authorization: Token <key>`. Hand the mechanism to `credentials.md` — obtaining, storing and sending the credential is its job, not this file's.

---

## Express

**Detect:** a `package.json` listing `express` as a dependency.

```bash
node -e "const p=require('./package.json');console.log(p.dependencies&&p.dependencies.express)"
```

Observed on the fixture: `~4.16.1`. Also check the installed version, which is what actually matters below:

```bash
node -e "console.log(require('express/package.json').version)"
```

Observed: `4.16.4`.

**Enumerate: read the route files.** This is the honest answer for Express, and it is a real conclusion rather than laziness — runtime introspection of the router stack was tried on the fixture and it is worse. Read two things:

1. `app.js` (or `server.js`, `index.js`) for the `app.use(...)` mounts, which supply the path prefixes.
2. Each mounted router file for its `router.get/post/put/delete` calls, which supply the leaves.

```bash
grep -rEn "\.(get|post|put|patch|delete|all|use|route)\(" app.js routes/
```

Real output from `express-shop`:

```
app.js:16:app.use(logger('dev'));
app.js:17:app.use(express.json());
app.js:18:app.use(express.urlencoded({ extended: false }));
app.js:19:app.use(cookieParser());
app.js:20:app.use(express.static(path.join(__dirname, 'public')));
app.js:25:  if (req.get('X-API-Key') !== API_KEY) {
app.js:31:app.use('/', indexRouter);
app.js:32:app.use('/users', requireApiKey, usersRouter);
app.js:35:app.use(function(req, res, next) {
app.js:40:app.use(function(err, req, res, next) {
app.js:43:  res.locals.error = req.app.get('env') === 'development' ? err : {};
routes/index.js:5:router.get('/', function(req, res, next) {
routes/users.js:5:router.get('/', function(req, res, next) {
```

Thirteen lines, of which **two** are mounts and **two** are routes. Sorting them out:

- **A mount is an `app.use` whose first argument is a path string.** Lines 31 and 32 take `'/'` and `'/users'`. Lines 16–20, 35 and 40 take a function first and are middleware — they run on everything and route nothing.
- **Lines 25 and 43 are neither.** `req.get('X-API-Key')` and `req.app.get('env')` are Express's *header* and *setting* accessors, which share the name `get` with the routing verb. The pattern over-matches on purpose; see below.

Concatenate mount and leaf: `GET /` and `GET /users`. Both confirmed live — 200 and 200.

**Why the pattern is that wide, and why the obvious narrower one is dangerous.** The natural first attempt is to look for `app.use` and `router.get`. Both halves of that are wrong:

- **Routes are commonly registered straight on the app object.** `app.get('/health', …)`, `app.post('/webhooks/stripe', …)`, `app.all(…)` and `app.route('/legacy').get(…).delete(…)` are ordinary Express and match no pattern built around `router.`.
- **`router` is a convention, not a rule.** Nothing requires the variable to be called that. `const api = express.Router(); api.get('/things', …)` is just as valid, and `api.get` matches nothing anchored to the word `router`.

Measured, against a scratch file registering six routes in those shapes plus one `app.use('/api', api)` mount, the narrow pattern returned **only the two `app.use` lines — zero routes**. Against two lines of an ordinary JSON API —

```js
app.get('/orders', (req, res) => res.status(200).json(orders));
app.post('/orders', (req, res) => res.status(201).json(created));
```

— it returned **nothing at all**, so the app would have been reported as having no routes. The wide pattern finds all six in the first case and both in the second.

That asymmetry is the whole point: **an over-match is a line you read and discard in two seconds; an under-match is an endpoint the owner never learns exists.** `req.get` in the output above costs nothing. So widen first and prune by eye, never the reverse.

Two gaps the pattern still has, both of which the eye closes:

- **`app.route('/path')` chains give up the path but not the verbs** when the `.get(…)` / `.delete(…)` links sit on their own continuation lines. The pattern flags the `app.route(` line; read the lines under it for the verbs.
- **A path built from a variable** (`app.get(BASE + '/things', …)`) shows up as a route with no readable URL. Read the variable.

**Why not introspect the router stack.** `app._router.stack` does exist and can be walked, but three things observed on this fixture rule it out as the primary method:

- **The accessor is version-dependent, and the wrong one throws rather than returning nothing.** On Express 4.16.4 the working property is `app._router`; reading `app.router` raises `Error: 'app.router' is deprecated!` and exits the process. Express 5 removes `_router` and makes `router` the supported name. A defensive `app._router || app.router` does not help — it crashes on Express 4 while evaluating the fallback it never needed.
- **Mount paths are stored only as compiled regexes**, so you rebuild the URL by un-escaping one. The layer for `/users` holds `^\/users\/?(?=\/|$)`; the layer for `/` holds `^\/?(?=\/|$)`. An un-escaping attempt written for this file produced `/?(?=/|$)/` as the app root — wrong, and wrong in a way that still looks like a route, so nothing downstream flags it.
- **Most of the stack is not routes.** The fixture's stack has twelve entries and only **two** are mounted routers. The rest are `query`, `expressInit`, `logger`, `jsonParser`, `urlencodedParser`, `cookieParser`, `serveStatic`, the `requireApiKey` guard, and two anonymous error handlers.

Introspection earns its place as a **cross-check** when the files leave something genuinely unresolved — routers mounted in a loop, paths built from variables, or an `app.route()` chain whose verbs you want confirmed rather than read off. Then walk `app._router.stack`, pin the Express major version first, and treat the result as a second opinion rather than the answer. Note that `require('./app')` on an express-generator layout does not bind a port: `app.js` exports the app and `bin/www` calls `listen`. So you can introspect without starting a server.

**Fallback when it will not boot:** reading the files *is* the method, so a broken app costs you nothing here — which is Express's one real advantage over the other two. What it costs you is the probe. Without a running server you cannot confirm the routes answer, so the list is **inferred** and must be announced as inferred, exactly like the Django and FastAPI fallbacks.

**Is there an API at all?** Routes are not an API, and Express is where that bites, because nothing in the framework distinguishes the two. Grep for what the handlers return:

```bash
grep -rEn "\.(json|send|sendStatus|render)\(" app.js routes/
```

Real output from `express-shop`:

```
app.js:17:app.use(express.json());
app.js:26:    return res.status(401).json({ error: 'missing or invalid X-API-Key' });
app.js:47:  res.render('error');
routes/index.js:6:  res.render('index', { title: 'Express' });
routes/users.js:6:  res.send('respond with a resource');
```

**Match `.json(`, never `res.json`.** A chained status call — `res.status(200).json(orders)`, one of the commonest ways to write an Express handler — does not contain the string `res.json`. Measured: against the two lines

```js
app.get('/orders', (req, res) => res.status(200).json(orders));
app.post('/orders', (req, res) => res.status(201).json(created));
```

a `res.json` grep returns **no matches**, and the rule below then reads "no JSON responses, therefore no API" for an app that is nothing but a JSON API. The owner is routed to "reads work, writes are impossible" for an app whose writes are fine — the exact misclassification the no-API rule exists to prevent, produced by the check meant to prevent it. No fixture will catch this for you: `express-shop` contains not one `app.get`, and a `res.json` grep hides even the fixture's own `res.status(401).json(...)` on `app.js:26`. Both Express patterns here had to be checked against files written specifically to break them.

Reading the corrected output: three of the five lines are `render` or `send` returning HTML and plain text; line 17 is middleware, not a response; line 26 is a genuine JSON body, but it is the **401 from the guard**, not a route's payload. So neither of the two routes returns JSON. Confirmed against the running app: both answer `200` with `Content-Type: text/html; charset=utf-8`, even when asked for `application/json`. This app has two working routes and no JSON API — the no-API case below, arrived at from a completely different direction from Django's.

Three traps in that check, each visible in the output above:

- **`app.use(express.json())` is not evidence of an API.** It parses *incoming* request bodies and says nothing about what goes out. That is line 17, and this app returns HTML from every route.
- **A JSON error body is not a JSON API.** Line 26 is real JSON, and it tells you only that a guard exists. Count a line as evidence of an API only when it is the response of a route you enumerated.
- **`res.send` is content-negotiated by argument type.** `res.send('a string')` sets `text/html`; `res.send({...})` sets `application/json`. Reading the call is not enough — check what the argument is, or just probe the route.

Which is why the grep is only ever the first pass. The check that settles it is the content type of a real response, measured.

**Auth shape:** in Express, auth is a middleware function, and **it can guard part of an app and not the rest**. The fixture's mounts show it plainly:

```
app.use('/', indexRouter);
app.use('/users', requireApiKey, usersRouter);
```

The extra argument on line 32 is the guard. Measured live: `GET /` returns **200** with no credentials; `GET /users` returns **401** with body `{"error":"missing or invalid X-API-Key"}`, and **200** with the header `X-API-Key: <key>`.

So never conclude "this app needs no auth" from one unauthenticated 200 — probe **every** route without credentials and record the status per route. A guard can also sit inside a router file rather than on the mount, and an anonymous inline function gives you no name to grep for; the status codes find it either way. Then hand the mechanism — a custom header here, so neither `Token` nor `Bearer` — to `credentials.md`.

---

## FastAPI

**Detect:** a Python file constructing the app object.

```bash
grep -rln "FastAPI(" --include=*.py .
```

Observed on the fixture: `./main.py`. The variable it is assigned to is the one you import next; here `app = FastAPI()` in `main.py`, so the import path is `main:app` — the same string in the `uvicorn` command that starts it.

**Enumerate: generate the OpenAPI schema in process.** FastAPI builds it from the live route table, so the prefixes are already applied:

```bash
python -c "
from main import app
for path, ops in app.openapi()['paths'].items():
    for method in ops:
        print(method.upper().ljust(7), path)
"
```

Real output from `fastapi-shop`:

```
POST    /token
GET     /api/orders
POST    /api/orders
GET     /api/orders/{order_id}
```

The source for those three order endpoints says `@router.get("")`, `@router.get("/{order_id}")` and `@router.post("")` — no `/api`, no `/orders`. The prefix comes from `APIRouter(prefix="/api/orders")` and is applied by `app.include_router(router)` on the last line of the file. **This is not cosmetic: `GET /orders` on the running app returns 404 and only `GET /api/orders` returns 200.** Enumerating from the decorators would have produced three URLs that do not exist.

**Do not iterate `app.routes` directly.** The obvious snippet —

```python
for r in app.routes:
    print(getattr(r, 'methods', None), r.path)      # do not use
```

— fails on current versions, and fails in the worst possible way. On FastAPI 0.141.1 with Starlette 1.6.0, `include_router` leaves an `_IncludedRouter` object in `app.routes` rather than flattening the router's routes into it, and that object has no `.path`:

```
{'HEAD', 'GET'} /openapi.json
{'HEAD', 'GET'} /docs
{'HEAD', 'GET'} /docs/oauth2-redirect
{'HEAD', 'GET'} /redoc
{'POST'} /token
Traceback (most recent call last):
  File "<string>", line 4, in <module>
    print(getattr(r, 'methods', None), r.path)
                                       ^^^^^^
AttributeError: '_IncludedRouter' object has no attribute 'path'
```

(`.methods` is a **set**, so the two verbs print in either order from run to run. Run it without `python -u` and the traceback may appear above the routes rather than below, because stderr is unbuffered and stdout is not.)

It printed four documentation routes and one login endpoint, reported **none** of the app's three actual endpoints, and then crashed. Wrap that loop in a `try` — as a tidy-looking script would — and you get a clean list of five routes with no API in it. `app.openapi()` is a supported public method and returns the flattened table.

**One condition on `app.openapi()`:** routes registered with `include_in_schema=False` are deliberately absent. The fixture demonstrates it — `/openapi.json`, `/docs`, `/docs/oauth2-redirect` and `/redoc` all carry `include_in_schema=False` and none of them appear in the output above, while `/token` carries `include_in_schema=True` and does. Those four are FastAPI's own documentation routes and their absence is correct, but an application route can be hidden the same way. If the owner mentions an endpoint the schema does not list, this is why.

**A second, independent route: ask the running app.** FastAPI serves the same schema over HTTP at `/openapi.json`:

```bash
curl -s http://<host>/openapi.json | python -c "
import sys, json
for p, ops in json.load(sys.stdin)['paths'].items():
    for m in ops: print(m.upper().ljust(7), p)"
```

Against the running fixture this returns exactly the four lines above. Two conditions: **the app must be running**, and the endpoint can be switched off — an app constructed as `FastAPI(openapi_url=None)` returns 404 there, and its `/docs` goes with it. Many production deployments do exactly that. A 404 at `/openapi.json` is therefore not evidence of no API; fall back to the in-process method.

Worth knowing before you rely on it: on this fixture `/openapi.json` returns **200 without credentials** while every `/api/orders` route returns **401**. The route list is public even though the data is not. That is normal and it is useful to you — but say so to the owner, because it is the kind of thing they may not have realised is exposed.

**Fallback when it will not boot:** `from main import app` executes the module, so a missing dependency, a database connection at import time or an absent environment variable stops it with a traceback. Then read the source and expand by rule: every `APIRouter(prefix=...)` prefixes every decorated path in that file, and `include_router(router, prefix=...)` adds a **further** prefix on top of the router's own. Both places must be checked — the prefix can live at either end, and an app that sets it in both concatenates them. An empty decorator path (`@router.get("")`) means the prefix itself is the route. Announce the list as **inferred** and probe it before promising it, since a prefix missed at either end makes every URL in the list a 404.

**Is there an API at all?** For FastAPI the question barely arises — it is a JSON framework — but do not skip the check, because a FastAPI app can serve HTML through `HTMLResponse`, a `response_class`, or a mounted template engine. Make the schema answer it: it declares the content type of every response.

```bash
python -c "
from main import app
for path, ops in app.openapi()['paths'].items():
    for method, op in ops.items():
        content = op.get('responses', {}).get('200', {}).get('content', {})
        print(method.upper().ljust(7), path.ljust(24), list(content) or '(no body)')
"
```

Real output from `fastapi-shop`:

```
POST    /token                   ['application/json']
GET     /api/orders              ['application/json']
POST    /api/orders              ['application/json']
GET     /api/orders/{order_id}   ['application/json']
```

Every route declares `application/json`, so all four are the API. A route declaring `text/html` is the no-API case for that route. Cross-check against the source for the escape hatches:

```bash
grep -rEn "response_class|HTMLResponse|Jinja2Templates|StaticFiles" --include=*.py .
```

On the fixture this returns nothing, which agrees with the schema. Where the two disagree, believe the schema — it is generated from the live route table, and the grep is only reading text.

**Auth shape:** the dependency names in the source tell you the scheme.

- `OAuth2PasswordBearer` means a **`Bearer`** token, obtained by posting form data to the `tokenUrl` named in its constructor. The fixture's `OAuth2PasswordBearer(tokenUrl="token")` matches the `POST /token` in the enumeration — the login endpoint is *in* the route list, which is a useful confirmation that you read the scheme right.
- `HTTPBearer` means a bearer token the app does not issue.
- `APIKeyHeader` / `APIKeyQuery` name the header or parameter directly in their constructor.

The schema also carries this: a `securitySchemes` block under `components` in `app.openapi()`. Measure it either way — `GET /api/orders` returns **401** with no credentials and **200** with `Authorization: Bearer <token>`. Hand the mechanism, including the token endpoint and the form field names it expects, to `credentials.md`.

---

## Next.js App Router

**Read this section knowing it rests on one app.** Django, Express and FastAPI each have a fixture here, and their commands were then run against files written specifically to break them. This section was written from a single real Next.js 16 application met during validation — an accounting app nobody wrote for this skill — and it has not been run against a second one. Every count and every status below was measured on that app; treat them as real, and treat the commands as less battle-tested than the other three. `## Not yet supported` below explains why that distinction is worth making rather than glossing.

It is here rather than in that list because `SKILL.md`'s Step 0 names Next.js with Better Auth as its strongest trigger for recommending the *other* path — so a Next.js owner is the likeliest person to read this file, and some of them will choose this path anyway.

**Detect:** `next` in the dependencies.

```bash
node -e "const p=require('./package.json');console.log(p.dependencies&&p.dependencies.next)"
```

Observed: `16.2.6`. Then tell the two routers apart, because they route differently and only one is covered here: an `app/` or `src/app/` directory containing `layout.tsx` is the **App Router**, which this section is about. A `pages/` directory is the older **Pages Router** — not checked here, and it should be said to be uncovered rather than guessed at.

**Enumerate: read the build's own route manifest.** Next.js writes `.next/app-path-routes-manifest.json` — source path to URL, for every route it serves. That is the Next.js equivalent of `manage.py shell` and `app.openapi()`, and like them it is the framework's own answer rather than yours:

```bash
python -c "
import json
m = json.load(open('.next/app-path-routes-manifest.json'))
for src, url in sorted(m.items(), key=lambda kv: kv[1]):
    print(url.ljust(30), src)
"
```

Real output, a slice of the 49 entries:

```
/                              /page
/admin                         /(app)/admin/page
/admin/users                   /(app)/admin/users/page
/api/auth/[...all]             /api/auth/[...all]/route
/api/exports/ledger            /api/exports/ledger/route
/api/integrations/ingest       /api/integrations/ingest/route
/icon.svg                      /icon.svg/route
/invoices                      /(app)/invoices/page
/invoices/[id]                 /(app)/invoices/[id]/page
/login                         /(auth)/login/page
```

Four things about that output, and the first is the reason to run it at all.

- **The file tree lies about the URL, and this is where "a route is a file" stops being true.** Directories in brackets are **route groups**: they organise the source and are stripped from the URL. `/(app)/invoices/page` is served at `/invoices`, and a path built by reading the tree gives `/(app)/invoices`, which is a 404. On this app **17 of the 49 entries** sit inside a group — `(app)` wraps 13 and `(auth)` wraps 4 — so a third of the app is at an address the tree does not give you. `[id]` and `[...all]` are the opposite case: those brackets *are* part of the URL, as parameters.
- **Methods are not in the manifest**, so read which of `GET`/`POST`/… each `route.ts` exports — the same instruction Django's `-` verbs column carries. **Match on `export`, not on `export function`**, for the reason the Express section gives at length:

  ```bash
  grep -rlE "^export .*(GET|POST|PUT|PATCH|DELETE|HEAD|OPTIONS)" src/app --include=route.ts | wc -l
  ```

  Count **files** (`-rlE … | wc -l`), then read them with `-rEn` when you want the lines: two verbs can share a file, so the line count and the handler count are different numbers and only one of them is the size of the API. Measured on this app, the wide pattern finds all **18** route handlers (19 matching lines) and the narrow `export (async )?function (GET|POST|…)` finds **17**. The one it misses:

  ```
  src/app/api/auth/[...all]/route.ts:4:export const { GET, POST } = toNextJsHandler(auth);
  ```

  A destructured re-export is how every Better Auth and NextAuth app mounts its catch-all — so the route the narrow pattern loses is the **login endpoint**, the one `credentials.md` needs before anything else can work, and it is lost in silence.
- **Not every entry is a file you can read.** `/icon.svg/route` is `src/app/icon.svg`, a metadata file Next.js serves as a route. Looking for a `route.ts` behind it wastes ten minutes; there is not one, and nothing says so.
- **The manifest is written by `next build`, and it describes that build rather than the working tree.** A route added since is simply absent, so check the dates rather than assume them:

  ```bash
  stat -c '%y %n' .next/app-path-routes-manifest.json
  find src -name '*.ts*' -printf '%T@ %TY-%Tm-%Td %TH:%TM %p\n' | sort -rn | head -1
  ```

  Observed on this app: the manifest at `2026-07-16 07:17` against a newest source file at `07:05` — built after the last edit, so the list is current. The other way round means the list is stale and must be announced as such.

  **A dev server is not a substitute, and this is the part to be careful about.** A dev server ran on this app on **16 August**, a month after that build, and left the file above untouched at its **July** timestamp. What it wrote instead was a separate tree under `.next/dev/`, and a `find` across the whole of `.next` returns exactly **one** `app-path-routes-manifest.json` — the build's. Where the dev tree does carry an equivalent, it holds a fraction of the app:

  | | `next build` output | `.next/dev/` after a dev session |
  |---|---|---|
  | `app-path-routes-manifest.json` | present, 49 entries | **absent** |
  | `server/app-paths-manifest.json` | 49 entries | **3** |
  | `server/server-reference-manifest.json` | 25 actions | **1** |

  The 3 and the 1 are the routes and the action that happened to be visited in that session. The counts and the timestamps are measured; the explanation — that a dev server compiles routes on demand, so its manifests describe what was opened rather than what exists — is **inferred from that layout and not from reading Next.js's source or re-running it**. Either way the instruction is the same and does not depend on the cause: **enumerate from a `next build` manifest, never from `.next/dev/`.** A dev-time list that under-reports 49 as 3 is the exact failure this whole file exists to prevent, arrived at by trusting the wrong artefact.

**Fallback when there is no `.next/`:** the manifest is build output, so a repository that has been cloned and never built has none. There are two honest moves and neither is "run their build yourself" — building writes into their working tree and can need a database, environment variables and a licence you were not given.

1. **Ask the owner to run `npm run build` once.** Their command, in their app, and `.next/` is in the gitignore every Next.js template ships with — check theirs and say so — so this changes nothing they track. It is much the better option: it gives you the real list. **Ask for the build specifically, not `npm run dev`**, for the reason measured above.
2. **Otherwise read the tree and expand by rule**, and say the list is **inferred**: every `route.ts` under `app/` (or `src/app/`) is an endpoint, every `page.tsx` is a page, the path is the directory path with **every `(group)` segment removed**, `[param]` kept as-is, and `route.ts` / `page.tsx` dropped from the end. That reproduces the manifest's URL for every hand-written route on the app above — it will not give you the entries Next.js generates for itself (`/_not-found`, an `icon.svg`), and it will not survive anything exotic — `basePath` or `rewrites` in `next.config`, a `middleware.ts` / `proxy.ts` rewriting paths (see the auth section below for the rename), or `generateStaticParams`.

**Is there an API at all?** Next.js is where this question needs asking most, and where the usual form of it gives the wrong answer. A `route.ts` is a JSON endpoint, so an app with 18 of them plainly "has an API" — and on this app that was true and nearly useless, because the app's actual work lives in Server Actions with no URL. Ask both halves:

```bash
# Half one: how many route handlers are reachable over HTTP?
grep -rlE "^export .*(GET|POST|PUT|PATCH|DELETE)" src/app --include=route.ts | wc -l

# Half two: how much is not?
python -c "
import json
m = json.load(open('.next/server/server-reference-manifest.json'))
acts = {(e['filename'], e['exportedName']) for e in m['node'].values()}
print(len(acts), 'server actions')
for f, n in sorted(acts)[:5]: print('  ', n.ljust(28), f)
"
```

Real output from the second, on the same app:

```
25 server actions
   setUserRoleAction            src/app/(app)/admin/users/_actions.ts
   setViewOwnerAction           src/lib/accountant/actions.ts
   acceptAccountantInviteAction src/lib/data/actions.ts
   addTxnAction                 src/lib/data/actions.ts
   answerAskAction              src/lib/data/actions.ts
```

(Without a build, the same list comes from the files: `grep -rl "use server" src --include=*.ts`, then the exported functions in each.)

**Those 25 are enumerable and they are not callable, and the difference matters more than the count.** Next.js compiles each into a build-time hash id, and invoking one means posting to a page URL with a `Next-Action` header carrying that id — an internal protocol with no compatibility promise, whose ids change when the build changes. Driving it is the same bargain this file already refuses for HTML form posts: replaying an undocumented encoding that breaks silently the next time somebody deploys. **So count them, report them, and do not build tools on them.** The section above is where that number belongs in the report.

The ordinary content-type check still applies to the routes that do exist, and it still has to be made against an **authenticated** response — on this app `GET /api/exports/ledger` answers `307` to `/login` when signed out, which tells you about the gate and nothing about the payload.

**Auth shape:**

- **A catch-all under `/api/auth/` is the tell.** `/api/auth/[...all]` with `toNextJsHandler` is Better Auth; `/api/auth/[...nextauth]` is NextAuth/Auth.js. Both mean a **session cookie** obtained by posting to a sign-in route under that same catch-all, and both put the login endpoint *in* the route list — which is a useful confirmation you read the scheme right, the same way FastAPI's `/token` is.
- **A file that gates paths before any handler runs, and on Next 16 it is not called `middleware.ts`.** Look for **both** names at the repository root and in `src/`: `middleware.ts` on Next 15 and earlier, **`proxy.ts` on Next 16**, which renamed it. The exported function is renamed too — `export function proxy(req)`, not `middleware` — so a grep for either word alone misses the other. **This is the Express under-reporting shape one more time, and it caught the first draft of this section**, which read "this app has none" on an app that has one.

  The validation app has `src/proxy.ts`, and it is worth reading as the example of what such a file does to your probes:

  ```ts
  const PROTECTED_PREFIXES = ["/home", "/invoices", "/bank", "/taxes",
                              "/reports", "/settings", "/import"];
  // ... if the session cookie is absent:
  url.pathname = "/login";
  url.searchParams.set("next", pathname);
  return NextResponse.redirect(url);

  export const config = { matcher: ["/home", "/home/:path*", "/invoices", ... ] };
  ```

  **Read the `matcher`, because it tells you which of your probe results are about the route and which are about the gate.** Here it lists the seven page prefixes and **no `/api/…` path at all** — so the API statuses below come from per-handler guards, while any probe of `/invoices` would have been answered by this file without the page ever running. A route that looks gated may not be; a route that looks fine may simply not be in the matcher.

  Note also the third branch in that file: it stands the gate down entirely when `NODE_ENV === "development"` and a `DEV_FAKE_SESSION` flag is set. `credentials.md` has the rule about building on a bypass; this is what one looks like in the wild.

  The probes, unauthenticated and measured during validation:

  ```
  GET /api/integrations/ingest   405
  GET /api/banksync/sync         405
  GET /api/exports/ledger        307  -> /login
  GET /api/cron/remind           401  text/plain
  ```

  Four routes, four different answers, and only one of them is a plain 401. Two are the 405 case above, and the 307 is a session gate redirecting to a login page — which `server.md` has a rule about, because a client that follows it gets a 200 and an HTML login form.
- **Hand-written handlers usually invent their own scheme, and none of it is discoverable from the framework.** This app's ingest route takes `Authorization: Bearer <key>` and says so only in the body of its own 401. There is **no `WWW-Authenticate` header anywhere on the app** — `credentials.md` has the fallback for that, and it was written because of this app.

---

## When there is no API

A template-rendered app returns HTML. Under the write rule this means **reads work and writes are impossible**, and that must be said the moment the checks below answer it — at step 2 of the interview, before any list of what the assistant may do is offered. Not at step 5, by which point the owner has already chosen this path over a better one, on the strength of a capability list that was never true.

(Step 2 is the earliest it can be said. Step 1 only establishes *which* app, and nothing has been read yet — so a rule to announce it there is a rule nobody can follow.)

Telling this case apart is not about the framework. Both Django fixtures are Django; one has an API and one does not.

| Check | `django_shop` | `django_html` |
|---|---|---|
| `rest_framework` in `INSTALLED_APPS` | yes | no |
| Views return | `Response(...)` | `render(request, "...html")` |
| Routes found by enumeration | 6 | 28 |
| `Content-Type`, authenticated, `Accept: application/json` | `application/json` | `text/html; charset=utf-8` |

**Route count is the misleading one, and it points the wrong way.** `django_html` enumerates *more* routes than `django_shop` — twenty-eight against six — because a single `path('admin/', admin.site.urls)` generates twenty-three admin routes. Every one of them serves HTML. A discovery pass that reports a number has reported nothing.

The check that settles it is the content type of a **successful, authenticated** response. An unauthenticated request is no good: `django_html`'s `/invoices/` returns `302` to `/login/` with `Content-Type: text/html` whether or not there is an API behind it. Signed in, the same URL with `Accept: application/json` returns `200 text/html; charset=utf-8` and an HTML document — that is the answer.

Express reaches the same verdict by a different road: the only JSON in `express-shop` is the guard's 401 error body, neither of its two routes returns any, and both answer `text/html` when asked for JSON.

**Do not post to HTML form endpoints as a browser would.** It requires scraping a CSRF token out of markup, replaying a form encoding, and reading a 302 as success. It breaks the first time somebody edits a template, and it fails silently: a re-rendered form full of validation errors is still HTTP 200, so a write that was rejected is indistinguishable from one that worked.

The honest options, offered in this order:

1. Add one JSON endpoint to the app.
2. Build agent access inside it — `start-an-app/references/mcp.md`.
3. Accept a read-only integration, which is often genuinely useful.

Reads from an HTML app are workable — the page is parseable and the shape rarely changes without someone noticing. It is only writes that cannot be done honestly.

---

## Not yet supported

**Rails, Laravel and Go.** None has a discovery section here, and the skill should say so plainly rather than improvise one.

The reason is that the sections above are not documentation summaries. Every command in them was run against a live app, and every one of them turned up something the documentation would not have: `show_urls` is absent from a stock Django install; `app.routes` crashes on current FastAPI and reports no API before it does; `app.router` throws on Express 4 rather than returning `undefined`. All three were the *obvious* method. A section written without a running app to check it against would have shipped all three.

Running it against one app is still not enough, and the Express section is the proof. Both of its greps passed against the fixture and both under-reported badly on any app written differently — one missed every `app.get` route, the other missed every `res.status(n).json(...)` response. A fixture only ever tells you the command works on *that* app. Catching those needed files written specifically to break the pattern. **So a stack ships here only once someone has run its commands against a live app and then tried to make them fail** — which is more work than reading the docs, and it is the whole reason this list is as long as it is.

**Next.js App Router sits between the two, and its section says so at the top.** Its commands were run against a live app — a real one, written by somebody else, for something else — but only one, and nobody has yet written the files that would break them. That is half the bar. The half outstanding is a second app and a deliberate attempt to make the two greps in it under-report, which is exactly the attempt that caught both Express failures. Until then its counts are evidence and its commands are a good first draft, and telling an owner which of those they are getting costs one sentence.

Rails, Laravel and Go each have the same shape of problem waiting — implicit routing that generates URLs found in no file, a stack-specific auth convention, and an HTML-only case that has to be recognised rather than guessed at — and none of it has been checked here.

**A stack listed on the strength of a route grep is a promise the skill cannot keep.** The owner finds out at step 5, after they have already chosen this path over a better one. "This is not covered yet" costs them five minutes. A confident wrong answer costs them the afternoon and the trust.

---

## Verify

- [ ] The route list came from a command that made the framework enumerate itself, not from a grep for path strings.
- [ ] If the app would not boot and the list was inferred, the words "inferred" and "may be incomplete" appear in what was told to the owner.
- [ ] Every enumerated route was sent one request and **its status recorded**, per route. Only 404s and routes broken on every verb were dropped. (DRF format-suffix routes are the usual casualty — check for a 500.)
- [ ] No route was dropped on a **405**. Every 405 was re-probed with the verb its source exports, and where the stack answers 401 before 405, it was re-probed again once a credential existed.
- [ ] The owner was asked what they *do* in the app, in their own words, **before** the route list was shown to them — and every item on that list was mapped to a route or reported as unreachable.
- [ ] The report gives two numbers, not one: how many things the app does, and how many of them can be reached over HTTP. Where the second is much smaller, that was said at interview step 2.
- [ ] Trailing-slash behaviour was measured, not assumed: the exact URL recorded is the one that returned 200, not the one that returned 301 or 307.
- [ ] Every route was probed **without** credentials and its status recorded. A single unauthenticated 200 was not read as "no auth on this app".
- [ ] Django: `grep "rest_framework" settings.py` was run, and its result agrees with the content types actually observed.
- [ ] FastAPI: the enumeration used `app.openapi()` or a live `/openapi.json`, never a bare loop over `app.routes`.
- [ ] Express: the route grep matched **`.get(`, not `router.get`** — routes registered as `app.get`, `app.route`, `app.all`, or on a router variable not called `router`, were looked for and either found or ruled out.
- [ ] Express: the API grep matched **`.json(`, not `res.json`** — a `res.status(n).json(...)` handler would have been found. No app was called no-API on a `res.json` grep alone.
- [ ] Express: the installed major version was checked before any `_router` introspection, and the route list was reconciled against `app.use` mounts plus the router files.
- [ ] The no-API test used an **authenticated** response's `Content-Type`, not a 302 to a login page.
- [ ] If the answer was no-API, "reads work, writes are impossible" was said before any capability list was offered.
- [ ] Next.js: the route list came from `.next/app-path-routes-manifest.json`, not from the file tree — or, where there was no build, the list was announced as **inferred** and every `(group)` segment was removed from every path by hand.
- [ ] Next.js: the manifest read was the **build's** (`.next/app-path-routes-manifest.json`), never anything under `.next/dev/` — measured, a dev tree held 3 of 49 app paths and 1 of 25 Server Actions. Its timestamp was compared against the newest source file, and a manifest older than the source was reported as a stale list.
- [ ] Next.js: **both** `middleware.ts` and `proxy.ts` were looked for, at the repository root and in `src/`. Where one exists, its `matcher` was read and the probe results were split into "answered by the gate" and "answered by the route".
- [ ] Next.js: the method grep matched **`export`, not `export function`** — an `export const { GET, POST } = toNextJsHandler(auth)` catch-all would have been found. Server Actions were counted and reported as unreachable, not offered as capabilities.
- [ ] Next.js: the owner was told this stack rests on one real app, unlike the other three.
- [ ] The stack is one of Django, Express, FastAPI or Next.js App Router. If not — including Next.js's older Pages Router — the owner was told it is not covered rather than given a guessed route list.
- [ ] The auth mechanism was named and handed to `credentials.md`; this file did not attempt to obtain or store the credential.
