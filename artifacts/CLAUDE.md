**Artifact Version 2.2 — Baselined 11-Jun-2026**  ·  Source requirements: ATS_requirement_v2_0_08-Jun-2026.docx + 11-Jun-2026 scope addendum  ·  Versioning rule: docs/VERSIONING.md

---

# CLAUDE.md — ATS Code Standards & Working Agreement
# Claude Code reads this before EVERY task. All generated code complies.

## Project
- Name: ATS Platform — STG Labs (Bengaluru)
- Purpose: Production enterprise Applicant Tracking System.
  Full lifecycle: Sourcing → Screening → Interview → Offer → Onboarding.
- Scope: STG Labs India hiring, India-based candidates. Compliance: India DPDP Act.
- Stack: Python 3.12 + FastAPI (async) + PostgreSQL 16 (SQLAlchemy 2.0 async + Alembic)
  + Redis 7 + Celery. Frontend: Next.js 14 + TypeScript + TanStack Query + shadcn/ui.
- Cloud: AWS ap-south-1 (ECS Fargate + RDS + ElastiCache + S3). Secrets: AWS Secrets Manager.
- Reference docs (read when relevant):
  docs/ARCHITECTURE.md, docs/COMPLIANCE.md, docs/SCHEMA_EVOLUTION.md,
  docs/SCHEMA_CHANGE.md, docs/PERFORMANCE_TESTING.md
  Specs: openspec/specs/<module>/spec.md (source of truth — read before implementing)

## Architecture (Router → Service → Repository — strict)
- router.py     : HTTP only. Validate input, call service, shape response. No business logic.
- service.py    : ALL business logic. No raw DB queries, no HTTP concepts.
- repository.py : ALL DB access. No business logic. Returns ORM objects.
- models.py     : SQLAlchemy ORM models only.
- schemas.py    : Pydantic v2 request/response schemas only.
- exceptions.py : One custom exception class hierarchy per module.
- Inter-module calls go through the other module's service interface ONLY.

## Code quality (non-negotiable)
- Type hints on EVERY parameter and return value.
- Docstring on every public function/method and every API route (format below).
- Max 40 lines per function, max 300 lines per file.
- structlog for logging — NEVER print(). Never log PII (email, mobile, resume text);
  log candidate_id / position_id instead.
- 80% minimum test coverage per module (pytest-cov).
- No N+1 queries — use selectinload. No secrets in code. UTC timestamps everywhere.
- Soft deletes (deleted_at). Audit every CREATE/UPDATE/DELETE via shared/audit.py.
- Optimistic concurrency: pass expected version on updates to versioned tables.
- Idempotency-Key honored on all mutating endpoints.

## Engineering mandate (binding — minimal, optimized, modular, maintainable)
This is a hard contract, not advice. It governs every line you generate.
The authoritative expansion is docs/ARCHITECTURE.md ("Engineering mandate").

**Write the minimum code that makes the intended function run correctly and
efficiently — not one line more.** No speculative features, no unused parameters,
imports, or variables, no dead branches, no "might need later" abstractions.
Prefer the standard library and existing project helpers over new dependencies.
Before finishing any unit of work, re-read the acceptance criteria in the
relevant spec and delete anything not traceable to one of them. If a line does
not earn its place, it does not ship.

The code that IS written must be:
1. Optimized — efficient against the SLOs in docs/ARCHITECTURE.md:
   async I/O, no N+1 (eager-load), indexed access, bounded/paginated results, no
   redundant computation, heavy work on Celery off the request path. Optimize the
   real hot path, never a hypothetical one; never trade correctness or readability
   for cleverness. The principal-performance-auditor subagent is the backstop.
2. Modular and easy to follow, with brief in-code comments — strict
   router→service→repository slice, one responsibility per function, small
   functions. Comments explain WHY (the non-obvious decision, the invariant being
   upheld), never WHAT the code already says. Every public function and route
   carries the 2–3 line docstring below.
3. Maintainable, with workflows and dependencies documented from line one and
   kept current — see "Living documentation" below.

## Docstring & API-doc standard (REQUIRED — keep to 2–3 lines)
Every function: one-line purpose, then Args and Returns (and Raises if it throws).
Be concrete about parameter names, types, and what comes back.

Python example:
    async def create_candidate(payload: CandidateUpload, actor: User) -> CandidateResponse:
        """Create a candidate and enqueue async profile extraction.

        Args:
            payload: Validated upload (file ref, source). 
            actor: Authenticated user performing the upload.
        Returns:
            CandidateResponse with candidate_id and extraction status='processing'.
        Raises:
            DuplicateCandidateError: if (name, mobile, email) already exists.
        """

FastAPI route: give the decorator a summary and description so it renders in OpenAPI:
    @router.post("/candidates/upload", response_model=UploadResponse, status_code=202,
                 summary="Upload a candidate profile",
                 description="Accepts a PDF/DOCX, stores it, and starts extraction. "
                             "Returns 202 with candidate_id; processing is async.")

## Living documentation (part of the definition of done — updated every change)
Documentation is written with the code in the SAME commit, never later. From the
first line, every module documents four things, kept current on every code/spec
change:
- Functional workflow — what the capability does, step by step.
- Data workflow — what it reads, writes, and emits (tables, S3, events).
- Integration points — the modules, queues, and external systems it calls.
- Cross-module dependencies — what it depends on, and what depends on it.

Where each lives:
- Per function — the 2–3 line docstring above.
- Per module — a short header block at the top of service.py (or the module's
  spec.md "Dependencies" section) stating its functional + data workflow,
  integration points, and inbound/outbound dependencies.
- API surface — auto-derived from FastAPI routes + summary/description (OpenAPI);
  never hand-maintained.
- Cross-module map — the Code map + Boundaries in docs/ARCHITECTURE.md.

Enforcement (this is what "auto-updated on every change" means in practice):
- A code or spec change that does not update the affected docs is NOT done; the
  principal-reviewer subagent blocks it.
- CI fails on stale docs: missing/empty docstrings on public functions, a module
  whose dependency header no longer matches its imports, or a spec diverged from
  its implementation (/opsx:verify).
- Edit spec.md first, regenerate code, then /opsx:verify before /opsx:archive.

## Database schema evolution (binding — extend, never break; log every change)
ANY schema extension or evolution — triggered whenever a new feature or module is
added, OR an existing feature or module is updated (whether via a spec change or a
code change) — that adds or alters a table, column, enum, index, constraint,
trigger, RLS policy, partition, or materialized view MUST follow this rule. It is
enforced by the principal-reviewer subagent and CI; a schema change that skips it is
NOT done.

1. Consult docs/SCHEMA_EVOLUTION.md FIRST — every time, before storing any new
   data. Walk its decision tree and choose the cheapest safe mechanism, in this
   order: metadata JSONB on the entity → lookup_values → custom_field_definitions
   → tags. Use a real column/table ONLY for hot-queried, indexed, or relational
   data. Never EAV.
2. Extend without impacting existing functionality (backward-compatible ONLY).
   Additive, expand→contract migrations: new columns are NULLable or carry a
   default; never drop, rename, or retype a populated column, and never mutate an
   enum in place, in the same release the running code depends on it — do it in
   phases (add new → backfill → switch reads/writes → drop old in a later
   release). Preserve existing indexes, constraints, RLS policies, and the
   version / deleted_at columns. Soft-delete, never hard-delete. Validate the
   migration against existing data (upgrade AND downgrade) before it ships.
3. One Alembic revision per change, reversible (downgrade implemented). The
   migration ships in the SAME commit as the spec/code change that needs it.
   Revision IDs must be ≤32 chars — `alembic_version.version_num` is
   `VARCHAR(32)`. Longer IDs cause `StringDataRightTruncationError` on the
   version stamp UPDATE, rolling back the entire migration under PostgreSQL
   transactional DDL. Pattern: `{NNNN}_{short_descriptor}` where total ≤32.
