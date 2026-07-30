**Artifact Version 2.3 — Baselined 04-Jul-2026**  ·  Source requirements: ATS_requirement_v2_0_08-Jun-2026.docx (+ 11-Jun-2026 No-Show & Budget/Multi-Currency addendum)  ·  Versioning rule: docs/VERSIONING.md

> Change log: v2.2 (11-Jun-2026) APPENDS §9 (No-Show status + mandatory reason capture) and
> §10 (Position Budget & Multi-Currency cost model). §§1–8 — including the v2.0+ §2A /
> BR-011..013 / REPORT-3 per-tranche TAT and `approved_at` — are unchanged and preserved.
> Change set 2026-06-24: approved_at is now REQUIRED on create (no default). Added `closed`
> status (system-set only via BR-050 auto-close). New list filters: title ILIKE,
> ageing_bucket, organization_name ILIKE, recruiter_id. ageing_days computed field on
> PositionSummary. RecruiterAssignmentResponse adds filled_count + in_progress_count.
> Change set P27 2026-07-04: (1) interview_levels.level_category ENUM('stg_labs','organization')
> added — required when configuring interview levels; (2) position auto-close now triggered by
> application status → 'onboarded' (in addition to existing 'hired' trigger via BR-050);
> (3) §1A updated to reflect interview kit ownership by interviews module.
> Change set 2026-07-22 (interview-status-lifecycle-phaseb, pulled forward from the
> originally-planned Phase C scope): `POST /api/v1/positions/{id}/interview-levels`
> gains new BR-060 config-time ordering validation — Org L1/L2 mandatory in every
> submitted set, STG L2 requires STG L1 present, Org L3-L6 gap-free if present.
> Previously no ordering validation existed at all (any combination was accepted).
> Same full-replace endpoint handles both initial setup and later additions — no new
> create-vs-edit distinction needed. See interviews/spec.md's BR-SEQ-001 (the
> corresponding usage-time sequencing gate this config-time validation feeds) and
> openspec/changes/interview-status-lifecycle-phaseb/design.md D9.

---

# Position Module Specification
# OpenSpec living spec — openspec/specs/positions/spec.md
# ATS Platform · STG Labs · Bengaluru
#
# Last updated : [DATE OF FIRST IMPLEMENTATION]
# Status       : Ready for implementation
# ─────────────────────────────────────────────────────────────


## 1. Purpose

Manages open positions (roles to be filled) for STG Labs portfolio companies.
A Position belongs to an Organisation and optionally a Department.
Supports JD upload with AI extraction, count management with full history,
and interview level setup. Position-level panelist CRUD is deferred to the
Interviews module (Phase 18/20).

## 1A. Module ownership boundary (interview kit)

Interview kit generation is exclusively owned by the `interviews/` module (per-level kit,
Phase 20/20B — `interview_level_kits` table). Position-level interview kits (previously
`interview_kits` table, Phase 15.1) have been removed from this module entirely.

This module retains: JD upload + extraction, interview level configuration, recruiter
assignment, no-show capture, and budget management. It does NOT generate or store
interview kits of any kind.


## 2. AI Agent — JDExtractorAgent

Name    : JDExtractorAgent
Type    : Celery background task
Provider: PLUGGABLE via `JD_EXTRACTION_PROVIDER` (implementation note, v2.2) —
          'local_nlp' (DEFAULT: offline gazetteer + section parser, no API key/network) or
          'anthropic' (claude-sonnet-4-6 via Anthropic API — requires ANTHROPIC_API_KEY) or
          'bedrock' (claude-sonnet-4-6 via AWS Bedrock bedrock-runtime, IAM role — no key on
          ECS Fargate; enable via BEDROCK_MODEL_ID + BEDROCK_REGION). local_nlp is the default so
          JD extraction works with zero external approvals; switch to anthropic/bedrock when a
          key or IAM role is available (config flip, no code change).
Trigger : Fired after every JD file upload (single version)

Extracted fields (persisted to job_descriptions table):
  primary_skills          JSONB   — array of primary skill strings
  secondary_skills        JSONB   — array of secondary skill strings
  good_to_have_skills     JSONB   — array of good-to-have skill strings
  roles_responsibilities  TEXT    — full paragraph of roles & responsibilities

Agent prompt contract:
  - Pass full extracted text of the JD (pdfplumber for PDF, python-docx for DOCX)
  - Return strict JSON with exact field names above
  - Return null for fields not found — never invent content
  - Retry 3 times on API failure with exponential backoff (1s, 2s, 4s)
  - On all retries exhausted: extraction_status = "failed", log ERROR

Position AI Requirement (from requirements doc):
  - Every time a position is created: capture Date & Time of creation
    → position_history record with change_type = 'created'
  - Every time position count changes: capture Date & Time + direction
    → position_history record with change_type = 'count_increase' or 'count_decrease'
    → DB trigger fn_track_position_count_change() handles this automatically
  - Every time JD changes: capture Date & Time + new JD file reference
    → position_history record with change_type = 'jd_change'
    → Service layer writes this record after marking new JD as current


## 2A. Position dates, count history & Turnaround Time (TAT)

Two dates anchor a position's lifecycle and reporting:

- **Position Creation Date** — `positions.created_at` (TIMESTAMPTZ, UTC): when the
  position was created in the ATS platform. Also recorded as a position_history row
  with change_type = 'created'. (No separate column — created_at IS the creation date.)
- **Position Approved Date** — `positions.approved_at` (TIMESTAMPTZ, UTC, NULLABLE):
  the date the organisation approved hiring for this position. Optional on create,
  settable later via PATCH. This is the Turnaround-Time anchor for the INITIAL
  position_count.

Position count changes are captured automatically (reiterated): every increase or
decrease writes a position_history row (change_type = 'count_increase' /
'count_decrease') holding the OLD and NEW count and the date & time (changed_at),
via the DB trigger fn_track_position_count_change(). The app does not do this manually.

**Turnaround Time (TAT) is measured per count tranche**, because the count can grow or
shrink over time and each tranche's clock starts when that tranche was approved/opened
— not at the position's original approval:

- The INITIAL position_count tranche's TAT starts at `approved_at`.
- Each position_count INCREASE opens a NEW tranche whose TAT starts at that increase's
  `position_history.changed_at` (the date & time of the change), NOT at approved_at.
- A position_count DECREASE reduces the open count.

Worked example: Position-1 has count 10, approved for hiring on 01-May-2026 → the TAT
for those 10 starts on 01-May-2026. If the count is increased to 20 on 20-May-2026, the
TAT for the additional 10 starts on 20-May-2026 — NOT on 01-May-2026. Reporting
reconstructs tranches from `approved_at` + the position_history count rows.


## 3. API Endpoints

