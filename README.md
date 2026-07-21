# Leon's Agent Skills

A collection of agent skills by [Leon van Zyl](https://github.com/leonvanzyl). Skills work with Claude Code, Cursor, Codex, OpenCode, and any other agent supported by the [`skills` CLI](https://github.com/vercel-labs/skills).

## Install

Install all skills from this repo:

```bash
npx skills add leonvanzyl/skills
```

Browse what's available first:

```bash
npx skills add leonvanzyl/skills --list
```

Install a specific skill:

```bash
npx skills add leonvanzyl/skills --skill <skill-name>
```

## Skills

No skills published yet — first ones coming soon.

<!-- When adding a skill, list it here:
| Skill | Description |
| ----- | ----------- |
| [my-skill](skills/my-skill) | What it does |
-->

## Adding a new skill

1. Copy [`TEMPLATE.md`](TEMPLATE.md) to `skills/<skill-name>/SKILL.md` (folder name should match the skill name: lowercase, hyphens).
2. Fill in the frontmatter (`name`, `description`) and write the instructions.
3. Commit and push — that's it. The `skills` CLI discovers everything under `skills/` automatically; no registry or manifest to update.

A skill can also ship supporting files (scripts, references, examples) alongside its `SKILL.md` in the same folder.
