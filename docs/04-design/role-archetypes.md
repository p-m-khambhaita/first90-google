# Role Archetypes Library

This document defines the role archetype library used by PathArchitect (to template initial plans), SignalScout (to score fit), and the Explore screen (to surface relevant exploration activities).

Archetypes are stored in the `role_archetypes` table (public read, no `user_id`). They represent **internal corporate role families** — not job titles. A user hired as a "Graduate Associate" may discover they are a natural fit for the Data & Analytics archetype, even if that is not what they were hired to do.

---

## Archetype Structure

Each archetype contains:
- **Name and description** — plain-language explanation of what this role family is about
- **Typical tasks** — concrete examples of day-to-day work
- **Energising markers** — task tags that, when rated high-energy by the user, increase fit score
- **Draining markers** — task tags that, when rated low-energy, decrease fit score
- **Skills developed** — what someone in this archetype builds over time
- **Transition paths** — adjacent archetypes this role commonly moves into
- **Exploration activity suggestions** — specific things a user can do to test fit

---

## Archetype 1 — Data & Analytics

**Description:** Turning raw data into decisions. This role archetype suits people who enjoy finding patterns, working with structured information, and communicating what numbers actually mean.

**Typical tasks:** Building dashboards, writing SQL queries, cleaning datasets, presenting data insights to stakeholders, writing data quality reports, supporting business decisions with evidence.

**Energising markers:** `data_analysis`

**Draining markers:** `client_calls`, `writing`, `people_interaction`

**Skills developed:** SQL and data tooling, statistical reasoning, data storytelling, stakeholder communication, attention to detail.

**Transition paths:** Product & Strategy, Operations & Process, Tech & Engineering

**Exploration activities:**
- Shadow a data analyst for a morning
- Ask to attend a data team standup
- Volunteer to pull a simple dataset for an ongoing project
- Coffee chat with someone in the data team

---

## Archetype 2 — Product & Strategy

**Description:** Shaping what gets built and why. This archetype suits people who enjoy thinking about user problems, synthesising information from multiple sources, and influencing direction without always having direct authority.

**Typical tasks:** Writing specs or briefs, running discovery research, facilitating workshops, synthesising customer feedback, building business cases, coordinating cross-functional teams, roadmap planning.

**Energising markers:** `strategy`

**Draining markers:** `process_improvement`, `financial_analysis`

**Skills developed:** Product thinking, stakeholder management, written communication, research methods, prioritisation frameworks.

**Transition paths:** Data & Analytics, Client & Account Management, People & Culture

**Exploration activities:**
- Shadow a product manager during a sprint planning or review session
- Ask to sit in on a user research call
- Read a product spec or PRD and write a one-page reaction
- Coffee chat with someone in product or strategy

---

## Archetype 3 — Tech & Engineering

**Description:** Building the things. This archetype suits people who enjoy working in code, solving technical problems with concrete solutions, and seeing the direct output of their work.

**Typical tasks:** Writing and reviewing code, debugging, building features, writing technical documentation, code reviews, participating in standups and retrospectives, maintaining infrastructure.

**Energising markers:** `coding`

**Draining markers:** `client_calls`, `people_interaction`

**Skills developed:** Software engineering, system design, testing, collaboration in engineering teams, technical communication.

**Transition paths:** Data & Analytics, Product & Strategy, Operations & Process

**Exploration activities:**
- Ask to pair-program with an engineer on a small task
- Attend a technical architecture review or design discussion
- Review an open pull request and write up your observations
- Coffee chat with a senior engineer

---

## Archetype 4 — Client & Account Management

**Description:** Being the bridge between the company and its clients. This archetype suits people who are energised by relationships, enjoy variety, and are comfortable with ambiguity and people-driven problem solving.

**Typical tasks:** Client calls and meetings, writing status updates and QBRs, managing client expectations, coordinating internal teams to deliver client outcomes, upsell conversations, contract renewals.

**Energising markers:** `client_calls`

**Draining markers:** `coding`, `financial_analysis`

**Skills developed:** Relationship management, commercial awareness, written and verbal communication, project coordination, resilience.

**Transition paths:** Product & Strategy, People & Culture, Operations & Process

