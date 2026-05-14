---
title: Troubleshooting Agents
description: Diagnose common agent failures — draft vs live confusion, region mismatches, `action_behaviour` blocking, autonomy limits, trigger failures. Load when an agent is misbehaving or a tool/trigger isn't running as expected.
---

# Troubleshooting Agents

Common issues and fixes for Relevance AI agents.

## Debugging Error Chains

Work backwards from symptoms to root cause. Don't fix symptoms -- fix root causes.

```
ERROR LOCATION          ACTUAL CAUSE
Step N: API call        Step 0: Agent/Trigger
(parameter missing)     (value never passed)
```

**Common root causes:**

| Symptom                   | Often Actually Caused By                                                        |
| ------------------------- | ------------------------------------------------------------------------------- |
| API parameter missing     | Agent didn't pass it in tool call                                               |
| `KeyError: 'transformed'` | Previous Python step errored or returned wrong type                             |
| Empty string in API call  | Upstream step received empty input                                              |
| `required property` error | params_schema mismatch in workforce edge                                        |
| Tool returns empty `{}`   | Output config not mapped (see creating.md "Fixing Auto-Generated Tool Outputs") |

## Critical Issues

### Edits Don't Show Up in Production — Draft Wasn't Published

**Symptom:** You called `relevance_update_agent` or `relevance_attach_tools_to_agent`, the call succeeded, but production behaviour hasn't changed.

