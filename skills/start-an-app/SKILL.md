---
name: start-an-app
description: Interview the user and scaffold a working full-stack web app tailored to what they actually want to build. Use when the user wants to start a new app, website, prototype, or SaaS; when they don't know what tech stack to pick; or when they want a solid working starting point fast. Covers project setup, database, sign-in, and optional AI features.
---

# Start an App

Turn an idea into a running web app through a short, friendly interview. The result is the user's actual app from the first commit — their name, their pages, only the infrastructure they need. It should never feel like a template.

## Ground rules

- Explain every choice like you would to a smart friend who doesn't code. Say "a place to store your data" before saying "database". Introduce each technical term once, briefly, then use it normally.
- Ask one question at a time. Always offer a recommended default so the user can just say "whatever you recommend".
- Recommend, then respect. If the user picks the non-recommended option, go with it without relitigating.
- The stack is fixed: Next.js, TypeScript, Tailwind, shadcn/ui, Drizzle, Better Auth. The interview chooses *within* it (which database, what kind of sign-in, AI or not) — it never swaps out these pieces.
- All commands, package names, and config live in the reference files, never in this file. Load only the references for the branches the user chose.
- If a reference command fails because a tool changed (renamed flag, different init flow), check that tool's official docs, use the current equivalent, finish the job, and tell the user at the end that this skill's reference file needs a refresh.

## Step 1 — Interview

Ask these in order, one at a time:

1. **"What are you building? Describe it like you'd describe it to a friend."**
   → Drives the app's name, its pages, and its data model. Listen for the nouns (the things the app manages) and the verbs (what users do with them).

2. **"Who's going to use it — just you for now, or other people / the public?"**
   → Just me / trying an idea: recommend **SQLite** ("your data lives in a simple file inside the project — nothing extra to install or run").
   → Other people / production ambitions: recommend **Postgres** ("a proper database server — a bit more setup now, no migration pain later").

3. **"Do people need to sign in?"**
   → No accounts: skip auth entirely.
   → Yes: recommend **email + password** as the default ("works immediately, nothing to configure").
   → If they want "Sign in with Google": say yes, and set expectations — it needs a free Google Cloud setup with a few copy-paste steps; offer to walk through it together or add it later.

4. **"Should the app have any AI features — like a chat, or generating text or content?"**
   → Only include AI plumbing if yes. If yes, mention they'll need an OpenRouter API key (free to create) and you'll show them where to get it.

5. **"When someone opens the app, what's the first thing they should see or do?"**
   → Drives the home page and navigation. This is what makes the result feel like their app.

## Step 2 — Build sheet

Restate the plan in plain words before touching anything. Example shape:

> Here's what I'll set up: **"TrailLog"** — a hiking journal. Your data is stored in a simple file (SQLite) since it's just for you. You'll sign in with email + password. No AI features for now. The home page shows your most recent hikes with a button to log a new one.

Get a clear go-ahead. Adjust anything they push back on.

## Step 3 — Scaffold

Work through these in order. Each reference has a **Verify** section — complete it before moving on.

1. Base project → `references/stack.md`
2. Database (SQLite or Postgres branch) → `references/database.md`
3. Sign-in, if chosen (email+password, optionally Google) → `references/auth.md`
4. AI features, if chosen → `references/ai.md`

## Step 4 — Make it theirs

- Name the project after their idea (package name, page titles, visible branding).
- Build the actual pages from interview answers 1 and 5: real navigation, a real home page, a schema whose tables are their nouns.
- Seed nothing generic: every visible string should make sense for *their* app.
- Done when: someone opening the app would know what it is without being told.

## Step 5 — Verify and hand off

- The dev server starts cleanly and the home page renders.
- Database migrations have run; creating one real record works end to end.
- If sign-in was chosen: signing up and signing back in works.
- Close with a plain-language summary: how to start the app, what each entry in `.env` is for, and two or three sensible next steps (first feature to add, how to deploy when ready).
