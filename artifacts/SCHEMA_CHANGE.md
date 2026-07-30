**Artifact Version 2.0 — Baselined 08-Jun-2026**  ·  Source requirements: ATS_requirement_v2_0_08-Jun-2026.docx  ·  Versioning rule: docs/VERSIONING.md

---

# Database Schema Changelog

Structured, append-only record of **every** change to the PostgreSQL schema —
tables, columns, enums, indexes, constraints, triggers, RLS policies, partitions,
and materialized views.

**This file is mandatory.** Per `.claude/CLAUDE.md` → "Database schema evolution",
any spec/code/feature/module change that touches the database MUST: (1) consult
`docs/SCHEMA_EVOLUTION.md` first, (2) extend without breaking existing
functionality (additive, expand→contract), and (3) add an entry here **in the
same commit** as the migration. The code-reviewer subagent and CI block changes
that skip this.

Newest entries on top. Do not edit or delete past entries — schema history is
append-only (corrections are added as new entries that reference the prior one).

---

## Entry template (copy for each change)

```
### [YYYY-MM-DD] <short title> — <Alembic revision id | "baseline">
- Baseline        : v<major.minor> (<date>)
- Author          : <name / agent>
- Trigger         : spec change | code change | new feature | new module  (link: <spec/PR/issue>)
- Module(s)       : <affected module(s)>
- Change type     : add table | add column | add enum value | add index |
                    add constraint | add trigger | add RLS policy | add partition |
                    add materialized view | other (describe)
- Objects         : <table.column / enum / index names affected>
- Storage decision: <mechanism chosen per SCHEMA_EVOLUTION.md decision tree —
                    metadata JSONB | lookup_values | custom_field_definitions |
                    tags | REAL COLUMN/TABLE>. If a real column/table was used,
                    state WHY (hot-queried / indexed / relational).
- Backward compat : <how existing rows & running code stay unaffected;
                    NULLable/default? expand→contract phase? data backfilled?>
- Migration       : <Alembic revision id>; downgrade implemented? <yes/no>
- Validation      : <tested upgrade + downgrade against existing data? how?>
- Rollback        : <how to revert safely>
- Notes           : <anything future readers need>
```

---

## Changelog

### [2026-07-21] Drop ck_interview_skip_reason (stale table-local CHECK) — 0052_drop_ivw_skip_reason_ck

- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (interview-status-lifecycle-phaseB branch)
- Trigger         : functional-test-found bug (`test_functional_p42_org_skip_gate.py`
                    → `test_sc1_stg_skip_without_reason_succeeds`) — `POST
                    /interviews/{id}/skip` returned 500 `INTERNAL_ERROR`
                    (unhandled `IntegrityError`) when `skip_reason` was omitted
                    for an `stg_labs`-category level, even though Phase B's
                    `design.md` D1 correctly made `skip_reason` optional for
                    `stg_labs` at the service layer
                    (`interviews/service.py::_validate_skip_reason()`, already
                    unit- and functional-test-confirmed correct). Root cause:
                    the DB-level CHECK added in `0042_app_ivw_status_expansion.py`
                    was never relaxed to match — it unconditionally required
                    `skip_reason IS NOT NULL` whenever `is_skipped = TRUE`, for
                    every level category. Spec:
                    `openspec/changes/interview-status-lifecycle-phaseb/design.md`
                    (Migration Plan section).
- Module(s)       : interviews
- Change type     : drop constraint (CHECK)
- Objects         : `interviews.ck_interview_skip_reason`
- Storage decision: N/A — this removes a constraint, no new data storage
                    mechanism involved. A table-local Postgres CHECK cannot
                    reference `interview_levels.level_category` (a different
                    table), so the category-conditional rule (skip_reason
                    optional for `stg_labs`, mandatory for `organization`)
                    cannot be expressed as a CHECK without denormalizing
                    `level_category` onto `interviews` or adding a trigger —
                    both rejected as unnecessary complexity given the
                    service-layer check (`_validate_skip_reason()`) is already
                    the correct, sufficient, and fully tested enforcement
                    point. Decision: drop the DB constraint; app layer is the
                    single source of truth for this rule.
- Backward compat : Purely additive-safe in the "removes a restriction" sense
                    — no existing rows affected, no data written or backfilled.
                    Backfill mandate (CLAUDE.md item 5): N/A — this migration
                    does not add a column or persist a new value derived from
                    another source; it only removes a CHECK constraint, so
                    there is nothing to backfill.
- Migration       : 0052_drop_ivw_skip_reason_ck; downgrade implemented? yes
                    (`op.create_check_constraint` recreates the exact original
                    condition `is_skipped = FALSE OR skip_reason IS NOT NULL`).
- Validation      : Applied `alembic upgrade head` locally, confirmed
                    `alembic current` shows `0052_drop_ivw_skip_reason_ck
                    (head)`. Re-ran
                    `app/modules/interviews/tests/test_functional_p42_org_skip_gate.py`
                    against the real DB post-migration — previously-failing
                    `test_sc1_stg_skip_without_reason_succeeds` now passes
                    (200, no IntegrityError); the org-gate scenarios that were
                    already passing remain green. Full
                    `app/modules/interviews/tests/` suite re-run for
                    regressions.
- Rollback        : `alembic downgrade -1` recreates
                    `ck_interview_skip_reason` with its original condition on
                    `interviews`. Safe as long as no `stg_labs` skip without a
                    reason was persisted between upgrade and downgrade (would
                    violate the recreated constraint on downgrade) — none
                    expected outside this dev/test window.
- Notes           : This is a DB-only fix; no application code changed in this
                    revision (the service-layer rule was already correct this
                    session). Documents why a plain conditional CHECK could
                    not be used: CHECK constraints cannot reference columns on
                    a different table.

### [2026-07-21] Partial unique index for offer_accepted/onboarded active-hire states — 0051_offer_accepted_hire_idx

- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (applications-status-lifecycle-phasea branch)
- Trigger         : code change (design.md D3) — `do_accept()`
                    (`app/modules/offers/_service_transitions.py`) now writes
                    `offer_accepted` to `applications.status` instead of the legacy
                    `hired` value. Spec: `openspec/changes/applications-status-lifecycle-
                    phasea/design.md` D3; `openspec/changes/applications-status-lifecycle-
                    phasea/specs/{applications,offers}/spec.md`.
- Module(s)       : applications, offers
- Change type     : add index
- Objects         : new partial unique index
                    `uq_applications_candidate_offer_accepted_or_onboarded` on
                    `applications(candidate_id) WHERE status IN ('offer_accepted', 'onboarded')`.
                    Existing `uq_applications_candidate_hired` (migration 0031) is UNCHANGED —
                    not dropped, not modified.
- Storage decision: REAL INDEX (extends an already-real, hot-queried uniqueness constraint on
                    an existing column — `applications.status`, already a real column per
                    migration 0042). No new column or table; this is a second partial index
                    on the existing enum column, following the exact pattern of 0031.
- Backward compat : Fully additive — new index only, no column/enum change, no existing index
                    dropped. `hired` remains a valid, queryable historical value; its own
                    index (`uq_applications_candidate_hired`) is untouched and still active
                    for defense-in-depth on any pre-existing 'hired' row.
- Backfill        : NOT performed (per CLAUDE.md backfill mandate, explicitly considered).
                    Existing `hired` rows are historically correct as recorded ("this candidate
                    WAS hired" at that time) and are intentionally NOT rewritten to
                    `offer_accepted` — `hired` stays a valid, backward-compat-only value; only
                    the go-forward write path changes. This migration adds a NEW index over a
                    NEW pair of status values it does not retroactively apply to historical
                    `hired` rows, so no pre-existing row's meaning becomes ambiguous (unlike the
                    incident the mandate codifies) — every `offer_accepted`/`onboarded` row that
                    will ever exist under the new index is written by code that already exists
                    under this same change.
- Migration       : `0051_offer_accepted_hire_idx`; downgrade implemented (drops only the new
                    index; `uq_applications_candidate_hired` is left in place either way).
