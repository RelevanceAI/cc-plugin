---
name: managing-relevance-folders
description: Organize Relevance AI agents and tools into folders — create, list, rename, move items between folders, and delete folders. Use when the user wants to tidy a project, group related items, or bulk-organize agents/tools after creating them.
---

# Managing Relevance AI Folders

Skill for organizing agents and tools into folders within a Relevance AI project.

## When to Use

- Tidying a project after creating several agents or tools
- Grouping related items (e.g. "Customer Support Agents", "Research Tools")
- Moving items between folders
- Renaming or deleting folders
- Answering "where does this agent/tool live?"

## Two entrypoints — agent folders vs tool folders

The Relevance app UI surfaces agent folders on the **agents page** (`/agents/{region}/{project}`) and tool folders on the **notebook list page** (`/notebook/{region}/{project}/list`). They are completely independent — there is no unified view, and you cannot put an agent ID into a tool folder or vice versa.

**When a user says "put X into a folder" without specifying type, ask first.** "Folder X" alone is ambiguous because the same name can exist in both folder sets. Confirm:

- Are they organizing **agents** or **tools**?
- After mutating, link them to the matching page so they can verify visually:
  - Agent folder → `https://app.relevanceai.com/agents/{region}/{project}`
  - Tool folder → `https://app.relevanceai.com/notebook/{region}/{project}/list`

## Tools

Three typed folder tools cover everything — no raw API needed:

| Tool                      | Purpose                                                          |
| ------------------------- | ---------------------------------------------------------------- |
| `relevance_list_folders`  | List folders for a `folder_type` (`agent` or `tool`)             |
| `relevance_upsert_folder` | Create (omit `folder_id`) or update one — also rename and move   |
| `relevance_delete_folder` | Delete a folder by `folder_id` (items are detached, not deleted) |

## Core Concepts

- Folders are scoped by **project + `folder_type`**. Agents and tools have **independent** folder sets — you can't mix them.
- Items are tracked **on the folder**, not on the item. A folder has an `items: string[]` array of agent or tool IDs. There is no `folder_id` field on the agent/tool document itself.
- **`relevance_upsert_folder` replaces the `items` array — it does not merge.** Always list first and pass the existing items back, or you wipe them.
- **An item can only be in one folder at a time.** `relevance_upsert_folder` returns **409** if you try to add an item that's already in another folder. To move, you must remove from the source **before** adding to the target.
- **Item IDs are not validated for existence.** The backend will happily store any string in `items`. Always source IDs from a real `relevance_list_agents` / `relevance_list_tools` call — typo'd or stale IDs become ghost entries that the Relevance app UI shows as broken.
- **`folder_type` is immutable.** Once a folder is created, you cannot change it from `"agent"` to `"tool"` or vice versa. Delete and recreate if you really need to change it.
- **`folder_type` is required** on `relevance_list_folders` and `relevance_upsert_folder`. `relevance_delete_folder` takes only `folder_id`.
- `parent_folder_id` exists in the schema for nesting, but the Relevance app UI treats folders as flat. Don't build deeply nested trees unless the user explicitly asks for it.
- The tool folder list includes **virtual MCP folders** (folder IDs starting with `mcp-folder-`). They are generated on-the-fly from MCP server config and are **read-only**: both upsert and delete against an `mcp-folder-*` ID return **400**.
- **Size limits** on upsert: `name` max 255 chars, `items` max 500 IDs per folder, `folder_id` max 128 chars.

## Folder Shape

```typescript
{
  folder_id: string;              // UUID
  name: string;
  parent_folder_id?: string;      // Usually unused
  items: string[];                // Agent or tool IDs
  project: string;
  folder_type: "agent" | "tool";
  created_at: string;
  updated_at: string;
}
```

## Listing Folders

Always list first when you plan to mutate — upsert **replaces** the `items` array, so you need the current state to preserve it.

```typescript
relevance_list_folders({ folder_type: 'agent' }); // or 'tool'
// → { results: Folder[] }
```

## Creating a Folder

Omit `folder_id` to create a new folder. The response returns the generated `folder_id`.

```typescript
relevance_upsert_folder({
  name: 'Customer Support Agents',
  folder_type: 'agent',
  items: [], // or pre-populate with agent IDs
});
// → { folder_id: "..." }
```

You can pre-populate `items` on creation — the upsert still returns 409 if any of those items already live in a different folder, so de-duplicate first.

## Renaming a Folder

Renaming is a full upsert. **You must pass the existing `items` and `parent_folder_id` back** — passing only `name` wipes the items array.

