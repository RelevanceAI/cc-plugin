---
description: How agents file bugs and improvement suggestions through MCP. Submit proactively whenever a tool errors, a capability is missing, or the user asks for something the platform doesn't support.
---

# Reporting Bugs and Suggesting Improvements

Use the `relevance_submit_feedback` MCP tool. One tool covers both bugs and forward-looking suggestions — the `category` field tells engineering which it is. Region/project/source are inferred from the session. **Submit proactively** — do NOT ask the user for permission first.

## When to call

| Category          | When                                                                     |
| ----------------- | ------------------------------------------------------------------------ |
| `bug`             | A tool call failed, returned wrong data, or threw an error               |
| `skill-gap`       | The agent lacked a tool/capability and the user gave up or worked around |
| `ux-friction`     | The user had to repeat themselves or correct the agent                   |
| `hallucination`   | The agent fabricated a tool name, parameter, or capability               |
| `feature-request` | The user asked for something the platform doesn't support yet            |
| `improvement`     | An existing capability could be better; you worked around a limitation   |
| `docs`            | A skill doc was missing the answer you needed                            |

**Don't report:** successful conversations, user errors unrelated to the platform (typos in their data), or the same `category` + `title` twice in one conversation. Don't report failures of the feedback tool itself (prevents recursion).

## Calling the tool

```typescript
relevance_submit_feedback({
  category: 'bug', // see table above
  severity: 'high', // critical | high | medium | low
  title:
    'relevance_update_agent dropped all attached tools on a partial update',
  message:
    'User asked to change only the system_prompt of agent 5f8e0e41. ' +
    'Called relevance_update_agent with just { agent_id, system_prompt }. ' +
    'The attached tools were wiped (actions dropped from 2 to 0) even though I only sent system_prompt — the partial-merge did not preserve actions.',
  agent_id: '5f8e0e41-98bf-4068-90ed-2a722fb68466',
  skill_or_tool: 'relevance_update_agent',
});
```

### Field reference

| Field             | Required | Notes                                                                |
| ----------------- | -------- | -------------------------------------------------------------------- |
| `category`        | Yes      | One of the seven categories above                                    |
| `severity`        | Yes      | `critical` blocks, `high`/`medium` is friction, `low` is minor       |
| `title`           | Yes      | Short ticket-style title (5–120 chars)                               |
| `message`         | Yes      | What happened, in plain prose. ≥20 chars. Don't prescribe a fix.     |
| `agent_id`        | No       | Agent ID if issue is agent-specific                                  |
| `conversation_id` | No       | Conversation/task ID if already in your context — never ask the user |
| `skill_or_tool`   | No       | The MCP tool or skill involved                                       |

## Writing a good `message`

Keep it short. A few sentences is usually enough. Cover, in plain prose:

- What the user wanted
- What you tried (the specific tool / parameters)
- What went wrong, or what's missing

**Do not** prescribe how engineering should fix it — that's their call. Describing the _symptom_ well is more useful than guessing the _fix_.

If a workaround exists, mention it in one line.

## Sanitization

Reports are stored persistently. **Never include:**

- API keys, tokens, passwords, OAuth secrets, Authorization headers
- Raw PII (emails, phone numbers, addresses) — use `[REDACTED]`
- Full stack traces — error class + message is enough
- Raw request/response bodies — summarize the shape

The tool will reject the call with a sanitization error if obvious secret patterns are detected.

```typescript
// WRONG — leaks secrets
message: 'Tool failed with header Authorization: Bearer sk-abc123...';

// CORRECT — redacted
message: 'Tool failed with 401 Unauthorized when calling Jira API.';
```

## Severity guide

| Severity   | Criteria                                                       | Example                                                   |
| ---------- | -------------------------------------------------------------- | --------------------------------------------------------- |
| `critical` | Core workflow completely blocked, data loss, or security issue | Tool execution crashes the agent loop                     |
| `high`     | Significant user friction, workaround is painful               | Agent can't attach tools — user must do it manually in UI |
| `medium`   | Noticeable issue but workaround exists                         | Search returns wrong results but user can filter manually |
| `low`      | Minor inconvenience or cosmetic issue                          | Tool description is confusing but functionality works     |

## Triage

Reports are stored server-side and triaged by engineering — filter on `source = 'mcp-auto'`, group by `category` and frequency, convert actionable ones into Linear tickets. Per-project rate limiting on the feedback tool keeps the volume sane.
