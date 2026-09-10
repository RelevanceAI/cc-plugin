---
title: Agent Triggers
description: Configure event-driven and scheduled triggers (webhooks, email, Slack, LinkedIn, recurring schedules) that feed agents work. An agent can have many triggers at once, including several recurring schedules. Load when setting up automation, adding a second trigger, choosing between polling and push, or writing the message a schedule sends.
---

# Agent Triggers

Configure agents to respond to external events.

> **Building a workforce instead?** Triggers attached to a **workforce** (multi-agent graph) use a slightly different (nested) config shape and live on the workforce's trigger node, not on an agent. See [`../managing-relevance-workforces/workforce-triggers.md`](../managing-relevance-workforces/workforce-triggers.md). Same vendors, two attachment points — pick whichever matches the entity that owns the run.

## Choosing the Right Trigger Pattern

Triggers **feed the agent's unit of action** — delivering work so the agent can act on it. The two main patterns are **event-driven** (react when something happens) and **scheduled** (run on a timer). Both are valid — the right choice depends on your data source and what the agent does.

### Event-Driven Triggers

Event-driven triggers fire **when something happens**. The external system pushes the event to Relevance AI, and the agent runs immediately in response.

| Trigger             | Event Source                                   | Unit of Action          |
| ------------------- | ---------------------------------------------- | ----------------------- |
| `custom_webhook`    | Zapier, Make, HubSpot workflow, CRM automation | One record with payload |
| `webhook`           | Simple HTTP POST from any source               | One request             |
| `gmail` / `outlook` | Incoming email                                 | One email to process    |
| `slack`             | Slack message                                  | One message/thread      |
| `unipile_linkedin`  | LinkedIn message                               | One conversation        |
| `google_calendar`   | Calendar event                                 | One meeting to prep     |

**Strengths:**

- **Immediate** — agent acts as soon as the event occurs
- **Efficient** — agent only runs when there's actual work
- **Clean scope** — each event is already one entity, no scanning needed

**Example: CRM field change triggers agent via webhook**

```
CRM (Salesforce/HubSpot/etc.)      Relevance AI
┌─────────────────────┐           ┌──────────────────────┐
│ Record updated       │           │ Webhook trigger       │
│ Meets criteria? ─YES─┼──HTTP──>  │ → Agent task:         │
│                 ─NO──│ (skip)    │   "Process Acme Corp" │
└─────────────────────┘           └──────────────────────┘
```

The CRM admin sets up a workflow/flow that fires when a field changes and meets the criteria. The agent receives one record at a time and acts on it.

**Example: Zapier catches a form submission**

```
Typeform → Zapier               Relevance AI
┌──────────────────────┐       ┌──────────────────────────┐
│ New submission        │       │ Custom webhook trigger     │
│ → Zap sends POST ────┼─HTTP─>│ → message_template maps   │
│   with form data      │       │   payload → Agent task:   │
│                       │       │   "Qualify this lead"     │
└──────────────────────┘       └──────────────────────────┘
```

No code needed — Zapier's webhook action sends the payload directly to the agent's `custom_webhook` URL. The `message_template` controls how the payload is passed to the agent.

**Example: Slack message triggers agent**

```
Slack workspace                 Relevance AI
┌──────────────────────┐       ┌──────────────────────┐
│ User posts in         │       │ Slack trigger          │
│ #support-requests ────┼──────>│ → Agent task:         │
│                       │       │   "Triage this ticket" │
└──────────────────────┘       └──────────────────────┘
```

The Slack trigger listens for messages in connected channels — each message becomes one agent task.

### Recurring (Scheduled) Triggers

Recurring triggers run on a schedule — every N minutes, daily, weekly, etc. They're the right choice for time-based work and for situations where event-driven triggers aren't available.

**Good fits for recurring triggers:**

| Use Case                          | Example                                              | Why Recurring Works                           |
| --------------------------------- | ---------------------------------------------------- | --------------------------------------------- |
| Periodic reports                  | Daily sales summary, weekly metrics                  | Time-based by nature — not event-driven       |
| Monitoring & health checks        | "Check system status every hour"                     | No external event to hook into                |
| Batch processing without webhooks | "Every morning, check for new rows in a spreadsheet" | Source system doesn't support outbound events |
| Digest / rollup tasks             | "Summarize all tickets from the past 24h"            | Aggregation across a time window              |
| Data sync from legacy systems     | "Pull new records from FTP/CSV daily"                | No API or webhook available                   |

