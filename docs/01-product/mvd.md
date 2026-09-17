# Minimum Viable Demo (MVD) — First90

## Purpose

The MVD is the smallest working version of First90 that demonstrates the core value proposition to a real user and validates the key hypothesis: *"An AI-generated, adaptive 30-60-90 day plan with daily check-ins will meaningfully reduce early-career anxiety and increase confidence."*

The MVD is not an MVP. It does not need to be polished, scalable, or complete. It needs to be real enough that a user can go through the core loop and feel the value.

---

## Core Loop (MVD)

```
Sign up → Enter role → Receive initial plan → Complete daily check-in → Receive micro-actions → Repeat
```

Everything outside this loop is out of scope for the MVD.

---

## In Scope

| Feature | Notes |
|---|---|
| Auth | Magic link via Supabase Auth. No social login yet |
| Role onboarding | Company name, job title, start date, rough role category (from archetype list) |
| Initial plan generation | PathArchitect v0.1 — generates a 30-60-90 plan from archetype template + role input. Single LLM call, not full agent loop |
| Plan view | List of goals by phase (Orient / Experiment / Decide). Mark complete. No editing yet |
| Daily check-in | Confidence score (1–10 slider), energy score (1–10 slider), free text (optional), 1–3 tag selection |
| Micro-actions | ExperienceCoach v0.1 — 1–3 actions for tomorrow based on check-in. Single LLM call |
| Suggestions inbox | Simple list of pending agent proposals. Accept or dismiss |
| Push notifications | Evening check-in reminder (Expo Notifications) |
| CommsCoach (Basic) | CommsCoach v0.1 — on-demand message rewrites, returning 2 variants. Single LLM call |

## Out of Scope (MVD)

- Advanced CommsCoach (proactive scripts, tone modes, 3 variants, deferred to v0.2)
- Role-fit scoring and archetype tracking (deferred to v0.2)
- Full Antigravity agent orchestration (use direct LLM calls in v0.1)
- SignalScout pattern detection (deferred to v0.2)
- On-device SLM privacy layer (deferred to v0.3)
- Social login, profile editing, account settings
- B2B / employer dashboard
- Web version

---

## Screens (MVD)

### 1. Onboarding flow (3 screens)
- **Screen 1.1 — Welcome:** App name, tagline, "Get started" CTA. Magic link email entry.
- **Screen 1.2 — Role setup:** Fields: company name, job title, start date, role category (dropdown from archetype list — 6–8 options). CTA: "Build my plan".
- **Screen 1.3 — Plan loading:** Animated state while PathArchitect generates the plan. Message: "Building your personalised 30-60-90 day plan..."

### 2. Home / Today screen
- Current phase badge (Orient / Experiment / Decide)
- Day counter ("Day 14 of 90")
- Today's focus: top 1–2 goals for the current week
- Check-in prompt card (if not completed today): "How's today going?"
- Micro-actions from last check-in
- Bottom nav: Today | Plan | Coach | Inbox

### 3. Plan screen
- Three collapsible sections: Orient (1–30), Experiment (31–60), Decide (61–90)
- Each section lists goals with status (not started / in progress / complete)
- Tap a goal to view detail and mark complete

### 4. Check-in screen
- Step 1: Confidence slider (1–10) with label ("How confident do you feel today?")
- Step 2: Energy slider (1–10) with label ("How energising was today's work?")
- Step 3: Free text (optional) — "Anything specific on your mind? (optional)"
- Step 4: Tags — tap to select up to 3 from Feelings/Context (`meetings`, `deadlines`, `relationships`, `workload`, `learning`, `unclear_expectations`, `imposter_syndrome`, `win`) or Activities (`data_analysis`, `strategy`, `coding`, `client_calls`, `writing`, `process_improvement`, `people_interaction`, `financial_analysis`)
- Submit → triggers ExperienceCoach → navigates to micro-actions result

### 5. Micro-actions result screen
- "Here's what I'd suggest for tomorrow:"
- 1–3 action cards, each with title, 1-line rationale, accept/dismiss
- Accepted actions appear in Today screen

### 6. Inbox screen
- List of pending suggestions from agents
- Each card: agent name, suggestion type, summary, accept/dismiss
- Empty state: "No suggestions right now. Check back after your next check-in."

### 7. CommsCoach screen
- Paste message box (multiline)
- Recipient dropdown (manager / peer / skip-level / external)
- Goal of message input (short text)
- CTA: "Get rewrites"
- Results view showing 2 improved variants, explanation of changes, and a copy button

---

## Success Criteria

| Metric | Target |
|---|---|
| User completes onboarding and receives a plan | 100% of test users |
| Plan feels relevant and personalised | 7+ out of 10 in user feedback |
| User completes at least 3 check-ins in first week | >60% of beta users |
| Micro-actions feel useful and actionable | 7+ out of 10 in user feedback |
| User would recommend to a peer | NPS > 30 at end of MVD test |

---

## Build Order

1. Supabase project + schema (users, roles, plans, plan_phases, goals, check_ins, suggestions, events)
2. Supabase Auth — magic link flow
3. Expo app scaffold (Expo Router, base navigation)
4. Onboarding flow (screens 1.1–1.3)
5. PathArchitect v0.1 — direct Claude/Gemini call, plan generation from archetype template
6. Plan screen
7. Check-in screen
8. ExperienceCoach v0.1 — direct Claude/Gemini call, micro-actions from check-in
9. Micro-actions result screen
10. CommsCoach v0.1 — direct Claude call, on-demand variants rewrite
11. CommsCoach screen
12. Today / Home screen
13. Inbox screen
14. Push notification — evening check-in reminder
13. Internal test with 3–5 users
14. Iterate on check-in UX based on feedback

---

## Tech Notes

- Use Claude Sonnet for PathArchitect v0.1 (plan quality matters more than speed here)
- Use Gemini Flash for ExperienceCoach v0.1 (speed matters — user is waiting on the check-in result screen)
- All LLM calls go via the BFF API — never directly from the Expo client
- Store all LLM outputs in `suggestions` table before presenting to user, even in v0.1
- No Antigravity orchestration yet — direct API calls only in MVD
