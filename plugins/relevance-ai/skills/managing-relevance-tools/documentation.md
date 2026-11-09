---
title: Documenting Tools with Markdown Notes
description: Use the native `markdown` step type to embed documentation inside a tool's step list so users can understand multi-step tools at a glance. Load when creating or editing any tool with three or more steps, or any tool with non-obvious logic.
---

# Documenting Tools with Markdown Notes

Multi-step tools are hard to read. A user opens a tool someone (or something) else built and sees a column of step boxes with terse identifiers — no narrative. Markdown notes solve this: they are first-class steps that render rich text between real steps in the tool editor, anchoring the data flow with prose.

**Always document non-trivial tools.** A tool with three or more steps, or any tool whose logic is non-obvious from its step names alone, must include at least one markdown note. The bar is "could a teammate who has never seen this tool understand what it does and why, in 30 seconds, from looking at the step list alone?" If no, add notes.

## The `markdown` step

`markdown` is a native, inline, side-effect-free transformation. It takes a single param `markdown` (a string) and produces no output. It is inserted into `transformations.steps[]` like any other step — use `relevance_add_tool_step` with `position` to place it.

```typescript
{
  name: "overview",                 // any unique identifier
  transformation: "markdown",        // discriminator — do not change
  params: { markdown: "### What this tool does\n..." }
}
```

The `params.markdown` value renders as rich text in place in the Relevance app UI. A markdown step needs no `output` key — omit it (an empty `output: {}` is dropped with a warning) — and no downstream step ever references it.

## When to insert a note

A short rubric. Apply it deliberately — not every step needs a note, and over-noting is worse than under-noting.

| Add a note…                                                                                                                                        | Don't add a note…                                                                                                                           |
| -------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **Header note** as `steps[0]` for any tool with ≥3 steps or any tool whose `prompt_description` does not already capture what the tool does        | For 1- or 2-step tools where the step names already tell the whole story                                                                    |
| Before a **`prompt_completion`** step whose prompt encodes non-obvious logic (output schema constraints, multi-step reasoning, judgement criteria) | Before a step whose `name` already says what's happening (`fetch_user`, `send_slack_message`), or a plain "summarise / analyse this" prompt |
| Before a step with **`if:`** — explain the branch condition and what happens on each side                                                          | One per step out of completeness — that's noise, not signal                                                                                 |
| Before a step with **`foreach:`** or `loop` — explain what's being iterated and why                                                                | At the end of the tool as a "summary" — nobody reads trailing notes                                                                         |
| Before any step that **reshapes data** in a non-obvious way (code steps, parse-and-pivot, JSON munging)                                            | Restating the `transformation` ID ("This is a `serper_google_search` step that does a Google search")                                       |
| Before a step that **templates a previous step's output in a non-obvious way** (e.g. `{{steps.a.transformed.b[0].c}}` chained into a prompt)       | When the only thing left to say is "then we call the API"                                                                                   |

If you would not be embarrassed to ask a colleague "should I add a note here?", default to yes.

## What to write

A note has a job. Pick one and execute on it.

**Header note (`steps[0]`).** Three to six lines. State:

1. What the tool does, in plain language.
2. What it takes (referencing the actual `params_schema` keys with `**bold**`).
3. What it produces (the shape of the final output).
4. One sentence on strategy — why this tool chains the steps it does.

**Inline note (mid-tool).** One short paragraph. Lead with the **why**, not the **what**. Reference the variables the next step touches (`{{steps.fetch_user.output.email}}`) so the note is anchored in the data flow rather than floating above it. If you find yourself describing the step's transformation type, delete that sentence — the type is already visible.

## Style

- Use **H3** (`###`) for the header note's title. No top-level H1 — the tool title is already an H1 in the UI.
- Plain prose for inline notes. No headings inside an inline note unless it's covering multiple steps.
- **Bold** for variable names and parameter keys. Backticks for `transformation` IDs and code-shaped values.
- Keep notes terse. Notes are read alongside steps, not as a standalone document. Anything that takes more than ~80 words probably belongs in `description` or `prompt_description` on the tool itself.
- Notes about the tool's **intent** belong in `description` / `prompt_description`. Notes about the **step list** belong in markdown notes.

## Anti-patterns

- **One note per step.** Signal-to-noise problem. The reader skips them all by the third one.
- **Paraphrasing the step name.** "Step 1: search. This step performs a search." Delete.
- **Trailing "summary" note at the bottom.** Nobody reads them. The header note is the only orientation device that works.
- **Dates, author names, TODOs, or "added because of bug X".** That belongs in the tool's version history, not in the step list. It rots in place.
- **Changelogs.** If you are tempted to write "v2: now handles edge case Y," put it in the tool description and rely on `relevance_list_tool_versions` for the actual history.
- **A header note that's just the tool description copy-pasted.** The header note should say things the description doesn't — the _strategy_ of the chain, not the _purpose_ of the tool.

## Worked example

**Tool:** `contact-history` — takes a customer email, looks them up in HubSpot, and returns the contact with their last 10 messages, oldest first. It only gathers: the calling agent reads the history and drafts the follow-up from its own system prompt, where the owner can see and edit it.

### Before (no documentation)

