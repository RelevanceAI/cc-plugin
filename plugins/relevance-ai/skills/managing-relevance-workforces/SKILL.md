---
name: managing-relevance-workforces
description: Manages Relevance AI workforces (multi-agent systems) - creating workflows, configuring nodes/edges, triggering execution, and debugging runs. Use when building multi-agent pipelines, connecting agents together, or debugging workforce execution.
---

# Managing Relevance AI Workforces

Skill for creating, configuring, triggering, and debugging Relevance AI workforces (multi-agent systems).

> **📚 Full API Documentation:** If the Relevance tools don't cover your use case, see `https://api-{region}.stack.tryrelevance.com/latest/documentation` (replace `{region}` with your project's region)
> **✅ RECOMMENDED APPROACH**: Workforces are the **official way** to build multi-agent systems.
> The legacy pattern of adding sub-agents to an agent's `actions` array is **DEPRECATED**.
> Always use workforces for agent-to-agent orchestration.

## Resource URLs

The following tools return a `url` field in their response pointing directly to the correct page in the Relevance AI app. **Always share this URL with the user immediately after the operation completes.**

| Tool                          | URL points to                       |
| ----------------------------- | ----------------------------------- |
| `relevance_create_workforce`  | Workforce build page                |
| `relevance_update_workforce`  | Workforce build page                |
| `relevance_publish_workforce` | Workforce build page                |
| `relevance_trigger_workforce` | Workforce task (triggered run) page |

The URL is already constructed with the correct region and project ID — just present it to the user.

## When to Use

- Creating multi-agent workflows (pipelines)
- Connecting agents together with edges
- Configuring handovers between agents
- Triggering workforce execution
- Debugging workforce runs (seeing what each agent produced)

## Terminology: Subagents vs Tools

Inside a workforce, an agent node that another agent calls is a **subagent** — NOT a "tool". The word "tool" is reserved for non-agent integrations (API calls, side-effect actions). When writing an orchestrator's system prompt or talking to the user about workforce structure, always say "subagent" for agent-typed nodes the orchestrator routes to. Phrases like "sub-agent tools available" are wrong and produce confusing UI pills.

## Designing Workforces: Agents for Reasoning, Tools for Integrations

When building a multi-agent system, follow this principle:

> **Agents handle reasoning and intelligence. Tools handle external integrations and side effects.**

### Use an agent (in the workforce) when:

- The task requires LLM reasoning (analysis, scoring, writing, classification)
- The output quality depends on prompt engineering and model capability
- You want to test/eval the reasoning independently
- The task is a distinct "role" (researcher, writer, qualifier, reviewer)

### Use a tool (attached to an agent) when:

- The task calls an external API or service
- The output is deterministic or comes from an external source
- The action has side effects (sending email, updating database, posting to Slack)
- The task is a utility function (search, scrape, parse, calculate)

### Anti-Pattern: Tools with LLM Steps for Reasoning

❌ **Don't** create tools that are just `prompt_completion` steps wrapping reasoning tasks. This adds unnecessary indirection and means evals test simulated tool outputs instead of actual agent reasoning.

**Exception:** Tools that combine external API calls with LLM processing are fine (e.g., scrape webpage → extract data with `prompt_completion`). The anti-pattern is tools where the _only_ step is an LLM call.

| ❌ Anti-Pattern                                                    | ✅ Correct Approach                                                   |
| ------------------------------------------------------------------ | --------------------------------------------------------------------- |
| "Score Lead" tool — single `prompt_completion` step                | Lead Qualifier **agent** with scoring logic in system prompt          |
| "Draft Email" tool — single `prompt_completion` step               | Outreach Drafter **agent** with writing instructions in system prompt |
| "Summarize Research" tool — single `prompt_completion` step        | Summarizer **agent** in the workforce pipeline                        |
| ✅ "Enrich Company" tool — API call + `prompt_completion` to parse | This is fine — the tool adds value by calling an external API         |

### Good Example: Sales Pipeline Workforce

```
Trigger → Lead Qualifier Agent → Outreach Drafter Agent
              │                         │
              ├─ Tool: Clearbit API      ├─ Tool: SendGrid API
              └─ Tool: CRM Lookup        └─ Tool: CRM Update
```

- **Lead Qualifier agent**: Reasons about lead quality using its system prompt. Calls Clearbit to enrich company data, CRM to check existing records.
- **Outreach Drafter agent**: Reasons about email tone and personalization. Calls SendGrid to send, CRM to log the outreach.
- **In evals**: Simulate Clearbit, SendGrid, CRM tools. The agents' reasoning is tested directly. Scope simulation and tool-usage checks **per graph node** (`node_configs[node_id]`, and `node_id` on a `tool_usage` check) so each agent node — and its sub-agents — is configured independently; see the `relevance-evals` skill.

