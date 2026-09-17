# Market Validation — First90

## Summary

The early-career support market has a clear, well-documented gap: most graduates, apprentices, and new hires enter companies with poor or no structured support, low confidence, and high attrition risk. Existing AI tools address job search and broad career development — none address the specific, high-value window of the first 90 days with agentic, adaptive, privacy-first tooling.

---

## Pain Point Evidence

### 1. Mentorship scarcity
- ~73% of Gen Z workers report they do not have access to a mentor at work
- Most young workers say mentorship dramatically boosts confidence, opens doors, and bridges theory to real-world work
- Smaller organisations (SMEs, scale-ups) rarely have formal mentoring structures — exactly where many graduates and apprentices land

### 2. Confidence deficit
- Only ~41% of early-career workers feel confident navigating their workplace in year one
- The top reported blockers: not knowing unwritten rules, fear of asking "stupid" questions, anxiety about senior communication
- Confidence recovers sharply when workers feel they have a clear plan and someone (or something) in their corner

### 3. Early attrition and internal immobility
- Internal mobility increases 1-year retention by 66–72%, with the strongest effect on Gen Z (LinkedIn Economic Graph, 2025)
- Most early-career leavers cite "no visible path forward" or "wrong fit for the role" — not compensation — as primary reasons
- Companies lose an average of 50–200% of a role's annual salary when an early-career hire leaves in year one (recruitment, onboarding, lost productivity)

### 4. Onboarding failure
- Most onboarding content is forgotten within 1 week if not reinforced
- Static 30-60-90 templates exist everywhere but are abandoned quickly — they don't adapt, prompt, or respond
- Managers report onboarding as one of their top pain points but say they lack time to do it well

### 5. Career coach inaccessibility
- Human career coaches cost £50–150/hr in the UK — completely inaccessible to most early-career workers
- Internal L&D budgets rarely extend to 1:1 coaching for junior staff
- The result: people who need structured support the most get it the least

---

## Competitive Landscape

| Product | Primary focus | Agentic plan? | 90-day scope? | Internal role-fit? | Privacy-first? | Price |
|---|---|---|---|---|---|---|
| **First90** | Early-career first 90 days | ✅ | ✅ | ✅ | ✅ | ~£8–15/mo |
| Path AI | General career growth & goals | ❌ | ❌ | ❌ | ❌ | ~$10/mo |
| Rocky.ai | Professional development coaching | ❌ | ❌ | ❌ | ❌ | ~$20/mo |
| BetterUp | Mid-career coaching (enterprise) | ❌ | ❌ | ❌ | ❌ | Enterprise |
| Himalayas AI | Career path discovery | ❌ | ❌ | ❌ | ❌ | Free/paid |
| Career Compass AI | CV, interview, job search | ❌ | ❌ | ❌ | ❌ | ~$15/mo |
| Google Career Dreamer | Skills mapping, career exploration | ❌ | ❌ | ❌ | ❌ | Free |
| Generic 30-60-90 templates | Structured plan document | ❌ | ✅ | ❌ | N/A | Free |
| Company onboarding portals | Internal HR onboarding | ❌ | Partial | ❌ | N/A | Enterprise |

### Key insight
No product sits at the intersection of: time-boxed to first 90 days + agentic adaptation + internal role-fit discovery + privacy-first on-device AI. This is First90's defensible positioning.

---

## Willingness to Pay

### B2C rationale
- Target user is employed and earning — even at apprentice/graduate rates (£18–28k UK)
- £8–15/month is comparable to Spotify, Netflix, or a Deliveroo order — normalised micro-subscription spend
- The 90-day bundle (£25–40 one-time) removes subscription fatigue and matches the product's natural lifecycle
- Comparable: a single session with a human career coach costs £50–150. First90 is always-on for a fraction of that
- Early adopters (apprentices, career-anxious grads) are highly motivated — they will pay for something that reduces anxiety and uncertainty

### B2B rationale
- Apprenticeship levy costs UK employers £X per apprentice — poor completion and attrition rates erode this investment
- Grad scheme operators spend £5–15k per hire on recruitment and onboarding. A £200–500/seat tool that improves retention ROI is easy to justify
- HR/People teams are actively seeking tools to demonstrate early-career programme quality without adding headcount

---

## Key Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Low daily check-in retention (users stop after week 2) | High | High | Invest heavily in check-in UX — short, warm, push-triggered. Make skipping frictionless. Track completion rate as north-star metric |
| Users don't trust AI-generated plan | Medium | High | Supervised autonomy: all plan changes are proposals. Explain reasoning behind every suggestion |
| Privacy concerns block sign-up | Medium | High | On-device default for reflections. Clear, plain-language privacy promise on onboarding. No dark patterns |
| Enterprise sales cycle too slow for solo founder | Medium | Medium | Stay B2C until £5–10k MRR. B2B only when there's inbound demand from an employer |
| Gemini/Claude API costs become significant at scale | Low | Medium | Use Gemini Flash for frequent low-complexity tasks (check-in analysis, signal scoring). Claude only for complex reasoning |
| Competitor (LinkedIn, Google) builds this natively | Low | High | Speed and focus. Be in market with a loved product before a platform player notices the niche |
| App Store approval delays | Low | Low | Submit early. Keep first build minimal to reduce review risk |

---

## Validation Plan (Pre-Launch)

### Stage 1 — Problem validation (now)
- [ ] Interview 10–15 graduates/apprentices currently in year 1 of a corporate role
- [ ] Key questions: biggest challenge in first 90 days, what support they have/lack, what they wish existed
- [ ] Success signal: 8+ of 15 name one of First90's five pain points unprompted

### Stage 2 — Concept validation (after MVD)
- [ ] Show 10 target users the MVD (Figma or working app)
- [ ] Measure: would you use this? would you pay £8–15/month?
- [ ] Success signal: 6+ of 10 say yes to both questions

### Stage 3 — Retention validation (closed beta)
- [ ] 20–50 beta users through a full 90-day cycle
- [ ] North-star metric: 7-day check-in completion rate > 60%
- [ ] Secondary: NPS > 40 at day 30 and day 90
