# MVD — Gemma Local Demo ("Role Fit Radar")

> **Scope of this document.** This is the *demo* MVD — the version we build and show in ~90 minutes, running **entirely locally on Gemma via Ollama**. It supersedes the full-product [MVD](mvd.md) for the purposes of the live demo. The full-product MVD (auth, Supabase, Expo, multi-agent, cloud LLMs) remains the roadmap target; nothing here changes it. This doc exists so the demo is buildable, truthful, and impressive within the time and model budget we actually have.

---

## 1. The One Idea

**Turn a new hire's own words about their week into a business signal: where they actually fit.**

A user describes their first few weeks in free text (typed or *spoken*). Gemma extracts what energised and drained them, maps it against First90's role-archetype library, and reveals — on a live **Role Fit Radar** — that the role they were *hired into* may not be where their energy points. It closes with one concrete experiment and a ready-to-send message to make it happen.

This is chosen deliberately over a plain message-rewriter because it is the thing a **corporate/HR buyer** pays for: early-talent retention through internal mobility ("help them find the right internal role before they leave to find it elsewhere"). It is intriguing because it tells someone something non-obvious about themselves, and it is *feasible on a 2B model* because every model call is short, bounded, and single-shot.

### The demo arc (one continuous screen)

1. **Speak or type** — "How have your first few weeks gone?" (seeded with a realistic paragraph so it lands on first load).
2. **Signal extraction** — Gemma pulls out distinct activities, each tagged and rated energising / draining / neutral.
3. **The reveal** — an animated 8-axis radar: *"Hired as: Client & Account Management. Your energy points to: Data & Analytics 82%, Product & Strategy 61%."*
4. **The insight** — one plain-language line naming the mismatch.
5. **The nudge** — one exploration experiment from the top archetype **+ a ready-to-send CommsCoach message** to request it.

---

## 2. Model & Runtime Reality (read before building)

| Fact | Value | Consequence for the build |
|---|---|---|
| Model | `gemma4:e2b-mlx` (effective-2B, on-device tuned) | Great at short extraction/classification; **not** used for long-form reasoning or scoring. |
| Runtime | Ollama, local, `http://localhost:11434` | No backend, no keys, no cloud. The privacy story is *literally true* on stage. |
| Endpoint | `POST /api/chat` | Use `stream: true` for the insight/message so tokens visibly stream. |
| Structured output | `format: "json"` **works**; `format: <schema>` is **ignored by the MLX build** | Do **not** rely on JSON-schema/enum enforcement. Constrain via the prompt + parse defensively. |
| Output quirk | MLX build sometimes emits junk *before* the JSON object | Parser must scan for the first valid `{...}` (see §6). |
| Latency | ~4.5s warm per call (temperature 0) | Two calls total. Cover with motion, not a spinner. Pre-warm the model on page load. |
| CORS | Browser → `localhost:11434` may be blocked | Serve the page over `http://localhost` and set `OLLAMA_ORIGINS=*` if it fights you. |

**Design principle that follows from this:** *the model classifies, JavaScript scores.* We never ask Gemma to compute fit percentages — it extracts tagged activities, and deterministic JS turns those into scores against the archetype markers. This is what makes the reveal reliable on a small model.

---

## 3. Data — Archetypes & Tag Vocabulary

Source of truth: [role-archetypes.md](../04-design/role-archetypes.md). Eight archetypes, each with exactly one **energising signature tag** and a set of **draining markers**, drawn from a fixed 8-tag vocabulary.

**Canonical activity tags (the only values Gemma may emit):**
`data_analysis` · `strategy` · `coding` · `client_calls` · `writing` · `process_improvement` · `people_interaction` · `financial_analysis`

| # | Archetype | Signature (energising) tag | Draining markers |
|---|---|---|---|
| 1 | Data & Analytics | `data_analysis` | `client_calls`, `writing`, `people_interaction` |
| 2 | Product & Strategy | `strategy` | `process_improvement`, `financial_analysis` |
| 3 | Tech & Engineering | `coding` | `client_calls`, `people_interaction` |
| 4 | Client & Account Management | `client_calls` | `coding`, `financial_analysis` |
| 5 | Marketing & Content | `writing` | `financial_analysis`, `coding` |
| 6 | Operations & Process | `process_improvement` | `strategy`, `client_calls` |
| 7 | People & Culture | `people_interaction` | `coding`, `financial_analysis` |
| 8 | Finance & Commercial | `financial_analysis` | `writing`, `people_interaction` |

