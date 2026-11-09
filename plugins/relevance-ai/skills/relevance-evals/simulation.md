---
name: relevance-evals/simulation
description: Tool simulation in eval scenarios — what to simulate, the config shape, levels, clearing, keeping coverage.
---

# Tool Simulation

## Tool Simulation Config

Tool simulation lets you override or simulate tool outputs during evaluation runs, so you can test agent/workforce logic without calling real external services. Applies to **pre-deploy scenario runs only** — performance dashboards score real production calls.

> **🛑 Default: simulate every tool that has a real-world side effect.** When you set up evals for an agent/workforce, **simulate any tool that does something real** — webhooks, email / Slack / SMS sends, CRM or database writes, payments, file uploads, anything irreversible — **unless the user explicitly says to run it live.** Set `tool_simulation_config` on the **test set** (so it covers every scenario) before running. If you don't, the eval triggers the agent for real and **fires those side effects on every run** (real webhooks sent, real records written).
>
> **How to decide (default to simulating):** look at the tool's name, description, and steps. If it **connects to an external service** (any API, webhook, integration, OAuth/API-key tool, 3rd-party app) **or modifies data** (creates / updates / deletes / sends / writes anything — to an external system or to Relevance knowledge/tables), **simulate it.** A clearly **read-only / pure-compute** tool — search, lookups, fetching public info, calculations, formatting — is the one case where simulating is optional rather than required, but simulated is still the default. When in doubt, simulate.
>
> **There is exactly one reason to switch a tool to live: the user asked for it.** Not "this tool is only arithmetic", not "the real output would be more realistic", not "the other scenarios are live" — if the user did not ask, leave it simulated. If a scenario was created with tools already simulated, **do not** call `relevance_set_eval_test_case_simulation_config` to undo that.
>
> **A simulated tool needs no connected account.** While it is simulated, its account and knowledge-table inputs stop being required, so don't connect, pick or fill in an account to make an eval run.
>
> This does **not** weaken your checks. A simulated tool call is still recorded, so a `tool_usage` check (e.g. "webhook fired ≥1×" / "fired exactly 0×") counts simulated calls exactly like live ones. Content checks (`string_contains`, LLM judge) work too: the simulated output is generated from the tool's own output schema, so it normally carries a realistic value. If a check asserts on one **specific** computed value, pin that value with a `simulation_prompt` — don't take the tool live to get it.

