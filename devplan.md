# DevPlan — Personalized Learning Plan Generator

> **Source product brief:** `idea2.txt`
> **Status:** DRAFT — not yet approved for execution
> **Date:** 2026-06-13
> **Target repo (TBD):** `D:\Freelance\fxrnd_kodaepik\learnai\` (proposed; see §3 Open Decisions)

---

## 1. Executive Summary

Build a web app where a learner submits ~10 example study sessions (with rich descriptions of topic, materials, duration, active/passive mix). An AI model (Claude) infers the learner's study style and extrapolates a full 12-month plan of ~250 sessions. The learner reviews/approves/rejects individual sessions. The system calculates 10 "learning metrics" (retention estimate, coverage %, time-to-mastery, etc.).

**Reuse of trading-app rank 1 patterns:** identical to BacktestAI (Next.js 14 + FastAPI + PostgreSQL + Celery + Redis + Claude). All architectural patterns from `backtestai/CLAUDE.md` apply directly — `AppError` hierarchy, `RequestIDMiddleware`, `configure_logging()`, `domains/<name>/` convention, `lib/api.ts` typed client, `useQuery`/`useMutation` over `useEffect`+`fetch`.

---

## 2. Goals & Non-Goals

### 2.1 MVP Goals (Phases 1–4)
- Public marketing site (nav, hero, USPs, workflow animation, testimonials, pricing, contact, footer)
- Email/password + Google/GitHub auth via Supabase
- User dashboard with sidebar
- Plan CRUD (one main subject + 0–5 supporting subjects, 12-month range)
- 10+ seed sessions per plan with rich descriptions
- Async AI plan generation via Celery + Claude
- Session review list (approve / reject / edit / note)
- 10 learning metrics engine
- Knowledge curve + topic coverage heatmap
- PDF + CSV + Notion export
- Stripe-powered 3-tier pricing

### 2.2 V2 Goals
- Plan A/B comparison, walk-forward testing, Monte Carlo simulation
- Portfolio learning (multi-subject)
- AI per-session rationale + plan coherence score
- Adaptive rescheduling from logged actual sessions
- FSRS spaced-repetition algorithm integration

### 2.3 V3 Goals
- Public REST API, team workspaces, plan marketplace, PostHog analytics

### 2.4 Non-Goals (out of scope)
- Real-time collaborative editing (V3 only)
- Mobile native apps (responsive web only)
- LMS / classroom-grade (use V3 marketplace instead)
- Live tutoring / chat with AI (no conversational UI in MVP)

---

## 3. Open Decisions (must resolve before Phase 1)

| # | Question | Recommendation |
|---|---|---|
| D1 | Repo location | `D:\Freelance\fxrnd_kodaepik\learnai\` (sibling to `backtestai/`) |
| D2 | Shared infra with backtestai? | No — separate VPS initially. Reuse patterns, not runtime. |
| D3 | Spaced-repetition algorithm | **FSRS v4** (open-source, modern, simpler than SM-2) |
| D4 | Topic tree source | User-defined for MVP, optional PDF syllabus upload in Phase 5 |
| D5 | AI model | Claude Sonnet 4.6 for plan generation (cost/quality balance) |
| D6 | Frontend framework | Next.js 14 App Router (same as backtestai) |
| D7 | Backend framework | FastAPI (same as backtestai) |
| D8 | Database | PostgreSQL 15+ with RLS via Supabase |
| D9 | Auth provider | Supabase Auth (JWT, OAuth, same as backtestai) |
| D10 | Payments | Stripe (3 tiers: Free / Pro / Enterprise) |
| D11 | Email | Resend |
| D12 | Calendar sync | Google Calendar (Phase 5) |
| D13 | Notion export | Notion API (Phase 5) |
| D14 | Anki export | Anki Connect (Phase 5, requires user desktop) |
| D15 | FSRS library | `py-fsrs` Python package or port the algorithm directly |

---

## 4. High-Level Architecture

Identical to backtestai (see `devplan.md` of that project for diagrams). Three diagrams to draft during Phase 1:

1. **VPS overview** — Nginx → Next.js + FastAPI; Celery + Redis; PostgreSQL; external: Claude / Stripe / Resend / Supabase
2. **Auth & request flow** — JWT verification at FastAPI middleware before any handler
3. **Plan generation pipeline** — `POST /plans` → 202 Accepted with `job_id` → Celery worker calls Claude → progress updates via WebSocket → `GET /plans/{id}/sessions` when complete

Critical difference from trading: **session generation is iterative, not one-shot**. Claude may need multiple passes (initial draft → rationale pass → confidence scoring pass). Plan for this in the prompt architecture.

---

## 5. Repository Structure

```
learnai/
├── backend/
│   ├── app/
│   │   ├── main.py                       # FastAPI app, lifespan, CORS
│   │   ├── core/
│   │   │   ├── config.py                 # pydantic-settings (env)
│   │   │   ├── exception_handlers.py     # AppError hierarchy (reused)
│   │   │   ├── logging.py                # configure_logging() (reused)
│   │   │   ├── middleware.py             # RequestIDMiddleware (reused)
│   │   │   ├── security.py               # JWT decode, RBAC dependencies
│   │   │   └── celery_app.py             # Celery instance, broker, routes
│   │   ├── domains/
│   │   │   ├── users/                    # profile, settings, billing
│   │   │   ├── plans/                    # learning_plans, plan CRUD
│   │   │   ├── sessions/                 # seed_sessions, generated_sessions
│   │   │   ├── metrics/                  # 10 learning metrics engine
│   │   │   ├── topics/                   # topic_trees, hierarchy
│   │   │   ├── resources/                # resources, recommendations
│   │   │   └── billing/                  # stripe webhooks, invoices
│   │   ├── services/
│   │   │   ├── claude_client.py          # Anthropic SDK wrapper
│   │   │   ├── fsrs.py                   # FSRS v4 algorithm
│   │   │   ├── metrics_calc.py           # pandas-based metric calculations
│   │   │   └── export.py                 # PDF, CSV, Notion exporters
│   │   ├── api/
│   │   │   ├── routes/                   # public routes (landing)
│   │   │   └── deps.py                   # shared FastAPI dependencies
│   │   └── db/
│   │       ├── base.py                   # SQLAlchemy declarative base
│   │       ├── session.py                # async engine, sessionmaker
│   │       └── migrations/               # Alembic
│   ├── tests/
│   │   ├── conftest.py
│   │   ├── unit/
│   │   └── integration/
│   ├── pyproject.toml
│   ├── requirements.txt
│   ├── alembic.ini
│   └── Dockerfile
├── frontend/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx                      # landing (Phase 1)
│   │   ├── (marketing)/
│   │   │   ├── features/page.tsx
│   │   │   ├── pricing/page.tsx
│   │   │   ├── contact/page.tsx
│   │   │   └── login/page.tsx
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx                # sidebar
│   │   │   ├── page.tsx                  # dashboard home
│   │   │   ├── plans/
│   │   │   │   ├── page.tsx              # list
│   │   │   │   ├── new/page.tsx          # wizard
│   │   │   │   └── [id]/
│   │   │   │       ├── page.tsx          # plan overview
│   │   │   │       ├── sessions/page.tsx # review list
│   │   │   │       ├── metrics/page.tsx
│   │   │   │       └── calendar/page.tsx
│   │   │   ├── journal/page.tsx          # logged actual sessions
│   │   │   ├── compare/page.tsx          # V2
│   │   │   └── settings/page.tsx
│   │   ├── providers.tsx                 # QueryClient + Toaster
│   │   └── error.tsx
│   ├── components/
│   │   ├── ui/                           # shadcn primitives
│   │   ├── marketing/                    # landing sections
│   │   └── dashboard/                    # sidebar, cards
│   ├── features/
│   │   ├── plans/
│   │   │   ├── components/               # PlanWizard, PlanCard
│   │   │   ├── hooks/                    # usePlan, usePlans
│   │   │   ├── api.ts
│   │   │   └── types.ts
│   │   ├── sessions/
│   │   │   ├── components/               # SessionList, SessionRow, SessionDetail
│   │   │   ├── hooks/                    # useSessions, useSessionMutations
│   │   │   ├── api.ts
│   │   │   └── types.ts
│   │   ├── metrics/
│   │   │   ├── components/               # MetricCard, KnowledgeCurve, Heatmap
│   │   │   ├── hooks/
│   │   │   └── api.ts
│   │   └── journal/
│   │       └── ...
│   ├── lib/
│   │   ├── api.ts                        # typed fetch client (reused)
│   │   ├── api-types.ts                  # generated from OpenAPI
│   │   ├── auth.ts                       # Supabase client wrapper
│   │   └── utils.ts
│   ├── public/
│   ├── package.json
│   ├── tailwind.config.ts
│   ├── next.config.mjs
│   └── Dockerfile
├── docker-compose.yml                    # backend + celery + redis + postgres
├── nginx.conf                            # reverse proxy config
├── .env.example
├── CLAUDE.md                             # project-specific guidance
└── README.md
```

---

## 6. Database Schema (PostgreSQL 15+)

### 6.1 Domain 1 — Users, auth & billing (5 tables)

Identical to backtestai:

```sql
CREATE TABLE users (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email         varchar(255) UNIQUE NOT NULL,
  password_hash varchar(255),
  full_name     varchar(255),
  avatar_url    varchar(500),
  timezone      varchar(50) DEFAULT 'UTC',
  role          varchar(20) NOT NULL DEFAULT 'free',  -- free | pro | enterprise | admin
  email_verified boolean NOT NULL DEFAULT false,
  created_at    timestamptz NOT NULL DEFAULT now(),
  updated_at    timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE oauth_accounts (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  provider        varchar(20) NOT NULL,  -- google | github
  provider_user_id varchar(255) NOT NULL,
  access_token    text,
  created_at      timestamptz NOT NULL DEFAULT now(),
  UNIQUE (provider, provider_user_id)
);

CREATE TABLE sessions (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id             uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  refresh_token_hash  varchar(255) NOT NULL,
  device_info         varchar(500),
  ip_address          inet,
  expires_at          timestamptz NOT NULL,
  created_at          timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE subscriptions (
  id                    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id               uuid NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
  stripe_customer_id    varchar(255),
  stripe_subscription_id varchar(255),
  plan                  varchar(20) NOT NULL DEFAULT 'free',
  status                varchar(20) NOT NULL DEFAULT 'active',
  current_period_end    timestamptz,
  created_at            timestamptz NOT NULL DEFAULT now(),
  updated_at            timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE invoices (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  subscription_id     uuid NOT NULL REFERENCES subscriptions(id) ON DELETE CASCADE,
  stripe_invoice_id   varchar(255) UNIQUE NOT NULL,
  amount_cents        integer NOT NULL,
  currency            varchar(3) NOT NULL,
  status              varchar(20) NOT NULL,
  paid_at             timestamptz,
  created_at          timestamptz NOT NULL DEFAULT now()
);
```

### 6.2 Domain 2 — Plans, sessions, metrics, topic tree (6 tables)

```sql
CREATE TYPE plan_status AS ENUM ('draft', 'queued', 'processing', 'completed', 'failed');
CREATE TYPE session_review_status AS ENUM ('pending', 'approved', 'rejected', 'edited', 'completed');
CREATE TYPE session_type AS ENUM ('learn', 'practice', 'review', 'project', 'assessment');
CREATE TYPE resource_type AS ENUM ('book', 'video', 'paper', 'course', 'interactive', 'practice', 'other');

CREATE TABLE learning_plans (
  id                    uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id               uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name                  varchar(255) NOT NULL,
  main_subject          varchar(255) NOT NULL,
  supporting_subjects   varchar(255)[] NOT NULL DEFAULT '{}',
  range_start           date NOT NULL,
  range_end             date NOT NULL,
  weekly_hours_target   numeric(4,1) NOT NULL DEFAULT 10.0,
  target_outcome        text,
  strategy_description  text NOT NULL,
  status                plan_status NOT NULL DEFAULT 'draft',
  celery_task_id        varchar(255),
  progress_pct          smallint NOT NULL DEFAULT 0,
  created_at            timestamptz NOT NULL DEFAULT now(),
  completed_at          timestamptz,
  CHECK (range_end > range_start),
  CHECK (array_length(supporting_subjects, 1) <= 5)
);

CREATE TABLE topic_trees (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  plan_id             uuid NOT NULL REFERENCES learning_plans(id) ON DELETE CASCADE,
  subject             varchar(255) NOT NULL,
  name                varchar(255) NOT NULL,
  parent_id           uuid REFERENCES topic_trees(id) ON DELETE CASCADE,
  depth               smallint NOT NULL DEFAULT 0,
  estimated_hours     numeric(6,1),
  is_weak_area        boolean NOT NULL DEFAULT false,
  created_at          timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE resources (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  topic_id            uuid REFERENCES topic_trees(id) ON DELETE SET NULL,
  title               varchar(500) NOT NULL,
  url                 varchar(1000),
  type                resource_type NOT NULL,
  estimated_minutes   integer,
  author              varchar(255),
  metadata            jsonb DEFAULT '{}'::jsonb,
  created_at          timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE seed_sessions (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  plan_id             uuid NOT NULL REFERENCES learning_plans(id) ON DELETE CASCADE,
  subject             varchar(255) NOT NULL,
  topic_id            uuid REFERENCES topic_trees(id) ON DELETE SET NULL,
  session_type        session_type NOT NULL DEFAULT 'learn',
  scheduled_at        timestamptz NOT NULL,
  duration_minutes    integer NOT NULL,
  active_pct          smallint NOT NULL DEFAULT 50,  -- 0..100
  passive_pct         smallint NOT NULL DEFAULT 50,
  resources           jsonb NOT NULL DEFAULT '[]'::jsonb,  -- [{type, title, url}]
  description         text NOT NULL,
  is_main_subject     boolean NOT NULL DEFAULT true,
  created_at          timestamptz NOT NULL DEFAULT now(),
  CHECK (active_pct + passive_pct = 100),
  CHECK (duration_minutes > 0)
);

CREATE TABLE generated_sessions (
  id                  uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  plan_id             uuid NOT NULL REFERENCES learning_plans(id) ON DELETE CASCADE,
  subject             varchar(255) NOT NULL,
  topic_id            uuid REFERENCES topic_trees(id) ON DELETE SET NULL,
  session_type        session_type NOT NULL DEFAULT 'learn',
  scheduled_at        timestamptz NOT NULL,
  duration_minutes    integer NOT NULL,
  active_pct          smallint NOT NULL DEFAULT 50,
  passive_pct         smallint NOT NULL DEFAULT 50,
  resources           jsonb NOT NULL DEFAULT '[]'::jsonb,
  fsrs_difficulty     numeric(3,1),  -- 1.0..10.0 (FSRS state)
  fsrs_stability      numeric(6,2),
  fsrs_due_date       timestamptz,    -- next review date (FSRS-computed)
  ai_rationale        text,
  ai_confidence       numeric(3,2),   -- 0.00..1.00
  review_status       session_review_status NOT NULL DEFAULT 'pending',
  user_note           text,
  completed_at        timestamptz,    -- user-logged actual completion
  actual_duration_minutes integer,
  created_at          timestamptz NOT NULL DEFAULT now(),
  CHECK (active_pct + passive_pct = 100)
);

CREATE TABLE plan_metrics (
  id                        uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  plan_id                   uuid NOT NULL UNIQUE REFERENCES learning_plans(id) ON DELETE CASCADE,
  total_sessions            integer NOT NULL DEFAULT 0,
  retention_rate_estimate   numeric(4,3),  -- 0.000..1.000
  topic_coverage_pct        numeric(5,2),  -- 0.00..100.00
  time_to_mastery_hours     numeric(7,1),
  active_passive_ratio      numeric(4,2),
  spaced_rep_score          numeric(4,3),  -- 0.000..1.000
  weak_area_coverage        numeric(5,2),  -- 0.00..100.00
  resource_diversity        numeric(4,3),  -- Shannon entropy 0..1
  practice_density          numeric(5,2),  -- practice sessions / week
  review_frequency          numeric(5,2),  -- review sessions / topic
  depth_progression_score   numeric(4,3),  -- 0.000..1.000
  calculated_at             timestamptz NOT NULL DEFAULT now()
);
```

### 6.3 Indexes (critical for query performance)

```sql
-- auth / hot path
CREATE INDEX ON sessions (refresh_token_hash);
CREATE INDEX ON subscriptions (stripe_customer_id);
CREATE INDEX ON invoices (stripe_invoice_id);

-- plan lookups
CREATE INDEX ON learning_plans (user_id, created_at DESC);
CREATE INDEX ON learning_plans (user_id, status);

-- session review list (most frequent dashboard query)
CREATE INDEX ON generated_sessions (plan_id, scheduled_at);
CREATE INDEX ON generated_sessions (plan_id, review_status);
CREATE INDEX ON generated_sessions (plan_id, session_type);

-- seed sessions
CREATE INDEX ON seed_sessions (plan_id, is_main_subject);

-- topic hierarchy traversal
CREATE INDEX ON topic_trees (plan_id, parent_id);
CREATE INDEX ON topic_trees (subject);

-- FSRS review scheduling (calendar view)
CREATE INDEX ON generated_sessions (fsrs_due_date) WHERE review_status IN ('approved', 'edited');
```

### 6.4 Row-Level Security (RLS)

Same pattern as backtestai. Enable on every domain table; policies restrict `user_id = auth.uid()`.

```sql
ALTER TABLE learning_plans ENABLE ROW LEVEL SECURITY;
CREATE POLICY plan_owner ON learning_plans
  USING (user_id = current_setting('app.current_user_id')::uuid);
-- ... repeat for generated_sessions, seed_sessions, topic_trees, plan_metrics
```

---

## 7. API Design

All endpoints under `/api/v1/`. Standard error shape `{code, message, details, request_id}`. JWT in `Authorization: Bearer <token>`.

### 7.1 Auth (Phase 1)
- `POST /auth/register` — email, password, full_name
- `POST /auth/login` — returns `{access_token, refresh_token}`
- `POST /auth/refresh` — rotate refresh token
- `POST /auth/logout` — revoke session
- `POST /auth/verify-email` — token from email link
- `POST /auth/forgot-password` / `POST /auth/reset-password`
- `GET /auth/oauth/{provider}/start` — redirect to OAuth
- `GET /auth/oauth/{provider}/callback`

### 7.2 Users (Phase 1)
- `GET /users/me` — current profile
- `PATCH /users/me` — update name, timezone, avatar
- `GET /users/me/sessions` — list active devices
- `DELETE /users/me/sessions/{id}` — revoke device

### 7.3 Learning Plans (Phase 2)
- `GET /plans` — list user's plans (paginated, filter by status)
- `POST /plans` — create draft plan
- `GET /plans/{id}` — full plan with topics
- `PATCH /plans/{id}` — update name, description, date range
- `DELETE /plans/{id}` — soft delete
- `POST /plans/{id}/submit` — start AI generation (returns 202 + `job_id`)
- `GET /plans/{id}/status` — `{status, progress_pct}` for polling
- `WS /plans/{id}/progress` — WebSocket progress stream

### 7.4 Topic Trees (Phase 2)
- `POST /plans/{id}/topics` — bulk import topic tree (JSON)
- `GET /plans/{id}/topics` — full tree (nested JSON)
- `PATCH /topics/{id}` — mark as weak area, update estimated hours
- `POST /plans/{id}/topics/import-syllabus` — PDF upload (Phase 5)

### 7.5 Seed Sessions (Phase 2)
- `GET /plans/{id}/seed-sessions`
- `POST /plans/{id}/seed-sessions` — bulk create (required before submit)
- `PATCH /seed-sessions/{id}`
- `DELETE /seed-sessions/{id}`

### 7.6 Generated Sessions (Phase 3)
- `GET /plans/{id}/sessions` — paginated, filter by `review_status`, `session_type`, `scheduled_at` range
- `PATCH /sessions/{id}` — approve / reject / edit / note
- `POST /sessions/{id}/complete` — log actual completion (body: `actual_duration_minutes`, optional `user_note`)
- `POST /plans/{id}/sessions/bulk-review` — bulk approve/reject by date range

### 7.7 Metrics (Phase 3)
- `GET /plans/{id}/metrics` — current snapshot
- `POST /plans/{id}/metrics/recalculate` — force refresh (usually automatic)

### 7.8 Journal (Phase 4)
- `GET /journal` — all logged actual completions across plans
- `GET /journal/streak` — current daily streak

### 7.9 Resources (Phase 5)
- `GET /resources?topic_id=...` — recommended resources for topic
- `POST /resources` — user-added resource

### 7.10 Billing (Phase 5)
- `POST /billing/checkout` — Stripe checkout session
- `POST /billing/portal` — Stripe Customer Portal redirect
- `POST /billing/webhook` — Stripe webhook handler
- `GET /billing/invoices`

### 7.11 V2 Endpoints (post-MVP)
- `POST /plans/{id}/variants` — create A/B variant
- `GET /plans/{id}/variants` — list variants
- `POST /plans/{id}/walk-forward` — run walk-forward test
- `POST /plans/{id}/monte-carlo` — run Monte Carlo (returns 202)
- `GET /plans/{id}/monte-carlo/{run_id}` — results
- `POST /portfolio` — multi-subject portfolio
- `GET /portfolio/{id}/synergy` — cross-subject synergy matrix
- `POST /plans/{id}/reschedule` — adaptive reschedule based on logged actuals

### 7.12 V3 Endpoints
- `GET /api/v2/external/plans` — public REST API
- `POST /api/v2/external/plans` — submit plan job
- `GET /workspaces/{id}` — team workspace
- `GET /marketplace/plans` — published plans
- `POST /marketplace/plans/{id}/purchase`

---

## 8. Frontend Structure

### 8.1 Routing

- `app/(marketing)/*` — server components, SSR for SEO
- `app/(dashboard)/*` — client components, RSC for initial data fetch
- `app/login`, `app/register` — client components
- Middleware in `middleware.ts` protects `(dashboard)/*` — redirect to `/login` if no session

### 8.2 State Management

- **Server state:** TanStack Query (`staleTime: 30_000`)
- **Auth state:** Supabase client (cookies-based session)
- **Local UI state:** Zustand (sidebar collapsed, wizard step, calendar view)
- **Form state:** React Hook Form + Zod resolvers
- **No `useEffect` + `fetch`** — enforced via lint rule

### 8.3 Key Components

| Component | Path | Notes |
|---|---|---|
| `PlanWizard` | `features/plans/components/` | 3-step wizard (goal → strategy → seed sessions) |
| `SeedSessionForm` | `features/plans/components/` | Rich form with topic picker, duration slider, active/passive toggle |
| `SessionList` | `features/sessions/components/` | Virtualized table, 250 rows; columns: date, subject, topic, type, duration, status, confidence |
| `SessionRow` | `features/sessions/components/` | Inline approve/reject buttons, click opens detail drawer |
| `SessionDetailDrawer` | `features/sessions/components/` | Full session info, AI rationale, resources, FSRS state, edit form |
| `KnowledgeCurve` | `features/metrics/components/` | Recharts LineChart, 52-week x-axis, dual y-axis (hours + mastery) |
| `CoverageHeatmap` | `features/metrics/components/` | Custom grid (rows = topics, cols = weeks), color = session density |
| `MetricsCard` | `features/metrics/components/` | 10 cards in a responsive grid; tooltip explains each metric |
| `PlanCalendar` | `features/plans/components/` | FullCalendar month/week view, drag-to-reschedule, color-coded by subject |
| `WorkloadChart` | `features/metrics/components/` | Stacked bar chart, weekly hours by session type |

### 8.4 shadcn/ui Primitives Required

`button`, `input`, `textarea`, `select`, `dialog`, `drawer`, `tabs`, `card`, `table`, `badge`, `tooltip`, `toast` (sonner), `slider`, `switch`, `calendar`, `popover`, `dropdown-menu`, `progress`, `separator`, `skeleton`, `form` (RHF), `command` (search), `date-picker`.

---

## 9. AI Prompt Architecture

### 9.1 Three-pass generation (vs one-shot)

Unlike trading (one Claude call returns trades), session generation benefits from three passes:

**Pass 1 — Plan outline (fast, cheap):**
- Input: topic tree, strategy description, seed sessions
- Output: weekly outline (week N: focus on topics X, Y, Z; review of A, B; practice of C)
- Model: Haiku 4.5 (cost-optimized)
- ~10–20 sec

**Pass 2 — Session detail (main work):**
- Input: weekly outline + topic tree + seed sessions + user style profile
- Output: full session list (250 sessions with topic, type, duration, active/passive, resources, FSRS state)
- Model: Sonnet 4.6 (quality)
- ~2–4 min, run as Celery job with progress updates every 25 sessions

**Pass 3 — Rationale + confidence:**
- Input: generated sessions + original style profile
- Output: per-session `ai_rationale` and `ai_confidence`
- Model: Sonnet 4.6
- Batched (50 sessions per call) to keep prompt sizes manageable

### 9.2 Versioned Prompts

Store prompts in `backend/app/services/prompts/v{N}/plan_generation_pass{1,2,3}.md` as Jinja templates. `v1` initially. Track version in `learning_plans` table for A/B comparison.

### 9.3 Caching

Cache generated plans keyed by `(topic_tree_hash, strategy_description_hash, seed_sessions_hash)`. If two users have identical inputs (rare but possible for popular syllabi), serve from cache. Use Redis with 7-day TTL.

---

## 10. Phased Implementation Plan

Each phase has: goal, exit criteria, task list, tests, files to create. Phases are sequential — don't start N+1 until N is done.

### Phase 0 — Project Setup (3–5 days)

**Goal:** greenfield repo, all services runnable locally, CI green.

**Tasks:**
0.1 Create repo at `D:\Freelance\fxrnd_kodaepik\learnai\` (per D1)
0.2 Initialize git, add `.gitignore`, `README.md`, `CLAUDE.md`
0.3 Backend: `pyproject.toml` with FastAPI, SQLAlchemy 2.0 async, Alembic, Celery, Redis, Anthropic SDK, python-jose, passlib, structlog, pydantic-settings
0.4 Frontend: `create-next-app` with TypeScript, App Router, Tailwind, shadcn/ui init
0.5 `docker-compose.yml`: postgres:15, redis:7, backend, celery-worker, celery-beat
0.6 `nginx.conf` for local dev
0.7 `.env.example` with all required vars (DATABASE_URL, REDIS_URL, ANTHROPIC_API_KEY, SUPABASE_URL, SUPABASE_ANON_KEY, STRIPE_SECRET_KEY, etc.)
0.8 Set up Alembic, create initial migration with users + auth tables
0.9 Copy rank 1 patterns from backtestai: `core/exception_handlers.py`, `core/logging.py`, `core/middleware.py`, `core/security.py`
0.10 Backend: `app/main.py` with lifespan, CORS, exception handlers, middleware
0.11 Backend health check: `GET /health` returns `{status: "ok"}`
0.12 Frontend: `app/page.tsx` minimal "LearnAI" landing
0.13 GitHub Actions: lint (ruff + eslint), type-check (mypy + tsc), test (pytest + vitest), build

**Exit criteria:** `docker-compose up` brings up all services; `curl localhost:8000/health` returns 200; `curl localhost:3000` returns Next.js page.

**Tests:**
- Backend: `tests/unit/test_health.py`
- Frontend: smoke test that `app/page.tsx` renders without errors

---

### Phase 1 — Public Site + Auth (Weeks 1–3)

**Goal:** marketing site live, users can sign up, log in, and see an empty dashboard.

**Sub-phase 1A — Public marketing site:**
1.1 `app/(marketing)/page.tsx` — landing page
1.2 `components/marketing/Navbar.tsx` — sticky, responsive, CTA
1.3 `components/marketing/Hero.tsx` — "Get a personalized 12-month learning plan in minutes, not weeks"
1.4 `components/marketing/USPs.tsx` — 4 cards (AI learns your style, 12 months from 10 examples, FSRS built-in, pro-grade metrics)
1.5 `components/marketing/WorkflowAnimation.tsx` — Framer Motion, 4 steps
1.6 `components/marketing/Testimonials.tsx` — 3–6 cards
1.7 `components/marketing/Pricing.tsx` — 3 tiers, monthly/annual toggle (Stripe wired in Phase 5)
1.8 `components/marketing/ContactForm.tsx` — RHF + Zod, Resend integration
1.9 `components/marketing/Footer.tsx` — links, legal, socials
1.10 SEO: metadata in each page, OG images, sitemap.xml, robots.txt
1.11 Lighthouse score ≥ 90 on landing

**Sub-phase 1B — Auth:**
1.12 `POST /auth/register` — Supabase auth, email verification email via Resend
1.13 `POST /auth/login` — returns JWT
1.14 `POST /auth/refresh` — token rotation
1.15 `POST /auth/forgot-password` / `POST /auth/reset-password`
1.16 `GET /auth/oauth/google/start` + callback
1.17 `GET /auth/oauth/github/start` + callback
1.18 `POST /auth/logout` — revoke session
1.19 Frontend: `app/(auth)/login/page.tsx`, `register/page.tsx`, `forgot-password/page.tsx`, `reset-password/page.tsx`
1.20 Frontend: `lib/auth.ts` — Supabase client wrapper, `useSession()` hook
1.21 Frontend: `middleware.ts` — protect `(dashboard)/*` routes
1.22 Frontend: `app/(dashboard)/layout.tsx` — sidebar, top nav, user menu
1.23 Frontend: `app/(dashboard)/page.tsx` — empty dashboard with "Create your first plan" CTA
1.24 Frontend: `app/(dashboard)/settings/page.tsx` — profile, password change
1.25 ErrorBoundary at root layout, sonner `<Toaster />` in providers

**Exit criteria:** new user can sign up with email, verify email, log in, see empty dashboard. Google OAuth works. Mobile responsive.

**Tests:**
- Backend: `tests/integration/test_auth.py` — register, login, refresh, logout, OAuth callback, rate limiting
- Backend: `tests/unit/test_security.py` — JWT encode/decode, RBAC dependency
- Frontend: vitest + Testing Library for LoginForm, RegisterForm
- E2E (Playwright, Phase 1D if time): sign up → log in → see dashboard

---

### Phase 2 — Dashboard + Plan CRUD (Weeks 4–6)

**Goal:** users can create, list, edit, delete learning plans with topic trees and seed sessions. No AI yet.

**Tasks:**
2.1 `POST /plans` — create draft plan (validates: 1 main subject, 0–5 supporting, range_end > range_start, weekly_hours_target 1–40)
2.2 `GET /plans` — paginated list, filter by status
2.3 `GET /plans/{id}` — full plan with topics
2.4 `PATCH /plans/{id}` — update name, description, date range (only if status = draft)
2.5 `DELETE /plans/{id}` — soft delete
2.6 `POST /plans/{id}/topics` — bulk import topic tree (JSON array, validates no cycles in parent_id)
2.7 `GET /plans/{id}/topics` — full tree as nested JSON
2.8 `PATCH /topics/{id}` — mark weak area, update estimated_hours
2.9 `POST /plans/{id}/seed-sessions` — bulk create (validates ≥ 10 with `is_main_subject=true` before allowing submit later)
2.10 `GET /plans/{id}/seed-sessions`
2.11 `PATCH /seed-sessions/{id}` / `DELETE /seed-sessions/{id}`
2.12 Frontend: `app/(dashboard)/plans/page.tsx` — list with status badges
2.13 Frontend: `features/plans/components/PlanCard.tsx`
2.14 Frontend: `features/plans/hooks/usePlans.ts` — useQuery
2.15 Frontend: `app/(dashboard)/plans/new/page.tsx` — 3-step wizard
2.16 Frontend: `features/plans/components/PlanWizard/Step1Goal.tsx` — main subject + supporting + date range
2.17 Frontend: `features/plans/components/PlanWizard/Step2Strategy.tsx` — rich text (Tiptap or react-markdown editor)
2.18 Frontend: `features/plans/components/PlanWizard/Step3SeedSessions.tsx` — interactive form, topic tree picker, duration slider, active/passive toggle
2.19 Frontend: `features/plans/components/TopicTreeEditor.tsx` — nested drag-drop (dnd-kit), add/edit/delete topics
2.20 Frontend: `features/plans/components/SeedSessionForm.tsx` — form with all fields
2.21 Frontend: `features/sessions/api.ts` — typed mutations
2.22 Frontend: `app/(dashboard)/plans/[id]/page.tsx` — plan overview, summary cards, "Start AI generation" button (disabled until ≥10 seed sessions)
2.23 RBAC: free tier limited to 1 plan; pro unlimited; enterprise unlimited + team

**Exit criteria:** user can create plan, add topic tree, add 10+ seed sessions, see plan overview, see "Start AI generation" button enabled. Plan persists across sessions.

**Tests:**
- Backend: `tests/integration/test_plans.py` — full CRUD, validation (date range, subject count, seed session count)
- Backend: `tests/integration/test_topics.py` — tree import, cycle detection, weak area toggle
- Backend: `tests/integration/test_seed_sessions.py` — CRUD, is_main_subject filtering
- Frontend: vitest for PlanWizard steps, TopicTreeEditor, SeedSessionForm
- E2E: create plan → add 10 sessions → see overview

---

### Phase 3 — AI Plan Generation (Weeks 7–10)

**Goal:** user clicks "Start AI generation", Celery worker calls Claude in 3 passes, sessions appear in review list.

**Tasks:**
3.1 `core/celery_app.py` — Celery instance, routes to `domains.plans.tasks`
3.2 `services/claude_client.py` — Anthropic SDK wrapper, retry logic, token tracking
3.3 `services/prompts/v1/plan_generation_pass1.md` — weekly outline (Jinja)
3.4 `services/prompts/v1/plan_generation_pass2.md` — session detail
3.5 `services/prompts/v1/plan_generation_pass3.md` — rationale + confidence
3.6 `domains/plans/tasks.py` — `generate_plan(plan_id)` Celery task, orchestrates 3 passes
3.7 Pass 1: load plan + topics + seed sessions + strategy → call Haiku → parse weekly outline JSON → store in Redis (transient)
3.8 Pass 2: load weekly outline + plan → call Sonnet in batches (4-week batches) → parse session list → bulk insert to `generated_sessions`, update `progress_pct` every batch
3.9 Pass 3: load all generated sessions + style profile → call Sonnet in batches (50 sessions/call) → update `ai_rationale` + `ai_confidence` per row
3.10 Update `learning_plans.status` transitions: `draft` → `queued` → `processing` → `completed` (or `failed`)
3.11 Compute initial FSRS state per session: `fsrs_difficulty=5.0`, `fsrs_stability=1.0`, `fsrs_due_date=scheduled_at + 1 day`
3.12 `POST /plans/{id}/submit` — kicks off Celery, returns 202 + `job_id`
3.13 `GET /plans/{id}/status` — for polling fallback
3.14 `WS /plans/{id}/progress` — WebSocket for real-time progress (FastAPI WebSocket + Redis pub/sub)
3.15 Frontend: `features/plans/components/GenerationProgress.tsx` — progress bar, current pass indicator
3.16 Frontend: `app/(dashboard)/plans/[id]/page.tsx` — subscribe to WebSocket, show progress
3.17 Frontend: toast on completion: "Plan ready — review your sessions"
3.18 Error handling: if any pass fails, mark plan `failed`, store error in `plan_metadata`, allow user to retry
3.19 Rate limiting: 1 active generation per user (per plan); 10 generations per day for free tier
3.20 Cost tracking: log Anthropic API tokens per generation, store in `learning_plans.metadata->cost_cents`
3.21 Cache layer: if `(topics_hash, strategy_hash, seed_sessions_hash)` matches a recent plan, skip pass 1+2 and serve from cache

**Exit criteria:** user submits plan, sees live progress, sessions appear in review list within 5 min. Failed generations surface a clear error + retry button. Progress works for 12-month plans with 250+ sessions.

**Tests:**
- Backend: `tests/unit/test_claude_client.py` — mock Anthropic, verify retry, token tracking
- Backend: `tests/unit/test_prompts.py` — Jinja templates render correctly with sample inputs
- Backend: `tests/integration/test_plan_generation.py` — submit plan → wait for Celery → assert sessions created
- Backend: `tests/integration/test_progress_stream.py` — WebSocket receives progress updates
- E2E: create plan → submit → wait → see review list

---

### Phase 4 — Session Review + Metrics (Weeks 11–13)

**Goal:** users review/approve/reject sessions, see metrics update live, see knowledge curve and heatmap.

**Sub-phase 4A — Review list:**
4.1 `GET /plans/{id}/sessions` — paginated, filter by `review_status`, `session_type`, `scheduled_at` range
4.2 `PATCH /sessions/{id}` — update `review_status`, `user_note`, override any field
4.3 `POST /sessions/{id}/complete` — log actual completion (`actual_duration_minutes`, optional note)
4.4 `POST /plans/{id}/sessions/bulk-review` — bulk approve/reject by date range or filter
4.5 Frontend: `app/(dashboard)/plans/[id]/sessions/page.tsx` — virtualized table
4.6 Frontend: `features/sessions/components/SessionList.tsx` — TanStack Table
4.7 Frontend: `features/sessions/components/SessionRow.tsx` — inline actions
4.8 Frontend: `features/sessions/components/SessionDetailDrawer.tsx` — full detail + edit form
4.9 Frontend: `features/sessions/components/BulkReviewBar.tsx` — sticky bar with bulk actions
4.10 Frontend: filter chips (status, type, date range, confidence level)
4.11 Frontend: optimistic updates for approve/reject

**Sub-phase 4B — Metrics engine:**
4.12 `services/metrics_calc.py` — pandas-based 10 metric calculations
4.13 `retention_rate_estimate` — based on FSRS intervals vs forgetting curve
4.14 `topic_coverage_pct` — covered_subtopics / total_subtopics in `topic_trees`
4.15 `time_to_mastery_hours` — sum of `duration_minutes` weighted by `topic.estimated_hours` / coverage
4.16 `active_passive_ratio` — sum(active_minutes) / sum(passive_minutes)
4.17 `spaced_rep_score` — how well `fsrs_due_date` distribution follows ideal review intervals
4.18 `weak_area_coverage` — sessions on `topic_trees.is_weak_area=true` / total weak area topics
4.19 `resource_diversity` — Shannon entropy of resource types in session list
4.20 `practice_density` — practice sessions per week
4.21 `review_frequency` — review sessions per unique topic
4.22 `depth_progression_score` — correlation between week index and avg topic depth
4.23 Auto-recalculate on every session mutation (Celery task or sync after PATCH)
4.24 `GET /plans/{id}/metrics` — current snapshot
4.25 `POST /plans/{id}/metrics/recalculate` — manual trigger
4.26 Frontend: `app/(dashboard)/plans/[id]/metrics/page.tsx` — 10 metric cards with tooltips
4.27 Frontend: `features/metrics/components/MetricCard.tsx`
4.28 Frontend: `features/metrics/components/MetricExplainer.tsx` — tooltip with formula + interpretation

**Sub-phase 4C — Visualization:**
4.29 `features/metrics/components/KnowledgeCurve.tsx` — Recharts LineChart, cumulative hours + mastery
4.30 `features/metrics/components/CoverageHeatmap.tsx` — custom grid, color = session density per topic per week
4.31 `features/metrics/components/WorkloadChart.tsx` — stacked bar, weekly hours by session type
4.32 Frontend: `app/(dashboard)/plans/[id]/page.tsx` — embed all 3 charts above the fold
4.33 Frontend: chart interactions (click heatmap cell → filter session list, click curve point → highlight week)

**Exit criteria:** user reviews sessions, sees metrics update in real time as they approve/reject, sees 3 charts that respond to interactions.

**Tests:**
- Backend: `tests/unit/test_metrics_calc.py` — each of 10 metrics with golden inputs/outputs
- Backend: `tests/integration/test_session_review.py` — approve/reject/bulk/complete
- Backend: `tests/integration/test_metrics_recalc.py` — mutation triggers recalc
- Frontend: vitest for SessionList, MetricCard, KnowledgeCurve data bindings
- E2E: review 20 sessions → metrics update → see charts

---

### Phase 5 — Payments + Export + Polish (Weeks 14–16)

**Goal:** paid plans work, exports work, landing page is polished, ready for launch.

**Sub-phase 5A — Stripe:**
5.1 `POST /billing/checkout` — create Stripe Checkout session, return URL
5.2 `POST /billing/portal` — Stripe Customer Portal redirect
5.3 `POST /billing/webhook` — handle `customer.subscription.created/updated/deleted`, `invoice.paid/failed`
5.4 Update `subscriptions` and `invoices` tables from webhook
5.5 RBAC: enforce plan limits on every endpoint (e.g., free = 1 plan, free = 3-month range)
5.6 Frontend: `app/(dashboard)/settings/billing/page.tsx` — current plan, upgrade button, invoice list
5.7 Frontend: `components/marketing/Pricing.tsx` — wire CTAs to checkout
5.8 Test mode end-to-end: upgrade free → pro → enterprise, downgrade, cancel

**Sub-phase 5B — Export:**
5.9 `services/export.py` — three exporters
5.10 `POST /plans/{id}/export/pdf` — generate via WeasyPrint or pdfkit, include cover, metrics, charts (as PNG), full session list
5.11 `POST /plans/{id}/export/csv` — all session fields
5.12 `POST /plans/{id}/export/notion` — push to Notion database (OAuth per user)
5.13 `POST /plans/{id}/export/anki` — convert review sessions to Anki deck via Anki Connect
5.14 `POST /plans/{id}/export/calendar` — generate `.ics` file
5.15 Frontend: `features/plans/components/ExportMenu.tsx` — dropdown with all 5 export types
5.16 Store exports in S3 / Supabase Storage, return signed URL

**Sub-phase 5C — Calendar sync:**
5.17 `services/calendar_sync.py` — Google Calendar OAuth per user
5.18 `POST /users/me/calendar/connect` — OAuth flow
5.19 `POST /plans/{id}/sync-calendar` — push approved sessions to user's Google Calendar
5.20 Daily Celery beat job: sync upcoming sessions for active users
5.21 Frontend: `features/plans/components/CalendarSyncButton.tsx`

**Sub-phase 5D — Polish:**
5.22 Landing page: A/B test hero copy, add 1 video demo
5.23 Onboarding flow for new users: 3-step tutorial overlay
5.24 Empty states for all dashboard pages
5.25 Loading skeletons everywhere
5.26 Error boundaries with retry buttons
5.27 Mobile responsiveness audit (iPhone SE, iPad, desktop)
5.28 Accessibility audit: axe-core, keyboard nav, screen reader
5.29 SEO: structured data, OG images per page
5.30 Analytics: PostHog events for all key actions (signup, plan_created, generation_submitted, session_approved, export_run)
5.31 Sentry: error tracking in frontend + backend
5.32 Documentation: user guide, API docs (auto-generated from OpenAPI), help center

**Exit criteria:** can sign up, create plan, generate, review, see metrics, export, upgrade to pro — all without admin intervention. Landing page Lighthouse ≥ 90. Mobile usable.

**Tests:**
- Backend: `tests/integration/test_billing.py` — webhook handlers, RBAC enforcement
- Backend: `tests/integration/test_export.py` — each export type, signed URL generation
- Frontend: vitest for ExportMenu, Pricing page, Billing page
- E2E: full user journey end-to-end

---

### V2 — Advanced Analytics (post-launch, ~6–8 weeks)

V1 — Plan A/B comparison + walk-forward + Monte Carlo
V2 — Portfolio learning + AI validation + adaptive rescheduling
V3 — Public API + team workspaces + marketplace

(V2 tasks to be detailed after V1 ships; see `idea2.txt` for scope.)

---

## 11. Testing Strategy

### 11.1 Backend (pytest)

- **Unit:** `tests/unit/` — pure functions, no DB. Coverage ≥ 80% for `core/`, `services/`, `metrics/`. All AI prompts have golden-output tests (recorded responses).
- **Integration:** `tests/integration/` — real DB (test DB via Docker), real Redis, mocked Anthropic API. One happy path + one failure path per endpoint.
- **Fixtures:** `conftest.py` provides `async_client`, `db_session`, `test_user`, `auth_headers`, `mock_claude`.
- **Migrations:** every Alembic migration tested with `alembic upgrade head` + `alembic downgrade base` round-trip.

### 11.2 Frontend (vitest + Testing Library)

- **Unit:** `*.test.ts` co-located with component. Render test + interaction test for each.
- **Hooks:** `*.test.ts` for all custom hooks. Mock TanStack Query.
- **Coverage:** ≥ 70% lines, 60% branches.

### 11.3 E2E (Playwright)

- Critical user journeys only (avoid bloat):
  1. Sign up → create plan → submit → see review list
  2. Approve/reject sessions → see metrics update
  3. Export PDF
  4. Upgrade to pro
- Run nightly in CI, before deploy.
- One E2E test per critical flow — total ≤ 8 tests.

### 11.4 Performance

- Locust script: 100 concurrent users, submit plans, review sessions. P95 latency < 500ms for review list, < 5s for plan submit (returns 202).
- Lighthouse CI: landing ≥ 90, dashboard ≥ 75.

---

## 12. Security

- **RLS** enabled on all domain tables
- **JWT** with 15-min access, 7-day refresh, refresh rotation
- **Rate limiting** per IP + per user: 100 req/min general, 10 plan generations/day
- **Input validation** via Pydantic + Zod on all endpoints
- **SQL injection** prevented by SQLAlchemy parameterized queries
- **CORS** restricted to frontend domain only
- **Secrets** in env vars, never in code, rotated quarterly
- **Anthropic API key** server-side only, never sent to frontend
- **Stripe webhook** signature verification
- **PDF export** rate-limited (5/day free, 50/day pro)
- **FSRS state** is user-specific, never shared

---

## 13. Deployment

### 13.1 VPS Setup

Same shape as backtestai:
- Ubuntu 22.04 LTS
- Nginx reverse proxy (port 80/443, Let's Encrypt)
- Docker Compose for `backend`, `celery-worker`, `celery-beat`, `redis`, `postgres`
- Node.js for Next.js (or static export behind Nginx)
- PM2 for process supervision
- Daily backups: `pg_dump` to S3, retain 30 days

### 13.2 CI/CD

GitHub Actions:
- **PR:** lint + type-check + unit tests + integration tests
- **Merge to master:** build Docker images, push to GHCR
- **Manual deploy:** SSH to VPS, `docker compose pull && docker compose up -d`
- **Post-deploy:** smoke test (curl `/health`, login as test user, submit a plan)

### 13.3 Monitoring

- Sentry: errors, performance
- PostHog: product analytics
- Prometheus + Grafana: server metrics (CPU, memory, queue depth, request rate)
- Alerts: PagerDuty on error rate > 1% or queue depth > 100

---

## 14. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Claude API rate limits during 3-pass generation | Medium | High | Queue with backpressure, cache by input hash, allow user to retry |
| 250 sessions per plan = expensive API call | High | High | Cache aggressively, use Haiku for pass 1, batch pass 3, monitor cost per plan |
| FSRS algorithm complexity (correctness) | Medium | Medium | Use battle-tested `py-fsrs` package, unit test with golden outputs |
| Topic tree cycles in user input | Low | Medium | Validate on import, server-side cycle detection |
| 12-month plan generation takes > 5 min | Medium | Medium | WebSocket progress keeps user engaged; allow background retry |
| Stripe webhook ordering issues | Low | High | Idempotency keys, log all webhook events, replay endpoint |
| RLS performance overhead | Low | Low | Index all `user_id` columns, measure with `EXPLAIN ANALYZE` |
| User creates plan with 0 seed sessions and clicks submit | High | Low | Disable submit button until ≥ 10 main-subject seed sessions (client + server validation) |
| Notion/Anki export API changes | Medium | Low | Version our API client, log version mismatches, surface clear errors |

---

## 15. Open Questions (defer until after Phase 0)

- Q1: Should the seed sessions be from real past data (e.g., calendar import) or manually entered? — **MVP: manual entry; V2: Google Calendar import**
- Q2: How to handle subject-specific content moderation (e.g., adult learning topics)? — **MVP: trust user input; V2: report/flag mechanism**
- Q3: Can users collaborate on a plan (e.g., study groups)? — **V3: team workspaces**
- Q4: Pricing for AI cost overage? — **Free: 1 generation/day; Pro: 10/day; Enterprise: unlimited**
- Q5: Multilingual support? — **English only for MVP; V2: i18n**
- Q6: Mobile app? — **Responsive web only; native apps deferred**

---

## 16. Definition of Done (per phase)

A phase is "done" when:
1. All tasks in that phase are merged to `master`
2. All tests pass in CI
3. `docker compose up` runs all services locally
4. The exit criteria are demonstrably met
5. A `CHANGELOG.md` entry is added
6. A demo recording or screenshot is attached to the phase PR
7. The next phase's plan is updated if any tasks shifted

---

## 17. References

- **Source product brief:** `idea2.txt`
- **Rank 1 patterns source:** `backtestai/CLAUDE.md`
- **Existing backtestai plans:** `backtestai/docs/superpowers/plans/2026-05-31-*`
- **FSRS algorithm spec:** https://github.com/open-spaced-repetition/fsrs4anki
- **Anthropic Claude API docs:** https://docs.anthropic.com/
- **Stripe Checkout docs:** https://stripe.com/docs/payments/checkout
- **Notion API docs:** https://developers.notion.com/
- **Anki Connect:** https://foosoft.net/projects/anki-connect/
- **FullCalendar React docs:** https://fullcalendar.io/docs/react

---

> **Status:** Awaiting approval of Open Decisions (D1–D15) and Phases 0–1 task list before any code is written. Phases 2+ will be re-scoped after Phase 1 ships.
