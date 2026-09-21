# Agent access (MCP)

Last verified: 2026-09-21

**Purpose:** Let AI agents — Claude, Claude Code, ChatGPT, Cursor, anything that speaks MCP — do the app's real work on behalf of a signed-in user. The app gains a second front door: the same actions, the same ownership rules, the same log, reached over an authenticated protocol instead of a browser. There are no API keys to mint, paste or rotate: the agent comes in through the app's own sign-in, and the person approves it on a consent screen.

**This file builds agent access *into* a Next.js app you control.** The tools call the same functions the buttons call, each person approves their own access, each can revoke it, and the agent can only ever do what that person could do. If the app's code is not to be touched — it isn't Next.js, it isn't theirs to edit, or they want something quicker — the `bring-your-own-agent` skill puts agent access *beside* the app instead. That is simpler and faster, but it works as one fixed login: no permission screen, and no way to give one person less access than another. Where this path is available it is the better one, which is why `bring-your-own-agent` points people here first when it can and leaves the choice with them.

> **Hard rule: every tool takes the user from the token and scopes every query to them. Never trust an id the model passed you.** A tool call arrives carrying the user's full authority, but the thing that *triggered* it may be text the user never wrote — a web page the agent read, an email it summarised, another tool's output. In the browser a person can only click what they already own; a tool argument is a string somebody else may have chosen. `logged()` below exists to make this mechanical rather than something to remember, and every tool goes through it.
>
> **Second hard rule: a valid token is not enough.** A token is a signed note that stays good for up to an hour whatever happens in the meantime. The endpoint asks the database on every call whether the user still exists and whether they have revoked this connection. Without that check, the Revoke button is a promise the app does not keep. See *A valid token is not enough* below.
>
> If Better Auth's or Anthropic's documentation and this file disagree on *how to structure this*, this file wins. If they disagree on a method name, an option, or an endpoint path, their docs win — update this file afterwards, because both of these move quickly.

**Prerequisite: sign-in must exist.** Tools act as somebody, and the whole authorisation flow hangs off the auth config, so there is nowhere to put this otherwise. If the user asked for agent access but said no to accounts, set up `references/auth.md` first and explain it in a sentence ("an agent has to sign in as *you*, so the app knows whose data it's touching") rather than treating it as a blocker.

> **Almost every example online is wrong, in six specific ways.** This area churned hard and the search results have not caught up — Step 2's research is not optional here. (1) Better Auth's MCP support **moved into its own package, `@better-auth/mcp`**. The `mcp()` and `oidcProvider()` plugins that used to ship inside `better-auth` are gone from its exports, and the new `mcp()` is a different thing: it *is* the OAuth provider, configured for MCP, and it cannot sit beside a separate `oauthProvider()`. Tutorials importing `mcp` from `better-auth/plugins` do not compile. (2) Several options this file itself once used no longer exist: `validAudiences` (now `resource`), `verifyAccessToken` on the resource client, string lifetimes like `"1h"` (now a number of seconds), and `clientDiscovery` as a top-level option. `jwt()` is now required beside the plugin. (3) The MCP TypeScript SDK **split in two**: `@modelcontextprotocol/server` and `@modelcontextprotocol/client` replace the single `@modelcontextprotocol/sdk`. Examples written against the old package use variadic `server.tool()`, pass a bare shape to `inputSchema` rather than a whole schema, and read `extra.authInfo` instead of `ctx.http?.authInfo`. (4) Anything telling you to provision Redis, or to set `basePath`, `maxDuration` or `redisUrl`, is describing a retired adapter. (5) Anything describing an `initialize` handshake, an `Mcp-Session-Id` header, a GET stream for server messages, or resumability via `Last-Event-ID` predates the stateless revision and is describing a protocol that no longer exists. (6) Better Auth's CLI moved too: the old `@better-auth/cli` package is frozen and marked unsupported; the current one is the `auth` package. If something here doesn't compile, read the installed package's own README and type declarations in `node_modules` — never a blog post.

## What you're building

Three parts, and it helps to say them out loud to the user in this order:

1. **An authorisation server.** Already there — it's Better Auth. The `mcp()` plugin adds the OAuth endpoints an agent needs to ask permission.
2. **A resource server.** One route, `/mcp`, that checks the token, checks the connection is still allowed, and runs the tool.
3. **The tools.** The app's own verbs, the same ones the buttons call.

**There is no session.** MCP is a stateless request/response protocol: no handshake to complete, nothing held open, nothing remembered between calls. Every request arrives carrying its own protocol version and the identity of the client that sent it, and the route is a pure function of that request and the token on it. This is why nothing below provisions Redis, and why the endpoint needs no sticky routing — it scales behind an ordinary load balancer, or a serverless platform that never runs the same instance twice. The SDK's handler still answers older clients, statelessly, from the same tools, so this is not a compatibility decision anybody has to make.

The one place it shows up in your own code is the tools: **a tool cannot remember anything between calls.** If something has to survive from one call to the next — a paging cursor, the id of a half-finished draft — the server mints it and returns it, and the agent hands it back as an ordinary tool argument. It is never stashed server-side against a connection, because there is no connection to stash it against.

**The endpoint is `[domain]/mcp`, not `/api/mcp`.** Next.js puts route handlers under `src/app/api/` by convention and it is an easy habit to follow straight past this, but nothing in the App Router requires that segment — `src/app/mcp/route.ts` serves `/mcp` perfectly well. It matters because this URL is not an internal detail: it is a string a person types into Claude's connector dialog, reads out to somebody, or puts in a README. `traillog.com/mcp` is what the ecosystem has settled on and what people guess first. Every other API route in the app stays under `/api`; this one is public-facing, so it doesn't. Better Auth keeps its own `/api/auth` base path unchanged.

The agent never sees a password. It gets sent to the app's own sign-in page, the user approves it on a consent screen, and the agent walks away with a scoped token the user can revoke.

## Install

```bash
pnpm add better-auth @better-auth/mcp @better-auth/oauth-provider @better-auth/cimd
pnpm add @modelcontextprotocol/server zod
```

`@better-auth/mcp` is the plugin. `@better-auth/oauth-provider` is what it is built on, and the app imports from it directly for the discovery documents and the client-side consent calls. `@better-auth/cimd` adds client metadata documents, the registration mechanism the spec now prefers.

**`mcp-handler` is not needed.** Earlier versions of this file used it. `requireMcpAuth` from `@better-auth/mcp` has already verified the token by the time the request reaches the MCP handler, and the SDK's own `createMcpHandler` has a `fetch(request, { authInfo })` that takes those verified claims with no adapter in between. `mcp-handler`'s `withMcpAuth` would verify the same token a second time.

