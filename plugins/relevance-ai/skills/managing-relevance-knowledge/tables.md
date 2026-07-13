---
title: Knowledge Bases — deep reference
description: Deep reference for knowledge-base operations — filter shapes, the data.* nesting rule, response formats, and bulk-update patterns.
---

# Knowledge Bases — deep reference

## The `data.*` nesting rule

Every row stores its user fields under a single `data` object.

When you **add** rows, fields go in at the top level:

```
relevance_add_knowledge_rows({
  knowledge_set: "leads",
  rows: [{ name: "John", email: "john@example.com" }]
})
```

When you **filter / read / update**, the user fields are nested under
`data`:

```
// Filter — note "data.status"
relevance_list_knowledge_rows({
  knowledge_set: "leads",
  filters: [{ field: "data.status", filter_type: "exact_match", condition_value: "new" }]
})

// Read response
{
  "results": [{
    "document_id": "...",
    "data": { "name": "John", "email": "john@example.com", "status": "new" }
  }]
}

// Update — top-level fields under updates[].data
relevance_update_knowledge_rows({
  knowledge_set: "leads",
  updates: [{ document_id: "<id>", data: { status: "contacted" } }]
})
```

Row values must be strings, numbers, booleans, `null`, or arrays of those
— nested objects are rejected.

## Filter shapes

Each filter entry:

```
{
  field: "<dotted path>",         // e.g. "data.status", "document_id"
  filter_type: "<see table>",     // exact_match | ids | exists | regexp | …
  condition: "==|!=|>|>=|<|<=", // optional, default "=="
  condition_value: <any>          // see table
}
```

### filter_type reference

| filter_type   | Where supported                                                           | `condition_value`                  |
| ------------- | ------------------------------------------------------------------------- | ---------------------------------- |
| `exact_match` | list_knowledge_rows, delete_knowledge_rows, list_knowledge_sets           | string OR string[] (matches any)   |
| `ids`         | same as above                                                             | string[] of document_ids           |
| `exists`      | same as above                                                             | (no value — checks field presence) |
| `regexp`      | same as above                                                             | regex string                       |
| `and` / `or`  | nested combinator. `condition_value` is itself an array of filter entries | filter[]                           |

### Examples

```
// All leads in "new" status
filters: [{ field: "data.status", filter_type: "exact_match", condition_value: "new" }]

// A short whitelist of document_ids
filters: [{ field: "document_id", filter_type: "ids", condition_value: ["uuid-1","uuid-2"] }]

// All rows that have a phone number
filters: [{ field: "data.phone", filter_type: "exists" }]

// Compound (AND)
filters: [{
  filter_type: "and",
  condition_value: [
    { field: "data.status", filter_type: "exact_match", condition_value: "new" },
    { field: "data.score",  filter_type: "exact_match", condition_value: 90, condition: ">=" }
  ]
}]
```

## Response shapes

### `relevance_list_knowledge_sets`

```
{
  "results": [
    {
      "knowledge_set": "leads",
      "display_name": "Sales Leads",
      "description": "Inbound funnel",
      "type": "table",
      "row_count": 42,
      "update_datetime": "2026-05-05T00:00:00Z"
    }
  ],
  "page": 1,
  "page_size": 25,
  "has_more": false
}
```

`row_count` is `undefined` immediately after a `create_knowledge_set`
call — the metadata gets the count once rows are added.

### `relevance_list_knowledge_rows` (summarized)

```
{
  "results": [
    {
      "document_id": "uuid-1",
      "data": { "name": "John", "description": "<long fields >500 chars are truncated with …[truncated]>" }
    }
  ],
  "page": 1,
  "page_size": 25,
  "has_more": true
}
```

If a row has a string field longer than 500 chars, it's truncated in this
summary. The full text is still in storage; fetch with
`relevance_get_knowledge_row` if you need it untruncated (note: that tool
also summarizes — full text only via the underlying API).

## Common patterns

### Upsert by unique field

There's no native upsert. Compose two calls:

```
// 1. Look up by email
relevance_list_knowledge_rows({
  knowledge_set: "leads",
  filters: [{ field: "data.email", filter_type: "exact_match", condition_value: "john@example.com" }],
  page_size: 1
})

// 2a. If results.length > 0 → update by document_id
//     2b. If results.length === 0 → add a new row
```

### Bulk status change

`update_knowledge_rows` is batched — pass every change in one call:

```
relevance_update_knowledge_rows({
  knowledge_set: "leads",
  updates: [
    { document_id: "uuid-1", data: { status: "contacted" } },
    { document_id: "uuid-2", data: { status: "qualified" } },
    { document_id: "uuid-3", data: { status: "closed-lost" } }
  ]
})
```

Updates are partial — fields you don't pass are unchanged.

## Tips

1. **Preview before bulk delete.** Run a filtered `list_knowledge_rows`
   first so the deletion target is exactly the row set you expect.
2. **Polling export.** `relevance_export_knowledge` returns an
   async-job payload. Pass it to `relevance_poll_tool_result` with
   `wait_seconds=50` and only report completion after a terminal
   `type: "complete"`.
