---
title: Agent MCP Servers
description: Give an agent tools from a remote MCP server (Notion, Linear, Atlassian, a custom server), remove or replace one, and set which of its tools the agent may run on its own, must ask before running, or never gets. Load before adding, removing or changing an agent's MCP servers or their tool approvals.
---

# Agent MCP Servers

An agent gets an MCP server's tools by referencing one of the project's MCP servers. The server, its URL and its credentials belong to the project; the agent only names it and says how each of its tools is approved. Changes go to the agent's draft, like any other edit: publish to make them live.

## Pick a server

1. `relevance_list_project_mcp_servers` lists the project's servers. Prefer one that is already there over asking for a new one.
2. Read each server's auth before adding it:
   - `shared_auth: true`: everyone who runs the agent acts as the account that connected the server. Say so before adding it to an agent other people run.
   - `authentication_type: "none"`: no account is involved.
   - Otherwise each person who runs the agent connects their own account. `connected` says whether you have.
3. `relevance_list_project_mcp_server_tools` lists what a server offers. Names are enough for choosing approvals.

Only a person can connect a new server or their own account, in the Relevance app (the agent's Tools tab, or Connectors). Never ask for a token, API key or header value in chat.

## Add, remove, replace

- `relevance_attach_mcp_server_to_agent` adds a server, optionally with its approvals. `needs_connection: true` means you haven't connected your own account; its tools only appear for people who have, so tell the user.
- `relevance_detach_mcp_server_from_agent` removes one. The server stays in the project.
- To replace a server, remove it and add the other. Approvals don't carry over: set them on the new one.
- The servers on an agent are under `mcp_servers` on `relevance_get_agent`. Use their `mcp_server_config_id`. An older server added by URL has only a `connection_key`, and carries its own credentials: you can set its default approval with that key, but the user removes it or sets per-tool approvals in the app.

## Tool approvals

| Value          | In the app     | Meaning                                      |
| -------------- | -------------- | -------------------------------------------- |
| `never-ask`    | Always allow   | The agent runs the tool on its own (default) |
| `always-ask`   | Ask every time | Each call waits for someone to approve it    |
| `action-error` | Don't allow    | The agent never gets the tool                |

- `default_tool_approval` sets every tool of the server and clears its per-tool overrides.
- `tool_approval_overrides` sets single tools: a list of `{ "tool_name": ..., "approval": ... }`, where an `approval` of `null` puts the tool back on the default. Sent with a default, the overrides apply after it, in order. Setting one needs the server's tool list, so it fails when the tools can't be listed: set the default instead, or ask the user to set it in the app.
- Set approvals when adding the server, or later with `relevance_update_agent_mcp_server_approvals`. The current values are on `mcp_servers` (an absent value means `never-ask`).
- Match what the tool does, not the server: reads usually `never-ask`, anything that sends, writes or deletes `always-ask`, and tools the agent has no business with `action-error`.

Who hears about a call waiting for approval is set by the agent's notification rules: see [notification-rules.md](notification-rules.md).
