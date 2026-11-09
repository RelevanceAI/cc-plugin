---
name: relevance-evals/workforces
description: Workforce evals — node-scoped tool simulation and tool_usage checks, sub-agent calls, a worked example.
---

# Evals on Workforces

## Agents vs Workforces

Evals and dashboards work for both individual agents and workforces (multi-agent systems). The tools are the same — only `resource_type` and a few fields differ:

| Concept           | Agent                                    | Workforce                                                                                                                      |
| ----------------- | ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `resource_type`   | `agent`                                  | `workforce`                                                                                                                    |
| `resource_id`     | The agent ID                             | The workforce ID                                                                                                               |
| Execution unit    | Conversation                             | Workforce task                                                                                                                 |
| What happens      | Creates a conversation from the scenario | Creates a workforce task from the scenario                                                                                     |
| Tool simulation   | Direct tool overrides via `tool_configs` | **Per-node** overrides via `node_configs[node_id]` (scopes each graph node — each sub-agent and tool node too — independently) |
| Tool-usage checks | Scoped to the agent's conversation       | **Scope to a node** with `node_id` to assert that node's (or its sub-agent's) tool/sub-agent calls                             |

> **Workforces are node-scoped, not agent-scoped.** A workforce is a graph of **nodes**; the same agent can appear in more than one node. Both tool simulation and tool-usage checks key off the graph `node_id`, so each occurrence is configured independently — and a node's **sub-agents are callable like tools**, so you can simulate or assert them too. See [Tool simulation per node](#tool-simulation-per-node) and [Node-scoped `tool_usage` checks](#node-scoped-tool_usage-checks) below. Discover a workforce's nodes with `relevance_get_workforce`, and a node's full toolset (its tools **and** sub-agents, with their `action_id`s) with `relevance_get_agent_tools` passing the node's `agent_id` + `workforce_context: { workforce_id, node_id }`.

## Tool simulation per node

A workforce is a graph of **nodes**, and the same agent can appear in more than one node — so scope simulation **per node** with `node_configs`, keyed by `node_id`. Each entry has the same shape as the agent config ([For Agents](simulation.md#for-agents)). This is what lets a node's **sub-agents** (and a sub-agent's own tools) be simulated independently of any other occurrence of that agent.

```typescript
{
  node_configs: {
    '<node_id>': {
      // Same shape as the agent config (simulation.md). Keys are action_ids of the
      // node's tools OR sub-agents (a sub-agent is callable like a tool).
      tool_configs: { '<tool_or_subagent_action_id>': { /* same as agent shape */ } },
    },
  },
}
```

**Discovering the ids** (do this before writing the config):

1. `relevance_get_workforce` → the graph, with a `node_id` for every node.
2. `relevance_get_agent_tools` with the node's `agent_id` + `workforce_context: { workforce_id, node_id }` → that node's **full** toolset: its directly-attached tools, graph-attached tools, **and** the sub-agents it can call. Each entry has an `action_id` and a `kind` of `"tool"`, `"sub_agent"` or `"mcp_tool"`. Use those `action_id`s as the `tool_configs` keys (and as `tool_id` in [tool-usage checks](#node-scoped-tool_usage-checks)).

> **Why per-node, not per-agent?** If the same agent runs in two nodes, `node_configs` lets each one simulate (or run live) independently. To simulate a **sub-agent's own** downstream tools, configure them under that sub-agent's node — discover them by calling `relevance_get_agent_tools` with that node's id.
>
> `agent_configs[agent_id]` is still accepted as a back-compat fallback (a node's `node_configs` entry overrides it per tool; tools that entry doesn't name keep the agent entry's setting), but prefer `node_configs` for new workforces.

### Tool nodes

A **tool node** that a `forced-handover` edge leads to (from a trigger, agent, tool or condition node) runs as a step of its own, so it is simulated by an entry under **its own** `node_id`. Key its tool by the `action_id` in `tool_node_action_ids` from `relevance_get_workforce` (the same id an agent gives that tool). Only `node_configs` applies to a tool node, since no agent runs it.

- **Simulated** (`simulate_output`) or **fixed** (`output_overrides.output`) output both work, as for an agent's tool. A fixed output applies only with `output_overrides_enabled: true`; without it the tool runs live. Set only one of them: an entry with both fails the run, as it does for an agent's tool. A simulated tool node's output is the JSON the simulator returns, so later nodes read its fields as they would after a real run.
- A simulated tool node needs no connected account, as for an agent's tool ([simulation](simulation.md)). With `input_overrides_enabled: true`, `input_overrides` override the named inputs the tool node would otherwise receive, whether its output is simulated, fixed or live. Once you set `input_overrides_enabled` yourself, a simulated tool node goes without only the inputs whose content type `input_override_settings.content_types_to_make_optional` lists, so list `oauth_account` and `knowledge_set` there to keep that.
- Only the `default` override applies: a tool node keeps no run count, so run-indexed overrides (`"0"`, `"1"`, …) are ignored.

```typescript
{
  node_configs: {
    '<tool_node_id>': {
      tool_configs: {
        '<tool_node_action_id>': { overrides: { default: {
          output_overrides_enabled: true,
          output_overrides: { simulate_output: { is_simulated: true } },
        } } },
      },
    },
  },
}
```

A tool node an agent calls over a `tool-call` edge runs as that agent's tool instead: simulate it under the **calling agent's** node, keyed by the `action_id` `relevance_get_agent_tools` returns for it. Its simulated output keeps the agent's `{ output }` shape rather than the tool's own fields.

## Node-scoped `tool_usage` checks

**Workforce scoping (`node_id`).** For a workforce, a `tool_usage` check optionally takes a `node_id` (from `relevance_get_workforce`). It then counts calls within **that node's own conversation(s)** only — aggregated across the node's runs, deduped per conversation. This is the only way to assert a **sub-agent's own** tool calls: without `node_id`, a workforce check sees only the orchestrator's (main) conversation. Two common shapes:

- **Orchestrator invoked a sub-agent** → `tool_usage` on the orchestrator node (or no `node_id`), with `tool_id` = the **sub-agent's `action_id`** (sub-agents are called like tools, so the call lands in the orchestrator's conversation).
- **A sub-agent used one of its own tools** → `tool_usage` with `node_id` = the **sub-agent's node**, and `tool_id` = that tool's `action_id` (both from `relevance_get_agent_tools` with the node's `agent_id` + `workforce_context: { workforce_id, node_id }`).

## Workforce example (node-scoped simulation + sub-agent tool-usage)

For a workforce, first discover node ids and each node's toolset, then scope simulation and checks per node.

```
// 1. Discover nodes (each agent node exposes its node_id and agent_id)
relevance_get_workforce(workforce_id: "<workforce_id>")
→ nodes: [{ node_id: "<summarizer_node>", config: { entity_link: { agent_id: "<summarizer_agent_id>" } }, ... }, ...]

// 2. Discover the summarizer node's tools AND sub-agents (with action_ids)
relevance_get_agent_tools(agent_id: "<summarizer_agent_id>",
  workforce_context: { workforce_id: "<workforce_id>", node_id: "<summarizer_node>" })
→ tools: [
    { action_id: "<send_email_action_id>",  kind: "tool" },
    { action_id: "<web_search_action_id>",  kind: "tool" },
  ]

// 3. Simulate the summarizer node's side-effecting tool (test-set level, per node)
relevance_update_eval_test_set(
  resource_type: "workforce", resource_id: "<workforce_id>", test_set_id: "<test_set_id>",
  tool_simulation_config: {
    node_configs: {
      "<summarizer_node>": {
        tool_configs: { "<send_email_action_id>": { overrides: { default: {
          output_overrides_enabled: true,
          output_overrides: { simulate_output: { is_simulated: true, simulation_prompt: "Return a sent-email confirmation." } }
        } } } }
      }
    }
  }
)

// 4. Node-scoped check: the summarizer node used web_search at least once
relevance_create_eval_check(resource_type: "workforce", resource_id: "<workforce_id>",
  name: "Summarizer searched the web",
  check_config: { type: "tool_usage", tool_id: "<web_search_action_id>", node_id: "<summarizer_node>",
                  position: "anywhere", operator: "at_least", count: 1 })
→ { "check_id": "<check_search>" }
```

Remember the `action_id`-not-`chain_id` rule applies to **sub-agents** too: to assert "the orchestrator invoked the Summarizer sub-agent", use a `tool_usage` check on the orchestrator node with `tool_id` = the Summarizer's `action_id` (a `kind: "sub_agent"` entry from step 2).

## Reading results

For a **workforce**, a node-scoped `tool_usage` check (one created with a `node_id`) is scored only against that node's conversation(s), so its `rule_results` entry is attributable to that node — name the checks accordingly (e.g. "Summarizer searched the web") so the results table makes clear which node each tool/sub-agent assertion covers.