## Tools

| Tool                                    | Description                                                                                 |
| --------------------------------------- | ------------------------------------------------------------------------------------------- |
| `relevance_list_workforces`             | List all workforces (summary includes `has_unpublished_draft`).                             |
| `relevance_get_workforce`               | Get workforce config. See Draft-First Workflow for `version`.                               |
| `relevance_create_workforce`            | Create a new workforce. See Draft-First Workflow.                                           |
| `relevance_update_workforce`            | Update a workforce — partial-merge into the current draft. See Draft-First Workflow.        |
| `relevance_publish_workforce`           | Publish the current draft. See Draft-First Workflow.                                        |
| `relevance_list_workforce_versions`     | List version history (paginated). `version_id`s can be passed to `relevance_get_workforce`. |
| `relevance_delete_workforce`            | Delete a workforce.                                                                         |
| `relevance_trigger_workforce`           | Trigger a workforce. See Draft-First Workflow for `version`. Returns `workforce_task_id`.   |
| `relevance_poll_workforce_task_result`  | Long-poll a workforce task to terminal state (default wait 50s, max 300).                   |
| `relevance_get_workforce_task_messages` | Get execution details and agent outputs.                                                    |
| `relevance_list_workforce_tasks`        | List tasks for a workforce with filtering.                                                  |
| `relevance_get_workforce_task_metadata` | Get task metadata: status, credits, creator.                                                |

## Draft-First Workflow

The workforces lifecycle mirrors agents (see [`../managing-relevance-agents/SKILL.md`](../managing-relevance-agents/SKILL.md)). The rules below are the canonical reference — tool descriptions and Quick Start cross-reference this section.

- **Edits save to a DRAFT.** `relevance_create_workforce` and `relevance_update_workforce` write to the draft; the live/active version is untouched until you publish. After any edit, tell the user "Saved as a draft — not yet live."
- **Reads default to the draft.** `relevance_get_workforce` returns the draft when one exists; the response's `viewed_version` says which version you actually got (`"draft"`, `"active"`, or a specific `version_id`). When `viewed_version` is `"draft"`, surface that to the user (e.g. "I'm looking at the unpublished draft"). Pass `version: "active"` to compare against live; don't mix draft and live config in one response.
- **Trigger also defaults to the draft.** `relevance_trigger_workforce` runs the draft when one exists. Response includes `triggered_version` — surface it. Pass `version: "active"` to run live. Individual agents (`relevance_trigger_agent`) and tools (`relevance_run_tool`) still work for isolated testing.
- **Publishing is explicit.** `relevance_publish_workforce` always shows an approval card. Ask "want me to publish?" first. Before publishing, inspect the graph: if any agents have their own unpublished drafts, list them by name and ask the user whether to publish those agent drafts too, or publish only the workforce — don't auto-publish agent drafts that may contain unrelated edits. Always pass a concise `version_description` (and `version_name`) when you publish — a clear, one-line summary of what changed and why, easy to understand, so version history is a useful trail. Reuse the summary you gave the user; never leave it blank.
- **Auto-layout.** `relevance_create_workforce` always positions the new graph automatically. `relevance_update_workforce` accepts an opt-in `tidy_up_graph: boolean` (default `false`) — set `true` only when the user explicitly asks to tidy / auto-layout an existing workforce; otherwise the update only positions nodes that don't already have a position and leaves the user's canvas arrangement intact.
- **Updates merge by `node_id` / `edge_id` — they DO NOT replace.** When you call `relevance_update_workforce` with a `workforce_graph`, the nodes and edges you pass are **upserted into the existing draft** by their IDs. Anything you don't mention is preserved — including tool, condition, and note nodes a human may have added in the builder. This is by design so an LLM that fetches → tweaks one agent → writes back can't accidentally delete graph state it didn't author.
  - To **delete** nodes or edges, pass `remove_node_ids: [...]` or `remove_edge_ids: [...]`. Removed nodes cascade-drop any edges that reference them; the response's `merge_summary` lists what was upserted, removed, and cascaded so you can confirm.
  - If an ID appears in both the incoming graph and a `remove_*` list, **remove wins** and `merge_summary.warnings` flags the conflict.
  - To **replace** a node's config, include the same `node_id` with the new config — the upsert overwrites the existing entry.
  - **Always read notes before editing.** Note nodes (`type: "note"`, `config: { content: "..." }`) carry user-written context. Fetch the graph first, read note `content`, and respect any constraints the user left behind before mutating the graph.

