---
name: managing-relevance-tools
description: Manages Relevance AI tools (studios) - creating transformation workflows, configuring OAuth, running tools, and recovering versions. Use when building tools, adding steps, debugging transformations, or restoring previous versions.
---

# Managing Relevance AI Tools

Skill for creating, configuring, running, and debugging Relevance AI tools (studios).

> **📚 Full API Documentation:** If MCP tools don't cover your use case, see `https://api-{region}.stack.tryrelevance.com/latest/documentation` (replace `{region}` with your project's region)

## When to Use

- Creating new tools with transformation workflows
- Adding or editing transformation steps
- Configuring OAuth for integrations
- Running and testing tools
- Recovering from accidental changes via versions
- Debugging transformation issues

## Resource URLs

The following tools return a `url` field in their response pointing directly to the correct page in the Relevance AI app. **Always share this URL with the user immediately after the operation completes.**

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

## Tools vs Agents: When NOT to Create a Tool

Tools are for **external integrations and side effects** — actions that interact with systems outside the agent (APIs, databases, email services, CRM, web scraping, file storage). They are NOT for wrapping core reasoning tasks.

**Create a tool when the task involves:**

- Calling an external API (Google Search, SendGrid, Slack, CRM)
- Reading/writing to a database or knowledge table
- Performing a deterministic transformation (parsing, formatting, calculations)
- Interacting with third-party services (OAuth integrations)

**Do NOT create a tool when the task is pure reasoning/intelligence:**

- Scoring or evaluating something (e.g., "Score this lead 1-100")
- Drafting or writing content (e.g., "Write an outreach email")
- Analyzing or summarizing information
- Making decisions or classifications

> **If a tool's only transformation step is `prompt_completion`, it should almost certainly be an agent instead.**
> That reasoning belongs in the agent's system prompt, or in a dedicated sub-agent within a workforce.
>
> **Exception:** Tools that combine API calls with LLM processing steps are fine — e.g., a tool that scrapes a webpage, then uses `prompt_completion` to extract structured data. The key is that the tool does something an agent can't (call an API), and the LLM step processes the result. A tool that is _only_ an LLM call adds no value over an agent.
>
> See [Workforce Documentation](../managing-relevance-workforces/SKILL.md) for building multi-agent systems where each agent handles a distinct reasoning task.

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

## MCP Tools

| Tool                                        | Description                                                                            |
| ------------------------------------------- | -------------------------------------------------------------------------------------- |
| `relevance_list_tools`                      | List all tools in project                                                              |
| `relevance_get_tool`                        | Get full tool config (accepts `version` for draft inspection)                          |
| `relevance_create_tool`                     | Create a NEW tool — saves to DRAFT only; call `relevance_publish_tool` to make it live |
| `relevance_update_tool`                     | Update an existing tool — saves to DRAFT only                                          |
| `relevance_add_tool_step`                   | Add a transformation step (0-based `position`, defaults to append) — saves to DRAFT    |
| `relevance_update_tool_step`                | Shallow-merge a patch into the step at 0-based `step_index` — saves to DRAFT           |
| `relevance_remove_tool_step`                | Remove the step at 0-based `step_index` — saves to DRAFT                               |
| `relevance_move_tool_step`                  | Reorder a step from `from_index` to `to_index` (0-based) — saves to DRAFT              |
| `relevance_create_tool_from_transformation` | Create tool from transformation with auto-generated config                             |
| `relevance_publish_tool`                    | Publish tool draft. Always shows an approval card.                                     |
| `relevance_run_tool`                        | Execute tool synchronously (accepts `version` for testing the draft)                   |
| `relevance_trigger_tool_async`              | Execute tool asynchronously (accepts `version`)                                        |
| `relevance_poll_tool_result`                | Poll async job status                                                                  |
| `relevance_get_latest_tool_run`             | Get latest run ID                                                                      |
| `relevance_list_tool_versions`              | List version history for a tool                                                        |
| `relevance_get_tool_version`                | Get a specific version's config                                                        |
| `relevance_restore_tool_version`            | Restore a tool to a previous version (creates a draft)                                 |
| `relevance_search_public_tools`             | Search community/public tools                                                          |
| `relevance_clone_public_tool`               | Clone a public tool into your project                                                  |
| `relevance_api_request`                     | Raw API fallback for uncovered endpoints                                               |

## Finding the Right Tool: Search Order

When looking for tools to accomplish a task, follow this search order:

