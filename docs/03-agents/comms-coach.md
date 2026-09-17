# CommsCoach — Agent Spec

## Purpose

Help the user communicate more confidently and effectively during their first 90 days. CommsCoach operates in two modes: **on-demand** (user pastes a draft message) and **proactive** (agent pre-generates a script before a known tough moment). It never sends messages on the user's behalf — all outputs are drafts for the user to review, edit, and send themselves.

---

## Triggers

### On-demand
- User opens the CommsCoach screen and pastes a draft email, Slack message, or Teams message.
- User describes a conversation they need to have ("I need to ask my manager for more responsibility").

### Proactive
- Orchestrator detects a plan task labelled `comms_required: true` is due within 48 hours.
- SignalScout flags a recurring communication anxiety pattern (e.g., user consistently tags check-ins with `stakeholder_comms` and `low_confidence`).
- A fit-check reflection is triggered (day ~45, day ~90) and the user needs a script for a manager development conversation.

---

## Inputs

| Input | Source | Notes |
|---|---|---|
| Draft message or conversation topic | User (direct input) | Free text; may be a full draft or just a topic |
| Recipient context | User (structured fields) | Who is this to? What is their relationship/seniority? |
| Goal of the message | User (structured fields) | What outcome does the user want? |
| Current plan phase | `plan_phases` | Affects tone guidance (phase 1 = more cautious; phase 3 = more assertive) |
| User tone preferences | `users.preferences` | Stored preference for default tone mode |
| Recent check-in context | `check_ins` (last 3) | Used to gauge current confidence level and tailor encouragement |

---

## Outputs

### On-demand rewrite
Returns **2–3 message variants**, each with:
- The rewritten message
- A bullet-point explanation of what changed and why
- A tone label (see Tone Modes below)

### Proactive script
Returns a **structured conversation script** with:
- Opening line
- Core ask or message body
- Anticipated response handling (1–2 scenarios)
- Closing / next step
- A brief coach note: why this approach fits the situation

All outputs are saved to the `suggestions` table (`suggestion_type: comms_draft`) and surface in the user's Suggestions inbox. The user can accept (copy to clipboard / share), edit, or dismiss.

---

## Tone Modes

| Mode | When to use | Character |
|---|---|---|
| **Professional-warm** | Emails to managers, senior stakeholders, clients | Respectful, clear, no filler. Friendly but not casual. |
| **Concise-direct** | Slack/Teams messages, quick updates, confirmations | Short sentences. One idea per message. No hedge words. |
| **Senior-peer** | Messages to colleagues at a similar level | Natural, collaborative, slightly informal. No over-explaining. |
| **Friendly-junior** | Introductory messages, coffee chat requests, thank-you notes | Warm, genuine, no corporate stiffness. Authentic curiosity. |

The default mode is inferred from the recipient context. The user can override via a toggle on the CommsCoach screen.

---

## Proactive Script Coverage

The following plan task types automatically trigger CommsCoach pre-generation when they fall within 48 hours:

| Task type | Script generated |
|---|---|
| `coffee_chat_request` | Message to request a coffee chat with a colleague in a target role |
| `shadowing_request` | Message to ask to shadow someone for a morning or day |
| `raise_development_conversation` | Structured script for the manager development conversation (phase 3) |
| `flag_a_mistake` | Script for proactively owning and communicating an error |
| `ask_for_more_responsibility` | Message or script for requesting stretch tasks or expanded scope |
| `first_client_intro` | Introductory message to a client or external stakeholder |
| `request_feedback` | Message asking a manager or senior colleague for structured feedback |

---

## Example Rewrite

**User input:**
> "Hi [Manager], just wanted to check in to see if maybe there might be any opportunities for me to get involved in a bit more work on the data side of things if that's okay?"

---

**Variant 1 — Professional-warm**
> "Hi [Manager], I've really enjoyed the data work I've been involved in so far and would love to take on more. Would you be open to discussing where I could contribute further on the data side? Happy to chat at our next 1:1."

*Changes: Removed hedge words ("just", "maybe", "a bit"). Led with a positive observation. Made the ask specific. Proposed a concrete next step.*

---

**Variant 2 — Concise-direct**
> "Hi [Manager], I'd like to take on more data-related work. Can we discuss at our next 1:1?"

*Changes: Cut to the core ask in two sentences. Removed all softening language.*

---

**Variant 3 — Senior-peer**
> "[Manager] — I've been gravitating towards the data side of things and want to dig in more. Could we carve out 10 minutes at our next 1:1 to talk about where that could go?"

*Changes: Informal but confident opener. "Carve out" signals proactiveness. Keeps it light and collaborative.*

---

## Prompt Skeleton

```
You are CommsCoach, a communications assistant for early-career professionals.

Context:
- User's current phase: {{plan_phase}} (day {{current_day}} of 90)
- Recipient: {{recipient_description}}
- Relationship: {{relationship}} (e.g., direct manager, senior colleague, peer)
- Goal: {{message_goal}}
- Requested tone mode: {{tone_mode}}

User's draft:
"""
{{draft_message}}
"""

Instructions:
1. Produce exactly {{variant_count}} rewritten variants (default: 3).
2. Label each variant with its tone mode.
3. For each variant, provide 2–4 bullet points explaining the key changes and why they improve the message.
4. Do not invent facts. If the draft is vague, keep the rewrite vague — do not assume context the user did not provide.
5. Never make the message longer than necessary. Conciseness is almost always an improvement.
6. Keep the user's voice. Do not replace genuine warmth with corporate stiffness.
7. If the draft is already strong, say so and provide only minor refinements.

Output format:
---
**Variant 1 — [Tone label]**
[Rewritten message]

*Changes:*
- [Change 1]
- [Change 2]
...
---
```

---

## Privacy

- Short drafts (<200 words) are processed by Gemma on-device — no data leaves the device.
- Longer drafts in Standard mode are PII-stripped before any cloud call. Cloud waterfall: Groq `llama-3.3-70b-versatile` (free) → OpenRouter free model → Gemini 2.5 Pro (paid, last resort). User sees the stripped version and must confirm before the cloud call is made.
- Completed drafts saved to `suggestions` are stored as structured JSON only (tone label + variant text). Raw user drafts are never persisted server-side.

---

## Constraints

- Never send a message on the user's behalf under any circumstances.
- Never assume context beyond what the user provides.
- Always produce at least 2 variants so the user has a genuine choice.
- If the user's draft contains content that could be professionally harmful (e.g. venting anger at a manager), flag it gently and offer a more constructive alternative — but still provide what was asked for.
- Agent outputs must always route through `suggestions` table. Direct write to other tables is prohibited.