### POST /api/v1/positions
Auth    : Bearer — roles: hr_admin, recruiter
Request :
  {
    "organization_id"     : "uuid (required)",
    "department_id"       : "uuid (optional)",
    "title"               : "string (required, min 3 chars, max 255)",
    "position_count"      : "integer (required, min 1)",
    "approved_at"         : "ISO8601 (REQUIRED) — org hiring-approval date; TAT anchor for the initial count tranche (Change-1: 2026-06-24)",
    "status"              : "string (optional, default 'open') — see status set below",
    "hold_reason"         : "string (REQUIRED when status = 'on_hold')",
    "hm_name"             : "string (optional)",
    "hm_email"            : "string (optional, valid email)",
    "hm_landline_isd"     : "string (optional)",
    "hm_landline"         : "string (optional)",
    "hm_mobile_isd"       : "string (optional)",
    "hm_mobile"           : "string (optional)",
    "hiring_manager_id"   : "uuid (optional — ATS user as HM)"
  }
Response 201 : PositionResponse
  Also creates: position_history record with change_type = 'created'
Response 400 : VALIDATION_ERROR
Response 404 : ORGANIZATION_NOT_FOUND | DEPARTMENT_NOT_FOUND

### GET /api/v1/positions
Auth    : Bearer — all authenticated roles
Query   :
  organization_id   (uuid, optional)
  department_id     (uuid, optional)
  status            (string, optional) — open / in_progress /
                    portco_confirmed_yet_to_offer / offered / offer_accepted /
                    onboarded / on_hold / closed
  search            (string, optional) — full-text on title
  title             (string, optional, min 1, max 200) — ILIKE partial match on p.title (Change-3)
  ageing_bucket     (string, optional) — one of: lte_7 / 8_to_15 / 16_to_30 / 31_to_45 /
                    46_to_90 / 91_to_120 / over_120 — filters on approved_at date range (Change-3)
  organization_name (string, optional, min 1, max 200) — ILIKE partial match on org name (Change-3)
  recruiter_id      (uuid, optional) — position has an active assignment for this recruiter (Change-3)
  offset            (int, default 0)
  limit             (int, default 20, max 100)
Response 200 : PagedResponse[PositionSummary]
  PositionSummary includes: id, title, position_code, organization_id, organization_name,
  department_id, department_name, status, position_count, current_jd_id, version, created_at,
  recruiter_assignments (with filled_count + in_progress_count), ageing_days (Change-2)
  ageing_days: int | null — server-computed as (CURRENT_DATE - approved_at::date); null when approved_at is null

### GET /api/v1/positions/{id}
Auth    : Bearer — all authenticated roles
Response 200 : PositionDetailResponse
  Includes: all position fields (incl. created_at = Position Creation Date and
  approved_at = Position Approved Date), interview_levels, current JD summary,
  history count. (panelists deferred to Phase 18/20)
Response 404 : POSITION_NOT_FOUND

### PATCH /api/v1/positions/{id}
Auth    : Bearer — roles: hr_admin, recruiter
Request : Partial — any position field except organization_id
  If position_count changes: position_history record written automatically
  by DB trigger.
Response 200 : PositionDetailResponse
Response 404 : POSITION_NOT_FOUND
Response 409 : INVALID_STATUS_TRANSITION

### PATCH /api/v1/positions/{id}/status
Auth    : Bearer — roles: hr_admin, recruiter
Request :
  {
    "status"       : "open | in_progress | on_hold | closed",
    "version"      : "int (required — optimistic concurrency)",
    "hold_reason"  : "string (REQUIRED when status = 'on_hold')",
    "close_reason" : {
      "code": "onboarded | on_hold_too_long | portco_deferred | portco_withdrew | others"
              " — context-sensitive: open→closed offers {portco_withdrew, onboarded,"
              " on_hold_too_long, others}; on_hold→closed offers {on_hold_too_long,"
              " portco_deferred, others} (updated 2026-07-04, see BR-P24-003)",
      "text": "string (REQUIRED when code = 'others')"
    }  -- REQUIRED for status='closed' from EITHER open or on_hold (updated 2026-07-04:
       -- no longer auto-set for open→closed)
  }
Action  : Updates status with transition guard. Records a position_history row
          (change_type = 'status_change', old/new status) with the timestamp —
          handled automatically by the DB trigger.
  in_progress→closed is BLOCKED manually (409 INVALID_STATUS_TRANSITION).
  Auto-close of in_progress→closed goes via applications.check_position_auto_close
  Celery task (applications/tasks.py::_check_and_close, BR-063) -- NOT the
  positions-module task of a similar name, removed in the NFR Phase 3 dead-code
  sweep (2026-07-24).
Response 200 : PositionDetailResponse (includes close_reason on closed positions)
Response 400 : HOLD_REASON_REQUIRED | CLOSE_REASON_REQUIRED (on_hold→closed only)
Response 409 : INVALID_STATUS_TRANSITION | POSITION_CLOSED

#### v2 Position status set (Position Current Status)
  open                          Position is open for sourcing
  in_progress                   Candidates are progressing through interviews
  portco_confirmed_yet_to_offer Portfolio company confirmed a candidate, no offer yet
  offered                       Offer extended
  offer_accepted                Candidate accepted the offer
  onboarded                     Candidate has joined
  on_hold                       Paused — hold_reason (free text) is MANDATORY
  closed                        All headcounts filled — SYSTEM-SET ONLY via BR-050 (auto-close);
                                not settable via POST/PATCH. A closed position cannot be edited.

### POST /api/v1/positions/{id}/jd
Auth    : Bearer — roles: hr_admin, recruiter
Content : multipart/form-data
  file  : binary (required) — .pdf or .docx, max 10 MB
Response 202 : JDUploadResponse
  {
    "jd_id"       : "uuid",
    "version"     : 2,
    "status"      : "processing",
    "uploaded_at" : "2025-01-15T10:30:00Z"
  }
  Also:
    - Marks previous JD is_current = FALSE
    - Creates new job_description record with is_current = TRUE
    - Enqueues JDExtractorAgent Celery task
    - Writes position_history record: change_type = 'jd_change'
Response 400 : INVALID_FILE_TYPE | FILE_TOO_LARGE

### GET /api/v1/positions/{id}/jd
Auth    : Bearer — all authenticated roles
Response 200 : Current JD details including pre-signed S3 URL (1 hour)
Response 404 : JD_NOT_FOUND

### POST /api/v1/positions/{id}/jd/re-extract
Auth    : Bearer — roles: hr_admin, recruiter
Notes   : Re-runs skill extraction on the current JD without uploading a new file.
          No new JD version is created; only extracted skill fields are refreshed.
          Use after gazetteer updates. Accepted at any time (no status guard).
Response 202 : JDUploadResponse (same schema as POST /jd)
Response 404 : JD_NOT_FOUND

### GET /api/v1/positions/{id}/jd/history
Auth    : Bearer — roles: hr_admin, recruiter
Response 200 : List of all JD versions for this position (oldest to newest)

