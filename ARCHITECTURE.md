# Architecture

System design for **claim-triage-service** — medical billing denial
classification with human review. For decision rationale, see
[DECISIONS.md](DECISIONS.md).

---

## Services at a glance

| Service | Role | Auth | Owns |
|---------|------|------|------|
| **Classifier Service** | AI/rule-based denial classification | `X-Service-Key` (service-to-service only) | Full denial payload, classification result, confidence, reasoning |
| **Review Service** | Human biller approve/override | JWT (demo user login) | Opaque `denial_id`, human decision, audit metadata only |
| **shared/** | Shared types (not a runtime service) | — | `DenialStatus`, `RecommendedAction` enums |

Classifier and Review have **separate databases**. They communicate **only
over HTTP**. Review never stores payer, CPT, or dollar-amount fields from
the denial — only a reference ID and the biller's choice.

---

## System diagram

```mermaid
flowchart TB
  subgraph clients [Clients]
    Biller[Biller browser or API client]
  end

  subgraph review [Review Service]
    ReviewAPI[Review API]
    ReviewDB[(Review DB)]
  end

  subgraph classifier [Classifier Service]
    ClassifierAPI[Classifier API]
    Worker[Celery Worker]
    ClassifierDB[(Classifier DB)]
  end

  subgraph infra [Shared infrastructure]
    Redis[(Redis broker)]
    SharedPkg[shared enums package]
  end

  Claude[Anthropic Claude API]
  CARC[Rule-based CARC fallback]

  Biller -->|"JWT login + review actions"| ReviewAPI
  ReviewAPI --> ReviewDB
  ReviewAPI -->|"HTTP + X-Service-Key"| ClassifierAPI

  ClassifierAPI --> ClassifierDB
  ClassifierAPI -->|"enqueue task"| Redis
  Worker -->|"consume task"| Redis
  Worker --> ClassifierDB
  Worker --> Claude
  Worker -.->|"if no key or API fails"| CARC

  ReviewAPI -.->|"imports types"| SharedPkg
  ClassifierAPI -.->|"imports types"| SharedPkg
  Worker -.->|"imports types"| SharedPkg
```

**Legend:** solid lines = runtime traffic; dashed = compile-time dependency
(shared package, fallback path).

---

## Classification flow (async + polling)

Review (or any authorized caller) submits a denial to Classifier. The API
returns immediately; the worker does the slow work.

```mermaid
sequenceDiagram
  participant Caller as Caller Review Service
  participant API as Classifier API
  participant DB as Classifier DB
  participant Redis as Redis
  participant Worker as Celery Worker
  participant LLM as Claude or CARC fallback

  Caller->>API: POST /denials plus X-Service-Key
  API->>DB: insert row status queued
  API->>Redis: enqueue classify task
  API-->>Caller: 202 Accepted denial id status queued

  loop Poll until terminal
    Caller->>API: GET /denials/id plus X-Service-Key
    API->>DB: read row
    API-->>Caller: status queued processing classified or failed
  end

  Worker->>Redis: pick up task
  Worker->>DB: status processing
  Worker->>LLM: classify denial
  LLM-->>Worker: action confidence reasoning
  Worker->>DB: status classified plus result fields
```

**Recommended actions** returned by Classifier: `appeal`, `resubmit_corrected`,
`write_off`, `bill_patient`, `request_info` — always with confidence score
and reasoning, never a bare label.

---

## Review flow

```mermaid
sequenceDiagram
  participant Biller as Biller
  participant Review as Review API
  participant ReviewDB as Review DB
  participant Classifier as Classifier API

  Biller->>Review: POST /login
  Review-->>Biller: JWT

  Biller->>Review: GET /queue JWT
  Review->>Classifier: GET /denials JWT proxied via service key
  Classifier-->>Review: classified denials
  Review-->>Biller: review queue

  Biller->>Review: POST /reviews/denial_id/decision JWT
  Review->>ReviewDB: store denial_id plus human decision
  Review-->>Biller: 201 confirmed
```

Review stores **what the human decided**, not a copy of the full denial
record. Classifier remains the system of record for classification data.

---

## Local vs target cloud layout

**Today (demo):** `docker-compose` runs Classifier API, Celery worker,
Review API, Redis, and SQLite or Postgres locally.

**Target (Terraform module — modeled, not live-deployed for portfolio):**

```mermaid
flowchart LR
  subgraph gcp [GCP target topology]
    CloudRunReview[Cloud Run Review Service]
    CloudRunAPI[Cloud Run Classifier API]
    CloudRunWorker[Cloud Run Celery Worker]
    CloudSQL[(Cloud SQL Postgres)]
    Memorystore[(Memorystore Redis)]
  end

  CloudRunReview -->|"X-Service-Key"| CloudRunAPI
  CloudRunAPI --> CloudSQL
  CloudRunAPI --> Memorystore
  CloudRunWorker --> Memorystore
  CloudRunWorker --> CloudSQL
  CloudRunWorker --> Anthropic[Anthropic API]
```

| Component | Local demo | Target production |
|-----------|------------|-------------------|
| Classifier API | Docker / uvicorn | Cloud Run service |
| Celery worker | Docker / celery worker | Cloud Run service (separate revision) |
| Review API | Docker / uvicorn | Cloud Run service |
| Task queue | Redis container | Memorystore (Redis) |
| Database | SQLite or Postgres container | Cloud SQL (Postgres) |
| Secrets | `.env` file | Secret Manager |
| CI | GitHub Actions (planned) | GitHub Actions → deploy |

---

## Task queue: Celery/Redis vs Cloud Tasks (demo vs GCP-native fork)

This belongs in ARCHITECTURE.md (not DECISIONS.md) — same product, different
hosting choices for the async classification pipeline.

### What we build (demo + Candid-pattern match)

Candid Health's JD describes **Celery + Redis** for task queues. Our demo
and Terraform "honest model" follow that shape:

- Classifier API enqueues a task after `POST /denials` returns 202.
- **Redis** is the broker; Celery worker **blocking-pops** work (worker
  consumes the queue — not the same as Review's HTTP polling).
- Worker calls Claude, writes `classified` or `failed` back to Classifier DB.
- Review Service stays stateless; the **browser** polls Review, and each
  poll triggers a fresh `GET` to Classifier.

```mermaid
flowchart LR
  API[Classifier API] -->|"enqueue"| Redis[(Redis)]
  Redis -->|"blocking pop"| Worker[Celery Worker]
  Worker --> Claude[Claude or CARC]
  Worker --> DB[(Classifier DB)]
```

### Documented production fork: GCP Cloud Tasks

If this were deployed GCP-natively without self-managed Celery/Redis:

| Concern | Celery + Redis (built) | Cloud Tasks (fork) |
|---------|------------------------|---------------------|
| Delivery | Worker pulls from Redis | **Push** HTTP to worker endpoint |
| Auth between services | `X-Service-Key` shared secret | **OIDC** token on task delivery |
| Retries / backoff | Celery task config + broker | Queue config (rate limits, retry policy) |
| Ops burden | Run Redis + worker fleet (Memorystore + Cloud Run) | Managed queue; no Redis to operate |

Cloud Tasks does not change the **async shape** (accept job fast, classify
later, poll for status) — only **who runs the queue** and **how the worker
is invoked**. The Review → Classifier polling contract stays the same.

---

## Repository layout (planned)

```
claim-triage-service/
├── CLAUDE.md             # session memory — how to work, repo state, build order
├── DECISIONS.md          # why we built it this way
├── ARCHITECTURE.md       # this file — what the system looks like
├── docker-compose.yml    # local full stack
├── shared/               # DenialStatus, RecommendedAction enums
├── classifier-service/   # API, worker, models, classifier logic
└── review-service/       # JWT auth, review queue, human decisions
```