> Key every tool — and sub-agent — by its **`action_id`**, never its `chain_id`: see [Tool ids](SKILL.md#tool-ids).

### What to Simulate (and What NOT to)

**Simulate these tools:**

- API integrations (search, weather, CRM lookup, company enrichment)
- External services (email sending, Slack posting, file uploads)
- Database operations (knowledge table reads/writes)
- Third-party tools (Clearbit, SendGrid, Salesforce)

**Writing a `simulation_prompt` — usually you don't need one.** The simulator is already given the tool's name, description, input schema, output schema, and the actual inputs of the call, so it normally returns something realistic on its own. Write a prompt in two cases:

- **The tool has no real output schema.** The simulator is then told the schema is unknown and has to invent field names as well as values, so the shape can drift between runs.
- **A check asserts a specific value** that the schema and inputs can't imply (a status, an id, a named record). A derivable value — `5 + 5` → `10` — needs no prompt.

To ground it, read the tool with `relevance_get_tool` to see its output schema, and if it has run before, `relevance_get_latest_tool_run` for a real output to copy the shape from. Then describe both the shape and the value the check needs.

**Don't create tools just to simulate them.** If you're tempted to create a tool whose only step is `prompt_completion` (an LLM call) so you can simulate its output in evals, that reasoning should live in the agent's system prompt instead.

**Simulation replaces the whole tool, its LLM steps included** — there is no simulating only the API part. A judgement made inside a tool (filtering, scoring, summarising) is invented by the simulator on every run and never tested, which is another reason it belongs in the agent's system prompt. When a tool legitimately extracts with an LLM step (see [Tools vs Agents](../managing-relevance-tools/SKILL.md#tools-vs-agents-what-goes-where)), pin the values a check depends on with a `simulation_prompt`.

### Config Levels (Precedence)

Tool simulation config can be set at three levels:

| Level              | Where it's set                                   | Applies to                                                                               | Overridden by                           |
| ------------------ | ------------------------------------------------ | ---------------------------------------------------------------------------------------- | --------------------------------------- |
| **Test set level** | `tool_simulation_config` on the test set         | Every run of the scenarios in the test set, however the run is started                   | Batch- and scenario-level config        |
| **Scenario level** | `tool_simulation_config` on the test case        | That specific scenario only                                                              | Nothing (wins **per tool** — see below) |
| **Batch level**    | `tool_simulation_config` on the evaluate request | Ad-hoc runs with `scenario_ids` only (**returns 400** if `test_set_id` is also provided) | Scenario-level config                   |

These three levels decide **where** a config is attached. Orthogonally, **within** a workforce config, a per-node entry (`node_configs[node_id]`) takes precedence over a legacy per-agent entry (`agent_configs[agent_id]`) **per tool**: tools the node entry doesn't name keep the agent entry's setting.

**Levels merge per tool, not wholesale.** A scenario-level config wins only for the tools it names; every other tool still inherits the batch and test-set configs. A scenario in several of the resource's test sets inherits from each. Where they disagree on a tool, a set that mocks or simulates it beats one that pins it live; otherwise the resource's default set is lowest and the newest set wins.

#### Clearing an override

Pass `null` as the `tool_simulation_config` to `relevance_set_eval_test_case_simulation_config` (test case) or `relevance_update_eval_test_set` (test set) to remove that level's config; the level then inherits from above. `null` is the only clear value — omitting the field keeps the stored config, and `{}` is rejected.

#### Keeping evals covered when the tool surface changes

A scenario's config is a snapshot; the agent or workforce it tests is not. **A tool attached to an agent — or an agent or tool node added to a workforce — after the scenarios were written has no entry, so it runs live.** Nothing backfills it.

So after changing the tool surface, check whether the resource has evals (`relevance_list_eval_test_sets`) and, if it does, cover the new tool before publishing.

**Do this once at the test-set level, not once per scenario.** Levels merge per tool and a scenario config only wins for the tools it names — a brand-new tool is named by no scenario, so a single `relevance_update_eval_test_set` covers every scenario in that set. Read the test set first and merge your entry in: that update replaces the config wholesale, so sending only the new tool would drop the rest.

You already have the ids you need: `relevance_attach_tools_to_agent` returns an `action_id_map`, and for a new workforce agent node call `relevance_get_agent_tools` with that node's id and add the entry under `node_configs[node_id]`. A new tool node that runs as a next step (a `forced-handover` edge leads to it) gets an entry under its own `node_id`, keyed by its id in `tool_node_action_ids` from `relevance_get_workforce`; one an agent calls over a `tool-call` edge goes under that agent's node instead (see [workforces.md](workforces.md#tool-nodes)).

**Clearing is not a safe "reset", and it is not a way to take a tool live.** Because levels merge per tool, clearing a test case's config falls back to the test set's config — but only for the tools that config actually names. A test set created without a `tool_simulation_config` simulates nothing, so clearing a scenario there puts **every** tool live. If a scenario was created with its tools already simulated, clearing it undoes that.

To pin one tool live while the test set simulates it, give it an explicit entry with `output_overrides_enabled: false` (not `simulate_output: { is_simulated: false }`, which does not disable simulation):

```typescript
{ tool_configs: { '<tool_action_id>': { overrides: { 'default': { output_overrides_enabled: false } } } } }
```

### For Agents

```typescript
{
  tool_configs: {
    '<tool_action_id>': {
      overrides: {
        'default': {
          output_overrides_enabled: true,
          output_overrides: {
            simulate_output: {
              is_simulated: true,
              simulation_prompt: 'Describe what the tool should return...',
              model: 'relevance-cost-optimized',
            },
          },
        },
      },
    },
  },
}
```

For a **workforce**, simulation is scoped per graph node with `node_configs`, including each tool node that runs as a next step — see [workforces.md](workforces.md#tool-simulation-per-node).

## Simulating one run

To try an agent or workforce once without saving a scenario, trigger it (`relevance_trigger_agent` or `relevance_trigger_workforce`) with `simulate: true`. Every tool then returns a stand-in except those named in `live_action_ids`: action_ids from `relevance_get_agent_tools`, or `tool_node_action_ids` from `relevance_get_workforce`. Name only tools that read.

- If any tool can't be simulated, the trigger refuses and names it.
- An agent's sub-agent can't be named live; its reply is simulated. In a workforce, handovers still happen and each node they reach is simulated.
- It saves no scenario or `tool_simulation_config`, so it covers no later run or publish.
- In the run's results, a tool call that didn't run carries `output_source` (`simulated-tool-output`, `debug-mode-output-override` or `action-confirm-mock-tool-output`). A finished call without it, or with `tool-run` in raw task messages, ran for real.
