---
name: relevance-llm-models
description: Which LLM model to configure on an agent, tool step, or workforce node — how to pick on task shape, what each vendor is structurally best at, the config traps that fail silently, and a dated list of current picks.
---

# Choosing an LLM model

**Never pick a model from your own training knowledge.** Model ids you recall from a vendor's API may not exist here, may be retired here, or may be two generations behind what the platform now offers. Everything you need is in this skill and in `relevance_list_llm_models`.

## When to use

Read this before setting any of:

- `model` or `fallbackModel` on an agent
- `params.model` or `params.fallback_model` on a tool step
- `llm_condition_model` on a workforce node
- the model on an eval check or a simulated tool output

For the concrete id to use right now, read [current-picks.md](current-picks.md).

## Pick on the property that fails

At the top of the market the published benchmarks no longer separate models — the leading scores sit inside each other's margins of error. Ranking by "which is smartest" produces a confident answer with nothing behind it. Rank on the property that will actually break the task:

| Ask                                                      | Because                                                                                                                                                                                                                                                  |
| -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| How many **sequential tool calls** will this make?       | Per-step accuracy is not task accuracy. Small per-step error compounds down a twenty-step run, so step _count_ — not the difficulty of any one step — is what should move you up a tier. Shortening the chain is often cheaper than upgrading the model. |
| **Would the next step catch a wrong answer?**            | Spend least where failure is loud and immediate (routing behind a validator, extraction the next step rejects). Spend most where failure is silent and expensive (money movement, customer commitments, unattended writes).                              |
| Is a **human watching it stream**?                       | Then the metric is time-to-first-token, not tokens per second. Reasoning happens entirely before the first token, so a model with high throughput and on-by-default thinking is a bad chat experience.                                                   |
| Does the input include **images, PDFs, audio or video**? | That is a hard capability gate. Check `supported_input_media` and stop — no amount of quality elsewhere substitutes.                                                                                                                                     |
| How much context, **really**?                            | A large `context_window` is a ceiling, not a promise. Recall degrades well before the limit, so retrieve and chunk rather than pouring a corpus in. Treat a big window as headroom for a long conversation.                                              |
| Is this **high volume**?                                 | Only then is price-per-token the right target. Otherwise optimise cost per _completed_ task: a cheap model that needs three attempts and an escalation costs more than a strong one that lands first time.                                               |

> **Reasoning effort moves cost and latency more than the model choice does.** Thinking tokens bill as output, and the effort ladder can swing time-to-first-token by an order of magnitude. Prefer a strong model at low effort over a weak model at high effort.

## Scale the model to the stakes

Default upward as a task gets harder or more consequential. A cheap model on a critical task is a false economy — one wrong answer costs more than the tokens it saved.

- **Complex or critical — step up**, even at several times the price. Money movement, customer-facing commitments, unattended writes, long unsupervised runs, anything no human will check before it lands.
- **Simple, high volume, or cheap to be wrong — step down.** Routing behind a validator, extraction the next step rejects, classification at scale.

Every row in [current-picks.md](current-picks.md) gives both directions from its default, so move along the row rather than inventing a model.

## What each vendor is structurally best at

These are architecture, not scoreboard — they stay true as versions change.

- **Google (Gemini)** — non-text input and volume. The broadest native multimodal support in one model, the lowest price floor, and very large context. Reach here first for vision, PDF, audio, video, screenshot-in-the-loop, and for high-volume steps where per-token price sets the architecture.
- **OpenAI (GPT)** — structured output and tier flexibility. The strictest JSON Schema honouring, the realtime/voice line, and the widest ladder of tiers within one family, so you can trade cost against capability without changing vendor or re-testing prompts.
- **Anthropic (Claude)** — long tool loops and code. Large context across the whole current line, and an adaptive-thinking effort ladder that lets one model serve both a cheap turn and a hard one.

> ⚠️ **This is a tiebreaker, not a ranking.** On general capability the frontier vendors are not meaningfully separated. Use the lane when the task leans on one of the structural properties above; never on the belief that one vendor is simply smarter.

## Prefer newer

Within a vendor family, a higher version number supersedes a lower one — a mid tier of the current generation usually beats the top tier of the previous one on both price and capability.

Two rules that follow:

1. If `relevance_list_llm_models` returns a family version — or a `release_date` — newer than anything [current-picks.md](current-picks.md) mentions, it is newer than this guidance. Prefer it, and tell the user the model guidance needs refreshing.
2. Vendors rename constantly and a tier name can be an alias that silently re-points. Do not infer a model's rank from a remembered tier name — take it from the listing.

Prefer the floating id over a dated snapshot. Pinned dates carry no extra information here and inherit no updates; pin only when a specific snapshot's behaviour is being relied on and someone owns re-validating it.

## Config traps that fail silently

Read these off the model's own config from `relevance_list_llm_models` — never off a config that worked on a different model.

- **Take valid effort values from `reasoning_capability`.** The ladders differ _within_ a vendor: some accept `none`, others start at `minimal`, others take a token budget instead of an effort at all. A value borrowed from a sibling model can be invalid.
- **Set effort explicitly — the defaults are not uniform.** Some families default to no reasoning at all (fast and cheap, wrong for a long tool loop); others default to high (slow and expensive, wrong for a chat turn). Inheriting the default is a decision you did not make.
- **Effort is written to the agent's top-level `thinking` field (`params.thinking` on a tool step), tagged with `_oneof_type_`.** The tag is `reasoning_capability.type`, except `openai_reasoning_effort` → `"openai"`, and `token_budget` → that capability's own `provider` (`"anthropic"` or `"google"`). The value field follows the tag: `reasoning_effort` for openai, `thinking_budget` for a token budget, `thinking_level` for `gemini_thinking_level`, `effort` for `claude_effort` and `claude_thinking`. A wrong shape is accepted without an error and ignored, so the model silently runs on its default.
- **CRITICAL: never disable thinking on a model that is in a tool loop.** On a `claude_thinking` model that means never setting `thinking_type` to `"disabled"`; elsewhere it means never picking the bottom of that model's ladder. With thinking off, a model can emit a tool call as plain text instead of a structured tool call. The turn returns successfully, the tool never runs, and the agent continues as though it did. If you need speed, lower the effort — do not switch reasoning off.
- **Set `fallbackModel` on agents that matter.** It switches the agent to a second model when the provider call fails; the stored effort does not carry across, so pick a fallback whose own default effort is safe for a tool loop. It does not catch a refusal the provider returns as a success.

## Never pick these

- **`openai-gpt-4o` and `openai-gpt-4o-mini`.** Superseded generations whose stored descriptions still recommend a model that is itself now two generations behind. This pair is the documented cause of previous bad model choices.
- **`openai-o1-latest`, `openai-o3-mini`, `openai-o4-mini`.** The standalone reasoning line has been folded into the main families' effort ladders. Use a current model at a higher effort instead.

If a resource you read back is on any of these, say so and offer to move it.

## When nothing here fits

Use `relevance-cost-optimized`, or `relevance-performance-optimized` for the hardest reasoning. These always resolve to a current model and never go stale. They are the right answer when the task has no particular shape — not a way to avoid reading this file.
