# Credentials

Last verified: 2026-08-16

**Purpose:** Get one working credential out of an app somebody else built, prove it works before anything is built on top of it, and — where the fallback reads the database directly — make the database itself refuse writes rather than trusting the code not to attempt them.

`discovery.md` names the mechanism. This file obtains it, sends it, and tests it. Everything below was run against live apps and live databases; every block of output is pasted from a terminal.

> **Shell:** the commands assume a POSIX shell — single-quoted `curl -w` format strings and heredocs. On Windows run them in Git Bash or WSL rather than PowerShell.

---

## The trap: session cookies and CSRF

Most existing apps authenticate with a session cookie plus a CSRF token, not a bearer token. A server posting to a Django endpoint with a valid session cookie and no CSRF token gets a 403 that explains nothing. **Reads work and writes fail**, which points suspicion at the write code rather than at auth.

Observed on `django_html`, with a session cookie obtained by posting the login form. First a read, with that cookie:

```
$ curl -s -o /dev/null -b jar.txt -w '%{http_code} %{content_type}\n' \
    http://127.0.0.1:8002/invoices/
200 text/html; charset=utf-8
```

Then a write, with **the same cookie**:

```
$ curl -s -i -b jar.txt -X POST -d "number=INV-777&amount=42.00" \
    http://127.0.0.1:8002/invoices/new/
HTTP/1.1 403 Forbidden
```

```
<title>403 Forbidden</title>
<h1>Forbidden (403)</h1>
CSRF verification failed. Request aborted.

Reason given for failure:
    CSRF token missing.
```

The identical request with the token attached:

```
$ curl -s -i -b jar.txt -X POST -H "X-CSRFToken: $TOK" \
    -d "number=INV-777&amount=42.00" http://127.0.0.1:8002/invoices/new/
HTTP/1.1 302 Found
Location: /invoices/4/
```

— and the row is there afterwards: `<li><a href="/invoices/4/">INV-777 - 42.00</a></li>`.

That 403 body is better than most; Django says the word "CSRF" out loud. The reason it still costs an hour is the *shape* of the failure, not the wording. Every read succeeds. Every list, every detail page, every probe run during discovery came back 200. The one thing that fails is the one thing that was added last, so the search starts in the new write code and stays there.

The obvious fix — exempting those routes from CSRF protection — is a real security regression in somebody's app, arrived at by accident while trying to make a demo work. **Never do it, and never suggest it.** `@csrf_exempt` takes ten seconds to add, is invisible in a diff of a folder nobody reviews, and outlives the demo. The app's owner ends up with a form anyone on the internet can make their logged-in users submit, because an integration wanted a shortcut.

Do this instead:

| Stack | Fetch the token | Send it as |
|---|---|---|
| Django | `GET` any page, read the `csrftoken` cookie | `X-CSRFToken` + the same cookie |
| Express | **nothing built in** — grep first, see below | `X-CSRF-Token`, if a layer exists |
| FastAPI | rarely present; usually token auth | — |

**The Express row is a warning, not a recipe.** Express ships no CSRF protection, so whether the app has any is a property of that app and nothing tells you but the source. On `express-shop`:

```
$ node -e "console.log(JSON.stringify(require('./package.json').dependencies))"
{"cookie-parser":"~1.4.4","debug":"~2.6.9","express":"~4.16.1",
 "http-errors":"~1.6.3","jade":"~1.11.0","morgan":"~1.9.1"}

$ grep -rin "csrf" app.js routes/ bin/
(no matches)
```

No `csurf`, no `csrf-csrf`, no hand-rolled middleware — so no token to fetch and none to send. Check both the dependency list and the source; a hand-rolled check is a plain middleware function and will not appear in `package.json`. Assuming a token is needed on an app that has none wastes an afternoon looking for an endpoint that was never written.

**If the app has a token-auth path — Django REST `TokenAuthentication`, an API-key middleware, an OAuth2 flow — prefer it.** It sidesteps all of this and it is what the app's own authors intended for non-browser clients. Measured on `django_shop`, a write through the token path, with no cookie jar, no `csrftoken` and no `X-CSRFToken` anywhere in the request:

