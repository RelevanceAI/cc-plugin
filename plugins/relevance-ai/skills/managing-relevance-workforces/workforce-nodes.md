---
title: Workforce Tool / Condition / Note Nodes
description: Attach tool nodes (workforce-level tool steps), condition nodes (route execution down one or many branches via LLM prompt or rule filters), and read/respect note nodes (canvas annotations a human left for you). Load when building a workforce that needs branching, embedded tool calls, or when editing a graph that contains user-authored notes.
---

# Workforce Tool / Condition / Note Nodes

A workforce graph is a DAG of five node kinds: `trigger`, `agent`, `tool`, `condition`, `note`. The `agents` + `triggers` inputs on `relevance_create_workforce` cover the first two — this doc covers the remaining three. Pass them via `tools` and `conditions` on either tool (on `relevance_update_workforce` they add nodes to an existing workforce, wired with `edges` whose ends are `{ kind, index }` or `{ node_id }`), and notes via the `workforce_graph.nodes` array on `relevance_update_workforce`.

> **Edge addressing.** Each edge end is either `{ kind, index }` or `{ node_id }`. `kind` is `trigger | agent | tool | condition` (source) or `agent | tool | condition` (target), and `index` is a zero-based index into the matching node array passed in the same call. `{ node_id }` names a node already in the workforce (update only). See [`SKILL.md`](SKILL.md) for the full edge reference.

## Tool Nodes