**Example: Daily reporting**

```
Schedule: Every day at 9am           Relevance AI
┌──────────────────────────┐       ┌──────────────────────┐
│ Timer fires               │       │ Recurring trigger     │
│ → message: "Generate the  ┼──────>│ → Agent task:         │
│   daily pipeline report"  │       │   Queries CRM, builds │
│                           │       │   summary, posts to   │
│                           │       │   Slack                │
└──────────────────────────┘       └──────────────────────┘
```

**Example: Scanning when webhooks aren't available**

```
Schedule: Every 30 min               Relevance AI
┌──────────────────────────┐       ┌──────────────────────┐
│ Timer fires               │       │ Recurring trigger     │
│ → message: "Check Google  ┼──────>│ → Agent task:         │
│   Sheet for new rows and  │       │   Reads sheet, acts   │
│   process any unhandled"  │       │   on new entries      │
└──────────────────────────┘       └──────────────────────┘
```

When the source system can't push events, a recurring trigger that polls is perfectly reasonable.

### Tradeoffs at a Glance

|                   | Event-Driven                                             | Recurring                                             |
| ----------------- | -------------------------------------------------------- | ----------------------------------------------------- |
| **Runs when**     | Something happens                                        | Timer fires                                           |
| **Latency**       | Immediate                                                | Depends on interval                                   |
| **Setup effort**  | Requires webhook/integration config in source system     | Just set a schedule                                   |
| **Best for**      | Per-entity reactions (new email, record change, message) | Reports, monitoring, polling sources without webhooks |
| **Watch out for** | Needs admin access to configure source system            | May run when there's nothing new to do                |

### Decision Guide

```
Is this time-based work (reports, digests, monitoring)?
  │
  ├─ YES → Recurring trigger
  │
  └─ NO → Does the source system support webhooks or integrations?
           │
           ├─ YES → Event-driven trigger (webhook, email, Slack, etc.)
           │
           ├─ PARTIALLY (e.g., Zapier/Make available) →
           │    Use Zapier/Make to bridge: source → webhook trigger
           │
           └─ NO → Recurring trigger that polls the source
```

> **Tip:** If your recurring trigger scans for individual entities and acts on each one, consider whether the source system could push those events instead (via webhook, Zapier, etc.). Event-driven triggers give you one-entity-per-task naturally, while a scanning agent has to manage its own deduplication and error recovery. But if webhooks aren't practical for your setup, a recurring scanner is a valid approach.

---

## Multiple Triggers Per Agent

**An agent can have as many triggers as you want, of any type, at the same time.** There is no
one-trigger-per-agent limit and no one-recurring-schedule-per-agent limit. Creating a second trigger
does **not** replace the first.

- To **add** a trigger, call `relevance_create_trigger` **without** `document_id`.
- To **edit** an existing trigger, pass its `document_id` (from `relevance_list_agent_triggers`).

That distinction is the whole thing: `document_id` is what makes the call an edit. If you pass the
same `document_id` twice, of course the second call overwrites the first — that's an update, not a
platform limit.

Combinations that are all fine:

- Five separate `weekly` recurring triggers (Mon–Fri), each with its own message.
- A `daily` digest **plus** a `gmail` trigger **plus** a `custom_webhook` on one agent.
- Two `slack` triggers on the same workspace, watching different channels.

### Worked example: weekdays only at 08:00 Sydney

Two valid shapes — pick by whether the days need _different_ instructions.

**Option A — five weekly triggers** (use when each day's message differs, e.g. Monday needs a
longer lookback to cover the weekend, or you want to pause Friday independently):

```typescript
for (const day of ['mon', 'tue', 'wed', 'thu', 'fri']) {
  await relevance_create_trigger({
    agent_id: '...',
    trigger_type: 'recurring',
    // no document_id — each call adds another trigger
    trigger_config: {
      name: `Morning digest (${day})`,
      message: '...', // see "Writing the Trigger Message" below
      schedule: {
        frequency: 'weekly',
        day_of_week: day,
        hour: '08:00',
        timezone: 'Australia/Sydney',
      },
    },
  });
}
```