Each archetype also carries its exploration activities (from the source doc) — used verbatim for the nudge, e.g. Data & Analytics → *"Shadow a data analyst for a morning."*

---

## 4. Scoring Algorithm (deterministic, in JS)

Input: the array of extracted `{text, tag, energy}` activities. Output: a 0–100 fit score per archetype for the radar.

```
for each archetype A:
    raw = 0
    for each activity act:
        if act.tag == A.signatureTag:
            raw += { energising: +3, neutral: +1, draining: -2 }[act.energy]
        else if act.tag in A.drainingMarkers:
            raw += { draining: -1, neutral: 0, energising: +1 }[act.energy]
    rawScore[A] = raw

# spread raw scores across the visible range so the radar reads clearly
min, max = range(rawScore)
fit[A] = round( 20 + (raw - min) / max(1, (max - min)) * 80 )   # 20..100
```

- **Hired archetype** is chosen by the user in a small "Hired as" control (defaults to *Client & Account Management* so the seeded example produces a satisfying mismatch).
- **Top signal** = highest `fit[A]` that is *not* the hired archetype.
- Ties and empty input degrade gracefully (all mid-range → flat radar → insight says "not enough signal yet, keep checking in").

---

## 5. Model Calls (exactly two)

### Call 1 — Extract (JSON mode, temperature 0)
- **System:** "You extract distinct work activities from a diary entry and output ONLY valid JSON, no prose, no markdown. Allowed tags (pick the single closest per activity): `<8 tags>`. Allowed energy: energising, draining, neutral. Energy cues: 'lost track of time', 'enjoyed', 'proud' → energising; 'dreaded', 'drained', 'boring', 'anxious' → draining. Shape EXACTLY: `{"activities":[{"text":"...","tag":"...","energy":"..."}]}`."
- **User:** the diary text.
- **Output:** parsed → scoring (§4).

### Call 2 — Narrate (JSON mode, streamed)
- **Input:** hired archetype, top-signal archetype, its fit score, the drained signature activities.
- **Output shape:** `{"insight":"<=2 sentences naming the mismatch, warm not clinical>", "experiment":"<one exploration activity from the top archetype>", "message":"<a short, confident Slack/email message the user could send to request that experiment>"}`.
- Streamed token-by-token into the UI so the close feels alive.

> Everything else (scores, chart, archetype labels) is computed in JS. Gemma only ever produces natural language and tagged activities.

---

## 6. Defensive JSON Parsing (required)

The MLX build can prepend junk and occasionally trail commentary. The parser must:
1. Scan the raw content for candidate objects: for each index of `{`, attempt `JSON.parse` on the balanced substring (or use an incremental `raw_decode`-style scan).
2. Accept the **first** object that contains the expected key (`activities` for Call 1, `insight` for Call 2).
3. On total failure, fall back to a safe default (empty activities / generic insight) and surface a small "couldn't read that clearly, try rephrasing" toast — never a stack trace on screen.

---

## 7. The Live Voice Module ("Live Check-in")

We have a **voice slot** in the input step. It has two interchangeable tiers behind one UI:

| Tier | What runs | Use for |
|---|---|---|
| **Local (demo default)** | Browser **Web Speech API** — `SpeechRecognition` for speech→text, `SpeechSynthesis` for the spoken prompt/summary. Reasoning stays on local Gemma. | The 90-minute local demo. Zero deps, zero keys, instant, works offline. |
| **Gemini Live (production upgrade)** | Google **Gemini Live** streaming multimodal voice for low-latency, natural turn-taking STT/TTS; Gemma (or Gemini) as the reasoning tier behind it. | Post-demo / cloud build. Named here as the swap-in; **not** wired for the local demo. |

**Why not "Gemini Live via Gemma" literally:** Gemini Live is a hosted real-time service and Gemma is a local open model — they are not the same layer. The spec treats voice as one slot with two tiers so we can demo locally today and upgrade to Gemini Live later without changing the product flow.

