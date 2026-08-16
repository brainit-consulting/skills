# Route discovery

Last verified: 2026-08-16

**Purpose:** Work out what an existing app can actually be asked to do over HTTP — which URLs exist, which of them return data rather than a web page, and how the app decides who is allowed in — without editing a single file the owner wrote.

Everything here was run against real apps. Every block of output below is pasted from a terminal, not paraphrased.

## Ask the framework, do not guess

Every stack here generates routes that appear in no source file.

A Django REST Framework router turns one line — `router.register(r"orders", OrderViewSet)` — into **six** URL patterns. FastAPI's `include_router` rewrites every path in the router with a prefix, so an endpoint whose source says `@router.get("")` is served at `/api/orders` and **nothing** answers at `/orders`. Django's `path('admin/', admin.site.urls)` is one line that becomes twenty-three routes. Not one of those URLs appears as a string anywhere in the project.

**A grep for literal paths finds none of them and reports an app with no API.** That is not a small error. The owner then chooses a read-only integration, or chooses this skill over a better one, on the strength of a list they believe is complete. "I found three endpoints" when the framework generated forty is worse than finding nothing, because nobody goes looking for the other thirty-seven.

So: run the command that makes the framework list its own routes. Inference is the fallback for an app that will not boot, and when you fall back you **say the list was inferred** — in the same breath as giving it, not in a footnote.

## Then probe what you found

Enumerating is half the job. The framework tells you what it *registered*, not what *works*, and the gap between the two is real:

- **A registered route can be broken.** DRF's router generates format-suffix routes (`/api/orders.json`) whether or not the view accepts a `format` argument. On the Django fixture the router enumerated **six** routes and only **four** answer: `/api/orders.json` and `/api/orders/1.json` both return **HTTP 500** with a valid token — `TypeError: OrderViewSet.list() got an unexpected keyword argument 'format'` — while `/api/orders/` and `/api/orders/1/` return 200. All six were in the enumerated list. Hand all six to an agent and a third of its tools fail every time they are used, with a stack trace that blames the app's own code and gives no hint that the route was auto-generated and never worked.
- **A trailing slash is not cosmetic, and the stacks disagree in opposite directions.** Measured on the fixtures:

  | Request | Django + DRF (:8000) | FastAPI (:8001) | Express (:3001) |
  |---|---|---|---|
  | `/api/orders/` (with slash) | **200** | 307 redirect | — |
  | `/api/orders` (no slash) | 301 redirect | **200** | — |
  | `/users` / `/users/` | — | — | **200** both |

  A client that does not follow redirects sees a 301 or a 307 and reports failure. A client that *does* follow them may drop the body on the way, so a POST silently becomes a GET.

