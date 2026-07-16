---
name: fix-a-clawnify-agent
description: Diagnose and fix a Clawnify AI agent that veers off its instructions, ignores specific rules, forgets things, repeats the same mistake, or burns a lot of API credits. Use when someone says their agent (e.g. "Ash") won't follow instructions, keeps going off-path, or is expensive — read the agent's own sessions, find the real cause, and fix the instructions / sessions / flow instead of patching blind.
---

# Fix a misbehaving Clawnify agent

You are the planner and thinker. The Clawnify agent — the customer's
WhatsApp / Telegram / Slack / email "AI employee", running OpenClaw on their
server — is the executor. When it misbehaves your job is to **read what it
actually did, find the real cause, and change the durable inputs.** Not to
patch instructions blind, and not to burn the agent's own (expensive) model
tokens on trial-and-error.

The person you're helping is often **not** an AI specialist. So: diagnose
first, then **explain the cause in plain language and correct the
misconception** before you change anything.

## The one rule: don't debug blind

"Veers off / ignores instructions" has **three different causes with three
different fixes.** You can only tell which one by reading the agent's actual
session. Patching instructions without looking is exactly why it "isn't
heading in the right direction" — and every blind test run costs model tokens.

| Cause | What you'll see in the session | Fix |
|---|---|---|
| **1. Session bloat** | Was fine early, degrades over a long-running chat; forgets/repeats; high `contextTokens`; compaction/truncation in the transcript; the rule is simply *absent* from recent context | Scope a **fresh session per task**; reset/retire the bloated one |
| **2. Instruction patchwork** | The rule *was* in context but a **contradicting or older** rule won, or it's buried in a wall of text | **Restructure** `AGENTS.md` (don't add another patch) |
| **3. Wrong structure** | A repeatable multi-step task the agent improvises differently each time | Move it to a **flow** (deterministic rails) |

Bloat and patchwork also explain most **surprise API costs**: a bloated
session re-sends a huge context on *every* turn, and the patch-and-rerun loop
itself burns tokens. Measure it — don't guess (Step 1).

## Step 1 — Observe (read what actually happened)

```bash
clawnify sessions                       # every agent's sessions on the server, newest first
clawnify sessions --agent ash --json    # ash only, with token counters (contextTokens = bloat signal)
clawnify sessions history <key> --json  # the transcript of one run — where it actually veered
clawnify agents skills ash              # what ash can ACTUALLY run (workspace + shared skills)
```

Look for, in order:
- **Which failure mode** (table above): was the instruction *present and
  ignored* (→ patchwork/structure) or *compacted away / absent* (→ bloat)?
- **`contextTokens`** on the session — large and growing = bloat.
- **A missing skill** — if the instruction says "use skill X" but `agents
  skills` doesn't list it, the agent literally can't. The fix is to grant it
  (`clawnify agents grant-skill X --to ash`, or `--all` for every agent) if it's
  already authored in main, else author it first (`agents pull main` → add
  `skills/X/SKILL.md` → `agents push`) — not to reword the instruction. (Note:
  what's in the agent's *workspace* is only part of it — `agents skills` shows
  the effective set incl. shared managed-root skills.)
- **Where the tokens go** — one giant session, or repeated retries after a
  veer? That's your cost answer, from data, not a hypothesis.

> If `clawnify sessions` isn't available yet in this org, fall back: ask the
> person to describe (or paste) one failing run, and read the current
> `AGENTS.md` from the org's synced repo. You still diagnose before you fix.

## Step 2 — Explain, then fix

Tell the person, plainly, which of the three it is and why. Then:

**Bloat →** don't reuse one ever-growing chat for the same task. Start a fresh
session per task/topic (or `/reset` the runaway one). This fixes the veer
*and* the cost.

**Patchwork →** rewrite `AGENTS.md`, don't append to it. Structure it: **role
→ a short list of hard rules up front → the steps → one example of right vs
wrong.** Delete contradictions. A tight, prioritized instruction set is
followed far better than a long patched one.

