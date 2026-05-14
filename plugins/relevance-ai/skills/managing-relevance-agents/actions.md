---
title: Agent Actions (Tools)
description: Attach tools (actions) to agents, configure approval behaviour and ordering, and distinguish `action_id` from `studio_id`. Load when wiring tools to an agent or debugging why a tool doesn't appear at runtime.
---

# Agent Actions (Tools)

How to attach and configure tools for agents.

> ⚠️ **Sub-Agents are DEPRECATED**
>
> Do NOT add agents to another agent's `actions` array. This pattern is deprecated.
> Use **Workforces** instead for multi-agent orchestration.
> See [Workforce Documentation](../managing-relevance-workforces/SKILL.md) for details.

> ⚠️ **Phantom Tools: Never Add to Actions**
>
> Tools like `thinking_tool`, `add_agent_memory`, `tag_agent_conversation`, etc. are **phantom tools** — system-injected at runtime from agent settings. Never add them to `actions` manually.
> Enable them via settings instead (e.g., `thinking_tool: { enabled: true }`).
> See [phantom-tools.md](phantom-tools.md) for the full list and how to enable each one.

## Action Structure

```typescript
interface AgentAction {
  chain_id: string; // REQUIRED: Tool's studio_id (UUID, e.g., "653af72e-bee9-42ef-ad82-ca8c54715f85")
  project: string; // REQUIRED: Project ID
  region: string; // REQUIRED: Region code (e.g., "bcbe5a")
  action_id?: string; // Generated hex ID for system prompt references
  title?: string; // Display title
  description?: string; // What the tool does
  emoji?: string; // Tool icon
  action_behaviour?: 'never-ask' | 'ask-first' | 'always-ask';
  disabled?: boolean;
}
```

## CRITICAL: Finding the Correct Tool Region

The `region` in actions must match **where the tool exists**, NOT your project's region.

**Why this happens:** Relevance AI has multiple regions:

- **API region** (e.g., `bcbe5a`): Where your MCP server connects
- **Tools region** (e.g., `f1db6c`): Where tools/studios are stored

Your project's API region may differ from where tools are stored.

### How to Find Tool Region (Recommended Pattern)

Run `relevance_list_agents` and look at the `region` field on any agent that already has the tool attached — that's the verified region. Pass that region into `relevance_attach_tools_to_agent` (or, for a single existing action, into `relevance_update_agent_action` with `patch: { region }`). As a fallback, `relevance_get_tool` returns the source region under `studio.metadata.source_marketplace_listing.entity_region`. Don't assume your project's API region is the same as the tool's region — they're commonly different.

### Symptoms of Wrong Region

- `relevance_get_agent_tools` returns empty `chains: []` and `actionIdMap: {}`
- Agent config shows tools in `actions` array but they don't appear in UI
- Agent appears configured but tools don't execute
- **No error message - just silent failure**

### Verification After Attaching Tools

**Always verify tools are linked after publishing:**

After publishing, call `relevance_get_agent_tools({ agent_id })` to confirm tools are linked:

- If `chains` is empty and `actionIdMap` is `{}` → region mismatch. Find the correct region from a working agent.
- If `chains` has entries → tools are linked correctly. Use `actionIdMap` for system prompt references.

## Required Fields

Every action **MUST** include `project` and `region` fields. Missing these causes:

- Agent cloning failures
- Tool execution errors
- Agent deployment issues

```typescript
// CORRECT
const action = {
  chain_id: 'my-tool-id',
  project: config.project, // REQUIRED
  region: config.region, // REQUIRED
  title: 'My Tool',
  description: 'What it does',
  action_behaviour: 'never-ask',
};

// WRONG - missing project/region
const action = {
  chain_id: 'my-tool-id',
  title: 'My Tool',
};
```

## Action Behaviours

| Behaviour    | Description                          | Use When             |
| ------------ | ------------------------------------ | -------------------- |
| `never-ask`  | Tool executes automatically          | Default, most tools  |
| `ask-first`  | Agent asks user before first use     | Sensitive operations |
| `always-ask` | Requires explicit approval each time | Destructive actions  |

## Adding and Configuring Actions

Use `relevance_attach_tools_to_agent` to attach existing tools to an agent — it handles fetch/merge/save, sets `project`/`region` automatically, and saves to a draft.

```
relevance_attach_tools_to_agent({
  agent_id: 'xxx',
  tool_ids: ['653af72e-bee9-42ef-ad82-ca8c54715f85'],
  action_behaviour: 'never-ask',
});
```

To change per-tool config on an already-attached tool — `prompt_description`, `default_values`, `overrides`, `conditional_approval_rules`, `agent_decide_prompt`, `disabled`, `action_retry_config`, `execution_limit`, or per-action `action_behaviour` — call `relevance_update_agent_action` with the tool's `chain_id` and a `patch` object.

