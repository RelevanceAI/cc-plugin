---
name: managing-relevance-agents
description: Manages Relevance AI agents - creating, configuring tools, writing system prompts, setting up triggers, and running tasks. Use when building agents, attaching tools, debugging execution, or viewing agent results.
---

# Managing Relevance AI Agents

Skill for creating, configuring, running, and debugging Relevance AI agents.

> **📚 Full API Documentation:** If the Relevance tools don't cover your use case, see `https://api-{region}.stack.tryrelevance.com/latest/documentation` (replace `{region}` with your project's region)

## When to Use

- Creating new agents
- Attaching tools/actions to agents
- Writing or updating system prompts
- Configuring agent memory (persistence across tasks)
- Setting up triggers (email, LinkedIn, webhooks, etc.)
- Running agent tasks
- Debugging agent execution issues

## Tools

| Tool                                   | Description                                                                                                                                                                                                          |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `relevance_list_agents`                | List all agents (supports search)                                                                                                                                                                                    |
| `relevance_get_agent`                  | Get agent config. Defaults to the unpublished draft when one exists; pass `version: "active"` to inspect live. Response includes `viewed_version`, `active_version_id`, `draft_version_id`, `has_unpublished_draft`. |
| `relevance_read_agent_system_prompt`   | Read a large `system_prompt` in a line-numbered window (`offset_line`/`limit_lines`, `limit_lines: 0` for size only) without loading the whole prompt                                                                |
| `relevance_search_agent_system_prompt` | Grep a `system_prompt` (`pattern` is a JS regex); returns matching line numbers + context                                                                                                                            |
| `relevance_edit_agent_system_prompt`   | Edit a `system_prompt` by verbatim, unique string replacement (saves to DRAFT) — no need to resend the whole prompt                                                                                                  |
| `relevance_create_agent`               | Create a NEW agent — saves to DRAFT only; call `relevance_publish_agent` to make it live                                                                                                                             |
| `relevance_update_agent`               | Update an existing agent — partial-merge, saves to DRAFT only                                                                                                                                                        |
| `relevance_attach_tools_to_agent`      | Attach tools to an agent — saves to DRAFT only                                                                                                                                                                       |
| `relevance_update_agent_action`        | Update per-tool config (action_behaviour, prompt_description, default_values, overrides, etc.) on a single attached tool — saves to DRAFT only                                                                       |
| `relevance_detach_tools_from_agent`    | Remove one or more attached tools from an agent — saves to DRAFT only                                                                                                                                                |
| `relevance_publish_agent`              | Publish the current draft to live. Always shows an approval card.                                                                                                                                                    |
| `relevance_list_agent_versions`        | List version history for an agent                                                                                                                                                                                    |
| `relevance_get_agent_version`          | Get a specific historical version's config                                                                                                                                                                           |
| `relevance_restore_agent_version`      | Duplicate a previous version into the current draft slot                                                                                                                                                             |
| `relevance_trigger_agent`              | Send message to agent (returns conversation_id; pair with `relevance_poll_agent_result`). Defaults to draft when one exists.                                                                                         |
| `relevance_poll_agent_result`          | Long-poll for agent run status and response (pair with `relevance_trigger_agent`)                                                                                                                                    |
| `relevance_get_agent_tools`            | Get agent's tools with action_ids. Defaults to the draft when one exists; response includes `viewed_version`.                                                                                                        |
| `relevance_get_agent_task_summary`     | Get task summary: status, messages, tool calls                                                                                                                                                                       |
| `relevance_list_agent_tasks`           | List recent tasks for an agent                                                                                                                                                                                       |
| `relevance_list_agent_task_messages`   | List raw messages in an agent task                                                                                                                                                                                   |
| `relevance_get_agent_task_metadata`    | Get task metadata: status, credits, runtime                                                                                                                                                                          |
| `relevance_list_agent_triggers`        | List agent triggers                                                                                                                                                                                                  |
| `relevance_create_trigger`             | Create trigger                                                                                                                                                                                                       |
| `relevance_delete_trigger`             | Delete trigger                                                                                                                                                                                                       |
| `relevance_list_oauth_accounts`        | List OAuth accounts for triggers                                                                                                                                                                                     |
| `relevance_submit_feedback`            | Report a bug or suggest an improvement — call proactively after errors or friction                                                                                                                                   |

