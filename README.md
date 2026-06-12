# SENTRY

**SENTRY** is a phishing email detection and analysis system. It guards Gmail users in real time through a Chrome extension, and gives security analysts a dashboard to triage what the model flags. The name reflects the role: SENTRY watches inbound mail, classifies it with an LLM-driven pipeline, and reports anything suspicious to a human reviewer.

This is the meta-repository — the front door for the whole system. The two runtime components live in sibling repos:

- **`Api/`** — [sentry-api](https://github.com/kudzaiprichard/sentry-api): the FastAPI backend. Owns the inference pipeline, both auth surfaces, and the database.
- **`dashboard/`** — [sentry_frontend](https://github.com/kudzaiprichard/sentry_frontend): the Next.js admin panel. Owns analyst triage, stats, user management, and install governance.

A third client — the Gmail Chrome extension — is not in this monorepo; it talks to the API on dedicated extension routes.

---

## Why SENTRY exists

Phishing is the inbox problem that no static filter solves cleanly: senders rotate domains, links are wrapped in shorteners, and brand impersonation lives inside HTML-rendered pages a few hops away from the original click. SENTRY answers it with three guarantees:

1. **A user-visible verdict in under a second.** The extension's `analyze` call returns Stage 1 of the pipeline (a Groq classification) inline. The user sees a phishing / suspicious / legitimate badge before they decide to click.
2. **A deeper second pass that is allowed to take its time.** The same email is handed off to a background pipeline that resolves shortened URLs, scrapes the destination pages with a three-attempt escalation (httpx → browser-UA httpx → headless Playwright), batches the cleaned text to Gemini for risk analysis, and aggregates the result back into the email's record. The user is unblocked while this runs.
3. **A human-friendly review surface.** Every verdict — Stage 1 and Stage 4 — is stored with confidence, reasoning, risk factors, link breakdowns, and per-page analysis. Analysts open the dashboard to filter, drill in, attach manual review notes, override a verdict, or re-queue an email.

It is **for** security teams supporting Gmail-based organisations: SOC analysts who triage flagged mail, IT admins who govern which installs may submit emails, and the small group of users who interact with the model output directly through the extension.

What makes it work as a system is the split between **fast and live** (extension surface) and **slow and thorough** (background pipeline + dashboard review), running through one API and one schema.

---

## System architecture

```mermaid
graph TB
    Analyst([Analyst / Admin])
    User([Gmail user])

    subgraph Frontend["sentry_frontend (Next.js 16, React 19)"]
        Dashboard["Admin dashboard<br/>localhost:3000"]
    end

    subgraph Extension["Chrome extension (separate repo)"]
        ExtClient["Gmail content script + background"]
    end

    subgraph Backend["sentry-api (FastAPI)"]
        DashAPI["Dashboard surface<br/>/auth · /users · /inference · /extension/installs"]
        ExtAPI["Extension surface<br/>/auth/extension · /emails/analyze · /health"]
        Detector["InferenceClassificationDetector<br/>(adapter, app.state.detector)"]
        Submitter["InferencePipelineSubmitter<br/>(adapter, app.state.pipeline_submitter)"]
        Pipeline["Background pipeline_runner<br/>asyncio tasks · 4 stages"]
        Cleanup["Token cleanup loops<br/>(JWT + install token)"]
    end

    subgraph Data["Postgres"]
        Users[("users · tokens")]
        Installs[("extension_installs ·<br/>extension_tokens ·<br/>extension_analyse_events")]
        Emails[("emails · links ·<br/>page_analyses")]
    end

    subgraph LLM["External LLMs"]
        Groq["Groq<br/>llama-3.1-8b-instant"]
        Gemini["Google Gemini<br/>gemini-1.5-flash"]
    end

    subgraph Web["Open web"]
        Pages["Destination pages<br/>(scraped via httpx + Playwright)"]
    end

    Analyst --> Dashboard
    User --> ExtClient

    Dashboard -- JWT bearer --> DashAPI
    ExtClient -- install token bearer --> ExtAPI

    DashAPI --> Users
    DashAPI --> Emails
    DashAPI --> Installs
    ExtAPI --> Installs
    ExtAPI --> Emails

    ExtAPI --> Detector
    Detector --> Groq
    ExtAPI --> Submitter
    Submitter --> Pipeline

    DashAPI --> Pipeline
    Pipeline --> Groq
    Pipeline --> Pages
    Pipeline --> Gemini
    Pipeline --> Emails

    Cleanup --> Users
    Cleanup --> Installs
```

---

## Component breakdown

| Component | Folder | Tech | GitHub |
|---|---|---|---|
| Backend API | `Api/` | Python 3.11, FastAPI 0.136, SQLAlchemy 2 (async + asyncpg), Alembic, PyJWT, slowapi, httpx, Playwright, Groq + Gemini SDKs | [kudzaiprichard/sentry-api](https://github.com/kudzaiprichard/sentry-api) |
| Admin dashboard | `dashboard/` | Next.js 16, React 19, TypeScript 5, TanStack Query 5, Zustand, axios, shadcn + Radix + Tailwind 4, recharts, sonner, next-themes | [kudzaiprichard/sentry_frontend](https://github.com/kudzaiprichard/sentry_frontend) |
| Chrome extension | external | Manifest V3, vanilla JS | not in this monorepo |
| Database | runtime | PostgreSQL 14+ (uses `gen_random_uuid()`, `JSONB`) | — |
| External LLMs | runtime | Groq (Stage 1 classifier), Google Gemini (Stage 3 page analysis) | — |

---

## End-to-end flow

There are two entry paths — extension live-analyse and dashboard manual submit — and they converge on the same `emails` table and the same background pipeline.

### Path 1 — Gmail user opens an email

1. The extension's content script extracts `sender / subject / body / auth` from the rendered Gmail message and `POST`s it to `/api/v1/emails/analyze` with its install bearer token.
2. The API's `require_install` dependency validates the token (SHA-256 lookup in `extension_tokens`, install must not be `BLACKLISTED`), bumps `last_seen_at` fire-and-forget, and rate-limits per-token-hash at `60/minute`.
3. `EmailAnalyseService` calls `app.state.detector.predict(...)` — the `InferenceClassificationDetector` adapter (`src/core/inference_detector.py`) runs Stage 1 (Groq) inline and maps the three-way `Classification` to a binary `predicted_label` plus a `phishing_probability`. The response goes straight back to the extension; the user sees the verdict.
4. The same service then calls `app.state.pipeline_submitter.submit(...)` — the `InferencePipelineSubmitter` adapter opens its own DB session, inserts the `Email` row, and `pipeline_runner.spawn`s the four-stage background task. The submission is wrapped in `try/except` so a pipeline failure cannot fail the Stage-1 response.
5. The background task runs Stage 1 again on its own session, then (unless early-exit fires at `≥0.85` confidence) Stage 2 link resolution, Stage 3 batched page analysis, Stage 4 aggregation. Each stage updates `pipeline_stage` and `pipeline_status` on the email row.

### Path 2 — Analyst submits manually from the dashboard

1. Analyst signs in at `/login`. The dashboard hits `POST /auth/login`, stashes both JWTs in cookies, and the axios request interceptor attaches `Authorization: Bearer <access>` on every subsequent call.
2. Analyst opens `/inference?tab=submit` (or `?tab=batch`) and posts to `POST /inference/emails` (or `/inference/emails/batch`). The handler returns `202 Accepted` with the email's id; the same `pipeline_runner.spawn` runs in the background.
3. TanStack Query invalidates `['inference', 'emails']` and `['inference', 'stats']`. The submit page navigates to the new email's detail (`?tab=detail&id=...`); the dashboard charts refresh.

### Convergence — analyst review

1. Analyst opens `/inference?tab=history`, filters by classification / confidence band / date / pipeline status / override trigger, and clicks a row → detail tab.
2. Detail tab calls `GET /inference/emails/{id}` — the email loads with all `links` (selectin-loaded) and any `page_analyses`. Until Stage 4 finalises, `pipeline_status` shows `running`; query refetch picks up the transition to `complete`.
3. Analyst attaches a manual-review note (`POST /inference/emails/{id}/manual-review`) or, if ADMIN, re-queues (`POST /inference/emails/{id}/reanalyze`) or deletes (`DELETE /inference/emails/{id}`).

### Background processes

- `pipeline_runner._in_flight` — module-level `set[asyncio.Task]` keeping pipeline tasks alive against GC.
- `start_token_cleanup()` — purges expired JWTs every `TOKEN_CLEANUP_INTERVAL_SECONDS` (default hourly).
- `start_extension_token_cleanup()` — same for `extension_tokens`.
- On shutdown, `lifespan` cancels both cleanup loops, calls `pipeline_runner.drain(timeout=30.0)` to let in-flight pipelines finish, and disposes the SQLAlchemy engine.

---

## Local development guide

You need to run the API first, then the dashboard. The dashboard's dev mode hardcodes the API base URL to `http://127.0.0.1:8000/api/v1`.

### 1. Prerequisites

- Python 3.11+, Node.js 20+, PostgreSQL 14+
- A Groq API key — <https://console.groq.com/keys>
- A Gemini API key — <https://aistudio.google.com/apikey>
- Optional: `playwright install chromium` if you want Stage 2 attempt-3 scraping to work locally

### 2. Database

Create a Postgres database and capture an `asyncpg` connection string, e.g. `postgresql+asyncpg://postgres:password@localhost:5432/sentry`.

### 3. API

```bash
cd Api
python -m venv venv
venv\Scripts\activate         # Windows
# source venv/bin/activate    # POSIX
pip install -r requirements.txt
cp .env.example .env
# edit DATABASE_URL, JWT_SECRET_KEY, ADMIN_PASSWORD, GROQ_API_KEY, GEMINI_API_KEY
alembic upgrade head
python main.py
```

The API listens on `http://127.0.0.1:8000`. On first boot it seeds an `ADMIN` user from `ADMIN_EMAIL` / `ADMIN_PASSWORD`.

### 4. Dashboard

```bash
cd dashboard
npm install
npm run dev
```

Open `http://localhost:3000`. You will be redirected to `/login`; sign in with the seeded admin credentials.

### Cross-repo dependencies (must be kept in sync)

| Concern | API source of truth | Dashboard mirror |
|---|---|---|
| Access token lifetime | `ACCESS_TOKEN_EXPIRE_MINUTES` env var | `TOKEN_LIFETIMES.ACCESS_TOKEN_MINUTES` in `src/lib/api-core/constants.ts` |
| Refresh token lifetime | `REFRESH_TOKEN_EXPIRE_DAYS` env var | `TOKEN_LIFETIMES.REFRESH_TOKEN_DAYS` |
| Allowed dashboard origin | `CORS_ORIGINS` env var | (no mirror — must include the dashboard's running origin) |
| User roles | enum `Role` in `src/modules/auth/domain/models/enums.py` | `USER_ROLES` constant |
| Wire format | `model_dump(by_alias=True)` produces camelCase | All TS interfaces in `src/lib/api-core/types.ts` and `src/features/*/api/*.types.ts` use camelCase verbatim — no transform layer |
| Refresh request shape | accepts `{refresh_token: ...}` (snake_case) | dashboard sends snake_case explicitly on this one route |

`CORS_ALLOW_CREDENTIALS=true` is required on the API because the dashboard ships cookies on every request; consequently `CORS_ORIGINS` cannot be `*`.

---

## Runtime sequence — login, browse, refresh

```mermaid
sequenceDiagram
    autonumber
    actor Analyst
    participant DB as Dashboard (browser)
    participant API as sentry-api
    participant PG as Postgres
    participant Pipe as pipeline_runner

    Analyst->>DB: open / (redirects to /login)
    DB->>API: POST /api/v1/auth/login {email, password}
    API->>PG: verify user, revoke prior tokens, insert access+refresh rows
    API-->>DB: 200 ApiResponse{success, value:{user, tokens}}
    DB->>DB: setAuthCookies() — access (30m), refresh (7d)

    Analyst->>DB: navigate to /inference
    DB->>API: GET /api/v1/auth/me  (Authorization: Bearer access)
    API-->>DB: 200 ApiResponse{value: UserDto}
    DB->>API: GET /api/v1/inference/stats/summary
    API->>PG: aggregate query
    API-->>DB: 200 ApiResponse{value: SummaryStatsResponse}
    DB->>API: GET /api/v1/inference/emails?page=1&pageSize=20
    API->>PG: paginated select
    API-->>DB: 200 PaginatedResponse{value:[...], pagination:{page,total,pageSize,totalPages}}

    Note over DB,API: 30 minutes pass — access token expires
    Analyst->>DB: click row → detail
    DB->>API: GET /api/v1/inference/emails/{id}
    API-->>DB: 401 ApiResponse{success:false, error:{code:"AUTH_FAILED", status:401}}
    Note over DB: axios interceptor queues request, fires refresh
    DB->>API: POST /api/v1/auth/refresh {refresh_token}
    API->>PG: revoke prior tokens, insert new pair
    API-->>DB: 200 ApiResponse{value:{accessToken, refreshToken}}
    DB->>DB: setAuthCookies()
    DB->>API: GET /api/v1/inference/emails/{id} (replayed with new bearer)
    API-->>DB: 200 ApiResponse{value: EmailDetailResponse}

    Analyst->>DB: POST manual review note
    DB->>API: POST /api/v1/inference/emails/{id}/manual-review
    API->>PG: update email row (manual_review_*, override_classification)
    API-->>DB: 200 ApiResponse{value: EmailDetailResponse, message:"Manual review recorded"}

    Note over DB,Pipe: For an extension submission, Path 1 above runs in parallel
    Pipe-->>PG: write pipeline_stage transitions, links, page_analyses, final_classification
    Note over DB,API: detail tab refetch picks up pipeline_status: "complete"

    Analyst->>DB: click logout
    DB->>API: POST /api/v1/auth/logout
    API->>PG: revoke all of this user's tokens
    API-->>DB: 200 ApiResponse{value:null}
    DB->>DB: clearAuthAndRedirect → /login
```

Error paths in this flow:
- A second concurrent 401 piggy-backs on the in-flight refresh (axios `failedQueue`) instead of firing a second one.
- If `/auth/refresh` itself returns 401, the dashboard clears cookies and redirects to `/login` — except when the current path is already `/login`, to avoid a reload loop on a stale session.
- Network errors (no response) are surfaced to the caller as `ApiError{code: "NETWORK_ERROR", status: 0}` for the UI to render as a toast.

---

## More

- **Contributing**: see [CONTRIBUTING.md](./CONTRIBUTING.md).
- **Deeper architecture**: see [ARCHITECTURE.md](./ARCHITECTURE.md) for data models, full API contract, and decisions about what lives where.
