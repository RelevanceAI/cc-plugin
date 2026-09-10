---
name: relevance-task-priorities
description: Set, raise, lower, or clear the task-priority tier of an agent, tool, or workforce so important work is scheduled ahead of lower-value work under the project's concurrency limits. Use when a resource is starved or queued rather than failing — "why is this agent so slow to start?", "make this agent run first", "deprioritise this tool".
---

# Relevance AI Task Priorities

When a resource isn't _failing_ but is _starved_ — an important agent/tool/workforce that queues behind lower-value work under the project's concurrency limits — task-priority tiers are the fix. Each agent, tool, or workforce carries a priority tier; the scheduler dispatches higher-priority tasks ahead of lower ones when capacity is contended. A resource with no tier set runs at the **medium** default.

The tiers are `high`, `medium`, and `low`.

These are **project-admin** operations; a caller without admin permission is denied.

| Tool                                      | Role                                                                                            |
| ----------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `relevance_list_resource_task_priorities` | Read the tiers currently set in the project (resources absent from the list are medium)         |
| `relevance_set_resource_task_priority`    | Set/raise/lower a tier: `{ resource_type: agent\|tool\|workforce, resource_id, priority_tier }` |
| `relevance_unset_resource_task_priority`  | Remove the override, reverting the resource to the medium default                               |

## Recommending a change

Diagnose before you set anything: a slow start is a queueing symptom, not an error — if the resource is erroring, that's a different problem (see `relevance-diagnostics`).

Recommend a tier change like any other fix — name the tool and the target — and only apply it when the user asks. Raising one resource to `high` is only meaningful relative to its neighbours: if everything is `high`, nothing is prioritised. When raising one resource, say which work it will now queue behind it.
