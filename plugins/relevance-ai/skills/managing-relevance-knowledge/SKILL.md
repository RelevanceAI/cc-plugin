---
name: managing-relevance-knowledge
description: Use whenever the user asks an agent to seed or update a structured-row store (CRM contacts, leads, support tickets) or to inspect knowledge-base metadata. Covers knowledge-base CRUD and row CRUD. File / document-URL ingestion is not available via the tools — direct the user to the Knowledge tab in the Relevance app UI. For Q&A over knowledge, attach the base to an agent and let the built-in `agent_knowledge` tool answer.
---

# Managing Relevance AI Knowledge

A _knowledge base_ is the user-facing name; the technical id (used as
`knowledge_set` in every tool) is what the API stores.

> **Feeding a base into a knowledge-search tool step?** Search runs only on a
> knowledge set, and its source value must be the **prefixed** form
> `knowledge:<knowledge_set_id>` — a bare id / dataset name resolves to a plain
> dataset and fails with _"Search can only be performed on a knowledge set."_ In a
> tool, wire the source to a `content_type: "knowledge_set"` input rather than a
> plain string. See
> [`managing-relevance-tools` › Knowledge Search](../managing-relevance-tools/patterns.md#knowledge-search-over-a-knowledge-set).

## File ingestion

File / document-URL ingestion is not available via the tools. If the user wants
any kind of file (local or remote URL) added to a knowledge base, point
them to the **Knowledge** tab in the Relevance app UI and stop. Do not substitute a
web-scrape tool, a PDF extractor, or a system-prompt paste-in — the
tools below operate on **structured rows** and on **bases that already
exist**, not on file uploads.

## Decision tree

```
Need to ...                                    Tool
─────────────────────────────────────────────  ─────────────────────────────────
List the bases I can see                       relevance_list_knowledge_sets
Inspect ONE base's metadata + schema           relevance_get_knowledge_set
Create a new base                              relevance_create_knowledge_set
Rename a base / change description             relevance_update_knowledge_set
Permanently delete a base                      relevance_delete_knowledge_set

List rows (with filters / paging)              relevance_list_knowledge_rows
Get one row by document_id                     relevance_get_knowledge_row
Add structured rows                            relevance_add_knowledge_rows
Update specific fields on existing rows        relevance_update_knowledge_rows
Delete rows by document_id                     relevance_delete_knowledge_rows

Export a base to CSV / XLSX                    relevance_export_knowledge
```

> **Answering questions over a knowledge base is not done via these
> tools.** Attach the base to an agent (`relevance_update_agent` with a
> `knowledge` entry of `usage_type: "tool"`) and the agent will answer
> through its built-in `agent_knowledge` tool. See "Attaching a
> knowledge base to an agent" below.

## Tool reference

| Tool                              | Required                        | Read-only?  |
| --------------------------------- | ------------------------------- | ----------- |
| `relevance_list_knowledge_sets`   | —                               | ✅          |
| `relevance_get_knowledge_set`     | `knowledge_set`                 | ✅          |
| `relevance_create_knowledge_set`  | `knowledge_set`                 | ❌ approval |
| `relevance_update_knowledge_set`  | `knowledge_set`, `updates`      | ❌ approval |
| `relevance_delete_knowledge_set`  | `knowledge_set`                 | ❌ approval |
| `relevance_list_knowledge_rows`   | `knowledge_set`                 | ✅          |
| `relevance_get_knowledge_row`     | `knowledge_set`, `document_id`  | ✅          |
| `relevance_add_knowledge_rows`    | `knowledge_set`, `rows`         | ❌ approval |
| `relevance_update_knowledge_rows` | `knowledge_set`, `updates`      | ❌ approval |
| `relevance_delete_knowledge_rows` | `knowledge_set`, `document_ids` | ❌ approval |
| `relevance_export_knowledge`      | `knowledge_set`                 | ❌ approval |

## When create returns "already exists"

`relevance_create_knowledge_set` is not idempotent. On a duplicate id it
returns `isError: true` with `row_count` and `ingestion_status` on the
existing base. Inspect with `relevance_list_knowledge_rows` before
deciding to reuse, wipe (`relevance_delete_knowledge_set` then
re-create), or pick a new id. Never wipe without explicit user approval.

## End-to-end: CRM-style agent

```
relevance_create_knowledge_set({ knowledge_set: "leads", description: "Inbound sales leads" })

relevance_add_knowledge_rows({
  knowledge_set: "leads",
  rows: [
    { name: "John", email: "john@example.com", company: "Acme",   status: "new" },
    { name: "Jane", email: "jane@example.com", company: "Tech Co", status: "new" }
  ]
})

// Note: filter fields use the data. prefix
relevance_list_knowledge_rows({
  knowledge_set: "leads",
  filters: [{ field: "data.status", filter_type: "exact_match", condition_value: "new" }]
})

relevance_update_knowledge_rows({
  knowledge_set: "leads",
  updates: [{ document_id: "<uuid>", data: { status: "contacted" } }]
})

// Attach base via relevance_update_agent (the patch lands as a draft;
// publish via relevance_publish_agent on user confirmation — see
// managing-relevance-agents).
relevance_update_agent({
  agent_id: "<agent_id>",
  patch: {
    knowledge: [
      { knowledge_set: "leads", usage_type: "tool" }
    ]
  }
})
```

## Attaching a knowledge base to an agent

Pass a `knowledge` array on `relevance_update_agent`'s `patch`. Each
item requires `knowledge_set` and `usage_type`; `prompt_description` is
optional.

| Field                | Required | Notes                                                                                                                           |
| -------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `knowledge_set`      | ✅       | id of an existing base                                                                                                          |
| `usage_type`         | ✅       | `"tool"` → exposed as an `agent_knowledge` tool the agent calls; `"instructions"` → injected into the system prompt at run time |
| `prompt_description` | ❌       | overrides the auto-generated description shown to the agent                                                                     |

```
relevance_update_agent({
  agent_id: "<agent_id>",
  patch: {
    knowledge: [
      { knowledge_set: "leads", usage_type: "tool" }
    ]
  }
})
```

The patch is partial-merge, but the `knowledge` array itself is
replaced wholesale on update — include every base you want the agent
to retain in a single call.

## Reference

- [tables.md](tables.md) — filter types, response shapes, bulk-update
  patterns, `data.*` field nesting, common pitfalls.