> **Deferred (Phase 18/20):** POST/GET/DELETE /positions/{id}/panelists,
> BR-004, BR-005, AC-006, AC-007, PanelistRequest, PanelistResponse, PanelistGroups.
> Panelist assignment moves to the Interviews module. The `position_panelists` table
> and `PositionPanelist` ORM model are retained as the FK target for that module.
> `PanelistNotFoundError` is retained in exceptions.py — consumed by interview-levels
> (CR-001) panelist validation.

> **Removed (2026-06-30):** All position-level interview kit endpoints (POST/GET/PATCH
> /positions/{id}/interview-kit, GET /positions/{id}/interview-kit/scorecard-template.xlsx)
> and scorecard endpoints (POST/GET /positions/{id}/scorecards, GET/PATCH/POST
> /scorecards/{id}) have been deleted. The `interview_kits`, `scorecards`, and
> `scorecard_entries` tables are dropped in Alembic 0035. `ScorecardNotFoundError` and
> `ScorecardNotEditableError` are retained in exceptions.py as stubs; `scorecard_models.py`
> and `scorecard_excel.py` are deleted. The per-level kit (interviews module, Phase 20/20B)
> is the sole kit mechanism going forward.

### POST /api/v1/positions/{id}/interview-levels
Auth    : Bearer — roles: hr_admin
Request :
  [
    {
      "level_type"     : "stg_labs | organization",
      "level_number"   : "integer (1-6)",
      "level_label"    : "string — e.g. 'STG Labs Level-1', 'Infosys Level-2'",
      "level_category" : "stg_labs | organization (REQUIRED — P27)",
      "sequence_order" : "integer (1 = first in pipeline)",
      "panelist_id"    : "uuid | null — optional assigned interviewer (CR-001)"
    }
  ]
Response 201 : List[InterviewLevelResponse]
  Creates/replaces the interview level configuration for this position.
  Each level may optionally carry a panelist_id linking to the global panelist master.
  Used when setting up the interview pipeline before candidates are mapped.
  InterviewLevelResponse includes:
    panelist_id    : uuid | null
    panelist_name  : string | null  (populated from eager-loaded InterviewPanelist.name)
    level_category : 'stg_labs' | 'organization'
Errors:
  400 VALIDATION_ERROR   — level_category missing or not one of 'stg_labs'/'organization'
  404 PANELIST_NOT_FOUND — panelist_id does not exist or is soft-deleted
  422 PANELIST_INACTIVE  — referenced panelist exists but is deactivated

**Known gap (spec-sync audit 2026-07-10, tracked as BUG-4 in
test_functional_p27_pos_close_autoclose.py:648):** `InterviewLevelRequest` and
`InterviewLevelResponse` in schemas.py do NOT actually declare a `level_category`
field — the service computes it internally from `level_type` instead of accepting/
returning it as documented above. This is a real code gap versus this spec
(spec is ahead of code here), not stale documentation — do not "fix" by removing
the field from this spec; fix the code to match.

Note (P27): level_category drives two downstream behaviours:
  1. Per-level interview kit generation (interviews module, §10) — kits are generated
     ONLY for levels with level_category = 'stg_labs'. Organization levels do not get
     a kit. The Celery task checks this field and exits silently for 'organization' levels.
  2. Application status auto-sync (interviews module, §13) — the mapping from
     interview outcome to application status enum value is derived from
     level_category + sequence_order. STG levels map to stg_lN_*; org levels to org_lN_*.

> Note: Once assigned, `panelist_id` is not cleared if the panelist is subsequently
> deactivated. The assignment is historical. The FK `ON DELETE SET NULL` applies only
> to physical deletion (prohibited by soft-delete policy).

### GET /api/v1/positions/{id}/interview-levels
Auth    : Bearer — all authenticated roles
Response 200 : List[InterviewLevelResponse] ordered by sequence_order
  Each row includes panelist_id, panelist_name (null when no panelist assigned),
  and level_category ('stg_labs' | 'organization').

### GET /api/v1/positions/{id}/history
Auth    : Bearer — roles: hr_admin, recruiter
Response 200 : List[PositionHistoryResponse]
  All changes: creation, count changes, JD changes, status changes
  Ordered by changed_at DESC


## 4. Business Rules

BR-001  Position title must be unique within an Organisation (case-insensitive)
        among active records. Return 409 DUPLICATE_POSITION_TITLE.
BR-002  position_count must be >= 1. Zero is not allowed.
BR-003  Status set (v2): open, in_progress, portco_confirmed_yet_to_offer,
        offered, offer_accepted, onboarded, on_hold. Default on create: open.
        Typical forward flow:
          open → in_progress → portco_confirmed_yet_to_offer → offered →
          offer_accepted → onboarded
        on_hold may be entered from any active status and exited back to the
        status it came from (or open). The service validates transitions and
        returns 409 INVALID_STATUS_TRANSITION for nonsensical jumps
        (e.g. onboarded → open).
BR-003a On-Hold requires a reason. When status is set to on_hold, hold_reason
        (free text) is MANDATORY. Enforced in the service (400 HOLD_REASON_REQUIRED)
        and at the DB level (ck_position_hold_reason).
BR-003b Every status change is recorded in position_history with the timestamp
        (change_type = 'status_change') via the DB trigger — satisfies the v2
        requirement to capture status-change date & time.
BR-004  [Deferred Phase 18/20] STG Labs panelists: maximum 2 (sequence_number 1 and 2 only).
BR-005  [Deferred Phase 18/20] Organisation panelists: maximum 4 (sequence_number 1 through 4).
BR-006  JD upload: file type must be PDF or DOCX. Max size 10 MB.
        File type validated by MIME inspection (python-magic), not extension.
BR-007  JD versioning: when a new JD is uploaded, the previous JD record
        has is_current set to FALSE. The new JD has is_current = TRUE.
        Previous JD files are retained in S3 (not deleted).
BR-008  Position creation MUST write a position_history record with
        change_type = 'created' and new_value = { "count": N, "status": "open" }.
BR-009  Every mutation produces an AuditLog entry.
BR-010  Soft delete only — deleted_at column. Position with active
        Applications cannot be soft-deleted.
BR-011  Position Creation Date = created_at (timestamp the position is created in the
        ATS). Also recorded as a position_history row (change_type = 'created').
BR-012  Position Approved Date = approved_at (TIMESTAMPTZ, REQUIRED on create as of
        Change-1 2026-06-24): the date the organisation approved hiring for this position.
        It is the TAT start for the INITIAL position_count.
BR-013  Turnaround Time (TAT) is computed PER COUNT TRANCHE, because counts change
        over time:
          • the initial position_count tranche's TAT starts at approved_at;
          • every position_count INCREASE opens a new tranche whose TAT starts at that
            increase's position_history.changed_at (date & time of the change), NOT at
            approved_at;
          • a position_count DECREASE reduces the open count.
        Worked example: Position-1 count = 10, approved 01-May-2026 → those 10 have TAT
        from 01-May-2026. Count raised to 20 on 20-May-2026 → the additional 10 have TAT
        from 20-May-2026 (not 01-May). Source: positions.approved_at + position_history
        count_increase/count_decrease rows (each carries changed_at), captured by
        fn_track_position_count_change().