- Validation      : `alembic upgrade head` / `alembic downgrade -1` run locally against the dev
                    DB; defensive pre-flight duplicate-resolution query (mirrors 0031's pattern)
                    included in `upgrade()` in case dev/test data already has a conflict.
- Rollback        : `alembic downgrade 0050_app_dropped_declined` drops the new index only; the
                    application-code revert (do_accept() reverting to write 'hired') is a plain
                    code revert, independent of this migration.
- Notes           : `has_hired_application_for_candidate()` (`applications/repository.py`) is
                    extended in the same PR to query `status IN ('offer_accepted', 'onboarded')`
                    instead of `status = 'hired'` — this index is the DB-level backstop for that
                    same service-layer guard (BR-012/BR-014).

### [2026-07-21] Restore accidentally-deactivated interview_levels (2 rows, position 6ed3b751) — no new migration

- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (data-correction, reports-interview-pipeline-progress branch)
- Trigger         : data-correction following an in-session incident: this position's Organization
                    Level 1/2 `interview_levels` rows were accidentally deactivated earlier in the
                    same session (root-caused and explained to the user). These 2 rows also carried
                    the SAME migration-0042 `level_type`/`level_category` desync described in the
                    entry immediately below this one (`level_type='stg_labs'` while
                    `level_category` was already the correct `'organization'`), and each had a
                    second historical duplicate copy from an earlier save (created 2026-06-18
                    09:26:05 vs. the restored copy's 09:47:05).
- Module(s)       : positions (interview_levels)
- Change type     : data correction (no schema change — existing columns, value fix only)
- Objects         : `interview_levels` rows `c9bc0fc0-76b6-4846-80b5-009a28881113` (L1) and
                    `d8fe11a0-0d23-436d-822f-872b30360eb2` (L2), position
                    `6ed3b751-c1fe-4bfa-b747-e5c0b7317b69` ("Engineering Manager - Platform", org
                    AI COE). Set `is_active=TRUE`, `level_type='organization'` (correcting the
                    desync to match already-correct `level_category='organization'`), and
                    `level_label` to `'AI COE Level 1'` / `'AI COE Level 2'` to match this
                    position's existing "AI COE Level N" naming convention (levels 3-6 already use
                    it). The older 09:26:05 duplicate copies (`a7a477b2-...`, `ffff1cc3-...`) and
                    the 6 already-correct rows (today's STG L1/L2, AI COE L3-6) are explicitly OUT
                    OF SCOPE and were confirmed untouched after the script ran.
- Storage decision: n/a — existing columns, no new storage mechanism introduced.
- Backward compat : no schema change. Reactivating these rows as `level_type='organization'` does
                    not collide with the position's active `level_type='stg_labs'` L1/L2 rows
                    under the partial unique index `uq_int_level_pos_type_num ON
                    interview_levels(position_id, level_type, level_number) WHERE is_active=TRUE`,
                    since `(position_id, 'organization', 1/2)` is a distinct tuple from
                    `(position_id, 'stg_labs', 1/2)`. `category_rank` (computed per
                    `level_category` via `_CATEGORY_RANK_SUBQUERY` in
                    `app/modules/interviews/repository.py`) confirmed correct post-restore: STG
                    ranks 1,2 and Organization ranks 1-6 contiguous, no gaps.
- Migration       : none. No Alembic revision — this is a data-value correction (reactivation +
                    field fix) on existing columns, not a schema change.
- Validation      : ran `backend/app/scripts/backfill_restore_ai_coe_engmgr_levels.py`
                    (ENVIRONMENT=local-gated, idempotent via `WHERE is_active = FALSE` guard,
                    single transaction, rollback-on-exception) against the local dev DB; re-ran a
                    second time and confirmed no-op (0 rows updated); verified all 8 active rows
                    for the position (STG L1/L2 + AI COE L1-6) and their `category_rank` values;
                    verified the 2 older duplicate rows remain untouched at `is_active=FALSE`,
                    `level_type='stg_labs'`.
- Rollback        : re-run the script's UPDATE with `is_active=FALSE` and `level_type='stg_labs'`
                    for the same 2 row ids, if ever needed; no expectation of this being required.
- Notes           : script path `backend/app/scripts/backfill_restore_ai_coe_engmgr_levels.py`,
                    same pattern as the sibling script referenced in the entry below. No linked
                    spec/PR yet — change lives on branch `dev/reports-interview-pipeline-progress`.

### [2026-07-21] interview_levels.level_type data correction (2 active rows) — no new migration

- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (data-correction, reports-interview-pipeline-progress branch)
- Trigger         : data-correction following a confirmed incident found during the
                    reports-interview-pipeline-progress build. Root cause: migration
                    `0042_app_ivw_status_expansion.py` (lines 128-138) backfilled the new
                    `level_category` column from the free-text `level_label` column instead of
                    the authoritative `level_type` column, for pre-existing `interview_levels`
                    rows. Rows whose label didn't contain "stg" (e.g. "Technical Round 1") got
                    `level_category='organization'` while `level_type` was left at 'stg_labs'.
                    Bug window is CLOSED: every level-creation path has set
                    `level_category=level_type` unconditionally since commit `3cfebbe5`
                    (2026-07-04; `app/modules/positions/levels_service.py:89`), and the
                    free-text label editor was replaced by a fixed-dropdown editor the same day.
                    One-time historical artifact, not a recurring bug.
- Module(s)       : positions (interview_levels)
- Change type     : data correction (no schema change — existing column, value fix only)
- Objects         : `interview_levels.level_type` — 2 specific active rows identified by
                    (position_id, level_number, is_active=TRUE):
                    `79869376-88e6-45c0-af15-a4c13e770def` L1 and
                    `f2bafdc6-96c7-4613-96dd-6a0b321f1d5f` L1 (both "Technical Round 1").
                    Corrected `level_type` from `'stg_labs'` to `'organization'` to match the
                    already-correct `level_category='organization'` (confirmed with the user:
                    both positions' standard is STG L1/L2 followed by org-name levels, so
                    "Technical Round 1" is an org-level round — `level_category` was right,
                    `level_type` was the wrong one).
- Storage decision: n/a — existing column, no new storage mechanism introduced.
- Backward compat : no schema change; running code is unaffected since `level_category` (the
                    column actually read by category-based logic) was already correct. 4 other
                    historically-desynced rows on a separate position
                    (`6ed3b751-c1fe-4bfa-b747-e5c0b7317b69`), all `is_active=FALSE`, are
                    explicitly OUT OF SCOPE — a separate, not-yet-authorized decision — and were
                    confirmed untouched after this script ran.
- Migration       : none. No Alembic revision — this is a data-value correction on an existing
                    column, not a schema change; per the schema-evolution rule, the historical
                    migration `0042_app_ivw_status_expansion.py` is not edited after the fact.
- Validation      : ran `backend/app/scripts/backfill_level_type_org_correction.py`
                    (ENVIRONMENT=local-gated, idempotent via `WHERE level_type != 'organization'`,
                    single transaction, rollback-on-exception) against the local dev DB; verified
                    both target rows now show `level_type='organization'` matching
                    `level_category='organization'`; verified the 4 parked rows on the other
                    position remain untouched at `level_type='stg_labs'`.
- Rollback        : re-run the script's UPDATE with `'stg_labs'` in place of `'organization'`
                    for the same 2 row ids, if ever needed; no expectation of this being required.
- Notes           : script path `backend/app/scripts/backfill_level_type_org_correction.py`. No
                    linked spec/PR yet — change lives on branch
                    `dev/reports-interview-pipeline-progress`.

### [2026-07-15] Dropped/Offer Declined statuses + offer-decline reason enum — 0050_app_dropped_declined

- Baseline        : v2.3 (04-Jul-2026)
- Author          : backend-engineer (database-migration skill)
- Trigger         : bug fix / capability removal — `applications-manual-status-lockdown`
                    change, triggered by a data-integrity incident (an application showed
                    a decided interview-level outcome with zero interview records ever
                    created, via the manual status-override path, BR-020/P35-D). This
                    migration adds 2 new, legitimate manual statuses (Dropped, Offer
                    Declined) as part of narrowing the manual status-change surface.
                    Branch: dev/applications-manual-status-lockdown.
- Module(s)       : applications
- Change type     : add enum value (x6 across 2 existing enums), add enum type, add column
- Objects         : application_status_enum (+`dropped`, +`offer_declined`);
                    application_pending_reason_enum (+`awaiting_candidate_availability`,
                    +`awaiting_recruiter_followup`, +`awaiting_internal_approval`, +`other`);
                    application_offer_declined_reason_enum (new type, 6 values);
                    applications.offer_declined_reason (new column)
- Storage decision: REAL COLUMN (application_offer_declined_reason_enum) — follows the
                    SAME pattern this table already uses for `pending_reason`: a fixed,
                    reportable set of values tied to a specific status. `docs/SCHEMA_EVOLUTION.md`
                    is currently a stub with no populated decision tree, so this decision
                    is grounded in this table's own existing precedent rather than that
                    doc. `dropped`'s reason reuses the existing `rejection_reason` TEXT
                    column — no new column needed for that one.
- Backward compat : All additive. New enum values via `ADD VALUE IF NOT EXISTS` (no
                    existing row could have held them before this migration — no
                    backfill needed or possible). New column NULLable, no default
                    required, existing rows unaffected.
- Migration       : 0050_app_dropped_declined; downgrade implemented: yes (drops the new
                    column + the new enum type; the enum VALUES added to
                    application_status_enum/application_pending_reason_enum are NOT
                    removed on downgrade — Postgres does not support removing enum
                    values, same accepted limitation as migration 0042).
- Validation      : `alembic upgrade --sql 0049_interview_redo_supersede:0050_app_dropped_declined`
                    and `alembic downgrade --sql 0050_app_dropped_declined:0049_interview_redo_supersede`
                    both generated clean, syntactically valid SQL (reviewed manually,
                    matches the established `op.execute("COMMIT")` + `ADD VALUE IF NOT
                    EXISTS` pattern from migration 0042). Applied to the live local dev DB
                    (`alembic upgrade head`) and validated — `alembic current` shows
                    `0050_app_dropped_declined (head)`, existing rows unaffected.
- Rollback        : `alembic downgrade 0049_interview_redo_supersede` — safe, drops only
                    the new column/type; the 6 new enum values remain defined but unused
                    by any downgraded code path.
- Notes           : This migration only adds the storage for Dropped/Offer Declined —
                    the actual BR-020 removal (hard-blocking manual level-outcome PATCHes)
                    is a service-layer code change with no schema footprint of its own,
                    tracked separately in the same `applications-manual-status-lockdown`
                    change's tasks.md section 2.

### [2026-07-14] Redo Interview Level: interviews.superseded_at + partial unique index — 0049_interview_redo_supersede

- Baseline        : v2.3 (04-Jul-2026)
- Author          : backend-engineer
- Trigger         : new feature — "Redo Interview Level" (recruiter/hr_admin redoes an
                    already-DECIDED selected/rejected level with a different panelist when
                    the panelist pairing is contested; blocks advancing to the next level
                    until the redo is also decided). Code change on branch
                    fix/interview-status-sync-and-terminal-state.
- Module(s)       : interviews
- Change type     : add column, add index (drop full unique index, replace with partial
                    unique index)
- Objects         : interviews.superseded_at ; index uq_interview_app_level (dropped) ;
                    index uq_interview_app_level_active (added, partial)
- Storage decision: REAL COLUMN + partial unique index (per SCHEMA_EVOLUTION.md decision
                    tree) — not metadata JSONB, because this value gates row-level
                    uniqueness (at most one ACTIVE interview per application+level) via a
                    partial unique index, which JSONB cannot express or enforce at the DB
                    level. This is relational, indexed, uniqueness-critical data, squarely
                    outside the JSONB/lookup_values/custom_field_definitions/tags ladder.
- Backward compat : `superseded_at` is nullable, defaults to NULL for every existing row.
                    Backfill mandate explicitly considered (not silent): no backfill is
                    needed or possible, because every existing interview row is, by
                    construction, the ONLY row for its (application_id,
                    interview_level_id) pair — the full unique index
                    `uq_interview_app_level` (in place since the interviews table was
                    created) enforced that for every row created before this migration.
                    There is no authoritative source recording "this row was actually
                    superseded" to derive from — until this migration, and the redo
                    feature it supports, no row could ever have BEEN superseded. NULL is
                    therefore unambiguously correct for 100% of existing rows, not merely
                    "structurally true with no rows affected". The replacement partial
                    unique index (`WHERE superseded_at IS NULL`) preserves the original
                    guarantee (one row per application+level) for all pre-existing,
                    non-superseded rows — behavior is unchanged until the first redo
                    creates a second (superseded, active) pair.
- Migration       : 0049_interview_redo_supersede; downgrade implemented — drops the
                    partial index, recreates the original full unique index, drops the
                    column. Downgrade WILL FAIL with a unique-violation if any level has
                    2+ rows at that point (i.e. a redo has actually happened); this is an
                    accepted, documented constraint (same pattern as other schema-shape-
                    change migrations in this codebase), not worked around with data-
                    destructive downgrade logic.
- Validation      : applied to local dev DB (`alembic upgrade head`, confirmed
                    `alembic current` = 0049_interview_redo_supersede (head)); downgrade
                    path exercised structurally (`alembic downgrade -1` then `alembic
                    upgrade head` again) against the current dev DB state, which has no
                    redos yet, so the downgrade succeeded — expected, since the documented
                    failure constraint only applies to a FUTURE state after a redo has
                    happened.
- Rollback        : `alembic downgrade -1` (safe only while no redo has occurred yet, or
                    after manually collapsing any superseded pairs — not automated here).
- Notes           : list_by_application / `_LIST_SQL` in interviews/repository.py now
                    filters `AND i.superseded_at IS NULL` so per-application interview
                    lists only ever show the active row per level; `get_detail` /
                    `_DETAIL_SQL` (explicit interview_id lookup) is intentionally left
                    unfiltered so a superseded row remains directly fetchable for
                    history/audit access.

### [2026-07-10] Data-correction: backfill legacy interview_feedback.outcome NULLs — no new migration

- Baseline        : v2.3 (04-Jul-2026)
- Author          : backend-engineer (data-correction pass, not a schema change)
- Trigger         : bug/gap tied to migration 0047_feedback_outcome_col — that migration added
                    `interview_feedback.outcome` as a nullable column with no backfill (correct
                    at the time, since the column didn't exist for pre-migration rows to fill).
                    Rows submitted BEFORE 0047 have outcome=NULL even though BR-SYNC-005 fired
                    correctly at submission time (application_status_history has the real event).
- Module(s)       : interviews (data only — no model/schema/migration change)
- Change type     : other (one-time data backfill script, no DDL)
- Objects         : interview_feedback.outcome (existing column from 0047; no new objects)
- Storage decision: N/A — no schema change. Reverse-derives the historical outcome from the
                    application's CURRENT status via the existing LEVEL_TO_APP_STATUS mapping
                    (applications/_service_helpers.py), and backfills ONLY when the application's
                    current status exactly matches the level+rank's mapped selected/rejected
                    status; ambiguous rows (status since overridden/reverted, or level+rank
                    unmapped) are left NULL rather than guessed.
- Backward compat : read-only reverse-mapping check + conditional UPDATE; no code path changes.
                    Idempotent (re-run backfills 0 additional rows since `outcome IS NULL` is
                    the selection predicate).
- Migration       : none — `backend/app/scripts/backfill_legacy_feedback_outcome.py`,
                    ENVIRONMENT=local gated, single transaction with rollback on exception.
- Validation      : unit tests (`app/scripts/tests/test_unit_backfill_legacy_feedback_outcome.py`)
                    cover the reverse-mapping logic against known (level_category, category_rank,
                    status) triples pulled from LEVEL_TO_APP_STATUS itself, plus the "overridden
                    since" / "unmapped level" no-guess cases. Ran against local dev DB: 3 rows
                    examined, 1 backfilled as 'selected' (confirmed: feedback
                    3b32b1fe-35d2-4ad6-9a65-e6ee97c35eba for application
                    96425fdf-4a93-4abb-9f3c-26484265d80d, Vikram Nair / Engineering Manager -
                    Backend_Test — now outcome='selected'), 0 backfilled as 'rejected', 2 left
                    NULL (same interview, stg_labs category_rank=3 — unmapped in
                    LEVEL_TO_APP_STATUS, application still at new_application — correctly left
                    unknown). Re-run confirmed idempotent: 0 additional rows backfilled.
- Rollback        : none needed (no DDL). If a backfilled value is later found wrong, correct
                    the specific interview_feedback.outcome row directly — no migration to revert.
- Notes           : this is a one-time data-correction script, not a repeatable migration; kept
                    in app/scripts/ alongside backfill_panelist_login_accounts.py for the same
                    reason (local-dev-only, hard-gated, documented for future reference).

### [2026-07-10] Add outcome_change_reason column to interview_feedback (BR-SYNC-006 guardrail) — 0048_feedback_outcome_reason

- Baseline        : v2.3 (04-Jul-2026)
- Author          : backend-engineer (feature — P36 follow-up mandatory-reason guardrail)
- Trigger         : spec change — openspec/specs/interviews/spec.md BR-SYNC-006 added: a
                    panelist's P36 feedback edit that changes `outcome` away from an
                    already-decided value (selected/rejected), including clearing it to
                    null, now requires a stated reason (mirrors applications/spec.md
                    BR-020's override_reason pattern for manual status overrides).
- Module(s)       : interviews, applications (remarks threading only, no schema change)
- Change type     : add column
- Objects         : interview_feedback.outcome_change_reason (TEXT, nullable)
- Storage decision: REAL COLUMN — same class of field as `outcome`/`recommendation` on
                    this table: low-cardinality, audit-trail text read back on GET, not
                    speculative/rarely-queried metadata, so JSONB/lookup-table
                    piggy-backing does not apply.
- Backward compat : additive, NULLable, no default (existing rows backfill to NULL,
                    meaning "no overturn reason recorded" — the correct value for rows
                    that predate this guardrail). No existing read/write path is broken.
- Migration       : 0048_feedback_outcome_reason; downgrade implemented: yes (drops the column)
- Validation      : applied via `alembic upgrade head` against local dev DB; `alembic current`
                    confirms head at 0048_feedback_outcome_reason.
- Rollback        : `alembic downgrade 0047_feedback_outcome_col` drops the column; safe since
                    it is additive and no code path requires it to exist.
- Notes           : reason is also threaded into application_status_history.remarks by
                    auto_sync_from_interview_outcome / revert_to_pending_from_interview_outcome_cleared
                    (applications/_service_helpers.py) in place of their fixed strings — no new
                    applications-module schema or error path.

### [2026-07-09] Add outcome column to interview_feedback (P36 — feedback edit/upsert) — 0047_feedback_outcome_col

- Baseline        : v2.3 (04-Jul-2026)
- Author          : backend-engineer (feature — panelist feedback edit support, PR #167)
- Trigger         : spec change — openspec/specs/interviews/spec.md BR-007 revised so
                    POST /interviews/{id}/feedback is an upsert (a panelist may edit their
                    submitted feedback) instead of hard-rejecting a resubmission with 409
                    FEEDBACK_ALREADY_SUBMITTED. `outcome` was previously used only transiently
                    to drive BR-SYNC-005 and never persisted; GET /interviews/my now needs to
                    surface the panelist's last-submitted outcome for edit prefill, which
                    requires it to be stored.
- Module(s)       : interviews
- Change type     : add column; add CHECK constraint
- Objects         : interview_feedback.outcome (VARCHAR(10), CHECK IN ('selected','rejected'))
- Storage decision: REAL COLUMN — mirrors the existing `recommendation` column on the same
                    table (same inline-CHECK pattern from 0023_interviews.py); it is a
                    low-cardinality field read back on every GET /interviews/my row (via a
                    LEFT JOIN), not speculative/rarely-queried metadata, so
                    JSONB/lookup-table piggy-backing does not apply.
- Backward compat : additive, NULLable, no default needed (existing rows backfill to NULL,
                    meaning "no outcome recorded" — indistinguishable from "not yet set",
                    which was already the pre-migration behavior for all existing rows).
                    No existing read/write path is broken; older rows without an outcome
                    behave exactly as before.
- Migration       : 0047_feedback_outcome_col; downgrade implemented: yes (drops the column)
- Validation      : applied via `alembic upgrade head`, then round-tripped `alembic downgrade -1`
                    + `alembic upgrade head` again against local dev DB to confirm both directions
                    are clean; `alembic current` confirms head.
- Rollback        : alembic downgrade 0046_interviews_read_perm
- Notes           : no `version`/OCC column exists on interview_feedback and none was added —
                    flagged to the requester rather than added unilaterally (this is a
                    single-owner self-edit path: only the submitting panelist, matched by
                    email, can update their own row, so lost-update risk from concurrent
                    multi-actor edits does not apply the way it would for a shared record).

### [2026-07-09] Add interviews:read permission, grant to interviewer + hiring_manager — 0046_interviews_read_perm

- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (bug fix — RBAC gap)
- Trigger         : bug fix — frontend nav (frontend/src/lib/navigation/nav-items.ts:55) has always
                    gated the "Interviews" link on an interviews:read permission string that never
                    existed anywhere (not in docs/schema.sql seed, no migration, not in
                    seed_dev.py._ALL_PERMISSIONS), so no role could ever satisfy it. Surfaced when a
                    panelist (role=interviewer, PR #162 panelist login accounts) hit a dead end
                    navigating to the page. GET /interviews/my already declares its own intent via
                    require_roles("interviewer", "hiring_manager") — this migration grants the
                    matching permission to both roles.
- Module(s)       : security, interviews
- Change type     : add permission seed row; add role_permissions grants (×2)
- Objects         : permissions (resource='interviews', action='read'); role_permissions grants for
                    roles interviewer and hiring_manager against that permission
- Storage decision: existing permissions / role_permissions tables (RBAC seed data) — not a new
                    table/column; a plain seed-data insert into the established RBAC mechanism.
- Backward compat : purely additive; no role previously had this permission to lose, so no existing
                    behavior changes. ON CONFLICT DO NOTHING on both inserts makes the migration
                    idempotent/re-runnable.
- Migration       : 0046_interviews_read_perm; downgrade implemented: yes (deletes the 2
                    role_permissions grants, then the permissions row)
- Validation      : applied via `alembic upgrade head` against local dev DB; verified via a
                    roles/permissions join query that both interviewer and hiring_manager now have
                    interviews:read.
- Rollback        : alembic downgrade 0045_candidature_withdrawn
- Notes           : also added "interviews:read" to seed_dev.py's `_ALL_PERMISSIONS` + the
                    interviewer/hiring_manager entries in `ROLE_PERMISSIONS`, and mirrored the
                    permission row + grants into docs/schema.sql's base seed block. Linked branch:
                    fix/interviewer-hiring-manager-interviews-read-permission.

### [2026-07-07] Rename application_status_enum 'withdrawn' → 'candidature_withdrawn' — 0045_candidature_withdrawn

- Baseline        : v2.3 (04-Jul-2026)
- Author          : Hareesh Shastry via backend-engineer
- Trigger         : spec change P34-P2 — rename 'withdrawn' to 'candidature_withdrawn' to
                    distinguish candidature-level withdrawal from offer withdrawal
                    (offers module has its own offer_status_enum.withdrawn — unchanged)
- Module(s)       : applications
- Change type     : rename enum value; drop + recreate partial index
- Objects         : application_status_enum (value renamed: 'withdrawn' → 'candidature_withdrawn');
                    index uq_app_cand_pos (WHERE clause updated to reference 'candidature_withdrawn')
- Storage decision: REAL COLUMN (existing) — applications.status is hot-queried, indexed, and part
                    of a partial unique constraint; enum rename in place; no new columns needed.
- Backward compat : ALTER TYPE … RENAME VALUE is a catalog-only operation; PostgreSQL stores enum
                    values by OID so existing rows automatically reflect the new label — no row
                    rewrite required. The partial unique index is dropped and recreated with the
                    updated WHERE clause in the same migration. offer_status_enum is untouched.
- Migration       : 0045_candidature_withdrawn; downgrade implemented? yes (reverses rename + index)
- Validation      : Tested upgrade + downgrade against dev DB; existing rows with status
                    'withdrawn' automatically read as 'candidature_withdrawn' post-migration.
- Rollback        : alembic downgrade -1 (reverts enum value name and index WHERE clause in one step)
- Notes           : Do NOT confuse with offer_status_enum 'withdrawn' — that value is in the offers
                    module and is intentionally NOT renamed. Only application_status_enum is affected.

### [2026-07-04] Add position_closed to application_status_enum — 0044_pos_closed_status

- Baseline        : v2.3 (04-Jul-2026)
- Author          : Hareesh Shastry via backend-engineer
- Trigger         : bug fix — position close did not auto-close active applications;
                    close_applications_for_position Celery task requires a new terminal
                    enum value to mark affected applications (P27 position-close bug fix)
- Module(s)       : applications, positions
- Change type     : add enum value
- Objects         : application_status_enum — new value 'position_closed'
- Storage decision: existing PGEnum extension — 'position_closed' is a typed status on
                    the applications table, used in WHERE clauses and transition guards;
                    not suitable for metadata JSONB (it must be a typed enum value so
                    the ORM, RLS filter, and status-machine checks all see it correctly).
- Backward compat : ALTER TYPE … ADD VALUE IF NOT EXISTS is additive; existing rows and
                    running code are unaffected. The ORM model's PGEnum list is extended
                    with 'position_closed'; create_type=False so no DDL is emitted by ORM.
- Migration       : 0044_pos_closed_status; downgrade is a no-op (PostgreSQL cannot
                    remove enum values — documented in migration docstring).
- Validation      : Upgrade applied against dev DB; IF NOT EXISTS guard ensures idempotency.
- Rollback        : alembic downgrade -1 is a no-op (pass). To fully revert, manually
                    UPDATE applications SET status='withdrawn' WHERE status='position_closed'
                    before removing application code referencing this value.
- Notes           : 'position_closed' is system-only (never user PATCH-able). Set exclusively
                    by the close_applications_for_position Celery task (queue: maintenance)
                    triggered from positions.service.change_status() when status → 'closed'.

### [2026-07-04] P27-C scorecard_template column on interview_level_kits — 0043_kit_scorecard_tmpl

- Baseline        : v2.3 (04-Jul-2026)
- Author          : backend-engineer (Phase P27-C)
- Trigger         : spec change — P27 addendum BR-P20-007: scorecard_template field added to kit
- Module(s)       : interviews
- Change type     : add column
- Objects         : interview_level_kits.scorecard_template (JSONB, NULL)
- Storage decision: REAL COLUMN — scorecard_template is populated per-kit row at generation time,
                    returned in the InterviewLevelKitResponse API payload, and stored alongside
                    technical_focus_areas. Not suitable for metadata_jsonb piggy-backing as it IS
                    an additional JSONB column on an existing table with its own semantic meaning.
- Backward compat : Column is NULLable — existing kit rows are unaffected (scorecard_template=NULL).
                    Running code that does not write scorecard_template continues to work unchanged.
- Migration       : 0043_kit_scorecard_tmpl; downgrade implemented (op.drop_column).
- Validation      : Upgrade applied against dev DB; downgrade tested (drop column).
- Rollback        : alembic downgrade -1 (drops column; existing rows retain other data).
- Notes           : Derived from generated technical_focus_areas at kit completion; one entry per
                    focus area (10 total) with criterion, guidance, and reference questions (BR-P20-007).

### [2026-07-04] P27-A application status expansion + interview workflow schema — 0042_app_ivw_status_expansion

- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (Phase P27-A)
- Trigger         : new feature — Phase P27 multi-level interview + application workflow (P27-A schema foundation)
- Module(s)       : applications, interviews, positions
- Change type     : add enum values (×24 to application_status_enum); add enum type (×2:
                    application_pending_reason_enum, level_category_enum); add column (×8);
                    add table (application_status_history); add index (×1); add constraint (×1);
                    data migration ('active' → 'new_application')
- Objects         : application_status_enum (24 new values);
                    NEW TYPE application_pending_reason_enum (17 values);
                    applications.pending_reason (application_pending_reason_enum, NULL),
                    applications.rejection_reason (TEXT, NULL),
                    applications.tentative_doj (DATE, NULL),
                    applications.onboarded_at (TIMESTAMPTZ, NULL);
                    NEW TABLE application_status_history (id, application_id, old_status,
                      new_status, pending_reason, rejection_reason, remarks, changed_by, changed_at);
                    NEW INDEX idx_app_status_hist_app ON application_status_history(application_id, changed_at DESC);
                    NEW TYPE level_category_enum ('stg_labs', 'organization');
                    interview_levels.level_category (level_category_enum, NOT NULL — backfilled from level_label);
                    interviews.is_skipped (BOOLEAN NOT NULL DEFAULT FALSE),
                    interviews.skip_reason (TEXT, NULL),
                    interviews.skipped_at (TIMESTAMPTZ, NULL);
                    NEW CONSTRAINT ck_interview_skip_reason CHECK(is_skipped=FALSE OR skip_reason IS NOT NULL)
- Storage decision: REAL COLUMNS/TABLE — all fields are queried by workflow state machines,
                    relational FKs (application_status_history → applications, users), indexed for
                    timeline queries, or driven by typed CHECK constraints. Not suitable for
                    metadata_ JSONB (typed enums, date columns, FK relationships required).
- Backward compat : All new columns are NULLable or have NOT NULL DEFAULT (is_skipped DEFAULT FALSE).
                    application_status_history is a new table — no existing rows affected.
                    level_category is backfilled before SET NOT NULL (expand→contract pattern).
                    'active' enum value remains in application_status_enum (cannot remove in PG);
                    all 'active' rows data-migrated to 'new_application'. Downgrade reverts data.
- Migration       : 0042_app_ivw_status_expansion; downgrade implemented.
                    LIMITATION: enum values added to application_status_enum cannot be removed on
                    downgrade (PostgreSQL limitation) — documented in migration comments.
- Validation      : Upgrade tested against dev DB; IF NOT EXISTS / IF EXISTS guards are idempotent.
                    level_category backfill uses ILIKE '%stg%' on level_label.
- Rollback        : alembic downgrade -1. Enum values persist; running code must not write them.
- Notes           : application_status_history replaces ad-hoc audit for application lifecycle.
                    level_category mirrors level_type values but is a separate typed column for
                    workflow routing logic. Interview skip columns support P27 skip-level flow.

### [2026-07-03] Position on_hold timestamps + portco_deferred rename — Phase P24 — 0041_position_onhold_timestamps
- Baseline        : v2.2 (11-Jun-2026)
- Author          : Hareesh Shastry
- Trigger         : new feature — Phase P24 on_hold timestamp capture + close_reason code rename
- Module(s)       : positions
- Change type     : add column (×2); data migration (JSONB code rename)
- Objects         : positions.on_hold_at (TIMESTAMPTZ NULL), positions.on_hold_resolved_at (TIMESTAMPTZ NULL);
                    positions.close_reason JSONB value portco_withdrew → portco_deferred
- Storage decision: REAL COLUMNS — on_hold_at and on_hold_resolved_at are queried for reporting
                    and drive the on_hold→open transition guard (app-count check); timestamps are
                    not suitable for metadata_ JSONB (hot-queried, indexed candidate for future
                    SLA reporting). Data migration renames the legacy code; no new column needed.
- Backward compat : Both columns are NULLable — existing rows (open/in_progress/on_hold/closed)
                    get NULL; running code reading close_reason handles any code value.
                    Code rename is in-place JSONB update; rows with NULL close_reason are unaffected.
- Migration       : 0041_position_onhold_timestamps; downgrade implemented (drop columns + rename back).
- Validation      : Upgrade/downgrade tested against dev DB; IF NOT EXISTS / IF EXISTS guards are safe.
- Rollback        : alembic downgrade -1
- Notes           : on_hold_at set when entering on_hold; on_hold_resolved_at set when leaving.
                    portco_deferred replaces portco_withdrew in CloseReasonCode Literal + test fixtures.

### [2026-07-02] Position close_reason JSONB column — Phase 23B — 0040_position_close_reason
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (Phase 23B — Position Status Engine)
- Trigger         : new feature — Phase 23B Position Status Engine (openspec/specs/positions/spec.md §8A)
- Module(s)       : positions
- Change type     : add column
- Objects         : positions.close_reason (JSONB, NULLable)
- Storage decision: REAL COLUMN — structure {"code": CloseReasonCode, "text": str|null} is surfaced
                    in PositionResponse and may be filtered/audited; JSONB is appropriate for a
                    typed-but-schema-light payload; not metadata_ (a dedicated, queryable field).
- Backward compat : NULLable — existing rows (open/in_progress/on_hold) get NULL, running code
                    reads close_reason as None (no change to existing behavior).
- Migration       : 0040_position_close_reason; downgrade implemented (DROP COLUMN IF EXISTS).
- Validation      : Upgrade/downgrade tested against dev DB; IF EXISTS guards are safe.
- Rollback        : alembic downgrade -1
- Notes           : Populated only on manual closes (open→closed, on_hold→closed).
                    Auto-close via Celery (BR-050) does not set close_reason.

### [2026-07-02] Candidate recruiter touchpoint — Phase 22C — 0039_recruiter_touchpoint
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (Phase 22C)
- Trigger         : new feature — STG Labs Recruiter Touchpoint (Phase 22C)
- Module(s)       : candidates, security
- Change type     : add column
- Objects         : candidates.recruiter_touchpoint_user_id (UUID FK → users.id, NULLable)
- Storage decision: REAL COLUMN — relational FK to users (needed for JOIN to derive full name,
                    auto-updated on every write, queryable for future recruiter analytics).
- Backward compat : NULLable — existing rows unaffected; no default required.
- Migration       : 0039_recruiter_touchpoint; downgrade implemented (DROP COLUMN).
- Validation      : Upgrade/downgrade tested against dev DB.
- Rollback        : alembic downgrade -1
- Notes           : Auto-populated by service on every PATCH /candidates/{id} call.

### [2026-07-01] Candidate profile fields — Phase 22A — 0038\_cand\_profile\_fields
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (Phase 22A)
- Trigger         : new feature — candidate profile enhancement (Phase 22A spec)
- Module(s)       : candidates
- Change type     : add column (×7)
- Objects         : candidates.technical_competencies, candidates.current_organization,
                    candidates.current_ctc, candidates.notice_period_days,
                    candidates.expected_ctc, candidates.offer_details,
                    candidates.is_hybrid_open
- Storage decision: REAL COLUMNS — recruiter-captured structured data queried in list/detail;
                    offer_details uses JSONB (composite sub-struct, not hot-queried individually);
                    all others are scalar columns matching SCHEMA_EVOLUTION.md §hot-queried criterion.
- Backward compat : All columns are NULL or have server_default='[]'. Existing rows unaffected.
                    Running code before migration: new columns return NULL (handled by Optional types).
- Migration       : 0038_cand_profile_fields; downgrade implemented: yes (drop_column ×7)
- Validation      : Offline upgrade + downgrade tested against dev DB.
- Rollback        : alembic downgrade 0037_restore_sd_status
- Notes           : Also fixes silent address persistence bug in tasks.py (_build_update_from_profile
                    never wrote address_enc despite address_enc column existing since baseline).

### [2026-06-30] Restore screening_decisions.status (ai_screening_decision_enum) — 0037\_restore\_sd\_status
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (Phase 21C bug fix)
- Trigger         : bug fix — migration 0036's DROP TYPE IF EXISTS screening_status_enum CASCADE
                    silently dropped screening_decisions.status (Phase 17A column, same enum name).
                    Surfaced as UndefinedColumnError in ApplicationService.get_application()
                    (_DETAIL_SQL LEFT JOINs screening_decisions and selects sd.status).
- Module(s)       : applications, screening
- Change type     : add enum (×1); add column (×1)
- Objects         : ai_screening_decision_enum (new enum, replaces old screening_status_enum for
                    screening_decisions); screening_decisions.status (restored column)
- Storage decision: REAL COLUMN — hot-queried by _DETAIL_SQL in applications/repository.py
                    (every GET /applications/{id} call); named type for constraint safety.
- Backward compat : additive; all existing screening_decisions rows receive DEFAULT 'pending'.
                    Running code unaffected — applications service already expected this column.
- Migration       : 0037_restore_sd_status; downgrade implemented: yes
                    (DROP COLUMN status; DROP TYPE ai_screening_decision_enum)
- Validation      : upgrade confirmed via information_schema.columns — status column present
                    with udt_name=ai_screening_decision_enum; GET /applications endpoint returns
                    200 after fix.
- Rollback        : alembic downgrade -1 (or alembic downgrade 0036_candidate_screenings);
                    drops status column + ai_screening_decision_enum.
- Notes           : screening/models.py updated to reference ai_screening_decision_enum
                    (create_type=False — enum created by this migration).

### [2026-06-30] Add candidate_screenings table and screening_status_enum — 0036_candidate_screenings
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (Phase 21A)
- Trigger         : new feature — Phase 21A Candidate Screening Workflow data layer (spec §11)
- Module(s)       : candidates/candidate_screenings
- Change type     : add table (×1); add enum (×1); add index (×3); add trigger (×1)
- Objects         : candidate_screenings (table), screening_status_enum (enum),
                    ix_cand_screenings_candidate_id, ix_cand_screenings_position_id,
                    ix_cand_screenings_status (indexes),
                    trg_candidate_screenings_upd (trigger → fn_set_updated_at_and_version)
- Storage decision: REAL TABLE — candidate_screenings is hot-queried (list per candidate,
                    shortlist gate check in Phase 21C), indexed on candidate_id + position_id +
                    status, and relational (FK to candidates, positions, organizations, users).
                    JSONB used for screening_questions and responses (variable-length arrays,
                    not individually queried, no separate table needed).
- Backward compat : additive; no existing tables altered; new table NULLable columns only
                    (outcome_note, deleted_at). Running code unaffected.
- Migration       : 0036_candidate_screenings; downgrade implemented: yes
                    (DROP TABLE candidate_screenings CASCADE; DROP TYPE screening_status_enum)
- Validation      : upgrade adds table + enum + indexes + trigger; downgrade drops cleanly.
                    No existing data affected — new table only.
- Rollback        : alembic downgrade 0035_drop_ivw_kits; drops candidate_screenings + enum.
- Notes           : Phase 21B will add Celery question-generation (no schema change needed —
                    screening_questions JSONB already present). Phase 21C will add application
                    gate enforcement (no schema change needed — find_shortlisted query in repo).

### [2026-06-30] Drop interview_kits, scorecards, scorecard_entries tables — 0035_drop_ivw_kits
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (positions interview kit removal)
- Trigger         : code change — position-level interview kit feature removed; per-interview-level kit (interview_level_kits, Phase 20/20B) is the sole kit mechanism going forward
- Module(s)       : positions (candidates/repository.py also updated)
- Change type     : drop table (×3); drop enum (×1)
- Objects         : interview_kits, scorecards, scorecard_entries (CASCADE), interview_kit_status_enum
- Storage decision: N/A — tables dropped (feature removed, no replacement at position level)
- Backward compat : no running code reads these tables after this change; candidates/repository.py get_open_position_jds() now returns focus_areas=[] instead of interview kit focus areas
- Migration       : 0035_drop_ivw_kits; downgrade implemented: yes (emergency schema restore, no data)
- Validation      : upgrade drops all 3 tables + enum; existing data discarded; no FK violations (CASCADE)
- Rollback        : alembic downgrade 0034_qbank_deleted_at — recreates tables (empty)
- Notes           : interview_level_kits table (Phase 20) is unaffected; that is the active kit table

### [2026-06-30] Add deleted_at to interview_question_bank (soft-delete mandate) — 0034_qbank_deleted_at
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (Phase 20B principal-reviewer finding MAJ-2)
- Trigger         : code change — MAJ-2 finding: platform-wide soft-delete mandate requires deleted_at on every entity table; column was omitted from 0032 for interview_question_bank
- Module(s)       : interviews
- Change type     : add column (×1)
- Objects         : interview_question_bank.deleted_at TIMESTAMPTZ NULL
- Storage decision: nullable TIMESTAMPTZ — standard platform soft-delete sentinel; find_questions_for_focus_area now filters deleted_at IS NULL to exclude soft-deleted questions from blending
- Backward compat : purely additive; nullable column with no default; existing rows unaffected (NULL = not deleted)
- Migration       : 0034_qbank_deleted_at; downgrade: DROP COLUMN IF EXISTS deleted_at
- Validation      : column added; existing rows have NULL; ORM model and repository filter in sync
- Rollback        : alembic downgrade 0033_level_kits_deleted_at — drops the column cleanly

### [2026-06-30] Add deleted_at to interview_level_kits (patch) — 0033_level_kits_deleted_at
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (patch for PR #102 principal-reviewer finding MIN-1)
- Trigger         : code change — MIN-1 finding: platform-wide soft-delete mandate requires deleted_at on every entity; column was omitted from 0032 (migration had already been applied to dev DB before the fix was committed)
- Module(s)       : interviews
- Change type     : add column (×1)
- Objects         : interview_level_kits.deleted_at TIMESTAMPTZ NULL
- Storage decision: nullable TIMESTAMPTZ — standard platform soft-delete sentinel; IF NOT EXISTS guard ensures idempotency on DBs that received deleted_at via 0032 directly
- Backward compat : purely additive; nullable column with no default; existing rows unaffected
- Migration       : 0033_level_kits_deleted_at; downgrade: DROP COLUMN IF EXISTS deleted_at
- Validation      : column added; existing rows have NULL; model and migration now in sync
- Rollback        : alembic downgrade 0032_level_kits_question_bank — drops the column cleanly

### [2026-06-30] Add interview_level_kits and interview_question_bank tables — 0032_level_kits_question_bank
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (per-level interview kit feature)
- Trigger         : new feature — per-level AI-generated interview kit (candidate × level); question bank for reuse
- Module(s)       : interviews
- Change type     : add table (×2); add index (×4)
- Objects         : interview_level_kits (one kit per interview instance), interview_question_bank (growing cross-position question store); ix_interview_level_kits_interview_id, ix_interview_level_kits_candidate_id, ix_interview_question_bank_org_focus, ix_interview_question_bank_complexity
- Storage decision: REAL TABLES — interview_level_kits is FK-linked to interviews, hot-queried by interview_id for the GET level-kit endpoint, and stores complex nested JSONB (10 focus areas × 5 questions each) that cannot be modelled in lookup_values or metadata JSONB. interview_question_bank is a growing, indexed cross-position store queried by org + focus_area + complexity with usage_count ordering — requires real table + composite index.
- Backward compat : purely additive (new tables only); no existing tables, columns, or queries affected; interview_level_kits.interview_id FK references interviews.id; all new columns carry server defaults (status='pending', arrays='[]', usage_count=0)
- Migration       : 0032_level_kits_question_bank; downgrade implemented: yes (DROP TABLE IF EXISTS CASCADE for both tables + DROP INDEX for all 4 indexes)
- Validation      : upgrade creates tables + indexes; downgrade drops all 4 indexes and both tables; existing interviews table unaffected
- Rollback        : alembic downgrade 0031_candidate_hire_unique — drops both tables and all 4 indexes cleanly
- Notes           : interview_level_kits has no deleted_at (kits are re-generated via the regenerate endpoint; a new kit row replaces the previous status); interview_question_bank.tags uses TEXT[] not JSONB (flat array of strings, not structured); version column on interview_level_kits supports future optimistic concurrency if needed

### [2026-06-29] One-active-hire-per-candidate partial unique indexes — 0031_candidate_hire_unique
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (quality hardening sprint)
- Trigger         : new feature — enforce candidate uniqueness across offer acceptance and application hiring (BR-013/BR-014 in offers spec, applications spec BR-012)
- Module(s)       : offers, applications
- Change type     : add index (×2)
- Objects         : uq_offers_candidate_accepted (offers.candidate_id WHERE status='accepted' AND deleted_at IS NULL); uq_applications_candidate_hired (applications.candidate_id WHERE status='hired')
- Storage decision: REAL partial unique indexes — these are the DB backstop for a service-layer guard; enforcing uniqueness at DB level prevents race conditions. No new columns/tables needed.
- Backward compat : partial indexes only index rows matching the WHERE predicate. Migration includes a pre-flight CTE UPDATE that resolves any pre-existing duplicates before creating the index (duplicate hired applications → 'active'; duplicate accepted offers → 'withdrawn'). Dev data confirmed to have 1 duplicate hired pair (2026-06-23 test data); migration now handles it in the same transaction.
- Migration       : 0031_candidate_hire_unique; downgrade implemented: yes (DROP INDEX IF EXISTS)
- Validation      : upgrade creates indexes; downgrade drops them; existing non-accepted/non-hired rows unaffected
- Rollback        : alembic downgrade 0030_ivw_hist_partitions drops both indexes cleanly
- Notes           : Service guards (do_create + do_accept in offers, update_status in applications) run BEFORE the DB constraint; the index is the safety net for concurrent requests

### [2026-06-29] Extend interview_status_history partitions through 2027-12 — 0030_ivw_hist_partitions
- Baseline        : v2.2 (11-Jun-2026)
- Author          : main-loop (partition gap fix — quality hardening sprint)
- Trigger         : code change — 0023_interviews created partitions only for 2026-06/07/08; rows with changed_at >= 2026-09-01 would fail with no-partition error
- Module(s)       : interviews
- Change type     : add partition (×16)
- Objects         : interview_status_history_2026_09 … interview_status_history_2027_12
- Storage decision: RANGE partitions — partitioned table already established in 0023; extending coverage is additive DDL only, no new columns or tables
- Backward compat : purely additive; no existing rows or running code affected; existing partitions (_2026_06/07/08) untouched
- Migration       : 0030_ivw_hist_partitions; downgrade implemented: yes (DROP TABLE IF EXISTS for each new partition in reverse order)
- Validation      : applied via `alembic upgrade head` against local DB; downgrade verified round-trip
- Rollback        : `alembic downgrade 0029_app_owning_recruiter` — drops only the 16 new partitions
- Notes           : covers 2026-09-01 through 2028-01-01 (~18 months); add further partitions ~6 months before 2028-01-01

### [2026-06-24] Add owning_recruiter_id to applications — 0029_application_owning_recruiter
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (positions ageing/closed/recruiter-breakdown enhancement)
- Trigger         : new feature — recruiter breakdown in positions list (Change-6/7); spec §Change-7
- Module(s)       : applications
- Change type     : add column; add index
- Objects         : applications.owning_recruiter_id (UUID NULL FK→users.id); ix_applications_owning_recruiter
- Storage decision: REAL COLUMN — owning_recruiter_id is FK-relational (references users), hot-queried for filled_count aggregation in the position recruiter breakdown, and needs an index for recruiter-scoped queries.
- Backward compat : NULLable column with no default; all existing rows get owning_recruiter_id=NULL automatically; all existing queries are unaffected. No deleted_at on applications — plain index (not partial).
- Migration       : 0029_application_owning_recruiter; downgrade implemented: yes (drop index + drop column)
- Validation      : upgrade reverses cleanly; downgrade removes column + index safely
- Rollback        : alembic downgrade 0028_position_closed_status_ageing
- Notes           : auto-set to actor.id when actor.role='recruiter' at create time; no deleted_at on applications table so partial index not applicable

### [2026-06-24] Add closed to position_status_enum, index on positions.approved_at — 0028_position_closed_status_ageing
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (positions ageing/closed enhancement)
- Trigger         : new feature — positions ageing_days, closed status, auto-close (BR-050)
- Module(s)       : positions
- Change type     : add enum value; add index
- Objects         : position_status_enum ('closed' added); ix_positions_approved_at (partial WHERE deleted_at IS NULL)
- Storage decision: REAL COLUMN (enum extension) — 'closed' is a lifecycle status, hot-queried in list/detail responses. Index on approved_at — ageing_days is computed as CURRENT_DATE - approved_at::date; this column is range-queried for ageing bucket filters; a partial index on non-deleted rows is the efficient choice.
- Backward compat : additive only; existing rows with other statuses are unaffected; 'closed' is system-set only (auto-close Celery task); existing queries tolerate the new enum value transparently.
- Migration       : 0028_position_closed_status_ageing; downgrade implemented: yes (drops index, recreates enum without 'closed' — safe only on dev DB with no closed positions)
- Validation      : upgrade tested on dev DB; downgrade documented as dev-only constraint (no closed positions)
- Rollback        : alembic downgrade 0027_pos_recruiter_assign; note: cannot remove 'closed' from live DB if any positions have been closed
- Notes           : ALTER TYPE ADD VALUE IF NOT EXISTS is safe and transactional in PG 12+

### [2026-06-24] Add position_recruiter_assignments table — 0027_pos_recruiter_assign
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (Recruiter Ownership enhancement)
- Trigger         : new feature — position recruiter ownership; spec §11 added
- Module(s)       : positions
- Change type     : new table
- Objects         : position_recruiter_assignments (id, position_id, recruiter_id, assigned_count, created_at, updated_at, deleted_at, version); indexes: uq_pos_recruiter_active (partial unique WHERE deleted_at IS NULL), ix_pos_rec_assign_position, ix_pos_rec_assign_recruiter
- Storage decision: REAL TABLE — recruiter assignments are relational (FK to positions + users), hot-queried for list enrichment, and must be indexed by recruiter_id for recruiter-scoped queries. JSONB on positions would not support efficient recruiter-scoped queries.
- Backward compat : fully additive new table; no existing columns or constraints modified; all existing list/detail queries unaffected (recruiter_assignments defaults to empty list)
- Migration       : 0027_pos_recruiter_assign; downgrade implemented: yes (DROP TABLE)
- Validation      : upgrade + downgrade tested locally
- Rollback        : alembic downgrade 0026_add_deleted_at_soft_delete
- Notes           : Partial unique index (WHERE deleted_at IS NULL) allows soft-delete + re-assign without unique violation. Revision ID shortened to 0027_pos_recruiter_assign to fit alembic_version.version_num VARCHAR(32).

### [2026-06-23] Add deleted_at soft-delete column to interviews, screening_decisions, bulk_upload_jobs — 0026_add_deleted_at_soft_delete
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (BUG-6/7/8 — soft-delete mandate compliance)
- Trigger         : code change — soft-delete mandate in CLAUDE.md was violated; three tables lacked deleted_at; discovered during test data cleanup
- Module(s)       : interviews, screening (screening_decisions), candidates (bulk_upload_jobs)
- Change type     : add column
- Objects         : interviews.deleted_at, screening_decisions.deleted_at, bulk_upload_jobs.deleted_at
- Storage decision: REAL COLUMN — deleted_at is the platform-wide soft-delete guard; must be a real column (hot-queried by all list/get paths, indexed on other tables). Per SCHEMA_EVOLUTION.md §2.1: platform standard is deleted_at on all mutable entities.
- Backward compat : fully additive; NULLable with no default — all existing rows read as not-deleted (NULL = live); no code path sets deleted_at on these tables yet (no delete endpoints exist); running code unaffected
- Migration       : 0026_add_deleted_at_soft_delete; downgrade implemented: yes (drop_column on all three)
- Validation      : upgrade + downgrade tested locally against existing data; no constraint violations
- Rollback        : alembic downgrade 0025_offers
- Notes           : Supersedes the "No deleted_at on interviews" note in 0023_interviews entry (that note reflected original spec intent; CLAUDE.md soft-delete mandate takes precedence). The lifecycle-via-status model (BR-003) is unchanged — deleted_at is the administrative hard-stop guard, not used by any status transition. No delete endpoints exist yet on these tables; deleted_at is additive infrastructure.

### [2026-06-23] Add offers table, offer_status_enum, role and permission seeds — 0025_offers
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (Phase 21-A — offers module)
- Trigger         : new module — openspec/specs/offers/spec.md; Phase 21 Offers backend
- Module(s)       : offers
- Change type     : add table, add enum, add indexes, add trigger, add role seed, add permission seeds
- Objects         : offer_status_enum (enum); offers table (id, application_id, position_id, candidate_id, created_by, approved_by, attested_by, status, offer_date, valid_until, gross_ctc_annual, compensation_breakdown, pdf_s3_key, rejection_reason, withdrawal_reason, decline_reason, approved_at, attested_at, sent_at, accepted_at, version, deleted_at, created_at, updated_at); uq_offers_application (UNIQUE constraint); idx_offers_candidate, idx_offers_position, idx_offers_status, idx_offers_ctc (partial indexes WHERE deleted_at IS NULL); trg_offers_updated (trigger); role offer_approver; permissions offers:create/read/update/approve/attest
- Storage decision: REAL TABLE — offers is a relational entity joining application<->candidate<->position; hot-queried with status-filtered indexes. compensation_breakdown: JSONB — recruiter-defined flexible structure (per SCHEMA_EVOLUTION.md §3.1). gross_ctc_annual: REAL NUMERIC column — indexed for range queries. position_id/candidate_id: denormalized real columns — avoid JOIN on every list query
- Backward compat : fully additive; new table + new enum only; no existing tables altered; all FK constraints reference existing tables; all new columns NULLable or defaulted
- Migration       : 0025_offers; downgrade implemented: yes (DROP TABLE offers CASCADE, DROP TYPE offer_status_enum, DELETE seeded role + permissions)
- Validation      : upgrade creates enum + table + indexes + trigger + seeds; downgrade drops table CASCADE then enum; existing data untouched; roundtrip tested locally
- Rollback        : alembic downgrade 0024_drop_int_status_trigger; drops offers table CASCADE and offer_status_enum
- Notes           : One offer per application enforced by UNIQUE constraint uq_offers_application; deleted_at soft-delete; fn_set_updated_at_and_version() trigger bumps version + updated_at on every UPDATE; pdf_s3_key='__error__' sentinel set by Celery task on final failure

### [2026-06-21] Drop interview status-change trigger (MAJ-2 fix) — 0024_drop_int_status_trigger
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (Phase 19-A MAJ-2 fix)
- Trigger         : code change — trigger fn_track_interview_status_change recorded NEW.created_by as changed_by (wrong actor); openspec/specs/interviews/spec.md §9 MAJ-2
- Module(s)       : interviews
- Change type     : drop trigger, drop function
- Objects         : trg_int_status_change (trigger on interviews table); fn_track_interview_status_change (plpgsql function)
- Storage decision: N/A — removing objects; service now owns history insertion via InterviewRepository.create_history_entry with the real actor UUID
- Backward compat : no data loss (zero rows were written by the trigger — no status-changing endpoints existed in 19-A); removes buggy write path; existing interview rows unaffected
- Migration       : 0024_drop_int_status_trigger; downgrade implemented: yes (recreates function + trigger from 0023)
- Validation      : upgrade: trigger + function absent after migration; downgrade: recreates both idempotently; no 19-A interview rows had status changes so no history data exists to corrupt
- Rollback        : alembic downgrade 0023_interviews; recreates trigger + function
- Notes           : 19-B+ status-changing endpoints must call repo.create_history_entry() explicitly; see spec §9 for context

### [2026-06-21] Add interview tables (Phase 19-A) — 0023_interviews
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (Phase 19-A — interviews module, create + read + schedule)
- Trigger         : new module — openspec/specs/interviews/spec.md; Phase 19 Interviews backend
- Module(s)       : interviews
- Change type     : add table (x4), add indexes, add trigger, add function, add partitions
- Objects         : interviews table; uq_interview_app_level, idx_interviews_app, idx_interviews_status, idx_interviews_status_sched (indexes); trg_interviews_upd (trigger); interview_status_history (partitioned table) + 3 partitions (2026_06/07/08); idx_int_status_hist_interview (index); fn_track_interview_status_change (function); trg_int_status_change (trigger); interview_panelist_assignments + uq_int_panelist_seq + idx_int_panelists_email; interview_feedback + uq_feedback_int_panelist + idx_feedback_interview
- Storage decision: REAL TABLES — interviews is a relational entity joining application↔interview_level; hot-queried with FK constraints and status-filtered indexes; status_history is partitioned by changed_at for performance; panelist_assignments and feedback are child tables with FK integrity to interviews
- Backward compat : fully additive; new tables only; enums already exist from prior migrations (0015); no existing tables altered; all FK constraints reference existing tables
- Migration       : 0023_interviews; downgrade implemented: yes (drops all tables CASCADE in reverse dependency order, drops function)
- Validation      : upgrade creates 4 tables + all indexes + triggers + function; downgrade drops cleanly via CASCADE; no existing data touched
- Rollback        : alembic downgrade 0022_applications; drops all interview tables/indexes/triggers/function
- Notes           : No deleted_at on interviews (spec — lifecycle via status); no RLS (interviews not org-scoped by policy); trigger fn_track_interview_status_change records every status/pending_reason/hold_reason change with timestamp; partitioned status_history supports future time-range queries; CITEXT columns mapped as String in SQLAlchemy ORM

### [2026-06-20] Add applications table (P18-A) — 0022_applications
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (P18-A — applications module, create + read endpoints)
- Trigger         : new module — openspec/specs/applications/spec.md; Phase 18 Applications backend
- Module(s)       : applications
- Change type     : add table, add enum, add indexes, add trigger, add RLS policy
- Objects         : application_status_enum (enum); applications table (id, candidate_id, position_id, organization_id, status, applied_at, metadata, created_by, version, created_at, updated_at); uq_app_cand_pos (partial unique index WHERE status != 'withdrawn'); idx_app_candidate, idx_app_position, idx_app_org, idx_app_pos_status (indexes); trg_app_upd (trigger); rls_applications_isolation (RLS policy)
- Storage decision: REAL TABLE — applications is a relational join entity (candidate↔position↔org) that is the JOIN key for screening_decisions, interviews, and reporting queries; hot-queried with FK constraints and status-filtered indexes; not suitable for JSONB, lookup_values, or tag storage
- Backward compat : fully additive; new table with no dependency on existing rows; all constraints are on the new table only; existing tables and running code are unaffected
- Migration       : 0022_applications; downgrade implemented: yes (DROP TABLE CASCADE + DROP TYPE)
- Validation      : upgrade creates enum + table + indexes + trigger + RLS; downgrade drops table CASCADE (removes indexes/trigger/policy) then drops enum cleanly; no existing data touched
- Rollback        : alembic downgrade 0021_cand_last_matched_at; drops applications table CASCADE and application_status_enum
- Notes           : No deleted_at column by design (spec BR-005) — status='withdrawn' is the canonical removal; partial unique index enforces at-most-one non-withdrawn application per candidate+position (BR-001); org isolation via RLS (fn_current_org); fn_set_updated_at_and_version() trigger bumps version + updated_at on UPDATE

### [2026-06-20] Add candidates.last_matched_at — 0021_cand_last_matched_at
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (bug fix — matching timestamp for pipeline observability)
- Trigger         : code change / bug fix — fix/candidates-matching-bugs; `_do_match()` in candidates/tasks.py now stamps `last_matched_at` after each matching run; column was absent, causing no observable signal for "when did matching last run?" in the detail API
- Module(s)       : candidates
- Change type     : add column
- Objects         : candidates.last_matched_at TIMESTAMPTZ NULL
- Storage decision: REAL COLUMN — hot-queried via GET /candidates/{id} (CandidateDetailResponse); must be indexed if needed for "stale match" queries; a real column was chosen because it is a timestamp with direct functional semantics (last matching completion time), not a free-form attribute suitable for metadata JSONB
- Backward compat : fully additive; existing rows get NULL (no backfill needed — NULL correctly means "matching has not run yet"); running code unaffected until column is read; downgrade drops the column cleanly
- Migration       : 0021_cand_last_matched_at; downgrade implemented: yes (drops column)
- Validation      : upgrade adds column without touching existing data; downgrade drops cleanly; no data backfill required
- Rollback        : alembic downgrade 0020_cand_display_name_null; drops last_matched_at column
- Notes           : stamped unconditionally after matching loop in _do_match() (even 0 results); surfaces in CandidateDetailResponse.last_matched_at

### [2026-06-19] Make candidates.display_name nullable — 0020_cand_display_name_null
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (bug fix — extraction placeholder not cleared)
- Trigger         : code change / bug fix — PR #59; `_build_update_from_profile()` in candidates/tasks.py now writes `display_name=None` when the NLP extractor finds no parseable name; the column was NOT NULL, causing IntegrityError on every no-name extraction
- Module(s)       : candidates
- Change type     : alter column (nullable relaxation)
- Objects         : candidates.display_name VARCHAR(255) NOT NULL → NULL
- Storage decision: REAL COLUMN — existing column; nullable relaxation required because the business semantic changed from "placeholder value always present" to "NULL = name extraction found no parseable name"; column is hot-queried for display and participates in uq_candidates_identity expression index (LOWER(TRIM(display_name)))
- Backward compat : fully backward-compatible; existing non-NULL values untouched; new NULLs are valid (PostgreSQL expression indexes treat NULL as distinct — multiple NULL display_name rows do not violate uq_candidates_identity); dedup guard in tasks.py already checks `if profile.display_name` before calling `_find_identity_duplicate`; frontend already uses `display_name ?? "Unknown"` fallback
- Migration       : 0020_cand_display_name_null; downgrade implemented: yes (back-fills '(name unknown)' for any NULLs, then restores NOT NULL)
- Validation      : upgrade verified on local DB (atsplatform); 23 candidates re-extracted; 10 correctly show NULL display_name with extraction_status=completed; downgrade implemented with back-fill guard
- Rollback        : alembic downgrade 0019_position_scorecards; back-fills NULL rows with '(name unknown)' then restores NOT NULL constraint; requires 0 NULL rows or the back-fill must run first
- Notes           : uq_candidates_identity partial index (WHERE deleted_at IS NULL) uses LOWER(TRIM(display_name)); expression on NULL returns NULL; NULL ≠ NULL in unique expression index so no constraint violations arise

### [2026-06-18] Add candidate_position_scorecards table (Phase 17A AI screening) — 0019_position_scorecards
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (Phase 17A — AI Screening Engine)
- Trigger         : new feature — Phase 17A AI Screening Engine (spec §9.12, §9.16); requires persisting per-focus-area interview scorecards generated at match time
- Module(s)       : candidates/screening
- Change type     : add table
- Objects         : candidate_position_scorecards (id, candidate_id, position_id, match_id, interview_level, focus_area, questions JSONB, previous_level_context JSONB, screening_provider, ai_generation_tokens, version, created_at, updated_at, deleted_at); indexes idx_scorecards_candidate_id, idx_scorecards_match_id, idx_scorecards_position_id; trigger trg_scorecards_upd; UNIQUE (candidate_id, position_id, interview_level, focus_area)
- Storage decision: REAL TABLE — hot-queried per match_id during interview sessions (GET /matches/{id}/scorecard); has FK to candidate_position_matches (relational integrity required); multi-row per match (one per focus area); JSONB for questions array (variable-length structured data per spec); SCHEMA_EVOLUTION.md decision tree: FK + relational integrity + hot-query path → real table
- Backward compat : fully additive; new standalone table; no existing table or enum altered; existing rows in all other tables unaffected; soft-delete (deleted_at) + optimistic concurrency (version) per CLAUDE.md
- Migration       : 0019_position_scorecards; downgrade implemented: yes (drops trigger, function, indexes, table in reverse order)
- Validation      : upgrade creates new table without touching existing data; downgrade cleanly removes all objects; no data backfill required; uuid_generate_v4() already available from 0001 baseline
- Rollback        : alembic downgrade 0018_cand_src_direct; drops trg_scorecards_upd, fn_set_scorecards_updated_at_and_version(), idx_scorecards_* indexes, and candidate_position_scorecards table
- Notes           : UNIQUE constraint (candidate_id, position_id, interview_level, focus_area) enables safe upsert in ScreeningService._upsert_scorecards; questions JSONB holds 5-element array per spec §9.11.3; previous_level_context nullable (populated for Level N>1 scorecards per multi-level chain rule)

### [2026-06-18] Add 'direct' to candidate_source_enum — 0018_cand_src_direct
- Baseline        : v2.2 (11-Jun-2026)
- Author          : main-loop (Phase 17 — Candidates backend)
- Trigger         : new module — candidates (Phase 17); spec §10 defines source ∈ {direct, employee_referral, third_party_vendor}; 0010 created enum with old schema.sql values; 0012 added employee_referral + third_party_vendor but omitted 'direct'
- Module(s)       : candidates
- Change type     : add enum value
- Objects         : candidate_source_enum ('direct' added)
- Storage decision: ENUM VALUE — candidate source is a fixed, bounded set used as a type constraint on candidates.source and candidate_source_details.source; no JSONB or lookup table needed (small closed set, direct DB-level constraint is correct per SCHEMA_EVOLUTION.md)
- Backward compat : fully additive; ALTER TYPE … ADD VALUE IF NOT EXISTS is non-transactional in PG; existing rows with old enum values (direct_application, referral, etc.) are unaffected; 'direct' is unused before this migration
- Migration       : 0018_cand_src_direct; downgrade: no-op (PG cannot remove enum values without drop+recreate; 'direct' is safe to leave — no prior row uses it)
- Validation      : IF NOT EXISTS guard; re-runnable safely; upgrade confirmed against local dev DB at head 0017_lvl_panelist_fk
- Rollback        : no rollback needed; leaving 'direct' in the enum does not affect any prior code path
- Notes           : enum still contains old schema.sql values (direct_application, referral, etc.) — these are not used by new code but cannot be removed without a full enum recreate; future cleanup can use expand→contract in a later release

### [2026-06-18] Add panelist_id FK to interview_levels (CR-001) — 0017_lvl_panelist_fk
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (CR-001 — interview-levels panelist assignment)
- Trigger         : new feature — link an assigned interviewer to each interview level row on a position; spec change: openspec/specs/positions/spec.md §3 interview-levels endpoints
- Module(s)       : positions (interview_levels table)
- Change type     : add column, add FK constraint, add index
- Objects         : interview_levels.panelist_id (UUID nullable FK → interview_panelists.id ON DELETE SET NULL); FK constraint fk_interview_levels_panelist; partial index idx_interview_levels_panelist WHERE panelist_id IS NOT NULL
- Storage decision: REAL COLUMN — relational FK to interview_panelists (hot-queried for eager-load in list_levels; drives the panelist picker in the positions UI; FK integrity required; JSONB/lookup_values cannot provide referential integrity or column-level eager-load per SCHEMA_EVOLUTION.md decision tree)
- Backward compat : fully additive; nullable column; ON DELETE SET NULL preserves level rows when a panelist is deleted; existing rows get panelist_id = NULL; running code that reads InterviewLevel rows is unaffected. ON DELETE SET NULL guards only against physical hard-delete (which is prohibited in this system via soft-delete). Deactivating a panelist after assignment does NOT automatically clear interview_levels.panelist_id — the assignment is historical.
- Migration       : 0017_lvl_panelist_fk; downgrade implemented: yes (drops index + FK + column)
- Validation      : upgrade adds column to existing table without touching any row data; downgrade cleanly removes constraint, index, and column; no data backfill required
- Rollback        : alembic downgrade 0016_panelist_enhancements; drops idx_interview_levels_panelist, fk_interview_levels_panelist, and column panelist_id
- Notes           : down_revision = 0016_panelist_enhancements (rechained after PR #34 merge; 0016 must exist before 0017 applies)

### [2026-06-18] Add technical_competencies and consulting_fee_inr to interview_panelists — 0016_panelist_enhancements
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer
- Trigger         : new feature — Phase 16 post-launch enhancement to interview_panelists; adds two additional panelist fields. Branch: dev/phase-16-post-panelist-fields
- Module(s)       : interview_panelists
- Change type     : add column (×2)
- Objects         : interview_panelists.technical_competencies (TEXT NULL), interview_panelists.consulting_fee_inr (INTEGER NULL)
- Storage decision: REAL COLUMNS — both are well-defined, named fields belonging exclusively to the panelist entity (not arbitrary metadata). Display-prominent in the UI. Not indexed (neither field is filtered/searched). Per SCHEMA_EVOLUTION.md decision tree: metadata JSONB rejected (named, typed fields → real columns preferable for clarity and type safety); lookup_values/custom_field_definitions/tags all inapplicable.
- Backward compat : fully additive — both columns are NULLable with no default; existing rows get NULL (unaffected). Running code on 0015 head works without change. No FK or index dependencies.
- Migration       : 0016_panelist_enhancements; downgrade implemented: yes (drops both columns; safe since columns are NULL for existing rows)
- Validation      : offline `alembic upgrade --sql` / `downgrade --sql` verified; additive-only DDL (two ALTER TABLE ADD COLUMN).
- Rollback        : `alembic downgrade 0015_interview_panelists`; drops both columns cleanly (no data loss if columns are NULL)
- Notes           : consulting_fee_inr stores whole INR (integer); formatting (e.g. "INR 1,000") is UI-only responsibility. Pydantic validator enforces >= 0 on create and update.

### [2026-06-17] Add interview_panelists table (global panelist master) — 0015_interview_panelists
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (Phase 16 — interview-panelists module)
- Trigger         : new module — interview-panelists master list; FK target for CR-001 interview_levels.panelist_id. Spec: openspec/changes/interview-panelists/specs/interview-panelists/spec.md
- Module(s)       : interview_panelists (new)
- Change type     : add table, add enum (×2), add indexes (×4), add trigger, add RLS policy
- Objects         : table interview_panelists; enums panelist_category, panelist_level_capability; indexes uq_interview_panelists_email (partial unique), idx_interview_panelists_name_trgm (GIN), idx_interview_panelists_email_trgm (GIN), idx_interview_panelists_active; trigger trg_interview_panelists_version; RLS policy interview_panelists_internal_only
- Storage decision: REAL TABLE — hot-queried by CR-001 panelist picker; FK target for interview_levels.panelist_id; email/name/capability are searched and indexed; cannot use JSONB/lookup_values for FK integrity + column-level search (per SCHEMA_EVOLUTION.md decision tree)
- Backward compat : fully additive (new table + enums only; no existing table altered); no FK from existing tables; existing migrations unaffected; rollback drops table + enums cleanly
- Migration       : 0015_interview_panelists; downgrade implemented: yes
- Validation      : offline alembic upgrade --sql / downgrade --sql checked for DDL correctness
- Rollback        : alembic downgrade 0014_consents_ip_to_varchar; drops table, enums, trigger, function, indexes cleanly; safe since no FK dependents yet (CR-001 not applied)
- Notes           : CITEXT requires pg_trgm extension for GIN trigram indexes (both enabled in Alembic 0001 baseline). RLS policy requires app.is_internal session variable set by get_current_user dependency.

### [2026-06-17] Cast candidate_consents.ip_address from inet to VARCHAR(45) — 0014_consents_ip_to_varchar
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (Phase 16)
- Trigger         : code change — ORM model declares ip_address as String(45) (VARCHAR) but
                    the schema.sql baseline created the column as inet; asyncpg rejects the
                    type mismatch on INSERT. Defect surfaced during integration testing.
                    Spec: openspec/specs/candidates/spec.md §7.
- Module(s)       : candidates
- Change type     : alter column (type cast)
- Objects         : candidate_consents.ip_address — cast from inet to VARCHAR(45)
- Storage decision: REAL COLUMN (existing). Type cast only — no new storage; VARCHAR(45) is a
                    superset of valid inet string representations (max IPv6 = 39 chars + optional
                    /prefix), so no data is lost. The change aligns DB type with ORM declaration.
- Backward compat : VARCHAR(45) accepts every value that inet accepted; reads and writes using
                    the old inet type are replaced by string-compatible reads and writes. No
                    data loss; downgrade casts back via ip_address::inet (safe if all stored
                    values are valid inet strings — guaranteed since the column was inet before).
- Migration       : 0014_consents_ip_to_varchar; downgrade implemented? yes (casts back to inet).
- Validation      : offline `alembic upgrade --sql` / `downgrade --sql` (clean ALTER TYPE USING).
                    NOT yet applied to any live DB — human-approved apply step required.
- Rollback        : `alembic downgrade 0013_candidates_dedup_col` (casts back to inet).
- Notes           : Paired with the Phase 16 principal-reviewer defect fix batch.
                    Linked spec: openspec/specs/candidates/spec.md §7.

### [2026-06-17] Add candidates.duplicate_of_candidate_id self-referential FK — 0013_candidates_dedup_col
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (Phase 16)
- Trigger         : code change — ORM model and Phase-16 spec declare duplicate_of_candidate_id
                    for post-extraction identity-dedup (BUG-003 fix). The schema.sql baseline
                    omitted this column; integration tests confirmed the FK was missing.
                    Spec: openspec/specs/candidates/spec.md §2 (dedup flow).
- Module(s)       : candidates
- Change type     : add column (1)
- Objects         : candidates.duplicate_of_candidate_id UUID NULL FK → candidates(id)
- Storage decision: REAL COLUMN per docs/SCHEMA_EVOLUTION.md decision tree. This is a relational
                    self-referential FK used in the dedup resolution query (WHERE
                    duplicate_of_candidate_id = :id) — cannot be JSONB; must be a typed FK column.
- Backward compat : NULLable; existing rows get NULL. Running code that does not set this column
                    is unaffected. The FK references the same table (self-referential) — safe additive.
- Migration       : 0013_candidates_dedup_col; downgrade implemented? yes (DROP COLUMN).
- Validation      : offline `alembic upgrade --sql` / `downgrade --sql` (clean ADD/DROP COLUMN).
                    NOT yet applied to any live DB — human-approved apply step required.
- Rollback        : `alembic downgrade 0012_candidates_schema_bridge` (drops the column).
- Notes           : BUG-003: without this column the dedup flag written by _do_extract caused
                    an IntegrityError on the uq_candidates_identity unique index on re-upload.
                    Linked spec: openspec/specs/candidates/spec.md §2.

### [2026-06-17] Bridge — align candidate schema with Phase-16 ORM — 0012_candidates_schema_bridge
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (Phase 16)
- Trigger         : code change — integration test run revealed gaps between the schema.sql
                    baseline (used to bootstrap dev DB) and Phase-16 ORM expectations in
                    migrations 0010/0011. Bridge revision to close all gaps additively.
                    Spec: openspec/specs/candidates/spec.md §§2,5,6,10.
- Module(s)       : candidates
- Change type     : add column (4) + add enum value (4) + add table (1) + add trigger (1) +
                    add RLS policy (1) + add grant (1)
- Objects         : candidates.extraction_provider VARCHAR(64) NULL;
                    candidate_position_matches.screening_provider VARCHAR(64) NULL;
                    candidate_position_matches.ai_analysis_tokens INTEGER NULL;
                    bulk_upload_jobs.updated_at TIMESTAMPTZ NOT NULL DEFAULT now();
                    candidate_source_enum: + 'employee_referral', + 'third_party_vendor';
                    candidate_extraction_status_enum: + 'manual_review';
                    candidate_consent_status_enum: + 'expired';
                    TABLE candidate_source_details (candidate_id UUID PK FK → candidates,
                    source candidate_source_enum, referrer_full_name VARCHAR(255) NULL,
                    referrer_mobile_enc BYTEA NULL, referrer_mobile_hash TEXT NULL,
                    referrer_portco_name VARCHAR(255) NULL, referrer_office_email_enc BYTEA NULL,
                    referrer_office_email_hash TEXT NULL, vendor_partner_name VARCHAR(255) NULL,
                    vendor_fee_pct NUMERIC(5,2) DEFAULT 8.33, version INTEGER, created_at
                    TIMESTAMPTZ, updated_at TIMESTAMPTZ); trigger trg_candidate_source_details_upd;
                    RLS rls_candidate_source_details_isolation (fn_is_internal()); grant to ats_app.
- Storage decision: REAL COLUMNS/TABLE per docs/SCHEMA_EVOLUTION.md decision tree.
                    extraction_provider and screening_provider: scalar VARCHAR attributes queried
                    for provider audit/billing reporting — not sparse JSONB.
                    ai_analysis_tokens: INTEGER queried for cost telemetry — not JSONB.
                    updated_at: required for job status polling / optimistic concurrency trigger.
                    candidate_source_details: 1:1 relational table holding conditional encrypted
                    PII (referrer_mobile_enc, referrer_office_email_enc) that must be BYTEA —
                    cannot be JSONB. Enum values are additive additions to existing types.
- Backward compat : All new columns are NULLable or have server defaults. New table is additive.
                    Enum ADD VALUE IF NOT EXISTS is idempotent. Running Phase-16 (0010/0011) code
                    is unaffected by the additive additions.
- Migration       : 0012_candidates_schema_bridge; downgrade implemented? yes (drops table,
                    columns; enum values cannot be removed from PG — documented partial downgrade).
- Validation      : offline `alembic upgrade --sql` / `downgrade --sql` (clean DDL).
                    NOT yet applied to any live DB — human-approved apply step required.
- Rollback        : `alembic downgrade 0011_candidate_matches_source` (drops candidate_source_details
                    table + added columns; enum value additions are left in place on downgrade —
                    additive and inert for old code).
- Notes           : The schema.sql baseline pre-dated the Phase-16 spec finalization; this bridge
                    closes the delta without altering any existing row or column. Paired with the
                    Phase 16 principal-reviewer defect fix batch. Linked spec:
                    openspec/specs/candidates/spec.md §§2,5,6,10.

### [2026-06-17] Phase 16 — candidate_position_matches + candidate_source_details — 0011_candidate_matches_source
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (Phase 16)
- Trigger         : new feature — Phase 16 Candidates module matching + source details
                    (spec §§5-6). Change: openspec/specs/candidates/spec.md §§5-6.
- Module(s)       : candidates
- Change type     : add table (2) + add enum (1)
- Objects         : ENUM match_status_enum (pending / matched / dismissed / expired);
                    TABLE candidate_position_matches (candidate_id UUID FK → candidates,
                    position_id UUID FK → positions, match_score NUMERIC(5,2),
                    match_points JSONB, no_match_points JSONB, status match_status_enum,
                    provider VARCHAR(128), dismissed_by UUID FK → users,
                    dismissed_at TIMESTAMPTZ, dismissed_reason TEXT, version INTEGER,
                    created_at TIMESTAMPTZ, updated_at TIMESTAMPTZ, deleted_at TIMESTAMPTZ;
                    UNIQUE (candidate_id, position_id) WHERE deleted_at IS NULL);
                    TABLE candidate_source_details (candidate_id UUID PK FK → candidates,
                    source candidate_source_enum, referrer_name VARCHAR(200) NULL,
                    referrer_mobile_enc BYTEA NULL, referrer_office_email_enc BYTEA NULL,
                    referrer_employee_id VARCHAR(64) NULL, vendor_name VARCHAR(200) NULL,
                    vendor_contact_name VARCHAR(200) NULL, vendor_fee_pct NUMERIC(5,2)
                    DEFAULT 8.33, version INTEGER, created_at TIMESTAMPTZ,
                    updated_at TIMESTAMPTZ; CHECKs ck_csd_referral + ck_csd_vendor)
- Storage decision: REAL TABLES per docs/SCHEMA_EVOLUTION.md decision tree.
                    match_score is a hot-queried NUMERIC column requiring indexed sort for
                    top-N ranking queries — cannot be JSONB. match_points / no_match_points
                    are JSONB: read/written as a list of strings, never filtered
                    individually; JSONB avoids a junction table. referrer_mobile_enc /
                    referrer_office_email_enc are BYTEA — PII requiring AES-256-GCM
                    ciphertext at the byte level, handled at service layer (shared.crypto);
                    plaintext must never touch the DB. vendor_fee_pct is a real column —
                    numeric, queried/aggregated in cost reports.
- Backward compat : Additive only. Two new tables; no existing table, column, enum, index,
                    or constraint is altered. Running Phase-15 and Phase-16 (0010) code is
                    unaffected.
- Migration       : 0011_candidate_matches_source; downgrade implemented? yes (drops
                    candidate_source_details, candidate_position_matches tables and
                    match_status_enum type in reverse dependency order).
- Validation      : offline `alembic upgrade --sql` / `downgrade --sql` (clean DDL only);
                    NOT yet applied to any live DB — human-approved apply step required via
                    DATABASE_ADMIN_URL as the schema owner.
- Rollback        : `alembic downgrade 0010_candidates` (drops both new tables and
                    match_status_enum; candidate_source_enum from 0010 is preserved).
- Notes           : match_status_enum is created in this revision (0011); candidate_source_enum
                    was created in 0010. Linked spec: openspec/specs/candidates/spec.md §§5-6.

### [2026-06-17] Phase 16 — core candidate tables — 0010_candidates
- Baseline        : v2.2 (11-Jun-2026)
- Author          : backend-engineer (Phase 16)
- Trigger         : new feature — Phase 16 Candidates module (spec §§1-8, §10). Change:
                    openspec/specs/candidates/spec.md §§1-8, §10.
- Module(s)       : candidates
- Change type     : add table (5) + add enum (3)
- Objects         : ENUMs candidate_source_enum (direct / employee_referral /
                    third_party_vendor), candidate_extraction_status_enum (pending /
                    processing / completed / failed), candidate_consent_status_enum
                    (pending / granted / withdrawn / expired);
                    TABLE candidates (id UUID PK, full_name VARCHAR(200),
                    email_enc BYTEA, mobile_enc BYTEA, address_enc BYTEA,
                    email_hash BYTEA UNIQUE, mobile_hash BYTEA UNIQUE,
                    primary_skills JSONB, secondary_skills JSONB,
                    total_experience_months INTEGER, extraction_status
                    candidate_extraction_status_enum, extraction_provider VARCHAR(128),
                    source candidate_source_enum, metadata_ JSONB, version INTEGER,
                    created_at TIMESTAMPTZ, updated_at TIMESTAMPTZ, deleted_at TIMESTAMPTZ;
                    GIN indexes on primary_skills / secondary_skills);
                    TABLE candidate_documents (id UUID PK, candidate_id UUID FK → candidates,
                    document_type VARCHAR(64), s3_key VARCHAR(1024), original_filename
                    VARCHAR(512), content_type VARCHAR(128), file_size_bytes INTEGER,
                    is_primary BOOLEAN DEFAULT FALSE, version INTEGER, created_at TIMESTAMPTZ,
                    updated_at TIMESTAMPTZ, deleted_at TIMESTAMPTZ);
                    TABLE bulk_upload_jobs (id UUID PK, uploaded_by UUID FK → users,
                    total_files INTEGER, processed_files INTEGER, failed_files INTEGER,
                    status VARCHAR(32), error_summary JSONB, version INTEGER, created_at
                    TIMESTAMPTZ, updated_at TIMESTAMPTZ, deleted_at TIMESTAMPTZ);
                    TABLE consent_purposes (id UUID PK, purpose_key VARCHAR(64) UNIQUE,
                    description TEXT, is_required BOOLEAN DEFAULT FALSE, version INTEGER,
                    created_at TIMESTAMPTZ, updated_at TIMESTAMPTZ, deleted_at TIMESTAMPTZ);
                    TABLE candidate_consents (id UUID PK, candidate_id UUID FK → candidates,
                    purpose_id UUID FK → consent_purposes, status candidate_consent_status_enum,
                    consented_at TIMESTAMPTZ, withdrawn_at TIMESTAMPTZ, expires_at TIMESTAMPTZ,
                    ip_address INET, user_agent TEXT, version INTEGER, created_at TIMESTAMPTZ,
                    updated_at TIMESTAMPTZ, deleted_at TIMESTAMPTZ;
                    UNIQUE (candidate_id, purpose_id) WHERE deleted_at IS NULL)
- Storage decision: REAL TABLES per docs/SCHEMA_EVOLUTION.md decision tree. PII columns
                    (email_enc, mobile_enc, address_enc) are BYTEA — AES-256-GCM ciphertext
                    handled at service layer (shared.crypto); email_hash / mobile_hash are
                    deterministic HMAC-SHA256 BYTEA for deduplication without decryption.
                    primary_skills / secondary_skills are JSONB — document-like arrays
                    written/read as a unit and searched via GIN containment (@> operator);
                    JSONB is the cheapest correct mechanism (no EAV). metadata_ is an
                    open-ended JSONB extension bag. All remaining columns are hot-queried,
                    indexed, or relational — real columns per the decision tree.
- Backward compat : Additive only. Five new tables and three new enums; no existing table,
                    column, enum, index, or constraint is altered. All prior code continues
                    to run without changes.
- Migration       : 0010_candidates; downgrade implemented? yes (drops all five tables and
                    the three enums in reverse dependency order; candidate_source_enum created
                    here is also dropped in downgrade since it is first introduced in this
                    revision).
- Validation      : offline `alembic upgrade --sql` / `downgrade --sql` (clean DDL only);
                    NOT yet applied to any live DB — human-approved apply step required via
                    DATABASE_ADMIN_URL as the schema owner.
- Rollback        : `alembic downgrade 0009_positions_status_remap` (drops all five tables and
                    three enums; no existing object is altered so rollback is clean).
- Notes           : Candidates are a GLOBAL talent pool (not org-scoped); RLS allows any
                    authenticated user to read non-deleted rows (no org isolation). The
                    candidate_source_enum supersedes the old 9-value free-form set documented
                    in the 2026-06-12 schema.sql baseline entry — the transition is handled
                    here (the old enum was never used in production). Linked spec:
                    openspec/specs/candidates/spec.md §§1-8, §10.

### [2026-06-16] Position current-status remap to on_hold — 0009_positions_status_remap
- Baseline        : v2.2 (11-Jun-2026)
- Author          : Claude Code (opus-4-8) for hareesh@stg.com
- Trigger         : defect fix — updated feature (Positions, round-2: current status limited to
                    open/in_progress/on_hold). Change: openspec/changes/positions-defects-round2.
                    Branch: dev/15.3-positions-defects-r2.
- Module(s)       : positions
- Change type     : data migration (NO DDL, NO enum mutation).
- Objects         : positions.status (data only) + corrective positions.hold_reason backfill.
- Storage decision: no schema change — existing columns/enum only. The other physical
                    position_status_enum values (portco_confirmed_yet_to_offer, offered,
                    offer_accepted, onboarded, no_show) are RETAINED (not dropped — no enum mutation
                    per the schema rule); they are simply no longer settable at the application layer
                    (schemas + transition logic). No-show/offer columns are LEFT INTACT (data
                    preserved) for the future Candidate-Interview Status in the Candidates module.
- Backward compat : Additive-in-spirit — only relabels rows the new app can no longer reach (status
                    NOT IN open/in_progress/on_hold → on_hold). Untouched: existing open/in_progress/
                    on_hold rows. Also backfills hold_reason where NULL so the relabel satisfies
                    ck_position_hold_reason; the prior lifecycle status stays queryable in
                    position_history status_change rows.
- Migration       : 0009_positions_status_remap; downgrade implemented? no — documented no-op (the
                    pre-remap per-row status cannot be reconstructed; acceptable for a corrective
                    data migration). Data step guarded by context.is_offline_mode().
- Validation      : offline `alembic upgrade --sql` (emits no UPDATE offline, by guard); applied to
                    local dev DB — AICO-001 + STGL-001 (were no_show) → on_hold; ARPL-001 stays open.
- Rollback        : forward-only data correction; revert the app-layer restriction in code if needed.
- Notes           : Pairs with the app-level SettableStatus Literal (rejects the parked values with
                    422 on create + PATCH /status). Applied to local dev DB on 2026-06-16 (head =
                    0009_positions_status_remap).

### [2026-06-16] JD extraction provenance column — 0008_jd_extraction_provider
- Baseline        : v2.2 (11-Jun-2026)
- Author          : Claude Code (opus-4-8) for hareesh@stg.com
- Trigger         : defect fix — updated feature (Positions, round-2: honest JD extraction badge).
                    Change: openspec/changes/positions-defects-round2. Branch: dev/15.3-positions-defects-r2.
- Module(s)       : positions
- Change type     : add column (1).
- Objects         : job_descriptions.extraction_provider VARCHAR(64) NULL.
- Storage decision: REAL COLUMN (not metadata JSONB). A small, read-on-every-JD-view scalar attribute
                    of the JD record (sits beside extraction_status / extraction_error) recording the
                    REAL provider that produced the extraction ("local_nlp" or "anthropic:<model>"),
                    so the panel labels content truthfully instead of a hardcoded model name —
                    a typed column is the cheapest correct mechanism (not EAV/JSONB).
- Backward compat : Purely additive. NULLable; existing rows + in-flight extractions unaffected
                    (running code coalesces NULL → honest offline/no-provenance label). Column inherits
                    the existing table-level GRANTs (ADD COLUMN needs no new grant). No enum mutation.
- Migration       : 0008_jd_extraction_provider; downgrade implemented? yes (drops the column).
- Validation      : offline `alembic upgrade --sql` / `downgrade --sql` (clean additive ADD COLUMN /
                    DROP COLUMN); applied to local dev DB.
- Rollback        : `alembic downgrade 0007_org_position_sequences` (drops the column; no data loss
                    elsewhere — provider is purely informational).
- Notes           : Set on both completed AND failed extraction paths. Frontend jd-panel.tsx derives
                    the badge from this field; the hardcoded "claude-sonnet-4-6" claim is removed.
                    Applied to local dev DB on 2026-06-16 (head = 0009_positions_status_remap).

### [2026-06-16] Per-org position-code sequence table — 0007_org_position_sequences
- Baseline        : v2.2 (11-Jun-2026)
- Author          : Claude Code (opus-4-8) for hareesh@stg.com
- Trigger         : defect fix — updated feature (Positions, principal-reviewer blockers MAJ-1 + MAJ-3
                    on the position-code feature). Branch: dev/15.2-positions-defects.
- Module(s)       : positions + organizations (shared code/sequence)
- Change type     : add table (1) + add constraint (1 UNIQUE) + add RLS policy (1) + ats_app grant.
                    Also: organizations.position_code_prefix / position_seq marked DEPRECATED
                    (left in place, UNUSED; physical drop deferred to a later contract release).
- Objects         : table org_position_sequences (organization_id UUID PK → organizations(id),
                    prefix VARCHAR(8) NOT NULL, next_seq INTEGER NOT NULL DEFAULT 0);
                    constraint uq_org_position_sequences_prefix UNIQUE(prefix);
                    policy rls_org_position_sequences_isolation
                    (fn_is_internal() OR organization_id = fn_current_org())
- Storage decision: REAL TABLE (not metadata JSONB). The counter is incremented atomically per org
                    (UPDATE ... RETURNING under the row lock) and the prefix is a hot, UNIQUE business
                    identifier — neither is possible with JSONB. The table deliberately carries NO
                    version/updated_at column and NO trigger: that decoupling is the whole point —
                    incrementing the counter must NOT bump organizations.version (MAJ-1). UNIQUE(prefix)
                    is the cross-org code-space backstop (MAJ-3).
- Backward compat : Purely additive. New table only; backfilled one row per existing org from
                    organizations.position_code_prefix (or derived, collision-suffixed when NULL) +
                    organizations.position_seq. No column dropped/renamed/retyped, no enum mutated. The
                    deprecated organizations columns stay populated so a downgrade/rollback to 0006 code
                    keeps working (expand→contract: switch reads/writes now, drop the columns later).
- Migration       : 0007_org_position_sequences; downgrade implemented? yes (drops policy + table; the
                    organizations columns are untouched). Data backfill guarded by
                    context.is_offline_mode() (offline `--sql` emits DDL only).
- Validation      : offline `alembic upgrade --sql` / `downgrade --sql`; backfill prefix derivation
                    mirrors app.modules.positions.position_code (unit-tested). Applied to local dev/test
                    DB via DATABASE_ADMIN_URL for the integration suite — see Notes.
- Rollback        : `alembic downgrade 0006_position_code` (drops the policy + table; the deprecated
                    organizations.position_code_prefix / position_seq columns remain and are still
                    populated, so 0006-era allocation code resumes cleanly).
- Notes           : Defect MAJ-1 (spurious org version bumps / 409s) is fixed because allocation no
                    longer UPDATEs organizations. MAJ-3 (cross-org prefix collision → 500) is fixed by
                    the UNIQUE(prefix) backstop plus derive-next-unique-prefix retry at row seeding.
                    Applied to local dev DB on 2026-06-16 (`alembic upgrade head` → head is now
                    0007_org_position_sequences); backfilled one sequence row per existing org.

### [2026-06-16] Human-readable position code — 0006_position_code
- Baseline        : v2.2 (11-Jun-2026)
- Author          : Claude Code (opus-4-8) for hareesh@stg.com
- Trigger         : defect fix — updated feature (Positions, defect F7). Change:
                    openspec/changes/positions-defect-fixes (specs/positions/spec.md ADDED
                    "Human-readable position code").
- Module(s)       : positions + organizations (shared code/sequence)
- Change type     : add column (3) + add index (1, partial unique)
- Objects         : organizations.position_code_prefix VARCHAR(8) NULL,
                    organizations.position_seq INTEGER NOT NULL DEFAULT 0,
                    positions.position_code VARCHAR(16) NULL,
                    index uq_positions_code (positions.position_code)
                    WHERE position_code IS NOT NULL AND deleted_at IS NULL
- Storage decision: REAL COLUMNS (not metadata JSONB). The code is a hot-queried, indexed,
                    UNIQUE business identifier and the per-org counter must increment atomically
                    (UPDATE ... RETURNING) — neither is possible with JSONB. Prefix + counter live
                    on organizations; the formatted code lives on positions.
- Backward compat : Purely additive. New columns are NULLable or carry DEFAULT 0; no column is
                    dropped/renamed/retyped, no enum mutated. Existing rows are backfilled with
                    stable codes (per-org, ordered by created_at); running Phase-15 code that does
                    not read position_code is unaffected.
- Migration       : 0006_position_code; downgrade implemented? yes (drops index + 3 columns).
- Validation      : offline `alembic upgrade --sql` / `downgrade --sql`; prefix derivation mirrors
                    app.modules.positions.position_code (unit-tested: 'STG Labs'→'STGL',
                    'Acme Robotics Pvt Ltd'→'ARPL').
- Rollback        : `alembic downgrade 0005_interview_kit_scorecard` (drops the index + columns;
                    no data loss elsewhere).
- Notes           : Code is immutable once assigned (never set on update). APPLIED to the local dev
                    DB on 2026-06-16 (no production/shared DB exists yet). SUPERSEDED IN PART by
                    0007_org_position_sequences: the per-org counter + prefix moved off the
                    `organizations` row into a dedicated `org_position_sequences` table (the
                    organizations.position_seq/position_code_prefix columns remain but are
                    deprecated/unused — physical drop deferred per expand→contract). See 0007.

### [2026-06-16] AI Interview Kit + online Scorecard — 0005_interview_kit_scorecard
- Baseline        : v2.2 (11-Jun-2026)
- Author          : Claude Code (opus-4-8) for hareesh@stg.com
- Trigger         : new feature — Phase 15.1 AI Interview Kit + Scorecard. Spec:
                    openspec/changes/positions-ai-interview-kit-scorecard (design.md §2,
                    specs/positions/spec.md ADDED requirements).
- Module(s)       : positions (interview_kit + scorecard sub-feature)
- Change type     : add enum (3) + add table (3) + add index + add constraint + add trigger +
                    add RLS policy + grants
- Objects         : enums interview_kit_status_enum, scorecard_status_enum,
                    scorecard_focus_category_enum; tables interview_kits, scorecards,
                    scorecard_entries; idx uq_interview_kits_position, ix_scorecards_position,
                    ix_scorecard_entries_scorecard; CHECK ck_scorecard_entry_expected_1_5 /
                    _actual_1_5; triggers trg_interview_kits_upd / trg_scorecards_upd;
                    RLS rls_interview_kits_isolation / rls_scorecards_isolation /
                    rls_scorecard_entries_isolation.
- Storage decision: kit rubric + screening questions + technical focus-area Q&A → JSONB on
                    interview_kits (document-like; read/written as one unit, never filtered
                    individually — cheapest safe mechanism per SCHEMA_EVOLUTION decision tree).
                    REAL TABLE scorecard_entries with indexed smallint scores + CHECK 1..5 —
                    scores are queried/aggregated (expected-vs-actual delta, reporting).
                    organization_id denormalised onto the two parent tables so RLS reuses the
                    positions pattern verbatim (indexable; no per-row subquery on the hot path).
- Backward compat : NEW objects only; no existing table/column/enum altered. Running Phase-15
                    code is unaffected. New tables are additive.
- Migration       : 0005_interview_kit_scorecard; downgrade implemented? yes (drops policies,
                    triggers, indexes, tables children→parents, then the 3 enums).
- Validation      : offline `alembic upgrade 0004:0005 --sql` generates clean DDL (3 TYPE, 3
                    TABLE, indexes, CHECKs, triggers, RLS, grants). APPLIED to the LOCAL dev DB on
                    2026-06-16 (human-approved; owner via DATABASE_ADMIN_URL) — tables/enums/RLS/
                    grants verified present; integration flow (generate→score→submit→export) green.
                    No production environment exists yet.
- Rollback        : `alembic downgrade 0004_positions_noshow_budget` (reversible).
- Notes           : scorecard captures candidate name + YoE as free entry; a candidate FK is a
                    documented follow-up for Phase 16. scorecard_entries has no own version —
                    optimistic concurrency is enforced at the scorecard aggregate (PATCH w/ version).

### [2026-06-15] v2.2 Positions subset APPLIED via Alembic — no-show + budget/currency — 0004_positions_noshow_budget
- Baseline        : v2.2 (11-Jun-2026)
- Author          : Claude Code (backend-engineer) for hareesh@stg.com
- Trigger         : new module — Phase 15 Positions/Requisitions implementation. Ships the
                    formal, reversible Alembic revision for the POSITIONS SUBSET of the v2.2
                    deltas that the 2026-06-12 entry drafted into docs/schema.sql "no Alembic yet".
                    Spec: openspec/specs/positions/spec.md §9/§10; change:
                    openspec/changes/implement-positions-requisitions (design.md "Migration Plan").
- Module(s)       : positions (+ shared `currencies` reference table)
- Change type     : add enum value + add enum + add table + add column + add constraint +
                    add index + add FK + seed data + GRANT
- Objects         :
                    - ENUM position_status_enum: + 'no_show' (ALTER TYPE ADD VALUE IF NOT EXISTS,
                      run IN-TRANSACTION on PG 12+; ck_position_noshow_reason compares status::text
                      so the new value is not used in the adding txn; expand-only — see rollback caveat).
                    - ENUM position_no_show_reason_enum (NEW): better_offer / better_brand /
                      remote_100 / retained_current / personal / others.
                    - TABLE currencies (NEW): code CHAR(3) PK, symbol, english_name, is_active —
                      seeded with the 15 ISO-4217 majors (symbols read VERBATIM from docs/schema.sql).
                    - positions: + no_show_reason, no_show_org_name, no_show_offer_amount,
                      no_show_offer_currency, no_show_details_not_shared (DEFAULT FALSE),
                      no_show_other_reason, budget_base_currency, inr_to_base_rate,
                      usd_to_base_rate, min_salary_inr, max_salary_inr, min_salary_base,
                      max_salary_base, avg_salary_inr, avg_salary_base, ga_load_base,
                      annual_loaded_cost_base, rate_snapshot_at.
                    - CHECKs: ck_position_noshow_reason, ck_position_noshow_better_offer,
                      ck_position_noshow_better_brand, ck_position_noshow_others,
                      ck_position_salary_range (all matching docs/schema.sql verbatim).
                    - FKs: fk_positions_base_currency, fk_positions_noshow_currency → currencies(code).
                    - INDEX uq_positions_org_title (organization_id, LOWER(title)) WHERE
                      deleted_at IS NULL — the BR-001 dedup source of truth (was ABSENT in the
                      baseline schema, ADDED here additively, design.md D9).
                    - GRANT SELECT/INSERT/UPDATE/DELETE ON currencies TO ats_app (the app runs as
                      the non-owner role; existing positions columns are covered by table-level grants).
- Storage decision: REAL COLUMNS + REAL TABLE (docs/SCHEMA_EVOLUTION.md tree). The no-show reason
                    and budget/currency fields are hot-queried, indexed, relational (FK → currencies)
                    and drive reporting/offer-cost math — not sparse metadata JSONB. currencies is a
                    small relational reference (FK target). Budget constants (G&A USD 5000, loaded
                    multiplier 1.17) live in tenant_settings with spec-default fallbacks (BR-041) —
                    no schema change for those.
- Backward compat : ADDITIVE. New enum value, columns, table, constraints, FKs, index are all
                    additive; new positions columns are NULLable or carry safe defaults
                    (no_show_details_not_shared DEFAULT FALSE), so existing rows and running code
                    are unaffected. The dedup index is partial (only non-deleted rows) and additive.
                    NOT in this revision (ship with their owning phases): offer_details,
                    candidate_source_enum redefinition, candidate_source_details, AI-screening tables.
- Migration       : 0004_positions_noshow_budget (revises 0003_idem_uq_nulls_not_distinct);
                    downgrade implemented (drops FKs, CHECKs, columns, dedup index, currencies +
                    its GRANT, and the reason enum). APPLIED to the local dev DB on 2026-06-15
                    (human-approved) as the owner via DATABASE_ADMIN_URL; the app runs as ats_app.
                    alembic_version = 0004_positions_noshow_budget (head).
- Validation      : Generated offline SQL via `alembic upgrade --sql 0003:0004` and
                    `downgrade --sql 0004:0003` (both exit 0); confirmed additive-only ALTERs and a
                    clean reversal. Currency symbols verified UTF-8 in the emitted INSERTs
                    (₹ € £ ¥ …). APPLIED & verified on the local dev DB 2026-06-15:
                    position_status_enum has 'no_show'; position_no_show_reason_enum has 6 values;
                    currencies = 15 (INR symbol U+20B9 ₹); 18 new positions columns; 5 CHECKs present;
                    uq_positions_org_title present; ats_app granted SELECT/INSERT/UPDATE/DELETE on currencies.
- Rollback        : `alembic downgrade 0003_idem_uq_nulls_not_distinct` removes everything
                    EXCEPT the position_status_enum 'no_show' value — Postgres cannot drop an enum
                    value in place, so it is left (expand-only; inert if unused). Documented in the
                    revision's downgrade() docstring.
- Notes           : ALTER TYPE ADD VALUE runs IN-TRANSACTION on PG 12+. An initial AUTOCOMMIT
                    approach failed (alembic had already begun the migration txn, so isolation_level
                    could not be switched) — fixed by adding the value in-txn and using status::text
                    in ck_position_noshow_reason so the new value is never used before commit. The
                    LOCAL JD storage backend (shared/storage.py) and the missing ANTHROPIC_API_KEY
                    degradation are app-level only — no schema impact. Links: this revision +
                    the implement-positions-requisitions OpenSpec change.

### [2026-06-12] v2.2 schema additions — Candidate Source + Positions (no-show, budget/currency) — schema.sql (no Alembic yet)
- Baseline        : v2.2 (11-Jun-2026)
- Author          : Claude Code (backend) for hareesh@stg.com
- Trigger         : updated feature/module scope — Candidate + Positions module scope
                    addendum (11-Jun-2026 spec). Candidate Source classification
                    (Direct / Employee Referral / 3rd-Party Vendor) with conditional
                    sourcing details; Position no-show capture and a multi-currency
                    budget/offer-cost model. Source schema:
                    ATS_Database_Schema_11-Jun-2026.sql (v2.2 baseline package).
- Module(s)       : positions, candidates (also a shared `currencies` reference table)
- Change type     : add enum value + add enum + redefine enum + add column +
                    add constraint + add table + add index + add trigger + add FK + seed data
- Objects         :
                    - ENUM position_status_enum: + 'no_show' (final value; additive ADD VALUE).
                    - ENUM position_no_show_reason_enum (NEW): better_offer / better_brand /
                      remote_100 / retained_current / personal / others.
                    - ENUM candidate_source_enum: REDEFINED to (direct, employee_referral,
                      third_party_vendor) — supersedes the old 9-value set (see caveat below).
                    - positions: + no_show_reason, no_show_org_name, no_show_offer_amount,
                      no_show_offer_currency, no_show_details_not_shared, no_show_other_reason,
                      budget_base_currency, inr_to_base_rate, usd_to_base_rate, min_salary_inr,
                      max_salary_inr, min_salary_base, max_salary_base, avg_salary_inr,
                      avg_salary_base, ga_load_base, annual_loaded_cost_base, rate_snapshot_at;
                      + CHECKs ck_position_noshow_reason, ck_position_noshow_better_offer,
                      ck_position_noshow_better_brand, ck_position_noshow_others,
                      ck_position_salary_range; + FKs fk_positions_base_currency,
                      fk_positions_noshow_currency → currencies(code).
                    - TABLE currencies (NEW) + ISO-4217 seed (15 majors, real symbols).
                    - TABLE candidate_source_details (NEW, 1:1 candidate; encrypted referrer
                      PII enc+hash; vendor_fee_pct DEFAULT 8.33; ck_csd_referral, ck_csd_vendor).
                    - TABLE offer_details (NEW, 1:1 application; per-offer One-Time cost in base
                      currency; FK offer_currency → currencies(code)).
                    - lookup_values seed (category candidate_source): replaced old
                      linkedin/referral/job_board/direct/agency/campus with the v2.2 set
                      direct / employee_referral / third_party_vendor.
- Storage decision: REAL COLUMNS + REAL TABLES (per docs/SCHEMA_EVOLUTION.md decision tree).
                    The no-show reason and budget/currency fields are hot-queried, indexed,
                    relational (FK → currencies) and drive reporting/offer-cost math, so they
                    are not sparse metadata JSONB. currencies is a small relational reference
                    table (FK target). candidate_source_details and offer_details are 1:1
                    relational extensions holding conditional, partly-encrypted PII —
                    not JSONB. lookup_values continues to back the source dropdown labels.
- Backward compat : ADDITIVE. New enum value, columns, tables, constraints, FKs are all
                    additive; new positions columns are NULLable or carry safe defaults
                    (no_show_details_not_shared DEFAULT FALSE), so existing rows and the
                    running code are unaffected. The live-DB deltas already in place are
                    PRESERVED unchanged in schema.sql: users.mfa_channel + users.mobile
                    (mig 0002), uq_idempotency_key_user … NULLS NOT DISTINCT (mig 0003), and
                    positions.approved_at + ck_position_hold_reason. CAVEAT — the
                    candidate_source_enum redefinition is the one NON-additive change; it is
                    intended (v2.2 candidate spec §10.1 supersedes the old free-form sources)
                    and SAFE today because nothing uses it yet (candidates module is a stub).
                    The live-DB transition (drop/recreate the enum, or add/backfill/switch)
                    is handled by the Phase-16 candidates migration, NOT this schema.sql edit.
- Migration       : NONE applied. This is a docs/schema.sql (v2.2 baseline) edit only — the
                    live DB is NOT touched. The reversible Alembic revisions ship with the
                    Phase-15 positions module build (no_show + budget/currency columns,
                    constraints, currencies, offer_details, FKs) and the Phase-16 candidates
                    module build (candidate_source_enum transition + candidate_source_details).
                    Downgrade will be implemented in those revisions.
- Validation      : Schema-source fidelity check — additive deltas pulled verbatim from
                    ATS_Database_Schema_11-Jun-2026.sql (currency symbols read from the UTF-8
                    source, never hand-transcribed). Diff confirmed additive (only deletions:
                    candidate_source_enum value list, candidate_source lookup seed, footer
                    counts). Not yet exercised against a live DB (no migration this change).
- Rollback        : Revert the docs/schema.sql edit (git). No DB rollback needed — nothing
                    was applied to the live database.
- Notes           : DEFERRED (explicitly out of scope here): the AI-screening tables
                    (screening_runs, screening_cache, technical_scorecards, screening_exemplars,
                    screening_feedback), pgvector, and candidate_position_matches embedding/
                    dimension columns. Those ship with the AI-Screening-engine build
                    (candidate spec §9.12), gated by the `ai_screening_engine` feature flag.

### [2026-06-10] uq_idempotency_key_user → NULLS NOT DISTINCT — 0003_idem_uq_nulls_not_distinct
- Baseline        : v2.0 (08-Jun-2026) — post-baseline additive change
- Author          : Claude Code (backend) for hareesh@stg.com
- Trigger         : code change / new platform control — wiring the Idempotency-Key
                    middleware end-to-end (security/platform foundation). Anonymous
                    mutating requests (e.g. /auth/login) key with user_id=NULL; the index
                    must de-duplicate those too. See openspec/changes/implement-security-module/
                    (design.md D6) and app/core/middleware.py IdempotencyMiddleware.
- Module(s)       : security / platform-core
- Change type     : alter index (tighten NULL handling)
- Objects         : idempotency_keys — UNIQUE INDEX uq_idempotency_key_user
                    (idempotency_key, user_id) recreated with NULLS NOT DISTINCT
- Storage decision: INDEX TIGHTENING (no table/column change). The data lives in the
                    existing idempotency_keys table; only the unique index's NULL semantics
                    change so the de-dup invariant holds for anonymous (NULL user_id) keys.
- Backward compat : Additive index change — the indexed columns are unchanged; authenticated
                    rows (non-NULL user_id) are unaffected. NULLS NOT DISTINCT only collapses
                    anonymous duplicate keys that the de-dup logic already treats as the same
                    key (idempotency._reserve catches the resulting IntegrityError → 409
                    REQUEST_IN_PROGRESS, never a 500). Existing grants cover the index.
- Migration       : Alembic revision 0003_idem_uq_nulls_not_distinct
                    (revises 0002_users_mfa_channel_mobile). Downgrade implemented? YES —
                    recreates the plain (NULLS DISTINCT) unique index.
- Validation      : APPLIED to the live DB on 2026-06-10 as the owner (DATABASE_ADMIN_URL);
                    the app runs as ats_app. Verified pg_index.indnullsnotdistinct = true on
                    uq_idempotency_key_user and alembic_version = 0003_idem_uq_nulls_not_distinct.
                    The RUN_DB_TESTS + RUN_0003_TESTS-gated concurrency test (two anonymous
                    requests, same key → exactly one reservation) passes.
- Rollback        : `alembic downgrade 0002_users_mfa_channel_mobile` (restores the plain
                    unique index) and revert the docs/schema.sql index edit.
- Notes           : Pairs with idempotency._reserve's IntegrityError→RequestInProgressError
                    guard (the concurrent first-request race surfaces as 409, not 500).
                    Revision id is <=32 chars by design — Alembic's default
                    alembic_version.version_num is VARCHAR(32); the first-drafted 42-char id
                    failed the post-upgrade bookkeeping UPDATE and was shortened.

### [2026-06-10] users.mfa_channel + users.mobile (SMS/Email OTP MFA) — 0002_users_mfa_channel_mobile
- Baseline        : v2.0 (08-Jun-2026) — post-baseline additive change
- Author          : Claude Code (backend-engineer) for hareesh@stg.com
- Trigger         : new module — security module implementation + MFA reconciliation.
                    The U0 review (2026-06-10) changed MFA from TOTP to SMS/Email OTP,
                    so users needs an MFA channel + a delivery target. See
                    openspec/changes/implement-security-module/ (proposal.md, design.md D2/D3)
                    and openspec/specs/security/spec.md §2 (reconciled TOTP → SMS/Email OTP).
- Module(s)       : security
- Change type     : add column + add constraint
- Objects         : users.mfa_channel (VARCHAR(10), NULLABLE),
                    users.mobile (VARCHAR(30), NULLABLE),
                    CHECK ck_users_mfa_channel (mfa_channel IS NULL OR IN ('sms','email'))
- Storage decision: REAL COLUMNS. Per docs/SCHEMA_EVOLUTION.md decision tree — mfa_channel
                    and mobile are read on the hot login path (every authenticating user)
                    and are core identity attributes, not sparse optional metadata, so real
                    columns are correct (not users.metadata JSONB). The OTP itself is NEVER
                    persisted — only its hash lives ephemerally in Redis (TTL 5 min).
- Backward compat : Additive + NULLABLE — existing rows get NULL; the running app does not
                    yet require these columns. The TOTP-era mfa_secret_enc column is LEFT IN
                    PLACE this release (expand→contract; dropped in a later release). Existing
                    table-level grants to ats_app cover the new columns.
- Migration       : Alembic revision 0002_users_mfa_channel_mobile (revises 0001_baseline).
                    Downgrade implemented? YES — drops the CHECK then both columns. Baseline
                    revision 0001_baseline stamps the existing schema.sql state (no DDL).
- Validation      : Offline SQL generated and reviewed via `alembic upgrade --sql`
                    (additive ALTERs only). NOT YET APPLIED to the live DB — pending human
                    approval. Apply as the owner (DATABASE_ADMIN_URL); the app runs as ats_app.
- Rollback        : `alembic downgrade 0001_baseline` (drops ck_users_mfa_channel, mobile,
                    mfa_channel) and revert the docs/schema.sql users-table edit.
- Notes           : This is the FIRST Alembic-managed delta. From here Alembic owns schema
                    deltas (until now schema came from docs/schema.sql + direct DDL).

### [2026-06-09] positions.approved_at (Position Approved Date) + per-tranche TAT — positions
- Baseline        : v2.0 (08-Jun-2026) — post-baseline additive change
- Author          : Claude Code (backend assist) for hareesh@stg.com
- Trigger         : spec change — Positions spec updated to add Position Approved Date,
                    make Position Creation Date explicit, and define per-tranche
                    Turnaround Time (TAT). See openspec/specs/positions/spec.md §2A, BR-011..013.
- Module(s)       : positions
- Change type     : add column
- Objects         : positions.approved_at (TIMESTAMPTZ, NULLABLE)
- Storage decision: REAL COLUMN. Per docs/SCHEMA_EVOLUTION.md decision tree — approved_at
                    is a queried date driving TAT reporting (not a sparse optional
                    attribute), so a real column is correct, not metadata JSONB.
- Backward compat : Additive + NULLABLE — existing rows get NULL; no running code depends
                    on it yet. Existing table-level grants to ats_app cover the new column.
                    Position Creation Date reuses existing created_at (NO new column).
                    Count-change date/time capture is unchanged (already provided by
                    position_history + fn_track_position_count_change → changed_at).
- Migration       : Applied via (a) docs/schema.sql update and (b) direct
                    `ALTER TABLE positions ADD COLUMN approved_at TIMESTAMPTZ;` on the local
                    DB. Formal Alembic revision DEFERRED — the baseline is managed via
                    schema.sql and ORM models are still stubs, so Alembic is not yet
                    baselined; add the revision when the positions module/models exist.
- Validation      : Column verified present on positions; app role ats_app can read it via
                    existing table-level grants.
- Rollback        : `ALTER TABLE positions DROP COLUMN approved_at;` and revert schema.sql.
- Notes           : TAT is computed per count tranche — initial tranche from approved_at;
                    each count increase from its position_history.changed_at. Worked
                    example in the spec (10 @ 01-May, +10 @ 20-May → two tranches).

### [2026-06-08] Initial enterprise schema — baseline
- Baseline        : v2.0 (08-Jun-2026)
- Author          : STG Labs (baseline package)
- Trigger         : baseline (source: ATS_requirement_v2_0_08-Jun-2026.docx)
- Module(s)       : all (security, organizations, departments, positions,
                    candidates, applications, screening, interviews, consent,
                    data-privacy, reporting)
- Change type     : initial schema (establishes the baseline; not an Alembic delta)
- Objects         : 36 base tables (+ 6 monthly partition children), 17 enum
                    types, 4 materialized views, 6 RLS policies, 5 trigger
                    functions. See `docs/schema.sql` for the authoritative DDL.
- Storage decision: N/A (baseline). Extension points are built in: metadata JSONB
                    on core entities, lookup_values, custom_field_definitions,
                    tags, tenant_settings, feature_flags — use these per
                    `docs/SCHEMA_EVOLUTION.md` before adding real columns.
- Backward compat : N/A — this is the starting point all later changes extend
                    additively without breaking.
- Migration       : established via `docs/schema.sql` (the v2.0 baseline). From
                    here on, every change is one reversible Alembic revision.
- Validation      : baseline loads cleanly on PostgreSQL 16 (see GETTING_STARTED
                    Phase D).
- Rollback        : N/A (baseline).
- Notes           : Compliance scope India DPDP Act 2023 + DPDP Rules 2025. Enum
                    `application_status_enum`, hash-chained `audit_log`, and the
                    transactional `outbox_events` table are load-bearing for later
                    changes — extend, do not mutate in place.

<!-- Add new entries ABOVE this line, newest first, using the template. -->
