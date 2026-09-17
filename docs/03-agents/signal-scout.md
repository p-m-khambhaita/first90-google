# SignalScout — Agent Spec

## Purpose

SignalScout is the background analytics agent. It has no direct user-facing interface. Its job is to watch for meaningful patterns across check-ins, events, and plan completion data, then trigger the right agent or surface the right insight at the right time. It also maintains the **role-fit score** for each archetype in the user's library.

Think of SignalScout as the nervous system of First90: it receives signals from everything the user does and decides when something is worth acting on.

---

## Triggers

| Trigger | Frequency | Scope |
|---|---|---|
| Post check-in | After every check-in is logged | Lightweight: pattern check on last 7 days |
| Daily scheduler | Every night at ~21:00 (user timezone) | Lightweight: stale task detection, upcoming task alert |
| Weekly scheduler | Every Sunday night | Full analysis: fit-score recalculation, plan completion rate, pattern summary |
| Day 45 | Once, on day 45 of role | Fit-check reflection trigger |
| Day 90 | Once, on day 90 of role | Final fit-check + end-of-programme reflection trigger |

---

## Inputs

| Input | Source | Notes |
|---|---|---|
| Check-in time series | `check_ins` | confidence_score, energy_score, tags, free_text summary |
| Plan completion data | `goals`, `plans` | Completed, skipped, and overdue goals |
| Events log | `events` | Exploration activities completed, agent calls made, user actions |
| Role-fit scores | `role_fit_scores` | Current scores per archetype, used as baseline for updates |
| Exploration activities | `exploration_activities` | Shadowing, coffee chats, stretch tasks — completion and energy outcome |
| Role archetype markers | `role_archetypes` | `energising_markers` and `draining_markers` arrays per archetype |

---

## Outputs

All SignalScout outputs are either:
1. **Agent calls** — internal orchestrator events triggering another agent
2. **Suggestions** — saved to `suggestions` table (`agent_name: signal_scout`) for surfacing in the app
3. **Score updates** — direct writes to `role_fit_scores` (the only table SignalScout may write to directly, via service role)

| Output type | Condition | Action |
|---|---|---|
| ExperienceCoach call | Confidence score <= 4 for 3+ consecutive check-ins | Trigger ExperienceCoach with `context: persistent_low_confidence` |
| ExperienceCoach call | Energy score <= 3 for 5+ check-ins in a 10-day window | Trigger ExperienceCoach with `context: sustained_low_energy` |
| PathArchitect call | Plan completion rate < 40% over last 14 days | Trigger PathArchitect with `context: plan_slippage` |
| PathArchitect call | User has skipped the same goal type 3+ times | Trigger PathArchitect with `context: recurring_goal_skip` |
| CommsCoach call | Comms-tagged task due within 48 hours | Trigger CommsCoach proactive script generation |
| Fit-check suggestion | Day 45 or Day 90 threshold reached | Create suggestion (`type: fit_check_prompt`) surfacing fit scores + reflection prompt |
| Positive streak suggestion | 5+ consecutive check-ins with confidence >= 7 | Create suggestion (`type: positive_reinforcement`) — brief celebratory message |
| Role-fit score update | After each check-in and after each exploration activity is completed | Recalculate scores (see Fit-Score Logic below) |

---

## Fit-Score Logic

Each role archetype in `role_archetypes` has two marker arrays:
- `energising_markers` — activity tags that, when rated high-energy by the user in check-ins, increase fit score (selected from the 8 unified activity tags: `data_analysis`, `strategy`, `coding`, `client_calls`, `writing`, `process_improvement`, `people_interaction`, `financial_analysis`)
- `draining_markers` — activity tags that, when rated low-energy, decrease fit score (selected from the 8 unified activity tags)

### Score update rules