**Cause:** Those tools save to a DRAFT only. The active version is still the previously-published one (or undefined if `relevance_create_agent` hasn't been published yet).

**Fix:** Confirm with the user, then call `relevance_publish_agent({ agent_id })`. The publish tool always shows an approval card (even with auto-approve enabled) — that's intentional.

**Inspecting state:** Call `relevance_get_agent` and check `has_unpublished_draft`, `active_version_id`, and `draft_version_id` on the response.

**Testing the draft without publishing:** Call `relevance_trigger_agent` — it defaults to the draft when one exists. The response's `triggered_version` field tells you which version actually ran. Pass `version: "draft"` to be explicit.

### Agent Max Output Tokens — 422 Error

**Symptom:** Setting `max_output_tokens` or `max_tokens` on an agent returns a 422 error.

**Cause:** `max_output_tokens` is NOT a top-level agent field — it lives inside `model_options`.

**Fix:** Call `relevance_update_agent` with `patch: { model_options: { max_output_tokens: 16000 } }`. See [creating.md](creating.md) for the full `model_options` reference.

### Agent Says "I'll Run the Tool" But Nothing Happens

**Symptom:** Agent announces it will use a tool but never actually executes it.

**Cause:** `action_behaviour` is set to `"always-ask"`, which requires user approval before each tool call.

**Fix:** Call `relevance_update_agent_action` with the tool's `chain_id` and `patch: { action_behaviour: "never-ask" }`. The agent-level `action_behaviour` set via `relevance_update_agent` is a fallback default — most cases need the per-tool override.

### Agent Runs Tool Then Silently Stops

**Symptom:** Tool executes successfully but agent ends conversation without providing results.

**Cause:** `autonomy_limit_behaviour: "terminate-conversation"` causes the agent to silently end instead of responding after reaching its limit.

**Fix:** Call `relevance_update_agent` with `patch: { autonomy_limit: 25, autonomy_limit_behaviour: "ask-for-approval" }`. The agent will then pause and ask to continue rather than silently stopping.

### Tools Not Executing

**Symptom:** Agent acknowledges tools but doesn't use them, or tool calls fail.

**Cause 1:** Missing `project` and `region` in action config. `relevance_attach_tools_to_agent` sets these automatically — if they're missing, the action was likely added by hand and is broken. Re-attach with `relevance_attach_tools_to_agent`, or call `relevance_update_agent_action` with `patch: { project, region }`.

**Cause 2:** Tool is disabled. Call `relevance_update_agent_action` with `patch: { disabled: false }`.

**Cause 3:** `action_behaviour: "always-ask"` blocking execution (see above).

### System Prompt Tool References Not Working

**Symptom:** `{{_actions.xxx}}` not resolving in system prompt.

**Cause:** Using wrong action ID.

**Fix:** Use backend-generated IDs, not your custom IDs. Call `relevance_get_agent_tools` and read the `actionIdMap` values into your system prompt.

### Agent Cloning Fails with "additional properties" Error

**Symptom:** Error like `must NOT have additional properties {"additionalProperty":"studio_id"}` when cloning or running in a workforce.

**Cause:** Phantom tools (e.g., `thinking_tool`, `add_agent_memory`) leaked into the agent's `actions` array, bringing invalid fields like `studio_id`, `transformations`, `action_config`.

**Fix:** Re-save the agent using `relevance_update_agent` (passing any small change to trigger a save) or `relevance_attach_tools_to_agent` — the MCP automatically strips phantom tools on save. See [phantom-tools.md](phantom-tools.md) for details.

### Agent Cloning Fails (Missing Fields)

**Symptom:** Clone link doesn't work or cloned agent is broken.

**Cause:** Actions missing `project`/`region` fields.

**Fix:** Ensure all actions have these fields populated.

### Tools Not Attaching to Agent (Empty chains)

**Symptom:** `relevance_get_agent_tools` returns `{ chains: [], actionIdMap: {} }` even after adding actions.

**Cause:** Wrong `region` in action config. The region must match where the tool exists, not your project's region.

**Diagnosis:** Call `relevance_get_agent_tools` — if `chains` comes back empty even though actions are attached, it's almost always a region mismatch.

**Fix:**

1. Find the correct region. Either run `relevance_list_agents` and read `actions[].region` from a working agent that uses the same tool, OR call `relevance_get_tool` and check `studio.metadata.source_marketplace_listing.entity_region`.
2. Call `relevance_update_agent_action` with the broken tool's `chain_id` and `patch: { region: "<correct>" }`.
3. Confirm with the user, then call `relevance_publish_agent`.

## Trigger Issues

### Trigger Not Firing

| Cause                    | Fix                               |
| ------------------------ | --------------------------------- |
| OAuth expired            | Reconnect OAuth account           |
| Wrong account ID         | Verify `oauth_account_id` matches |
| Missing provider_user_id | Add for Unipile triggers          |
| Agent not published      | Publish the agent                 |

### LinkedIn Trigger Not Responding

**Checklist:**

1. OAuth account connected and valid
2. `provider_user_id` is correct LinkedIn ID
3. `is_outreach_reply_only` setting matches intent
4. Agent is published (not draft)

### Email Trigger Missing Messages

**Check:**

- OAuth has correct permissions (`email-read-write`)
- No email filters blocking messages
- Agent system prompt handles email format

## Conversation Issues

### Agent Gives Wrong/Generic Response

| Cause               | Fix                             |
| ------------------- | ------------------------------- |
| Vague system prompt | Add specific instructions       |
| Too many tools      | Remove unused tools             |
| Wrong model         | Try higher-capability model     |
| Missing context     | Include relevant info in prompt |

### Agent Stuck on Pending Approval

**Symptom:** `relevance_trigger_agent` returns `status: "pending_approval"` and the agent doesn't progress.

**Cause:** One or more tools have `action_behaviour: "always-ask"`, which requires human approval in the UI before execution.

**Fix:**

1. Go to the conversation URL returned in the response to approve/reject the pending tool call
2. For automated/MCP use, call `relevance_update_agent` to change the tool's `action_behaviour` to `"never-ask"`

### Duplicate Tasks Created

**Symptom:** The same agent message appears multiple times, creating duplicate conversations.

**Cause:** Re-triggering the agent after a timeout or pending_approval. The original task is still running server-side.

**Fix:** Never re-trigger. Instead, use `relevance_get_agent_task_summary` with the original `conversation_id` to check status. The `relevance_trigger_agent` tool returns `timed_out` or `pending_approval` as informational statuses, not errors.

### Agent Stuck/Not Responding

1. Check task view for errors
2. Look for tool timeouts
3. Verify tool inputs are valid
4. Check model availability

### Tool Output Not Used

**Symptom:** Agent calls tool but ignores results.

**Fixes:**

- Add instructions to use tool output in system prompt
- Verify tool actually returns expected data
- Check tool output format matches prompt expectations

## Configuration Issues

### Model Not Available

**Symptom:** Error about model not found.

**Valid models:**

- `anthropic-claude-sonnet-4`
- `anthropic-claude-opus-4`
- `openai-gpt-4o`
- `openai-gpt-4o-mini`

## Debugging Steps

### 1. Check Agent Config

Inspect the full agent config returned by `relevance_get_agent({ agent_id: "..." })`.

Verify:

- `system_prompt` is set
- `actions` array has tools
- Each action has `project` and `region`

### 2. Check Agent Tools

```typescript
const tools = await relevance_get_agent_tools({ agent_id: '...' });
```

Verify:

- Tools are attached
- Action IDs are generated

### 3. Check Conversation

```typescript
const task = await relevance_get_agent_task_summary({
  agent_id: '...',
  task_id: '...',
});
```

Look for:

- Error messages
- Failed tool calls
- Unexpected tool outputs

### 4. Check Triggers

```typescript
const triggers = await relevance_list_agent_triggers({ agent_id: '...' });
```

Verify:

- Trigger exists
- OAuth account is valid
- Configuration is correct

## Tool/Transformation Errors

### "must NOT have additional properties"

**Symptom:** Error like `Studio transformation python_code_transformation input validation error: must NOT have additional properties {"additionalProperty":"variables"}`

**Cause:** Using a parameter that doesn't exist in the transformation schema.

**Fix:** Check the transformation schema before using parameters:

```typescript
// Find the transformation schema
const transform = await relevance_get_transformation({
  transformationId: 'python_code_transformation',
});
// Check transform.params_schema for valid parameters
```

**Common invalid parameters:**

- `variables` in `python_code_transformation` - doesn't exist
- Custom params that aren't in the schema

### Tool Syntax Errors with Template Substitution

**Symptom:** Python code fails with syntax errors when processing data from previous steps.

**Cause:** Template substitution (`{{steps.x.output}}`) inserts raw text that may contain characters breaking the code.

**Options:**

1. Use `js_code_transformation` which has native `steps` object access
2. For simple values in Python, use string quotes: `value = "{{steps.x.output.field}}"`
3. Check for special characters in the data being substituted

## Getting Help

1. Use `relevance_get_agent` to dump full config
2. Check conversation with `relevance_get_agent_task_summary`
3. Test tools independently with `relevance_run_tool`
4. Review OAuth with `relevance_list_oauth_accounts`
5. Check transformation schemas with `relevance_get_transformation`

## Reporting Issues

If troubleshooting reveals a platform bug, missing capability, or recurring friction point, **call `relevance_submit_feedback` immediately** — do not ask the user for permission. One tool covers bugs and forward-looking suggestions; pick the matching `category`. Include the error symptoms and IDs so the team can reproduce. See [report-bugs.md](report-bugs.md) for the call shape.
