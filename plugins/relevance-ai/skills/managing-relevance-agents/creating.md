---
title: Creating Agents
description: Create agents from scratch and configure core settings — model, system prompt, temperature, autonomy, memory, thinking, and phantom-tool flags. Load when building a new agent or changing its core config.
---

# Creating Agents

Complete workflow for creating Relevance AI agents.

## Agent Structure

```typescript
interface AgentConfig {
  agent_id?: string; // Auto-generated if not provided
  name: string; // Display name
  description: string; // Brief description
  system_prompt: string; // Core instructions
  model?: string; // LLM model
  emoji?: string; // Icon
  temperature?: number; // 0-1 creativity
  actions?: AgentAction[]; // Attached tools
  params_schema?: object; // Input variables
  autonomy_limit?: number; // Max tool calls (default: 20)
  autonomy_limit_behaviour?: string; // What happens at limit
  model_options?: {
    // Model configuration
    max_output_tokens?: number; // Max response length (e.g. 16000)
  };
}
```

### `model_options` — NOT a top-level field

`max_output_tokens` and other model settings live inside `model_options`, **not** as top-level agent fields. Setting them at the top level causes a 422 error.

```typescript
// WRONG — 422 error: unknown top-level field
relevance_update_agent({
  agent_id: '...',
  patch: { max_output_tokens: 16000 },
});

// CORRECT — nested inside model_options
relevance_update_agent({
  agent_id: '...',
  patch: { model_options: { max_output_tokens: 16000 } },
});
```

## Agent Features (Enabled via Settings)

Agents have built-in capabilities enabled through settings — **not** by adding tools to `actions`. These generate "phantom tools" at runtime:

| Setting                                           | What It Enables                        |
| ------------------------------------------------- | -------------------------------------- |
| `thinking_tool: { enabled: true }`                | Internal reasoning scratchpad          |
| `memory: { enabled: true, memory_level: "user" }` | Persistent memory across conversations |
| `tags: ["support", "sales"]`                      | Conversation tagging                   |
| `is_scheduled_triggers_enabled: true`             | Agent can schedule future actions      |
| `escalations: { email: { emails: [...] } }`       | Escalate to human managers             |
| `enable_python_executor: true`                    | Agent can write and run Python         |

**See [phantom-tools.md](phantom-tools.md) for the full reference** — how to enable each one, what they do, and critical gotchas about never putting them in `actions`.

## Agent Configuration Options

### Action Behaviour (Tool Execution)

Controls whether the agent asks for user approval before running tools.

| Value          | Behavior                                                      |
| -------------- | ------------------------------------------------------------- |
| `"never-ask"`  | Agent runs tools automatically without approval               |
| `"ask-first"`  | Agent asks for approval on first use, then runs automatically |
| `"always-ask"` | Agent asks for approval before EVERY tool call                |

```typescript
{
  action_behaviour: "never-ask",  // Recommended for most agents
  actions: [
    {
      chain_id: "my-tool",
      action_behaviour: "never-ask"  // Per-tool override
    }
  ]
}
```

**⚠️ Common Issue:** If your agent says "I'll run the tool" but nothing happens, check if `action_behaviour` is set to `"always-ask"`. This blocks execution pending user approval in the UI.

### Autonomy Settings

| Field                      | Valid Values                                     | Description                    |
| -------------------------- | ------------------------------------------------ | ------------------------------ |
| `autonomy_limit`           | number (default: 20)                             | Max tool calls before stopping |
| `autonomy_limit_behaviour` | `"ask-for-approval"`, `"terminate-conversation"` | What happens at limit          |

```typescript
{
  autonomy_limit: 50,
  autonomy_limit_behaviour: "ask-for-approval"  // Recommended
}
```

**⚠️ Critical Behaviour Differences:**

| Value                      | Effect                                           |
| -------------------------- | ------------------------------------------------ |
| `"ask-for-approval"`       | Agent pauses and asks user if it should continue |
| `"terminate-conversation"` | Agent **silently stops** without any response    |

**Common Error:** Using `"stop"` - this value is invalid and causes 422 error.

**⚠️ Silent Termination Issue:** If `autonomy_limit_behaviour: "terminate-conversation"` and the agent runs a tool, it may silently end the conversation without providing results. Use `"ask-for-approval"` to ensure the agent always responds.

## Creation Workflow

### Step 1: Create Basic Agent

