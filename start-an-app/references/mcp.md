# Agent access (MCP over OAuth)

Last verified: 2026-08-16

**Purpose:** Let Claude — or any assistant that speaks MCP — do the app's real work as the signed-in user, behind the same sign-in and the same permission checks as the buttons. No API keys to mint, paste or rotate: the assistant goes through the app's own front door and the person approves it on a consent screen.

> Compatibility note: this area churns hard. Better Auth's own `mcp()` plugin is **deprecated** — use `@better-auth/oauth-provider`. Check the installed source rather than trusting any example, including this one, and treat every version number here as "what was true when last verified".

The whole design rests on one rule: **the tools call the same functions the buttons call.** No second query path, no second permission check. That is the only reason the tool layer cannot drift away from what the UI enforces.

## Install

```bash
pnpm add mcp-handler @modelcontextprotocol/server @better-auth/oauth-provider
```

## Configure

### One file for the strings that must agree

Four values have to match character for character — the plugin's audience, the discovery document's `resource`, the audience checked when a token is verified, and the URL somebody types into Claude. When they disagree the flow completes right up to the first tool call and then fails on an audience mismatch with no useful error.

```ts
// src/lib/mcp/resource.ts
const BASE_URL = (
  present(process.env.BETTER_AUTH_URL) ??
  present(process.env.APP_URL) ??
  "http://localhost:3000"
).replace(/\/$/, "");                    // no trailing slash, ever

export const ISSUER = `${BASE_URL}/api/auth`;   // Better Auth issues from its base path
export const MCP_RESOURCE = `${BASE_URL}/mcp`;
```

Use a helper that treats blank as missing (`present()` above). `??` only falls back on `null`/`undefined`, so a variable that exists but is empty sails through — and `new URL("")` throws. If that value reaches a root layout, every route in the app answers 500.

### Scopes: two, named for the app

```ts
export const MCP_SCOPES = ["board:read", "board:write"] as const;
```

Not one per table. A consent screen listing nine scopes is a consent screen nobody reads, which defeats having one. Name them after what a person would recognise, and describe them in plain words on the consent page ("Read your boards" / "Add and change cards"), never as raw scope strings.

### The endpoint is `/mcp`, not `/api/mcp`

Nothing in the App Router requires the `api` segment, and this URL is not an internal detail — it is a string somebody types into a connector dialog or reads out loud.

```ts
// src/app/mcp/route.ts
const handler = withMcpAuth(mcp, verifyToken, {
  required: true,                              // see below
  requiredScopes: [MCP_SCOPES[0]],
  resourceMetadataPath: "/.well-known/oauth-protected-resource/mcp",
});
export { handler as POST };                    // POST only — GET/DELETE must 405
```

**`required: true` is not optional despite the name.** It defaults to `false`, and with the default a request carrying no token *runs the tools anyway* — the app serves unauthenticated traffic, the client never sees a 401, and so it never starts the sign-in flow. It is the single most expensive line to leave out.

### Discovery

Serve the protected-resource document at **both** `/.well-known/oauth-protected-resource/mcp` (the path-suffixed form RFC 9728 defines, which a spec-following client looks for first) and the bare `/.well-known/oauth-protected-resource`.

## The traps

Every one of these cost real debugging time. They are the reason this file exists.

### 1. Advertised scopes are not required scopes

A client asks for exactly what discovery tells it about. `mcp-handler` reuses `requiredScopes` — which is the **gate** every token must clear — as the `scope` it advertises in the 401 challenge. Set it to `board:read` alone and no client ever learns `board:write` exists, so **read-only becomes the only grant anybody can obtain**.

Widening `requiredScopes` is the wrong fix: it gates as well as advertises, so a read-only token would be turned away at the door and "approve for reading only" would stop working. Gate on read, advertise both.

### 2. There are three advertising surfaces, and the third defeats the obvious fix

1. The **401 challenge** on `POST /mcp`.
2. The **protected-resource document** — world-readable, CORS-open, cached. A spec-following client reads this *before* it ever sees a 401.
3. The **authorization-server document**, which emits `scopes_supported: opts.advertisedMetadata?.scopes_supported ?? opts.scopes` — and a scope **must** be in `opts.scopes` or authorize rejects it as `invalid_scope`.

So a scope you want to keep out of discovery is published anyway unless you set `advertisedMetadata.scopes_supported` explicitly to the public list. The library validates it is a subset of `scopes`, so authorize keeps working for a client that asks deliberately. **Curl all three when you think you are done** — getting two of three right is the expected failure.

### 3. Never spread your scope list into `clientRegistrationAllowedScopes`

```ts
// WRONG — adding a scope to MCP_SCOPES silently widens this too
clientRegistrationAllowedScopes: ["offline_access", ...MCP_SCOPES],
```

With `allowUnauthenticatedClientRegistration: true`, that means **any stranger can register a client pinned to any scope you ever add** — including an admin one — and the type-checker will be perfectly happy. Write these lists out as literals with a comment saying why they are not derived.

