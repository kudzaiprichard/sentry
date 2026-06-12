# SENTRY — Architecture

This document is the deeper technical reference. It covers the data model that crosses the API/dashboard boundary, the full HTTP contract, and the decisions about what lives where.

For the high-level pitch and end-to-end flow, see [README.md](./README.md). For contributing rules, see [CONTRIBUTING.md](./CONTRIBUTING.md).

---

## 1. Where the boundaries are

```mermaid
graph LR
    subgraph DB["dashboard (Next.js)"]
        UI["UI / Zustand auth store"]
        Q["TanStack Query cache"]
        Ax["axios + interceptors<br/>(api-clients.ts)"]
    end

    subgraph API["sentry-api (FastAPI)"]
        subgraph Surf["Presentation"]
            DR["Dashboard routers<br/>auth · users · inference · extension/installs"]
            ER["Extension routers<br/>auth/extension · emails · health"]
        end

        subgraph Core["src/core (composition)"]
            Det["InferenceClassificationDetector<br/>(adapter)"]
            Sub["InferencePipelineSubmitter<br/>(adapter)"]
        end

        subgraph Modules["src/modules (domain)"]
            AuthMod["auth/<br/>users · tokens"]
            InfMod["inference/<br/>emails · links · page_analysis"]
            ExtMod["extension/<br/>extension_installs · extension_tokens ·<br/>extension_analyse_events"]
        end

        subgraph Bg["Background"]
            PR["pipeline_runner<br/>(asyncio tasks)"]
            Cln["token cleanup loops"]
        end
    end

    PG[("PostgreSQL")]

    UI --> Q --> Ax
    Ax -- "Authorization: Bearer <jwt>" --> DR
    DR --> AuthMod
    DR --> InfMod
    DR --> ExtMod
    ER --> ExtMod
    ER -. via app.state .-> Det
    ER -. via app.state .-> Sub
    Det --> InfMod
    Sub --> InfMod
    InfMod --> PR
    PR --> InfMod
    AuthMod --> PG
    InfMod --> PG
    ExtMod --> PG
    Cln --> AuthMod
    Cln --> ExtMod
```

The system has **three** code-level layers across two repos:

1. **Dashboard** consumes the API. It owns nothing the API depends on.
2. **API presentation + modules** speak HTTP and own the database.
3. **API core** holds composition adapters that bridge modules without violating the cross-module rule.

The API/dashboard boundary is HTTP only — there is no shared package, no codegen, no message bus.

---

## 2. Data models that cross the boundary

The API serialises with `model_dump(exclude_none=True, by_alias=True)`, so every wire field is camelCase. The dashboard mirrors these shapes verbatim in `src/lib/api-core/types.ts` and `src/features/*/api/*.types.ts` — there is no transform layer.

### 2.1 Envelopes

```ts
interface ApiResponse<T> {
  success: boolean
  message?: string
  value?: T          // mutually exclusive with `error`
  error?: ErrorDetail
}

interface PaginatedResponse<T> extends ApiResponse<T[]> {
  pagination?: { page: number; pageSize: number; total: number; totalPages: number }
}

interface ErrorDetail {
  title: string
  code: string         // see §4 for all codes the dashboard handles specially
  status: number
  details?: string[]
  fieldErrors?: Record<string, string[]>
}
```

### 2.2 Auth

```ts
interface UserDto {
  id: string                  // uuid
  email: string
  username: string
  firstName: string
  lastName: string
  role: 'ADMIN' | 'IT_ANALYST'
  isActive: boolean
  createdAt: string           // ISO 8601
  updatedAt: string
}

interface AuthTokens { accessToken: string; refreshToken: string }
interface AuthResponseDto { user: UserDto; tokens: AuthTokens }
```

API entities backing these:
- `users` — `email`, `username`, `firstName`, `lastName`, `passwordHash` (bcrypt, never serialised), `role` (enum `ADMIN` / `IT_ANALYST`), `isActive`. Inherits `id` (UUID, server-side `gen_random_uuid()`), `createdAt`, `updatedAt` from `BaseModel`.
- `tokens` — every issued JWT is also persisted so revocation is enforceable. `verify_token` decodes the JWT *and* checks the row.