**Option B — one `custom_cron` trigger** (use when all five days do the same thing):

```typescript
relevance_create_trigger({
  agent_id: '...',
  trigger_type: 'recurring',
  trigger_config: {
    name: 'Morning digest (weekdays)',
    message: '...',
    schedule: {
      frequency: 'custom_cron',
      cron_expression: '0 8 ? * MON-FRI *',
      timezone: 'Australia/Sydney', // optional but recommended — defaults to UTC
    },
  },
});
```

> **Set `timezone` on `custom_cron` too.** It's optional and defaults to UTC, but a cron expression
> written against a fixed UTC offset drifts by an hour when the local zone changes for daylight
> saving. Naming the IANA zone makes the platform handle the shift.

Five triggers means five entries in the trigger list and five things to keep in sync; one cron means
one message for all five days. Neither is a workaround for the other — choose on whether the days
need different instructions.

## Trigger Types

| Type               | Description                   | Use Case                                       |
| ------------------ | ----------------------------- | ---------------------------------------------- |
| `gmail`            | Email received                | Email assistant, auto-replies                  |
| `outlook`          | Outlook email                 | Enterprise email workflows                     |
| `google_calendar`  | Calendar events               | Meeting prep, scheduling                       |
| `slack`            | Slack messages                | Team automation                                |
| `unipile_linkedin` | LinkedIn messages             | Sales outreach                                 |
| `unipile_whatsapp` | WhatsApp messages             | Customer support                               |
| `unipile_telegram` | Telegram messages             | Bot automation                                 |
| `webhook`          | Simple webhook endpoint       | Basic HTTP POST trigger                        |
| `custom_webhook`   | Webhook with message template | Zapier, Make, CRM workflows with field mapping |
| `recurring`        | Scheduled execution           | Daily reports, checks                          |

## Managing Triggers

### List Agent Triggers

```typescript
relevance_list_agent_triggers({ agent_id: '...' });
```

### Create Trigger