### Live Check-in UX
- A **mic button** on the input step. Tap → the app speaks the prompt ("How have your first few weeks gone?"), then listens.
- Live **interim transcript** appears as the user talks (Web Speech interim results), settling into the diary textarea — so the user sees their words become the input to the radar.
- User can edit the transcript before analysing (voice is a fast path, not a black box).
- On reveal, an optional **"read it back"** toggle speaks the insight line via `SpeechSynthesis` — a strong, human demo beat.
- **Fallback:** if Web Speech is unavailable (non-Chrome, blocked mic), the mic button hides and typing remains the primary path. Voice is additive, never required.

### Feasibility flag
Web Speech STT/TTS is a ~20–30 min add on top of the core build. Treat it as the **first stretch goal**: build the typed flow end-to-end first, then layer voice. If the clock runs out, the demo is still complete without it.

---

## 8. UI Specification (improved)

A single-page, single-scroll experience that feels like a premium product, not a form. iOS-first visual language even though it renders in a browser for the demo.

### 8.1 Layout — one screen, three acts
A centered column (max-width ~520px, phone-shaped) that progresses top-to-bottom as the user acts. Earlier acts collapse into compact summaries as later ones appear, so the whole story stays on one scroll.

```
┌───────────────────────────────┐
│  First90 · Role Fit Radar     │  ← slim header, wordmark + phase pill
├───────────────────────────────┤
│  ACT 1 — INPUT                │
│  "How have your first few     │
│   weeks gone?"                │
│  ┌─────────────────────────┐  │
│  │ [ diary textarea ]      │  │  ← seeded example text
│  └─────────────────────────┘  │
│  Hired as: [ dropdown ]       │
│  [ 🎤 Speak ]   [ Analyse → ] │
├───────────────────────────────┤
│  ACT 2 — REVEAL (appears)     │
│   ◆ Radar chart (8 axes)     │  ← animated draw-in
│   Hired ▲  vs  Signal ▲      │  ← two overlaid polygons + legend
│   Signal chips: data 82% …    │
├───────────────────────────────┤
│  ACT 3 — INSIGHT + NUDGE      │
│  “You light up around data…”  │  ← streamed
│  Try this: shadow the data…   │
│  ✉︎ Ready-to-send message      │  ← streamed, copy button
└───────────────────────────────┘
```

### 8.2 Visual design tokens
- **Type:** system UI stack (`-apple-system, "SF Pro", Inter, system-ui`). Large, confident headings (28–32px), calm 16px body, 13px labels in uppercase tracking.
- **Palette (dark-first, calm, "trustworthy tech"):**
  - Background: deep slate `#0E1116` with a subtle top-down gradient to `#141A22`.
  - Surface cards: `#1A222D` at ~80% with 1px `#243040` hairline borders, 16px radius, soft shadow.
  - Text: primary `#EAF0F7`, secondary `#9AA7B6`.
  - **Energising** accent: teal-green `#35D0A5`. **Draining** accent: warm amber `#F2A65A`. Neutral: `#5A6B7D`.
  - **Hired** polygon: cool blue `#4C7DF0`. **Signal** polygon: the teal accent. This blue-vs-teal contrast *is* the story.
- **Motion:** everything eases in (200–320ms, `cubic-bezier(.2,.8,.2,1)`). The radar polygon **draws/scales from center** over ~700ms. Streamed text uses a soft caret. No harsh spinners anywhere.
- **Density:** generous whitespace; one clear primary action visible at a time.

### 8.3 Component notes

**Radar chart (custom SVG, no chart library).**
- 8 axes at 45° intervals, labelled with archetype short names.
- Concentric grid rings at 20/40/60/80/100 with faint stroke.
- Two polygons: **Hired** (blue, thin, dashed, low fill) and **Signal** (teal, solid, ~25% fill) so the gap between "where you were placed" and "where your energy is" is visually obvious.
- Vertices are subtle dots; the top-signal vertex gets a small glow + label callout.
- Draw-in animation: polygons scale from 0→1 from center; axis labels fade in staggered.

**Signal chips.** Below the radar, a ranked row: `Data & Analytics 82% ▲` etc., top 3 highlighted. Hired archetype chip marked with a small "hired" tag for contrast.

**Insight card.** Streamed sentence in larger, warm type — the emotional peak. Optional 🔊 "read it back" (TTS) button.

