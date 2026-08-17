---
name: bring-your-own-agent
description: Add agent access to an app that already exists, so tools like Claude can do its real work — reading it and changing it the way the owner would. Use when someone wants Claude to work with an app they already have, rather than building a new one. Reads the app to find what it can do, agrees a list of capabilities with the owner, then generates a small MCP server that sits beside the app and talks to the app's own API. Adds one folder and edits nothing they wrote. Route discovery is written down and verified for Django, Express and FastAPI, and covered for the Next.js App Router on the evidence of one real app; other stacks are read the same way, and where there is no verified playbook this skill says so rather than guessing.
---

# Bring your own agent

Give an app somebody already has an assistant that can do its real work — look things up in it, add things to it, answer questions about what is in it.

The result is **one new folder** inside their project, called `agent-access/`. It holds a small program that speaks **MCP** — the standard way an assistant like Claude connects to something and is handed a short list of things it is allowed to do. That program reaches the app over the web, through the doorway the app already offers other programs — its **API** — so the app's own checks and rules apply to everything the assistant does, exactly as they would to a person clicking around in it. **Not one file the owner wrote is changed**, and deleting the folder puts the app back exactly as it was. Say that to them in those words; it is the difference between this and a change to their codebase.

The trade that buys it is real and has to be said out loud: **the assistant acts as one fixed login.** Anything that login can see or change, the assistant can see or change too. There is no permission screen and no way to give one person less access than another. Some apps can do far better than that, which is why **Step 0 comes before anything is built** — as soon as you know what the app is made of, and before a single word is said about what the assistant might be allowed to do.

## Ground rules

- Explain every choice the way you would to a smart friend who doesn't code. Say "a place to store your data" before "database". Introduce each technical term once, briefly, then use it normally.
- **Use their word for the thing.** If they call it a site, a system, or "the bookings thing", call it that too.
- **Never edit a file they already had.** Not a line of code, not a setting, not a single character. The single exception is *appending* to their `.gitignore`, and that is asked for first — and even that should almost never be needed, because the generated folder carries its own. This is the whole promise of the skill: an owner who agreed to "a folder I can delete" and finds edits inside their own app has been given something they did not agree to and now have to maintain.
- **Writes go through the app's own API. Always** — never straight into the database underneath it, no exception and no flag to enable one. Writing straight to the database skips the app's own checks, permissions, hooks and record of who did what, and leaves the app in a state its own code believes impossible. That does not fail at the time — it surfaces weeks later, somewhere unrelated, as a bug nobody can reproduce.
- **Reads prefer the API too.** Reading the database directly is a fallback for data the app has no web address for, and only through a login the database itself refuses to let write — a restriction the database enforces, not a promise in the code that a later edit quietly breaks. **Where such a login cannot be created, say so, and the default becomes reading through the API.** That is not a rare corner: `credentials.md` covers the common case — SQLite, which has no logins at all — where nothing in the database can enforce it. There the default is API-only reads; a direct read is taken only where the owner has been told, in the same breath as it is offered, that **the file's own permissions rather than the database are doing the work**, and that they are only as strong as the account the app runs under. What is never allowed is letting the database be implied to be protecting something it is not.
- **No delete tool. Not "unless they ask".** Throwing work away is a decision for a person, in the app, where they can see what they are throwing away. It is also the only mistake that leaves nothing behind to inspect afterwards; every other wrong call leaves something you can go and look at.
- **Anything that lists things has a hard maximum.** Without one, a single question hands over the entire account, or the answer is cut off part-way through and the assistant is handed something it cannot read at all.
- **Never present a partial discovery as a complete one.** Say what was searched, what was found, and what could not be determined. "I found three things it can do" when the app really has forty is worse than finding nothing, because nobody goes looking for the other thirty-seven — and the owner chose this path over a better one on the strength of a list they believed was the list.
- **Never report a tool as working when it has never actually been run.** "This should work" is not a hand-off. See *What "verified" means here* at the end.
- **Recommend once, then respect the answer.** If they choose the simpler thing after hearing the better one, build it well and do not reopen it.
- All commands, package names and config live in the reference files, never in this one. Load only the reference for the branch you are actually on.
- **What the app is built with — its *stack* — decides how much of this is proven.** Finding what an app can be asked to do is written down and verified for **Django, Express and FastAPI**, and written down for the **Next.js App Router** on the evidence of one real application rather than a fixture — so that section is honest about being a first draft, and the owner is told so. Other stacks (Rails, Laravel, Go and the rest) are read the same way in principle, but nothing here has been checked against a running one, so say plainly that the stack is not covered rather than improvising a list. `references/discovery.md` has the reasoning: saying so costs the owner five minutes, and a confident wrong answer costs them the afternoon and the trust.

## Step 0 — Could your app do this properly?

**Some apps can, and those owners must be told so — by name, early, and in plain words — even though it means this skill steps aside.**

It is numbered 0 because it comes before anything is built and before you discuss what the assistant might be allowed to do. In practice you need to know what the app is made of first, so it fires the moment you recognise the stack in Step 2 — and never later than that.

