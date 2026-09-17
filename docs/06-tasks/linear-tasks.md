# Linear Tasks — First90 MVD

> Import or copy these into your Linear workspace. Suggested setup: one project ("First90 MVD"), one label per epic below. Assignable to a collaborator per section.

---

## Epic: Setup & Infrastructure

**Label:** `infra` | **Priority:** Urgent (blocks everything)

---

**[S1] Create Supabase project**
Set region to EU (`eu-west-2`). Copy the project URL and anon key into `.env.local` (gitignored). Add placeholder variable names to `.env.example`. Service role key goes into Supabase Edge Function secrets, not `.env`.
- Acceptance: Supabase dashboard shows project in EU region. `.env.local` has `EXPO_PUBLIC_SUPABASE_URL` and `EXPO_PUBLIC_SUPABASE_ANON_KEY` set.

---

**[S2] Write and run initial DB migration**
Create all core tables: `users`, `roles`, `plans`, `plan_phases`, `goals`, `check_ins`, `events`, `suggestions`, `role_archetypes`, `role_fit_scores`, `exploration_activities`.
Full DDL in `docs/02-architecture/data-model.md`.
- Acceptance: All tables exist in Supabase; schema matches the data model doc.

---

**[S3] Enable RLS on all tables**
Apply `user_id = auth.uid()` policy on SELECT, INSERT, UPDATE, DELETE for all user-owned tables. `role_archetypes` is public read. `events` is insert-only from the service role.
See `docs/02-architecture/data-model.md` — RLS section.
- Acceptance: A test user cannot read another user's rows. `role_archetypes` is readable without auth.

---

**[S4] Configure Supabase Auth (magic link + Google OAuth)**
Enable magic link email sign-in. Enable Google OAuth provider. Configure iOS deep link callback for Expo (`exp://` scheme for dev, custom scheme for production).
- Acceptance: Magic link flow completes end-to-end in Expo Go. Google OAuth flow returns to app.

---

**[S5] Scaffold Expo app (`app/`)**
Expo SDK 54+, Expo Router, TypeScript, bottom tab navigation (Today | Plan | Coach | Stats | Inbox). No UI libraries — custom components only.
- Acceptance: `npx expo start` runs, bottom tabs render, TypeScript compiles with no errors.

---

**[S6] Scaffold Supabase Edge Functions (`supabase/functions/`)**
Deno + TypeScript. Create `_shared/supabase.ts` (service role client), `_shared/llm.ts` (Groq → OpenRouter → Gemini waterfall), `_shared/pii.ts` (PII stripper). One stub function per agent endpoint. Free tier: 500k invocations/month. No Docker needed.
- Acceptance: `supabase functions serve` runs locally; stub function returns 200.

---

**[S7] Configure Edge Function secrets (Groq, OpenRouter, Gemini)**
Set API keys as Supabase secrets (never in code): `supabase secrets set GROQ_API_KEY=... OPENROUTER_API_KEY=... GEMINI_API_KEY=...`. Verify secrets are accessible from a deployed function.
- Acceptance: A deployed Edge Function reads `Deno.env.get('GROQ_API_KEY')` and makes a successful Groq API call.

---

**[S8] Configure Edge Function deploy via GitHub Actions**
GitHub Actions workflow: on push to `main`, run `supabase functions deploy` for all functions. No Docker/Artifact Registry needed.
- Acceptance: A push to main deploys all Edge Functions. Functions are visible in Supabase dashboard.

---

## Epic: Auth & Onboarding

**Label:** `auth` | **Priority:** High

---

**[A1] Magic link sign-in screen**
Email input field. On submit, call `supabase.auth.signInWithOtp({ email })`. Show "Check your email" confirmation state. Handle deep link return and navigate to onboarding or home.
- Acceptance: Magic link email is received; tapping the link opens the app and navigates correctly.

---

**[A2] Google OAuth sign-in**
Use Expo AuthSession. On completion, call `supabase.auth.signInWithIdToken`. Navigate to onboarding or home.
- Acceptance: Google OAuth flow completes on physical iOS device; session is established.

---

**[A3] Role setup screen**
Fields: company name (text), job title (text), start date (date picker), role archetype (component from A4).
CTA: "Build my plan" → POST `/roles` to BFF.
- Acceptance: All fields validate; POST creates the role in Supabase.

---

**[A4] Archetype picker component**
Fetches all rows from `role_archetypes` table. Renders as scrollable card grid (name + short description). Single selection. Passes selected `archetype_id` to parent.
- Acceptance: 8 archetypes display; one can be selected; selection is passed to role setup form.

---

