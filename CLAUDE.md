# claim-triage-service — Project Memory

Context for AI assistants and future sessions. Read this before touching any
file. For diagrams and topology detail, see [ARCHITECTURE.md](ARCHITECTURE.md).
For decision rationale, see [DECISIONS.md](DECISIONS.md).

---

## Who this is for

**Zeon Stewart** — ~4 yrs SWE (Flatiron 2022 → ThoughtWorks Jan 2023–Feb
2026), job searching NYC/Farmingdale. Flagship portfolio: **ATL311**
(Next.js/Prisma/Supabase/Claude, live, 57 tests/94% coverage). This
project is a second portfolio piece targeting AI-native / health-tech
roles — **Candid Health** is the primary pattern match, plus PermitFlow
and Vimcal from the same job search, plus a newer **founding-engineer**
JD (Go, Python, Postgres, GCP/AWS, Kubernetes, Terraform, CI/CD, real
ownership / culture-building) that pushed the infra timeline earlier than
originally planned.

**Domain framing** (from Candid Health's JD language): medical billing
denial triage. Classifier Service recommends an action with confidence +
reasoning; Review Service is the human biller override layer. Same
discipline as ATL311: **never ship a bare AI answer.**

**Example user:** a medical biller (denial specialist) at a clinic or RCM
company — logs into Review Service, sees AI recommendations with confidence
and reasoning, approves or overrides. Never touches Classifier directly.

---

## How to work in this repo

Built through strict pairing in prior sessions — that process continues:

- **Propose before writing.** Explain reasoning and wait for explicit
  approval before creating or editing files (especially multi-file changes).
- **Small proposals.** One decision or one file at a time where possible —
  not a bundled ten-step plan presented as already decided.
- **Ask before assuming.** Surface design forks and ask; decisions below
  were each debated, not picked by default.
- **Zeon types application code; assistant explains and reviews.** The
  goal is that Zeon can defend every part unscripted in an interview.

---

## Architecture (agreed)

Two microservices — **no shared database**, HTTP-only contract:

| Service | Role | Auth | Data |
|---------|------|------|------|
| **Classifier Service** | Denial classification (Celery/Redis queue, Claude + CARC fallback) | `X-Service-Key` — never browser-facing | Own Postgres/SQLite; full denial payload |
| **Review Service** | Human biller UI/API | JWT (seeded demo user) | Own DB; opaque `denial_id` + human decision only — no payer/CPT/dollar fields |

**Classifier outputs:** `RecommendedAction` — `appeal`, `resubmit_corrected`,
`write_off`, `bill_patient`, `request_info` — plus confidence + reasoning.

### Async flow (canonical — use this in code)

1. `POST /denials` returns **202** with status **`queued`** immediately.
   The API server **never** calls Claude itself.
2. API saves row + enqueues to **Redis**. Celery worker does a **blocking
   pop** off the queue (worker pulls work; Redis is not "polled" by the
   worker in the HTTP sense).
3. Worker is the **only** process that calls Claude (or CARC fallback).
   On success, row status becomes **`classified`** (or **`failed`**).
4. **Review Service is stateless.** The browser polls Review on a ~3s
   interval; each poll triggers **one fresh** `GET` to Classifier
   (e.g. `GET /denials?status=classified` or `GET /denials/{id}`).
   Review does **not** run its own background polling loop.

```
Biller → Review Service (JWT) → [browser poll ~3s] → Classifier API (X-Service-Key)
                                                          │
                                                  saves "queued" + enqueues
                                                          │
                                            Redis queue → Celery worker → Claude
                                                          │
                                                  writes "classified" to DB
```

**Status lifecycle (Classifier DB):** `queued` → `processing` → `classified`
or `failed`. Use `classified` not `complete` — human review in Review
Service is a separate step.

### Production queue fork (documented in ARCHITECTURE.md, not a DECISIONS entry)

- **Demo / Candid-pattern path:** self-managed Celery + Redis (what we build).
- **GCP-native fork:** Cloud Tasks — push-based delivery, OIDC tokens instead
  of shared-secret header, retries/backoff/rate limits as queue config.

---

## DECISIONS.md — status of all five

Format:

```markdown
## [short decision title]
**Options considered:** A vs. B
**Chosen:** X
**Why:** one or two sentences
```

| # | Topic | Rehearsed? | Summary |
|---|-------|------------|---------|
| 1 | Separate services vs one app | Yes | Repeatable boundary: auth each side, no shared DB, HTTP-only |
| 2 | Async + polling vs blocking API | Yes | See refined Why below |
| 3 | Duplicate enum vs shared package | **No — ask before building `shared/`** | On disk: shared package; Zeon has not articulated why yet |
| 4 | Auth now vs deferred | Yes | Secure-by-default beats secure-by-audit |
| 5 | Container-early build order | Yes | Verified-by-default beats verified-by-audit |

### Decision 1 — separate services (solid)

**Chosen:** two services. **Why:** establishes a repeatable pattern rather
than a one-off split — the next service follows the same template (auth,
no shared DB, HTTP contract) instead of re-deriving the boundary.

### Decision 2 — async + polling (solid, refined Why)

**Chosen:** async + polling (Celery + Redis).

**Why (rehearsed):** the API server's limited connection pool must stay
free to serve other requests. Blocking on Claude's variable latency lets
enough concurrent submissions exhaust that pool and **stall the entire
API**, not just claim submissions. The actual waiting happens in a
separately-scalable Celery worker pool built for slow, external-dependency
work.

**Correction to keep:** Review Service is stateless — each browser poll
triggers one fresh call to Classifier; no persistent background loop on
Review's server side.

### Decision 3 — shared package (DECIDED ON DISK, NOT REHEARSED)

[DECISIONS.md](DECISIONS.md) says **shared package**. Prior conversation
also argued for **duplicating** enums (small stable vocab, avoids hidden
coupling). **Do not build `shared/` until Zeon reasons through this
out loud** — same ask-first process as decisions 1, 2, 4, 5.

### Decision 4 — auth now (solid)

**Chosen:** auth now — `X-Service-Key` on Classifier, JWT on Review.

**Why (rehearsed):** secure-by-default beats secure-by-audit. Auth on each
endpoint as it's written means you can't finish an endpoint without deciding
who may call it. Retrofitting means manually verifying every route got
covered; a missed one fails silently and keeps working for anyone.

### Decision 5 — container-early (solid)

**Chosen:** containerize minimal runnable piece immediately, then add features
inside containers.

**Why (rehearsed):** same shape as decision 4 — verified by default vs
verified by audit. A `docker build` failure after one new file has one
likely cause; a failure after ten files does not.

---

## Doc split

| File | Purpose |
|------|---------|
| [DECISIONS.md](DECISIONS.md) | **Why** — one entry per real fork; grows as decisions get rehearsed |
| [ARCHITECTURE.md](ARCHITECTURE.md) | **What** — topology, flows, demo-vs-prod, Celery/Redis vs Cloud Tasks |
| CLAUDE.md (this file) | **Session memory** — how to work, current state, build order |

---

## Repo state — last verified 2026-07-06

**Remote:** https://github.com/zsociety47/claim-triage-service (`main`)

**Committed (5 commits):**

- `.gitignore` (venv, `.env`, `*.db`, `.DS_Store`, `.idea/`)
- `classifier-service/.env.example` (Postgres, Redis, service secret, Anthropic key)
- `DECISIONS.md` (all 5 entries — #3 Why is draft, not Zeon-rehearsed)
- `ARCHITECTURE.md` (diagrams, flows, GCP topology, Cloud Tasks fork)

**Not started:**

- `shared/` package
- `classifier-service/` application code (`models.py`, `database.py`, `main.py`, …)
- `review-service/` application code
- `docker-compose.yml`, Dockerfiles, Terraform, GitHub Actions CI

**Always re-check** `git status` and `find` before assuming plan equals disk.

---

## Build order (container-early, per decision 5)

1. ~~Scaffold — `.gitignore`, `.env.example`, `DECISIONS.md`~~ **done**
2. **`shared/` package** — pending decision 3 rehearsal
3. `models.py` — Denial schema (`idempotency_key`, status enum aligned to `queued` / `classified` / `failed`)
4. `database.py`
5. Minimal `main.py` — health + one endpoint, deliberately small
6. `Dockerfile` — containerize before complexity grows
7. `docker-compose.yml` — Redis + Classifier API
8. Flesh out Classifier inside container — Celery worker (`include=["tasks"]`),
   `X-Service-Key` auth, `classifier.py`
9. Review Service — minimal → Dockerfile → compose → JWT + Classifier client
10. Terraform module — honest GCP model (Cloud Run, Cloud SQL, Memorystore)
11. GitHub Actions CI — lint + test
12. Keep `ARCHITECTURE.md` current as infra lands

---

## Known engineering gotchas — do not rediscover silently

- **Celery task registration:** `Celery(..., include=["tasks"])` in
  `celery_app.py` — worker only imports `celery_app.py`, not `main.py`;
  without `include`, "Received unregistered task" even when task exists.
- **passlib/bcrypt:** pin `bcrypt==4.0.1` with `passlib==1.7.4` — bcrypt
  5.x breaks passlib version detection.
- **Idempotency, two layers:** (1) API checks `idempotency_key` before
  insert; (2) Celery task skips if status is already `classified` — covers
  duplicate HTTP and broker redelivery separately.
- **CARC fallback** in `classifier.py` when no `ANTHROPIC_API_KEY` or Claude
  fails — demo runs offline without API credits.

---

## Broader job search context

Original application push targeted Candid Health, PermitFlow, Vimcal (outreach
drafted separately). Founding-engineer JD layered on top. Status of that
push is independent of this repo's build progress.
