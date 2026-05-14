---
title: Running and Testing Agents
description: Trigger agents, poll for results, inspect conversations and tool calls. Load when running or testing an agent, or debugging tool-execution failures inside a conversation.
---

# Running and Testing Agents

How to trigger agents, manage conversations, and view results.

## Triggering Agents

### Trigger Agent

Use `relevance_trigger_agent` to send a message. Returns immediately with a `conversation_id`:

> **Note:** `conversation_id` (returned by trigger/poll tools) and `task_id` (used by view/metadata tools) are the same value. Use the `conversation_id` from a trigger result as the `task_id` when calling `relevance_get_agent_task_summary`, `relevance_list_agent_task_messages`, or `relevance_get_agent_task_metadata`.

```typescript
const result = await relevance_trigger_agent({
  agent_id: '...',
  message: 'Research the latest AI developments',
});
// Returns { conversation_id } immediately — this is the task_id for other agent task tools
// Use relevance_poll_agent_result to check status and get the response
```

### Polling Pacing

When polling for results with `relevance_poll_agent_result`, **always pass `wait_seconds`** (default 50s, max 300s). The platform-API holds the connection open until the agent reaches a terminal status (`completed`, `failed`, `pending_approval`) or the wait window elapses, so one call covers many internal polls.

```typescript
relevance_poll_agent_result({
  agent_id: '...',
  conversation_id: '...',
  wait_seconds: 50,
});
```

If a poll returns a non-terminal `status: "in_progress"`, **re-poll** — do NOT call `relevance_trigger_agent` again. Re-triggering starts a new conversation; the existing one is still running.

A `polling_hint` field on the response only appears for `pending_approval` and includes the conversation URL the human approver must visit.

### Wait for Completion

There is no synchronous trigger. To wait for the agent's response, call `relevance_poll_agent_result` with `wait_seconds` after triggering — the platform-API holds the connection open until a terminal status (`completed`, `failed`, `pending_approval`) or the wait window elapses.

```typescript
const { conversation_id } = await relevance_trigger_agent({
  agent_id: '...',
  message: 'Research the latest AI developments',
});
const task = await relevance_poll_agent_result({
  agent_id: '...',
  conversation_id,
  wait_seconds: 50,
});
// Re-poll while task.status === "in_progress"
```

### Continue Existing Conversation

```typescript
relevance_trigger_agent({
  agent_id: '...',
  conversation_id: 'previous-conversation-id',
  message: 'Tell me more about transformers',
});
```

## Viewing Agent Tasks

### Get Task Summary

```typescript
relevance_get_agent_task_summary({
  agent_id: '...',
  task_id: 'conversation-id', // Use the conversation_id as task_id
});
```

Returns summarized view with:

- Status
- User messages
- Agent's final response
- Tool calls with output previews
- Extracted artifacts (images, files, PDFs)

### Full Mode

## Testing Workflow

### 0. Validate Tools First

Before testing the agent, validate each attached tool individually to catch broken output configs:

```typescript
const result = await relevance_run_tool({
  studio_id: 'tool-id',
  params: { query: 'test input' },
});
// ✅ Returns meaningful data → tool works
// ❌ Returns {} or empty → fix output config before agent testing
```

This prevents wasting agent credits on tools that silently return empty results.

### 1. Basic Functionality Test

```typescript
// Simple message to verify agent responds
relevance_trigger_agent({
  agent_id: '...',
  message: 'Hello, what can you help me with?',
});
```

### 2. Tool Usage Test

```typescript
// Message that requires tool use
relevance_trigger_agent({
  agent_id: 'research-assistant',
  message: 'Search for recent news about OpenAI',
});
```

### 3. Multi-Turn Test

```typescript
// Start conversation
const result1 = await relevance_trigger_agent({
  agent_id: '...',
  message: "Research Tesla's latest quarterly earnings",
});

// Continue with follow-up
relevance_trigger_agent({
  agent_id: '...',
  conversation_id: result1.conversation_id,
  message: 'How does this compare to last quarter?',
});
```

## Debugging Agent Execution

### Check Conversation for Errors

```typescript
const task = await relevance_get_agent_task_summary({
  agent_id: '...',
  task_id: 'conversation-id', // conversation_id is the task_id
});

// Look for:
// - status: "failed" or "error"
// - Tool calls that failed
// - Error messages in agent response
```

### Common Issues

| Symptom                       | Likely Cause                     | Fix                                                            |
| ----------------------------- | -------------------------------- | -------------------------------------------------------------- |
| Agent doesn't use tools       | System prompt unclear            | Make tool instructions explicit                                |
| Tool execution fails          | Missing project/region           | Add to action config                                           |
| Wrong tool called             | Tool descriptions unclear        | Update action descriptions                                     |
| Slow response                 | Too many tools                   | Remove unused tools                                            |
| `pending_approval` status     | `action_behaviour: "always-ask"` | Approve in UI, or change to `"never-ask"` for automated use    |
| `in_progress` after long wait | Agent still running              | Re-poll with `relevance_poll_agent_result` — do NOT re-trigger |

## URL Patterns

### Conversation View

```
https://app.relevanceai.com/agents/{region}/{project}/{agentId}/{taskId}

Example:
https://app.relevanceai.com/agents/bcbe5a/your-project/my-agent/task-123
```

### Agent Edit Page

```
https://app.relevanceai.com/agents/{region}/{project}/{agentId}/edit/instructions
```

## API Direct Access

### List Tasks (Conversations)

Use `relevance_list_agent_tasks` to list agent tasks (conversations).

### List Task Messages

Use `relevance_list_agent_task_messages` to get raw messages from an agent task:

```typescript
const messages = await relevance_list_agent_task_messages({
  agent_id: '...',
  task_id: 'task-id',
});
// Returns raw message list from the task
```

### Get Task Metadata

Use `relevance_get_agent_task_metadata` to get task metadata (status, credits used, runtime):

```typescript
const metadata = await relevance_get_agent_task_metadata({
  agent_id: '...',
  task_id: 'task-id',
});
// Returns status, credits consumed, runtime duration, etc.
```

### List Jobs in Conversation

```typescript
relevance_api_request({
  method: 'POST',
  endpoint: `/agents/${agentId}/tasks/${taskId}/jobs/list`,
  body: { page_size: 50 },
});
```

## Dry Run / Safe Testing Pattern

For agents with potentially destructive tools (e.g. sending emails, deleting data), set `action_behaviour: "always-ask"` on those tools using `relevance_update_agent` during testing.

When triggered, the agent will pause at `pending_approval` status before executing the tool. You can:

1. Review the tool call parameters in the Relevance AI app (use `relevance_get_project_info` for the URL)
2. Approve or reject the call
3. Use `relevance_get_agent_task_summary` to see the result after approval

Once verified, switch back to `"never-ask"` for production use.

## Best Practices

1. **Test incrementally** - Start with simple messages before complex workflows
2. **Check tool calls** - Use task view to see which tools were called
3. **Verify tool outputs** - Ensure tools return expected data
4. **Test with diverse inputs** - Don't rely on a single test; use varied inputs to reveal edge cases
5. **Monitor for timeouts** - Long tool operations may timeout
6. **Never re-trigger on timeout or pending_approval** - Use `relevance_get_agent_task_summary` to poll instead
7. **Report issues proactively** — if a tool call fails unexpectedly, the agent misbehaves, or you spot a "we should add X" gap, call `relevance_submit_feedback` immediately with the matching `category`. See [report-bugs.md](report-bugs.md) for the call shape.