```typescript
relevance_create_agent({
  name: 'Research Assistant',
  description: 'Helps research topics using web search',
  system_prompt: `You are a helpful research assistant.

When the user asks about a topic:
1. Use Google Search to find relevant information
2. Analyze and synthesize the results
3. Provide a clear summary with sources`,
});
```

### Step 2: Get Agent ID

The `relevance_create_agent` response includes the new `agent_id` directly. If you've lost it, call `relevance_get_agent` (or list with `relevance_list_agents`) to retrieve it.

### Step 3: Add Tools and Configure (saves as draft)

Use `relevance_attach_tools_to_agent` to attach tools — it handles fetch/merge/save for you and saves to a draft:

```typescript
relevance_attach_tools_to_agent({
  agent_id: agent.agent_id,
  tool_ids: ['gtm-google-search'],
  action_behaviour: 'never-ask',
});
```

For other field changes (system_prompt, model, temperature, memory, etc.), use `relevance_update_agent` — partial-merge into the draft:

```typescript
relevance_update_agent({
  agent_id: agent.agent_id,
  patch: {
    temperature: 0.3,
    memory: { enabled: true, memory_level: 'user' },
  },
});
```

### Step 4: Verify OAuth Integrations

If any attached tools require OAuth (check `params_schema` for `metadata.content_type === "oauth_account"`), verify the connection:

1. Call `relevance_list_oauth_accounts` to check connected accounts
2. If a required provider is missing, call `relevance_get_project_info` to get the integrations page URL and direct the user to connect it

### Step 5: Test the Draft

```typescript
relevance_trigger_agent({
  agent_id: agent.agent_id,
  message: 'Research the latest AI developments',
});
// Response includes triggered_version: "draft" so you know which version ran.
```

### Step 6: Ask the User, Then Publish

After the user has reviewed the draft and confirmed they want it live, publish it.
`relevance_publish_agent` always shows an approval card (even with auto-approve enabled).

```typescript
relevance_publish_agent({ agent_id: agent.agent_id });
```

## Best Practice: Examine Working Agents First

Before creating a new agent, look at how similar working agents are configured. Run `relevance_list_agents`, then `relevance_get_agent` on one that already uses the tools you want — read the `region` field on each entry of its `actions` array. This is CRITICAL: tools must be attached with the **region the tool lives in**, which is often different from your project's region. The same fetch also surfaces working `action_behaviour` patterns and system-prompt structure to copy.

## Agent Lifecycle

```text
1. Create basic agent (relevance_create_agent — saves to DRAFT only)
        |
2. Get agent to see ID (relevance_get_agent)
        |
3. Add tools / configure (relevance_attach_tools_to_agent / relevance_update_agent — saves draft)
        |
4. Verify OAuth (relevance_list_oauth_accounts + relevance_get_project_info)
        |
5. Test the draft (relevance_trigger_agent — defaults to draft)
        |
6. Ask the user → Publish (relevance_publish_agent — always confirms with the user)
        |
7. Add triggers if needed (relevance_create_trigger)
```

## Model Options

| Model                       | Use Case                  |
| --------------------------- | ------------------------- |
| `anthropic-claude-sonnet-4` | Best quality/cost balance |
| `anthropic-claude-opus-4`   | Highest quality           |
| `openai-gpt-4o`             | Fast, good quality        |
| `openai-gpt-4o-mini`        | Fast, cheaper             |

## Common Patterns

- **Search and Summarize:** attach a search tool and a scraper. System prompt: search → read pages → synthesize.
- **Data Enrichment:** attach search + contact-finder + a profile scraper (e.g. LinkedIn). System prompt: search for the entity → extract details → find contact info.
- **Event-Driven:** attach the relevant data-fetching tools, leave triggers to `relevance_create_trigger`. System prompt explains how to react when the trigger fires.

## Testing

After creating an agent, test it with `relevance_trigger_agent` to verify it works as expected. For production agents, see the `relevance-evals` skill for setting up formal evaluation suites.

---

## Quality Checklist

- [ ] Clear, specific name and description
- [ ] System prompt defines role and tool usage
- [ ] All necessary tools attached
- [ ] All actions have project and region fields
- [ ] Correct action_behaviour for each tool
- [ ] Model appropriate for task complexity
- [ ] Each tool individually validated with `relevance_run_tool`
- [ ] Agent tested with `relevance_trigger_agent`
- [ ] Triggers configured (if needed)
