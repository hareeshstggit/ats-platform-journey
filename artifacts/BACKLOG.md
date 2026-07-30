**Artifact Version 2.2 — Baselined 11-Jun-2026**  ·  Versioning rule: docs/VERSIONING.md

---

# ATS Platform — Backlog & Tech Debt Reference

**Last updated: 2026-07-28**  ·  Maintainer: Claude Code, on behalf of hareesh@stg.com

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
| 2 | Notifications real fan-out — email via AWS SES for interview.scheduled/offer.sent | 🟡 Code complete, principal-reviewer APPROVE, merge blocked on GitHub Actions quota (PR #196) |
| 3 | CI test gap — Postgres/Redis services not provisioned (18 backend tests fail in CI) | 🔴 Queued (3rd) |
| 4 | Frontend test debt — nav-items.test.ts, position-schema.test.ts, others (6+ pre-existing failures) | 🔴 Queued (4th) |
| 5 | e2e CI job-design gap — MSW can't intercept proxied backend calls | 🔴 Queued (5th) |

## 2. Pending merges (all blocked on GitHub Actions quota, refills 2026-07-31 midnight IST)

| Branch | What | Status |
|---|---|---|
| `dev/notifications-email-fanout` | PR #196, item 2 above | principal-reviewer APPROVE, ready to merge |
| `dev/seed-legal-transaction-demo` | Real-HTTP golden-path seed script + reference Excel | Built + verified, PR not yet raised |
| `dev/seed-uat-recruitment-funnel` | Config-driven UAT funnel seeder (see §9) | Built + verified at 120-candidate proof scale, PR not yet raised |

## 3. Business-rule gaps (unverified candidates — spot-check before building, none touched since found)

1. 🔴 Offers has no mirrored org-rejection-ban check at offer-create time (BR-014 exists for applications, not offers).
2. 🔴 Screening: reverting shortlisted→screen_rejected doesn't cascade-invalidate existing interviews (orphaned pending interview records).
3. ❓ Applications: manual status→`onboarded` bypassing override_reason gate — likely false positive (onboarded already requires `onboarded_at`), recheck before building.
4. 🔴 Interviews: feedback outcome submittable with no `scheduled_at` set — no validation gap-check done yet.
5. 🔴 Interviews: kit regeneration on reschedule (`PATCH /schedule`) lacks an idempotency/duplicate-prevention guard.
6. 🔴 Interviews: BR-SYNC-005 auto-sync is fire-and-forget with a bare exception-swallow, no retry/logging (reliability/observability item, not really a missing BR).

## 4. Tech debt — data/query correctness

- 🔴 `positions/repository.py::get_interview_level()` (used during interview-creation validation) doesn't filter `is_active`, unlike `child_repository.py::list_levels()` (the dropdown). A deactivated level's ID could theoretically pass validation via a stale cached ID or direct API call — not exploited in practice, worth closing.
- 🔴 `screening/repository.py::list_decisions()` JOINs to candidates/positions without soft-delete guards (unlike applications' equivalent pattern) — reconfirmed live pattern-class, still not fixed.
- 🔴 `category_rank` SQL duplicated across 3 modules (interviews, reporting, applications) — kept manually in sync, never extracted to a shared fragment; divergence risk.
- ❓ `positions/schemas.py`'s `InterviewLevelRequest`/`InterviewLevelResponse` missing `level_category` field (code behind spec, BUG-4).
- 🔴 Organizations/Departments DELETE endpoints documented in spec, never built (4 ACs unverifiable).

## 5. Tech debt — tests/CI

- 🔴 `offers/tests/test_functional_hiring_uniqueness.py` — 3 of 6 tests fail against the live stack (drives hire-uniqueness via a manual status PATCH to `hired`, which is 422-blocked since BR-054). Needs a rewrite to go through the real `POST /offers/{id}/accept` path.
- 🔴 3 pre-existing failures found during the 2026-07-23 mypy-cleanup pass, root cause not investigated: `tests/unit/test_seed_dev.py::test_run_seed_fresh_creates_users_grants_and_audits`, `departments/tests/test_repository.py::test_list_scopes_to_org_searches_and_orders`, `organizations/tests/test_repository.py::test_list_applies_search_and_active_filters_and_orders_by_name`.
- 🔴 3 pre-existing `positions` module test failures: `test_service.py::test_change_status_stale_version_raises_409` (test itself is stale — needs a rewrite to a real reachable scenario), `test_tasks.py::test_extract_storage_miss_persists_failed` + `test_extract_success_persists_result` (mock/fixture drift, not chased to root cause).
- 🔴 No frontend component test for `positions-ageing-report.tsx` / `positions-ageing-bucket-strip.tsx`.
- 🔴 = Execution Queue item 3: backend CI provisions no real Postgres/Redis — ~18 tests fail (Celery result-backend retries, 2 tests dialing Postgres directly).
- 🔴 = Execution Queue item 4: frontend `nav-items.test.ts` (2 failures, stale expected nav list) + `position-schema.test.ts` (3 failures) — both pre-existing, confirmed multiple times, never fixed.
- 🔴 = Execution Queue item 5: `e2e` Playwright job has no backend/DB service — every login-dependent spec times out. Needs either real backend+DB services added to the job, or MSW extended to cover the proxied routes.

## 6. Tech debt — code hygiene (oversized files, 300-line/40-line caps)

**Explicitly split out of NFR Phase 3 (2026-07-24)** as its own item — 38 files (18 backend + 20 frontend) exceed the 300-line cap, including core service files. High regression risk if split blindly; needs individual per-file decomposition analysis before scheduling. Known offenders: `interviews/service.py` (1412 lines), `interviews/repository.py` (1034), `applications/service.py` (661→666), `applications/_service_helpers.py` (525+), `application-list.tsx`, plus ~34 others (full list was captured during the Phase 3 line-count audit — re-run `wc -l` sweep if the exact list is needed again, file sizes will have shifted).

## 7. Tech debt — dependencies & secrets

- 🔴 Hardcoded plaintext DB password in one-time backfill scripts' DSN strings (`backend/app/scripts/backfill_*.py`) — pre-existing convention across all of them, should move to `os.environ`.
- 🔴 3 declared dependencies with zero imports anywhere in `backend/`: `sentry-sdk`, `prometheus-client` (both likely reserved for parked NFR observability work), `openpyxl` (retained as FK target per GO_LIVE_CHECKLIST's Scorecard-CRUD-removed note, PR #50 — confirm genuinely dead or living outside `app/` before removing). Currently `deptry`-ignored, not removed.

## 8. NFR (parked since 2026-07-10; unpark condition met 2026-07-23 — Dashboard+Reporting shipped)

- ✅ Phase 3 — repo-wide dead-code/quality sweep (PR #195, docstrings+dead-code only; oversized-file hygiene split to §6 above).
- ⏸️ Phase 2b — perf root-cause diagnosis (nav-response-time regression). Not resumed.
- ⏸️ Phase 2c — load-testing harness + 200-250 concurrent-user capacity validation. Nothing built yet.
- ❓ Watch-item: `_pipeline_progress_sql.py`'s event CTEs compute over the full matched-position set before pagination — feeds into Phase 2c when it happens.

## 9. Feature backlog (not started / deferred)

- ⏸️ Multi-panelist per interview level (schema+backend+frontend) — explicitly deferred, "build when asked."
- 🔴 Onboarding workflow — new module, stub only (`position_status_enum.onboarded` marker exists, nothing else).
- 🔴 Consent + DPDP module — new module, stub only.
- ✅ Notifications real fan-out — see Execution Queue item 2 / §2 pending merges.
- ✅ (superseded/obsolete, confirm-close) Positions-ageing widget on the Positions list — the full Positions Ageing *report* shipped (PR #173) on the reports dashboard instead.
- 🔴 Remaining reporting endpoints, all still stub: candidate-uploads, positions/creation, positions/jd-changes, screening, interviews/pipeline (lifetime matview), interviews/candidate-status, transition-times, selection-rates, panelists/workload, departments/candidate-status-summary.
- ⏸️ **UAT recruitment-funnel data — 500 candidates / 50 positions full scale.** Script built + verified at 120-candidate proof scale (`dev/seed-uat-recruitment-funnel`, §2). Full run explicitly deferred by user (month-end budget) — resume by restoring the 2 dropped categories (Professional Services, Product/Platform Consultants) for the full 14-category run; budget ~4x the candidates at similar per-unit cost.

---

## How items move through this doc

New item found → add to the relevant section with 🔴. Work starts → 🟡. Merged/shipped → ✅ with
the PR/commit reference, then drop to a one-line mention or remove if no longer useful context.
Explicitly deferred by user choice → ⏸️ with the reason. Never let an item sit stale without a
status matching its actual state — if unsure whether something is still open, mark ❓ and verify
before the next time it's relevant, don't silently trust an old entry.