### 4. `offline_access` must be advertised

It is not a scope of your resource at all — it is what a client asks for to be issued a refresh token, and Claude Code asks for it as a matter of course. Dynamic registration **pins a client to the list it discovered at registration time**, so leave it out and registration succeeds, the client then asks for a refresh token as usual, and the authorize call fails with `invalid_scope` *before any consent screen is drawn* — nothing on screen explains why.

### 5. Revoking access leaves the registration behind

Revoking clears the *consent*. The OAuth *client* row survives, still pinned to the scopes it was created with. Change your scope list afterwards and that client keeps asking with its old one and fails with `invalid_scope` for ever, with no way to recover from inside the app.

Make revoke drop an orphaned registration too — guarded on nobody else using it, since one client row can serve several people and revoking your own access must never reach into someone else's.

### 6. Consent is all-or-nothing unless you build it

The provider **does** support granting a subset: the consent endpoint takes an optional space-separated `scope`, validates it is a subset of the signed original, and the narrowed set flows into the token. But you have to render the checkboxes and post the narrowed list. Without that, "approve read-only and upgrade later" is advice nobody can follow.

Three traps if you build it: never post `scope: ""` (send a refusal instead), and always pass `openid` and `offline_access` through untouched or you silently kill the id_token and the refresh token an hour later.

### 7. A tool's return value goes to the model

Wrap every tool in a `logged()` helper that takes the userId **from the token**, checks the per-tool scope, records reads as well as writes, and writes to the app's ordinary activity log with `via: "mcp"`. It also stringifies the return value into the call log *and* hands it to the model — so **no tool may ever return a token, a signup link, or any other secret**. Destructure what you return; do not spread an object that happens to carry one.

### 8. Server-side guards that read `headers()` do not work here

Identity inside `/mcp` is a bearer token, not a session cookie, and a Next navigation throw (`redirect()`, `notFound()`) surfaces as a 500 rather than a refusal. Any admin check needs its own `isAdmin(userId)` that reads the row.

## The tools — task-shaped, named for the verbs

Name them for what somebody would ask for, not for tables:

| Tool | Scope | Hint |
|---|---|---|
| `list_my_boards` | read | readOnly |
| `get_board` | read | readOnly — full board, capped |
| `search_cards` | read | readOnly — across everything they can see |
| `get_card_comments` | read | readOnly — the thread, oldest first |
| `add_card`, `move_card`, `update_card` | write | `destructiveHint: false` |
| `assign_card`, `comment_on_card`, `add_tray` | write | `destructiveHint: false` |

**No delete tool.** Throwing work away is a decision for a person, in the app, where they can see what they are throwing away.

Every list tool takes a `limit` with a hard maximum enforced **in the schema and in the query**. An uncapped list either truncates mid-JSON into something unparseable, or hands one call the entire account.

Reads matter as much as writes: a tool that only reports a *count* of something (comments, files) leaves an assistant acting blind on it. If a count is worth returning, the thing itself is usually worth a tool.

## What the person can see and control

- **A consent screen wearing the app's own design.** A page that looks foreign reads as phishing. Name the client, warn that the name is self-chosen and checked by nobody, describe the scopes in plain words, show which account it will act as, and give Approve and Deny **equal** weight.
- **A connected-apps page** listing what has access, what it may do, when it was granted and when it was last used — with a revoke that kills the *next call*, not just a row.
- **The call log**, reads included. When a person reads their own board that is noise; when an assistant does, it is how the data left the app, so the number of rows it took is worth seeing.

## Verify

Run these; do not reason about them.

```bash
# 401 with a challenge that names the scopes and points at the metadata
curl -i -X POST $APP/mcp -H 'content-type: application/json' -d '{}'

# 405 — the old session shapes are gone
curl -i -X GET $APP/mcp
curl -i -X DELETE $APP/mcp

# both documents, and the authorization-server one
curl $APP/.well-known/oauth-protected-resource/mcp
curl $APP/.well-known/oauth-protected-resource
curl $APP/.well-known/oauth-authorization-server
```

- [ ] `POST /mcp` with no token → **401**. A 200 means `required: true` is missing.
- [ ] `resource` matches the audience and `authorization_servers[0]` matches the issuer, character for character, no trailing slash.
- [ ] Any scope you meant to keep private is absent from **all three** surfaces.
- [ ] `claude mcp add --transport http <name> $APP/mcp`, then `/mcp` → the app's own sign-in, the consent screen, tools appear.
- [ ] A read tool returns only what that person can actually see — checked with a **second account**, and failing server-side rather than returning an empty list.
- [ ] A token without the write scope is refused **by the tool**, and the refusal is still logged.
- [ ] A write tool's row appears in the UI on refresh, logged `via: "mcp"`.
- [ ] Revoking makes the very next call fail, without restarting the server.
