# Design Decisions

Log of architecture and process choices for claim-triage-service. Each entry
records what we considered, what we picked, and why — so the reasoning is
defensible in an interview, not just the code.

---

## Classifier and Review as separate services vs. one app
**Options considered:** one combined app with classification and human
review in the same codebase and database, vs. two independently
deployable services that only talk over HTTP with no shared database
**Chosen:** two separate services
**Why:** establishes a repeatable pattern rather than a one-off split.
The boundary (auth on each side, no shared DB, HTTP-only contract) is
now a known shape — the next service we add follows the same template
instead of requiring the boundary to be re-derived from scratch.

---

## Async job + polling vs. blocking API until classification finishes
**Options considered:**
- **Blocking (synchronous):** `POST /denials` calls Claude inside the
  request handler and does not return until classification is finished.
  One HTTP round-trip; simpler stack (no Celery worker, no Redis queue).
- **Async + polling:** `POST /denials` validates input, writes a row with
  status `queued`, enqueues a Celery task, and returns **202 Accepted**
  immediately with the denial ID and current status. A separate Celery
  worker process picks up the task, calls Claude (or the CARC fallback),
  and updates the row. The caller **polls** `GET /denials/{id}` (or
  `GET /denials?status=classified`) until status reaches a terminal state
  (`classified` or `failed`). Status lifecycle: `queued` → `processing`
  (worker active) → `classified` or `failed`.

**Chosen:** async + polling (Celery + Redis)

**Why:** the API server's connection pool must stay free to serve other
requests. Blocking on Claude's variable latency lets enough concurrent
submissions exhaust that pool and stall the entire API, not just denial
submissions. Accepting the job immediately and processing in a separately
scalable Celery worker pool isolates slow LLM work from the API, avoids
gateway timeouts, and lets the UI show `queued` / `processing` right after
submit instead of a frozen page. Review Service stays stateless — each
browser poll triggers one fresh call to Classifier; Review does not run
its own background polling loop. Matches the Candid Health stack pattern
(Celery + Redis for task queues).

**Status naming:** `classified` not `complete` — Classifier owns
classification only; human review in Review Service is a separate step.

---

## Shared enum package vs. duplicating enums in each service
**Options considered:**
- **Duplicate:** each service defines its own copy of `DenialStatus`,
  `RecommendedAction`, etc. No cross-service import — maximum independence,
  but two places to update when a value is added.
- **Shared package:** a small `shared/` package at the repo root holds
  enums/constants; both Classifier and Review import it. Single source of
  truth for vocabulary; services remain separate (shared types only, not
  a shared database).

**Chosen:** shared package

**Why:** status and action values are a small, stable vocabulary that
both services must agree on for the HTTP contract to work. A shared
package avoids drift (Classifier returning `appeal` while Review expects
`APPEAL`) without coupling the services at the data layer — still no
shared DB, only shared types.

---

## Auth now vs. deferring authentication
**Options considered:**
- **Defer:** build endpoints open locally, add auth later. Faster first
  iteration; easy to forget or retrofit messily.
- **Auth now:** security boundary from day one, even in demo mode.

**Chosen:** auth now, with different mechanisms per service:
- **Classifier Service:** `X-Service-Key` header checked on every endpoint.
  Service-to-service only — never called directly from a browser.
- **Review Service:** JWT login for the human biller (seeded demo user).
  Review calls Classifier over HTTP, attaching the shared service key.

**Why:** two separate services only stay separate if the boundary is real.
Classifier holds PHI-adjacent denial payload data and must not be public.
Review is the human-facing entry point and needs its own identity model.
Building auth upfront avoids bolting it on after endpoints and tests exist.

---

## Container-early build order vs. Docker as a final phase
**Options considered:**
- **Features first:** finish Classifier and Review completely, then add
  Docker, compose, Terraform, and CI at the end.
- **Container-early:** get a minimal runnable API working, add a
  `Dockerfile` and `docker-compose.yml` (Redis included) immediately,
  then add Celery, auth, and classifier logic **inside** containers that
  already work. Terraform and GitHub Actions after both services run
  locally via compose.

**Chosen:** container-early

**Why:** containerizing early means Docker is never a last-minute
translation exercise — every feature is added to an environment that
already runs the same way locally and in cloud. Demonstrates infra
literacy (founding-engineer JD: GCP, Terraform, CI/CD) without
pretending live production infra exists for a portfolio demo.
