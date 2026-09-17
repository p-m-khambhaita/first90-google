# AGENTS.md — First90 AI Agent Guide

> This file is the primary context document for any AI coding agent (Claude Code, Cursor, Copilot, etc.) working in this repository. Read it before writing any code.

---

## 1. Project Vision

**First90** is an agentic AI app for early-career professionals — graduates, degree apprentices, and new hires — navigating their first 90 days in a new role.

**Core promise:** *"Your AI mentor for the first 90 days. Figure out your role. Build your confidence. Find where you really fit."*

It is a **tightly scoped** product. It is not a career coach, CV builder, or job-search tool. It covers exactly one moment: starting a new job and not knowing where you stand. Do not introduce features that expand beyond this scope.

First90 is the flagship product of **KatalYst**, a solo-founder micro-SaaS studio. Keep implementations lean, practical, and shippable. No over-engineering.

---

## 2. Tech Stack

| Layer | Choice | Notes |
|---|---|---|
| Mobile app | Expo SDK 54+ / React Native | iOS first, Android second, Web is a stretch goal |
| Navigation | Expo Router (file-based) | Native tabs |
| State management | Zustand or React Query | TBD — decide at scaffold time |
| UI | Custom design system | No Expo UI, RN Paper, or UI libraries |
| Push notifications | Expo Notifications | FCM + APNs |
| Backend | Supabase Edge Functions (Deno + TypeScript) | BFF pattern — free tier: 500k invocations/month. No Docker, no separate infra. |
| Database | Supabase (Postgres) | EU region |
| Auth | Supabase Auth | Magic link + Google OAuth |
| Storage | Supabase Storage | Growth log exports, future file uploads |
| Vector search | pgvector (optional) | Semantic search over check-ins |
| Agent orchestration | Antigravity (Google, Gemini-native) | Not used in MVD — direct API calls only |
| LLM: on-device | Gemma 3 (2–4B INT4 via ExecuTorch) | Primary for ExperienceCoach. Free, private, no API key. |
| LLM: free cloud | **Groq** (primary free cloud option) | Free tier: ~14k req/day. Models: `llama-3.3-70b-versatile`, `gemma2-9b-it`. Use for all cloud fallback before spending money. |
| LLM: model router | **OpenRouter** (optional) | Aggregates 200+ models. Free models available (`:free` suffix). Good for swapping providers without code changes. |
| LLM: paid cloud | Gemini 2.5 Pro (Google) | Only if Groq quality insufficient for PathArchitect. Pay-per-token. Default to Groq first. |

---

## 3. Repository Structure

```
first90/
├── AGENTS.md                   ← you are here
├── README.md
├── CONTEXT.md                  ← full project context (drop into AI assistants)
├── docs/
│   ├── 01-product/             ← overview, validation, mvd, prd
│   ├── 02-architecture/        ← C4 diagrams, agent flows, data model, privacy
│   ├── 03-agents/              ← per-agent specs (prompts, inputs, outputs, LLM choice)
│   ├── 04-design/              ← phases, role archetypes, UX flows
│   ├── 05-research/            ← pain points, competitors
│   └── 06-tasks/               ← backlog.md
├── supabase/
│   ├── migrations/             ← DB migrations
│   └── functions/              ← Edge Functions (one function per agent endpoint)
│       ├── _shared/            ← shared helpers (llm.ts, pii.ts, supabase.ts)
│       ├── agents-path-architect/
│       ├── agents-experience-coach/
│       └── agents-comms-coach/
└── app/                        ← Expo React Native app (not yet scaffolded)
```

Agent prompt specs live in `docs/03-agents/`. When you modify an agent prompt or flow, update the spec file in that directory as part of the same commit using the `agent:` commit prefix.

---

## 4. Architecture Rules (Non-Negotiable)

### 4.1 Edge Function–only cloud LLM access
The mobile app **never** calls any cloud LLM API directly. All **cloud** LLM calls (Groq, OpenRouter, Gemini) go through Supabase Edge Functions (`supabase/functions/`). This is a security and PII-control boundary — API keys stay server-side, do not break it.

**Exception:** On-device Gemma (ExperienceCoach primary, short CommsCoach drafts) runs entirely in the app via ExecuTorch — no Edge Function involved. This is by design: on-device inference requires no API key and no network call.

