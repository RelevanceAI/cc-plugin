---
name: managing-relevance-knowledge
description: Use whenever the user wants records or documents in a knowledge base, or an agent to use one — seeding or updating structured rows (CRM contacts, leads, support tickets), turning a short document you can read into knowledge an agent always follows, attaching a base to an agent (prompt mode or tool mode), running an agent, workforce or tool over every row, or inspecting knowledge-base metadata. A file in the workspace or attached to the conversation goes in with relevance_import_knowledge_file, or from the Knowledge tab in the Relevance app UI; never retype a file's rows.
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

## Files and documents

`relevance_import_knowledge_file` uploads a file into a knowledge table: `path`
for a file in the workspace (`/user/...` or `/project/...`), or `file_url` for a
file attached to the conversation. With `file_url`, pass the `file_name` it was
attached with if you have it; otherwise the name at the end of the link is used.

- **A spreadsheet or CSV file** becomes one row per row. A workbook's first
  sheet is imported unless you name another with `sheet`; when several sheets
  hold rows, ask the user which one, and import one sheet per call. A JSON
  file becomes one row per record.
- **A document or a PDF** becomes one row of its text. Turning a long document
  or a PDF into searchable knowledge, or reading a scanned PDF with OCR, is done
  from the **Knowledge** tab in the Relevance app.
- The import creates the table if it doesn't exist and otherwise adds to it,
  so importing the same file twice adds its rows twice. A name that belongs to
  a deleted table is refused; import under another name.
- A finished import returns `table_row_count`, the rows the table now holds,
  and `columns`. A filter names a column as `data.<column>`.
- `in_progress` means the upload is still running: check it with
  `relevance_get_knowledge_set`, as `polling_hint` says, and never import the
  file again.
- An import refused because an earlier upload into the table is still running
  added nothing: wait for that upload to finish, then import again.
- If the import tool isn't available, send the user to the Knowledge tab.

A short document an agent should always follow (a style guide, a policy, a page
of FAQs) whose text you can read can also go in as one row per document,
attached in prompt mode. See "Short documents an agent should always follow"
below. Either way the text goes in a knowledge base rather than being pasted
into `system_prompt`, so the base can be updated without editing the prompt.