`zod` is listed because tool schemas are written with it directly. It is a dependency of this app, not something to rely on inheriting through another package.

If `better-auth` is already installed from `references/auth.md`, upgrade it in the same command rather than leaving it behind — the plugin packages declare it as a peer dependency at the same release, and a skew between them fails later, at a confusing moment, rather than at install time.

Two reasons this file insists on the current release rather than whatever is already in the lockfile: this plugin's surface is still settling (see the box above: four options changed under this file in a few months), and the plugins it replaces carried a refresh-token replay flaw in older versions. This is security-relevant code. Do not build it on a stale install.

**Nothing new goes in `.env`.** `BETTER_AUTH_URL` now does three jobs — the app's origin, the base of the OAuth issuer, and the audience stamped into every token — so in production it has to be the real public URL. Set it to anything else and the flow completes right up to the first tool call, then fails on an audience mismatch with no useful error. That one variable is the whole go-live switch.

## Configure

### One file for the strings that must agree

Several separate things have to be character-identical: the plugin's `resource`, the discovery document's `resource`, the audience checked at verify time, and the URL the user types into Claude. Put the strings in one place so they cannot drift.

`src/lib/mcp-resource.ts`:

```ts
// Blank counts as missing. `??` only falls back on null and undefined.
const present = (value: string | undefined) => (value && value.trim() ? value.trim() : undefined);

const BASE_URL = (
  present(process.env.BETTER_AUTH_URL) ??
  present(process.env.APP_URL) ??
  "http://localhost:3000"
).replace(/\/$/, "");

export const ISSUER = `${BASE_URL}/api/auth`;
export const MCP_RESOURCE = `${BASE_URL}/mcp`;

// Two scopes, named after the app. Not one per table.
export const MCP_SCOPES = ["hikes:read", "hikes:write"] as const;

// What each scope means, in the app's words. The consent screen and the
// Connected apps page show these, never the raw strings.
export const SCOPE_LABELS: Record<string, string> = {
  "hikes:read": "See your hikes",
  "hikes:write": "Add and edit hikes",
  openid: "Confirm who you are",
  profile: "See your name",
  email: "See your email address",
  offline_access: "Stay connected without asking you to sign in each time",
};
```

This file must not import `"server-only"`: `src/lib/auth.ts` imports it, and the Better Auth CLI and any `tsx` script that loads the auth config cannot load that marker.

**Treat a blank variable as a missing one**, which is what `present()` is for. A variable that exists but is empty passes straight through `??`, and `new URL("")` throws. If that value reaches a root layout, every route in the app answers 500. `APP_URL` is the second choice because `references/seo.md` uses it for the same public address; where both are set they must be the same value.

No trailing slash, ever — the spec prefers it absent and a stray one is a mismatch like any other. **`ISSUER` includes `/api/auth`, and this was measured, not assumed:** on a running build, the authorisation-server document's `issuer`, the `iss` claim inside an issued token, and the `iss` parameter on the redirect back from consent were all `http://localhost:3000/api/auth`. Better Auth issues from its own base path, not from the bare origin.

Two scopes, not nine. A consent screen listing `hikes:read`, `hikes:write`, `photos:read`, `photos:write`… is a consent screen nobody reads, which defeats the point of having one.

### The plugin

Extend `src/lib/auth.ts` — this adds plugins alongside whatever is already there:

```ts
import { jwt } from "better-auth/plugins";
import { nextCookies } from "better-auth/next-js";
import { cimd } from "@better-auth/cimd";
import { fetchClientMetadataResource } from "@better-auth/cimd/node";
import { mcp } from "@better-auth/mcp";
import { MCP_RESOURCE, MCP_SCOPES } from "@/lib/mcp-resource";

export const auth = betterAuth({
  baseURL: process.env.BETTER_AUTH_URL,
  // ...existing config

  plugins: [
    // ...existing plugins
    jwt(), // required: the plugin signs access tokens with it
    mcp({
      loginPage: "/sign-in",
      consentPage: "/oauth/consent",
      resource: MCP_RESOURCE,

      // Dynamic registration is the older way for an agent to get a client id.
      // It is OFF by default and both options are needed to switch it on. It
      // stays on beside client metadata documents (cimd, below) because the
      // client picks — see "How the agent gets a client id".
      allowDynamicClientRegistration: true,
      allowUnauthenticatedClientRegistration: true,

      scopes: ["openid", "profile", "email", "offline_access", ...MCP_SCOPES],
      clientRegistrationDefaultScopes: ["openid", "profile", "email"],
      // Written out, NOT derived from MCP_SCOPES. This is the list a stranger
      // may register a client for; a scope added to MCP_SCOPES later must not
      // land here without somebody deciding it should.
      clientRegistrationAllowedScopes: ["offline_access", "hikes:read", "hikes:write"],
    }),
    cimd({
      fetchClientMetadataResource,
      // The MCP spec revision whose metadata-document profile this pins.
      // Step 2's research confirms the current one; do not copy this blind.
      metadataProfile: "mcp-2026-07-28",
    }),
    nextCookies(), // must stay last
  ],

  rateLimit: {
    enabled: true,
    storage: "database",
    customRules: {
      // `references/settings.md` adds its own rules to this object later.
      "/oauth2/register": { window: 60, max: 5 },
      "/oauth2/token": { window: 60, max: 30 },
    },
  },
});
```

What changed from the older shape of this config, so a half-remembered version does not creep back in:

- **`mcp()` replaces `oauthProvider()`.** It is the OAuth provider, configured for MCP, and the two cannot be combined. It takes every option `oauthProvider` does, plus `resource`.
- **`resource` replaces `validAudiences`.** One string: issued tokens are bound to it, it is published as `resource` in the protected-resource document, and it is the expected audience at verify time. HTTPS only, except on a loopback host for local development.
- **`jwt()` is required** unless the plugin's own opt-out is set. Leave it on.
- **Token lifetimes are numbers of seconds.** The defaults — an hour for access tokens, thirty days for refresh tokens — are right for this, so the config above sets neither. `"1h"` is a type error.
- **Client discovery is not a top-level option.** It comes from the `cimd()` plugin sitting beside `mcp()`. `fetchClientMetadataResource` is required, and is imported from `@better-auth/cimd/node`.

The authorisation response should carry the `iss` parameter naming the issuer: the spec says the authorisation server SHOULD send it, and clients MUST check it when it is present. The plugin sends it — measured on the redirect back from the consent screen.

One thing the discovery document advertises that this app never uses: `client_credentials` appears in `grant_types_supported`. A token from that grant has no user as its subject. `logged()` below refuses any token without a `sub`, so such a token cannot reach anybody's data. If the release you install lets you limit grant types to `authorization_code` and `refresh_token`, do it.