### 4.2 Supervised autonomy — suggestions table
Agents **never** write directly to `plans`, `goals`, or other user-facing state tables. Every agent output goes to the `suggestions` table first. The user accepts or dismisses from the app. On acceptance, the BFF applies the change. This is the core trust contract with the user.

```
Agent output → suggestions table → user accepts → BFF applies → plans/goals updated
```

Never skip this flow. Not even for "small" changes.

### 4.3 PII stripping before every cloud LLM call
Before any user-provided free text reaches any cloud LLM (Groq, OpenRouter, Gemini), the Edge Function must apply PII stripping. See `docs/02-architecture/privacy.md` for replacement rules. The stripped version is what gets logged, not the original.

```typescript
// Required before any cloud LLM call involving user text
const strippedText = stripPII(userText, { companyName, managerName });
```

### 4.4 RLS on all Supabase tables
Every user-data table must have Row Level Security enabled with `user_id = auth.uid()` policy. The `role_archetypes` table is public read (no user_id). The `events` table is insert-only from the backend service role; users read their own.

### 4.5 Stateless Edge Functions
Each Edge Function invocation is stateless. All state lives in Supabase DB. Do not use module-level mutable variables — they are not guaranteed to persist between invocations.

---

## 5. The Four AI Agents

Each agent has a full spec in `docs/03-agents/`. Key facts:

### PathArchitect
- **Model:** Groq `llama-3.3-70b-versatile` (free, try first) → Gemini 2.5 Pro (paid fallback if plan quality insufficient), temperature 0.4, max 2000 tokens (initial) / 800 (update)
- **When it runs:** Role setup (initial plan), weekly cron (Sunday evening), SignalScout slippage escalation, user-initiated re-plan
- **Output type:** `suggestions.suggestion_type = 'initial_plan'` or `'plan_update'`
- **Critical:** Initial plan generation is the most important LLM call in the product. Plan quality at this step sets the tone for 90 days. Do not cut corners on the prompt or temperature.
- **Never:** Propose more than 2 goal additions or 2 goal removals in a single weekly review.

### ExperienceCoach
- **Model:** Gemma edge (on-device, quantised 2–4B), temperature 0.6, max 400 tokens
- **When it runs:** After every check-in submission
- **Output type:** `suggestions.suggestion_type = 'micro_actions'`
- **Fallback chain:** On-device Gemma → Groq `gemma2-9b-it` (free) → static graceful response and log. Never pay for ExperienceCoach cloud calls if avoidable.
- **Critical:** Speed matters and privacy matters — run on-device by default. Never route check-in free text to a cloud LLM without PII stripping + user consent.
- **Tone:** Warm, peer-level, practical. Not a therapist, not a corporate wellness bot.

### CommsCoach
- **Model:** Gemma edge for short drafts (<200 words); Groq `llama-3.3-70b-versatile` for full rewrites (free cloud, consent prompt required); Gemini 2.5 Pro only if Groq hits rate limits
- **When it runs:** On-demand (user pastes a draft), or proactively when SignalScout detects a tough-conversation task is due
- **Output type:** `suggestions.suggestion_type = 'comms_variants'`
- **MVD scope:** 2 variants on-demand. Proactive scripts and 3-variant generation are v0.2+.
- **Privacy rule:** Messages under 200 words → on-device. Longer messages → show consent prompt before cloud call.

### SignalScout
- **Model:** Rule-based (no LLM) for daily/post-check-in runs. Weekly plain-language digest is a cloud call via Edge Function to Groq `gemma2-9b-it` (free); static text fallback if Groq fails.
- **When it runs:** Daily lightweight, weekly full analysis, after every check-in (triggered by ExperienceCoach signal summary)
- **Output:** Triggers other agents; updates `role_fit_scores`; does not write to `suggestions` directly
- **MVD status:** Deferred to v0.2. Do not implement in MVD.

---

## 6. Data Model

Full DDL is in `docs/02-architecture/data-model.md`. Key tables:

