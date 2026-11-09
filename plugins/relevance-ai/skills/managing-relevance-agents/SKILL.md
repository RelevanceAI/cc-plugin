---
name: managing-relevance-agents
description: Manages Relevance AI agents - creating, configuring tools, writing system prompts, setting up triggers and notification rules, running tasks, and running an agent over many items one task each. Use when building agents, attaching tools, working through a list (leads, rows, a CSV, a table), telling people when an agent's tasks need approval or fail, debugging execution, or viewing agent results.
---

# Managing Relevance AI Agents

Skill for creating, configuring, running, and debugging Relevance AI agents.

> **📚 Full API Documentation:** If the Relevance tools don't cover your use case, see `https://api-{region}.stack.tryrelevance.com/latest/documentation` (replace `{region}` with your project's region)

## When to Use

- Creating new agents
- Attaching tools/actions to agents
- Writing or updating system prompts
- Configuring agent memory (persistence across tasks)
- Setting up triggers (email, LinkedIn, webhooks, schedules, and tool triggers that start one task per item a tool returns)
- Running an agent over many items, one task each: a list, a CSV, a knowledge table, or new records from a source ([many-items.md](many-items.md))
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
| `relevance_get_agent_version`          | Get a specific historical version's config, or with `compare_to` only what changed from another version                                                                                                              |
| `relevance_restore_agent_version`      | Duplicate a previous version into the current draft slot                                                                                                                                                             |
| `relevance_trigger_agent`              | Send message to agent (returns conversation_id; pair with `relevance_poll_agent_result`). Defaults to draft when one exists.                                                                                         |
| `relevance_poll_agent_result`          | Long-poll for agent run status and response (pair with `relevance_trigger_agent`)                                                                                                                                    |
| `relevance_get_agent_tools`            | Get agent's tools with action_ids. Defaults to the draft when one exists; response includes `viewed_version`.                                                                                                        |
| `relevance_get_agent_task_summary`     | Get task summary: status, messages, tool calls                                                                                                                                                                       |
| `relevance_list_agent_tasks`           | List recent tasks for an agent                                                                                                                                                                                       |
| `relevance_list_agent_task_messages`   | List raw messages in an agent task                                                                                                                                                                                   |
| `relevance_get_agent_task_metadata`    | Get task metadata: status, credits, runtime                                                                                                                                                                          |
| `relevance_list_agent_triggers`        | List agent triggers — the source of every `document_id`                                                                                                                                                              |
| `relevance_create_trigger`             | Add a trigger (omit `document_id`) or edit one (pass it). An agent can hold many triggers, including several recurring schedules                                                                                     |
| `relevance_update_trigger_status`      | Pause (`paused`) or resume (`in_progress`) a trigger — use instead of delete-and-recreate                                                                                                                            |
| `relevance_delete_trigger`             | Delete trigger                                                                                                                                                                                                       |
| `relevance_list_oauth_accounts`        | List OAuth accounts for triggers                                                                                                                                                                                     |
| `relevance_submit_feedback`            | Report a bug or suggest an improvement, without asking permission                                                                                                                                                    |

## Resource URLs

These tools return a ready-made link. **Share it in the reply that reports the result** — it already carries the correct region and project, so present it as-is.

| Tool                          | Opens                                                          |
| ----------------------------- | -------------------------------------------------------------- |
| `relevance_create_agent`      | The agent's builder (shows the draft while one is unpublished) |
| `relevance_update_agent`      | The agent's builder                                            |
| `relevance_publish_agent`     | The agent's builder                                            |
| `relevance_trigger_agent`     | **That run's output**                                          |
| `relevance_poll_agent_result` | **That run's output**                                          |

The two kinds are not interchangeable: when the user asks to see a test result or what the agent did, give the run's link. A builder link is configuration, and offering it as the run's output sends the user to a page with no result on it.

Where a response names the link — `task_url` for a run, `edit_url` for the builder — use the name rather than guessing. A response may instead carry a single `url`; the table above tells you what it opens.

## Critical Rules

### Draft-First Editing — Publish Is Always Explicit

Edits save to a draft until you publish. **Every edit saves to a draft. Publishing is always a separate, user-confirmed step.**

