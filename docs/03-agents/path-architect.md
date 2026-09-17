# PathArchitect — Agent Spec

## Purpose

PathArchitect builds and maintains the user's personalised 30-60-90 day plan. It is the first agent to run (triggered by role setup) and the one most responsible for the user's first impression of First90.

It operates on a simple principle: start from a role-archetype template, then adapt weekly based on what the user has actually done, skipped, or flagged as irrelevant. The plan should always feel like *theirs* — not a generic document.

---

## Trigger Conditions

| Trigger | Event type | Notes |
|---|---|---|
| Role setup complete | `role_setup_complete` | Runs once. Generates the initial plan. |
| Weekly schedule | `weekly_plan_review` (cron) | Runs every Sunday evening. Reviews completion and proposes adjustments. |
| SignalScout escalation | `plan_slippage_detected` | Triggered when >40% of active goals are overdue. |
| User requests re-plan | `user_requested_replan` | Manual trigger from the Plan screen. |

---

## Inputs

```typescript
interface PathArchitectInput {
  trigger: 'initial' | 'weekly_review' | 'slippage' | 'user_request';
  user: {
    id: string;
    timezone: string;
    preferences: Record<string, unknown>;
  };
  role: {
    id: string;
    company_name: string;   // PII-stripped to [company] before LLM call
    job_title: string;
    start_date: string;     // ISO date
    archetype_id: string | null;
  };
  archetype: RoleArchetype | null;  // Full archetype record if archetype_id is set
  current_plan: Plan | null;        // null on initial trigger
  phases: PlanPhase[];              // empty on initial trigger
  goals: Goal[];                    // empty on initial trigger
  recent_check_ins: CheckIn[];      // last 14 days (scores + tags only, no free text)
  events: Event[];                  // last 30 days
  day_number: number;               // days since role start_date
}
```

---

## Outputs

All outputs are written to the `suggestions` table. Nothing is applied directly to plans or goals without user acceptance.

### Initial plan (`suggestion_type: 'initial_plan'`)

```typescript
interface InitialPlanContent {
  phases: {
    phase_number: 1 | 2 | 3;
    name: string;
    theme: string;
    start_day: number;
    end_day: number;
    goals: {
      title: string;
      description: string;
      success_signal: string;
      due_day: number;
    }[];
  }[];
  rationale: string;  // 1-2 sentence explanation shown to user
}
```

### Plan update (`suggestion_type: 'plan_update'`)

```typescript
interface PlanUpdateContent {
  changes: {
    action: 'add_goal' | 'remove_goal' | 'update_goal' | 'reschedule_goal';
    goal_id?: string;       // for update/remove/reschedule
    phase_number?: number;  // for add
    data?: Partial<Goal>;   // for add/update
    reason: string;         // plain-language reason shown to user
  }[];
  summary: string;          // 1-2 sentence summary shown in suggestions inbox
}
```

---

## Behaviour by Trigger

### Initial plan generation

1. Load the role archetype template (if archetype selected) or use the generic early-career template.
2. Personalise the template based on job title and start date.
3. Generate 4–6 goals per phase (12–18 goals total) with clear success signals and suggested due days.
4. Phase 1 goals are prescriptive and relationship-focused (meet key colleagues, understand team structure, shadow one process).
5. Phase 2 goals introduce exploration (one stretch task, one coffee chat with someone in a different role, one reflection on energy patterns).
6. Phase 3 goals shift to positioning (compare fit scores, draft manager conversation, define a 90-day-forward direction).
7. Write to `suggestions` as `initial_plan`. On acceptance, create the `plans`, `plan_phases`, and `goals` records.

### Weekly review

1. Calculate completion rate for the current phase's goals.
2. Identify goals that are overdue by >7 days and not in progress.
3. Identify goals the user has marked as skipped (and reason, if provided).
4. Propose: reschedule overdue goals, remove irrelevant ones, or add 1–2 new goals if the user is ahead.
5. Never propose removing more than 2 goals or adding more than 2 goals in a single weekly review.
6. Write to `suggestions` as `plan_update`.

