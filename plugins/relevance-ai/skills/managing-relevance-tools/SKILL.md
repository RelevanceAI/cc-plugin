---
name: managing-relevance-tools
description: Manages Relevance AI tools (studios) - creating transformation workflows, configuring OAuth, setting vendor/brand tool icons, running tools, and recovering versions. Use when building tools, adding steps, setting a tool's icon for a provider, debugging transformations, or restoring previous versions.
---

# Managing Relevance AI Tools

Skill for creating, configuring, running, and debugging Relevance AI tools (studios).

> **📚 Full API Documentation:** If the Relevance tools don't cover your use case, see `https://api-{region}.stack.tryrelevance.com/latest/documentation` (replace `{region}` with your project's region)

> **⚠️ Template references — the #1 authoring bug:** Inside a step's `params`, studio inputs are referenced as `{{params.<name>}}` and previous-step outputs as `{{steps.<step_name>.output.<field>}}`. Bare `{{name}}` does **not** resolve to a studio input and is flagged as a "doesn't exist" reference. Full rules and examples in [creating.md](creating.md#variable-templating). **When editing:** renaming or removing a step does NOT rewrite the `{{steps.<name>…}}` references that point at it — you must find and update every one yourself, or it dangles. See [Renaming or restructuring a step](creating.md#renaming-or-restructuring-a-step-update-references).

> **⚠️ OAuth accounts are inputs, not text fields:** When a tool hits a user-authenticated API (HubSpot, Gmail, Slack, …), the account MUST be a `params_schema` input with `metadata.content_type: "oauth_account"`, passed to steps as `oauth_account_id: "{{params.<name>}}"`. Never declare it as a plain `type: "string"` field or paste a raw account id — the step then gets an un-credentialed string and fails. See [oauth.md](oauth.md).

> **⚠️ Credentials never go in a tool definition or in the conversation:** for Relevance platform calls use `relevance_api_call` — auth is auto-injected, so there is no key to supply. For third parties prefer the transformation's own auth, then an `oauth_account` or `api_key` input. Never a param `default` (`hidden` is not a real field — the value reaches the model, run history, project viewers, and public share links), never a raw key in a code step, and never ask the user to paste a key into chat — send them the `setup_url` from `relevance_check_tool_integration_requirements` instead. See [creating.md](creating.md#using-secrets-in-code-steps).