**Nudge + message card.** The experiment as a one-liner with an icon; the CommsCoach message in a chat-bubble style block with a **Copy** button (and a tiny "Regenerate" that re-runs Call 2 only).

**Mic / Live control.** Prominent but secondary to Analyse. While listening: pulsing ring animation + live interim transcript feeding the textarea. Clear listening/processing/idle states.

### 8.4 States & edge cases
- **Loading (extraction):** the textarea's activities animate into small tagged pills as they're detected (or a calm "reading your week…" shimmer if we keep it simple). Never a blank spinner.
- **Empty / too short input:** Analyse stays disabled until ~15+ words; helper text nudges for more detail.
- **Weak signal:** flat-ish radar → insight gracefully says there isn't a clear direction yet.
- **Model/parse error:** small non-blocking toast; the seeded example still works so the demo can always recover.
- **First call cold start:** fire a tiny warm-up call to Gemma on page load so the first real Analyse is fast.

### 8.5 Accessibility & polish
- Full keyboard path (Tab to Analyse, Enter to submit).
- Respect `prefers-reduced-motion` — disable the draw-in, keep the final state.
- Sufficient contrast on all text; chips and polygons distinguishable without relying on color alone (dashed vs solid, labels).
- Mic is additive; nothing requires audio.

---

## 9. In Scope / Out of Scope (demo)

**In scope**
- Single-page local web app (static HTML/CSS/JS, no build step).
- Two Gemma calls via Ollama (extract, narrate) with defensive parsing.
- Deterministic archetype scoring in JS from the marker table.
- Animated SVG Role Fit Radar (hired vs signal).
- Streamed insight + experiment + copyable CommsCoach message.
- Seeded example input for a reliable cold-open.
- **Stretch 1:** Live voice check-in (Web Speech STT/TTS).
- **Stretch 2:** "read it back" TTS on the insight.

**Out of scope (abstracted away for the demo)**
- Supabase / Postgres / RLS / auth — hardcoded single user, in-memory state.
- Antigravity / multi-agent orchestration — direct function calls.
- ExecuTorch on-device runtime — Ollama *is* the local-model story.
- PII stripping / privacy modes — everything is local, so it's moot on stage (mention verbally).
- PathArchitect 30-60-90 plan generation — worst fit for e2b; skip or show static.
- SignalScout time-series analysis, push notifications, Expo/native, employer dashboard.
- Gemini Live wiring — named as the production voice upgrade, not built locally.

---

## 10. Build Budget (~90 min)

| Time | Task |
|---|---|
| 0:00–0:10 | Scaffold single file; confirm `gemma4:e2b-mlx` reachable; warm-up call; CORS check. |
| 0:10–0:35 | Call 1 (extract) + defensive parser + JS scoring against markers; verify on seeded example. |
| 0:35–0:55 | SVG radar (grid, two polygons, labels, draw-in animation) wired to scores. |
| 0:55–1:10 | Call 2 (narrate) streamed → insight + experiment + copyable message. |
| 1:10–1:20 | Visual polish pass (tokens, motion, states). |
| 1:20–1:30 | **Stretch:** Web Speech voice input; else buffer + prompt tuning on real inputs. |

---

## 11. Success Criteria (demo)

- A cold audience member's spoken/typed week produces a radar with a **clear, plausible top signal** within ~10s.
- The **hired-vs-signal mismatch** is legible at a glance (blue vs teal polygons).
- The insight line reads as **specific and human**, not generic.
- The CommsCoach message is **copy-paste sendable** without editing.
- The whole flow runs **fully offline on the laptop** — provable by turning off Wi-Fi mid-demo.

---

## 12. Production Upgrade Path (post-demo, for the buyer conversation)

- **Voice:** Web Speech → **Gemini Live** streaming multimodal voice for natural turn-taking.
- **Reasoning:** local `gemma4:e2b` for on-device/private tasks → larger Gemma or frontier model in the cloud for plan generation.
- **Persistence & multi-user:** the demo's in-memory state → Supabase schema already specified in [data-model.md](../02-architecture/data-model.md), with role-fit scores accumulating over real check-ins (the SignalScout story).
- **From single reveal → longitudinal signal:** the demo shows one snapshot; the product tracks fit drift across the 90 days — the actual retention insight an employer buys.
