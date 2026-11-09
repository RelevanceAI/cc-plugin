---
title: Running and Testing Tools
description: Run and debug tools — trigger-and-poll invocation, polling pacing, per-step testing, output inspection, common errors like timeouts and `variable_not_found`. Load when a tool fails or you need to test it end-to-end.
---

# Running and Testing Tools

How to execute tools, handle async operations, and debug issues.

## Running a Tool

Tools run asynchronously: trigger the run, then poll for the result. (Same trigger + poll model as agents and workforces.)

```typescript
// 1. Trigger the run — returns a job_id immediately
const { job_id } = await relevance_trigger_tool({
  studio_id: 'my-tool',
  params: { query: 'test input' },
});

// 2. Poll for the result — always pass wait_seconds
relevance_poll_tool_result({
  studio_id: 'my-tool',
  job_id: job_id,
  wait_seconds: 50,
});
```

### Polling Pacing

Always pass `wait_seconds` (default 50s, max 300s). The platform-API holds the connection open until the tool reaches a terminal `type` (`complete` or `failed`) or the wait window elapses, so one call covers many internal polls.

If a poll returns a non-terminal `type` (`inprogress` or `timeout`), **re-poll** — do NOT call `relevance_trigger_tool` again. Re-triggering starts a fresh job; the existing one is still running.

Work you start can outlast your turn, and every check on it is a tool call, so a turn that keeps waiting can end with no reply at all. Check on a run first: one check waits about its wait_seconds, and most runs finish within it. If it is still running, estimate when it will finish from its progress (an eval batch's completed_runs of total_runs) or from how long the same work took recently. If that is more than about 10 minutes away, or you have already waited about 10 minutes, stop checking: tell the user what is still running, give its link and when you expect it to finish, and offer to check back then if you can schedule one. If they would rather you keep waiting, keep waiting, but reply with what you have before your turn can run out.

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

> Never run something that writes just to try it, and that includes checking a fix: trigger the agent or workforce with `simulate: true`, naming in `live_action_ids` only tools that read, so every other tool returns a stand-in instead of running; or exercise the write in an eval that stands in for it; or run it only once the user has agreed to that specific run and named where it may write, such as a test recipient. After any run, a finished tool call whose `output_source` is missing or `tool-run` ran for real: say which of those sent or changed something, and where. A stand-in shows what it would send, not that it arrives, so call that write unproven. Running a tool directly has no `simulate` option, so test a writing tool's read steps with `relevance_run_transformation`. The runs below are for tools that only read, or for that agreed run.

> ⚠️ **A minimal run only proves the tool doesn't crash — not that it works the way an agent drives it.** After editing a tool, also test the branch the consumer actually takes (not just an early return) with every parameter it sends, on one or two items, and confirm the intended result; where that branch writes, use the route above, not a live run. Don't report "it works" off a smoke test alone.

### 1. Test with Minimal Input (smoke test)

```typescript
const { job_id } = await relevance_trigger_tool({
  studio_id: 'my-tool',
  params: { query: 'simple test' },
});
relevance_poll_tool_result({ studio_id: 'my-tool', job_id, wait_seconds: 50 });
```

### 1b. Test the real path with every parameter (after edits)

Run the tool the way the consumer invokes it — every param it sends on the path that matters, with one or two items — and check the result, not just that the call returned (a tool that writes takes the route above):

```typescript
const { job_id } = await relevance_trigger_tool({
  studio_id: 'my-tool',
  params: {
    /* EVERY field the consumer sends on the real path */
  },
});
relevance_poll_tool_result({ studio_id: 'my-tool', job_id, wait_seconds: 50 });
```

If the tool is consumed by an agent, prefer an eval, with a stand-in for each tool that writes, so the test reflects the arguments the agent actually assembles — see [relevance-evals](../relevance-evals/SKILL.md).

### 2. Test Each Step

If a multi-step tool fails, test its read steps individually with `relevance_run_transformation`; a write step takes the route above.
After a fix to a later step, test that step the same way first. Before asking the user for a full rerun, say what the last one cost: its `total_credits` from `relevance_list_tool_runs` or `relevance_get_tool_run`.

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
4. Treat a provider "not found" as a question about this request first. Read the step's options for widening the search and try the provider's generic authenticated API-call step. For Google Drive shared drives, check whether the step takes `drive_id` or `supports_all_drives`, then check whether an earlier project tool run reached the same folder before blaming access.

### Timeout

- Long-running tools may timeout
- Use async execution for tools > 30 seconds
- Consider breaking into smaller tools

### "Variable not found"

1. Check variable name spelling
2. Verify previous step actually outputs that variable
3. Check the previous step's `output` mapping as saved, not as you meant to write it — read it from the `edited` echo in your last step-edit response, or from `relevance_get_tool`
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

### Run a past run again

Pass its `task_id` as `rerun_task_id` and the tool sends that run's stored inputs. Don't copy inputs from `relevance_get_tool_run` into `params`: it hides credential-looking values, so a copy can differ from what was sent. Put only the inputs to change in `params`; each one replaces that input whole.

```typescript
relevance_trigger_tool({
  studio_id: 'my-tool-id',
  rerun_task_id: '<task_id from list>',
  params: { person_id: 'new-person-id' },
});
```

## Sharing links

Send back the `url` a tool returned rather than assembling one — the create/update/publish tools return the link that opens the tool's editor.

### Run History

View in the Relevance AI dashboard under tool settings.

## Best Practices

1. **Start simple** - Test with basic inputs first
2. **Check each step** - Review step outputs in response
3. **Use descriptive names** - Makes debugging easier
4. **Handle errors in Python** - Add try/catch blocks
5. **Log intermediate values** - Use print() in Python for debugging
6. **Test edge cases** - Empty inputs, special characters, etc.
