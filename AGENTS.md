# Working in this repo

This repo holds agent skills used on real client work. A change here reaches
people through three different routes, and the job isn't done until all three
have been considered.

## This repo is the source of truth

`H:\skills` is where skills are edited. Two other places carry copies:

| Where | What it is | What to do |
| --- | --- | --- |
| `H:\DreamForgeSoftwareAgentSkills` (`skills/<name>/`) | The plugin distribution, published separately | **Sync after every change.** It drifts silently |
| `leonvanzyl/skills` | Upstream, the skill this forks | Offer changes as a PR he can decline |

**Never edit the DreamForge copy directly.** It has fallen behind before — three
refinements sat only in this repo for weeks — and edits made there are lost the
next time it is synced.

Syncing it is a copy and a commit:

```bash
cp -r "H:/skills/<skill>/." "H:/DreamForgeSoftwareAgentSkills/skills/<skill>/"
```

**Its files are CRLF and this repo's are LF**, so a plain `diff -rq` reports every
file as different when almost nothing has changed. Always compare with
`diff --strip-trailing-cr` before concluding the copies have diverged. Its
`core.autocrlf=true` normalises on commit, so copying LF files in produces a clean
diff with no line-ending noise.

## Offering changes upstream

Enhancements that came out of real builds are offered to Leon as a pull request —
**always as a menu he can decline, never as a request.** Include everything,
including the opinionated parts; the point is that he gets to choose, not that we
pre-select what seems safe.

Practicalities, each learned the hard way:

- **Branch from `upstream/main`, not from this repo's `main`.** The fork has
  diverged too far, and the layout differs — upstream nests skills under
  `skills/<name>/` while this repo has them at the root. A PR from `main` shows
  dozens of commits of restructuring and is unreviewable.
- **One commit per concern**, so he can cherry-pick. Where an enhancement forces
  churn in his repo (a renamed variable, a changed default), offer both: the
  low-churn version as an earlier commit, the full version later in the sequence,
  and explain the choice in the PR body.
- **A cross-repo PR needs a fork inside his network.** `brainit-consulting/skills`
  is a fork of `lexxxical/leonvanzyl-skills` — a different network — so GitHub
  refuses with "no commits between". Push PR branches to
  **`brainit-consulting/skills-upstream-pr`**, which exists for this.
- `main` here tracks `origin`. It once tracked `upstream`, which meant a bare
  `git push` aimed at Leon's repository.

## House style

The prose is the product. Match it or an addition reads as bolted on.

- Reference files open with `# Title`, `Last verified: YYYY-MM-DD`, and a
  `**Purpose:**` line, and close with a `## Verify` section of checkable items.
- `SKILL.md` stays thin. **No commands in it, ever** — commands, package names and
  config live in `references/`, loaded only for the branch the user chose.
- Plain language, aimed at "a smart friend who doesn't code". British spelling.
- **State the failure a rule prevents, and why the error message won't point at
  it.** A rule with no failure attached hasn't earned its place. The recurring
  test is whether meeting it by surprise would cost an hour.
- Prefer checks that can be measured over ones that can be eyeballed:
  `getComputedStyle(...)`, `getBoundingClientRect()`, an HTTP status code.

**The evidence bar: rules come from real builds, not from speculation.** When
something was observed but the cause was never proven, write it as a Verify line
("check X matches Y") rather than a procedure resting on a guess — and say which
it is.

Commits are conventional-ish — `fix(<skill>):`, `feat(<skill>):`, `docs:` — with a
body that explains the failure in the same voice as the skill itself.