BR-050  (Change-5, 2026-06-24; revised P27, 2026-07-04; task location corrected
        NFR Phase 3, 2026-07-24) Auto-close: applications.check_position_auto_close
        Celery task (applications/tasks.py::_check_and_close, BR-063) is enqueued
        when an application status transitions to 'onboarded'. The task checks: if
        ALL non-'withdrawn' applications for the position have status = 'onboarded'
        → sets position status = 'closed' (audit action: position_auto_closed,
        actor=system). Idempotent — safe to call multiple times. A 'closed' position
        cannot be edited via PATCH or status-change (409).
        Historical note: an earlier 'hired'-keyed check on this same trigger, and a
        separate positions-module task of a similar name, are both dead as of
        applications-status-lifecycle-phaseA (2026-07-21) and the NFR Phase 3
        dead-code sweep (2026-07-24) respectively — 'onboarded' via the
        applications-module task is the sole live mechanism.

BR-051  (Change-2, 2026-06-24) ageing_days = CURRENT_DATE - approved_at::date, computed
        server-side in the list query. NULL when approved_at IS NULL. Exposed on
        PositionSummary as ageing_days: int | null.

BR-052  (Change-3, 2026-06-24) ageing_bucket filter validation: if the ageing_bucket
        query parameter is supplied but is not one of the 7 allowed values, the service
        raises 422. Allowed: lte_7, 8_to_15, 16_to_30, 31_to_45, 46_to_90, 91_to_120,
        over_120.

BR-053  (Change-7, 2026-06-24) Recruiter breakdown: RecruiterAssignmentResponse includes
        filled_count (count of hired applications where owning_recruiter_id = recruiter_id)
        and in_progress_count (= max(0, assigned_count - filled_count)), both computed via
        a single bulk LEFT JOIN query (no N+1).

BR-060  (Added 2026-07-22, interview-status-lifecycle-phaseb — pulled forward from the
        originally-planned Phase C scope) `POST /api/v1/positions/{id}/interview-levels`
        enforces sequential, gap-free level configuration against the FULL submitted set
        on every call (matches existing full-replace semantics — no new create-vs-edit
        distinction). Previously NO ordering validation existed; levels could be
        submitted in any order/combination.
          - STG L1: optional. STG L2: optional, but if present in the submitted set,
            STG L1 must also be present — else 400 LEVEL_CONFIG_SEQUENCE_GAP.
          - Org L1 and Org L2: MANDATORY — every submission must include both, else 400
            MANDATORY_ORG_LEVEL_MISSING (naming the missing rank).
          - Org L3-L6: optional, may be added in a LATER call to this same endpoint. If
            present, must be gap-free from Org L2 (e.g. Org L4 cannot be present without
            Org L3) — else 400 LEVEL_CONFIG_SEQUENCE_GAP (naming the missing rank).
        This config-time validation is what the interviews module's usage-time
        BR-SEQ-001 sequencing gate (interviews/spec.md) relies on being already true —
        it does not re-validate position-level configuration itself.

BR-061  (Added 2026-07-23, positions-closed-lockdown-phasec) Shared closed-position
        lookup: `PositionService.get_position_status(position_id) -> str | None`
        is a pure read-only method other modules' service layers call before a
        mutation — returns the position's current status, or `None` if the
        position doesn't exist or is soft-deleted. This is the single mechanism
        every consuming module's closed-position guard is built on (never a raw
        cross-module query) — see BR-062 below and the corresponding rules in
        applications/spec.md, interviews/spec.md, offers/spec.md.

BR-062  (Added 2026-07-23) `PATCH /positions/{id}/recruiters` (recruiter
        reassignment) is rejected 409 POSITION_CLOSED when the position's
        status is `closed` — closes the one positions-module-owned gap found
        by a full endpoint-coverage audit (position-self-edit/JD/interview-
        levels were already guarded via the existing `PositionClosedError`
        subresource-write check).