### How the agent gets a client id

An agent that has never met this app needs a `client_id` before it can ask for anything. There are two mechanisms, and knowing which is which is what keeps the options above from looking arbitrary.

**Client ID Metadata Documents are what the spec now prefers.** The client's `client_id` *is* an HTTPS URL, and that URL serves a small JSON document describing the client — its name, its redirect URIs. The authorisation server fetches it, checks the document's own `client_id` matches the URL it came from, and validates the redirect URI against it. Nothing is registered, so there is no anonymous write endpoint and no row created per connection. This is what `cimd()` adds, and it puts `client_id_metadata_document_supported` into the discovery document by itself.

**Dynamic registration is the older mechanism and is now deprecated in the spec**, though it stays through a deprecation window. It is what the two `allow*` options enable: the agent POSTs its details and gets a `client_id` back.

Support both, and do not be tempted to leave dynamic registration off because it is the plugin's default. **The client picks**, working down from credentials it already has, to metadata documents if this server advertises them, to dynamic registration if it doesn't — so a server without the last rung strands every agent that hasn't moved up yet. (That order is how clients are described as behaving; it was not observed here.)

`allowUnauthenticatedClientRegistration` sounds alarming and is required for the fallback to work at all: the agent registers *before* anybody has signed in, because signing in is what it is about to ask for. Registration creates a client record, not access — nothing can be read until a human approves it on the consent screen. The rate limit is there because registration is the one endpoint an anonymous caller can reach, and each fresh connection makes a new client row.

A client registering dynamically must declare an `application_type`, and command-line and desktop tools declare `native` so that loopback redirect URIs are accepted. Omitting it means `web`, under which a `localhost` redirect can be rejected outright.

### Generating the schema — the first run needs help

The plugins add a good number of tables: OAuth clients, resources, the link between them, consents, access and refresh tokens, client assertions, signing keys. Settle the whole plugin list before generating, so it is one migration rather than three.

```bash
pnpm dlx auth@latest generate --config src/lib/auth.ts --output src/lib/db/auth-schema.ts -y
pnpm db:generate
pnpm db:migrate
```

**The CLI is the `auth` package.** `@better-auth/cli` is frozen and marked unsupported on the registry; it still installs, which is how it ends up generating a schema with half the tables missing.

Three things broke the first run on a real build, each with an error that does not point at the cause:

1. **The plugin touches its own tables while the config is loading.** On a fresh project those tables are not in the Drizzle schema yet — that is what this command is about to write — so a background lookup rejects and takes the process down before the file is written. Let the run finish anyway:

   ```bash
   NODE_OPTIONS="--unhandled-rejections=warn" pnpm dlx auth@latest generate --config src/lib/auth.ts --output src/lib/db/auth-schema.ts -y
   ```

   Only the first generation needs this. Once the tables are in `auth-schema.ts`, later runs are clean.

2. **If `auth.ts` imports from `auth-schema.ts`, the file has to exist first.** `references/settings.md` makes it do so (the first-account-becomes-admin hook counts users). Write a placeholder the CLI will overwrite:

   ```ts
   // src/lib/db/auth-schema.ts — placeholder, overwritten by `auth generate`
   import { pgTable, text } from "drizzle-orm/pg-core";
   export const user = pgTable("user", { id: text("id").primaryKey() });
   ```

   And make that one import relative — `from "./db/auth-schema"` — because the CLI's loader resolved `@/lib/db` but not `@/lib/db/auth-schema` through the path alias.

3. **Nothing `auth.ts` imports may import `"server-only"`.** The CLI loads the config outside Next, where that marker package either is not installed or throws by design. In practice this means `src/lib/email.ts`, `src/lib/mcp-resource.ts` and the email templates go without it. The same rule covers anything a `tsx` seed script imports.

### Discovery

An agent finds all of this by fetching well-known documents from the **domain root**. The plugin writes the documents, but Better Auth is mounted under `/api/auth` and its catch-all serves nothing outside that path, so the root needs route handlers that hand the request over. Expecting the catch-all to cover it is a half-hour of confusion.

`src/lib/mcp/discovery.ts`:

```ts
import "server-only";
import {
  oauthProviderAuthServerMetadata,
  oauthProviderOpenIdConfigMetadata,
} from "@better-auth/oauth-provider";
import { auth } from "@/lib/auth";

// These documents are public and meant to be read from anywhere, hence the
// open CORS. Nothing else in the app gets these headers.
const cors = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Methods": "GET, OPTIONS",
  "Access-Control-Allow-Headers": "*",
};

function open(handler: (request: Request) => Promise<Response>) {
  return async (request: Request) => {
    const response = await handler(request);
    const headers = new Headers(response.headers);
    for (const [name, value] of Object.entries(cors)) headers.set(name, value);
    return new Response(response.body, { status: response.status, headers });
  };
}

export const OPTIONS = () => new Response(null, { status: 204, headers: cors });

/** Which sign-in server protects /mcp. The mcp() plugin writes the document. */
export const protectedResource = open((request) => auth.handler(request));

/** Where to register, ask for access and swap a code for a token. */
export const authorizationServer = open(oauthProviderAuthServerMetadata(auth));

export const openIdConfiguration = open(oauthProviderOpenIdConfigMetadata(auth));
```

Then one-line routes, each re-exporting from it:

```ts
// src/app/.well-known/oauth-protected-resource/mcp/route.ts
// src/app/.well-known/oauth-protected-resource/route.ts
export { protectedResource as GET, OPTIONS } from "@/lib/mcp/discovery";

// src/app/.well-known/oauth-authorization-server/route.ts
// src/app/.well-known/oauth-authorization-server/api/auth/route.ts
export { authorizationServer as GET, OPTIONS } from "@/lib/mcp/discovery";

// src/app/.well-known/openid-configuration/route.ts
export { openIdConfiguration as GET, OPTIONS } from "@/lib/mcp/discovery";
```

**The protected-resource document is no longer written by hand.** The plugin's request hook recognises both well-known paths when the request reaches `auth.handler`, so the root route only has to forward the request as it arrived. The hand-written version this file used to carry, and its `verify.ts` companion, are gone — delete them if an older build has them.

Why two paths for each document: RFC 9728 and RFC 8414 insert the resource's or issuer's own path after the well-known segment. The resource is `/mcp`, so its document is looked for at `/.well-known/oauth-protected-resource/mcp`; the issuer is `/api/auth`, so its document is looked for at `/.well-known/oauth-authorization-server/api/auth`. Clients differ in which form they try first, and a document served only at the other one is a document they do not find. Mount both.