## Resource URLs

The following tools return a `url` field in their response pointing directly to the correct page in the Relevance AI app. **Always share this URL with the user immediately after the operation completes.**

| Tool                      | URL points to                |
| ------------------------- | ---------------------------- |
| `relevance_create_agent`  | Agent instructions edit page |
| `relevance_update_agent`  | Agent instructions edit page |
| `relevance_publish_agent` | Agent instructions edit page |
| `relevance_trigger_agent` | Agent task page              |

The URL is already constructed with the correct region and project ID — just present it to the user.

## Critical Rules

### Draft-First Editing — Publish Is Always Explicit

Edits save to a draft until you publish. **Every edit saves to a draft. Publishing is always a separate, user-confirmed step.**

- **Creating a new agent** (`relevance_create_agent`): saves to a DRAFT. The new agent has a `draft_version_id` and no `active_version_id` until you call `relevance_publish_agent`. `relevance_trigger_agent` defaults to draft, so the agent is testable as soon as creation returns.
- **Updating an existing agent** (`relevance_update_agent` or `relevance_attach_tools_to_agent`): saves to a DRAFT. The previously-published "live" version remains untouched until you explicitly publish. `relevance_update_agent` deep-merges your patch into the current draft, so you only need to send the fields you want to change.
- **Going live**: call `relevance_publish_agent`. This always shows an approval card to the user — even when auto-approve is on. Do not call publish without first asking the user "want me to publish this?" in chat. Always pass a concise `version_description` (and `version_name`) — a clear, one-line summary of what changed and why, easy to understand, so version history is a useful trail. Reuse the summary you gave the user; never leave it blank.
- **Testing the draft**: `relevance_trigger_agent` (and `_sync`) defaults to the draft when one exists, so you can run the in-progress version without affecting production. The response includes `triggered_version` so you can tell the user which version actually ran.
- **Restoring a previous version**: `relevance_list_agent_versions` to find the version_id, `relevance_get_agent_version` to inspect it, then `relevance_restore_agent_version` to bring it back as a fresh draft (followed by `relevance_publish_agent` to make it live).

#### Reading an agent while a draft exists

`relevance_get_agent` and `relevance_get_agent_tools` default to the unpublished draft when one exists — so the LLM and the user are looking at the same thing. The response's `viewed_version` field tells you what you read (`"draft"`, `"active"`, or a specific `version_id`), alongside `active_version_id`, `draft_version_id`, and `has_unpublished_draft`.

**When `viewed_version` is `"draft"`, say so to the user explicitly** — e.g. "I'm looking at the unpublished draft — there are unsaved changes here that aren't live yet." This matters because anything you propose is rooted in the draft, not the live config; the user needs to know the baseline before approving edits or asking you to publish.

If the user wants to see what's currently live or compare draft vs live, re-fetch with `version: "active"` and narrate the diff. Do not silently mix draft and live config in the same response — pick one and label it.

### Sub-Agents are DEPRECATED

> ⚠️ **Do NOT add agents to another agent's `actions` array.** This pattern is deprecated.
>
> Use **Workforces** instead for multi-agent orchestration. See [Workforce Documentation](../managing-relevance-workforces/SKILL.md).

### Actions Require project/region (CRITICAL)

Every action MUST include `project` and `region` fields:

```typescript
{
  chain_id: "tool-id",
  project: config.project,    // Your project ID
  region: "f1db6c",           // Tool's region - MAY DIFFER FROM PROJECT!
  title: "My Tool",
  action_behaviour: "never-ask"
}
```

**CRITICAL:** The `region` must be **where the tool exists**, not your project's region!

- Find correct region by checking working agents or tool metadata
- Wrong region = tools silently fail to attach (empty chains)
- See [actions.md](actions.md) for how to find correct region

