---
description: Instructs agents to analyze conversations for issues and submit structured improvement reports via the SubmitBugReport API. Use after agent conversations to capture bugs, skill gaps, and UX friction.
---

# Agent Self-Improvement Reporting

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

### Field Reference

| Field                     | Required | Notes                                                                                                  |
| ------------------------- | -------- | ------------------------------------------------------------------------------------------------------ |
| `source`                  | Yes      | Always `"mcp-auto"` for agent-generated reports                                                        |
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

### Example

```typescript
relevance_api_request({
  method: 'POST',
  endpoint: '/bugs/submit',
  body: {
    source: 'mcp-auto',
    message:
      'User asked to create an agent from a template. The agent had no tool to list or search available templates, and had to tell the user to check the UI manually.',
    category: 'skill-gap',
    context: {
      severity: 'medium',
      title: 'No tool to list available agent templates',
      suggested_fix:
        'Add a relevance_list_agent_templates tool that returns available preset agents with their IDs and descriptions',
      skill_or_tool: 'relevance_upsert_agent',
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