**From check-ins:**
```
For each tag in check_in.tags:
  If tag matches an energising_marker for archetype A:
    AND check_in.energy_score >= 7:
      role_fit_scores[A] += 0.05 (capped at 1.0)
  If tag matches a draining_marker for archetype A:
    AND check_in.energy_score <= 4:
      role_fit_scores[A] -= 0.03 (floored at 0.0)
```

**From exploration activities:**
```
For each completed exploration_activity targeting archetype A:
  If activity.energy_outcome == 'high':
    role_fit_scores[A] += 0.10
  If activity.energy_outcome == 'neutral':
    role_fit_scores[A] += 0.02
  If activity.energy_outcome == 'low':
    role_fit_scores[A] -= 0.05
```

**Decay:**
Scores that have not been updated in 14+ days decay by 0.01/day (minimum 0.0). This prevents old signals from permanently dominating the score.

### Surfacing
Scores are ranked and displayed as a plain-language list at the day-45 and day-90 fit-check moments. Example:

> **Your current fit signals:**
> 1. Data & Analytics — Strong (0.72)
> 2. Product & Strategy — Emerging (0.41)
> 3. Client & Account Management — Weak (0.18)
>
> Based on 34 check-ins and 2 exploration activities.

Raw scores are never shown — only the plain-language banding.

| Score range | Label |
|---|---|
| 0.60–1.00 | Strong |
| 0.35–0.59 | Emerging |
| 0.10–0.34 | Weak |
| 0.00–0.09 | No signal |

---

## Pattern Detection

SignalScout uses a lightweight rule-based pattern detector (not a separate LLM call) for the post-check-in and daily runs. Only the weekly full-analysis run uses an LLM for plain-language pattern summarisation.

### Rule-based checks (no LLM)

```
low_confidence_streak:     confidence_score <= 4 for N consecutive check-ins (N=3)
sustained_low_energy:      energy_score <= 3 for M check-ins in a 10-day window (M=5)
plan_slippage:             goals completed / goals due < 0.40 over last 14 days
recurring_skip:            same goal.title skipped 3+ times
upcoming_comms_task:       goal with comms_required=true due within 48h, no CommsCoach suggestion exists
day_milestone:             current_day IN [45, 90]
positive_streak:           confidence_score >= 7 for 5+ consecutive check-ins
```

### Weekly LLM digest (Groq cloud, Edge Function)

Once per week, SignalScout runs a lightweight LLM call to generate a plain-language insight summary from the past 7 days of check-in data. This is surfaced as a weekly digest card in the app.

```
You are SignalScout, an analytics agent for First90.

Here is a summary of the user's last 7 check-ins:
{{check_in_summary_json}}

Plan completion this week: {{completion_rate}}% ({{completed}}/{{due}} goals)

Write a short weekly insight (3–4 sentences) that:
1. Acknowledges the week honestly (don’t sugarcoat a hard week or underplay a good one)
2. Identifies one specific pattern — positive or concerning
3. Connects the pattern to one concrete suggestion for next week
4. Ends with one encouraging sentence

Tone: warm, direct, not clinical. Like a thoughtful mentor reviewing your week.
Do not use bullet points. Write in plain prose.
```

---

## Privacy

- SignalScout runs in a Supabase Edge Function via the service role.
- It operates on structured data only (scores, tags, completion rates). It never reads free-text `check_ins.free_text` directly.
- The weekly LLM call receives only a structured JSON summary of check-in scores and tags — never the raw free-text field.
- PII stripping is not required because SignalScout does not process free text.

---

## Constraints

- SignalScout may only write directly to `role_fit_scores`. All other outputs must go via `suggestions` or internal agent calls.
- Never trigger more than one instance of the same agent call within a 24-hour window for the same user and context type.
- Day-milestone triggers (day 45, day 90) fire exactly once per role, tracked in `events` table (`event_type: milestone_triggered`).
- Fit scores must never be surfaced as raw numbers in the user interface. Always use the plain-language banding table above.
- Do not trigger any LLM call for users who have not completed at least 3 check-ins. Wait for a baseline.
