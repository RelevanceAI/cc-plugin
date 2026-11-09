---
title: OAuth Account Configuration
description: Wire OAuth account selection into tools — `params_schema` setup, passing the account into transformation steps, provider reference, and agent-tool quirks. Load when building a tool that hits a user-authenticated API (HubSpot, Gmail, etc.).
---

# OAuth Account Configuration

How to add OAuth account selection to tools for third-party integrations.

> **Directing a user to connect an account?** Always use the per-provider `setup_url` from `tools_needing_oauth[].providers[].setup_url` or `missing_oauth[].setup_url` (same pattern as API keys). It deep-links to the integrations page with the correct provider drawer already open. Never hand-roll a generic integrations URL — the user shouldn't have to scroll a grid to find the provider they need. See [integration-setup.md](integration-setup.md) for the end-to-end detection → connect → verify workflow.

## Overview

When a tool needs to access a third-party API that requires OAuth authentication:

1. Add an OAuth account parameter to the tool
2. Pass the account ID to transformation steps that need it

**Key rules:**

- Never hardcode `oauth_account_id` in tool step params
- Always make it a top-level `params_schema` input
- Reference in steps as `{{params.<name>}}` (e.g. `{{params.oauth_account_id}}`) — bare `{{oauth_account_id}}` does not resolve
- Set to "Set Manually" by default (never auto-assign)

## ❌ Anti-pattern: plain-string / hardcoded account

**The most common OAuth mistake: declaring the account as a plain text input (or pasting a raw account id into a step).** A `type: "string"` property _without_ `metadata.content_type: "oauth_account"` renders a free-text box, not the account selector. The tool then receives an un-credentialed string, and every subsequent step that calls the API fails — even though the input "looks" filled in.

```typescript
// ❌ WRONG — plain string input + hardcoded id. Renders a text box; steps fail.
params_schema: {
  properties: {
    hubspot_account: {
      type: "string",
      title: "HubSpot Account ID",
      default: "b9384e6a-f271-4f5a-929f-cefe2ebf7f0e" // raw id pasted in — does NOT carry credentials
    }
  }
}

// ✅ RIGHT — OAuth account input. Renders the account selector; steps get real credentials.
params_schema: {
  properties: {
    hubspot_account: {
      type: "string",
      title: "HubSpot Account",
      description: "Select your HubSpot account",
      metadata: {
        content_type: "oauth_account",
        oauth_permissions: [{ provider: "hubspot", types: ["hubspot-connect-app"] }]
      }
    }
  },
  required: ["hubspot_account"]
}
```