Everything above is a trade: no per-person permission, no approval screen, one fixed login, in exchange for touching nothing. That is a good trade when the app cannot support the alternative. It is a bad one when it can — and the owner is the only person who can weigh it, which they can only do if they are told.

**Raise it when any of these is true:**

- **Next.js with proper sign-in already in it (Better Auth).** The strongest case, and the one with a proven path: `start-an-app/references/mcp.md` builds agent access *inside* an app like that, and the result is better in every dimension that matters — the assistant's tools call the same code the app's own buttons call, each person approves their own access, each can take it away again, and the assistant can only ever do what that person could do.

  **There is also a measured reason, and it is worth saying out loud, because it is about how much they get rather than how tidy it is.** On the one real Next.js app this skill has been run against — an accounting app with genuine sign-in and a real API — **two capabilities out of about twenty-five were reachable from outside**. Not because the app was badly built, but because in a modern Next.js app most of the work lives in Server Actions, and those can be *listed* but not *called*: their names are build-time hashes over a protocol nobody documents. The routes that remain are a thin edge of the app. So on this stack the trade is not "slightly less tidy" — it is most of the app being out of reach, and only the in-app path gets it back. If the number matters to them, `references/discovery.md` shows how to count it for *their* app before anyone commits to anything.
- **Any app with real user accounts and a permission model.** In-app is better here in principle, and **this skill has no proven playbook for it**. Say exactly that: *the better version exists, and I cannot build it for you here* — rather than quietly recommending the one thing on offer because it is the one thing on offer.
- **Sensitive data — money, health, other people's records, several customers' data in one system.** "Anything that login can reach, the assistant can reach" is a far heavier sentence for accounting software than for a to-do list. Raise it as a question about *their data*, not as a technical footnote: what is actually in here, and who would mind if it were read?

**How to put it**, where the good way is genuinely buildable:

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

Where the good way exists but you have no proven way to build it — the second trigger — say that instead of the first line. It would be new work, in their language, invented as you went. That is a real thing to weigh, and hiding it to make the offer look tidier is the failure this step exists to prevent.

**Three rules, and they matter more than the wording:**

1. **Recommend once.** Then respect the answer. If they choose this skill anyway — for speed, to try the idea, because they would rather their app was not touched — build it well and do not relitigate.
2. **Record the choice.** The generated `README.md` says the better path was offered and declined, and on what date. Without that line, whoever reads the folder in six months finds a pile of workarounds and assumes nobody knew better.
3. **Never decide for them.** Do not silently refuse, and do not quietly hand off to another skill. A skill that decides an app is "too important" for it and stops has taken a judgement that belongs to the person who owns the app.

## Step 1 — Which app?

A folder, or a repository to clone. **Confirm it is really there, and really the thing they mean, before touching anything** — say the full path back to them, and what you can see in it, in one line. They cannot confirm a location they were never shown.

Real projects are messier than the tutorial: several apps in one repository, folders inside folders, generated code, other people's code vendored in. **When it is not obvious where the app actually lives, ask** — do not pick the first thing you recognise and build on it. Getting this wrong is not a small error; everything after it — what the app can do, how it signs in, every tool built — belongs to the wrong app.

You come out of this step with one agreed folder, and the next step is what is inside it.

## Step 2 — What is in it?

Work out what the app can be asked to do. `references/discovery.md` is the method, per stack — including how to make the app list its own web addresses (its **routes**) rather than guessing at them, and how to tell an app with an API from one that only draws web pages for people to look at.

**The first thing you learn here is what the app is built with, and that is Step 0's trigger.** Go and have that conversation before the rest of this step is put to them.

**If it is not one of the four stacks named in the ground rules above, say so plainly — and then give them a choice rather than leaving the interview standing there.** Either you read the routes out of their code by hand and they confirm every one of them before a single tool is built on it, in which case the list is announced as unverified and stays unverified all the way to the hand-off; or they wait until the stack has a written and tested section here. What is never on offer is a route list produced by improvising a method and then presented as a discovery pass.

Four things about this step decide whether the rest is honest:

- **Ask them what they *do* in the app before you look at the code.** In their own words — "raise an invoice", "chase a late payer", "reconcile the bank" — written down before a single route has been enumerated. Then map each item to something discovery found: **the unmapped ones are the finding**, and they are what turns one number into two — *"your app does about twenty-eight things; three of them have a web address a program can call, and two are worth a tool"*. Asked afterwards, the answer is worthless: the routes suggest the capabilities, and the missing ones are invisible because nothing on the list is where they would have been.
- **Ask the app itself, do not guess.** Most apps generate routes that appear in no file anyone wrote — one line of somebody's code quietly becomes six web addresses, or twenty-three. Searching the code for them finds none of those and reports an app with nothing to offer.
- **The found list is read by eye before anyone sees it.** The searches in `discovery.md` deliberately cast wide and catch things that are not routes at all, because a search narrow enough to look tidy misses whole parts of the app — and an extra line you discard in two seconds costs nothing, while a missing one is something the owner never learns their app could do. So somebody reads the raw result and prunes it. That is a person's job, not an automatic pass.
- **Then show it and let them correct it.** In plain words — what the app appears to be able to do, what was searched for, and anything that could not be determined. **The owner edits a list rather than writing one.** They will correct what is wrong, and be reminded of things they would never have thought to ask for.