So after enumerating, send one request to each route and keep the ones that answer. A route list is a claim; a status code is evidence.

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
grep -rn "app.use\|router\.\(get\|post\|put\|patch\|delete\)" app.js routes/
```

Real output from `express-shop`:

```
app.js:16:app.use(logger('dev'));
app.js:17:app.use(express.json());
app.js:18:app.use(express.urlencoded({ extended: false }));
app.js:19:app.use(cookieParser());
app.js:20:app.use(express.static(path.join(__dirname, 'public')));
app.js:31:app.use('/', indexRouter);
app.js:32:app.use('/users', requireApiKey, usersRouter);
app.js:35:app.use(function(req, res, next) {
app.js:40:app.use(function(err, req, res, next) {
routes/index.js:5:router.get('/', function(req, res, next) {
routes/users.js:5:router.get('/', function(req, res, next) {
```

Eleven lines, of which **two** are mounts. The rule that separates them: a mount is an `app.use` whose **first argument is a path string**. `app.use(logger('dev'))` and `app.use(function(req, res, next) {...})` take a function first and are middleware — they run on everything and route nothing. Lines 31 and 32 take `'/'` and `'/users'`.

Concatenate mount and leaf: `GET /` and `GET /users`. Both confirmed live — 200 and 200.

**Why not introspect the router stack.** `app._router.stack` does exist and can be walked, but three things observed on this fixture rule it out as the primary method:

- **The accessor is version-dependent, and the wrong one throws rather than returning nothing.** On Express 4.16.4 the working property is `app._router`; reading `app.router` raises `Error: 'app.router' is deprecated!` and exits the process. Express 5 removes `_router` and makes `router` the supported name. A defensive `app._router || app.router` does not help — it crashes on Express 4 while evaluating the fallback it never needed.
- **Mount paths are stored only as compiled regexes**, so you rebuild the URL by un-escaping one. The layer for `/users` holds `^\/users\/?(?=\/|$)`; the layer for `/` holds `^\/?(?=\/|$)`. An un-escaping attempt written for this file produced `/?(?=/|$)/` as the app root — wrong, and wrong in a way that still looks like a route, so nothing downstream flags it.
- **Most of the stack is not routes.** The fixture's stack has twelve entries and only **two** are mounted routers. The rest are `query`, `expressInit`, `logger`, `jsonParser`, `urlencodedParser`, `cookieParser`, `serveStatic`, the `requireApiKey` guard, and two anonymous error handlers.

Introspection earns its place as a **cross-check** when the files are ambiguous — routers mounted in a loop, paths built from variables. Then walk `app._router.stack`, pin the Express major version first, and treat the result as a second opinion rather than the answer. Note that `require('./app')` on an express-generator layout does not bind a port: `app.js` exports the app and `bin/www` calls `listen`. So you can introspect without starting a server.

**Fallback when it will not boot:** reading the files *is* the method, so a broken app costs you nothing here — which is Express's one real advantage over the other two. What it costs you is the probe. Without a running server you cannot confirm the routes answer, so the list is **inferred** and must be announced as inferred, exactly like the Django and FastAPI fallbacks.

**Is there an API at all?** Routes are not an API, and Express is where that bites, because nothing in the framework distinguishes the two. Grep for what the handlers return:

```bash
grep -rn "res.json\|res.send\|res.render" app.js routes/
```

Real output from `express-shop`:

```
app.js:47:  res.render('error');
routes/index.js:6:  res.render('index', { title: 'Express' });
routes/users.js:6:  res.send('respond with a resource');
```

`res.json` appears **nowhere**. Confirmed against the running app: both routes answer `200` with `Content-Type: text/html; charset=utf-8`, even when asked for `application/json`. So this app has two working routes and no JSON API — the no-API case below, arrived at from a completely different direction from Django's.

Two traps in that check:

- **`app.use(express.json())` is not evidence of an API.** It parses *incoming* request bodies and says nothing about what goes out. The fixture has it on line 17 and returns HTML from every route.
- **`res.send` is content-negotiated by argument type.** `res.send('a string')` sets `text/html`; `res.send({...})` sets `application/json`. Reading the call is not enough — check what the argument is, or just probe the route.

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
['GET', 'HEAD'] /openapi.json
['GET', 'HEAD'] /docs
['GET', 'HEAD'] /docs/oauth2-redirect
['GET', 'HEAD'] /redoc
['POST'] /token
Traceback (most recent call last):
  ...
AttributeError: '_IncludedRouter' object has no attribute 'path'
```

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

**Is there an API at all?** For FastAPI the question barely arises — it is a JSON framework, and every route in the schema returns JSON unless it declares an HTML `response_class` or the app mounts a template engine (`Jinja2Templates`). Check for those two before assuming; if neither appears, the enumerated routes are the API.

**Auth shape:** the dependency names in the source tell you the scheme.

- `OAuth2PasswordBearer` means a **`Bearer`** token, obtained by posting form data to the `tokenUrl` named in its constructor. The fixture's `OAuth2PasswordBearer(tokenUrl="token")` matches the `POST /token` in the enumeration — the login endpoint is *in* the route list, which is a useful confirmation that you read the scheme right.
- `HTTPBearer` means a bearer token the app does not issue.
- `APIKeyHeader` / `APIKeyQuery` name the header or parameter directly in their constructor.

The schema also carries this: a `securitySchemes` block under `components` in `app.openapi()`. Measure it either way — `GET /api/orders` returns **401** with no credentials and **200** with `Authorization: Bearer <token>`. Hand the mechanism, including the token endpoint and the form field names it expects, to `credentials.md`.

---

## When there is no API

A template-rendered app returns HTML. Under the write rule this means **reads work and writes are impossible**, and that must be said at step 1 of the interview, not discovered at step 5 — by which point the owner has already chosen this path over a better one, on the strength of a capability list that was never true.

Telling this case apart is not about the framework. Both Django fixtures are Django; one has an API and one does not.

| Check | `django_shop` | `django_html` |
|---|---|---|
| `rest_framework` in `INSTALLED_APPS` | yes | no |
| Views return | `Response(...)` | `render(request, "...html")` |
| Routes found by enumeration | 6 | 28 |
| `Content-Type`, authenticated, `Accept: application/json` | `application/json` | `text/html; charset=utf-8` |

**Route count is the misleading one, and it points the wrong way.** `django_html` enumerates *more* routes than `django_shop` — twenty-eight against six — because a single `path('admin/', admin.site.urls)` generates twenty-three admin routes. Every one of them serves HTML. A discovery pass that reports a number has reported nothing.

The check that settles it is the content type of a **successful, authenticated** response. An unauthenticated request is no good: `django_html`'s `/invoices/` returns `302` to `/login/` with `Content-Type: text/html` whether or not there is an API behind it. Signed in, the same URL with `Accept: application/json` returns `200 text/html; charset=utf-8` and an HTML document — that is the answer.

Express reaches the same verdict by a different road: `res.json` appears nowhere in `express-shop`, and both its routes return `text/html` when asked for JSON.

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

Rails, Laravel and Go each have the same shape of problem waiting — implicit routing that generates URLs found in no file, a stack-specific auth convention, and an HTML-only case that has to be recognised rather than guessed at — and none of it has been checked here.

**A stack listed on the strength of a route grep is a promise the skill cannot keep.** The owner finds out at step 5, after they have already chosen this path over a better one. "This is not covered yet" costs them five minutes. A confident wrong answer costs them the afternoon and the trust.

---

## Verify

- [ ] The route list came from a command that made the framework enumerate itself, not from a grep for path strings.
- [ ] If the app would not boot and the list was inferred, the words "inferred" and "may be incomplete" appear in what was told to the owner.
- [ ] Every enumerated route was sent one request. Routes that did not answer were dropped from the list, not passed on. (DRF format-suffix routes are the usual casualty — check for a 500.)
- [ ] Trailing-slash behaviour was measured, not assumed: the exact URL recorded is the one that returned 200, not the one that returned 301 or 307.
- [ ] Every route was probed **without** credentials and its status recorded. A single unauthenticated 200 was not read as "no auth on this app".
- [ ] Django: `grep "rest_framework" settings.py` was run, and its result agrees with the content types actually observed.
- [ ] FastAPI: the enumeration used `app.openapi()` or a live `/openapi.json`, never a bare loop over `app.routes`.
- [ ] Express: the installed major version was checked before any `_router` introspection, and the route list was reconciled against `app.use` mounts plus the router files.
- [ ] The no-API test used an **authenticated** response's `Content-Type`, not a 302 to a login page.
- [ ] If the answer was no-API, "reads work, writes are impossible" was said before any capability list was offered.
- [ ] The stack is one of Django, Express or FastAPI. If not, the owner was told it is not covered rather than given a guessed route list.
- [ ] The auth mechanism was named and handed to `credentials.md`; this file did not attempt to obtain or store the credential.
