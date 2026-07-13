---
title: Workforce Triggers
description: Attach Slack / Gmail / Outlook / recurring / custom_webhook / Salesforce / HubSpot / Jira / Calendar / LinkedIn / WhatsApp / Telegram / tool_trigger trigger nodes to a workforce. Load when building a workforce that needs to run on an external event, schedule, or as a tool of another workforce.
---

# Workforce Triggers

Configure trigger **nodes** on a workforce so it runs on an external event (email, Slack message, webhook, CRM record change, …), on a schedule, or as a tool another agent can call. Pass the trigger configs to `relevance_create_workforce` via the `triggers` argument.

> **Sibling skill:** [`../managing-relevance-agents/triggers.md`](../managing-relevance-agents/triggers.md) covers **agent-level** triggers (attached to a standalone agent via `relevance_create_trigger`). Workforce triggers attach to the workforce graph's trigger node and use a different (nested) config shape — same underlying vendors, two attachment points. Pick whichever maps to the entity that owns the run.

## Two Trigger Modes

A workforce trigger node has two modes:

- **`manual`** — runs when something calls `relevance_trigger_workforce` (or a manual "Test" run in the Relevance app UI). This is the default when you omit `triggers`.
- **`auto`** — runs on an external event or schedule defined by `sync_data.config`. Requires the vendor-specific config (see Trigger Types below).

```typescript
relevance_create_workforce({
  name: 'My workforce',
  agents: [{ agentId: 'agent_id_here' }],
  triggers: [
    { type: 'manual' }, // (default — same as omitting `triggers`)
  ],
});
```

> **Every workforce always carries a manual trigger.** When you pass `triggers` with only `auto` entries, a `{ type: 'manual' }` node is appended automatically (every workforce always starts with one). That preserves the `relevance_trigger_workforce` testing path, which fires the manual trigger. Caller-supplied trigger indices in `edges[].source.index` stay stable — the manual is appended at the end. The appended manual is also auto-wired: it mirrors the target of your first trigger edge (so a manual test run drops into the same downstream graph as the auto trigger). If you supply a manual trigger yourself, you own its wiring — the auto-wire only applies to the manual that was appended for you. To remove the manual entirely, send a follow-up `relevance_update_workforce` with `remove_node_ids` (rare).

## Multiple Trigger Nodes

A workforce can have **more than one** trigger node — for example a `recurring` daily run **and** a `custom_webhook` for ad-hoc requests, both feeding the same agent. Add them to the `triggers` array and use `trigger_index` on `edges` to route each one:

```typescript
relevance_create_workforce({
  name: 'Lead Pipeline',
  agents: [{ agentId: 'qualifier_id' }],
  triggers: [
    {
      type: 'auto',
      sync_data: {
        config: {
          type: 'recurring',
          recurring: {
            name: 'Daily scan',
            message: 'Scan CRM for new leads',
            schedule: { frequency: 'daily', hour: '09:00', timezone: 'UTC' },
          },
        },
      },
    },
    {
      type: 'auto',
      sync_data: {
        config: {
          type: 'custom_webhook',
          custom_webhook: {
            webhook_name: 'Ad-hoc lead',
            mapping: {},
            message_template: '{{$}}',
          },
        },
      },
    },
  ],
  edges: [
    {
      source: { kind: 'trigger', index: 0 },
      target: { kind: 'agent', index: 0 },
    }, // recurring → qualifier
    {
      source: { kind: 'trigger', index: 1 },
      target: { kind: 'agent', index: 0 },
    }, // webhook → qualifier
  ],
});
```

`source: { kind: "trigger", index: N }` selects which trigger node feeds the edge. Indexes refer to the order of entries in the `triggers` array you passed to `relevance_create_workforce`.

## sync_data shape

Every `auto` trigger has the same outer shape — only the inner `config` varies per vendor:

```typescript
sync_data: {
  config: {
    type: '<vendor>',
    <vendor>: { /* vendor-specific fields */ },
  },
}
```

The `sync_data.destination = { type: "workforce", workforce_id }` is filled in automatically; you only supply `config`.

## OAuth handling per vendor (save-anyway vs connect-first)

Whether a trigger can save without an OAuth account depends on the vendor's `private_settings.required` array in [`apps/nodeapi/src/routeConfigAndSchemas/sync_configs.ts`](../../../../apps/nodeapi/src/routeConfigAndSchemas/sync_configs.ts). Two categories:

### Save-anyway vendors

`private_settings.required: []` — the workforce saves as a draft without an OAuth account; the trigger fires once the user connects the provider.

