# ITS Platform & Thrive Project — Current State

**Date:** April 4, 2026
**Author:** VDA (Veterans' Digital Ally)
**Location:** `~/projects/its-platform` (monorepo) + `~/projects/thrive-demo` (proposal docs)

---

## 1. What Exists

### 1.1 Monorepo: `~/projects/its-platform`

A Turborepo + npm workspaces monorepo containing 8 shared core packages and 2 tenant apps. Extracted from the production DAP-Cohort-26 codebase (cybersecurity ITS) and parameterized for multi-tenant use.

**Build status:** Both apps build successfully. 75 BKT engine tests pass.
**CI/CD:** None yet. No GitHub Actions, no deployment pipeline, no security scanning in CI.
**Git:** Initialized, not pushed to remote.

**File count:** 63 source files, ~2,979 lines of core logic + 460 lines of tests.

### 1.2 Thrive Demo: `~/projects/thrive-demo`

Contains only `thrive-ai-proposal.docx` — the VDA proposal to Thrive Network for the Scale Accelerator Assessment Engine. Not a git repo.

---

## 2. Monorepo Structure

```
its-platform/
├── package.json                    # npm workspaces root, Turborepo
├── turbo.json                      # Build pipeline: build, dev, test, deploy
├── package-lock.json
│
├── packages/
│   ├── core-bkt/                   # Bayesian Knowledge Tracing engine
│   ├── core-data/                  # Firestore data access + audit logging
│   ├── core-auth/                  # Auth middleware (strategy pattern)
│   ├── core-ai/                    # Chat handler, exam engine, skill tagger
│   ├── core-api/                   # Express app factory + rate limiting
│   ├── core-survey/                # Survey CRUD handlers
│   ├── core-ui/                    # Shared React components
│   └── tenant-config/              # Config schema + validator
│
└── apps/
    ├── dap/                        # DAP-Cohort-26 tenant (placeholder)
    └── thrive/                     # Thrive Network tenant (scaffold)
```

---

## 3. Package Details

### `@its/core-bkt` — Bayesian Knowledge Tracing Engine
**Lines:** 392 (engine) + 460 (tests) = 852 total
**Tests:** 75 passing
**Dependencies:** Zero. Pure JavaScript math. Portable to any JS runtime.

This is the adaptive learning engine. It is not a wrapper around an LLM — it is a standalone probabilistic model that tracks what a student knows.

**What it implements:**
- **Standard BKT:** Posterior update + learning transition. `P(L_n) = P(L|obs) + (1 - P(L|obs)) * P(T)`.
- **Source-specific parameters:** 8 observation sources (ctf, github, tutor, lab, exam, peer-teach, review, lecture), each with calibrated guess/slip/transition probabilities. A correct answer on an exam (25% guess baseline) carries different weight than a correct answer in chat (12% guess baseline).
- **Bloom's taxonomy weighting:** Learning transition rate scales by cognitive level. A "remember" observation (weight 0.3) produces a smaller mastery update than an "analyze" observation (weight 1.0) or "create" (weight 1.5).
- **Source-aware mastery ceilings:** Prevents mastery inflation from low-quality evidence. CTF alone caps at 33% for analyze-level. Exam caps at 85%. Peer-teaching has no ceiling. When 3+ distinct sources contribute, all ceilings are removed.
- **Time-based exponential decay:** `P(L_decayed) = P(L) * e^(-λ * days)`. Decay rate varies by knowledge type: procedural skills (hands-on commands) decay slower (λ=0.005, half-life 139 days) than conceptual skills (λ=0.008, half-life 87 days).
- **EMA (Exponential Moving Average):** Secondary model running alongside BKT. Smooths observation noise. Used for dual mastery agreement checks.
- **Dual mastery levels:** Promotion requires BOTH models to agree AND sufficient evidence:
  - Mastered: BKT ≥ 0.85, EMA ≥ 0.75, 200+ observations, 3+ sources, must include peer-teach or lecture
  - Proficient: BKT ≥ 0.65, EMA ≥ 0.55, 150+ observations, 2+ sources
  - Developing: BKT ≥ 0.40, EMA ≥ 0.35, 75+ observations, 1+ sources
  - Novice: everything below those thresholds
- **SM-2 spaced repetition:** Skills crossing the developing threshold (P(L) ≥ 0.40) are auto-enrolled in spaced review scheduling.

**Key design decision:** The `getDecayRate()` function accepts a tenant-provided knowledge type map, so each tenant defines which of their skills are procedural, conceptual, or applied. The engine doesn't know or care about the domain.

### `@its/core-data` — Firestore Data Access Layer
**Lines:** 771 (data-access) + 23 (firebase-admin) + 38 (audit-logger) + 28 (utils) = 860 total
**Dependencies:** firebase-admin

The abstraction layer between core logic and the database. All Firestore collection names are configurable via `initCollections()` — this is the seam for future Supabase migration.

**What it provides:**
- Conversation logging (with SharePoint sync flag for DAP)
- Skill state CRUD with transactional updates (prevents lost observations under concurrent requests)
- BKT observation logging with full instrumentation (pL before/after, EMA, decay applied, source params, ceiling applied)
- Weekly mastery snapshots
- Weekly report storage
- BKT consent management (per-member opt-in/out)
- Learner profile storage
- Exam session CRUD
- AI usage tracking (daily/monthly/lifetime token counts with atomic rollover)
- Spaced repetition scheduling (SM-2 algorithm)
- Amendment requests (student requests to modify their profile data, admin approval flow)
- Instructor mastery overrides (admin can manually set mastery with audit trail)
- Participant notes (instructor annotations per student)

**Collections managed (13):** ai_conversations, skill_states, ai_skill_snapshots, weekly_reports, ai_learner_profiles, ai_exam_sessions, ai_usage, ai_spaced_repetition, amendment_requests, bkt_observations, cohort_members, mastery_overrides, participant_notes

### `@its/core-auth` — Authentication Middleware
**Lines:** 123 (middleware) + 75 (Azure AD strategy) + 34 (Firebase Auth strategy) = 232 total
**Dependencies:** jose (JWT verification), @its/core-data

Strategy pattern supporting multiple auth providers per tenant.

**What it provides:**
- `createAuthMiddleware(strategy, options)` — factory that returns Express middleware
- Service-to-service auth via ADMIN_API_TOKEN (timing-safe comparison)
- Role loading from Firestore `roles` collection (cached 5 minutes)
- `requireRole(...roles)` — RBAC middleware factory
- `onFirstLogin` hook — tenant provides a function to auto-provision new users
- Two strategies implemented:
  - **Azure AD (MSAL):** JWT validation against Microsoft's JWKS endpoint, optional group membership check. Used by DAP.
  - **Firebase Authentication:** Validates Firebase ID tokens via Admin SDK. Used by Thrive.

### `@its/core-ai` — AI Chat, Exam, and Skill Tagging
**Lines:** 280 (chat-handler) + 344 (exam-engine) + 84 (skill-tagger) + 183 (chat-sessions) = 891 total
**Dependencies:** @anthropic-ai/sdk, @its/core-bkt, @its/core-data

The AI interaction layer. All components are factory functions that accept tenant configuration.

**`createChatHandler(config)` — the main tutoring interface:**
- Accepts: system prompt, skill taxonomy, knowledge types, mode handlers, context fetchers, model config
- Validates and sanitizes input (message length, page context, history)
- Loads persistent session history or uses client-provided history
- Fetches consent-gated skill context (proficiency tiers injected into prompt)
- Fetches spaced review due items
- Runs tenant-provided context fetchers in parallel (e.g., DAP fetches meeting transcripts, recording consumption)
- Mode handler hook: tenants register custom modes (DAP registers CTF flag handling, Thrive does not)
- Calls Claude with prompt caching on system message (90% cost reduction on cached reads)
- Parses skill metadata from Claude's response (strips `<!-- ALAI_META:... -->`)
- Fire-and-forget post-response pipeline: usage tracking → audit logging → conversation logging → BKT update with source-aware step + decay + EMA → observation logging → spaced rep enrollment → SM-2 review advancement

**`createExamEngine(config)` — adaptive practice exams:**
- Accepts: domain definitions, skill mappings, label/match functions, knowledge types, model choice
- Generates MCQ questions via Claude at the student's Bloom's level (Remember/Understand/Apply/Analyze based on current mastery)
- Manages exam sessions: start, resume, submit answer, save progress, submit final
- Transactional answer processing (prevents double-submit)
- Fire-and-forget BKT update per answer: narrows to domain-specific skills only (not all cert skills — this was a specific inflation bug fix)

**`createSkillTagger(skillDefinitions)` — dual-mode skill classification:**
- Keyword matching: zero API cost, runs on every interaction. Requires 2+ keyword hits or 1 strong match (skill ID or label).
- Claude metadata parsing: extracts `<!-- ALAI_META:{"skills":[],"difficulty":"","correct":true} -->` from response. Validates skill IDs against tenant taxonomy, validates Bloom's level, enforces max 3 skills.
- Falls back to keyword tagger when Claude metadata is missing or malformed.

**Chat sessions:** Persistent conversation storage. List, get, create, archive. Supports tutor, ctf, exam modes.

### `@its/core-api` — Express App Factory
**Lines:** 156 (app-factory) + 35 (rate-limiter) = 191 total
**Dependencies:** express, cors, helmet, compression, express-rate-limit, @its/core-auth, @its/core-ai, @its/core-data, @its/core-survey

Creates a fully configured Express app from a tenant config.

**What it provides:**
- `createApp(config, authStrategy)` — returns `{ app, authMiddleware, requireAdmin, requireRole }`
- Security headers: Helmet (CSP, HSTS, COOP, referrer policy), Permissions-Policy
- CORS with tenant-specific allowed origins
- Body parsing with 50kb limit
- Origin validation for mutations in production (checks against tenant's `allowedHosts`)
- CSRF protection (X-Requested-With header required for POST/PUT/DELETE)
- Global rate limiting: 500 requests / 15 minutes
- Per-user rate limiting: 100 messages / hour (keyed on auth token hash, not user-controlled)
- Auto-mounts core routes: chat, chat sessions, exam (if domain config provided), health check
- Returns auth middleware for tenant to add custom routes

### `@its/core-survey` — Survey Handlers
**Lines:** ~400 across 5 files
**Dependencies:** sanitize-html, exceljs, @its/core-data

Copied from DAP with import paths updated. Handles survey submission, results retrieval, deletion, import (JSON), export (XLSX), admin settings.

### `@its/core-ui` — Shared React Components
**Lines:** ~350 across 8 files
**Dependencies:** react-markdown (peer: react, react-dom)

Generic UI components extracted from DAP:
- **AppShell:** Configurable app shell with tab navigation, error boundary, header/footer slots, chat bubble slot
- **ChatMessage:** Message bubble with markdown rendering (assistant/user)
- **NavTabs:** Tab navigation bar with badges
- **Breadcrumb:** Navigation breadcrumb trail
- **ExportDropdown:** Dropdown button for export actions
- **ConsentGate:** Data collection consent modal
- **tokenHelper:** Token storage and refresh management
- **useSpeech:** Web Speech API wrapper (TTS/STT)
- **fileHelpers:** JSON file download/upload utilities

### `@its/tenant-config` — Configuration Schema
**Lines:** 63
**Dependencies:** None

Validates tenant configuration objects. Enforces required fields (tenantId, displayName, systemPrompt, skillTaxonomy, authStrategy, adminEmail, allowedOrigins, allowedHosts). Validates auth strategy is one of `azure-ad` or `firebase-auth`. Validates skill definitions have id, label, category. Applies defaults for optional fields.

---

## 4. Thrive Tenant App: `apps/thrive/`

### Status: Scaffold — builds and renders, no backend wired

The Thrive app is a working scaffold that proves the multi-tenant model. It has tenant-specific configuration, content, and a minimal UI. It does NOT yet have: Zoho integration, benchmark assessment processing, action plan generation, coaching summary generation, or a Firebase project to deploy to.

### `tenant.config.js`
Defines Thrive as a tenant:
- **tenantId:** `thrive`
- **displayName:** Thrive Network
- **Auth:** Firebase Authentication (email/password)
- **Admin:** admin@thethrivenetworks.org
- **Exam config:** 12 benchmark domains mapped to skills
- **No mode handlers** (no CTF)
- **No context fetchers** (no Zoho integration yet)
- **Allowed hosts:** app.thethrivenetworks.org, thrive-network-prod.web.app

### `content/system-prompt.js`
81-line ALAI system prompt for business accelerator context. Derived from the Thrive AI Proposal. Covers:
- Role definition (Scale Accelerator guide)
- Program description (12-month cohort, 25 participants, July 1 start)
- 12 assessment domains
- Assessment workflow explanation (Zoho → benchmark rules → AI enrichment → action plan)
- AI approach (deterministic rules are not overridden, AI adds industry context)
- Guardrails (no financial projections, no legal advice, disclaimer required, no cross-participant data)
- Learner proficiency adaptation
- Skill tagging metadata instructions

### `content/skill-taxonomy.js`
12 business skill domains across 3 categories:
- **Foundations:** Vision & Goal Setting, Financial Literacy, Legal & Compliance, Leadership & Communication
- **Growth & Revenue:** Marketing & Branding, Sales & Revenue, Customer Retention, Strategic Growth
- **Operations & Systems:** Operations & Processes, HR & Team Building, Tech & Digital, Sustainability

Each skill has: id, category, label, prerequisites, keywords (for skill tagger). Knowledge types defined: procedural (sales, operations, tech), conceptual (vision, legal, strategy, sustainability), applied (everything else).

### `content/benchmark-rules.js`
Stub with 12 monthly benchmark domains. Structure defined, rules arrays empty — awaiting Thrive's actual assessment content (per proposal Section 5.3, content delivery due April 17, 2026).

### `src/App.jsx`
Minimal React app using `AppShell` from `@its/core-ui`. Three tabs (Home, Learn, Community). Learn tab renders the skill taxonomy. Scaffold notice displayed.

---

## 5. DAP Tenant App: `apps/dap/`

### Status: Placeholder — entry point only, not wired to DAP-Cohort-26

Contains only `index.html`, `main.jsx` (one-line placeholder), `vite.config.js`, and `package.json`. The actual DAP-Cohort-26 application lives at `~/projects/DAP-Cohort-26` and is untouched. DAP was included in the monorepo to validate that Turborepo builds two tenant apps in parallel, not to replace the existing deployment.

---

## 6. What ALAI Is Today (in the production DAP-Cohort-26)

ALAI (Ally-Led Adaptive Intelligence) as deployed in DAP-Cohort-26 is a 1,242-line orchestration layer (`claude-proxy.js`) that mediates between students and Claude. It is NOT a thin chatbot wrapper.

**What it does per message:**
1. Validates input, loads persistent session history
2. Checks for CTF flag submissions (SHA-256 hash matching against active challenge config)
3. If CTF flag found: initiates multi-exchange conversational evaluation (up to 3 rounds of Socratic probing before final scoring with partial credit + appeal system)
4. Fetches consent-gated BKT skill context (proficiency tiers: beginner/intermediate/advanced)
5. Fetches cert readiness (percentage ready for target certifications)
6. Fetches spaced review due items
7. Fetches recording consumption context (which meetings the student watched, how far, which chapters)
8. Fetches meeting transcript from SharePoint (keyword-matched to student's question, cached 7 days)
9. Injects CTF guide content based on active flag (section-aware: Mission Briefing, Recommended Approach, Explain It Back, Dig Deeper)
10. Injects walkthrough activity state (reading/idle/stuck/speaking)
11. Calls Claude Sonnet with prompt caching
12. Parses skill metadata from response
13. Fire-and-forget: usage tracking, audit logging, conversation logging, source-aware BKT update with decay + EMA, observation logging, spaced rep enrollment, SM-2 advancement

**What it is NOT (the gaps):**
- No tool use — Claude gets one assembled prompt, gives one answer. Cannot query student data mid-conversation.
- No multi-step planning — cannot decide "explain first, then quiz" across messages.
- No cross-session memory — each session starts fresh (mastery scores persist, conversation context does not carry over between sessions).
- No proactive intervention — waits for student input, never initiates.
- No RAG — guide content is statically injected by flag ID, not dynamically retrieved based on the question's semantic meaning.

**The interaction model gap:** The BKT/EMA/decay/ceiling machinery is real educational technology running silently behind the conversation. The student never feels it because Claude has no ability to act on it. Claude sees a proficiency snapshot but can't investigate, can't change strategy, can't pull up history. The intelligence is in the plumbing, not in the conversation.

---

## 7. What's Missing — From Here to a Full AI Tutoring Platform

### Infrastructure (CI/CD, Security, Deployment)
- [ ] No CI/CD pipeline — no GitHub Actions, no automated tests on push, no security scanning
- [ ] No supply chain hardening — using npm (not pnpm), no frozen lockfiles in CI, no SHA-pinned actions
- [ ] No secrets management — no .env.example files, no GitHub Secrets configured
- [ ] No deployment automation — no Firebase preview channels, no staging environment
- [ ] No monitoring — no error tracking, no uptime monitoring, no alerting
- [ ] No Supabase setup — core-data speaks Firestore only, no Supabase adapter exists
- [ ] Ship Safe audit returned 61.4/100 (C) — 12 AI/LLM security findings, 23 API security findings, 12 dependency CVEs

### ALAI Agent Capabilities
- [ ] Tool use — let Claude query mastery data, submission history, and due reviews mid-conversation via Anthropic tool_use API
- [ ] Adaptive scaffolding — progressive hint revelation based on skill level (hint ladder data exists in DAP, needs wiring)
- [ ] Cross-session memory — inject previous session topics/outcomes on session start
- [ ] Proactive nudges — when activity state shows "stuck" (4+ min idle), inject system message offering help
- [ ] RAG for content retrieval — replace static guide injection with vector search against embedded content
- [ ] Multi-agent orchestration — separate agents for assessment, coaching, content, safety

### Thrive-Specific
- [ ] Zoho CRM integration — assessment ingestion via Zoho API (webhook or polling)
- [ ] Benchmark rules engine — deterministic if/then logic from Thrive's assessment content
- [ ] Action plan generator — structured output from benchmark results + AI industry context
- [ ] Coaching summary generator — staff-facing output highlighting key risks per participant
- [ ] Admin dashboard — participant view (attendance %, homework %, graduation reqs), cohort rollup
- [ ] Benchmark config interface — admin can update rules without code changes
- [ ] Per-participant data export + bulk CSV
- [ ] Firebase project creation and deployment
- [ ] Content from Thrive (assessment content due April 17, 2026 per proposal)

---

## 8. Thrive Proposal Summary

**Client:** Thrive Network (Heather McBroom, Executive Director)
**Program:** Scale Accelerator — 12-month cohort, 25 participants, starts July 1, 2026
**Budget:** $20,000 cap (one-time implementation + first month)
**Deadline:** System complete and tested by June 15, 2026

**Stage 1 (must-have):** Assessment engine — Zoho ingestion → benchmark rules → AI enrichment → action plans + coaching summaries + dashboard
**Stage 3 (if budget allows):** Bilingual intake bot — Facebook Messenger FAQ + smart voicemail + routing
**Stage 2 (future grant):** Alumni business health monitoring — document upload + AI review + dashboard

**Content dependency:** Thrive delivers assessment content + benchmark rules by April 17, 2026. Content lock date: May 2. Pilot (Heather): May 4-15. Go-live: June 15.
