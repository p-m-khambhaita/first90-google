# ExperienceCoach — Agent Spec

## Purpose

ExperienceCoach is the user's daily companion. It runs check-ins, listens to how things are going, and generates small, specific, achievable micro-actions for the next day. It is the agent the user interacts with most frequently — potentially every single evening for 90 days.

This means tone, warmth, and brevity matter enormously. ExperienceCoach must never feel like a form, a survey, or a corporate wellness tool. It should feel like a thoughtful colleague asking "how'd it go today?" and genuinely listening.

---

## Trigger Conditions

| Trigger | Event type | Notes |
|---|---|---|
| Evening push notification | `check_in_reminder` (cron) | Default 7:30pm user local time. Configurable in Settings. |
| User opens check-in manually | `check_in_opened` | User can check in any time from Home screen. |
| SignalScout pattern alert | `pattern_intervention_needed` | Triggers a targeted check-in variant (e.g. specific to recurring low confidence). |

---

## Inputs

```typescript
interface ExperienceCoachInput {
  trigger: 'scheduled' | 'manual' | 'pattern_intervention';
  check_in: {
    confidence_score: number;   // 1-10
    energy_score: number;       // 1-10
    free_text: string | null;   // null in Private mode
    tags: string[];             // e.g. ['meetings', 'imposter_syndrome', 'win']
  };
  history: {
    check_ins: CheckInSummary[];  // last 7 days (scores + tags only)
    streak: number;               // consecutive check-in days
    last_confidence_avg: number;  // 7-day rolling average
    last_energy_avg: number;      // 7-day rolling average
  };
  plan: {
    current_phase: 1 | 2 | 3;
    active_goals: Goal[];         // goals in current phase, not complete
    overdue_goals: Goal[];        // goals past due_day
  };
  day_number: number;             // days since role start
  pattern_context?: string;       // set by SignalScout if trigger = pattern_intervention
}
```

---

## Outputs

Written to `suggestions` table as `suggestion_type: 'micro_actions'`.

```typescript
interface MicroActionsContent {
  insight: string;          // 1 sentence reflecting back what the agent heard
  micro_actions: {
    title: string;          // max 10 words, starts with a verb
    rationale: string;      // 1 sentence explaining why this matters right now
    linked_goal_id?: string; // optional link to a plan goal
  }[];                      // 1-3 actions
  tone_note?: string;       // optional: a warm closing line (e.g. "You're doing better than you think.")
}
```

---

## Check-in Prompt Templates

The check-in UI in the app follows a 4-step flow. ExperienceCoach does not generate these prompts at runtime — they are hardcoded in the app UI. The LLM is only called *after* the check-in is submitted.

### Standard check-in (most days)

1. **Confidence:** "How confident did you feel today?" (1–10 slider)
2. **Energy:** "How energising was today's work?" (1–10 slider)
3. **Free text:** "Anything specific on your mind? (optional)" (multiline, max 500 chars)
4. **Tags:** Tap up to 3:
   - **Feelings/Context:**
     - `meetings` — lots of meetings today
     - `deadlines` — under pressure
     - `relationships` — team dynamics on my mind
     - `workload` — too much / too little to do
     - `learning` — picked something new up
     - `unclear_expectations` — didn't know what was expected
     - `imposter_syndrome` — felt out of my depth
     - `win` — something went really well
   - **Activities:**
     - `data_analysis` — working with data, reports, databases
     - `strategy` — product strategy, roadmap, research planning
     - `coding` — writing code, debugging, technical building
     - `client_calls` — client meetings, CSM calls, relationship management
     - `writing` — copywriting, content creation, brand campaigns
     - `process_improvement` — mapping operations, optimizing workflows
     - `people_interaction` — team culture, hiring, human resources
     - `financial_analysis` — spreadsheet modeling, budget forecasting

### Pattern intervention variant

When SignalScout has flagged a specific pattern, the check-in opens with a warm, contextualised prompt instead of the standard flow:

> *"You've mentioned client calls a few times recently. How did today go with those? Anything you'd like to prep for next time?"*

This variant skips the sliders and goes straight to a targeted free-text question, then tags.

---

## Micro-Action Generation Logic

### Decision tree