```typescript
{
  studio_id: "contact-history",
  title: "Get Contact History",
  prompt_description: "Use to get a customer's HubSpot contact and their last 10 messages before writing to them",
  transformations: {
    steps: [
      { name: "find_contact", transformation: "hubspot_api_call", params: { oauth_account_id: "{{params.hubspot_account}}", method: "POST", path: "/crm/v3/objects/contacts/search", body: { filterGroups: [{ filters: [{ propertyName: "email", operator: "EQ", value: "{{params.email}}" }] }] } }, output: { contact: "{{response_body.results[0]}}" } },
      { name: "fetch_conversations", transformation: "hubspot_api_call", params: { oauth_account_id: "{{params.hubspot_account}}", method: "POST", path: "/crm/v3/objects/emails/search", body: { filterGroups: [{ filters: [{ propertyName: "associations.contact", operator: "EQ", value: "{{steps.find_contact.output.contact.id}}" }] }], properties: ["hs_email_from_email", "hs_timestamp", "hs_email_text"], limit: 100 } }, output: { conversations: "{{response_body.results}}" } },
      { name: "last_ten", transformation: "js_code_transformation", params: { code: "const all = steps.fetch_conversations.output.conversations || [];\nconst messages = all.map((m) => m.properties || {}).sort((a, b) => new Date(a.hs_timestamp) - new Date(b.hs_timestamp)).slice(-10).map((p) => ({ from: p.hs_email_from_email, sent_at: p.hs_timestamp, body: (p.hs_email_text || '').slice(0, 1000) }));\nreturn { messages, scanned: all.length, kept: messages.length };" } }
    ],
    output: { contact: "{{steps.find_contact.output.contact}}", messages: "{{steps.last_ten.output.transformed.messages}}", scanned: "{{steps.last_ten.output.transformed.scanned}}" }
  }
}
```

The reader has to chase variable names through the steps to work out what the tool returns, and why a code step trims the conversation.

### After (notes added)

```typescript
{
  studio_id: "contact-history",
  title: "Get Contact History",
  prompt_description: "Use to get a customer's HubSpot contact and their last 10 messages before writing to them",
  transformations: {
    steps: [
      {
        name: "overview",
        transformation: "markdown",
        params: {
          markdown: "### Gather a HubSpot contact's recent history\n\nTakes a customer **email** and returns their contact record and last 10 messages, oldest first, with how many messages it read. It only gathers: the calling agent reads the history and drafts the follow-up from its own instructions, where the owner can see and edit them."
        }
      },
      { name: "find_contact", transformation: "hubspot_api_call", params: { oauth_account_id: "{{params.hubspot_account}}", method: "POST", path: "/crm/v3/objects/contacts/search", body: { filterGroups: [{ filters: [{ propertyName: "email", operator: "EQ", value: "{{params.email}}" }] }] } }, output: { contact: "{{response_body.results[0]}}" } },
      { name: "fetch_conversations", transformation: "hubspot_api_call", params: { oauth_account_id: "{{params.hubspot_account}}", method: "POST", path: "/crm/v3/objects/emails/search", body: { filterGroups: [{ filters: [{ propertyName: "associations.contact", operator: "EQ", value: "{{steps.find_contact.output.contact.id}}" }] }], properties: ["hs_email_from_email", "hs_timestamp", "hs_email_text"], limit: 100 } }, output: { conversations: "{{response_body.results}}" } },
      {
        name: "why_last_ten",
        transformation: "markdown",
        params: {
          markdown: "HubSpot returns every message on the contact with full bodies. This step keeps the last 10 in **{{steps.fetch_conversations.output.conversations}}**, puts them oldest first and trims each body, so the agent reads a short thread and still sees how many there were."
        }
      },
      { name: "last_ten", transformation: "js_code_transformation", params: { code: "const all = steps.fetch_conversations.output.conversations || [];\nconst messages = all.map((m) => m.properties || {}).sort((a, b) => new Date(a.hs_timestamp) - new Date(b.hs_timestamp)).slice(-10).map((p) => ({ from: p.hs_email_from_email, sent_at: p.hs_timestamp, body: (p.hs_email_text || '').slice(0, 1000) }));\nreturn { messages, scanned: all.length, kept: messages.length };" } }
    ],
    output: { contact: "{{steps.find_contact.output.contact}}", messages: "{{steps.last_ten.output.transformed.messages}}", scanned: "{{steps.last_ten.output.transformed.scanned}}" }
  }
}
```

Two notes. The header orients the reader and says where the judgement lives; the inline note explains the non-obvious reshape. There is no note before `find_contact` or `fetch_conversations` — their names already say what they do. There is no trailing summary.

## Inserting notes via the tools

Notes are added the same way as any step. To add several, put them in one `relevance_update_tool_steps` call:

```typescript
relevance_update_tool_steps({
  studio_id: 'contact-history',
  ops: [
    // Add a header note as the first step
    {
      op: 'add',
      step: {
        name: 'overview',
        transformation: 'markdown',
        params: {
          markdown: "### Gather a HubSpot contact's recent history\n\n...",
        },
      },
      position: 0,
    },
    // Insert a note before the step now at index 3 (the header above shifted it)
    {
      op: 'add',
      step: {
        name: 'why_last_ten',
        transformation: 'markdown',
        params: { markdown: 'HubSpot returns every message on the contact...' },
      },
      position: 3,
    },
  ],
});
```

Always call `relevance_get_tool` first to confirm step indices before inserting a note mid-chain — indices shift as steps are added, and each op's index refers to the step array after the ops before it.

## When **not** to document

A handful of cases where notes are wrong:

- The tool is a thin wrapper around a single transformation. Use `description` and `prompt_description` on the tool itself.
- The tool was created by `relevance_create_tool_from_transformation` and is a one-step tool — let the underlying transformation's metadata speak for itself.
- The "documentation" you would add is really a TODO or a release note. Put that in the PR / commit / version history, not in the tool body.
