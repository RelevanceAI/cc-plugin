---
title: Phone Agent Best Practices
description: Make an agent a phone agent (runtime engine, voice, call transfer) and build production phone agents using the three-phase pre-call / phone / post-call pattern, with latency guidance and prompt structure. Load when building or designing a voice agent or optimising call latency.
---

# Phone Agent Best Practices

Reference for building production-grade phone agents on Relevance AI.

## Architecture: The Three-Phase Pattern

Phone agents work best as part of a workforce pipeline, not standalone:

```
Pre-Call Agent  →  Phone Agent  →  Post-Call Agent
(research)        (conversation)    (processing)
```

**Pre-Call Agent** — Front-load everything. CRM lookup, account history, recent interactions, talking points. The phone agent should never pause mid-call to research.

**Phone Agent** — Focused on the live conversation. Receives pre-loaded context as params. Minimal tool calling. Drives a singular outcome (book a meeting, capture intake, triage an incident).

**Post-Call Agent** — Processes the call outcome. CRM updates, summary generation, follow-up task creation. Runs after the call ends.

**When a standalone phone agent is fine:** Simple, single-purpose calls with no pre-research and no post-call integrations.

## Making an Agent a Phone Agent

An agent takes and places calls only when `runtime.engine` is `"phone_call"`. `relevance_create_agent` can't set it, so create the agent, then set the engine and the phone settings in one `relevance_update_agent` patch:

```typescript
relevance_update_agent({
  agent_id: '...',
  patch: {
    runtime: {
      engine: 'phone_call',
      phone_call: {
        first_message_mode: 'agent-generates-first-message',
        end_call_tool_enabled: true,
        // voice, transfer_call_tool and the other settings below go here too
      },
    },
  },
});
```

- Don't set `runtime.enabled`. With no engine set, `runtime.enabled: true` makes every run fail with "Unhandled Runtime"; fix it with `runtime.enabled: false`.
- The update's result shows `agent.runtime_engine`; it is absent on a default agent. Only `"phone_call"` takes calls, so check it before you tell the user the phone agent is built.

## Core Design Principles

### 1. Front-Load Context, Minimize Tool Calls

Every tool call during a live call adds latency. The agent says "please hold" and waits.

- Do all research and lookups BEFORE the call in a pre-call agent
- Pass context into the phone agent via params
- If tools are unavoidable, make them lightweight and fast (sub-2s response)

### 2. Drive a Singular Action

Each phone agent should drive ONE clear outcome:

- Book a meeting
- Capture an intake form
- Triage an incident
- Verify identity
- Confirm an appointment

Do not combine multiple objectives. Multi-objective agents lose conversational focus.

### 3. Keep Responses Short

Phone is not chat. People zone out after 8-10 seconds of continuous speech.

- 2-3 sentences per turn maximum
- Ask ONE question at a time
- Use verbal acknowledgments: "Got it", "Makes sense", "Right"

## Prompt Engineering for Voice

### Phone Call Experience Guidelines

Every phone agent system prompt should include:

```
## Phone Call Experience Guidelines

You are in the middle of a live phone call. Speak naturally and conversationally.

Phone call audio may be interrupted by background noise -- ignore it.

If the caller's audio cuts out briefly, wait a moment, then say: "Sorry, I think I lost you for a second -- could you say that last bit again?"

Keep responses concise. 2-3 sentences per turn, then pause for their response.

Use verbal acknowledgements: "mm-hmm", "right", "got it", "makes sense".

If there is a long silence (5+ seconds), gently prompt: "Still with me?" or "Take your time."

Never say "as an AI" unless directly asked. If asked, be honest.
```

### What to Ban in Prompts

Explicitly prohibit:

- Markdown formatting (bold, headers, bullets) -- the agent may say "asterisk asterisk" aloud
- Reading JSON or structured data verbatim
- Em dashes in spoken output
- Stacking multiple questions in one turn
- Long monologues (set a sentence limit)

### Text Normalization

Phone agents must speak data naturally:

- Emails: "charlie dot birch at relevance dot ai"
- Phone numbers: "oh four one two, three four five, six seven eight"
- Dates: "March nineteenth" not "2026-03-19"
- Money: "fifteen hundred dollars" not "$1,500"
- URLs: avoid reading URLs -- offer to send via email/SMS instead

### Conversation Flow Design

Structure prompts around **phases**, not checklists:

```
Phase 1: Opening (30 seconds)
- Greeting, introduce yourself, confirm it is a good time

Phase 2: Core Discussion (5-7 minutes)
- Weave information gathering into natural conversation
- Do NOT present as a checklist

Phase 3: Closing (30 seconds)
- Summarise what you heard back for confirmation
- Explain next steps
- Thank them
```

## Voice Configuration

### Voice Providers

`runtime.phone_call.voice` is optional: leave it out and calls use a default ElevenLabs voice. If you set it, `provider`, `model` and `voiceId` are all required, or the update is rejected.

| `provider`   | `model`                                                          |
| ------------ | ---------------------------------------------------------------- |
| `elevenlabs` | `eleven_v4_turbo`, `eleven_flash_v2_5`, `eleven_multilingual_v2` |
| `cartesia`   | `sonic-3`, `sonic-2`                                             |
| `openai`     | `gpt-4o-mini-tts`                                                |