Omit `document_id` — the agent keeps any triggers it already has. See
[Multiple Triggers Per Agent](#multiple-triggers-per-agent).

```typescript
relevance_create_trigger({
  agent_id: '...',
  trigger_type: 'gmail',
  trigger_config: {
    oauth_account_id: '...',
    oauth_account_label: 'My Gmail',
  },
});
```

The response includes the new `document_id` and `created` — `true` when the call added a trigger,
`false` when it replaced an existing one. Read it rather than assuming: a `false` on a call you meant
as a create means the id was already taken, not that the agent is limited to one trigger.

### Enable / Disable Trigger

```typescript
relevance_update_trigger_status({
  agent_id: '...',
  trigger_id: 'trigger-doc-id', // document_id from relevance_list_agent_triggers
  status: 'paused', // 'in_progress' = enable/resume, 'paused' = disable
});
```

To turn a trigger off (or back on), **pause/resume it — don't delete and recreate it**. New triggers are enabled by default.

### Delete Trigger

```typescript
relevance_delete_trigger({ document_id: 'trigger-doc-id' });
```

## Trigger Configurations

### Email Triggers (Gmail/Outlook)

```typescript
relevance_create_trigger({
  agent_id: '...',
  trigger_type: 'gmail', // or "outlook"
  trigger_config: {
    oauth_account_id: 'oauth-account-uuid',
    oauth_account_label: 'Work Gmail', // optional
  },
});
```

### LinkedIn Trigger

```typescript
relevance_create_trigger({
  agent_id: '...',
  trigger_type: 'unipile_linkedin',
  trigger_config: {
    oauth_account_id: '...',
    provider_user_id: '...', // LinkedIn user ID
    is_outreach_reply_only: false, // true = only reply to outreach
  },
});
```

### WhatsApp/Telegram Triggers

```typescript
relevance_create_trigger({
  agent_id: '...',
  trigger_type: 'unipile_whatsapp', // or "unipile_telegram"
  trigger_config: {
    oauth_account_id: '...',
    provider_user_id: '...',
  },
});
```

### Slack Trigger

Listens for messages in specific Slack channels.

**Setup steps:**

1. Find Slack OAuth account: `relevance_list_oauth_accounts()`
2. Resolve channel names to IDs: `relevance_list_slack_channels({ oauth_account_id: "..." })`
3. Create the trigger

```typescript
relevance_create_trigger({
  agent_id: '...',
  trigger_type: 'slack',
  trigger_config: {
    oauth_account_id: 'slack-oauth-account-uuid',
    channels: ['C07AGHNGV9Q'], // Channel IDs from relevance_list_slack_channels
    keywords: {
      // Optional: only trigger on matching messages
      values: ['help', 'support'],
      config: { case_sensitive: false },
    },
    user_ids: [], // Optional: filter to specific Slack user IDs
    thread_reply_mode: 'auto', // 'auto' = reply in thread, 'none' = new message
    should_mention_bot: true, // true = only when @mentioned
  },
});
```

**Notes:**

- `channels` requires Slack channel **IDs** (e.g. `C07AGHNGV9Q`), not names. Use `relevance_list_slack_channels` to look up IDs.
- Set `should_mention_bot: false` to respond to every message in the channel.
- `keywords` is optional — omit or pass `{ values: [] }` to match all messages.

### Google Calendar Trigger

Triggers the agent on calendar events (e.g., upcoming meetings).

```typescript
relevance_create_trigger({
  agent_id: '...',
  trigger_type: 'google_calendar',
  trigger_config: {
    oauth_account_id: 'google-oauth-account-uuid',
    calendar_id: 'primary', // 'primary' or a specific calendar ID
    events: {
      notifications: [
        {
          timeOffset: {
            quantity: 10,
            unit: 'minutes',
            direction: 'before',
          },
          message: { type: 'raw' },
        },
      ],
    },
  },
});
```

### Teams Trigger

Listens for messages in Microsoft Teams channels.

```typescript
relevance_create_trigger({
  agent_id: '...',
  trigger_type: 'teams',
  trigger_config: {
    oauth_account_id: 'teams-oauth-account-uuid',
    tenant_id: 'microsoft-tenant-id',
    channels: [],
    keywords: {
      values: [],
      config: { case_sensitive: false },
    },
    excluded_keywords: [],
    thread_reply_mode: 'auto',
    should_mention_bot: true,
  },
});
```

### Recurring Trigger

Schedule agents to run automatically at specified intervals.

**Frequency Options:**

Every frequency except `custom_cron` requires `hour` (HH:mm) **and** `timezone` (IANA string) — yes, even `minutely` and `hourly`. A schedule missing a required field is rejected with a 400 before it is saved.

| Frequency     | Required Fields                             | Description                                                                                                   |
| ------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `minutely`    | `hour`, `timezone`, `minute_interval`       | Every N minutes (min: 10)                                                                                     |
| `hourly`      | `hour`, `timezone`, `hour_interval`         | Every N hours (min: 1)                                                                                        |
| `daily`       | `hour`, `timezone`                          | Once per day                                                                                                  |
| `weekly`      | `hour`, `timezone`, `day_of_week`           | Once per week                                                                                                 |
| `monthly`     | `hour`, `timezone`, `day_of_month`          | Once per month (`day_of_month`: 1-31 or `'last_day'`)                                                         |
| `annually`    | `hour`, `timezone`, `day_of_month`, `month` | Once per year                                                                                                 |
| `no_repeat`   | `hour`, `timezone`, `date`                  | One-time execution (`date`: YYYY-MM-DD, must be in the future)                                                |
| `custom_cron` | `cron_expression`                           | 6-field cron expression (minute intervals min 10). `hour` not required; `timezone` optional (defaults to UTC) |

**Example: Every 15 minutes**

```typescript
relevance_create_trigger({
  agent_id: '...',
  trigger_type: 'recurring',
  trigger_config: {
    name: 'Check LinkedIn Comments',
    message: 'Process this LinkedIn post: 123456789',
    schedule: {
      frequency: 'minutely',
      minute_interval: '15', // "10", "15", "30", "45"
      hour: '09:00', // Required even for minutely
      timezone: 'UTC',
    },
  },
});
```

**Example: Daily at 9am**

```typescript
relevance_create_trigger({
  agent_id: '...',
  trigger_type: 'recurring',
  trigger_config: {
    name: 'Daily Report',
    message: 'Generate the daily sales report',
    schedule: {
      frequency: 'daily',
      hour: '09:00', // HH:mm format
      timezone: 'America/New_York',
    },
  },
});
```

**Example: Every Monday at 8am**

```typescript
relevance_create_trigger({
  agent_id: '...',
  trigger_type: 'recurring',
  trigger_config: {
    name: 'Weekly Summary',
    message: 'Generate weekly metrics summary',
    schedule: {
      frequency: 'weekly',
      day_of_week: 'mon', // "mon"|"tue"|"wed"|"thu"|"fri"|"sat"|"sun"
      hour: '08:00',
      timezone: 'UTC',
    },
  },
});
```

**Example: Custom cron**

```typescript
relevance_create_trigger({
  agent_id: '...',
  trigger_type: 'recurring',
  trigger_config: {
    name: 'Business Hours Check',
    message: 'Run health check',
    schedule: {
      frequency: 'custom_cron',
      cron_expression: '0 9 ? * MON-FRI *', // 9am Mon-Fri
      timezone: 'America/New_York', // optional; without it the cron runs in UTC
    },
  },
});
```

### Writing the Trigger Message

The `message` is the entire instruction the agent receives when the schedule fires. Getting it wrong
is the most common reason a correctly-configured recurring trigger produces useless runs. Four
properties of a scheduled run drive everything below:

1. **The message is static.** The same text is sent every single fire. It's a standing instruction,
   not a one-off request.
2. **The run starts a fresh conversation.** No history, no earlier context, no memory of the last
   run. Whatever the agent needs must be in the message or the system prompt.
3. **The agent knows when it fired, but not what you meant by it.** Each run carries a timestamp, so
   a window expressed as a duration ("the last 24 hours") resolves correctly. But that timestamp
   reaches the agent in **UTC**, not the schedule's timezone — which is a _different weekday_ from
   local for a morning schedule in Sydney or an evening one in Los Angeles — and nothing tells it
   which trigger fired or how often. So: durations are safe; "it's Monday" is not. If a run's
   behaviour genuinely depends on the local day, put that day in its own trigger rather than asking
   the agent to work it out.
4. **It runs whether or not there's work.** A daily trigger fires on quiet days too.

So the message should:

- **Name the run.** Open with a short label and cadence: `"MORNING DIGEST — daily 08:00 run."` This
  is how the agent (and anyone reading the task list) tells one schedule's runs from another's.
- **Give a window as a duration, not a date.** `"cover everything from the last 24 hours"` — never a
  hardcoded date, and never "work out what day it is and pick a lookback" (that's a coin flip on
  every run, and it silently breaks around weekends and holidays).
- **Prefer a window the tool understands natively.** Gmail's `newer_than:3d`, an API's
  `since`/`updated_after` parameter, a relative filter — these are exact. Ask the agent to compute
  an absolute date only when the tool gives you no relative option.
- **State the exit condition.** `"If there is nothing new, reply NOTHING TO REPORT and stop."`
  Without it, an agent with nothing to do will pad, re-report old items, or invent work — on every
  quiet day, at full cost.
- **Name where the output goes** — post to Slack, send an email, or just reply.

**❌ Bad**

```
MORNING_DIGEST — Monday run. Lookback window: 72 hours (since Friday).
Calculate the date 3 days ago and use it as after_date for the Gmail fetch.
Use today's date for the calendar lookup.
```

Assumes it only ever fires on Monday (a `daily` schedule fires seven days a week), makes the agent
do date arithmetic, and never says what to do when the inbox is empty.

**✅ Good** — the Tue–Fri trigger (`custom_cron`, `0 8 ? * TUE-FRI *`):

```
MORNING DIGEST — scheduled 08:00 run, Tuesday to Friday.

1. Fetch email from the last 24 hours using the Gmail search filter `newer_than:1d`.
2. Fetch today's calendar events.
3. Post a summary to #daily-digest: urgent emails first, then meetings.

If there is no new email and no meetings, reply "NOTHING TO REPORT" and stop —
do not post to Slack.
```

**✅ Good** — the Monday trigger (`weekly`, `day_of_week: 'mon'`), same agent, second trigger:

```
MORNING DIGEST — scheduled 08:00 Monday run, covering the weekend.

1. Fetch email from the last 72 hours using the Gmail search filter `newer_than:3d`.
2. Fetch today's calendar events.
3. Post a summary to #daily-digest: urgent emails first, then meetings.

If there is no new email and no meetings, reply "NOTHING TO REPORT" and stop —
do not post to Slack.
```

> **One trigger per distinct instruction.** Monday needs a 72-hour window and the other weekdays
> need 24, so that's two triggers — each with its own fixed window — not one message asking the
> agent to work out the day and branch. Two triggers on one agent is the normal shape here, and each
> can be paused or edited on its own. See
> [Multiple Triggers Per Agent](#multiple-triggers-per-agent).

### Webhook Triggers

There are two webhook trigger types. **Use `custom_webhook` for most integrations** (Zapier, Make, CRM workflows) — it supports message templates and field mapping. Use plain `webhook` only when you need a simple endpoint with no message transformation.

#### Custom Webhook (recommended for Zapier, Make, CRM integrations)

```typescript
relevance_create_trigger({
  agent_id: '...',
  trigger_type: 'custom_webhook',
  trigger_config: {
    webhook_name: 'Zapier Webhook', // optional: display name
    webhook_description: 'Receives form submissions', // optional: description
    message_template: '{{$}}', // required: template for agent message
    // Use {{$}} to pass the entire payload, or {{field_name}} for specific fields
    mapping: {
      unique_id: '', // optional: jq path for idempotency (e.g. ".data.id")
      thread_id: '', // optional: jq path for conversation threading (e.g. ".data.thread_id")
    },
  },
});
```

#### Simple Webhook

Use when you just need a URL to POST to with no message transformation.

```typescript
relevance_create_trigger({
  agent_id: '...',
  trigger_type: 'webhook',
  trigger_config: {}, // Returns webhook URL to call
});
```

## Finding OAuth Account IDs

Most triggers require OAuth account IDs:

```typescript
// List all OAuth accounts
relevance_list_oauth_accounts()

// Returns:
{
  results: [
    {
      account_id: "a1eda1a8-7571-...",
      provider: "google",
      label: "Work Gmail",
      tokens: [...]
    },
    {
      account_id: "b2edb2b9-8682-...",
      provider: "unipile",
      provider_user_id: "linkedin-user-id",
      ...
    }
  ]
}
```

For Unipile triggers (LinkedIn/WhatsApp/Telegram), you also need `provider_user_id` from the OAuth account.

## Trigger Document IDs

Each trigger has a `document_id` assigned by the platform. **Treat it as opaque — never construct,
guess, or derive one.** Always read it back from `relevance_list_agent_triggers`:

```typescript
const triggers = await relevance_list_agent_triggers({ agent_id: '...' });
const digest = triggers.results.find((t) => t.data.config.type === 'recurring');

relevance_delete_trigger({ document_id: digest.document_id });
```

You need the `document_id` to update, pause/resume, or delete a trigger. You do **not** need it to
create one — omitting it is what makes the call create a new trigger rather than edit an existing
one.

## Example: Email Assistant Setup

```typescript
// 1. List OAuth accounts to find Gmail
const accounts = await relevance_list_oauth_accounts();
const gmail = accounts.results.find((a) => a.provider === 'google');

// 2. Create email trigger
relevance_create_trigger({
  agent_id: 'email-assistant',
  trigger_type: 'gmail',
  trigger_config: {
    oauth_account_id: gmail.account_id,
  },
});
```

## Example: LinkedIn Outreach Bot

```typescript
// 1. Find LinkedIn OAuth account
const accounts = await relevance_list_oauth_accounts();
const linkedin = accounts.results.find(
  (a) => a.provider === 'unipile' && a.provider_type === 'linkedin'
);

// 2. Create LinkedIn trigger
relevance_create_trigger({
  agent_id: 'linkedin-bot',
  trigger_type: 'unipile_linkedin',
  trigger_config: {
    oauth_account_id: linkedin.account_id,
    provider_user_id: linkedin.provider_user_id,
    is_outreach_reply_only: true, // Only respond to outreach replies
  },
});
```