**Structure →** author a **flow** for the repeatable task: deterministic steps,
each with a narrow instruction, and the heavy/deterministic parts (fetch data,
fill a template, render the deck) run *without* the model — less veer, less cost.

Where instructions live and how edits land:
- Edit **`AGENTS.md`** (and `skills/`, `flows/`). Push via the org's **GitHub
  sync repo** (auto-deploys to the box) or `clawnify deploy`. See the
  `use-the-clawnify-cli` skill for the commands.
- **No gateway restart needed** — the agent picks up new `AGENTS.md` / skills /
  flows on its **next session**.
- **Never edit any `CLAWNIFY-*.md`** file (`CLAWNIFY-SOUL/AGENTS/TOOLS.md`) —
  those are platform-owned and Clawnify overwrites them on every sync. Your
  edits there vanish.

## How Clawnify agents actually work (what people get wrong)

Preempt these — they cause "I set it up but the agent ignores it":

- **Adding a document to Knowledge does NOT mean the agent reads it.**
  (1) A doc must be **published by a human** — an agent "proposing" it only
  creates a draft. (2) Once published, the agent only *automatically* sees its
  **title + one-line description** (a capped, ~30-item index); the **actual
  content is fetched only when the agent chooses to search** it
  (`company_knowledge_search`). So if the agent ignores a doc, check: is it
  published? not hidden? still in the index? does its `AGENTS.md` tell it to
  search? — **re-uploading won't help.** Depth: the `organize-company-knowledge` skill.
- **Memory** is written by the agent via a tool (`memory_write`) and surfaced
  as an index + per-turn recall — it is **not** a file/folder you drop notes
  into.
- **The dashboard "recurring skill" cards are manual-click only** — the
  cron/every/at schedule shown is **not wired to a scheduler.** For real
  scheduled or event-driven work use a **cron job or a flow/webhook**, not a
  recurring-skill card.
- **Native channels** (WhatsApp/Telegram/Slack) receive messages "for free"
  via the allowlist — there is **no trigger/webhook to set up** for incoming
  messages. (Only Clawnify *integrations* have dashboard triggers.)

## Answering "would a different agent / a flow help?"

- **Same agent for the same task → confusion?** The agent identity isn't the
  problem; a reused, ever-growing **session** is (bloat, above). A second agent
  doesn't fix the mechanism and doubles what you manage. Fresh sessions do.
- **Would a flow be better?** Yes *if* the task is repeatable and structured —
  it constrains the agent and moves deterministic work off-model (less veer,
  less cost). No if the underlying step instructions are vague — a flow
  structures, it doesn't clarify.

## Guardrails

- **Diagnose before you change anything.** One transcript read beats five blind
  patches.
- **Never** edit `CLAWNIFY-*.md` (overwritten) — edit `AGENTS.md`.
- **Confirm with the human before anything irreversible or outward-facing:**
  sending messages on the agent's live channels, deleting an agent or app,
  billing/plan changes, rotating credentials. Self-service is not unattended
  autonomy.
- **Connecting an integration is a hard stop, not a confirm.** If a fix needs a
  Clawnify integration that `clawnify connections list` shows as not connected
  (○), you *cannot* connect it — that needs the user's OAuth sign-in in a
  browser. Run `clawnify connections connect <id>` and relay the steps to the
  user; never attempt it yourself or ask for their password.
- **Never put a secret through `clawnify env set`** — it exposes the value in
  the command and your context. Secrets and API keys go in the dashboard
  (Settings → Environment Variables / API Keys); guide the user there. Use
  `env set` only for non-secret config. And remember `agents create` / `env set`
  **restart the gateway** — confirm with the user before you run them on a live,
  busy box.
- Prefer the **smallest durable fix** (a session habit, a tightened `AGENTS.md`,
  one flow) over more instructions.