### Slippage response

1. Triggered when >40% of active phase goals are overdue (by SignalScout).
2. More aggressive than weekly review: may propose simplifying the plan significantly.
3. Always includes a warm, non-judgmental framing in the `summary`: "Life gets busy. Here's a lighter version of your plan for the next two weeks."
4. Proposes moving overdue goals to a later phase or removing them entirely.

---

## System Prompt Skeleton

```
You are PathArchitect, an AI agent that builds personalised 30-60-90 day plans for early-career professionals.

Context:
- User: [job_title] at [company] (Day [day_number] of 90)
- Role archetype: [archetype_name] — [archetype_description]
- Trigger: [trigger_type]

Your job:
[INITIAL] Generate a personalised 30-60-90 day plan with 3 phases:
  Phase 1 (Days 1-30): Orient & Observe — understand, build relationships, map energy
  Phase 2 (Days 31-60): Experiment & Explore — stretch tasks, shadowing, role-fit discovery
  Phase 3 (Days 61-90): Decide & Position — reflect, compare fit, prepare manager conversation

For each phase, generate 4-6 goals. Each goal must have:
  - title: short, actionable (max 8 words)
  - description: 1-2 sentences explaining what to do and why
  - success_signal: one concrete observable indicator of completion
  - due_day: suggested day relative to role start (e.g. 14, 21, 45)

[WEEKLY_REVIEW] Review the current plan. Recent check-in data: [check_in_summary].
Completed goals: [completed_count]. Overdue goals: [overdue_goals].
Propose specific changes only. Be conservative — max 2 additions, max 2 removals.

Rules:
- Be warm, encouraging, and specific. Avoid corporate jargon.
- Goals must be realistic for someone new to a professional environment.
- Never include goals that require authority the user doesn't have yet.
- Phase 1 goals must be observable and completable without senior approval.
- Always explain why each goal matters in the description.
- Output valid JSON matching the schema provided.
```

---

## Role Archetype Template Structure

Each archetype in `role_archetypes` provides a starting template via its `typical_tasks` array. PathArchitect uses this to seed Phase 1 and Phase 2 goals with role-appropriate content.

Example mapping for **Data Analyst** archetype:

| Phase | Goal seeded from archetype |
|---|---|
| Orient (1–30) | "Shadow a data reporting process end-to-end" |
| Orient (1–30) | "Identify which datasets and tools the team uses daily" |
| Experiment (31–60) | "Build one small analysis or report independently" |
| Experiment (31–60) | "Request an informational chat with a senior analyst" |
| Decide (61–90) | "Reflect: does working with data energise or drain you?" |

If no archetype is selected, PathArchitect uses a generic template with relationship-building and observation goals only, and adds a "discover your archetype" goal in Phase 1.

---

## LLM Choice

- **Primary:** Groq `llama-3.3-70b-versatile` (free — try this first)
- **Fallback:** Gemini 2.5 Pro (`gemini-2.5-pro`) — only if Groq plan quality is insufficient after testing
- **Temperature:** 0.4 (creative enough for personalised phrasing, grounded enough for structured output)
- **Output format:** JSON (structured output mode)
- **Max tokens:** 2000 (initial plan), 800 (weekly update)
- **Cost note:** Runs once per user at signup + weekly. Groq free tier (14k req/day) is more than sufficient for any early beta.

---

## Constraints

- Never auto-apply plan changes. All outputs go to `suggestions` first.
- Never propose goals that require the user to have authority they don't yet have.
- Phase 1 goals must be completable within the first 30 days without any dependencies.
- If the user has completed all goals in a phase early, PathArchitect should advance the phase start, not pad with filler goals.
- The plan must always have at least 3 goals in the current active phase.