If you find an existing tool with this mistake, replace the plain-string input with the `oauth_account` form above and re-point the step's `oauth_account_id: "{{params.<name>}}"`. For an agent that should always use one account, set it on the agent's tool instead — see [OAuth for Agent-Called Tools](#critical-oauth-for-agent-called-tools).

## Define OAuth Parameter

Add a parameter with `content_type: oauth_account` in the metadata:

```typescript
params_schema: {
  type: "object",
  properties: {
    ahrefs_account: {
      type: "string",
      title: "Ahrefs Account",
      description: "Select your Ahrefs account for backlink data",
      metadata: {
        content_type: "oauth_account",
        oauth_permissions: [
          {
            provider: "ahrefs",
            types: ["pipedream-ahrefs-read-write"]
          }
        ]
      }
    }
  },
  required: ["ahrefs_account"]
}
```

### Key Fields

| Field                                   | Description                                             |
| --------------------------------------- | ------------------------------------------------------- |
| `metadata.content_type`                 | Must be `oauth_account` to show the account selector    |
| `metadata.oauth_permissions`            | Array of required permissions                           |
| `metadata.oauth_permissions[].provider` | OAuth provider name (e.g., `ahrefs`, `google`, `slack`) |
| `metadata.oauth_permissions[].types`    | Array of permission type strings required               |

## Pass OAuth to Transformation Steps

Reference the OAuth parameter using `oauth_account_id`:

```typescript
{
  name: "get_backlinks",
  transformation: "ahrefs_-_ahrefs-get-backlinks-one-per-domain",
  params: {
    oauth_account_id: "{{params.ahrefs_account}}",  // Reference the param
    target: "{{params.target_url}}",
    limit: 100
  }
}
```

## Common OAuth Providers

| Provider              | Permission Type                     | Description                        |
| --------------------- | ----------------------------------- | ---------------------------------- |
| `hubspot`             | `hubspot-connect-app`               | HubSpot CRM API                    |
| `ahrefs`              | `pipedream-ahrefs-read-write`       | Ahrefs API access                  |
| `google`              | `email-read-write`                  | Gmail read/write                   |
| `google`              | `calendar-read-write`               | Google Calendar                    |
| `google`              | `google-docs-read-write`            | Google Docs                        |
| `google_sheets`       | `google-sheets-read-write`          | Google Sheets                      |
| `google_drive`        | `pipedream-google-drive-read-write` | Google Drive                       |
| `google_drive_native` | `google-drive-read-write`           | Google Drive, Docs, Sheets, Slides |
| `slack`               | `slack-channel-post`                | Post to Slack channels             |
| `slack`               | `slack-notifications`               | Slack notifications                |
| `github`              | `pipedream-github-read-write`       | GitHub API access                  |
| `notion`              | `pipedream-notion-read-write`       | Notion API access                  |
| `twitter`             | `twitter-read-write`                | Twitter/X API access               |
| `linkedin`            | `unipile-linkedin`                  | LinkedIn via Unipile               |

## Complete Example: Ahrefs Backlink Tool

```typescript
relevance_create_tool({
  title: 'Backlink Analyzer',
  description: 'Analyze backlinks using Ahrefs',
  params_schema: {
    type: 'object',
    properties: {
      // OAuth account selector - shown as dropdown in UI
      ahrefs_account: {
        type: 'string',
        title: 'Ahrefs Account',
        description: 'Select your Ahrefs account',
        order: 0,
        metadata: {
          content_type: 'oauth_account',
          oauth_permissions: [
            {
              provider: 'ahrefs',
              types: ['pipedream-ahrefs-read-write'],
            },
          ],
        },
      },
      target_url: {
        type: 'string',
        title: 'Target URL',
        description: 'URL to analyze',
        order: 1,
      },
    },
    required: ['ahrefs_account', 'target_url'],
  },
  transformations: {
    steps: [
      {
        name: 'get_backlinks',
        transformation: 'ahrefs_-_ahrefs-get-backlinks-one-per-domain',
        params: {
          oauth_account_id: '{{params.ahrefs_account}}',
          target: '{{params.target_url}}',
          limit: 100,
          mode: 'domain',
          select: ['url_from', 'domain_from', 'domain_rating', 'anchor'],
        },
        output: { backlinks: '{{backlinks}}' },
      },
    ],
  },
});
```

## Listing Available OAuth Accounts

```typescript
// List all connected OAuth accounts, or one provider's
relevance_list_oauth_accounts({ provider: 'ahrefs' });

// Returns:
{
  results: [
    {
      account_id: '<your-oauth-account-id>',
      provider: 'ahrefs',
      label: 'Ahrefs',
      connected_by_you: true,
      tokens: [{ permission_types: ['pipedream-ahrefs-read-write'] }],
    },
  ],
  setup_url: 'https://app.relevanceai.com/integrations/...?provider=ahrefs&type=oauth',
}
```

A token needing reconnection carries `refresh_token_invalid: true`. `connected_by_you: false` marks an account this user didn't connect, usually one a teammate shared with the project: runs on it act as whoever connected it, in their account and with their access. With `provider`, `setup_url` is where the user connects an account of their own. Pass `detail: 'full'` for each token's scopes, metadata and dates.

## Finding Transformation OAuth Requirements

Use `relevance_get_transformation_requirements({ transformation_id })` — it lists the OAuth providers and API keys the step requires, and cross-references them against the project's connected accounts and configured keys in one call. Missing providers come back with `status: 'missing'` plus a setup URL the user can click. So does a provider whose accounts are connected without the permission the step needs; `accounts_missing_permission` counts them, and the fix is a reconnect with that permission ticked, or another connector that covers the job.

```typescript
relevance_get_transformation_requirements({
  transformation_id: 'ahrefs_-_ahrefs-get-backlinks-one-per-domain',
});
```

If you only need the raw schema (e.g. you're inspecting `oauth_permissions` metadata directly), call `relevance_get_transformation({ transformation_id })` and read `input_schema.properties.oauth_account_id.metadata.oauth_permissions`.

## Choosing the account

A provider can have several connected accounts, and a blank account input runs as the provider's first one, whoever that is. So choose one on purpose, once per provider per conversation.

First make sure it is the right provider. One vendor can have several, each with its own permissions: Google Docs can be written through `google` with `google-docs-read-write`, or through `google_drive_native`. An account without the step's permission cannot run it. Before asking the user to connect or reconnect, list their accounts with no provider filter and `detail: 'full'`; if another connected provider covers the job, switch to its native step.

1. List the provider's accounts with `relevance_list_oauth_accounts({ provider })`. An account is **usable** when it is shared with the project (`namespace: "project"`, or no `namespace` on an older account) or connected by this user (`connected_by_you: true`), and none of its tokens has `refresh_token_invalid: true`. Never offer or use a colleague's private account.
2. If the account marked `default_oauth_account: true` is usable, use it: the user chose it. If the only usable account has `connected_by_you: true`, use that. Otherwise ask the user, and keep using the answer:
   - Several usable accounts: ask which one, offering them.
   - The only usable account is someone else's: name it by its label, say it isn't one they connected and that runs will act as whoever connected it, and offer either using it or connecting their own at the listing's `setup_url`.
   - The only account needing reconnection is someone else's: the user can't reconnect it, so offer connecting their own at the listing's `setup_url`, or asking whoever connected it to reconnect.

   When the build uses an account the user didn't connect, name it again when you hand the build back.

3. Pass its `account_id` everywhere the account goes:

| Where                          | How                                                                                                  |
| ------------------------------ | ---------------------------------------------------------------------------------------------------- |
| Running a tool step            | the step's OAuth input, usually `oauth_account_id`                                                   |
| Running a tool                 | the tool's OAuth input                                                                               |
| Attaching tools to an agent    | `oauth_account_ids` on `relevance_attach_tools_to_agent`, one per provider                           |
| A tool already on an agent     | `default_values` via `relevance_update_agent_action`, which merges by key, so send just the account  |
| An agent trigger               | `oauth_account_id` inside `trigger_config` on `relevance_create_trigger`                             |
| A workforce trigger            | the trigger node's `oauth_account_id` in `relevance_create_workforce` / `relevance_update_workforce` |
| Work you hand to another agent | the `account_id` in its instructions                                                                 |

No account of that provider connected at all? Leave the input unset and send the user the setup link: omit it from `default_values` (send `null` to remove one already there), never an empty string. Once they connect, set the new `account_id` wherever the input goes: a blank input runs as the provider's first account, which may not be theirs. If accounts exist but none is usable, follow step 2 and the permission check above: ask the user to choose, reconnect or switch connector. Never write a placeholder or made-up account id: a value that matches no account fails every real run, even after one is connected.

## CRITICAL: OAuth for Agent-Called Tools

Put the account on the **agent's tool**, not on the tool itself. When an agent calls a tool, the account comes from that attached tool's `default_values`, and the input is set to "Set manually" (`overrides.params_schema.properties.<param>.metadata.is_fixed_param: true`) so the agent can't change it.

`relevance_attach_tools_to_agent` sets both: it fills in the account passed in `oauth_account_ids`, or the provider's only account. When that only account is someone else's, its `message` says so: ask as in [Choosing the account](#choosing-the-account) step 2. For a tool that's already attached, set it with `relevance_update_agent_action` as in the table above.

Don't set the account as a `default` in the tool's own `params_schema`. Every agent and user of the tool shares that default, so a private account there acts as its owner for everyone.

---

## Troubleshooting

### OAuth account not showing in dropdown

1. Ensure `content_type: oauth_account` is in metadata
2. Ensure `oauth_permissions` matches the provider and types
3. User must have connected an account with matching permissions

### "Invalid OAuth account"

The account ID doesn't exist or doesn't have the required permissions. User needs to reconnect OAuth.

### "Please enter a value for 'select' field"

Some transformations have required fields beyond OAuth. Check the transformation schema for all required parameters.

### "You need to add your chains_xxx_api_key API key" (Agent calling tool)

The agent's tool has no account for its OAuth input. Set it in the attached tool's `default_values` (see "OAuth for Agent-Called Tools" above).