4. Log every schema change in docs/SCHEMA_CHANGE.md, in the SAME commit, using
   the structured template at the top of that file: date · baseline · author ·
   trigger (new/updated feature or module; spec or code change) · change type ·
   storage mechanism chosen (per SCHEMA_EVOLUTION.md, with the reason a real
   column was needed if used) · Alembic revision id · backward-compatibility &
   rollback note · linked spec/PR. This log is the durable, structured record for
   documentation and future reference. No schema change ships without a
   docs/SCHEMA_CHANGE.md entry — never skip one.
5. **Backfill mandate (binding, no exceptions — added 2026-07-10 after a live
   incident):** when a new column starts persisting a value that existing rows
   implicitly already HAD via some other authoritative source (a status/history
   table, a related entity, a prior transient-only field), "no existing rows
   affected (backfills to NULL)" is NOT sufficient justification on its own.
   Structurally true ≠ functionally correct: NULL for "this was already decided
   before the column existed" is indistinguishable from NULL for "never decided,"
   and any feature reading that column (a lock, a display label, a report) will
   silently misbehave for every pre-existing row. In the SAME PR as the migration,
   either (a) write a backfill that derives the correct value from the existing
   authoritative source for every row where it can be determined unambiguously
   (matching this project's established backfill-script pattern —
   `backend/app/scripts/backfill_*.py`, ENVIRONMENT=local-gated, idempotent,
   single transaction, rollback-on-exception — never guess when the source data
   is ambiguous, leave those rows NULL and log them for manual review), or (b)
   explicitly document in the migration's own docstring AND the
   docs/SCHEMA_CHANGE.md entry why no backfill is possible or needed (e.g. the
   value is genuinely un-derivable, or no existing rows could ever have had it).
   Silence on this point is not an acceptable default — principal-reviewer must
   affirmatively confirm one of (a) or (b) was considered, not assume it away.
   Incident this rule codifies: migration `0047_feedback_outcome_col` added
   `interview_feedback.outcome` with a technically-true "no rows affected"
   docstring; a real pre-existing row's decided outcome was already recorded in
   `application_status_history` (the authoritative source) but never backfilled,
   which surfaced as a shipped feature (BR-SYNC-006) appearing broken for a real
   user-tested record. Fixed by `backend/app/scripts/backfill_legacy_feedback_outcome.py`
   — this incident is the reason the rule exists, not a hypothetical.

## Versioning & baseline (read before generating any code)
- Current baseline: **v2.2 — 11-Jun-2026** (= v2.0 + the 11-Jun-2026 scope addendum).
  Manifest: `.claude/VERSION.md`; Git tag: `v2.2-baseline-2026-06-12`. v2.0 source:
  ATS_requirement_v2_0_08-Jun-2026.docx.
- Every spec, schema, and doc carries a version header in its first lines.
  Before implementing a module, read that header and confirm you are building
  against the current baseline. If a file's header date is older than the
  stated baseline, check the manifest's "re-based files" list: an artifact the
  current baseline did NOT re-base keeps its older header and is still current
  (only candidates/positions specs + docs/schema.sql were re-based to v2.2);
  flag (do not build from) only a file the baseline DID re-base but that is stale.
- Filenames inside the repo stay canonical (spec.md, schema.sql, CLAUDE.md,
  openspec/project.md); the date lives in the header, the baseline package
  name, and the Git tag. Never rename these files to add a date.
- The full rule is in docs/VERSIONING.md. When a new requirements doc arrives,
  a new dated baseline is cut and tagged before code is generated against it.

## OpenSpec workflow
- Before implementing a feature: read openspec/specs/<module>/spec.md.
- If no spec exists for the work, run /opsx:propose <name> first and have it reviewed.
- After implementing: /opsx:verify, then /opsx:archive. Keep specs in sync with code.

### Spec-implementation sync mandate (binding, no exceptions — added 2026-07-10
### after a full spec-vs-code drift audit found gaps across 8 modules)
Build order is always spec-first: the spec.md is the contract, code implements it —
never the reverse. This mandate closes the loop for what happens after code ships:
- Any change request, enhancement, or bug fix that alters observable API behavior,
  a schema/enum/status value, a permission rule, or an error code MUST update the
  affected openspec/specs/<module>/spec.md in the SAME change — mandatorily, BEFORE
  the PR is raised. A PR that changes behavior without a matching spec update is
  NOT done; principal-reviewer blocks it.
