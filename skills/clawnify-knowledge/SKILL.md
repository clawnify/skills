---
name: clawnify-knowledge
description: Route a question to the right Clawnify knowledge source before answering or building. Covers how-to-use-Clawnify product docs, your org's company knowledge (brand, pricing, policy, SOPs), a specific client or project's reference, and durable memory. Use whenever a task needs a company fact, a platform how-to, or client-specific context from a connected Clawnify organization.
---

# Clawnify knowledge router

You are connected to a Clawnify organization through the Clawnify connector.
Several knowledge sources sit behind it, and each answers a different kind of
question. Pick the right one and retrieve from it: these sources are current
and scoped to this organization, and your training data is neither.

## Route by question type

| The question is about | Use | Notes |
|---|---|---|
| How to use Clawnify itself: deploy an app, set up an agent, connections, platform features | docs.clawnify.com | Product docs; `https://docs.clawnify.com/llms.txt` indexes them. Check them before building against the platform. |
| Which of the org's apps handles something | `clawnify_docs_search` | Searches the org's apps by name and description. |
| The company's own facts: brand, pricing, policy, SOP, legal, FAQ | `company_knowledge_search`, then `company_knowledge_get` | The org's source of truth, a versioned wiki linked with `[[wikilinks]]`. |
| One specific client, engagement or project | `projects_list`, then `project_search` | Working reference for that project, not org-wide authority. |
| Something decided or learned before | `memory_recall` | Save durable facts back with `memory_write`, and `memory_pin` the ones that matter most. |
| A third-party library, framework or API | That vendor's own docs | Not in the Clawnify corpus. |
| Code in the repository you are working in | Read the code | The code beats any summary of it. |

## Rules

- **Retrieve, don't recall.** For a company, product or client fact, query the
  source. Never answer a Clawnify fact from memory of training data.
- **Cite company knowledge.** An answer grounded in a `company_knowledge` result
  cites it as `Per <Title> v<N>`.
- **A proposal is not published.** `company_knowledge_propose` creates a draft
  that a person reviews. Never present a proposal as a live fact.
- **No secrets or per-customer facts in company knowledge.** Those belong in a
  project, or nowhere.
- **Look before you build.** Before creating an app or an agent, check
  docs.clawnify.com for the current contract and `clawnify_list_apps` or
  `clawnify_list_agents` for what already exists.
