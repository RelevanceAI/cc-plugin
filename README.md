# Relevance AI — Claude Code Plugin

Build and manage AI agents, tools, and workforces on [Relevance AI](https://relevanceai.com) directly from Claude Code.

## Install

```bash
# Add the marketplace (one-time)
claude plugin marketplace add RelevanceAI/cc-plugin

# Install the plugin
claude plugin install relevance-ai@relevance-ai-plugins
```

Or from inside Claude Code:
```
/plugin marketplace add RelevanceAI/cc-plugin
/plugin install relevance-ai@relevance-ai-plugins
```

## What's included

- **MCP Server** — Remote MCP connection to Relevance AI API (tools, agents, workforces, knowledge, analytics)
- **Skills** — Detailed guides for building agents, tools, workforces, and more

## Skills

| Skill | Description |
|-------|-------------|
| `managing-relevance-agents` | Creating agents, attaching tools, system prompts, memory, triggers |
| `managing-relevance-tools` | Building tools, transformations, state_mapping, icons, OAuth |
| `managing-relevance-workforces` | Multi-agent systems, nodes/edges, debugging |
| `managing-relevance-knowledge` | Knowledge table CRUD operations |
| `managing-relevance-folders` | Organizing agents and tools into folders |
| `relevance-analytics` | Agent usage analytics and metrics |
| `relevance-evals` | Agent & workforce evaluations, checks, and production monitoring |
| `relevance-diagnostics` | Diagnosing project/agent/workforce issues and recommending fixes |
| `relevance-task-ops` | Reading the Task Ops monitor page (errored / escalated / pending tasks) |
| `relevance-slide-builder` | Slideshows, templates, versions, and brand kits |

## Setup

After installing, you need to authenticate with Relevance AI:

1. Run `/mcp` in Claude Code
2. Select the `relevance-ai` server
3. Click **Authenticate** — this opens your browser to log in
4. Once authenticated, MCP tools are available immediately

This is a one-time setup.

## Requirements

- Claude Code v1.0.33+
- A [Relevance AI](https://relevanceai.com) account