- **ElevenLabs** is what the app gives a new phone agent, and the only provider that takes a voice ID outside the app's voice list, such as a cloned voice; a cloned voice needs the project's own ElevenLabs API key.
- **Cartesia Sonic-3** costs less per minute of speech than ElevenLabs.
- `voiceId` comes from the provider's voice list; let the user pick one in the agent's voice settings in the Relevance app rather than guessing an ID.
- `speed` is 0.7-1.2: slightly slower suits empathetic calls. On outbound calls it only applies to ElevenLabs voices.

### Transcription (STT)

**Deepgram Nova-3** is recommended:

- Native streaming with sub-300ms latency
- Multi-language support with built-in diarization
- 36% lower word error rate with accents/noise vs Whisper

### Runtime Configuration

Key `runtime.phone_call` settings:

| Setting                   | Recommendation                                                                                    | Why                                                                                                   |
| ------------------------- | ------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `first_message_mode`      | `agent-speaks-first` with a `first_message` (inbound), `agent-generates-first-message` (outbound) | Inbound callers expect immediate greeting; without `first_message`, `agent-speaks-first` says nothing |
| `end_call_tool_enabled`   | `true` always                                                                                     | Phantom tool for graceful call endings                                                                |
| `recording_enabled`       | `true` for production (see compliance note below)                                                 | Required for QA and compliance                                                                        |
| `silence_timeout_seconds` | 30                                                                                                | Prevents zombie calls                                                                                 |

**Recording note:** Check jurisdictional consent requirements before enabling. See the Compliance section below.

### Transferring a Call to a Person

A phone agent can hand the call to a person. Add `transfer_call_tool` under `runtime.phone_call` in the same `relevance_update_agent` patch, with `enabled: true` (it defaults to off, and the agent gets no transfer tool without it) and at least one destination:

```typescript
transfer_call_tool: {
  enabled: true,
  destinations: [
    {
      type: 'number',
      number: '+61255501234', // E.164
      description: 'Sales team: transfer when the caller wants to buy or upgrade',
      message: 'Transferring you to our sales team now.', // '' transfers silently
      // extension: '123', // optional, outbound calls only
    },
  ],
},
```

The agent decides when to transfer from each destination's `description`, so say when to use it there and in the system prompt. On outbound calls `message` is spoken as written; on inbound calls it is an instruction for what the agent says, and `extension` is not dialled.

### Agent Settings for Phone

| Setting                    | Recommendation                    | Why                                          |
| -------------------------- | --------------------------------- | -------------------------------------------- |
| `autonomy_limit_behaviour` | `terminate-conversation`          | Cannot ask for approval mid-call             |
| `autonomy_limit`           | 20                                | High enough for full call flow               |
| `temperature`              | 0 - 0.3                           | Consistent, predictable responses            |
| `model`                    | `relevance-performance-optimized` | Phone agents need speed                      |
| `action_behaviour`         | `never-ask`                       | Tools must run without approval during calls |

## Latency Management

Target sub-800ms mouth-to-ear response time.

```
STT transcription:     200-400ms
LLM inference:         200-500ms
TTS synthesis:         75-100ms (first byte)
Network overhead:      50-100ms
Total target:          < 800ms per turn
```

Strategies:

1. **Pre-load all context** -- most impactful
2. **Use fast, lightweight tools** (sub-2s)
3. **Use `relevance-performance-optimized` model**
4. **Filler phrases** for unavoidable pauses: "Let me check that for you"
5. **Post-call processing** for CRM updates and summaries

## Post-Call Processing

### Silent JSON Extraction

The phone agent's prompt should include:

```
After the call ends, extract all captured information into a JSON object
and send it using the {{_actions.tool_id}} tool.

Do NOT read the JSON out loud on the call.
```

### What to Capture

- Call metadata: status, duration, confidence level
- Contact info: name, team, role
- Structured data: whatever the call was designed to capture
- Conversation summary: 3-5 sentence handoff note
- Follow-up actions: callback time, tasks created

## Common Pitfalls

1. **Tool-heavy phone agents** -- every tool call is a "please hold" moment
2. **Multi-objective agents** -- leads to unfocused, poor-quality calls
3. **No escalation path** -- callers trapped in loops with no way to a human. Add a transfer destination (see [Transferring a Call to a Person](#transferring-a-call-to-a-person))
4. **Reading structured data aloud** -- JSON/lists spoken verbatim sounds robotic
5. **Cost-optimized model** -- adds noticeable latency
6. **Missing post-call processing** -- nothing recorded to CRM
7. **Instructions in spoken templates** -- parenthetical instructions get read aloud. Use comments (`<!-- -->`) for instructions
8. **Premature call termination** -- the phantom `end_call_tool` fires on any perceived farewell. Add guardrails: "Only end the call after the full closing sequence is complete"

## Compliance

- **Recording consent:** Check jurisdictional requirements (some require two-party consent). When in doubt, announce recording at call start
- **AI disclosure:** If asked, always be honest about being AI. Some jurisdictions require proactive disclosure
- **Outbound calls:** Check applicable regulations for AI-generated calls. Provide an opt-out mechanism
- **Data retention:** Follow org retention and access control policies for stored transcripts