- **Creating a new agent** (`relevance_create_agent`): saves to a DRAFT. The new agent has a `draft_version_id` and no `active_version_id` until you call `relevance_publish_agent`. `relevance_trigger_agent` defaults to draft, so the agent is testable as soon as creation returns.
- **Changing the tool surface breaks existing evals' coverage.** A tool attached after the scenarios were written has no simulation entry, so it runs for real on the next eval. If the agent has test sets, extend the test-set config to cover it before publishing — see the `relevance-evals` skill.
- **Updating an existing agent** (`relevance_update_agent` or `relevance_attach_tools_to_agent`): saves to a DRAFT. The previously-published "live" version remains untouched until you explicitly publish. `relevance_update_agent` deep-merges your patch into the current draft, so you only need to send the fields you want to change. Arrays such as `knowledge` are replaced whole, and two edits to the same agent sent together keep only the later one, so make them one after another.
- **Going live**: call `relevance_publish_agent`. Publishing is irreversible — do not call it without first asking the user "want me to publish this?" in chat. Always pass a concise `version_description` (and `version_name`) — a clear, one-line summary of what changed and why, easy to understand, so version history is a useful trail. Reuse the summary you gave the user; never leave it blank. If the agent has a pre-publish check config (see the `relevance-evals` skill), publishing runs those checks against the draft and auto-publishes on pass — poll the returned `check_id` with `relevance_poll_pre_publish_check`.
- **Testing the draft**: `relevance_trigger_agent` defaults to the draft when one exists, so you can run the in-progress version. The draft's tools still run for real unless you pass `simulate: true` (see [running.md](running.md#trying-an-agent-whose-tools-write)). The response includes `triggered_version` so you can tell the user which version actually ran.
- **Restoring a previous version**: `relevance_list_agent_versions` to find the version_id, `relevance_get_agent_version` with `compare_to: "active"` to see what it would change, then `relevance_restore_agent_version` to bring it back as a fresh draft (followed by `relevance_publish_agent` to make it live).

#### Reading an agent while a draft exists

`relevance_get_agent` and `relevance_get_agent_tools` default to the unpublished draft when one exists — so the LLM and the user are looking at the same thing. The response's `viewed_version` field tells you what you read (`"draft"`, `"active"`, or a specific `version_id`), alongside `active_version_id`, `draft_version_id`, and `has_unpublished_draft`.

**When `viewed_version` is `"draft"`, say so to the user explicitly** — e.g. "I'm looking at the unpublished draft — there are unsaved changes here that aren't live yet." This matters because anything you propose is rooted in the draft, not the live config; the user needs to know the baseline before approving edits or asking you to publish.

If the user wants to see what's currently live, re-fetch with `version: "active"`. To compare draft vs live, call `relevance_get_agent_version` with the draft's `draft_version_id` and `compare_to: "active"`: it returns only the changed fields and a line diff of the system prompt, so don't load both versions to diff them yourself. If `system_prompt_diff_truncated` is set, read the rest of each version's prompt with `relevance_read_agent_system_prompt`. Do not silently mix draft and live config in the same response — pick one and label it.

### Agent Variables Are Part of the Prompt

`{{name}}` in an agent's prompt or tool inputs is replaced with that agent variable's saved value on every run, and builders often keep the policies, playbooks and reference text the agent depends on there. Before you explain, diagnose or change an agent, read every variable its config uses (`variables` in `relevance_get_agent`), and keep every reference when you rewrite its prompt. See [variables.md](variables.md).

### Work on Many Items: One Task per Item

When the job is the same work done to each of many items (every new lead, every ticket, every row of a sheet, every prospect on a list), build the agent for **one** item and let the platform start one task per item: an event trigger when the source sends each item, a tool trigger when a tool has to fetch them, a bulk run for a list the user already has (a CSV or a knowledge table), started with the bulk-run tool or Run in bulk in the app. Never have one task work through the list, and never run an agent once per item yourself, except for a handful of items (five or fewer) the user wants done now, once a try shows the agent works. Routes, setup and what not to build: [many-items.md](many-items.md).

