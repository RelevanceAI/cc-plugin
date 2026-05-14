---
title: Creating Tools
description: Detailed workflow for creating Relevance AI tools — config shape, transformation steps, params/state mapping, output schemas, step editing, and version handling. Load when the SKILL.md overview isn't enough and you need step-by-step config detail.
---

# Creating Tools

Complete workflow for creating Relevance AI tools (studios).

## Creating New Tools

### Option 1: Direct Creation with `relevance_create_tool`

Auto-generates a UUID and saves the new tool as a DRAFT — call `relevance_publish_tool` to make it live:

```typescript
relevance_create_tool({
  title: "My New Tool",           // REQUIRED
  description: "What it does",
  prompt_description: "When AI should use this",
  params_schema: { ... },
  transformations: { steps: [...] },
  state_mapping: { ... }
})
// Returns { studio_id: "auto-generated-uuid", url }
// Tool is saved as a DRAFT — not yet live. Call relevance_publish_tool to publish.
```

### Option 2: Create from Transformation

Fastest way - auto-generates params_schema, state_mapping, and bindings:

```typescript
relevance_create_tool_from_transformation({
  transformationId: 'serper_google_search',
  title: 'Google Search', // optional custom title
  description: 'Search the web', // optional custom description
});
// Searches existing tools first to avoid duplicates
// Returns { studio_id, tool, wasExisting: boolean }

// Force create even if duplicate exists:
relevance_create_tool_from_transformation({
  transformationId: 'serper_google_search',
  forceCreate: true,
});
```