- A recurring spec-vs-code drift audit (same method as the 2026-07-10 sweep: read
  each module's spec.md against its router/service/repository/schemas/models) is
  not a one-time cleanup — re-run it periodically (at minimum, before any AWS
  go-live gate and after any batch of merges that touched a module's contract)
  to catch drift that slips through despite the same-PR rule above.
- This is the mechanism that keeps spec and implementation in sync on an ongoing
  basis, not just at baseline cut time — treat spec drift itself as a defect
  class, on the same footing as a functional bug.

## Progress capture & compaction (binding — lightweight)
After each PR merges to main: (1) update `memory/resume-pointer.md` with what landed
(2–3 lines); (2) flip the relevant row in `docs/GO_LIVE_CHECKLIST.md` inline in the
SAME PR as the code — no separate docs PRs, no phase logs, no DEVELOPMENT_JOURNEY.md
updates. Run `/compact` after any large phase; no pre-compact docs PR required —
`resume-pointer.md` is the durable restore point. To restore context after a compact,
read `resume-pointer.md` first, then `docs/GO_LIVE_CHECKLIST.md` and the relevant spec.

## Subagents (delegate — see .claude/agents/)
- backend-engineer       : implement a module to its spec
- unit-test-engineer     : write unit tests (mock the repository layer); includes async context manager mock patterns, failure-cascade tests for bulk ops, side-effect non-invocation tests, error sanitisation tests
- functional-test-engineer : standalone per-feature smoke tests against REAL running stack (real DB/Redis/Celery) — catches defects unit mocks hide (session corruption, constraint cascades, auth wiring, bulk partial-failure); MANDATORY pre-commit gate for every new/modified endpoint AND every bug fix; FIRST step is always confirming the live server has loaded the new code (restart if stale — unit tests pass against files on disk, not the running process); reports bugs, does NOT fix them
- integration-test-engineer : full end-to-end tests, all workflows, all edge cases — runs AFTER functional tests are clean
- principal-reviewer     : SENIOR review gate — correctness, security, layering, minimalism, perf, reliability → ONE mandate-anchored verdict (opus/high — see "Model tier mandate" below)
- principal-reliability-engineer : on-demand specialist — failure modes, retries, idempotency, recovery; AWS SRE (failover, backup/DR, SLOs, runbooks) (opus/xhigh)
- principal-performance-auditor  : on-demand specialist — deep profiling, query plans, indexing, load/scale (opus/xhigh)
- ux-ui-engineer        : Next.js UI to the UX/UI guardrails — accessible, enterprise-class, minimal-click
Default loop for a module: backend-engineer → unit-test-engineer → **functional-test-engineer** →
integration-test-engineer → **principal-reviewer** (the single review gate + final sign-off before
merge is requested), pulling in principal-reliability-engineer / principal-performance-auditor on
demand when a change warrants a deep dive (I/O- or hot-path-heavy, new integration, or pre-deploy).
The human approves the merge. **functional-test-engineer is a hard gate: any [BLOCK MERGE] from it
stops the pipeline; integration-test-engineer and principal-reviewer do not run until functional
tests are clean.** For UI/frontend work, use ux-ui-engineer (it MUST read
.claude/rules/ats-ux-ui-guardrails.md first).

## Quality gate & first-time pass rate (binding — 100% standard, zero rework)
This section exists because P1 bugs repeatedly escaped to the user after agents declared "done".
The root causes — testing stopped at enqueue, architecture gaps not analysed at build time,
spec silences accepted without clarification — are closed here. Every rule below is a hard gate.

**Rule 1 — Clarify before building, never after.**
If a spec is silent on a recovery path, edge case, or cross-cutting concern, raise it as an
explicit question to the user BEFORE spawning backend-engineer. Do not assume. Do not build
against an assumption and discover the gap via a bug report.

**Rule 2 — Async features: verify the OUTCOME, not just the enqueue.**
For any Celery-backed endpoint the functional-test-engineer MUST:
  trigger → explicit wait (poll or timed sleep) → assert DB rows changed → assert API
  response reflects the completed result.
Asserting only the 202 response is a test failure, not a test pass. No async feature is done
until the result is confirmed in the database.

**Rule 3 — Celery tasks touching RLS-protected tables: `set_rls_context` is mandatory.**
Every Celery async body that queries a table with a PostgreSQL RLS policy MUST call
`await set_rls_context(session, org_id=None, is_internal=True)` immediately after opening
the session. The `positions` table is RLS-protected. principal-reviewer blocks any Celery
task that omits this call on an RLS-protected table.

**Rule 4 — "Done" requires UI-verified golden path, not just passing tests.**
For every feature with a UI surface, the functional-test-engineer or ux-ui-engineer MUST
confirm the full golden path works end-to-end in the browser (or via real HTTP against the
live stack) before the PR is raised. Tests verify code correctness; only a live run verifies
feature correctness. A feature declared done without this step is NOT done.

**Rule 5 — Functional tests MUST clean up every entity they create (binding — no exceptions).**
Every functional test that creates data (candidates, positions, applications, screenings, interviews,
offers, or any other entity) MUST delete that data before the test exits — whether it passes or
fails. Cleanup runs in a `finally` block or a pytest `yield` fixture's teardown so it fires even on
assertion failure. principal-reviewer blocks any functional test file that creates entities without
a corresponding cleanup teardown. This rule exists because test data accumulates in the shared
localhost DB, pollutes UI views, and causes false failures in subsequent runs.

**Cleanup mechanics — soft-delete vs hard-delete (critical distinction):**
  - positions, candidates, screenings, offers, interview_levels: SOFT-DELETE (`deleted_at = NOW()`)
  - applications: NO `deleted_at` column — MUST hard-delete via psycopg2 SQL in FK-safe order:
      1. interview_feedback           (FK → interviews)
      2. interview_panelist_assignments (FK → interviews)
      3. interview_level_kits         (FK → interviews)
      4. interview_status_history     (FK → interviews)
      5. interviews                   (FK → applications)
      6. offers                       (FK → applications)  ← soft-delete or hard-delete
      7. screening_decisions          (FK → applications)
      8. application_status_history   (FK → applications)
      9. applications                 (hard DELETE)
    Wrong: soft-deleting the parent position does NOT cascade-delete applications — they have
    no deleted_at and will remain visible in the UI until hard-deleted.

**Pre-run stale data check (mandatory — functional-test-engineer runs this first):**
Before executing any FT suite, query for leftover FT-prefixed rows:
  `SELECT COUNT(*) FROM positions WHERE title ILIKE 'FT-%' AND deleted_at IS NULL`
  `SELECT COUNT(*) FROM candidates WHERE display_name ILIKE 'FT-%' AND deleted_at IS NULL`
If either count > 0: log a warning and run the bulk cleanup SQL before proceeding. Stale data
from a prior failed teardown must not pollute the current run's assertions.

**Post-teardown verification (mandatory — every fixture):**
After teardown, assert the primary entity is gone:
  - positions/candidates: assert GET /{id} returns 404
  - applications: assert it no longer appears in GET /applications?candidate_id=...
Teardown that silently succeeds without verification is not compliant.

**Naming convention for test entities (mandatory — makes bulk cleanup possible):**
All entities created by functional tests MUST use identifiable prefixes so stale data can be
found and purged if teardown fails:
  - positions: title = "FT-<TestName>-<uuid4_short>"  (already in use — keep this pattern)
  - candidates: display_name = "FT-<TestName>-<uuid4_short>" (direct SQL insert)
  - Other entities (offers, interviews, screenings): linked to test candidates/positions —
    cleaned up explicitly in the FK-safe hard-delete order above, NOT assumed to cascade

**Rule 6 — Dependency/integration-point mapping is mandatory BEFORE dispatching any build
that changes an existing shipped feature's contract or scope (binding — added 2026-07-31
after the interview-pipeline-progress-report incident).**
This incident: the report's ask went through 3 separate clarification cycles (status-dropdown
scope, then which statuses, then single-level vs all-levels) instead of one front-loaded pass —
a direct violation of this project's own documented anti-pattern in
`docs/TOKEN_OPTIMIZATION_PRACTICE.md` (front-load ambiguity resolution; run all foreseeable
edge-case checks in one batched pass; ask once). Worse, `design.md` for the resulting
architecture change mapped the implementation files (SQL/service/schema) but missed: the UI
entry point (no trigger existed yet to reach all-levels mode at all), the shared-component/type
consumers of the response shape (`pipeline-progress-chart-tile.tsx`,
`pipeline-progress-grid.tsx` would have silently broken), and an unstated row-identity invariant
(pagination grain changing from one-row-per-position to one-row-per-position×level). Each gap
surfaced only as a review-round finding, not at design time — the single biggest cost driver of
the whole build.
Binding rule: before dispatching `backend-engineer`/`ux-ui-engineer` on any change that alters
an EXISTING shipped feature's response shape, status/enum set, grouping/pagination grain, or
permission surface, the main loop must produce and confirm, in one pass, before writing
`design.md`:
  1. Every UI entry point that reaches the changed capability today (not just the primary
     screen) — a changed backend contract with no matching UI trigger is an incomplete design.
  2. Every consumer of the shared type/response shape being changed (`grep` the type name across
     `frontend/src` and the backend schema's importers) — list them by file, don't assume "the
     obvious ones."
  3. Any invariant implicit in the CURRENT behavior that the change could silently break (e.g.
     "one row per X" becoming "one row per X per Y") — state it explicitly, even if it seems
     obvious, because it is exactly what a downstream consumer silently depends on.
  4. All foreseeable scope-defining questions for the user asked TOGETHER in one clarification
     pass, not iteratively discovered across multiple round-trips.
This mapping is a required section of `design.md` itself (not a separate document) — a
`design.md` missing it is not ready for `backend-engineer`/`ux-ui-engineer` dispatch.

**Rule 7 — Live query-plan (`EXPLAIN`) verification is mandatory for any change touching
aggregation, ordering, or pagination grain, at BUILD time, not deferred to review (binding —
added 2026-07-31, same incident as Rule 6).**
Round 2 of `principal-reviewer`'s review found that round 1's "fix" for a redundant
correlated-subquery evaluation had NOT actually taken effect — the code, its docstring, and
`docs/BACKLOG.md` all claimed the fix landed, and every test passed, yet a live
`EXPLAIN (ANALYZE, BUFFERS)` still showed 2 `SubPlan`s. This is the load-bearing lesson: passing
unit/functional tests confirm correct RESULTS, not correct QUERY PLANS — a query can return the
right answer while doing 2x the necessary work, and no amount of reading the code or re-running
the test suite reveals that; only a live `EXPLAIN` does. The gap existed because the fix was
verified by re-reading the diff, never by actually running it against Postgres.
Binding rule: whenever a change modifies a `GROUP BY`, adds/changes a window function
(`RANK()`/`ROW_NUMBER()`/etc.), a correlated subquery, an `ORDER BY` tied to pagination, or the
grain a query aggregates/paginates over, the agent making the change (not just the reviewer)
MUST run `EXPLAIN (ANALYZE, BUFFERS)` against the real local Postgres for at least one
representative query BEFORE reporting the task as done, and paste the plan's key evidence (no
duplicate `SubPlan`/`SeqScan` where an index or single evaluation is expected) into its report.
`principal-reviewer` independently re-runs its own `EXPLAIN` regardless — this rule does not
replace that check, it moves the same check earlier so a wrong "fixed" report never reaches
review in the first place.

