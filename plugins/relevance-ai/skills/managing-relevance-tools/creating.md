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

> Validate a newly created tool that only reads with `relevance_trigger_tool` (then `relevance_poll_tool_result`). A step writes when it sends or posts anything, or adds, changes or deletes data anywhere, a knowledge table included, since it may hold live data. Judge a step by what it does, not its name: a find-or-create step writes, and so can a search sent as a POST. A step you can't classify counts as one that writes. Never run something that writes just to try it, and that includes checking a fix: trigger the agent or workforce with `simulate: true`, naming in `live_action_ids` only tools that read, so every other tool returns a stand-in instead of running; or exercise the write in an eval that stands in for it; or run it only once the user has agreed to that specific run and named where it may write, such as a test recipient. After any run, a finished tool call whose `output_source` is missing or `tool-run` ran for real: say which of those sent or changed something, and where. A stand-in shows what it would send, not that it arrives, so call that write unproven. Running a tool directly has no `simulate` option, so for a tool that writes, test its read steps with `relevance_run_transformation`, and set up the eval's stand-in as [relevance-evals](../relevance-evals/simulation.md) describes.

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

Inside loops, reference the current item as `{{foreach.item}}` (and its fields as `{{foreach.item.<field>}}`). A bare `{{item}}` does **not** resolve — it silently becomes the literal string `"undefined"`.

```typescript
params: {
  url: '{{foreach.item.url}}';
}
```

## Multi-Step Example

```typescript
{
  studio_id: "search-and-shortlist",
  title: "Search and Shortlist",
  params_schema: {
    type: "object",
    properties: {
      topic: { type: "string", title: "Topic", description: "What to search the web for" }
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
        name: "shortlist",
        transformation: "js_code_transformation",
        params: {
          code: `
const all = steps.search.output.results || [];
const results = all.slice(0, 5).map((r) => ({ title: r.title, link: r.link, snippet: r.snippet }));
return { results, scanned: all.length, kept: results.length };
`
        }
      }
    ],
    output: {
      results: "{{steps.shortlist.output.transformed.results}}",
      scanned: "{{steps.shortlist.output.transformed.scanned}}",
      kept: "{{steps.shortlist.output.transformed.kept}}"
    }
  }
}
```

