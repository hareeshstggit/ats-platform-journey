**Artifact Version 2.0 — Baselined 08-Jun-2026**  ·  Project: STG Labs ATS  ·  Versioning rule: docs/VERSIONING.md

---

# Architecture

This document describes the high-level architecture of the STG Labs ATS. If you
are new to the codebase and want to know where to make a change, you are in the
right place. It is deliberately short — read it once, and revisit it a couple of
times a year rather than on every change. It records only the things that are
unlikely to change; behaviour lives in `openspec/specs/`, patterns in
`docs/ENTERPRISE_ARCHITECTURE.md`. The high-level map and the invariants here are
stable and revised only when the system's shape changes; the granular living
documentation — function docstrings, per-module dependency headers, and the
auto-generated API reference — updates with **every** code or spec change (see
"Engineering mandate" and "Living documentation" below).

## Bird's-eye view

The ATS accepts candidate profiles and job descriptions from recruiters (through
a web UI or an API), uses Claude AI agents to extract structured data and to rank
candidate–position fit, and tracks each candidate through the hiring lifecycle:
sourcing → screening → interview → offer → onboarding.

- **Input:** uploaded PDF/DOCX files, plus structured data recruiters enter.
- **Output:** a queryable hiring pipeline, AI match assessments (% + 5 for / 5
  against), and reports.
- **Tenancy:** serves STG Labs and its portfolio companies; multi-tenant; India-only.

The work splits into a synchronous request path (HTTP → DB) and an asynchronous
pipeline (file upload → Claude extraction → matching) connected by a job queue.

## Engineering mandate (binding on every change — human or Claude Code)

This is a hard contract, not advice. It governs every line generated.

**Write the minimum code that makes the intended function run correctly and
efficiently — not one line more.** No speculative features, no unused
parameters, imports, or variables, no dead branches, no premature abstractions,
no "might need it later." Before finishing any unit of work, re-read the
acceptance criteria in the relevant `openspec/specs/<module>/spec.md` and delete
anything not traceable to one of them. If a line does not earn its place, it does
not ship.

The code that *is* written must be:

1. **Optimized.** Efficient by construction against the SLOs in
   `docs/ENTERPRISE_ARCHITECTURE.md`: async I/O, no N+1 queries (eager-load
   relations), indexed access paths, bounded/paginated result sets, no redundant
   computation, heavy work pushed off the request path onto Celery. Optimize the
   real hot path, never a hypothetical one, and never trade correctness or
   readability for cleverness. The `principal-performance-auditor` subagent is the backstop.

2. **Modular and easy to follow, with brief in-code comments.** Hold the strict
   `router → service → repository` slice; one responsibility per function; small
   functions. Comments explain *why* — the non-obvious decision or the invariant
   being upheld — never *what* the code already states. Every public function and
   route carries the 2–3 line docstring from `CLAUDE.md` (purpose, Args, Returns,
   Raises).

3. **Maintainable, with workflows and dependencies documented from line one and
   kept current** — functional workflow, data workflow, integration points, and
   cross-module dependencies are written alongside the code and updated in the
   same change, every time. See "Living documentation" below for where each lives
   and how staleness is made a build failure.

## Code map

Top-level layout. Each line is the one thing that part is responsible for.

```
ats-platform/
├── backend/            FastAPI application + workers (the system)
│   └── app/
│       ├── main.py     Entry point: builds the app, mounts routers + middleware
│       ├── core/       Cross-cutting platform (no business logic)
│       ├── shared/     Reusable infrastructure used by every module
│       └── modules/    One folder per business capability
├── frontend/           Next.js 14 UI (TypeScript)
├── infrastructure/     Terraform (AWS ap-south-1)
├── docs/               schema.sql + this doc + the reference docs
├── openspec/           Living specifications — the source of truth for behaviour
└── .claude/            CLAUDE.md working agreement + agents/ subagents
```

### `app/core/` — the platform (no domain logic)
```
config.py               Settings + secrets resolution (AWS Secrets Manager)
database.py             Async engine/session; sets the RLS session variables
security.py             JWT, password hashing, TOTP, permission resolution
middleware.py           Correlation ID, rate limit, security headers, idempotency
dependencies.py         get_current_user, require_permission, get_db
exception_handlers.py   Maps every ATSException to the standard error envelope
```

### `app/shared/` — infrastructure used by every module
```
crypto.py               KMS envelope encrypt/decrypt + HMAC for searchable PII
storage.py              S3 upload + pre-signed URLs
outbox.py               publish_event() + the Celery relay that drains the outbox
                        (relay_outbox_events runs on a Celery-beat schedule, added 2026-07-24)
audit.py                write_audit_log() with hash-chaining
email.py                Dev-only OTP/MFA delivery stub (login flow) -- logs intent, never a
                        real send. Real email fan-out for business events lives in
                        modules/notifications/, not here (see Code map above).
pagination.py           Shared paged-response shape
```

