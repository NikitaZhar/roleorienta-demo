# roleorienta — Technical Overview

> Engineering companion to the product overview. This document is for technical
> reviewers (CTOs, tech leads, platform recruiters) and summarizes the architecture,
> stack, and engineering decisions behind the project. For the product/feature
> description see the client-facing overview; for the full specification see
> [`docs/technical-design.md`](docs/technical-design.md) and current status in
> [`docs/project-notes.md`](docs/project-notes.md).

**Status:** actively developed. Stages 1–2 are underway (status as of 2026-09-17).
The reliability foundation — reliable messaging, migrations, idempotent scheduling,
integration tests against real infrastructure — is already in place, and the
collection/normalization pipeline and API are being built on top on a rolling basis.
The full vision spans Stages 1–5.

---

## What it is (one line)

A job-search, monitoring and analysis backend that collects vacancies from ATS
platforms and company career pages, normalizes them into structured fields, tracks
change history, builds employer technology profiles, and follows a candidate from
discovery through interview results — with data honesty (missing data is marked
`UNKNOWN`, never guessed) as a design rule.

## Architecture

Multi-module Maven monorepo with two deployable Spring Boot applications sharing a
PostgreSQL database, a RabbitMQ broker, and S3-compatible object storage for raw
snapshots:

- **job-api** — REST layer: search, subscriptions, companies, tech profiles,
  applications, notes, interviews, error reports. Owns the DB schema (applies Flyway
  migrations).
- **job-worker** — background processor: scheduling, outbox publication, employer
  discovery, collection, normalization, deduplication, revisions, tech-profile
  aggregation, notifications.
- **shared/common** — domain and messaging contracts used by both apps.

```
              ┌─────────────┐        events         ┌───────────────┐
  client ───▶ │   job-api   │ ───────────────────▶  │   job-worker  │
              │ (REST, auth)│      (RabbitMQ,        │ (scheduler,   │
              │  owns schema│       at-least-once)   │  collectors,  │
              └──────┬──────┘                        │  normalizers) │
                     │                               └───┬───────┬───┘
                     │      shared PostgreSQL            │       │
                     └──────────────┬───────────────────┘       │ raw snapshots
                                    ▼                            ▼
                              ┌───────────┐               ┌────────────┐
                              │ PostgreSQL│               │ S3 / MinIO │
                              └───────────┘               └────────────┘
```

The services are decoupled through the broker: the API records intent and writes
events to a transactional outbox; the worker consumes them and performs long-running
collection asynchronously. Request latency stays independent of scraping time, and
the worker scales separately.

## Tech stack

| Area              | Technology                                                        |
|-------------------|-------------------------------------------------------------------|
| Language          | Java 21 (LTS)                                                      |
| Framework         | Spring Boot 4.1.1                                                  |
| Persistence       | Spring Data JPA + PostgreSQL; Flyway migrations (DB-first schema)  |
| Search            | PostgreSQL full-text (`tsvector` generated column, GIN, `ts_rank`) |
| Messaging         | RabbitMQ via Spring AMQP; transactional outbox                     |
| Object storage    | S3-compatible (MinIO in local/dev) for raw snapshots              |
| Auth / sessions   | Spring Security + Spring Session JDBC; bcrypt; CSRF                |
| Build / packaging | Maven (multi-module) + Cloud Native Buildpacks; Docker            |
| Testing           | JUnit 5, Testcontainers (PostgreSQL + RabbitMQ), MockMvc, WireMock |
| CI                | GitHub Actions (build, unit + integration tests; SBOM + Trivy)    |
| Planned (Stage 3+)| Kubernetes + KEDA, Helm, Argo CD (GitOps), Gateway API + Envoy,    |
|                   | cert-manager + Sealed Secrets, Prometheus/Grafana/Tempo           |

## Key engineering decisions (ADRs)

The design is recorded as ADRs in `docs/technical-design.md`. The most notable:

- **Reliable messaging from day one (ADR-10).** RabbitMQ + transactional outbox were
  introduced in Stage 1, not deferred: domain state and the events announcing it are
  written in one DB transaction, then relayed to RabbitMQ with publisher confirms.
  This removes the dual-write race and gives at-least-once, idempotent delivery.
- **Shared database, schema owned by the API (ADR-2).** Flyway is the single source
  of truth; Hibernate only validates. Both apps read the same PostgreSQL.