BR-063  (Added 2026-07-23) Auto-close (`applications/tasks.py::_check_and_close`,
        BR-015) now persists a human-readable remark alongside the reason code:
        `close_reason = {"code": "all_onboarded", "text": "Suitable candidates
        for all counts of the position have been identified and onboarded.
        Hence, closed the Position."}` — the exact fixed sentence, not
        parameterized per position. Displayed on the Positions detail page
        below the status badge whenever `close_reason.text` is set (manual
        close's free-text reason displays the same way).


## 5. Acceptance Criteria

AC-001  Create position → 201, position_history 'created' record written
AC-002  Upload JD (PDF) → 202, JDExtractorAgent enqueued, position_history
        'jd_change' record written
AC-003  JD extraction completes → job_descriptions record updated with
        primary_skills, secondary_skills, good_to_have_skills populated
AC-004  Update position_count (increase) → position_history 'count_increase'
        record written automatically by DB trigger
AC-005  Update position_count (decrease) → position_history 'count_decrease'
        record written automatically by DB trigger
AC-006  [Deferred Phase 18/20] Add STG Labs panelist-3 → 409 PANELIST_LIMIT_EXCEEDED
        (max 2 for stg_labs)
AC-007  [Deferred Phase 18/20] Add organisation panelist-5 → 409 PANELIST_LIMIT_EXCEEDED
AC-008  Set status = on_hold without hold_reason → 400 HOLD_REASON_REQUIRED;
        with hold_reason → 200 and a position_history status_change row written
AC-008a Invalid status transition (onboarded → open) → 409
AC-009  Full-text search positions with search="python developer" →
        returns positions whose title matches
AC-010  GET position history → shows created, count changes, JD changes
        in reverse chronological order
AC-011  Create/patch a position with approved_at set → stored; GET returns approved_at
        (Position Approved Date) and created_at (Position Creation Date)
AC-012  TAT tranches: count 10 approved 01-May-2026, raised to 20 on 20-May-2026 →
        derivable that 10 positions have TAT start 01-May and 10 have TAT start 20-May
        (from approved_at + the position_history count_increase changed_at)


## 6. Reporting Requirements

REPORT-1: Organisation >> Position ID >> Position Title >> Count >>
          Date and Time of Creation [Group by Date and Time]
  Query source: mv_position_history_report WHERE change_type = 'created'

REPORT-2: Organisation >> Position ID >> Position Title >> JD ID >>
          JD File >> Date and Time of JD Change [Group by JD change date]
  Query source: mv_jd_change_history

REPORT-3: Turnaround Time (TAT) — Organisation >> Position ID >> Position Title >>
          Tranche (initial / each count change) >> Tranche start date
          (approved_at for the initial count; the count-change changed_at for increases)
          >> Count in tranche >> fill date / current age [TAT per tranche]
  Data source: positions.approved_at + mv_position_history_report (count_increase /
  count_decrease rows). The reporting module computes per-tranche TAT; tranche start
  dates come from approved_at and each count change's changed_at.


## 7. Files to Create

  modules/positions/
    router.py       (all endpoints above)
    service.py      (create, get, list, update, open_position, close_position,
                     upload_jd, set_interview_levels, get_history)
    repository.py   (all DB queries)
    models.py       (Position, JobDescription, PositionPanelist [retained for Phase 18/20 FK],
                     PositionHistory, InterviewLevel —
                     InterviewLevel gains level_category ENUM('stg_labs','organization') NOT NULL
                     added in P27 migration NNNN_interview_level_category)
    schemas.py      (PositionRequest, PositionResponse, PositionDetailResponse,
                     PositionSummary, JDUploadResponse, JDResponse,
                     InterviewLevelRequest, InterviewLevelResponse,
                     PositionHistoryResponse)

  Alembic migration (P27): NNNN_interview_level_category
    — ALTER TABLE interview_levels ADD COLUMN level_category level_category_enum NOT NULL
      DEFAULT 'organization' (default eases backfill; recruiter must correct stg_labs levels).
    — CREATE TYPE level_category_enum AS ENUM('stg_labs','organization').
    — Downgrade: ALTER TABLE interview_levels DROP COLUMN level_category; DROP TYPE.
    — docs/SCHEMA_CHANGE.md entry required.
                    [PanelistRequest, PanelistResponse, PanelistGroups deferred to Phase 18/20]
    exceptions.py   (PositionNotFoundError, DuplicatePositionTitleError,
                     InvalidStatusTransitionError, PanelistNotFoundError [retained — CR-001],
                     JDNotFoundError)
                    [PanelistLimitExceededError, PanelistSlotOccupiedError deferred to Phase 18/20]
    tasks.py        (extract_job_description — JDExtractorAgent)
    agents/
      __init__.py
      jd_extractor.py   (JDExtractorAgent class)
    tests/
      test_service.py
      test_router.py
      test_agents.py

  Alembic migration: create_positions_jd_panelists_history_levels_tables


## 8. Dependencies

  modules/organizations/repository.py  — get_by_id
  modules/departments/repository.py    — get_by_id
  shared/storage.py                    — upload_to_s3, get_presigned_url
  shared/audit.py                      — write_audit_log
  core/config.py                       — ANTHROPIC_API_KEY, AWS_BUCKET_NAME
  anthropic                            — Python SDK
  pdfplumber                           — PDF text extraction
  python-docx                          — DOCX text extraction
  python-magic                         — MIME type validation

# ════════════════════════════════════════════════════════════════════
# 8A. POSITION STATUS ENGINE — Phase 23B   (added 2026-07-02)
# ════════════════════════════════════════════════════════════════════

## 8A.1 Status transition matrix (Phase 23B + P24; open→closed row superseded
## 2026-07-04 by the P24-gaps follow-up — see BR-NEW-002 and §8B.2 BR-P24-003)

| From        | To          | Trigger     | Guard                     | Reason handling                           |
|-------------|-------------|-------------|---------------------------|-------------------------------------------|
| open        | in_progress | AUTO        | none                      | none                                      |
| open        | on_hold     | MANUAL      | none                      | hold_reason mandatory; on_hold_at = now() |
| open        | closed      | MANUAL      | none                      | close_reason mandatory (user-provided;    |
|             |             |             |                           | one of portco_withdrew/onboarded/         |
|             |             |             |                           | on_hold_too_long/others — no auto-set)    |
| in_progress | on_hold     | MANUAL      | none                      | hold_reason mandatory; on_hold_at = now() |
| in_progress | open        | MANUAL      | none (undocumented until  | none — added to spec 2026-07-10, matches  |
|             |             |             | this spec-sync pass)      | validation.py's existing _allowed_targets |
| in_progress | closed      | AUTO only   | MANUAL blocked (409)      | close_reason AUTO-SET to onboarded        |
| on_hold     | open        | MANUAL      | 0 applications (409 else) | on_hold_resolved_at = now()               |
| on_hold     | in_progress | MANUAL      | none                      | on_hold_resolved_at = now()               |
| on_hold     | closed      | MANUAL      | none                      | close_reason mandatory                    |

## 8A.2 New business rules

BR-NEW-001  First application AUTO-transitions position open→in_progress.
            ApplicationService.create_application() calls
            PositionRepository.set_in_progress_if_open() atomically after insert.
            Only fires when the position is still 'open' (idempotent — no-op if already
            in_progress). Audit action: position_status_changed with
            trigger=first_application_created.

BR-NEW-002  Close reason handling (superseded 2026-07-04 by the P24-gaps follow-up —
            see BR-P24-003 in §8B.2 for the current, context-sensitive behavior):
            open→closed: user MUST provide close_reason (portco_withdrew/onboarded/
              on_hold_too_long/others) — no longer auto-set.
            on_hold→closed: user MUST provide close_reason (on_hold_too_long/
              portco_deferred/others).
            close_reason structure: {"code": CloseReasonCode, "text": "..."} — text is
            NULL unless code=others, where text is REQUIRED.
            Service enforces via assert_close_reason(); returns 400 CLOSE_REASON_REQUIRED
            for BOTH the open→closed and on_hold→closed paths.

BR-NEW-003  in_progress→closed is BLOCKED for manual PATCH /status requests.
            assert_transition_allowed() rejects this transition with 409
            INVALID_STATUS_TRANSITION. Auto-close via applications.check_position_auto_close
            Celery task (BR-050, BR-063) is the only permitted path to close an
            in_progress position.

## 8A.3 Schema additions (Phase 23B)

  positions: close_reason JSONB NULL — Alembic 0040_position_close_reason.
    Structure: {"code": CloseReasonCode, "text": str|null}. NULL for non-closed positions.
    Schema-evolution: real column (not metadata_) because close_reason is surfaced in
    PositionResponse and may be queried for audit reporting.

## 8A.4 Acceptance criteria (Phase 23B)

AC-NEW-001  POST /applications with an open position → position transitions to in_progress
            automatically; audit log has position_status_changed with trigger=first_application_created.
AC-NEW-002  PATCH /positions/{id}/status {status:closed, close_reason:{code:portco_deferred}} → 200,
            position.close_reason populated, position.status=closed (on_hold→closed path).
AC-NEW-003  PATCH /positions/{id}/status {status:closed} with no close_reason body (open→closed)
            → 400 CLOSE_REASON_REQUIRED (superseded 2026-07-04 — see AC-P24-005 in §8B.4;
            no longer auto-set).
AC-NEW-004  PATCH /positions/{id}/status {status:closed, close_reason:{code:others}} with no text
            → 422 Validation Error (Pydantic model_validator in CloseReason).
AC-NEW-005  PATCH /positions/{id}/status with in_progress→closed manually → 409
            INVALID_STATUS_TRANSITION.
AC-NEW-006  PATCH /positions/{id}/status {status:closed, close_reason:{code:others, text:"..."}} → 200.

# ════════════════════════════════════════════════════════════════════
# 8B. POSITION STATUS ENGINE — PHASE P24  (added 2026-07-03)
# ════════════════════════════════════════════════════════════════════

## 8B.1 P24 enhancements

Three new behaviors added on top of Phase 23B:

1. **on_hold timestamps** — `on_hold_at TIMESTAMPTZ NULL` and `on_hold_resolved_at TIMESTAMPTZ NULL`
   added to positions. Set atomically by `PositionService._status_changes()`.

2. **open→closed requires user-provided close_reason** (superseded by the P24-gaps follow-up,
   2026-07-04 — see BR-P24-003 below): the service no longer auto-sets `close_reason`; the
   caller must supply one of 4 codes, context-sensitive to the FROM status.

3. **on_hold→open application-count guard** — `PositionRepository.count_applications_for_position()`
   returns the total count; `assert_on_hold_open_requires_zero_apps()` raises 409 if > 0.
   A position with ANY existing application (regardless of status) must return to in_progress,
   not open — open implies no prior candidate activity.

## 8B.2 New business rules (P24)

BR-P24-001  on_hold_at is set to UTC now() when a position transitions TO on_hold
            (from either open or in_progress). on_hold_resolved_at is set to UTC now() when
            transitioning FROM on_hold (to open, in_progress, or closed).

BR-P24-002  on_hold→open is blocked if the position has any applications (total count > 0).
            Returns 409 INVALID_STATUS_TRANSITION. If applications exist, the position
            must return to in_progress instead.

BR-P24-003  portco_withdrew renamed to portco_deferred in CloseReasonCode (updated terminology
            — the client deferred the hire, not withdrew). JSONB data migrated in Alembic 0041.
            **Superseded 2026-07-04 (P24-gaps follow-up):** portco_withdrew was RE-ADDED as a
            distinct, valid code — the two are not synonyms after all. close_reason is now
            context-sensitive to the FROM status: open→closed offers
            {portco_withdrew, onboarded, on_hold_too_long, others}; on_hold→closed offers
            {on_hold_too_long, portco_deferred, others}. Both codes are permitted by
            CloseReasonCode; which subset is offered is enforced in the UI
            (status-change-dialog.tsx), not by a backend context check.

## 8B.3 Schema additions (P24)

  positions: on_hold_at TIMESTAMPTZ NULL — Alembic 0041_position_onhold_timestamps.
  positions: on_hold_resolved_at TIMESTAMPTZ NULL — Alembic 0041_position_onhold_timestamps.
  Both real columns (not metadata_): used in transition-guard logic and reporting.
  Data migration in 0041: close_reason.code='portco_withdrew' → 'portco_deferred'.

## 8B.4 Acceptance criteria (P24)

AC-P24-001  PATCH status=on_hold → 200; response.on_hold_at is a non-null datetime;
            on_hold_resolved_at is null.
AC-P24-002  PATCH status=in_progress (from on_hold) → 200; response.on_hold_resolved_at
            is a non-null datetime.
AC-P24-003  PATCH status=open (from on_hold) with no applications → 200;
            response.on_hold_resolved_at is set; status=open.
AC-P24-004  PATCH status=open (from on_hold) with existing applications → 409
            INVALID_STATUS_TRANSITION.
AC-P24-005  PATCH status=closed (from open) with no close_reason body → 400
            CLOSE_REASON_REQUIRED (superseded 2026-07-04: close_reason is no longer
            auto-set for open→closed; the caller must supply one).
AC-P24-006  PATCH status=closed (from on_hold) with close_reason.code=portco_withdrew → 200
            (superseded 2026-07-04: portco_withdrew is a valid CloseReasonCode value again;
            the UI restricts it to the open→closed dialog, but the backend schema accepts it
            from either FROM status).
AC-P24-007  Application PATCH status=onboarded (filling all headcounts) → Celery
            auto-closes position within 15s; response.close_reason.code=onboarded.
            (corrected NFR Phase 3, 2026-07-24 -- 'hired' trigger described here was
            already unreachable since 2026-07-21, do_accept() no longer writes it.)


# ════════════════════════════════════════════════════════════════════
# 9. NO-SHOW STATUS   (v2.2 — added 11-Jun-2026)
# ════════════════════════════════════════════════════════════════════

## 9.1 New status
'no_show' is added to the Position Current Status set (now: open, in_progress,
portco_confirmed_yet_to_offer, offered, offer_accepted, onboarded, on_hold,
no_show). It means the selected candidate did not join. This extends the set in
§3 (status endpoint) and §4 (BR-003).

## 9.2 Mandatory No-Show reason — exactly one of
  better_offer        — candidate accepted a better offer elsewhere
  better_brand        — candidate chose a stronger/preferred brand
  remote_100          — candidate wanted a 100% remote role
  retained_current    — candidate was retained in their current company
  personal            — personal reasons
  others              — any other reason

## 9.3 Conditional captures per reason
- better_offer (MANDATORY): no_show_org_name (organization that made the offer)
  AND no_show_offer_amount (the offer made) + no_show_offer_currency. If the
  candidate has not shared, set no_show_details_not_shared = true (org/amount
  may then be blank).
- better_brand (MANDATORY): no_show_org_name (organization the candidate decided
  to / joined). If not shared, set no_show_details_not_shared = true.
- others (MANDATORY): no_show_other_reason — free text.
- remote_100 / retained_current / personal: no extra capture.

## 9.4 Endpoint / behaviour
PATCH /api/v1/positions/{id}/status with status='no_show' requires
no_show_reason and that reason's conditional fields (unless
no_show_details_not_shared = true for better_offer / better_brand). A
position_history status_change row capturing the reason + details is written
with its timestamp (existing trigger/service).