`relevance_add_knowledge_rows` is for a handful of records the user typed into
the conversation or you composed. Never copy a file's rows into it, however few:
the file goes in by import. Whichever way rows go in, confirm the columns with
the user first, and tell them that a knowledge base cannot be kept private to
one person: others in the project can see it, and who exactly is set in the
Relevance app.

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
Add a few rows the user typed (never a file's) relevance_add_knowledge_rows
Update specific fields on existing rows        relevance_update_knowledge_rows
Delete rows by document_id                     relevance_delete_knowledge_rows
Delete columns and their values in every row   relevance_delete_knowledge_columns

Export a base to CSV / XLSX                    relevance_export_knowledge
Turn a file into a base, or add a file to one  relevance_import_knowledge_file

Run an agent or workforce once per row         relevance_start_bulk_run, or Run in bulk in the app
Fill a column by running a tool on every row   A tool column + Run tool on all rows, in the app

Attach a base to an agent                      relevance_update_agent (patch.knowledge)
```

> **Answering questions over a knowledge base is not done via these
> tools.** Attach the base to an agent with `relevance_update_agent`: prompt
> mode (`usage_type: "instructions"`) for a few short documents, tool mode
> (`"tool"`) for anything bigger. See "Attaching a knowledge base to an agent"
> below.

> **Running something over every row** isn't done with these tools. Start an
> agent or workforce once per row with the bulk-run tool, or Run in bulk on its
> page in the app; a tool column's Run tool on all rows is started in the app.
> See [Work Over Many Items](../managing-relevance-agents/many-items.md).
> Results are collected in the table's rows, and the user gets a file by
> exporting the table (`relevance_export_knowledge`, or the Knowledge tab).
> Neither route edits an uploaded file, so never promise an edited copy of it.

## Tool reference

| Tool                                 | Required                              | Read-only?  |
| ------------------------------------ | ------------------------------------- | ----------- |
| `relevance_list_knowledge_sets`      | —                                     | ✅          |
| `relevance_get_knowledge_set`        | `knowledge_set`                       | ✅          |
| `relevance_create_knowledge_set`     | `knowledge_set`                       | ❌ approval |
| `relevance_update_knowledge_set`     | `knowledge_set`, `updates`            | ❌ approval |
| `relevance_delete_knowledge_set`     | `knowledge_set`                       | ❌ approval |
| `relevance_list_knowledge_rows`      | `knowledge_set`                       | ✅          |
| `relevance_get_knowledge_row`        | `knowledge_set`, `document_id`        | ✅          |
| `relevance_add_knowledge_rows`       | `knowledge_set`, `rows`               | ❌ approval |
| `relevance_update_knowledge_rows`    | `knowledge_set`, `updates`            | ❌ approval |
| `relevance_delete_knowledge_rows`    | `knowledge_set`, `document_ids`       | ❌ approval |
| `relevance_delete_knowledge_columns` | `knowledge_set`, `fields`             | ❌ approval |
| `relevance_export_knowledge`         | `knowledge_set`                       | ❌ approval |
| `relevance_import_knowledge_file`    | `knowledge_set`, `path` or `file_url` | ❌ approval |

## When create returns "already exists"

`relevance_create_knowledge_set` is not idempotent. On a duplicate id it
returns `isError: true` with `row_count` and `ingestion_status` on the
existing base. Inspect with `relevance_list_knowledge_rows` before
deciding to reuse, wipe (`relevance_delete_knowledge_set` then
re-create), or pick a new id. Never wipe without explicit user approval.
A deleted base's id stays reserved: `relevance_create_knowledge_set`
refuses it, and the base cannot be deleted again, so pick a new id. Other
writes to that id land on the hidden deleted base, where nobody sees them.

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

Pass a `knowledge` array on `relevance_update_agent`'s `patch`. Each item needs
`knowledge_set` (the base's id) and `usage_type`; `prompt_description` is
optional and only used in tool mode. The array is replaced whole on update:
take the current `knowledge` from `relevance_get_agent` and send every base the
agent should keep, plus the new one.

### Prompt mode or tool mode

- **`"instructions"` (prompt mode):** the base's rows are pasted into the system
  prompt on every run, wherever a `{{_knowledge.<knowledge_set>}}` variable
  sits. **Without that variable the agent never sees the base.** Use it for a
  few short documents or a small reference table the agent should always
  follow, roughly under 10 KB of text. Every run pays for all of it, and only
  the first 1,000 rows and 80,000 characters (about 20k tokens) are sent; the
  rest is cut off without warning.
- **`"tool"` (tool mode):** the agent gets a search tool,
  `agent_knowledge_<knowledge_set>`, that reads the base when the agent calls
  it. Use it for anything bigger, or knowledge the agent needs only sometimes.
  It needs no variable.

An agent whose instructions read a table needs the base attached in one of
these modes, or a tool with a Get knowledge (`retrieve_data`) step. A tool that
only writes rows can't read them. In a bulk run the row arrives in the task's
message, so the agent needs neither.

### Short documents an agent should always follow

1. **Rows first.** Create the base and add one row per document, with the full
   text rather than a summary:
   `{ text: "<document text>", source_file_name: "writing_guidelines.docx" }`.
   A row holds at most 64 KB. A variable for a base with no rows puts the word
   `undefined` in the prompt.
2. **Attach and reference.** Send one `relevance_update_agent` whose patch
   carries the full `knowledge` array and a `system_prompt` with the variable
   under a line that says what it is. For a long prompt, attach first and then
   add the variable with `relevance_edit_agent_system_prompt`, one after the
   other. A variable listed in `knowledge_variables_appended` was added to the
   end of the prompt for you; move it if it reads better elsewhere.
3. **One write to the agent at a time.** Every edit saves the whole agent, so
   two edits to it sent together keep only the later one. Wait for each result
   before the next edit, and keep every `{{_knowledge.<knowledge_set>}}`
   variable when you rewrite the prompt.
4. **Verify.** `relevance_get_agent` lists the base under `knowledge`. A base
   named in `knowledge_not_in_prompt` has no variable, so add
   `{{_knowledge.<knowledge_set>}}` with `relevance_edit_agent_system_prompt`.
   Then run the draft with a question only the document answers.

```
relevance_update_agent({
  agent_id: "<agent_id>",
  patch: {
    knowledge: [
      { knowledge_set: "leads", usage_type: "tool" },   // already attached, kept
      { knowledge_set: "writing_guidelines", usage_type: "instructions" }
    ],
    system_prompt: "You write the morning news briefing.\n\nFollow these writing guidelines:\n{{_knowledge.writing_guidelines}}"
  }
})
```

### Detaching

Taking a prompt-mode base out of `knowledge`, or switching it to `"tool"`, does
not remove its variable, and the variable alone keeps pasting the rows into
every run. Delete the variable from the prompt as well.

## Reference

- [tables.md](tables.md) — filter types, `raw_filters` in knowledge tool steps, response shapes, bulk-update
  patterns, `data.*` field nesting, common pitfalls.
