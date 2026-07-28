---
name: audit-my-app
description: >-
  Audit a web app / codebase across three lenses — (1) Security vulnerabilities,
  (2) Code quality (performance, bugs, tech debt), and (3) UI/UX issues (missing
  pages, dead links, accessibility) — then write a dated Markdown report to
  audit/YYYY-MM-DD.md and offer to fix the high-confidence findings. Use this
  whenever the user asks to "audit my app", run a "security audit", "review my
  app/code", "check for vulnerabilities", "find bugs / tech debt", "review code
  quality", or "check for broken/dead links or missing pages" — even if they
  only name one of the three areas. Prefer this skill for any whole-app review,
  health-check, or "is my app any good / safe?" request.
---

# Audit My App

Produce an honest, evidence-backed audit of an application and save it as a
dated report. The value of an audit is **trust**: every finding must point at
real code the reader can open, and the report must not pad itself with generic
advice. A short report full of true, located findings beats a long one full of
plausible guesses.

The user has opted into **report-first, then offer fixes**: investigate, write
the report, summarize it, and *then* offer to apply the safe fixes. Do not
change code during the audit itself.

## The three lenses

1. **Security** — could someone abuse this? Injection/XSS, secrets in source,
   auth/session, unsafe storage, dependencies, headers/CSP, etc.
2. **Code quality** — performance, correctness bugs, and tech debt /
   maintainability.
3. **UI / UX** — missing pages, dead links, broken asset references,
   accessibility, responsiveness, broken/empty states.

The detailed checks for each lens live in
[references/audit-checklist.md](references/audit-checklist.md) — read it before
the corresponding pass so you don't miss whole categories. It's organized by
lens with grep patterns and example findings.

## Workflow

### 0 — Orient (do this first)

- **Get today's real date** as `YYYY-MM-DD`; don't guess it. If the environment
  already states the current date, use that. Otherwise run a command:
  `Get-Date -Format 'yyyy-MM-dd'` (PowerShell) or `date +%F` (bash). This names
  the report `audit/<date>.md`.
- **Map the app.** Glob the tree; read `package.json` / config / the entry HTML
  or server file. Identify the **stack** (static site, SPA + framework, backend
  API, full-stack), the **entry points**, and the **routes/pages**. The audit's
  depth and the checks that apply both depend on this.
- **Set scope.** Default to the whole repo, but skip generated/vendored
  directories (`node_modules`, `dist`, `build`, `.git`, `vendor`, `coverage`)
  unless asked. Honor any path the user named.
- Create the `audit/` directory at the project root if it doesn't exist.

### 1 — Security pass · 2 — Code-quality pass · 3 — UI/UX pass

For each lens: open the checklist section, then **investigate with tools** —
`Grep` for risky patterns, `Glob` to enumerate files/links, `Read` to confirm.

The discipline that makes this skill worth running:

- **Verify before you report.** A grep hit is a *lead*, not a finding. Read the
  surrounding code and confirm the issue is real and reachable before writing it
  down. `innerHTML = "<b>Saved</b>"` with a constant string is not an XSS; the
  same line fed user input is. Say which.
- **Locate everything.** Every finding cites `path/to/file.ext:line`. If you
  can't point to a line, it isn't a finding yet — it's a question.
- **Don't invent coverage.** If a lens/category is clean, say so explicitly and
  list what you checked. An empty "Security" section reads as "didn't look";
  "No injection sinks found — checked innerHTML, eval, document.write, template
  rendering" reads as "looked, found nothing."
- **Separate confirmed from potential.** Mark uncertain items as *Potential* and
  say what you'd need to confirm them (e.g., runtime behavior, a value you can't
  see statically).

#### Dead links & missing pages (part of lens 3, called out because it's easy to do half-way)

Be systematic, not anecdotal:

1. Enumerate every internal reference — `href`, `src`, `action`, `<link>`,
   route definitions, framework `<Link to>` / router paths, and relative imports.
2. For each, resolve the target and check it exists (a file in the repo, or a
   declared route). Anything that resolves to nothing is a **dead link**; a nav
   item or route that points to a page that was never built is a **missing page**.
3. Report them as a table so the reader can scan and fix quickly.

External URLs can't be verified statically — list them under "couldn't verify"
rather than asserting they're broken, unless you have a way to check them.

### 4 — Write the report

Write `audit/<YYYY-MM-DD>.md` using the template below. If a report already
exists for today, append a clearly-marked new run section rather than silently
overwriting earlier findings.

### 5 — Summarize & offer fixes

In chat: give the headline (overall risk + the counts), name the top 3–5 things
to fix, and link the report path. Then offer to apply the **high-confidence,
low-risk** fixes (typos, dead links, missing `rel="noopener"`, obvious bugs,
clear-cut hardening). Group them so the user can pick. Leave anything that
changes behavior, needs a product decision, or is uncertain for them to direct.
Don't touch code until they choose.

## Report template

Use this structure exactly (drop a subsection only if truly N/A, and say why):

```markdown
# App Audit — <YYYY-MM-DD>

- **Project:** <name>
- **Scope:** <paths audited> (excluded: <generated/vendored dirs>)
- **Stack:** <detected stack>
- **Generated by:** audit-my-app skill

## Summary
- **Overall risk:** <Critical | High | Medium | Low>
- **Findings:** <N> total — 🔴 <c> Critical · 🟠 <h> High · 🟡 <m> Medium · 🔵 <l> Low · ⚪ <i> Info
- <One paragraph: the app's overall health and the single most urgent fix.>

## 🔒 Security
### <SEV> <Short title>
- **Where:** `path/to/file.ext:line`
- **Issue:** <what it is, concretely>
- **Why it matters:** <impact / how it could be abused>
- **Fix:** <specific, actionable recommendation>

<Repeat per finding. If clean: "No issues found. Checked: <list of categories>.">

## 🧱 Code Quality
### Performance
### Bugs / Correctness
### Tech debt & maintainability
<Same finding format under each. State "none found — checked X, Y, Z" where clean.>

## 🎨 UI / UX
### Missing pages & dead links
| Link / target | Found in | Status |
|---|---|---|
| `./reports.html` | `index.html:88` | ❌ missing file |
| `#settings` route | `app.js:51` | ❌ no matching route |

### Accessibility
### Responsiveness & visual
### Broken / empty states
<Findings as above.>

## Prioritized fix list
1. **[Critical]** <finding> — `file:line`
2. **[High]** <finding> — `file:line`
...

## Notes & limitations
- <What wasn't checked: runtime/dynamic behavior, third-party services, external URLs, etc.>
- <Assumptions made.>
```

## Severity guide

- **🔴 Critical** — exploitable now / data loss / app broken for users (e.g.
  secret committed, auth bypass, stored XSS on a real input).
- **🟠 High** — serious but bounded, or needs a precondition (e.g. reflected XSS,
  injection behind auth, a crash on a common path).
- **🟡 Medium** — real issue, limited impact or harder to hit (missing CSP,
  noticeable perf regression, a dead link in primary nav).
- **🔵 Low** — minor / best-practice (missing `rel="noopener"`, small tech debt,
  a dead link in a footer).
- **⚪ Info** — observation worth noting, not a defect.

## Notes

- Tailor checks to the stack — don't report SQL injection on a static site, or
  "no CSP" as Critical for a localhost-only tool; use judgment about real risk.
- This is a static review. Where a finding truly depends on running the app
  (e.g. console errors, actual responsive breakpoints), say so and, if the tools
  are available and the user is willing, offer to verify it live rather than
  asserting it.