## Regression prevention gates (binding — Gates 1–4 overridable only via 3-request rule; Gate 5 has NO override)

These gates exist because regression rework consumed 60–70% of agent cost in multiple sessions
(root-cause analysis 2026-07-04), and a separate live-testing session (2026-07-08) burned
unrouted main-loop tokens on bug fixes that should have gone through the subagent pipeline.
Every rule is a hard gate. Gates 1–4 can only be lifted via the 3-request override rule below —
a single user request, even an explicit one, is NOT sufficient. Gate 5 cannot be lifted at all.

**Gate 1 — Run module unit tests BEFORE functional-test-engineer (mandatory).**
After every `backend-engineer` output, run the affected module's unit tests inline before
spawning any other agent:
  `cd backend && python -m pytest app/modules/<module>/tests/test_unit*.py -q`
If any test is red: fix it before proceeding. Never send red unit tests to functional-test-engineer
or principal-reviewer. This gate catches the majority of regressions in seconds.

**Gate 2 — Tech debt items tagged "user-visible failure" fixed next session, not deferred.**
Any item in `resume-pointer.md` Tech Debt Queue that causes a user-visible failure (500, wrong
data, broken UI action) MUST be the first task of the next working session. It cannot be
deferred to "after the next feature". Tracking without fixing is not acceptable.

**Gate 3 — Spec pre-read mandatory for ANY status/enum/transition change.**
Before touching `_allowed_targets`, `ALL_STATUSES`, `_ALLOWED_FROM`, any `*_enum` value, any
status constant, or any transition/allowlist map: read the relevant spec section first.
No exception. 60 seconds of reading prevents a full regression cycle.

**Gate 4 — `cavecrew-builder` for surgical ≤2-file / ≤10-line fixes.**
When the fix is well-understood and scoped to ≤2 files with ≤10 lines changed, use
`cavecrew-builder`, not `backend-engineer`. `cavecrew-builder` hard-refuses scope creep and
cannot accidentally add adjacent "helpful" changes that introduce regressions.

**3-request override rule (the ONLY exception mechanism for Gates 1–4 — binding):**
No gate in this section can be waived by a single user request, not even an explicit one.
To override any gate, the user must ask to skip it **3 times in a row in consecutive messages**.
- First ask: acknowledge the request, name the gate, explain why it exists, continue enforcing.
- Second ask: acknowledge again, note "second request", continue enforcing.
- Third ask: lift the gate for that single instance only, log the override in the response.
This rule applies to the user's own instructions too — one or two requests is not sufficient.

**Gate 5 — full subagent pipeline on every bug fix (binding — ZERO exceptions, no override).**
This gate exists because a live-testing session on 2026-07-08 ran 3 consecutive bug fixes
entirely in the main loop — no `cavecrew-investigator` for root-cause lookup, no
`cavecrew-builder`/`ux-ui-engineer` for the edit, no `principal-reviewer` before merge —
and the resulting cost escalation triggered this rule. Every bug fix, regardless of size,
severity, or how "obvious" the root cause seems, MUST route through all three stages:
1. **`cavecrew-investigator`** — root-cause lookup. Never grep/read logs or trace a stack
   trace directly in the main loop to diagnose a bug; dispatch the lookup.
