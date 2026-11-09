---
name: relevance-task-ops
description: Read the Task Ops (monitor) page for a project — the operator surface that lists every agent task — and manage task alarms on task errors, pending approvals and escalations. Use when the user asks to see their tasks, how many are errored / escalated / awaiting approval, to list the failing tasks, to open a specific task and read its error, or to set up, change or remove an alarm on an agent's or workforce's task error rate, pending-approval rate or escalation rate.
---

# Task Ops Skill

Skill for **reading** the Task Ops page (the `monitor` page, titled "Tasks") — the surface an operator uses to triage every agent task in a project. This skill gives visibility into what that page shows: it surfaces state, counts, and failures, but does **not** diagnose root causes — diagnosis composes this skill (see [Related skills](#related-skills)). Its only writes are task alarms, on the task error rate, the pending-approval rate or the escalation rate (see [Task alarms](#task-alarms)).

## Overview

Task Ops has three levels, and this skill maps a tool to each:

- **Tab counts** — how many tasks are awaiting approval / escalated / errored. Use `relevance_get_analytics_task_status_counts` (the analytics counts tool, windowed).
- **Task list** — the table of tasks for a selected tab/view (task name, agent, state, dates). Use `relevance_list_tasks`. Each errored task also carries a category-level `active_failures` summary.
- **Task detail (drill-in)** — opening one task to read its full message timeline, where the actual error text lives. Use the agent-task tools.

## When to use what

| You want to…                                                             | Use…                                                                                 |
| ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------ |
| Count tasks by status (how many errored / escalated / awaiting approval) | `relevance_get_analytics_task_status_counts` (pick a window, e.g. last 7 days)       |
| List the actual tasks in a tab (e.g. "show me the errored tasks")        | `relevance_list_tasks` with `view: "errored"` (or `approvals` / `escalated` / `all`) |
| Read why a specific task failed (the real error text)                    | Drill in — `relevance_get_agent_task_summary` → `relevance_list_agent_task_messages` |
| Narrow to a time window or a single agent                                | `relevance_list_tasks` with `date_range` and/or `agent_id`                           |

## Concepts

- **Task** — a conversation (message thread between a user, an agent, and its tools). Every row on the Task Ops page is a task. Its `task_id` is the conversation id.
- **Views (tabs)** — Task Ops groups tasks by `conversation.state`, matching the monitor page tabs:
  - **`all`** — every task that isn't `completed`.
  - **`approvals`** — `state = pending-approval` (a tool call is waiting for human approval).
  - **`escalated`** — `state = escalated` (the agent handed the task to a human).
  - **`errored`** — `state ∈ {errored-pending-approval, unrecoverable, cancelled, timed-out}` **OR** the task has at least one active failure. This mirrors the Errors tab exactly. Note `cancelled` is a distinct terminal state that lands in this tab; treat it as "stopped", not necessarily a hard failure.
- **`active_failures`** — a per-task, deduplicated list of unresolved tool failures, each `{ action, tool_title, error_type }`. This is a **category-level summary** (which action failed and its error class) — it is present on the task list, so you can see _what kind_ of failure a task hit without opening it. It is cleared when the failure resolves.
- **Error text vs summary** — the list gives you the **state** and the `active_failures` **summary** only. The **actual error message / stack / detail** lives inside the task's messages (as error messages in the timeline). To read it, drill into the task (see below).

## Tools

### Counts: `relevance_get_analytics_task_status_counts`

The analytics counts tool (documented fully in the `relevance-analytics` skill). Takes a required ISO-datetime window (`start_date` / `end_date`) and optional `agent_ids`; returns `{ total, to_review, escalated, errored, by_state }` for tasks **created in the window**.

Two caveats to carry into your read:

- **Its named buckets overlap and are not additive** (`escalated` is also inside `to_review`; `errored-pending-approval` is in both `to_review` and `errored`). Use `total` and `by_state` for exact, mutually-exclusive numbers. Its `errored` bucket matches this skill's `errored` view semantics (errored states OR active failures).
- **Match the page's window deliberately when comparing numbers.** The page shows two different windows: the tab **badges** count **all-time**, while the task **list** defaults to the **last 7 days inclusive** — from `now − 6 days` (to the minute), with no upper bound. A count over any other window (e.g. a midnight-aligned "last 7 days") will match neither; that's the window, not a data discrepancy. To mirror the page's default list window pass `start_date = now − 6 days`, `end_date = now`. Always say which window you counted.

### List: `relevance_list_tasks`

Lists tasks for the Task Ops surface. Key inputs:

- **`view`** — `"all"` | `"approvals"` | `"escalated"` | `"errored"`. Omit (or `"all"`) for every **non-completed** task (matching the All tab). This is the usual way to pick a tab. To see states the views exclude (e.g. completed tasks), pass explicit `states`.
- **`agent_id`** — restrict to one agent's tasks (omit for the whole project, matching the page).
- **`date_range`** — `{ field?, from?, to? }` (ISO 8601). Omit for **all-time** (matches the tab badges). The page's task list defaults to the **last 7 days inclusive** — pass `{ from: <now − 6 days> }` with no `to` to match what the user sees in the list.
- **`states`** — explicit `conversation.state` values, if you need a filter the `view` presets don't cover.
- **`search`** — free-text over task titles. **`page` / `page_size`** — pagination (default 20).

Each result includes `task_id`, `agent_id`, `title`, `state`, `has_active_failures`, `active_failures`, `errored_at`, and `updated_at`.

The list is all-time by default while the counts tool is windowed — when comparing the two, query the list with the same `date_range` (on `insert_datetime`) as the counts window.

### Drill-in: reading a task's error

Once you have a `task_id` + `agent_id` from `relevance_list_tasks`, read the failure detail with the standard agent-task tools:

1. `relevance_get_agent_task_summary(agent_id, task_id)` — status, messages, and tool calls in one view.
2. `relevance_list_agent_task_messages(agent_id, task_id)` — the message/tool-run timeline, including the error message(s). Where it returns an `output_preview`, that is truncated at 500 characters, so use it to locate a failure, not to read a value in full; for what a tool actually received, read the run itself with `relevance_list_tool_runs`.
3. `relevance_get_agent_task_metadata(agent_id, task_id)` — task metadata (`has_errored`, timings, created-by).

## Reading the Errors tab (surface read, not diagnosis)

A typical "show me what's failing on my tasks" read:

1. `relevance_get_analytics_task_status_counts` over the window the user cares about (default: the page's list window — last 7 days inclusive, `start_date = now − 6 days`) — report the numbers and the window (e.g. "12 errored, 3 escalated, 1 awaiting approval in the last 7 days").
2. `relevance_list_tasks(view: "errored")` (same `date_range`) — list the errored tasks. Group them by `active_failures[].error_type` / `action` to show _what kinds_ of failures are hitting (e.g. "8 tasks failing on `send_email` with `auth_config`").
3. For any task the user wants to understand, drill in with `relevance_get_agent_task_summary` / `relevance_list_agent_task_messages` to surface the actual error text.

Stop at surfacing what happened. Root-causing _why_ and proposing/applying fixes is the diagnostics skill's job.

## Task alarms

A task alarm notifies email, Slack or Microsoft Teams when too large a share of an agent's or workforce's tasks is in one Task Ops tab. Create, change and remove it with `relevance_create_alarm`, `relevance_update_alarm` and `relevance_delete_alarm`, passing the `type` from the table below with `resource_type` (`agent` or `workforce`) and `resource_id`, plus the alarm's `alarm_id` to change or remove it. The type is required and is never inferred. `relevance_list_alarms` and `relevance_get_alarm` show these alarms, and `config.source_config.source_type` says which kind each one is.

| Alarm             | `type`             | `source_type`     | Fires when too many tasks… | Its rate from `relevance_get_analytics_task_status_counts`        |
| ----------------- | ------------------ | ----------------- | -------------------------- | ----------------------------------------------------------------- |
| Task errors       | `task_errors`      | `task_error`      | error                      | `errored` ÷ (`total` − `by_state.completed`)                      |
| Pending approvals | `task_approvals`   | `task_approval`   | wait for approval          | `by_state["pending-approval"]` ÷ (`total` − `by_state.completed`) |
| Escalations       | `task_escalations` | `task_escalation` | are escalated              | `by_state.escalated` ÷ (`total` − `by_state.completed`)           |

An agent or workforce has at most one task alarm of each type. The alarms of different types are separate: change only the one the user names.

- **What it measures.** The rate is the tasks in the alarm's view (`errored`, `approvals` or `escalated`) divided by all tasks that are not `completed`, counting the tasks created in the last `condition_config.period` before each check. The alarm fires when that rate is above `condition_config.threshold`. An agent's alarm counts all of the agent's tasks, including its runs inside workforces. A workforce's alarm counts every task its agents ran in that workforce.
- **Notifications, and an optional diagnosis.** By default the alarm only notifies its channels, and nobody follows up for the user. With `auto_diagnose: true`, each breach notification it sends also wakes Invent to find the cause: which failures caused a task-error breach, why tasks wait in a pending-approval breach, or why tasks are escalated in an escalation breach. Invent sends its ranked fixes to the alarm's own email or Slack channels, or to its creator's account email when the alarm has only Microsoft Teams channels. Offer it when the user wants the cause, not only the notification. Without a diagnosis, do not promise to come back when the alarm fires.

### Pending-approval alarms

- **Each task, or too many tasks.** To hear about every task that starts to wait, use one of the agent's notification rules, not this alarm: a `notifications_config` entry with a `status_change` trigger on `pending-approval` (see `managing-relevance-agents/notification-rules`). A workforce has no notification rules of its own, but its agents' rules fire for their tasks inside it, so add the rule to the agent whose tool asks. If the request could mean either, ask the user whether they want to hear about each task or about too many waiting tasks.
- **Approved tasks leave the count.** A task stops counting as soon as someone approves it. Most older tasks are already approved, so the 14-day rate reads much lower than the rate in the current period. Read both, and suggest a threshold above the higher one.
- **Agents that ask before every task.** A current rate near 100% means people take longer than one period to approve. A short period then fires at every check, even at normal approval speed. Suggest a period longer than the team's usual approval time, with a threshold above the rate in that period. Or tell the user that this alarm measures how fast people approve, not a fault, so they can decide not to set it. For example, an agent gets 12 tasks an hour and the team approves in about 2 hours. In the last hour 83% of its tasks wait, so a 1-hour period fires all day. Over 1 day about 8% wait, so a 1-day period with a 20% threshold fires only when approvals fall about 5 hours behind.

### Escalation alarms

- **Each task, or too many tasks.** Each escalation already sends an escalation notice to the agent's escalation channels, so to hear about every escalated task, use those, not this alarm. The channels are the agent's `escalations` setting, or a `notifications_config` entry with a `status_change` trigger on `escalated`. To change who gets the notices, change the channels of that entry, or ask the user to change `escalations` in the Relevance app, because the tools cannot change it. Never add a `status_change` entry for `escalated` to an agent that has `escalations` set: the entry takes over the notices, and the people in `escalations` stop getting them.
- **Answered tasks leave the count.** A task stops counting as soon as someone replies to its escalation. Most older escalations are already answered, so the 14-day rate reads lower than the rate in the current period. Read both, and suggest a threshold above the higher one.
- **Agents that escalate by design.** Some agents hand many tasks to a person on purpose, so their rate is high when nothing is wrong. Suggest a threshold above the agent's own rate, or a longer period.

### Choosing the threshold: the user chooses, you suggest

If the user gave a number, use it. Otherwise:

1. Read the history for the last 14 days, and compute the rate from the table:
   - **Agent:** `relevance_get_analytics_task_status_counts` with `agent_ids: [id]`.
   - **Workforce:** no read gives the alarm's exact rate, so always tell the user that the rate is approximate. Read the workforce's agents with `relevance_get_workforce`, then call `relevance_get_analytics_task_status_counts` with `agent_ids` set to them. The counts also include runs those agents made outside the workforce. For errors, do not use the `error_rate` from `relevance_get_analytics_resource_breakdown`: it leaves out tasks that only have tool failures, so it can read 0% while tasks are failing. Say it plainly, for example: "About 33% of its tasks errored (2 of 6). This is approximate: it also counts its agents' runs outside the workforce."
   - **Pending approvals and escalations:** also read the current period (`start_date` = now − the period, `end_date` = now), because approved or answered tasks lower the 14-day rate.
   - **No history, or no access:** start at 8% and say it is a starting point, not a measured rate.
2. Tell the user the rate and the volume, for example: "This agent errored on 3% of its 1,200 tasks in the last 14 days."
3. Suggest a threshold higher than that rate (for pending approvals and escalations, higher than both rates), and ask the user to choose. A threshold at or below the current rate fires on normal traffic.

The counts cover only the tasks the user can see. When the rate may not cover every task, say that it is approximate.

### Choosing the period: the user chooses, you suggest

Always check this, even when the user picks the threshold. A period with fewer than `min_sample_size` tasks never fires.

1. Work out the tasks in one period: the tasks in the last 14 days ÷ 14 × the period length in days.
2. Find the shortest period in which that number is at least `min_sample_size`. For example, an agent with 30 tasks a day has about 1 task in each hour, so a 1-hour period never fires; 12 hours holds about 15 tasks.
3. If even 14 days holds fewer than `min_sample_size` tasks, suggest a lower `min_sample_size` that fits, and tell the user that in a small sample one errored task is already a high rate.
4. For pending approvals, also make the period longer than the team's usual approval time (see [Pending-approval alarms](#pending-approval-alarms)). For escalations, make it longer than the team's usual reply time (see [Escalation alarms](#escalation-alarms)).

Suggest that period, say why, and ask the user to choose.

## Related skills

Load these on demand when the flow reaches them:

- **`relevance-diagnostics/SKILL`** — when the user wants to know _why_ tasks are failing and what to fix: the macro→micro diagnosis routine. This Task Ops read is one of its inputs.
- **`relevance-analytics/SKILL`** — full docs for `relevance_get_analytics_task_status_counts` and the other metric tools (trends, breakdowns, error records).
- **`relevance-evals/SKILL`** — pre-deploy tests and production performance dashboards / regression monitoring, rather than the live task queue.
