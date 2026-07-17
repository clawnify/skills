---
name: organize-company-knowledge
description: >-
  Turn a dump of raw company files into a clean, foldered, citable Company
  Knowledge corpus — the company's brain. Use when the user has dropped many
  files into the Company Knowledge section (or connected a Drive) and wants the
  agent to distill and organize them: cluster by topic, distill each into a
  canonical markdown doc (merging across sources into reusable playbooks), file
  it into a folder, tag and link it, and propose it for review.
metadata:
  clawnify:
    displayName: Organize my knowledge
    trigger: manual
    inputs: []
---

# Organize Company Knowledge

The user dumped a pile of files and wants them turned into the company's
**brain** — organized, official Company Knowledge that every agent can draw on
and cite. Your job is to **curate, not mirror**: distill the material that is
*answer authority or reusable know-how* (policies, pricing, brand, SOPs, product
facts, FAQs, legal — **and the playbooks hidden inside the company's own
deliverables**) into a small set of clean markdown docs, each filed into the
right folder and linked to its neighbours — then hand the result to a human to
publish.

Nothing you create goes live automatically. You **propose**; a person reviews
and publishes from the dashboard.

## The mental model (read this first)

- **This builds the company brain, not a client filing cabinet.** Company
  Knowledge is the org-wide corpus every agent retrieves from and cites (`Per
  <Title> v<N>`). The goal of organizing a dump is to grow that shared brain.
  Almost everything answer-authority or reusable belongs here.

