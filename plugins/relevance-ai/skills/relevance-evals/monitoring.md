---
name: relevance-evals/monitoring
description: Performance dashboards on live runs — setup, tools, reading results, investigating failures, alarms.
---

# Monitoring with Performance Dashboards

## Post-Deploy Workflow (Performance Dashboards)

Performance dashboards monitor live production traffic by sampling conversations and scoring them with checks. They start scoring once the agent or workforce is published, and can be created on a draft beforehand.

> **Heads-up:** dashboards only score the **latest published version** of the agent/workforce. Creating a dashboard on an unpublished agent or workforce succeeds, but no runs will fire until it's published — especially worth calling out when you've just generated a new agent/workforce and haven't published it yet. If `relevance_create_performance_dashboard` returns a `publish_hint` field, surface it to the user verbatim.

### How a performance dashboard works

1. The dashboard is enabled with `sample_rate ∈ (0, 1]` on an agent or workforce.
2. Each time a production task settles into a status that passes the dashboard's `task_status` filter, it is sampled with probability `sample_rate`. An agent conversation settles at the end of every run, so a multi-turn conversation can be scored once per turn.
3. Each sampled task is scored against the dashboard's checks by the LLM judge.
4. Scores feed the timeseries chart and the per-test table — surfaced via `relevance_get_performance_dashboard_timeseries` and `relevance_list_performance_dashboard_runs`.

### Which statuses to score (`task_status`)

Leave `task_status` unset to score every status a task settles in. When you set it, use the resource's real end states:

- **Agent:** a run ends `idle`. A conversation is `completed` only once someone marks it complete, so `['completed']` alone scores only those. For finished runs use `['idle', 'completed']`.
- **Workforce:** a task ends `completed` (or `execution-limit-reached`) by itself.

### Setup

```
┌──────────────────────────────────────────────────────────────────┐
│  1. Create or reuse the checks (max 50)                          │
│     └─► relevance_list_eval_checks / relevance_create_eval_check │
│     └─► created/updated checks: calibrate on finished runs first │
│  2. Create the dashboard from those check_ids                    │
│     └─► relevance_create_performance_dashboard with              │
│         check_ids: [...], sample_rate, enabled                   │
└──────────────────────────────────────────────────────────────────┘
```

Example:

```
relevance_create_eval_check(resource_type: "agent", resource_id: "<agent_id>",
  name: "Greets user", check_config: { type: "llm_judge", prompt: "The agent greets the user" })
→ { "check_id": "<check_greet>" }
relevance_create_eval_check(resource_type: "agent", resource_id: "<agent_id>",
  name: "No raw errors leaked",
  check_config: { type: "llm_judge", prompt: "The reply contains no raw API error messages, stack traces, or internal IDs" })
→ { "check_id": "<check_no_errors>" }

relevance_create_performance_dashboard(
  resource_type: "agent", resource_id: "<agent_id>",
  name: "Production quality watch",
  check_ids: ["<check_greet>", "<check_no_errors>"],
  sample_rate: 0.1,             // 10% chance per settle
  enabled: true
)
```

### Performance Dashboard Tools

