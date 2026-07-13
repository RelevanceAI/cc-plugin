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

**The most common OAuth mistake: declaring the account as a plain text input (or pasting a raw account id into a step).** A `type: "string"` property _without_ `content_type: "oauth_account"` renders a free-text box, not the account selector. The tool then receives an un-credentialed string, and every subsequent step that calls the API fails — even though the input "looks" filled in.

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

If you find an existing tool with this mistake, replace the plain-string input with the `oauth_account` form above and re-point the step's `oauth_account_id: "{{params.<name>}}"`. (Hardcoding the id at the **tool** level via `default` + `is_fixed_param` is only valid once the param already carries the OAuth metadata — see [OAuth for Agent-Called Tools](#critical-oauth-for-agent-called-tools).)

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

| Field               | Description                                             |
| ------------------- | ------------------------------------------------------- |
| `content_type`      | Must be `oauth_account` to show the account selector    |
| `oauth_permissions` | Array of required permissions                           |
| `provider`          | OAuth provider name (e.g., `ahrefs`, `google`, `slack`) |
| `types`             | Array of permission type strings required               |

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

| Provider        | Permission Type                     | Description            |
| --------------- | ----------------------------------- | ---------------------- |
| `hubspot`       | `hubspot-connect-app`               | HubSpot CRM API        |
| `ahrefs`        | `pipedream-ahrefs-read-write`       | Ahrefs API access      |
| `google`        | `email-read-write`                  | Gmail read/write       |
| `google`        | `calendar-read-write`               | Google Calendar        |
| `google_sheets` | `google-sheets-read-write`          | Google Sheets          |
| `google_drive`  | `pipedream-google-drive-read-write` | Google Drive           |
| `slack`         | `slack-channel-post`                | Post to Slack channels |
| `slack`         | `slack-notifications`               | Slack notifications    |
| `github`        | `pipedream-github-read-write`       | GitHub API access      |
| `notion`        | `pipedream-notion-read-write`       | Notion API access      |
| `twitter`       | `twitter-read-write`                | Twitter/X API access   |
| `linkedin`      | `unipile-linkedin`                  | LinkedIn via Unipile   |

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
// List all connected OAuth accounts
relevance_list_oauth_accounts();

// Returns:
{
  results: [
    {
      account_id: '<your-oauth-account-id>',
      provider: 'ahrefs',
      label: 'Ahrefs',
      tokens: [
        {
          token_id: '...',
          permission_types: ['pipedream-ahrefs-read-write'],
          scopes: ['ahrefs'],
        },
      ],
    },
  ];
}
```

## Finding Transformation OAuth Requirements

Use `relevance_get_transformation_requirements({ transformation_id })` — it lists the OAuth providers and API keys the step requires, and cross-references them against the project's connected accounts and configured keys in one call. Missing providers come back with `status: 'missing'` plus a setup URL the user can click.

```typescript
relevance_get_transformation_requirements({
  transformation_id: 'ahrefs_-_ahrefs-get-backlinks-one-per-domain',
});
```

If you only need the raw schema (e.g. you're inspecting `oauth_permissions` metadata directly), call `relevance_get_transformation({ transformation_id })` and read `input_schema.properties.oauth_account_id.metadata.oauth_permissions`.

## CRITICAL: OAuth for Agent-Called Tools

Agent actions can't carry `oauth_account_id` directly — the action config schema has no `params` field, and trying to add one fails with `"must NOT have additional properties"`. Common symptom: `"You need to add your chains_<provider>_api_key API key"`.

**Fix:** put the OAuth account on the **tool**, not the action. Call `relevance_get_tool` first, then `relevance_update_tool` with the **complete** `params_schema` (every existing property) — `update_tool` replaces `params_schema` wholesale, so a partial one drops the other params. On the OAuth param set: `default: "<oauth_account_id>"` (provides the value when the tool is called without it) AND `metadata.is_fixed_param: true` (hides the field from the UI). Both are required — `is_fixed_param` alone does not provide a value. Keep the rest of the OAuth metadata (`content_type: "oauth_account"`, `oauth_account_provider`, `oauth_permissions`) intact.

### Key Insights

| Field                   | Purpose                                                                   |
| ----------------------- | ------------------------------------------------------------------------- |
| `is_fixed_param: true`  | Hides the field from UI (user doesn't see it)                             |
| `default: "account-id"` | **Actually provides the value** when tool is called without the parameter |

**Both are required for agent-called tools:**

- `is_fixed_param` alone does NOT provide a value
- `default` is what the tool actually receives when called by an agent

### OAuth Setup Checklist for Agent Tools

- [ ] Add `oauth_account_id` parameter to tool's `params_schema`
- [ ] Set `"type": "string"`
- [ ] Add `metadata.content_type: "oauth_account"`
- [ ] Add `metadata.oauth_account_provider: "hubspot"` (or appropriate provider)
- [ ] Add `metadata.is_fixed_param: true` (hides from UI)
- [ ] **Add `"default": "your-oauth-account-id"`** (CRITICAL!)
- [ ] Add `metadata.oauth_permissions` array

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

The tool's `params_schema` is missing the `default` value for `oauth_account_id`. Add it to the tool definition (see "OAuth for Agent-Called Tools" section above).
