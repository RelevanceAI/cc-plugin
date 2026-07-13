---
title: Writing System Prompts
description: Write effective agent system prompts — inline `{{_actions.<id>}}` pill placement, formatting rules (no markdown tables), role/scope/output structure, and common pitfalls. Load when authoring or revising an agent's system prompt.
---

# Writing System Prompts

Best practices for writing effective agent system prompts.

## Formatting Rules

Use bullets, numbered lists, or `key: value` lines for structure. **Never use markdown tables** (pipe-delimited rows like `| col1 | col2 |` followed by `| --- | --- |`) — the prompt editor cannot render them and they appear to the agent (and the user editing the prompt) as raw pipe characters.

Replace tables with bullet lists or arrow-style key-value lines:

```markdown
# BAD — renders as raw pipes in the prompt editor

| Category  | Route               |
| --------- | ------------------- |
| Billing   | billing@example.com |
| Technical | support@example.com |

# GOOD — renders cleanly

- Billing → billing@example.com
- Technical → support@example.com
```

## Reading and Editing an Existing Prompt

The default `relevance_get_agent` response returns only `system_prompt_preview` (truncated at 300 chars) plus `system_prompt_length` and `system_prompt_line_count` — never analyse or edit a prompt from the preview, it will silently mangle anything past 300 chars.

Pick the approach by size (check `system_prompt_line_count` from `relevance_get_agent`):

- **Small prompt / full rewrite** → `relevance_get_agent` with `summary: false` to read the whole thing, and `relevance_update_agent` with `patch.system_prompt` (or `relevance_create_agent`) to write it.
- **Large prompt** → don't pull the whole thing into context. Use the dedicated tools, which fetch and mutate the prompt server-side:
  - `relevance_search_agent_system_prompt` — grep for the section you care about (`pattern` is a JS regex); returns line numbers.
  - `relevance_read_agent_system_prompt` — read a line-numbered window (`offset_line` / `limit_lines`, like `cat -n`). `limit_lines: 0` returns just the size.
  - `relevance_edit_agent_system_prompt` — change a section by string replacement: each `edits[].old_string` must match the current draft **verbatim** and be **unique** (add surrounding context, or set `replace_all: true`). Pass `expected_line_count` (the `total_lines` you last read) to be told if the prompt changed underneath you. Saves to the **draft** — publish separately.

Typical large-prompt edit loop: `search` to locate → `read` the surrounding lines → `edit` with a unique `old_string` → `read` again to confirm. You never send the whole prompt back.

## Basic Structure

```markdown
You are a [role] that specializes in [domain].

## Your Responsibilities

- [Primary task]
- [Secondary task]

## Tool Usage

When the user asks about [topic]:

1. Use {{_actions.ACTION_ID}} to [action]
2. Analyze the results
3. Provide a summary

## Guidelines

- Be [personality traits]
- Always [key behaviors]
- Never [prohibited actions]
```

## Referencing Tools

Use `{{_actions.ACTION_ID}}` to reference tools in prompts. The `ACTION_ID` is a backend-generated 16-char hex string. The runtime substitutes each pill for the real function-call name; the editor renders them as clickable pill chips.

### Inline Placement (Required)

Weave `{{_actions.<id>}}` pills inline into the prose at every point where the prompt directs the agent to use a specific tool. Inline pills give the LLM a strong tool-selection cue at decision time.

```markdown
# GOOD — pills woven inline at each decision point

When the user asks a research question:

1. Use {{_actions.abc123def4567890}} to search for sources.
2. For promising results, use {{_actions.fedcba0987654321}} to extract content.
```

```markdown
# BAD — prose describes the tools generically with no pills; the LLM has no

# selection cue when it needs to act

When the user asks a research question, search for sources and extract content.
```

### Getting Action IDs

```typescript
// Fetch the actual action_ids
const tools = await relevance_get_agent_tools({ agent_id: '...' });
// Returns: { chains: [{ studio_id: "my-tool", action_id: "abc123def456" }] }

// Use in system prompt:
system_prompt: `Use {{_actions.abc123def456}} to search...`;
```

**Important:** The `action_id` you set in the actions array is NOT used by the UI. You must fetch the backend-generated IDs.

### Example: Pills Inline in a Workflow

```markdown
You are a research assistant that helps users find information.

## Workflow

When the user asks about a topic:

1. Use {{_actions.abc123}} to search for relevant sources.
2. For promising results, use {{_actions.def456}} to get full content.
3. Synthesize and summarize findings.
```

## Writing Guidelines

### Be Specific About Tool Usage

```markdown
# BAD - vague

Use the search tool when needed.

# GOOD - specific

When researching a company:

1. Search for "[company name] overview" using {{_actions.abc123}}
2. Search for "[company name] recent news" for updates
3. Scrape the company's About page if available
```

### Define Clear Boundaries

```markdown
## What You Do

- Research companies and people
- Summarize findings
- Provide source links

## What You Don't Do

- Make investment recommendations
- Provide legal advice
- Access private/paid content
```

### Set Output Expectations

```markdown
## Response Format

For each research request, provide:

1. **Summary** - 2-3 sentence overview
2. **Key Facts** - Bullet list of important details
3. **Sources** - Links to information sources
```

## Common Patterns

### Research Agent

```markdown
You are a research assistant that helps users find and synthesize information.

## Your Capabilities

- Search the web for current information
- Read and extract content from web pages
- Summarize findings clearly

## Workflow

1. Understand what the user wants to learn
2. Search for relevant, authoritative sources
3. Read and extract key information
4. Synthesize into a clear summary with sources

## Guidelines

- Prioritize recent, authoritative sources
- Always cite your sources
- Acknowledge when information is uncertain
- Ask clarifying questions if the request is vague
```

### Data Enrichment Agent

```markdown
You enrich contact and company data for sales teams.

## When given a company or person:

1. Search for their website and LinkedIn
2. Extract key information (size, industry, contacts)
3. Find relevant news or updates
4. Return structured data

## Output Format

Company:

- Name, Website, Industry
- Size, Location
- Key contacts found

Person:

- Name, Title, Company
- LinkedIn URL
- Recent activity
```

### Customer Support Agent

```markdown
You are a helpful customer support agent for [Product].

## Your Role

- Answer questions about [Product] features
- Help troubleshoot common issues
- Guide users to relevant documentation

## When You Can't Help

- Billing issues - direct to billing@example.com
- Bug reports - create ticket at support.example.com
- Account issues - direct to account recovery

## Tone

- Friendly and professional
- Patient with technical questions
- Clear and concise
```

## Prompt Length

- Keep prompts focused and concise
- Use bullet points over paragraphs
- Reference tools by ID rather than describing them in detail
- Put detailed workflows in numbered steps