### `app/modules/<module>/` — the business capabilities
Each module is a vertical slice with the same shape:
`router.py` (HTTP) → `service.py` (logic) → `repository.py` (DB), plus
`models.py`, `schemas.py`, `exceptions.py`, and `tests/`. A module talks to
another module only through that module's `service`.

```
security        Auth, RBAC, sessions, API keys. Build this first; all else depends on it.
organizations   Tenants (portfolio companies). Tenant root.
departments     Departments within an organization.
positions       Roles to fill: JD upload, panelists, count/JD/status history.
candidates      Profile upload (bulk), dedup, and the AI agents (below).
screening       Recruiter shortlist / reject decisions with reasons.
interviews      Interview levels, scheduling, status model, panel feedback.
offers          Offer lifecycle: draft, approval, PDF, accept/decline.
notifications   Email fan-out (AWS SES) for outbox events -- interview.scheduled, offer.sent.
reporting       Read-only analytics over materialized views.
consent         DPDP consent capture/withdrawal; gates candidate AI processing.
data_privacy    Data-subject requests (access/erasure) + retention enforcement.
```

### AI agents (inside the modules that own them)
```
candidates/agents/profile_extractor.py   Reads a resume → name, experience, skills, contacts
candidates/agents/job_matcher.py         Ranks a candidate vs OPEN positions (% + 5 for / 5 against)
positions/agents/jd_extractor.py         Reads a JD → primary/secondary/good-to-have skills, R&R
```

### Background jobs
```
worker.py                       Celery app
modules/<x>/tasks.py            Async work: extraction, matching, notifications,
                                report refresh, retention sweep, partition creation
```
Queues are isolated (`extraction`, `matching`, `notifications`, `reports`,
`maintenance`) so a report backlog never delays candidate extraction.

### Data model
```
docs/schema.sql                 The whole PostgreSQL 16 schema (start here for data)
backend/app/alembic/            Migrations (additive; see docs/SCHEMA_EVOLUTION.md)
```

## The two main flows

**Request lifecycle.** HTTP request → middleware (correlation id, rate limit,
idempotency, sets `app.current_org`/`app.is_internal` for RLS) → `router`
validates and calls `service` → `service` runs business logic and calls
`repository` → `repository` reads/writes PostgreSQL → response shaped by
`router`. Every mutation writes an audit row.

**Upload → extract → match pipeline.** Recruiter uploads PDF/DOCX → file stored
in S3, `candidate` row created, an `extraction` job enqueued → `ProfileExtractorAgent`
fills structured fields → an outbox event fires → `JobMatchingAgent` produces
`candidate_position_matches` for recruiter review. All of this is off the request
path; the upload call returns 202 immediately.

## Architecture Invariants

These are the load-bearing rules. Breaking one is a design bug, not a style nit.

- **Architecture Invariant: layering is one-directional.** Routers hold no
  business logic; repositories hold no business logic; services hold no SQL and
  no HTTP concepts. Cross-module access goes through the other module's service,
  never its repository or tables.
- **Architecture Invariant: PII never exists in plaintext at rest or in logs.**
  Candidate email/mobile/address live only as `*_enc` (AES-256 via KMS) plus a
  deterministic `*_hash` for search/dedup. Plaintext is resolved in the service
  layer for authorised users only. Logs carry `candidate_id`, never PII.
- **Architecture Invariant: tenant data is isolated in the database, not just the
  app.** `positions`, `applications`, `departments` carry RLS keyed to
  `app.current_org`. The app must connect as a non-owner role for RLS to apply.
- **Architecture Invariant: candidates are intentionally NOT org-scoped.** They
  form a shared talent pool; visibility is enforced in the service layer by role
  and the application/match links. This is a deliberate trade-off, not an oversight.
- **Architecture Invariant: AI output is advisory.** Agents never write a final
  decision. A match is reviewed/dismissed by a recruiter; the matcher only
  produces `candidate_position_matches` rows. Agents run async, never on the
  request path.
- **Architecture Invariant: candidate AI processing is consent-gated.** No agent
  runs for a candidate whose `consent_status` is not `granted`.
- **Architecture Invariant: every state change is captured.** Mutations write an
  append-only, hash-chained audit row; position and interview status changes (and
  interview status reasons) are written to their history tables with a timestamp.
- **Architecture Invariant: domain events go through the transactional outbox.**
  Events are written in the same transaction as the business change and drained
  by the relay — never dispatched inline. This is why a crash cannot lose an event.
- **Architecture Invariant: writes are concurrency-safe and retry-safe.** Mutable
  rows carry a `version` column (optimistic locking → 409 on conflict); mutating
  endpoints honor an `Idempotency-Key` (a retried upload never duplicates).
- **Architecture Invariant: the schema grows additively.** New requirements go to
  `metadata` JSONB → `custom_field_definitions` → `lookup_values` → `tags` before
  a real column; never EAV; PII never in JSONB. Hot tables are month-partitioned.
  The full rule is `docs/SCHEMA_EVOLUTION.md`.
