# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with the Relevance AI plugin.

## Project Overview

This is a Claude Code plugin that provides integration with the Relevance AI platform via a remote MCP server. It exposes tools for managing tools (studios), agents, triggers, workforces (multi-agent systems), marketplace listings, conversations, and OAuth accounts via the Relevance AI API.

## MCP Authentication

**NEVER run `claude mcp auth` via Bash.** This command may not exist in all Claude Code versions and will error.

Authentication happens automatically: when you first call any `relevance_*` MCP tool, the server returns a 401 and Claude Code opens the user's browser for OAuth login. Just call a tool (e.g., `relevance_list_agents`) and the auth flow triggers itself.

If the user says auth isn't working, tell them to:

1. Run `/mcp` in Claude Code to check server status
2. Restart Claude Code and try again
3. Verify they're on Claude Code v1.0.33+

## Claude Code Workflow

**This section defines how Claude Code should behave when using this plugin.** Non-technical users work here, so Claude Code handles operations automatically.

### Proactive Testing: Define Test Rubric Before Building

When creating a new **agent, tool, or workforce**, proactively create a testing rubric BEFORE building:

1. **Design Phase** - Create a testing rubric with 3-5 qualitative checks
2. **Present to User** - Ask: "Before I build this, here's my testing plan. Does this cover what you need?"
3. **Get Approval** - Wait for user confirmation or adjustments
4. **Build** - Create the agent/tool/workforce
5. **Execute Tests** - Run through the testing rubric
6. **Report Results** - Show pass/fail for each check

Testing rubric templates and examples are in each skill: `managing-relevance-agents/creating.md`, `managing-relevance-tools/creating.md`, `managing-relevance-workforces/SKILL.md`.

---

## API Documentation

**If the MCP tools are not sufficient for your needs, refer to the full Relevance AI API documentation:**

https://api-f1db6c.stack.tryrelevance.com/latest/documentation

## Critical: Always Check Skills Before Working

**Before working on agents, tools, workforces, or knowledge tables, ALWAYS read the relevant skill files first.** This CLAUDE.md is a slim summary — the skills contain the detailed patterns, code examples, and gotchas you need to avoid errors.

- Building/editing an agent? → Read `managing-relevance-agents/` files first
- Building/editing a tool? → Read `managing-relevance-tools/` files first
- Building a workforce? → Read `managing-relevance-workforces/` files first
- Working with knowledge tables? → Read `managing-relevance-knowledge/` files first

## Critical API Patterns

Most common gotchas at a glance. Full details and examples are in the skill files above.

### Publish Behavior

| Operation                    | Auto-publishes? | Notes                                                                         |
| ---------------------------- | --------------- | ----------------------------------------------------------------------------- |
| `relevance_upsert_tool`      | Yes             | `relevance_publish_tool` returns friendly "already published" if called after |
| `relevance_save_agent_draft` | Yes (default)   | Pass `autoPublish: false` to save draft only                                  |
| `relevance_create_workforce` | Configurable    | Use `shouldPublish: true` (default)                                           |

### No Partial Updates for Agents

Tools auto-merge on update. **Agents do NOT** — always fetch first, merge, then save:

```typescript
const { agent } = await relevance_get_agent({ agentId: 'x' });
agent.name = 'new';
await relevance_save_agent_draft({ agentId: 'x', agentConfig: agent });
```

### `state_mapping` is REQUIRED for Tools

Every tool MUST have a `state_mapping` field. Without it, `{{search_query}}` won't resolve. Do NOT use curly braces in state_mapping values — use `"params.name"` not `"{{params.name}}"`. See `managing-relevance-tools/creating.md` for examples.

### Agent Actions Require project/region

Every action must include `project` and `region` fields. The `region` must match **where the tool exists**, not your API region. See `managing-relevance-agents/actions.md` for finding correct regions.

### Workforces, Not Sub-Agents

Adding sub-agents to an agent's `actions` array is **DEPRECATED**. Use workforces instead. See `managing-relevance-workforces/`.

### Phantom Tools: Never Persist in Actions

Phantom tools are system-injected at runtime from agent settings. Never add them to `actions` manually. Enable via settings (e.g., `thinking_tool: { enabled: true }`). The MCP's `saveDraft` strips them automatically. See `managing-relevance-agents/phantom-tools.md`.

### Attaching Tools to Agents

Use `relevance_attach_tools_to_agent` — it handles fetch, merge, save, publish, action ID retrieval, and system prompt injection in one call. Test each tool with `relevance_trigger_tool` first — tools that return empty `{}` need their output config fixed.

### Reserved Variable Prefixes

| Prefix              | Purpose                         |
| ------------------- | ------------------------------- |
| `secrets.*`         | Project-level encrypted secrets |
| `snippets.*`        | Project content templates       |
| `_knowledge.*`      | Knowledge set content           |
| `_workforce_node.*` | Workforce system                |
| `_mcp.*`            | MCP integration                 |
| `_actions.*`        | Agent action references         |

### Template Resolution Priority

Resolution order: `secrets.*` → `snippets.*` → `_knowledge.*` → reserved prefixes → state object (`params.*`, `steps.*`, `node_context.*`)

## Environment Variables & Project Configuration

| Variable            | Required | Description                            |
| ------------------- | -------- | -------------------------------------- |
| `RELEVANCE_PROJECT` | Yes      | Project ID                             |
| `RELEVANCE_API_KEY` | Yes      | API key (starts with `sk-`)            |
| `RELEVANCE_REGION`  | Yes      | Region code (e.g., `f1db6c`, `bcbe5a`) |

For backward compatibility, `US_RELEVANCE_*` variants are also supported.

## Handling Large MCP Tool Outputs

Tools like `relevance_get_task_view` and `relevance_trigger_agent_sync` can return 100KB-500KB+ JSON responses.

```bash
# CORRECT - limit input size FIRST
timeout 3s head -c 50000 /path/to/file.json | jq ...

# CORRECT - use timeout prefix
timeout 5s jq '.results[0]' /path/to/file.json
```

Don't grep large JSON files. Use `head -c` before piping. Set timeouts on bash commands.

---

## Skills

Detailed guides for Relevance AI operations are in `skills/`:

| Skill                            | What it covers                                                                             |
| -------------------------------- | ------------------------------------------------------------------------------------------ |
| `managing-relevance-agents/`     | Creating agents, attaching tools, system prompts, memory, phantom tools, triggers, testing |
| `managing-relevance-tools/`      | Building tools, transformations, search order, state_mapping, output fixes, secrets, OAuth |
| `managing-relevance-workforces/` | Multi-agent systems, nodes/edges, threading, triggering, debugging                         |
| `managing-relevance-knowledge/`  | Knowledge table CRUD operations                                                            |
| `relevance-analytics/`           | Agent usage analytics and metrics                                                          |
| `relevance-evals/`               | Agent evaluations, test cases, automated testing                                           |
