# Test Plan

> Phased test rollout for this project. Strategy is frozen at the top
> (§1–§5); cookbook patterns at the bottom (§6) fill in as phases ship.
> Read before writing any new test.
>
> Refresh: re-run `/10x-test-plan --refresh` when stale (see §8).
>
> Last updated: 2026-06-28 (Phase 1 → change opened)

## 1. Strategy

Tests follow three non-negotiable principles for this project:

1. **Cost × signal.** The cheapest test that gives a real signal for the
   risk wins. Do not promote to e2e because e2e "feels safer." Do not put a
   vision model on top of a deterministic visual diff that already catches
   the regression.
2. **User concerns are first-class evidence.** Risks anchored in "the team
   is worried about X, and the failure would surface somewhere in <area>"
   carry the same weight as PRD lines or hot-spot data.
3. **Risks are scenarios, not code locations.** This plan documents *what
   could fail* and *why we believe it's likely* — drawn from documents,
   interview, and codebase *signal* (churn, structure, test base). It does
   NOT claim to know which line owns the failure. That knowledge is
   produced by `/10x-research` during each rollout phase. If the plan and
   research disagree about where the failure lives, research is the
   ground truth.

Hot-spot scope used for likelihood weighting: `src/`, `migrations/`. Git
history spans 30 days with 3 commits — project inception only; churn data
has insufficient signal. Likelihood ratings rely on PRD, roadmap, and Phase
2 interview.

## 2. Risk Map

The top failure scenarios this project must protect against, ordered by
risk = impact × likelihood. Risks are failure scenarios in user / business
terms, not test names. The Source column cites the *evidence that surfaced
this risk* — never a specific file as "where the failure lives" (that is
research's job, see §1 principle #3).

| # | Risk (failure scenario) | Impact | Likelihood | Source (evidence — not anchor) |
|---|---|---|---|---|
| 1 | Threshold crossed, no email sent — cron runs to completion but Resend delivery fails silently; user never knows the alert triggered | High | Medium | PRD NFR ("missed notification is core product failure"); interview Q1 (VIX at 23 triggers 2–3×/year, no recovery window, stock market reaction time critical); roadmap S-05 |
| 2 | Cron Worker crashes or hits CPU budget mid-evaluation — unhandled exception or timeout silently skips the evaluation for the day | High | Medium | Roadmap F-02 risk note (free tier CPU limit; no built-in retry on Cron Trigger); interview Q3 (infrastructure alignment is the biggest risk professionally); PRD NFR |
| 3 | Stooq returns bad or missing data accepted without validation — fetch failure, changed column format, or HTTP error treated as valid; incorrect closes written to D1 | High | Medium | Roadmap F-02 unknown (Stooq has no official API contract, endpoint and column format can change without notice); PRD BL (RSI derived from daily closes sourced from Stooq) |
| 4 | RSI calculation produces wrong values — off-by-one in lookback period, wrong Wilder's smoothing formula, or edge case with fewer than 14 closes causes threshold evaluation against incorrect RSI | High | Medium | PRD BL (RSI calculation is custom server-side; correct result is the product's core guarantee); roadmap F-02 (custom implementation, no tests yet) |
| 5 | Cross-user data isolation failure — missing `user_id` scope on a CRUD endpoint allows user A to read or mutate user B's alerts | High | Low-Medium | PRD NFR ("each user's alerts fully isolated"); PRD Access Control (multi-user by design); archived `2026-06-26-backend-scaffold` (users table establishes isolation boundary) |
| 6 | IDOR — valid JWT for user A used to access or delete an alert owned by user B via a manipulated alert ID | High | Low | PRD NFR isolation; abuse/authorization lens (ownership check is distinct from authentication check); PRD Access Control |

### Risk Response Guidance

| Risk | What would prove protection | Must challenge | Context `/10x-research` must ground | Likely cheapest layer | Anti-pattern to avoid |
|---|---|---|---|---|---|
| #1 | When a threshold is crossed, Resend is called with the correct recipient and content; a `trigger_events` row is written; no duplicate call for the same alert+day | "cron completed = email sent" (trigger_events write ≠ Resend delivery) | How Resend SDK errors are surfaced; trigger_events dedup logic; whether delivery errors are swallowed silently | Integration test with a Resend stub asserting call args + dedup | Testing only that trigger_events was written, not that Resend was invoked |
| #2 | An evaluation Worker that throws an unhandled exception surfaces the error (logged or recorded) rather than silently succeeding | "cron invocation = successful evaluation" | How the cron handler catches or propagates errors; whether partial D1 writes are possible on crash; CPU budget under real workload | Unit test of the evaluation function with an injected throw; verify the error surfaces rather than being swallowed | Testing only the happy path |
| #3 | When Stooq returns an HTTP error, empty body, or unexpected column format, the system rejects the data and does not write closes to D1 | "HTTP 200 = data valid" | Stooq response format and expected columns; where response parsing happens; what a missing or extra column looks like at runtime | Unit tests with mocked Stooq responses: missing fields, wrong types, 4xx/5xx | Testing only with a perfect valid response |
| #4 | A fixed series of daily closes produces an RSI value matching a hand-computed Wilder's RSI result sourced from an external reference, not from the implementation output | "it runs = it's correct" (oracle problem — expected value copied from implementation) | RSI formula variant used (Wilder's smoothing vs simple average), lookback period, handling of <14 closes, how the first average is seeded | Pure unit test with a known input/output pair from an external RSI reference (e.g. published Wilder tables) | Using the implementation's own output as the expected value (tautological assertion that passes even when the formula is wrong) |
| #5 | A request authenticated as user A against user B's alert ID returns 403 or 404, not user B's data | "logged in = authorized for this resource" | All CRUD endpoints; how user_id is extracted from the JWT and applied in D1 queries | Integration tests with two D1 fixture users; cross-user ID requests asserted 403/404; includes registration with duplicate email → 409 | Testing only with a single user (isolation failure is invisible with one user) |
| #6 | PUT/DELETE on an alert owned by user B using user A's valid JWT returns 403 or 404, not success | "auth middleware passed = resource is mine" | Alert ownership check at the endpoint level (not just the auth middleware) | Integration test with two fixture users and explicit cross-user ID manipulation | Testing only that auth is required, not that resource ownership is verified per request |

