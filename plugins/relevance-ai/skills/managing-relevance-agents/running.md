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
// Returns { conversation_id, ... } immediately — conversation_id is the
// task_id for other agent task tools. The response also points at the page
// showing this run's output. Use relevance_poll_agent_result for the result.
```

The link on this response is the one to give the user for this run's result. It is not the agent's builder page — see [Sharing links](#sharing-links).

One call starts one run. To run an agent once for each of many items, don't call it once per item past a handful (five or fewer): see [Work Over Many Items](many-items.md).

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

A poll has `agentResponse` only once the run completes or fails. Before that, `last_agent_text` is the newest thing the agent said, which may be hours old. Report progress from `latest_activity`: it prefers running or pending tools over newer finished calls, otherwise the newest message. Its `at` is the message creation or tool request time, not the last state change. If `context_compression_failed` is set, the platform couldn't compress the run's history. Once that outgrows the model's context, the agent stops seeing older messages, so it can redo steps or lose what earlier steps found. For what to do, see `relevance-diagnostics`, "A run stopped seeing older messages".

Work you start can outlast your turn, and every check on it is a tool call, so a turn that keeps waiting can end with no reply at all. Check on a run first: one check waits about its wait_seconds, and most runs finish within it. If it is still running, estimate when it will finish from its progress (an eval batch's completed_runs of total_runs) or from how long the same work took recently. If that is more than about 10 minutes away, or you have already waited about 10 minutes, stop checking: tell the user what is still running, give its link and when you expect it to finish, and offer to check back then if you can schedule one. If they would rather you keep waiting, keep waiting, but reply with what you have before your turn can run out. An earlier run's `active_execution_time_ms` from `relevance_get_agent_task_metadata` is the best guide to how long this one will take.

A poll response points at the page showing this run's output, so you always have the link to hand back when you report the result — you do not have to keep the one from the trigger.

A `polling_hint` field on the response only appears for `pending_approval` and includes the conversation URL the human approver must visit. To decide on a waiting task yourself, see [Deciding on a waiting task](#deciding-on-a-waiting-task).

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

### Continue an Existing Conversation

A conversation is a thread: pass `conversation_id` and the agent sees every earlier turn. Omit it and you get a brand-new thread with no memory of what came before.

**Default to continuing.** When the user builds on what the agent just did — "try again", "now with an example", "what about Q3?" — reuse the `conversation_id` from the previous trigger. Start a fresh thread only when the user asks to start over, or when the next request is a genuinely separate unit of work (a different subject, customer, or document) where the earlier context would mislead the agent. When in doubt, continue the existing thread.

```typescript
const first = await relevance_trigger_agent({
  agent_id: '...',
  message: "Research Tesla's Q3 earnings",
});
// …poll to a terminal status…

