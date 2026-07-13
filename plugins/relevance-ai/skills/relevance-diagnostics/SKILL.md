---
name: relevance-diagnostics
description: Diagnose what is wrong in a Relevance AI project and recommend fixes. Use when the user asks to "analyse my project", "diagnose this agent/workforce", "why is X failing?", "what's wrong / what should I fix?", "what's failing?", or "any regressions?". Finds root causes and recommends fixes by consuming the analytics, eval, and error read tools.
---

# Relevance AI Diagnostics

Find what is broken in a project — and why — then recommend the fix. Run the macro→micro routine below over the analytics, error, and eval read tools.

## Overview

Orchestrate the read-only tools you already have into this routine:

1. **Macro** — a cheap, parallel project snapshot (counts, trends, breakdown).
2. **Rank** — sort resources by error rate / failure volume to find the worst offenders.
3. **Micro** — deep-dive the flagged subset in parallel (scoped errors, activity, task transcripts, performance dashboards).
4. **Findings** — a ranked, human-facing report: what's wrong, the likely cause, and the recommended fix.

> **Recommend fixes; do not apply them.** Every finding ends in a _recommended_ action and names the mutating tool that would carry it out. Assemble the macro snapshot from the parallel calls below.

Diagnostics **consumes** the analytics, eval, and error read tools rather than redirecting to them — pull in `relevance-evals` (eval/dashboard results) and `relevance-analytics` (raw numbers) on demand. For a diagnosis ask, don't bounce the user to those skills; consume them.

## Concepts

