---
title: Workforce Tool / Condition / Note Nodes
description: Attach tool nodes (workforce-level tool steps), condition nodes (route execution down one or many branches via LLM prompt or rule filters), and read/respect note nodes (canvas annotations a human left for you). Load when building a workforce that needs branching, embedded tool calls, or when editing a graph that contains user-authored notes.
---

# Workforce Tool / Condition / Note Nodes

A workforce graph is a DAG of five node kinds: `trigger`, `agent`, `tool`, `condition`, `note`. The `agents` + `triggers` inputs on `relevance_create_workforce` cover the first two — this doc covers the remaining three. Pass them via `tools`, `conditions`, and (for notes — read-only on create, writable on update) the `workforce_graph.nodes` array on `relevance_update_workforce`.

> **Edge addressing.** Every edge identifies its endpoints as `{ source: { kind, index }, target: { kind, index } }` where `kind` is `trigger | agent | tool | condition` (source) or `agent | tool | condition` (target). `index` is a zero-based index into the matching node array you passed at create time. See [`SKILL.md`](SKILL.md) for the full edge reference.

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

When **updating** an existing workforce via `relevance_update_workforce`, each condition node's `edge_order` is rebuilt from current outgoing edges: existing entries keep their relative position; newly-added edges get appended at the end. If you need a new edge to evaluate _before_ an existing one, pass an explicit condition node patch with the desired `edge_order` array in `workforce_graph.nodes`.

### Condition payload — prompt-based vs rule-based

Each outgoing edge from a condition node MUST carry a `condition` payload:

| Type           | When to use                                                                                              | Shape                                                         |
| -------------- | -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| `prompt-based` | The decision needs LLM reasoning — semantic intent, judgement, soft criteria                             | `{ type: "prompt-based", config: { prompt: "..." } }`         |
| `rule-based`   | The decision is a structural filter over input fields — value equality, comparisons, AND/OR combinations | `{ type: "rule-based", config: { filters: [<FilterItem>] } }` |

Prompt-based conditions optionally use `llm_condition_model` on the node (e.g. `llm_condition_model: 'openai-gpt-5'`) to override the default model. Rule-based filters use the same shape as `FilterItem` filters elsewhere in the platform — refer to the relevant tool docs when constructing them.

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