**Exploration activities:**
- Sit in on a client call (observer role)
- Help prepare a client status update or deck
- Coffee chat with an account manager or CSM
- Shadow a QBR or renewal conversation

---

## Archetype 5 — Marketing & Content

**Description:** Building the narrative. This archetype suits people who enjoy creative work, writing, and understanding how messages land with different audiences.

**Typical tasks:** Writing copy, building campaigns, managing social media, creating content (blog posts, case studies, videos), analysing campaign performance, briefing designers, managing editorial calendars.

**Energising markers:** `writing`

**Draining markers:** `financial_analysis`, `coding`

**Skills developed:** Copywriting and editing, campaign management, SEO/content strategy, audience analysis, briefing and collaborating with creatives.

**Transition paths:** Product & Strategy, Client & Account Management, People & Culture

**Exploration activities:**
- Offer to draft a social post or internal newsletter section
- Shadow a campaign planning meeting
- Review the company's recent marketing output and write a short critique
- Coffee chat with a content or brand team member

---

## Archetype 6 — Operations & Process

**Description:** Making the machine run smoothly. This archetype suits people who are energised by efficiency, enjoy finding and fixing broken processes, and are comfortable with detail and systems.

**Typical tasks:** Process mapping, writing SOPs, coordinating logistics, managing tooling and systems, reporting on operational metrics, identifying bottlenecks, project management support.

**Energising markers:** `process_improvement`

**Draining markers:** `strategy`, `client_calls`

**Skills developed:** Process design, project coordination, operational reporting, systems thinking, stakeholder coordination.

**Transition paths:** Data & Analytics, Product & Strategy, Tech & Engineering

**Exploration activities:**
- Ask to join an ops or project management standup
- Shadow a process improvement or tooling project
- Map out a current process you interact with and identify one improvement
- Coffee chat with an operations or project manager

---

## Archetype 7 — People & Culture

**Description:** Taking care of the humans. This archetype suits people who are energised by conversations about people, development, fairness, and organisational health.

**Typical tasks:** Supporting recruitment, onboarding, learning & development, employee engagement surveys, HR admin, policy documentation, wellbeing initiatives, culture projects.

**Energising markers:** `people_interaction`

**Draining markers:** `coding`, `financial_analysis`

**Skills developed:** Interpersonal communication, HR processes, facilitation, empathy-led problem solving, project coordination.

**Transition paths:** Client & Account Management, Product & Strategy, Marketing & Content

**Exploration activities:**
- Ask to help with an onboarding session or buddy programme
- Shadow an L&D or HR business partner for a day
- Contribute to a culture or engagement initiative
- Coffee chat with someone in the People team

---

## Archetype 8 — Finance & Commercial

**Description:** Keeping score and driving commercial decisions. This archetype suits people who are energised by numbers, commercial thinking, and helping the business understand its financial performance.

**Typical tasks:** Financial modelling, budgeting, forecasting, variance analysis, management reporting, business partnering, pricing analysis, commercial review presentations.

**Energising markers:** `financial_analysis`

**Draining markers:** `writing`, `people_interaction`

**Skills developed:** Financial modelling, Excel/Google Sheets, commercial awareness, business partnering, stakeholder communication.

**Transition paths:** Data & Analytics, Product & Strategy, Operations & Process

**Exploration activities:**
- Ask to sit in on a budget review or financial planning meeting
- Shadow a finance business partner
- Review the company's last set of management accounts (if available)
- Coffee chat with someone in finance or commercial

---

## Notes for PathArchitect

When generating an initial plan, PathArchitect should:
1. Ask the user to select their current job title and describe their role in 1–2 sentences.
2. Map the described role to the **closest matching archetype** — this becomes the "hired archetype".
3. Use the hired archetype's typical tasks and energising markers to populate phase 1 goals.
4. In phase 2, introduce exploration tasks from 1–2 **adjacent archetypes** (from the transition paths list) to begin testing fit.
5. Never lock the user into their hired archetype — the goal is discovery, not confirmation.

## Notes for SignalScout

- The `energising_markers` and `draining_markers` arrays in this document are the source of truth for fit-score computation.
- When a new archetype is added to this library, `role_archetypes` table must be updated and SignalScout's scoring logic re-validated.
- Archetypes are not mutually exclusive. A user can show strong signals for two archetypes simultaneously.