## 3. Phased Rollout

Each row is a discrete rollout phase that will open its own change folder
via `/10x-new`. Status moves left-to-right through the values below; the
orchestrator updates Status as artifacts appear on disk.

| # | Phase name | Goal (one line) | Risks covered | Test types | Status | Change folder |
|---|---|---|---|---|---|---|
| 1 | Bootstrap + pipeline units | Bootstrap Vitest + Workers pool; first unit tests for RSI calculation, Stooq response validation, and cron error surfacing | #2, #3, #4 | unit (Workers runtime emulation) | change opened | context/changes/testing-bootstrap-pipeline-units |
| 2 | Auth and isolation integration | Verify user isolation and IDOR protection on auth + CRUD endpoints against local D1, including duplicate-email registration edge case | #5, #6 | integration (Hono + D1 local) | not started | — |
| 3 | Notification pipeline integration | Verify alert evaluation → Resend call path; assert correct args and dedup; confirm no duplicate email per alert per day | #1 | integration (Resend stub) | not started | — |
| 4 | Quality gates wiring | Lock typecheck + lint + unit+integration suite as required CI gates | cross-cutting | CI config | not started | — |

## 4. Stack

The classic test base for this project. No AI-native layer — all
high-risk logic is deterministic; classic unit + integration gives full
signal at a fraction of the cost.

| Layer | Tool | Version | Notes |
|---|---|---|---|
| unit + integration (Worker) | Vitest + `@cloudflare/vitest-pool-workers` | latest | Workers runtime emulation; D1 binding available in test context; none yet — see §3 Phase 1 |
| HTTP edge mocking (Stooq / Resend) | Vitest built-in mocks or MSW | latest | Mock only at the HTTP edge; never mock internal Worker modules; none yet — see §3 Phase 1 |
| Angular component tests | none yet | — | Frontend is currently empty; revisit when S-01 auth forms land |
| e2e | none planned for MVP | — | No critical flow requires the full deployed Workers shape at this scale |

**Stack grounding tools (current session):**
- Docs: none — not available in current session; stack evidence from local manifests only; checked: 2026-06-28
- Search: none — not available in current session; checked: 2026-06-28
- Runtime/browser: none — not available in current session; checked: 2026-06-28
- Provider/platform: none — not available in current session; checked: 2026-06-28

## 5. Quality Gates

The full set of gates that must pass before a change reaches production.
"Required after §3 Phase N" means the gate is enforced once that rollout
phase lands; before that, the gate is planned.

| Gate | Where | Required? | Catches |
|---|---|---|---|
| lint + typecheck | local + CI | required | syntactic / type drift |
| unit + integration (Worker) | local + CI | required after §3 Phase 1 | pipeline logic regressions, isolation failures |
| post-edit hook | local (agent loop) | recommended after §3 Phase 1 | regressions at edit time |
| pre-prod smoke (health + manual eval trigger) | between merge + prod | optional | environment-specific failures on Cloudflare |

## 6. Cookbook Patterns

How to add new tests in this project. Each sub-section is filled in once
the relevant rollout phase ships; before that, it reads "TBD."

### 6.1 Adding a Worker unit test

TBD — see §3 Phase 1 (RSI calculation, Stooq validation, and cron error
surfacing patterns).

### 6.2 Adding a Worker integration test

TBD — see §3 Phase 2 (Hono endpoint + local D1 binding, two-fixture-user
pattern).

### 6.3 Adding a test with an external service stub

TBD — see §3 Phase 3 (Resend stub call-assertion pattern; Stooq mock
response pattern from Phase 1).

### 6.4 Adding a test for a new API endpoint

TBD — see §3 Phase 2. Rule of thumb: prefer integration test over e2e
unless the failure mode requires the full deployed Worker shape.

## 7. What We Deliberately Don't Test

- **Angular component snapshot tests** — no Angular components with business
  logic exist yet; when they do, snapshot tests for layout/forms break
  constantly and catch nothing that a typed template check would not catch
  first. Re-evaluate if the Angular layer acquires significant client-side
  validation logic. (Source: PRD scope; tech-stack.md `skipTests: true`
  global default.)
- **E2e against the live Cloudflare deployment** — Workers runtime emulation
  in Phase 1 gives sufficient signal at MVP scale with a single user.
  Re-evaluate when multi-user concurrency or Cloudflare-specific routing
  becomes a documented risk.
- **D1 migration SQL correctness** — migrations are forward-only SQL with
  manual verification steps baked into each slice plan (local + remote
  `PRAGMA table_info` checks). Automated migration tests add complexity
  without proportional signal at this scale.

## 8. Freshness Ledger

- Strategy (§1–§5) last reviewed: 2026-06-28
- Stack versions last verified: 2026-06-28
- AI-native tool references last verified: n/a (no AI-native layer)

Refresh (`/10x-test-plan --refresh`) when:

- a new top-3 risk surfaces from the roadmap or archive,
- a recommended tool's `checked:` date is older than three months,
- the project's tech stack changes (new framework, new test runner),
- §7 negative-space no longer matches what the team believes.