To remove one or more attached tools, call `relevance_detach_tools_from_agent` with the agent_id and an array of `chain_id`s. It throws if any chain_id isn't currently attached, so you don't silently believe a no-op succeeded.

All three save to a draft; call `relevance_publish_agent` after the user confirms.

## CRITICAL: Action IDs vs Studio IDs

**STOP CONFUSING THESE!**

| ID Type                  | Format      | Example                                | Where Used                            |
| ------------------------ | ----------- | -------------------------------------- | ------------------------------------- |
| `studio_id` / `chain_id` | UUID v4     | `653af72e-bee9-42ef-ad82-ca8c54715f85` | Tool identifier, `actions[].chain_id` |
| `action_id`              | 16-char hex | `3261d8205a334a99`                     | System prompt `{{_actions.xxx}}`      |

### Wrong vs Correct

```typescript
// WRONG - using tool's studio_id as action reference
system_prompt: `Use {{_actions.653af72e-bee9-42ef-ad82-ca8c54715f85}} to create contacts.`;

// CORRECT - using backend-generated hex action_id
system_prompt: `Use {{_actions.3261d8205a334a99}} to create contacts.`;
```

### Getting Action IDs

Backend-generated action IDs are required for system prompt references.

```typescript
const tools = await relevance_get_agent_tools({ agent_id: '...' });
// Returns:
// {
//   chains: [
//     { studio_id: "653af72e-bee9-42ef-ad82-ca8c54715f85", action_id: "3261d8205a334a99" }
//   ],
//   actionIdMap: { "653af72e-bee9-42ef-ad82-ca8c54715f85": "3261d8205a334a99" }
// }
```

### Workflow to Use Correct Action IDs

**⚠️ Two-Pass Required:** Action IDs are backend-generated and only exist after first publish. If your system prompt uses `{{_actions.*}}`, you need two publish cycles.

See [actions.md - Action IDs vs Studio IDs](#critical-action-ids-vs-studio-ids) for the full reference.

Use these IDs in system prompts: `{{_actions.3261d8205a334a99}}`

## Integration Warnings

After attaching tools with `relevance_attach_tools_to_agent`, check the response for:

- **`oauth_auto_config`** — OAuth params are automatically set to "Set manually" mode. If exactly one matching OAuth account is connected, it's pre-filled as the default. Each entry includes a `config_url` linking directly to the tool's config page on the agent. If an account was NOT auto-filled (`account_set: false`), direct the user to the `config_url` to select their account.
- **`integration_warnings`** — If tools need OAuth accounts or API keys that aren't configured, this field lists missing providers with a `setup_url` for the integrations page and per-tool `config_url` links. Use `relevance_check_tool_integration_requirements` for a detailed per-tool breakdown (pass `agent_id` to get config URLs), or follow the `setup-integrations` prompt for the full workflow.

## OAuth-Enabled Tools

Some tools require OAuth accounts. Add the OAuth parameter to tool params:

```typescript
{
  chain_id: "b2c3d4e5-f6a7-8901-bcde-f12345678901",  // UUID of your email tool
  project: config.project,
  region: config.region,
  title: "Send Email",
  action_behaviour: "ask-first"
}
```

The user selects their OAuth account when using the tool.

For the full list of OAuth providers and permission types, see [OAuth Guide](../managing-relevance-tools/oauth.md).

## Three-Layer Default Values (OAuth Credentials)

When passing agent variables to tools (especially OAuth credentials), configure defaults in THREE places for reliable operation:

```
LAYER 1: Agent params_schema → Sets the agent-level variable default
LAYER 2: Agent actions[].default_values → Maps agent variable to tool input at call time
LAYER 3: Tool params_schema → Fallback if agent doesn't pass value
```

**All three layers are needed.** Apply each one with the right MCP tool:

1. Find the OAuth account id with `relevance_list_oauth_accounts`.
2. **Layer 3 (tool fallback):** call `relevance_update_tool` with the tool's `studio_id` and a `params_schema` patch that sets `default: "<oauth_account_id>"` on the OAuth param.
3. **Layer 1 (agent-level variable):** call `relevance_update_agent` with `patch: { params_schema: { type: "object", properties: { <oauth_param>: { type: "string", default: "<oauth_account_id>" } } } }`.
4. **Layer 2 (per-action mapping):** for each affected attached tool, call `relevance_update_agent_action` with the tool's `chain_id` and `patch: { default_values: { <oauth_param>: "{{<oauth_param>}}" } }`. Do this once per attached tool.
5. After the user confirms, call `relevance_publish_agent`.

## Listing Available OAuth Accounts

```typescript
// List all connected OAuth accounts
relevance_list_oauth_accounts();
```