| Table | Purpose |
|---|---|
| `users` | User identity and preferences |
| `roles` | A job/role the user holds. One active role at a time in MVD. |
| `plans` | The 30-60-90 plan for a role |
| `plan_phases` | Orient (1–30), Experiment (31–60), Decide (61–90) |
| `goals` | Goals within a phase. Has `status`, `due_day`, `success_signal`. |
| `check_ins` | Daily check-in: confidence (1–10), energy (1–10), free_text, tags |
| `events` | Append-only audit log. Never update or delete. |
| `suggestions` | All agent outputs awaiting user acceptance. Central to the architecture. |
| `role_archetypes` | Library of 8 role archetypes. Public read. Seeded via migration. |
| `role_fit_scores` | Per-user, per-archetype fit score. Updated by SignalScout. |
| `exploration_activities` | Exploration tasks tied to candidate archetypes. |

### Key Supabase patterns

```typescript
// Always use service role key in BFF for agent writes to suggestions
const supabaseAdmin = createClient(url, SERVICE_ROLE_KEY);

// Always use anon key + JWT validation for user-scoped requests
const supabaseUser = createClient(url, ANON_KEY, {
  global: { headers: { Authorization: `Bearer ${jwt}` } }
});
```

---

## 7. LLM Usage Decisions

| Task | Primary model | Fallback | Where it runs |
|---|---|---|---|
| Initial 30-60-90 plan generation | Groq `llama-3.3-70b-versatile` | Gemini 2.5 Pro (if quality insufficient) | Cloud (Edge Function) |
| Weekly plan adaptation | Groq `llama-3.3-70b-versatile` | Gemini 2.5 Pro | Cloud (Edge Function) |
| Fit-check reflections (day 45, day 90) | Groq `llama-3.3-70b-versatile` | Gemini 2.5 Pro | Cloud (Edge Function) |
| Check-in micro-actions | Gemma edge (on-device) | Groq `gemma2-9b-it` → OpenRouter free → static | **On-device by default** |
| Short message rewrites (<200 words) | Gemma edge (on-device) | Groq `llama-3.3-70b-versatile` | **On-device by default** |
| Full CommsCoach rewrites (>200 words) | Groq `llama-3.3-70b-versatile` | OpenRouter free → Gemini 2.5 Pro | Cloud (Edge Function), consent prompt required |
| Weekly SignalScout digest | Groq `gemma2-9b-it` | Static text fallback | Cloud (Edge Function) |
| Signal pattern detection (daily) | Rule-based | — | No LLM — pure logic |

**Cost-first decision rule:** Always attempt the free option first. On-device Gemma for short tasks. Groq free tier for cloud tasks. OpenRouter `:free` models if Groq is rate-limited. Only reach for Gemini 2.5 Pro (paid) if Groq plan quality is demonstrably insufficient after testing.

**Model IDs:**
- Groq: `llama-3.3-70b-versatile` (planning/comms), `gemma2-9b-it` (check-in fallback) — free tier, 14k req/day
- OpenRouter: any `:free` model (e.g. `google/gemma-2-9b-it:free`, `meta-llama/llama-3.1-8b-instruct:free`)
- Gemini 2.5 Pro: `gemini-2.5-pro` — paid, last resort only
- Gemma edge: quantised Gemma 3 (2B or 4B INT4) via ExecuTorch — free, fully on-device

Always use **structured output / JSON mode** for all agent calls. Agents return typed JSON schemas — see individual specs in `docs/03-agents/`.

---

## 8. Privacy Rules (Non-Negotiable)

1. **Free text from check-ins never reaches a cloud LLM without PII stripping and (in Standard mode) user opt-in.** This is a product promise, not just a technical detail.
2. **Private mode means on-device only.** If the user has Private mode enabled, ExperienceCoach runs the on-device SLM. No cloud call. Period.
3. **Supabase stores EU region only.** Do not change the region.
4. **No localStorage or sessionStorage.** Use Supabase or in-memory for all state (sandbox limitation).
5. **Consent prompt required before any cloud call involving free text.** Show the user what will be sent (stripped version) and get confirmation.
6. **Log the stripped version, not the original.** The original free text stays on-device or in Supabase encrypted — it never appears in application logs.

See `docs/02-architecture/privacy.md` for the full architecture, on-device model spec, GDPR notes, and data residency table.

---

## 9. API Design Conventions

