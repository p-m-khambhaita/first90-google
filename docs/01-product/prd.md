# Product Requirements Document — First90

> Version 0.1 — MVD scope. This PRD covers the Minimum Viable Demo only. Post-MVD features are listed in the Out of Scope section.

---

## Overview

First90 is an agentic AI app for early-career professionals navigating their first 90 days in a new role. It provides a personalised, adaptive 30-60-90 day plan, daily check-ins, confidence tracking, communication coaching, and internal role-fit discovery — delivered through four AI agents running on Antigravity.

This PRD defines the requirements for the **Minimum Viable Demo (MVD)**: a working iOS app that demonstrates the core loop end-to-end with real LLM calls and real Supabase data.

---

## Goals

| Goal | Metric |
|---|---|
| Demonstrate the core value proposition to early beta users | 10 users complete the full onboarding + first check-in |
| Validate that PathArchitect generates useful, personalised plans | >70% of beta users rate their generated plan as "relevant" or "very relevant" |
| Validate that ExperienceCoach micro-actions are actionable | >60% of suggested micro-actions are accepted by users |
| Validate CommsCoach usefulness | >50% of users reuse or copy a CommsCoach variant |
| Collect qualitative feedback to inform v1 | 5+ structured user interviews completed post-MVD |

---

## Users

### Primary user (MVD)
A graduate, apprentice, or new hire who has started a new role in the last 0–30 days. They are:
- Mobile-first, comfortable with AI tools
- Anxious about performing well but lacking structured support
- Willing to spend 3–5 minutes per day on a check-in if it feels valuable
- Not expecting perfection — they understand this is an early product

### Out of scope for MVD
- B2B / employer-facing features
- Users more than 30 days into a role
- Android users

---

## User Stories

### Onboarding
- **US-01:** As a new user, I can sign up with my email (magic link) or Google account so that I don't need to create a password.
- **US-02:** As a new user, I can enter my company name, job title, and start date so that First90 has the context to build my plan.
- **US-03:** As a new user, I can select a role archetype from a visual picker so that PathArchitect can template my initial plan.
- **US-04:** As a new user, I receive a generated 30-60-90 day plan within 30 seconds of completing setup so that I have immediate value.

### Plan
- **US-05:** As a user, I can view my 30-60-90 day plan organised by phase so that I know what I'm working towards.
- **US-06:** As a user, I can see each goal's title, description, success signal, and due day so that I understand what "done" looks like.
- **US-07:** As a user, I can accept or dismiss agent suggestions so that I stay in control of my plan.
- **US-08:** As a user, I can mark a goal as complete so that my progress is tracked.

### Check-in
- **US-09:** As a user, I can complete a daily check-in with a confidence score, energy score, free text, and tags so that ExperienceCoach has data to work with.
- **US-10:** As a user, I receive 1–3 micro-actions after each check-in so that I always have a clear next step.
- **US-11:** As a user, I receive a push notification in the evening reminding me to check in so that I build a daily habit.

### CommsCoach
- **US-12:** As a user, I can paste a draft message and receive 2 improved variants with explanations so that I can communicate more confidently.
- **US-13:** As a user, I can copy a variant to my clipboard in one tap so that I can use it immediately.

### Confidence tracker
- **US-14:** As a user, I can view a chart of my confidence scores over time so that I can see my progress.
- **US-15:** As a user, I can see my current check-in streak and average confidence score so that I'm motivated to maintain consistency.

---

## Functional Requirements

### Authentication
- `FR-AUTH-01` Magic link sign-in via Supabase Auth
- `FR-AUTH-02` Google OAuth sign-in via Expo AuthSession
- `FR-AUTH-03` Session persisted across app restarts
- `FR-AUTH-04` Sign-out clears session and navigates to sign-in

### Role setup
- `FR-SETUP-01` User inputs company name (text), job title (text), start date (date picker)
- `FR-SETUP-02` User selects one role archetype from the library
- `FR-SETUP-03` On completion, BFF creates `role`, triggers PathArchitect, creates `plan` + `plan_phases` + `goals`
- `FR-SETUP-04` Setup must complete (plan generated) before home screen is shown