- `hubspot` (line 762)
- `recurring` (`shared_settings.required: ["schedule"]`)
- `custom_webhook` (`shared_settings.required: ["mapping", "message_template"]`)
- `freshdesk`
- `zoominfo` (org-level API key, not OAuth)
- `relevance_meeting_bot`
- `tool_trigger` (no OAuth account involved)

HubSpot, before OAuth is connected:

```typescript
relevance_create_workforce({
  name: 'Inbound triage',
  agents: [{ agentId: 'orchestrator-id' }],
  triggers: [
    {
      type: 'auto',
      sync_data: { config: { type: 'hubspot', hubspot: {} } },
    },
  ],
});
```

### Connect-first vendors

`private_settings.required` includes `oauth_account_id` (and sometimes more); `SaveWorkforce` rejects the save without it, so `relevance_list_oauth_accounts` is a precondition.

| Vendor                                         | `private_settings.required`                                       | Source                        |
| ---------------------------------------------- | ----------------------------------------------------------------- | ----------------------------- |
| `gmail`                                        | `["oauth_account_id"]`                                            | sync_configs.ts:468           |
| `outlook`                                      | `["oauth_account_id"]`                                            | sync_configs.ts:725           |
| `slack`                                        | `["oauth_account_id"]`                                            | sync_configs.ts:1303          |
| `teams`                                        | `["oauth_account_id"]`                                            | sync_configs.ts:1355          |
| `google_calendar`                              | `["oauth_account_id", "oauth_account_label", "calendar_id"]`      | sync_configs.ts:507           |
| `teams_calendar`                               | `["oauth_account_id", "oauth_account_label", "calendar_id"]`      | sync_configs.ts:617           |
| `google_drive`                                 | `["oauth_account_id"]`                                            | sync_configs.ts:1504          |
| `jira`                                         | `["oauth_account_id"]`                                            | sync_configs.ts:1458          |
| `salesforce`                                   | `["oauth_account_id"]`                                            | sync_configs.ts:952           |
| `unipile_linkedin` / `_whatsapp` / `_telegram` | `["oauth_account_id", "oauth_account_label", "provider_user_id"]` | sync_configs.ts:787, 812, 837 |

When the provider isn't connected, save the workforce without the trigger, ask the user to connect it in Integrations, then add the trigger via `relevance_update_workforce` with the populated `oauth_account_id`. Saving with `oauth_account_id` omitted is rejected by `SaveWorkforce` (and pre-validated by the tools).

## Trigger Types

These are the trigger `type` values the workforce builder exposes — and therefore the only ones the tools accept. The list mirrors `apps/builder-app/features/workforce/definitions.ts`. Backend-only sync types (`webhook`, `pipedream`, `sharepoint`, `notion`, `confluence`, `zoom`, `external_relay`, `bulk_*`, etc.) are deliberately not callable through `relevance_create_workforce` / `relevance_update_workforce`.