**If the app has no API at all** — it draws web pages and nothing else — then **reads work and writes are impossible**, and that must be said here, before any list of what the assistant may do is offered. Discovered at Step 5 it costs them an hour and a decision they would have made differently. `discovery.md` has the three honest options and the reason why pretending to be a browser is not one of them.

## Step 3 — What may Claude do?

Take the list from Step 2 and agree what the assistant is actually allowed to do with it.

**Reads and writes are agreed separately, in that order, and the second is not implied by the first.** Agreeing that Claude may look at the orders is not agreeing that it may create one. Ask twice, plainly, and write down which answer was which — this list becomes the tools, and the tools are what the owner will be shown at the end.

Say out loud that **there will be no way to delete anything**, and why. It is better heard now than noticed later as something missing.

Everything agreed here becomes a tool named after the app's own words — list the orders, look one up, add one. Nothing agreed here becomes a general-purpose "do anything to the app" tool; `references/server.md` explains why that is a template in disguise and leaves the owner nothing to review.

## Step 4 — How does it get in?

The assistant needs a way to sign in to the app. `references/credentials.md` obtains one, sends it, and proves it works before anything is built on top of it — per stack, including the trap that costs the most hours: most existing apps expect a browser's own sign-in plus a second hidden token, and a request missing that second one fails in a way that looks like a bug in the new code rather than a missing credential.

Two things are yours, not the reference's:

- **Create a login for the assistant, rather than borrowing a person's**, wherever the app allows it. Switching it off later then does not lock a real person out of their own work.
- **Say the one-identity disclosure at the moment the credential is chosen** — not once the thing is built. `credentials.md` gives the words. It is not a warning about something that might happen; it describes how this works on the first day and every day after.

If any read has to come from the database directly, the second disclosure belongs here too, in the same breath as offering it: the assistant can then see records the app would normally hide, because rules the app applies in its own code — *only show a customer their own orders* — are invisible to the database underneath it.

Where a check is supposed to prove something is *refused*, name which refusal counts. `credentials.md` is precise about this for a reason: a test whose failure looks identical to its success proves nothing, and the one case where that matters is exactly the case the test exists for.

## Step 5 — Build, connect, verify

`references/server.md` is the specification for what gets generated: the folder, the tools, the limits on them, the record it keeps of every call, and where the credential lives. It is a specification with worked examples, not a file to copy — two runs of this skill should produce servers that differ only where the apps differ.

The server runs **on the owner's own machine, started for them by whatever program they talk to Claude in**, with nothing listening on a network and no secret exposed over one. Say what that means before they find out: it works from Claude on that computer. It does not work from their phone, from the website, or for anybody else. That is how most of these work today, and it is worth saying rather than discovering.

Then connect it to the program they will actually use and make real calls through it. **Do not close this step on a clean build.** Verification has its own section below because it is the part that gets skipped, and it is the only part that distinguishes a hand-off from a hand-wave.

## Step 6 — Deploy?

Offered at hand-off, explained, and never assumed — the same shape `start-an-app` uses for putting an app online. It is a deliberate second step the owner chooses, not something done mid-build.

There is one case for it: the assistant runs somewhere that cannot start a program next to their app. If that is not their situation, the answer is no and the conversation is one line long.

If they do want it, **say the cost before anything is built, not afterwards**: once the server can be reached over a network, whoever can reach it has everything that one login can do. A single secret string is the entire defence — there is nothing behind it, no second check, no approval screen. `references/deploy.md` covers it, and it does not treat a server that answered once as a server that works.

## What "verified" means here

This skill did not build the app and cannot know what the right answer to "list the orders" looks like. So verification is a collaboration: **the skill does the work, the owner confirms, and nothing is taken on trust.**

**Reads — one real call per tool, shown raw.** Paste back what actually came out, not a summary of it, and ask: *"is this your data, and does it look complete?"* That one question catches the two failures the skill cannot see for itself — being pointed at the wrong copy of the app, and an answer that is a slice of the data while claiming to be all of it.

**Writes — one real write, confirmed twice.** First read it back through a *different* tool. Then have the owner find it in the app's own interface. **The second is not redundant.** It is the only check that the write went through the app's own rules rather than around them, and it is what catches a call that looked successful and produced something the app itself does not consider valid.

**Undo what you wrote.** A verification write is a real record in a real system. Say what it will create *before* creating it, make it obviously disposable, and remove it afterwards **through the app** — so the removal is as legitimate as the write was.

**Where a real call is impossible** — the app will not run on this machine, there are no credentials, the database is empty — report those tools **unverified** and name what was missing. Never "this should work". A tool that has never been run is not a working tool, and saying so plainly is worth more than a confident hand-off that falls over on the owner's first question.