> **⚠️ Auto-generated tools often have broken output configs.** The tool may have empty step output mappings (`"output": {}`) and invalid final output references. This means the tool executes but **returns empty results**. Always validate with `relevance_run_tool` after creation. See [Fixing Auto-Generated Tool Outputs](#fixing-auto-generated-tool-outputs) below.

## Updating Existing Tools

`relevance_update_tool` auto-fetches the current draft and shallow-merges your fields. It saves to a DRAFT only — call `relevance_publish_tool` when ready:

```typescript
relevance_update_tool({
  studio_id: 'existing-tool-id',
  emoji: '🔍', // Only this changes, everything else preserved
});
// Then publish when the user confirms:
// relevance_publish_tool({ tool_id: 'existing-tool-id' });
```

## Tool Structure

```typescript
interface Tool {
  studio_id: string; // Unique identifier (slug or UUID)
  title: string; // Display name
  description?: string; // What the tool does
  prompt_description: string; // Instructions for AI on when to use
  emoji?: string; // Icon

  params_schema: JSONSchema; // Input parameters
  output_schema?: JSONSchema; // Output structure

  transformations: {
    steps: TransformationStep[]; // Sequential steps to execute
  };

  state_mapping?: Record<string, string>; // How data flows
}
```

## Transformation Step Structure

```typescript
interface TransformationStep {
  name: string; // Unique identifier for this step
  transformation: string; // Transformation type ID
  params: Record<string, string>; // Input params (use {{variable}} templating)
  output?: Record<string, string>; // Output mapping

  // Optional control flow
  foreach?: { iterator: string; item_key: string };
  if?: string;
}
```

## Creating a Basic Tool

```typescript
relevance_create_tool({
  title: 'My Tool',
  description: 'What this tool does',
  prompt_description: 'Instructions for AI on when to use this tool',
  params_schema: {
    type: 'object',
    properties: {
      query: { type: 'string', title: 'Search Query' },
    },
    required: ['query'],
  },
  transformations: {
    steps: [
      {
        name: 'search',
        transformation: 'serper_google_search',
        params: { search_query: '{{query}}' },
        output: { results: '{{organic}}' },
      },
    ],
  },
});
```

## Variable Templating

### Input Parameters

Reference tool inputs with `{{param_name}}`:

```typescript
params: {
  search_query: "{{query}}",      // From params_schema
  num_results: "{{count}}"
}
```

### Previous Step Outputs

Reference previous step outputs with `{{step_name.field}}`:

```typescript
// Step 1 output: { results: "..." }
// Step 2 access:
params: {
  text: '{{search.results}}';
}
```

### Loop Items

Inside loops, use `{{foreach.item}}` or just `{{item}}`:

```typescript
params: {
  url: '{{foreach.item.url}}';
}
```

## Multi-Step Example

```typescript
{
  studio_id: "search-and-summarize",
  title: "Search and Summarize",
  params_schema: {
    type: "object",
    properties: {
      topic: { type: "string", title: "Topic" }
    },
    required: ["topic"]
  },
  transformations: {
    steps: [
      {
        name: "search",
        transformation: "serper_google_search",
        params: { search_query: "{{topic}}" },
        output: { results: "{{organic}}" }
      },
      {
        name: "summarize",
        transformation: "prompt_completion",
        params: {
          prompt: "Summarize these results about {{topic}}:\n\n{{search.results}}",
          model: "anthropic-claude-sonnet-4"
        },
        output: { summary: "{{answer}}" }
      }
    ]
  },
  state_mapping: {
    topic: "params.topic",
    search: "steps.search.output",
    summarize: "steps.summarize.output"
  }
}
```

## Input Naming Rules

> **⚠️ Every `params_schema` property MUST have `title` and `description`.** Tools with unnamed inputs are unusable — agents cannot reason about what to pass, and users editing the tool have no context. Always infer meaningful names from the property key, the transformation it feeds into, and the user's stated intent.

```typescript
// WRONG — blank or missing title/description
{ query: { type: "string" } }

// CORRECT — named and described
{ query: { type: "string", title: "Search Query", description: "The search term to look up" } }
```

## Parameter Schema Types

### String Parameter

```typescript
{
  query: {
    type: "string",
    title: "Search Query",
    description: "What to search for"
  }
}
```

### Number Parameter

```typescript
{
  count: {
    type: "number",
    title: "Result Count",
    default: 10
  }
}
```

### Boolean Parameter

```typescript
{
  include_images: {
    type: "boolean",
    title: "Include Images",
    default: false
  }
}
```

### Enum Parameter

```typescript
{
  format: {
    type: "string",
    title: "Output Format",
    enum: ["json", "markdown", "text"]
  }
}
```

### Array Parameter

```typescript
{
  urls: {
    type: "array",
    title: "URLs to Process",
    items: { type: "string" }
  }
}
```

## Conditional Steps

Run steps conditionally:

```typescript
{
  name: "optional_step",
  transformation: "prompt_completion",
  if: "{{include_analysis}}",  // Only run if truthy
  params: { ... }
}
```

## Emoji / Icon

The `emoji` field accepts a unicode emoji or a full CDN URL to a brand SVG. **Prefer the brand icon when the tool wraps a known provider** (Slack, HubSpot, Gmail, etc.). See the **Tool Icons** section in [SKILL.md](SKILL.md#tool-icons) for the URL pattern, the list of available filenames, and the common-mistake table.

## Testing After Creation

```typescript
// Sync execution
relevance_run_tool({
  studio_id: 'my-tool',
  params: { query: 'test query' },
});
```

## Publishing

Both create and update save to a DRAFT — publishing is always a separate, user-confirmed step:

- **Creating a new tool** (`relevance_create_tool`): saves to a draft. The new tool has only a draft version; call `relevance_publish_tool` to make it live.
- **Updating an existing tool** (`relevance_update_tool`): saves to a draft only — the live version is unchanged until you call `relevance_publish_tool`.

`relevance_publish_tool` always shows an approval card to the user (even with auto-approve enabled), so confirm in chat before calling it:

```typescript
relevance_publish_tool({ tool_id: 'my-tool' });
```

Test the draft first using `relevance_run_tool` with `version: "draft"`:

```typescript
relevance_run_tool({
  studio_id: 'my-tool',
  params: { query: 'test query' },
  version: 'draft',
});
```

---

## Editing Transformation Steps

To edit transformation steps, use the dedicated per-step tools (all save to DRAFT only):

- `relevance_add_tool_step({ studio_id, step, position? })` — insert a new step. `position` is 0-based; omit (or pass a value `>= step_count`) to append.
- `relevance_update_tool_step({ studio_id, step_index, patch })` — shallow-merge `patch` into the step at 0-based `step_index`.
- `relevance_remove_tool_step({ studio_id, step_index })` — remove a single step (0-based).
- `relevance_move_tool_step({ studio_id, from_index, to_index })` — reorder a step (both 0-based; mirrors UI drag).

Always call `relevance_get_tool` first to confirm step indices in the current draft. Out-of-range indices throw with the current step count in the error message, so the LLM can recover.

---

## `state_mapping` is REQUIRED

**This is the #1 cause of "missing required property" errors when tools are called by agents.**

Every tool MUST have a `state_mapping` field that maps input params to template variables. Without this, `{{search_query}}` won't resolve even when the agent passes the parameter.

### How `state_mapping` works

The `state_mapping` connects two things:

- **Keys** = alias names used in `{{template}}` references (these are just plain names, NO prefix)
- **Values** = JSONPath into the tool's internal state (these DO use `params.` or `steps.` prefixes)

**The `params.` prefix belongs ONLY in state_mapping values, NEVER in params_schema property names:**

```json
{
  "params_schema": {
    "type": "object",
    "properties": {
      "search_query": { "type": "string" }
    }
  },
  "state_mapping": {
    "search_query": "params.search_query",
    "search": "steps.search.output"
  },
  "transformations": {
    "steps": [
      {
        "name": "search",
        "transformation": "serper_google_search",
        "params": { "search_query": "{{search_query}}" }
      }
    ]
  }
}
```

| Where                            | Uses `params.` prefix? | Example                                 |
| -------------------------------- | ---------------------- | --------------------------------------- |
| `params_schema` property names   | **NO**                 | `"search_query": { "type": "string" }`  |
| `state_mapping` keys             | **NO**                 | `"search_query": "params.search_query"` |
| `state_mapping` values           | **YES**                | `"search_query": "params.search_query"` |
| `transformations.steps[].params` | **NO**                 | `"search_query": "{{search_query}}"`    |

**CRITICAL:** Do NOT use curly braces in state_mapping values!

```json
// WRONG - curly braces cause params to not resolve
"state_mapping": { "name": "{{params.name}}" }

// CORRECT - no curly braces
"state_mapping": { "name": "params.name" }
```

### JS code steps vs non-JS steps

**JS code steps** have native access to a `steps` global object at runtime (e.g., `steps.search.output`), so they can access inter-step data directly without state_mapping. Prefer the native `steps` object over template injection for complex data. Do NOT put inter-step outputs in state_mapping for JS steps -- this creates phantom params the UI flags as missing.

**Non-JS steps** (prompt_completion, api_call, etc.) rely entirely on state_mapping for template resolution. When a non-JS step needs a previous step's output, you MUST add a state_mapping entry:

```json
"state_mapping": {
  "query": "params.query",
  "search_results": "steps.search.output"
}
```

## Fixing Auto-Generated Tool Outputs

Tools created by `relevance_create_tool_from_transformation` often return empty `{}` because output fields aren't mapped. Call `relevance_get_transformation` for the underlying transformation and read its `output_schema` to find the available fields. Then call `relevance_update_tool` with the corrected step config: explicit `output` mappings on each step (`"output": { "field": "{{field}}" }`), the final output reference pointing at a real field on the step (`"answer": "{{stepname.actual_field}}"`), and matching `state_mapping` entries so the templates resolve. When in doubt, find a working tool that uses the same transformation and copy its output pattern.

## Discover Transformation Outputs via `output_schema`

**Don't guess output fields - check the transformation's `output_schema` first!**

```typescript
const t = await relevance_get_transformation({
  transformationId: 'browserless_scrape',
});
// t.output_schema.properties.output.properties.page  → use {{output.page}}
// t.output_schema.properties.credits_cost            → use {{credits_cost}}
```

## Using Secrets in Code Steps

Secrets are accessed via **template syntax** `{{secrets.secret_name}}`, NOT as JavaScript/Python objects.

```javascript
// WRONG - secrets is not a defined JS object
const apiKey = secrets.my_api_key; // Error: secrets is not defined

// CORRECT - template substitution (resolved before code runs)
const apiKey = '{{secrets.my_api_key}}';
```

**Secret names MUST start with `chains_` prefix.** If you reference `{{secrets.my_key}}`, the secret must be named `chains_my_key` in the project.

**Alternative:** Pass credentials as tool input parameters with hidden defaults to avoid the `chains_` prefix:

```json
{
  "params_schema": {
    "properties": {
      "_api_key": { "type": "string", "default": "sk-...", "hidden": true }
    }
  }
}
```

Then use template substitution: `api_key = '{{_api_key}}'`

**For API calls**, prefer transformations with built-in `authorization_config` (e.g., `hubspot_api_call`) which handle auth automatically.

---

## Step Naming Rules

- **Never prefix step names with `steps.`** -- the namespace is added automatically by the template resolver
- Step names should be plain identifiers: `calc_date`, `fetch_contacts`, `format_output`
- `steps.` belongs only in REFERENCES: `{{steps.calc_date.output}}` or state_mapping values like `"steps.calc_date.output"`
- If the UI shows `steps.calc_date` as the output variable, the actual step name is just `calc_date`

```json
// WRONG - double-prefixed, breaks references
{ "name": "steps.search", ... }

// CORRECT - plain identifier
{ "name": "search", ... }
```

## Template Injection Size Limit

Template injection (`{{steps.stepName.output}}`) has a size limit (~5-10KB). If a previous step returns a large response (e.g., batch API results with HTML bodies), the injection truncates silently, causing `JSON.parse()` to fail.

**Symptoms:** Step reports 0 results even though the API call succeeded. Same API call works in a standalone tool.

**Workaround:** Split into a list tool (returns IDs/metadata only) + a reader tool (reads one record at a time). This keeps each response small enough for template injection.

## Best Practices

1. **Use descriptive step names** - Plain identifiers, never prefix with `steps.`
2. **Map outputs explicitly** - Don't rely on implicit variable names
3. **Test incrementally** - Add one step at a time
4. **Document with prompt_description** - Help AI know when to use the tool
5. **Handle errors in Python steps** - Add try/catch for robustness
6. **Validate every tool before attaching to agents** - Test with `relevance_run_tool` to catch empty output issues
7. **Use backtick template literals in JS steps** - Never use single quotes (`'{{param}}'`), which break with apostrophes in values