### Autonomy Settings

```typescript
{
  autonomy_limit: 50,                              // Max tool calls
  autonomy_limit_behaviour: "terminate-conversation"  // Valid: "ask-for-approval" | "terminate-conversation"
}
```

**Note:** `"stop"` is NOT valid for `autonomy_limit_behaviour` - use `"terminate-conversation"`.

Missing project/region causes cloning failures and tool execution errors.

### Action IDs for System Prompts (Two-Pass Required!)

If the agent has any attached tools, the second pass is **required** — weave `{{_actions.<id>}}` references inline into the system_prompt prose where each tool is used. The runtime substitutes each pill for the real function-call name; the prompt editor renders them as clickable pill chips. Plain prose like "use the Google Search tool" produces no pill and gives the LLM a weak tool-selection cue.

```
Pass 1: Create agent + attach tools + placeholder prompt   (saved to draft)
Pass 2: Fetch real action IDs → Update prompt with pills   (saved to draft)
```

**Why?** Action IDs are deterministic hex strings derived from action config — they don't exist until tools are attached. After the first draft save, call `relevance_get_agent_tools` (with `version: "draft"`) to read the real `action_id` for each attached tool, or use the `action_id_map` returned by `relevance_attach_tools_to_agent`. Then call `relevance_update_agent` with a system_prompt patch that inserts those IDs inline. Both passes save to draft — publish only after the user has reviewed and confirmed.

See [creating.md - Action IDs](creating.md#action-ids-vs-studio-ids) for full workflow with code examples.

## Quick Start: Create Agent

1. **Create basic agent** — saves to a draft (does NOT publish):

   ```
   relevance_create_agent({
     name: "Research Assistant",
     description: "Helps research topics",
     system_prompt: "You are a research assistant.\nWhen asked about a topic:\n1. Search for information\n2. Summarize findings"
   })
   ```

2. **Get the agent** to see the generated ID:

   ```
   relevance_get_agent({ agent_id: "..." })
   ```

3. **Add tools** — saves to a draft (does NOT publish):

   ```
   relevance_attach_tools_to_agent({
     agent_id: "...",
     tool_ids: ["google-search"],
     action_behaviour: "never-ask",
   })
   ```

4. **Test the draft** — trigger uses the draft when one exists:

   ```
   relevance_trigger_agent({ agent_id: "...", message: "Research AI trends" })
   // Response includes triggered_version: "draft"
   ```

5. **Publish when the user confirms** — always asks even with auto-approve:

   ```
   relevance_publish_agent({ agent_id: "..." })
   ```

## Reference Files

- [creating.md](creating.md) - Full creation workflow
- [actions.md](actions.md) - Tool/action configuration
- [phantom-tools.md](phantom-tools.md) - Phantom tools reference (thinking, memory, tags, etc.)
- [system-prompts.md](system-prompts.md) - Writing effective prompts
- [memory.md](memory.md) - Agent memory configuration and usage
- [triggers.md](triggers.md) - Email, LinkedIn, webhook triggers
- [running.md](running.md) - Running and testing agents
- [troubleshooting.md](troubleshooting.md) - Common issues and fixes

For organizing agents into folders, see the [`managing-relevance-folders`](../managing-relevance-folders/SKILL.md) skill.

## URL Patterns

```
# Agent edit page
https://app.relevanceai.com/agents/{region}/{project}/{agentId}/edit/instructions

# Task view
https://app.relevanceai.com/agents/{region}/{project}/{agentId}/{taskId}

# Clone link
https://app.relevanceai.com/form/{region}/{project}/clone/agent/{agentId}
```

## Reporting Issues — proactive

If anything goes wrong or feels missing while managing agents, **call `relevance_submit_feedback` immediately** — do not ask the user for permission first. One tool covers bugs, missing capabilities, hallucinations, UX friction, feature requests, and docs gaps; pick the matching `category`. Region/project/source are inferred from the session. See [report-bugs.md](report-bugs.md) for the call shape.
