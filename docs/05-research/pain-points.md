# Pain Points Research

This document captures the validated pain points underpinning First90's product decisions. Each pain point includes the evidence base, implications for the product, and the feature or design decision it informs.

---

## 1. Most early-career workers have no mentor

**Evidence:**
- ~73% of Gen Z workers report they do not have access to a mentor (multiple sources, 2024–2025).
- Mentorship is consistently cited as the #1 factor Gen Z say would help them in their career — above salary increases and training budgets.
- Most organisations have informal mentorship programmes that collapse within 3–6 months due to lack of structure and manager bandwidth.

**User experience:** The new hire is told "you'll have a buddy" but the buddy is overwhelmed with their own work and the relationship fades after week two. The new hire is left to figure things out alone.

**Product implication:** First90 must function as a credible mentorship substitute. ExperienceCoach and CommsCoach in particular must feel like getting advice from someone who knows both the user *and* how professional environments work — not a generic chatbot.

**Feature it informs:** ExperienceCoach (check-ins, micro-actions), CommsCoach (proactive and on-demand), phase transition "moments".

---

## 2. Early-career confidence is fragile and situation-specific

**Evidence:**
- Only ~41% of Gen Z workers say they feel confident navigating today's work environment (LinkedIn, 2025).
- Confidence dips are not uniform — they spike around specific situations: first client interaction, first mistake, first time disagreeing with a manager, first performance conversation.
- Studies consistently show that confidence in early-career workers is the strongest predictor of retention at 12 months — stronger than job satisfaction or salary.

**User experience:** The user feels generally capable but has specific anxiety flashpoints. A single bad interaction with a senior stakeholder can set confidence back weeks. They don't know how to prepare for these moments in advance.

**Product implication:** Confidence tracking must be granular enough to capture situation-specific dips, not just a general weekly trend. CommsCoach proactive scripts must be tied to *specific upcoming moments*, not just offered as a generic feature.

**Feature it informs:** Check-in tag system (situational tags), SignalScout pattern detection (low_confidence_streak), CommsCoach proactive trigger on upcoming comms tasks.

---

## 3. Static 30-60-90 plans are abandoned within two weeks

**Evidence:**
- The majority of early-career 30-60-90 day plan resources are static Word documents or templates that new hires fill in during induction and never revisit.
- HR and L&D teams confirm that follow-through on 30-60-90 plans drops to near zero by week three in most companies.
- The core failure mode: plans are written against an idealised version of the role, not the reality the user encounters in week one.

**User experience:** The user fills in a 30-60-90 template in their first week because they were told to. By week two, the reality of the job bears no resemblance to the plan. They stop using it.

**Product implication:** First90's plan must be treated as a living document that adapts to what is actually happening. PathArchitect's weekly adaptation cycle is non-negotiable — this is the core product differentiator against static templates.

**Feature it informs:** PathArchitect weekly adaptation, SignalScout plan_slippage trigger, plan screen showing current week's goals (not the full 90-day list).

---

## 4. Many graduates are in the wrong role but have no visibility on alternatives

**Evidence:**
- Gen Z are increasingly reporting feeling "stuck" in roles they have outgrown or that don't match their strengths, with no clear path forward (Employment Hero UK, 2025; People Management, 2026).
- Internal mobility is one of the strongest retention levers available to employers — employees who make an internal move are 66–72% more likely to stay for at least one year (LinkedIn Economic Graph, 2025).
- Yet most graduates in their first role have no structured mechanism to discover what other internal roles exist or whether they would be a better fit.

**User experience:** The user is 6 months in and increasingly demotivated. They're not underperforming, but they're not energised either. They don't know if this is just "work being work" or if there's a role in the same company that would suit them better. Nobody has ever asked them what energises them.

**Product implication:** Role-fit discovery must be a core feature, not a nice-to-have. The archetype system and exploration activities are central to the product's value proposition — not an add-on.

**Feature it informs:** Role archetypes library, SignalScout fit-score logic, Phase 2 Explore screen, fit-check reflections at day 45 and 90, growth log export.

---

## 5. Unwritten rules and unclear expectations cause disproportionate anxiety

**Evidence:**
- A consistent theme in early-career research is the gap between what new hires are explicitly told and what they're implicitly expected to know.
- "Psychological safety" research (Edmondson, Harvard Business School) shows that people new to a team consistently underperform not due to lack of skill, but fear of appearing incompetent — so they ask fewer questions and miss critical context.
- New hires from non-traditional or lower socioeconomic backgrounds are disproportionately affected, as they lack informal networks that would otherwise fill this knowledge gap.

**User experience:** The user doesn't know when to email vs Slack, whether it's normal to disagree in meetings, how formal the relationship with their manager should be, or what "good" looks like in their role. They spend significant energy trying to infer these rules rather than doing their actual work.

**Product implication:** Check-in free text and tags must be able to capture this type of anxiety. ExperienceCoach micro-actions should explicitly address unwritten-rule situations (e.g. "It's normal to ask your manager how they prefer to communicate in the first week"). CommsCoach should reduce the activation energy for situations that feel disproportionately high-stakes.

**Feature it informs:** ExperienceCoach check-in prompts, CommsCoach on-demand flow, Phase 1 goal templates ("Have a communication norms conversation with your manager by day 7").

---

## 6. Human career coaching is inaccessible to most early-career workers

**Evidence:**
- Professional career coaches in the UK charge £50–150/hr, with most structured programmes costing £300–500 for a short engagement.
- Graduate and apprentice salaries in the UK typically range from £20,000–£28,000 in 2025–2026, making regular coaching economically inaccessible for the majority.
- Company-provided coaching is typically reserved for senior managers and above; early-career workers are explicitly excluded from most executive coaching programmes.

**User experience:** The user knows they could benefit from structured support but cannot afford it. Free resources (YouTube, Reddit, LinkedIn posts) exist but are unstructured and require significant effort to synthesise into actionable guidance.

**Product implication:** Pricing must be positioned as a fraction of the cost of a single coaching session. The £25–40 one-time 90-day bundle is the key anchor: comparable to 20–30 minutes with a human coach, but providing 90 days of continuous support.

**Feature it informs:** Monetisation model (B2C freemium with affordable paid tier), product tone (warm, accessible, not corporate), marketing positioning.

---

## Summary

| Pain point | Severity | Who it affects most | Primary feature response |
|---|---|---|---|
| No mentor | High | All early-career users | ExperienceCoach, CommsCoach |
| Fragile confidence | High | All, especially first 30 days | Check-in system, proactive CommsCoach |
| Plans abandoned | Medium | All users who've tried other tools | PathArchitect adaptation cycle |
| Wrong role, no visibility | High | Users in generic graduate schemes | Archetype system, Explore screen |
| Unwritten rules anxiety | Medium | First-generation and non-traditional backgrounds | Check-in prompts, Phase 1 goals |
| Coaching inaccessibility | High | Salary-constrained early-career workers | Pricing model, product tone |