// Follow-up in the SAME thread — the agent still has the Q3 research in context
relevance_trigger_agent({
  agent_id: '...',
  conversation_id: first.conversation_id,
  message: 'How does that compare to last quarter?',
});
```

> Continuing a finished conversation is not the same as re-triggering a running one. While a run is `in_progress` or `pending_approval`, don't trigger at all — re-poll (see [Polling Pacing](#polling-pacing)). Once it reaches a terminal status, a follow-up should trigger again with the same `conversation_id`.

## Deciding on a waiting task

A task at `pending_approval` (or `escalated`, while an escalation card waits) is waiting for a person to approve or reject a tool call, or to answer an escalation. The decision is the user's: give them the task's link unless they have asked you to decide.

If they have, and `relevance_submit_agent_task_decisions` is among your tools, send every decision in one call:

```typescript
relevance_submit_agent_task_decisions({
  agent_id: '...',
  task_id: '...',
  decisions: [
    { action_request_id: '...', decision: 'approve' },
    { action_request_id: '...', decision: 'reject' },
    {
      action_request_id: '...',
      decision: 'respond',
      response_text: 'Use the Q3 numbers',
    },
  ],
});
```

- `relevance_get_agent_task_metadata` lists what the task waits on under `waiting_for`, each item with its `action_request_id`, the decision it takes, and the tool's inputs or the escalation's question. A call that misses one, or names one that isn't waiting, changes nothing and lists them again.
- Every waiting item needs a decision, because they are sent together and anything left out would be dropped unanswered.
- `approve` and `reject` apply to tool calls and `respond` to escalations; a field the decision doesn't take is refused.
- `approve` runs the inputs as the agent set them. To change one, reject it and tell the agent what to change with `relevance_trigger_agent`; it asks again with a new approval.
- An input shown redacted, cut, or only by name under `omitted_inputs` (with `omitted_input_count`) is approved as it is; the reply lists these under `approved_unseen`, so tell the user. An action missing a required input can only be approved in the app.
- A sub-agent's items are decided from the task that started it; the tool names that task.
- An `escalated` task with nothing in `waiting_for` is answered by messaging it: `relevance_trigger_agent` with its `conversation_id`.

Then re-poll. Without the tool, the user decides in the app.

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

## Testing Workflow

### 0. Validate Tools First

Before testing the agent, validate each attached tool that only reads to catch broken output configs (a tool that writes is never run just to try it; see the tools skill's Testing Workflow):

```typescript
const { job_id } = await relevance_trigger_tool({
  studio_id: 'tool-id',
  params: { query: 'test input' },
});
const result = await relevance_poll_tool_result({
  studio_id: 'tool-id',
  job_id,
  wait_seconds: 50,
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

| Symptom                       | Likely Cause                     | Fix                                                                                                                                                     |
| ----------------------------- | -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Agent doesn't use tools       | System prompt unclear            | Make tool instructions explicit                                                                                                                         |
| Tool execution fails          | Missing project/region           | Add to action config                                                                                                                                    |
| Wrong tool called             | Tool descriptions unclear        | Update action descriptions                                                                                                                              |
| Slow response                 | Too many tools                   | Remove unused tools                                                                                                                                     |
| `pending_approval` status     | `action_behaviour: "always-ask"` | The user decides in the app (or you, if asked: [Deciding on a waiting task](#deciding-on-a-waiting-task)), or change to `"never-ask"` for automated use |
| `in_progress` after long wait | Agent still running              | Re-poll with `relevance_poll_agent_result` — do NOT re-trigger                                                                                          |

## Sharing links

Never assemble an app URL yourself — send back a url a tool returned. Assembling one from the ids you happen to have produces a link that resolves to a page but shows the wrong thing (a task id that doesn't belong to the agent renders an empty task, not an error).

Pick the link by what the user asked for:

| The user wants                | Give them the link from                                  |
| ----------------------------- | -------------------------------------------------------- |
| the result of a run           | the trigger or poll response for that run                |
| the agent's config            | a create/update/publish response                         |
| to connect/fix an integration | `setup_url` / `config_url` (integration-check responses) |

Where a response names the link — `task_url` for a run, `edit_url` for the builder — use the name rather than guessing; otherwise the tool you called tells you which it is.

A builder link opens whatever version the builder loads by default, which is the **draft** when an unpublished draft exists — so it is how a user reads a draft, but it is never the link to a run's output.

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

Use `relevance_get_agent_task_metadata` to get task metadata (status, runtime, whether it errored):

```typescript
const metadata = await relevance_get_agent_task_metadata({
  agent_id: '...',
  task_id: 'task-id',
});
// Returns status, runtime and whether it errored.
```

## Trying an agent whose tools write

> Never run something that writes just to try it, and that includes checking a fix: trigger the agent or workforce with `simulate: true`, naming in `live_action_ids` only tools that read, so every other tool returns a stand-in instead of running; or exercise the write in an eval that stands in for it; or run it only once the user has agreed to that specific run and named where it may write, such as a test recipient. After any run, a finished tool call whose `output_source` is missing or `tool-run` ran for real: say which of those sent or changed something, and where. A stand-in shows what it would send, not that it arrives, so call that write unproven.

```typescript
relevance_trigger_agent({
  agent_id: '...',
  message: '...',
  simulate: true,
  live_action_ids: ['<action_id of a tool that only reads>'],
});
// In the poll, a finished toolCalls entry without output_source ran for real.
```

The agent's notification rules still send on a simulated run's status changes. A simulated trigger saves no scenario, so it doesn't guard later publishes: that stays the `tool_simulation_config` on a saved scenario's test set, as [relevance-evals](../relevance-evals/simulation.md) describes. `action_behaviour: "always-ask"` makes a real run stop at that tool's approval: it is the setting for an agent that should act only with approval, not a switch to flip for a test and back.

## Best Practices

1. **Test incrementally** - Start with simple messages before complex workflows
2. **Check tool calls** - Use task view to see which tools were called
3. **Verify tool outputs** - Ensure tools return expected data
4. **Test with diverse inputs** - Don't rely on a single test; use varied inputs to reveal edge cases
5. **Monitor for timeouts** - Long tool operations may timeout
6. **Never re-trigger on timeout or pending_approval** - Use `relevance_get_agent_task_summary` to poll instead
7. **Report platform issues** — when a failure is the platform's (not the input, not your own config), or you spot a "we should add X" gap, call `relevance_submit_feedback` with the matching `category`, without asking the user for permission. See [report-bugs.md](report-bugs.md) for the call shape.