What the documents said on a running build, which is what to compare yours against: the protected-resource document's `resource` was exactly `MCP_RESOURCE`, its `authorization_servers` had one entry equal to `ISSUER`, and its `scopes_supported` listed the app's two scopes and not `offline_access`. The authorisation-server document's `issuer` equalled `ISSUER`, and it carried both `registration_endpoint` and `client_id_metadata_document_supported: true`.

### Scopes: advertised, required and registrable are three different lists

Each of these cost real debugging time on an earlier build. They are about the OAuth provider's options and about how clients behave, so they apply whichever adapter sits in front of the tools.

**Advertised scopes are not required scopes.** A client asks for exactly what discovery tells it about. `requiredScopes` on the route is the *gate* every token must clear, and it stays at `hikes:read`. Widening it to make the write scope easier to find is the wrong fix: a read-only token would then be turned away at the door, and "approve for reading only" would stop working. Gate on read, advertise both. An earlier adapter reused the gate list as the `scope` in its 401 challenge, so no client ever learned the write scope existed and read-only was the only grant anybody could obtain. If the challenge on your build carries a `scope` value, check it names both.

**There are three advertising surfaces, and the third defeats the obvious fix.**

1. The **401 challenge** on `POST /mcp`.
2. The **protected-resource document** — readable by anyone, CORS-open, cached. A spec-following client reads this *before* it ever sees a 401.
3. The **authorisation-server document**, whose `scopes_supported` is the provider's `advertisedMetadata.scopes_supported` if that is set, and otherwise the whole `scopes` option. A scope **must** be in `scopes`, or the authorise call rejects it as `invalid_scope`.

So a scope meant to stay out of discovery — an admin one, say — is published anyway unless `advertisedMetadata.scopes_supported` is set to the public list. The provider checks that list is a subset of `scopes`, so authorising still works for a client that asks for the private scope deliberately. **Curl all three surfaces when you think you are done.** Getting two of three right is the expected failure.

**Never spread the scope list into `clientRegistrationAllowedScopes`.** That is why the config above writes it out. With `allowUnauthenticatedClientRegistration: true`, a derived list means any stranger can register a client pinned to any scope you ever add, an admin one included, and the type-checker will not object.

**`offline_access` must be advertised by the authorisation server.** It is not a scope of the resource at all. It is what a client asks for to be issued a refresh token, and Claude Code asks for it as a matter of course. Dynamic registration pins a client to the list it discovered at registration time. Leave `offline_access` out of `scopes` or out of the registrable list and registration succeeds, the client asks for a refresh token as usual, and the authorise call fails with `invalid_scope` *before any consent screen is drawn*. Nothing on screen explains why. It belongs in the authorisation-server document and not in the protected-resource document, which is what the measured documents above show.

### The endpoint

`src/app/mcp/route.ts` — note the path, per the rule above:

```ts
import { requireMcpAuth } from "@better-auth/mcp";
import { createMcpHandler, McpServer, originValidationResponse } from "@modelcontextprotocol/server";
import { auth } from "@/lib/auth";
import { MCP_RESOURCE } from "@/lib/mcp-resource";
import { connectionStillAllowed } from "@/lib/mcp/still-allowed";
import { registerTools } from "@/lib/mcp/tools";

export const runtime = "nodejs";

const mcp = createMcpHandler(() => {
  const server = new McpServer({ name: "traillog", version: "1.0.0" });
  registerTools(server);
  return server;
});

// A missing or bad token is a real 401 with WWW-Authenticate, which is what
// makes an agent start sign-in. Reading is the least a connection needs; the
// tools that write check hikes:write themselves.
const authed = requireMcpAuth(
  auth,
  async (request, claims) => {
    const scopes = typeof claims.scope === "string" ? claims.scope.split(" ").filter(Boolean) : [];
    const rawClientId = claims.client_id ?? claims.azp;
    const clientId = typeof rawClientId === "string" ? rawClientId : "";

    // See "A valid token is not enough".
    if (!(await connectionStillAllowed(String(claims.sub ?? ""), clientId))) {
      const metadata = `${new URL(MCP_RESOURCE).origin}/.well-known/oauth-protected-resource/mcp`;
      return new Response(
        JSON.stringify({
          jsonrpc: "2.0",
          error: { code: -32001, message: "This connection has been revoked." },
          id: null,
        }),
        {
          status: 401,
          headers: {
            "Content-Type": "application/json",
            "WWW-Authenticate": `Bearer error="invalid_token", error_description="This connection has been revoked", resource_metadata="${metadata}"`,
          },
        },
      );
    }

    return mcp.fetch(request, {
      authInfo: {
        token: request.headers.get("authorization")?.replace(/^\S+\s+/, "") ?? "",
        clientId,
        scopes,
        expiresAt: claims.exp,
        // The user's id comes from the token's subject and nowhere else.
        extra: { userId: claims.sub },
      },
    });
  },
  { resource: MCP_RESOURCE, requiredScopes: ["hikes:read"] },
);

// POST only. No CORS headers here on purpose: this route carries the token,
// so a browser page on another site is refused.
export async function POST(request: Request) {
  const refused = originValidationResponse(request, [new URL(MCP_RESOURCE).hostname]);
  return refused ?? authed(request);
}
```

`requireMcpAuth` checks the token's signature, issuer, audience and expiry against the app's own signing keys. No token, or a bad one, gets a `401` carrying `WWW-Authenticate: Bearer resource_metadata="…"`; a token missing a required scope gets a `403` with `insufficient_scope` naming what is missing, so a client can ask for more in one round trip. Measured on a running build: an empty `POST /mcp` answered `401` with that header, and `GET /mcp` answered `405`.

**POST only.** Older versions of this route exported `GET` for a standing message stream and `DELETE` to tear a session down; both went when sessions did. Not exporting them means Next.js answers `405 Method Not Allowed`, which is exactly what the spec asks a current server to say to a client still trying the old shapes.

Do not give this route the permissive CORS headers the discovery documents get. Those are public metadata and are meant to be readable from anywhere; this is the route that carries the token. A server is required to check the `Origin` header when one is present and refuse a request that doesn't belong — `originValidationResponse` does it — and a blanket `Access-Control-Allow-Origin: *` copied down from the well-known routes quietly removes that.

Do not soften the 401 into a friendly error object. A `200` carrying `isError: true` is an ordinary tool failure, so Claude hands the text to the model and moves on. Only a real 401 makes it stop and ask the user to sign in.

### A valid token is not enough

Found on a real build, and the reason this section exists: access tokens are signed notes checked without the database, and they live for an hour. Deleting a consent removes a row. It does not reach into the agent's memory and take the token back. So a Revoke button wired only to "delete the consent" leaves the agent working for up to an hour after the user believed they had cut it off — and the same is true after the account itself is deleted.