```
$ curl -s -i -X POST -H "Authorization: Token $K" \
    -H "Content-Type: application/json" \
    -d '{"customer":"Ada","total":"9.99"}' http://127.0.0.1:8000/api/orders/
HTTP/1.1 201 Created
{"customer":"Ada","total":"9.99"}
```

Same framework, same CSRF middleware installed, no CSRF anything required. That is the whole argument for using the token path where one exists.

---

## The scheme word is the credential

Before the per-stack sections, the failure that costs the most hours and is invisible in every error message it produces.

A token sent in an `Authorization` header is prefixed with a scheme word. **Get the word wrong and the server reports that you sent nothing at all.** Measured in both directions on the same afternoon.

Django REST Framework, with a **valid** token and the wrong word:

```
$ curl -s -H "Authorization: Bearer $K" http://127.0.0.1:8000/api/orders/
{"detail":"Authentication credentials were not provided."}

$ curl -s http://127.0.0.1:8000/api/orders/          # no header at all
{"detail":"Authentication credentials were not provided."}
```

Byte for byte the same response. The correct word gets in:

```
$ curl -s -H "Authorization: Token $K" http://127.0.0.1:8000/api/orders/
[{"id":1,"customer":"Ada","total":"9.99"}]
```

FastAPI, the mirror image — a **valid** token with DRF's word on it:

```
$ curl -s -H "Authorization: Token fixture-oauth2-token" http://127.0.0.1:8001/api/orders
{"detail":"Not authenticated"}

$ curl -s http://127.0.0.1:8001/api/orders          # no header at all
{"detail":"Not authenticated"}
```

Both frameworks parse the header, fail to recognise the scheme, discard the whole thing and report an *absent* credential. So the message sends you to check whether the header is being set, whether the proxy is stripping it, whether the environment variable is loaded — all of which are fine. Compare with a genuinely wrong token, which says something completely different:

```
$ curl -s -H "Authorization: Token 0000000000000000000000000000000000000000" \
    http://127.0.0.1:8000/api/orders/
{"detail":"Invalid token."}

$ curl -s -H "Authorization: Bearer nope" http://127.0.0.1:8001/api/orders
{"detail":"Invalid authentication credentials"}
```

**"Invalid token" means the word is right and the value is wrong. "Not provided" means the word is wrong — or there is genuinely no header.** Learning to read those two apart is most of debugging this.

**It is the word, not its capitalisation.** Both frameworks match the scheme word case-insensitively — measured, `Authorization: token <key>` returns **200** from DRF and `authorization: bearer <token>` and `Authorization: BEARER <token>` both return **200** from FastAPI. So `Token` versus `token` is never the problem, and time spent on it is wasted; `Token` versus `Bearer` always is.

**The measurable check: read the `WWW-Authenticate` header on the 401.** The server names its own scheme there, and it is not a guess:

```
$ curl -s -i http://127.0.0.1:8000/api/orders/ | grep -i www-authenticate
WWW-Authenticate: Token

$ curl -s -i http://127.0.0.1:8001/api/orders | grep -i www-authenticate
www-authenticate: Bearer
```

