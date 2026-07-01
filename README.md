# Clawnify Agent Skills

Open-source [Agent Skills](https://github.com/vercel-labs/skills) for building on
[Clawnify](https://clawnify.com) — the delivery platform for AI work. Each skill
is a self-contained `SKILL.md` that teaches an AI coding agent (Claude Code,
Cursor, Codex, OpenClaw, …) how to work with a part of the platform. No Clawnify
account required.

## Install

```bash
# Install one skill into your project's agent dirs
npx skills add https://github.com/clawnify/skills --skill build-a-clawnify-app

# List what's available
npx skills add https://github.com/clawnify/skills --list
```

The skill is written into whatever agent directories exist in the current
project (`.claude/skills/`, `.cursor/rules/`, `.openclaw/skills/`, …).

Already using the Clawnify CLI? The same skills ship bundled, version-pinned to
your CLI:

```bash
clawnify skills add build-a-clawnify-app
clawnify skills list
```

## Available skills

| Skill | What it teaches |
|-------|-----------------|
| [`build-a-clawnify-app`](./skills/build-a-clawnify-app/SKILL.md) | Author a Clawnify app end-to-end — the canonical Hono + React + tRPC + `@clawnify/db` stack, `schema.ts` as source of truth, MCP-tagged tRPC procedures, and `X-Clawnify-*` identity headers. |

## About this repo

This is the public distribution mirror. Skills are authored in the Clawnify
codebase and published here; the same `SKILL.md` files are what the Clawnify CLI
bundles, so `npx skills` and `clawnify skills` never drift.
