# First90

> Your AI mentor for the first 90 days. Figure out your role. Build your confidence. Find where you really fit.

First90 is an agentic AI-powered app for early-career professionals — graduates, apprentices, and new hires — navigating their first 90 days at a company.

It is **not** a generic career coach, CV tool, or job-search assistant. It is scoped entirely to one moment: starting a new job and not knowing where you stand or where you fit.

---

## Who it's for

- Graduates in their first corporate role
- Degree and higher apprentices joining a company
- New hires in their first 1–3 professional roles (aged ~18–28)
- Anyone starting a new role internally (lateral move, promotion, department transfer)

---

## The Three Phases

| Phase | Days | Theme |
|---|---|---|
| **Orient & Observe** | 1–30 | Understand the team, map energy, start a strengths profile |
| **Experiment & Explore** | 31–60 | Try stretch tasks, shadow other roles, have exploratory coffee chats |
| **Decide & Position** | 61–90 | Compare role fit, prep the manager development conversation |

---

## The Four AI Agents

| Agent | Job |
|---|---|
| **PathArchitect** | Builds and weekly-adapts the personalised 30-60-90 plan |
| **ExperienceCoach** | Runs daily check-ins, generates micro-actions |
| **CommsCoach** | Rewrites messages, drafts scripts for tough conversations |
| **SignalScout** | Background agent — spots patterns, triggers others, tracks role-fit scores |

All agents run via **Antigravity** (Google, Gemini-native). All outputs are proposals — the user always decides.

---

## Tech Stack

| Layer | Choice |
|---|---|
| Mobile app | Expo React Native (iOS first) |
| Backend | Node/TypeScript BFF on Cloud Run |
| Database & Auth | Supabase (Postgres + Auth + pgvector) |
| Agent orchestration | Antigravity |
| LLMs | Claude (complex tasks) + Gemini (fast/frequent) |
| On-device AI | Quantised 2–4B SLM via ExecuTorch (privacy layer) |

---

## Repo Structure

```
first90/
├── README.md
├── CONTEXT.md                  ← Full AI assistant context file
├── 01-product/
│   ├── overview.md
│   ├── validation.md
│   ├── mvd.md
│   └── prd.md
├── 02-architecture/
│   ├── system-context.md
│   ├── containers.md
│   ├── agent-flows.md
│   ├── data-model.md
│   └── privacy.md
├── 03-agents/
│   ├── path-architect.md
│   ├── experience-coach.md
│   ├── comms-coach.md
│   └── signal-scout.md
├── 04-design/
│   ├── phases.md
│   ├── role-archetypes.md
│   └── ux-flows.md
├── 05-research/
│   ├── pain-points.md
│   └── competitors.md
├── 06-tasks/
│   └── backlog.md
├── supabase/
│   └── migrations/
├── app/                        ← Expo app (scaffolded later)
└── api/                        ← BFF API (scaffolded later)
```

---

## Getting Started

> Development has not started yet. This repo currently contains product and architecture documentation.

When development begins:

```bash
# Clone
git clone https://github.com/pmkhambhaita/first90.git
cd first90

# App
cd app
npx expo install
npx expo start

# API
cd api
npm install
npm run dev
```

---

## Studio

First90 is the flagship product of **KatalYst** — a micro-SaaS studio building AI tools for early professionals.

---

## Commit convention

```
feat:    new feature
fix:     bug fix
docs:    documentation only
chore:   tooling, config, deps
agent:   agent prompt or flow change
```
