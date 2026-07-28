# Audit checklist

Detailed, per-lens checks. Each item is a *lead to investigate*, not an
automatic finding — read the code and confirm before reporting. Grep patterns
are starting points; adapt them to the stack. Tune severity to real-world risk
(a localhost-only tool ≠ a public app handling money).

## Contents
- [1. Security](#1-security)
- [2. Code quality](#2-code-quality)
- [3. UI / UX](#3-ui--ux)

---

## 1. Security

### Injection & XSS (the most common web sink)
- DOM XSS sinks fed untrusted data: `innerHTML`, `outerHTML`, `insertAdjacentHTML`,
  `document.write`, jQuery `.html()`, React `dangerouslySetInnerHTML`, Vue `v-html`,
  Angular `bypassSecurityTrust*`, template literals injected into the DOM.
  - Grep: `innerHTML|outerHTML|insertAdjacentHTML|document\.write|dangerouslySetInnerHTML|v-html|bypassSecurityTrust`
  - Confirm the value's **source**: a constant/escaped value is fine; user input,
    URL params, localStorage, or server data is the finding. Note `textContent`
    usage as the safe pattern.
- Dynamic code execution: `eval(`, `new Function(`, `setTimeout("…string…")`,
  `setInterval("…string…")`. Grep: `eval\(|new Function\(`
- Server-side injection (if there's a backend): SQL built by string concatenation
  (use parameterized queries), shell/command exec (`child_process.exec`,
  `os.system`, `subprocess` with `shell=True`), path traversal (`../` from user
  input into `fs`/`open`), SSRF (user-controlled URL passed to fetch/requests),
  insecure deserialization (`pickle`, `yaml.load`, `JSON`→eval).

### Secrets in source
- API keys, tokens, passwords, private keys committed to the repo or shipped to
  the client. Grep (case-insensitive): `api[_-]?key|secret|password|token|private[_-]?key|BEGIN .*PRIVATE KEY|AKIA[0-9A-Z]{16}|xox[baprs]-|ghp_|sk-`
- `.env` files committed; secrets in client-side JS (anything in the browser
  bundle is public); credentials in config checked into git.

### Auth, sessions, access control
- Missing/weak auth on sensitive routes; client-side-only access checks (trivially
  bypassed); JWT with `alg: none` or hardcoded signing secret; tokens in
  `localStorage` (XSS-exfiltratable) vs httpOnly cookies; missing CSRF protection
  on state-changing requests; no rate limiting on auth endpoints; predictable IDs
  enabling IDOR.

### Data handling & storage
- Sensitive data (PII, tokens) in `localStorage`/`sessionStorage`/cookies without
  `httpOnly`/`Secure`/`SameSite`; logging of secrets/PII; no input validation on
  user-supplied data before use/storage.

### Transport, headers, config
- Missing security headers / CSP; `http://` resources on an `https://` page
  (mixed content); over-permissive CORS (`Access-Control-Allow-Origin: *` with
  credentials); `target="_blank"` without `rel="noopener noreferrer"` (reverse
  tabnabbing) — Grep: `target=["']_blank["']`; clickjacking (no frame-ancestors /
  X-Frame-Options).

### Dependencies
- Outdated/known-vulnerable packages. If `package.json`/lockfile exists, note
  obviously old major versions and suggest `npm audit`. Flag unpinned/`*` versions
  and dependencies loaded from untrusted CDNs without SRI (`integrity=`).

---

## 2. Code quality

### Performance
- Work on the hot path / main thread: layout thrash (read-then-write in loops),
  unthrottled `scroll`/`resize`/`mousemove`/`input` handlers (should debounce/
  throttle), `O(n²)` over large collections, re-rendering/rebuilding the whole DOM
  on every change instead of patching.
- Network/asset weight: large unoptimized images, no lazy-loading, render-blocking
  scripts in `<head>` without `defer`/`async`, no caching headers, huge bundles,
  shipping dev builds.
- Repeated expensive work that could be memoized/cached; N+1 queries on a backend;
  missing DB indexes on filtered/sorted columns.

### Bugs & correctness
- Unhandled promise rejections / missing `try-catch` around async or JSON.parse;
  swallowed errors (`catch {}`); off-by-one and boundary errors; `==` vs `===`
  surprises; timezone/`new Date(string)` parsing bugs; race conditions; mutation
  of shared state; event listeners added on every render without cleanup (leaks);
  using a value before it's defined; incorrect null/undefined handling.
- Grep helpers: `catch\s*\(\s*\)|catch\s*{|JSON\.parse\(|== null|innerHTML\s*\+=`

### Tech debt & maintainability
- Markers: `TODO`, `FIXME`, `HACK`, `XXX`, `@deprecated`. Grep: `TODO|FIXME|HACK|XXX`
- Dead code (unused exports/vars/files), commented-out blocks, duplicated logic
  that should be shared, copy-pasted constants/magic numbers/magic strings,
  inconsistent naming, very large files/functions doing too much, missing module
  boundaries. Note files that have grown unusually large for their role.
- Absent tests where logic is non-trivial; no input validation at boundaries.

---

## 3. UI / UX

### Missing pages & dead links (be exhaustive — see SKILL.md)
- Enumerate internal targets: HTML `href`/`src`/`action`/`<link>`/`poster`,
  framework router routes and `<Link to>`/`routerLink`, relative module imports,
  CSS `url(...)` assets. Grep starters:
  `href=["'][^http]|src=["'][^http]|url\(|to=["']|path:\s*["']`
- For each, resolve and verify the target exists (a real file, or a declared
  route). Misses → dead link / missing page. Anchor links (`#id`) → verify the
  `id` exists in the rendered page. Put results in the report's table.

### Accessibility (WCAG basics)
- Images without `alt`; form inputs without associated `<label>`/`aria-label`;
  buttons/links with no discernible text (icon-only without `aria-label`);
  non-semantic clickable `<div>`/`<span>` (no role/keyboard handler); heading
  order skips; color-contrast that looks too low; focus not visible / focus traps;
  `tabindex` misuse; missing `lang` on `<html>`; no skip-link.
- Grep starters: `<img(?![^>]*alt=)|role=|aria-|tabindex`

### Responsiveness & visual
- Missing/!broken viewport meta; fixed pixel widths that overflow small screens;
  no breakpoints; tap targets < ~44px on touch; horizontal scroll on mobile;
  content hidden behind notches/URL bars (`100vh` vs `100dvh`, safe-area insets).

### Broken / empty / error states
- No empty-state for lists that can be empty; no loading or error states for async
  data; forms with no validation feedback; actions with no success/failure
  feedback; dead-end flows (a button that goes nowhere).

### Runtime-only checks (note as needing live verification)
- Console errors/warnings, actual layout at real breakpoints, real focus order,
  network failures. These can't be confirmed by static reading — if the tools and
  the user's permission allow, offer to run the app and check; otherwise list them
  as "verify at runtime."
