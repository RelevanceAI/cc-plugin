---
title: Transformations Reference
description: Implementation reference for transformations — input/output patterns, parameter schemas, gotchas per type, and worked examples for `prompt_completion`, `browserless_scrape`, `api_call`, code steps, and more. Load when the catalog entry isn't enough.
---

# Transformations Reference

Complete reference for transformation types and their outputs.

## Use the typed Relevance tools — never the raw API for transformations

For anything transformation-related, reach for the typed tools. They validate, they return full schemas, and they short-circuit the failure mode where the agent guesses parameter names from convention.

| What you want                                                              | Use this tool                               |
| -------------------------------------------------------------------------- | ------------------------------------------- |
| Browse available steps (with filters by integration / use_case / category) | `relevance_list_transformations`            |
| Inspect a step's full input/output schema before configuring params        | `relevance_get_transformation`              |
| Test a step in isolation with candidate params before saving a tool        | `relevance_run_transformation`              |
| Check whether the OAuth account / API key needed by a step is connected    | `relevance_get_transformation_requirements` |
| Wrap a step as a standalone tool                                           | `relevance_create_tool_from_transformation` |

Every transformation-related need has a typed tool above — if you find a gap, file feedback rather than working around it.

If `relevance_get_transformation` returns a response where `input_schema` is missing, or `input_schema.properties` is empty, surface that to the user. Do not guess parameter names from external API conventions — the schema being absent is a signal, not an invitation.

## Step output vs tool output (two different things)

Two layers of "output" exist and they map to different fields and different tools — don't conflate them.

