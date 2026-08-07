**Artifact Version 2.2 — Baselined 11-Jun-2026**  ·  Versioning rule: docs/VERSIONING.md

---

# ATS Platform — Backlog & Tech Debt Reference

**Last updated: 2026-08-05**  ·  Maintainer: Claude Code, on behalf of hareesh@stg.com

Single, query-free reference for every open item — active queue, pending merges, tech debt,
deferred features. Check here first; no need to ask for a backlog recap.

**Maintenance rule (binding):** update this file inline, in the same turn/commit, whenever an
item completes, changes status, or a new one is found — same living-doc discipline as
`docs/GO_LIVE_CHECKLIST.md`. This file is the queryable snapshot; `memory/resume-pointer.md`
remains the chronological narrative (what happened, when, why) — this doc never carries dated
narrative, only current state.

**Status legend:** 🔴 Not started · 🟡 In progress / partial · ✅ Done · ⏸️ Parked / deferred ·
❓ Needs verification before trusting

---

## 1. Active execution queue (user-ordered, month-end push — one PR each)

| # | Item | Status |
|---|------|--------|
| 1 | NFR Phase 3 — repo-wide dead-code + missing-docstring sweep | ✅ Live on production (PR #195) |
| 2 | Notifications real fan-out — email via AWS SES for interview.scheduled/offer.sent | ✅ Merged (PR #196) |
| 3 | CI test gap — Postgres/Redis services not provisioned (18 backend tests fail in CI) | 🔴 Queued (3rd) |
| 4 | Frontend test debt — nav-items.test.ts, position-schema.test.ts, others (6+ pre-existing failures) | 🔴 Queued (4th) |
| 5 | e2e CI job-design gap — MSW can't intercept proxied backend calls | 🔴 Queued (5th). **Confirmed 2026-08-07 (PR #218):** every spec now fails at the login/MFA step specifically ("Verify and continue" stuck disabled — the MFA verify call never resolves) — reproduced identically by `organizations.spec.ts`, `positions.spec.ts`, and the new `pipeline-retry-badge.spec.ts` (all pre-existing or unrelated to that PR's own diff), confirming this is a single root cause blocking ALL e2e coverage, not per-spec flakiness. |
| 6 | `terraform-plan.yml` CI check has no path to ever pass — no AWS-credentials step exists anywhere in the workflow (confirmed via `git log`: file untouched since the original project-scaffold commit `537b06d`), so `terraform init -backend-config=environments/<env>/backend.hcl` cannot authenticate to the real S3 backend. **Discovered 2026-08-07 (PR #219, async-pipeline-durability Phase 6)** — this is the FIRST PR in the project's history to touch `infrastructure/terraform/**`, so the workflow's `on.pull_request.paths` filter never triggered it before now. Not caused by Phase 6; not fixable within any single PR's scope — needs a real AWS account, GitHub Actions secrets (`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` or an OIDC role), and an `aws-actions/configure-aws-credentials` step added to the workflow, which is an infra/credentials decision requiring explicit user approval, not a code fix. Every future PR touching Terraform will show this same 3-way (`dev`/`staging`/`prod`) failure until it's addressed. | 🔴 Queued (6th) |

## 2. Pending merges

None open. (`feat/pipeline-progress-all-levels` PR #206 merged 2026-08-02, then superseded same-week
by PR #209's status-groups redesign after live user testing rejected #206's shape — see §9 and
`memory/resume-pointer.md` for the full 3-round review history.)

## 3. Business-rule gaps (unverified candidates — spot-check before building, none touched since found)

1. 🔴 Offers has no mirrored org-rejection-ban check at offer-create time (BR-014 exists for applications, not offers).
2. 🔴 Screening: reverting shortlisted→screen_rejected doesn't cascade-invalidate existing interviews (orphaned pending interview records).
3. ❓ Applications: manual status→`onboarded` bypassing override_reason gate — likely false positive (onboarded already requires `onboarded_at`), recheck before building.
4. 🔴 Interviews: feedback outcome submittable with no `scheduled_at` set — no validation gap-check done yet.
5. 🔴 Interviews: kit regeneration on reschedule (`PATCH /schedule`) lacks an idempotency/duplicate-prevention guard.
6. 🔴 Interviews: BR-SYNC-005 auto-sync is fire-and-forget with a bare exception-swallow, no retry/logging (reliability/observability item, not really a missing BR).

## 4. Tech debt — data/query correctness

- ✅ `positions/repository.py::get_interview_level()` (2026-07-29, `dev/tech-debt-batch1`) — added `is_active IS TRUE`, matching `child_repository.py::list_levels()`. Sole caller confirmed write-path-only (interview-creation validation); no read/history path affected. Now returns 404 `INTERVIEW_LEVEL_NOT_FOUND` for a deactivated level id at create-time — `interviews/spec.md` 404-trigger line synced in the same branch. Merged with PR #204's multi-panelist eager-load (`selectinload(InterviewLevel.panelists)`) on the same method — both survive together.
- 🔴 Dead `sys.path.insert(..., "../../..")` + `import sys` in 3 backfill scripts (`backfill_legacy_feedback_outcome.py`, `backfill_owning_recruiter_id.py`, `backfill_panelist_login_accounts.py`) + `seed_uat_dataset.py` (same dead line, same reason) — targets the repo root, which has no `app` package; imports actually resolve via the editable install. Removed from the other 2 (`backfill_level_type_org_correction.py`, `backfill_restore_ai_coe_engmgr_levels.py`) in `dev/tech-debt-batch1`; these 4 pre-date that batch, left alone as out of scope.
- ✅ `screening/repository.py::list_decisions()` (2026-07-29, `dev/tech-debt-batch1`) — added `AND deleted_at IS NULL` to both JOINs, matching `applications/repository.py`'s pattern.
- 🔴 `category_rank` SQL duplicated across 3 modules (interviews, reporting, applications) — kept manually in sync, never extracted to a shared fragment; divergence risk.
- ❓ `positions/schemas.py`'s `InterviewLevelRequest`/`InterviewLevelResponse` missing `level_category` field (code behind spec, BUG-4).
- 🔴 Organizations/Departments DELETE endpoints documented in spec, never built (4 ACs unverifiable).
- 🔴 CR-002 multi-panelist-per-level: `positions/levels_service.py::_levels_to_response()` dropped the
  defensive `if panelist else None` null-guard when mapping `PanelistSummary.panelist_name` — since
  `interview_panelists` is RLS-protected, a caller for whom RLS filters out the panelist row would hit
  an `AttributeError` instead of a clean `None`. Restore the guard or document why it's provably unreachable.
- 🔴 CR-002 multi-panelist-per-level: `positions/models.py` crossed the 300-line cap (279→310) — extract
  `InterviewLevel`/`InterviewLevelPanelist` into a sibling module, matching the precedent already set by
  splitting `levels_service.py` out of `subresource_service.py` for the same reason.
- 🔴 CR-002 multi-panelist-per-level: `InterviewLevel.panelists` relationship uses `lazy="selectin"`,
  breaking the file's own convention (`panelist` sibling uses `lazy="raise"` deliberately, so a missing
  eager-load fails loudly instead of silently N+1-ing). Both real read paths already pass an explicit
  `selectinload`, so `lazy="raise"` should be safe — change it to match convention.
- 🔴 CR-002 multi-panelist-per-level (`dev/interview-level-multi-panelist`): panelists 2-3 configured
  via `POST /positions/{id}/interview-levels` do NOT get an automatic `interview_panelist_assignments`
  slot at interview-creation time — only slot 1 auto-assigns (dual-written to the legacy
  `interview_levels.panelist_id` column so CR-001's existing auto-assign branch keeps firing).
  Scheduling with 2-3 panelists currently requires the existing manual
  `POST /interviews/{id}/panelists` endpoint as a follow-up step. Needs a product decision: extend
  auto-assign to cover slots 2-3, or leave manual — note BR-004 caps STG slots at 2 while BR-064
  allows 3 for any level, and `add_panelist` enforces BR-004's cap, so naively extending auto-assign
  would bypass its own cap for STG levels.
- ❓ UX inconsistency (spec-faithful on both sides, not a bug): `GET /departments` orders by name ASC (spec AC-004), `GET /organizations` orders by updated_at DESC (no ordering AC) — two sibling admin list screens sort differently. Needs a product decision on whether organizations should get a name-ordering AC to match.
- ⏸️ Pipeline Progress chart segment ink (`SEGMENT_INK_BY_SLOT` in `pipeline-group-bar-labels.tsx` / `pipeline-colors.ts`) is light-mode-only; 13/18 (slot, bucket) combinations fail WCAG AA on a dark surface — principal-reviewer independently recomputed contrast (M3 finding, CHANGES-REQUESTED on commit `cffee13`, 2026-08-05), worst cases slot 6/since_start 1.15:1, slot 3/since_start 1.21:1, slot 4/since_start 1.22:1, vs the 4.5:1 AA threshold. No user impact today — confirmed no `ThemeProvider` or `.dark`-class-toggling code exists anywhere in `frontend/src/`, so `darkMode: ["class"]` in tailwind.config is unreachable dead config today — but a dark-mode ink table is a prerequisite before any future dark-mode/theme-toggle feature ships.
- 🔴 Alembic migration `0011_candidate_matches_source_details.py` is UNREPLAYABLE — it can never
  execute, from a clean environment or anywhere else, via this project's documented bootstrap path
  (load `docs/schema.sql` → stamp `0001_baseline`, which is stamp-only with zero DDL → `alembic
  upgrade head`). Two independent reasons, both verified live against real Postgres (principal-
  reviewer round 3, async-pipeline-durability Phase 3, correcting round 2's wrong premise that this
  was merely a naming/shape "drift" needing reconciliation): (1) `docs/schema.sql`'s bootstrap
  already creates `candidate_position_matches` and its FULL (non-partial) `uq_cand_pos_match` index
  before migration 0011 would ever run, so 0011's own `create_table` + index DDL has no clean state
  left to apply against; (2) independently, 0011's partial-index predicate
  (`postgresql_where=status::text != 'dismissed'`) is invalid regardless — Postgres rejects it with
  `InvalidObjectDefinitionError: functions in index predicate must be marked IMMUTABLE`, since the
  enum-to-text cast on this column is STABLE, not IMMUTABLE. The live DB (and every environment
  provisioned the documented way) carries the full index only; `CandidateRepository.upsert_match`'s
  `ON CONFLICT` target works correctly against it with or without an `index_where=` predicate
  (verified live both ways) — no code fix is needed for `upsert_match` itself. This migration needs
  either a real fix (drop the invalid predicate, confirm it's dead code post-bootstrap) or explicit
  retirement — flagged here, not attempted in this change.

## 5. Tech debt — tests/CI

- 🔴 `offers/tests/test_functional_hiring_uniqueness.py` — 3 of 6 tests fail against the live stack (drives hire-uniqueness via a manual status PATCH to `hired`, which is 422-blocked since BR-054). Needs a rewrite to go through the real `POST /offers/{id}/accept` path.
- 🔴 `tests/unit/test_seed_dev.py::test_run_seed_fresh_creates_users_grants_and_audits` — pre-existing failure from the 2026-07-23 mypy-cleanup pass, root cause still not investigated.
- ✅ `departments/tests/test_repository.py::test_list_scopes_to_org_searches_and_orders` (2026-07-29, `dev/tech-debt-batch1`) — root-caused as a real code defect (spec AC-004 requires name-order, code ordered by updated_at desc). Fixed the repository, not the test. Independently confirmed by NFR Phase 2b's own review (same root cause, same conclusion) — merged together with PR #197's `COUNT(*) OVER()` windowed-query optimization on the same query.
- ✅ `organizations/tests/test_repository.py::test_list_applies_search_and_active_filters_and_orders_by_updated_at` (renamed from `..._orders_by_name`, 2026-07-29, `dev/tech-debt-batch1`) — root-caused as a genuinely stale test (organizations spec has no ordering AC). Fixed the test's assertion, repository untouched.
- 🔴 Pagination edge case inherited from PR #197's `COUNT(*) OVER()` conversion (departments/organizations + 4 other repos): a page request past the end of the result set (`offset >= total`) returns `total = 0` (from `rows[0].total_count if rows else 0`) instead of the true total — the old separate `SELECT count(*)` didn't have this gap. A UI landing on an out-of-range page (e.g. after deletions) would show "0 results" and lose the ability to page back. Found during PR #199's merge-conflict review (2026-08-02); not fixed there (out of scope for that change) — needs its own fix across all 6 converted repos.
- 🔴 3 pre-existing `positions` module test failures: `test_service.py::test_change_status_stale_version_raises_409` (test itself is stale — needs a rewrite to a real reachable scenario), `test_tasks.py::test_extract_storage_miss_persists_failed` + `test_extract_success_persists_result` (mock/fixture drift, not chased to root cause).
- 🔴 No frontend component test for `positions-ageing-report.tsx` / `positions-ageing-bucket-strip.tsx`.
- 🔴 = Execution Queue item 3: backend CI provisions no real Postgres/Redis — ~18 tests fail (Celery result-backend retries, 2 tests dialing Postgres directly).
- 🔴 = Execution Queue item 4: frontend `nav-items.test.ts` (2 failures, stale expected nav list) + `position-schema.test.ts` (3 failures) — both pre-existing, confirmed multiple times, never fixed.
- 🔴 `frontend/src/components/positions/interview-org-labels.test.tsx` fails on `main` (pre-existing, root-caused during `dev/interview-level-multi-panelist` review — unrelated to CR-002). The read-only interviewer view in `interview-levels-editor.tsx`'s `!canConfig` branch (~line 175-193) renders only `l.level_label` (e.g. "Bar raiser") and never the org-prefixed `"({Org} : Level-N)"` format the test expects. Needs a fix to either the component (add the org-prefix label to the read-only row) or the test's expectation — flag both `interview-org-labels.test.tsx` and `interview-levels-editor.tsx` when picked up.
- 🔴 = Execution Queue item 5: `e2e` Playwright job has no backend/DB service — every login-dependent spec times out. Needs either real backend+DB services added to the job, or MSW extended to cover the proxied routes.

## 6. Tech debt — code hygiene (oversized files, 300-line/40-line caps)

**Analysis complete 2026-07-29 — full per-file decomposition plan in `docs/CODE_HYGIENE_DECOMPOSITION_PLAN.md`** (13 parallel read-only planning passes, re-swept to **48 files over 300 lines**, up from 38 at the Phase 3 audit). **Tier 1 ("free wins") executed same day — PR #198 merged**, branch `dev/hygiene-tier1-free-wins` — 14 files split across 9 commits (security/router.py, interviews/router.py, candidates/{tasks,screening/service,router}.py backend; lib/api/candidates.ts, mocks/position-handlers.ts, panelist-list.tsx, position-detail.tsx, interview-levels-editor.tsx, screening-detail.tsx, applications-in-candidate-card.tsx, screening-list.tsx, positions-ageing-report.tsx frontend). principal-reviewer APPROVE-WITH-NITS on round 2 (round 1 CHANGES-REQUESTED: 2 gratuitously-exported helper functions + a test asserting a hardcoded string instead of the task symbol — both fixed; round 2 found 1 doc-contradiction nit — fixed). Zero behavior/API/permission change anywhere — verified via byte-identical OpenAPI dump + route-table diff against `main`. Deferred from Tier 1: `positions/router.py`, `jd-panel.tsx`, `use-positions.ts` (would have conflicted with NFR Phase 2b PR #197 — now merged, safe to resume in Tier 2). Remaining tiers (2-4, including the highest-risk `interviews/service.py` 1413 lines and `applications/service.py`) not started — see the plan doc's "Recommended execution order."
- 🔴 `frontend/src/components/reports/interview-pipeline-progress-report.tsx` — 366 lines, ~22% over the 300-line cap (was already 332 lines before this change). Grew during the pipeline-progress-all-levels build (all-levels entry point, level column, Select-All wiring); flagged by `ux-ui-engineer` in round 3 but not split there (out of scope for that fix). Needs its own decomposition pass, same as the Tier 1/2 hygiene work above.

## 7. Tech debt — dependencies & secrets

- ✅ Hardcoded plaintext DB password in one-time backfill scripts' DSN strings (`backend/app/scripts/backfill_*.py`, 2026-07-29, `dev/tech-debt-batch1`) — all 5 scripts moved to `get_settings().DATABASE_ADMIN_URL` (existing project helper, not raw `os.environ`), stripping the `+asyncpg` driver suffix these psycopg2-based scripts don't understand.
- 🔴 3 declared dependencies with zero imports anywhere in `backend/`: `sentry-sdk`, `prometheus-client` (both likely reserved for parked NFR observability work), `openpyxl` (retained as FK target per GO_LIVE_CHECKLIST's Scorecard-CRUD-removed note, PR #50 — confirm genuinely dead or living outside `app/` before removing). Currently `deptry`-ignored, not removed.

## 8. NFR (parked since 2026-07-10; unpark condition met 2026-07-23 — Dashboard+Reporting shipped)

- ✅ Phase 3 — repo-wide dead-code/quality sweep (PR #195, docstrings+dead-code only; oversized-file hygiene split to §6 above).
- ✅ Phase 2b — perf root-cause diagnosis + fixes via principal-performance-auditor. Branch `dev/nfr-phase2b-perf-fixes`, **PR #197 merged**. Findings P0-P9:
  - ✅ P0 — Postgres IPv6 connect tax (`DATABASE_URL`/replica/admin → 127.0.0.1; Redis deliberately untouched, listens IPv6-only on this machine).
  - ✅ P1 — bcrypt blocking event loop (`security/service.py` verify/hash_password → `run_in_executor`).
  - ✅ P2 — DB pool undersized (`core/database.py` → explicit pool_size=20/max_overflow=30/pool_recycle=1800).
  - ✅ P3 — JD extraction async fix. 4 principal-reviewer rounds, each catching real narrowing issues (commit-ordering race, missing frontend polling, a bug in the polling fix itself, then 2 mutation-tested test-coverage gaps) — all closed. Also surfaced and documented a real Windows-only Celery infra issue (`docs/LOCAL_DEV.md`: `--pool=solo` required, default prefork crash-loops on this dev machine — not a code defect, but looks exactly like one from the outside).
  - ✅ P4 — report queries LIMIT-after-full-computation, fixed in `_pipeline_progress_sql.py` (page_positions CTE pushes LIMIT before the 5 event-join CTEs).
  - ✅ P5 — unbounded sub-collection endpoints. `applications`/`interviews` `/status-history` now paginate (limit/offset, default 50/max 200); spec updated same-PR. `positions/{id}/history` has the same unbounded shape but lives in `subresource_service.py` — follow-up, not yet scheduled.
  - ✅ P7/P6 — safely-parallelizable sequential awaits (~5 DB round trips/request), CLOSED. `set_rls_context` (2→1 round trip, unit-tested in `backend/tests/unit/test_core_database.py` including placeholder-to-GUC pairing so a bind-swap can't silently pass), `User.role` selectin→joined, 4 repos' count+rows → `COUNT(*) OVER()` (departments' assertion moved to the passing `test_list_without_search_omits_like` per M2), `Role.permissions` `lazy="selectin"` → `lazy="raise"` per M4 (closes the last round trip; only usage anywhere in `backend/` is a class-level `.join(Role.permissions)` in `security/repository.py`, unaffected by the loader-strategy change). N1 (unused fixture param), N2 (per-fixture post-teardown verification), N3 (idempotent teardown safety net) also closed. principal-reviewer: APPROVE after remediation round.
    - **Root cause of the pre-existing `departments` test failure, now confirmed and FIXED:** `openspec/specs/departments/spec.md` AC-004 requires the list ordered by name; `departments/repository.py` ordered by `updated_at desc` instead — spec-vs-code drift, a real defect. Corrected on `dev/tech-debt-batch1` (PR #199), merged together with this Phase 2b's `COUNT(*) OVER()` windowed-count optimization on the same query — see §5 above. `organizations`' equivalent failure had no ordering AC in its spec, so that one was a genuinely stale test — fixed alongside (test assertion only, repository untouched).
    - Follow-up (N5, not blocking): `departments/repository.py` and `positions/repository.py` each still issue their own extra `set_config` round trip on top of `set_rls_context` — same finding family as P7/P6, not yet scheduled.
  - ⏸️ P8/P9 — index gaps + unrefreshed materialized views. Explicitly pre-production risk per auditor, not a current bottleneck — deferred to pre-go-live.
- ⏸️ Phase 2c — load-testing harness + 200-250 concurrent-user capacity validation. Auditor's own view: numbers would be meaningless until P0/P2 land (now done) — still nothing built, tool choice (Locust/k6/custom) an open decision for the user.
- ✅ Watch-item resolved 2026-07-28 (see P4 above): `_pipeline_progress_sql.py`'s event CTEs now join only against the current page's positions, not the full matched-position set.
- ❓ Watch-item (new, pipeline-progress-all-levels, 2026-07-31/08-02): the single-level fix above does NOT extend to `_pipeline_progress_all_levels_sql.py` — its event CTEs still compute over the FULL matched-position set before pagination (1 LATERAL CTE for on_hold becomes up to 5 for the 4 new cross-cutting measures, and the row grain multiplies by each position's active-level count instead of one row per position). Partially mitigated by measure-scoped SQL generation (`build_all_levels_rows_sql`, principal-reviewer finding 9) which only builds CTEs for requested measures, but the full-matched-set-before-pagination shape itself is unchanged. Two more confirmed-via-live-EXPLAIN cost items, neither addressed by the finding-9 mitigation: (a) the 4 direct-event CTEs (scheduled/selected/rejected/pending) filter on `ash.new_status::text ~ '_selected$'`-style regex instead of an indexable enum equality; (b) the event-CTE-to-`matching_positions` join is a `Join Filter` evaluated on a computed `CASE` expression (deriving `level_key` from `pending_reason`), not an equijoin on an indexable column. Measured at dev scale: 128-151ms for all 9 measures, worst single grain row fans out to 375 join rows. Feeds into Phase 2c when it happens. The `category_rank` double-SubPlan issue (round 1 finding) is fixed — `RANK() OVER (...)` replaced the correlated `COUNT(*)+1` subquery; live EXPLAIN against real populated data (77 positions/809 levels) confirms 0 `SubPlan`s, 1 `WindowAgg`.

## 9. Feature backlog (not started / deferred)

- ✅ Multi-panelist per interview level (schema+backend+frontend) — PR #204, principal-reviewer APPROVE-WITH-NITS (5 rounds), merged 2026-08-02; see §4 tech-debt notes on the panelist-2-3 auto-assign gap + 3 deferred minors.
- ✅ Interview Pipeline Progress report — status-groups redesign (`status_group` single-select, 6
  groups, 3-date-bucket rule, mandatory org-then-group gate, new bordered/grouped chart) — PR #209,
  principal-reviewer APPROVE-WITH-NITS (3 rounds), merged 2026-08-04. Supersedes PR #206's
  9-measure/all-levels shape, which the user rejected on live testing.
- 🔴 Onboarding workflow — new module, stub only (`position_status_enum.onboarded` marker exists, nothing else).
- 🔴 Consent + DPDP module — new module, stub only.
- ✅ Notifications real fan-out — see Execution Queue item 2 / §2 pending merges.
- ✅ (superseded/obsolete, confirm-close) Positions-ageing widget on the Positions list — the full Positions Ageing *report* shipped (PR #173) on the reports dashboard instead.
- 🔴 Remaining reporting endpoints, all still stub: candidate-uploads, positions/creation, positions/jd-changes, screening, interviews/pipeline (lifetime matview), interviews/candidate-status, transition-times, selection-rates, panelists/workload, departments/candidate-status-summary.
- ⏸️ **UAT recruitment-funnel data — 500 candidates / 50 positions full scale.** Script built + verified at 120-candidate proof scale (`dev/seed-uat-recruitment-funnel`, §2). Full run explicitly deferred by user (month-end budget) — resume by restoring the 2 dropped categories (Professional Services, Product/Platform Consultants) for the full 14-category run; budget ~4x the candidates at similar per-unit cost.
- 🔴 **`reconcile_screenings` beat-scheduling precondition (async-pipeline-durability, deferred from Phase 2 by principal-reviewer round 1, C2).** The task is coded and unit-tested (`candidates/screening/tasks.py`) but deliberately NOT registered in `celery_app.py`'s `beat_schedule` — live-measured against the dev DB: 181,820 unscreened pairs, 94% belonging to candidates with no `ai_assisted_evaluation` consent (a permanent skip, not a stuck row), which would create an unbounded LLM-enqueue loop (144,000/day) on the same `screening` queue interview-kit generation now also depends on, if scheduled at any tight cadence today. Needs, before scheduling: (1) a consent filter excluding permanently-ineligible pairs from the sweep entirely, (2) a staleness filter (don't re-enqueue a pair enqueued moments ago), (3) a per-pair attempt ledger + terminal state (mirroring the extraction/matching/kit reconcilers' `retry_count`/terminal-fail contract, which this task currently has no equivalent of at all). Tracked here per `openspec/specs/candidates/spec.md` §9.5 and `candidates/screening/tasks.py`'s own module docstring, both of which point at this entry.
- 🔴 **DPDP retention enforcement + reporting materialized-view refresh — placeholder task bodies, no owner (async-pipeline-durability, deferred from Phase 2 by principal-reviewer round 1, C3).** Both beat-schedule-worthy jobs were REMOVED from `celery_app.py`'s `beat_schedule` this phase (not added, as an earlier draft of this change incorrectly claimed) because their task bodies are still placeholders: `data_privacy.enforce_data_retention` (`app/modules/data_privacy/tasks.py`) is an empty no-op, `reporting.refresh_reporting_views` (`app/modules/reporting/tasks.py`) raises `NotImplementedError`. The DPDP one is the higher-priority gap — `proposal.md` itself calls it "a live compliance exposure" (specced as a daily storage-limitation sweep, nothing currently anonymises anything). Needs: implement the real task body for each (querying `candidates.retention_expires_at` for the DPDP one; refreshing whichever materialized views `reporting/spec.md` names for the other), THEN add both back to `beat_schedule` (`enforce-dpdp-retention` daily, `refresh-reporting-materialized-views` every 10 min — cadences already decided, just needs real bodies to schedule against).
- 🔴 **`level_kit_agent.py`'s `_invoke_anthropic`/`_invoke_bedrock` bypass `llm_gateway` entirely
  (async-pipeline-durability, flagged Phase 4 by principal-reviewer round 1, Major-6).** Both
  build their own raw, unbounded, unconfigured clients (`anthropic.Anthropic(...)`,
  `boto3.client("bedrock-runtime", ...)` with no `Config`/timeout) — this is the LEAST-bounded
  LLM path in the system, on the exact Celery queue (`interviews`/kit generation) Phases 2/3
  explicitly hardened for retry/redelivery safety. Out of D8's scope to fix (D8 only bounds
  `llm_gateway`'s own Anthropic/Bedrock/Gemini clients) — needs its own change to either route
  through `llm_gateway.complete()` (the existing architectural-inconsistency note in
  `interviews/spec.md`'s changelog already flags this agent as not using the shared gateway) or
  get its own module-scope timeout-bound clients mirroring `llm_gateway_providers.py`'s pattern.
- 🔴 **D8's circuit breaker is per-process, not cross-process (async-pipeline-durability, flagged
  Phase 4 by principal-reviewer round 1, Major-2).** `docker-compose.yml`'s worker runs prefork
  with no `--concurrency` set (CPU-count child processes), each holding its own in-memory
  `_breaker_state` dict — a single hung task's Celery-level retries can be redelivered to a
  different child process each attempt, so one task's retry chain may never accumulate to
  `LLM_CIRCUIT_BREAKER_THRESHOLD` in any single process. A cross-process (Redis-backed) breaker
  would close this gap; correctly documented as a known limitation in `core/constants.py` and
  `llm_gateway.py` rather than fixed, since building one is a bigger scope decision than D8's.
- 🔴 **`LLM_PROVIDER_TIMEOUT_SECONDS=60.0` is a judgment call, not a measured figure
  (async-pipeline-durability, flagged Phase 4 by principal-reviewer round 1, Major-4).** No
  measured Anthropic/Bedrock completion-latency figure exists anywhere in this repo (the only
  `timeout=30` usages found were 8 unrelated functional-test files hitting localhost over
  httpx). 60s is a reasoned middle ground (well under Anthropic's own 600s SDK default, with
  headroom above a normal `max_tokens=4096` completion) but should be revisited with a real
  measured p95/p99 completion latency once Anthropic/Bedrock actually go live in production,
  before relying on this bound under real load.
- ✅ **RESOLVED (async-pipeline-durability Phase 6, principal-reliability-engineer AWS SRE
  review round 1, C5).** This entry originally characterized the worker-side metrics gap as
  "single-process best-effort... an under-count (never fabricated)" — the reviewer correctly
  identified that characterization as ITSELF wrong: with the original per-child
  `worker_process_init`/first-bind-wins design, the metric WRITER (whichever prefork child a
  reconciler task landed on) and the metric SERVER (whichever child won the port race) could be
  DIFFERENT children, producing a STALE gauge value between the rare cycles that happened to land
  on the binding child, or an ABSENT series entirely for a pipeline that never did — worse than an
  under-count, and it directly contradicted `metrics.py`'s own "never left stale" claim. Fixed by
  switching to `prometheus_client`'s real multiprocess mode: `PROMETHEUS_MULTIPROC_DIR` (set on the
  worker container only, `docker-compose.yml`/`ecs` module) makes every Counter/Gauge write to a
  per-PID file instead of process memory; `start_worker_metrics_server` (`shared/metrics.py`) now
  serves the aggregated result via `multiprocess.MultiProcessCollector`, called ONCE from
  `celery_app.py`'s `worker_init` signal (fires in the pre-fork parent, not per child — no port
  race to guard against); `worker_process_shutdown` marks each exiting child's file dead. The API
  process's own `/metrics` (main.py) still never reflects these values in production (separate ECS
  task from the worker) — that half of the original finding stands and is now documented directly
  in `main.py`'s mount comment instead of here.
  **Round 2 correction (R2-C1)**: round 1's live check above exercised only a Counter, which
  aggregates correctly under EITHER `multiprocess_mode` — it could not have caught (and did not
  catch) that the Gauge (`pipeline_oldest_pending_age_seconds`) was left on the DEFAULT
  `multiprocess_mode='all'`, which exports one series per PID and is never cleaned up by
  `mark_process_dead` (that only cleans the LIVE modes). Reviewer proved live: one process wrote
  250 then exited, another wrote 0 then exited, exposition showed BOTH values — a `Maximum`
  statistic downstream would read 250 forever after the backlog clears. Fixed by setting
  `multiprocess_mode="livemostrecent"` explicitly; a REAL committed test now exists
  (`backend/app/shared/tests/test_metrics_multiprocess.py`) exercising the Gauge specifically
  (not just the Counter), verified to fail against the buggy default and pass against the fix.
- 🔴 **Worker ECS service pinned to `desired_count=1` — scaling it requires a metrics-shape fix
  first (async-pipeline-durability Phase 6, AWS SRE review round 2, R2-M2).** `multiprocess_mode`
  (see the entry above) is per-CONTAINER, not per-cluster — 2+ worker tasks each carry independent,
  independently-resetting Counter/Gauge state that lands on the SAME CloudWatch dimension set
  (`pipeline` only). `infrastructure/terraform/modules/ecs/variables.tf`'s `worker_desired_count`
  carries a Terraform `validation` block hard-failing `plan`/`apply` above 1, specifically so this
  can't be silently reintroduced by a routine capacity bump. Needs, before ever scaling: either a
  `TaskId` dimension combined with an alarm shape that SUMs each task's own per-task RATE (NOT a
  shared cumulative-value SUM — that wouldn't fix resets and would double-count during overlap;
  each task's own series is monotonic within its lifetime, so its disappearance just ends that
  series instead of resetting a shared one), or an equivalent per-task-aware aggregation fix.
  **Round 3/4 correction (R3-M2, wording fixed round 4): the risk is task-replacement counter
  reset, NOT deploy overlap.** Every worker task replacement (deploy, scale-in, unhealthy-task
  swap) resets that task's Counter to 0 on death, regardless of `desired_count`. ECS's deployment
  overlap (`maximumPercent=200%`/`minimumHealthyPercent=100%` — a replacement starts BEFORE the old
  task drains) is a red herring here: all 3 `pipeline_*` alarms use `Maximum`, so the overlap
  window itself is a no-op (`max(old=50, new_task=0) == 50`) — applying beat's stop-then-start fix
  (`deployment_maximum_percent=100`/`deployment_minimum_healthy_percent=0`) here would NOT actually
  help, which is why it was deliberately not applied; the worker's 120s `stopTimeout` would also
  make that fix cost 2-3 minutes with zero Celery consumers on all 6 queues on every deploy, a
  separate and sufficient reason not to. The real risk is narrower than "a noisy RATE dip": a real
  terminal-failure burst recorded just before a task replacement can be MASKED (silently dropped
  from the alarm's view, not merely under-reported) by the reset. The gauge-based
  `pipeline_oldest_pending_age_seconds` alarm is unaffected (not a Counter; recomputed fresh from
  the DB every sweep regardless of which task runs it) and remains the primary detector for the
  incident class this phase exists to catch. Documented consistently in the variable description
  and the runbook.
- 🔴 **Queue-depth / oldest-message-age alarms scoped out of the monitoring module (async-
  pipeline-durability Phase 6, D10; principal-reviewer final-gate M2 — this entry was
  missing, the module's own comment pointed here to nothing).** `infrastructure/terraform/
  modules/monitoring/main.tf`'s header explains the reasoning but the entry it promised never
  got written: Redis `LLEN` is not a native CloudWatch metric, and no existing Celery-side
  gauge exists to extend (checked `celery_app.py`) — building one would mean a guessed custom
  exporter, out of scope for this phase. The 3 `pipeline_*` alarms (stuck-rows volume,
  terminal-failures, oldest-pending-age) are the primary signal instead, per design.md's own
  explicit reasoning ("the actual incident had an EMPTY queue with a FULL table of stranded
  rows; a queue-depth-only alarm would not have caught it"). If queue-depth/oldest-message-age
  visibility is ever wanted anyway (e.g. for capacity planning, not incident detection), it
  needs a Celery-side gauge (there isn't one) publishing via the same `shared/metrics.py`
  worker-listener path already built.
- 🔴 **Age-gauge reconciler queries have no dedicated "entered non-terminal state" timestamp
  column (async-pipeline-durability Phase 6, C4; principal-reviewer final-gate M2 — this entry
  was missing, `_reconciler_tasks.py`'s own docstring pointed here to nothing).**
  `candidates/_reconciler_tasks.py`/`interviews/_reconciler_tasks.py`'s
  `pipeline_oldest_pending_age_seconds` gauge estimates age as `(now() - updated_at) +
  retry_count * STUCK_ROW_SLA_SECONDS` rather than reading a real elapsed-time column, because
  no such column exists — `updated_at` gets bumped by the same `fn_set_updated_at_and_version`
  trigger on every reconciler claim UPDATE, so a bare measure resets every ~60s (the actual C4
  defect this estimate works around). The estimate assumes beat's actual schedule matches
  `STUCK_ROW_SLA_SECONDS` (true today, both hardcoded to 60s, but not enforced to stay in sync).
  A real fix — a dedicated timestamp column set once on first entering the non-terminal state,
  never touched by the reconciler's own claim UPDATE — needs an additive Alembic migration on
  `candidates`/`interview_level_kits` (per `docs/SCHEMA_EVOLUTION.md`'s decision tree, a real
  column: hot-queried, indexed) — not built this phase, tracked here instead of guessed.
  **Related, logged together (m8, same final-gate round):** all 3 age-gauge queries aggregate
  over ALL non-terminal rows every 60s (unbounded), while the row-scan they precede is capped
  at `LIMIT 100` — cost scales with backlog size, in exactly the incident scenario the metric
  exists to detect. Currently ~1-6ms locally (live `EXPLAIN`-verified, single index scan /
  Hash Semi Join, no duplicate SubPlan) and off the request path (Celery beat, not an API
  path) — not a defect today, just a scaling assumption worth re-measuring if the non-terminal
  backlog ever grows by orders of magnitude.
- 🔴 **Two pre-go-live Terraform verification items needing real registry/AWS access, neither
  possible in the authoring sandbox (principal-reviewer final gate, m1 + m4).** (a) `elasticache`
  module's Redis security group allows ingress from the whole VPC CIDR rather than being scoped
  to the `ecs` module's task security group specifically — the `ecs` module now exists in the same
  root plan and `elasticache/outputs.tf` already exports `security_group_id`, so tightening this is
  possible in principle, but wiring it would make `elasticache` take `ecs`'s task SG as an input
  while `ecs` already takes `elasticache`'s endpoint as an input, and confirming that doesn't
  produce a real apply-time cycle needs a `terraform graph`/`plan` run, not available here. (b)
  `ecs` module's `cloudwatch_agent_image` variable stays at `public.ecr.aws/cloudwatch-agent/
  cloudwatch-agent:latest` — a floating tag risks a silent upstream schema break (C1/C2's failure
  mode arriving a different way) but the exact versioned ECR tag could not be verified live
  (`gallery.ecr.aws/cloudwatch-agent/cloudwatch-agent` is JS-rendered, returned empty content via
  every fetch attempted — the same JS-rendering limitation the AWS SRE review hit twice already
  with other AWS docs). Both need someone with real AWS/registry access before go-live, not a
  guess shipped as if verified.
- 🔴 **Project-wide OpenSpec format migration.** All existing `openspec/specs/<module>/spec.md` files (candidates, interviews, reporting, positions, offers, data-privacy, etc.) use this project's own house format — numbered sections + BR-xxx business rules — not the OpenSpec-tool-native `### Requirement:`/`#### Scenario:` format. Surfaced 2026-08-05 when writing delta specs for `async-pipeline-durability`: no existing requirement headers to copy for a proper MODIFIED block, so those deltas used ADDED throughout. User confirmed (2026-08-05): keep house format for now, don't block the reliability change on this, but track a full project-wide reformat to make every spec file uniformly OpenSpec-native (`### Requirement:` + `#### Scenario:`, one file per module, no numbered-section/BR-xxx house style) as its own future change. Large — one file alone (`candidates/spec.md`) is 1700+ lines.

---

## How items move through this doc

New item found → add to the relevant section with 🔴. Work starts → 🟡. Merged/shipped → ✅ with
the PR/commit reference, then drop to a one-line mention or remove if no longer useful context.
Explicitly deferred by user choice → ⏸️ with the reason. Never let an item sit stale without a
status matching its actual state — if unsure whether something is still open, mark ❓ and verify
before the next time it's relevant, don't silently trust an old entry.