No summarise step: the agent that calls this tool reads the results and writes the answer from its system prompt (see [Tools vs Agents](SKILL.md#tools-vs-agents-what-goes-where)).

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

### Never declare a JSON document as a string

If a parameter holds structured data, type it `array` or `object` with an `items` / `properties` schema. Do **not** declare it `type: "string"` and describe it as "a JSON array" — that forces a calling agent to escape a whole document by hand and count its brackets, and one brace out of place still arrives as a valid argument and blows up in the step that parses it.

```typescript
// CORRECT — the caller emits real JSON, which it cannot mis-escape
selections: {
  type: "array",
  title: "Selections",
  items: { type: "object", properties: { id: { type: "number" } } }
}

// WRONG — the caller must hand-escape a document inside a string
selections_json: { type: "string", title: "Selections (JSON array string)" }
```

A code step reading it needs no change either way: `{{params.selections}}` renders an array or object as JSON text, so `json.loads(r'''{{params.selections}}''')` keeps working.

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
const { job_id } = await relevance_trigger_tool({
  studio_id: 'my-tool',
  params: { query: 'test query' },
});
relevance_poll_tool_result({ studio_id: 'my-tool', job_id, wait_seconds: 50 });
```

> **After calling `relevance_set_tool_output`, always re-run the draft and inspect the returned `output` object** to confirm the top-level keys match what you declared. See [outputs.md](outputs.md) for the full model.

## Publishing

Both create and update save to a DRAFT — publishing is always a separate, user-confirmed step:

- **Creating a new tool** (`relevance_create_tool`): saves to a draft. The new tool has only a draft version; call `relevance_publish_tool` to make it live.
- **Updating an existing tool** (`relevance_update_tool`): saves to a draft only — the live version is unchanged until you call `relevance_publish_tool`.

`relevance_publish_tool` is irreversible, so confirm in chat before calling it. Always pass a concise `version_description` (and `version_name`) — a clear, one-line summary of what changed and why, easy to understand (see [Version Management](versions.md)):

```typescript
relevance_publish_tool({
  tool_id: 'my-tool',
  version_name: 'Add Slack notification step',
  version_description:
    'Posts a Slack message to #sales after the lead is enriched.',
});
```

Test a draft that only reads with `relevance_trigger_tool` and `version: "draft"` (one that writes follows the validation rule at the top of this page):

```typescript
const { job_id } = await relevance_trigger_tool({
  studio_id: 'my-tool',
  params: { query: 'test query' },
  version: 'draft',
});
relevance_poll_tool_result({ studio_id: 'my-tool', job_id, wait_seconds: 50 });
```

---

## Editing Transformation Steps

To edit transformation steps, use `relevance_update_tool_steps` (saves to DRAFT only). It takes every edit to one tool in one call, applies the `ops` in order and saves once. If any op fails, nothing is saved.

```typescript
// Rename step 1, update the step that reads it, and drop step 3, in one save.
relevance_update_tool_steps({
  studio_id: 'my-tool',
  ops: [
    { op: 'update', step_index: 1, patch: { name: 'web_search' } },
    {
      op: 'update',
      step_index: 2,
      patch: {
        params: { prompt: 'Summarize: {{steps.web_search.output.results}}' },
      },
    },
    { op: 'remove', step_index: 3 },
  ],
});
```

All indices are 0-based, and each one refers to the step array as it stands after the earlier ops in the same call:

- `{ op: 'add', step, position? }` — insert a new step. Omit `position` (or pass a value `>= step_count`) to append.
- `{ op: 'update', step_index, patch }` — merges `params`/`output`/`output_schema` one level deep; other fields replace. `null` clears a field other than `name` and `params` (`output: null` removes the step's mapping); inside `params` it removes that param (`{ params: { thinking: null } }`).
- `{ op: 'remove', step_index }` — remove a single step.
- `{ op: 'move', from_index, to_index }` — reorder a step.

Pass `output` / `output_schema` alongside `ops` to set the tool's outputs in the same save (same semantics as `relevance_set_tool_output`). The response echoes each added or updated step under `edited` — its final index, `name`, `transformation`, `if`, `output` and a summary of its `params`. Confirm the edit from that echo before running the tool. Long text and nested values (such as a loop's inner steps) are echoed as `<…>` markers; read those back with `relevance_get_tool`.

For a single edit, the singular tools remain: `relevance_add_tool_step({ studio_id, step, position? })`, `relevance_update_tool_step({ studio_id, step_index, patch })`, `relevance_remove_tool_step({ studio_id, step_index })` and `relevance_move_tool_step({ studio_id, from_index, to_index })` behave like the matching op.

> **⚠️ Never issue several step-edit calls to the same tool at once — the last save wins.** Each call loads the draft, edits it and saves the whole draft, so concurrent calls silently discard each other's edits. Put every edit to one tool in one `relevance_update_tool_steps` call, or make singular calls one after another.

Always call `relevance_get_tool` first to confirm step indices in the current draft. Out-of-range indices throw with the current step count in the error message, so the LLM can recover.

## Renaming or restructuring a step (update references!)

> **⚠️ Renaming a step does NOT rewrite the references that point at it.** A step edit only touches the step you name — it does not cascade. A step named `search` is read elsewhere as `{{steps.search.output.<field>}}`; if you rename it to `web_search` (or change which keys its `output` exposes), every one of those references silently becomes a dangling `{{steps.search.output.…}}` that is flagged as "doesn't exist" and that fails at runtime. This is the most common edit-time regression.

**Whenever you change a step's `name`, or change the keys in its `output` mapping, fix every reference before running the draft — later steps and the tool's `output` map in the same `relevance_update_tool_steps` call, `state_mapping` with `relevance_update_tool` right after:**

1. **Find every reference.** Call `relevance_get_tool` and scan, for the old name/field:
   - every **later** step's `params` (templates like `{{steps.<old_name>.output.…}}`),
   - the tool-level runtime `output` map (set via `relevance_set_tool_output`),
   - `state_mapping` values (bare JSONPaths like `steps.<old_name>.output.…`, no curly braces).
2. **Update every occurrence** to the new name/field — do not stop at the first hit.
3. **Re-run the draft** (`relevance_trigger_tool({ studio_id, version: 'draft' })`, then `relevance_poll_tool_result`) and confirm there are no "doesn't exist" / "Variable not found" errors before publishing. For a tool that writes, check the steps before the write with `relevance_run_transformation` instead.

```typescript
// You rename step "search" → "web_search" with an `update` op.

// ❌ WRONG — the downstream reference still points at the old name and now dangles:
{
  name: "summarize",
  params: { prompt: "Summarize: {{steps.search.output.results}}" } // "search" no longer exists
}

// ✅ RIGHT — the reference was updated in the same call:
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

**JS code steps** have native access to `steps` and `params` globals at runtime (e.g., `steps.search.output`, `params.topic`). Prefer them over template injection inside JS: `{{...}}` pastes raw text into the code, so a value containing a backtick or `${` breaks out of its string.

**Non-JS steps** (`prompt_completion`, `api_call`, etc.) reach studio inputs and step outputs via template substitution: `{{params.X}}` for studio inputs, `{{steps.X.output.Y}}` for previous-step outputs. No `state_mapping` entry is needed for either.

## Fixing Empty Tool Outputs

A tool whose stored config carries an explicit empty output map (`"output": {}`) returns `{}` at run time — the empty map replaces the step's real output, and a tool-level empty map makes the whole tool return `{}`. Never write one: creating a tool, adding a step, updating a step or setting the tool's outputs drops any empty `output` map the call sends (including on steps nested in a loop or branch) and returns a warning naming it. An empty map already stored on a step stays until you clear it. Omit `output` to keep the current mapping; to clear one, patch the step (or set the tool's outputs) with `output: null`.

Fix in two steps, or use the same two steps to deliberately narrow what a tool returns:

1. **Map each step's outputs.** Call `relevance_get_transformation` for the underlying transformation, look up its available fields, then set an explicit `output: { "<field>": "{{<field>}}" }` mapping on each step — one `update` op per step in a single `relevance_update_tool_steps` call.
2. **Declare the tool's runtime output.** Call `relevance_set_tool_output` (or pass them in that same `relevance_update_tool_steps` call) with the paired fields: `output: { answer: '{{steps.<step_name>.output.<actual_field>}}', ... }` and `output_schema: { metadata: { field_order: ['answer', ...] }, properties: { answer: { metadata: { render_mode: 'markdown' } } } }`.

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

Work down this ladder and stop at the first rung that fits — most tools need no credential of their own.

1. **Calling the Relevance platform API — there is no key to supply.** Use a `relevance_api_call` step; auth is auto-injected with the run's own key, which is also what makes the tool survive cloning and marketplace publishing. See [relevance_api_call](transformations.md#relevance_api_call) (GET/POST/PUT, relative `path`). A **Python** code step is likewise already authenticated by the sandbox. A **JS** code step is **not** — a JS step that needs the platform API must hand the call to a `relevance_api_call` step rather than build its own request.
2. **Third-party API — prefer a transformation with built-in `authorization_config`** (`hubspot_api_call`, `slack_api_call`, …). It resolves auth server-side, so the tool definition holds nothing.
3. **OAuth account** — a `params_schema` input with `metadata.content_type: "oauth_account"`, passed to the step as `oauth_account_id: "{{params.<name>}}"`. See [oauth.md](oauth.md).
4. **API key input** — a `params_schema` input with `metadata.content_type: "api_key"` and `metadata.project_key_name`. This renders the key-vault picker, not a free-text box.
5. **`{{secrets.chains_*}}`** — last resort, for a key the user has already stored in the project themselves.

> **❌ Never store a credential as a param `default`, and never inline a raw key into a code step.** `hidden` is not a supported schema field — it is silently ignored, so nothing is hidden. A credential in a param default is sent to the model on every turn, written to run history, readable by anyone with viewer access to the project, exposed by public share links, and copied on clone or marketplace publish. Unlike `{{secrets.*}}`, it is never redacted from step output.

> **❌ Never ask the user to paste, type, or send a key or token in the conversation.** If a step genuinely needs a key the project doesn't have, call `relevance_check_tool_integration_requirements` (or `relevance_check_api_key_availability`) and give the user the `setup_url` it returns — they add it on the Integrations page, where it is stored encrypted and never handed back.

### Referencing a stored secret

Secrets are accessed via **template syntax** `{{secrets.secret_name}}`, NOT as JavaScript/Python objects.

```javascript
// WRONG - secrets is not a defined JS object
const apiKey = secrets.chains_my_api_key; // Error: secrets is not defined

// CORRECT - template substitution (resolved before code runs)
const apiKey = '{{secrets.chains_my_api_key}}';
```

**Secret names MUST start with `chains_` prefix.** Reference it by its full name, `{{secrets.chains_my_key}}`; a name without the prefix fails. The prefix is a security boundary, not a naming chore — it stops a tool from reading platform-managed keys. Never route around it.

**Secrets resolve only inside a saved tool's step config**, never in run params: `relevance_run_transformation` and `relevance_trigger_tool` refuse params that contain `{{secrets.chains_*}}`. To test a step that needs a secret, put the reference in the tool's steps, then trigger the tool.

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
6. **Validate every tool before attaching to agents** - Run one that only reads with `relevance_trigger_tool` (then `relevance_poll_tool_result`) to catch empty output issues; for one that writes, probe its read steps with `relevance_run_transformation` and follow the validation rule at the top of this page.
7. **Use backtick template literals in JS steps** - Never use single quotes (`'{{param}}'`), which break with apostrophes in values