### Plan
- `FR-PLAN-01` Home screen shows current phase name, day counter, and goal list for current phase
- `FR-PLAN-02` Goals display title, due day, status (not started / in progress / complete)
- `FR-PLAN-03` Tapping a goal shows full detail: description, success signal, agent notes
- `FR-PLAN-04` Suggestions inbox shows all pending agent proposals with accept/dismiss actions
- `FR-PLAN-05` Accepting a plan-update suggestion updates `goals` table; dismissing sets `suggestions.status = dismissed`

### Check-in
- `FR-CHECKIN-01` Check-in screen has confidence slider (1–10), energy slider (1–10), optional free text, optional tag multi-select
- `FR-CHECKIN-02` Tags are predefined vocabulary. Feelings/Context: `meetings`, `deadlines`, `relationships`, `workload`, `learning`, `unclear_expectations`, `imposter_syndrome`, `win`. Activities: `data_analysis`, `strategy`, `coding`, `client_calls`, `writing`, `process_improvement`, `people_interaction`, `financial_analysis`.
- `FR-CHECKIN-03` Submitting a check-in saves to `check_ins` table and triggers ExperienceCoach
- `FR-CHECKIN-04` ExperienceCoach returns 1–3 micro-actions saved to `suggestions` and displayed inline after submission
- `FR-CHECKIN-05` Push notification sent at user-configured time each evening (default 8pm)
- `FR-CHECKIN-06` One check-in per calendar day maximum

### CommsCoach
- `FR-COMMS-01` CommsCoach screen has: message input (multiline), recipient type selector (manager / peer / skip-level / external), goal of message input (short text)
- `FR-COMMS-02` On submit, BFF calls CommsCoach agent and returns 2 variants
- `FR-COMMS-03` Each variant shows: label, full rewritten message, bullet list of changes
- `FR-COMMS-04` Copy button on each variant copies full message to clipboard
- `FR-COMMS-05` Short messages (<200 words) processed on-device; longer messages show consent prompt before cloud call

### Confidence tracker
- `FR-STATS-01` Line chart of confidence scores over last 30 days
- `FR-STATS-02` Current streak (consecutive days with a check-in)
- `FR-STATS-03` Average confidence score over last 7 days

---

## Non-Functional Requirements

| Requirement | Target |
|---|---|
| Plan generation time | < 30 seconds from setup completion |
| Check-in submission + micro-actions | < 10 seconds end-to-end |
| CommsCoach rewrite (on-device) | < 5 seconds |
| CommsCoach rewrite (cloud) | < 15 seconds |
| App launch to home screen (cold) | < 3 seconds |
| RLS enforcement | All user data isolated by `user_id = auth.uid()` |
| PII in cloud prompts | Zero raw names or company names; all replaced before send |
| Crash-free sessions | > 95% |

---

## Out of Scope (MVD)

- SignalScout (pattern detection + role-fit scoring)
- Role-fit scores and archetype comparison screen
- Exploration tracks (shadowing tasks, coffee chat scripts)
- Proactive CommsCoach scripts, advanced tone modes, and 3-variant generation (MVD generates 2 variants on-demand only)
- Manager development conversation preparation
- Growth log export
- Android support
- B2B / employer dashboard
- Web app
- PathArchitect weekly plan adaptation (initial generation only for MVD)
- In-app feedback / rating prompt

---

## Dependencies

| Dependency | Owner | Status |
|---|---|---|
| Supabase project created | @pmkhambhaita | Not started |
| Antigravity project set up | @pmkhambhaita | Not started |
| Anthropic API key (Claude) | @pmkhambhaita | Not started |
| Google Cloud project + Gemini API key | @pmkhambhaita | Not started |
| Cloud Run service configured | @pmkhambhaita | Not started |
| Apple Developer account (for TestFlight) | @pmkhambhaita | Not started |

---

## Open Questions

| # | Question | Priority | Status |
|---|---|---|---|
| OQ-01 | Should the archetype picker allow "I don't know" as an option, and if so, how does PathArchitect handle it? | High | Open |
| OQ-02 | What is the minimum number of check-ins before ExperienceCoach has enough data to surface meaningful patterns? | High | Open |
| OQ-03 | Should free-text check-in content be stored in Supabase at all in MVD, or only on-device? | High | Open |
| OQ-04 | Does the plan show all 90 days of goals up front, or only the current phase? | Medium | Open |
| OQ-05 | What happens when a user joins First90 more than 30 days into their role? | Medium | Open |
| OQ-06 | Should MVD support multiple concurrent active roles? | Low | Likely no — defer |