- **Macro vs micro.** Macro is the cheap, project-wide snapshot that always runs first; micro is the gated, per-resource deep-dive (see [Guardrails](#guardrails)).
- **Resource.** An agent, a workforce, or a tool. `relevance_get_analytics_resource_breakdown` covers all three via `resource_type`. Note: the eval tools' `resource_type` (`relevance_list_performance_dashboards`, `relevance_list_performance_dashboard_runs`) accepts only `agent | workforce` — there are no tool-scoped dashboards.
- **Severity.** Each finding is ranked **High / Medium / Low** by impact (error rate × volume of still-occurring errors, escalations, blocked users).
- **Two entry points, one skill.** _Project-wide_ ("analyse my project") runs the full ladder. _Direct single-resource_ ("diagnose this agent") skips the ladder and goes straight to micro.
- **Recency & versions.** Error data describes the config that was live _when each error happened_, so a finding may already be fixed. Before reporting one, ask: did this error happen _again_ after the resource last changed, given something has run since? That recurrence check is what decides whether it's still open — see [Step 3b](#step-3b--cross-check-recent-edits--versions).
- **Window.** Default to the **last 7 days** unless the user names a window. Date-format quirk: `get_analytics_resource_breakdown` / `get_analytics_usage_timeseries` / `get_analytics_agent_activity_timeseries` / `get_analytics_usage_breakdown` take `YYYY-MM-DD`; `get_analytics_errors` / `get_analytics_task_status_counts` take full ISO 8601 datetimes (e.g. `2026-03-01T00:00:00.000Z`).
- **Access-limited analytics.** If an analytics tool returns a permission / forbidden / entitlement / access-denied error, mark that diagnostic signal **unavailable**, continue with the tools that did return, and surface it under **Blind spots**: do **not** retry repeatedly, do **not** treat unavailable analytics as zero issues, and state which checks could not be performed.

## Workflow 1 — Project-wide diagnosis ("analyse my project")

```
        ┌──────────────────────────────────────────────┐
        │ 0. SCOPE     size the project; if large,       │
        │    confirm a narrowing first                   │
        └───────────────────────┬──────────────────────┘
                                ▼
        ┌──────────────────────────────────────────────┐
        │ 1. MACRO     parallel, cheap, ALWAYS first     │
        └───────────────────────┬──────────────────────┘
                                ▼
        ┌──────────────────────────────────────────────┐
        │ 2. RANK      shortlist by error volume         │
        └───────────────────────┬──────────────────────┘
                                ▼
        ┌──────────────────────────────────────────────┐
        │ 3. MICRO     deep-dive, ≤10 resources/run      │
        └───────────────────────┬──────────────────────┘
                                ▼
        ┌──────────────────────────────────────────────┐
        │ 3b. RECENCY  did it recur after the change?    │
        │    diff only breaks no-traffic ties            │
        └───────────────────────┬──────────────────────┘
                                ▼
        ┌──────────────────────────────────────────────┐
        │ 4. FINDINGS  ranked report + recommended fixes │
        └──────────────────────────────────────────────┘
```

### Step 0 — Scope

```
relevance_get_project_info()   → project name / region / app URLs (report header + links; NOT counts)
relevance_list_folders()       → folder structure, for narrowing the set
relevance_list_agents()        → agent inventory (and relevance_list_workforces) — to size the project
```

`get_project_info` does **not** return counts — to size the project, tally `list_agents` / `list_workforces` and read `list_folders` for structure (there is no single project-count call today). If the project is large, confirm a narrowing with the user before the micro fan-out — e.g. "a specific folder", "the highest error-rate agents", or "a shorter window". A cheap macro snapshot is always fine to run first regardless.

### Step 1 — Macro snapshot (run in parallel)

Fire these **in a single message** if your harness supports parallel tool calls:

```
relevance_get_analytics_task_status_counts({ start_date, end_date })        → total / to_review / escalated / errored
relevance_get_analytics_usage_timeseries({ metric: "total_tasks", start_date, end_date })       → trend
relevance_get_analytics_resource_breakdown({ resource_type: "agent", start_date, end_date })      → per-agent scorecard
```

### Step 2 — Rank the worst offenders

From the breakdown, **shortlist** the worst offenders by `error_count` (the raw failure count on each row), cross-checking `error_rate`. A 100% `error_rate` on 1 task is noise; a moderate rate on high volume is real. This is only the shortlist for the micro deep-dive — final severity is decided later by [Step 3b](#step-3b--cross-check-recent-edits--versions), which ranks by errors that are still happening, not by raw window totals. Pull recent error messages to see _what_ is failing:

```
relevance_get_analytics_resource_breakdown({ resource_type: "workforce", start_date, end_date })   → workforce scorecard
relevance_get_analytics_resource_breakdown({ resource_type: "tool",      start_date, end_date })   → which tools error most
relevance_get_analytics_errors({ start_date, end_date, page_size: 50 })   → all origins (omit resource_type)
```

### Step 3 — Micro deep-dive (parallel, capped)

For each flagged resource (**≤10 per run** — see [Guardrails](#guardrails)), fan out in parallel. The block below is the **agent** branch:

```
relevance_get_analytics_errors({ agent_ids: ["<id>"], start_date, end_date, resource_type: "agent" })
relevance_get_analytics_agent_activity_timeseries({ agent_ids: ["<id>"], start_date, end_date })
relevance_list_performance_dashboards({ resource_type: "agent", resource_id: "<id>" })
```

For a flagged **workforce**, swap in `workforce_ids: ["<id>"]` on the errors call and `resource_type: "workforce"` on the dashboards call. For a flagged **tool**, there is no scoped errors call (`get_analytics_errors` has no tool-id filter): fetch tool-origin errors with `resource_type: "tool"` and keep only the rows whose `source_id` is the flagged tool, and pull its stats with `relevance_get_analytics_resource_breakdown({ resource_type: "tool", resource_ids: ["<id>"] })`. Performance dashboards don't apply to tools.

**Drill a failing task straight from the error record — don't scan.** Each `get_analytics_errors` row carries a `source_context`; use it to jump directly to the transcript that failed:

- **Agent errors** → `source_context.{ agent_id, conversation_id }` → `relevance_list_agent_task_messages({ agent_id, task_id: conversation_id })` — the agent task id _is_ the conversation id.
- **Workforce errors** → `source_context.{ workforce_id, workforce_task_id }` → `relevance_get_workforce_task_messages({ workforce_id, task_id: workforce_task_id })`. Recognise these by `source_context.workforce_id` — a row's `source_type` is only ever `agent` or `tool`, never `workforce`.
- **Tool errors** have no direct transcript: `get_analytics_errors` gives the failing tool's id (`source_id`) and the error message, but there's no per-tool task-message tool. Drill via the _invoking agent's_ conversation when `source_context.conversation_id` is present; otherwise stay at the error-message + breakdown level.

When a resource has no error rows to start from, list its failing tasks directly with `relevance_list_tasks({ view: "errored", agent_id })` — the Task Ops list, filterable by view/state/date, so you land on errored tasks instead of scanning recents by hand. Prefer the error-context drill above when error rows exist; the errors carry the categorised message, the task list only carries state + `active_failures`.

Classify each error by who can fix it — platform (transient), user (auth/config/billing), or the agent's own logic — and let that drive the recommended fix. When the root cause is a missing/wrong integration (auth_config), surface the integration's connect link if your harness can fetch one, rather than describing the Integrations UI in prose.

### Step 3b — Cross-check recent edits & versions

**Before a candidate becomes a ranked finding, decide whether it's _still open_, and rank it by what's still happening — not by the raw window total.** Error data is historical: an error from last Tuesday describes the config that was live last Tuesday, which may have been fixed since.

**The rule** is one question: did this error happen _again_ after the resource last changed, given something has run since the change?

- **Yes, it recurred** → still open. Rank it by how many times it recurred after the change.
- **No, but there was traffic** → likely fixed; something ran cleanly since the change.
- **Nothing ran since the change** → recurrence can't tell you. Fall back to the config: if the change plausibly addresses the root cause, call it "likely resolved — verify"; otherwise "untested since the change — verify".

Recurrence is what decides this. The config only matters in that last case, where there's no traffic to judge by.

**Route the check to the resource that actually errored — this is the most common mistake.** A **tool-sourced** error (a `get_analytics_errors` row whose source is a tool) is about the **tool**: check the **tool's** timeline and the calling agent's action config, _not_ the agent's system prompt. An **agent** error → the agent's versions; a **workforce** error → the workforce's versions.

**1 — Find when that resource last changed:**

```
agent error     → relevance_get_agent({ agent_id })         + relevance_list_agent_versions({ agent_id })
tool error      → relevance_get_tool({ studio_id })         + relevance_list_tool_versions({ studio_id })
workforce error → relevance_get_workforce({ workforce_id }) + relevance_list_workforce_versions({ workforce_id })
```

The change date = the active version's publish time (or `update_date_`) of that resource. Note `has_unpublished_draft` too.

**2 — Measure recurrence (precise) and traffic (volume) — two different signals.**

- **Recurrence — count the `error_time`s you already have.** Each Step 3 error row for _this error class_ carries a full-timestamp `error_time`. Count how many landed **at or after the change** vs before. This is the signal that decides recurrence: it's specific to this error class and uses full timestamps, so it's correct even when the resource was published the **same day**. No extra call.
- **Traffic — one scoped breakdown call for volume.** "No recurrence" only means "fixed" if something actually _ran_ since the change. Re-scope the breakdown for the **erroring resource** to the window since the change and read its **volume** only:

```
agent error     → relevance_get_analytics_resource_breakdown({ resource_type: "agent",     resource_ids: ["<id>"], start_date: "<change-date>", end_date: "<today>" })   → task_count
tool error      → relevance_get_analytics_resource_breakdown({ resource_type: "tool",      resource_ids: ["<id>"], start_date: "<change-date>", end_date: "<today>" })   → execution_count
workforce error → relevance_get_analytics_resource_breakdown({ resource_type: "workforce", resource_ids: ["<id>"], start_date: "<change-date>", end_date: "<today>" })   → workforce_tasks
```

`get_analytics_resource_breakdown` takes `YYYY-MM-DD`, so use the change **date** — coarse is fine for "was there _any_ traffic." Don't use this call's aggregate `error_count` for recurrence: it counts every error class and its day-granularity miscounts a same-day change. The `error_time` count above is the one to trust.

**3 — Classify:**

| After the change…                                          | Verdict                                                                                                                                                            | Where it goes                         |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------- |
| **Traffic since the change _and_ the error recurred**      | **Confirmed open** — rank by the post-change count                                                                                                                 | Findings (ranked)                     |
| **Traffic since the change _and_ the error did NOT recur** | **Likely resolved** — a clean run since the change is strong evidence                                                                                              | Likely already resolved               |
| **No traffic since the change**                            | recurrence can't decide; fall back to the config (the rule above) — plausibly fixed → "likely resolved — verify"; otherwise → "untested since the change — verify" | Likely already resolved / Blind spots |

**A config diff can never re-open a no-recurrence-with-traffic issue.** The real fix may live in a field, a resource, or a version you didn't look at, or be a model / provider / dependency change that doesn't show in the config at all. The diff only earns its keep when there's no traffic to judge recurrence by.

**If `has_unpublished_draft` is true** → the user may be mid-fix. Inspect the draft (`relevance_get_agent({ version: "draft" })`, etc.) and whether it touches the root cause. If it does, recommend **testing/publishing that draft** rather than authoring a new fix — and note the live errors persist until it's published.

> Optional, when the diff is legible: snapshot error-time vs current config (`relevance_get_agent_version` / `relevance_get_tool_version`) to _explain_ an open finding or strengthen a "likely resolved" — never to keep a non-recurring issue open. Workforces have no per-version snapshot tool, so corroborate from the current `relevance_get_workforce` config plus recurrence.

### Step 4 — Emit findings

Synthesise into the [findings format](#findings-output-format) below.

## Workflow 2 — Direct single-resource diagnosis ("diagnose this agent")

When the user names one agent/workforce (or one is in the target-resource context), **skip the scope ladder and the project-wide macro** — run the Step 3 micro tools scoped to that one id, plus a single scoped breakdown in place of the macro:

```
        diagnose <resource>
                │
                ▼
        MICRO on that resource only  (Step 3 tools, scoped to the one id)
                │
                ▼
        FINDINGS  (same format, scope = the single resource)
```

```
relevance_get_analytics_resource_breakdown({ resource_type: "agent", resource_ids: ["<id>"], start_date, end_date })
relevance_get_analytics_errors({ agent_ids: ["<id>"], start_date, end_date })
relevance_get_analytics_agent_activity_timeseries({ agent_ids: ["<id>"], start_date, end_date })
relevance_list_performance_dashboards({ resource_type: "agent", resource_id: "<id>" })
```

For a workforce, swap in `relevance_get_analytics_resource_breakdown({ resource_type: "workforce", resource_ids: [...] })`, `relevance_list_workforce_tasks`, and `relevance_get_workforce_task_messages`.

**Health check with no reported symptom** (e.g. a periodic "is this agent still OK?" review, not a specific complaint): work the signals in order of value — **performance dashboards / evals first** (if any exist, read the sampled dashboard runs and recent eval batches for pass rates and failing checks) → **recent errors** → and only if there's nothing else to go on, **sample a couple of recent tasks/conversations**. A clean task list alone is not a health check — always check dashboards/evals when they exist before concluding "healthy".

Run the [Step 3b recency cross-check](#step-3b--cross-check-recent-edits--versions) here too — a single-resource diagnosis is exactly where the user has probably just edited the thing they're asking about, so an already-fixed error is the most likely false positive.

## Findings output format

Emit a human-facing Markdown report. Severity is **High / Medium / Low**. Name the mutating tool that would apply each fix, but don't apply it.

```
## Diagnosis — <scope> · <window>
**Summary:** <total tasks, overall error rate, headline concern>

### Findings (ranked)
1. **[High] <title>** — <resource type + name/id>
   - Evidence: <**post-change** error_count (of total in window), error_rate, error category; eval pass-rate when available>
   - Likely cause: <root cause>
   - Recency: <change date + verdict — e.g. "9 of 12 errors landed after the Mar 3 change; traffic since: yes → confirmed open">
   - Recommended fix: <concrete action + the mutating tool that would apply it>
2. **[Medium] …**

### Likely already resolved
- <issue> — no recurrence after the <date> change despite traffic since (a clean run was observed); or, with no traffic since, the changed config addresses the root cause. Confirm with a re-run before closing.

### Blind spots
- <active agents/workforces with no performance dashboard configured>
- <resources changed in-window with no runs since — untested, verify with a run>

### Next steps
- <offer to drill deeper>
- <for a production agent, or after the user acts on a fix — recommend a periodic check-up so regressions surface early>
```

Only put **confirmed-open** issues (the error recurred after the last change, with traffic since) in **Findings (ranked)**, ordered by the **post-change** count; route non-recurring / draft-addressed issues to **Likely already resolved**, and changed-but-untested resources to **Blind spots**, per [Step 3b](#step-3b--cross-check-recent-edits--versions). Omit any section when it's empty.

When the window is clean (no errors, no escalations), still emit the report — say so explicitly under **Summary** and use **Blind spots** to flag missing observability (e.g. active agents with no performance dashboard) so the user knows what _wasn't_ visible.

## Recommend a periodic check-up

For a production agent — or right after the user acts on a fix — recommend re-checking on a cadence so regressions surface early rather than in front of end users. If a performance dashboard exists, point at it; if not, recommend setting one up (see `relevance-evals`). If your harness can schedule a future check (a wake-up message or reminder), offer to schedule the re-check; if it can't, tell the user when to come back and what to look at. The concrete scheduling and notification mechanics are harness-specific — this skill only recommends the check-up.

## Dedicated Tools

### Macro / ranking (analytics)

| Tool                                         | Role in the routine                                                                                                          |
| -------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `relevance_get_analytics_task_status_counts` | Macro health snapshot: total / to_review / escalated / errored                                                               |
| `relevance_get_analytics_usage_timeseries`   | Trend for one metric over the window                                                                                         |
| `relevance_get_analytics_resource_breakdown` | Per-resource scorecard with `error_rate` + `error_count` — `resource_type` = agent / workforce / tool; the ranking primitive |
| `relevance_get_analytics_errors`             | Recent errors (with `source_context` for transcript drill) + daily error counts; scope with `resource_type` / `agent_ids`    |
| `relevance_get_analytics_usage_breakdown`    | Who consumed the most (credits / executions) — cost context only                                                             |

### Micro / deep-dive (resource + eval)

| Tool                                                                                                   | Role in the routine                                                                                                     |
| ------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| `relevance_get_analytics_agent_activity_timeseries`                                                    | Per-agent state trend (running / idle / pending-approval)                                                               |
| `relevance_list_agent_task_messages`                                                                   | Transcript of a failing task — the root-cause source (drill path in [Step 3](#step-3--micro-deep-dive-parallel-capped)) |
| `relevance_list_tasks`                                                                                 | Task Ops list: tasks by view (`errored`/`approvals`/`escalated`/`all`), project-wide or per-agent, date-filterable      |
| `relevance_list_workforce_tasks` / `relevance_get_workforce_task_messages`                             | Workforce task list + transcript — the list _does_ support status and date-range filters (unlike the agent list)        |
| `relevance_list_performance_dashboards`                                                                | Whether monitoring exists (a gap = a blind spot)                                                                        |
| `relevance_list_performance_dashboard_runs`                                                            | Sampled production failures with the judge's reason                                                                     |
| `relevance_get_agent` / `relevance_get_tool` / `relevance_get_workforce`                               | Recency check: `has_unpublished_draft`, `active_version_id`, draft config (Step 3b)                                     |
| `relevance_list_agent_versions` / `relevance_list_tool_versions` / `relevance_list_workforce_versions` | Recency check: version history + timestamps → error-time vs current version (Step 3b)                                   |
| `relevance_get_agent_version` / `relevance_get_tool_version`                                           | Recency check: snapshot a version's full config to **diff** error-time vs current at the root cause (Step 3b)           |

## Guardrails

- **Macro first, always cheap.** The snapshot is a fixed handful of parallel calls — run it before anything expensive.
- **≤10 resources per micro run.** The deep-dive fan-out is the gated step. If more than 10 resources are flagged, diagnose the top 10 by severity and offer to continue.
- **Window cost.** Widen the window only when the user asks — longer windows multiply the data and the micro-step cost.
- **Folder / id scoping.** On large projects, scope the sweep to a folder or an explicit id set before fanning out.
- **Lean on parallel tool calls when available.** Emit the macro snapshot and the micro fan-out as parallel tool_use blocks in one message when your harness supports parallel tool calls. Not every harness does (and it may be off), so don't depend on it: the same calls still run serially, just slower — keep the fan-out within the ≤10 cap so a serial run stays bounded.

## Common Issues

### Macro snapshot is empty / all zeros

No traffic in the window. Confirm the window and that the agents/workforces are actually being invoked, then either widen the window or report "no activity in <window>" rather than inventing findings.

### High error rate on a tiny task count

A 100% `error_rate` on 1–2 tasks is noise, not a finding — rank by `error_count` and treat `error_rate` as secondary (see [Step 2](#step-2--rank-the-worst-offenders)).

### No performance dashboards exist

No dashboards = production isn't monitored. That's a **blind spot**, not a hard failure — flag it under Blind spots and offer to set one up (load `relevance-evals`).

### Flagging an error a recent edit already fixed

The most damaging false positive is recommending a fix the user already made. Run the [Step 3b recency cross-check](#step-3b--cross-check-recent-edits--versions): if the error **stopped recurring after the resource's last change and there's been traffic since**, it's resolved — even when you can't see _why_ in the config, since the fix may be in the tool, a different field, or a model/provider change.

Two traps to avoid. First, don't keep an issue open just because a config diff doesn't "show" the fix; recurrence-with-traffic outranks the diff. Second, the inverse: on a low-traffic project where **nothing has run since the change**, don't claim resolved either — mark it **untested — verify**.

## Related skills

Load these on demand (skill_key shown) when the flow reaches them — don't pre-load:

- **`relevance-analytics/SKILL`** — full per-tool docs for the `get_*` retrieval tools the macro/ranking steps call (input shapes, response shapes, date-format quirks).
- **`relevance-evals/SKILL`** — test sets / checks / performance dashboards; load it for the eval pass-rate signal, the "Investigating recent failures" runbook, and to set up monitoring for a diagnosed blind spot.
- **`relevance-task-ops/SKILL`** — the Task Ops (monitor page) queue surface: browsing tasks by view with `relevance_list_tasks` and drilling into a task's messages; load it when the user wants to see the queue itself rather than a diagnosis.
