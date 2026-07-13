---
title: Running and Testing Tools
description: Run and debug tools — sync/async invocation, polling pacing, per-step testing, output inspection, common errors like timeouts and `variable_not_found`. Load when a tool fails or you need to test it end-to-end.
---

# Running and Testing Tools

How to execute tools, handle async operations, and debug issues.

## Synchronous Execution

For tools that complete quickly:

```typescript
relevance_run_tool({
  studio_id: 'my-tool',
  params: { query: 'test input' },
});
```

Returns the tool output directly.

## Asynchronous Execution

For long-running tools:

```typescript
// 1. Start async execution
const { job_id } = await relevance_trigger_tool_async({
  studio_id: 'my-tool',
  params: { query: 'test input' },
});

// 2. Poll for results — always pass wait_seconds
relevance_poll_tool_result({
  studio_id: 'my-tool',
  job_id: job_id,
  wait_seconds: 50,
});
```

### Polling Pacing

Always pass `wait_seconds` (default 50s, max 300s). The platform-API holds the connection open until the tool reaches a terminal `type` (`complete` or `failed`) or the wait window elapses, so one call covers many internal polls.

If a poll returns a non-terminal `type` (`inprogress` or `timeout`), **re-poll** — do NOT call `relevance_trigger_tool_async` again. Re-triggering starts a fresh job; the existing one is still running.

### Poll Response Types

| Type         | Meaning                            |
| ------------ | ---------------------------------- |
| `complete`   | Finished successfully (terminal)   |
| `failed`     | Execution failed (terminal)        |
| `inprogress` | Currently executing — re-poll      |
| `timeout`    | Server-side wait elapsed — re-poll |

## Getting Latest Run

Useful for getting example task IDs:

```typescript
relevance_get_latest_tool_run({
  studio_id: 'my-tool',
});
```

## Testing Workflow

> ⚠️ **A minimal run only proves the tool doesn't crash — not that it works the way an agent drives it.** After editing a tool, also test the branch the consumer actually takes (the success/write path, not just an early return) with the full argument set, and confirm the intended effect happened — don't report "it works" off a smoke test alone.

### 1. Test with Minimal Input (smoke test)

```typescript
relevance_run_tool({
  studio_id: 'my-tool',
  params: { query: 'simple test' },
});
```

### 1b. Test the real path with the full argument set (after edits)

Run the tool the way the consumer invokes it — every param it sends on the path that matters — and check the side effect landed, not just that the call returned:

```typescript
relevance_run_tool({
  studio_id: 'my-tool',
  params: {
    /* EVERY field the consumer sends on the real path */
  },
});
```

If the tool is consumed by an agent, prefer running an eval / a real agent conversation so the test reflects the arguments the agent actually assembles — see [relevance-evals](../relevance-evals/SKILL.md).

### 2. Test Each Step

If a multi-step tool fails, test transformations individually by creating a minimal tool with just that step.

### 3. Check Step Outputs

Examine the full response to see what each step returned:

```json
{
  "output": {
    "search": { "results": [...] },
    "analyze": { "analysis": "..." }
  },
  "steps": {
    "search": { "status": "completed", "output": {...} },
    "analyze": { "status": "completed", "output": {...} }
  }
}
```

### 4. Debug Python Steps

For Python steps, add print statements (they appear in logs):

```python
import json
data = {{input}}
print(f"Input type: {type(data)}")
print(f"Input value: {data}")
return {"result": data}
```

## Common Issues

### Tool Returns Empty Output

1. Check transformation outputs are mapped correctly
2. Verify `{{answer}}` vs `{{text}}` for prompt_completion
3. Check `{{transformed.field}}` for Python steps

### Step Failed

1. Check the step status in response
2. Look for error messages
3. Verify input parameters are correct type

### Timeout

- Long-running tools may timeout
- Use async execution for tools > 30 seconds
- Consider breaking into smaller tools

### "Variable not found"

1. Check variable name spelling
2. Verify previous step actually outputs that variable
3. Check output mapping in previous step
4. **Did you just rename a step?** Renaming doesn't rewrite references — search every later step's `params`, the tool `output` map, and `state_mapping` for the old `{{steps.<old_name>…}}` and update them. See [creating.md](creating.md#renaming-or-restructuring-a-step-update-references).

## Inspecting tool runs

### List recent runs for a tool

```typescript
relevance_list_tool_runs({ tool_id: 'my-tool-id', page_size: 10 });
```

### Get full details of a specific run

```typescript
relevance_get_tool_run({
  tool_id: 'my-tool-id',
  task_id: '<task_id from list>',
});
```

## URL Patterns

### Tool Editor

```
https://app.relevanceai.com/notebook/{region}/{project}/{studioId}
```

Open this to see/edit the tool visually.

### Run History

View in the Relevance AI dashboard under tool settings.

## Best Practices

1. **Start simple** - Test with basic inputs first
2. **Check each step** - Review step outputs in response
3. **Use descriptive names** - Makes debugging easier
4. **Handle errors in Python** - Add try/catch blocks
5. **Log intermediate values** - Use print() in Python for debugging
6. **Test edge cases** - Empty inputs, special characters, etc.