### 2.3 Inference — emails, links, page analysis

The `Email` row is the central entity; analysts and admins reason in terms of emails.

```ts
type Classification    = 'phishing' | 'suspicious' | 'legitimate'
type OverrideTrigger   = 'page_high_risk' | 'page_medium_risk' | 'all_low' | 'all_failed' | 'early_exit'
type PipelineStatus    = 'pending' | 'running' | 'complete' | 'failed'
type PipelineStage     = 'queued' | 'classification' | 'link_resolution' | 'page_analysis' | 'aggregation' | 'done'
type ResolveStatus     = 'success' | 'failed' | 'timeout' | 'blocked'
type ScrapeStatus      = 'success' | 'blocked' | 'timeout' | 'js_required'
type RiskLevel         = 'high' | 'medium' | 'low'

interface EmailSummaryResponse {
  id: string
  receivedAt: string
  sender: string
  subject: string
  classification: Classification | null         // Stage 1
  confidence: number | null                      // Stage 1
  finalClassification: Classification | null     // Stage 4
  finalConfidence: number | null
  pipelineStatus: PipelineStatus
  overrideTrigger: OverrideTrigger | null
  manualReviewFlag: boolean
  linkCount: number
  createdAt: string
  updatedAt: string
}

interface EmailDetailResponse extends EmailSummaryResponse {
  reasoning: string | null
  riskFactors: string[] | null
  llmModel: string | null
  aggregationNote: string | null
  pipelineStage: PipelineStage
  pipelineError: string | null
  processedAt: string | null
  finalisedAt: string | null
  manualReviewNote: string | null
  manualReviewBy: string | null
  manualReviewAt: string | null
  manualOverrideClassification: Classification | null
  submittedBy: string | null         // user id (dashboard) or null
  submittedByInstall: string | null  // install id (extension) or null
  links: LinkResponse[]
}

interface LinkResponse {
  id: string
  originalUrl: string
  isShortened: boolean
  shortener: string | null
  resolvedUrl: string | null
  resolveStatus: ResolveStatus | null
  redirectHops: number
  intermediateDomains: string[] | null
  httpStatus: number | null
  pageAnalysis: PageAnalysisResponse | null
}

interface PageAnalysisResponse {
  id: string
  pageTitle: string | null
  metaDescription: string | null
  hasLoginForm: boolean
  hasPaymentForm: boolean
  externalDomains: string[] | null
  faviconMatchesDomain: boolean | null
  pagePurpose: string | null
  impersonatesBrand: string | null
  requestsCredentials: boolean
  requestsPayment: boolean
  riskLevel: RiskLevel | null
  riskConfidence: number | null
  riskReasons: string[] | null
  summary: string | null
  scrapeStatus: ScrapeStatus | null
  llmModel: string | null
  analysedAt: string | null
}
```

`Email` carries two FKs that cross module boundaries:
- `submitted_by` → `users.id` (the analyst, when submitted via the dashboard).
- `submitted_by_install` → `extension_installs.id` (the install, when submitted via the extension).

The inference module does not import the extension model — it only references `extension_installs.id` by table name. The composition runs through `src/core/extension_pipeline_submitter.py`.

### 2.4 Extension installs

```ts
type InstallStatus = 'ACTIVE' | 'BLACKLISTED'

interface InstallResponse {
  id: string
  googleSub: string                  // Google account subject (stable user id)
  email: string
  status: InstallStatus
  extensionVersion: string | null
  lastSeenAt: string | null
  blacklistedAt: string | null
  blacklistedBy: string | null       // admin user id
  blacklistReason: string | null
  createdAt: string
  updatedAt: string
}

interface InstallDetailResponse extends InstallResponse {
  environmentJson: Record<string, unknown> | null
  activeTokenCount: number
}

interface ExtensionRegisterResponse {
  token: string                      // raw bearer — only returned at issue
  expiresAt: number                  // epoch millis
  user: { email: string; sub: string }
}

interface AnalyseEventResponse {
  id: string
  installId: string
  predictedLabel: string
  confidenceScore: number
  modelVersion: string
  latencyMs: number
  requestId: string | null
  createdAt: string
}
```

