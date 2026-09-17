# Agent Flows — First90

This document describes the two primary agent interaction flows using sequence diagrams.

---

## Flow 1: Daily Check-in → ExperienceCoach → SignalScout

This is the core daily loop. It runs every evening when the user completes a check-in.

```mermaid
sequenceDiagram
  actor User
  participant App as Mobile App
  participant API as BFF API
  participant DB as Supabase
  participant AG as Antigravity
  participant EC as ExperienceCoach
  participant SS as SignalScout
  participant PA as PathArchitect
  participant LLM as Gemini Flash

  User->>App: Completes check-in (confidence, energy, text, tags)
  App->>API: POST /check-ins
  API->>DB: INSERT check_ins
  API->>DB: INSERT events (check_in_submitted)
  API-->>App: 200 OK (check-in saved)
  API->>AG: trigger(check_in_submitted, { check_in_id, user_id })

  AG->>DB: Load recent check-ins (last 7 days)
  AG->>DB: Load current plan + active goals
  AG->>EC: run(check_in, history, plan)

  EC->>LLM: Generate micro-actions (1–3)
  LLM-->>EC: Micro-actions JSON
  EC->>DB: INSERT suggestions (type: micro_actions)
  EC-->>AG: done, signal_summary

  AG->>SS: run(signal_summary, user_id)
  SS->>DB: Load confidence/energy time-series (last 30 days)
  SS->>DB: Load events log
  SS->>DB: Load role_fit_scores

  alt Persistent pattern detected (e.g. 3+ low confidence days)
    SS->>AG: trigger(pattern_alert, { pattern_type, severity })
    AG->>EC: run(pattern_intervention)
    EC->>LLM: Generate targeted intervention suggestion
    LLM-->>EC: Intervention suggestion
    EC->>DB: INSERT suggestions (type: intervention)
  end

  alt Plan slippage detected (>40% goals overdue)
    SS->>AG: trigger(plan_slippage, { overdue_count })
    AG->>PA: run(plan_review)
    PA->>LLM: Generate plan adjustment proposal
    LLM-->>PA: Plan delta JSON
    PA->>DB: INSERT suggestions (type: plan_update)
  end

  SS->>DB: UPDATE role_fit_scores (based on energy tags)
  AG-->>API: agent_run_complete

  App->>API: GET /suggestions?status=pending
  API->>DB: SELECT suggestions WHERE status=pending
  API-->>App: Pending suggestions
  App-->>User: Micro-actions displayed
```

### Notes
- The App polls for suggestions after submitting the check-in (or uses a Supabase Realtime subscription).
- ExperienceCoach always runs. SignalScout pattern detection and PathArchitect plan review are conditional.
- All suggestions require user acceptance before any state changes to plans or goals.
- Gemini Flash is used throughout this flow for latency. Claude is not called here.

---

## Flow 2: CommsCoach (On-Demand)

The user pastes a draft message or names a situation. CommsCoach returns variants with explanations.

```mermaid
sequenceDiagram
  actor User
  participant App as Mobile App
  participant API as BFF API
  participant DB as Supabase
  participant AG as Antigravity
  participant CC as CommsCoach
  participant LLM as Claude Sonnet

  User->>App: Opens CommsCoach, pastes draft message + context
  App->>API: POST /comms-coach { draft, context, tone_preference }

  API->>API: PII strip (replace names with [manager], [company], etc.)
  API->>DB: Load user's current plan phase
  API->>DB: Load user's tone preferences

  API->>AG: trigger(comms_coach_requested, { draft_stripped, context, phase, preferences })

  AG->>CC: run(draft, context, phase, preferences)
  CC->>LLM: Generate 2–3 message variants with rationale
  LLM-->>CC: Variants JSON ({ variant_name, rewritten_message, changes: [] })

  CC->>DB: INSERT suggestions (type: comms_variants)
  CC-->>AG: done
  AG-->>API: { suggestion_id }
  API-->>App: { suggestion_id }

  App->>API: GET /suggestions/:id
  API->>DB: SELECT suggestion WHERE id=:id
  API-->>App: Variants + rationale
  App-->>User: Displays variants side-by-side with change explanations

  User->>App: Selects preferred variant (copies to clipboard)
  App->>API: PATCH /suggestions/:id { status: accepted, selected_variant }
  API->>DB: UPDATE suggestions
  API->>DB: INSERT events (comms_variant_selected)
```

### Notes
- PII stripping happens in the BFF API before any text reaches Claude. Names, company names, and colleague references are replaced with typed placeholders.
- Claude Sonnet is used here (not Gemini) because message quality and nuance are more important than speed for communications coaching.
- The user copies their chosen variant manually — the app never sends a message on their behalf.
- If a "tough conversation" task is due in the user's plan (e.g. "have development chat with manager"), CommsCoach can be triggered proactively by SignalScout. The flow is identical except the trigger is `proactive_script_needed` rather than `comms_coach_requested`.

---

## Flow 3: Role Setup → PathArchitect (Initial Plan Generation)

Runs once when the user completes onboarding.

```mermaid
sequenceDiagram
  actor User
  participant App as Mobile App
  participant API as BFF API
  participant DB as Supabase
  participant AG as Antigravity
  participant PA as PathArchitect
  participant LLM as Claude Sonnet

  User->>App: Completes role setup (company, title, start date, archetype)
  App->>API: POST /roles { company_name, job_title, start_date, archetype_id }
  API->>DB: INSERT roles
  API->>DB: INSERT events (role_setup_complete)
  API->>AG: trigger(role_setup_complete, { role_id, user_id, archetype_id })

  AG->>DB: Load role_archetypes WHERE id=archetype_id
  AG->>PA: run(role, archetype_template)

  PA->>LLM: Generate personalised 30-60-90 plan (phases, goals, success signals)
  LLM-->>PA: Plan JSON

  PA->>DB: INSERT plans
  PA->>DB: INSERT plan_phases (orient, experiment, decide)
  PA->>DB: INSERT goals (per phase)
  PA->>DB: INSERT events (plan_generated)

  PA-->>AG: done
  AG-->>API: plan_ready
  API-->>App: { plan_id }

  App->>API: GET /plans/:id
  API->>DB: SELECT plan + phases + goals
  API-->>App: Full plan
  App-->>User: Displays 30-60-90 plan
```

### Notes
- This is the most important LLM call in the product. Plan quality at this step sets the tone for the entire 90-day experience. Claude Sonnet is used.
- The plan is written directly to Supabase (no `suggestions` table step) because it is the initial state, not a change proposal.
- If the user does not select an archetype during onboarding, PathArchitect uses a generic "early-career professional" template and refines it after the first 2 weeks of check-in data.
