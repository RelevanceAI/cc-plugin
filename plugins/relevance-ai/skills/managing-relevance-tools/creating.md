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

  transformations: {
    steps: TransformationStep[]; // Sequential steps to execute
    output?: Record<string, string> | null; // Tool's runtime output map — see outputs.md
  };

  output_schema?: Record<string, unknown>; // Display contract paired with `transformations.output` — see outputs.md for shape
  state_mapping?: Record<string, string>; // How data flows
}
```

## Transformation Step Structure

```typescript
interface TransformationStep {
  name: string; // Unique identifier for this step
  transformation: string; // Transformation type ID (e.g. "prompt_completion", "markdown")
  params: Record<string, string>; // Input params (use {{variable}} templating)
  output?: Record<string, string>; // Output mapping

  // Optional control flow
  foreach?: { iterator: string; item_key: string };
  if?: string;
}
```

> **Document multi-step tools as you build them.** Any tool with three or more steps, or any tool with non-obvious logic, MUST include at least one `markdown` step (a native side-effect-free step that renders rich text between real steps). See [documentation.md](documentation.md) for the rubric on when, what, and where to add notes — this is the single biggest lever for making the tools you build readable.

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
        params: { search_query: '{{params.query}}' },
        output: { results: '{{organic}}' },
      },
    ],
  },
});
```

## Variable Templating

### Input Parameters

Inside a step's `params`, reference a studio-level input as `{{params.<name>}}` — the `params.` prefix is required. Bare `{{name}}` does **not** resolve to a studio input and is the canonical "doesn't exist" reference bug.

```typescript
params: {
  search_query: "{{params.query}}",   // CORRECT — studio input declared in params_schema
  num_results: "{{params.count}}"
}

// WRONG — bare names do not resolve to studio inputs;
// they are flagged as "doesn't exist" references and fail at runtime.
params: {
  search_query: "{{query}}"
}
```

This applies anywhere a step interpolates a studio input: `params` values, prompts inside `prompt_completion`, URLs in `api_call`, JS/Python source in code steps. If you find yourself wanting to drop the `params.` prefix, you are looking at the bug.

### Previous Step Outputs

Reference a previous step's output as `{{steps.<step_name>.output.<field>}}`:

```typescript
// Step 1 named "search" — runs serper_google_search, raw response contains `organic`.
// Step 2 reads it via the full path:
params: {
  text: '{{steps.search.output.organic}}';
}
```

If you set an explicit `output` mapping on a step (e.g. `output: { results: "{{organic}}" }`), the alias is available at `{{steps.search.output.results}}` instead.

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
        params: { search_query: "{{params.topic}}" },
        output: { results: "{{organic}}" }
      },
      {
        name: "summarize",
        transformation: "prompt_completion",
        params: {
          prompt: "Summarize these results about {{params.topic}}:\n\n{{steps.search.output.results}}"
        },
        output: { summary: "{{answer}}" }
      }
    ]
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
  if: "{{params.include_analysis}}",  // Only run if truthy
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

> **After calling `relevance_set_tool_output`, always re-run the draft and inspect the returned `output` object** to confirm the top-level keys match what you declared. See [outputs.md](outputs.md) for the full model.

## Publishing

Both create and update save to a DRAFT — publishing is always a separate, user-confirmed step:

- **Creating a new tool** (`relevance_create_tool`): saves to a draft. The new tool has only a draft version; call `relevance_publish_tool` to make it live.
- **Updating an existing tool** (`relevance_update_tool`): saves to a draft only — the live version is unchanged until you call `relevance_publish_tool`.

`relevance_publish_tool` always shows an approval card to the user (even with auto-approve enabled), so confirm in chat before calling it. Always pass a concise `version_description` (and `version_name`) — a clear, one-line summary of what changed and why, easy to understand (see [Version Management](versions.md)):

```typescript
relevance_publish_tool({
  tool_id: 'my-tool',
  version_name: 'Add Slack notification step',
  version_description:
    'Posts a Slack message to #sales after the lead is enriched.',
});
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
- `relevance_move_tool_step({ studio_id, from_index, to_index })` — reorder a step (both 0-based).

Always call `relevance_get_tool` first to confirm step indices in the current draft. Out-of-range indices throw with the current step count in the error message, so the LLM can recover.

## Renaming or restructuring a step (update references!)

> **⚠️ Renaming a step does NOT rewrite the references that point at it.** The per-step tools only touch the step you name — they do not cascade. A step named `search` is read elsewhere as `{{steps.search.output.<field>}}`; if you rename it to `web_search` (or change which keys its `output` exposes), every one of those references silently becomes a dangling `{{steps.search.output.…}}` that is flagged as "doesn't exist" and that fails at runtime. This is the most common edit-time regression.

**Whenever you change a step's `name`, or change the keys in its `output` mapping, do this in the same edit batch:**

1. **Find every reference.** Call `relevance_get_tool` and scan, for the old name/field:
   - every **later** step's `params` (templates like `{{steps.<old_name>.output.…}}`),
   - the tool-level runtime `output` map (set via `relevance_set_tool_output`),
   - `state_mapping` values (bare JSONPaths like `steps.<old_name>.output.…`, no curly braces).
2. **Update every occurrence** to the new name/field — do not stop at the first hit.
3. **Re-run the draft** (`relevance_run_tool({ studio_id, version: 'draft' })`) and confirm there are no "doesn't exist" / "Variable not found" errors before publishing.

```typescript
// You rename step "search" → "web_search" via relevance_update_tool_step.