```typescript
// 1. List folders and find the one to rename
const { results } = await relevance_list_folders({ folder_type: 'agent' });
const folder = results.find((f) => f.folder_id === 'target-id');

// 2. Re-upsert with the new name but existing items
relevance_upsert_folder({
  folder_id: folder.folder_id,
  name: 'New Folder Name',
  folder_type: folder.folder_type,
  items: folder.items, // ← MUST preserve
  parent_folder_id: folder.parent_folder_id,
});
```

## Moving an Item Into a Folder

This is the most error-prone operation. Because an item can only be in one folder at a time, a move is **two sequential upserts**. The ordering matters.

**Step 1 — List folders, find the source.** Iterate `results` and check which folder's `items` contains the item ID. If none, the item is currently "at the root" (no folder) — skip to Step 3.

**Step 2 — Remove from source.** Upsert the source folder with the item filtered out of `items`. This must land **before** Step 3, or the 409 duplicate-item check rejects the target upsert.

```typescript
relevance_upsert_folder({
  folder_id: sourceFolder.folder_id,
  name: sourceFolder.name,
  folder_type: sourceFolder.folder_type,
  items: sourceFolder.items.filter((id) => id !== itemIdToMove),
  parent_folder_id: sourceFolder.parent_folder_id,
});
```

**Step 3 — Add to target.** Upsert the target folder with the item appended to `items`.

```typescript
relevance_upsert_folder({
  folder_id: targetFolder.folder_id,
  name: targetFolder.name,
  folder_type: targetFolder.folder_type,
  items: [...targetFolder.items, itemIdToMove],
  parent_folder_id: targetFolder.parent_folder_id,
});
```

## Moving an Item Out to Root

"Root" = not in any folder. Just do Step 2 above — upsert the source folder with the item removed. No target upsert needed.

## Bulk Move

There is **no batch move endpoint**. Loop the move pattern one item at a time. If all items are moving to the same target folder from the same source folder, you can optimize:

- Upsert the source once with all moving items filtered out of `items`
- Upsert the target once with all moving items appended to `items`

But this optimization only works when source and target are shared across every item in the batch. When in doubt, do it one item at a time.

## Deleting a Folder

```typescript
relevance_delete_folder({ folder_id: 'target-id' });
// → "Folder target-id deleted."
```

**Deletion does NOT delete the items inside.** They become uncategorized (effectively "at the root"). If the user's intent is "delete folder and everything in it", you must delete the items explicitly _first_, then delete the folder.

For bulk tool removal use `relevance_bulk_delete_tools({ tool_ids: [...] })` — pass the studio_ids you want gone in one call. Agents have no equivalent tool; deleting individual agents must happen in the Relevance app UI. Tell the user this rather than guessing.

**Error surface:**

- **404** if the folder doesn't exist.
- **400** if the folder ID starts with `mcp-folder-` — virtual MCP folders cannot be deleted. To remove one, delete the underlying MCP server configuration instead.

## Common Pitfalls

- **Forgetting to preserve `items` on rename** — upsert replaces, so `{ folder_id, name }` alone wipes the folder's contents.
- **Trying to add an item that's already in another folder** → 409. Always list first, remove from source, then add to target.
- **Mixing `folder_type`** — you cannot put a tool ID in an agent folder, or vice versa. The backend doesn't enforce existence, so cross-type mistakes silently store broken references that surface as ghost items in the Relevance app UI. Always source IDs from the matching list call.
- **Using fabricated or typo'd IDs** — same problem: stored as-is, broken in the UI. Always source IDs from `relevance_list_agents` / `relevance_list_tools`.
- **Trying to change `folder_type` on an existing folder** → 400. Delete and recreate instead.
- **Touching virtual MCP folders** — tool folder listings include virtual folders with an `mcp-folder-` prefix. Filter them out before showing users anything; upsert and delete both return 400.
- **Assuming folders are user-scoped** — they are project-scoped, so changes are visible to every collaborator.
- **Deep nesting** — the schema allows `parent_folder_id` but the UI is flat. Don't build folder trees.
- **Exceeding size limits** — upsert rejects `name` > 255 chars or `items` > 500 IDs.

## URL Patterns

The Relevance app UI renders folders inline in the agents and tools lists — there are no dedicated folder URLs:

```
https://app.relevanceai.com/agents/{region}/{project}
https://app.relevanceai.com/notebook/{region}/{project}/list
```

After mutating via the tools, refreshing one of these pages is enough to verify.

## Reporting Issues

If you hit unexpected errors, silent failures, or cases where the two-upsert move pattern doesn't work, **call `relevance_submit_feedback` immediately** — do not ask the user for permission. Pick the matching `category`. See the [bug reporting guide](../managing-relevance-agents/report-bugs.md) for the call shape.