#### A Tool That Lists the Work Starts the Agent

When the job is "for each new lead / ticket / row, do X" on a schedule, the tool that fetches those records belongs in a **tool trigger** ("Tool as trigger" in the app), not in the agent's tools; when the source can push each record as it happens, an event trigger is better still (see the [decision guide](triggers.md#decision-guide)). The trigger runs the tool on a schedule and starts one task per record, and the agent handles the one record its task carries. Attached to the agent instead, the tool returns every record into one task: the agent loops over all of them, a failure part-way through stops the rest, and only a tool trigger's `dedupe_field` remembers which records were already handled.

Users rarely ask for this by name. "Every hour, go through the new leads", "check the sheet for new rows" and "work through the open tickets" all mean it. Choose it yourself, and in the message that proposes it, tell them why in their words: each lead gets its own run, one bad lead doesn't hold up the others, and (with a de-duplication field set) a lead that was already handled isn't picked up again. If an agent you're editing already runs such a tool itself, say so and offer to convert it. In a workforce, the trigger is a `tool_trigger` node on the workforce ([workforce-triggers.md](../managing-relevance-workforces/workforce-triggers.md)). Setup and conversion: [triggers.md](triggers.md#tool-trigger).

**Keep the tool on the agent** when one run needs the whole list (a digest, a count, a comparison), when someone asks about one record on demand (a chat agent that looks a ticket up), or when the tool needs inputs that change every run. A list to work through once ("these 100 rows") is not one of these: that is Run in bulk ([many-items.md](many-items.md#run-in-bulk-a-list-you-already-have)).

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
  autonomy_limit: 50,                        // Max tool calls
  autonomy_limit_behaviour: "ask-for-approval" // Valid: "ask-for-approval" | "terminate-conversation"
}
```

**Note:** `"stop"` is NOT valid for `autonomy_limit_behaviour`. `"ask-for-approval"` pauses and asks to continue; `"terminate-conversation"` can end the task without a reply (see [creating.md](creating.md)).

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
- [mcp-servers.md](mcp-servers.md) - MCP servers on an agent: adding, removing and replacing one, and approving its tools
- [system-prompts.md](system-prompts.md) - Writing effective prompts
- [variables.md](variables.md) - Agent variables: the values `{{name}}` in the prompt and tool inputs stand for; reading, adding, renaming and removing them
- [memory.md](memory.md) - Agent memory configuration and usage
- [triggers.md](triggers.md) - Email, LinkedIn, webhook and recurring/scheduled triggers; tool triggers ("Tool as trigger": a tool on a schedule, one task per item it returns); personal failure alarms; writing the message a schedule sends
- [many-items.md](many-items.md) - Work over many items: build for one item, then fan out (event trigger, tool trigger, Run in bulk over a CSV or knowledge table, a tool column); map then reduce; what not to build (queues, logs of handled items, per-item orchestrators)
- [notification-rules.md](notification-rules.md) - Notification rules (the app's alarm rules): an email, Slack or Teams message when a task waits for approval, is escalated, fails or finishes (`notifications_config`)
- [running.md](running.md) - Running and testing agents
- [troubleshooting.md](troubleshooting.md) - Common issues and fixes

For organizing agents into folders, see the [`managing-relevance-folders`](../managing-relevance-folders/SKILL.md) skill.

## App URLs

Send back a url a tool returned; never assemble one from a region, project, and id. A hand-built link resolves to a page but shows the wrong thing — a task id that doesn't belong to the agent renders an empty task rather than an error, so the mistake is invisible to you and confusing to the user.

If you are ever checking a url, every app URL is **section-first** — `{app}/{section}/{region}/{project}/…`; the region and project never come before the section.

See [Sharing links](running.md#sharing-links) for which link answers which request.

## Reporting Issues

When a failure is the platform's (not the input, not your own config) or a capability is missing while managing agents, **call `relevance_submit_feedback` without asking the user for permission**. One tool covers bugs, missing capabilities, hallucinations, UX friction, feature requests, and docs gaps; pick the matching `category`. Region/project/source are inferred from the session. See [report-bugs.md](report-bugs.md) for the call shape.
