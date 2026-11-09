---
name: managing-relevance-agents/notification-rules
description: Agent notification rules — an email, Slack or Teams message each time one of an agent's tasks waits for approval, is escalated, fails or finishes, or when the agent is published or its triggers change. Load when the user asks to be notified, pinged, emailed or told about an agent's tasks, and when an agent that a trigger or schedule runs gets a tool that asks for approval.
---

# Agent notification rules

A notification rule tells people about each of one agent's tasks as it happens. It is an entry in the agent's `notifications_config`. The app lists these as alarm rules on the agent's Alarms page (Agent › Build › Alarms), so point the user there. The alarms `relevance_create_alarm` makes are a different thing: they report a rate across many tasks, an eval score or a trigger failing (`relevance-task-ops`, `relevance-evals/monitoring`). When a response carries `alarms_url`, that link opens this same Alarms page.

## What they ask for, and what sets it up

| They want to hear                                  | Set up                                                                                                                                                      |
| -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| when a task is waiting for their approval          | a notification rule with `{ "type": "status_change", "statuses": ["pending-approval"] }`                                                                    |
| when a task fails                                  | `{ "type": "agent_error" }` for a run that stopped on an error; add `{ "type": "tool_error" }` when a failed tool call matters even if the agent carried on |
| when a task times out or can't recover             | `status_change` on `["timed-out", "unrecoverable"]`                                                                                                         |
| when a task finishes                               | `status_change` on `["idle"]`: an agent task returns to `idle` each time it finishes a run                                                                  |
| when a task is handed to a person                  | the agent's escalations, below                                                                                                                              |
| when someone publishes it, or changes its triggers | `{ "type": "agent_published" }` / `{ "type": "agent_triggers_updated" }`                                                                                    |
| when too many tasks fail, wait or are escalated    | a task alarm (`relevance-task-ops`), not a notification rule                                                                                                |
| when its quality drops                             | an eval alarm (`relevance-evals/monitoring`), not a notification rule                                                                                       |

"Let me know when it needs me", said about an agent whose tool asks for approval, is the first row.

## An approval nobody hears about

A tool call set to ask (`action_behaviour: "always-ask"`) waits in the task until someone approves it in the Relevance app. Nobody is told unless the agent has a notification rule on `pending-approval`, the agent has an older Slack notification (`external_message_relay_config`) on that status, or the task came from a Slack trigger that replies in its thread. When a person is watching the chat, they see it. When a trigger or schedule runs the agent and none of those reports the wait, set up the pending-approval notification rule for whoever approves, on the channel the user or their plan named; ask where they want to hear only if nobody did. On a new build, add it after your last test run of the draft, so no test run sends one; once it is in place, any run that reaches the approval sends it, test runs included. Say in your reply where approvals wait and who is told.

## Setting one up

1. Read the agent with `relevance_get_agent` and `summary: false`, and keep every entry already in `notifications_config.notifications`. If it has `escalations` or an older Slack notification (`external_message_relay_config`), the app shows its older Escalations section, where these rules don't appear: tell the user. A rule on a status replaces the older Slack notification for that status, approve and reject buttons included, so add one there only after telling the user and hearing yes.
2. Add the entry: `{ "id": "<any unique id>", "enabled": true, "trigger": {...}, "channels": [...] }`.
3. Save it with `relevance_update_agent`, `patch: { notifications_config: { notifications: [...every entry...] } }`. The list is replaced whole, so an entry left out is deleted.
4. It reaches live runs once the agent is published: publish as you normally would, or tell the user it waits on the draft.

Channels, one entry per destination type:

- **Email:** `{ "type": "email", "emails": ["..."] }`.
- **Slack:** `{ "type": "slack", "oauth_account_id": "...", "channel_id": "C..." }`. Find the id with `relevance_list_slack_channels` for that Slack account. The Relevance AI app must be in the channel (`/invite @Relevance AI`).
- **Teams:** `{ "type": "teams", "oauth_account_id": "...", "team_id": "...", "channel_id": "..." }`. The tools can't list Teams channels: ask the user for the ids, or have them set it in the app.

`tool_error` takes `tool_ids`, the tools' `chain_id`s (not their `action_id`s), to limit it to some tools; leave it out for every tool.

## How they behave

- **The published version's rules are the ones that send.** Until the agent is published with them, a run uses the draft's rules only when the published version has none, or the agent was never published. A test run in that state sends real notifications.
- **Inside a workforce.** A workforce has no notification rules of its own, but an agent's rules fire for its tasks inside a workforce. Put the rule on the agent whose tool asks.
- **A link, not buttons.** A Slack or Teams notification carries a link to the task, where the person approves. Approve and reject buttons in Slack come from elsewhere: a Slack trigger with `approve_reject_config: { enabled: true }` puts them in the thread of a task it started (see `managing-relevance-agents/triggers`), and an older Slack notification with the same setting puts them in its channel.
- **Escalations.** Escalation notices go to the agent's `escalations` setting, which the tools can't change, or to a notification rule on `escalated`. Never add an `escalated` rule to an agent that has `escalations` set: the rule takes over the notices, and the people in `escalations` stop getting them. On an agent with neither `escalations` nor an older Slack notification on `escalated`, an `escalated` rule is the only way escalations reach anyone, and it is what gives the agent its escalate tool: never remove or disable it unless the user asks for exactly that.
- **No mentions, no test send.** A notification posts to the channel without @-mentioning anyone, and there is no way to send a test one. A rule is proven by the first task that reaches its status.