## 9.5 Business rules (no-show)
BR-031  status='no_show' ⇒ no_show_reason mandatory (400 NOSHOW_REASON_REQUIRED).
BR-032  no_show_reason='better_offer' ⇒ (no_show_org_name AND no_show_offer_amount)
        OR no_show_details_not_shared=true (else 400 NOSHOW_OFFER_DETAILS_REQUIRED).
BR-033  no_show_reason='better_brand' ⇒ no_show_org_name OR
        no_show_details_not_shared=true (else 400 NOSHOW_BRAND_DETAILS_REQUIRED).
BR-034  no_show_reason='others' ⇒ no_show_other_reason required
        (400 NOSHOW_OTHER_REASON_REQUIRED).
BR-035  Every no_show transition is captured in position_history with reason +
        details, timestamped.

## 9.6 Acceptance criteria (no-show)
AC-029  status=no_show without reason → 400 NOSHOW_REASON_REQUIRED.
AC-030  better_offer with org+amount → 200; better_offer with
        details_not_shared=true and blank org/amount → 200.
AC-031  better_offer with neither details nor the not-shared flag → 400.
AC-032  better_brand with org → 200; with not-shared flag → 200.
AC-033  others without free text → 400; with free text → 200.
AC-034  Any no_show → a position_history row with reason + timestamp.

