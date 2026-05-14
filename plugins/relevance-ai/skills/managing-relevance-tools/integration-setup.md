---
title: Integration Setup
description: Entry point for detecting and resolving missing OAuth accounts and API keys required by tools. Use when attaching tools to agents, diagnosing auth failures, or before triggering agents.
---

# Integration Setup

Tools authenticate with external services in one of two ways. This doc is the router — use the MCP tools below to detect what's missing, then jump to the right deep-dive.

## OAuth vs API keys

|                       | **OAuth**                                                                        | **API keys**                                                                                                  |
| --------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| Scope                 | Per tool parameter — the user picks an account for each tool                     | Per project/org — one key is reused by every tool that needs that provider                                    |
| Tool schema           | Declared as a `params_schema` property with `content_type: "oauth_account"`      | Declared by the underlying transformation, not by the tool itself — so it does not appear as a tool parameter |
| Where users configure | Integrations page → OAuth provider drawer (login flow)                           | Integrations page → API key provider drawer (paste secret)                                                    |
| MCP can auto-fill     | Yes — `relevance_attach_tools_to_agent` sets the account ID as the param default | No — the key is looked up at runtime from the project/org                                                     |

## When to Check

- **After `relevance_attach_tools_to_agent`** — its `integration_warnings` lists tools needing OAuth or API keys.
- **When tools fail** — errors like `"You need to add your chains_xxx_api_key"` or `"Invalid OAuth account"` indicate missing integrations.
- **Before triggering an agent** — verify integrations are configured so the run doesn't fail mid-conversation.

## Detecting missing integrations

```typescript
relevance_check_tool_integration_requirements({
  tool_ids: ['tool-id-1', 'tool-id-2'],
  agent_id: 'agent-xxx', // optional — adds per-tool config_url
});
```

Response shape (trimmed):

```json
{
  "tools": [
    {
      "studio_id": "tool-id-1",
      "title": "Search HubSpot Contacts",
      "oauth_requirements": [
        {
          "provider": "hubspot",
          "status": "missing",
          "setup_url": "https://app.relevanceai.com/integrations/us-east/proj-xxx?provider=hubspot&type=oauth"
        }
      ],
      "api_key_requirements": [
        {
          "provider": "firecrawl",
          "provider_label": "Firecrawl API Key",
          "status": "missing",
          "has_platform_key": false,
          "setup_url": "https://app.relevanceai.com/integrations/us-east/proj-xxx?provider=firecrawl&type=api_key"
        }
      ]
    }
  ],
  "action_required": true,
  "missing_oauth": [
    /* deduped by provider across all tools */
  ],
  "missing_api_keys": [
    /* deduped by provider across all tools */
  ],
  "setup_url": "https://app.relevanceai.com/integrations/..."
}
```

**Status values:**

- `configured` — user or org already has a key/account; tool will work.
- `platform_key_available` (API keys only) — Relevance has a shared key fallback; the tool will work, user can optionally add their own.
- `missing` — user action required. Send them the entry's `setup_url`.

## Directing the user

Always prefer the per-entry `setup_url` over the top-level one — it opens the provider's drawer directly. For API keys, show the `provider_label` (e.g. "Firecrawl API Key"), never the raw `provider` identifier.

> Your tool _"Search HubSpot Contacts"_ needs a HubSpot OAuth connection. Please connect here: {setup_url}

After the user reports they've added it, re-run `relevance_check_tool_integration_requirements` (or `relevance_check_api_key_availability` for keys only) to verify.

## What next

- **API-key specifics** — multi-key providers (Twilio, AWS Bedrock, Azure OpenAI, GCP Vertex, Marketo, ZoomInfo), LLM platform-key semantics, common error strings, and `provider_label` guidance → see [api-keys.md](api-keys.md).
- **OAuth param authoring** — how to declare `content_type: "oauth_account"`, and the `is_fixed_param` + `default` pattern for agent-called tools → see [oauth.md](oauth.md).

## Quick Reference

| Tool                                            | Purpose                                                                      |
| ----------------------------------------------- | ---------------------------------------------------------------------------- |
| `relevance_check_tool_integration_requirements` | Check what integrations tools need, cross-referenced against configured ones |
| `relevance_check_api_key_availability`          | Check which API keys are configured at project/org level                     |
| `relevance_list_oauth_accounts`                 | List connected OAuth accounts                                                |
| `relevance_list_integration_providers`          | List available integration providers                                         |
