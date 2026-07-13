---
name: relevance-task-ops
description: Read the Task Ops (monitor) page for a project — the operator surface that lists every agent task. Use when the user asks to see their tasks, how many are errored / escalated / awaiting approval, to list the failing tasks, or to open a specific task and read its error.
---

# Task Ops Skill

Skill for **reading** the Task Ops page (the `monitor` page, titled "Tasks") — the surface an operator uses to triage every agent task in a project. This skill gives visibility into what that page shows. It is **read-only**: it surfaces state, counts, and failures, but does **not** diagnose root causes — diagnosis composes this skill (see [Related skills](#related-skills)).

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
2. `relevance_list_agent_task_messages(agent_id, task_id)` — the raw message/tool-run timeline, including the error message(s) with the actual failure text.
3. `relevance_get_agent_task_metadata(agent_id, task_id)` — task metadata (`has_errored`, timings, created-by).

## Reading the Errors tab (surface read, not diagnosis)

A typical "show me what's failing on my tasks" read:

1. `relevance_get_analytics_task_status_counts` over the window the user cares about (default: the page's list window — last 7 days inclusive, `start_date = now − 6 days`) — report the numbers and the window (e.g. "12 errored, 3 escalated, 1 awaiting approval in the last 7 days").
2. `relevance_list_tasks(view: "errored")` (same `date_range`) — list the errored tasks. Group them by `active_failures[].error_type` / `action` to show _what kinds_ of failures are hitting (e.g. "8 tasks failing on `send_email` with `auth_config`").
3. For any task the user wants to understand, drill in with `relevance_get_agent_task_summary` / `relevance_list_agent_task_messages` to surface the actual error text.

Stop at surfacing what happened. Root-causing _why_ and proposing/applying fixes is the diagnostics skill's job.

## Related skills

Load these on demand with `read_skill` when the flow reaches them:

- **`relevance-diagnostics/SKILL`** — when the user wants to know _why_ tasks are failing and what to fix: the macro→micro diagnosis routine. This Task Ops read is one of its inputs.
- **`relevance-analytics/SKILL`** — full docs for `relevance_get_analytics_task_status_counts` and the other metric tools (trends, breakdowns, error records).
- **`relevance-evals/SKILL`** — pre-deploy tests and production performance dashboards / regression monitoring, rather than the live task queue.