## Quick Start: Create Workforce

1. **Create the workforce** — saves to a draft (does NOT publish). Omitting `edges` produces a linear chain with forced-handover edges between the agents in order. Omitting `triggers` produces a single `manual` trigger (the workforce runs only when `relevance_trigger_workforce` is called):

   ```
   relevance_create_workforce({
     name: "Research Pipeline",
     description: "Research -> Summarize -> Report",
     agents: [
       { agentId: "researcher-agent-id" },
       { agentId: "summarizer-agent-id" },
       { agentId: "reporter-agent-id" },
     ],
   })
   ```

   To run the workforce on an external event or schedule instead, pass `triggers` (see [`workforce-triggers.md`](workforce-triggers.md) for per-vendor configs):

   ```
   relevance_create_workforce({
     name: "Daily Sales Report",
     agents: [{ agentId: "reporter-agent-id" }],
     triggers: [{
       type: "auto",
       sync_data: {
         config: {
           type: "recurring",
           recurring: {
             name: "Daily 9am",
             message: "Generate today's sales report",
             schedule: { frequency: "daily", hour: "09:00", timezone: "UTC" },
           },
         },
       },
     }],
   })
   ```

   Multiple triggers on one workforce are also supported — see [`workforce-triggers.md`](workforce-triggers.md#multiple-trigger-nodes).

   > **OAuth behaviour is per-vendor.** Only a subset of vendors allow saving the trigger without OAuth — primarily `hubspot`, `recurring`, `custom_webhook`, `freshdesk`, `zoominfo`, `relevance_meeting_bot`, and `tool_trigger` (the "save-anyway" set). Every other OAuth vendor (`slack`, `gmail`, `outlook`, `teams`, `google_calendar`, `teams_calendar`, `google_drive`, `jira`, `salesforce`, `unipile_*`) requires `oauth_account_id` at save time — the platform's `SaveWorkforce` rejects the call otherwise, and the tools pre-validate with a clear per-field error. Before adding a connect-first trigger, run `relevance_list_oauth_accounts` and verify the provider is connected; if it isn't, save the workforce without that trigger and tell the user to connect the integration before you wire it up. See [`workforce-triggers.md`](workforce-triggers.md#oauth-handling-per-vendor-save-anyway-vs-connect-first) for the full per-vendor table.

   To add **tool steps** or **condition routing** (e.g. classifier → urgent-handler vs normal-handler), see [`workforce-nodes.md`](workforce-nodes.md) — pass `tools` and `conditions` arrays alongside `agents` and wire them with the `edges` array.

2. **Tell the user** the workforce is saved as a draft and offer to test it.

3. **Test the draft** (no publish needed — see Draft-First Workflow):

   ```
   relevance_trigger_workforce({ workforce_id: "...", message: "Research AI trends" })
   ```

4. **Poll for completion**:

   ```
   relevance_poll_workforce_task_result({ workforce_id: "...", task_id: "...", wait_seconds: 50 })
   ```

5. **Inspect the run**:

   ```
   relevance_get_workforce_task_messages({ workforce_id: "...", task_id: "..." })
   ```

6. **Publish when the user confirms** (see Draft-First Workflow — ask before publishing any agent drafts in the graph):

   ```
   relevance_publish_workforce({
     workforce_id: "...",
     version_name: "Add QA reviewer agent",
     version_description: "Routes drafted replies through a reviewer agent before they're sent.",
   })
   ```

## Orchestrator Pattern & Connected-Node Pills

A workforce can be wired in two shapes — pick based on the intent:

- **Linear handover chain** (Quick Start above): agents run in order via `forced-handover` edges. No special prompt wiring — each agent just receives the previous agent's output. Use for pipelines (research → summarise → report).
- **Orchestrator + subagents**: one agent routes to others on demand via `tool-call` edges. The orchestrator's `system_prompt` references each subagent by node ID, and at runtime each pill becomes the subagent's tool-call name. Use when the orchestrator decides which subagent to call (router, dispatcher, intent-classifier).

### The `{{_workforce_node.<node_id>}}` placeholder

In an orchestrator's `system_prompt`, refer to each connected subagent (or connected tool node) with a `{{_workforce_node.<node_id>}}` pill, where `<node_id>` is the **workforce-graph `node_id`** of that subagent — NOT the underlying `agent_id`. The renderer:

- Substitutes the pill with the node's name at runtime (`CreateWorkforceNodeMapping`).
- Renders it as an **agent-style pill** when the node is `type: "agent"`, and a tool-style pill when `type: "tool"`. Same placeholder, different visual based on the node's type.

This is the sibling of `{{_actions.<id>}}` pills (see [`../managing-relevance-agents/system-prompts.md`](../managing-relevance-agents/system-prompts.md)) — same inline-only rule, just for a different ID space:

- `{{_actions.<id>}}` — references an action **attached to the agent itself**.
- `{{_workforce_node.<node_id>}}` — references a node connected via a `tool-call` edge in the **workforce graph**.

Both follow the same contract: inline pills at decision points, no trailing references block (see "Inline only" below). The runtime presents the agent's attached tools AND its workforce-connected subagents via the function-calling interface, so a hand-written list duplicates that catalogue in either case.

> **Important:** Use `{{_workforce_node.<node_id>}}` only for subagents/tools connected to the orchestrator with `tool-call` edges in the SAME workforce graph. Pills referencing nodes that aren't connected won't substitute correctly and won't render as proper pills in the editor.

### Inline only — do NOT add a trailing references block

Weave each `{{_workforce_node.<node_id>}}` pill inline into the routing prose at the decision point — that's it. Do NOT hand-write a trailing `## Subagent References` / `## Sub-agent References` / `## Subagents` / `## Connected Subagents` block that re-lists every pill below the routing rules.

At runtime the orchestrator already has every connected subagent (and connected tool node) presented via the function-calling interface — a hand-written list duplicates that catalogue inside the prompt, burns input tokens on every turn, and adds editor clutter. This is the **same rule** that applies to `{{_actions.<id>}}` pills in `../managing-relevance-agents/system-prompts.md`: inline only, no trailing references block, in either case.

```markdown
# GOOD — inline pills only, no trailing references block

You are a routing orchestrator with two subagents.

Routing rules:

1. If the user's message is "beep", call the {{_workforce_node.node_beep_uuid}} subagent and return its response.
2. If the user's message is "boop", call the {{_workforce_node.node_boop_uuid}} subagent and return its response.
3. Otherwise reply: not beep or boop.
```

```markdown
# BAD — trailing "## Subagent References" block re-lists what's already inline

You are a routing orchestrator with two subagents.

Routing rules:

1. If the user's message is "beep", call the {{_workforce_node.node_beep_uuid}} subagent.
2. If the user's message is "boop", call the {{_workforce_node.node_boop_uuid}} subagent.
3. Otherwise reply: not beep or boop.

## Subagent References

- Beep subagent: {{_workforce_node.node_beep_uuid}}
- Boop subagent: {{_workforce_node.node_boop_uuid}}
```

If you find yourself writing `## Subagent References` (or any equivalent trailing list of `{{_workforce_node.<id>}}` pills, with any heading), delete it.

### Looking up node_ids

`relevance_create_workforce` and `relevance_get_workforce` return `workforce.workforce_graph.nodes[]`. Each node carries:

- `node_id` — the UUID for the placeholder
- `type` — `"agent"` or `"tool"`
- `config.entity_link.agent_id` (for agent nodes) or `config.entity_link.tool_id` (for tool nodes) — the underlying entity

To find a subagent's `node_id`, match the subagent's `agent_id` against `config.entity_link.agent_id` on the returned nodes.

### Two-pass workflow for orchestrator prompts

Like `{{_actions.<id>}}`, orchestrator pills are a **two-pass** workflow because node_ids only exist after the workforce is created:

1. **Create the subagent agents.** Returns each subagent's `agent_id`.
2. **Create the orchestrator agent** with a placeholder/draft `system_prompt` (no pills yet — node_ids don't exist).
3. **Create the workforce** with all agents (orchestrator + subagents) and explicit `edges` of type `tool-call` from the orchestrator to each subagent. The response's `workforce.workforce_graph.nodes[]` gives you the `node_id → agent_id` mapping.
4. **Update the orchestrator's `system_prompt`** via `relevance_update_agent`, weaving `{{_workforce_node.<node_id>}}` pills inline at every point the orchestrator should route to a given subagent.

### Worked example: beep/boop router

```
relevance_create_agent({ name: "Beep", system_prompt: "Reply: beep" })
// → { agent_id: "agent_beep_xxx" }
relevance_create_agent({ name: "Boop", system_prompt: "Reply: boop" })
// → { agent_id: "agent_boop_xxx" }
relevance_create_agent({ name: "Orchestrator", system_prompt: "(placeholder)" })
// → { agent_id: "agent_orch_xxx" }

relevance_create_workforce({
  name: "beep boop",
  agents: [
    { agentId: "agent_orch_xxx" },  // index 0 — orchestrator
    { agentId: "agent_beep_xxx" },  // index 1 — subagent
    { agentId: "agent_boop_xxx" },  // index 2 — subagent
  ],
  edges: [
    { source: { kind: "trigger", index: 0 }, target: { kind: "agent", index: 0 }, edge_type: "forced-handover" },
    { source: { kind: "agent",   index: 0 }, target: { kind: "agent", index: 1 }, edge_type: "tool-call" },
    { source: { kind: "agent",   index: 0 }, target: { kind: "agent", index: 2 }, edge_type: "tool-call" },
  ],
})
// `source: { kind: "trigger", index: 0 }` points at the (default) manual trigger.
// If you omit the trigger edge while supplying other edges, one is auto-added
// to agents[0]. Include it explicitly to route the trigger into a different agent.
// → response.workforce.workforce_graph.nodes[] carries node_id per agent.
// Match by config.entity_link.agent_id to find the Beep and Boop node_ids.
// Say they are "node_beep_uuid" and "node_boop_uuid".

relevance_update_agent({
  agent_id: "agent_orch_xxx",
  system_prompt: `
You are a routing orchestrator. You have two subagents available: one named Beep and one named Boop.

Follow these rules every time:

1. If the user's message is "beep" (case-insensitive, trimmed), immediately call the {{_workforce_node.node_beep_uuid}} subagent. Do not reply in text — call the subagent.
2. If the user's message is "boop", immediately call the {{_workforce_node.node_boop_uuid}} subagent.
3. Otherwise reply with exactly: not beep or boop

Never explain your decision. Never add extra words.
`,
})
```

Notes on the example:

- The prose says "subagent" everywhere — never "sub-agent tool".
- Pills are woven **inline** where each route happens. There is NO trailing `## Subagent References` block (or `## Tool References`, or any equivalent) — the runtime already presents the connected subagents to the orchestrator via the function-calling interface, so a hand-written list is duplicate token spend.
- The orchestrator is the source of `tool-call` edges; the subagents are targets.

## Testing: Define Before Building

**IMPORTANT:** Before building any workforce, create a testing rubric and get user approval.

### Rubric Template

```
Testing Rubric for "[Workforce Name]":

□ End-to-End Flow
  - [Trigger → Final output works with typical input]
  - [All agents in chain execute successfully]

□ Agent Handovers
  - [Data passes correctly between agents]
  - [Each agent receives expected context]

□ Edge Cases
  - [Handles agent failures gracefully]
  - [Timeouts are handled appropriately]

□ Output Quality
  - [Final output meets business requirements]
  - [Intermediate outputs are logged/accessible]
```

### Testing Workflow

1. **Present rubric to user** before building
2. **Get approval** or incorporate feedback
3. **Build the workforce**
4. **Trigger with test input** using `relevance_trigger_workforce`
5. **Check execution** using `relevance_get_workforce_task_messages`
6. **Report results** with pass/fail for each check

### Example: Research Pipeline Rubric

```
Testing Rubric for "Lead Research Pipeline":

□ End-to-End Flow
  - Given a LinkedIn URL, produces enriched lead report
  - All 3 agents (Researcher → Enricher → Reporter) complete

□ Agent Handovers
  - Researcher output includes profile data
  - Enricher receives profile and adds company context
  - Reporter receives all data and formats final output

□ Edge Cases
  - Invalid LinkedIn URL produces clear error (not silent failure)
  - Private profiles handled with partial data message

□ Output Quality
  - Final report is structured and actionable
  - Sources/data provenance is clear
```

---

## Reference Files

- [concepts.md](concepts.md) - Core concepts: nodes, edges, threading, handovers
- [workforce-triggers.md](workforce-triggers.md) - Attaching non-manual triggers (Slack/Gmail/recurring/custom_webhook/...) to a workforce
- [workforce-nodes.md](workforce-nodes.md) - Tool nodes, condition nodes (branching/routing), and respecting note nodes when editing
- [debugging.md](debugging.md) - How to debug workforce execution

## URL Patterns

```
# Workforce edit page
https://app.relevanceai.com/workforces/{region}/{project}/{workforceId}

# Workforce task view
https://app.relevanceai.com/workforces/{region}/{project}/{workforceId}/tasks/{taskId}
```