All BFF endpoints follow REST:

```
POST /roles                          — create role, trigger PathArchitect
GET  /plans/:id                      — fetch plan with phases and goals
POST /check-ins                      — save check-in, trigger ExperienceCoach
GET  /check-ins?role_id=...          — fetch check-in history
GET  /suggestions?status=pending     — fetch pending agent suggestions
PATCH /suggestions/:id               — accept or dismiss a suggestion
POST /agents/path-architect/init     — internal: trigger PathArchitect (MVD)
POST /agents/experience-coach/run    — internal: trigger ExperienceCoach (MVD)
POST /agents/comms-coach/rewrite     — run CommsCoach on-demand
```

Every request validates the Supabase JWT. Extract `user_id` from the JWT payload — never trust a `user_id` from the request body.

---

## 10. Check-in Tag Vocabulary

These are the **only** valid tags. Do not add new ones without updating both the spec and the `FR-CHECKIN-02` requirement.

**Feelings/Context:**
`meetings` · `deadlines` · `relationships` · `workload` · `learning` · `unclear_expectations` · `imposter_syndrome` · `win`

**Activities (role-fit signals for SignalScout):**
`data_analysis` · `strategy` · `coding` · `client_calls` · `writing` · `process_improvement` · `people_interaction` · `financial_analysis`

---

## 11. Role Archetypes (Seed Data)

8 archetypes must be seeded into `role_archetypes` at migration time (see `docs/04-design/role-archetypes.md`):

1. Data & Analytics
2. Product & Strategy
3. Tech & Engineering
4. Client & Account Management
5. Marketing & Content
6. Operations & Process
7. People & Culture
8. Finance & Commercial

Each archetype has `energising_markers` and `draining_markers` arrays — these are the input to SignalScout's fit-score algorithm. Do not change the marker values without also updating SignalScout's scoring logic.

---

## 12. MVD Build Order

Follow this order. Do not start a later step until the previous is confirmed working:

1. **Supabase project + schema** — create project, run migration (all core tables + RLS)
2. **Seed role_archetypes** — 8 archetypes from `docs/04-design/role-archetypes.md`
3. **Supabase Auth** — magic link + Google OAuth; iOS deep link callback for Expo
4. **Expo app scaffold** — Expo SDK 54+, Expo Router, TypeScript, bottom tab nav
5. **Edge Function scaffold** — Supabase Edge Functions (Deno + TypeScript); shared Supabase service role client in `_shared/supabase.ts`; shared LLM wrapper in `_shared/llm.ts`
6. **Auth flow** — sign-in screen → magic link → session → navigate to onboarding
7. **Onboarding flow** — company name, job title, start date, archetype picker
8. **PathArchitect v0.1** — Edge Function calls Groq `llama-3.3-70b-versatile`; writes plan + phases + goals to DB on acceptance
9. **Plan screen** — 3 collapsible phases, goal cards, mark complete
10. **Check-in screen** — confidence slider, energy slider, free text, tag picker
11. **ExperienceCoach v0.1** — on-device Gemma (ExecuTorch) primary; Groq `gemma2-9b-it` cloud fallback via Edge Function; writes micro-actions to suggestions
12. **Micro-actions result screen** — accept/dismiss actions after check-in
13. **CommsCoach v0.1** — short drafts (<200 words) run on-device Gemma; longer drafts via Edge Function calling Groq `llama-3.3-70b-versatile`; 2–3 variants + explanations
14. **CommsCoach screen** — paste message, recipient, goal, show variants, copy button
15. **Today/Home screen** — phase badge, day counter, today's focus, check-in prompt card
16. **Inbox screen** — pending suggestions list, accept/dismiss
17. **Confidence tracker** — line chart of confidence scores, streak, 7-day avg
18. **Push notifications** — evening check-in reminder via Expo Notifications
19. **PII stripping utility** — regex + NER before any cloud LLM call
20. **Consent prompt** — modal before cloud LLM calls involving free text
21. **End-to-end test** — happy path on physical iOS device with real data
22. **RLS audit** — verify cross-user data isolation

---

## 13. Code Conventions

