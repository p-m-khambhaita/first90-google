# MVD Backlog

> Minimum Viable Demo task list. Covers everything needed to get to a working, demo-able version of First90 with PathArchitect + ExperienceCoach running end-to-end.

Status values: `todo` · `in-progress` · `done` · `blocked`

---

## 🏗️ Setup & Infrastructure

| # | Task | Status | Notes |
|---|---|---|---|
| S1 | Create Supabase project | `todo` | Enable pgvector extension |
| S2 | Run initial DB migration (core tables) | `todo` | `users`, `roles`, `plans`, `plan_phases`, `goals`, `check_ins`, `events`, `suggestions` |
| S3 | Configure Supabase Auth (magic link + Google OAuth) | `todo` | iOS deep link callback needed for Expo |
| S4 | Enable RLS on all tables | `todo` | `user_id = auth.uid()` policy; service role key for agents |
| S5 | Scaffold Expo app (`app/`) | `todo` | Expo SDK 54+, Expo Router, TypeScript |
| S6 | Scaffold Supabase Edge Functions (`supabase/functions/`) | `todo` | Deno + TypeScript. Free tier: 500k invocations/month. No Docker needed. |
| S7 | Connect Edge Functions to Supabase (service role client) | `todo` | `SUPABASE_SERVICE_ROLE_KEY` as Supabase secret; shared client in `_shared/` |
| S8 | Configure LLM provider API keys as Supabase secrets + smoke test | `todo` | `supabase secrets set GROQ_API_KEY=... OPENROUTER_API_KEY=... GEMINI_API_KEY=...`; test with Groq primary, OpenRouter overflow, Gemini last resort |
| S9 | Configure Edge Function deploy via GitHub Actions | `todo` | GitHub Actions → `supabase functions deploy`. No Docker/Artifact Registry. |

---

## 🔐 Auth & Onboarding

| # | Task | Status | Notes |
|---|---|---|---|
| A1 | Magic link sign-in screen | `todo` | Email input → Supabase Auth |
| A2 | Google OAuth sign-in | `todo` | Expo AuthSession |
| A3 | Role setup screen | `todo` | Company name, job title, start date, archetype selection |
| A4 | Archetype picker component | `todo` | Fetches `role_archetypes` table; card-based selection |
| A5 | Create role + plan on setup complete | `todo` | POST `/roles` → BFF creates role, triggers PathArchitect |
| A6 | Onboarding complete → navigate to home | `todo` | Mark `roles.is_active = true` |

---

## 🤖 PathArchitect Agent (MVD)

| # | Task | Status | Notes |
|---|---|---|---|
| P1 | Seed `role_archetypes` table with 6 archetypes | `todo` | See `04-design/role-archetypes.md` |
| P2 | Build PathArchitect prompt (initial plan generation) | `todo` | Claude; inputs: role, archetype, start date |
| P3 | BFF endpoint: `POST /agents/path-architect/init` | `todo` | Triggers on role setup; writes plan + phases + goals to DB |
| P4 | Plan view screen (home) | `todo` | Shows current phase, goals list, day counter |
| P5 | Goal card component | `todo` | Title, due day, status, success signal |
| P6 | Accept/dismiss suggestion flow | `todo` | Reads from `suggestions`; user taps accept → applies to `goals` |

---

## 📋 ExperienceCoach Agent (MVD)

| # | Task | Status | Notes |
|---|---|---|---|
| E1 | Build ExperienceCoach prompt (check-in analysis) | `todo` | Gemini Flash; inputs: scores, tags, recent history |
| E2 | BFF endpoint: `POST /agents/experience-coach/run` | `todo` | Called after check-in saved; writes micro-actions to `suggestions` |
| E3 | Check-in screen | `todo` | Confidence slider (1–10), energy slider (1–10), free text, tag picker |
| E4 | Tag picker component | `todo` | Multi-select; predefined tag vocabulary |
| E5 | Save check-in to Supabase | `todo` | `POST /check-ins` → BFF → Supabase |
| E6 | Micro-actions display | `todo` | Show 1–3 suggested actions after check-in |
| E7 | Push notification scheduling | `todo` | Expo Notifications; evening reminder at user-set time |

---

## 💬 CommsCoach (MVD — basic)

| # | Task | Status | Notes |
|---|---|---|---|
| C1 | CommsCoach screen | `todo` | Text input for draft message, recipient context |
| C2 | Build CommsCoach prompt (on-demand variants) | `todo` | Claude; 2 variants for MVD (not 3) |
| C3 | BFF endpoint: `POST /agents/comms-coach/rewrite` | `todo` | Returns variants; saves to `suggestions` |
| C4 | Variants display component | `todo` | Card per variant, label, changes list, copy button |

---

## 📊 Confidence Tracker

| # | Task | Status | Notes |
|---|---|---|---|
| T1 | Confidence chart component | `todo` | Simple line chart of confidence scores over time |
| T2 | Fetch check-in history | `todo` | `GET /check-ins?role_id=...` → BFF → Supabase |
| T3 | Stats screen (basic) | `todo` | Chart + current streak + avg confidence |

---

## 🧭 Navigation & Shell

| # | Task | Status | Notes |
|---|---|---|---|
| N1 | Bottom tab navigator | `todo` | Home (plan), Check-in, CommsCoach, Stats |
| N2 | Header component | `todo` | Day counter ("Day 12 of 90"), phase badge |
| N3 | Suggestions inbox screen | `todo` | List of pending suggestions; accept/dismiss |
| N4 | Empty states for all screens | `todo` | No goals, no check-ins, no suggestions |
| N5 | Loading skeletons | `todo` | Plan screen, check-in history, suggestions |

---

## 🔒 Privacy

| # | Task | Status | Notes |
|---|---|---|---|
| PR1 | PII stripping utility | `todo` | Regex + replacement for names, company names before cloud LLM calls |
| PR2 | Consent prompt before cloud LLM call | `todo` | Modal: "To help with this, we'll send anonymised context to Claude. OK?" |
| PR3 | Privacy settings screen (stub) | `todo` | Toggle for private mode; data deletion CTA |

---

## 🧪 Testing & QA

| # | Task | Status | Notes |
|---|---|---|---|
| Q1 | Manual end-to-end test: sign up → plan generated | `todo` | Happy path |
| Q2 | Manual end-to-end test: check-in → micro-actions | `todo` | Happy path |
| Q3 | Manual end-to-end test: CommsCoach rewrite | `todo` | Happy path |
| Q4 | Test RLS policies (verify cross-user data isolation) | `todo` | Critical security check |
| Q5 | Test on physical iOS device | `todo` | Expo Go or TestFlight build |

---

## 📦 MVD Definition of Done

The MVD is complete when a user can:
1. Sign up, set up their role, and receive a generated 30-60-90 day plan
2. Complete a daily check-in and receive 1–3 micro-actions
3. Paste a message into CommsCoach and receive 2 improved variants
4. View their confidence trend over time
5. Accept or dismiss suggestions from agents

All of the above must work on a physical iOS device with real Supabase data and real LLM calls.