## 9.7 UX / UI (no-show)
- 'No-Show' appears in the position status dropdown. Selecting it reveals a
  REQUIRED "No-Show reason" single-select.
- Reason-conditional reveal (progressive disclosure):
    • Better Offer → Organization Name (text), Offer Made (amount + currency
      picker), and a checkbox "Candidate has not shared these details" (checking
      it clears + makes org/amount optional).
    • Better Brand → Organization (text) + the same "not shared" checkbox.
    • Others → "Other reasons" (multiline text, required).
    • 100% Remote / Retained in Current Company / Personal Reasons → no extra fields.
- Save is blocked until the reason's mandatory fields (or the not-shared checkbox)
  are satisfied; field-level validation messages.

## 9.8 Data model (no-show) — additive
  position_status_enum gains 'no_show'.
  New enum position_no_show_reason_enum
    ('better_offer','better_brand','remote_100','retained_current','personal','others').
  positions add: no_show_reason, no_show_org_name, no_show_offer_amount,
    no_show_offer_currency (FK currencies), no_show_details_not_shared BOOLEAN,
    no_show_other_reason TEXT — with CHECK constraints per BR-031..034.


# ════════════════════════════════════════════════════════════════════
# 10. POSITION BUDGET & MULTI-CURRENCY COST MODEL   (v2.2 — added 11-Jun-2026)
# ════════════════════════════════════════════════════════════════════

## 10.1 Currency reference
A currencies reference (ISO 4217 majors, xe.com-style) provides code, symbol,
and English name — e.g. USD $ "US Dollar", EUR € "Euro", INR ₹ "Indian Rupee",
GBP £ "Pound Sterling", JPY ¥ "Japanese Yen", AED د.إ "UAE Dirham", SGD S$
"Singapore Dollar", and so on. The Position Budget Base Currency is selected
from this list.

## 10.2 Position currency + rate INPUTS (captured on the position)
  budget_base_currency   — selected currency code (e.g. EUR)
  inr_to_base_rate       — 1.00 INR = X base   (e.g. 1 INR = 0.00907109 EUR)
  usd_to_base_rate       — 1.00 USD = Y base   (e.g. 1 USD = 0.86885382 EUR)
  min_salary_inr         — Position Minimum Salary Budget in INR
  max_salary_inr         — Position Maximum Salary Budget in INR

## 10.3 DERIVED budget values (computed by the ATS, displayed read-only)
All conversions use the rates captured in §10.2; the rates used are snapshotted
alongside the computed values (rates change over time).
  min_salary_base   = min_salary_inr × inr_to_base_rate          (req. 6)
  max_salary_base   = max_salary_inr × inr_to_base_rate          (req. 8)
  avg_salary_inr    = (min_salary_inr + max_salary_inr) / 2      (req. 9.1)
  avg_salary_base   = (min_salary_base + max_salary_base) / 2    (req. 9.2)
  ga_load_base      = 5000 × usd_to_base_rate                    (req. 9.3: USD 5,000 G&A load in base)
  annual_loaded_cost_base = (avg_salary_base + ga_load_base) × 1.17   (req. 9.4)

## 10.4 Display
The position screen shows side by side:
  - Min / Max / Average salary in INR
  - Min / Max / Average salary in Base Currency (with symbol)
  - G&A Load (base) and Annual Loaded Cost (base)
Base-currency values are formatted with the selected currency symbol + code.

## 10.5 One-Time cost (per OFFERED candidate) — req. 9.5
Computed when a candidate is offered for the position; needs the candidate's
Source (Candidate §10) and the offer figures. All amounts in Position Base Currency:
  recruitment_fee_base = 1200 × usd_to_base_rate           (USD 1,200 recruitment fee)
  signon_bonus_base    = sign-on/joining bonus if offered, else 0
  sourcing_fee_base    = EXACTLY ONE of:
       employee_referral_fee_base    if candidate.source = employee_referral
       vendor_fee_base = 0.0833 × offered_annual_ctc_base
                                     if candidate.source = third_party_vendor
       0                              if candidate.source = direct
  one_time_cost_base = (recruitment_fee_base + signon_bonus_base + sourcing_fee_base) × 1.17

Formula (as specified): [(Recruitment Fee) + (Sign-on/Joining Bonus, if offered)
+ (Employee Referral OR 3rd Party Vendor Partner Fee; exactly one, only when the
candidate is NOT direct/recruiter-sourced)] × 1.17.

NOTE: the Employee Referral Fee amount is CONFIGURABLE (tenant_settings
`employee_referral_fee`, in base currency / USD-equivalent). The requirement
fixes the vendor fee at 8.33% of CTC but does not state the referral fee amount —
set this value before go-live. (Flagged as an assumption.)

## 10.6 Business rules (budget)
BR-036  budget_base_currency mandatory; must be an active currencies row.
BR-037  inr_to_base_rate and usd_to_base_rate mandatory, > 0, NUMERIC(18,8).
BR-038  max_salary_inr ≥ min_salary_inr (400 INVALID_SALARY_RANGE).
BR-039  All derived values are computed server-side (never client-supplied),
        persisted with the rate snapshot used, and recomputed when inputs/rates
        change. Money is NUMERIC(18,2); rounding is half-up, documented.
BR-040  One-time cost is computed at offer; exactly one sourcing fee applies,
        keyed off candidate.source; vendor_fee_base = 8.33% × offered_annual_ctc_base.
BR-041  Constants (configurable in tenant_settings): G&A load = USD 5,000;
        loaded-cost multiplier = 1.17; recruitment fee = USD 1,200;
        one-time-cost markup = 1.17; vendor fee % = 8.33.

## 10.7 Acceptance criteria (budget)
AC-035  base=EUR, rates + min/max INR set → min/max/avg base computed correctly.
AC-036  ga_load_base = 5000 × usd_to_base_rate; annual_loaded_cost_base =
        (avg_salary_base + ga_load_base) × 1.17 (asserted to 2 decimals).
AC-037  max_inr < min_inr → 400 INVALID_SALARY_RANGE.
AC-038  Offer to vendor-sourced candidate, CTC=X base → vendor_fee_base=0.0833·X,
        one_time_cost_base=(1200·usd_to_base + signon + 0.0833·X)×1.17.


# ════════════════════════════════════════════════════════════════════
# 11. RECRUITER OWNERSHIP   (v2.2 enhancement — 2026-06-24)
# ════════════════════════════════════════════════════════════════════