- **Architecture Invariant: reads scale off the primary.** Reporting reads the
  replica and prefers materialized views; writes go to the primary.
- **Architecture Invariant: secrets come from AWS Secrets Manager, timestamps are
  UTC, and the compliance scope is India DPDP only.** No secrets in code; no
  overseas data-protection or AI controls.
- **Architecture Invariant: minimal code, always documented.** No line ships that
  is not required for the function to run efficiently; and no change to code or a
  spec is complete until its docstrings, module dependency header, and any
  affected map in this file are updated in the same commit. Enforced by the
  `code-reviewer` subagent and CI — never left to memory.

## Boundaries

- **API boundary** — `app/modules/*/router.py` and the OpenAPI schema. This is the
  contract for the frontend and for API clients. Request/response shapes are
  Pydantic models in `schemas.py`; they never leak ORM objects.
- **AI boundary** — `app/modules/*/agents/*`. Prompts and model calls are confined
  here; the rest of the system sees only validated structured results.
- **Data boundary** — `repository.py` is the only place that touches the DB. The
  data model itself is `docs/schema.sql`.

## Living documentation (kept in lockstep with code and specs)

Documentation is part of the definition of done, not a follow-up task. From the
first line of code, every module documents four things, and they are updated in
the *same* change as any code or spec edit — never later:

- **Functional workflow** — what the capability does, step by step.
- **Data workflow** — what it reads, writes, and emits (tables, S3, events).
- **Integration points** — the modules, queues, and external systems it calls.
- **Cross-module dependencies** — what it depends on, and what depends on it.

Where each lives, so nothing drifts:

- **Per function** — the 2–3 line docstring (purpose, Args, Returns, Raises).
- **Per module** — a short header block at the top of `service.py` (or the
  module's `spec.md` "Dependencies" section) stating its functional and data
  workflow, integration points, and inbound/outbound dependencies.
- **API surface** — generated automatically from the FastAPI route signatures and
  their `summary`/`description` (OpenAPI/Swagger). This layer is never hand-kept;
  it is derived from the code, and Postman reads the same source.
- **Cross-module map** — the "Code map" and "Boundaries" sections of this file,
  plus the per-module dependency headers.

How "always current" is enforced — the mechanism behind "auto-updated on every
change":

- A change to code or a spec that does not update the affected docs is **not
  done**; the `code-reviewer` subagent blocks it.
- CI fails the build on stale documentation: missing or empty docstrings on
  public functions, a module whose dependency header no longer matches its
  imports, or a spec and its implementation that have diverged (`/opsx:verify`).
- The OpenSpec loop keeps behaviour in sync: edit the `spec.md` first, regenerate
  code, then `/opsx:verify` before `/opsx:archive`.

In short: specs and docstrings are written/updated alongside the code in the same
commit; the API reference is auto-derived from the code; and review plus CI make
staleness a build failure rather than a hope.

## Where to make a change (searchable starting points)

- New endpoint or changed behaviour → the module's `spec.md` first, then its
  `router`/`service`/`repository`.
- New field on an entity → `docs/SCHEMA_EVOLUTION.md` decides where it goes.
- New background job → `modules/<x>/tasks.py` + the right Celery queue.
- Auth, permissions, rate limits, idempotency → `app/core/`.
- Encryption, storage, outbox, audit, email → `app/shared/`.
- Anything AI → the relevant `agents/` file.

## Baseline artifact placement

Canonical locations for foundational artifacts (source: v2.0 baseline):

| Artifact | Path |
|---|---|
| Database schema (DDL) | `docs/schema.sql` |
| Alembic migrations | `backend/alembic/versions/` |
| OpenAPI contract | auto-generated at runtime (`/openapi.json`) |
| Module behaviour specs | `openspec/specs/<module-hyphenated>/spec.md` |
| Terraform (IaC) | `infrastructure/` |
| CI/CD workflows | `.github/workflows/` |
| Local dev orchestration | `docker-compose.yml` (root) |
| Environment template | `.env.example` (root) |
| Architecture decision records | `docs/` |
| DPDP compliance docs | `docs/COMPLIANCE.md` |
| Token-optimization playbook | `docs/TOKEN_OPTIMIZATION_PRACTICE.md` |
| Schema change log | `docs/SCHEMA_CHANGE.md` |
| Schema evolution rules | `docs/SCHEMA_EVOLUTION.md` |
| Version / release baseline | `.claude/VERSION.md` |
| Progress restore point | `memory/resume-pointer.md` |
| Delivery tracker | `docs/GO_LIVE_CHECKLIST.md` |

## What this document is not

This is a navigational map. For the *why* behind the patterns (RLS, envelope
encryption, outbox, partitioning, SLOs) read `docs/ENTERPRISE_ARCHITECTURE.md`;
for *behaviour* read `openspec/specs/<module>/spec.md`; for the *data model* read
`docs/schema.sql`; for *compliance* read `docs/COMPLIANCE.md`. Keep this file
short — if it starts duplicating those, trim it.