**[A5] Edge Function: POST /roles**
Creates row in `roles`. Inserts `role_setup_complete` event. Triggers PathArchitect v0.1 (inline Groq call via Edge Function — no Antigravity yet). Stores the generated plan as a pending suggestion in `suggestions` (`suggestion_type: 'initial_plan'`). Returns `{ suggestion_id }`. The plan is NOT written to `plans`/`goals` until the user accepts the suggestion.
- Acceptance: Role is created in Supabase; a pending `initial_plan` suggestion exists; no plan rows exist yet.

---

**[A6] Plan loading screen + navigate to suggestion review**
Animated loading state while PathArchitect runs ("Building your personalised 30-60-90 day plan…"). When the suggestion is ready, navigate to a plan review screen where the user can inspect and accept the proposed plan. On acceptance, the Edge Function applies the suggestion and navigates to Home/Today screen.
- Acceptance: Loading screen shows during suggestion generation; review screen loads when ready; home screen loads only after user accepts.

---

## Epic: PathArchitect Agent (MVD)

**Label:** `agent-path` | **Priority:** High

---

**[P1] Seed role_archetypes table**
Write a Supabase migration that inserts all 8 archetypes from `docs/04-design/role-archetypes.md` with correct `typical_tasks`, `energising_markers`, `draining_markers`, `skills_developed`, and `transition_paths` values.
- Acceptance: 8 rows in `role_archetypes`; data matches the spec doc exactly.

---

**[P2] Build PathArchitect prompt (initial plan generation)**
Groq `llama-3.3-70b-versatile`, temperature 0.4, max 2000 tokens, JSON output mode. Fallback: Gemini 2.5 Pro if Groq quality is insufficient after testing.
System prompt skeleton is in `docs/03-agents/path-architect.md`.
Inputs: role (PII-stripped company name), archetype template, start date, day number.
Output: JSON with 3 phases × 4–6 goals each, each goal with `title`, `description`, `success_signal`, `due_day`.
- Acceptance: Prompt returns valid JSON matching the `InitialPlanContent` schema. Plan feels personalised to the archetype.

---

**[P3] Store initial plan as suggestion (PathArchitect init)**
Store the plan JSON in `suggestions` (`suggestion_type: 'initial_plan'`). Do NOT write directly to `plans`, `plan_phases`, or `goals`. On user acceptance (via the review screen), the Edge Function applies the suggestion and writes to `plans`, `plan_phases` (3 rows), and `goals` (12–18 rows). Insert `plan_generated` event.
- Acceptance: On PathArchitect run, a pending `initial_plan` suggestion is created. No plan rows exist until the user accepts. Accepted suggestion produces a plan with 3 phases and 12–18 goals.

---

**[P4] Plan screen**
Three collapsible sections: Orient (1–30), Experiment (31–60), Decide (61–90).
Each section lists goals with status badge (`todo` / `in_progress` / `done` / `skipped`).
Tapping a goal opens a detail view with description and success signal.
- Acceptance: All goals load from Supabase; collapsing/expanding phases works; goal status reflects DB state.

---

**[P5] Mark goal as complete**
Tap a goal → detail view → "Mark complete" button → PATCH `/goals/:id` → updates `goals.status = 'done'` in DB.
- Acceptance: Marking a goal complete persists across app restarts.

---

**[P6] Accept/dismiss suggestion flow (Inbox)**
Inbox screen shows all `suggestions` where `status = 'pending'`.
Accept → BFF applies the change to plans/goals. Dismiss → sets `suggestions.status = 'dismissed'`.
- Acceptance: Accepting a `plan_update` suggestion updates the goal; dismissing removes it from inbox. Neither action is reversible from the app.

---

## Epic: ExperienceCoach Agent (MVD)

**Label:** `agent-coach` | **Priority:** High

---

**[E1] Build ExperienceCoach prompt (micro-action generation)**
On-device Gemma (ExecuTorch) primary. Temperature 0.6, max 400 tokens, JSON output mode.
System prompt skeleton is in `docs/03-agents/experience-coach.md`.
Inputs: confidence score, energy score, tags, free text (if provided), 7-day history summary, active goals, day number.
Output: JSON with `insight` (1 sentence), `micro_actions` (1–3 items), optional `tone_note`.
Decision tree in spec doc must be followed for action selection.
- Acceptance: Prompt returns valid JSON. Actions start with a verb and are completable in 1–2 days.

---

**[E2] Edge Function: POST /agents/experience-coach/run**
Called after check-in is saved if on-device inference fails or model not yet downloaded. Loads last 7 days of check-ins + active goals from Supabase. Calls Groq `gemma2-9b-it` (free). Falls back to OpenRouter free model, then static graceful response. Writes result to `suggestions` table as `micro_actions`. Returns `{ suggestion_id }`.
- Acceptance: Micro-actions appear in suggestions table within 10 seconds of check-in submission.