The fix is one small query on every call, before the tool runs.

`src/lib/mcp/still-allowed.ts`:

```ts
import "server-only";
import { and, eq } from "drizzle-orm";
import { db } from "@/lib/db";
import { oauthConsent, user } from "@/lib/db/schema";

/**
 * On every call: does the user still exist, and have they left this
 * connection approved? This is what makes Revoke stop the next call.
 */
export async function connectionStillAllowed(userId: string, clientId: string): Promise<boolean> {
  if (!userId || !clientId) return false;
  const [row] = await db
    .select({ id: oauthConsent.id })
    .from(oauthConsent)
    .innerJoin(user, eq(oauthConsent.userId, user.id))
    .where(and(eq(oauthConsent.userId, userId), eq(oauthConsent.clientId, clientId)))
    .limit(1);
  return Boolean(row);
}
```

Where the app has an admin-only agent surface, add the role to the `where`. The inner join is what covers a deleted account: no user row, no match.

When it returns false the route answers a real `401` with `WWW-Authenticate`, as above, so the agent goes back to sign-in and the consent screen rather than reporting a tool error.

It relies on a consent row being written when the user clicks Allow. That was measured: after approving a dynamically registered client, `oauth_consent` held one row with the client id, the user id and the granted scopes.

**Be honest about how far this was tested.** On the build this was found in, the check was written, compiled, and sat in front of six successful tool calls made with a real token — so it does not wrongly refuse an approved connection. The other half, *revoke and then call again with the same unexpired token*, **was not run** before that session ended. It is the first item in Verify below for that reason. Run it; do not inherit the assumption.

Revoking should also remove what the database *can* remove, so the agent cannot quietly fetch itself a fresh token. See *Connected apps* below.

## The tools

### They call the same code the buttons call

Put the app's reads and writes in `src/lib/<domain>.ts` — `listHikes({ userId, limit })`, `createHike({ userId, ... })` — and have both the server actions and the tools call those. A tool that writes its own query is a tool whose ownership check drifts from the one the UI enforces, and nobody notices until the drift is a leak. One function, two callers, one `where`.

Have those functions throw a small domain error class for anything a person could act on — "That time isn't free any more. Pick another." — with the message written to be shown as it stands. The page prints it next to the form; the wrapper below hands the same sentence to the model.

Guard ids at the door of those functions too. An id arriving from an agent, or from a URL, that is not shaped like the app's ids should come back as "not found" before it reaches the database, where a malformed UUID is a thrown exception rather than an empty result.

**Guards that read `headers()` do not work here.** Identity inside `/mcp` is a bearer token, not a session cookie, so a helper that calls `auth.api.getSession({ headers: await headers() })` finds nobody. A Next navigation throw (`redirect()`, `notFound()`) surfaces as a 500 rather than a refusal. These shared functions take `userId` as an argument and never look for a session themselves, and an admin check needs its own `isAdmin(userId)` that reads the row.

### The wrapper that enforces the hard rule

`src/lib/mcp/log.ts` — every tool goes through this, which is what makes "take the user from the token" structural instead of a thing to remember:

```ts
import "server-only";
import type { CallToolResult, ServerContext } from "@modelcontextprotocol/server";
import { db } from "@/lib/db";
import { mcpCallLog } from "@/lib/db/schema";
import { SCOPE_LABELS } from "@/lib/mcp-resource";
import { AppError } from "@/lib/errors"; // the app's own "a person can act on this" error

// What a tool hands back: the data for the agent, and how many rows of the
// user's data that was, for the call log.
export type ToolOutput = { data: unknown; rows?: number };

function text(value: string, isError = false): CallToolResult {
  return { content: [{ type: "text", text: value }], ...(isError ? { isError: true } : {}) };
}

export function logged<A>(
  tool: string,
  scope: string,
  run: (args: A, userId: string) => Promise<ToolOutput>,
) {
  return async (args: A, ctx: ServerContext): Promise<CallToolResult> => {
    const authInfo = ctx.http?.authInfo;
    const userId = typeof authInfo?.extra?.userId === "string" ? authInfo.extra.userId : "";
    // Also what stops a client_credentials token, which has no subject.
    if (!userId) throw new Error("No user on this token");

    const clientId = authInfo?.clientId || null;
    const startedAt = Date.now();
    const write = (row: { ok: boolean; rowCount?: number; error?: string }) =>
      db.insert(mcpCallLog).values({
        tool,
        userId,
        clientId,
        args: args ?? null,
        ok: row.ok,
        durationMs: Date.now() - startedAt,
        rowCount: row.rowCount ?? null,
        error: row.error ?? null,
      });

    // The route only checks that the connection may read. Whether it may
    // write is decided here, per tool.
    if (!authInfo?.scopes.includes(scope)) {
      const message = `This connection wasn't allowed to: ${SCOPE_LABELS[scope] ?? scope}. Connect again and allow it.`;
      await write({ ok: false, error: message });
      return text(message, true);
    }

    try {
      const { data, rows } = await run(args, userId);
      await write({ ok: true, rowCount: rows });
      return text(JSON.stringify(data));
    } catch (error) {
      const message = error instanceof Error ? error.message : String(error);
      await write({ ok: false, error: message });
      // An AppError is written to be read, so the model gets it as a result
      // it can act on. Anything else is a fault: logged, then rethrown.
      if (error instanceof AppError) return text(message, true);
      throw error;
    }
  };
}
```

**A domain error comes back as a result, not a thrown exception.** Measured: asking to book a slot that had just been taken returned `isError: true` with the text "That time isn't free any more. Pick another." — which is a sentence a model can read and respond to by offering another time. A thrown error reaches it as a generic failure with nothing to act on.

**Whatever a tool returns goes to the model.** `logged()` stringifies `data` and hands it over as the result. So no tool may ever return a token, a signup link, a password hash or any other secret. Pick out the fields you return; do not spread a database row that happens to carry one.

The table goes in `src/lib/db/schema.ts` alongside the app's own — Postgres branch shown, SQLite uses `text` ids and `integer` timestamps per `references/database.md`:

```ts
export const mcpCallLog = pgTable("mcp_call_log", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: text("user_id").references(() => user.id, { onDelete: "set null" }),
  clientId: text("client_id"),
  tool: text("tool").notNull(),
  args: jsonb("args"),
  ok: boolean("ok").notNull(),
  durationMs: integer("duration_ms"),
  rowCount: integer("row_count"),
  error: text("error"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});