| Layer                   | Edited via                                                             | Purpose                                                                                                                                       |
| ----------------------- | ---------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **Step output (alias)** | `relevance_update_tool_step` (patch the step's `output`)               | Maps a step's transformation output to state variables that later steps reference (e.g. `output: { text: "{{answer}}" }`). Local to one step. |
| **Tool output**         | `relevance_set_tool_output` (paired fields `output` + `output_schema`) | Configures what the tool returns at runtime (`output`) and how it's displayed (`output_schema`). See [outputs.md](outputs.md).                |

Steps go through `relevance_add_tool_step` / `relevance_update_tool_step` / `relevance_remove_tool_step` / `relevance_move_tool_step`; outputs go through `relevance_set_tool_output`.

`relevance_update_tool_step` patches the user-editable fields of a step: `name`, `display_name`, `params`, `output`, `output_schema`, `if`, `foreach`, `continue_on_error`, `use_fallback_on_skip`, `invent_instructions`, `default_output_values`. Of those, `params`, `output`, and `output_schema` merge one level deep — a patch like `{ params: { model: "X" } }` preserves sibling keys (`prompt`, `system_prompt`, `temperature`). Other fields replace wholesale.

Other step fields (`transformation`, `version`, `version_metadata`, `saved_params`, `parent_step`, `branch_id`) are not patchable through this tool. To change a step's transformation type or position, use `relevance_add_tool_step` / `relevance_remove_tool_step` / `relevance_move_tool_step`.

## Picking a step type: the ladder

Before you ask _which_ transformation, decide _what kind_ of step. Work down this ladder and stop
at the first rung that fits:

1. **A dedicated step for that action** — e.g. `send_email_gmail`, `create_google_calendar_event_v2`.
2. **The provider's generic authenticated API-call step** — `google_api_call_v2`, `slack_api_call`,
   `hubspot_api_call`, … It handles auth for you, so it reaches any endpoint the connected
   account's scopes permit, including actions with no dedicated step. **Its parameters differ per
   provider** — most take a relative `path`, a few take a full `url`, some need an extra tenant
   param — so call `relevance_get_transformation` before setting `params`. See
   [the per-provider API-call catalogue](transformations-catalog.md#per-provider-api-call-steps--the-escape-hatch-for-theres-no-step-for-that).
3. **A tool that already does it, run as a step** — via [`run_chain`](#run_chain) (see below).
4. **The generic `api_call` step** — when the provider has no integration at all and you supply auth yourself.
5. **A code step** — last resort, and for reshaping data between steps rather than calling the
   service itself.

### Names don't follow the provider's API verbs

Search by concept, not by the provider's method name. The Google Calendar _read_ step is called
"Check Google calendar availability" (`check_google_calendar_v2`) — searching "list events" finds
nothing. When a name-based search comes up empty, browse the tag instead:
`relevance_list_transformations({ integration: "Google" })`.

### Searching: single keyword first

`search` matches **every word** of the query against a step's name and description. A longer query
returns _fewer_ results, not better ones — `"get events google calendar"` returns nothing while
`"calendar"` returns the set. So:

1. Start with **ONE** keyword.
2. If that's still wrong, drop `search` and browse by `integration` or `use_case` — these return
   the whole tagged set. Every result echoes back its own `integrations` / `use_cases` values, so
   one call teaches you the real filter vocabulary.
3. Still nothing? Work rungs 2–4 of the ladder above before telling the user it doesn't exist.

**Zero results means your query was probably wrong, not that the step is missing.** And the
transformation list is only one of three catalogues — see [`run_chain`](#run_chain).

## Picking a transformation: native first

Before consulting the per-type sections below, decide **which** transformation to use. `relevance_list_transformations` returns results partitioned into `native` and `pipedream` arrays:

1. **`native`** — built and maintained by Relevance AI. Always prefer these.
2. **`pipedream`** — third-party integrations (IDs shaped like `{provider}_-_{action}`). Use only when no native equivalent exists.

Quick tells you are about to pick a third-party entry:

- The `transformation_id` contains `_-_` (e.g. `jira_-_jira-create-issue`, `sharepoint_-_create-folder`).
- The result is in the `pipedream` array of the search response.

Quick tells you have the native version:

- The `transformation_id` is a clean snake_case name (e.g. `jira_native_create_issue`, `zendesk_native_create_ticket`, `linear_create_ticket`).
- The result is in the `native` array of the search response.

When in doubt, browse with `relevance_list_transformations({ integration: "<Provider>" })` — a single keyword or a tag filter, not a phrase (see [Searching](#searching-single-keyword-first)) — and inspect both arrays before choosing. Never silently fall back to a third-party entry — if no native equivalent exists, surface that to the user (e.g. "There's no native action for this, but a third-party one exists — want me to use it?"). When referring to a `pipedream` entry in user-facing prose, call it a "third-party" integration.

## Documentation

### markdown

A native, inline, side-effect-free step that renders rich markdown between real steps in the Relevance app UI. Use it to document multi-step tools so users can understand the chain at a glance. No output — never reference it from a downstream step.

```typescript
{
  name: "overview",
  transformation: "markdown",
  params: {
    markdown: "### What this tool does\nTakes **{{params.topic}}** and returns a summarised brief..."
  },
  output: {}
}
```

**Output:** none.

**When to insert these steps**, what to write in them, and the style rules live in [documentation.md](documentation.md). The short version: add a header note as step 0 for any tool with ≥3 steps or non-obvious logic, and an inline note before a branch (`if:`), loop (`foreach`), non-obvious data reshape, or an LLM step whose prompt encodes non-obvious logic. **Do not** add one per step — over-noting is worse than under-noting.

## LLM / Text Processing

### prompt_completion

Text generation with LLM.

```typescript
{
  name: "analyze",
  transformation: "prompt_completion",
  params: {
    prompt: "Analyze this: {{params.input}}"
  },
  output: {
    result: "{{answer}}"  // NOT {{text}}!
  }
}
```

**Output:** `{{answer}}`

**Models:** default to `relevance-cost-optimized` (or `relevance-performance-optimized` for the hardest steps) — these auto-select a current model and never go stale. To pin a specific one, call `relevance_list_llm_models` and pick an id that is not flagged deprecated or retired; never guess an id from memory. When the step has a shape worth optimising for — high-volume extraction, strict JSON output, long documents, image input — load the `relevance-llm-models/SKILL` guide first.

### prompt_completion_vision

Image analysis with LLM.

```typescript
{
  name: "analyze_image",
  transformation: "prompt_completion_vision",
  params: {
    prompt: "Describe this image",
    image: "{{params.image_url}}"
  },
  output: {
    description: "{{answer}}"
  }
}
```

## Web Scraping & Search

### browserless_scrape

Scrape webpage content.

```typescript
{
  name: "scrape",
  transformation: "browserless_scrape",
  params: {
    website_url: "{{params.url}}",
    method: "Text"
  },
  output: {
    content: "{{output.page}}"  // NOT {{content}}!
  }
}
```

**Output:** `{{output.page}}`

### serper_google_search

Google web search.

```typescript
{
  name: "search",
  transformation: "serper_google_search",
  params: {
    search_query: "{{params.query}}",
    num: 10
  },
  output: {
    results: "{{organic}}"
  }
}
```

**Output:** `{{organic}}`, `{{knowledge_graph}}`

### jina_reader

Clean web content for LLM.

```typescript
{
  name: "clean",
  transformation: "jina_reader_-_jina_reader-convert-to-llm-friendly-input",
  params: {
    url: "{{params.url}}"
  },
  output: {
    text: "{{text}}"
  }
}
```

## Apify Actors

### run_apify_dynamic

Run any Apify actor (2000+ scrapers).

```typescript
{
  name: "scrape_maps",
  transformation: "run_apify_dynamic",
  params: {
    actor_id: "compass/crawler-google-places",
    inputs: {
      searchStringsArray: ["{{params.query}}"],
      maxCrawledPlaces: 20
    }
  },
  output: {
    places: "{{items}}"
  }
}
```

**Output:** `{{items}}`

**Popular actors:**

- `compass/crawler-google-places` - Google Maps
- `apify/google-search-scraper` - Deep Google search
- `apify/instagram-scraper` - Instagram data
- `apify/linkedin-scraper` - LinkedIn profiles
- `apify/website-content-crawler` - Full website crawl
- `apify/contact-info-scraper` - Find emails/phones

## API Calls

### api_call

Make HTTP requests to **external** APIs. No auth injection — you must pass headers manually.

```typescript
{
  name: "fetch_data",
  transformation: "api_call",
  params: {
    url: "https://api.example.com/data",
    method: "GET",
    headers: { "Authorization": "Bearer {{params.api_key}}" }
  },
  output: {
    result: "{{response_body}}"
  }
}
```

**Output:** `{{response_body}}`, `{{status}}`

### relevance_api_call

Make HTTP requests to **Relevance platform** APIs. **Auth is auto-injected** from the caller's context — no API keys needed. This is critical for marketplace tools: cloners get their own auth automatically.

> **⚠️ Never hand-roll Relevance auth.** To reach the platform API from a tool, always use this step — never an `api_call` step with a manual `Authorization` header, and never a code step carrying a Relevance key. There is no key to paste, and a hardcoded one breaks the moment the tool is cloned or published.

```typescript
{
  name: "fetch_collections",
  transformation: "relevance_api_call",
  params: {
    path: "/replicate/collections",   // Relative path (not full URL)
    method: "GET"                      // GET, POST, or PUT only
  },
  output: {
    collections: "{{response_body.collections}}"
  }
}
```

**Output:** `{{response_body}}`, `{{status}}`, `{{url}}`, `{{body}}`, `{{response_headers}}`

**Parameters:**

| Param                           | Type          | Required | Description                                                                                  |
| ------------------------------- | ------------- | -------- | -------------------------------------------------------------------------------------------- |
| `path`                          | string        | Yes      | Relative API path (e.g., `/replicate/collections`, `/knowledge/list`). Leading `/` optional. |
| `method`                        | string (enum) | Yes      | `"GET"`, `"POST"`, or `"PUT"` (no DELETE/PATCH)                                              |
| `body`                          | object        | No       | JSON body for POST/PUT requests. Ignored for GET.                                            |
| `raise_error_on_error_response` | boolean       | No       | If `true`, step fails on non-2xx. Default `false` (returns error in `response_body`).        |

**POST example with body:**

```typescript
{
  name: "update_metadata",
  transformation: "relevance_api_call",
  params: {
    path: "/knowledge/sets/{{params.conversation_id}}/update_metadata",
    method: "POST",
    body: {
      updates: { conversation: { tags: "{{params.tags}}" } }
    }
  },
  output: {
    result: "{{response_body}}"
  }
}
```

**Key differences from `api_call`:**

| Feature          | `relevance_api_call`                  | `api_call`                      |
| ---------------- | ------------------------------------- | ------------------------------- |
| Target           | Relevance platform APIs only          | Any external URL                |
| Auth             | **Auto-injected** from caller context | Manual (via `headers` param)    |
| URL param        | `path` (relative, e.g., `/auth/info`) | `url` (full URL)                |
| Methods          | GET, POST, PUT only                   | GET, POST, PUT, DELETE, PATCH   |
| Marketplace-safe | Yes — cloners use their own auth      | Depends on how auth is provided |

**When to use `relevance_api_call`:**

- Calling platform proxy endpoints (`/replicate/*`, `/knowledge/*`, `/agents/*`)
- Any internal platform operation from within a tool step
- Tools that need to work for marketplace cloners (no hardcoded keys)

**When to use `api_call`:**

- Calling external third-party APIs (not Relevance platform)
- Endpoints that need DELETE or PATCH methods
- When you need custom headers or query params

## Running Another Tool as a Step

### run_chain

Runs one of your other tools as a step inside this tool ("Run another tool" in the builder). This
is rung 3 of the [ladder](#picking-a-step-type-the-ladder), and it matters because **the builder's
step picker draws from three separate catalogues** while `relevance_list_transformations` only
searches one:

| Catalogue                     | Found with                       | Added as                    |
| ----------------------------- | -------------------------------- | --------------------------- |
| Transformations               | `relevance_list_transformations` | the transformation directly |
| The project's own tools       | `relevance_list_tools`           | a `run_chain` step          |
| Relevance's public tool steps | `relevance_search_public_tools`  | a `run_chain` step          |

So a step the user can see in the builder UI may be completely absent from a transformation
search — that is expected, not a bug. **If the user names a step you can't find, check the other
two catalogues before concluding it doesn't exist.** (Many Google Calendar, HubSpot, Notion and
Trello "steps" users refer to are public tool steps, not transformations.)

### ⚠️ A bare `studio_id` only resolves in the CURRENT project

`run_chain` looks up `studio_id` in the caller's own project unless you tell it otherwise. Public
tool steps live in a Relevance-owned project, so passing just their id fails with a misleading

> `Tool with id <yourProject>-<studio_id> cannot be found, or you do not have permission to access it.`

and that error is raised during setup, so `raise_subtool_errors: false` does **not** swallow it —
the parent step fails. Pick the row that matches your case:

| Your tool is…                                                                               | Pass                                                                                              |
| ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| in the current project (incl. one you just created or cloned)                               | `studio_id` alone                                                                                 |
| a public tool step you want to run as-is                                                    | `studio_id` **+ `project` + `region`** — read both off the `relevance_search_public_tools` result |
| a public tool you want to modify, or that has its own `run_chain` steps / connected account | `relevance_clone_public_tool` first, then the **new id** it returns, no `project`/`region`        |

Prefer cloning when the public tool nests its own `run_chain` steps or binds an OAuth account:
running in place does not rewrite those, so the nested lookups resolve against _your_ project and
the account id travels over unchanged. Cloning rewrites both.

```typescript
// A public tool step, run in place — project + region come from the search result.
{
  name: "get_events",
  transformation: "run_chain",
  params: {
    studio_id: "get_events_from_google_calendar",
    project: "{{params.source_project}}",   // required for a public tool
    region: "{{params.source_region}}",     // required for a public tool
    params: {
      google_account: "{{params.google_account}}",
      search_from_time: "{{params.start}}"
    },
    raise_subtool_errors: true
  },
  output: {
    events: "{{output.events}}"
  }
}
```

| Param                  | Type    | Notes                                                                                                 |
| ---------------------- | ------- | ----------------------------------------------------------------------------------------------------- |
| `studio_id`            | string  | The tool to run. See the table above — a public tool step needs `project` + `region` too.             |
| `params`               | object  | The subtool's own input params — call `relevance_get_tool` on it first to get their real names.       |
| `raise_subtool_errors` | boolean | Default `false` (subtool errors are captured and the chain continues). Set `true` to fail the parent. |
| `project` / `region`   | string  | Required whenever the tool lives outside the current project — including every public tool step.      |

**Output:** `{{output}}` (the subtool's own output object — reach into it as `{{output.<field>}}`), plus `{{status}}`, `{{errors}}`, `{{job_info}}`. Downstream, an unaliased read is `{{steps.<name>.output.output.<field>}}` — note the doubled `output`; alias it in the step's own `output` mapping to avoid that.

## Code Execution

### js_code_transformation

JavaScript code with native access to `steps` object.

```typescript
{
  name: "process",
  transformation: "js_code_transformation",
  params: {
    code: `
const data = steps.api_step.output.response_body;
const filtered = data.results.filter(item => item.score > 0.5);
return { filtered };
`
  }
}
```

**Output:** `{{transformed.field}}` - Same as Python, wrapped in `.transformed`

**Key feature:** `steps.stepName.output` gives you native JavaScript objects - no parsing needed.

**Use JavaScript when:** Processing JSON from previous steps, simple filtering/mapping, no special library requirements.

### python_code_transformation

Custom Python logic with template substitution.

> ⚠️ The parameter field is **`code`** — NOT `source_code`. Passing `source_code` (or any other alias) will be rejected by the schema with "must NOT have additional properties".

```typescript
{
  name: "process",
  transformation: "python_code_transformation",
  params: {
    code: `
import json

# For simple values, direct substitution works:
value = "{{steps.previous.output.simple_field}}"

# For objects/arrays, template substitution also works:
data = {{steps.previous.output.response_body}}
processed = [item for item in data if item['score'] > 0.5]
return {"filtered": processed}
`
  },
  output: {
    result: "{{transformed.filtered}}"  // Note: wrapped in .transformed
  }
}
```

**Output:** `{{transformed.field}}` - Python returns are wrapped in `.transformed`

**Access pattern:**

```typescript
// Python returns: {"my_data": [...]}
// Next step uses: {{steps.python_step.output.transformed.my_data}}
```

**Use Python when:** You need pandas, complex datetime manipulation, numpy, or other Python-specific libraries.

**Template substitution limitation:** `{{...}}` inserts raw text into the code string. This can cause syntax errors with JSON containing unescaped quotes or special characters. For complex JSON, JavaScript's native `steps` access may be simpler.

### Python: Accessing Tool Input Parameters

**`steps['params']` does NOT exist.** To access tool input parameters in Python, you MUST use template substitution `'{{params.<name>}}'` (note the `params.` prefix). The `steps` dict only contains outputs from previous transformation steps, not the tool's input params.

```python
# WRONG - KeyError: 'params'
url = steps['params']['website_url']

# WRONG - bare name does not resolve; literal string "{{website_url}}" leaks into your code
url = '{{website_url}}'

# CORRECT - template substitution for input params
url = '{{params.website_url}}'
```

**Use direct dict access** (`steps['...']`) for previous step outputs — it's cleaner and avoids JSON parsing issues.

**Use template substitution** (`{{params.<name>}}`) for tool input params and when injecting values into string literals (e.g., SQL queries, URLs).

## Loop Transformation

### loop

Iterate over arrays.

```typescript
{
  name: "process_urls",
  transformation: "loop",
  params: {
    items: "{{params.urls}}",  // MUST be actual array, not JSON string
    execution_mode: "parallel",
    error_handling: "continue",
    steps: [
      {
        name: "scrape",
        transformation: "browserless_scrape",
        params: {
          website_url: "{{foreach.item}}"  // Current item
        }
      }
    ]
  },
  output: {
    all_content: "{{results}}"
  }
}
```

**Output:** `{{results}}` - Array of `{inner_step: {outputs}}`

**Critical:** Loop requires actual array, not JSON string. Parse with Python first if needed.

## File Processing

### file_to_text_llm_friendly

Convert files to text.

```typescript
{
  name: "extract",
  transformation: "file_to_text_llm_friendly",
  params: {
    file: "{{params.file_url}}"
  },
  output: {
    content: "{{text}}"
  }
}
```

Supports: PDF, Word, CSV, Excel

## Quick Reference Table

"Key Output" is the field name on the transformation's raw response (use it inside that step's own `output:` mapping). "Access Pattern" is how a **downstream** step reads that value — always via the full `{{steps.<step_name>.output.<field>}}` path.

| Transformation               | Key Output            | Downstream Access Pattern                                                         |
| ---------------------------- | --------------------- | --------------------------------------------------------------------------------- |
| `markdown`                   | _(none)_              | Documentation step — see [documentation.md](documentation.md)                     |
| `prompt_completion`          | `{{answer}}`          | `{{steps.<name>.output.answer}}`                                                  |
| `python_code_transformation` | `return {"key": val}` | `{{steps.<name>.output.transformed.key}}`                                         |
| `browserless_scrape`         | `{{output.page}}`     | `{{steps.<name>.output.output.page}}` (or via your output mapping)                |
| `serper_google_search`       | `{{organic}}`         | `{{steps.<name>.output.organic}}` (or via your output mapping)                    |
| `loop`                       | `{{results}}`         | `{{steps.<name>.output.results}}` — array of `{inner_step: {...}}`                |
| `run_apify_dynamic`          | `{{items}}`           | `{{steps.<name>.output.items}}`                                                   |
| `api_call`                   | `{{response_body}}`   | `{{steps.<name>.output.response_body}}`                                           |
| `relevance_api_call`         | `{{response_body}}`   | `{{steps.<name>.output.response_body}}` (auth auto-injected)                      |
| `run_chain`                  | `{{output}}`          | `{{steps.<name>.output.output.<field>}}` — alias it to avoid the doubled `output` |

## Email Transformations

### send_email (OAuth — Gmail/Outlook)

Sends email through user's Gmail or Outlook via OAuth.

**Critical:** Provider must use the `_oneof_type_` discriminator pattern:

```json
{
  "name": "send_email",
  "transformation": "send_email",
  "params": {
    "provider": { "_oneof_type_": "Gmail" },
    "oauth_account_id": "{{params.gmail_account}}",
    "to": ["recipient@example.com"],
    "subject": "{{params.subject}}",
    "body": "{{params.email_body}}"
  }
}
```

| Param              | Type   | Notes                                                                        |
| ------------------ | ------ | ---------------------------------------------------------------------------- |
| `provider`         | object | `{"_oneof_type_": "Gmail"}` or `{"_oneof_type_": "Outlook"}` -- NOT a string |
| `oauth_account_id` | string | Connected OAuth account ID                                                   |
| `to`               | array  | MUST be array, not string                                                    |
| `subject`          | string | Email subject                                                                |
| `body`             | string | Supports Markdown (auto-rendered to HTML)                                    |

Optional: `cc`, `bcc` (arrays), `thread_id`, `draft` (boolean), `attachments` (filename -> URL map).

### send_email_step (Mailgun — Server-Side)

For internal/agent use. No OAuth needed.

```json
{
  "transformation": "send_email_step",
  "params": {
    "recipientEmails": ["user@example.com"],
    "subject": "Subject here",
    "body": { "html": "<p>HTML body</p>" }
  }
}
```

## Common Gotchas

### 1. prompt_completion uses `{{answer}}` NOT `{{text}}`

```yaml
# WRONG
output:
  result: "{{text}}"

# CORRECT
output:
  result: "{{answer}}"
```

### 2. Python outputs are wrapped in `.transformed`

```yaml
# Python returns: {"items": [...]}
# Access via:
{{steps.python_step.output.transformed.items}}    # CORRECT
{{steps.python_step.output.items}}                # WRONG — no .transformed wrapper
```

### 3. Loop requires actual array, not JSON string

```yaml
# If LLM generates JSON string, parse first:
- name: parse
  transformation: python_code_transformation
  params:
    code: |
      import json
      return {"items": json.loads("""{{steps.llm_step.output.json_str}}""")}

- name: loop_step
  transformation: loop
  params:
    items: '{{steps.parse.output.transformed.items}}' # Now an actual array
```

### 4. Handle markdown code blocks from LLMs

````python
json_str = """{{steps.step.output.json_output}}"""
json_str = json_str.strip()
if json_str.startswith("```"):
    json_str = json_str.split("\n", 1)[1]
if json_str.endswith("```"):
    json_str = json_str[:-3]
data = json.loads(json_str.strip())
````

### 5. Python: Handle undefined/empty template values

**Problem:** Template variables resolve to literal strings like `'undefined'` or `'null'` when the parameter isn't provided.

```python
# WRONG - This will be truthy even when contact_id is "undefined"
if '{{params.contact_id}}':
    do_something('{{params.contact_id}}')

# CORRECT - Explicitly check for invalid values
contact_id = '{{params.contact_id}}'
if contact_id and contact_id not in ['', 'undefined', 'null', 'None']:
    do_something(contact_id)

# For numeric IDs (like HubSpot), also check .isdigit()
if contact_id and contact_id not in ['', 'undefined', 'null', 'None'] and contact_id.isdigit():
    associations.append({'to': {'id': contact_id}})
```

**Common invalid values to check:**

- `''` (empty string)
- `'undefined'` (JavaScript undefined)
- `'null'` (JavaScript null)
- `'None'` (Python None as string)

### 6. Validate IDs before API calls

**Problem:** Passing undefined IDs to external APIs causes confusing errors.

**Symptom:** "Object not found. objectId are usually numeric" with `"id":["undefined"]`

```python
# Add validation step before API call
deal_id = '{{params.deal_id}}'
if not deal_id or deal_id in ['', 'undefined', 'null', 'None'] or not deal_id.isdigit():
    raise ValueError(f'Invalid deal_id: {deal_id}. Must be a valid numeric ID.')
return deal_id
```

### 7. Don't access array[0] on potentially empty results

**Problem:** Search returns empty array, accessing `results[0]` causes error.

**Symptom:** "Failed to extract output: Error: Value of search.results[0] is null"

```yaml
# WRONG
output:
  contact: '{{steps.search.output.results[0]}}'

# CORRECT - return the whole results, let caller check if empty
output:
  found: '{{steps.search.output.total > 0}}'
  total: '{{steps.search.output.total}}'
  results: '{{steps.search.output.results}}'
```

### 8. Use Python for timestamps, not {{now()}}

**Problem:** `{{now()}}` template function doesn't work in some API call bodies.

```yaml
# WRONG
- name: create_note
  transformation: hubspot_api_call
  params:
    body:
      properties:
        hs_timestamp: '{{now()}}' # May not work

# CORRECT - Use Python step first
- name: get_timestamp
  transformation: python_code_transformation
  params:
    code: |
      import datetime
      return int(datetime.datetime.now().timestamp() * 1000)
  output:
    timestamp: '{{transformed}}'

- name: create_note
  transformation: hubspot_api_call
  params:
    body:
      properties:
        hs_timestamp: '{{steps.get_timestamp.output.timestamp}}'
```

### 9. JS code steps: use backtick template literals, NOT single quotes

**Problem:** Single-quoted template literals break when values contain apostrophes.

`{{param}}` is resolved server-side before the JS code runs (string substitution, not JS evaluation), so backticks are safe for this purpose. However, they are not a security boundary — do not use them for untrusted input that could contain backticks or `${...}` syntax.

```javascript
// WRONG - breaks if param contains an apostrophe (e.g., "O'Brien")
const name = '{{params.name}}';

// BETTER - handles apostrophes in values
const name = `{{params.name}}`;

// BEST for complex data - use native steps access (no template injection needed)
const name = steps.previous_step.output.name;
```

### 10. JS code steps: handle array params that may serialize as strings

When a tool param is typed as `array`, `{{params.<name>}}` may serialize as a comma-separated string. Always handle both:

```javascript
let items = `{{params.items}}`;
try {
  items = JSON.parse(items);
} catch {
  items = items
    .split(',')
    .map((s) => s.trim())
    .filter(Boolean);
}
```

### 11. Variable shadowing in JS code steps (only when using `state_mapping`)

If your tool has a `state_mapping`, its keys are pre-declared by the runtime as variables in the JS code step's scope. Never `const`/`let`-declare a variable with the same name as a state_mapping key — it shadows the injected value. If you don't use `state_mapping` (the default for new tools), this gotcha doesn't apply.

### 12. `params_schema` must use proper JSON Schema structure

`params_schema` MUST be a valid JSON Schema object with `properties` wrapper:

```json
// CORRECT
{
  "type": "object",
  "required": ["email"],
  "properties": {
    "email": {"type": "string", "title": "Email"}
  }
}

// WRONG - flat format, UI cannot render it
{
  "email": {"type": "string", "title": "Email"}
}
```

### 13. Step-level conditions not supported

The `condition` property on transformation steps returns 422. Handle conditional logic within the steps themselves (e.g., instruct the LLM to return null, or handle in a downstream JS step). Use `if` for conditional steps instead.