- **TypeScript everywhere.** No plain JS files.
- **Conventional commits:**
  - `feat:` new feature
  - `fix:` bug fix
  - `docs:` documentation only
  - `chore:` tooling, config, deps
  - `agent:` agent prompt or flow change — always include the spec file update
- **No comments unless the WHY is non-obvious.** The code explains what; comments explain hidden constraints or workarounds.
- **No feature flags, no backwards-compat shims.** Just change the code.
- **No direct LLM calls from the Expo app.** Ever.
- **All agent outputs go via suggestions table.** No exceptions.
- **Privacy first:** If a feature processes free text, document where it runs (on-device / cloud) in its spec file.

---

## 14. Open Questions (Unresolved)

Do not make assumptions on these without discussing with the owner first:

| # | Question |
|---|---|
| OQ-01 | Should the archetype picker allow "I don't know" as an option? |
| OQ-02 | Minimum number of check-ins before ExperienceCoach surfaces meaningful patterns? |
| OQ-03 | Should free-text check-in content be stored in Supabase at all in MVD, or only on-device? |
| OQ-04 | Does the plan show all 90 days of goals up front, or only the current phase? |
| OQ-05 | What happens when a user joins First90 more than 30 days into their role? |
| OQ-06 | Should MVD support multiple concurrent active roles? (Likely no.) |

---

## 15. What NOT to Build in MVD

- SignalScout pattern detection or role-fit scoring
- Archetype comparison screen
- Exploration tracks (shadowing tasks, coffee chat scripts)
- Proactive CommsCoach scripts, tone modes, 3-variant generation
- Manager development conversation preparation
- Growth log export (PDF)
- Android support
- B2B / employer dashboard
- Web app
- PathArchitect weekly plan adaptation (initial generation only)
- Full Antigravity orchestration (use direct API calls for MVD)
- On-device SLM (ExecuTorch) privacy layer
- Social login, profile editing, account settings beyond basics
- In-app feedback / rating prompt

---

## 16. Key External Dependencies

| Dependency | Notes |
|---|---|
| Supabase project | EU region (`eu-west-2`); Edge Functions enabled |
| Groq API key | Primary free cloud LLM — PathArchitect, CommsCoach, ExperienceCoach fallback |
| OpenRouter API key | Free model overflow if Groq is rate-limited |
| Gemini API key | Last resort only — PathArchitect if Groq quality insufficient |
| Antigravity | Google agent orchestration — not needed for MVD |
| Apple Developer account | For TestFlight internal testing |
| Expo account | For EAS Build and push notification credentials |

---

## 17. Performance Targets

| Operation | Target |
|---|---|
| Plan generation (PathArchitect) | < 30 seconds |
| Check-in → micro-actions | < 10 seconds end-to-end |
| CommsCoach rewrite (on-device) | < 5 seconds |
| CommsCoach rewrite (cloud) | < 15 seconds |
| App cold launch to home screen | < 3 seconds |
| Crash-free sessions | > 95% |

---

## 18. Useful References

- Full project context: [`CONTEXT.md`](CONTEXT.md)
- Product requirements: [`docs/01-product/prd.md`](docs/01-product/prd.md)
- MVD spec and build order: [`docs/01-product/mvd.md`](docs/01-product/mvd.md)
- Data model DDL + ER diagram: [`docs/02-architecture/data-model.md`](docs/02-architecture/data-model.md)
- Privacy architecture: [`docs/02-architecture/privacy.md`](docs/02-architecture/privacy.md)
- Agent flows (sequence diagrams): [`docs/02-architecture/agent-flows.md`](docs/02-architecture/agent-flows.md)
- PathArchitect spec: [`docs/03-agents/path-architect.md`](docs/03-agents/path-architect.md)
- ExperienceCoach spec: [`docs/03-agents/experience-coach.md`](docs/03-agents/experience-coach.md)
- CommsCoach spec: [`docs/03-agents/comms-coach.md`](docs/03-agents/comms-coach.md)
- SignalScout spec: [`docs/03-agents/signal-scout.md`](docs/03-agents/signal-scout.md)
- Role archetypes library: [`docs/04-design/role-archetypes.md`](docs/04-design/role-archetypes.md)
- Task backlog: [`docs/06-tasks/backlog.md`](docs/06-tasks/backlog.md)