```

`pnpm db:generate` then `pnpm db:migrate` — this one is the app's own table, so it does not need the Better Auth CLI.

**`args` records what the agent asked for**, as the tool received it, after schema defaults. It is what makes the log answer "why did it say it couldn't find anything?" — the answer is usually visible in the arguments the model guessed. It also means the log holds whatever the agent was told to write: a name, a phone number. It is shown only to the person whose data it is, and to admins; say so where the log is rendered.

`onDelete: "set null"` rather than `cascade`, for the same reason `references/ops.md` gives: the record of what an agent did outlives the account it did it to.

**This log records reads as well as writes**, which is the one place this skill departs from `references/ops.md`'s "log writes, not reads". When the caller is a person, a read is noise. When the caller is an agent, a read *is* the event worth seeing — it is how data leaves the app. Tools that write should additionally make sure the ordinary activity log says `via: "agent"`, so it keeps telling the truth about who changed what.

**One gap to know about:** a call the SDK rejects before any tool runs — arguments that fail the schema, a tool name that does not exist — never reaches `logged()`, so it is not in the log. Measured: a request for five times the allowed `limit` came back as an input validation error and left no row. Everything that reaches a tool is logged, scope refusals and domain errors included.

### Writing a tool

```ts
// src/lib/mcp/tools.ts
import "server-only";
import { z } from "zod";
import type { McpServer } from "@modelcontextprotocol/server";
import { listHikes, createHike } from "@/lib/hikes";
import { logged } from "./log";

export function registerTools(server: McpServer) {
  server.registerTool(
    "list_hikes",
    {
      title: "List hikes",
      description:
        "List the signed-in person's hikes, newest first. Use to answer questions about what they have walked, or to find a hike before changing it.",
      inputSchema: z.object({
        since: z.string().optional().describe("Only hikes on or after this date (YYYY-MM-DD)."),
        limit: z.number().int().min(1).max(50).default(20),
      }),
      annotations: { readOnlyHint: true },
    },
    logged("list_hikes", "hikes:read", async ({ since, limit }, userId) => {
      const hikes = await listHikes({ userId, since, limit });
      return { data: hikes, rows: hikes.length };
    }),
  );

  server.registerTool(
    "log_hike",
    {
      title: "Log a hike",
      description: "Record a hike the person has been on. Ask them for the trail and date if either is missing.",
      inputSchema: z.object({
        trail: z.string().min(1).max(200),
        date: z.string().describe("YYYY-MM-DD"),
        distanceKm: z.number().positive().optional(),
        notes: z.string().max(2000).optional(),
      }),
      annotations: { readOnlyHint: false, destructiveHint: false },
    },
    logged("log_hike", "hikes:write", async (input, userId) => ({
      data: await createHike({ userId, ...input }),
      rows: 1,
    })),
  );
}
```

`inputSchema` takes a whole `z.object({...})`, not a bare shape — the shape overload still compiles and is marked deprecated — and the registration function is `registerTool`; `server.tool()` is from the retired SDK. A tool with no arguments still passes `z.object({})`.

**The scope argument is per-tool for a reason.** `requiredScopes` on the route decides who may *connect*; it cannot decide who may write, because it sees the same value for every call. Without the check inside `logged()`, a connection the user approved for reading only can call every write tool in the app, and the consent screen they read becomes a lie.

Where a value has two audiences, send it twice: words for the person the model is talking to ("Friday, September 25, 10:00 AM") and the machine form for the next tool call (the ISO instant). A model handed only the ISO string does timezone arithmetic, badly.

### The rules that make tools good rather than merely present

- **Task-shaped, not table-shaped.** Name them after what a person would ask for — `log_hike`, `weekly_summary` — not `create_row` or `query_table`. A mechanical CRUD-per-table mapping technically works and produces a tool list an agent uses badly, because no single tool matches anything anyone actually wants.
- **Reads and writes are separate tools.** Never one tool with a `method` or `action` argument. Anthropic rejects catch-all tools outright in connector review, and it makes the next rule impossible.
- **Every tool declares `readOnlyHint: true` or `destructiveHint`.** These drive whether Claude runs a tool without asking. Label a write as read-only and it will fire without confirmation.
- **The description says what it does and when to use it.** It does not contain instructions aimed at the model, and it does not oversell — a description that claims more than the tool does is how an agent picks the wrong one. Say where the ids a tool needs come from ("ids come from `list_hikes`").
- **A tool that is missing something says so in its description and lets the model ask.** `log_hike` above does this — "Ask them for the trail and date if either is missing" — and for an app like this it is the whole answer. Do not reach for sampling, roots, or logging to get it instead: all three are deprecated in the spec. The protocol's own route for a server that needs more input is to return an `input_required` result the client retries, and it is rarely worth the machinery here.
- **Every list tool takes a `limit` with a hard maximum.** Results are capped in the clients — Claude Code's default is a token budget in the tens of thousands — and an uncapped list either truncates mid-JSON into something unparseable, or hands a single call the entire account. Cap it in the schema *and* in the query, and return a cursor if there is more.
- **If a count is worth returning, the thing itself is usually worth a tool.** A tool that only reports *how many* comments or files something has leaves the agent acting blind on them. Reads matter as much as writes.
- **Destructive tools need a reason to exist.** The default is no delete tool at all. Throwing work away is a decision for a person, in the app, where they can see what they are throwing away. If the interview genuinely called for it, require an id the agent had to read first, mark it `destructiveHint: true`, and say so in the description.
- **Names stay under 64 characters** and read as `verb_noun`.

Pick the four or five tools that cover what the user said the app is *for* in Step 1a. A short list of tools that match real intentions beats a complete list that mirrors the schema.

## What the user can see and control

An agent working out of sight is exactly the thing this skill says an app must never have. Three pieces, and none of them is optional.

### Sign-in has to carry the request through

The agent sends the user to `loginPage` with a signed OAuth query in the URL. Put `oauthProviderClient()` from `@better-auth/oauth-provider/client` on the auth client:

```ts
// src/lib/auth-client.ts
import { createAuthClient } from "better-auth/react";
import { inferAdditionalFields } from "better-auth/client/plugins";
import { oauthProviderClient } from "@better-auth/oauth-provider/client";
import type { auth } from "@/lib/auth";