`ExtensionInstall` is keyed by `google_sub` (unique). Tokens are stored as SHA-256 hashes in `extension_tokens` — the raw token is only returned to the caller at issue time and is never recoverable from the database.

### 2.5 Stats responses

```ts
interface SummaryStatsResponse {
  total: number
  classifications: { phishing: number; suspicious: number; legitimate: number }
  pipelineStatus: { pending: number; running: number; complete: number; failed: number }
  manualReviewCount: number
  windowStart: string
  windowEnd: string
}
interface VerdictBucketResponse { bucket: string; phishing: number; suspicious: number; legitimate: number }
interface TriggerCountResponse  { trigger: OverrideTrigger; count: number }
interface ModelUsageResponse    { stage1: { model: string; count: number }[]; stage3: { model: string; count: number }[]; apiCallsEstimated: number }
interface BrandCountResponse    { brand: string; count: number }
```

---

## 3. Full API contract

All routes are under `/api/v1`. Auth column legend: `JWT` = dashboard JWT, `INSTALL` = extension install token, `JWT+ADMIN` = JWT with `role=ADMIN`, `OPTIONAL` = no auth required (extension health).

### 3.1 Auth (dashboard)

| Method | Path | Auth | Body | Response |
|---|---|---|---|---|
| POST | `/auth/register` | — | `{email, username, firstName, lastName, password}` | `201` `AuthResponseDto` |
| POST | `/auth/login` | — | `{email, password}` | `200` `AuthResponseDto` |
| POST | `/auth/refresh` | — | `{refresh_token}` *(snake_case)* | `200` `AuthTokens` |
| POST | `/auth/logout` | JWT | — | `200` `null` |
| GET | `/auth/me` | JWT | — | `200` `UserDto` |
| PATCH | `/auth/me` | JWT | `{firstName?, lastName?, username?}` | `200` `UserDto` |

> Registration is technically open on the API but the dashboard does not expose it — admin user creation goes through `POST /users`. See `dashboard/src/features/auth/api/auth.api.ts`.

### 3.2 Users (dashboard, ADMIN only)

| Method | Path | Body | Response |
|---|---|---|---|
| GET | `/users?page=&pageSize=&role=&isActive=` | — | `PaginatedResponse<UserDto>` |
| GET | `/users/{id}` | — | `UserDto` |
| POST | `/users` | `{email, username, firstName, lastName, password, role}` | `201` `UserDto` |
| PATCH | `/users/{id}` | `{firstName?, lastName?, username?, role?, is_active?}` | `200` `UserDto` |
| DELETE | `/users/{id}` | — | `200` `null` |
| POST | `/users/{id}/activate` | — | `200` `UserDto` |
| POST | `/users/{id}/deactivate` | — | `200` `UserDto` |

### 3.3 Inference (dashboard)

`require_authenticated` is applied at the router level for both inference routers. `require_admin` is applied per route where noted.

