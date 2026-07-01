# Clawnify Agent Skills

Open-source [Agent Skills](https://github.com/vercel-labs/skills) for building
on Clawnify. Each skill is a self-contained `SKILL.md` that teaches an AI coding
agent (Claude Code, Cursor, Codex, OpenClaw, …) how to work with a part of the
platform.

## Install with `npx skills`

Any agent that supports the Agent Skills spec can install these directly from
this repo — no Clawnify account required:

```bash
# Install one skill
npx skills add https://github.com/clawnify/skills --skill build-a-clawnify-app

# See what's available first
npx skills add https://github.com/clawnify/skills --list
```

The skill is written into whatever agent directories you have in the current
project (`.claude/skills/`, `.cursor/rules/`, `.openclaw/skills/`, …).

## Install with the Clawnify CLI

If you already use the Clawnify CLI, the same skills ship bundled — no network
fetch, version-pinned to your CLI:

```bash
clawnify skills add build-a-clawnify-app   # one skill
clawnify skills add --all                  # everything
clawnify skills list                       # what's bundled
```

Both paths install the **same** `SKILL.md` files from this directory — the CLI
bundler reads them at build time (see `apps/cli/scripts/bundle-skills.mjs`), so
there is no drift between `npx skills` and `clawnify skills`.

## Available skills

| Skill | What it teaches |
|-------|-----------------|
| [`build-a-clawnify-app`](./build-a-clawnify-app/SKILL.md) | Author a Clawnify app end-to-end — the canonical Hono + React + tRPC + `@clawnify/db` stack, `schema.ts` as source of truth, MCP-tagged tRPC procedures, and `X-Clawnify-*` identity headers. |

## Adding a skill

Skills are authored in the Clawnify monorepo (`skills/` there is canonical) and
published to the public [`clawnify/skills`](https://github.com/clawnify/skills)
mirror. To add one:

1. Create `skills/<name>/SKILL.md` with YAML frontmatter (`name`, `description`).
   Keep it public-safe — no internal-only links or references.
2. Add a row to `SKILLS` in `apps/cli/scripts/bundle-skills.mjs` so the Clawnify
   CLI ships it too.
3. Bump the CLI version and release, then sync the `skills/` tree to the public
   mirror.
