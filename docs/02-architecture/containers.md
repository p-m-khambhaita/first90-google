# Containers — First90 (C4 Level 2)

This diagram zooms into the First90 system boundary and shows the major containers (applications and data stores) and how they communicate.

```mermaid
C4Container
  title Containers — First90

  Person(user, "Early-career professional", "Uses the mobile app")

  Container_Boundary(first90, "First90") {
    Container(app, "Mobile App", "Expo React Native", "iOS and Android client. Renders UI, captures check-ins, displays plans and suggestions. Communicates only with the BFF API.")
    Container(api, "BFF API", "Node.js / TypeScript / Fastify", "Backend for Frontend. Handles auth validation, orchestrates LLM calls, writes to Supabase, and triggers agent jobs via Antigravity.")
    Container(orchestrator, "Agent Orchestrator", "Antigravity (Google)", "Coordinates the four AI agents. Reads state from Supabase, calls LLM providers, writes suggestions back to Supabase.")
  }

  ContainerDb(db, "Postgres Database", "Supabase Postgres", "Stores all structured data: users, roles, plans, check-ins, suggestions, events, fit scores, archetypes.")
  ContainerDb(auth, "Auth Service", "Supabase Auth", "Manages user identity. Magic link and Google OAuth. Issues JWTs consumed by BFF API.")
  ContainerDb(storage, "File Storage", "Supabase Storage", "Stores exported growth logs and any future user-uploaded files.")

  System_Ext(claude, "Anthropic Claude API", "Complex reasoning and long-context tasks")
  System_Ext(gemini, "Google Gemini API", "Fast, frequent, structured tasks")
  System_Ext(push, "FCM / APNs", "Push notifications via Expo")

  Rel(user, app, "Interacts with", "Touch / tap")
  Rel(app, api, "Makes requests to", "HTTPS / REST")
  Rel(api, db, "Reads and writes", "Supabase SDK / SQL")
  Rel(api, auth, "Validates JWT via", "Supabase Auth SDK")
  Rel(api, orchestrator, "Triggers agent jobs via", "Antigravity SDK")
  Rel(api, push, "Schedules notifications via", "Expo Push API")
  Rel(orchestrator, db, "Reads state and writes suggestions", "Supabase service role")
  Rel(orchestrator, claude, "Calls for complex tasks", "HTTPS / Anthropic API")
  Rel(orchestrator, gemini, "Calls for fast tasks", "HTTPS / Google AI API")
  Rel(storage, api, "Accessed via", "Supabase Storage SDK")
```

## Container Descriptions

### Mobile App (Expo React Native)
The user-facing client. Built with Expo SDK and Expo Router for file-based navigation. The app is intentionally thin — it renders UI, captures user input, and displays data returned by the BFF API. It never calls LLM APIs directly. All state is server-authoritative; the app holds only ephemeral UI state locally.

Platforms: iOS (primary), Android (secondary), Expo Web (stretch goal).

### BFF API (Node.js / TypeScript)
The Backend for Frontend sits between the mobile app and all backend services. Responsibilities:
- Validate Supabase JWTs on every request
- Translate app requests into Supabase reads/writes
- Trigger Antigravity agent jobs when appropriate events occur (e.g. check-in submitted, plan update accepted)
- Handle push notification scheduling
- Apply PII stripping before any text reaches an LLM provider

Hosted on Google Cloud Run (serverless, scales to zero). Stateless — all state lives in Supabase.

### Agent Orchestrator (Antigravity)
Coordinates the four AI agents. Receives jobs from the BFF API (e.g. `check_in_submitted`, `role_setup_complete`), loads relevant state from Supabase via a service role, calls Claude or Gemini as appropriate, and writes the resulting suggestions back to the `suggestions` table in Supabase. The mobile app then surfaces these suggestions to the user for acceptance.

All agent logic, prompt templates, and routing rules live here. See `03-agents/` for individual agent specs.

### Postgres Database (Supabase)
The system of record for all structured data. Row Level Security (RLS) is enabled on all tables — users can only access their own data. The orchestrator accesses the database via a service role (bypasses RLS) for reading cross-user archetype data and writing agent outputs.

### Auth Service (Supabase Auth)
Handles user identity. Supports magic link (email) and Google OAuth. Issues short-lived JWTs that the BFF API validates on every request. No passwords stored.

### File Storage (Supabase Storage)
Used for exporting the user's "growth log" (90-day summary) as a PDF. May also be used for future features such as CV uploads or document context for CommsCoach.