---

**[E3] Check-in screen**
4-step flow (each step on its own card):
1. Confidence slider (1–10) — "How confident did you feel today?"
2. Energy slider (1–10) — "How energising was today's work?"
3. Free text (optional, max 500 chars) — "Anything specific on your mind?"
4. Tag picker — multi-select up to 3 tags (see tag vocabulary in AGENTS.md §10)
CTA: "Submit" → POST `/check-ins`.
- Acceptance: All 4 steps complete; form data posts to BFF; navigates to micro-actions result.

---

**[E4] Tag picker component**
Two sections: Feelings/Context and Activities. Chips/pill buttons, multi-select up to 3. Selected state is visually distinct.
Full tag list in `docs/01-product/prd.md` (FR-CHECKIN-02) and in AGENTS.md §10.
- Acceptance: All tags render in correct categories; max 3 selection enforced; selected tags passed to parent.

---

**[E5] BFF endpoint: POST /check-ins**
Validates JWT. Enforces one check-in per calendar day (reject if one already exists for today's date). Saves to `check_ins`. Inserts `check_in_submitted` event. Triggers ExperienceCoach. Returns micro-actions suggestion_id.
- Acceptance: Second check-in on same day is rejected with 409. Check-in persists in DB. ExperienceCoach runs.

---

**[E6] Micro-actions result screen**
"Here's what I'd suggest for tomorrow:" heading.
1–3 action cards, each with title and 1-line rationale.
Accept / dismiss per action. Accepted actions appear on Today screen tomorrow.
- Acceptance: Actions load from suggestion; accept/dismiss correctly updates suggestions table.

---

**[E7] Push notification scheduling**
Expo Notifications. Evening check-in reminder at 8pm user local time (configurable). Notification deep-links to check-in screen.
- Acceptance: Notification arrives at configured time on a physical iOS device; tapping it opens the check-in screen.

---

## Epic: CommsCoach (MVD)

**Label:** `agent-comms` | **Priority:** Medium

---

**[C1] CommsCoach screen**
Multiline message input (paste area). Recipient type selector (manager / peer / skip-level / external). Goal of message (short text input). CTA: "Get rewrites". Results view below showing 2 variants.
- Acceptance: Screen renders; inputs are usable; CTA triggers rewrite request.

---

**[C2] Build CommsCoach prompt (on-demand variant generation)**
Provider plan: on-device Gemma for short drafts (<200 words); Groq `llama-3.3-70b-versatile` for longer rewrites (free cloud); OpenRouter free model if Groq rate-limited; Gemini 2.5 Pro last resort. Temperature 0.5, JSON output mode.
Prompt skeleton is in `docs/03-agents/comms-coach.md`.
Inputs: PII-stripped draft message, recipient type, message goal, current plan phase.
Output: JSON with 2 variants — each with `label`, `rewritten_message`, `changes: string[]`.
- Acceptance: Prompt returns 2 meaningful variants with bullet-point explanations of changes.

---

**[C3] Edge Function: POST /agents/comms-coach/rewrite**
Apply PII stripping to message text. Load user's current plan phase. Route short drafts (<200 words) to on-device Gemma; longer drafts call Groq via Edge Function (Groq → OpenRouter → Gemini waterfall). Write result to `suggestions`. Return `{ suggestion_id }`.
Privacy rule: messages < 200 words → skip consent (MVD simplification, flag for post-MVD). Log stripped version only.
- Acceptance: Variants returned within 15 seconds. PII stripping removes company/manager names before any cloud provider call. Stripped prompt is logged, not original.

---

**[C4] Variants display component**
Two cards side by side (or stacked on narrow screens). Each shows: variant label, full rewritten message, bulleted list of changes, copy-to-clipboard button.
- Acceptance: Both variants render; copy button copies full message text to clipboard.

---

## Epic: Confidence Tracker

**Label:** `stats` | **Priority:** Medium

---

**[T1] BFF endpoint: GET /check-ins**
Returns check-in history for a role, ordered by date desc. Accepts `?role_id=` query param. Returns `confidence_score`, `energy_score`, `date`, `tags` per entry (no free text).
- Acceptance: Returns correct check-ins for the authenticated user's role. Another user's check-ins are not returned.

---

**[T2] Confidence chart component**
Line chart of confidence scores over last 30 days. Use Victory Native or react-native-chart-kit. X axis: date. Y axis: 1–10. Show current 7-day average as a dashed reference line.
- Acceptance: Chart renders with real check-in data; updates when new check-in is submitted.

---

**[T3] Stats screen**
Shows: confidence chart (T2), current streak (consecutive days with a check-in), 7-day average confidence score.
- Acceptance: Streak is correctly calculated; all values update after a new check-in.

---

## Epic: Navigation & Shell

**Label:** `shell` | **Priority:** High (needed throughout)

---

**[N1] Bottom tab navigator**
Tabs: Today | Plan | Coach | Stats | Inbox. Expo Router file-based tabs. Active tab indicator. Icons for each tab.
- Acceptance: All 5 tabs navigate to their screens; active state is visible.

---

**[N2] Header component**
Shows: day counter ("Day 12 of 90"), phase badge ("Orient" / "Experiment" / "Decide"). Reusable across screens.
- Acceptance: Day counter is correctly computed from `roles.start_date`. Phase badge reflects current phase.

---

**[N3] Suggestions inbox screen**
Lists all `suggestions` with `status = 'pending'` for the current role. Each card: agent name, suggestion type, summary text, accept and dismiss actions.
Empty state: "No suggestions right now. Check back after your next check-in."
- Acceptance: Inbox shows pending suggestions; accept/dismiss update suggestion status.

---

**[N4] Empty states for all main screens**
No-goals state (Plan screen), no-check-ins state (Stats screen), no-suggestions state (Inbox screen). Each with a short explanation and a relevant CTA.
- Acceptance: Empty states render correctly for a new user with no data.

---

**[N5] Loading skeletons**
Skeleton placeholder components for Plan screen, check-in history, and suggestions list while data loads.
- Acceptance: Skeletons render during data fetch; they disappear cleanly when data loads.

---

## Epic: Privacy

**Label:** `privacy` | **Priority:** High (required before any user testing)

---

**[PR1] PII stripping utility**
Function: `stripPII(text: string, context: PIIContext): string`.
Replaces: manager/colleague names, company name, email addresses, phone numbers.
Uses exact match for known values (company_name from `roles`), regex for emails/phones, and optionally a lightweight NER step for other names.
Logs the stripped output (not the original) for audit.
Spec: `docs/02-architecture/privacy.md`.
- Acceptance: Known PII is replaced with typed placeholders. Original text is not logged. Unit tests cover all replacement rule types.

---

**[PR2] Consent prompt before cloud LLM call (free text)**
Modal component: shows the stripped version of what will be sent, asks "Is that OK?" with Cancel / Send buttons.
Triggered by CommsCoach for messages > 200 words, and by check-in submission in Standard mode if free text is provided.
- Acceptance: Modal shows stripped content (not original). Cancel aborts the cloud call. Send proceeds.

---

**[PR3] Privacy settings screen (stub)**
Toggle: Private mode / Standard mode.
Data deletion CTA: "Delete all my data" (shows a confirmation alert — does not auto-delete in MVD stub).
- Acceptance: Toggle persists mode preference to Supabase `users.preferences`. Deletion alert shows.

---

## Epic: Testing & QA

**Label:** `qa` | **Priority:** High (gates MVD release)

---

**[Q1] E2E test: sign up → plan generated**
Manual walkthrough on a physical iOS device. Sign up with magic link, complete role setup, verify plan generates within 30 seconds, verify 3 phases and 12+ goals.
- Acceptance: Full happy path works on device with real Supabase data and a real Groq LLM call.

---

**[Q2] E2E test: check-in → micro-actions**
Manual walkthrough. Complete check-in with scores, tags, and optional free text. Verify micro-actions appear within 10 seconds. Accept one action. Verify it appears on Today screen.
- Acceptance: Full happy path works end-to-end on device.

---

**[Q3] E2E test: CommsCoach rewrite**
Manual walkthrough. Paste a message, select recipient and goal, trigger rewrite. Verify 2 variants are returned with change explanations. Copy one variant to clipboard.
- Acceptance: Full happy path works. Clipboard contains the correct text.

---

**[Q4] RLS security audit**
Create two test users in Supabase. Verify that User A cannot read User B's check-ins, plans, goals, or suggestions via direct Supabase calls (bypassing the BFF). Verify that `role_archetypes` is readable by both.
- Acceptance: No cross-user data leakage confirmed. All queries return only the authenticated user's data.

---

**[Q5] Physical device test**
Install via Expo Go or TestFlight build on a real iOS device. Confirm: cold launch < 3 seconds, push notifications work, no crashes during the core loop.
- Acceptance: 95%+ crash-free sessions over a 30-minute test session.

---

## Suggested Linear Setup

```
Project: First90 MVD
Epics (Labels): infra · auth · agent-path · agent-coach · agent-comms · stats · shell · privacy · qa
Statuses: Backlog → Todo → In Progress → In Review → Done
Priorities: Urgent (infra, blocking) → High → Medium
```

Owner handles: project setup, API keys (Groq, OpenRouter, Gemini), Apple Developer account, Expo account.
Collaborator can own: any individual task above — they are written to be self-contained with links to the relevant spec doc.
