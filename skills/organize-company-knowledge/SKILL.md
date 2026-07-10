---
name: organize-company-knowledge
description: >-
  Turn a dump of raw company files into a clean, foldered, citable Company
  Knowledge corpus. Use when the user has dropped many files into the Company
  Knowledge section (or connected a Drive) and wants the agent to distill and
  organize them: cluster by topic, distill each into a canonical markdown doc,
  file it into a folder (tag), link related docs, and propose it for review.
metadata:
  clawnify:
    displayName: Organize my knowledge
    trigger: manual
    inputs: []
---

# Organize Company Knowledge

The user dumped a pile of files and wants them turned into organized, official
company knowledge. Your job is to **curate, not mirror**: distill the material
that is *answer authority* (policies, pricing, brand, SOPs, product facts, FAQs,
legal) into a small set of clean markdown docs, each filed into the right folder
and linked to its neighbours — then hand the result to a human to publish.

Nothing you create goes live automatically. You **propose**; a person reviews
and publishes from the dashboard.

## The mental model (read this first)

- **Folders are tags.** There is no separate folder field. A doc's folder is its
  `tags`. A nested folder is a slash-joined tag: a doc tagged `pricing/enterprise`
  shows up under **pricing → enterprise** in the dashboard's folder tree. Untagged
  docs sit at the root.
- **Structure is also links.** Use `[[Doc Title]]` wikilinks between related docs.
  These render as the knowledge graph next to the folder tree. Forward references
  to docs you haven't written yet are fine — they mark what's worth writing next.
- **Distill, don't copy.** A 40-page pricing PDF becomes ONE canonical `pricing/`
  doc, not 40 pages of extracted noise. The value is the curated statement, not
  the raw file.
- **Governed and citable.** Published Company Knowledge is what agents cite ("Per
  *Refund Policy* v3"). Keep each doc a single, authoritative statement.

## Procedure

### 1. Learn the existing taxonomy BEFORE inventing folders

The single most important step — this is what keeps folders coherent instead of
sprouting `pricing`, `Pricing`, and `price-list` as three bogus folders.

- Call `company_knowledge_list` and read the `tags` already in use across the
  corpus. That set **is** the current folder structure.
- Reuse existing folders wherever the material fits. Only create a new folder
  (tag) when nothing existing fits, and prefer a shallow, obvious name
  (`sales`, `support`, `people`, `legal`, `pricing`, `product`).
- If the org has no docs yet, propose a small, conventional top level:
  `policy`, `pricing`, `brand`, `sops`, `legal`, `faq` — matching the doc types.

### 2. Survey the dumped material

- Use `company_knowledge_search` with broad, business-topic queries ("pricing and
  discounts", "refunds and returns", "brand voice and tone", "onboarding SOP",
  "PTO and leave", "data processing / DPA") to pull the salient content the user
  dropped in. Files uploaded to the Company Knowledge section are auto-indexed and
  searchable to you.
- Group what you find into **topics**. Each coherent topic → one proposed doc.
- Prefer a few dense, canonical docs over many thin ones.

> Note: today you read the dumped material through `company_knowledge_search`
> (semantic retrieval). If a `company_files_list` / `company_file_read` tool is
> available to you, enumerate and read each dumped file directly for a more
> complete pass — otherwise work topic-by-topic from search.

### 3. For each topic, distill → file → link → propose

Call `company_knowledge_propose` once per topic with:

- **`title`** — the canonical name ("Refund Policy", "Enterprise Pricing").
- **`description`** — one load-bearing line; it drives the knowledge index and
  ranking. Required.
- **`content`** — the distilled markdown. A clean, self-contained statement.
  Use `[[Other Doc Title]]` to link related docs.
- **`doc_type`** — one of policy / sop / pricing / product / brand / faq / legal /
  reference.
- **`tags`** — the folder path, e.g. `["pricing/enterprise"]`. This is where it
  files. Reuse taxonomy from step 1.
- **`source_ref`** — provenance: what you distilled it from (the source file's id
  or name). **Always set this.** It lets a re-run recognise what's already been
  organized instead of duplicating, and lets the reviewer trace a doc back to its
  source.

You can also put the metadata in the markdown itself as YAML frontmatter — it is
parsed on save and wins over the fields, so this is equivalent and round-trips
with export:

```markdown
---
title: Enterprise Pricing
doc_type: pricing
tags: [pricing/enterprise]
---
# Enterprise Pricing
Canonical statement here. See [[Discount Rules]].
```

### 4. Avoid duplicates on a re-run

Before proposing, check `company_knowledge_list` for an existing doc with the
same `source_ref` or an obviously-equivalent title. If one exists, update the
thinking into that topic's proposal rather than creating a second near-duplicate.

### 5. Hand off to the human

When done, tell the user plainly: how many docs you proposed and the folders you
filed them under, and that they should **review in the Knowledge → Map view and
publish** — nothing reaches their agents until they do. Point out anything you
were unsure how to file.

## Guardrails

- **Never publish.** `company_knowledge_propose` only. Publishing is a human,
  owner/admin action.
- **Curate, don't mirror.** Don't propose one doc per file. Merge, distill, drop
  noise. If a file isn't answer-authority (a random spreadsheet, a signed
  contract PDF), leave it as a raw file — don't force it into the corpus.
- **No secrets or per-customer facts.** Credentials, API keys, and one-customer
  details never belong in Company Knowledge.
- **Reuse the taxonomy** (step 1). Inventing ad-hoc tags per file recreates the
  mess you're here to fix.
- **Ceiling:** this is a one-shot / periodic *curate* pass. Keeping a large,
  constantly-changing Drive continuously in sync is a different mechanism
  (scheduled connector → markdown sync), not this skill. If the user wants
  always-fresh mirroring of a big source, say so and stop — don't turn this into
  a whole-Drive re-crawler.