For OAuth fields per vendor, see [OAuth handling per vendor](#oauth-handling-per-vendor-save-anyway-vs-connect-first) above.

### Available (default)

The standard workforce triggers. Always present in the builder picker (subject to the feature-gate note below).

| Type                    | OAuth | Use case                                          |
| ----------------------- | ----- | ------------------------------------------------- |
| `manual`                | —     | Manual / API-triggered runs (default)             |
| `recurring`             | No    | Scheduled execution (cron, every N min, daily, …) |
| `custom_webhook`        | No    | Zapier / Make / CRM workflows with field mapping  |
| `tool_trigger`          | Yes   | Workforce run as a tool from another agent        |
| `gmail`                 | Yes   | Incoming Gmail                                    |
| `outlook`               | Yes   | Incoming Outlook email                            |
| `slack`                 | Yes   | Slack message in connected channel                |
| `teams`                 | Yes   | Microsoft Teams message (feature-gated)           |
| `google_calendar`       | Yes   | Google Calendar event                             |
| `teams_calendar`        | Yes   | Microsoft Teams Calendar event                    |
| `hubspot`               | Yes   | HubSpot CRM event                                 |
| `jira`                  | Yes   | Jira issue event (feature-gated)                  |
| `google_drive`          | Yes   | Google Drive file change (feature-gated)          |
| `filesystem`            | —     | Mounted filesystem changes (feature-gated)        |
| `relevance_meeting_bot` | No    | Relevance meeting-bot post-call event             |
| `zoominfo`              | No    | ZoomInfo intent-data polling (legacy)             |
| `freshdesk`             | No    | Freshdesk ticket events (legacy)                  |

### Premium

Adds entitlement: Pro plan +5000 credits/month per connected account.

| Type               | OAuth | Use case         |
| ------------------ | ----- | ---------------- |
| `unipile_linkedin` | Yes   | LinkedIn message |
| `unipile_whatsapp` | Yes   | WhatsApp message |
| `unipile_telegram` | Yes   | Telegram message |

### Enterprise

Requires the Enterprise plan.

| Type         | OAuth | Use case                            |
| ------------ | ----- | ----------------------------------- |
| `salesforce` | Yes   | Salesforce CRM (SOQL query) polling |

### External (link-only)

`zapier` lives outside these tools — point users at `https://zapier.com/apps/relevance-ai/integrations`. Do not try to set up `zapier` as a `sync_data.config.type`; it is not a backend sync type.

### Coming soon (not callable)

`whatsapp` is listed in the UI for awareness but is not yet wired up. The tools reject it. Use `unipile_whatsapp` if the user wants WhatsApp today.

### Feature-gated triggers

`teams`, `jira`, `google_drive`, and `filesystem` are gated behind PostHog feature flags and may not be visible in every org. If `relevance_list_oauth_accounts` doesn't return the provider you need, or the user reports they can't see a trigger in the builder, the trigger is gated for that org and the workforce save will fail. Surface this back to the user instead of retrying.

### Recurring (schedule)

```typescript
sync_data: {
  config: {
    type: 'recurring',
    recurring: {
      name: 'Daily summary',
      message: 'Generate the daily sales summary',
      schedule: {
        frequency: 'daily', // 'minutely' | 'hourly' | 'daily' | 'weekly' | 'monthly' | 'annually' | 'no_repeat' | 'custom_cron'
        hour: '09:00', // required for every frequency except custom_cron
        timezone: 'America/New_York', // required for every frequency except custom_cron
        // minutely:  minute_interval: '15' (min 10)
        // hourly:    hour_interval: '2' (min 1)
        // weekly:    day_of_week: 'mon'
        // monthly:   day_of_month: '1' (1-31 or 'last_day')
        // annually:  day_of_month + month
        // no_repeat: date: 'YYYY-MM-DD' (future)
        // custom_cron: cron_expression: '0 9 ? * MON-FRI *' (hour/timezone not needed)
      },
    },
  },
}
```

### custom_webhook (Zapier / Make / CRM flow)

```typescript
sync_data: {
  config: {
    type: 'custom_webhook',
    custom_webhook: {
      webhook_name: 'Zapier inbound',
      message_template: '{{$}}', // '{{$}}' = whole payload; '{{field_name}}' = a field
      mapping: {
        unique_id: '.data.id',     // optional: jq path for idempotency
        thread_id: '.data.thread_id', // optional: jq path for threading
      },
    },
  },
}
```

### tool_trigger (workforce fired by a tool's output)

Required: `name`, `run_schedule`, `tool_config.{id, region, project, output_field}` (see `sync_configs.ts:345-406`).

`run_schedule` takes the same shape as a recurring trigger's `schedule` — the same per-frequency required fields apply (every frequency except `custom_cron` needs `hour` + `timezone`; `minutely` also needs `minute_interval` >= 10, etc.).

**Precondition:** the source tool must declare `output_field` via `relevance_set_tool_output` AND must actually return that key at runtime. `output_field` is read by literal key lookup against the tool's runtime output. See [`../managing-relevance-tools/outputs.md`](../managing-relevance-tools/outputs.md) for the full model.

> **⚠️ Don't change the source tool's outputs after wiring it as a trigger.** Removing or renaming a key that any `tool_trigger.output_field` points at causes the trigger to fail with `Tool output field not found`, without surfacing the failure to the user. If you must change the tool's output shape, surface it explicitly to the user and update the trigger's `output_field` in the same change.

```typescript
sync_data: {
  config: {
    type: 'tool_trigger',
    tool_trigger: {
      name: 'Run lead-qualifier workforce',
      run_schedule: {
        frequency: 'minutely',
        minute_interval: '15', // min 10
        hour: '09:00', // required even for minutely
        timezone: 'UTC',
      },
      tool_config: {
        id: '<tool-id from relevance_list_tools>',
        region: '<from relevance_get_project_info>',
        project: '<project-id>',
        // `output_field` matches a key the source tool declares via `relevance_set_tool_output`.
        output_field: 'qualified_leads',
      },
    },
  },
}
```

**End-to-end example.** Create the tool with its step(s) and the paired output map + schema in one shot, then reference it from the trigger:

```typescript
const tool = await relevance_create_tool({
  title: 'Beep/boop sequence',
  transformations: {
    steps: [
      {
        name: 'gen',
        transformation: 'javascript_code',
        params: {
          code: "const beeps = Array(5).fill('beep'); const boops = Array(3).fill('boop'); return { sequence: [...beeps, ...boops].join(' ') };",
        },
        output: { sequence: '{{transformed.sequence}}' },
      },
    ],
    output: { sequence: '{{steps.gen.output.sequence}}' },
  },
  output_schema: {
    metadata: { field_order: ['sequence'] },
    properties: { sequence: { metadata: { render_mode: 'raw' } } },
  },
});

const project = await relevance_get_project_info();

await relevance_create_workforce({
  name: 'Beep/boop workforce',
  agents: [{ agentId: orchestrator.agent_id }],
  triggers: [
    {
      type: 'auto',
      sync_data: {
        config: {
          type: 'tool_trigger',
          tool_trigger: {
            name: 'Run on tool output',
            run_schedule: {
              frequency: 'minutely',
              minute_interval: '15',
              hour: '09:00',
              timezone: 'UTC',
            },
            tool_config: {
              id: tool.studio_id,
              region: project.region,
              project: project.project_id,
              output_field: 'sequence',
            },
          },
        },
      },
    },
  ],
});
```

### Gmail / Outlook (incoming email)

```typescript
sync_data: {
  config: {
    type: 'gmail', // or 'outlook'
    gmail: {
      oauth_account_id: '<from relevance_list_oauth_accounts>', // REQUIRED
      is_attachments_enabled: true,
      is_outreach_reply_only: false,
    },
  },
}
```

### Slack (channel messages)

Resolve channel IDs first with `relevance_list_slack_channels`:

```typescript
sync_data: {
  config: {
    type: 'slack',
    slack: {
      oauth_account_id: '<slack-oauth-account-uuid>', // REQUIRED
      channels: ['C07AGHNGV9Q'], // channel IDs, not names
      keywords: { values: [], config: { case_sensitive: false } }, // empty = match all
      user_ids: [],
      thread_reply_mode: 'auto', // 'auto' or 'none'
      should_mention_bot: true,
    },
  },
}
```

### Teams (channel messages)

```typescript
sync_data: {
  config: {
    type: 'teams',
    teams: {
      oauth_account_id: '<teams-oauth-account-uuid>', // REQUIRED
      tenant_id: '<microsoft-tenant-id>',
      channels: [],
      keywords: { values: [], config: { case_sensitive: false } },
      excluded_keywords: [],
      thread_reply_mode: 'auto',
      should_mention_bot: true,
    },
  },
}
```

### Google Calendar / Teams Calendar

```typescript
sync_data: {
  config: {
    type: 'google_calendar', // or 'teams_calendar'
    google_calendar: {
      oauth_account_id: '<google-oauth-account-uuid>', // REQUIRED
      oauth_account_label: 'Work calendar', // REQUIRED
      calendar_id: 'primary', // REQUIRED
      events: {
        notifications: [
          {
            timeOffset: { quantity: 10, unit: 'minutes', direction: 'before' },
            message: { type: 'raw' },
          },
        ],
      },
    },
  },
}
```

### Salesforce (SOQL polling)

```typescript
sync_data: {
  config: {
    type: 'salesforce',
    salesforce: {
      oauth_account_id: '<salesforce-oauth-account-uuid>', // REQUIRED
      salesforce_query:
        'SELECT Id, Name, Email FROM Lead WHERE CreatedDate > YESTERDAY',
      page_size: 100,
      dedupe_field: 'Id',
    },
  },
}
```

### HubSpot

```typescript
sync_data: {
  config: {
    type: 'hubspot',
    hubspot: {
      // HubSpot is the canonical save-anyway vendor — both fields are optional;
      // omit them to save the trigger as a draft and wire OAuth up later.
      oauth_account_id: '<hubspot-oauth-account-uuid>', // optional
      provider_user_id: '<hubspot-portal-user-id>', // optional
    },
  },
}
```

### Jira

```typescript
sync_data: {
  config: {
    type: 'jira',
    jira: {
      oauth_account_id: '<jira-oauth-account-uuid>', // REQUIRED
      cloud_id: '<atlassian-cloud-id>',
      events: ['issue_created', 'issue_updated'],
      jql_filter: 'project = ENG AND priority = High',
    },
  },
}
```

### Google Drive

```typescript
sync_data: {
  config: {
    type: 'google_drive',
    google_drive: {
      oauth_account_id: '<google-oauth-account-uuid>', // REQUIRED
      oauth_account_label: 'Work drive',
    },
  },
}
```

### Filesystem

Watches a mounted filesystem volume for changes. Feature-gated — the user must have the `FilesystemSync` flag enabled.

```typescript
sync_data: {
  config: {
    type: 'filesystem',
    filesystem: { volume_key: '<volume-key>' },
  },
}
```

### Relevance Meeting Bot

```typescript
sync_data: {
  config: {
    type: 'relevance_meeting_bot',
    relevance_meeting_bot: { bot_id: '<bot-id>' },
  },
}
```

### ZoomInfo (legacy)

ZoomInfo polls intent data. The config requires an API key set up on the org (no OAuth). Set this up the first time in the Relevance app UI; the resulting config can be inspected with `relevance_get_workforce` and replicated.

```typescript
sync_data: {
  config: {
    type: 'zoominfo',
    zoominfo: {
      intent_topics: ['<topic>'],
      signal_score_min: '<score-enum>',       // see UI for valid values
      audience_strength_min: '<strength-enum>',
      company_country: 'US',
      next_page_number: 1,
    },
  },
}
```

### Freshdesk (legacy)

```typescript
sync_data: {
  config: {
    type: 'freshdesk',
    freshdesk: {
      whitelist: [
        {
          source_object: 'ticket',
          source_properties: [
            { field_name: 'priority', whitelist_values: ['High'] },
          ],
        },
      ],
    },
  },
}
```

### Unipile LinkedIn / WhatsApp / Telegram

```typescript
sync_data: {
  config: {
    type: 'unipile_linkedin', // or 'unipile_whatsapp' | 'unipile_telegram'
    unipile: {
      oauth_account_id: '<unipile-oauth-account-uuid>', // REQUIRED
      oauth_account_label: '<label-from-relevance_list_oauth_accounts>', // REQUIRED
      provider_user_id: '<provider-user-id>', // REQUIRED
      provider: 'linkedin', // 'linkedin' | 'whatsapp' | 'telegram'
      is_outreach_reply_only: false,
    },
  },
}
```

## Finding OAuth Account IDs

```typescript
relevance_list_oauth_accounts();
// → { results: [{ account_id, provider, label, provider_user_id, ... }] }
```

Pick the account whose `provider` matches the trigger vendor (e.g. `google`, `slack`, `microsoft`, `unipile`, `hubspot`, `salesforce`, `jira`). For Unipile triggers, also pass `provider_user_id` from the same OAuth account.

## Updating a Trigger After Creation

Trigger nodes are part of the workforce graph, so updating them goes through `relevance_update_workforce` (not `relevance_create_trigger`, which is for agent-level triggers). Fetch the current graph, patch the trigger node's `config`, and pass the modified `workforce_graph` back:

```typescript
const wf = await relevance_get_workforce({ workforce_id: '...' });
const triggerNode = wf.workforce_graph.nodes.find((n) => n.type === 'trigger');
triggerNode.config = {
  type: 'auto',
  sync_data: {
    config: {
      type: 'recurring',
      recurring: {
        /* new schedule */
      },
    },
  },
};
await relevance_update_workforce({
  workforce_id: '...',
  workforce_graph: wf.workforce_graph,
});
```

## Enabling / Disabling a Trigger

To turn a trigger off (or back on), **pause/resume it — don't remove and re-add the node**:

```typescript
relevance_update_workforce_trigger_status({
  workforce_id: '...',
  trigger_group_id: '<trigger node_id from relevance_get_workforce>',
  status: 'paused', // 'in_progress' = enable/resume, 'paused' = disable
});
```

Works only after the workforce is **published** — trigger syncs are persisted on publish, so toggling a draft-only trigger errors with a "publish first" message. New triggers are enabled by default on publish.

## Common Pitfalls

- **Manual + auto in one node is not a thing.** Each trigger node is one mode. Add a second trigger node if you need both.
- **`destination` is auto-filled.** Don't pass it; `{ type: "workforce", workforce_id }` is injected for you.
- **`channels` requires IDs.** Slack/Teams channel names are not accepted. Resolve with `relevance_list_slack_channels` or the equivalent Teams lookup.
- **OAuth triggers save before the account is connected** — see [OAuth-backed triggers: save first, connect later](#oauth-backed-triggers-save-first-connect-later) for the rule, the HubSpot-no-OAuth example, and the user-response template. Do NOT abort the workforce build because OAuth is missing; do NOT call `relevance_list_oauth_accounts` as a precondition.
- **Polling triggers (`salesforce`, `recurring`) run on a backend schedule.** Don't expect them to fire instantly after creation; the first poll happens at the next scheduled tick.
