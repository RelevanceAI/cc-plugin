---
name: relevance-analytics
description: Retrieves usage analytics for Relevance AI agents and projects. Use when asked about agent/workforce/tool usage, credit spend, error rates, breakdowns, task-status counts, most active agents, execution counts, or usage trends over time.
---

# Relevance AI Analytics

Skill for retrieving and analyzing usage data, credit spend, error rates, resource breakdowns, and task-status counts.

## Dedicated Tools

Six tools are available for analytics queries. Optional filters follow one convention: **omit a filter to include everything** (all agents, all workforces, all origins).

### relevance_get_analytics_resource_breakdown

Per-resource scorecard. Pick `resource_type`: `agent`, `workforce`, or `tool`. Returns the matching dashboard breakdown table. Use it to rank resources and find the error-prone or expensive ones.

```typescript
// Per-agent scorecard
relevance_get_analytics_resource_breakdown({
  resource_type: 'agent',
  start_date: '2026-03-01',
  end_date: '2026-03-31',
});
```

**Response by `resource_type`:**

- `agent` → `{ agents: [{ agent_id, agent_name, task_count, error_rate, error_count, total_actions, credits_used, top_actions: [{ action_name, execution_count }] }] }`
- `workforce` → `{ workforces: [{ workforce_id, workforce_name, workforce_tasks, agent_tasks, error_rate, error_count, total_actions, credits_used, top_agents }] }`
- `tool` → `{ actions: [{ action_id, action_name, execution_count, error_rate, error_count, credits_used, top_agents: [{ agent_id, agent_name, task_count }] }] }`

Each row carries both `error_rate` (percentage) and `error_count` (raw failure count) — rank by `error_count` when volume matters. Response field names keep the API's legacy "action" vocabulary (`actions`, `action_id`, `total_actions`) — those refer to tools.

Optional filter: `resource_ids` — IDs of the selected `resource_type`. Omit to include all.

### relevance_get_analytics_errors

Recent errors from agent conversations, workforce tasks, and tool runs, plus a daily error-count time series. Use `resource_type` to scope to one origin; paginate with `cursor`.

```typescript
relevance_get_analytics_errors({
  start_date: '2026-03-01T00:00:00.000Z',
  end_date: '2026-03-31T23:59:59.999Z',
  resource_type: 'tool',
  page_size: 50,
});
```

**Response:** `{ errors: [{ source_type, source_id, source_name, source_context: { agent_id, agent_name, conversation_id, workforce_id, workforce_name, workforce_task_id }, error_time, error_message }], has_more, next_cursor, time_series: [{ date, count }] }`

Per-row `source_type` is only `agent` or `tool` — rows never carry `source_type: "workforce"`. Workforce-origin errors are identified by `source_context.workforce_id`. `time_series` is first-page only — omitted when `cursor` is set.

Inputs use ISO 8601 datetimes. Optional: `agent_ids`, `workforce_ids`, `resource_type` (`agent` | `workforce` | `tool`; omit for all origins), `page_size` (1–100), `cursor`, `timezone` (IANA, for day bucketing). ID filters are `agent_ids` / `workforce_ids` only — there is no `resource_ids` or tool-id filter on this tool.

### relevance_get_analytics_usage_timeseries

Daily usage time series for one project metric, with a summary. Use for usage trends over time (the error trend lives in `relevance_get_analytics_errors`).

**Metrics:** `total_tasks`, `total_actions`, `total_credits`, `tasks_to_review`, `daily_active_users`, `tasks_escalated`

```typescript
// Credit spend over the last 30 days
relevance_get_analytics_usage_timeseries({
  metric: 'total_credits',
  start_date: '2026-03-01',
  end_date: '2026-03-31',
});
```

**Response:** `{ metric_type, data_points: [{ timestamp, value, grouped_by }], summary: { total, average, min_value, max_value } }`

Optional: `agent_ids` (omit for all agents).

### relevance_get_analytics_task_status_counts

