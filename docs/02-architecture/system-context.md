# System Context — First90 (C4 Level 1)

This diagram shows First90 at the highest level: the people who interact with it and the external systems it depends on.

```mermaid
C4Context
  title System Context — First90

  Person(user, "Early-career professional", "Graduate, apprentice, or new hire navigating their first 90 days")

  System(first90, "First90", "Agentic AI app. Provides personalised 30-60-90 day plans, daily check-ins, communications coaching, and role-fit discovery.")

  System_Ext(supabase, "Supabase", "Managed Postgres database, authentication, and storage")
  System_Ext(claude, "Anthropic Claude API", "LLM for complex planning, detailed coaching, and fit-check reflections")
  System_Ext(gemini, "Google Gemini API", "LLM for fast, frequent tasks: check-in analysis, micro-action generation, signal scoring")
  System_Ext(antigravity, "Antigravity", "Google agent orchestration framework. Coordinates the four AI agents.")
  System_Ext(push, "Push Notification Service", "Firebase Cloud Messaging (FCM) and Apple Push Notification service (APNs) via Expo")

  Rel(user, first90, "Uses", "iOS / Android app")
  Rel(first90, supabase, "Reads and writes", "HTTPS / Supabase SDK")
  Rel(first90, claude, "Calls", "HTTPS / Anthropic API")
  Rel(first90, gemini, "Calls", "HTTPS / Google AI API")
  Rel(first90, antigravity, "Orchestrates agents via", "HTTPS / Antigravity SDK")
  Rel(first90, push, "Sends notifications via", "FCM / APNs")
  Rel(push, user, "Delivers push notifications to", "iOS / Android")
```

## Explanation

**The user** is an early-career professional using the First90 mobile app. They interact entirely through the iOS or Android client — they have no direct contact with any backend system.

**First90** is the overall system boundary. Internally it consists of a React Native mobile client, a Node/TypeScript BFF API, and an agent orchestration layer. From the outside, it appears as a single app.

**Supabase** acts as the system of record: structured data (plans, check-ins, suggestions, events, fit scores) and user authentication. First90 never stores sensitive data in the cloud without user consent — see `privacy.md`.

**Anthropic Claude** handles the tasks that benefit most from deep reasoning and long-context understanding: initial plan generation, detailed communications coaching, and 45/90 day fit-check reflections.

**Google Gemini** handles high-frequency, lower-latency tasks: parsing daily check-ins, generating micro-actions, and computing signal scores. Gemini Flash is preferred here for cost and speed.

**Antigravity** is Google's Gemini-native agent orchestration framework. It coordinates the four First90 agents (PathArchitect, ExperienceCoach, CommsCoach, SignalScout), manages their state access, and routes events between them.

**Push Notification Service** (FCM + APNs via Expo) delivers the evening check-in reminder and any time-sensitive agent nudges to the user's device.