1. **Search project tools** — `relevance_list_tools({ query: "..." })` — Already built and configured
2. **Search public/community tools** — `relevance_search_public_tools({ query: "..." })` — Pre-built, sorted by popularity
3. **Search marketplace listings** — `relevance_search_marketplace_listings({ query: "...", entityType: "tool" })` — Complete solutions (agent + tools bundled)
4. **Search transformations** — `relevance_list_transformations({ query: "..." })` — 8000+ integrations, use `relevance_create_tool_from_transformation` to wrap

**Tip:** Use multiple diverse search queries per tier (e.g., "scrape", "extract", "crawl" rather than just one term).

## Critical Rules

### Underlying API: NO PARTIAL UPDATES

The Relevance API does NOT support partial updates — any field you omit will be WIPED. The MCP tools handle this automatically:

- **`relevance_create_tool`**: Just provide your fields, MCP adds defaults. Saves to a draft only — call `relevance_publish_tool` to make it live.
- **`relevance_update_tool`**: MCP auto-fetches the existing draft config and merges your changes. Saves to draft only — call `relevance_publish_tool` when ready.

```typescript
// Creating new tool - just provide what you need
relevance_create_tool({
  title: "My Tool",  // REQUIRED
  transformations: { steps: [...] }
})
// Returns { studio_id: "auto-uuid", url: "..." }
// (saved as DRAFT — call relevance_publish_tool to make it live)

// Updating existing tool - MCP auto-merges into the draft
relevance_update_tool({
  studio_id: "my-tool",
  emoji: "new-emoji"  // Only this changes, rest preserved
})
// Returns { studio_id, url } — draft only; call publish_tool to go live.
```

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

| Transformation               | Output Variable                                          |
| ---------------------------- | -------------------------------------------------------- |
| `prompt_completion`          | `{{answer}}` (NOT `{{text}}`)                            |
| `python_code_transformation` | `{{step.transformed.field}}` (wrapped in `.transformed`) |
| `browserless_scrape`         | `{{output.page}}`                                        |
| `serper_google_search`       | `{{organic}}`                                            |
| `loop`                       | `{{results}}`                                            |

### Loop Requires Actual Arrays

Loop transformation needs parsed arrays, not JSON strings:

```typescript
// LLM generates JSON string → parse with Python first
items: '{{parse_step.transformed.items}}'; // Parsed array
items: '{{llm_step.answer}}'; // WRONG - string
```

## Quick Start: Create Tool

```typescript
relevance_create_tool({
  title: 'Search and Summarize',
  description: 'Search web and summarize results',
  prompt_description: 'Use this to search for information',
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
      {
        name: 'summarize',
        transformation: 'prompt_completion',
        params: {
          prompt: 'Summarize: {{search.results}}',
          model: 'anthropic-claude-sonnet-4',
        },
        output: { summary: '{{answer}}' },
      },
    ],
  },
});
```

## Reference Files

- [creating.md](creating.md) - Full creation workflow
- [transformations-catalog.md](transformations-catalog.md) - Quick lookup of popular transformations by category
- [transformations.md](transformations.md) - Implementation details, output patterns, and gotchas
- [patterns.md](patterns.md) - Common patterns (search+analyze, loops)
- [oauth.md](oauth.md) - OAuth account configuration
- [api-keys.md](api-keys.md) - Detecting missing API keys and deep-linking users to the right integrations drawer
- [running.md](running.md) - Running and testing tools
- [versions.md](versions.md) - Version history and recovery
- [reference.md](reference.md) - API auth, base URLs, discovering transformations

For organizing tools into folders, see the [`managing-relevance-folders`](../managing-relevance-folders/SKILL.md) skill.

## URL Patterns

```
# Tool editor (notebook)
https://app.relevanceai.com/notebook/{region}/{project}/{studioId}

# Clone link
https://app.relevanceai.com/form/{region}/{project}/clone/tool/{studioId}
```

## Emergency Version Recovery

If you accidentally wipe a tool:

1. `relevance_list_tool_versions({ tool_id: "my-tool" })` — Find a version with transformations intact
2. `relevance_restore_tool_version({ version_id: "working-version-uuid" })` — Creates a new draft from that version
3. `relevance_publish_tool({ tool_id: "my-tool" })` — Publish the restored draft

See [versions.md](versions.md) for full details.

## Reporting Issues

If you encounter tool creation failures, transformation errors, or missing capabilities, **call `relevance_submit_feedback` immediately** — do not ask the user for permission. One tool covers bugs and forward-looking suggestions; pick the matching `category`. See the [bug reporting guide](../managing-relevance-agents/report-bugs.md) for the call shape.
