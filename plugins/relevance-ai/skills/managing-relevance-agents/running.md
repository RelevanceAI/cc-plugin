# Running and Testing Agents

How to trigger agents, manage conversations, and view results.

## Triggering Agents

### Trigger Agent

Use `relevance_trigger_agent` to send a message. Returns immediately with a `conversation_id`:

```typescript
const result = await relevance_trigger_agent({
  agentId: '...',
  message: 'Research the latest AI developments',
});
// Returns { conversation_id } immediately
// Use relevance_poll_agent_result to check status and get the response
```

### Synchronous (Wait for Completion)

Use `relevance_trigger_agent_sync` to trigger and wait for completion (up to 120s):

```typescript
const result = await relevance_trigger_agent_sync({
  agentId: '...',
  message: 'Research the latest AI developments',
});
// Returns { status, response, toolCalls } when done
```

### Continue Existing Conversation

```typescript
relevance_trigger_agent({
  agentId: '...',
  conversationId: 'previous-conversation-id',
  message: 'Tell me more about transformers',
});
```

## Viewing Conversations

### Get Conversation Details

```typescript
relevance_get_task_view({
  agentId: '...',
  taskId: 'conversation-id',
});
```

Returns summarized view with:

- Status
- User messages
- Agent's final response
- Tool calls with output previews
- Extracted artifacts (images, files, PDFs)

### Full Mode

For complete raw API response:

```typescript
relevance_get_task_view({
  agentId: '...',
  taskId: 'conversation-id',
  fullMode: true,
});
```

## Testing Workflow

### 0. Validate Tools First

Before testing the agent, validate each attached tool individually to catch broken output configs:

```typescript
const result = await relevance_run_tool({
  studioId: 'tool-id',
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
  agentId: '...',
  message: 'Hello, what can you help me with?',
});
```

### 2. Tool Usage Test

```typescript
// Message that requires tool use
relevance_trigger_agent({
  agentId: 'research-assistant',
  message: 'Search for recent news about OpenAI',
});
```

### 3. Multi-Turn Test

```typescript
// Start conversation
const result1 = await relevance_trigger_agent({
  agentId: '...',
  message: "Research Tesla's latest quarterly earnings",
});

// Continue with follow-up
relevance_trigger_agent({
  agentId: '...',
  conversationId: result1.conversation_id,
  message: 'How does this compare to last quarter?',
});
```

## Debugging Agent Execution

### Check Conversation for Errors

```typescript
const task = await relevance_get_task_view({
  agentId: '...',
  taskId: 'conversation-id',
});

// Look for:
// - status: "failed" or "error"
// - Tool calls that failed
// - Error messages in agent response
```

### Common Issues

| Symptom                   | Likely Cause                     | Fix                                                         |
| ------------------------- | -------------------------------- | ----------------------------------------------------------- |
| Agent doesn't use tools   | System prompt unclear            | Make tool instructions explicit                             |
| Tool execution fails      | Missing project/region           | Add to action config                                        |
| Wrong tool called         | Tool descriptions unclear        | Update action descriptions                                  |
| Slow response             | Too many tools                   | Remove unused tools                                         |
| `pending_approval` status | `action_behaviour: "always-ask"` | Approve in UI, or change to `"never-ask"` for automated use |
| `timed_out` status        | Agent takes >120s                | Use `relevance_get_task_view` to poll — do NOT re-trigger   |

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

### List Conversations

Use `relevance_list_conversations` to list agent conversations.

### List Jobs in Conversation

```typescript
relevance_api_request({
  method: 'POST',
  endpoint: `/agents/${agentId}/tasks/${taskId}/jobs/list`,
  body: { page_size: 50 },
});
```

## Task View Pagination

`relevance_get_task_view` supports cursor-based pagination for handling large conversations:

```typescript
// First request
const page1 = await relevance_get_task_view({
  agentId: 'my-agent',
  taskId: 'conversation-id',
  pageSize: 100,
});
// Response includes next_cursor if more messages exist

// Get older messages using cursor
if (page1.next_cursor) {
  const page2 = await relevance_get_task_view({
    agentId: 'my-agent',
    taskId: 'conversation-id',
    pageSize: 100,
    cursor: { before: page1.next_cursor }, // Get messages BEFORE this timestamp
  });
}
```

**Cursor options:**

- `cursor.before` - Get messages before this ISO timestamp (for going back in time)
- `cursor.after` - Get messages after this ISO timestamp (for new messages)

## Dry Run / Safe Testing Pattern

For agents with potentially destructive tools (e.g. sending emails, deleting data), set `action_behaviour: "always-ask"` on those tools using `relevance_patch_agent` during testing.

When triggered, the agent will pause at `pending_approval` status before executing the tool. You can:

1. Review the tool call parameters in the Relevance AI app (use `relevance_get_project_info` for the URL)
2. Approve or reject the call
3. Use `relevance_get_task_view` to see the result after approval

Once verified, switch back to `"never-ask"` for production use.

## Best Practices

1. **Test incrementally** - Start with simple messages before complex workflows
2. **Check tool calls** - Use task view to see which tools were called
3. **Verify tool outputs** - Ensure tools return expected data
4. **Test with diverse inputs** - Don't rely on a single test; use varied inputs to reveal edge cases
5. **Monitor for timeouts** - Long tool operations may timeout
6. **Never re-trigger on timeout or pending_approval** - Use `relevance_get_task_view` to poll instead