Send one unauthenticated request and read that line before writing any credential code. (The header name's casing varies by server — match case-insensitively.)

---

## Django

Two mechanisms, and an app can use both at once — a token for `/api/`, a session for everything else.

### DRF token

**Where it is issued:** the `authtoken` app stores it in the database, not in source. `rest_framework.authtoken` in `INSTALLED_APPS`, or `TokenAuthentication` in a view's `authentication_classes`, is what `discovery.md` hands over.

**How to create one for this purpose.** Make a dedicated user rather than reusing a person's account — the disclosure at the end of this file is about one fixed identity, and a named one is far easier to explain, audit and switch off:

```bash
python manage.py shell -c "
from django.contrib.auth.models import User
from rest_framework.authtoken.models import Token
u, _ = User.objects.get_or_create(username='agent')
t, _ = Token.objects.get_or_create(user=u)
print('token:', t.key)
"
```

`get_or_create` returns the existing token if one is already there. DRF's default model stores **one token per user** — the `user` field is a `OneToOneField`, and creating a second raises `IntegrityError: duplicate key value violates unique constraint "authtoken_token_user_id_key"`. So this is safe to run twice, and there is no way to issue the agent a second key alongside a person's existing one on the same account. To rotate, delete the row and create it again, and expect the old key to stop working immediately — which is the strongest argument for a dedicated `agent` user: rotating or revoking it does not log a real person out of their own integration.

**Send it as:**

```
Authorization: Token <key>
```

**How to test it** — the three-request sequence, and all three matter:

```bash
curl -s -o /dev/null -w '%{http_code}\n' <url>                              # expect 401
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Token $K" <url> # expect 200
curl -s -H "Authorization: Bearer $K" <url>                                 # expect the 401 above
```

The third looks pointless and is the one worth keeping: if it returns 200, this app is not using DRF's `TokenAuthentication` and the rest of your assumptions are wrong too.

### Session login

**Where it is issued:** by posting the login form. `SessionAuthentication`, `@login_required` or a `LoginView` in the URLconf is what `discovery.md` hands over.

**How to obtain one.** Django's login form is itself CSRF-protected, so it is two requests — fetch the page for a token, then post:

```bash
CSRF=$(curl -s -c jar.txt http://<host>/login/ \
       | grep -oP 'name="csrfmiddlewaretoken" value="\K[^"]+')

curl -s -b jar.txt -c jar.txt -o /dev/null -w '%{http_code} %{redirect_url}\n' \
     -d "username=<user>&password=<pass>&csrfmiddlewaretoken=$CSRF" \
     -e http://<host>/login/ http://<host>/login/
```

Observed: `302 http://127.0.0.1:8002/accounts/profile/`, and the jar then holds both cookies:

```
#HttpOnly_127.0.0.1  ...  sessionid  f1j3tx9i3g5vbyjht4eoixfeqtfgtu8c
127.0.0.1            ...  csrftoken  Cze43KAMvnh96B1poLyFOFZ507Nh9WLA
```

**A 200 from that POST is a failure, not a success.** Django re-renders the login page with an error inside a 200 when the password is wrong. A working login redirects. Check for `302`, never for `200`.

**Send it as:** the `sessionid` cookie on every request, plus — on every unsafe method — the `csrftoken` cookie *and* the same value in an `X-CSRFToken` header. **Both.** Django compares the two against each other, so having one is no better than having neither. It does at least tell you which is missing, in the reason line:

| What was sent | Reason given for failure |
|---|---|
| cookie, no header | `CSRF token missing.` |
| header, no cookie | `CSRF cookie not set.` |
| both, over HTTPS, no `Referer` | `Referer checking failed - no Referer.` |

All three are a 403 with the same title and the same page. Only that line separates them, which is why it is worth logging the `<pre>` block rather than the status.

**The `-e` (Referer) argument above is not decoration, and this is the one that waits until production.** Over HTTPS — and only over HTTPS — Django additionally requires a `Referer` header on unsafe methods. Measured, against the same fixture served with `SECURE_PROXY_SSL_HEADER` set and `X-Forwarded-Proto: https` on the request, sending a valid session cookie **and** a valid CSRF token:

```
HTTP/1.1 403 Forbidden

Reason given for failure:
    Referer checking failed - no Referer.
```

Add the header and the identical request succeeds:

```
$ curl ... -e https://127.0.0.1:8003/invoices/new/ -X POST ...
HTTP/1.1 302 Found
Location: /invoices/5/
```

So the integration is built against `http://localhost`, where the check does not run, passes every test, and starts returning 403 the moment it is pointed at the real deployment. Nothing changed but the scheme, and nothing in the 403 says so unless you read the reason line. Send a same-origin `Referer` on every unsafe request from the start and it never comes up.

---

## Express

**There is no convention here, and inventing one is the mistake.** Express has no built-in auth, so the credential is whatever a middleware function in that app decided to read. It can be a header, a cookie, a query parameter, a bearer token validated against a library, or a comparison against a literal string. `discovery.md` finds the guard by probing every route unauthenticated; this file's job is to read that guard and copy its expectations exactly.

Read the middleware. On `express-shop` it is six lines of `app.js`:

```js
var API_KEY = 'fixture-api-key';

function requireApiKey(req, res, next) {
  if (req.get('X-API-Key') !== API_KEY) {
    return res.status(401).json({ error: 'missing or invalid X-API-Key' });
  }
  next();
}
```

Everything you need is in it: the header name, and the fact that the comparison is against a constant rather than a lookup. Where the value is a literal or an environment variable, **there is nothing to create** — the owner has to give it to you, and if they cannot find it, it is in the source or the deployment's environment. Where it is looked up in a store, create a dedicated row the same way as the Django token above.

**Measured, with and without:**

```
$ curl -s -i http://127.0.0.1:3001/users
HTTP/1.1 401 Unauthorized
{"error":"missing or invalid X-API-Key"}

$ curl -s -i -H "X-API-Key: fixture-api-key" http://127.0.0.1:3001/users
HTTP/1.1 200 OK
respond with a resource
```

Two failure shapes worth knowing:

**The right key in the wrong header is a 401 that names the right header and still does not help you.**

```
$ curl -s -H "Authorization: Bearer fixture-api-key" http://127.0.0.1:3001/users
{"error":"missing or invalid X-API-Key"}
```

That is the correct secret, rejected. `Authorization: Bearer` is the reflex after working on any other stack, and the message says "missing or invalid" — which reads as a bad key, so the next hour goes on regenerating a key that was fine.

**Casing is not the bug.** HTTP header names are case-insensitive and Express's `req.get()` honours that:

```
$ curl -s -o /dev/null -w '%{http_code}\n' -H "x-api-key: fixture-api-key" \
    http://127.0.0.1:3001/users
200
```

So `x-api-key` and `X-API-Key` are the same header. If a header-based credential is failing, the name is wrong or the value is wrong — do not spend time on the capitalisation.

**A guard on one mount says nothing about the others.** `express-shop` mounts `app.use('/', indexRouter)` unguarded and `app.use('/users', requireApiKey, usersRouter)` guarded — `GET /` is 200 with no credential at all. One working credential does not mean every route needs it, and one public route does not mean none do.

---

## FastAPI

**Where it is issued:** by the app, if the dependency is `OAuth2PasswordBearer`. Its constructor names the endpoint — `OAuth2PasswordBearer(tokenUrl="token")` means `POST /token`, and `discovery.md`'s route list will show that endpoint, which is a useful confirmation that the scheme was read correctly.

**How to obtain one.** The OAuth2 password flow is **form-encoded**, and the field names are fixed by the spec: `username` and `password`, whatever the app calls its users.

```
$ curl -s -X POST -d "username=ada&password=fixture-pass" http://127.0.0.1:8001/token
{"access_token":"fixture-oauth2-token","token_type":"bearer"}
```

**Sending those same credentials as JSON returns 422, not 401** — and the body blames the fields rather than the encoding:

```
$ curl -s -X POST -H "Content-Type: application/json" \
    -d '{"username":"ada","password":"fixture-pass"}' http://127.0.0.1:8001/token
{"detail":[{"type":"missing","loc":["body","username"],"msg":"Field required","input":null},
           {"type":"missing","loc":["body","password"],"msg":"Field required","input":null}]}
```

"Field required" for two fields that are plainly present in the request you just sent. `OAuth2PasswordRequestForm` reads form data only, so a JSON body parses to nothing and every field reads as missing. JSON is the obvious default for a JSON framework, which is why this one catches people. Use `-d "a=b&c=d"`, or `data=` rather than `json=` in a client library.

**Send it as:**

```
Authorization: Bearer <access_token>
```

Use the `access_token` value from the response, not the whole JSON object, and note `token_type` is `"bearer"` lower-case in the body while the header word is conventionally `Bearer` — the header is matched case-insensitively by FastAPI, but do not copy the body value into the header and hope.

**How to test it:**

```
$ curl -s -H "Authorization: Bearer fixture-oauth2-token" http://127.0.0.1:8001/api/orders
[{"id":1,"customer":"Ada","total":"9.99"}]

$ curl -s -X POST -d "username=ada&password=wrong" http://127.0.0.1:8001/token
{"detail":"Incorrect username or password"}     # 401 — the login itself failed

$ curl -s -H "Authorization: Bearer nope" http://127.0.0.1:8001/api/orders
{"detail":"Invalid authentication credentials"} # 401 — the token failed
```

Those last two are both 401 and they mean opposite things. The first says the username or password is wrong; the second says the login worked and the token did not. Read the body, not the status.

**Where the dependency is `APIKeyHeader` or `APIKeyQuery` instead**, the constructor names the header or parameter directly and there is no token endpoint — treat it like the Express case: the owner supplies the value.

---

## The read-only database user

This section is about **reads**. Writes go through the app's API, always, and no credential described here changes that — an agent writing SQL skips the app's validation, permissions, hooks and audit logging, and leaves the app in a state its own code believes impossible. That surfaces later, somewhere unrelated, as a bug nobody can reproduce. So the database credential below is not a write credential with the writes turned off by convention; it is a credential that **cannot** write, and the rest of this section is about making sure of that rather than hoping.

The fallback reads SQL directly. That must be enforced by the database, not by the server's good intentions — a later edit can change code, and cannot grant itself a permission it does not have.

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

Expected: a permission error. **If that INSERT succeeds, stop.** The fallback is not safe to enable; use API-only reads and tell the user why.

Both blocks above were run against a real Postgres 18.6 and MySQL 8.4.11 holding a real Django schema, and the refusals are real:

```
postgres=> SELECT current_user;
 agent_readonly

postgres=> SELECT id, username, is_superuser FROM auth_user;
 id | username | is_superuser
----+----------+--------------
  1 | ada      | f

postgres=> INSERT INTO django_session DEFAULT VALUES;
ERROR:  permission denied for table django_session
postgres=> UPDATE auth_user SET is_superuser = true WHERE username = 'ada';
ERROR:  permission denied for table auth_user
postgres=> DELETE FROM authtoken_token;
ERROR:  permission denied for table authtoken_token
postgres=> DROP TABLE django_session;
ERROR:  must be owner of table django_session
```

```
mysql> SELECT CURRENT_USER();
agent_readonly@%

mysql> INSERT INTO django_session () VALUES ();
ERROR 1142 (42000): INSERT command denied to user 'agent_readonly'@'127.0.0.1' for table 'django_session'
mysql> UPDATE auth_user SET is_superuser=1 WHERE username='ada';
ERROR 1142 (42000): UPDATE command denied to user 'agent_readonly'@'127.0.0.1' for table 'auth_user'
mysql> DROP TABLE django_session;
ERROR 1142 (42000): DROP command denied to user 'agent_readonly'@'127.0.0.1' for table 'django_session'
```

Test more than `INSERT`. `SELECT`-only is refused for `UPDATE`, `DELETE` and `DROP` as well, but that is a property of the grant you wrote, and a grant written slightly differently — `GRANT ALL` narrowed by hand, a role inherited from elsewhere — can pass an `INSERT` test and still allow a `DELETE`.

### What `ALTER DEFAULT PRIVILEGES` does, and the half it does not

That last Postgres line is easy to paste and never test. It works — a table created **after** the grants, by the role that ran them, is readable with no further `GRANT`:

```
postgres=# CREATE TABLE orders_order (id serial primary key, customer text, total numeric(10,2));
CREATE TABLE

postgres=> SELECT * FROM orders_order;          -- as agent_readonly
 id | customer | total
----+----------+-------
  1 | Ada      |  9.99

postgres=> INSERT INTO orders_order (customer, total) VALUES ('Grace', 1.00);
ERROR:  permission denied for table orders_order
```

**But it only covers tables created by the role that ran it.** Verified — a second role creating a table in the same schema:

```
app_migrator=> CREATE TABLE orders_refund (id serial primary key, amount numeric(10,2));
CREATE TABLE

postgres=> SELECT * FROM orders_refund;         -- as agent_readonly
ERROR:  permission denied for table orders_refund
```

So if the app's migrations run as a different database user from the one that set these grants, every table added from then on is invisible to the agent. That surfaces weeks later as one tool returning `permission denied for table orders_refund` while everything else still works — a message that points at permissions in general and says nothing about *which role created the table*. Run the grants **as the same role the app's migrations run as**, or re-run the `GRANT SELECT ON ALL TABLES` line after each deployment that adds one.

MySQL has no equivalent problem: `GRANT SELECT ON <db>.*` is database-wide and covers tables created afterwards regardless of who created them. Verified both ways — a table created by `root` and a table created by a second, non-root account were each readable by `agent_readonly` with no further grant, and neither was writable by it.

Two more things measured, one per engine:

- **Postgres: the block names `public` and only `public`.** A table in another schema is not covered — `ERROR: permission denied for schema reporting`. List the schemas (`SELECT nspname FROM pg_namespace WHERE nspname NOT LIKE 'pg\_%' AND nspname <> 'information_schema';`) and repeat the `USAGE` and `SELECT` grants for each one the app actually uses.
- **MySQL: the `@'%'` part of the name is matched, not decorative.** An account created as `'agent_readonly'@'localhost'` refuses a TCP connection from 127.0.0.1: `ERROR 1045 (28000): Access denied for user 'ro_localonly'@'127.0.0.1' (using password: YES)`. That is the same error as a wrong password, so the password gets regenerated first and the host part is found last.

### SQLite: the rule cannot be met, and pretending otherwise is worse

**Check the engine before writing any of the above.** For Django — `discovery.md` shows how to find the real `settings.py` rather than guessing at it:

```bash
grep -n "ENGINE" <project>/settings.py
```

Both Django apps checked here answer `'django.db.backends.sqlite3'`, and that is not unusual — it is what `startproject` writes by default, and a great many small apps ship on it and never change it. So this is not an edge case to note and move past. It is the likeliest single answer, and the safety rule above **cannot be met on it**.

**SQLite has no users, no roles and no `GRANT`.** Not "these are awkward to configure" — the statements do not exist:

```
sqlite> CREATE USER agent_readonly WITH PASSWORD 'x';
OperationalError: near "USER": syntax error
sqlite> GRANT SELECT ON invoices_invoice TO agent_readonly;
OperationalError: near "GRANT": syntax error
```

A SQLite database is a file, and anything that can open the file can write to it. There is no identity for the database to restrict.

**SQLite does offer read-only modes, and they are not the thing the rule asks for.** All three refuse the write:

```
mode=ro              INSERT -> OperationalError: attempt to write a readonly database
immutable=1          INSERT -> OperationalError: attempt to write a readonly database
PRAGMA query_only=1  INSERT -> OperationalError: attempt to write a readonly database
```

Every one of those is chosen by the connecting code, so the connecting code can unchoose it. Measured, in a single process:

```
### mode=ro INSERT
    !! OperationalError: attempt to write a readonly database
### SECOND connection in the SAME process, flag simply omitted -> INSERT
    -> committed
### row count after the bypass
    -> (1,)
```

```
### query_only=1 INSERT
    !! OperationalError: attempt to write a readonly database
### same connection turns it off: PRAGMA query_only = 0 then INSERT
    -> committed
```

**This is the part that has to be said plainly, because the prove-it test above does not catch it.** Connect through `mode=ro`, run that `INSERT`, and it is refused with a convincing error. The test passes. It passes because you configured the connection you are testing — and a later edit that opens a second connection without the flag, or flips one `PRAGMA`, walks straight through. **A refused `INSERT` on SQLite is evidence about the connection string, not about the database.** On Postgres and MySQL the same test is evidence about the database, because the permission lives in the database and the connection cannot grant itself one.

**What *is* enforced outside the connection is the file's permissions.** Verified — the file made read-only, then opened by an ordinary read-write connection that asks for no special mode:

```
file mode now: 444   os.access(W_OK)=False
### read-only FILE, ordinary connection, SELECT
    -> (4,)
### read-only FILE, ordinary connection, INSERT
    !! OperationalError: attempt to write a readonly database
```

Reads work, the write is refused, and the refusal comes from the filesystem rather than from anything the connection asked for. Note the error is **word for word identical** to the `mode=ro` one, so the message cannot tell you which of the two you are looking at.

That boundary is real **only while the account the app runs as cannot change it** — and on an ordinary single-user machine it can:

```
### the same process removes the read-only bit and writes
    -> committed
```

```
--- the same account removes its own deny entry ---
Successfully processed 1 files; Failed processing 0 files
INSERT after the same account removed its own deny entry -> committed
```

So: unless the database file is owned by an OS account the app cannot become, with permissions the app's own account cannot alter, a read-only file is a slower promise in code rather than an enforced boundary.

One practical caution from the same run, on Windows: a blunt `icacls <file> /deny "<user>:(W)"` stopped **reads** as well — `sqlite3.OperationalError: unable to open database file`, even connecting with `mode=ro`. Removing the write bit (`chmod 0400` / the read-only file attribute) left reads working; a deny ACE did not. Test the read after changing permissions, not just the write.

**So, for SQLite:**

1. **Prefer API-only reads.** Everything in this file above the database section still applies; the app's own API enforces its own rules and needs no database identity at all.
2. If direct reads are genuinely required, say to the owner, in the same breath as offering it, that **the database cannot enforce this and the file permissions are doing the work** — and that the protection is only as strong as the account the app runs under.
3. Do not present `mode=ro` as the safety mechanism. Set it — it is a seatbelt against an honest mistake, and it costs nothing — but do not let it be what anyone is relying on, and do not let a refused `INSERT` through it be recorded as the proof.

### Where you cannot create a user

Where you cannot create a user — a managed database, no admin access — say so plainly and fall back to API-only reads. Do not use the app's own credentials.

The app's own database user can write, by definition; borrowing it puts the whole "enforced, not promised" rule back to nothing while looking like it was followed. A connection string that works is not a connection string that is safe, and nothing downstream will ever tell you which one you have.

---

## What the user must be told, in these words or closer

**When the credential is chosen:**

> This assistant will act as one fixed login. Anything that login can see or
> change, the assistant can see or change too. There is no per-person permission
> screen, and no way to give one person less access than another.

**When the database fallback is chosen:**

> Reading your database directly means the assistant can see rows your app would
> normally hide — anything your app filters in its own code, like "only show a
> customer their own orders", the database does not know about.

Say both at the point the choice is made, not once the thing is built. Neither is a footnote and neither is a warning about a risk that might happen — both describe how it will work on the first day and every day after.

---

## Verify

- [ ] One unauthenticated request was sent and its `WWW-Authenticate` header read, before any credential code was written. The scheme word came from that header, not from a guess.
- [ ] The credential was tested with the **wrong** scheme word as well as the right one. A 200 from the wrong word means the mechanism is not what it was thought to be.
- [ ] A 401 saying "credentials were not provided" / "Not authenticated" was read as *wrong scheme word*, not as *missing header*, before the header plumbing was investigated.
- [ ] Where a token-auth path exists it was used, rather than a session cookie.
- [ ] No route was exempted from CSRF protection, and no such change was suggested. No file the owner wrote was edited.
- [ ] Django session login: the login POST was checked for **302**, not 200. A 200 was treated as a failed login.
- [ ] Django writes: both the `csrftoken` cookie and the matching `X-CSRFToken` header were sent, plus a same-origin `Referer`, and a write was actually performed and the row confirmed — not assumed from a successful read.
- [ ] Django 403s: the reason line inside the body was read, not just the status code. `CSRF token missing`, `CSRF cookie not set` and `Referer checking failed` are three different faults behind one status.
- [ ] Express: the guard middleware was read for the exact header name, and the credential was tested against **every** route, not just one.
- [ ] FastAPI: the token request was sent **form-encoded**. A 422 naming `username` and `password` as missing was read as wrong encoding, not wrong credentials.
- [ ] A dedicated account was created for the agent where the app allows it, rather than reusing a person's login.
- [ ] Database fallback: `agent_readonly` was created, connected as, and an `INSERT` attempted. The real permission error was seen. `UPDATE` and `DELETE` were attempted too.
- [ ] Postgres: the grants were run as the **same role the app's migrations run as**, or a plan exists to re-run `GRANT SELECT ON ALL TABLES` after deployments that add tables.
- [ ] Postgres: every schema the app uses was granted, not `public` alone.
- [ ] SQLite: no read-only user was claimed. `mode=ro` / `immutable=1` / `query_only` were not presented as an enforced boundary, and the owner was told the file permissions are doing the work — or the fallback was declined in favour of API-only reads.
- [ ] The app's own database credentials were not reused for the fallback.
- [ ] The one-identity disclosure was said when the credential was chosen, and the row-visibility disclosure when the database fallback was chosen — at the point of the choice, in these words or closer.