// ❌ WRONG — the downstream reference still points at the old name and now dangles:
{
  name: "summarize",
  params: { prompt: "Summarize: {{steps.search.output.results}}" } // "search" no longer exists
}

// ✅ RIGHT — the reference was updated in the same batch:
{
  name: "summarize",
  params: { prompt: "Summarize: {{steps.web_search.output.results}}" }
}
```

> If a downstream reference would be expensive to chase, prefer **not** renaming — the step name is internal plumbing, and a clear `display_name` gives the human-readable label without breaking refs.

---

## `state_mapping` (optional aliasing)

`state_mapping` is **not required** for normal tool authoring. `{{params.X}}` and `{{steps.X.output.Y}}` resolve directly inside a step's `params` — you do not need a `state_mapping` entry to make them work.

`state_mapping` exists only as an aliasing layer for the studio-level tool output (configured via `relevance_set_tool_output`) or for legacy tools that surface short aliases. If you don't need either, omit `state_mapping`.

### Reference table

| Where                            | Correct form                                       | Notes                                                        |
| -------------------------------- | -------------------------------------------------- | ------------------------------------------------------------ |
| `params_schema` property names   | `"search_query": { "type": "string" }`             | Plain key, no prefix.                                        |
| `transformations.steps[].params` | `"search_query": "{{params.search_query}}"`        | Studio inputs ALWAYS use `{{params.<name>}}`.                |
| `transformations.steps[].params` | `"text": "{{steps.search.output.results}}"`        | Step outputs use the full `steps.<name>.output.*` path.      |
| `state_mapping` keys (if used)   | `"search_query"`                                   | Plain alias, no prefix, no curly braces.                     |
| `state_mapping` values (if used) | `"params.search_query"` or `"steps.search.output"` | JSONPath into internal state — **never** wrapped in `{{ }}`. |

### When you do use `state_mapping`

```json
// CORRECT — values are bare JSONPaths, no curly braces
"state_mapping": {
  "answer": "steps.summarize.output.summary"
}

// WRONG — curly braces inside state_mapping values break resolution
"state_mapping": {
  "answer": "{{steps.summarize.output.summary}}"
}
```

### JS code steps vs non-JS steps

**JS code steps** have native access to a `steps` global object at runtime (e.g., `steps.search.output`). Prefer the native `steps` object over template injection for complex data inside JS.

**Non-JS steps** (`prompt_completion`, `api_call`, etc.) reach studio inputs and step outputs via template substitution: `{{params.X}}` for studio inputs, `{{steps.X.output.Y}}` for previous-step outputs. No `state_mapping` entry is needed for either.

## Fixing Auto-Generated Tool Outputs

Tools created by `relevance_create_tool_from_transformation` often return empty `{}` because output fields aren't mapped. Fix in two steps:

1. **Map each step's outputs.** Call `relevance_get_transformation` for the underlying transformation, look up its available fields, then `relevance_update_tool_step` to set an explicit `output: { "<field>": "{{<field>}}" }` mapping on each step.
2. **Declare the tool's runtime output.** Call `relevance_set_tool_output` with the paired fields: `output: { answer: '{{steps.<step_name>.output.<actual_field>}}', ... }` and `output_schema: { metadata: { field_order: ['answer', ...] }, properties: { answer: { metadata: { render_mode: 'markdown' } } } }`.

When in doubt, find a working tool that uses the same transformation and copy its output pattern.

## Discover Transformation Outputs

**Don't guess output fields — inspect the transformation first.**

```typescript
const t = await relevance_get_transformation({
  transformationId: 'browserless_scrape',
});
// Inspect `t` to see the field names the transformation actually returns,
// then reference them as `{{steps.<step_name>.output.<field>}}` from later
// steps or from `relevance_set_tool_output`.
```

## Check auth requirements before wiring a step

**Before** wiring a step whose transformation hits a third-party API (HubSpot, Gmail, Slack, Ahrefs, …), check what auth it needs:

```typescript
relevance_get_transformation_requirements({
  transformation_id: 'hubspot_-_create-contact',
});
// Reports the OAuth providers / API keys the step requires.
```

If it reports an **OAuth** requirement, the account MUST be declared as a proper OAuth input on the tool — a `params_schema` property with `metadata.content_type: "oauth_account"` — and passed into the step as `oauth_account_id: "{{params.<name>}}"`.

> **❌ Do NOT declare the account as a plain `type: "string"` text input, and never paste a raw account id into the step.** A plain string renders a free-text box (not the account selector), so the step receives an un-credentialed string and every downstream call fails. See [oauth.md](oauth.md) for the full pattern and a WRONG→RIGHT example.

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

Then use template substitution: `api_key = '{{params._api_key}}'`

**For API calls**, prefer transformations with built-in `authorization_config` (e.g., `hubspot_api_call`) which handle auth automatically.

---

## Step Naming Rules

- **Never prefix step names with `steps.`** -- the namespace is added automatically by the template resolver
- Step names should be plain identifiers: `calc_date`, `fetch_contacts`, `format_output`
- `steps.` belongs only in REFERENCES: `{{steps.calc_date.output.<field>}}`
- If the UI shows `steps.calc_date` as the output variable, the actual step name is just `calc_date`
- **Renaming an existing step is safe only if you also update every `{{steps.<old_name>…}}` reference** — see [Renaming or restructuring a step](#renaming-or-restructuring-a-step-update-references).

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
