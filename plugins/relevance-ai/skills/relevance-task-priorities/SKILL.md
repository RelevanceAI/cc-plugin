---
name: relevance-task-priorities
description: Read, set, raise, lower, or clear the task-priority tier (run priority) of an agent, tool, or workforce so important work is scheduled ahead of lower-value work under the project's concurrency limits. Use when the user asks what priority their resources run at — "what priority are my workforces?", "which of my agents run first?" — or when a resource is starved or queued rather than failing — "why is this agent so slow to start?", "make this agent run first", "deprioritise this tool".
---

# Relevance AI Task Priorities

When a resource isn't _failing_ but is _starved_ — an important agent/tool/workforce that queues behind lower-value work under the project's concurrency limits — task-priority tiers are the fix. Each agent, tool, or workforce carries a priority tier; the scheduler dispatches higher-priority tasks ahead of lower ones when capacity is contended. A resource with no tier set runs at the **medium** default.

The tiers are `high`, `medium`, and `low`. The Relevance app calls them **Critical**, **Standard** and **Background**, and labels the setting "Run priority". Use the app's names when talking to the user.

These are **project-admin** operations; a caller without admin permission is denied.

| Tool                                      | Role                                                                                            |
| ----------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `relevance_list_resource_task_priorities` | Read the tiers currently set in the project (resources absent from the list are medium)         |
| `relevance_set_resource_task_priority`    | Set/raise/lower a tier: `{ resource_type: agent\|tool\|workforce, resource_id, priority_tier }` |
| `relevance_unset_resource_task_priority`  | Remove the override, reverting the resource to the medium default                               |

## Reading priorities

The tier is not part of an agent's or workforce's config, so `relevance_get_agent` and `relevance_get_workforce` never show it. Read it with `relevance_list_resource_task_priorities`: one call returns every tier set in the project.

For an overview, join that list with `relevance_list_workforces` and `relevance_list_agents` (and `relevance_list_tools` if asked) to get names, and report every resource the user asked about, not only the ones with a row. A resource with no row is Standard.

Which tier a task actually runs at:

- A workforce's tier applies to every task in that workforce and overrides its agents' and tools' own tiers.
- An agent's or tool's own tier applies when it runs on its own, and normally inside a workforce that has no tier.

A tier only matters when the organisation is at its concurrency limit, and then only against other waiting work in the same project; below the limit everything starts straight away. It gives a head start in the queue, not a strict order. Tiers only take effect in organisations where Relevance has turned priority scheduling on, so don't promise the user a tier will change when their tasks start.

If the list call returns a 403, the user is not a project admin. Say so; don't guess the tiers.

## Recommending a change

Diagnose before you set anything: a slow start is a queueing symptom, not an error — if the resource is erroring, that's a different problem (see `relevance-diagnostics`).

Recommend a tier change like any other fix — name the tool and the target — and only apply it when the user asks. Raising one resource to `high` is only meaningful relative to its neighbours: if everything is `high`, nothing is prioritised. When raising one resource, say which work it will now queue behind it.

If the set and unset tools aren't available to you, a project admin can change the tier in the Relevance app: "Run priority" in an agent's or tool's settings, or "Set run priority" on a workforce.