2. **`cavecrew-builder`** (≤2-file/≤10-line edits) or **`ux-ui-engineer`** (frontend/UI
   changes) or **`backend-engineer`** (backend changes beyond cavecrew-builder's scope) —
   the actual fix. Never edit the fix directly in the main loop.
3. **`principal-reviewer`** — mandatory APPROVE / APPROVE-WITH-NITS verdict before
   requesting human merge. No fix goes from "browser-verified" straight to a merge request.

**This gate does NOT fall under the 3-request override rule above.** It cannot be lifted
by any number of user requests, explicit or otherwise, in any session. The only way to
change this gate is to edit this file (CLAUDE.md) directly — not a runtime request to
skip it, not a "just this once," not a 3-in-a-row override. If a fix's root cause is
already obvious, `cavecrew-investigator` still runs (confirms it cheaply); if a fix is a
one-line change, `cavecrew-builder` still runs (applies it, hard-refuses scope creep).

## Model tier & CI independence mandate (binding — added 2026-07-24, NO override, same class as Gate 5)

This section exists because a status review (2026-07-23/24) found that principal-reviewer took
3 rounds to close out a single change (a missed 22nd endpoint, then a missed spec-sync update)
even at Sonnet/high, and separately found that this repo's CI had run `ruff check .` as the FIRST
step of a single sequential job for its entire history — meaning `mypy` and the full `pytest`
suite had **never run once** until that check finally went green, silently hiding 52 mypy errors
and 18 environment-dependent test failures for as long as the repo existed. Neither gap was a
missing rule; both were structural — a single-tier model roster with no chokepoint upgrade, and a
CI pipeline where one early failure masks everything behind it. This mandate closes both, and
carries Gate 5's own restriction: **it cannot be lifted by any number of user requests, explicit
or otherwise, not by the 3-request override rule, in any session. The only way to change this
mandate is to edit this file (CLAUDE.md) directly** — not a runtime request to skip it.

**1. Model tier is two-tier, not one — binding on the 3 named roles below, no exceptions.**
`principal-reviewer`, `principal-reliability-engineer`, and `principal-performance-auditor` (the
`.claude/agents/*.md` files carrying this exact naming — file names and their `name:` frontmatter
are load-bearing, not cosmetic) pin `model: opus`. Every other agent in the roster
(`backend-engineer`, `ux-ui-engineer`, `unit-test-engineer`, `functional-test-engineer`,
`integration-test-engineer`) stays on `model: sonnet`. This split is not a suggestion to
re-evaluate per task — it is fixed until this file is edited to say otherwise. Rationale: these 3
roles are either the single merge chokepoint every change passes through exactly once
(`principal-reviewer`) or genuinely on-demand deep-dive specialists engaged only for the highest
blast-radius categories (data loss, broken recoverability, perf regressions at 200+ concurrent
users) — in both cases the model-tier cost delta (~1.7–2.5x Sonnet, see
docs/TOKEN_OPTIMIZATION_PRACTICE.md for the worked numbers) is trivial against the cost of a
defect that chokepoint or specialist was supposed to catch and didn't.

**2. CI jobs must be independent, parallel jobs — never sequential steps inside one job.**
`backend-ci.yml` and `frontend-ci.yml` (and any CI workflow added later) MUST structure lint,
type-check, and test as separate jobs with no `needs:` dependency between them, so that a failure
in one is reported on its own and never prevents another from running or reporting. A workflow
that puts `ruff check .` → `mypy .` → `pytest ...` as sequential steps inside a single job is
NOT compliant with this mandate, full stop — the exact failure mode this section exists to close.

**3. Dependency-drift detection is mandatory in CI, not optional tooling.**
Every backend CI run MUST include a dependency-drift check (`deptry` or equivalent) that fails the
build if a first-party module imports a package not declared in `pyproject.toml`. This is the
direct fix for the `psycopg2-binary` incident (2026-07-23): a package installed locally outside the
tracked dependency list, invisible to CI until CI itself broke on it. Skipping this check on any
future CI-workflow edit is itself a violation of this mandate.

**4. Cost-alert before/after and the token-optimization showcase (defined under "Token-optimized
development" below) are enforced under this same no-override class**, not the softer 3-request
rule that governs Gates 1–4. They cannot be skipped by any user request, however explicit, in any
session — only an edit to this file changes them.

## Live-verification & environment-parity mandate (binding — added 2026-08-07, NO override, same class as Gate 5)

This section exists because the `async-pipeline-durability` change (6 phases, 2026-08-05 through
2026-08-07, closing a live incident where 48 candidates sat stuck for 4+ hours with zero alert) ran
to many more review rounds than necessary — not because the review rigor was wrong (every single
phase found a genuinely severe, production-grade defect that only live execution could catch — the
rigor is what caught them), but because specific, repeatable process gaps let defects survive one
extra round before being caught, when they could have been caught in the round they were introduced.
Two concrete incidents from that change anchor this mandate: (1) a "fix" for a circuit-breaker
Gauge's stale-reading bug set `multiprocess_mode` incorrectly on the first attempt — proven wrong
only when a reviewer actually executed the pinned `prometheus_client` library against two real
processes, not by reading the diff; the same fix was accepted as correct by an earlier round that
had only read the code; (2) a fork-vs-spawn multiprocessing bug and an em-dash in a Terraform
security-group description were both invisible to every local Windows test run and to
`terraform validate`, and surfaced only on the first PR that ever ran through real Linux GitHub
Actions CI — meaning the "definition of done" up to that point had never actually included running
on the target platform. Every rule below closes one specific instance of "this could have been
caught a round earlier, for a fraction of the cost."

**1. A proposed fix to a defect touching an external library, service, or infrastructure config
(Celery, Redis/`rediss://`, Terraform/AWS resource attributes, Prometheus client internals, any
third-party SDK) is not "done" until it has been executed against the real thing — not merely read
back or reasoned about.** Reading the diff and confirming it "looks correct" against documentation
or prior knowledge is not sufficient for this class of change; training data and documentation are
frequently stale or incomplete for exact library/version behavior (e.g. `multiprocess_mode`
defaults, `rediss://` SSL parameter requirements, AWS security-group description character-set
restrictions). The agent making the fix — not only the reviewer — runs the live check BEFORE
reporting the fix as complete, the same way Rule 7 already requires a live `EXPLAIN` for query-plan
changes. This closes the incident-1 gap: the wrong fix would not have been accepted as "verified"
if verification had meant execution instead of a read-through.
**2. CI on the actual target platform is part of the definition of done for every phase, not a
follow-up check run only when convenient.** Local test runs (Windows dev machines in this project's
case) do not exercise platform-specific behavior that differs from the CI runner (Linux) or from
real cloud APIs (AWS) — `multiprocessing`'s OS-default start method, a Terraform provider's
per-field validation coverage, and container/OS-level behavior generally. A phase is not complete
until its actual CI pipeline has run and been read, not merely until local Gate 1 is green. This
closes the incident-2 gap: both defects were structurally invisible to every check that ran before
CI, and were only found because CI was watched through to completion rather than treated as a
formality after local tests passed.
**3. Tooling required to verify a change lives in the main loop's own environment before the change
is dispatched, not provisioned reactively after a reviewer flags its absence.** If a phase's own
verification needs a CLI, SDK, or credential the implementing agent's sandbox cannot reach (no
network access, no installed binary), the main loop provisions it itself — as happened mid-session
with the Terraform CLI, after 3 review rounds had already passed without a single real `terraform
validate` run — before dispatching the next round, not after the next reviewer asks for it again.
**4. A specialist review is never a substitute for the holistic final-gate review, and both always
run — never one instead of the other, regardless of how clean the specialist's sign-off looks.**
4 rounds of a deep AWS-SRE specialist review closed a Terraform/observability change at
APPROVE-WITH-NITS; the subsequent holistic `principal-reviewer` gate — looking at the same change
with a wider lens — found a defect (a `rediss://` URL configuration that would crash the application
at boot) that would have made every one of the specialist's 4 rounds of alarm/metrics work
inoperable in production, because the specialist's review scope never crossed into "does the
runtime this alarms on actually start." Skipping the holistic gate after a clean specialist sign-off
— for time, cost, or because the specialist review already felt thorough — is exactly the shortcut
this rule forbids. Both gates run on every change that has both a specialist and a holistic review
step in its pipeline; neither is optional because the other passed.

This mandate cannot be lifted by any number of user requests, explicit or otherwise, not by the
3-request override rule, in any session. The only way to change it is to edit this file directly.

## Review-round & script-verification discipline (binding — added 2026-09-02)

This section exists because G15d (a ~300-line CI-only script) took 6 principal-reviewer
rounds to close, each finding a real defect the last round missed — proof early rounds tested
only the reported shape instead of sweeping real data. A same-night near-miss (an env-var
override silently losing to `.env`'s `DATABASE_ADMIN_URL`, briefly running `alembic upgrade`
against the real local dev DB, no damage — single-transaction rollback) is the same class:
verification that should happen BEFORE an action didn't.

1. **Sweep-test from round 1.** Any function normalizing/comparing/parsing real DB- or
   production-observed strings must have its first test pass built from ALL current real
   instances + adversarial mutations, never hand-picked fixtures. State the sweep size in the
   PR/commit ("swept N real instances") — its absence is the tell that this was skipped.
2. **APPROVE-WITH-NITS → fix inline + self-verify.** No `principal-reviewer` re-dispatch unless
   the nit fix itself surfaces a new Major/Critical.
3. **3-round cap on CI/tooling scripts** (`backend/app/scripts/`, CI workflow files): after 3
   consecutive CHANGES-REQUESTED on the same file, STOP and ask the user to continue or
   ship-with-tracked-debt. Never self-authorize a 4th round.
4. **Confirm the DDL target before running it.** Before any `alembic stamp`/`upgrade` or raw
   DDL command against any DSN, state the resolved target DB name as its own line immediately
   before running it. A DDL command with no stated target line preceding it is non-compliant
   on sight.
5. **Fail loud on missing config.** A standalone script reading DSN/config from env vars must
   raise a clear "set X" error on missing config — never silently substitute a
   wrong-but-plausible fallback.
6. **Scope-chain checkpoint.** One BACKLOG item spawning ≥2 downstream items → state a
   scope+cost checkpoint to the user before continuing into the next one, not a silent chain.

`principal-reviewer`'s standing checklist additionally confirms: (a) for any normalize/compare
fix, was the sweep size stated and does it look genuinely exhaustive, not sampled; (b) for any
DDL-adjacent change, was the target DB explicitly confirmed before execution.

## Debt-escalation & pre-investigation discipline (binding — added 2026-09-03)

This section exists because debt fixed in `fix/main-ci-break` (133 mypy errors, 3 stale-test
clusters) was already fully documented in `docs/BACKLOG.md` §5 since 2026-08-08, referenced
again 2026-08-18/19 and 2026-09-01 — never newly discovered — yet sat unfixed a month because
every PR correctly deferred it as "not this PR's job," with nothing ever owning it. It was then
re-investigated from scratch on 2026-09-02 without re-reading the section that already had the
answer. A related near-miss: a fix was dispatched on an investigator's UNVERIFIED claim about
Pydantic v2's serialization behavior — wrong, and caught only because `principal-reviewer`
executed the claim independently before approving.

1. **BACKLOG-pre-read before any investigation dispatch.** Grep `docs/BACKLOG.md` for the
   symptom/file/test name first; state the result ("BACKLOG grep: no match, fresh
   investigation" or "BACKLOG grep: matched §N, reconciling"). Fresh investigation only starts
   on "no match."
2. **Debt-escalation clock.** Any `docs/BACKLOG.md` item tagged as blocking CI for ALL PRs gets
   a 14-day clock from first logged. Past 14 days unfixed, surface it to the user explicitly as
   needing a scheduling decision — never silently re-defer again. Extends Gate 2's
   "user-visible failure fixed next session" to explicitly include "CI red on main for every
   PR."
3. **Orchestrator-level live-verification of third-party claims.** Any investigation claim
   about a third-party library's runtime behavior that will justify writing NEW APPLICATION
   code must be executed by the main loop itself BEFORE the fix is dispatched — state the
   executed result as its own line. Not deferred to the fixing agent's own pre-completion
   check, not left to be caught by review after the fact.

`principal-reviewer`'s standing checklist additionally confirms: (c) does this change's failure
class already have a `docs/BACKLOG.md` entry, and if so was it reconciled rather than
duplicated; (d) for any fix resting on a third-party-behavior claim, was that claim's execution
shown, not just asserted.

## Proactive deviation flagging (binding — added 2026-09-03)

The user's overarching, non-negotiable mandate for this platform is Enterprise-class,
production-grade Quality, Performance, Scalability, Reliability, Security, Observability,
Modularity, Maintainability, and optimized/extensible Code, Design, and Architecture. This
section makes flagging risk to that mandate a standing duty, not a courtesy.

Whenever, during any task — not only at completion, and even if tangential to what was asked —
a material risk to any of those 10 dimensions is noticed in code already built, being built, or
about to be built, it is surfaced explicitly, by name, the moment it's noticed. Never assumed
already seen, never folded silently into other output where it could be missed, never deferred
to an end-of-session summary. Every flag is paired with a concrete, ECONOMICAL remediation
option — the cheapest sound fix, not the most thorough one — so the user is choosing between a
real option and "do nothing," never left with only a warning.

Threshold: material risk to one of the 10 dimensions, calibrated the same way
`principal-reviewer`'s Critical/Major/Minor/Nit tiers work — not stylistic nitpicks. Flagging
everything, always, is itself a violation of this mandate's own economy: it degrades signal and
adds cost without adding protection.

## Token-optimized development (binding — every task)

Compute cost is real; discipline keeps it proportionate to value delivered.

**Agent routing — always use the most specific agent:**
- UI work → `ux-ui-engineer`. Never fall back to `general-purpose` for UI tasks.
- Lookups ("where is X / what calls Y / map dir") → `cavecrew-investigator` first
  (~60% fewer tokens than Explore or general-purpose for pure lookups).
- Bounded 1–2 file edits → `cavecrew-builder`. Diff audits → `cavecrew-reviewer`.
- Never route to `general-purpose` or `Explore` for tasks a dedicated agent covers.

**Spec pre-read before backend-engineer (mandatory — prevents ~$10–15 CHANGES-REQUESTED cycle):**
Before spawning `backend-engineer` on any module, the main loop explicitly reads:
- `spec.md §audit` — exact audit action strings (e.g. `panelist_created`, not `create`)
- `spec.md §permissions` — exact roles for each endpoint
- `spec.md §errors` — exact error codes + HTTP status for each exception

**Skills for mechanical work (invoke via Skill tool — never write inline in main loop):**
- `/commit-message` — structured commit + provenance trailers; never write commits free-form.
- `/summarize-changes` — PR descriptions and change summaries.
- `/changelog` — release notes. Skills produce compact, structured output.

**Cost alerts — BEFORE and AFTER every task (binding, no override — added 2026-07-24, enforced
under the Model tier & CI independence mandate below; this rule applies to every task, not just
"non-trivial" ones):**
- **Before starting.** State a `[COST ALERT]` estimate for every task, scaled to size — a single-
  file edit gets a one-line estimate ("Est. <$1, 1 file"), a focused-agent task states its $
  estimate, a full module build states scope + estimate + a ~$25 check-in point, a broad sweep
  states scope (modules + dimensions) before any work starts. Format for anything beyond a
  one-liner: `[COST ALERT] Est. ~$X. Scope: [Y agents, Z files]. Proceed?`
- **After finishing.** State the ACTUAL cost/tokens spent for every task, no exceptions — compare
  against the stated estimate and flag any overrun beyond ~2x. A task that skips the before-estimate,
  the after-actual, or both is not compliant, regardless of how small it looked going in.
- This supersedes the old "simple tasks: no alert" carve-out — there is no tier that skips either
  half of this rule. Scale the *format* to the task's size, never skip the *substance*.

**Token-optimization practice showcase (binding, no override — added 2026-07-24):**
After completing any task, name which specific docs/TOKEN_OPTIMIZATION_PRACTICE.md practice(s) (or
CLAUDE.md agent-routing/gate rule) were applied, and state concretely how each one reduced this
task's actual token/dollar spend — not a generic restatement of the practice, a specific claim
about this run (e.g. "delegated the repo-wide reference lookup to cavecrew-investigator instead of
grepping in the main loop — kept ~15K tokens of raw match output out of main context" or "used
cavecrew-builder for the 1-line import fix instead of backend-engineer — skipped the spec pre-read
and a high-effort dispatch for a change that didn't need either"). If no practice meaningfully
applied to a trivial task, say so explicitly rather than omitting the section — silence is not
compliant, an explicit "N/A, single Read call" is.

**Audit scope discipline:**
Before any "audit" or broad "review the codebase" task: state which modules and which quality
dimensions are in scope. Narrow scope costs 50–70% less than a broad sweep. Default to a targeted
slice unless the user explicitly requests a broad sweep.

**Model + effort pinning (binding — see docs/TOKEN_OPTIMIZATION_PRACTICE.md §D8–D12; model tiers
themselves are governed by the "Model tier & CI independence mandate" below, which supersedes the
single-tier statement this paragraph used to make):**
Every named subagent in `.claude/agents/` pins its `model:` explicitly — never omitted/inherited —
plus an `effort:` default matched to its actual judgment load. As of the 2026-07-24 model-tier
mandate, the roster is two-tier, not one: `principal-reviewer` (opus/high — the standing merge
gate) and the two on-demand specialists `principal-reliability-engineer` / `principal-performance-
auditor` (opus/xhigh) are pinned to Opus; `backend-engineer`/`ux-ui-engineer` (sonnet/high) and
`unit-test-engineer`/`functional-test-engineer`/`integration-test-engineer` (sonnet/medium) stay on
Sonnet 5. **Max effort is reserved for cases where it is absolutely required to protect quality or
a hard review gate — never a standing default**; using it by default measurably slows throughput
without a proportional quality gain on routine work. The main loop's own mechanical tool calls
(grep/glob/read/count-style investigation) default to low effort; reserve high/max for design
decisions, synthesis, and adversarial review. Before the first agent call in a session that
overrides `model`, verify it resolves once (not once per call) — a bad model id fails mid-run
after tokens are already spent. Cost-alert dollar thresholds ($10/$25 above) stay fixed across
model-tier changes — do not recalibrate them when the orchestrator model changes.

**CI-minute discipline (binding, no override — added 2026-08-08):** PR #221's e2e-CI fix alone
burned an estimated 150-250 of GitHub Actions' ~2000-minute monthly allotment across 6 full CI
reruns — one of them (round 4) a 5-line docs-only edit that still paid for a full ~15-minute run on
both workflows. Run `gh api users/<owner>/settings/billing/usage` per the existing GH Actions
usage-check practice before any CI-triggering batch; the three rules below close the concrete waste
found this session:
1. Any commit whose diff is confined to `.md` files, OpenSpec artifacts, or other pure
   documentation/bookkeeping (no code, config, test, or CI-workflow file touched) MUST include
   `[skip ci]` in its commit message — GitHub Actions honors this and never triggers the workflow,
   so a doc-only commit costs zero CI minutes instead of a full run.
2. The instant a fix is independently confirmed wrong (a reviewer refutes it, re-reading the
   diff/logs contradicts the claimed root cause, or its CI run is running noticeably longer than
   the established baseline for no good reason), cancel the in-flight run immediately — don't let a
   run that's already known pointless finish anyway.
3. Batch multiple small doc/bookkeeping edits into ONE commit instead of one push per edit — same
   discipline as the existing PR-batching practice (5-8 bug fixes per PR). Each push is a separate
   CI trigger; five 1-line doc edits pushed separately cost 5x a single batched commit.
Explicitly NOT covered here — needs the user's own scoping call, not a unilateral change: trimming
Playwright's e2e job's 4-browser-project matrix (full matrix only for critical/NFR specs, a
reduced 1-2-browser matrix for routine functional specs) — tracked as tech debt in
`docs/BACKLOG.md`; the reduced matrix cannot ship "without any errors" until the currently-known
flaky webkit tests (same BACKLOG entry) are actually fixed, not just excluded from the count.

**CI-cost-under-real-billing mandate (binding, no override — added 2026-08-19).** CR#1's build
session (2026-08-18) pushed 9 times to one branch — 2 rounds discovering CI-only-visible defects
(a Windows-only venv path, a missing Celery worker in `frontend-ci.yml`) that local Windows
testing structurally could not catch, plus 3 review-driven fix rounds each re-triggering full CI
— and took the account from ~1,381 to 1,991 of the 2,000-minute August allotment (confirmed live
on the GitHub billing UI, 2026-08-19 — ~9 minutes of free headroom left, $0 billed so far because
discounts still fully offset the $12.72 gross usage), with no `$0` spending-limit budget
configured — meaning every further Actions minute this cycle, past that ~9-minute margin, is now
real per-minute billing, not just quota risk. This mandate closes the gap: some of those 9 pushes
were genuinely necessary (CI is the only real Linux runtime this repo has — Rule 2 of the
Live-verification mandate above still stands, do not fake that check locally), but the
*iterative discovery* pattern — push, read
failure, fix, push again — is the expensive part, and much of it was avoidable with more
verification done before the first push, not after.
1. **Local-verification-first, before ANY push touching code/config/test/CI-workflow files:**
   run the full local equivalent of every CI job — `ruff check .`, `mypy .` (matching CI's exact
   invocation, not a subset that skips the `tests/` exclude and reports false positives), the
   full `pytest` suite including `RUN_DB_TESTS=1` integration tests, `tsc --noEmit`, `eslint`,
   `vitest`. For any NEW Playwright/e2e coverage specifically, run it locally against the local
   dev stack first — a local pass doesn't guarantee a Linux-CI pass (the Windows-venv-path class
   of defect is exactly what a local Windows run cannot catch), but it eliminates every defect
   that IS catchable locally before spending a single real CI minute discovering it.
2. **Review the diff before the push it belongs to, not after a CI failure surfaces what review
   would have caught.** Dispatch `principal-reviewer` (and any needed specialist) against the
   local, unpushed diff — reading files and running local commands doesn't need Actions. Fix
   every finding, THEN push once. A push that immediately draws a real review round of
   CHANGES-REQUESTED is a push that should have waited for the review it triggers.
3. **Consolidate to the minimum number of CI-triggering pushes a change genuinely needs.** Batch
   backend + frontend + tests + docs for one logical change into as few pushes as the actual
   dependency chain allows — never push speculatively to "see what CI says" when the same check
   is available locally. When a real CI failure IS found and fixed, batch that fix with any other
   already-known-needed fix rather than pushing each fix the instant it's ready.
4. **State the real dollar cost, not just elapsed minutes, in every CI-triggering cost alert once
   the account is over its included allotment** — at GitHub's private-repo Linux-runner rate
   (`pricePerUnit` from the billing-usage API, ~$0.006-0.008/min), a ~15-minute `frontend-ci.yml`
   run now costs real money on every trigger, and the cost alert must say so explicitly, not just
   log the minute count.
This mandate does not relax Rule 2 of the Live-verification & environment-parity mandate — CI on
the real target platform remains part of the definition of done. It closes the gap between "CI
is necessary" and "CI should be the first place a catchable defect is found."

**Playbook is living — update it in the same PR that surfaces a new technique:**
`docs/TOKEN_OPTIMIZATION_PRACTICE.md` is the durable, org-shareable record of all cost-optimization
practices. When a new technique is identified, an anti-pattern is discovered, or a benchmark is
measured: update the playbook in the SAME PR as the triggering change and add a dated Changelog entry.
Never let the playbook drift from actual practice. This is binding.

## Verification discipline (binding — added 2026-07-14)

This section exists because a single bug-fix session ran to ~$1K by re-verifying every
angle inline instead of trusting the gate. Live, exploratory self-verification in the main loop — creating
fresh test fixtures, chasing every tangential finding to its root cause, re-testing scenarios
nobody asked about — is the single most expensive failure mode observed to date, more costly
than any individual bug. The fix is procedural, not "try harder to be brief":

1. **Verify the reported scenario once, then stop.** When a user reports a specific repro, the
   main loop's own live-HTTP verification (if any is done at all — prefer delegating to
   functional-test-engineer) is scoped to THAT scenario, confirmed pass/fail, done. Do not expand
   into adjacent scenarios ("let me also check what happens if..." ) unless the ORIGINAL
   verification itself concretely fails and root-causing requires it.
2. **Trust the review gate.** principal-reviewer's APPROVE is the correctness signal once a fix
   is dispatched with a clear, complete brief (exact bug, exact fix, exact files). Do not
   re-derive what the reviewer already independently re-verified. The main loop's job is to brief
   well and act on the verdict — not to duplicate the reviewer's own re-verification pass.
3. **One finding, one fix, one re-check — not a chain.** If self-verification surfaces a NEW
   defect, root-cause it minimally, dispatch ONE fix, re-verify that ONE thing, then move on.
   Do not let one discovery cascade into re-deriving unrelated system behavior (e.g., a stray
   404 during verification does not license auditing an entire unrelated data model).
4. **Cost checkpoint, not cost narration.** If verifying a single fix requires more than one
   fresh test-fixture setup or more than one root-cause detour, STOP and either hand the rest to
   functional-test-engineer/cavecrew-investigator, or surface a `[COST ALERT]` and ask whether to
   continue deeper or ship at current confidence — do not silently keep digging.
5. **Ad-hoc cleanup scripts touch only what was created THIS run.** Never delete/soft-delete an
   entity merely REFERENCED by something created this session (e.g. a `position_id` FK) without
   first confirming, via its own identifying field, that it was ALSO created this session (the
   project's `FT-`/`Test`-prefix convention exists precisely for this check). A referenced entity
   with no such prefix is presumed a pre-existing shared fixture — leave it alone. (Incident:
   2026-07-14, an ad-hoc cleanup blanket-deleted a shared dev-DB position's `interview_levels`
   because it was merely referenced by a created test application; see
   `memory/tech-debt-issue5-fixture-broken.md`.)

## NFR compliance checklist (binding — every PR, not just NFR-labeled work)

Every code change — feature, fix, or refactor — carries these consequences whether or not it's
labeled "performance" or "reliability" work. Confirm each applies (or is explicitly N/A) before
requesting merge; principal-reviewer's mandate checklist already covers items 1–4 structurally —
this section names the outcome each one is actually protecting:
1. **Performance & scalability (target: 200+ concurrent users, no perceptible screen-response
   slowdown).** Async I/O throughout, no N+1 (`selectinload`/explicit JOIN), indexed access on
   every new filter/lookup column, bounded/paginated results, heavy work offloaded to Celery —
   never on the request path. NFR Phases 2b (root-cause perf diagnosis) and 2c (load-testing
   harness, 200-concurrent-user validation) remain parked as tech debt pending Dashboard/Reporting
   completion — do not let new feature velocity push their eventual start further out; flag this
   to the user proactively per the existing `project_nfr-deferred-for-dashboard-reporting` memory.
2. **Reliability & resilience.** Idempotency-Key on every mutating endpoint, optimistic
   concurrency on every versioned update, RLS context set in every Celery task touching an
   RLS-protected table (binding Rule 3). Fire-and-forget patterns (BR-SYNC-005-style: a sync
   step whose failure must not roll back the primary write) are acceptable ONLY when paired with
   structured logging on every swallowed exception (see Observability below) — a bare
   `except Exception: pass` with no log line is never acceptable, because it can silently discard
   unrelated work in the same transaction (incident: 2026-07-14, a swallowed CHECK-constraint
   violation discarded an otherwise-correct feedback update and status revert in the same request
   — see `memory/feedback_interview-workflow-regression-lessons.md`).
3. **Recoverability.** Soft-delete + audit on every entity lifecycle; every schema change
   additive and reversible (existing Database Schema Evolution section); every migration's
   downgrade path actually implemented, not stubbed.
4. **Observability.** structlog only, never print(); no PII (email/mobile/resume text) in any
   log line — log the entity id instead. Every exception-swallowing block (fire-and-forget syncs,
   Celery task bodies) MUST emit at least a `logger.warning(...)` carrying the relevant entity id
   — silence on failure is not acceptable, full stop. This is the concrete, checkable form of
   "observability" for this codebase: if a background sync fails in production, there must be a
   log line naming which entity it failed for.
5. **100% functionality, minimal lines.** The Engineering Mandate's "no line that doesn't earn
   its place" and Quality Gate Rules 1–5 are how "100% functionality implemented" and "minimal
   token/cost" are actually achieved together — they are not in tension. Under-scoping a fix to
   save tokens (skipping Gate 5's pipeline, skipping a spec update, skipping cleanup) produces
   rework that costs far more than doing it right once. The lean target is fewer *exploratory*
   tokens (Verification discipline above), not fewer *correctness* tokens.

## Agent provenance & review gate (binding)
Every change records WHO produced it and HOW it was verified, so traceability is durable and
versioned — not buried in chat. This is enforced in the SAME change, never after:
- **Commit trailers** on each commit: `Agent: <producing-agent(s)>` and
  `Reviewed-by: <reviewer-agent — verdict>` (e.g. `principal-reviewer — APPROVE`), above the existing
  `Co-Authored-By` trailer. Keeps `git log` queryable for agent attribution.
- **PR body** carries the Provenance + Mandate-compliance block from `.github/pull_request_template.md`
  (agent pipeline, principal-reviewer verdict, mandate checklist). API-created PRs follow the same shape.
- The **principal-reviewer** verdict is the merge-readiness signal; a feature change reaches "request
  human merge" only at APPROVE / APPROVE-WITH-NITS (nits fixed). The human still approves the merge.

## Git
- Branch per feature: dev/<phase>-<feature>. Conventional commits (feat:, fix:, test:, chore:).
- PR for every feature; never push directly to main.
- **Local-dev sync (standing):** After EVERY merge to main — before starting any new feature work —
  run `git checkout main && git pull origin main && alembic upgrade head` and confirm
  `alembic current` shows `(head)`. Local dev DB and main must always be at the same
  Alembic head.

### GitHub Actions minutes — usage check is mandatory, not optional (binding — added 2026-08-02)

This project ran multiple sessions (2026-07-30 through 2026-08-02) where every PR's CI checks
instant-failed with `The job was not started because recent account payments have failed or
your spending limit needs to be increased` — the Actions minutes allotment was silently
exhausted, and every merge across that window sat blocked for days with no proactive warning.
The gap was never checking actual usage against the plan's allotment before triggering more CI
runs — each new push/PR/rerun just kept consuming minutes into a quota nobody was watching.

**Rule:** before starting any batch of work that will trigger multiple CI runs (several PRs,
several pushes to open PRs, several `gh run rerun`s) — and at minimum once per session that
touches CI — check current Actions usage:
```bash
gh api users/<owner>/settings/billing/usage
```
(the older `settings/billing/actions` endpoint returns `410 — moved`; this is the current
replacement, grouped by month/SKU — `"Actions Linux"` quantity is the minutes consumed this
billing cycle, `"unitType":"Minutes"`; `netAmount: 0.0` with `discountAmount == grossAmount`
means still inside the plan's included allotment, a non-zero `netAmount` means already billing
past it). For an org-owned repo use `gh api orgs/<org>/settings/billing/usage` instead.

**How this changes behavior, not just observation:** if usage is trending toward the
allotment, group and batch upcoming commits/merges instead of triggering a fresh CI run per
small push — same discipline as the existing PR-batching practice
([[feedback_pr-batching-strategy]]: 5-8 bug fixes per PR), now driven by an actual number
instead of a guess. Flag it to the user proactively when usage looks like it's approaching
the allotment — don't wait for the instant-fail signature to be the first sign, the way this
incident played out.

## External-sharing artifact sync (binding — added 2026-07-30)

The public GitHub Pages repo `hareeshstggit/ats-platform-journey` (a separate, standalone
repo — `isFork: false`, `parent: null`, zero git relationship to this repo) mirrors
`ONBOARDING.md` plus 18 real, full, un-condensed reference files for external/cross-org
sharing (`ShareOnboardingGuide`'s Claude-org-scoped link only reaches STG/STG Labs
accounts — this Pages site is the only channel that works for anyone outside that org).
A stale mirror defeats the entire point of a "hardcopy for reuse" — a recipient opening
the public URL must always see what's actually on `main` today, not a snapshot from
whenever the mirror was last built.

**Rule:** after EVERY PR merge to `main`, check whether the merge touched any of the 18
mirrored files or `ONBOARDING.md` itself:

- `.claude/CLAUDE.md`, `docs/ARCHITECTURE.md`, `docs/SCHEMA_EVOLUTION.md`,
  `docs/SCHEMA_CHANGE.md`, `openspec/specs/positions/spec.md`,
  `.claude/rules/ats-ux-ui-guardrails.md`, `docs/BACKLOG.md`, `memory/resume-pointer.md`,
  `docs/TOKEN_OPTIMIZATION_PRACTICE.md`, `docs/EXECUTION_METHODOLOGY.md`, all 8
  `.claude/agents/*.md` files.
- If yes: re-copy the changed file(s) byte-perfect (`git show main:<path>` — never
  hand-retype, avoids transcription risk on large files), and if `ONBOARDING.md` itself
  changed, rebuild `index.html` (`npx marked --gfm` + the existing styled wrapper), then
  commit and push to `ats-platform-journey`'s `main` branch. GitHub Pages auto-rebuilds
  on push — no separate Pages-API call needed after the first-time setup.
- If no referenced file changed: no action needed for that merge.

This is a same-batch step, not a separate follow-up task — do it as part of closing out
the merge, the same way `docs/BACKLOG.md`/`memory/resume-pointer.md` get updated inline
per the Progress capture rule above. Full mechanism, incident history (why gist-hosting
was tried and rejected — GitHub forces `text/plain`+`nosniff` on all raw gist content,
verified via `curl -I`), and the exact build steps are recorded in the durable memory
`project_external-sharing-github-pages.md`.

## ATS UX/UI guardrail

Before creating or changing any ATS UX/UI mockup, screen, component, page, route, design artifact, or production UI, follow the project rule in `.claude/rules/ats-ux-ui-guardrails.md` (the binding rules — *what* the UI must satisfy). The end-to-end process for *how* we design, build, and test the UI is `docs/UI_STRATEGY.md` (spec-first → design → build → verify), executed by the `ux-ui-engineer` subagent.