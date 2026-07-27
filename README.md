# Relevance AI — Claude Code Plugin

Build and manage AI agents, tools, and workforces on [Relevance AI](https://relevanceai.com) directly from Claude Code.

## Install

```bash
# Add the marketplace (one-time)
claude plugin marketplace add RelevanceAI/cc-plugin

# Install the plugin
claude plugin install relevance-ai@cc-plugin
```

Or from inside Claude Code:
```
/plugin marketplace add RelevanceAI/cc-plugin
/plugin install relevance-ai@cc-plugin
```

The marketplace must be added first — the `@cc-plugin` suffix only resolves once it's registered.

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

No setup needed — authentication happens on first use. Just ask Claude to do something with
Relevance AI (e.g. "list my agents"). The first tool call opens your browser to log in, and
from then on you're authenticated. This is a one-time flow.

If the browser doesn't open, run `/mcp`, select the `relevance-ai` server, and authenticate
from there. Restarting Claude Code also helps.

## Requirements

- Claude Code v2.0+
- A [Relevance AI](https://relevanceai.com) account
