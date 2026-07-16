---
name: use-the-clawnify-cli
description: Use the Clawnify CLI end-to-end — authenticate, scaffold, run locally, deploy apps and websites, link projects, manage orgs, and inspect deployed apps. Use when deploying to Clawnify from a terminal or wiring the CLI into scripts and coding agents.
---

# Use the Clawnify CLI

Canonical skill: how to drive the `clawnify` CLI. Same content for
humans and for AI coding agents operating a terminal.

For **authoring app code** (stack, schema, tRPC patterns), use the
`build-a-clawnify-app` skill — this one covers the tooling around it.

## TL;DR

```bash
npm i -g clawnify        # or: npx clawnify@latest <command>
clawnify login           # browser sign-in, paste-code fallback
clawnify init my-app     # scaffold an app or website
cd my-app && clawnify dev
clawnify deploy          # → https://<slug>.apps.clawnify.com
```

- **Auth once, org lazily.** `login` only signs you in; the first
  command that needs an organization asks once and remembers it.
- **A directory links to one app.** `deploy` from a linked directory
  updates that app; `clawnify link <id-or-slug>` adopts an existing one.
- **Non-interactive flags exist for every prompt** — use them in
  scripts and CI (`-y`, `--update-existing`, `--create-new`, `--org`).

## Authenticate

```bash
clawnify login    # opens the browser; also prints the sign-in URL
clawnify whoami   # current user, active org (if linked)
```

`login` works from any browser: if the browser runs on the same
machine, sign-in completes automatically; otherwise the page shows a
code to paste back at the `Paste code here if prompted >` prompt
(useful over SSH — no port forwarding needed).

**Headless / CI / agents — skip `login` entirely.** Set `CLAWNIFY_TOKEN` (a
Supabase access token) and it wins over any stored session, reading/writing no
files; `CLAWNIFY_ORG_ID` (or a per-command `--org`) pins the org. This is how a
coding agent runs the CLI without a browser.

### Organizations

You belong to one or more orgs; apps live inside an org.

```bash
clawnify org list             # orgs you belong to (* marks active)
clawnify org use <slug-or-id> # switch the active org
```

No need to pick one up front — the first org-scoped command prompts
once and persists the choice. Any single command can override with
`--org <slug-or-id>`.

## Create and run locally

```bash
clawnify init [dir]                  # interactive: app or website
clawnify init my-app -y              # non-interactive defaults (blank app)
clawnify init --type app --template crud --name my-app
clawnify init --type website --name my-site
clawnify dev [dir] [--port 8787]     # local dev server
```

`init` also installs AI files (rules + skills + MCP config) for any
coding agents detected in the project; skip with `--no-ai-files`.

## Deploy

```bash
clawnify deploy                      # deploy the current directory
clawnify deploy path/to/app
clawnify deploy --from owner/repo    # deploy straight from GitHub
clawnify deploy --from owner/repo --path apps/web   # monorepo subdir
clawnify deploy --dry-run            # list files that would upload
```

First deploy creates the app and links the directory (writes a local
project file); later deploys update the same app. On a slug conflict
you'll be asked — or decide up front:

```bash
clawnify deploy -y                   # reuse/update the existing app
clawnify deploy --update-existing    # same, explicit
clawnify deploy --create-new         # take the suggested new slug
```

Link an existing deployed app to the current directory instead of
creating a new one:

```bash
clawnify link <app-id-or-slug>
```

## Inspect and manage deployed apps

```bash
clawnify ls                # list deployed apps in the active org
clawnify logs <app-id>     # build logs
clawnify open <slug>       # open the live app in the browser
clawnify rm <app-id>       # delete an app
clawnify pull schema --from <app-id-or-slug>   # fetch live schema.sql
clawnify docs [slug]       # usage README for an app (org index if omitted)
```

`pull schema` refuses to clobber local uncommitted changes unless you
pass `--force`.

## Manage agents (the FDE surface)

Drive an org's AI agents from the terminal — observe them and fix them. See the
`fix-a-clawnify-agent` skill for the workflow; this is the command list. All
org-scoped and **repo-independent** (reads session data + the box, never a
GitHub repo). Add `--server <id>` when the org has more than one server.

```bash
clawnify servers list                       # the org's servers (an org can have several)
clawnify agents list                        # agents on a server (main + specialists)
clawnify agents skills [agent]              # what an agent can ACTUALLY run (not just its workspace)
clawnify sessions --agent <slug> --json     # stored sessions + token counters (contextTokens = bloat)
clawnify sessions history <key> --json      # one session's transcript — where it veered
clawnify agents pull [dir] --agent <slug>   # fetch AGENTS.md + skills/ + flows/ to edit
clawnify agents push [dir]                  # push edits back (additive; next session, no restart)
clawnify agents grant-skill <s> --to <a>    # share a main-authored skill (or --all for every agent)
clawnify agents create <slug> --yes         # new specialist (RESTARTS the gateway)
clawnify env set <KEY> <VALUE>              # org custom env — NON-SECRET only (restarts gateway)
clawnify connections connect <id>           # how to connect an integration (dashboard OAuth; you can't)
```

**Boundary:** everything above is autonomous *except* the human-auth actions —
connecting an integration (OAuth) and handing over a secret/API key. For those,
explain the dashboard steps; never fake them. `agents create` / `env set`
restart the gateway, so confirm before running them on a busy box.

## Connections & env names

```bash
clawnify connections list          # the org's naming catalog (● connected / ○ not)
clawnify connections connect <id>  # how to connect one — dashboard OAuth; the CLI/agent can't do it for you
```

Lists the active org's integrations (canonical ids + connected state),
provider API keys (fixed names + set state), and custom env vars. Read
it before declaring `credentials` or env names in `clawnify.json` and
reuse the exact identifiers it shows — a misspelled id or key name
doesn't fail the build, it silently resolves to nothing at runtime.

## Workspaces (one repo, one org's apps)

```bash
clawnify workspace init [dir] [--org <slug-or-id>]
```

Scaffolds a monorepo bound to an org; apps scaffolded inside land
under `apps/` and inherit the org on deploy.

## Wire up coding agents

```bash
clawnify ai-files install   # rules + skills + MCP server in one step
clawnify skills list        # bundled skills
clawnify skills add <name>  # install one into detected agents
clawnify mcp                # stdio MCP bridge (Claude Desktop, etc.)
```

`ai-files install` detects agent config dirs in the project (Claude
Code, Cursor, OpenClaw, Codex, Aider) and writes the always-on rules
block, installs bundled skills, and adds the Clawnify MCP server to
`.mcp.json`. Target one agent with `--agent <name>`; undo with
`clawnify ai-files uninstall`.

## The raw escape hatch

```bash
clawnify api /servers                               # any GET
clawnify api /servers/<id>/agents -X POST -d '{…}'  # any method + JSON body
```

`clawnify api <path>` makes an authenticated request to any Clawnify API path —
same auth and org scoping as every other command (it can only do what you can).
Use it for anything the porcelain doesn't wrap; the curated commands stay small
because this is always there.

## Tips

- `npx clawnify@latest <command>` always runs the newest release.
- Every org-scoped command accepts `--org` — prefer it in scripts over
  relying on the saved active org.
- If sign-in ever fails mid-flow, just run `clawnify login` again; it
  is safe to repeat.
