# Clawnify for Claude

Work on your [Clawnify](https://clawnify.com) organization from Claude. This
plugin connects Claude to your organization's company knowledge, project
reference, memory, AI agents, apps and flows, and adds skills that teach Claude
how to build and run things on Clawnify. It works in Claude chat, Cowork and
Claude Code.

## What you get

- **The Clawnify connector** (`https://mcp.clawnify.com/mcp`): search and cite
  your company knowledge, read a client's project reference, recall and save
  durable memory, inspect and adjust your agents, and create, update and
  publish apps and flows.
- **Skills** that load when the task calls for them:

| Skill | Use it to |
|-------|-----------|
| [`clawnify-knowledge`](https://github.com/clawnify/skills/blob/main/skills/clawnify-knowledge/SKILL.md) | Pick the right knowledge source (product docs, company knowledge, a project, memory) before answering, and cite what it found. |
| [`fix-a-clawnify-agent`](https://github.com/clawnify/skills/blob/main/skills/fix-a-clawnify-agent/SKILL.md) | Find out why an agent ignores its instructions, forgets, repeats itself or costs too much, and fix the cause. |
| [`organize-company-knowledge`](https://github.com/clawnify/skills/blob/main/skills/organize-company-knowledge/SKILL.md) | Turn a dump of company files into a clean, foldered, citable knowledge base, proposed for a person to publish. |
| [`build-a-clawnify-app`](https://github.com/clawnify/skills/blob/main/skills/build-a-clawnify-app/SKILL.md) | Build an app on the Clawnify stack: Hono, React, `@clawnify/db`, typed routes and platform identity. |
| [`build-a-clawnify-website`](https://github.com/clawnify/skills/blob/main/skills/build-a-clawnify-website/SKILL.md) | Build and ship an Astro website with built-in forms and the draft-then-publish flow. |
| [`use-the-clawnify-cli`](https://github.com/clawnify/skills/blob/main/skills/use-the-clawnify-cli/SKILL.md) | Drive the `clawnify` CLI from a terminal: sign in, scaffold, run locally, deploy. |

## Install

**Claude (chat and Cowork):** in **Customize > Plugins**, add Clawnify from the
directory, or choose **Add > Add marketplace** and enter `clawnify/skills`.
Then open the plugin's **Connectors** tab and connect Clawnify, signing in with
your Clawnify account.

**Claude Code:**

```bash
/plugin marketplace add clawnify/skills
/plugin install clawnify@clawnify
```

Then run `/mcp`, pick `clawnify`, and sign in with your Clawnify account in
the browser window that opens.

**Skills only, any coding agent:** the skills also install on their own, with
no Clawnify account, through [`npx skills`](https://github.com/vercel-labs/skills):

```bash
npx skills add https://github.com/clawnify/skills --skill build-a-clawnify-app
```

## Use it

Ask Claude about your company ("what's our refund policy?"), a client ("what
did we agree with Acme on scope?"), or your agents ("why does my support agent
keep ignoring the escalation rule?"). Claude picks the matching skill, calls the
Clawnify connector, and cites company knowledge as `Per <Title> v<N>`.

## Data

The plugin holds no data of its own. When Claude calls a Clawnify tool, the
request, including the text of your question or the content Claude is writing,
goes to `mcp.clawnify.com`, Clawnify's server, signed in as you. Every request
is scoped to your organization and your role in it. The skills are plain
instructions and send nothing anywhere. Clawnify's
[privacy policy](https://app.clawnify.com/privacy) covers how that data is handled.

## License

MIT, see [LICENSE](./LICENSE).
