---
description: Instructs agents to report bugs, skill gaps, and UX friction via the SubmitBugReport API. Use when encountering issues during agent conversations.
---

# Bug Reporting

When a significant issue is encountered during a conversation, the agent should analyze what went wrong and submit a structured report via the bug report API.

## Reporting Limits

- **Maximum 2 reports per conversation.** After 2 reports, stop reporting — batch remaining issues into a single summary if needed.
- **Never report failures of the reporting endpoint itself.** If `POST /bugs/submit` fails, do not retry or report the failure — this prevents infinite recursion.
- **Deduplicate within a conversation.** Before submitting, check whether you already reported an issue with the same `category` + `skill_or_tool` combination. If so, skip it.

## When to Report

Report when ANY of the following occur during a conversation:

- **Bug**: A tool call failed, returned unexpected data, or the agent hit an error
- **Skill gap**: The agent lacked knowledge or a tool to fulfill the user's request
- **UX friction**: The user had to repeat themselves, correct the agent, or abandon a task
- **Feature request**: The user explicitly asked for something the platform doesn't support
- **Hallucination**: The agent fabricated capabilities, tool names, or configuration options

Do NOT report:

- Successful conversations with no issues
- User errors unrelated to the platform (e.g. typos in their own data)
- Issues already reported in the same conversation
- Failures of the `POST /bugs/submit` endpoint itself

## How to Report

Use the `relevance_api_request` MCP tool to call the `POST /bugs/submit` endpoint.

```typescript
relevance_api_request({
  method: 'POST',
  endpoint: '/bugs/submit',
  body: {
    source: 'mcp-auto',
    region: '<current region>',
    project: '<current project ID>',
    message: 'Detailed description of the issue',
    category: 'bug | skill-gap | ux-friction | feature-request | hallucination',
    context: {
      severity: 'low | medium | high | critical',
      title: 'Short summary (under 100 chars)',
      suggested_fix: 'Concrete suggestion for how to fix',
      agent_id: 'agent ID if available',
      conversation_id: 'conversation/task ID if available',
      skill_or_tool: 'MCP tool or skill involved',
    },
  },
});
```

> **Note:** `region` and `project` are required by the API. Use the region and project ID from the current MCP session context (e.g. `relevance_get_project_info`).

### Field Reference

| Field                     | Required | Notes                                                                                                  |
| ------------------------- | -------- | ------------------------------------------------------------------------------------------------------ |
| `source`                  | Yes      | Always `"mcp-auto"` for agent-generated reports                                                        |
| `region`                  | Yes      | Current region code (e.g. `"bcbe5a"`)                                                                  |
| `project`                 | Yes      | Current project ID                                                                                     |
| `message`                 | Yes      | Detailed description: what the user wanted, what happened, what should have happened                   |
| `category`                | Yes      | One of: `bug`, `skill-gap`, `ux-friction`, `feature-request`, `hallucination`                          |
| `context.severity`        | Yes      | `critical` = blocks core workflow, `high` = significant friction, `medium` = noticeable, `low` = minor |
| `context.title`           | Yes      | Short, actionable — like a ticket title                                                                |
| `context.suggested_fix`   | Yes      | Be specific — name the tool, skill, or code change needed                                              |
| `context.agent_id`        | No       | Include when the issue is agent-specific                                                               |
| `context.conversation_id` | No       | The task_id or conversation_id                                                                         |
| `context.skill_or_tool`   | No       | The specific MCP tool or skill that was involved                                                       |

### Sanitization Rules

Reports are stored persistently. **MUST NOT include:**

- API keys, tokens, passwords, or OAuth secrets
- Auth headers, session cookies, or bearer tokens
- Raw PII (emails, phone numbers, addresses) — use `[REDACTED]` placeholders
- Full stack traces — include only the error class/code and message
- Raw request/response bodies — summarize the shape and relevant fields only

```typescript
// WRONG — leaks secrets
message: 'Tool failed with header Authorization: Bearer sk-abc123...';

// CORRECT — redacted
message: 'Tool failed with 401 Unauthorized when calling Jira API. Auth header was present but rejected.';
```

### Writing a Good `message`

The `message` field is the most important part of the report. Structure it clearly so the engineering team can understand and reproduce the issue without follow-up questions.

**Use this format:**

```
## Summary
One-sentence description of what went wrong.

## Steps to Reproduce
1. What the user asked for
2. What tool/action was attempted
3. What parameters were used (redact secrets)

## Expected Behavior
What should have happened.

## Actual Behavior
What actually happened — include error messages (sanitized), HTTP status codes, or unexpected return values.

## Impact
What was the consequence — data loss, blocked workflow, user had to work around manually, etc.

## Recommendations
Concrete suggestions: name the specific tool, endpoint, or code change needed.
```

**Tips for actionable reports:**

- Lead with the user's intent, not just the error
- Include the exact MCP tool name and parameters that triggered the issue
- Note whether a workaround exists and what it is
- If data was lost or corrupted, describe what changed (before vs after)
- Be specific in recommendations — "add a list_templates tool" is better than "fix this"

### Example

```typescript
relevance_api_request({
  method: 'POST',
  endpoint: '/bugs/submit',
  body: {
    source: 'mcp-auto',
    region: 'bcbe5a',
    project: 'a3cd1b1efb8c-4da4-8728-81a8bc7333ca',
    message:
      '## Summary\nrelevance_api_request with POST /agents/upsert wiped entire agent config when attempting a partial update.\n\n## Steps to Reproduce\n1. User asked to update only the system_prompt of agent 5f8e0e41\n2. Called relevance_api_request with POST /agents/upsert, body: { agent_id, partial_update: true, system_prompt: "test" }\n3. partial_update: true was silently ignored\n\n## Expected Behavior\nOnly system_prompt should change; other fields (actions, model, knowledge) should be preserved.\n\n## Actual Behavior\nEntire agent config replaced — actions dropped from 2 to 0, system_prompt went from 42K chars to 4 chars, model reset to default.\n\n## Impact\nProduction agent actively processing SDR call transcripts was wiped. Recovered via version rollback.\n\n## Recommendations\n1. Block /agents/upsert from relevance_api_request — use relevance_patch_agent instead\n2. Make partial_update actually work, or reject it with a validation error',
    category: 'bug',
    context: {
      severity: 'critical',
      title: 'POST /agents/upsert via raw API silently wipes agent config',
      suggested_fix:
        'Block /agents/upsert from relevance_api_request allowlist. Dedicated tools (relevance_patch_agent) already handle safe updates with fetch-merge-save pattern.',
      agent_id: '5f8e0e41-98bf-4068-90ed-2a722fb68466',
      skill_or_tool: 'relevance_api_request',
    },
  },
});
```

## Severity Guide

| Severity   | Criteria                                                       | Example                                                   |
| ---------- | -------------------------------------------------------------- | --------------------------------------------------------- |
| `critical` | Core workflow completely blocked, data loss, or security issue | Tool execution crashes the agent loop                     |
| `high`     | Significant user friction, workaround is painful               | Agent can't attach tools — user must do it manually in UI |
| `medium`   | Noticeable issue but workaround exists                         | Search returns wrong results but user can filter manually |
| `low`      | Minor inconvenience or cosmetic issue                          | Tool description is confusing but functionality works     |

## Triage Process

Reports submitted via `POST /bugs/submit` are stored server-side and triaged by the engineering team:

1. Filter reports where `source = 'mcp-auto'`
2. Assess severity and category
3. Convert actionable reports into Linear tickets
4. Group related reports to identify patterns
5. Prioritize based on frequency and severity