- **Leader-lock in PostgreSQL, no extra infra (ADR-12).** A PostgreSQL advisory /
  ShedLock leader-lock guarantees a single scheduler/publisher across replicas;
  outbox capture uses `SELECT … FOR UPDATE SKIP LOCKED`. `minReplicaCount: 1` keeps
  planner/publisher alive during quiet periods (ADR-4).
- **Per-ATS adapters, not a universal parser (ADR-5).** Each provider (Greenhouse,
  Personio, Lever, Ashby, …) implements a unified `Provider` contract with capability
  metadata, turning an impossible generic-parser problem into a solvable per-source one.
- **Deterministic extraction, no LLM in the base version (ADR-13).** A curated,
  versioned taxonomy plus rule-based extractors — results are reproducible and
  re-computable when the parser version changes.
- **Honest coverage as a first-class state (ADR-15).** Coverage is a
  `CoverageAssessment` with an explicit "not verified / unknown" state, phrased
  narrowly ("not found on X, Y as of [date]") rather than as absolute absence.
- **Provider ≠ Source ≠ Company (ADR-16).** Integration type, collection area, and
  the verified employer relationship are separate concepts; `EmployerCandidate`
  models discovered-but-unconfirmed sources behind a confidence gate.

## Data pipeline

The worker processes a small set of bounded, idempotent task types:

| Task | Responsibility |
|------|----------------|
| `DISCOVER_EMPLOYER` | resolve company/system, create/update a source or queue for confirmation |
| `DISCOVER_PAGE`     | read a list/feed, checkpoint, detect postings |
| `FETCH_POSTING`     | fetch detail, normalize, persist changes |
| `REPROCESS_POSTING` | re-run a stored snapshot with a new parser version |
| `AGGREGATE_COMPANY_PROFILE` | recalculate a company tech profile |
| `MATCH_SUBSCRIPTIONS` | match changes and generate personal notifications |

Data-quality stages: normalization (structured fields, originals kept separately) →
deduplication (exact keys, with minimal fuzzy matching planned for Stage 2, ADR-14) →
revisions (significant changes only) → tech-profile aggregation (recency-weighted,
with minimum sample size).

## Data model & extraction

Structured extraction follows "unknown is marked explicitly, never guessed":

- **Salary (A09):** `SalaryNormalizer` → `NormalizedSalary` with explicit `UNKNOWN`
  for unmapped period/basis.
- **Location (A01):** `LocationNormalizer` detects `REMOTE / HYBRID / UNKNOWN`; splits
  "City, Country" only for non-remote.
- **Languages (A07):** two independent fields — `mentioned` (YES/NO/UNKNOWN) and
  `modality` (REQUIRED/PREFERRED/UNSPECIFIED) — in `posting_language`.
- **Skills/tech (A08):** taxonomy-driven alias→canonical mapping, with a `stance`
  field (REQUESTED / NEGATED / MIGRATION).
- **Experience (A08):** seniority (JUNIOR/MEDIOR/SENIOR/UNKNOWN) + minimum years.
- **Revision history (A06/A16):** `posting_revision` records only real
  non-empty-to-different field transitions between crawls.

Raw snapshots are stored (S3) so interpretation can be re-derived without creating
false freshness — reprocessing does not update `lastSeenAt` (A18).

## REST API design

- API-first, OpenAPI as source of truth; versioned paths (`/api/v1/...`).
- Unified errors as `application/problem+json` (RFC 9457) with JSON-Pointer `errors[]`.
- Cursor-based pagination (stable under inserts), strong `ETag` / `If-None-Match`.
- Optimistic concurrency: `412` on failed `If-Match`, `409` on application conflict.
- Idempotent writes (e.g. create application keyed by client key).

Representative endpoints: `GET /api/v1/vacancies`, `GET /api/v1/vacancies/{id}`,
`GET /api/v1/vacancies/{id}/history`, `GET /api/v1/companies/{id}/tech-profile`,
`POST /api/v1/applications`, `PATCH /api/v1/applications/{id}`.

## Reliability & data quality

At-least-once delivery via outbox + publisher confirms (with `mandatory` /
`basic.return` handling, A15); idempotent consumers keyed by task; exponential
backoff + jitter and a DLQ after retries. A durable `PendingChange` journal makes
notifications independent of the broker. Incomplete crawls never auto-close postings;
closures require explicit status or a confirmed absence.