| Tool                                             | Description                                                                                                                           |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| `relevance_create_performance_dashboard`         | Create a dashboard from existing checks (`check_ids`, max 50), `sample_rate`, `enabled`, and optional `task_status`                   |
| `relevance_list_performance_dashboards`          | List dashboards for an agent or workforce (returns each dashboard's checks, sample rate, enabled flag, status filter)                 |
| `relevance_update_performance_dashboard`         | Update dashboard **metadata** (name / sample_rate / enabled / task_status) using patch semantics; `task_status: []` clears the filter |
| `relevance_delete_performance_dashboard`         | Delete a dashboard (loses historical runs; underlying checks are NOT deleted)                                                         |
| `relevance_get_performance_dashboard_timeseries` | Score timeseries for a dashboard between two dates                                                                                    |
| `relevance_list_performance_dashboard_runs`      | List eval runs from a dashboard, optionally filtered to a single check and/or failed-only                                             |

### Performance Dashboard Check Management (attach / detach existing checks)

Mirror of the test-case surface — attach/detach checks on a dashboard (**not** via `relevance_update_performance_dashboard`). Creating and editing the checks themselves lives in [the check library](SKILL.md#checks-are-a-reusable-library-reference-them-by-id).

| Tool                                          | Description                                                                                                                      |
| --------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `relevance_add_performance_dashboard_rule`    | Attach existing checks to a dashboard by `check_id` (`check_ids`); fails if it would exceed 50                                   |
| `relevance_remove_performance_dashboard_rule` | Detach a check from a dashboard by `check_id` — unlinks here only; a dashboard must keep ≥1 check (delete the dashboard instead) |

Both take the dashboard's `config_id`.

### Dashboard URLs

`relevance_create_performance_dashboard`, `relevance_list_performance_dashboards`, `relevance_update_performance_dashboard`, and `relevance_get_performance_dashboard_timeseries` all return a dashboard `url`. The per-check mutators (`relevance_add_performance_dashboard_rule`, `relevance_remove_performance_dashboard_rule`) embed the URL in their text response. Share the dashboard URL with the user the same way you share eval-run URLs.

### Reading dashboards

- **Timeseries** for the chart view (window must be ≤ 31 days):

  ```
  relevance_get_performance_dashboard_timeseries(
    resource_type, resource_id,
    config_id: "<dashboard_id>",
    start_date: "2026-05-06T00:00:00Z",   // last 7 days
    end_date:   "2026-05-13T00:00:00Z",
    check_id:   "<optional check_id to scope to a single check>"
  )
  ```

- **Per-run table** — `start_date` and `end_date` are **required**:

  ```
  relevance_list_performance_dashboard_runs(
    resource_type, resource_id,
    config_id: "<dashboard_id>",
    start_date: "2026-05-06T00:00:00Z",   // last 7 days
    end_date:   "2026-05-13T00:00:00Z",
    failed_only: true,                    // optional
    check_ids:  ["<check_id>", ...]       // optional — scope failure filter to specific checks
  )
  ```

  When `failed_only=true` and `check_ids` is omitted, the tool automatically fetches the dashboard's full check set and filters to failures across all of them — so a single call covers the dashboard, no per-check loop needed.

### Runbook: "Investigating recent failures"

When the user asks "what's failing in production?" / "monitor my recent failures" / "any regressions" / "how's my agent doing":

1. `relevance_list_performance_dashboards(resource_type, resource_id)` — if none exist, propose once to go through a few real runs together and turn them into checks; a dashboard follows from those. Otherwise pick the enabled dashboard(s). (If you already fetched the agent/workforce, its `quality` block — `has_performance_dashboard`, `performance_dashboard_active` — already tells you whether monitoring exists, so you can decide whether to offer setup without this call; you still need the list to pick _which_ enabled dashboard to pull runs from.)
2. For each dashboard, list recent failed runs in **one call**:
   ```
   relevance_list_performance_dashboard_runs(
     resource_type, resource_id,
     config_id: "<dashboard_id>",
     start_date: "<7 days ago, ISO 8601>",
     end_date:   "<now, ISO 8601>",
     failed_only: true,
     page_size: 20
   )
   ```
3. Group failures by check name (eval-run results expose this as `rule_name` per check — backend storage term). For each, surface 1-2 representative failed runs with the LLM judge's `reason` field.
4. Summarise crisply: sample window, the dominant failing check(s), and the dominant failure reason.
5. Suggest the next step from where the failure comes from. Compare a failed run with the last run that handled the same kind of input and went well (a run the dashboard didn't score still counts):
   - The agent decided differently on the same input → its instructions, tools or knowledge.
   - Same decisions, a different result from a tool → that tool, or the system it connects to; fix it there.
   - The output was right and the check failed it → the check (see [authoring.md](authoring.md), "Authoring checks").

   When the fix is in the agent or a tool, also turn the failure into a regression test by adding a scenario (`relevance_create_eval_test_case`) that exercises the same case.

### Recommend a periodic check-up (only when unmonitored)

When a dashboard has an alarm configured, degradation is reported automatically — no scheduled check-up is needed; don't offer one. Recommend a periodic re-check only for a resource with no monitoring/alarms yet, as part of that one proposal: if your harness can schedule a future check (a wake-up message or reminder), offer to schedule it and run this runbook then; if it can't, tell the user when to come back and what to look at. The scheduling and notification mechanics themselves are harness-specific — this skill only recommends the check-up and defines what to review.

### Alarms — get told instead of remembering to look

Alarms notify email, Slack, or Microsoft Teams when one dashboard check or its overall score degrades. They attach to agents or workforces, never individual tools. Offer one after setting up a dashboard, or when a fetched agent/workforce has a dashboard but `quality.alarms.evals` counts no `enabled` or `paused` alarm; never re-offer one that exists, and offer to resume one when all are paused. Use the Relevance app UI if alarm tools are unavailable.

Create, change and remove them with `relevance_create_alarm`, `relevance_update_alarm` and `relevance_delete_alarm`, passing `type: "evals"` with the resource and (to change one) its `alarm_id`. The type is required and is never inferred. Read alarms with `relevance_list_alarms` and `relevance_get_alarm`, passing the resource and (for one alarm) its `alarm_id`. These reads include visible eval, trigger, task-error, pending-approval and escalation alarms; use `config.source_config` to identify the kind. A fetched agent/workforce's `quality.alarms` already counts each kind you can see (`evals`, `triggers`, `task_errors`, `task_approvals`, `task_escalations`) as `enabled` and `paused`, plus how many enabled ones `auto_diagnose`, so list only when you need a specific alarm. Like the list, it counts only alarms visible to you.

Before creating, list existing alarms and ask which channels to use. Never invent recipients or connection IDs. One channel entry per type; one email entry contains all recipients. A caller-scoped empty list does not prove no teammate created an alarm.

- Thresholds use fractions: `0.8` means 80%.
- Results are bucketed; `2 of 3` breaching buckets is less noisy than `1 of 1`.
- Low-traffic resources need a meaningful minimum sample size.
- Use at least a 60-minute cooldown unless the user requests faster repeats.

An alarm is inert when its dashboard is not scoring live traffic or its rule does not belong to that dashboard. State is `ok`, `alarm`, or `insufficient_data`; the latter usually means too little traffic or too high a sample minimum. Only current state and latest timestamps are retained, not firing history.

### Agent notification rules (a separate thing from alarms)

An alarm reports a rate or a score. To tell someone about each task as it happens, when it waits for approval, is escalated, fails or finishes, use the agent's notification rules (`notifications_config`): see `managing-relevance-agents/notification-rules`.

### When dashboards return no data

- `enabled: false` on the dashboard — re-enable.
- `sample_rate: 0` — bump up; `0` disables evaluation entirely.
- No live traffic on the agent or workforce in the time range — verify it's actually being invoked in production.
- Date range too narrow / too old — widen it.
- An agent dashboard whose status filter (`filters.task_status`, or the older `filters.conversation_status`) has `completed` but not `idle` — agent runs end `idle`, so it only scores conversations someone marked complete. Offer to add `idle` or clear the filter (`task_status: []`).

### "A performance dashboard must have at least one check"

`relevance_remove_performance_dashboard_rule` refuses to drain the last check. Either add a replacement first via `relevance_add_performance_dashboard_rule`, or delete the dashboard entirely via `relevance_delete_performance_dashboard`.