Current task-status counts: `total`, plus `to_review` (needs human action), `escalated`, and `errored` buckets, and `by_state` (raw count per conversation state). Counts reflect the current state of tasks created in the window. Use for a quick health snapshot.

```typescript
relevance_get_analytics_task_status_counts({
  start_date: '2026-03-01T00:00:00.000Z',
  end_date: '2026-03-31T23:59:59.999Z',
});
```

**Response:** `{ total, to_review, escalated, errored, by_state: { [state]: count } }`

> The named buckets **overlap and are not additive** — `escalated` is also counted in `to_review`, and `errored-pending-approval` is in both `to_review` and `errored`. Don't sum them; use `total` and `by_state` for exact, mutually-exclusive counts. `errored` matches the Task Ops errored tab (errored states OR active failures attached), so it can exceed the sum of errored states in `by_state`.

Inputs use ISO 8601 datetimes. Optional: `agent_ids` (omit for all agents). For the per-task list use `relevance_list_tasks` (filterable by view/state/agent/date — the Task Ops queue surface; see the `relevance-task-ops` skill); for trends use `relevance_get_analytics_usage_timeseries`.

### relevance_get_analytics_agent_activity_timeseries

Agent state time-series (running, idle, pending-approval) and task creation trends.

```typescript
relevance_get_analytics_agent_activity_timeseries({
  agent_ids: ['agent-id-here'],
  start_date: '2026-03-01',
  end_date: '2026-03-31',
});
```

**Response:** `{ states_results: { timeseries: [{ insert_date_, source_id, event_value, frequency, bucket }], total_change, change_per_event_value }, tasks_created_results: { timeseries, total_change } }`

Optional: `agent_ids` (omit for all agents).

### relevance_get_analytics_usage_breakdown

Usage ranked by resource type — find which agents, tools, or workforces consume the most.

```typescript
// Which agents used the most credits?
relevance_get_analytics_usage_breakdown({
  resource_type: 'agent',
  metric: 'credit',
  start_date: '2026-03-01',
  end_date: '2026-03-31',
});
```

**Metrics:** `credit` (credit spend) or `executions` (execution count).

**Response:** `{ query: { group, metric, from, to }, series: [{ id, actor: { id, type, metadata: { label, avatar_url } }, metric, value }] }` (the response echoes the API's legacy vocabulary: `group`, and `metric: "action"` for executions).

## Common Queries

### Project health snapshot

Call `relevance_get_analytics_task_status_counts` for total / to-review / escalated / errored, and `relevance_get_analytics_errors` (with `resource_type`) for the actual error messages.

### Find error-prone or expensive resources

Use `relevance_get_analytics_resource_breakdown` with the relevant `resource_type` — returns error rate, error count, and credits per agent / workforce / tool in one call.

### Track a metric over time

Use `relevance_get_analytics_usage_timeseries` with the metric you care about (e.g. `total_credits`, `tasks_escalated`) for daily trends.

### Who spent the most

Use `relevance_get_analytics_usage_breakdown` with `resource_type` `agent` / `tool` / `workforce`.

## Tips

1. **Date formats:** `relevance_get_analytics_resource_breakdown`, `relevance_get_analytics_usage_timeseries`, `relevance_get_analytics_agent_activity_timeseries`, and `relevance_get_analytics_usage_breakdown` take `YYYY-MM-DD`. `relevance_get_analytics_errors` and `relevance_get_analytics_task_status_counts` take full ISO 8601 datetimes (e.g. `2026-03-01T00:00:00.000Z`).
2. **Access-limited tools:** some analytics tools need elevated project permissions or product entitlement. If a tool returns a permission / forbidden / entitlement / access-denied error, treat that signal as unavailable — don't retry repeatedly or read it as zero data.
3. **One breakdown tool:** `relevance_get_analytics_resource_breakdown` covers agents, workforces, and tools — switch with `resource_type`.
4. **Pagination:** `relevance_get_analytics_errors` supports cursor-based pagination via `cursor`.
