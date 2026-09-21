# AI features (AI SDK + OpenRouter)

Last verified: 2026-09-21

**Purpose:** Give the app AI abilities (chat, text generation) through the AI SDK, with OpenRouter as the provider so one API key reaches many models.

## Install

```bash
pnpm add ai @ai-sdk/react @openrouter/ai-sdk-provider
```

Tell the user: create a free key at https://openrouter.ai/keys, then append to `.env`:

```
OPENROUTER_API_KEY=<their key>
OPENROUTER_MODEL=<model id from the check-what's-current step>
```

No model id is written in this file on purpose: they change too often, and an id that has been retired fails at the first message. The check-what's-current step picks a current one from https://openrouter.ai/models that suits the feature and the user's budget, and that exact id goes here.

Keeping the model in `.env` means they can swap models later without touching code.

## Configure

Provider helper `src/lib/ai.ts`:

```ts
import { createOpenRouter } from "@openrouter/ai-sdk-provider";

const apiKey = process.env.OPENROUTER_API_KEY;
const modelId = process.env.OPENROUTER_MODEL;

export const aiConfigured = Boolean(apiKey && modelId);

// null when the key or the model is missing, so importing this file can never throw.
export const model = apiKey && modelId ? createOpenRouter({ apiKey })(modelId) : null;
```

No `!` on either value and no client built unless both are present: a blank key must leave the rest of the app running.

Chat endpoint `src/app/api/chat/route.ts`:

```ts
import { streamText, convertToModelMessages, type UIMessage } from "ai";
import { model } from "@/lib/ai";

export async function POST(req: Request) {
  if (!model) {
    return Response.json(
      { error: "AI is not set up yet. Add OPENROUTER_API_KEY and OPENROUTER_MODEL." },
      { status: 503 },
    );
  }
  const { messages }: { messages: UIMessage[] } = await req.json();
  const result = streamText({ model, messages: convertToModelMessages(messages) });
  return result.toUIMessageStreamResponse();
}
```

This route spends the user's OpenRouter credit on every call, so it must not be open to strangers. If the app has accounts, the first line of `POST` is `await requireUserAction()` from `src/lib/auth-guards.ts` (written by `references/pages.md`). That file does not exist yet when this step runs, so do not leave the route open in the meantime: check the session inline now, the way `references/storage.md` does, with `const session = await auth.api.getSession({ headers: req.headers }); if (!session) return new Response("Not signed in", { status: 401 });`, and swap it for the guard when pages has run. That guard throws, and an uncaught throw in a route handler is a 500, so catch it and answer 401 — a signed-out request is refused, not crashed. If the app has no accounts and is deployed, put a simple rate limit on the route. The page that uses the feature reads `aiConfigured` on the server and shows the notice instead of the chat box when it is false.

On the client, build the chat UI with `useChat` from `@ai-sdk/react` pointed at `/api/chat`. Shape the feature to the interview: a support-style chat, a "generate description" button (use `generateText` server-side for one-shot generations), or whatever the user actually asked for — a generic chatbot page is the fallback, not the goal.

Set the system prompt from the interview so the AI knows what app it lives in.

## Verify

Do not create an account to run these. The first account belongs to the real person and is made in Step 6.

Checked now:

- `pnpm exec tsc --noEmit` passes, and `grep -rn "process\.env\.[A-Z_]*!" src` finds nothing.
- With the key or the model blank, `pnpm build` passes, the app still runs, and a POST to `/api/chat` answers 503 with the "not set up yet" message rather than 500.
- If the app has accounts and the guard is in place: a signed-out POST to `/api/chat` answers 401.
- If the app has no accounts: with a real key and model in `.env`, one POST to `/api/chat` returns a streamed reply.

Deferred, because they need the finished app or an account:

- Deferred to Step 5, once the pages exist: the AI feature is reachable from the app's navigation, not orphaned, and without a key it shows a plain "add your OpenRouter key" notice instead of crashing.
- Deferred to Step 6, after the user has signed up: with a real key and model in `.env`, sending one message from the app returns a streamed reply.