> **⚠️ Knowledge search reads a knowledge set, not a dataset:** the knowledge-search step (`transformation: "search"`) takes its source from `dataset_id`, which MUST resolve to a `knowledge:<knowledge_set_id>` value (or `knowledge:*` for all knowledge). Expose the source as a `params_schema` input with `metadata.content_type: "knowledge_set"` (renders the knowledge-set picker, which produces the `knowledge:`-prefixed value), or hardcode `dataset_id: "knowledge:<id>"` (get ids from `relevance_list_knowledge_sets`). Never a plain `type: "string"` input defaulting to a bare dataset name — it "looks filled in" but the step fails at run time with _"Search can only be performed on a knowledge set."_ See the [Knowledge Search pattern](patterns.md#knowledge-search-over-a-knowledge-set).

## When to Use

- Creating new tools with transformation workflows
- Adding or editing transformation steps
- Documenting a multi-step tool with markdown notes (see [documentation.md](documentation.md))
- Configuring OAuth for integrations
- Running and testing tools
- Recovering from accidental changes via versions
- Debugging transformation issues

## Resource URLs

The following tools return a `url` field in their response pointing directly to the correct page in the Relevance AI app. **Share this URL in the reply that reports the result.**

| Tool                     | URL points to         |
| ------------------------ | --------------------- |
| `relevance_create_tool`  | Tool code editor page |
| `relevance_update_tool`  | Tool code editor page |
| `relevance_publish_tool` | Tool code editor page |

The URL is already constructed with the correct region and project ID — just present it to the user.

## Tool Icons

The `emoji` field on `relevance_create_tool` / `relevance_update_tool` accepts either a unicode emoji or a URL to an SVG. **When the tool wraps a known third-party provider, always prefer the brand icon from the Relevance CDN over a unicode emoji.**

### CDN URL pattern

```
https://cdn.jsdelivr.net/gh/RelevanceAI/content-cdn@latest/vendors/icons/{filename}
```

`{filename}` is the full file name including `.svg`. Pass the full URL as the `emoji` value — do not pass just the slug (`"google-mail"` will render as a blank placeholder).

### Decision rule

1. Tool wraps a known provider (Slack, HubSpot, Gmail, etc.) → use the CDN URL with the exact filename from the list below.
2. No provider match → use a relevant unicode emoji (e.g. `"🔍"`, `"🎲"`).
3. Never invent a filename. If you can't verify the icon exists, fall back to unicode.

### Available filenames

Pass `.svg` on the end of any of these. Lowercase only.

```
airtable, anthropic, apollo, ashby, avoma, bigquery, brightdata, browserless,
calendar, canva, cohere, confluence, database, databricks, facebook, file,
firecrawl, gamma, github, gong, google, google-calendar, google-docs,
google-drive, google-gemini, google-mail, google-sheets, groq, hubspot,
huggingface, instagram, jira, key-command, linear, linkedin, microsoft-outlook,
microsoft_onedrive, microsoft_teams, mistral, monday, mysql, notion, openai,
openrouter, outreach, perplexity, postgres, redis, relevance, replicate,
salesforce, salesloft, sendgrid, serpapi, serper, sharepoint, slack, snowflake,
supabase, tavily, telegram, trello, twilio, twitter, webflow, webhook,
whatsapp, x, xai, youtube, zoho_crm, zoom, zoominfo
```

### Non-obvious filenames (common mistakes)

| If you're thinking… | The actual filename is…               |
| ------------------- | ------------------------------------- |
| `gmail`             | `google-mail.svg`                     |
| `outlook`           | `microsoft-outlook.svg` (hyphen)      |
| `teams`             | `microsoft_teams.svg` (underscore)    |
| `onedrive`          | `microsoft_onedrive.svg` (underscore) |
| `zoho` / `zohocrm`  | `zoho_crm.svg` (underscore)           |

### Worked example

```typescript
relevance_create_tool({
  title: 'Send Slack message',
  description: 'Post a message to a Slack channel',
  emoji:
    'https://cdn.jsdelivr.net/gh/RelevanceAI/content-cdn@latest/vendors/icons/slack.svg',
  // ...rest of config
});
```

## Tools vs Agents: What Goes Where

Tools are for **external integrations and side effects** — actions that interact with systems outside the agent (APIs, databases, email services, CRM, web scraping, file storage). They are NOT for wrapping core reasoning tasks.

**Create a tool when the task involves:**

- Calling an external API (Google Search, SendGrid, Slack, CRM)
- Reading/writing to a database or knowledge table
- Performing a deterministic transformation (parsing, reshaping, date maths, calculations)
- Interacting with third-party services (OAuth integrations)

**Do NOT create a tool when the task is pure reasoning/intelligence:**

- Scoring or evaluating something (e.g., "Score this lead 1-100")
- Drafting or writing content (e.g., "Write an outreach email")
- Analyzing or summarizing information
- Making decisions or classifications

### Default: the tool returns data, the agent's system prompt judges

A tool fetches, writes and computes, and returns the records plus how many it scanned and kept. The agent that calls it carries the judgement in its system prompt: which items matter when no field decides it, the summary, the wording and the shape of the answer.

- **Filter exact criteria in the fetch first.** When a field answers the question (sender, label, date, status, a marker address), filter on it in the fetch step's own query or parameters. Add a code step only for what those can't express (grouping, comparing against the current time, de-duplicating).
- **Look before you filter.** Before writing a filter over data you haven't seen, run the read step once on a small page with `relevance_run_transformation`: a guessed keyword that matches nothing returns an empty list that reads as "nothing found". Only probe a step that reads, never one that writes. A step writes when it sends or posts anything, or adds, changes or deletes data anywhere, a knowledge table included, since it may hold live data. Judge a step by what it does, not its name: a find-or-create step writes, and so can a search sent as a POST. A step you can't classify counts as one that writes.
- **Return records and counts, not a write-up.** Return the fields the agent needs rather than whole bodies it won't read, plus the counts, so an empty result can be told apart from a broken fetch. Page the fetch and trim long bodies: everything returned is billed at the agent's model rate and re-sent on every turn.
- **Records the agent handles one by one start it instead.** When each record should get its own task (each new lead qualified, each ticket answered), the fetching tool is not attached to the agent: it becomes a [tool trigger](../managing-relevance-agents/triggers.md#tool-triggers-one-task-per-item) that starts one task per record, and it returns an empty list, not an error, when nothing is new. A list the user already has (a CSV, a knowledge table) needs no fetching tool: Run in bulk starts one task per row ([Work Over Many Items](../managing-relevance-agents/many-items.md)). Attach it only when one run needs the whole list at once: a digest, a count, a comparison.
- **The smell:** an agent prompt that says "return the tool's output unchanged". The judgement is in the wrong place; move the tool's LLM step into the agent's system prompt.

Why: the owner can read and edit an agent's system prompt, while a prompt inside a tool step is buried in the tool editor. Evals simulate the whole tool, so judgement inside it is never tested. And each LLM step is a second model call on every run.

**Exceptions.** Put an LLM step in the tool only when no agent reads its output (a tool column run on every row of a knowledge table with Run tool on all rows, or a tool triggered directly or published for others to call; an agent that runs on a schedule still reads it), when extracting fields from each of many long documents the agent couldn't read in one go (scraped pages, PDFs), when a later step in the same tool needs the result (prefer `decide` for a label), or when several agents call the tool and must judge the same way, so the rule lives in one place. Filtering or summarising a list the agent can read is never one of these.

Reasoning big enough to be its own role belongs in a dedicated agent within a workforce — see [Workforce Documentation](../managing-relevance-workforces/SKILL.md).

### Example: Sales Lead Pipeline

**Reasoning tasks → use agents, not tools:**

| Task                 | ❌ Wrong: Tool with LLM step                     | ✅ Right: Agent in workforce                                   |
| -------------------- | ------------------------------------------------ | -------------------------------------------------------------- |
| Score a lead         | Tool with `prompt_completion` that scores leads  | **Lead Qualifier agent** with scoring logic in system prompt   |
| Draft outreach email | Tool with `prompt_completion` that writes emails | **Outreach Drafter agent** with email writing in system prompt |

**Integration tasks → correctly tools (attached to agents):**

| Task                 | Correct implementation              |
| -------------------- | ----------------------------------- |
| Look up company data | Tool calling Clearbit/LinkedIn API  |
| Send email           | Tool calling SendGrid/email API     |
| Update CRM           | Tool calling Salesforce/HubSpot API |

The agents handle reasoning; tools handle external actions. In evals, you simulate the external tools (Clearbit, SendGrid) while testing the agents' actual reasoning.

> **Tools have no evals or performance dashboards.** Evals (test sets/scenarios/checks) and live-scoring performance dashboards are configured on an **agent or workforce**, never on an individual tool — a tool has no eval/dashboard resource type. A tool's own runtime health is visible through **usage analytics** (execution counts, error rates, credits) and its run history, not through evals or a monitoring dashboard. So after building or publishing a tool, don't offer an eval set or a performance dashboard for the tool itself — offer to attach and test it, or build the agent that uses it (evals/dashboards then go on that agent or workforce). See [relevance-evals](../relevance-evals/SKILL.md) and [relevance-analytics](../relevance-analytics/SKILL.md).

## Tools

| Tool                                        | Description                                                                                                                      |
| ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `relevance_list_tools`                      | List all tools in project                                                                                                        |
| `relevance_get_tool`                        | Get full tool config (accepts `version` for draft inspection)                                                                    |
| `relevance_create_tool`                     | Create a NEW tool — saves to DRAFT only; call `relevance_publish_tool` to make it live                                           |
| `relevance_update_tool`                     | Update an existing tool — saves to DRAFT only                                                                                    |
| `relevance_update_tool_steps`               | Every step edit to one tool in one call (`add`/`update`/`remove`/`move` ops, applied in order, one save) — saves to DRAFT        |
| `relevance_add_tool_step`                   | Add a transformation step (0-based `position`, defaults to append) — saves to DRAFT                                              |
| `relevance_update_tool_step`                | Merge a patch into the step at 0-based `step_index` (`null` clears a field) — saves to DRAFT                                     |
| `relevance_remove_tool_step`                | Remove the step at 0-based `step_index` — saves to DRAFT                                                                         |
| `relevance_move_tool_step`                  | Reorder a step from `from_index` to `to_index` (0-based) — saves to DRAFT                                                        |
| `relevance_set_tool_output`                 | Edit the tool's runtime output map (`transformations.output`) and `output_schema` — saves to DRAFT. See [outputs.md](outputs.md) |
| `relevance_create_tool_from_transformation` | Create tool from transformation with auto-generated config                                                                       |
| `relevance_publish_tool`                    | Publish tool draft. Always shows an approval card.                                                                               |
| `relevance_trigger_tool`                    | Run a tool — returns a `job_id` to poll (accepts `version` for testing the draft)                                                |
| `relevance_poll_tool_result`                | Poll for the tool result using the `job_id`                                                                                      |
| `relevance_get_latest_tool_run`             | Get latest run ID                                                                                                                |
| `relevance_list_tool_versions`              | List version history for a tool                                                                                                  |
| `relevance_get_tool_version`                | Get a specific version's config                                                                                                  |
| `relevance_restore_tool_version`            | Restore a tool to a previous version (creates a draft)                                                                           |
| `relevance_search_public_tools`             | Search public tool steps                                                                                                         |
| `relevance_clone_public_tool`               | Install a public tool step into your project                                                                                     |

## Finding the Right Tool: Search Order

This ladder applies **both** when looking for a tool to attach to an agent **and** when looking for
a step to put inside a tool you're building — rungs 1–2 are addable as a step via `run_chain`.
Follow it **unless the user has explicitly asked you to build from scratch — in that case, skip
straight to creating a new one**:

1. **Search project tools** — `relevance_list_tools({ query: "..." })` — Already built and configured
2. **Search public tool steps** — `relevance_search_public_tools({ query: "..." })` — pre-built, sorted by popularity. Some harnesses narrow this to Relevance's own curated steps
3. **Search transformations** — `relevance_list_transformations({ search: "..." })` — 8000+ integrations, use `relevance_create_tool_from_transformation` to wrap

> ⚠️ Note the parameter differs: tools take `query`, transformations take **`search`**.

**Tip:** Issue several **single-keyword** queries per tier ("scrape", then "extract", then "crawl") rather than one long phrase. `relevance_list_transformations` requires every word of `search` to match, so a phrase like "get events from google calendar" returns nothing while "calendar" returns the set. When a search comes up empty, browse by tag instead — `{ integration: "Google" }` or `{ use_case: "Calendar" }`.

**These are separate catalogues, and the builder shows more than you can search.** A step the user
can see in the picker may be a public tool step (rung 2), not a transformation (rung 3) — so an
empty transformation search does not mean the capability is missing. There is also a marketplace of
complete solutions (agent + tools bundled); some harnesses can search and install from it, others
cannot — **your own tool list is the authority on which catalogues you can reach.** Where you can't
reach one the user names, say so plainly, offer to build the equivalent from Relevance's own steps,
and otherwise ask them to add it in the builder themselves — **never tell the user a step doesn't
exist because your search missed it.** See [transformations.md](transformations.md#picking-a-step-type-the-ladder).

### Prefer native transformations

When `relevance_list_transformations` returns results, they are partitioned into two groups:

- **`native`** — built and maintained by Relevance AI. Stable contracts, first-class support, ongoing improvements. **This is the default choice.**
- **`pipedream`** — third-party integrations (transformation IDs follow the pattern `{provider}_-_{action}`). Treated as legacy. Use only when no native equivalent exists for the provider/action the user needs.

Decision rule when both exist for the same provider:

1. Pick the `native` entry by default.
2. Only pick `pipedream` if there's no native equivalent for the specific action.
3. Never silently fall back to a `pipedream` entry. If only a third-party option is available, surface that to the user — e.g. "I couldn't find a native action for SharePoint create-folder, but there's a third-party one. Want me to use it?"

This applies to `relevance_create_tool_from_transformation` too — always pass the native `transformation_id` when one exists.

When referring to a `pipedream` entry in user-facing prose, call it a "third-party" integration. (`pipedream` is the internal field value the API uses; "third-party" is the user-facing term.)

## Critical Rules

### Underlying API: the merge is TOP-LEVEL ONLY

`relevance_update_tool` auto-fetches the current draft and merges your fields **at the top level only**: a top-level field you omit is preserved; a top-level field you provide **replaces the previous value wholesale** (no deep merge into it).

- **`relevance_create_tool`**: Just provide your fields, the tool adds defaults. Saves to a draft only — call `relevance_publish_tool` to make it live.
- **`relevance_update_tool`**: Auto-fetches the existing draft config and shallow-merges your **top-level** fields. Saves to draft only — call `relevance_publish_tool` when ready.

```typescript
// Creating new tool - just provide what you need
relevance_create_tool({
  title: "My Tool",  // REQUIRED
  transformations: { steps: [...] }
})
// Returns { studio_id: "auto-uuid", url: "..." }
// (saved as DRAFT — call relevance_publish_tool to make it live)

// Updating a scalar top-level field - safe, the rest is preserved
relevance_update_tool({
  studio_id: "my-tool",
  emoji: "new-emoji"  // Only this changes; other top-level fields preserved
})
```

> ⚠️ **`params_schema` is a SINGLE top-level field, so providing it REPLACES the entire schema — it is NOT merged property-by-property.** If you send a `params_schema` that omits properties the tool already has, those params are silently dropped, and the agent loses the ability to send them — the tool can then misbehave or silently do nothing. **To add or change ONE param you MUST re-send the COMPLETE `params_schema` — every existing property plus your change:**

```typescript
// WRONG — silently drops every other param, breaking the tool:
relevance_update_tool({
  studio_id: 'my-tool',
  params_schema: {
    type: 'object',
    properties: { new_flag: { type: 'boolean' } },
  },
});

// RIGHT — fetch the full schema, add your property, send the whole thing back:
const tool = relevance_get_tool({ studio_id: 'my-tool' });
relevance_update_tool({
  studio_id: 'my-tool',
  params_schema: {
    ...tool.params_schema,
    properties: {
      ...tool.params_schema.properties, // keep ALL existing params
      new_flag: { type: 'boolean', description: '...' },
    },
  },
});
// To intentionally REMOVE params, list them in params_to_remove: ["old_param"].
```

The **same re-send-the-whole-thing rule applies to every top-level field** that holds a structure — e.g. `state_mapping`. Only `params_schema` drops are guarded against automatically; a partial `state_mapping` (or any other structured top-level field) is replaced silently with no warning, so preserve those by hand.

### Editing an existing complex tool safely

Complex tools (many params, `state_mapping`, multi-step transformations) are where silent data loss bites. Before editing one:

1. **`relevance_get_tool` first** and work from its real config — never edit from memory.
2. **Preserve every existing property** you are not deliberately changing (see the `params_schema` rule above; steps/outputs have their own granular tools — do not pass `transformations` to `update_tool`).
3. **Diff before you publish.** Compare the draft against the pre-edit config and confirm nothing you didn't intend to touch changed (params, `state_mapping` keys, step outputs). If your harness/UI can show a diff, review it; otherwise call `relevance_get_tool` on the draft and eyeball it.
4. **Verify along the real path** (see [running.md](running.md#testing-workflow)) before telling the user it works.

### Creating Tools from Transformations

Fastest way to create a tool - auto-generates params_schema, state_mapping, and bindings:

```typescript
relevance_create_tool_from_transformation({
  transformationId: 'serper_google_search',
  title: 'Google Search', // optional
});
// Searches existing tools first to avoid duplicates
// Returns { studio_id, tool, wasExisting: boolean }
```

See [creating.md](creating.md) for full workflow.

### Key Output Patterns

"Raw field" is the field name on the transformation's response (use inside that step's own `output:` mapping). "Downstream access" is how a later step reads the value.

| Transformation               | Raw field                                         | Downstream access                         |
| ---------------------------- | ------------------------------------------------- | ----------------------------------------- |
| `prompt_completion`          | `{{answer}}` (NOT `{{text}}`)                     | `{{steps.<name>.output.answer}}`          |
| `python_code_transformation` | `return {"key": val}` (wrapped in `.transformed`) | `{{steps.<name>.output.transformed.key}}` |
| `browserless_scrape`         | `{{output.page}}`                                 | `{{steps.<name>.output.output.page}}`     |
| `serper_google_search`       | `{{organic}}`                                     | `{{steps.<name>.output.organic}}`         |
| `loop`                       | `{{results}}`                                     | `{{steps.<name>.output.results}}`         |

### Loop Requires Actual Arrays

Loop transformation needs parsed arrays, not JSON strings:

```typescript
// LLM generates JSON string → parse with Python first
items: '{{steps.parse_step.output.transformed.items}}'; // Parsed array
items: '{{steps.llm_step.output.answer}}'; // WRONG - string
```

## Quick Start: Create Tool

```typescript
relevance_create_tool({
  title: 'Search the Web',
  description: 'Search the web and return the top results',
  prompt_description:
    'Use this to find current information on the web. Returns the title, link and snippet of the top results.',
  params_schema: {
    type: 'object',
    properties: {
      query: {
        type: 'string',
        title: 'Search Query',
        description: 'What to search for',
      },
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
      {
        name: 'shortlist',
        transformation: 'js_code_transformation',
        params: {
          code: `
const all = steps.search.output.results || [];
const results = all.slice(0, 5).map((r) => ({ title: r.title, link: r.link, snippet: r.snippet }));
return { results, scanned: all.length, kept: results.length };
`,
        },
      },
    ],
    output: {
      results: '{{steps.shortlist.output.transformed.results}}',
      scanned: '{{steps.shortlist.output.transformed.scanned}}',
      kept: '{{steps.shortlist.output.transformed.kept}}',
    },
  },
  output_schema: {
    metadata: { field_order: ['results', 'scanned', 'kept'] },
    properties: {
      results: { metadata: { render_mode: 'table' } },
      scanned: { metadata: { render_mode: 'raw' } },
      kept: { metadata: { render_mode: 'raw' } },
    },
  },
});
// No summarise step: the agent that calls this tool writes the answer from its system prompt.
```

Need to edit later? Metadata via `relevance_update_tool`. Steps via `relevance_update_tool_steps` (every step edit to one tool in one call — several step-edit calls to the same tool at once lose all but the last save), or `relevance_add_tool_step` / `relevance_update_tool_step` / `relevance_remove_tool_step` / `relevance_move_tool_step` for a single edit. Outputs via `relevance_set_tool_output`.

## Tool Outputs

Configure outputs via the paired fields `output` (the runtime map — what the tool returns) and `output_schema` (the display contract — how the canvas Outputs panel and downstream consumers see the result). They mirror what you see on `relevance_get_tool`.

Set both on `relevance_create_tool` for one-shot creation, or edit later via `relevance_set_tool_output`. Read [outputs.md](outputs.md) before the first call, and whenever editing a tool that drives a `tool_trigger` on an agent or a workforce.

**When to set them:** whenever the tool is consumed by another entity (an agent, a `tool_trigger`, a marketplace listing). In practice: always, unless the tool is a one-off you'll only run by hand. The runtime keys in `output` MUST match the keys declared in `output_schema.properties`.

`render_mode` is one of `auto | hide | html | image | json | markdown | raw | table` (default `auto`).

**Common gotcha — `tool_trigger`:** a trigger of `type: 'tool_trigger'`, on an agent or a workforce, references one of the tool's runtime output keys by literal lookup (`tool_config.output_field`). If you remove or rename a key the trigger depends on, the trigger fails on every run, which is easy to miss: only the trigger's status and trigger alarms can show it, and only where trigger run recording is on. See [outputs.md](outputs.md) and [`managing-relevance-workforces/workforce-triggers.md`](../managing-relevance-workforces/workforce-triggers.md).

## Reference Files

- [creating.md](creating.md) - Full creation workflow
- [outputs.md](outputs.md) - `set_tool_output` reference: Manual vs passthrough modes, `tool_trigger` compatibility (REQUIRED reading before the first `set_tool_output` call, and when editing a tool that drives a tool trigger)
- [documentation.md](documentation.md) - Documenting tools with markdown notes (REQUIRED reading when building any tool with ≥3 steps or non-obvious logic)
- [transformations-catalog.md](transformations-catalog.md) - Quick lookup of popular transformations by category
- [transformations.md](transformations.md) - Implementation details, output patterns, and gotchas
- [patterns.md](patterns.md) - Common patterns (search, loops, per-item extraction)
- [oauth.md](oauth.md) - OAuth account configuration
- [api-keys.md](api-keys.md) - Detecting missing API keys and deep-linking users to the right integrations drawer
- [running.md](running.md) - Running and testing tools
- [versions.md](versions.md) - Version history and recovery

For organizing tools into folders, see the [`managing-relevance-folders`](../managing-relevance-folders/SKILL.md) skill.

## App URLs

Send back a url a tool returned; never assemble one. The create/update/publish tools return the tool's editor link, and the integration-check tools return `setup_url` / `config_url` — for provider-specific deep links see [api-keys.md](api-keys.md).

If you are ever checking a url, every app URL is **section-first** — `{app}/{section}/{region}/{project}/…`; the region and project never come before the section.

## Emergency Version Recovery

If you accidentally wipe a tool:

1. `relevance_list_tool_versions({ tool_id: "my-tool", detail: "full" })` — Find a version with transformations intact
2. `relevance_restore_tool_version({ version_id: "working-version-uuid" })` — Creates a new draft from that version
3. `relevance_publish_tool({ tool_id: "my-tool" })` — Publish the restored draft

See [versions.md](versions.md) for full details.

## Reporting Issues

When a failure is the platform's (not the input, not your own config) or a capability is missing, **call `relevance_submit_feedback` without asking the user for permission**. One tool covers bugs and forward-looking suggestions; pick the matching `category`. See the [bug reporting guide](../managing-relevance-agents/report-bugs.md) for the call shape.
