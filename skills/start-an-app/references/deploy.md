# Deploying

Last verified: 2026-07-28

**Purpose:** Take the app that runs on the user's machine and make it work at a real web address, for real people. **Loaded only when the user has said yes to the offer in Step 5** — putting something on the internet is theirs to decide, not a finishing touch.

The commands below are Vercel's, because it is the shortest path for a Next.js app and needs no configuration. The *shape* is the same on any host: the app reads environment variables that live in files on the user's laptop, and the deployment cannot see a single one of them. Everything here follows from that.

## The failure this prevents

A deploy that skips this file does not error. It builds, it goes green, the URL loads, the landing page renders — and then the first click fails, or sign-in refuses every password, or the app reads an empty database. The user is looking at what appears to be a working site, which is the worst possible place to debug from.

So: **check before deploying, and verify on the live URL afterwards.** Not locally. Locally it already works; that is the problem.

## If the app was built on the SQLite branch, start here

SQLite lives in a file inside the project. Most hosts give a deployment a read-only or ephemeral filesystem, and every instance gets its own — so the file either cannot be written or silently stops being shared, and data appears to vanish between requests.

**Deploying a SQLite app means moving it to Postgres first**, which is the `references/database.md` Postgres branch with a hosted connection string: same Drizzle schema, same queries, `pnpm db:generate` and `pnpm db:migrate` against the new database, and the driver swapped in `src/lib/db/index.ts`. Say this plainly and early rather than after a confusing deploy — it is a twenty-minute job, and it is much easier before there is data in the file worth keeping.

## Step 1 — What does the app actually read?

Find every environment variable the source depends on:

```bash
grep -rho "process\.env\.[A-Z0-9_]*" src | sort -u
```

That list — minus anything the host sets itself (`VERCEL_*`, `NODE_ENV`) — is what production needs. Then see what is actually there:

```bash
vercel env ls production
```

Compare the two lists by hand. It takes ten seconds and it is the single highest-value check in this file.

What is typically missing, and why:

| Variable | Why it's missing | Value |
|---|---|---|
| `POSTGRES_URL` | Local Docker's `localhost:5432` is meaningless to a deployment | The hosted database's connection string |
| `BETTER_AUTH_SECRET` | Written into `.env`, which is gitignored and never uploaded | **Generate a fresh one** — see below |
| `BETTER_AUTH_URL` | Same, and its local value is `http://localhost:3000` | The deployed origin — see Step 3 |
| `OPENROUTER_API_KEY`, payment keys, Google client id/secret | All hand-written into `.env` | The same values, unless going properly live |
| Anything `NEXT_PUBLIC_` | Inlined at **build** time, so it must exist before the build, not just at runtime | Whatever it should be in production |

`BLOB_READ_WRITE_TOKEN` is the exception that looks like the rule: connecting a Blob store in the dashboard sets it for you. `references/storage.md` covers it.

**Generate a new auth secret for production rather than reusing the development one.** It is one command, and it means a secret that has sat in a local file, in shell history, and possibly in a screenshot is not the one signing real users' sessions:

```bash
openssl rand -base64 32
```

Add each value:

```bash
printf '%s' "<value>" | vercel env add NAME production
```

To **change** one that already exists, remove it first — `add` does not overwrite:

```bash
vercel env rm NAME production --yes
printf '%s' "<new value>" | vercel env add NAME production
```

**Environment variables are read at deploy time, so changing one changes nothing until you redeploy.** Every fix in this file ends with another deploy.

### Values added by the CLI do not read back

`vercel env pull` writes an empty string for variables added this way — they are stored write-only. `NAME=""` in the pulled file means "cannot be shown", **not** "was saved empty, add it again". Re-adding them because the pull looked wrong is a loop that ends where it started.

The way to confirm a value landed is to use the deployed app. That is what Step 5 is for.

## Step 2 — Deploy

```bash
vercel deploy --prod --yes
```

`--prod` puts it on the project's own address rather than a one-off preview URL. Without it the user gets a preview deployment, which is fine for a look on a phone but is protected (see Step 3) and is not what they mean by "online".

Expect to deploy **twice**: the first one tells you the address, and the address is what `BETTER_AUTH_URL` has to be. That is the process, not a retry — say so.

## Step 3 — The address, and why it might not be public

Two things surprise people, and both look like the app is broken.

**The name may be taken.** `<project>.vercel.app` is global across every account, not per-user. If someone else has it, the CLI silently assigns a variant — `<project>-alpha.vercel.app` — and reports it in one line that is easy to read past. Read the `Aliased` line; that is the real address.

**Only the project's own production domain is public.** Deployment Protection covers everything else, including a domain attached by hand with `vercel alias set`: it answers `302` to a Vercel login page, so it works perfectly for the signed-in owner and shows a sign-in screen to everybody they send it to. The person who tests it is the one person who cannot reproduce the problem.

Check with a request, not a browser you are already logged into:

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://<domain>/
```

`200` is public. `302` is protected.

To get a different name, change what the production domain follows — the project name — rather than bolting an alias on the side:

```bash
vercel project rename <old-name> <new-name>
vercel domains add <new-name>.vercel.app <new-name>
```

**Renaming a project is the user's call.** The old address stops being canonical and anyone already sent a link is affected. Ask; a tidier URL is not worth a surprise.

## Step 4 — Sign-in after the move

Better Auth checks the origin of every sign-in request against its configured URL. `references/auth.md` documents this as a localhost port problem; deployment is the same failure with a different cause, and the message — `Invalid origin` — still says nothing about what is actually wrong.

- `BETTER_AUTH_URL` must be the deployed origin exactly: scheme, host, no trailing slash.
- If **more than one hostname** serves the app, the extras must be trusted explicitly or a sign-in from the one the user actually typed is refused:

  ```ts
  export const auth = betterAuth({
    // ...
    trustedOrigins: [
      "https://<the app's domain>",
      "https://<any other domain that serves it>",
    ],
  });
  ```

- Google sign-in, if it was set up, needs the production callback URL added in the Google console: `https://<domain>/api/auth/callback/google`. Google matches exactly, and `redirect_uri_mismatch` never mentions that the deployment is new.

## Step 5 — Verify, on the live URL

Locally proves nothing here. Every check is against the deployed address.

- `curl -s -o /dev/null -w "%{http_code}" https://<domain>/` returns **200**, not a redirect to a login page.
- The front door renders, and any `NEXT_PUBLIC_` flag has taken effect — those are baked in at build time, so a wrong one needs a rebuild, not just a redeploy.
- **Sign up with a new account on the live site, sign out, sign back in.** One action that exercises `POSTGRES_URL`, `BETTER_AUTH_SECRET` and `BETTER_AUTH_URL` together: if all three are right it works, and if any one is wrong it doesn't.
- **Create one real record through the app's own UI and reload.** That proves the deployment is talking to the database the migrations ran on, which a page that merely renders does not.
- Uploads, if set up: upload a file and confirm it renders after a refresh — where a missing Blob store shows up.
- Payments, if set up: the test-mode checkout still completes from the deployed origin.
- The host's runtime logs are clean after all of the above (`vercel logs <deployment-url>`).

## Hand off

In plain words:

- The address, and that anyone with the link can open it.
- That the database behind it is the **live** one. Anything they do on the live site is real.
- Which environment variables now exist in two places, and that changing one on the host needs a redeploy to take effect.
- That every future push gets its own preview URL, private to them.

**A real launch is still a separate conversation** — a custom domain, payments out of test mode, search engines. Deploying is where this skill stops.
