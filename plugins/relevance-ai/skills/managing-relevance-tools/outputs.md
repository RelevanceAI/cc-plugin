---
name: tool-outputs
description: How to configure what a tool returns at runtime and how it's displayed. Load this skill before the first `set_tool_output` call, and whenever editing a tool that drives a workforce `tool_trigger`.
---

# Tool Outputs

A tool's output configuration is two paired fields:

| Field           | What it controls                                                                                                                                                                                                                                                          |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| runtime output  | The runtime map — `{ <key>: '{{steps.<step>.output.<field>}}', ... }`. Determines what the tool actually returns.                                                                                                                                                         |
| `output_schema` | The display contract — `{ metadata: { field_order: [...] }, properties: { <key>: { metadata: { render_mode?, external_name?, content_type? } } } }`. Determines how the canvas Outputs panel renders the result and what downstream agents / `tool_trigger` listings see. |

**Heads up — the same data has two names depending on the tool you're calling:**

| Tool                                                          | Where the runtime map lives                                                     |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| `relevance_get_tool` (read) / `relevance_create_tool` (write) | **Nested:** `transformations.output`                                            |
| `relevance_set_tool_output` (write)                           | **Top-level parameter:** `output` (writes `transformations.output` server-side) |

> **Do NOT pass `output: ...` at the top level to `relevance_create_tool`.** On create, the runtime map goes under `transformations.output`. Only `set_tool_output` takes a top-level `output` parameter.

`output_schema` is top-level in every case (no rename).

The runtime keys (whatever you write them as) MUST match the keys declared in `output_schema.properties` — they're paired by design. Think of the runtime map as the tool's "final step": it runs after the last transformation and reshapes the result into the keys you declare. If it's omitted, the tool returns the last transformation step's `output` value verbatim.

## `relevance_set_tool_output`

```typescript
relevance_set_tool_output({
  studio_id: 'random-person',
  output: {
    first_name: '{{steps.gen_first_name.output.transformed.value}}',
    last_name: '{{steps.gen_last_name.output.transformed.value}}',
    age: '{{steps.gen_age.output.transformed.value}}',
  },
  output_schema: {
    metadata: { field_order: ['first_name', 'last_name', 'age'] },
    properties: {
      first_name: { metadata: { render_mode: 'raw' } },
      last_name: { metadata: { render_mode: 'raw' } },
      age: { metadata: { render_mode: 'raw' } },
    },
  },
});
// Runtime result: { first_name: "...", last_name: "...", age: 42 }
```

### Partial updates

`output` and `output_schema` are independent. Omit a field to leave it unchanged. Pass `null` to clear it.

```typescript
// Change only the display order, leave the runtime map alone.
relevance_set_tool_output({
  studio_id: 'random-person',
  output_schema: {
    metadata: { field_order: ['age', 'first_name', 'last_name'] },
    properties: {
      first_name: { metadata: { render_mode: 'raw' } },
      last_name: { metadata: { render_mode: 'raw' } },
      age: { metadata: { render_mode: 'raw' } },
    },
  },
});
```

### Passthrough

Pass `output: null` to drop the runtime map. The tool will return the last step's `output:` verbatim. You can still declare `output_schema` to tell downstream consumers what to expect:

```typescript
relevance_set_tool_output({
  studio_id: 'summarize',
  output: null,
  output_schema: {
    metadata: { field_order: ['summary'] },
    properties: { summary: { metadata: { render_mode: 'markdown' } } },
  },
});
```

### Reset

Pass both as `null` to clear them:

```typescript
relevance_set_tool_output({
  studio_id: 'random-person',
  output: null,
  output_schema: null,
});
```

## `render_mode` values

`auto | hide | html | image | json | markdown | raw | table`. Defaults to `auto`. Pick `raw` for plain text/numbers, `markdown` for prose, `json` for structured data, `table` for arrays of objects.

## When to set what

Default to declaring both `output` and `output_schema` so the runtime keys and the display contract stay in sync. Use the **runtime map** to:

- Expose values from a step that **isn't the last one** (e.g. "return the seed from step 1 and the numbers from step 3").
- **Flatten** a nested last-step output (e.g. `python_code_transformation` returns `{ transformed: { … } }`; the user wants the inner keys at the top level).
- **Rename or select a subset** of the keys the tool would otherwise expose.

Skip the runtime map (passthrough) only when the last step already returns exactly what you want — and even then, declare `output_schema` so the canvas + downstream LLMs see the contract.

## ⚠️ `tool_trigger` compatibility

A workforce trigger of `type: 'tool_trigger'` references **one of the tool's runtime output keys** by name (`tool_config.output_field`). At trigger time, the sync provider does a literal key lookup against the tool's runtime output payload. If the named field is missing, the trigger fails with `Tool output field not found`, **without surfacing the failure to the user.**

Before changing a tool's outputs when it is the source of an existing `tool_trigger`:

1. Read the current tool with `relevance_get_tool` and note which keys downstream consumers depend on.
2. Ask the user whether any workforce triggers reference this tool — and if so, which `output_field` they read.
3. If you remove or rename a key the trigger depends on, **say so explicitly** in chat: name the trigger, name the field, and confirm before saving.

The same constraint is documented from the workforce side at [`managing-relevance-workforces/workforce-triggers.md`](../managing-relevance-workforces/workforce-triggers.md).

## Verifying after a change

Always re-run the draft after editing outputs:

```typescript
relevance_run_tool({
  studio_id: 'random-person',
  params: {},
  version: 'draft',
});
// Confirm the response's `output` field has the expected top-level keys.
```

If the tool returns `{}` or unexpected keys, the typical fixes are:

- A step's `output:` mapping is empty → patch the step via `relevance_update_tool_step` to alias the transformation's response.
- An `output` template references a key that doesn't exist on the named step → use `relevance_run_transformation` to inspect that step's raw response shape.
- The runtime keys don't match the schema keys → call `set_tool_output` again with matching `output` and `output_schema`.

## See also

- [`SKILL.md`](SKILL.md) — overall tool authoring.
- [`transformations.md`](transformations.md) — per-step `output:` alias map (the layer underneath this one).
- [`managing-relevance-workforces/workforce-triggers.md`](../managing-relevance-workforces/workforce-triggers.md) — `tool_trigger` setup and the `output_field` precondition.
