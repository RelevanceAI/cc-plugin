---
title: API Key Integrations
description: Detect, interpret, and resolve missing API-key integrations for Relevance tools via MCP. Use when agents' tools fail with "API key missing" errors, when attaching tools to an agent, or before triggering agents that depend on API-key providers.
---

# API Key Integrations

Deep reference for API-key integrations. For the detection → setup → verify workflow and the OAuth-vs-API-key overview, see [integration-setup.md](integration-setup.md) first.

## When to Check

- **After `relevance_attach_tools_to_agent`** — its `integration_warnings.tools_needing_api_keys` lists any attached tool whose transformations require a key the project doesn't have.
- **On "API key missing" errors** — runtime errors like `"You need to add your chains_xxx_api_key"` or `"Missing API key for provider: <name>"` mean the backend tried to read an unconfigured key.
- **Before triggering an agent** — verify keys are in place so the run doesn't fail mid-conversation.

Because the requirement lives on the underlying transformation, you cannot see it from the tool's `params_schema` alone — always use `relevance_check_tool_integration_requirements` (or `relevance_check_api_key_availability` for a project-wide read).

> **Always show `provider_label` to the user, not `provider`.** The `provider` field is the internal identifier (e.g., `twilio_account_sid`, `aws-bedrock-iam-key-id`) — pass it through to other MCP calls and deep-link URLs, but never paste it into a user-facing message. Use `provider_label` (e.g., "Twilio Account SID") for anything the user will read. The label falls back to the raw identifier only when the provider is not in the catalog.

## Project-level lookup

To inspect what's configured across the project and org without going via a tool:

```typescript
relevance_check_api_key_availability({
  providers: ['firecrawl', 'serper'], // optional filter
});
```

Each entry in `api_keys[]` carries `key` (the internal identifier), `label` (the friendly display name — use this when talking to the user), `project_context.api_key_provided`, `organization_context.api_key_provided`, and a per-provider `setup_url`.

## Interpreting `status`

| Status                   | Meaning                                                               | What to tell the user                                                                                                  |
| ------------------------ | --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `configured`             | User (or their org) has saved a key for this provider.                | Nothing — the tool will work.                                                                                          |
| `platform_key_available` | User has no key, but Relevance has a shared platform key as fallback. | Optional: "You can add your own key for higher rate limits and separate billing, but this tool will work without one." |
| `missing`                | No user key **and** no platform fallback.                             | Required: direct the user to the `setup_url` for that provider.                                                        |

## Messaging examples

Each missing-key entry includes a `setup_url` that deep-links to the integrations page **with that provider's drawer already open** (via `?provider=<value>&type=api_key`). Prefer it over the generic `setup_url` at the top level.

When exactly one key is missing (using `provider_label` from the response):

> Your tool _"Firecrawl Scraper"_ needs a **Firecrawl API Key** before it can run. Add yours here: https://app.relevanceai.com/integrations/us-east/proj-xxx?provider=firecrawl&type=api_key

When `has_platform_key: true`, frame it as optional:

> _"Web Search"_ will use Relevance's shared **Serper API Key** by default. If you want your own (for separate rate limits or billing), add it here: https://app.relevanceai.com/integrations/us-east/proj-xxx?provider=serper&type=api_key

After the user reports they've added the key, verify:

```typescript
relevance_check_api_key_availability({ providers: ['firecrawl'] });
// expect project_context.api_key_provided: true
```

## Deep-link URL pattern

The MCP tools always emit the `setup_url` for you. If you ever need to construct one manually, the pattern is:

```
{baseUrl}/integrations/{region}/{projectId}?provider={providerValue}&type=api_key
```

Opening this URL lands the user on the integrations page with the right provider drawer already expanded.

## Multi-Key Providers

Some providers are exposed as several independent key entries that must all be configured for the tool to work. The MCP returns one entry **per key** — treat each independently and send the user once per missing key. Each entry's `provider_label` will be human-readable (e.g., "Twilio Account SID", "Twilio Auth Token") — use those in your message to the user rather than the raw `provider` identifiers listed below.

| Provider                       | Keys that appear together                                                                                           |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------- |
| Twilio                         | `twilio_account_sid`, `twilio_auth_token`                                                                           |
| Postgres                       | `postgres-connection-string` (SSL keys are optional)                                                                |
| AWS Bedrock                    | `aws-bedrock-iam-key-id`, `aws-bedrock-iam-secret-access-key`, `aws-bedrock-iam-region`, `aws-bedrock-iam-role-arn` |
| Azure OpenAI                   | `azure-openai-key`, `azure-openai-url`, `azure-openai-model`                                                        |
| Google Cloud (Vertex / Gemini) | `gcp_project`, `gcp_location`, `gcp_private_key`, `gcp_client_email`                                                |
| Marketo                        | `marketo-client-id`, `marketo-client-secret`, `marketo-instance-url`                                                |
| ZoomInfo                       | `zoom_info_email`, `zoom_info_clientid`, `zoom_info_private_key`                                                    |

When you see multiple missing keys for the same provider family, ask the user to configure them as a set — the tool will still fail if only one is present.

## LLM Provider Keys

Keys for LLM vendors (`openai`, `anthropic`, `google`, `cohere`, `groq`, `xai`, `mistral`, `openrouter`, `fireworksai`, plus the BYO variants above) almost always return `has_platform_key: true`, meaning the tool will run fine without any user action. Avoid noisy prompts about adding an LLM key — only bring it up if:

- The user explicitly asked to bring their own key (e.g. for separate billing or higher rate limits), or
- `has_platform_key` is `false` (rare — usually an enterprise/self-hosted setup), in which case the user must add one.

## Common Errors

| Error string                                        | Cause                                                        | Fix                                                                                                              |
| --------------------------------------------------- | ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| `"You need to add your chains_xxx_api_key API key"` | Tool ran but the backing transformation couldn't find a key. | Run `relevance_check_tool_integration_requirements` on the tool to find the provider, then send the `setup_url`. |
| `"Missing API key for provider: <name>"`            | Same as above, surfaced from the transformation layer.       | Same fix.                                                                                                        |
| Tool silently errors with no useful message         | Most often a missing integration.                            | Always run `relevance_check_tool_integration_requirements` first when debugging tool failures.                   |

## Quick Reference

| Tool                                            | Purpose                                                                                                     |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `relevance_check_tool_integration_requirements` | Check a set of tools for missing OAuth accounts **and** API keys, with per-provider `setup_url` deep links. |
| `relevance_check_api_key_availability`          | List API keys across the project/org, optionally filtered, each with a deep-link `setup_url`.               |

See also [integration-setup.md](integration-setup.md) for the combined OAuth + API-key overview, and [oauth.md](oauth.md) for the OAuth-side authoring/connection flow.
