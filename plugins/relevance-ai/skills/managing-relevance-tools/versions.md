---
title: Version Management
description: Recover tools from accidental edits using version snapshots, and apply targeted per-step edits without overwriting the rest. Load when a tool's config is wiped or you need to safely edit one step in isolation.
---

# Version Management

How to recover tools from accidental changes using version history.

## Overview

Every time you publish a tool, a version snapshot is saved. If you accidentally wipe a tool's config, you can restore from a previous version.

## Available MCP Tools

| Tool                             | Description                                                |
| -------------------------------- | ---------------------------------------------------------- |
| `relevance_list_tool_versions`   | List version history for a tool                            |
| `relevance_get_tool_version`     | Get a specific version's full config                       |
| `relevance_restore_tool_version` | Restore a tool to a previous version (creates a new draft) |
| `relevance_publish_tool`         | Publish the restored draft                                 |

## Emergency Recovery Workflow

### 1. List Versions

```
relevance_list_tool_versions({
  tool_id: "my-tool-id"
})
```

Returns version snapshots with `version_id`, `created_at`, and `config` (including `transformations`).

### 2. Find Working Version

Look at the `config` field in each version to find one with your transformations intact.

To inspect a specific version in detail:

```
relevance_get_tool_version({
  version_id: "version-uuid"
})
```

### 3. Restore Version

```
relevance_restore_tool_version({
  version_id: "working-version-uuid"
})
```

This creates a new draft from the specified version.

### 4. Publish Restored Draft

```
relevance_publish_tool({
  tool_id: "my-tool-id"
})
```

## Editing Steps Without Wiping the Rest

For step edits, use the dedicated per-step tools, all of which save to DRAFT only:

- `relevance_add_tool_step` — insert at a 0-based `position` (default = append; pass `position >= step_count` to append explicitly).
- `relevance_update_tool_step` — shallow-merge a patch into one step by 0-based `step_index`.
- `relevance_remove_tool_step` — remove one step by 0-based `step_index`.
- `relevance_move_tool_step` — reorder by `from_index` → `to_index` (both 0-based; mirrors UI drag).

Always call `relevance_get_tool` first to confirm current step indices. The same principle holds for the agent side: never pass `actions` to `relevance_update_agent` — use `relevance_attach_tools_to_agent` / `relevance_update_agent_action` / `relevance_detach_tools_from_agent`.

## Version Retention

- Versions are kept for recent publishes
- Old versions may be pruned over time

## API Reference (for `relevance_api_request` fallback)

| Operation       | Method | Endpoint                                             |
| --------------- | ------ | ---------------------------------------------------- |
| List versions   | POST   | `/tools/versions/list` with `{ tool_id, page_size }` |
| Get version     | GET    | `/tools/versions/{version_id}/get`                   |
| Restore version | POST   | `/tools/versions/restore` with `{ version_id }`      |
| Publish         | POST   | `/tools/versions/publish` with `{ tool_id }`         |