## 11.1 Purpose
Assign ownership of position headcount slots to specific recruiters.
One or more recruiter-count pairs can be assigned per position.
sum(assigned_count) must not exceed position_count. Optional — positions may have
zero assignments.

## 11.2 New endpoints

### PATCH /api/v1/positions/{id}/recruiters
Auth   : Bearer — roles: recruiter, hr_admin
Request: {"assignments": [{"recruiter_id": "uuid", "assigned_count": int}]}
         Empty list clears all assignments.
Action : Replace-all — soft-deletes existing, inserts new assignments.
         Validates each recruiter_id has role=recruiter.
         Validates sum(assigned_count) <= position_count.
         Writes audit log action: position_recruiters_set.
Response 200: list[RecruiterAssignmentResponse]
Response 404: POSITION_NOT_FOUND | RECRUITER_NOT_FOUND
Response 422: RECRUITER_ASSIGNMENT_OVERFLOW

### GET /api/v1/positions/recruiter-options
Auth   : Bearer — all authenticated roles
Response 200: list[RecruiterOption] — {id, display_name, email}
Notes  : Org-scoped via RLS. Returns only active users with role=recruiter.
         Must be registered BEFORE /{id} route in the router.

## 11.3 GET /api/v1/positions and GET /api/v1/positions/{id} response extensions
Both PositionSummary and PositionDetailResponse now include:
  recruiter_assignments: list[{recruiter_id, recruiter_name, assigned_count}]
  (empty list if no assignments set)

## 11.4 Business rules
BR-045  sum(assigned_count) <= position_count. Excess → 422 RECRUITER_ASSIGNMENT_OVERFLOW.
BR-046  Each recruiter_id must be an active user with role=recruiter. Else 404 RECRUITER_NOT_FOUND.
BR-047  Assignments are optional. A position with no assignments is valid.
BR-048  Replace-all semantics: every PATCH replaces the entire assignment set.
BR-048a (2026-07-14) When a replace-all PATCH leaves the position with exactly one active
        recruiter assignment, this also triggers applications/BR-022's recruiter-sync
        (via the applications module's service interface — see applications/spec.md BR-022).
BR-049  Audit action: position_recruiters_set.

## 11.5 New table
position_recruiter_assignments — Alembic 0027_pos_recruiter_assign.
Columns: id, position_id (FK positions), recruiter_id (FK users), assigned_count,
         created_at, updated_at, deleted_at, version.
Partial unique index (WHERE deleted_at IS NULL) allows soft-delete + re-assign.

## 11.6 Acceptance criteria
AC-045  PATCH with valid recruiter + count ≤ position_count → 200 with assignment list.
AC-046  PATCH with sum > position_count → 422 RECRUITER_ASSIGNMENT_OVERFLOW.
AC-047  PATCH with non-recruiter user_id → 404 RECRUITER_NOT_FOUND.
AC-048  PATCH with empty list → 200, clears all assignments.
AC-049  GET /positions and GET /positions/{id} include recruiter_assignments field.
AC-050  GET /recruiter-options returns active recruiter users for the org.
AC-039  Offer to a direct candidate → sourcing_fee_base = 0.
AC-040  Changing usd_to_base_rate recomputes ga_load_base, loaded cost,
        recruitment fee, and one-time cost with the new rate snapshot.

## 10.8 UX / UI (budget)
- Base Currency: searchable dropdown listing "SYMBOL  CODE — English Name"
  (xe.com style).
- Exchange-rate inputs: two high-precision numeric fields — INR→Base and
  USD→Base — with helper text "1.00 INR = …" / "1.00 USD = …". May be pre-filled
  from a rates provider with manual override.
- Salary inputs: Min and Max in INR (currency-formatted).
- A READ-ONLY computed panel updates live as inputs change: Min/Max/Avg in INR
  and in Base; G&A Load (base); Annual Loaded Cost (base). Each value shows the
  currency symbol; tooltips show the formula and the rate used.
- One-Time cost panel (offer screen): shows recruitment fee, sign-on bonus, the
  applicable sourcing fee (labelled "Referral" or "Vendor 8.33%"), and the total
  with the ×1.17 markup — read-only, recalculated from the offer inputs.

## 10.9 Data model additions (additive — see docs/SCHEMA_EVOLUTION.md)
  New table currencies (code CHAR(3) PK, symbol, english_name, is_active), seeded
    with majors.
  positions add: budget_base_currency CHAR(3) FK currencies, inr_to_base_rate
    NUMERIC(18,8), usd_to_base_rate NUMERIC(18,8), min_salary_inr NUMERIC(18,2),
    max_salary_inr NUMERIC(18,2), and persisted derived min_salary_base,
    max_salary_base, avg_salary_inr, avg_salary_base, ga_load_base,
    annual_loaded_cost_base (NUMERIC(18,2)), rate_snapshot_at TIMESTAMPTZ.
  New table offer_details (application_id PK/FK, offer_currency, offered_annual_ctc_base,
    signon_bonus_base, recruitment_fee_base, sourcing_fee_base, one_time_cost_base,
    inr_to_base_rate, usd_to_base_rate, computed_at) — holds the per-offer One-Time cost.


# ════════════════════════════════════════════════════════════════════
# 12. GET /positions/ageing-summary   (P25 — added 2026-07-03)
# ════════════════════════════════════════════════════════════════════

Returns the count of active (non-closed, non-deleted, approved_at IS NOT NULL) positions
per ageing bucket, in a single aggregation query.

**Endpoint:** `GET /api/v1/positions/ageing-summary`
**Auth:** required — same roles as GET /positions (any authenticated role)
**Query params:**
  - `organization_id` (UUID, optional) — filter to one org; if omitted, caller's RLS context applies

**Response 200:**
```json
{
  "lte_7": 3,
  "8_to_15": 5,
  "16_to_30": 2,
  "31_to_45": 1,
  "46_to_90": 0,
  "91_to_120": 0,
  "over_120": 0
}
```

**Business rules:**
- BR-SUM-001: Only positions where `deleted_at IS NULL AND status != 'closed' AND approved_at IS NOT NULL` are counted. Positions with `approved_at IS NULL` (unapproved) are excluded from all buckets.
- BR-SUM-002: Bucket ranges match `_AGEING_BUCKET_SQL` in `list_helpers.py` exactly (same INTERVAL boundaries, inclusive start, exclusive end).
- BR-SUM-003: If `organization_id` provided, filter to that org and set RLS scope. If absent, no org filter (caller's RLS context from token applies).
- BR-SUM-004: A position that falls in no bucket (e.g. approved_at IS NULL) is never counted; bucket counts always sum to total active positions with approved_at.

**Acceptance criteria:**
- AC-SUM-001: GET with no org_id → 200 with 7 integer counts.
- AC-SUM-002: GET with valid org_id → 200 with counts scoped to that org.
- AC-SUM-003: Unauthenticated → 401.