export const authClient = createAuthClient({
  plugins: [inferAdditionalFields<typeof auth>(), oauthProviderClient()],
});
```

With that in place the sign-in form needs no special case. The plugin copies the page's OAuth query into the `signIn.email` request, the server answers with a redirect instead of the usual body, and Better Auth's client follows it to the consent screen. The only thing the form must do is *not* push to the dashboard when that happened:

```ts
const { data, error } = await signIn.email({ email, password });
if (error) { /* show it */ return; }
if (data?.redirect && data.url) return; // already on its way to consent
router.push("/dashboard");
```

### The consent screen

`src/app/oauth/consent/page.tsx`, styled like the rest of the app — this is where somebody decides whether Claude gets their data, and a page that looks nothing like the app they just signed into reads as a phishing attempt.

A server component does the reading: `client_id` and `scope` are in the query string, and `auth.api.getOAuthClientPublic({ query: { client_id }, headers })` returns the client's registered name and URL. **If the visitor is signed out, redirect to sign-in keeping the whole query string** — a plain "must be signed in" guard drops it, and the user signs in to find the app that asked has been forgotten. Then show:

- **Who is asking**, by the client name it registered with, and a plain warning that this name is chosen by whoever is connecting.
- **What it will be able to do**, in the app's own words from `SCOPE_LABELS`. "Read your hikes" and "Add and edit hikes" — never the raw scope strings. A scope the page has no label for gets a line saying so, not silence.
- **Whose account it will act as** — their email, visible, so a signed-in-as-the-wrong-person mistake is caught here.
- Two buttons of equal weight. Allow is not the primary-coloured one.

The buttons are a client component calling `authClient.oauth2.consent({ accept })`. The plugin attaches the page's signed query itself, so nothing about the request is sent from your code; follow the URL that comes back.

**Consent is all-or-nothing unless you build the choice.** The provider does support granting a subset: the consent call takes an optional space-separated `scope`, checks it is a subset of the signed original, and the narrowed set flows into the token. But the page has to draw the checkboxes and post the narrowed list. Without that, "approve read-only and upgrade later" is advice nobody can follow. Three rules if you build it: never post `scope: ""` (send a refusal instead), and always pass `openid` and `offline_access` through untouched, or you silently lose the id_token and, an hour later, the refresh token.

Do not auto-approve, and do not skip the screen for "trusted" clients — a client's name is chosen by the client, whether it registered dynamically or handed over a metadata document it hosts itself, so anything can call itself anything.

### Connected apps

`references/settings.md` builds this section: what has access, what it can do, when it was granted, when it was last used, and a Revoke button. `auth.api.getOAuthConsents({ headers })` lists them; last-used comes from `mcp_call_log`.

**Revoke is three things, not one.** Read from the plugin's source on a real build: deleting a consent removes the consent row and nothing else. The agent's refresh token would go on fetching new access tokens, and the plugin's own revocation endpoint needs the client's credentials and the token itself, which the user never has. So the server action does this, through Better Auth's own adapter:

```ts
const consent = await auth.api.getOAuthConsent({ query: { id: consentId }, headers }); // checks ownership
await auth.api.deleteOAuthConsent({ body: { id: consentId }, headers });

const { adapter } = await auth.$context;
const mine = [
  { field: "clientId", value: consent.clientId },
  { field: "userId", value: session.user.id },
];
await adapter.deleteMany({ model: "oauthAccessToken", where: mine });
await adapter.deleteMany({ model: "oauthRefreshToken", where: mine });
```

That stops the agent getting a *new* token. `connectionStillAllowed()` in the route is what stops the one it is already holding. Both are needed, and the page may only say "the next call is refused" once the revoke-then-call test in Verify has actually passed.

**Revoking leaves the registration behind, so clear that too.** The OAuth *client* row survives a revoke, still pinned to the scopes it registered with. Change the scope list afterwards and that client keeps asking with its old one and fails with `invalid_scope` for ever, with no way to recover from inside the app. After the deletes above, drop the client row as well — but only when no other consent still points at it. One client row can serve several people, and revoking your own access must never reach into someone else's. Clients that identified themselves with a metadata document have no row to clear.

### The call log

`references/ops.md` renders `mcp_call_log` on `/settings/system`: which tool, the arguments, which client, worked or failed and why, how long, how many rows went out. This is the panel that answers "what did Claude actually do?" and "why did it say it couldn't find anything?"

## Testing locally

**Claude Code talks to localhost.** No tunnel, no deploy — the whole flow works the moment the routes exist:

```bash
claude mcp add --transport http traillog http://localhost:3000/mcp
```

Then `/mcp` inside Claude Code to run the sign-in. The browser opens the app's own sign-in page, then the consent screen, and the tools appear. This is the loop to iterate in. Flags such as `--scope` or `--header` go before the name.

**The app has to stay on the port in `BETTER_AUTH_URL`.** Tokens and discovery documents are bound to that exact origin. Running the dev server on another port because 3000 was busy produces a flow that completes and a first tool call that fails on the audience.

**Claude.ai and Claude Desktop cannot reach localhost** — they connect from Anthropic's servers, not from the user's machine. To test as a real connector, tunnel:

```bash
cloudflared tunnel --url http://localhost:3000
```

Set `BETTER_AUTH_URL` to the tunnel's public URL and restart the dev server, or every token comes out stamped with the wrong audience. Then add the tunnel URL plus `/mcp` under **Settings → Connectors → Add custom connector** on claude.ai.

If discovery fails, check it by hand before guessing — these must line up exactly:

```bash
curl -s http://localhost:3000/.well-known/oauth-protected-resource/mcp | jq
curl -s http://localhost:3000/.well-known/oauth-authorization-server | jq
curl -si -X POST http://localhost:3000/mcp | head -20
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:3000/mcp
```

The third must be a `401` carrying a `WWW-Authenticate: Bearer resource_metadata="..."` header. A `200` means the route is not behind `requireMcpAuth`. The fourth must be `405`.

That third command deliberately sends nothing but the method — it is asking "is auth wired?", and the answer arrives before the request body is ever looked at. It is **not** a test of whether the endpoint speaks MCP. A real call under the current revision carries three headers — `MCP-Protocol-Version`, `Mcp-Method` naming the RPC, and `Mcp-Name` when the method names a thing (`tools/call`, `resources/read`, `prompts/get`) — and a `_meta` envelope in `params`. **Measured: the envelope needs the protocol version *and* the client's capabilities.** Send the version alone and the server answers `400` with `Invalid _meta envelope … clientCapabilities: missing`, which reads like a server fault and is not one.

```bash
curl -s -X POST http://localhost:3000/mcp \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H 'Mcp-Method: tools/list' \
  -H "MCP-Protocol-Version: $VERSION" \
  -d "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"tools/list\",\"params\":{\"_meta\":{
        \"io.modelcontextprotocol/protocolVersion\":\"$VERSION\",
        \"io.modelcontextprotocol/clientCapabilities\":{},
        \"io.modelcontextprotocol/clientInfo\":{\"name\":\"check\",\"version\":\"1.0.0\"}}}}" | jq