A tool node is a **direct reference to an existing tool (studio/chain) by id**. Unlike attaching a tool to an agent (which makes the tool one of the agent's available actions), a tool node is a first-class step in the workforce graph — typically used when an agent should hand off to a specific tool and then resume, or when one tool's output feeds another.

```typescript
relevance_create_workforce({
  name: 'Enrich + Score',
  agents: [{ agentId: 'agent_classifier' }],
  tools: [{ tool_id: 'tool_company_enrichment' }],
  edges: [
    {
      source: { kind: 'agent', index: 0 },
      target: { kind: 'tool', index: 0 },
      edge_type: 'tool-call',
      tool_input_config: {
        domain: { input_source: { type: 'auto' } },
      },
    },
    {
      source: { kind: 'tool', index: 0 },
      target: { kind: 'agent', index: 0 },
      trigger_message_prefix: 'Enrichment result for the input domain: ',
    },
  ],
});
```

### How to find a `tool_id`

Cross-reference [`../managing-relevance-tools/SKILL.md`](../managing-relevance-tools/SKILL.md) for the canonical tool lifecycle.

```typescript
relevance_list_tools(); // returns { results: [{ studio_id, title, emoji, description, ... }] }
```

`tool_id` is the same value as `studio_id` from `relevance_list_tools`. Pick the one whose title/description matches what the workforce needs, and pass it to `tools: [{ tool_id }]`.

If the user describes a tool that doesn't exist yet, create it first via `relevance_create_tool` (see `managing-relevance-tools/SKILL.md`), then reference it from the workforce.

### `tool_input_config` — wiring inputs into a tool

Tool nodes receive inputs via the **incoming edge**, not on the node itself. Each input is mapped to one of two sources:

```typescript
tool_input_config: {
  <param_name>: {
    input_source: { type: 'manual' | 'auto' },
  },
}
```

- `manual` — the upstream agent (or whoever connected the edge) provides the value at runtime, either as a fixed string in `default_values` or because the orchestrating LLM is filling it in.
- `auto` — the value is sourced automatically from the upstream node's output / scratchpad. Use when the tool's input maps 1:1 to something the previous node produced.

Both forms are loose passthrough at the tool boundary; the backend re-validates against the tool's `input_schema` when the workforce saves. To find the parameter names, fetch the tool: `relevance_get_tool({ tool_id })` returns the tool's `input_schema`.

### Valid edges to/from tool nodes

| Edge               | edge_type                        | Special fields                                                                          |
| ------------------ | -------------------------------- | --------------------------------------------------------------------------------------- |
| `trigger → tool`   | `forced-handover`                | none                                                                                    |
| `agent → tool`     | `tool-call` or `forced-handover` | `tool_input_config`, `action_behaviour`, `prompt_for_when_to_use`                       |
| `tool → agent`     | `forced-handover`                | `threading_behavior`, `trigger_message_prefix` (prepended to the agent's input message) |
| `tool → tool`      | `forced-handover`                | `tool_input_config`                                                                     |
| `tool → condition` | `forced-handover`                | none                                                                                    |

## Condition Nodes

A condition node routes execution down one of several outgoing edges based on per-edge **condition payloads**. Use for "if/elif/else" branching, intent classification, or any decision that can't be expressed as plain agent prompting.

A condition node evaluates outgoing edges **in declared order** and follows the **first** edge whose condition passes — like `if / elif / else`. (This is the only mode the runtime currently supports.)

> **`condition_type` is fixed at `"run-one"`.** A `run-multiple` option is not supported (`apps/builder-app/.../ConditionPanel.vue:210-217` and `useWorkforceBuilderInteractions.ts:65`) — you cannot set `condition_type` on a condition node passed to `relevance_create_workforce` / `relevance_update_workforce`.

```typescript
relevance_create_workforce({
  name: 'Triage',
  agents: [
    { agentId: 'agent_classifier' },
    { agentId: 'agent_urgent' },
    { agentId: 'agent_normal' },
  ],
  conditions: [{ label: 'Urgency?' }],
  edges: [
    // classifier feeds the condition node
    {
      source: { kind: 'agent', index: 0 },
      target: { kind: 'condition', index: 0 },
    },
    // condition routes to urgent OR normal — first passing edge wins
    {
      source: { kind: 'condition', index: 0 },
      target: { kind: 'agent', index: 1 },
      condition: {
        type: 'prompt-based',
        config: {
          prompt:
            'Is the upstream message describing a customer-impacting outage, escalation, or security incident?',
        },
      },
    },
    {
      source: { kind: 'condition', index: 0 },
      target: { kind: 'agent', index: 2 },
      condition: {
        type: 'rule-based',
        config: { filters: [] }, // empty filter = catch-all
      },
    },
  ],
});
```

### Routing order

The order outgoing edges are evaluated in is determined by the order you list them in the `edges` array — `condition_node.config.edge_order` is backfilled from the insertion order. Put the most specific branch first; put a catch-all (empty `filters: []` or a permissive prompt) last.

When **updating** with `edges`, extending a saved condition with other surviving routes requires a complete order in the same call. Give each new edge an `edge_id` and patch the condition's `config.edge_order` with every final outgoing edge ID exactly once, in priority order. Preserve its other config and metadata. Put specific branches before a catch-all; listing them after one deliberately leaves the catch-all with priority.

For a saved condition whose only route is `catchall`, use the freshly read condition node as `savedCondition`:

```typescript
relevance_update_workforce({
  workforce_id: 'workforce-id',
  agents: [{ agentId: 'agent_urgent' }],
  edges: [
    {
      edge_id: 'urgent-route',
      source: { node_id: savedCondition.node_id },
      target: { kind: 'agent', index: 0 },
      condition: {
        type: 'prompt-based',
        config: { prompt: 'Is this urgent?' },
      },
    },
  ],
  workforce_graph: {
    nodes: [
      {
        ...savedCondition,
        config: {
          ...savedCondition.config,
          edge_order: ['urgent-route', 'catchall'],
        },
      },
    ],
  },
});
```

`edge_id` is optional on create and update; omitted IDs are generated. The `edges` input only adds edges: IDs must be unique within the request and must not collide with saved edges or raw `workforce_graph.edges` patches, even if those IDs are being removed. To edit an existing edge, use `workforce_graph.edges`. If an addition is retried after saving, inspect the graph before making another change.

Missing, malformed, duplicate or incomplete orders are rejected before saving. An order must match the outgoing edges after edits and removals, so removed edges and edges from other conditions do not belong in it. Refetch if the graph changed. New conditions, and saved conditions whose other routes are all removed, use input order when no explicit order is supplied. Any explicit order submitted alongside additions must be complete. Raw-only graph updates keep the existing merge behavior, preserving listed positions and appending unlisted edges.

### Condition payload — prompt-based vs rule-based

Each outgoing edge from a condition node MUST carry a `condition` payload:

| Type           | When to use                                                                                              | Shape                                                         |
| -------------- | -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| `prompt-based` | The decision needs LLM reasoning — semantic intent, judgement, soft criteria                             | `{ type: "prompt-based", config: { prompt: "..." } }`         |
| `rule-based`   | The decision is a structural filter over input fields — value equality, comparisons, AND/OR combinations | `{ type: "rule-based", config: { filters: [<FilterItem>] } }` |

Prompt-based conditions optionally use `llm_condition_model` on the node (e.g. `llm_condition_model: 'relevance-cost-optimized'`) to override the default model — never hardcode a versioned model id, call `relevance_list_llm_models` if you need a specific one.

A rule-based edge passes when every filter passes. A filter is `{ filter_type, condition, target, source }`:

- `target`: the value to check, as a variable: `{{node_context.nodes.<node_id>.output.response}}` for an agent's reply, `{{node_context.nodes.<node_id>.output.<field>}}` for a tool's output. The node must be saved first, so add the filter with `relevance_update_workforce`.
- `source`: the string to compare with.
- `exact_match`, `contains`, `starts_with` and `ends_with` take `==` or `!=`. `numeric` and `date` take `>`, `<`, `>=` or `<=`.
- `exact_match` needs `value_type`: `string`, `number`, `boolean` or `date`. `contains` needs `value_type`: `string` or `array`.
- `{ filter_type: 'and' | 'or', value: [<filters>] }` groups filters.

```typescript
{ filter_type: 'contains', value_type: 'string', condition: '==', target: '{{node_context.nodes.<node_id>.output.response}}', source: 'urgent' }
```

#### Judging prompt-based conditions with a decision model

Instead of an LLM, a condition node can judge its prompt-based edges with a **decision model**: set `decision_condition_model` on the condition when creating the workforce. A decision model reads the previous node's output and picks the most likely edge, or none of them, in well under a second, with a probability for each; it does not explain its choice. Rule-based edges and the routing order work exactly as with an LLM.

- Take the id from `relevance_list_decision_models` — never write one from memory. It lists only the models this project may use, and each entry's `has_user_key` says whether the project already holds the provider key the model needs (`project_key_name`). If the list is empty, decision models are not available here: judge the condition with an LLM instead.
- `decision_condition_model` and `llm_condition_model` are mutually exclusive on a node.
- The node stores it as `config.decision_condition_model`. A node patch passed to `relevance_update_workforce` replaces the whole node, so keep `decision_condition_model` in the config you send, or the node falls back to an LLM.
- If the key is missing, the run fails at the condition with an "add your API key" error rather than falling back to an LLM.

### Valid edges to/from condition nodes

| Edge                  | edge_type         | Required                                          |
| --------------------- | ----------------- | ------------------------------------------------- |
| `trigger → condition` | `forced-handover` | none                                              |
| `agent → condition`   | `forced-handover` | none                                              |
| `tool → condition`    | `forced-handover` | none                                              |
| `condition → agent`   | `forced-handover` | `condition` payload                               |
| `condition → tool`    | `forced-handover` | `condition` payload, optional `tool_input_config` |

A condition node cannot connect to another condition node directly — chain them through an agent or tool node if you need multi-stage routing.

## Note Nodes

Note nodes are **canvas annotations a human left on the graph**. They have no runtime effect — they don't fire, route, or otherwise participate in execution. But they're load-bearing for you, the LLM, when editing the graph.

### Read notes before editing

Before mutating a workforce, fetch the current graph and check for notes:

```typescript
const wf = await relevance_get_workforce({ workforce_id });
const notes = wf.workforce_graph.nodes.filter((n) => n.type === 'note');
// each note: { node_id, type: 'note', config: { content: string, label?: string }, metadata: { position } }
```

Note `content` often carries constraints or intent the user didn't put anywhere else ("don't ever bypass the legal-review agent", "this branch is on hold until Q3", "the urgent threshold was negotiated with sales, don't change it without checking"). **Respect what notes say.** If a note conflicts with what the user is asking you to do now, surface the conflict and ask the user to confirm before proceeding.

### Creating / editing notes

Notes are only writable through `relevance_update_workforce` (not `create_workforce`):

```typescript
relevance_update_workforce({
  workforce_id,
  workforce_graph: {
    nodes: [
      {
        node_id: '<uuid>', // generate a new one for a new note
        type: 'note',
        config: {
          content:
            'Daily classifier was tuned for B2B leads only — do not repurpose for support tickets.',
        },
      },
    ],
  },
});
```

Updates merge by `node_id` (see [`SKILL.md`](SKILL.md) Draft-First Workflow) so this adds the note without disturbing anything else. To remove a note, pass its id in `remove_node_ids`.

## End-to-End Example: Triage Pipeline

A complete workforce that classifies inbound webhooks into urgent vs. normal:

```typescript
// 1. Create three agents and a tool
const classifier = await relevance_create_agent({
  name: 'Classifier',
  system_prompt: '...',
});
const urgent = await relevance_create_agent({
  name: 'Urgent Handler',
  system_prompt: '...',
});
const normal = await relevance_create_agent({
  name: 'Normal Handler',
  system_prompt: '...',
});
const enrichment = (await relevance_list_tools()).results.find(
  (t) => t.title === 'Company Enrichment'
);

// 2. Wire them up
await relevance_create_workforce({
  name: 'Inbound Triage',
  agents: [
    { agentId: classifier.agent_id }, // index 0
    { agentId: urgent.agent_id }, // index 1
    { agentId: normal.agent_id }, // index 2
  ],
  tools: [{ tool_id: enrichment.studio_id }], // index 0
  conditions: [{ label: 'Urgent?' }], // index 0
  triggers: [
    {
      type: 'auto',
      sync_data: {
        config: {
          type: 'custom_webhook',
          custom_webhook: { webhook_name: 'Inbound triage' },
        },
      },
    },
  ],
  edges: [
    // trigger -> classifier
    {
      source: { kind: 'trigger', index: 0 },
      target: { kind: 'agent', index: 0 },
    },
    // classifier enriches via tool, then resumes
    {
      source: { kind: 'agent', index: 0 },
      target: { kind: 'tool', index: 0 },
      edge_type: 'tool-call',
      tool_input_config: { domain: { input_source: { type: 'auto' } } },
    },
    {
      source: { kind: 'tool', index: 0 },
      target: { kind: 'agent', index: 0 },
    },
    // classifier feeds the condition node
    {
      source: { kind: 'agent', index: 0 },
      target: { kind: 'condition', index: 0 },
    },
    // urgent vs normal routing
    {
      source: { kind: 'condition', index: 0 },
      target: { kind: 'agent', index: 1 },
      condition: {
        type: 'prompt-based',
        config: {
          prompt: 'Is the lead urgent (>$50k ARR or competitor-mention)?',
        },
      },
    },
    {
      source: { kind: 'condition', index: 0 },
      target: { kind: 'agent', index: 2 },
      condition: { type: 'rule-based', config: { filters: [] } }, // catch-all
    },
  ],
});
```