| Method | Path | Auth | Body / Query | Response |
|---|---|---|---|---|
| POST | `/inference/emails` | JWT | `{sender, subject, body, receivedAt}` | `202` `SubmitEmailResponse` |
| POST | `/inference/emails/batch` | JWT | `{emails: [...]}` (≤ `INFERENCE_BATCH_MAX`) | `202` `SubmitEmailBatchResponse` |
| GET | `/inference/emails` | JWT | `page, pageSize, classification, minConfidence, maxConfidence, startDate, endDate, pipelineStatus, overrideTrigger, sender` | `PaginatedResponse<EmailSummaryResponse>` |
| GET | `/inference/emails/{id}` | JWT | — | `EmailDetailResponse` |
| GET | `/inference/emails/{id}/status` | JWT | — | `EmailStatusResponse` |
| GET | `/inference/emails/{id}/links` | JWT | — | `EmailLinksResponse` |
| GET | `/inference/links/{id}` | JWT | — | `LinkWithPageResponse` |
| POST | `/inference/emails/{id}/reanalyze` | JWT+ADMIN | `{body}` | `202` `SubmitEmailResponse` |
| POST | `/inference/emails/{id}/manual-review` | JWT | `{note, overrideClassification?}` | `200` `EmailDetailResponse` |
| DELETE | `/inference/emails/{id}` | JWT+ADMIN | — | `200` `null` |
| GET | `/inference/stats/summary?startDate=&endDate=` | JWT | default 30-day window | `SummaryStatsResponse` |
| GET | `/inference/stats/verdicts-over-time?startDate=&endDate=&bucket=day\|week` | JWT | — | `VerdictBucketResponse[]` |
| GET | `/inference/stats/override-triggers?startDate=&endDate=` | JWT | — | `TriggerCountResponse[]` |
| GET | `/inference/stats/model-usage?startDate=&endDate=` | JWT | — | `ModelUsageResponse` |
| GET | `/inference/stats/impersonated-brands?limit=&startDate=&endDate=` | JWT | `limit` 1–50, default 10 | `BrandCountResponse[]` |

### 3.4 Extension auth

Rate-limited via `@limiter.limit`. Limits read from `server.rate_limit.extension_*` (env-tunable).

| Method | Path | Auth | Body / Headers | Response |
|---|---|---|---|---|
| POST | `/auth/extension/register` | — | `X-Google-Access-Token` header (required) + body `{email, sub, environment: {extensionVersion, ...}}`. **3/minute** | `200` `ExtensionRegisterResponse` |
| POST | `/auth/extension/renew` | INSTALL | — **10/minute** | `200` `ExtensionTokenResponse` (rotates the current token) |
| POST | `/auth/extension/logout` | INSTALL (logout-variant; idempotent on already-revoked) | — | `200` `null` |

### 3.5 Extension analyse

| Method | Path | Auth | Body | Response |
|---|---|---|---|---|
| POST | `/emails/analyze` | INSTALL | `{sender, subject, body, auth?}`. **60/minute keyed by token-hash** | `200` `AnalyseEmailResponse` ({phishingProbability, predictedLabel, thresholdUsed, modelVersion, reviewLow, reviewHigh, shouldAlert, message?}) |

`should_alert=true` when phishing probability ≥ `INFERENCE_ALERT_THRESHOLD` (0.90). Body bytes > `INFERENCE_EMAILS_MAX_BODY_BYTES` (102400) → `400 BAD_REQUEST`. Detector not loaded → `503 SERVICE_UNAVAILABLE`.

### 3.6 Extension health