```

`$VERSION` is the revision the installed SDK implements and has to be identical in the header and in `_meta` — take it from the SDK rather than typing one in, because a guess fails as a mismatch rather than as a version error. A `400` from this command with a valid token means the token was accepted and the *request* is malformed; a `401` means the token was not.

### Testing the whole flow without restarting your own session

Adding an MCP server to the Claude Code session doing the build needs a restart, which loses the session. A stand-in client tests the same path, and it is the only way to get a real token into a `curl`. It took about ten minutes on a real build and found nothing wrong, which is a result:

1. A short Node script `POST`s to `/api/auth/oauth2/register` with `application_type: "native"`, `token_endpoint_auth_method: "none"`, a loopback redirect such as `http://127.0.0.1:8976/callback`, and the scopes it wants. It gets a `client_id` back. This is exactly what Claude Code does.
2. It makes a PKCE verifier and challenge, builds the `/api/auth/oauth2/authorize` URL — including `resource=<MCP_RESOURCE>` — prints it, and listens on that loopback port.
3. **The user opens the URL in the browser where they are signed in** and clicks Allow. They see the real consent screen, which is also the only way the builder gets that page looked at.
4. The listener receives `code`, `state` and `iss`, checks the state, exchanges the code at `/api/auth/oauth2/token` with the verifier and the same `resource`, and saves the tokens to a scratch file.
5. Call tools with the saved token using the `curl` shape above.

Decode the token's middle segment and check `iss`, `aud` and `scope` against `mcp-resource.ts` with your own eyes. Never print the token itself, and delete the scratch file afterwards. Leave the test connection in Connected apps for the user to revoke — that is the revoke test.

If sign-in from Claude Code fails at the redirect step, the cause is almost always redirect matching: it listens on a fresh ephemeral port each time and registers a loopback callback, and a client registering dynamically has to declare `application_type: "native"` for a `localhost` redirect to be accepted at all. Check that the client sent it and that the plugin's redirect matching is port-agnostic, rather than working around it by widening what the app accepts.

## Going to production

- `BETTER_AUTH_URL` becomes the real public URL on the host. That is the entire change; everything else derives from it. The plugin accepts an `http` resource only on a loopback host, so a production value has to be `https`.
- Give the user the connector URL — their domain plus `/mcp`, so `https://traillog.com/mcp` — and show them where it goes in Claude. They will not find it on their own.
- The discovery routes forward to `auth.handler` using the request as it arrived. Behind a proxy that rewrites the origin, check the documents still name the public URL. On Vercel the forwarded headers are already right.
- Dynamic registration creates a client row per fresh connection. Fine for one person; if the app gets popular, mention that old unused clients are worth pruning. Clients that identify themselves with a metadata document instead don't create rows at all, so this shrinks as the ecosystem moves across.
- This area is still moving. Before upgrading `better-auth` and its plugin packages, read their release notes for option renames, and regenerate the schema afterwards — the plugin's tables have changed shape between releases. Take that upgrade deliberately, not by accident during an unrelated one.

## Harnesses that can't do OAuth

Some automation tools — n8n, a self-hosted script, a home-grown harness — only send a static header. The temptation is to add a personal access token, and it is worth saying plainly what that costs: a long-lived credential that grants everything the user can do, that lives in another system's settings screen, that never expires, and that is the one people paste into a chat message. Every agent that matters here — Claude, Claude Code, ChatGPT, Cursor — does OAuth properly.

Don't build it unless the user asks for it specifically and understands that. If they do, four rules keep it survivable: issue it from `/settings` so it is visible where access is managed; verify it in the same place that verifies OAuth tokens so the tool layer never learns there are two kinds of caller; give it the same two scopes, not a bypass; and list it in Connected apps with a revoke button beside the others.

## Verify

Run these; do not reason about them.

- **First, because it has never been observed passing:** with a connection made and a tool call working, revoke it in Connected apps and make the same call again with the same unexpired token. It must answer `401` — without restarting the server, and without waiting for the token to run out. If it answers `200`, the Revoke button is lying and nothing else on this list matters yet.
- The same for a deleted account: a token issued to it is refused on the next call.
- The endpoint answers at `/mcp` — `src/app/mcp/route.ts` — and there is no `src/app/api/mcp/` directory left behind.
- An unauthenticated `POST /mcp` returns `401` with a `WWW-Authenticate` header containing `resource_metadata`, not a `200`.
- `GET /mcp` and `DELETE /mcp` return `405` — the route exports `POST` and nothing else.
- The authorisation server document advertises the registration mechanisms actually wired: `registration_endpoint` for dynamic registration, and `client_id_metadata_document_supported` from `cimd()`.
- The redirect back from the consent screen carries an `iss` parameter matching the issuer in the authorisation server document.
- The well-known documents return JSON at both the bare and the path-suffixed locations, and two pairs match character for character: the protected-resource document's `resource` against `MCP_RESOURCE`, and its `authorization_servers[0]` against the `issuer` in the authorisation-server document. No trailing slash on either.
- A decoded access token's `iss` is `ISSUER`, its `aud` includes `MCP_RESOURCE`, and its `scope` is what was approved.
- `offline_access` appears in the authorisation server document and does not appear in the protected-resource document.
- Both app scopes are discoverable while the route still gates on `hikes:read` alone. Any scope meant to stay private is absent from **all three** surfaces: the 401 challenge, the protected-resource document and the authorisation-server document.
- `clientRegistrationAllowedScopes` in `src/lib/auth.ts` is a written-out list, not a spread of `MCP_SCOPES`.
- Adding the server in Claude Code opens the app's own sign-in page, then a consent screen wearing the app's design, and the tools appear afterwards. Signing in from that page lands on consent, not on the dashboard.
- A read tool returns only the signed-in user's rows, and asking for more than the schema's maximum is rejected rather than clamped silently.
- **Second account test:** signed in as a different user, ask the agent for the first account's record by its id. It fails server-side — an empty result is not good enough, because that means the query ran.
- A write tool creates a real row that shows up in the app's own UI on refresh, and the activity log records it as the app's verb, done by an agent.
- A domain error — a taken slot, a record that doesn't exist — comes back as a tool result with `isError: true` and a sentence, not as a protocol failure.
- Approving read access only, then asking the agent to write, is refused by the tool — not merely absent from the consent screen — and the refusal is in the call log.
- After a revoke, the client row is gone when nobody else had approved that client, and still there when somebody had.
- No tool's result contains a token, a link that signs somebody in, or any other secret.
- `/settings/system` lists the calls that were just made, including the reads and the refusals, with the tool name, the arguments and the client.
- Every tool has a `title`, a `readOnlyHint` or `destructiveHint`, and a description that names a real reason to call it — no `create_row`, no catch-all with a `method` argument.
- With `BETTER_AUTH_URL` absent, or present and blank, the app still starts, and the system page reports the MCP endpoint as not configured rather than crashing.
