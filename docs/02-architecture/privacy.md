# Privacy Architecture — First90

## Principle

> Sensitive data stays on your device by default. First90 helps you without needing to know everything about you.

First90 handles some of the most personal data a professional tool can touch: daily reflections on confidence, anxiety, relationships with colleagues, and feelings about work. The privacy architecture is designed from first principles around this reality — not bolted on after the fact.

---

## The Two Privacy Modes

### Private Mode

| Property | Value |
|---|---|
| Free-text reflections | Processed on-device only. Never sent to any server. |
| LLM for reflections | On-device quantised SLM (2–4B params, ExecuTorch) |
| Confidence + energy scores | Synced to Supabase (numerical only, no text) |
| Tags | Synced to Supabase |
| Plan data | Synced to Supabase |
| Cloud LLM calls | None for check-in processing |
| Plan generation | Cloud LLM (Claude) with only role title, archetype, and phase — no personal data |
| Limitations | Micro-actions less contextualised. No pattern detection across free-text. SignalScout works on scores and tags only. |

### Standard Mode (default)

| Property | Value |
|---|---|
| Free-text reflections | Stored encrypted in Supabase after user opts in |
| LLM for reflections | Gemini Flash via BFF API (PII-stripped before sending) |
| Confidence + energy scores | Synced to Supabase |
| Tags | Synced to Supabase |
| Plan data | Synced to Supabase |
| Cloud LLM calls | Yes — for check-in analysis, micro-actions, plan updates |
| Plan generation | Cloud LLM (Claude) with role context + recent check-in summaries |
| Limitations | Relies on PII stripping being effective. See stripping rules below. |

Users can switch modes at any time from Settings. Switching to Private mode stops syncing free text immediately. Previously synced text is not deleted from Supabase automatically — the user must trigger a data deletion request.

---

## On-Device AI Layer

### What runs on-device
- Free-text reflection parsing (tag extraction, sentiment classification)
- Quick message rewrites (CommsCoach light, Standard mode only if latency matters)
- Instant micro-action suggestions when offline

### Model choice
- **Target:** 2–4B parameter quantised model (e.g. Phi-3 Mini, Gemma 2B, or Llama 3.2 3B)
- **Runtime:** ExecuTorch (iOS/Android via React Native native module)
- **Quantisation:** INT4 or INT8 for memory efficiency on mobile hardware
- **Model size budget:** <1.5GB on-device storage per model
- **Fallback:** If on-device model is unavailable (first launch before download completes), Standard mode processing is used with an explicit user notification.

### Model delivery
- Model files are downloaded on first launch over Wi-Fi only (unless user overrides)
- Stored in app-private storage (not accessible to other apps)
- Updated via silent background download when a new model version is available

---

## PII Stripping (Standard Mode)

Before any free text or user-provided content is sent to a cloud LLM (Claude or Gemini), the BFF API applies PII stripping:

### Replacement rules

| Pattern | Replaced with |
|---|---|
| Manager/boss references ("my manager Sarah", "Sarah told me") | `[manager]` |
| Colleague names (detected via NER or common-name list) | `[colleague]` |
| Company name (extracted from user's `roles.company_name`) | `[company]` |
| Client/customer names | `[client]` |
| Project names (if previously flagged by user) | `[project]` |
| Email addresses | `[email]` |
| Phone numbers | `[phone]` |
| Locations (office addresses, meeting rooms) | `[location]` |

### Implementation approach
1. **Exact match:** Replace known values from the user's Supabase profile (`company_name`, manager name if provided).
2. **Regex patterns:** Email, phone, dates.
3. **NER model:** Lightweight on-device NER (spaCy-style) to catch person names not in the user's profile before text leaves the device.
4. **Post-strip review:** The stripped version is logged (not the original) so the BFF API can audit what was sent.

### Limitations
- NER is imperfect. Some names will not be detected, especially unusual or non-Western names.
- Users are informed of this limitation during onboarding and shown the stripped version before it is sent (opt-in "preview before sending" setting).

---

## Data Residency

| Data type | Where stored | Encrypted at rest? |
|---|---|---|
| Confidence + energy scores | Supabase (EU region) | Yes (Supabase default) |
| Tags | Supabase (EU region) | Yes |
| Plan data, goals | Supabase (EU region) | Yes |
| Free text (Standard mode, opted in) | Supabase (EU region) | Yes |
| Free text (Private mode) | Device only | Yes (iOS/Android secure enclave) |
| LLM prompt content | In-flight only (BFF → LLM API) | TLS in transit, not stored |
| Model weights (on-device) | App-private device storage | Yes |

---

## User-Facing Privacy Promise

This is the exact copy shown to users during onboarding and in the Privacy section of Settings:

---

*"First90 is built for you, not about you.*

*Your daily reflections — how you're really feeling, what's hard, what's worrying you — stay on your device by default. We use a small AI model that runs entirely on your phone to process them. Nothing leaves unless you choose.*

*When you ask for deeper help — like building your 90-day plan or rewriting a message — we do send some context to a cloud AI. Before we do, we strip out names, your company, and anything that could identify the people you're writing about. We show you exactly what we're sending if you want to check.*

*We will never sell your data. We will never train an AI on your reflections. You can delete everything, at any time, from Settings.*

*We built First90 because we believe you deserve a tool that helps you without needing to know everything about you."*

---

## GDPR Compliance Notes

- **Lawful basis:** Contractual necessity (processing required to deliver the service the user signed up for).
- **Data minimisation:** Only scores and tags are synced by default. Free text is opt-in.
- **Right to erasure:** Users can trigger full account and data deletion from Settings. Supabase cascade delete handles all tables. On-device data is cleared by uninstalling the app.
- **Data portability:** Users can export their full data (check-ins, plans, events) as JSON from Settings. Growth log export as PDF via Supabase Storage.
- **DPA:** Supabase acts as a data processor. Anthropic and Google act as sub-processors for LLM calls. Data Processing Agreements required with all three before launch.
- **Age:** First90 is not intended for users under 16. Age gate at sign-up.