## Security

- **Implemented:** session auth — Spring Security + Spring Session JDBC, `AppUser`
  with USER/ADMIN roles, bcrypt, `/api/v1/auth` (self-service register, login, logout),
  CSRF via cookie, admin bootstrap from external config; public feed reads stay open.
- **Designed / on the roadmap (A13/A14):** SSRF control via a single vetted HTTP
  client (scheme allow-list, private/loopback/metadata deny-list, DNS-rebinding checks,
  redirect revalidation, size/timeout limits); XXE control for XML feeds (DTD/external
  entity bans, expansion limits). A per-resource "operation × ownership" privacy matrix
  (A23) governs access to interviews, notes, notifications and error reports.

## Observability (from pilot → Stage 3)

Pilot minimum: source state, freshness, error log, undelivered-notification tracking.
Stage 3 adds Prometheus metrics (API p95/error rate, source age, crawl outcomes, queue
length/task age, DLQ, outbox publish delay), Grafana dashboards stored in-repo, alerts
(non-empty DLQ, oldest-task-age, growing publish delay), and one end-to-end trace
(scheduler → outbox → RabbitMQ → worker) with `taskId` / `crawlRunId` / `sourceId` in
JSON logs.

## Testing

- **Unit (no DB):** `SkillExtractorTest`, `LanguageExtractorTest`,
  `SalaryNormalizerTest`, `LocationNormalizerTest` — deterministic extraction rules.
- **Integration (Testcontainers PostgreSQL + RabbitMQ):** `OutboxDeliveryIntegrationTest`,
  `SourceSchedulerIntegrationTest` — end-to-end delivery and scheduling.
- **Web (MockMvc + Spring Security + Testcontainers):** auth endpoints and feed reads.

## Running locally

```bash
# Prerequisites: JDK 21, Docker

./dev-up.sh      # PostgreSQL (port 5433), RabbitMQ, MinIO, WireMock source stub
./dev-seed.sh    # seed a demo Greenhouse provider + source
./run-api.sh     # start job-api
./run-worker.sh  # start job-worker
```

`docker-compose.yml` orchestrates the infrastructure; WireMock fixtures simulate three
job postings with varied language/skill/experience mentions for a realistic demo.

## Deployment & CI/CD

Separate Docker images per app (unprivileged user, base images pinned by digest,
external secrets). CI: compile → unit + integration tests → image build + SBOM + Trivy
scan → publish by commit tag → deploy by digest. Expand→migrate→contract schema
migrations keep old and new workers compatible during rollout. Stage 3+ targets
Kubernetes with KEDA (RabbitMQ `QueueLength` scaler), Helm, Argo CD (GitOps),
Gateway API + Envoy, and cert-manager + Sealed Secrets.

## Roadmap (Stages 0–5)

- **Stage 0** — market & source validation (access, fields, rate limits, terms).
- **Stage 1** *(underway)* — working product in Compose: two apps, PostgreSQL +
  RabbitMQ + outbox from day one, one adapter per platform, normalization, point
  dedup + coverage, history, structured requirements, session auth.
- **Stage 2** *(underway)* — reliable monitoring: retries/DLQ, multiple ATS adapters,
  expanded discovery, minimal fuzzy matching, tech profiles, vacancy comparison.
- **Stage 3** — Kubernetes & observability (parallel to Stage 2, doesn't delay pilot).
- **Stage 4** — delivery & evidence: full CI (kind, scanning, Argo CD), DB recovery,
  experiment reports.
- **Stage 5** — extensions: wider discovery, optional LLM enrichment with verification,
  more ATS/markets, full candidate profile, geosearch, GDPR machinery.

See [`docs/project-notes.md`](docs/project-notes.md) for the current implemented-vs-planned
breakdown.

## Engineering highlights

Built production-first rather than prototype-first: asynchronous service decomposition,
reliable messaging with the transactional outbox pattern and idempotent scheduling,
versioned schema migrations, deterministic and explainable data extraction, an
API-first REST surface (RFC 9457, ETags, cursor pagination, optimistic concurrency),
and integration tests against real PostgreSQL and RabbitMQ. As features land, they build
on this foundation instead of retrofitting it.