- **Deliverables are the richest brain material — distill the PATTERN, not the
  client.** An agency's dump is mostly *deliverables*: proposals, strategies,
  case studies, decks. A single raw proposal is not company fact — but the
  **pattern across all of them** ("how we structure a winning proposal", "how we
  run a festival digital strategy") is exactly the company's intelligence, and it
  IS citable. So **cluster deliverables across clients into canonical playbooks**
  — ONE "Proposal Playbook" distilled from seventeen proposals, not seventeen
  client-named docs. Strip the client-specific numbers; keep the reusable method.

- **Client-scoped facts are the rare exception, and you default away from them.**
  Only genuinely client-bound working facts (a specific client's budget, contacts,
  live campaign state) are *not* brain material. Those belong in that client's
  **Project** — but only when the org actually runs a per-client agent that needs
  them. Do **not** route deliverables there to "keep them separate": that buries
  the brain in silos. When unsure, it goes in the **brain** (the corpus).

- **Three axes inside Company Knowledge, kept separate.** A doc has:
  - a single **`folder`** — its ONE home, a slash-nested path like
    `playbooks/strategy`, shown as the browse tree (a doc appears in exactly one
    place). Omit for a top-level (Unfiled) doc.
  - many **`tags`** — cross-cutting labels for search/filter (e.g. `festival`,
    `enterprise`). NOT folders. A doc lives in one folder but can carry several
    tags.
  - optional **`meta`** — structured properties distilled from the material
    (e.g. `effective_date`, `owner`). Preserved for the human reviewer.

- **The dump's folder structure is a HINT, not a spec.** Whatever folders the
  client shipped (`Training Docs/`, `Strategies/`, `Proposals/`, `Brand/`) are
  *signal* about topic — read them, get inspired, reuse the parts already
  coherent. But you are curating, not copying: if their structure is messy,
  redundant, or one-doc-per-client, **re-derive a cleaner taxonomy** (e.g. collapse
  a `Proposals/` folder of per-client files into one `playbooks/` doc). Do not
  preserve a structure just because it was theirs.

- **Structure is also links.** Use `[[Doc Title]]` wikilinks between related docs.
  These render as the knowledge graph next to the folder tree. Forward references
  to docs you haven't written yet are fine — they mark what's worth writing next.

- **Binary assets are not knowledge.** Fonts, logo files, raw images, signed
  contract scans — these carry no distillable *answer authority*. Don't force
  them into the corpus. Reference them from the relevant doc (e.g. the brand doc
  lists the brand fonts and logo files by name); leave the blobs in the file
  library.

## Procedure

### 1. Survey the dump and read its structure as signal

- Use `company_knowledge_search` with broad, business-topic queries ("pricing and
  discounts", "brand voice and tone", "onboarding SOP", "digital strategy
  approach", "proposal structure", "case study results") to pull the salient
  content the user dropped in. Files uploaded to the Company Knowledge section are
  auto-indexed and searchable to you.
- Note the **folders the dump arrived in** — they hint at topic (a `Proposals/` or
  `Strategies/` folder tells you there's a playbook to distill out of many
  deliverables). Use them as inspiration, not as the final taxonomy.

> Note: today you read the dumped material through `company_knowledge_search`
> (semantic retrieval). If a `company_files_list` / `company_file_read` tool is
> available to you, enumerate and read each dumped file directly for a more
> complete pass — otherwise work topic-by-topic from search.

### 2. Cluster into canonical brain docs (merge across sources)

Group the material into a SMALL set of **topics**, each becoming one canonical
doc. The key move for a deliverable-heavy dump:

- **Many deliverables of one kind → one playbook.** Seventeen proposals → a
  "Proposal Playbook". Nine strategies → a "Digital Strategy Playbook" (split by
  segment only if the method genuinely differs, e.g. festivals vs. artists).
- **Training / SOP files → the how-we-work docs** (`doc_type: sop`).
- **Brand deck / overview → brand + positioning docs** (`doc_type: brand`).
- **Prefer a few dense, canonical docs over many thin ones.** If you're about to
  write a doc whose title is a client's name, stop — that's a signal you're
  mirroring a deliverable instead of distilling the pattern.

### 3. Learn the existing taxonomy BEFORE inventing folders

The single most important step — this is what keeps folders coherent instead of
sprouting `pricing`, `Pricing`, and `price-list` as three bogus folders.

- Call `company_knowledge_list` and read the `folder` values already in use across
  the corpus. That set **is** the current folder tree. Note the `tags` too (the
  label vocabulary), so you reuse those instead of coining synonyms.
- Reuse existing folders wherever the material fits. Only create a new folder when
  nothing existing fits, and prefer a shallow, obvious name (`playbooks`, `sales`,
  `brand`, `people`, `legal`, `pricing`, `product`). If the org has no docs yet,
  propose a small, conventional top level matching the doc types.

### 4. Distill each cluster → propose it

Call `company_knowledge_propose` once per topic with:

- **`title`** — the canonical name ("Proposal Playbook", "Enterprise Pricing").
  A pattern/method name, not a client name.
- **`description`** — one load-bearing line; it drives the knowledge index and
  ranking. Required.
- **`content`** — the distilled markdown. A clean, self-contained statement that
  **merges the sources and resolves overlaps** — the authoritative method the org
  would publish, not a summary of each file. Use `[[Other Doc Title]]` to link
  related docs.
- **`doc_type`** — one of policy / sop / pricing / product / brand / faq / legal /
  reference.
- **`folder`** — the single home path, e.g. `playbooks/strategy`. Reuse taxonomy
  from step 3. Omit for a top-level doc.
- **`tags`** — cross-cutting labels (e.g. `["festival","q3"]`), NOT the folder.
  Optional.
- **`meta`** — optional structured properties, e.g.
  `{ "owner": "Strategy", "reviewed": "2026-01-01" }`.
- **`source_ref`** — provenance: the source file id(s)/name(s) you distilled from.
  **Always set this** so a re-run recognises what's already organized instead of
  duplicating, and the reviewer can trace a doc back to its source.

You can also put the metadata in the markdown itself as YAML frontmatter — it is
parsed on save and wins over the fields, so this is equivalent and round-trips
with export (`folder`/`tags` are separate keys; any other key is preserved as a
`meta` property):

```markdown
---
title: Festival Digital Strategy Playbook
doc_type: sop
folder: playbooks/strategy
tags: [festival, launch]
owner: Strategy
---
# Festival Digital Strategy Playbook
The canonical method, distilled across past strategies. See [[Proposal Playbook]].
```

### 5. Avoid duplicates on a re-run

Before proposing, check `company_knowledge_list` for an existing doc with the same
`source_ref` or an obviously-equivalent title. If one exists, fold the new thinking
into that doc's proposal rather than creating a second near-duplicate.

### 6. Hand off to the human

When done, tell the user plainly: how many docs you proposed and the folders you
filed them under, that they should **review in the Knowledge → Map view and
publish** (nothing reaches their agents until they do), and anything you dropped
as asset/noise or were unsure how to file.

## Guardrails

- **Grow the brain; don't silo it.** Deliverables become shared, citable playbooks
  in the corpus. Never route reusable know-how into a per-client Project to "keep
  it separate" — that buries the company's intelligence. When unsure, it's brain.
- **Never publish.** `company_knowledge_propose` only. Publishing is a human,
  owner/admin action.
- **Curate, don't mirror.** Don't propose one doc per file, and don't preserve the
  dump's folder structure just because it was theirs. Merge across sources into
  playbooks, distill, drop noise, re-derive a cleaner taxonomy.
- **No secrets or per-customer facts in Company Knowledge.** Credentials, API keys,
  and one-customer working details (a specific client's budget/contacts) never
  belong in the org corpus — strip them when distilling a deliverable into a
  playbook.
- **Reuse the taxonomy** (step 3). Inventing ad-hoc folders/tags per file recreates
  the mess you're here to fix.
- **Ceiling:** this is a one-shot / periodic *curate* pass. Keeping a large,
  constantly-changing Drive continuously in sync is a different mechanism
  (scheduled connector → markdown sync), not this skill. If the user wants
  always-fresh mirroring of a big source, say so and stop — don't turn this into
  a whole-Drive re-crawler.