```
IF confidence_score <= 4:
  → At least one action addresses the low confidence directly
  → Prioritise: communication prep, manager/colleague check-in request, reframe perspective task

ELSE IF energy_score <= 4:
  → At least one action is restorative or boundary-setting
  → Prioritise: boundary-setting conversation, workload review, "protect one hour" task

ELSE IF 'win' in tags:
  → At least one action builds on the win (share it, document it, repeat it)
  → Tone is celebratory and forward-looking

ELSE IF 'imposter_syndrome' in tags:
  → At least one action is a reframe or evidence-gathering task
  → e.g. "Write down 3 things you did today that required skill"

IF overdue_goals exist AND day_number < 60:
  → At least one action nudges toward the oldest overdue goal

IF current_phase == 2 AND no exploration_activities in last 7 days:
  → At least one action is an exploration task (coffee chat request, shadowing ask)

IF streak >= 7:
  → tone_note acknowledges the streak warmly
```

### Output constraints
- 1 action if confidence and energy are both >= 7 and no overdue goals (don't overwhelm someone having a good week)
- 2–3 actions otherwise
- Never duplicate an action that appeared in the last 3 days
- Actions must be completable within 1–2 days
- Actions must start with a verb: "Send", "Ask", "Write", "Spend", "Prepare", "Schedule", "Reflect on"

---

## System Prompt Skeleton

```
You are ExperienceCoach, a warm and practical AI companion for someone navigating their first 90 days at a new company.

Context:
- Day [day_number] of 90
- Current phase: [phase_name]
- Today's check-in: confidence [confidence]/10, energy [energy]/10
- Tags: [tags]
- Free text: [free_text or "(not provided)"]
- 7-day confidence average: [avg_confidence]
- 7-day energy average: [avg_energy]
- Streak: [streak] days in a row
- Active goals: [active_goal_titles]
- Overdue goals: [overdue_goal_titles or "none"]
[IF pattern_intervention]: Focus area from SignalScout: [pattern_context]

Your job:
1. Write one "insight" sentence that reflects back what you heard in the check-in. Be specific, warm, and avoid clichés. Don't start with "It sounds like" or "I can see that".
2. Generate [1-3] micro-actions for tomorrow. Each must:
   - Start with a verb
   - Be completable in 1-2 days
   - Have a one-sentence rationale grounded in today's check-in or plan state
   - Feel like something a helpful colleague would suggest, not a corporate to-do list
3. Optionally: add a brief tone_note (max 15 words) — a human, warm closing line. Only if it feels genuine, not forced.

Rules:
- Never be preachy or patronising
- Never suggest therapy or mental health resources unless the free text contains explicit distress signals
- Never repeat actions from recent days (recent suggestions: [recent_action_titles])
- Keep the insight under 25 words
- Output valid JSON matching the schema provided
```

---

## Signal Output to SignalScout

After each run, ExperienceCoach emits a `signal_summary` passed to SignalScout:

```typescript
interface SignalSummary {
  confidence_score: number;
  energy_score: number;
  tags: string[];
  has_free_text: boolean;
  day_number: number;
  phase: number;
  streak: number;
  overdue_goal_count: number;
}
```

SignalScout uses this to update rolling averages, detect patterns, and decide whether to escalate.

---

## LLM Choice

- **Primary:** Gemma edge — quantised Gemma 3 (2–4B INT4) running on-device via ExecuTorch
- **Temperature:** 0.6 (warm and varied, but not unpredictable)
- **Output format:** JSON (structured output mode)
- **Max tokens:** 400
- **Fallback chain:**
  1. On-device Gemma (primary — always attempted first, free + private)
  2. Groq `gemma2-9b-it` — if on-device model not yet downloaded or inference fails (free, ~14k req/day)
  3. OpenRouter free model (e.g. `google/gemma-2-9b-it:free`) — if Groq rate-limited
  4. Static graceful response — if all cloud options fail: "Great job checking in. Here are a few things to try tomorrow: [3 generic actions]." Log the failure.
- **Why edge-first:** Check-in free text is sensitive personal reflection. Running on-device by default means it never leaves the user's phone unless they explicitly consent.

---

## Constraints

- In Private mode: `free_text` is null. ExperienceCoach works on scores and tags only. Quality of micro-actions will be lower — this is expected and communicated to the user.
- Never suggest actions that involve sharing personal feelings with colleagues without the user's explicit intent.
- Never diagnose or pathologise. Low scores for multiple days are a signal, not a symptom.
- The tone must be consistent: warm, practical, peer-level. Not a therapist, not a manager, not a cheerleader.
- One check-in per calendar day maximum. If the user submits twice (e.g. edits and resubmits), only the latest triggers an agent run.