| Method | Path | Auth | Response |
|---|---|---|---|
| GET | `/health` | OPTIONAL install token | `{status: "ok", name, version, model_version}` (note: `model_version` is **snake_case** by contract — read directly by the extension's `utils/api.js::checkConnection`) |

Returns `200` with no body validation when no Authorization is sent. With an Authorization header: validates the install token and returns `401 AUTH_FAILED` if revoked/expired/unknown, `403 NOT_WHITELISTED` if blacklisted. Returns the literal string `"unknown"` for `model_version` when no detector is loaded (never null).

### 3.7 Extension admin (dashboard, ADMIN only)

| Method | Path | Body / Query | Response |
|---|---|---|---|
| GET | `/extension/installs?page=&pageSize=&email=&domain=&status=&version=&lastSeenAfter=&lastSeenBefore=` | — | `PaginatedResponse<InstallResponse>` |
| POST | `/extension/installs/domains/blacklist` | `{domain, reason}` | `BlacklistDomainResponse` |
| GET | `/extension/installs/{id}` | — | `InstallDetailResponse` |
| GET | `/extension/installs/{id}/activity?page=&pageSize=` | — | `PaginatedResponse<AnalyseEventResponse>` |
| POST | `/extension/installs/{id}/blacklist` | `{reason}` | `InstallResponse` |
| POST | `/extension/installs/{id}/unblacklist` | — | `InstallResponse` |
| POST | `/extension/installs/{id}/revoke-tokens` | — | `RevokeTokensResponse` |

> **Route-order gotcha**: `domains/blacklist` is declared before `{install_id}` because FastAPI matches routes in definition order — without that, the literal `domains` would be parsed as a UUID and 422.

---

## 4. Error codes the dashboard handles specially

The full envelope is described in [CONTRIBUTING.md → API contract conventions](./CONTRIBUTING.md#api-contract-conventions). Codes the dashboard reacts to differently:

| Code | Status | Surface | Dashboard behaviour |
|---|---|---|---|
| `AUTH_FAILED` | 401 | both | axios interceptor queues request, hits `/auth/refresh`, replays. Refresh fails → clear cookies + redirect to `/login` (unless already on `/login`) |
| `NOT_WHITELISTED` | 403 | extension | renders the "not authorised" badge in `extension-admin` views |
| `RATE_LIMITED` | 429 | extension | toast + back-off |
| `VALIDATION_ERROR` | 400 | both | `fieldErrors` populates per-field inline errors in forms |
| `NETWORK_ERROR` | 0 (synthesised) | dashboard only | toast — "Unable to connect to the server" |

---

## 5. Internals — what lives where, and why

### 5.1 The two adapters in `src/core/`

The cross-module rule says a module cannot import another module's `domain/` or `internal/` layer. Two real cases violate the spirit of this rule unless we have a composition layer:

- **Live analyse needs the inference Stage-1 classifier.** The extension's `/emails/analyze` endpoint must run Stage 1 inline. The extension module cannot import `EmailClassificationService`. So `src/core/inference_detector.py` defines `InferenceClassificationDetector`, which implements the extension's `Detector` protocol and delegates to `EmailClassificationService`. It also handles the three-way → binary mapping (`PHISHING` / `LEGITIMATE` / `SUSPICIOUS` → `phishing_probability` + `predicted_label` with a review band centred at 0.5).
- **Live analyse must enqueue the full pipeline.** After the inline Stage-1 response is built, the same endpoint must hand the email to the inference background pipeline. The extension cannot import `InferenceService`. So `src/core/extension_pipeline_submitter.py` defines `InferencePipelineSubmitter`, which opens its own `async_session()`, calls `InferenceService.submit`, and is wrapped in `try/except` at the controller so a pipeline failure cannot fail the response.

Both adapters are stateless and wired onto `app.state.detector` / `app.state.pipeline_submitter` in `src/core/lifespan.py`. Wiring through `app.state` (rather than a static import inside the extension) is what avoids the import cycle at module-load time.

### 5.2 The pipeline runner — strong refs and graceful drain

`pipeline_runner._in_flight` is a module-level `set[asyncio.Task]` that holds strong references to in-flight pipeline tasks. `asyncio.create_task` only holds a *weak* reference, so a task with no other referent can be GC'd mid-run (the loop logs a warning and the coroutine simply stops). `spawn(...)` adds the task to the set; `add_done_callback(_in_flight.discard)` removes it on completion.

On shutdown, `lifespan` calls `pipeline_runner.drain(timeout=30.0)` which `asyncio.wait_for`s the union of in-flight tasks. Past 30s, the remaining tasks are abandoned and a warning is logged. This is the right tradeoff for an analyst-facing system: better to lose a few seconds of enrichment than to hang shutdown indefinitely.

### 5.3 Why raw ASGI middleware

`RequestLoggingMiddleware` is raw ASGI on purpose — `BaseHTTPMiddleware` buffers response bodies and breaks SSE streaming. For the same reason, `slowapi` is wired through `@limiter.limit` only and `SlowAPIMiddleware` is omitted; `RateLimitExceeded` is caught in `error_handlers.py` and returned with `code="RATE_LIMITED"`.

### 5.4 BaseRepository and session ownership

`BaseRepository[T]` provides `get_by_id`, `get_one`, `exists`, `get_all`, `paginate` (returns `(records, total)`), `count`, `create`, `create_many`, `update`, `delete`. Use it for equality filters; write custom `select()` in the subclass for `ilike`, `OR`, joins, bulk update, or complex ordering.

Sessions:
- `get_db` opens `session.begin()` — commits on clean return, rolls back on exception. The dependency owns the transaction.
- `get_db_readonly` has no transaction.
- Repository write methods call `flush` (never `commit`) because the dependency owns the transaction.

### 5.5 Lifespan startup / shutdown

```
Startup:
  setup logging
  seed_admin()                                    # creates ADMIN from env if none exists
  app.state.detector = InferenceClassificationDetector()
  app.state.pipeline_submitter = InferencePipelineSubmitter()
  asyncio.create_task(start_token_cleanup())      # JWT sweep
  asyncio.create_task(start_extension_token_cleanup())

Yield  (server runs)

Shutdown:
  cancel both cleanup tasks (await with CancelledError suppression)
  pipeline_runner.drain(timeout=30.0)
  engine.dispose()
```

Cleanup loops must catch `CancelledError` and re-raise to exit cleanly. New background tasks follow the same pattern.

### 5.6 What lives in the dashboard, not the API

- **Token storage policy.** Cookies (with `sameSite=lax`, `secure` only in prod) are a dashboard choice. The API issues opaque strings and does not care where they are stored.
- **Silent refresh queueing.** The 401 → refresh → replay pattern (with a single in-flight refresh) is implemented in `axios.interceptors`. The API just exposes `/auth/refresh`.
- **Tab-as-search-param navigation.** `/inference?tab=...` and `/extension/installs?tab=...` are dashboard-only. The API has no tab concept.
- **Recharts visualisations.** All chart math is dashboard-side; the API exposes pre-aggregated counts via `/inference/stats/*` and the dashboard does the rendering.
- **`USER_ROLES` and `TOKEN_LIFETIMES` constants.** Hand-mirrored from the API. There is no codegen.

### 5.7 Audit data and what is *not* stored

`extension_analyse_events` only stores non-PII fields (`install_id`, `model_version`, `predicted_label`, `confidence_score`, `latency_ms`, `request_id`). **No** sender, subject, or body content lands in this table. This is deliberate — the extension's analyse path is stateless from the dashboard's perspective, so the audit row exists for governance (rate-limit forensics, install activity drill-down) without storing email content twice. The actual email content (sender / subject / body / classification) is stored once, in the `emails` table, via the pipeline-submission adapter.

---

## 6. Decision log (concise)

- **Cookies, not localStorage**, for token storage on the dashboard. Reason: cookies can be marked `secure` and `sameSite`, which closes the easy XSS exfil path; localStorage cannot.
- **Hand-mirrored TS types over OpenAPI codegen.** Reason: the API is small and changes infrequently; codegen tooling adds a build step we do not yet need. Reconsider once the surface stabilises and a third client appears.
- **Raw asyncio over Celery / RQ / Arq.** Reason: pipelines are short (single-digit minutes worst-case), state lives in Postgres, and the cost of running a separate worker process is not justified at current scale. The strong-ref set + graceful drain mitigates the GC risk.
- **slowapi via decorator only, no middleware.** Reason: `BaseHTTPMiddleware` buffers responses and would break SSE if/when it is added. Decorator-only means the limit fires before the controller body and the error handler converts it to the standard envelope.
- **Background pipeline submission from the extension is fire-and-forget, wrapped in try/except.** Reason: the extension user must always see a Stage-1 verdict; a Postgres hiccup in the submitter must not make the response 500. The submission failure is logged and the inline response still returns.
- **Three-way classification mapped to a binary + review band on the extension surface.** Reason: the extension's existing protocol is binary (`predicted_label: 0 | 1`) with an optional review window; `SUSPICIOUS` parks the probability inside that window so the extension UI can render a "review" badge without the API exposing a third classification value to the extension client.
- **`extension_analyse_events` stores no email content.** Reason: governance/audit needs install-keyed counts and latency, not the message itself. The `emails` table already stores content for the pipeline; duplicating it in the audit table would double the PII footprint with no analytical benefit.
