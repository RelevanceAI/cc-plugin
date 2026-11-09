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
  emoji?: string; // Icon: a unicode emoji or an image link. A temporary file link is saved as a permanent copy or refused
  temperature?: number; // 0-1 creativity
  actions?: AgentAction[]; // Attached tools
  params_schema?: object; // Agent variables: what each one is — see variables.md
  params?: object; // Agent variables: each one's value
  knowledge?: AgentKnowledgeItem[]; // Attached bases — see managing-relevance-knowledge
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

**See [phantom-tools.md](phantom-tools.md) for the full reference** — how to enable each one, what they do, and critical gotchas about never putting them in `actions`.

No setting lets an agent run Python. Build a tool with a [`python_code_transformation`](../managing-relevance-tools/transformations.md#python_code_transformation) step and attach it.

> **Evaluator agents** (`runtime_engine: "evaluator"` on `relevance_create_agent`) exist only to grade conversations as a `custom_agent` eval check; they refuse triggers, sub-agent calls and workforce nodes. Load `relevance-evals/evaluator-agents` before creating one.

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

**First, check what each tool is for.** A tool that fetches the records a scheduled agent should work through one at a time (new leads, open tickets, new rows) is not attached: it becomes the agent's [tool trigger](triggers.md#tool-triggers-one-task-per-item), so each record gets its own task. A list the user already has (a CSV, a knowledge table) needs no fetching tool at all: write the prompt for one row and start it with the bulk-run tool, or Run in bulk in the app. See [A Tool That Lists the Work Starts the Agent](SKILL.md#a-tool-that-lists-the-work-starts-the-agent) and [Work Over Many Items](many-items.md).

**Before attaching tools, pick the right one — prefer native over third-party.** Follow this order:

1. **Reuse first — unless the user explicitly asked you to build from scratch.** `relevance_list_tools({ query: "..." })` — if the project already has a tool that does the job, attach it. Skip this step only when the user has asked for a fresh build.
2. **Native transformation second.** `relevance_list_transformations({ search: "..." })` returns results partitioned into `native` and `pipedream` arrays. Wrap a `native` entry via `relevance_create_tool_from_transformation` and attach it.
3. **Third-party only as fallback.** A `pipedream` entry (IDs follow the `{provider}_-_{action}` pattern) is a third-party integration — use it only when no native equivalent exists for the requested provider/action. Refer to it as a "third-party" integration in user-facing prose. See [`managing-relevance-tools/SKILL.md`](../managing-relevance-tools/SKILL.md#prefer-native-transformations) for the full hierarchy.

Use `relevance_attach_tools_to_agent` to attach tools — it handles fetch/merge/save for you and saves to a draft:

```typescript
relevance_attach_tools_to_agent({
  agent_id: agent.agent_id,
  tool_ids: ['gtm-google-search'],
  action_behaviour: 'never-ask',
});
```

If the agent already has evals, the new tool is not covered by them and will run for real on the next eval — extend the test set's simulation config before publishing, using the `action_id_map` this call returns. See the `relevance-evals` skill.

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
// Only reads and replies: a real run, so the user sees real results in their tasks.
relevance_trigger_agent({
  agent_id: agent.agent_id,
  message: 'Research the latest AI developments',
  simulate: false,
});
// Response includes triggered_version: "draft" so you know which version ran.
// Has a tool that sends or changes something (if you can't tell, count it as one):
// simulate first with the reads live, then run it for real once the user agrees to
// that run and says where it may write. See running.md#trying-an-agent-whose-tools-write.
```

### Step 6: Ask the User, Then Publish

After the user has reviewed the draft and confirmed they want it live, publish it.
`relevance_publish_agent` is irreversible, so confirm in chat before calling it.

Always pass a concise `version_description` (and `version_name`) — a clear, one-line summary of what changed and why, easy to understand. This is the version-history trail; reuse the summary you gave the user and never leave it blank.

```typescript
relevance_publish_agent({
  agent_id: agent.agent_id,
  version_name: 'Initial build',
  version_description:
    'Support agent that answers billing questions from the help-centre knowledge base.',
});
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

Never hardcode a versioned model id — models get retired, and a retired id silently degrades the agent. Call `relevance_list_llm_models` and pick one that is not flagged deprecated or retired.

With no specific reason to prefer a particular model, use `relevance-cost-optimized` (or `relevance-performance-optimized` for the hardest tasks). These always resolve to a current model, so they never go stale. A new agent runs on `relevance-cost-optimized` until its `model` is set, so leaving it unset is choosing it.

When the agent does have a shape — a long tool loop, a latency-sensitive chat agent, image or PDF input, high-volume extraction — load the `relevance-llm-models/SKILL` guide instead of defaulting. It maps the shape to a current model and covers the reasoning-effort and parameter settings that have to be set alongside it.

## Common Patterns

- **Search and Summarize:** attach a search tool and a scraper. System prompt: search → read pages → synthesize.
- **Data Enrichment:** attach search + contact-finder + a profile scraper (e.g. LinkedIn). System prompt: search for the one entity its task names → extract details → find contact info. To enrich a whole list, start it once per row with the bulk-run tool, or Run in bulk in the app; fields with nothing to judge can be a tool column on the table instead ([many-items.md](many-items.md)).
- **Event-Driven:** attach the relevant data-fetching tools, leave triggers to `relevance_create_trigger`. System prompt explains how to react when the trigger fires.

## Testing

After creating an agent that only reads and replies, test it with `relevance_trigger_agent`. Test one with a tool that writes in an eval with a stand-in for that tool, as the tools skill's [Testing Workflow](../managing-relevance-tools/running.md#testing-workflow) says. For production agents, see the `relevance-evals` skill for setting up formal evaluation suites.

---

## Quality Checklist

- [ ] Clear, specific name and description
- [ ] System prompt defines role and tool usage
- [ ] All necessary tools attached
- [ ] All actions have project and region fields
- [ ] Correct action_behaviour for each tool
- [ ] Model set for the agent's shape (a tool loop is a shape), not left on the default by accident
- [ ] Each tool that only reads validated with `relevance_trigger_tool` (then `relevance_poll_tool_result`); each tool that writes checked in an eval with a stand-in for it
- [ ] Agent tested with `relevance_trigger_agent`
- [ ] Triggers configured (if needed)
