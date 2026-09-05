**Artifact Version 2.2 — Baselined 11-Jun-2026**  ·  Versioning rule: docs/VERSIONING.md

---

# ATS Platform — Backlog & Tech Debt Reference

**Last updated: 2026-08-08**  ·  Maintainer: Claude Code, on behalf of hareesh@stg.com

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

## Table of Contents

- PRIORITY (read first)
- 0. Go-Live readiness
- 1. Active execution queue
- 2. Pending merges
- 3. Business-rule gaps
- 4. Tech debt — data/query correctness
- 5. Tech debt — tests/CI
- 6. Tech debt — code hygiene
- 7. Tech debt — dependencies & secrets
- 8. NFR
- 9. Feature backlog
- How items move through this doc

---

## PRIORITY (updated 2026-09-03 — read before anything else in this file)

1. **`fix/main-ci-break` (PR #231) — MERGED to `main` 2026-09-03 (commit `97c18d78`).** Cleared
   43 of the original 48 tracked `test`-job failures + all 133 `typecheck` errors. G15d
   (PR #228), G19 (PR #230), G15e (PR #229) also all MERGED same day — the full G15/G15b/
   G15c-G19 series is closed except G15c/G17 (already-parked, low-urgency, unrelated to today).
2. **Process root-cause + concrete measures (2026-09-03) — DONE, in `.claude/CLAUDE.md`.** 4
   new binding sections: review-round & script-verification discipline; debt-escalation &
   pre-investigation discipline; proactive deviation flagging (the 10-dimension standing duty);
   coverage-gate risk-impact caveat (keeps the 80% gate fixed, but a red gate is a hard blocker
   only if the specific uncovered delta touches one of the 10 dimensions — tracked debt
   otherwise). Plus the branch-protection process substitute (no plan upgrade — a
   `pr-status-summary` CI job + a merge-gating process rule).
3. **Bucket (b) — FIXED 2026-09-03, branch `fix/bucket-b-category-rank-fixture`, review in
   progress, NOT YET MERGED.** Both root causes closed: (a) the hardcoded `POSITION_ID` FK
   violation (4 of 5 tests) — replaced with a `position_id` fixture resolving a real position
   dynamically; (b) the stale BR-SEQ-001 test (the 5th) — rewritten to prove the still-relevant
   inflated-rank regression under the current linear-chain gate. All 5 tests pass locally
   against real Postgres. `principal-reviewer` round 1: CHANGES-REQUESTED (BACKLOG not updated
   in the same change — this entry; a few cheap nits) — fixed same session, round 2 pending.
4. **Coverage-gate risk-impact assessment — DONE (2026-09-03), per item 2's new caveat; the
   80% gate itself is confirmed NOT lowered.** Live-measured via a full `--cov-report=json` run:
   real application code (`app/modules/`, `app/shared/`, `app/workers/`) is at **88.2%**,
   already above the gate. The entire aggregate shortfall (66%) is structural — 10 standalone
   `app/scripts/` files (seed/backfill/demo/cleanup tools, 1,543 statements, 15.7% covered)
   dragging the denominator down. Confirmed via repo-wide grep: zero production code imports
   anything from `app/scripts/` — these scripts are never on any request path in any
   environment. Per-dimension assessment: **no impact on any of the 10 dimensions** — inert
   against Quality/Performance/Scalability/Reliability/Security/Observability/Modularity/
   Maintainability/Code-Design/Architecture-Extensibility. **User decision (2026-09-05):
   leave as-is** — no `omit=` config change; keep applying the Coverage-gate risk-impact
   caveat per-PR as already practiced (PRs #235/#236/#237). Closed.
   **Separately flagged, NOT part of the gate math but a real gap**: `app/modules/offers/
   tasks.py` is 0% covered (89 statements) — a Celery task file, genuinely untested, touching
   Reliability/Observability. Tracked in §5 below, not blocking, needs its own pass.

---

## 0. Go-Live readiness (tracked closely — 1-week dev freeze from 2026-08-08 pending user's date-planning decisions)

User directive 2026-08-08: stop new development for 1 week (starting once PR #222 merges); use the
week to clarify go-live readiness so a real AWS Bedrock go-live date can be set and worked backward
from. This section is the tracked list that decision is based on — update it inline the moment any
item's status or the user's scope decision changes, same living-doc discipline as the rest of this
file. Source: `docs/GO_LIVE_CHECKLIST.md` (full detail) — this section is the trackable summary.

### 0.0 HIGHEST PRIORITY on resumption — Change Requests documented, not yet built (execution order below)

**`nfr-response-time-slo-validation` (CR#1.A) — EXECUTES FIRST, ahead of everything else.**
User's explicit instruction (2026-08-10): put this ahead of CR#1 in the queue. Full OpenSpec
change at `openspec/changes/nfr-response-time-slo-validation/` (4/4 artifacts complete).
Origin: user asked for the platform to hit an industry-standard enterprise-SaaS response time
(≤2s) — verified as a reasonable target (sits between Nielsen's 1s "flow of thought" threshold
and Google Core Web Vitals' 2.5s "good" LCP threshold). Summary: adds the missing frontend
targets (LCP ≤2.0s, INP ≤200ms) that never existed anywhere; stands up a k6 load-testing
harness (closing the long-parked BACKLOG §8 Phase 2c gap); runs the FIRST real measured
baseline against every existing target (backend p95 read<150ms/write<300ms, 200+ concurrent
users — none ever empirically validated); codifies perf testing must run against a production
build, never `next dev` (confirmed live 2026-08-10: `/reports` took 39.8s to compile on first
hit in dev mode — a false signal a load test must never be taken against); explicitly exempts
async AI-feature latency from these synchronous-request targets. New cross-cutting
`performance-slo` capability (no single module owns response-time SLOs, same precedent as
`pipeline-reliability`). Read the OpenSpec change's own design.md before starting task 1.1.

**`candidate-ai-match-screen-consolidation` (CR#1) — MERGED (PR #223, 2026-08-18) and ARCHIVED.**
All 22 tasks (1.1-7.4) done. Full Gate 5 pipeline run: `backend-engineer` → `ux-ui-engineer` →
independent Gate-1 re-runs → `functional-test-engineer` (found + cleared a real BLOCK MERGE
bug, BUG-1 — the D1 offline fallback never actually fired on a real Celery-queued permanent
provider failure) → `integration-test-engineer` (full lifecycle + a real Playwright golden
path) → `principal-reviewer` × 3 rounds (round 1 CHANGES-REQUESTED: 1 critical + 5 majors,
fixed; round 2 APPROVE-WITH-NITS: 5 nits, fixed; round 3 APPROVE-WITH-NITS: 2 more doc-only
nits from its own re-verification, fixed) — zero Critical/Major findings survived any round.
This branch's first real GH Actions CI run also found and fixed 2 environment-parity gaps
invisible to local Windows testing (a hardcoded Windows venv path; `frontend-ci.yml`'s `e2e`
job never starting a Celery worker at all) — exactly the class of defect the binding
Live-verification mandate exists to catch. Summary: stop auto-firing matching/screening on
upload (extraction-only); collapse "Trigger Job Matching"/"Trigger AI Screening" into one
LLM-primary "AI Job Match" trigger (offline scorer demoted to fail-safe-only fallback,
itself now also hard-gated at the same threshold, never persisting a sub-threshold row);
hard-gate results at a configurable ≥75% match (default) at BOTH write- and read-time, each
result carrying a reworked, pair-specific 5+5 (match points / gaps to verify); unify
screening-question generation onto one mechanism (`candidate_screenings/`) used by both a new
per-position "Screen" action and the existing top-right entry point; retire
`candidates/screening/`'s match-decision+scorecard write path (table stays, per the
additive-migration rule). `openspec/specs/candidates/spec.md` +
`openspec/specs/pipeline-reliability/spec.md` updated in the same PR (binding spec-sync
mandate). One accepted, documented residual: a narrow audit-log mislabeling edge case
(`PermanentProviderError` fallback + gate excludes everything → audit row records the
configured provider instead of "offline"; nothing persists to `candidate_position_matches`
either way) — see the change's own `tasks.md` task 7.1 for the full rationale. New tech debt
logged: G14 (`job_matcher.py` over the 300/40-line caps, deferred — systemic across 19+ files).
**G14 DONE (Tier 5 code-hygiene, 2026-08-31 — see §6 below).**
Next up per the user's confirmed execution order: CR#2
(`interview-kit-candidate-aware-scheduled-generation`).

**`interview-kit-candidate-aware-scheduled-generation` (CR#2) — MERGED (PR #224, 2026-08-19) and
ARCHIVED.** All 6 task sections (1-6) done. Full Gate 5 pipeline: `backend-engineer` →
`ux-ui-engineer` (rename) → Gate 1 → `functional-test-engineer` (found a real, previously-latent
bug, M1 — see below) → `integration-test-engineer` (full STG L1/L2 lifecycle, found 1
pre-existing concurrency defect, not fixed, logged) → `principal-reviewer` × 3 full rounds + 1
scoped confirm pass (round 1/2 fixed the M1 bug + a design misconception; round 3
CHANGES-REQUESTED — 1 critical, 5 major, all fixed and independently re-confirmed:
**APPROVE-WITH-NITS**). Real bug found and fixed: `tasks.py`'s (later `_kit_context.py`'s)
candidate-context JOIN was an INNER JOIN against `candidates`, whose RLS policy has no
`fn_is_internal()` escape — a soft-deleted candidate silently dropped the whole kit-generation
context row. Fixed to `LEFT JOIN`; proven against genuine RLS enforcement after this session
also found and fixed the local dev DB's RLS being disabled on 5 candidate-related tables
(re-applied migrations 0010/0011's exact DDL, user-approved, local-only). First real CI run: 3
red checks, all pre-existing on `main`, zero new regressions (see §5 below for the detailed
triage, including a newly-identified root cause — `backend-ci.yml`'s `test` job has no Celery
worker at all). Summary: the 5 questions per focus area become candidate-experience-aware
(non-PII signal only, `experience_years`/`experience_months` coarse-banded), the 10 focus areas
stay position-driven; kit generation drops its create-time trigger, keeping only the
schedule-time trigger; "Schedule Interview" action renamed to "Schedule Interview & Generate
Interview Kit"; `local_kit` offline path retained as fail-safe-only. `openspec/specs/
interviews/spec.md` updated in the same PR (BR-P20-006 revised, BR-P20-012/AC-014/AC-020 added).
3 new tech-debt items logged (§4/§5): an `interview_level_kits` concurrency defect (no unique
constraint on `interview_id`), a `candidate_documents` RLS/schema divergence, and the
`backend-ci.yml` no-Celery-worker gap — none blocking, none part of this CR's diff.
Next up per the user's confirmed execution order: whatever the user selects next from §0.0/§0.1.

### 0.1 Hard blockers (not scope choices — must happen regardless of scope decisions below)

| # | Blocker | Status | Notes |
|---|---|---|---|
| G1 | Terraform `iam` module is a skeleton (`task_role_arn`/`execution_role_arn` both `null`) | 🔴 | Blocks `terraform apply` outright — nothing else in AWS can start until real. |
| G2 | No AWS account / OIDC deploy role / Secrets Manager populated | 🔴 | `deploy.yml` fails immediately without `AWS_DEPLOY_ROLE_ARN`. User-owned action item (account provisioning), not a code fix. |
| G3 | `deploy.yml` has never run successfully end-to-end, even once | 🔴 | Needs a real dry run against a live AWS env before trusting it for go-live. |
| G4 | RDS not provisioned in cloud | 🔴 | Only local Podman Postgres exists today; migrations never run against a cloud DB. |
| G5 | Domain, TLS, WAF, CDN not applied | 🔴 | IaC authored, never applied. |
| G6 | Security pen-test + DPDP DPIA not scheduled | 🔴 | External/compliance-owned; needs lead time to schedule a vendor/reviewer. |
| G7 | Performance/SLO load testing (NFR Phase 2c) | 🟡 | **First real baseline recorded 2026-08-17, deep-dive reviewed by `principal-performance-auditor` 2026-08-18** (`docs/ARCHITECTURE.md`'s SLO section, GH Actions run [32053886296](https://github.com/hareeshstggit/ats-platform-project/actions/runs/32053886296)) — all 4 read endpoints missed the <150ms p95 target (218-231ms measured), a closed-loop saturation artifact (Little's Law: 30 zero-think VUs ≈ ~2,300 real users, real per-request service demand only ~4.4ms, est. 75-90% of the gap is queueing artifact not app latency). Login missed <300ms badly (4874.5ms p95) — root-caused to a GIL-bound bcrypt-executor-thread-pool ceiling (throughput capped by vCPU count, not thread count), NOT single-worker queueing as first hypothesized — a 1-VU zero-contention run in the same workflow already shows p95=793.75ms (2.9x the code comment's ~272ms/call estimate), a fixed floor above the 300ms target before any concurrency; the exact 30-VU queueing arithmetic doesn't cleanly reconcile with either number, so this is a qualitative mechanism, not an exact-match calculation — see G13, which tracks the zero-concurrency floor breach itself. **This is a local dev-stack/CI-runner baseline on a `seed_dev`-sized dataset, not production** — real production validation is G11, still 🔴. Not yet 🟡→✅ since the harness has run only once, against a topology (2-vCPU runner, zero-think-time closed-loop load, small dataset) known to differ from intended production sizing/traffic shape. |
| G8 | UAT + runbooks + backup/DR + on-call readiness | 🔴 | None started. |
| G9 | **Bedrock Claude models are NOT natively hosted in `ap-south-1`** — access from India works only via Bedrock's Global Cross-Region Inference (verified live 2026-08-08, see sources in conversation history). Real candidate/resume PII would leave India during 4 AI-feature inference calls even though app/DB/S3 stay in `ap-south-1`. | ❓ Awaiting legal/DPO | Feeds directly into D1 (DPDP scope) and D3 (Bedrock-vs-Gemini sequencing) below — get this confirmed by legal/DPO before committing to Bedrock as the AI backend; could reshape the AI-provider decision entirely. |
| G10 | API Fargate service (the actual user-facing HTTP compute) doesn't exist in Terraform yet | 🔴 | Only worker/beat/cwagent services are authored (`ecs` module). No ALB, no API service, no sizing decided. Recommended starting point: 2-3 tasks, 1 vCPU/2GB each, autoscale to 4-5 under peak (see conversation history 2026-08-08 for full NFR-grounded sizing rationale — RDS `db.r6g.large` Multi-AZ+replica, ElastiCache `cache.t4g.small` already right-sized, Celery worker's `desired_count=1` cap needs its own metrics fix before scaling). **G7's first real baseline (2026-08-17, corrected by the 2026-08-18 auditor deep-dive) directly supports this multi-task recommendation** — login's 4874.5ms p95 is a genuine bcrypt-thread-pool ceiling bound by vCPU count, not uvicorn-worker queueing — so more vCPUs per task (not more uvicorn workers) is the actual lever; treat sizing as a starting request informed by real data, still not a locked number until G11's production run. |
| G11 | **Post-go-live production load-test validation** (added 2026-08-10, follows CR#1.A `nfr-response-time-slo-validation`) — re-run the same k6 harness against real deployed AWS infra once it exists, using a `constant-arrival-rate` load profile (~20 rps ≈ 200 users × 10s think time, not the baseline's zero-think closed-loop 30 VUs) and a production-scale (not `seed_dev`-sized) dataset; validate ECS autoscaling triggers fire before the SLO breaches (~350-400ms, not at 500ms); validate CloudFront's real effect on LCP once the CDN skeleton is applied (G5); reconcile local-dev-stack baseline numbers vs. real production numbers in `docs/ARCHITECTURE.md`. | 🔴 Blocked on go-live | **Explicitly deferred until after deployment to real AWS infra** — not Bedrock-specific (AI-feature latency is exempt from these targets by design, D4), this is about needing Multi-AZ RDS/ElastiCache/ECS autoscaling and a realistic dataset/traffic-shape to actually exist before measuring against them. The k6 harness itself, and the FIRST (local) baseline, are built and run now as part of CR#1.A — this gate is only the second, production run. |
| G12 | **Frontend LCP/INP measurement + generator/SUT co-location caveat** (added 2026-08-17, principal-reviewer Majors 4+5 on `nfr-response-time-slo-validation`) — k6 cannot measure browser paint/interaction metrics (it is not a browser); LCP ≤2.0s/INP ≤200ms (`docs/ARCHITECTURE.md` SLO section) need a separate browser-based tool (e.g. Lighthouse CI) as a follow-up, not built as part of this harness. Separately: `load-test.yml`'s k6 generator, the FastAPI backend, Postgres, and Redis all run co-located on ONE 2-vCPU/7GB GH Actions runner (single-worker uvicorn) — any VU count run there measures runner CPU contention as much as application latency; `docs/PERFORMANCE_TESTING.md` caps the workflow's default VU count accordingly for THIS topology. Reaching the spec's real 200-250-concurrent-user target needs either a larger runner or separate generator/SUT infra. | 🔴 | Cross-ref G11 (post-go-live production re-run) — this row is the PRE-go-live local/CI-topology caveat; G11 remains the separate, later, real-infra validation. |
| G13 | **`POST /auth/login` breaches its <300ms p95 target at 1 VU, zero concurrency** (added 2026-08-18, `principal-performance-auditor` deep-dive + `principal-reviewer` Major 3 on `nfr-response-time-slo-validation`) — the CR#1.A baseline's 1-VU run measured p95=793.75ms with no concurrent load at all, meaning this is a genuine fixed-cost SLO gap, not a queueing/measurement artifact like the 4 read-endpoint misses. Root cause: bcrypt (12 rounds) hashing on the request path, GIL-bound in the thread-pool executor (`backend/app/modules/security/service.py`). **Decided 2026-08-29: accept as a documented latency exemption** — user explicitly declined reducing `BCRYPT_ROUNDS` (keeps current security posture) and confirmed vCPU sizing (G10) doesn't lower the floor anyway. No code change; `docs/ARCHITECTURE.md`'s SLO section updated in the same commit with the exemption rationale, matching the existing JD-extraction/matching/screening/kit-generation exemption pattern. | ✅ Closed (exemption) | Closed as a decision, not a fix — see `docs/ARCHITECTURE.md` "Performance & Scalability (SLOs)" for the exemption note. |
| G14 | **`backend/app/modules/candidates/agents/job_matcher.py` was over CLAUDE.md's 300-line file cap (251→345→347 lines) and `match_candidate()` was over the 40-line function cap (57→81→82 lines)** (added 2026-08-18, `principal-reviewer` O1 on `candidate-ai-match-screen-consolidation`) — **file-cap half CLOSED 2026-08-31** (Tier 5 backend catch-up, `docs/BACKLOG.md` §6): 347→299 lines, `_MATCH_PROMPT_TEMPLATE` + parse/normalize moved to `_match_prompt.py`. **Function-cap half CLOSED 2026-09-01** (pure extraction refactor, zero behavior change): `_build_offline_points()` (57 lines) split into `_positive_offline_points()` (24), `_negative_offline_points()` (25), `_pad_to_exact()` (10) + a slimmed `_build_offline_points()` (16); `match_candidate()` (55 lines) split into `_resolve_llm_provider()` (14), `_match_via_llm()` (38) + a slimmed `match_candidate()` (34, docstring kept verbatim). Every function in the file is now ≤40 total lines (largest: `_offline_match`/`_match_via_llm` at 38). `principal-reviewer` round 1 CHANGES-REQUESTED: the function-cap fix re-broke the file cap it had just closed (299→373, since the 4 new points-helpers + 2 constants landed inline) — fixed by moving `_build_offline_points`/`_positive_offline_points`/`_negative_offline_points`/`_pad_to_exact` + the 2 generic-points constants (now tuples, per round-1 nit) to a new sibling `_offline_points.py` (112 lines), same pattern as `_match_prompt.py`'s own split; `job_matcher.py` now 276 lines. Public exports unchanged (`match_candidate`/`MatchResult` only, confirmed by repo-wide grep). `pytest app/modules/candidates/` 390 passed/0 failed, `ruff check` clean on both files (incl. `--select E501` — no new long lines), `mypy` clean on both files (repo-wide `mypy .` has 133 pre-existing errors, unrelated, in 3 other files — tracked separately below). | ✅ | Closed — file cap (276/112 lines across 2 files) and function cap (all ≤40 lines) both done, re-verified after round-1 fix. |
| G15 | **The migration chain cannot provision a working database from source of truth.** Original 2026-08-27 characterization ("5 missing + 3 phantom columns on 0010 alone") and the follow-up 2026-09-01 "confined to 0010_candidates_tables.py alone" claim were BOTH wrong — this is the 3rd characterization of this issue this session, and the only one arrived at by actually running the real provisioning path end-to-end (`psql docs/schema.sql` -> `alembic stamp 0001_baseline` -> `alembic upgrade head`) rather than reading code. Live replay found: (1) `0010`'s divergence from schema.sql is total, not 5+3 columns — different enum type names (`candidate_extraction_status_enum`/`candidate_consent_status_enum` vs. schema.sql's real `extraction_status_enum`/`consent_status_enum`), different column names (`uploaded_by`/`duplicate_of_candidate_id` vs. `created_by`/`is_active`/`consent_id`/`resume_s3_key`), different index/constraint/trigger names, a seed-data mismatch (0010's own `consent_purposes` INSERT would add an unwanted 4th row); (2) `0011` and `0012` are ALSO not clean — both collide too (0011's `candidate_position_matches`/`candidate_source_details` tables + their differently-named indexes/triggers already exist via schema.sql; 0012 assumes 0010's wrong enum names and tries to recreate 0011's table a second time); (3) 3 completely unrelated, genuinely-already-applied migrations (`0047`, `0048`, `0061` — dated as recently as TODAY) also collide, because schema.sql is kept hand-current with each shipped migration's additive columns/indexes, a pattern with no reason to stop at `0012`. | ✅ Closed | **Fixed 2026-09-01** (supersedes the "one new corrective migration" plan below — superseded because live testing proved the divergence far exceeded what a single forward migration could cleanly patch): `0010_candidates_tables.py`/`0011_candidate_matches_source_details.py`/`0012_candidates_schema_bridge.py` rewritten in place. **Corrected framing (post-`principal-reviewer` round 1 — the honest scope, not the overstated one):** it is FALSE that these 3 migrations were "never applied anywhere" — `0011`'s RLS block (byte-identical here) and `0010`'s RLS block (identical except one `USING` predicate, `deleted_at IS NULL` -> `true`, correct since `candidate_documents` has no `deleted_at`) both genuinely ran on real DBs (proven by the pre-fix `ci_schema_snapshot.sql`'s real `pg_dump` output and by `0057`'s own docstring), and `0012`'s 4 column adds are also real (its own prior docstring conceded the snapshot "already carries all 4 columns"). What's actually true, and the real basis for this fix: the table/enum/index/trigger-**creating** DDL in all 3 files provably never ran on any real DB — schema.sql independently ships differently-named final objects — so only that dead-code DDL was removed/guarded; every statement that DID run for real is preserved byte-identically or behavior-identically. Table/enum creation replaced with an existence assertion (raises loudly if schema.sql wasn't loaded first — there was no "creation to skip," so "no-op-guarded creation" was also an inaccurate label, now corrected in the docstrings), 2 phantom enum types + the extra seed row + a duplicate table-create removed outright, RLS enable + policies kept unconditional (genuinely missing from schema.sql, and `0057`/`0059` depend on them existing under these exact names) — now with `DROP POLICY IF EXISTS` before each `CREATE POLICY` (project convention, matches `0059`) so a DB bootstrapped from the new snapshot + stamped at baseline self-heals instead of erroring at `0010`/`0011`. `0047`/`0048`/`0061` (real, already-applied migrations — left untouched structurally per the "don't rewrite 0012+ without proof" rule) got a 1-line `IF NOT EXISTS`/`CREATE INDEX IF NOT EXISTS` guard each; `0047`'s docstring now states its dependency on schema.sql carrying an equivalent CHECK. `0012`'s 4 added columns (`extraction_provider`, `screening_provider`, `ai_analysis_tokens`, `bulk_upload_jobs.updated_at`) are now also folded into `docs/schema.sql` directly, matching how `0047`/`0048`/`0061`'s equivalents are represented there — makes `0012` a permanent guarded no-op like the others, consistent with this project's own schema.sql-currency convention. **Real drift item found and closed in the same fix (not a hypothetical — principal-reviewer's Major 1):** `docs/schema.sql`'s `users.mfa_channel` CHECK was unnamed, so Postgres auto-named it `users_mfa_channel_check`, while `0002_users_mfa_channel_mobile.py`'s own `duplicate_object`-guard checked for a DIFFERENT name (`ck_users_mfa_channel`) that could never collide — the regenerated snapshot had silently baked in a duplicate, functionally-identical CHECK constraint. Fixed by naming the constraint in schema.sql (`CONSTRAINT ck_users_mfa_channel`), which makes `0002`'s existing guard actually fire; re-verified via a fresh full replay + object-name-set diff that the duplicate is gone (see `docs/SCHEMA_CHANGE.md`'s 2026-09-01 entry, corrected). `ruff`/`mypy` clean, full local `pytest` green (1660 passed). See **G15b** for the separate CI-enforcement fix that makes this permanent, and **G15d** below for the residual name-matches-but-definition-differs drift class this incident itself proves still needs closing. |
| G15b | **CI never actually replays the Alembic migration chain against a genuinely empty DB — `backend-ci.yml`/`frontend-ci.yml`/`load-test.yml` all load `docs/ci_schema_snapshot.sql` (a `pg_dump` of an already-correct DB) then `alembic stamp head`, which records the target revision without ever executing `upgrade()`.** This is the STRUCTURAL reason G15's drift went undetected for as long as it did, and the same undetected-drift class already caused 2 separate RLS gaps (`0057`, `0059`) before G15 itself was found. | ✅ Closed (backend-ci.yml) | **Fixed 2026-09-01, same change as G15** (not deferred, per the user's explicit "permanent, not recurring" framing): `backend-ci.yml`'s `test` job's "Load CI schema snapshot" + "Alembic stamp head" steps replaced with "Load docs/schema.sql" + "Alembic stamp 0001_baseline" + a new "Alembic upgrade head (REAL replay)" step. Every PR touching `backend/**` now genuinely replays 0002-through-head against a schema.sql-bootstrapped DB — a future migration reintroducing G15's class of drift fails CI immediately. Chose every-PR over periodic: the replay itself is fast (~10-20s locally, same postgres:18 image CI uses) and schema.sql already seeds all 9 reference tables inline (no separate seed-tail step needed on this path, unlike the snapshot file), so the CI-minute cost concern the original framing worried about did not materialize once actually measured. `frontend-ci.yml`/`load-test.yml` intentionally left on the (now-corrected) snapshot shortcut — out of this change's scope, tracked as a smaller residual gap below. |
| G15c | **`frontend-ci.yml`/`load-test.yml` still bootstrap via the (now-corrected) `docs/ci_schema_snapshot.sql` snapshot + `alembic stamp head`, not a real replay** (added 2026-09-01, deliberately out of scope for G15/G15b to keep blast radius contained to the workflow explicitly named in that fix). These 2 workflows don't touch migration files directly, so the risk is lower than backend-ci.yml's, but they would still silently tolerate a future migration/schema.sql conflict until the next backend-ci.yml run catches it. | 🔴 | Apply the same 3-step replacement (load schema.sql -> stamp 0001_baseline -> upgrade head) to these 2 workflows if/when their own CI-minute budget allows — not urgent since backend-ci.yml now provides the real detection on every backend PR either way. |
| G15d | **`backend-ci.yml`'s new real replay (G15b) only catches a migration whose object NAME collides with schema.sql — it cannot catch a migration whose `IF NOT EXISTS`/existence-check guard matches on name only while the underlying DEFINITION silently differs.** Found 2026-09-01 (`principal-reviewer` Major 1 on G15's own fix): `docs/schema.sql`'s `users.mfa_channel` CHECK was unnamed (auto-named `users_mfa_channel_check` by Postgres) while `0002`'s `duplicate_object` guard checked for a different name (`ck_users_mfa_channel`) that could never collide — the replay ran green, `alembic upgrade head` succeeded, and the regenerated snapshot still ended up with a duplicate, functionally-identical CHECK constraint that G15b's own replay-based CI gate did not and structurally cannot flag, because the guard only ever asked "does an object with this NAME exist," never "does an object with this DEFINITION exist." Closed for this one instance (schema.sql now names the constraint so the two names match), but the general class remained open until fixed below. | ✅ Closed | **Fixed 2026-09-02, corrected same day (round 2, post-`principal-reviewer` Majors 1-4):** `backend/app/scripts/check_schema_definition_drift.py` introspects `pg_catalog`/`information_schema` directly on both the DB the real Alembic replay (G15b) just built and a scratch DB loaded from `docs/ci_schema_snapshot.sql`, now covering columns, indexes, constraints (incl. numeric/expression/boolean CHECK value-sets, not just quoted-string ones -- round 1's normalization silently discarded every non-quoted-literal element, collapsing e.g. `experience_years IN (1,2,3)` vs `IN (4,5,6)` to an identical empty set; fixed to canonicalize a CHECK's `ARRAY[...]` only when EVERY element is a quoted literal, verbatim otherwise), triggers, enum types, **RLS policies** (`pg_policy` predicate/role/command), and **each table's RLS-enabled flag** (`relrowsecurity`/`relforcerowsecurity`) -- round 1 compared neither, so a widened policy predicate (`USING (true)`) or RLS silently disabled on a table would have passed clean; a real tenant-isolation bypass. Moved to run AFTER `pytest` in `backend-ci.yml` (round 1 ran it before, risking the exact early-failure-masks-everything shape the Model-tier & CI-independence mandate exists to close). Added `backend/app/scripts/tests/test_unit_check_schema_definition_drift.py` (11 cases; zero coverage existed before, the exact gap that let round 1's bug hide). **Live proof-of-catch (round 2, fresh replay + fresh snapshot scratch DBs, not the round-1 already-correct state):** clean pass with no false positive; then reproduced ALL FOUR of -- the original `ck_users_mfa_channel` value-set drift, a synthetic numeric-IN-list drift (`experience_years IN (1,2,3)` vs `(4,5,6)`), `ALTER POLICY rls_positions_isolation ON positions USING (true)`, and `ALTER TABLE candidate_documents DISABLE ROW LEVEL SECURITY` -- simultaneously, confirmed the script reports all 4 findings with exit code 1, reverted every mutation, confirmed clean pass again. Also confirmed live: reverting just the normalization fix (leaving the rest) makes 4 of the 11 new unit tests fail, proving the suite would have caught round 1's bug. **Rounds 3-5 (post-round-2 `principal-reviewer` findings, same fix, not a new item):** round 3 fixed the cast-strip regex to handle chained multi-word casts (`'x'::character varying::text`) and doubled-quote-escaped literals (`'c''d'`). Round 4 replaced the round-3 open-ended cast continuation with a closed set (`_MULTIWORD_TYPE`/`_CAST_RE`) after live-confirming the open form swallowed a following SQL keyword (`a::boolean AND b` vs `AND NOT b` both collapsed to `(a)`); re-keyed the `constraint`/`trigger` categories on `(table, name)` instead of name-only (same-named objects on different tables were silently dropped from comparison); added a `RuntimeError` guard in `_fetch_map` for a duplicate identity key; added `main()` exit-code tests. Round 5 (2026-09-02): **M1** -- the module/`_normalize_check_clause` docstrings falsely claimed only a pure-literal `ARRAY[...]` gets its casts stripped; corrected to state casts are stripped CLAUSE-WIDE, and pinned the exact known-masked live pair (`status <> 'on_hold'::interview_status_enum` vs `::position_status_enum`) as a tracked-blind-spot regression test, documented in this row's "does not cover" list below. **M2** -- `backend-ci.yml`'s 3 drift-check steps now gate on `steps.replay.outcome == 'success'` (not just `!cancelled()`), so they never run against a DB whose Alembic replay itself failed/left it half-migrated; `psycopg2.connect` failures now raise a clear one-line `RuntimeError` (`_connect` helper) instead of a bare traceback. Also removed the hardcoded `_FALLBACK_REPLAYED_DSN` literal (`_default_replayed_dsn()` now raises if neither `SCHEMA_DRIFT_REPLAYED_URL` nor `DATABASE_ADMIN_URL` is set -- `backend-ci.yml` always sets the latter), trimming the file to 298 lines (was 306, over the 300-line cap). Test suite now 24 test functions (several parametrized) in `test_unit_check_schema_definition_drift.py`. **What this still does NOT cover** (out of scope, not claimed): sequences, views/materialized views, foreign-key ON DELETE/ON UPDATE actions as a distinct category (covered only incidentally via `pg_get_constraintdef`), RLS FORCE ROW LEVEL SECURITY on non-table relations (partitioned-table children -- `relkind = 'r'` only), and **a CHECK clause differing only in its cast's target type** (e.g. a stale enum type name inside a cast) -- the `enum_type` and `column` categories independently catch this for the enum-rename shape only (where the column's own declared type changes); an expression-internal cast-type change with an unchanged column type (e.g. `CHECK (s::integer > 5)` vs `CHECK (s::numeric > 5)`) is not covered by any category (round 5 M1, documented tracked blind spot, pinned by a regression test). `ruff`/`mypy` clean on all touched/new files; unit tests green. |
| G15e | **The full migration chain's DOWNGRADE path was broken at `0028_position_closed_status_ageing.py`, an untouched, genuinely-already-applied migration — found by `principal-reviewer`'s own reversibility test on G15's fix, not part of G15's scope.** `ALTER TABLE positions ALTER COLUMN status TYPE TEXT` failed with `FeatureNotSupportedError: cannot alter type of a column used in a trigger definition — trigger trg_pos_status_change on table positions depends on column "status"`, surfaced only when downgrading a full replayed chain from head past `0028` (rolled back cleanly under transactional DDL — not a live incident, no real DB was harmed). G15b's new CI replay only exercises `upgrade head`, never a full downgrade — same class of "the chain has never actually been exercised end-to-end" gap G15 itself closed for upgrades, now confirmed to also apply to downgrades. | ✅ Closed | **Fixed 2026-09-02** (branch `fix/g15e-downgrade-trigger-dependency`): the trigger turned out to be only 1 of 3 objects on `positions` that depend on the `status` column and block its type change — each was found live, one at a time, by actually running the downgrade rather than reading code (exactly the discipline this item exists to enforce). `downgrade()` now drops, in order, `trg_pos_status_change` (its `WHEN (OLD.status IS DISTINCT FROM NEW.status)` clause references the column), `ck_position_hold_reason` and `ck_position_noshow_reason` (2 CHECK constraints referencing `status` directly), and the column's own `DEFAULT 'open'` (found last — `DROP TYPE position_status_enum` failed with `DependentObjectsStillExistError` even after the first 3 were fixed) — then restores all 4, verbatim per `docs/schema.sql:313,358-362,500-502`, after the type is rebuilt. **Live-verified full round-trip**: `schema.sql` load → `stamp 0001_baseline` → `upgrade head` → `downgrade 0027_pos_recruiter_assign` → `upgrade head` again, all clean, zero errors. Post-round-trip inspection confirms all 4 objects restored exactly (`enum_range` includes `closed`, both CHECK constraints present, trigger present, `column_default = 'open'::position_status_enum`). `ruff`/`mypy` clean. |
| G16 | **`audit_log` and `interview_status_history` are monthly-partitioned tables with NO automated partition-maintenance mechanism — both `docs/schema.sql` and `docs/ci_schema_snapshot.sql` only define partitions through `2026_08` (August), and zero migration, Celery beat task, or script anywhere in `backend/app/` creates future partitions.** Found live 2026-09-01 (first real CI run after the calendar rolled over to September, triggered by this session's local→origin push once Actions quota reset): `backend-ci.yml`'s `test` job failed at "Seed local test users" with `asyncpg.exceptions.CheckViolationError: no partition of relation "audit_log" found for row` — every `INSERT INTO audit_log` (and, by the same defect, every write to `interview_status_history`) fails the instant the wall clock crosses a month boundary with no partition pre-created for it. This is not a code regression from any recent PR — it is a dormant operational gap that was always going to fire on 2026-09-01 regardless of what shipped, and will fire again on 2026-10-01, 2026-11-01, etc. unless fixed. Confirmed via direct schema read (`docs/schema.sql:927-931` for `audit_log`, `:814-818` for `interview_status_history` — line numbers as of commit `edc5752`, shifted afterward once the fix added more partition DDL) and a repo-wide grep for any partition-creation code outside the 2 migrations that created the *initial* June/July/August partitions — zero hits. | ✅ Closed (local/CI) | **Fixed 2026-09-01**: (1) immediate gap closed by `backend/alembic/versions/0060_audit_log_partitions.py` — adds `audit_log_2026_09` through `audit_log_2027_12` (16 months, mirrors `0030_ivw_hist_partitions`'s horizon); live-verified upgrade/insert-into-Sept-partition/downgrade/re-upgrade round-trip against local Postgres. (2) ongoing automation: new daily Celery beat task `app.shared.partition_maintenance.ensure_partitions` (registered in `celery_app.py`'s `beat_schedule`, `maintenance` queue) tops up BOTH `audit_log` and `interview_status_history` to a rolling 6-month-ahead buffer — this also closes `interview_status_history`'s own latent "breaks again in 2027-12" gap left by `0030`'s one-time fix, not just `audit_log`'s immediate one. Verified live: the task's core logic ran against a synthetic far-future date, created the missing partitions, then a second run created zero (idempotent no-op); the real task ran against today's actual date and logged `partition_maintenance_no_action` for both tables (already covered through 2027-12). **Privilege finding (verified live, not assumed):** the app's normal RLS-restricted role `ats_app` fails with `permission denied for schema public` on this DDL — confirmed by directly running the `CREATE TABLE ... PARTITION OF` statement as `ats_app`; the task instead opens a scoped engine on `DATABASE_ADMIN_URL` (owner role), a new pattern for a Celery task (previously only migrations/scripts used it). `docs/schema.sql` and `docs/ci_schema_snapshot.sql` updated with the new partitions in the same commit (see `docs/SCHEMA_CHANGE.md`). `principal-reviewer` round 1 found the existence-check query itself had a real gap — `relname`-only matching against `pg_class` with no schema/parent scoping, so a same-named decoy relation in an unrelated schema would false-positive as "already covered" and silently skip creating the real partition (proved live: with a decoy present, 6 of 7 required partitions were created and the 7th silently skipped, no error). Fixed to join through `pg_inherits`/`pg_class` scoped to the actual parent table; re-verified live against the exact same decoy scenario (now creates all 7). See G17 for a separately-tracked gap this fix surfaced but does not close: the production deployment prerequisites (Terraform `DATABASE_ADMIN_URL` wiring, RDS default-privilege grants) this task now depends on. |
| G17 | **`partition_maintenance.py`'s daily beat task (G16's fix) needs `DATABASE_ADMIN_URL` in the WORKER's ECS task definition, and `ALTER DEFAULT PRIVILEGES` applied on any fresh RDS instance — neither is in Terraform today.** Found 2026-09-01 during G16's fix review. `infrastructure/terraform/modules/ecs/main.tf`'s worker task def passes only `CELERY_BROKER_URL`/`CELERY_RESULT_BACKEND`/`PROMETHEUS_MULTIPROC_DIR` — no `DATABASE_URL`, no `DATABASE_ADMIN_URL`, no `secrets` block (this is a **new** hard requirement G16 introduces — every prior Celery task ran under the RLS-scoped `ats_app` session only). Without it, the new task raises `RuntimeError` and dies on every daily tick in the only environment that matters. `beat` itself does NOT need this credential — it only publishes the schedule message; the task body and DB connection run on the worker that consumes it (corrected during review — an earlier draft of this row said "worker AND beat"). Separately, `ats_app`'s write access to auto-created partitions comes from `pg_default_acl` (`ALTER DEFAULT PRIVILEGES ... GRANT ... TO ats_app`), applied today only via `docs/LOCAL_DEV.md` + all 3 CI workflows' manual setup steps — absent from Terraform, so on a fresh RDS instance every partition the maintenance task creates would exist but be unwritable by the app, reproducing the G16 incident in a new shape. Also worth flagging as a deliberate, recorded trade-off rather than a silent widening: handing the DB **owner** credential to the worker container increases blast radius beyond the least-privilege `ats_app` design — accepted because partition DDL structurally requires ownership and no narrower grant exists for `CREATE TABLE ... PARTITION OF`. **Observability gap, same item**: the task has no alarm/metric of its own — a silently-failing daily tick (e.g. from this exact missing-credential gap) would go undetected until the 6-month buffer runs out. `docs/RUNBOOK_ASYNC_PIPELINE.md`'s §4 Escalate has a triage entry for this task's failure signature in the meantime. | 🔴 | Same class of gap as G1 (IAM skeleton)/G10 (API Fargate service missing) — this project's Terraform has never been exercised for the worker service's full runtime config. Needs: (a) add `DATABASE_ADMIN_URL` as a Secrets-Manager-sourced env var on the worker ECS task def only, (b) add an `ALTER DEFAULT PRIVILEGES` step to whatever provisions a fresh RDS instance (a bootstrap SQL script or a `null_resource`/`aws_rds_cluster` provisioner — needs a real decision, not a guess), (c) a metric/alarm for task failure (e.g. a Celery task-failure signal already exists for the other beat jobs per the runbook — extend that same mechanism here rather than inventing a new one). Not blocking local/CI (G16 is fully closed there) — blocking real AWS go-live only. |
| G18 | ~~`0027_pos_recruiter_assign.py`'s `downgrade()` fails with `FeatureNotSupportedError`...~~ **DUPLICATE, mis-attributed — corrected 2026-09-02.** The real file is `0028_position_closed_status_ageing.py` (not `0027_pos_recruiter_assign.py` — that filename doesn't exist; the actual `0027` file is `0027_position_recruiter_assignments.py`, confirmed via direct read to have zero `ALTER COLUMN`/`status` references at all, so the original premise was factually wrong, not just mis-cited). This is the exact same bug as **G15e** — same migration, same `ALTER TABLE positions ALTER COLUMN status TYPE TEXT` statement, same `trg_pos_status_change` dependency — found independently by an earlier review round on G15's own fix and logged here under the wrong filename before G15e was opened for the correct one. | ✅ Closed (duplicate of G15e) | No separate fix needed — G15e's fix (drop/restore `trg_pos_status_change` + 2 CHECK constraints + the column's own `DEFAULT`, all live-verified via a full round-trip) already closes this. This row kept only so a future search for the old (wrong) filename lands here and points to G15e instead of re-investigating from scratch. |
| G19 | **`backend/app/modules/interviews/_calendar_tasks.py`'s calendar-invite Celery task runs a raw SQL query against `interview_panelist_assignments.deleted_at` — a column that does NOT exist in the correct schema.** Confirmed: absent from both `docs/schema.sql`'s `CREATE TABLE interview_panelist_assignments` (no `deleted_at` — only `id`/`interview_id`/`sequence_number`/`panelist_user_id`/`name`/`email`/`mobile_isd`/`mobile_number`/`created_at`) and the `InterviewPanelistAssignment` ORM model (`backend/app/modules/interviews/models.py:75-94`, same column set) — both agree, so this is a genuine application-code defect, not a schema/migration gap. Surfaced 2026-09-01 only because that day's Celery-worker CI fix (`backend-ci.yml`) started actually running Celery tasks in CI for the first time ever, combined with G15's real-replay schema (no phantom `deleted_at` column left over from any historical drift to mask the bad query). Every invocation logs `calendar_invite_task_error: column "deleted_at" does not exist` and silently no-ops (fire-and-forget task, swallows the exception, fails no test, blocks nothing) — meaning calendar invites for interview panelists are very likely silently failing in real production today too, not just in CI. | ✅ Closed | **Fixed 2026-09-02** (branch `fix/g19-calendar-invite-deleted-at-bug`): root-caused via `cavecrew-investigator` — `interview_panelist_assignments` has no soft-delete concept at all (`remove_panelist()` hard-DELETEs assignment rows), so an assignment row existing already fully captures "still assigned"; the `deleted_at IS NULL` clause was dead weight referencing a column that never existed, present since the file's creation (commit `2ab2cdde`, PR #215). Removed the clause (minimal fix — deliberately did NOT add a speculative JOIN to check the panelist's own `is_active` status in the separate `interview_panelists` master-directory table, since the query never joined there and nothing in the investigation supported that being the original intent). New regression test `backend/app/modules/interviews/tests/test_functional_calendar_invite_panelist_query.py` (RUN_DB_TESTS=1, real Postgres — the ONLY way to catch this class of bug, since every existing unit test in this module mocks `session.execute` and would return canned data regardless of the SQL text) seeds a real interview + 2 panelist assignments and calls `_load_calendar_payload` directly. **Mutation-tested twice** (once during the fix, once independently re-verified): pass → reintroduce `AND deleted_at IS NULL` → fails with the exact real-world `UndefinedColumnError` → revert → pass again. Includes a post-rollback verification test confirming the fixture's transaction rollback actually leaves zero residue (Rule 5). Full module suite (254 passed/170 skipped), `ruff`/`mypy` clean. |

### 0.2 Scope decisions — need the user's call, not more code

| # | Decision | Status | What it moves |
|---|---|---|---|
| D1 | Phased vs. full-scope go-live for Onboarding/Consent(DPDP)/Data-Privacy | ❓ Awaiting user | `data_privacy.enforce_data_retention` is a literal no-op today — a live DPDP compliance gap if launch requires it live, not cosmetic. Likely the single biggest date-mover. |
| D2 | Notifications completeness at launch | ❓ Awaiting user | Only 2 events (interview.scheduled/offer.sent) wired via SES, feature-flagged OFF pending SES provisioning. SMS/task-center/digests/escalation/failed-delivery-retry all unbuilt — spec itself requires failed-delivery handling before production enablement. |
| D3 | AI-backend Plan A/B/C (2026-08-08) — replaces the earlier "Bedrock-vs-Gemini" framing (Gemini is an explicit local-testing-only stopgap on the user's own key, not a real production candidate) | ❓ Awaiting legal/DPO on A+B | See §0.3 table below. Recommended sequencing: build toward A now, get G9's legal answer in parallel, keep C tested as the safety net. |
| D4 | Reporting completeness at launch | ❓ Awaiting user | Positions-ageing + Interview-Pipeline-Progress are live; 9 other report endpoints (candidate-uploads, screening, pipeline lifetime matview, etc.) are still stub. |
| D5 | Integrations (SSO, HRIS, calendar, job boards, vendors, e-signature) at launch | ❓ Awaiting user | None built. Likely all post-launch, but needs an explicit confirm. |

### 0.3 AI-backend Plan A/B/C (2026-08-08) — detail behind D3

| Dimension | Plan A — Bedrock CRIS (ap-south-1) | Plan B — Direct Anthropic API | Plan C — Local-only / offline |
|---|---|---|---|
| What it is | App/DB/S3 stay in `ap-south-1`; AI calls route via Bedrock's Global Cross-Region Inference to reach Claude | App/DB/infra stay `ap-south-1`; only the AI call goes directly to Anthropic's own API | Every AI feature runs on its already-built local heuristic path as primary, not fallback |
| Cost ($/token) | Parity with direct Anthropic pricing — no Bedrock markup | Same $/token as Bedrock (Anthropic's pricing-parity policy) | Zero per-call vendor cost |
| Real cost driver | Model tier choice: Haiku 4.5 for volume features, Sonnet 5 only for interview-kit gen, never Opus | Same model-tier guidance as A | Lower AI-output quality → more recruiter correction time (a real cost, just not a vendor invoice line) |
| Billing shape | Rolls into existing AWS invoice | Separate Anthropic invoice/contract | No AI vendor invoice at all |
| Cross-region data-transfer fee | Possible (CRIS routing through AWS's own plumbing) | None (direct HTTPS to Anthropic, bypasses AWS routing) | N/A |
| DPDP / data-residency | **Open — G9, needs legal/DPO sign-off.** PII leaves India during inference. | **Does NOT dodge the question — just moves it.** Needs its own check against Anthropic's DPA. Not a free compliance pass. | **Zero exposure.** Nothing leaves your infrastructure. |
| Infra ask | Bedrock model access + throughput quota, IAM Bedrock role | Secrets-Manager-held Anthropic API key — simpler than A, no Bedrock quota request | None — no external AI vendor to provision |
| Switching cost | Near-zero — `llm_gateway` provider env-var flip | Near-zero — same mechanism, `anthropic` provider already built | Zero — it's the system's existing default degrade path |
| When to use | Default direction if G9 clears | If G9 fails on Bedrock specifically but Anthropic's own DPA is acceptable | Safety net regardless — launch here if both A and B are blocked, upgrade later |

### 0.4 Module completion matrix (2026-08-08) — binding: update inline the moment ANY PR lands, same
turn/commit as the merge, per Progress capture. This is the durable per-module UI/Backend/API/DB
truth table — check here before asking "is X done," don't re-derive from GO_LIVE_CHECKLIST.md by
hand.

| # | Module | UI | Backend | API | Database | Notes |
|---|---|---|---|---|---|---|
| 1 | Security / Platform (auth, MFA, RBAC, RLS) | ✅ | ✅ | ✅ | ✅ | Live foundation everything else depends on. |
| 2 | Organizations | ✅ | ✅ | ✅ | ✅ | Live — CRUD, dedup, RLS, audit. |
| 3 | Departments | ✅ | ✅ | ✅ | ✅ | Live — org-nested CRUD. |
| 4 | Positions / Requisitions | ✅ | ✅ | ✅ | ✅ | Live — full lifecycle, budget, JD, panelists, levels. |
| 5 | Interview Panelists (global master list) | ✅ | ✅ | ✅ | ✅ | Live — CRUD, dedup, RLS. |
| 6 | Candidates (profiles, dedup, PII encryption) | ✅ | ✅ | ✅ | ✅ | Live — backend + UI, resume-parsing AI wired. |
| 7 | Applications (pipeline, stage transitions) | ✅ | ✅ | ✅ | ✅ | Live — full 26-status lifecycle. |
| 8 | Screening (knockout, AI match/rank) | 🟡 | ✅ | ✅ | ✅ | Backend + decision layer live; **UI not built.** |
| 9 | Interviews (scheduling, scorecards, feedback) | ✅ | ✅ | ✅ | ✅ | Live — all phases, AI kit generation included. |
| 10 | Offers / Approvals | ✅ | ✅ | ✅ | ✅ | Live — state machine, compliance engine, PDF+S3. |
| 11 | Onboarding / Preboarding | ⬜ | ⬜ | ⬜ | ⬜ | Nothing built — only an enum marker on `positions`/`applications`. No dedicated table. |
| 12 | Consent (DPDP) | ⬜ | ⬜ | ⬜ | 🟡 | Spec written; tables (`candidate_consents`, `consent_purposes`) exist in schema — **unused**, no backend wired. |
| 13 | Data-Privacy (retention/deletion/export/legal hold) | ⬜ | ⬜ | ⬜ | 🟡 | `data_subject_requests` table exists; retention-enforcement task is a literal no-op — live DPDP gap, feeds D1. |
| 14 | Reporting / Analytics | 🟡 | 🟡 | 🟡 | ✅ | 2 of 11 report endpoints live (positions-ageing, interview-pipeline-progress); matviews for the rest exist, unused. |
| 15 | Notifications | ⬜ | 🟡 | 🟡 | ✅ | Only 2 events wired via SES, **feature-flagged off**. No in-app task center/digest UI. |
| 16 | Integrations (SSO, HRIS, calendar, job boards, vendors, e-sign) | ⬜ | ⬜ | ⬜ | ⬜ | Nothing built at all. |

**10 of 16 modules fully live end-to-end.** The remaining 6 are exactly what D1/D2/D4/D5 above are
scoping decisions about.

### 0.5 AWS infrastructure request checklist (2026-08-08) — for IT

| Category | What to request | Detail / Why | Status |
|---|---|---|---|
| Account & Access | Dedicated AWS account (or sub-account under an Org) | Separate billing/quotas/IAM from any other project | Open — user action |
| Account & Access | IAM OIDC identity provider (GitHub Actions trust) | Avoids long-lived access keys for CI/CD | Open — blocks `deploy.yml` (G2) |
| Account & Access | Scoped deploy role | ECS/ECR/RDS/ElastiCache/S3/Secrets-Manager/CloudWatch only, not account-wide admin | Open — blocks `terraform apply` (G1) |
| Account & Access | Break-glass human admin access | Time-boxed, audited, separate from the deploy role | Open |
| Region | `ap-south-1` (Mumbai) | Already the project's documented choice — India DPDP data-residency driver | **Reconfirm** given the Bedrock CRIS finding (G9) |
| Networking | VPC, public+private subnets, 2+ AZs | Required for RDS/ECS Multi-AZ resilience | Open |
| Networking | NAT Gateway(s) | Private-subnet egress | Open — recurring cost line, not one-time |
| Networking | Security groups per component | Tighten `elasticache`'s SG (currently whole-VPC-CIDR) before go-live | Partially authored, needs tightening |
| Compute | ECS Fargate cluster | No EC2 fleet to patch — already the chosen model | Worker/beat authored; **API service not yet built (G10)** |
| Database | RDS PostgreSQL 16, Multi-AZ + read replica | `db.r6g.large` recommended (NFR-grounded sizing, see G10) | Terraform skeleton only, no instance class chosen |
| Cache/Broker | ElastiCache Redis, Multi-AZ, transit-encrypted | `cache.t4g.small` already right-sized | Partially authored |
| Storage | S3: resumes/JD, offer PDFs, Terraform state, logs | KMS-encrypted, versioned, lifecycle-tiered | Open |
| Secrets | AWS Secrets Manager | DB creds, JWT secret, SES/Anthropic keys | Open |
| AI | Bedrock model access (Claude) | Opt-in per account/region via console | **Hold — pending G9 legal/DPO answer** |
| AI | Bedrock throughput quota increase | New-account default quotas often low | Hold, same as above |
| Email | SES production access | Currently sandboxed; needs domain DKIM/SPF/DMARC + a support case | Open — blocks Notifications going live |
| DNS/TLS/CDN | Route53 zone, ACM cert, CloudFront + WAF | Terraform skeletons exist, unapplied | Open |
| Observability | CloudWatch (partially wired) | 3 pipeline metrics + alarms already exist | Confirm if Sentry needs its own paid quota |
| CI/CD | ECR repository | Ties to the OIDC deploy role above | Open |

### 0.6 Already-tracked items feeding into this decision (cross-references, not duplicated here)
- Prompt caching (#8) and cost/token-tracking + daily digest (#9) below — both explicitly scoped to land alongside/just-before the Bedrock cutover (D3).
- The AWS Terraform apply-blocking prerequisite (G1) and the `terraform-plan.yml` CI gap (#6 below) are the same underlying AWS-credentials/IAM gap surfacing in two places (build-time CI check vs. real apply) — resolving G1/AWS-account-provisioning likely closes both at once.

---

## 1. Active execution queue (user-ordered, month-end push — one PR each)

| # | Item | Status |
|---|------|--------|
| 1 | NFR Phase 3 — repo-wide dead-code + missing-docstring sweep | ✅ Live on production (PR #195) |
| 2 | Notifications real fan-out — email via AWS SES for interview.scheduled/offer.sent | ✅ Merged (PR #196) |
| 3 | CI test gap — Postgres/Redis services not provisioned (18 backend tests fail in CI) | ✅ Closed (PR #221, 2026-08-08) — real `postgres:18`/`redis:7` services + `docs/ci_schema_snapshot.sql` + `alembic stamp head` + `RUN_DB_TESTS=1`. Surfaced 72 pre-existing `needs_db` failures never run in CI before; 23 fixed live, 49 tracked as a new §5 entry. |
| 4 | Frontend test debt — nav-items.test.ts, position-schema.test.ts, others (6+ pre-existing failures) | ✅ Closed (`dev/frontend-component-test-debt`, 2026-09-05) |
| 5 | e2e CI job-design gap — MSW can't intercept proxied backend calls | ✅ Closed (PR #221, 2026-08-08) — real backend + Postgres/Redis boot before Playwright's `webServer`; `BACKEND_ORIGIN` feeds `next.config.mjs`'s proxy rewrite, closing the server-side ECONNREFUSED gap. `auth.spec.ts`'s stale "welcome back" assertion updated in the same PR. |
| 6 | `terraform-plan.yml` CI check has no path to ever pass — no AWS-credentials step exists anywhere in the workflow (confirmed via `git log`: file untouched since the original project-scaffold commit `537b06d`), so `terraform init -backend-config=environments/<env>/backend.hcl` cannot authenticate to the real S3 backend. **Discovered 2026-08-07 (PR #219, async-pipeline-durability Phase 6)** — this is the FIRST PR in the project's history to touch `infrastructure/terraform/**`, so the workflow's `on.pull_request.paths` filter never triggered it before now. Not caused by Phase 6; not fixable within any single PR's scope — needs a real AWS account, GitHub Actions secrets (`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` or an OIDC role), and an `aws-actions/configure-aws-credentials` step added to the workflow, which is an infra/credentials decision requiring explicit user approval, not a code fix. Every future PR touching Terraform will show this same 3-way (`dev`/`staging`/`prod`) failure until it's addressed. | 🔴 Queued (6th) |
| 7 | Local dev-stack watchdog (`scripts/dev-stack-watchdog.ps1`) — item (a)'s original 2026-08-08 finding turned out to be a symptom of a much more severe root cause, found and fixed 2026-08-27: the Startup-folder shortcut auto-launching `-Watch` at every Windows logon has existed since 2026-08-04 (confirmed via the shortcut's own file timestamps); the specific leaked-process trail found this session traces back to the most recent reboot/logon on 2026-08-24 (Windows' own process table only reaches back to the last boot). Running 24/7 whether or not the platform was in active use, its `Start-Backend`/`Start-Celery`/`Start-CeleryBeat` functions spawned a brand-new process on ANY failed health check without first confirming the existing one was actually dead — under sustained load (health-check timeouts: 3s backend, 5s Celery ping, both too tight), this produced duplicate Celery worker/beat PID pairs roughly every 90s (directly observed this session) and left 20+ uvicorn processes bound to or attempting to bind port 8000 (most of these are processes that failed to bind — `EADDRINUSE` — plus `--reload` child processes on a socket inherited from an already-dead parent, not 20+ processes all successfully sharing the bind), eventually exhausting Postgres's connection pool twice in one session (97/100 held by `ats_app`, blocking all login/functional tests both times). Item (b) (the venv→system-Python re-exec quirk) is not uvicorn-specific as originally logged — the same pattern was observed on the Celery worker and beat launchers too. **Fixed**: (1) `Test-Backend`/`Test-Celery` timeouts widened to 8s/10s; (2) `Start-Backend`/`Start-Celery`/`Start-CeleryBeat` now check for an existing live process via `Win32_Process` command-line match (not a port-listener or pidfile check — both proved unreliable: a dead process can still own a LISTEN socket, and reusing the pidfile-based `Test-CeleryBeat` as its own beat guard was a no-op, since that's the exact check that had just failed) before spawning, and skip the spawn if one is found, reporting the service unhealthy instead of silently masking it; (3) the Startup-folder auto-launch shortcut disabled (renamed `.lnk.disabled`, not deleted — reversible) — the always-on background loop is replaced by the pre-existing one-shot mode, run manually whenever extraction/JD analysis isn't working, per user's explicit request to stop unnecessary always-on background consumption. Both guard designs and the disabled-shortcut claim were live-verified (principal-reviewer round 1 caught the port/pidfile-check flaws; round 2 re-verified the corrected process-match guards against live repro of each failure mode). Item (b) remains unfixed, still believed harmless, not investigated further. | ✅ (a) fixed 2026-08-27; (b) still 🟡 low priority |
| 8 | **Prompt caching for the 4 AI-calling agents** (JD extraction, screening-question generation, candidate matching, interview-kit generation) — none of the 4 currently use `cache_control` (confirmed via repo-wide grep, zero matches), so the large, mostly-static system-instruction block each agent sends gets paid for in full on every single call, even though the same instructions repeat across hundreds/thousands of calls with only the job-description/resume/candidate content actually varying. **Scope (2026-08-08):** (a) split each agent's prompt into a static "instructions" segment (rubric, output-schema instructions, few-shot examples if any) + a dynamic "content" segment (the actual JD/resume/candidate text) — this restructuring is the real work, not the caching call itself; (b) mark the static segment with `cache_control: {type: "ephemeral"}` in `llm_gateway_providers.py`'s Anthropic/Bedrock call sites (Gemini's `google-genai` SDK has an analogous context-caching API, different shape — check `google.genai.types` for the equivalent before wiring it, don't assume the Anthropic shape carries over); (c) verify empirically (per the binding live-verification mandate) that a cache hit actually reduces billed input tokens for at least one real call, not just that the API accepts the parameter — a silently-ignored cache_control that still bills full price would be worse than not building this, since it would look optimized without being optimized; (d) Bedrock-specific note: prompt caching support and minimum cacheable-prompt-length vary by model within Bedrock (check the specific Claude model version's documented cache support before assuming parity with direct Anthropic API). **Where this lands:** JD extraction and interview-kit generation are the best first targets (longest, most static system prompts per the go-live checklist's own feature descriptions — 1-5 rubric + 10 screening Qs + 10 focus areas × 5 Q&A for kits); screening-question generation and matching likely have shorter/more-variable prompts, lower priority. Not blocking — do this as its own scoped phase, ideally timed alongside or just after the AWS Bedrock go-live migration (Bedrock's actual model+pricing will be known by then, informing exact savings estimate). | 🟡 Queued, scoped, high value |
| 9 | **Live AI cost/token-usage tracking + daily admin email digest.** User's ask: consolidate token/cost usage across all AI-calling features into a live view, and email ALL admin-role users a daily digest at 22:00 IST, timed for just before the AWS Bedrock go-live migration. **Feasibility: YES**, and every piece the mechanism needs already exists in this codebase, confirmed 2026-08-08: (a) per-call token counts are already captured in code (`llm_gateway_providers.py`, `level_kit_agent.py`) — currently discarded after the call, not persisted anywhere; (b) real email fan-out via AWS SES already exists and is live (BACKLOG #2, PR #196) — this is a new notification TYPE on an already-built delivery mechanism, not a new integration; (c) Celery beat + timezone-aware crontab scheduling already exists (Phase 2 of `async-pipeline-durability`, PR #215) — a `22:00 Asia/Kolkata` (IST has no DST, fixed UTC+5:30 year-round, so a plain UTC crontab offset is safe with no seasonal-drift risk) beat entry is the same pattern already in `celery_app.py`'s `beat_schedule`; (d) admin-user enumeration is a straightforward query against the existing RBAC roles in `app/modules/security/` (`super_admin`/`hr_admin` or whichever role set counts as "Admin" — confirm the exact role list with the user when this is actually scheduled for build, don't guess it now). **Design decision to make when building (not now):** self-tracked token-count-based cost ESTIMATE (persist per-call token counts + known per-model $/token pricing, compute an estimate — simpler, no AWS API dependency, but an estimate can drift from the real bill if pricing changes or a call type is missed) vs. a live pull from the AWS Cost Explorer API (`ce:GetCostAndUsage`, authoritative real billed $ broken down by service/model if cost-allocation tags are set up on the Bedrock resources — but Cost Explorer data has a documented ~24h refresh lag, which is actually FINE for a "yesterday's spend" daily digest at 22:00, just not usable for true real-time). Recommend Cost Explorer as the authoritative source for the EMAIL digest (real $ the user actually cares about) plus the self-tracked token counters as a supplementary real-time signal for the "live usage" view mentioned in the ask (a dashboard/metric, not the email) — this two-source design avoids the self-tracked estimate silently drifting from the real bill while still giving a live (not 24h-stale) number somewhere. **Not blocking now** — explicitly scoped by the user to be picked up just before AWS Bedrock go-live, once the real model/pricing/IAM setup is known. | 🟡 Queued, scoped, deferred to pre-Bedrock-go-live |
| 10 | ~~Downstream AI calls (matching, screening) fire unconditionally even when extraction just flagged the row a duplicate.~~ **CLOSED — MOOT (confirmed 2026-08-29).** Originally logged 2026-08-08 against `_run_extraction`'s then-unconditional `celery_app.send_task("candidates.match_candidate_to_positions", ...)` / `"candidates.screen_candidate"` calls. Re-read the current code: `_extraction_tasks.py`'s `_run_extraction` no longer calls `send_task` at all — `candidates-ai-match-screen-consolidation` (CR#1, merged PR #223, 2026-08-18/19) retired the automatic post-extraction match/screen fan-out entirely. Matching is now a single on-demand `POST` trigger (`CandidateService.trigger_match` → `_enqueue_matching`, called only from the explicit "AI Job Match" UI action, never from extraction); the `candidates.screen_candidate` task doesn't exist anywhere in the codebase anymore (screening's own match-decision write path was retired by the same CR). No AI-cost waste on duplicates remains to fix — the architecture change already closed this independently of the original ask. | ✅ Closed (moot) |
| 11 | **Tier the e2e job's 4-browser-project matrix to cut CI minutes** — user's ask 2026-08-08, after PR #221's e2e fix alone burned an est. 150-250 of GH Actions' ~2000 min/month allotment across 6 reruns. **Feasibility: YES** — Playwright supports this natively via project-level `grep`/`grepInvert` tag filtering: tag critical/NFR specs (`auth.spec.ts` — gates every downstream flow; `a11y.spec.ts` — binding WCAG mandate) `@critical`, run those across all 4 projects (chromium/webkit/mobile/reduced-motion) as today; run everything else (organizations/positions/pipeline-retry-badge — routine functional coverage) on 1-2 projects only (e.g. chromium, optionally +mobile) via `grepInvert`. **Blocking prerequisite (largely satisfied 2026-08-08, not fully):** the reduced matrix must run "without any errors" per the user's own requirement — the deterministic root cause of the 2 flaky webkit tests (§5) is fixed (`fix/e2e-webkit-flake-prod-build`, PR #222), cutting the flake from every-run to 1-in-4 runs, but NOT to zero — a residual believed to be ordinary webkit-on-CI network timing remains, absorbed by `retries: 1`. If this reduced matrix is ever built with webkit still in the routine-tier bucket, that bucket would inherit this same residual, not a genuine zero-error guarantee — re-check the actual rate at build time rather than assuming §5's "largely fixed" note still holds. **Still needs, before building:** the user's own critical-vs-routine spec classification — my own guess (auth+a11y critical, rest routine) is a starting proposal, not a decision. Not built — scoping only, per user's explicit "not doing it now." | 🔴 Queued, scoped, needs user's critical-spec classification |

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

- 🔴 **`positions/_service_writes.py`'s create-position write path passes a caller-supplied
  `organization_id` straight into `PositionRepository.set_org_scope()`, which moves
  `app.current_org` to whatever org the request claims — so for a non-internal (org-scoped)
  user, RLS's `rls_positions_isolation` check ends up validating against the SAME
  request-supplied value, not the actor's own organization** (found 2026-08-29,
  `principal-reviewer`, while confirming the N5 perf-cleanup decision above didn't hide a
  correctness issue). `rls_positions_isolation` is a `FOR ALL` policy with only a `USING`
  clause, so Postgres defaults `WITH CHECK` to the same predicate — meaning nothing outside this
  code path re-validates `organization_id` against the actor's own org before the INSERT/UPDATE
  goes through. `OrganizationService.ensure_exists` only confirms the target org EXISTS, not
  that the actor may act on it; `organizations` itself has no RLS policy at all. **Currently
  latent, not live**: every user in this single-tenant STG-Labs deployment is internal
  (`core/dependencies.py`'s `is_internal = user.organization_id is None`), so the non-internal
  branch where this matters is never exercised today, and there's no `FORCE ROW LEVEL SECURITY`
  or dedicated non-owner app role configured either — RLS may be inert for the current DB
  connection regardless. Needs its own scoped authz fix (an explicit actor-org check on this
  write path) before any multi-tenant/non-internal-user rollout — NOT closed by the N5
  perf-cleanup decision above, which correctly left `set_org_scope` in place but for the wrong
  original reason (see N5's corrected entry).
- ✅ **`candidates` table has no `UPDATE`/`ALL` RLS policy for the `ats_app` role — only `candidates_read_all` (SELECT)** (2026-08-25, `dev/tech-debt-batch2-data-query`) — was worse than originally logged: INSERT was also silently blocked (no `INSERT`/`ALL` policy either), not just UPDATE. Fixed via new `candidates_insert_all`/`candidates_update_all` RLS policies (migration `0057_candidates_write_rls.py`) plus widening `candidates_read_all` from `USING (deleted_at IS NULL)` to `USING (true)` — matching every other RLS-protected table's app-layer-filtering pattern instead of relying on RLS to hide soft-deleted rows. That widening meant 6 other read paths were silently relying on RLS for `deleted_at` filtering and needed an explicit app-layer guard added: `interviews/_kit_context.py` (LEFT JOIN + `AND c.deleted_at IS NULL`), `applications/repository.py`, `offers/repository.py`, `offers/tasks.py`, `candidate_screenings/repository.py`, `candidates/_extraction_tasks.py`, `candidates/_matching_tasks.py`. Live-verified via a real probe against the local dev DB, and by fixing the actual tracked stale orphan this entry originally flagged (`FT-SoftDelKit-96f844ca`, now cleaned up).
- ✅ **`candidate_documents`/`bulk_upload_jobs`/`candidate_consents` had no INSERT/UPDATE RLS policy for the `ats_app` role — same bug class as the `candidates` entry above** (2026-08-27, `dev/tech-debt-rls-candidate-docs-consents`) — found live by functional-test-engineer during Tier-3 code-hygiene work (`dev/hygiene-tier3-backend-batch1`), reproducing `POST /candidates/upload` as a 500 `InsufficientPrivilegeError` on `candidate_documents`. Migration 0010 enabled RLS on all 3 tables but only ever defined `candidate_documents`'s SELECT policy (`candidate_docs_read_all`) — `bulk_upload_jobs`/`candidate_consents` had zero policies for any command, ever; live introspection showed even `candidate_docs_read_all` was absent on this environment (`pg_policies` returned zero rows for all 3). Fixed via `0059_docs_bulkjobs_consents_rls.py` — 8 new `USING (true)`/`WITH CHECK (true)` policies (SELECT+INSERT+UPDATE on `candidate_documents`/`bulk_upload_jobs`; SELECT+INSERT only on `candidate_consents` — no UPDATE policy there since no withdraw/UPDATE endpoint exists yet, per principal-reviewer's CHANGES-REQUESTED round 1: don't widen RLS on a DPDP proof-of-consent table with no code path behind it). No DELETE policy on any (no production code path hard-deletes these entities — the hard DELETEs that do exist are test/script cleanup only, 3 sites via the owner role which bypasses RLS, 2 via `async_session_factory`/`ats_app` which silently no-op under RLS, see the new tracked entry below). Live-verified as the actual `ats_app` role (`SET ROLE` inside a rolled-back admin transaction): INSERT+UPDATE+SELECT succeeded on `candidate_documents`/`bulk_upload_jobs`, INSERT+SELECT on `candidate_consents`. Downgrade↔upgrade round-tripped cleanly, row counts unchanged. Unit suite 266 passed; functional tests (`test_functional_bulk_upload.py`, `test_functional_duplicate_file_error.py`) 7/7 passed in isolation. **CI cannot regress-test this fix** — `backend-ci.yml`'s schema bootstrap runs `alembic stamp head` against a pre-baked snapshot with RLS disabled on all these tables; local-dev-verified only, tracked below.
- ✅ `docs/ci_schema_snapshot.sql` (2026-08-08) doesn't enable RLS on `candidates`/`candidate_documents`/`bulk_upload_jobs`/`candidate_consents` at all, and `backend-ci.yml` stamps rather than replays migrations — meaning CI's `ats_app` is unrestricted on all 4 tables and CI has zero coverage for this entire defect class (both `0057` and `0059` escaped to live use as a direct result). **Closed 2026-09-01 as part of G15/G15b** (see `docs/BACKLOG.md` §0.1 G15/G15b, `docs/SCHEMA_CHANGE.md`'s 2026-09-01 entry) — supersedes the interim "regenerated as a byproduct of 0061" note below: `backend-ci.yml`'s `test` job no longer depends on the snapshot at all (it now loads `docs/schema.sql` + genuinely replays `alembic upgrade head`, which includes all candidates-domain RLS policies for real), and the snapshot itself was regenerated a 2nd time from that successful replay, superseding both the 0061-byproduct regeneration and the file's remaining staleness (stale `candidate_source_enum` values, missing `candidate_source_details` constraints, missing `offer_details` table, stale trigger name — see SCHEMA_CHANGE.md for the full diff). **Historical note, interim regeneration (2026-08-08 -> 2026-09-01, since superseded):** the first regeneration attempt used a plain schema-only `pg_dump`, which silently dropped this file's hand-curated seed tail (bootstrap `organizations` row + 8 reference-data `COPY` blocks: `roles`/`permissions`/`role_permissions`/`lookup_values`/`currencies`/`consent_purposes`/`feature_flags`/`tenant_settings`) — this would have left every CI-seeded user with zero permissions, with `seed_dev.py`'s grant step raising no error. Fixed by re-appending the seed tail and verifying non-zero row counts on all 9 reference tables. **This mechanic still applies to the current file** — any future regeneration MUST re-append this seed tail (or, for `backend-ci.yml` specifically, is now moot since that job bootstraps from `schema.sql`, which seeds all 9 tables inline).
- 🟡 `backend/app/modules/candidates/repository.py`'s `get_bulk_job()` reads via `session.get()` and never filters `deleted_at`, even though `BulkUploadJob.deleted_at` is modeled — latent (nothing currently writes it), found during the `0059` RLS review.
- 🟡 **`test_functional_async_pipeline_phase1.py:76,77,93` and `phase3.py:106` hard-delete cleanup silently deletes 0 rows under RLS**, found during the `0059` RLS review's principal-reviewer round 2. Both use `async_session_factory` (the `ats_app`-scoped app connection) to hard-delete `BulkUploadJob`/`CandidateDocument` rows in teardown; since none of `candidates`/`candidate_documents`/`bulk_upload_jobs`/`candidate_consents` has a DELETE policy (correctly — no production code path needs one), `ats_app` DELETEs against them affect 0 rows with no exception, per the same silent-failure asymmetry `0059`'s own docstring documents. `phase1.py`'s `_created_bulk_job_ids` teardown additionally has no post-teardown verification (unlike its `_created_candidate_ids` sibling, which does) — violates the binding Rule 5 post-teardown-verification requirement. Fix: switch both teardowns to the admin engine (the pattern `tests/integration/conftest.py:47` already uses for the 3 sites that correctly bypass RLS) and add the missing post-teardown assert.
- 🔴 **`docs/schema.sql` does not reflect migration `0057_candidates_write_rls`** (found 2026-08-25,
  principal-reviewer review of `dev/tech-debt-batch2-data-query`, Minor 8) — the canonical schema
  doc's `candidates` RLS section still shows only the original narrow `candidates_read_all`
  (`USING (deleted_at IS NULL)`), not the two new INSERT/UPDATE policies or the widened SELECT
  policy this migration ships. Same class as the already-tracked `0011`/`candidate_documents`
  drift entries above (`docs/schema.sql` vs. real migration history). Explicitly out of scope for
  this batch — flagged only, not fixed here; needs its own pass reconciling `docs/schema.sql`
  against the full migration chain, not a one-off patch.
- 🔴 **Local dev Celery worker/beat processes accumulate without being killed on restart — same pattern also confirmed on uvicorn.** Found 2026-08-24 (`dev/fix-screening-questions-quality` functional test pass): 96 duplicate Celery worker/beat processes had piled up since ~13:45 the same day (~90s apart, a respawn-without-kill pattern), unrelated to any specific code change. Separately, same day, 2 independent `uvicorn --reload` processes (different PIDs, different parent PIDs, started ~13:45 and ~16:24) were both found bound/listening on `:8000` simultaneously — backend was still responding correctly (`/health` OK) so not disruptive, left untouched rather than risk killing the wrong one mid-session, but it's the identical root cause. Anyone restarting uvicorn/Celery locally should confirm the OLD process actually died before starting a new one — needs either a startup script that kills-by-port/PID first, or a documented manual check.
- ✅ **`test_functional_21b_question_generator.py`'s `_POLL_TIMEOUT_S = 30` is shorter than this dev environment's real AI-path latency** (2026-08-25, `dev/tech-debt-batch2-data-query`, commit `97e3f11`) — bumped to 90s to clear the ~60s circuit-breaker-trip-then-fallback window this environment's `SCREENING_QUESTION_PROVIDER=gemini` outage produces. `interviews/`'s own `_KIT_POLL_TIMEOUT_S` is a different code path (`LevelKitAgent`'s inline retry, not the shared `llm_gateway` circuit breaker) — left untouched, no change needed there.
- 🔴 **`test_functional_21b_question_generator.py` creates `candidate_screenings` rows with no `finally`/teardown anywhere in the file** (found 2026-08-25 reviewing the above fix, `dev/tech-debt-batch2-data-query`) — a Rule 5 cleanup gap, pre-existing, not introduced by the poll-timeout commit. Flagged for a future pass, not fixed here.
- ✅ `positions/repository.py::get_interview_level()` (2026-07-29, `dev/tech-debt-batch1`) — added `is_active IS TRUE`, matching `child_repository.py::list_levels()`. Sole caller confirmed write-path-only (interview-creation validation); no read/history path affected. Now returns 404 `INTERVIEW_LEVEL_NOT_FOUND` for a deactivated level id at create-time — `interviews/spec.md` 404-trigger line synced in the same branch. Merged with PR #204's multi-panelist eager-load (`selectinload(InterviewLevel.panelists)`) on the same method — both survive together.
- ✅ Dead `sys.path.insert(..., "../../..")` + `import sys` in 3 backfill scripts (`backfill_legacy_feedback_outcome.py`, `backfill_owning_recruiter_id.py`, `backfill_panelist_login_accounts.py`) + `seed_uat_dataset.py` (same dead line, same reason) (2026-08-25, `dev/tech-debt-batch2-data-query`, commit `97e3f11`) — removed from all 4, matching the pattern already applied to the other 2 scripts in `dev/tech-debt-batch1`; imports resolve via the editable install regardless.
- ✅ **`candidates/_extraction_tasks.py`'s persist-UPDATE rowcount check** (added 2026-08-25, commit `97e3f11`) is a related, defensive fix to the `candidates` RLS entry above — added the same rowcount-verification guard the file's own claim-UPDATE already used to the 3 UPDATE calls that persist parsed candidate fields, so a future RLS/access-control regression blocking those writes fails loudly (`extraction_status='failed'`) instead of silently reporting `'completed'` with no data persisted.
- ✅ `screening/repository.py::list_decisions()` (2026-07-29, `dev/tech-debt-batch1`) — added `AND deleted_at IS NULL` to both JOINs, matching `applications/repository.py`'s pattern.
- ✅ `category_rank` SQL duplicated across 3 modules (2026-08-26, `dev/tech-debt-batch3-data-query`)
  — promoted to `backend/app/shared/sql_fragments.py::CATEGORY_RANK_SUBQUERY` (new file, alongside
  `experience_band.py`'s precedent), imported by `interviews/repository.py` (also fixed 3 further
  in-file re-inlinings at `get_level_category_rank`/`get_level_interview_status`/
  `get_offer_gate_levels` that weren't reusing its own pre-existing `_CATEGORY_RANK_SUBQUERY`
  constant) and `applications/repository.py` (previously a deliberate, documented "duplicated
  rather than imported to keep repositories decoupled" copy — resolved properly via the shared
  module instead of leaving the duplication). Correction to this item's own original framing:
  investigation found `reporting/_pipeline_progress_sql.py`'s copy was NOT actually dead despite
  its own docstring's "removed" claim — it was still live, re-exported into
  `_pipeline_progress_group_sql.py`'s `position_sub_dims` CTE (the stg/org select/reject
  status-group query path), so it's the 3rd module fixed here too, not skipped. Verified: Gate 1
  unit tests green (`interviews`/`applications`/`reporting`, no behavior change), plus a live run
  of `test_category_rank_regression.py` (RUN_DB_TESTS=1) — 4/5 pass confirming the relocated SQL
  is byte-identical in output; the 1 failure
  (`test_validate_level_sequence_not_wrongly_gated_by_inflated_org_rank`) is unrelated pre-existing
  staleness (references `get_org_pair_decided`, a gate mechanism removed entirely by the
  2026-07-22 BR-SEQ-001 rewrite — confirmed via repo-wide grep, zero remaining references outside
  this test's own comments and orphaned bytecode) — flagged below, not fixed (out of this item's
  scope).
- ✅ **`test_category_rank_regression.py::test_validate_level_sequence_not_wrongly_gated_by_inflated_org_rank`
  (found 2026-08-26, FIXED 2026-09-03, branch `fix/bucket-b-category-rank-fixture`).** Rewritten
  as `test_validate_level_sequence_names_correct_non_inflated_prior_rank`: proves the same
  inflated-computed-rank concern under BR-SEQ-001's current single-linear-chain rule (the old
  `get_org_pair_decided`/rank>=3 assertion this test used to make no longer applies) — confirms
  `get_level_category_rank` still computes rank 2 (not inflated to 3) despite an inactive
  duplicate, and that `_validate_level_sequence`'s resulting error names the correct,
  non-inflated prior rank's real label. The "succeeds once selected" half is already covered at
  the unit layer (`test_unit_level_sequencing.py`'s parametrized cases) — not duplicated here.
  Same fix batch also closed the other 4 tests in this file — see the next entry.
- ✅ `positions/schemas.py`'s `InterviewLevelRequest`/`InterviewLevelResponse` missing `level_category`
  field (2026-08-26, `dev/tech-debt-batch3-data-query`, BUG-4): added `level_category: OrgType` as a
  real required field on both schemas (previously silently derived from `level_type` in
  `levels_service.py`). Added a `model_validator` on `InterviewLevelRequest` enforcing
  `level_category == level_type` (422 VALIDATION_ERROR on mismatch — `exception_handlers.py`
  hard-codes 422 for every `RequestValidationError`) — the spec is silent on
  divergence and today's behavior never lets the two differ, so the invariant is now validated
  instead of unenforced. `levels_service.py::_build_level`/`_levels_to_response` thread the
  payload/ORM value through instead of re-deriving it. Updated every caller of
  `InterviewLevelRequest`/the `POST .../interview-levels` body across `backend/` (unit tests,
  functional tests, integration tests, seed scripts) plus the frontend
  `InterviewLevelRequest`/`InterviewLevelResponse` TS types and the one call site
  (`interview-levels-editor.tsx`) and MSW mocks that construct them. `openspec/specs/positions/spec.md`'s
  "Known gap" note updated to reflect the gap is closed.
- 🔴 Organizations/Departments DELETE endpoints documented in spec, never built (4 ACs unverifiable).
- ✅ CR-002 multi-panelist-per-level (2026-08-26, `dev/tech-debt-batch3-data-query`): restored the
  `positions/levels_service.py::_levels_to_response()` null-guard on `PanelistSummary.panelist_name`
  (`lp.panelist.name if lp.panelist else None`) — also relaxed `PanelistSummary.panelist_name` from
  `str` to `str | None` in `positions/schemas.py`, since the guard was otherwise unenforceable
  (a required `str` field would still reject `None` at the Pydantic boundary).
- ✅ CR-002 multi-panelist-per-level (2026-08-26, `dev/tech-debt-batch3-data-query`): extracted
  `InterviewLevel`/`InterviewLevelPanelist` (+ their `interview_level_type_enum`/`level_category_enum`
  PGEnum definitions) into `positions/_interview_level_models.py`, matching the precedent set by
  splitting `levels_service.py` out of `subresource_service.py`. `positions/models.py` is now 249
  lines (was 310). Updated every import site: `positions/levels_service.py`, `child_repository.py`,
  `repository.py`, `screening/repository.py`, and `positions/tests/test_repository.py`.
- ✅ CR-002 multi-panelist-per-level (2026-08-26, `dev/tech-debt-batch3-data-query`): changed
  `InterviewLevel.panelists` from `lazy="selectin"` to `lazy="raise"` to match the file's own
  `panelist` sibling convention. Verified first: both real read paths
  (`child_repository.py::list_levels()` and `child_repository.py::get_interview_level()` — the
  latter moved here from `positions/repository.py` in the Tier-3 hygiene split, 2026-08-27) already
  pass an explicit `selectinload(InterviewLevel.panelists)`; the only other `.panelists` accesses are
  on freshly-constructed (unpersisted) `InterviewLevel` instances in `levels_service.py`'s
  `_build_level()`/`set_levels()`, which set the attribute directly rather than lazy-loading it.
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

- 🟡 **`candidate_documents` local dev DB schema divergence, found while fixing an unrelated
  RLS-drift blocker (CR#2 `interview-kit-candidate-aware-scheduled-generation`, 2026-08-19).**
  Migration `0010_candidates_tables.py`'s RLS policy on this table references `deleted_at`, but
  the column doesn't exist on the local dev DB's actual `candidate_documents` table — applying
  the migration's `CREATE POLICY` DDL failed with an undefined-column error while the sibling
  `candidates`/`bulk_upload_jobs`/`candidate_consents`/`candidate_position_matches` RLS fixes
  (same session) succeeded independently.
  **Practical blocker resolved 2026-08-27** (`dev/tech-debt-rls-candidate-docs-consents`,
  migration `0059_docs_bulkjobs_consents_rls.py`, see §4 above): confirmed via
  `information_schema.columns` that `candidate_documents` genuinely has no `deleted_at` column on
  this (and presumably every) environment bootstrapped via `docs/schema.sql`'s shape, and no
  migration ever drops one — 0010's committed file is simply out of sync with what was actually
  applied. `0059` deliberately writes `USING (true)` for this table's SELECT/UPDATE policies
  instead of referencing `deleted_at`, so the RLS fix no longer depends on that column existing.
  Root-cause archaeology remains open (why does 0010's `create_table` for this table not match the
  live shape at all — see also `s3_version_id`, `content_sha256` nullability, `file_type` CHECK —
  same class of drift as the already-tracked `0011_candidate_matches_source_details.py`
  unreplayable-migration entry above) but no longer blocks any real functionality.
- ✅ **BUG-2 — `interviews/tests/test_functional_level_kit.py` fixture drove `applications.status`
  = `'active'` (fixed 2026-08-26, `dev/tech-debt-batch3-data-query`).** Root cause confirmed live
  (not the enum-drift framing originally guessed): `'active'` genuinely still exists as a raw
  Postgres enum label (`pg_enum` lists it), but `applications/models.py`'s `_APPLICATION_STATUS`
  `PGEnum(...)` — the ORM-side value list — has excluded `'active'` since Alembic 0042
  data-migrated existing rows to `'new_application'` and stopped writing it (the file's own
  header comment already documented this: "'active' value persists in DB enum type but is no
  longer written"). Reading the fixture's row back therefore raised an unhandled
  `sqlalchemy.exc.LookupError` deep in row deserialization (`_object_value_for_elem`), surfaced
  to the API as a generic 500 — never a clean validation error. Fixed by changing the fixture's
  raw-SQL insert to `'new_application'` (`test_functional_level_kit.py:139`, in
  `seeded_kit_data`'s psycopg2 insert).
- ✅ **BUG-3 — shared `FuncTest P18B` position's two interview levels had zero panelist
  assignments (fixed 2026-08-26/26, `dev/tech-debt-batch3-data-query`, corrected on review).**
  Confirmed live via direct `InterviewService.create_interview()` reproduction: once BUG-2
  stopped masking it, `_validate_interview_level` (`interviews/_service_helpers.py:164`) raised
  `LevelHasNoPanelistsError` (422, BR-064/CR-002) because `interview_level_panelists` had zero
  rows for `LEVEL_STG_R1`/`LEVEL_ORG_R1` — this is a data gap, not a code defect (CR-002's gate
  is working as designed). Root cause of the gap: `seed_uat_recruitment_funnel.py` and
  `seed_legal_transaction_demo.py` both POST `panelist_id` (singular) to the
  `POST .../interview-levels` body, but `InterviewLevelRequest` only has `panelist_ids` (a list)
  — Pydantic's default `extra="ignore"` silently dropped the key, so every level either script
  created shipped with zero panelists (610/1013 active rows repo-wide). **Correction to an
  earlier revision of this entry:** that revision claimed the fix was "permanent" via a real
  `interview_level_panelists` row inserted for both levels — that row was actually inserted by
  hand during the build session, live, outside version control (`git diff main...HEAD` showed
  no such INSERT), so it was not durable and would fail again on a fresh DB or reseed. Fixed for
  real by (a) an idempotent `_ensure_level_panelists` module-scoped autouse pytest fixture in
  `test_functional_softdeleted_candidate_kit.py` (check-then-insert-if-missing, one row per level,
  no teardown delete — the insert is meant to persist, since the underlying gap is shared-DB-wide,
  not per-test-run), and (b) fixing the actual root cause in both seed scripts
  (`panelist_id` -> `panelist_ids: [<uuid>]`) plus `extra="forbid"` on `InterviewLevelRequest` so
  this class of silent key-drop fails loudly in the future. The former per-test insert+delete
  workaround blocks in `test_functional_softdeleted_candidate_kit.py` were removed
  (`:180-224` and `:310-327`, plus their matching `finally`-block deletes).
  **Scope clarification (2026-08-26, principal-reviewer confirm-pass residual M2-residual):**
  commit `0165ee9`'s "12 sites" count covered `app/scripts/`'s 2 seed scripts only — it did not
  sweep `backend/tests/integration/`, so `test_interview_levels_panelist.py` still carried the
  same pre-CR-002 singular `panelist_id` shape (request body key + response-assertion reads) and
  started hard-422ing once `extra="forbid"` landed. Fixed separately, same day: `_set_levels`
  helper now sends `panelist_ids: [<uuid>]`, and the 3 assertion sites (valid-panelist,
  panelist-omitted, GET-returns-name tests) now read `level["panelists"][0]["panelist_id"/
  "panelist_name"]` per `InterviewLevelResponse`'s real CR-002 shape instead of stale top-level
  fields. The file's pre-existing, unrelated `ORG_NAME_REQUIRED` failure at `_create_panelist`
  (already tracked in the §5 CI entry above) is untouched by this fix. **Live re-run after the
  fix (`RUN_DB_TESTS=1`, 2026-08-26) confirms 5/5 API tests still red, exactly the same bucket the
  §5 entry above already tracks — no new failure class:** `test_set_levels_with_valid_panelist`,
  `..._inactive_panelist_returns_422`, `test_get_levels_returns_panelist_name` hit
  `ORG_NAME_REQUIRED` at `_create_panelist` (as expected); `test_set_levels_with_unknown_panelist_
  returns_404` and `test_set_levels_without_panelist_still_works` skip panelist creation
  entirely and instead hit `MANDATORY_ORG_LEVEL_MISSING` (400) — previously invisible behind the
  singular-`panelist_id` 422 this fix removed, but already named as a tracked mismatch for this
  same file in the §5 "Update 2026-08-18" entry below (`ORG_NAME_REQUIRED`/
  `MANDATORY_ORG_LEVEL_MISSING` validation mismatches) — not a new defect, just unmasked by this
  fix. Root cause: `_set_levels`'s helper posts a single `stg_labs` level only, never the
  Organization L1/L2 pair `validation.assert_level_set_valid` (D9) requires in the same payload —
  a test-design gap predating CR-002, out of scope for this fix.
  **This fix does not mean the whole file is green** — see the 🟡 item immediately below for 2
  newly-surfaced, unrelated failures this fix exposed (Celery queue contention, a BR-SEQ-001
  test-design mismatch).
- 🟡 **New, previously-masked findings surfaced by fixing BUG-2/BUG-3 — NOT fixed, out of scope
  for this pass, flagged for a follow-up ticket:** (a) `test_lk01_schedule_interview_enqueues_
  kit_generation` and `test_lk03_get_level_kit_200_response_shape` now fail on 2 *different*
  issues that BUG-2's earlier 500 always short-circuited before either could be reached: (i) the
  Celery `screening` queue was observed steady at ~36 backlogged messages for 30+s with zero
  drain during this session (shared dev stack, concurrent agent load) — `_poll_level_kit`'s hard
  20s timeout is too tight under that contention, kit status observed stuck at `'processing'`
  (task already picked up, not stuck in queue) rather than failing; (ii) `test_lk03` creates its
  second interview at `LEVEL_ORG_R1` on the *same* shared application `test_lk01` already used —
  `_validate_level_sequence`'s BR-SEQ-001 gate (2026-07-22 rewrite) now correctly blocks Org L1
  until the application's STG-chain interview reaches a Select outcome, which `test_lk01`'s
  interview never does (it stays `pending`/scheduled). This test's design predates BR-SEQ-001 and
  was never compatible with it — masked until now by BUG-2. Both are real, reproducible (not
  flaky — failed identically on 2 consecutive full runs), and independent of BUG-2/BUG-3's fix
  correctness (confirmed via direct in-process `InterviewService.create_interview()` calls: no
  `LookupError`, no `LevelHasNoPanelistsError` after the fix). Gate 1 (`pytest app/modules/
  interviews/ -m "not functional"`) is green: 253 passed, 5 skipped, 0 failed.
- ✅ **`test_functional_level_kit.py`'s `seeded_kit_data` module fixture teardown was missing an
  `application_status_history` delete (found+fixed alongside BUG-2, 2026-08-26).** Masked by
  BUG-2: while `applications.status` stayed unreadably stuck at `'active'`, BR-019's
  auto-set-pending-on-interview-create never fired (its `_STATES_FOR_PENDING_SYNC` gate doesn't
  include `'active'`), so no `application_status_history` row was ever written and the teardown's
  omission never mattered. Once BUG-2 was fixed, status genuinely transitions
  `new_application -> pending`, writing a history row that the teardown's `DELETE FROM
  applications` then violated on FK, silently rolling back the entire teardown (caught by a bare
  `except Exception: conn.rollback()` with no log line) and leaving orphaned `FT-LevelKit-*` rows
  behind on every run. Fixed: added the missing `DELETE FROM application_status_history` in the
  canonical FK-safe order (`test_functional_level_kit.py:194-198`) and replaced the silent
  swallow with a diagnostic print naming the failed `app_id`, so a future teardown break is
  visible instead of silently leaving stale data (CLAUDE.md Rule 5 / NFR Observability).

- ✅ **`interview_level_kits` has no unique constraint on `interview_id`, allowing a real race to
  create 2 rows for one interview (found by integration-test-engineer, CR#2
  `interview-kit-candidate-aware-scheduled-generation`, 2026-08-19, task 5.2). FIXED
  2026-08-26 (`dev/tech-debt-batch4-interview-level-kits`, migration
  `0058_uq_ivw_level_kits_iid`).** Added a `UNIQUE` constraint on
  `interview_level_kits.interview_id` (dropping the old plain index it replaces) — the DB now
  fails the losing concurrent INSERT fast instead of silently letting 2 rows exist. Code-side:
  `InterviewRepository.create_level_kit_with_savepoint` (`repository.py`) wraps the insert in
  `begin_nested()` (same pattern as `create_feedback_with_savepoint`); `_create_kit_stub`
  (`_kit_context.py`) now calls it; `_run_generate_level_kit` (`tasks.py`) catches the loser's
  `IntegrityError` and re-fetches the winner's row instead of crashing. Live-verified: two
  genuinely concurrent inserts (`asyncio.gather`) for the same fresh `interview_id` against real
  Postgres — one won, one hit `IntegrityError` cleanly (no crash), re-fetched the winner, exactly
  1 row persisted, `get_level_kit_by_interview_id` returned it with no `MultipleResultsFound`.
  0 pre-existing duplicate rows confirmed before the migration (no backfill needed). Gate 1:
  253 passed, 5 skipped, 0 failed. See `docs/SCHEMA_CHANGE.md` [2026-08-26] entry.

- 🔴 **GitHub Actions is hard-blocked for the rest of August 2026 — PR #225 merged WITHOUT CI, user-authorized (2026-08-24).** August usage hit 2,116 of the ~2,000-minute included allotment; with no spending limit raised above $0, every job on PR #225's first run returned `runner_id: 0` and the check-run annotation `"The job was not started because recent account payments have failed or your spending limit needs to be increased."` — confirmed via `gh api .../check-runs/{id}/annotations`, not a code/test failure. `netAmount` still shows `$0.0` (discounts fully offsetting), consistent with "blocked" not "billed." User was given 3 options (raise the spending limit now, wait for the 2026-09-01 reset, or merge on local verification alone) and explicitly chose the third. PR #225 merged (`18065cb`) on the strength of: Gate 1 run twice independently by the orchestrator (152 passed), a real-stack functional-test-engineer pass, and 2 full `principal-reviewer` rounds against the local diff (APPROVE-WITH-NITS) — but the actual GitHub-Actions-Linux-runner execution (the Live-verification mandate's Rule 2 "CI on the actual target platform is part of the definition of done") never happened for this PR. **Action needed before/at the next CI-triggering push:** either raise the spending limit or wait for the reset, then verify PR #225's merge commit retroactively passes CI on `main` — if it doesn't, that's a real environment-parity gap this exact mandate exists to catch, discovered late.
- 🔴 **Backend `typecheck` (mypy) CI job's known 133/3-file pre-existing red (seed scripts + one migration) is still unfixed** — same entry as tracked above for prior PRs; PR #225 would have hit this same pre-existing red had CI actually run (confirmed the 3 files are untouched by this PR).
- 🔴 `backend/app/modules/positions/schemas.py` is over CLAUDE.md's 300-line file cap (376
  lines, 2026-08-26, `dev/tech-debt-batch3-data-query` review round — `extra="forbid"` +
  docstring added to `InterviewLevelRequest` for the M2 panelist_id/panelist_ids fix pushed it
  further over an already-over-cap file, was 371 before that). Not fixed in this pass — same
  systemic, repo-wide 300-line-cap gap as `models.py`'s prior split and G14 above; needs its own
  scoped decomposition pass (e.g. hive off `InterviewLevelRequest`/`InterviewLevelResponse`/
  `PanelistSummary` into a sibling `_interview_level_schemas.py`, matching the `models.py` ->
  `_interview_level_models.py` precedent), not a one-off extraction here.
- 🟡 **`GET /positions/{id}/history`'s new pagination (2026-08-29, §8 P5 follow-up) caps the
  response at 200 rows with no "load more" UI** — `position-history.tsx` still renders the page
  as if it were the full list. A position accruing >200 history rows will silently show a
  partial view with no indication. Needs a paginated UI affordance before it's a real gap in
  practice (unlikely at current scale, but unbounded). See §8 for the full pagination detail and
  a related DB-side index gap (P8/P9) found in the same review.

## 5. Tech debt — tests/CI

- 🔴 **Same dead-UUID class as bucket (b) — NOT scope-closed at 2, an earlier version of this
  entry claimed that without independently re-deriving it and was wrong.** Live-verified against
  local Postgres 2026-09-03 (principal-reviewer, bucket-b review round 2): **19 files** carry one
  of **6 distinct** hardcoded position UUIDs, and **all 6 are dead** (zero matching rows; only 3
  positions exist in the DB today). Named examples: `backend/app/modules/interviews/tests/
  test_functional_p29_sequencing.py:40` (carries `22ed4a54-...`, the same one bucket (b) fixed,
  plus dead `LEVEL_L1_ID`/`LEVEL_L2_ID`/`LEVEL_COUNT = 4`) and `backend/app/scripts/
  seed_positions_data.py:44` (an 8-position ageing-bucket map, `22ed4a54-...` at that line) — but
  17 more files carry the other 5 dead UUIDs (`c180a718-...` in 8 files, `6d76a990-...` in 3,
  `b4a16264-...` in 3, `166ab03e-...` in 1, `6ed3b751-...` in 2). Carriers do NOT all use the
  `POSITION_ID` variable name — e.g. `test_functional_p20_screening.py:58` holds `c180a718-...`
  as `POSITION_WITH_LEVELS`, and `backfill_level_type_org_correction.py:17`/`seed_positions_data.py:38`
  carry `6ed3b751-...` in a docstring/dict-key — a future scoped pass must grep the UUID values
  themselves, not assume a shared variable name. All `RUN_FUNCTIONAL_TESTS`-gated (don't block
  CI) or standalone scripts (not on any request path) — correctly out of scope for the bucket-b
  fix itself. Separately, a hardcoded local-dev DB password (`P@ssword1-1`) appears across 31
  files — pre-existing, unrelated to this UUID class, flagging here so it isn't "discovered"
  again later. Needs its own scoped pass — same fix shape as bucket (b): resolve
  each position dynamically instead of hardcoding.
- 🔴 **`test_functional_level_kit.py` confirmed as one of the 8 files carrying dead
  `c180a718-...` (`POSITION_ID`)** (found 2026-09-03, functional-test-engineer, `dev/level-kit-
  agent-llm-gateway`) — 11/14 cases in this file error at fixture setup with
  `ForeignKeyViolation: candidates_created_by_fkey`. Same local-DB-recreate root cause as the
  dead-UUID class above, but a DIFFERENT variable/entity: `RECRUITER_USER_ID =
  "4a678325-bdf5-408f-82f0-2ee9bb7152ad"` is also stale (real `recruiter@ats.test` id is now
  `bdee609b-9cee-4ee2-8136-2ae85f6fff18`) — a hardcoded user id, not a position id, so it's
  outside the position-UUID class's own scoped-pass fix shape. Pre-existing, unrelated to the
  llm_gateway migration this branch actually ships — reported, not fixed, per functional-test-
  engineer's mandate. Needs its own fix: re-derive both IDs dynamically or from a live query.
- ❓ **Schema-constrained happy path for `level_kit_agent`'s Anthropic call (`schema=_OUTPUT_SCHEMA`
  passed to `llm_gateway.complete()`) never exercised against a real 200 response** (found
  2026-09-03, functional-test-engineer, `dev/level-kit-agent-llm-gateway`) — this local
  environment has no `ANTHROPIC_API_KEY`/Bedrock AWS credentials, and the real `GEMINI_API_KEY`
  is quota-exhausted (20 req/day free tier). All 3 providers' graceful-degradation/retry/circuit-
  breaker paths were verified live against the real async Celery/DB stack (no defects), but the
  actual point of this fix — Anthropic returning schema-conforming JSON via `output_config` —
  has zero live confirmation. Not a merge blocker (the code path degrades correctly either way;
  this is an environment-credential gap, not a code defect), but flagged so it isn't silently
  assumed verified: re-check with a real `ANTHROPIC_API_KEY` smoke call before/at AWS Bedrock
  go-live, same class as the already-tracked NFR go-live-gated items.
- 🔴 **Gemini's path in `level_kit_agent.py` still blocks the Celery worker's event loop**
  (found 2026-09-03, principal-reviewer round 1, `dev/level-kit-agent-llm-gateway`, M5) —
  `_gemini()`/`_call_gemini()`/`_invoke_gemini()` are still fully synchronous (`time.sleep`
  retries, blocking `google-genai` SDK call), explicitly out of scope for that change (only
  Anthropic/Bedrock were converted to async + routed through `llm_gateway`). Gemini is now
  reached through `run()`'s newly-`async` dispatch and is the CURRENTLY-CONFIGURED local
  provider (`INTERVIEW_KIT_PROVIDER=gemini`) — so this is the live path, not a dormant one.
  Needs its own pass: convert `_gemini`/`_call_gemini`/`_invoke_gemini` to async
  (`asyncio.sleep` for retries, `run_in_executor` for the blocking SDK call, matching the
  pattern `llm_gateway_providers.py`'s bedrock/gemini functions already use), or route
  Gemini through `llm_gateway` too (`llm_gateway_providers.py`'s `call_gemini` already
  exists and is already bounded/async — the remaining work is entirely in
  `level_kit_agent.py`'s dispatch, not the gateway).
- 🔴 **`app/modules/offers/tasks.py` — 0% test coverage, 89 statements** (found 2026-09-03
  during the coverage-gate risk-impact assessment, PRIORITY item 4). A Celery task file with
  zero automated coverage — touches Reliability/Observability per the 10-dimension mandate,
  not inert like the standalone `app/scripts/*` files driving the aggregate 66%. Not blocking
  any of today's merges (it's inside the 88.2%-covered real-app-code slice, which already
  clears the 80% gate in aggregate) — needs its own scoped test-writing pass.
- 🔴 **`tests/integration/test_candidates_flow.py:160` — `_require_offline_providers` has the
  same `autouse=True`-module-scope + denylist shape principal-reviewer found and fixed in
  `test_positions_defects_flow.py` (branch `fix/main-ci-break`, round 4, 2026-09-03).** Its
  condition only skips on `provider == "anthropic"`, so `gemini`/`bedrock` pass the guard and
  violate this file's "deterministic offline scorer" premise — same class as the fixed
  `_require_offline_provider` in the sibling file. Flagged, not fixed (out of scope for that
  branch's diff, which only touched this file's resume-download rename) — same fix shape when
  picked up: narrow to an allowlist (`if provider != "local_nlp": skip`) and scope the fixture
  to only the tests that actually need an offline provider, not `autouse=True` module-wide.
- 🔴 `offers/tests/test_functional_hiring_uniqueness.py` — 3 of 6 tests fail against the live stack (drives hire-uniqueness via a manual status PATCH to `hired`, which is 422-blocked since BR-054). Needs a rewrite to go through the real `POST /offers/{id}/accept` path.
- 🔴 **`positions/tests/test_functional_p6_4_closed_lockdown_e2e.py` — stale since 2026-07-31, root-caused during Tier-3 hygiene batch 1's review (2026-08-27).** The module-scoped `closed_fixture` creates an interview via a level that has no `interview_level_panelists` row — `_make_panelist` (line 444, called after the failing assert) inserts into the global `interview_panelists` directory instead, a table the `LEVEL_HAS_NO_PANELISTS` gate (landed `89db1f8`, multi-panelist levels, 2026-07-31) doesn't read. Test written 2026-07-23, never updated for the gate that shipped 8 days later — reproduces deterministically in isolation (`1 failed, 24 errors`), NOT a rate-limit/throughput artifact. Fix: seed `interview_level_panelists` for the created level inside `_create_position`, before interview-create.
- 🔴 **`positions/tests/test_functional_p24_position_status.py` + `test_functional_p23b_position_status.py` — stale since 2026-07-04, same review.** Both assert that open→closed with no reason auto-sets `portco_deferred` — that behavior was deliberately removed 2026-07-04 (`d1d003e`, "enforce user reason on open→closed"; `positions/validation.py:85` now raises `CLOSE_REASON_REQUIRED`). Tests last touched 2026-07-28 without updating this assertion. Deterministic, reproduces in isolation. Fix: update both to expect `CLOSE_REASON_REQUIRED` instead of the auto-set behavior.
- 🟡 **The backend functional test suite is skip-by-default (`RUN_FUNCTIONAL_TESTS=1` gated) — the above 2 stale-test defects sat undetected for 4-8 weeks as a direct result**, since Gate 1's routine unit runs never exercise them. No action proposed here beyond flagging the systemic risk; a periodic scheduled functional-suite run (not gated on a specific PR touching the file) would catch this class of drift proactively instead of waiting for an unrelated review to stumble into it.
- ✅ `tests/unit/test_seed_dev.py::test_run_seed_fresh_creates_users_grants_and_audits` (2026-08-08, `chore/ci-real-db-e2e-fix` round 3) — root-caused: the test's hardcoded `== 7` was stale (2026-07-23 mypy-cleanup pass) AND it compared `summary["users"]` (covers both `TEST_USERS` + `NAMED_RECRUITER_USERS`) against `len(TEST_USERS)` alone, an existing scope mismatch independent of the stale count. Fixed to compute `total_users = len(TEST_USERS) + len(NAMED_RECRUITER_USERS)` instead of a magic number, so it self-adjusts as either list grows.
- ✅ `departments/tests/test_repository.py::test_list_scopes_to_org_searches_and_orders` (2026-07-29, `dev/tech-debt-batch1`) — root-caused as a real code defect (spec AC-004 requires name-order, code ordered by updated_at desc). Fixed the repository, not the test. Independently confirmed by NFR Phase 2b's own review (same root cause, same conclusion) — merged together with PR #197's `COUNT(*) OVER()` windowed-query optimization on the same query.
- ✅ `organizations/tests/test_repository.py::test_list_applies_search_and_active_filters_and_orders_by_updated_at` (renamed from `..._orders_by_name`, 2026-07-29, `dev/tech-debt-batch1`) — root-caused as a genuinely stale test (organizations spec has no ordering AC). Fixed the test's assertion, repository untouched.
- 🔴 Pagination edge case inherited from PR #197's `COUNT(*) OVER()` conversion (departments/organizations + 4 other repos): a page request past the end of the result set (`offset >= total`) returns `total = 0` (from `rows[0].total_count if rows else 0`) instead of the true total — the old separate `SELECT count(*)` didn't have this gap. A UI landing on an out-of-range page (e.g. after deletions) would show "0 results" and lose the ability to page back. Found during PR #199's merge-conflict review (2026-08-02); not fixed there (out of scope for that change) — needs its own fix across all 6 converted repos.
- 🔴 3 pre-existing `positions` module test failures: `test_service.py::test_change_status_stale_version_raises_409` (test itself is stale — needs a rewrite to a real reachable scenario), `test_tasks.py::test_extract_storage_miss_persists_failed` + `test_extract_success_persists_result` (mock/fixture drift, not chased to root cause).
- 🔴 No frontend component test for `positions-ageing-report.tsx` / `positions-ageing-bucket-strip.tsx`.
- ✅ = Execution Queue item 3: backend CI provisions real Postgres/Redis (`chore/ci-real-db-e2e-fix`, PR #221, 2026-08-08) — `test` job now gets `postgres:18`/`redis:7` services + `docs/ci_schema_snapshot.sql` + `alembic stamp head` + `RUN_DB_TESTS=1`. The ~18 tests that failed for lacking real services now pass. See the new needs_db entry below for what surfaced once `RUN_DB_TESTS=1` actually ran those tests for the first time. **Separately, the `test` job's own coverage gate still fails** (`Coverage failure: total of 66 is less than fail-under=80`) — pre-existing, unchanged across this PR's commits, same class of pre-existing CI blocker as the `typecheck`/`component-test` entries below (needs its own coverage-raising pass, not caused by or fixed by this PR).
- ✅ = Execution Queue item 4 — **CLOSED 2026-09-05, branch `dev/frontend-component-test-debt`.** All 15 originally-tracked failures across 6 files fixed, plus 3 more defects found only via live-verification during the fix (2 layered on top of the originally-scoped `position-form-drawer.test.tsx` currencies fix; 1 in `status-change-dialog.test.tsx`), plus one `principal-reviewer`-caught regression in the fix's own first pass (see below). Final state: full component-test suite 80/80 files, 534/534 tests green; `tsc --noEmit` and `eslint` clean project-wide. Per-file summary:
  - `nav-items.test.ts` (2 failures) — test-only: component's `NAV_ITEMS` had grown 2 new ungated sections ("Panelists", "Applications") in a prior nav restructure; 2 tests' expected arrays were stale. Updated both to the real 11-item/5-item arrays in actual source order.
  - `position-schema.test.ts` (3 failures) — test-fixture-only: shared `BASE` fixture's `approved_at: ""` was failing the schema's own `.min(1)` requirement before ever reaching the budget cross-field rules under test. Added a valid date.
  - `status-change-dialog.test.tsx` (3 failures, +1 found during fix) — test-only: 3 tests assumed an old lifecycle model vs. the current (correct) `allowedTargets()` rules; updated expected dropdown options/defaults/selectable-target. A 4th, previously undiscovered mismatch surfaced once those 3 were fixed: the INVALID_STATUS_TRANSITION test's fallback-text assertion could never match because the component does verbatim server-message passthrough (`err.message || fallback`) and the mock's message was truthy — fixed the assertion to match the verbatim message, added a new test covering the untested `|| fallback` branch, and renamed 3 stale test/describe names that had drifted from their own (now-corrected) assertions.
  - `position-form-drawer.test.tsx` (4 of 7 failures, +2 found during fix) — MSW had no `GET /currencies` handler anywhere (`server.listen({ onUnhandledRequest: "error" })` hard-fails on it); added one (new `mocks/positions/currency-handlers.ts`, wired into the `positionHandlers` barrel), reusing the real `CURRENCIES` array. That alone did NOT fully fix the 4 tests — live verification surfaced a second, independent gap: a required `approved_at` field (same one from `position-schema.test.ts`) was never filled by the shared `fillRequired()` helper nor present on the file's `EDIT_POSITION` fixture, blocking every create/edit submission client-side before the mocked API was ever reached. Fixed both gaps.
  - `interview-org-labels.test.tsx` (1 failure) — genuine component fix: the read-only (`!canConfig`) interviewer view in `interview-levels-editor.tsx` (the branch is at lines 64-91, not "~175-193" as previously logged here) rendered only `l.level_label`, never the org-prefixed `"(Org : Level-N)"` format. Added it, computed via a new shared `orgLabelFor()` helper (`interview-level-options.ts`) also used by `buildOptions()`.
  - `positions-list.test.tsx` (2 failures, +1 found during fix) — 2 genuine component bugs: a duplicate "Clear filters" button (rendered both by `NoResultsState`'s action AND, unconditionally whenever filters are active, by the filter bar) — removed the redundant one from `NoResultsState`, keeping the filter bar's persistent button; and a 3rd, previously-masked failure this label bug had been hiding (a stale test asserting a `search=` URL param for a debounce effect that actually writes `title=` — dead since the same `search`→`title` filter-bar redesign, commit `a666cfa`, 2026-06-24) — fixed the test to match.
  - **Correction to this fix's own first pass (`principal-reviewer` round 1, CHANGES-REQUESTED):** the first attempt WRONGLY changed the title filter's label from `"Title"` to `"Search by title"`, reasoning (via a `--follow`-scoped `git log`) that "Title" looked like an undocumented regression. A whole-tree `git log -S "Search by title"` found the real history: commit `a666cfa` deliberately replaced a free-text global-search box (`id="pos-search"`, label "Search by title", `search` param) with today's dedicated title filter (`id="pos-title-filter"`, label "Title", `title` param) as part of the same filter-bar redesign — "Title" was correct all along. Reverted the label, fixed the test to match the correct label instead of the other way around. Also caught: the new org-prefix span in `interview-levels-editor.tsx` duplicated information already embedded in `level_label` for all real, spec-conformant data (`buildOptions()` already writes `level_label` as `"{Org} Level N"` per `openspec/specs/positions/spec.md:326`) — gated the new span to only render when it adds information the label doesn't already convey (legacy free-text labels like the test's "Bar raiser" still get it; real levels created via the current editor no longer show a duplicate).
- ✅ = Execution Queue item 5: `e2e` Playwright job now provisions a real backend + Postgres/Redis before Playwright's `webServer` boots (`chore/ci-real-db-e2e-fix`, PR #221, 2026-08-08) — `BACKEND_ORIGIN` feeds `next.config.mjs`'s `/api/v1/*` rewrite so Next's server-side proxy calls reach something real instead of ECONNREFUSED. `frontend/e2e/auth.spec.ts`'s stale "welcome back" assertion updated to the current universal `/reports` landing in the same PR. The systematic ECONNREFUSED/MSW-vs-real-backend failure is closed; 3 `mobile`-project (Pixel 5) tests hit genuine issues newly reachable now that mobile e2e exercises real interactive UI, all 3 quarantined + logged separately below: `auth.spec.ts`'s logout (topbar overlap), `organizations.spec.ts`'s create-org submit (drawer overlap), `pipeline-retry-badge.spec.ts` (locator-strategy gap, NOT the `page.goto()` bug first suspected -- see its own entry, which WAS real and independently fixed via nav-link navigation matching `gotoOrganizations`/`gotoPositions`). **Confirmed green via real CI (gh run 31250303308):** 43 passed / 60, 2 flaky (retry-masked, see the webkit entry directly below), 15 skipped (the quarantines + pre-existing skips), 0 hard failures.
- 🟡 **2 flaky `webkit`-project e2e tests, retry-masked (real CI, gh run 31250303308, PR #221 round 4, 2026-08-08)** — `organizations.spec.ts:46` (write-role create-an-organization) and `pipeline-retry-badge.spec.ts:76` (Failed/Failed-after-3-attempts). Both time out on `frontend/e2e/helpers/auth.ts:31`'s `page.waitForResponse` for `POST /auth/login` (Playwright's 30s test timeout), then pass on the automatic retry — originally a DETERMINISTIC every-run pattern. **Structural root cause fixed (`fix/e2e-webkit-flake-prod-build`, commit `1fad9ac`, PR #222, 2026-08-08)** — switched CI's Playwright `webServer` from `next dev` to a production build+start, removing the Next.js on-demand-route-compile race (confirmed via code read: `auth.ts` already used the correct `Promise.all([waitForResponse, click])` pattern, so this was never a test-code race). **Result across 4 independent real CI runs on this fix (run 31253719568 x2, run 31255151924, run 31256030051):** 3 runs clean (`45 passed, 15 skipped`, 0 flaky), 1 run showed `organizations.spec.ts:46` flaky again (1-in-4, vs. the prior 2-every-run deterministic pattern) — a large reduction, not full elimination. Residual is consistent with ordinary webkit-on-Linux-CI network-stack timing variance, not a compile race, not a code defect in the test or app — exactly the class of thing `retries: 1` exists to absorb, and did (job still reports overall success). **User decision (2026-08-08): merge as-is, log residual here rather than chase further** — not investigating deeper unless the residual rate climbs meaningfully above 1-in-4 in future runs.
- 🔴 **CI's e2e job runs `next start`, not production's actual deployment artifact (found via principal-reviewer round 2, `fix/e2e-webkit-flake-prod-build`, 2026-08-08).** `next.config.mjs` sets `output: "standalone"` and `frontend/Dockerfile` ships `CMD ["node", "server.js"]` — production runs the standalone server, but CI's `webServer.command` now runs `npm run build && npm start` (`next start` over the regular `.next` build), confirmed harmless for THIS PR's purpose (route-precompile, proven by 45 passing tests through real logins) but a residual environment-parity gap. **Also flagged, unverified — needs a live check before anyone relies on it either way:** whether `next.config.mjs`'s `rewrites()` closure resolves `BACKEND_ORIGIN` at build time (baked into the standalone bundle) or at runtime (`next start`/`node server.js` both re-read `process.env` per request) — if build-time-baked, prod's actual runtime env var value wouldn't behave the way CI's `next start` path does today. Not chased in that PR (out of scope, minimalism floor) — needs its own investigation.
- 🔴 **`[mobile] auth.spec.ts::logout returns to login` QUARANTINED for the `mobile` project only (PR #221 round 3, 2026-08-08)** — live-verified via real CI (gh run 31249164374): the "Log out" button's click is intercepted by the sibling role-label `<span>` ("Recruiter", `user-menu.tsx`) on the Pixel 5 viewport — a genuine topbar responsive-layout overlap (the `UserMenu` flex row doesn't fit the banner's other content at this width). Newly surfaced because mobile e2e never reached real interactive UI before this PR. Needs a UX pass (`.claude/rules/ats-ux-ui-guardrails.md`), not a blind CSS patch from a CI-infra change.
- 🔴 **`[mobile] organizations.spec.ts::create an organization → it persists and appears in the list` QUARANTINED for the `mobile` project only (PR #221 round 3, 2026-08-08)** — live-verified via real CI (gh run 31249164374): the "Create organization" submit click is intercepted by a sibling `<div class="space-y-2 flex-1">` field-group element inside the create drawer's fieldset, on the Pixel 5 viewport. A genuine drawer/fieldset responsive-layout overlap, same class of newly-surfaced mobile gap as the logout entry above. Needs its own UX pass.
- 🔴 **`[mobile] pipeline-retry-badge.spec.ts` QUARANTINED for the `mobile` project only (PR #221 round 3, 2026-08-08)** — real CI (gh run 31249164374) showed the `/candidates` heading never appearing; first suspected to be the same `page.goto()` session-drop bug found and fixed elsewhere in this PR (this spec's own `loginAsHrAdmin` + `page.goto("/candidates")` did in fact have that bug, independently fixed via a `gotoCandidates` nav-link helper matching `gotoOrganizations`/`gotoPositions`) — but round 3's own local live-verification (isolated `--project=mobile` run, retried) found the failure PERSISTS after that fix, for a genuinely different reason: on the Pixel 5 viewport the candidates table renders as a definition-list card layout (`list`/`listitem`/`term`/`definition`) below the table breakpoint, with no `role="row"` element at all, so this test's `page.getByRole("row", { name: ... })` locators match zero elements and every row-scoped assertion fails. A locator-strategy gap this spec never had to handle before real mobile e2e existed (previously ran only against MSW-mocked auth, never reaching this render at all). Needs a mobile-aware locator (e.g. scope by the enclosing `listitem` instead of `role="row"`), not chased here.
- 🔴 **`needs_db`/`RUN_DB_TESTS=1` integration tests actually ran in CI for the first time ever (PR #221, 2026-08-08)** — surfaced 72 pre-existing failures across 7 files, entirely unrelated to the CI-provisioning change itself (never caught before because these tests had never executed against a live DB in CI). Root-caused live against a real snapshot-loaded Postgres: **fixed** 24 of the 72 — `tests/integration/test_positions_flow.py` (24→6), `test_positions_defects_flow.py` (9→4) — root cause was 2 shared test fixtures (`_create_position`) missing the required `PositionCreate.approved_at` field ("Change-1", no default), added after these tests were last exercised; fix is test-only (added the field to the fixture payloads) — plus `tests/unit/test_seed_dev.py`'s own stale `== 7` assertion (round 3, see its own ✅ entry above), which is why this section no longer lists it under "remaining". **48 remain, NOT fixed here, each its own root cause:** (a) 27 in `test_interview_panelists.py` + 5 in `test_interview_levels_panelist.py` — `category="internal"` panelist creation requires an explicit `org_name` (`interview_panelists/service.py::_resolve_org_name` raises `OrgNameRequiredError` rather than auto-resolving "STG Labs") — a product/spec decision, not a mechanical fix; (b) 5 in `app/modules/interviews/tests/test_category_rank_regression.py` — FK violation inserting a fixture `interview_levels` row against a `position_id` that was never actually persisted, a separate fixture defect; (c) 6 in `test_positions_flow.py` + 4 in `test_positions_defects_flow.py` — budget/status-transition/history assertions stale against current behavior, e.g. `test_status_limited_to_three_settable_values` expects `open→in_progress` to succeed but the current status state machine returns 409 `INVALID_STATUS_TRANSITION` (needs a spec-informed fix per Gate 3, not a quick patch); (d) 1 in `test_candidates_flow.py::test_resume_download_returns_302_with_location_header` (endpoint now returns 200+JSON with a signed URL, not a 302 redirect). Each needs its own Gate-5 pipeline pass to root-cause + fix; not attempted here beyond (a)-(d)'s classification. **Re-verified against the FULL `pytest -q` run (not just these 7 files), real CI (gh run 31250303340):** `48 failed, 1836 passed, 656 skipped` — these counts are exact and confirmed at the full-suite level, not just the curated 7-file subset. **The 48 is a CI number, not a local one — do not compare a local `RUN_DB_TESTS=1` run against it directly (this caused a false-alarm re-investigation during the G15 fix, 2026-09-01).** Locally the count is **46**, for 2 independent, harmless, environment-driven reasons, not because anything was fixed or newly broken: (i) `test_category_rank_regression.py` is `5 failed` in CI but only `1 failed` locally — its hardcoded fixture position id (`22ed4a54-21ba-4a52-a23c-78135397ca27`) happens to already exist in the shared local dev DB (so 4 of its 5 FK-dependent tests pass locally that FK-violate in CI's fresh DB); (ii) `test_interview_kit_candidate_aware_flow.py` is `0 failed` in CI (it needs a real Celery worker, which CI now has per the earlier same-day fix) but `2 failed` locally (no local worker running) — net `48 - 4 + 2 = 46`. Both are pre-existing, confirmed via a control run against unmodified `main` producing the identical 46/1774/657 counts and identical failing-test-name set.
- 🔴 **`frontend/e2e/a11y.spec.ts::authenticated shell has no serious/critical a11y violations` QUARANTINED (`test.skip`, PR #221 round 2, 2026-08-08 — principal-reviewer M4)** — fails on real data: `color-contrast` (serious), the sidebar's nav-group section labels ("Sourcing"/"Evaluation"/"Pipeline"/"Admin", `text-muted-foreground/60` on white) measure 2.87:1, below WCAG AA's 4.5:1. Genuinely pre-existing (pure CSS, data-independent) — masked until now because this spec could never reach the authenticated shell in CI before this PR's login/MSW fix (docs/BACKLOG.md's `pipeline-progress` chart-ink entry, §3, is the same class of pre-existing contrast gap in a different component). Fix: raise the label's opacity/color in the sidebar-nav component (exact file not yet located — needs its own pass), then un-quarantine.
- 🔴 **`frontend/e2e/positions.spec.ts`'s interviewer read-only test QUARANTINED (`test.skip`, PR #221, 2026-08-08)** — `src/lib/navigation/nav-items.ts`'s `ROLE_NAV_OVERRIDES` restricts `interviewer`'s nav to `["Interviews"]` only, an intentional, documented restriction ("Interviewers only work their interview queue") that predates this PR. The test's own premise (interviewer navigates to Positions via the nav and sees it read-only) no longer holds — never caught before because this spec could never reach real auth against the real backend under MSW's fixture-based mocking. Needs a product decision: (a) rewrite to `page.goto("/positions")` directly (tests route-level access, bypassing nav-discoverability) or (b) drop the test as testing an unsupported scenario. **Coverage note (principal-reviewer, PR #221 round 1):** with this test quarantined AND the write-role test already skipped (Phase 15 fast-follow), `positions.spec.ts` currently has ZERO active e2e coverage — flagging the cliff explicitly rather than letting it go unnoticed.
- 🔴 **Backend `typecheck` (mypy) CI job has 133 pre-existing errors in exactly 3 files, confirmed unrelated to `chore/ci-real-db-e2e-fix` (PR #221, 2026-08-08)** — real CI (gh run 31250303340, verbatim `Found 133 errors in 3 files (checked 288 source files)`): `backend/app/scripts/seed_legal_transaction_demo.py` (68), `backend/app/scripts/seed_uat_recruitment_funnel.py` (64), `backend/alembic/versions/0054_pipeline_retry_count.py` (1) — 2 seed scripts + 1 migration, none touched by this PR (already red identically across main's last 3 backend-CI runs). `tests/unit/test_seed_dev.py`'s own pre-existing mypy debt (an `attr-defined` on `seed_dev.audit` + stale `# type: ignore` comments, confirmed via `mypy --shadow-file` against `origin/main`'s unmodified content) is real locally but sits OUTSIDE CI's mypy target set (`tests/` is excluded from the CI job's scope) — a separate observation, not folded into the 133/3-file count above. This was the same class of pre-existing debt as `component-test`'s frontend failures (also confirmed pre-existing on main, zero `frontend/src` files touched by this PR) — both blocked backend-ci.yml/frontend-ci.yml from ever going fully green on ANY PR until fixed, independent of what that PR actually changes. **The `component-test` half is now closed** (§1 item 4, above — `dev/frontend-component-test-debt`, 2026-09-05, incl. the `GET /currencies` MSW-unmatched-handler gap this note originally flagged); the mypy/133-errors half remains open, needs its own scoped pass (likely a `dev/mypy-cleanup` or similar) — not chased here. **Update
2026-08-18 (CR#1, PR #223 first real CI run):** the backend `test` job also has 48 pre-existing
failures, confirmed byte-for-byte identical (same test names + error messages) against `main`'s
own last real run — all in `tests/integration/test_interview_levels_panelist.py` and
`test_interview_panelists.py` (`ORG_NAME_REQUIRED`/`MANDATORY_ORG_LEVEL_MISSING` validation
mismatches) plus `test_candidates_flow.py::test_resume_download_returns_302...` (a signed-URL
redirect assertion) — none touched by CR#1, none newly introduced. Same "blocks CI from ever
going fully green on ANY PR" class as typecheck/component-test above.
**Update 2026-08-19 (CR#2, PR #224 first real CI run) — root cause for a slice of this bucket
now identified: `backend-ci.yml`'s `test` job never starts a Celery worker at all** (confirmed
via `grep -n "celery\|worker" .github/workflows/backend-ci.yml` — zero matches; same class of
gap `frontend-ci.yml`'s `e2e` job had before PR #221's fix, just never applied to this job). Any
integration test that enqueues a real Celery task and polls for its completion times out here,
regardless of whether the task logic is correct — already the reason
`test_positions_defects_flow.py::test_jd_extraction_completes_inline_and_persists_skills` sits
stuck at `"status":"processing"` in the count above. CR#2's own new
`test_interview_kit_candidate_aware_flow.py`'s two scenarios hit the identical wall in CI
(`Failed: Level kit did not settle within 25.0s`) despite passing locally against a real running
worker (integration-test-engineer's local run, task 5.2) — confirmed not an application-logic
regression, purely this CI-environment gap. `48→50` failed on this run; the 2 new names are this
same root cause, not 2 new distinct defects. Needs its own fix (add a `celery -A app.workers.
celery_app worker` background step to `backend-ci.yml`'s `test` job, matching PR #221's
`frontend-ci.yml` precedent) — out of scope for CR#2 itself (a CI-infra change, not part of that
CR's diff); tracked here so the next PR that touches a Celery-backed integration test doesn't
re-discover this from scratch.
**Fixed 2026-09-01 (branch `chore/backend-ci-celery-worker`)** — `backend-ci.yml`'s `test` job now
starts a background Celery worker before `pytest` (plus a readiness-wait step and an `if:
failure()` step dumping `celery.log`), using the identical command and queue list as
`frontend-ci.yml`'s `e2e` job. Queue coverage verified complete against `celery_app.py`: the 6
`task_queues` plus `task_default_queue = "maintenance"`, which is itself in the `-Q` list, so
unrouted tasks are covered too. No `celery beat`/`-B` anywhere in CI, so G16's
`partition-maintenance` → `ensure_partitions` still never fires in CI despite the worker now
listening on `maintenance`. Expected to clear the 2 `test_interview_kit_candidate_aware_flow.py`
timeouts; **actual post-fix failure count recorded here once the real CI run reports** (not
assumed) — the third test named above,
`test_positions_defects_flow.py::test_jd_extraction_completes_inline_and_persists_skills`, tests
the inline extraction path, not the enqueue path, so it may not be covered by this fix.
**Real CI outcome (2026-09-01, first run after the worker landed on `main`, commit `7b21bd3`):**
confirmed the fix worked for its intended target — `test_interview_kit_candidate_aware_flow.py`'s
2 tests now show `.s` (1 passed, 1 skipped), zero failures from that file. But the same real
worker introduced a genuinely NEW regression via a race this test file's own local helpers never
accounted for: `test_candidates_flow.py`'s `_run_extraction`/`_do_extract` calls now run
concurrently with the router's real Celery enqueue to the SAME task, and the inline call can lose
the OCC claim (`_extraction_tasks.py` D7 guard) and return `'processing'` silently — surfaced as
`test_extraction_completes_and_fields_persisted` (`assert 'processing' == 'completed'`) and
`test_identity_dedup_sets_duplicate_flag_post_extraction` (dedup flag never set because the
duplicate-detection branch never ran) both going red. Net CI failure count unchanged at 50/50
(2 fixed, 2 new, same run). **Fixed same day** (`_run_extraction` rewritten to poll the DB to
settlement when it loses the race, mirroring `test_candidate_ai_match_screen_flow.py`'s
already-established pattern) — both target tests re-verified passing locally against real
Postgres; full-file local re-run (`22 passed, 3 skipped, 1 failed`) confirms zero other
regressions, the 1 remaining failure (`test_resume_download_returns_302_with_location_header`)
independently reproduced identical on unmodified `main` via `git stash` — pre-existing, already
tracked earlier in this same §5 entry ((d) above), unrelated to this fix. Note: locally there is
no competing worker, so both tests pass on unmodified `main` too — the local run proves
no-regression, not that the race fix works. The probative evidence is real CI run `33509579759`,
where the identical 20×0.5s poll pattern in `test_candidate_ai_match_screen_flow.py` passed all
5 tests with the worker running. **Final real-CI confirmation (commit `8f22f3c`, run
`33512829002`, 2026-09-01):** `48 failed, 1771 passed, 658 skipped` — back to the exact original
tracked baseline (verified via full failed-test-name diff against the pre-worker-fix run
`546ac3b`: every remaining failure name is identical modulo per-run UUIDs/timestamps; the only
delta from `546ac3b` is the 2 `test_interview_kit_candidate_aware_flow.py` names now absent).
**This whole saga (Celery-worker CI fix → new race exposed → race fixed) is closed, net -2
failures, zero new regressions, confirmed end-to-end on real CI, not just locally.**

**UPDATE 2026-09-02/03 (branch `fix/main-ci-break`, NOT YET a PR, NOT YET merged) — the
typecheck 133-error debt and 43 of the 48 tracked `test`-job failures above are now fixed.**
This branch was built WITHOUT first re-reading this exact section — the org_name rule, the
resume-download 302→200 change, and the mypy 133-error breakdown were independently
re-investigated from scratch (full `cavecrew-investigator` dispatches) despite already being
documented here in full, in the (a)-(d) breakdown above, since 2026-08-08. That rediscovery
cost is itself a named root-cause finding, discussed with the user 2026-09-03 (see
`memory/resume-pointer.md` and the chat record for the full analysis).
- **typecheck: 133 → 0 mypy errors, repo-wide** (all 3 files: `0054_pipeline_retry_count.py`,
  `seed_legal_transaction_demo.py`, `seed_uat_recruitment_funnel.py`) — added type hints
  throughout, fixed a `psycopg2.connect(dict)` overload mismatch and a possibly-`None`
  `fetchone()` index. Confirmed long-standing debt (git history), not a regression.
- **(a) org_name cluster (27+5 = 32 failures) — FIXED.** Confirmed via git history (not just
  re-derived): the rule flip is commit `e420363` (2026-06-20, PO-confirmed) — internal
  panelists now require caller-supplied `org_name` (422 if blank), external always gets
  `org_name="External"` regardless of caller input. Updated `test_interview_panelists.py` +
  `test_interview_levels_panelist.py`'s setup/expectations to match; app code untouched
  (confirmed correct as shipped).
- **(d) resume-download (1 failure) — FIXED.** Confirmed via git history: commit `32e62b7`
  (2026-06-29) changed `GET /candidates/{id}/resume` from 302 redirect to 200 JSON `{"url":...}`
  intentionally (frontend now does authenticated fetch, avoiding JWT-in-`<a href>` leakage);
  the UNIT test was updated in that same commit, this INTEGRATION test was not. Renamed +
  fixed to match the unit test's already-correct shape.
- **(c) — ALL 10 of 10 now FIXED, across 4 sub-clusters (counts sum to exactly 10, do not
  add them to any other bullet in this list — bucket (c) is fully closed by these 4 items
  alone):**
  1. **Money-precision (4 of 10** — 1 in `test_positions_flow.py`, 3 in
     `test_positions_defects_flow.py`)**.** A "fix" for
     `test_budget_computed_on_create_with_eur_base` initially changed `BudgetPanel`'s schema
     fields from `float` to `Decimal` — `principal-reviewer` caught this was a REGRESSION of a
     deliberate prior fix (PR #46: `float` specifically because Pydantic v2 serializes
     `Decimal` as a JSON STRING, breaking the frontend's `typeof v === "number"` guards).
     Reverted; real fix applied instead: wrapped 22 test assertions in `Decimal(str(x))` (the
     actual bug was `Decimal(json_decoded_float)` exposing the IEEE-754 binary expansion — a
     test-harness artifact). Added a wire-type-pinning test + a `spec.md` BR-039 line so this
     specific contract can't flip a 3rd time.
  2. **Mandatory-Org-L1/L2 (2 of 10**, both in `test_positions_flow.py`**).** Commit `8e67469`
     (2026-07-22, PO-confirmed sequencing model) requires every `set_levels` call to include
     Organization Level-1 AND Level-2 — predates `test_interview_levels_panelist.py`'s
     `_set_levels` helper (only ever sent 1 `stg_labs` level) and 2 tests in
     `test_positions_flow.py`. Fixed the helper + both tests' response-shape/DB-query
     assertions (now 3 levels per position, not 1). Also fixed a real latent bug this
     unmasked: `lp_cleanup`'s FK-delete order was missing `interview_level_panelists` before
     `interview_levels` — invisible until these tests started actually creating rows instead
     of erroring out first.
  3. **Idempotency-replay `approved_at` (1 of 10**, `test_positions_flow.py`**).** Same class
     as (2) — a hand-built POST body predating a required field, not a new latent bug as an
     earlier commit on this branch briefly mischaracterized it.
  4. **Status-transition + JD-extraction (3 of 10** — 2 in `test_positions_flow.py`, 1 in
     `test_positions_defects_flow.py`**), fixed 2026-09-03 (commit `7878614`).** Gate 3 spec
     pre-read done first (`openspec/specs/positions/spec.md` §8A.1):
  `open→in_progress` is trigger=`AUTO` ONLY (BR-NEW-001, fires exclusively via
  `ApplicationService.create_application()` on first application) — never a valid MANUAL
  PATCH, so `test_status_limited_to_three_settable_values` and
  `test_history_aggregates_all_change_types_newest_first` (both PATCHed it directly) were
  stale tests, not app bugs; retargeted onto legal MANUAL paths
  (`open→on_hold→in_progress→on_hold`). The JD-extraction mismatch was root-caused via git
  history, not "may not be covered by the Celery-worker fix" as this entry previously
  speculated: commit `9d0dc0a` (2026-07-28, "perf(nfr-2b): move local_nlp JD extraction off
  request path") deliberately made extraction ALWAYS enqueue via Celery for every provider —
  this is an ENQUEUE-path test, not an inline-path test, and was simply never updated after
  that change. Renamed `test_jd_extraction_completes_inline_and_persists_skills` →
  `test_jd_extraction_persists_skills_after_async_extraction`; fixed the upload assertion to
  expect `202`/`"processing"` and added a poll-to-terminal-status loop before the
  skill-persistence checks (which are this test's actual, still-valid scope). Zero app code
  touched in either fix.
- **Coverage gate (was 65.86% vs 80% required) — NOT re-measured after this branch's fixes,
  policy decision still pending** (raise/lower/scope the threshold vs. find genuinely-missing
  coverage — a large chunk of the gap is structural: skipped/environment-gated tests and test
  files themselves count in the denominator).
**MERGED 2026-09-03 as PR #231 (`97c18d78`).** Net: this branch cleared **43 of the original
48** `test`-job failures + all 133 `typecheck` errors — NOT all 48: bucket (b)'s 5 failures in
`app/modules/interviews/tests/test_category_rank_regression.py` were never in this branch's
scope and were correctly left untouched here. **Bucket (b) itself fixed separately, same day,
on `fix/bucket-b-category-rank-fixture`** — see the ✅ entries above. Only the coverage-gate
policy question (see PRIORITY item 4) remains open from this whole arc.

- 🟡 **Latent inline-vs-real-Celery-worker race, 2 items flagged for future investigation
  (2026-09-01, found while fixing the `test_candidates_flow.py` regression above — explicitly
  NOT fixed here, out of scope for that fix):**
  (a) `backend/tests/integration/test_screening_flow.py` (lines 255-265) has the SAME latent
  exposure via its own local `_run_extraction` helper — an inline `_do_extract`/`_run_extraction`
  call that can lose the OCC claim to the real CI Celery worker exactly like
  `test_candidates_flow.py` did, just not yet observed failing in any real CI run. Worth
  rewriting to the same poll-to-settlement pattern proactively, not urgent since it hasn't
  actually failed yet.
  (b) `_extraction_tasks.py`'s OCC claim guard (`_do_extract`'s version-matched `UPDATE
  ... WHERE version = candidate.version`) only catches two deliveries racing on the SAME
  pre-commit version — it does NOT block a delivery arriving after the claim has already
  committed and bumped the version. Already documented in `_extraction_tasks.py`'s module
  docstring (D7/C1) and consciously accepted, with `celery_app.py`'s `visibility_timeout` vs
  `task_time_limit` reasoning as the mitigation — re-flagged here for a dedicated pass, not newly
  discovered. Needs its own future investigation (out of scope here — this fix only patched the
  test helper, per the brief, and was explicitly told not to touch the guard itself).

## 6. Tech debt — code hygiene (oversized files, 300-line/40-line caps)

**Analysis complete 2026-07-29 — full per-file decomposition plan in `docs/CODE_HYGIENE_DECOMPOSITION_PLAN.md`** (13 parallel read-only planning passes, re-swept to **48 files over 300 lines**, up from 38 at the Phase 3 audit). **Tier 1 ("free wins") executed same day — PR #198 merged**, branch `dev/hygiene-tier1-free-wins` — 14 files split across 9 commits (security/router.py, interviews/router.py, candidates/{tasks,screening/service,router}.py backend; lib/api/candidates.ts, mocks/position-handlers.ts, panelist-list.tsx, position-detail.tsx, interview-levels-editor.tsx, screening-detail.tsx, applications-in-candidate-card.tsx, screening-list.tsx, positions-ageing-report.tsx frontend). principal-reviewer APPROVE-WITH-NITS on round 2 (round 1 CHANGES-REQUESTED: 2 gratuitously-exported helper functions + a test asserting a hardcoded string instead of the task symbol — both fixed; round 2 found 1 doc-contradiction nit — fixed). Zero behavior/API/permission change anywhere — verified via byte-identical OpenAPI dump + route-table diff against `main`. Deferred from Tier 1 (would have conflicted with NFR Phase 2b PR #197, now merged): `positions/router.py` (done, Tier 5 backend catch-up, 2026-08-31 — see below), `jd-panel.tsx` (pending), `use-positions.ts` (done, Tier 3 frontend batch 2, see below). **Tier 2 (open-question resolution) executed 2026-08-26, branch `dev/hygiene-tier2-decisions`** — all 3 open questions resolved via AskUserQuestion, then executed: deleted confirmed-dead `create-application-panel.tsx` (338 lines, zero importers) and its now-orphaned dependency `create-application-confirm.tsx` (187 lines); deleted 4 confirmed-dead hooks from `use-interviews.ts` (`useCreateInterview`/`useAddPanelist`/`useRemovePanelist`/`useUpdateInterviewStatus`) plus their now-dead api-client functions and type exports; extracted `status-change-dialog.tsx` 305→270 lines via a pure-helpers file with its own new unit test. principal-reviewer round 1 CHANGES-REQUESTED (left a dead-code chain behind: the orphaned confirm-dialog file + 3 dead api functions + 2 dead types + 3 stale doc comments + missing spec-sync + no helper test) — all fixed same session. **Round 3 (principal-reviewer's second CHANGES-REQUESTED, same branch, 2026-08-26):** the AC-061/AC-062 SCREENING_REQUIRED affordance previously existed only in dead code (`candidate-screenings-section.tsx` — zero importers, and `onStartScreening` was never actually wired even there); per user product decision, wired it onto the real live apply path instead — `applications-in-candidate-card.tsx` now shows an inline Alert + "Start Screening" button on 422 SCREENING_REQUIRED, opening `ScreeningStartDialog` pre-selected for the position, with `candidateName` threaded down from `CandidateDetail`'s already-fetched `useCandidate()` data (no new fetch). `candidate-screenings-section.tsx` deleted (confirmed dead, no test file). Spec AC-061/AC-062 and the stale doc-comment findings (interviews.ts header, screening-start-dialog.tsx header, this doc's §Tier-2 note) corrected to name the real component. **Tier 3 batch 1 (backend, 4 files) executed 2026-08-27, branch `dev/hygiene-tier3-backend-batch1`:** `positions/service.py` 375→299 (split into `_service_helpers.py`/`_service_writes.py`); `candidates/repository.py` 713→247 (6-way free-function split by concern — CRUD/documents/enrichment/matches/consents/bulk_jobs, `CandidateRepository`'s public interface byte-identical, AST-verified); `candidates/service.py` 591→298 (split into 4 `_service_*.py` files; `_enqueue_extraction`'s static-method gotcha confirmed obsolete — a prior batch already moved `router.py`'s call to the public `enqueue_extractions`; ~115 test `@patch` decorators retargeted, all resolve-verified); `security/service.py` 389→320 (extracted `_mfa_helpers.py`, only the 4 named MFA-challenge helpers — login/session/refresh/logout untouched). Zero public-interface change anywhere (AST-diffed base vs branch), zero import cycles, zero security regression (OTP/attempt-cap logic line-identical). 2 principal-reviewer rounds: round 1 CHANGES-REQUESTED (zero `docs/` update despite the mandate, a stale `dev_router.py` reference to a renamed function, one genuinely-lost docstring on `build_current_user`, no live functional-test-engineer gate) — docs/docstring findings fixed same session, live check dispatched separately. **Tier 3 batch 2 (backend, `positions/repository.py`) executed 2026-08-27, branch `dev/hygiene-tier3-backend-batch2`:** 457→361 lines — split position-code allocation into `position_code_repository.py` (59 lines) and ageing-summary aggregation into `ageing_repository.py` (55 lines), both offers-style free-function delegation; folded the JD/interview-level single-item reads (`current_jd`/`get_jd`/`get_interview_level`) into `child_repository.py` (108→159 lines), fixing the pre-existing inconsistency where those reads lived in `repository.py` while the matching writes (`add_jd`/`replace_levels`/etc.) already lived in `child_repository.py`. `PositionRepository`'s 24 public methods are name/signature-identical before and after (AST-diffed against `main`, independently re-derived by principal-reviewer, not just the implementing agent's `dir()` check) — the 3 external modules holding an instance (`applications/service.py`, `candidates/candidate_screenings/tasks.py` + `.../service.py`, `interviews/service.py` via `_service_helpers.py` — 4 call sites total across those 3 modules) needed zero changes; all 4 modules' test suites (1085 tests total) pass, plus `ruff`/`mypy` clean. `repository.py` remains 61 lines over the 300-line cap after this batch's named remedy (the 3-way split) — NOT fully closed by design, one concrete next step identified but deliberately deferred (not this batch's scope, avoids mid-refactor scope creep): `list()` (48 lines, itself a pre-existing 40-line-cap violation) builds its `count_sql`/`list_sql` strings inline where `list_helpers.py` already exists and already hosts the sibling `build_position_where` — moving that construction there would bring the file to ~340 lines and close `list()`'s function-cap violation as a side effect. 2 principal-reviewer rounds: round 1 CHANGES-REQUESTED (a stale WHY-comment in `subresource_service.py` and this doc's own CR-002 note both still asserted `current_jd`/`get_interview_level` live on `repository.py` post-move — the exact defect class Tier 3 batch 1 was blocked for, caught again here) — fixed same session. **Tier 3 frontend batch 1 (2026-08-27, branch `dev/hygiene-tier3-frontend-batch1`):** `feedback-list-drawer.tsx` 435→236 (BR-SEQ-001/D3/D4 edit-outcome sub-form extracted to `edit-feedback-outcome-form.tsx`); `offer-detail-card.tsx` 348→241 (status-driven action bar extracted to `offer-action-bar.tsx`); `mocks/org-handlers.ts` 309→122 (split into `org-store.ts` + `department-handlers.ts`, the named live-binding hazard closed via an explicit `getStore()` accessor). `interview-pipeline-progress-report.tsx` confirmed already resolved by 2 unrelated prior PRs (299 lines, under cap) — corrected the stale note instead of splitting a compliant file. 2 principal-reviewer rounds: round 1 CHANGES-REQUESTED (the new live-binding regression test wrote before reset and read after, so it could never fail on the bug it claimed to catch) — fixed by reordering to reset-then-write-then-read, empirically validated by both reviewer and implementer independently injecting the same stale-cache bug; round 2 APPROVE-WITH-NITS. **Tier 3 frontend batch 2 (2026-08-27, branch `dev/hygiene-tier3-frontend-batch2`):** `dept-form-drawer.tsx` 346→291 (banner stack extracted to `dept-form-status-banners.tsx`, 116 lines; seeding effect + all 14 `emitOrgEvent` calls, the named hazard, verified line-identical to `main`); `use-positions.ts` 305→17 lines (now a pure `export *` barrel over 4 new concern files — `use-position-queries.ts`/`use-position-jd.ts`/`use-position-levels.ts`/`use-position-reference-data.ts` — all 16 real importers resolve with zero edits, AST/grep-verified); `positions-list.tsx` 501→290 (columns + filter bar extracted; the plan's named filter-state design question resolved by NOT moving the debounced state, keeping the extracted filter bar purely presentational); `application-list.tsx` 402→214 (ARIA next-actions menu extracted byte-identical to `main`, confirmed via `diff`, plus a genuinely new `app-next-actions-menu.test.tsx` covering wraparound/focus-return/role-gating the parent's own tests never covered). Full frontend suite's pre-existing baseline (undercounted here at "5" until corrected 2026-08-28 during Tier 3 batch 3's review): **15 failures across 6 files** — `nav-items.test.ts` (2), `position-schema.test.ts` (3), `positions-list.test.tsx` (2), `position-form-drawer.test.tsx` (4), `status-change-dialog.test.tsx` (3), `interview-org-labels.test.tsx` (1) — independently confirmed identical on `main` via isolated worktree, unrelated to this batch. 1 principal-reviewer round: CHANGES-REQUESTED (zero `docs/` update — this same entry — the third consecutive instance of this exact defect class this Tier) — fixed same session. **Tier 3 frontend batch 3 — LAST FRONTEND BATCH, executed 2026-08-27, branch `dev/hygiene-tier3-frontend-batch3`:** `panelist-form-drawer.tsx` 716→438 (fee format/parse + `seedFrom`/`validate` extracted to `panelist-form-drawer.helpers.ts`; submit/error/lifecycle handlers extracted to a new `usePanelistFormSubmit` hook; Deactivate/Reactivate confirm blocks extracted to `panelist-lifecycle-actions.tsx` — the BUG-002/BUG-003 re-seed guard, `values`/`version` state + `seededVersionRef`, deliberately stayed in the drawer component, passed into the hook only as arguments, per the plan doc's own explicit warning against moving that state across a hook boundary); `candidate-upload-drawer.tsx` 617→216 (types/validation extracted to `candidate-upload-drawer.helpers.ts`; submit/207-partial-success handling to `useCandidateUploadSubmit`; the two mode forms to `single-resume-upload-form.tsx`/`bulk-resume-upload-form.tsx`, spec §10.5 progressive disclosure preserved verbatim); `create-interview-drawer.tsx` 483→410 (BR-SEQ-001 `categoryRank`/`getLevelReadiness`/`sequenceBlockMessage` extracted to a new zero-React-dependency `lib/interviews/level-sequence.ts`). Zero prior test coverage on the first two files — added 13+12 tests (panelist helpers/hook) and 15+9 tests (candidate helpers/hook); the interview file already had a 569-line suite (`create-interview-drawer.test.tsx`, 15 tests) plus `application-interview-panel.test.tsx` (4 tests), both re-run and confirmed green unchanged, plus a new 15-test `level-sequence.test.ts` covering every readiness branch directly for the first time. All 3 files' sole external importers (`panelist-list.tsx`, `candidate-list.tsx`, `application-interview-panel.tsx`) confirmed unchanged (`tsc --noEmit` + `eslint` + grep-verified import paths/prop shapes). `panelist-form-drawer.tsx` (438) and `create-interview-drawer.tsx` (410) remain over the 300-line cap after this batch — accepted residuals, same class as the `security/service.py`/`positions/repository.py` residuals above: further extraction would mean either moving BUG-002/BUG-003-adjacent state across the hook boundary the plan doc warned against, or decomposing JSX the task scope didn't call for. **Rule 4 gap closed same session** (principal-reviewer round 1 CHANGES-REQUESTED): the 64 new tests covered only the extracted pure helpers/hooks, not the JSX that got re-parented on the 2 previously-untested drawers — added `candidate-upload-drawer.test.tsx` (3 render-smoke tests) and `panelist-form-drawer.test.tsx` (2 render-smoke tests) confirming both drawers mount with their real fields and the category/mode toggles drive the correct conditional rendering; also removed 2 dead write-only `useRef`s in `single-resume-upload-form.tsx`/`bulk-resume-upload-form.tsx` (M1, pre-existing on `main` but shipped into brand-new files by this batch).
  - 🟡 `backend/app/modules/security/service.py` — 320 lines, 20 over the 300-line cap (was 389).
    **Accepted residual — deliberately not closed further.** The only remaining paths to 300 are
    touching login/session/refresh/logout (this file's highest-blast-radius auth paths, which the
    decomposition plan explicitly rules out touching) or more docstring compression (already tried
    once, cost `build_current_user` its Args/Returns contract — restored, not re-attempted).
    Consistent with the existing systemic-overcap note below (`interviews/service.py` alone is
    1413 lines) — one-off enforcement here would be inconsistent enforcement. Revisit only as part
    of a repo-wide cap policy decision, not a standalone pass. `principal-reviewer` verdict:
    acceptable. Tier 3 frontend batch 3 (2026-08-27) closed out the last 3 named frontend
    files — see the Tier 3 frontend batch 3 entry above. Tier 4 (backend + frontend, highest-risk)
    — DONE, see the Tier 4 entries below; the entire 48-file sweep is complete as of 2026-08-29.
  - 🟡 `backend/app/modules/positions/repository.py` — 361 lines, 61 over the 300-line cap (was
    457, before Tier 3 batch 2 above). The plan doc's named remedy for this file (split
    position-code + ageing-summary out, fold JD/interview-level reads into `child_repository.py`)
    is fully executed; the residual is in the position CRUD/list methods and the
    already-separately-split recruiter delegation wrappers, neither named by the plan doc for
    further extraction. Revisit only if a future pass specifically re-scopes this file.
- ✅ (resolved, confirmed 2026-08-27 during Tier 3 frontend batch 1 scoping) `frontend/src/components/reports/interview-pipeline-progress-report.tsx` — this entry's "366 lines, needs its own decomposition pass" is now stale: two later, unrelated fixes (`ea69919`/`b05a9d8`, the render-time page-clamp regression-test fix and the viewport-responsive page-size fix) already moved the report's spec-commentary block to `openspec/specs/reporting/spec.md` §3 and trimmed the component itself, bringing it to 299 lines (under the 300-line cap) with its own dedicated `interview-pipeline-progress-report.test.tsx`. No further split scheduled — re-flag only if it grows back over cap.
- ✅ `backend/app/modules/interviews/agents/level_kit_agent.py` — **resolved 2026-09-05,
  branch `dev/code-hygiene-level-kit-extraction`:** 431→286 lines, genuinely under the
  300-line cap. Three zero-logic extractions, same pattern each time: (1) the pure
  prompt/schema constants (`_SYSTEM`, `_USER_TEMPLATE`, `_OUTPUT_SCHEMA`,
  `_LEVEL_DEPTH_HINTS`, `_LEVEL_DEPTH_HINT_DEFAULT`) moved to a new sibling
  `_level_kit_prompts.py` (91 lines), re-imported back (`_SYSTEM`/`_OUTPUT_SCHEMA` via
  an explicit `as`-reexport since `test_level_kit_agent.py` imports both directly from
  `level_kit_agent`, one of them via an inline import inside a test method — caught
  only by actually running the suite, not by the top-of-file import grep); (2) the
  Gemini provider's retry-loop/request-building bodies moved to
  `_level_kit_gemini.py` (84 lines, `_call_gemini_impl`/`_invoke_gemini_impl`); (3) the
  Anthropic/Bedrock provider's retry-loop/request-building bodies moved to a new
  sibling `_level_kit_anthropic_bedrock.py` (141 lines, `_call_anthropic_impl`/
  `_invoke_anthropic_impl`/`_call_bedrock_impl`/`_invoke_bedrock_impl`) — the
  `llm_gateway.complete`/`asyncio.sleep` references (module-attribute mock.patch
  targets, same mechanism as `_genai`/`time.sleep` in (2)) stay resolved in
  `level_kit_agent.py`'s delegators and are passed into the free functions as
  already-resolved callables. `LevelKitAgent._call_anthropic`/`_invoke_anthropic`/
  `_bedrock`/`_invoke_bedrock`/`_call_gemini`/`_invoke_gemini` all kept as thin
  delegator methods with identical names/signatures. Every helper function in all 3
  new files is also ≤40 lines (confirmed via AST, docstrings tightened to fit where
  needed). Zero behavior change: `LevelKitAgent`'s public method names/signatures
  confirmed identical via `inspect`; `test_level_kit_agent.py` and `test_level_kit.py`
  — 350 passed/109 skipped before and after (same counts, same 1 pre-existing
  unrelated warning); `ruff` and `mypy --strict` clean on all 4 files. Same branch
  also closed `_extraction_tasks.py::_do_extract` via the identical pattern — see
  that file's own entry below. `principal-reviewer` round 1 (covering both files
  together): APPROVE-WITH-NITS, no Major/Critical — findings + fixes noted on the
  `_do_extract` entry below rather than duplicated here.
- ✅ **Tier 5 backend catch-up (2026-08-31, branch `dev/hygiene-tier5-backend-catchup`)** — a
  fresh re-audit found 3 backend files still/newly over the 300-line cap after the 48-file sweep
  closed. `candidates/agents/job_matcher.py` (**G14, file-cap half DONE, function-cap half still
  open — see G14's own row above**) 347→299: the module docstring's M3 offline-gate decision
  (previously stated 2+ times across the docstring) tightened to a single mention;
  `_MATCH_PROMPT_TEMPLATE` plus its JSON-fence-strip/parse/pad-to-5 normalization moved to a new
  sibling `_match_prompt.py` (94 lines — `build_match_prompt`/`parse_match_items`/
  `normalize_points`, zero dependency back on `job_matcher.py`, empirically confirmed via a fresh
  `import` — avoids a circular import); `match_candidate()` **82→55 total lines (not 81→28 as
  first reported — the "28" used a code-after-docstring count against an original total-lines
  baseline, corrected by `principal-reviewer`'s independent re-derivation; 55 is still 15 over
  the 40-line cap)** via the new `_build_llm_results()` helper (a real seam — LLM item→
  `MatchResult` construction was one cohesive block, extracting it left the provider-selection/
  fallback control flow as its own clean unit, but that remaining logic still needs its own
  follow-up pass). `positions/router.py` 360→191 (deferred from Tier 1 on 2026-07-29, a genuine
  process miss — never picked up in Tiers 2-4): mirrored the `recruiter_router.py`/
  `interviews/_router_*.py` sibling-router pattern — JD upload/get/re-extract/history split to
  `_router_jd.py` (132 lines), interview-levels + position-history split to
  `_router_levels_history.py` (94 lines). Both new sibling routers use a bare `APIRouter()`,
  matching the 5-of-5 existing `_router_*.py` convention — a first attempt added
  `tags=["positions"]` to match `recruiter_router.py`, but that file's explicit tag is a
  pre-existing wart (it double-tags 2 routes on `main` too), not the real convention; the addition
  broke the byte-identical-OpenAPI invariant below and was reverted (caught by
  `principal-reviewer`'s confirm-pass, which also caught that the fix commit's own claim of
  byte-identical OpenAPI had gone stale the moment the tags were added — corrected here).
  `candidates/schemas.py` (308 lines) — **evaluated,
  exempted, not split**: the inheritance-chain argument is real
  (`CandidateDetailResponse(CandidateSummary)`, plus fan-in from `OfferDetails`/
  `SourceDetailsResponse`/`ConsentResponse`/`MatchResponse`), and this file is smaller than both
  already-exempted siblings (`interviews/schemas.py` 321, `positions/schemas.py` 376) — see
  `docs/CODE_HYGIENE_DECOMPOSITION_PLAN.md`'s exempt list (narrowed wording per reviewer: NOT "no
  natural grouping at all" — `BulkUploadResponse`/`BulkUploadStatusResponse`/`BulkStatus` (~25
  lines) have zero back-reference from the main chain and could extract cleanly; the exemption
  rests on the inheritance-chain argument, not on there being nothing separable). Zero
  public-interface change: `job_matcher.py`'s AST top-level diff against `main` shows only 1 new
  private helper added; `_MATCH_PROMPT_TEMPLATE` (a module-level binding, not a function) was
  relocated with zero external consumers (grep-confirmed). **`positions/router.py`'s route SET
  and OpenAPI contract are byte-identical pre/post (verified by dumping `app.routes` and
  `app.openapi()` on both commits, plus resolving all 15 `/positions*` URLs through both route
  tables and confirming identical winners including `/ageing-summary` vs. `/{position_id}`) — but
  registration ORDER changed** (the mounted group now registers earlier, ahead of `/positions`,
  `/ageing-summary`, `/{position_id}`, `/{position_id}/status`). This is safe today because every
  mounted route is either a distinct segment-1 literal (`/recruiter-options` — which already had
  to precede `/{position_id}` on `main` too) or carries a distinct literal in segment 2+ (`/jd`,
  `/jd/re-extract`, `/jd/history`, `/interview-levels`, `/history`, `/recruiters`), so no mounted
  pattern matches a CRUD path — but it creates a latent constraint: a FUTURE route added directly
  in `router.py`'s own body with a `/{param}` or `/{position_id}/{param}` shape WOULD be shadowed
  by the mounted sub-routers. A comment recording this now lives in `router.py` itself, right
  above the 3 `include_router()` calls. 2 test-only patch-target fixes required in
  `positions/tests/test_router.py` (`test_upload_jd_enqueues_after_commit` /
  `test_re_extract_jd_enqueues_after_commit` patched `positions.router.extract_job_description`
  — now patches `positions._router_jd.extract_job_description`, since that's where the route
  handler actually lives; internal test wiring only, not a public-interface change). Gate 1:
  550 passed, 250 skipped, 0 failed (`candidates`+`positions` test suites). `ruff check` +
  `mypy` clean on all 5 touched/created files. 2 principal-reviewer rounds: round 1
  CHANGES-REQUESTED (G14's table row never flipped, a stale "Deferred from Tier 1" note left
  intact in the very commit fixing its consequence, the 81→28 metric mismatch above, and the
  false "registration order identical" claim — all doc-only, fixed same session; code itself was
  clean on first pass). Local-only per user instruction (GitHub Actions billing-blocked until
  2026-09-01) — not pushed, no PR.
- ✅ **Tier 5 frontend catch-up (2026-08-31, branch `dev/hygiene-tier5-frontend-a`)** — a fresh
  re-audit found 3 frontend files from the original 2026-07-29 plan (`docs/CODE_HYGIENE_
  DECOMPOSITION_PLAN.md`) that were never executed: 2 deferred from Tier 1 and never picked up in
  Tiers 2-4 (same class of process miss as `positions/router.py` above), 1 that WAS split at Tier 1
  but grew back over cap since. `positions/jd-panel.tsx` 473→293: the already-isolated
  presentational sub-tree (`providerLabel`/`extractionBadge`/`SkillGroup`/`OverridableText`/
  `ExtractedView`) extracted to a new `jd-extracted-view.tsx` (158 lines); the version-history
  toggle+list block extracted to a new, self-contained `jd-version-history.tsx` (75 lines — owns
  its own `historyOpen` state and `useJdHistory` call, since neither was read anywhere else in the
  panel). The `jdPanelView` pure selector stayed in `jd-panel.tsx` per the plan's own explicit
  constraint (it's imported directly by `jd-panel.test.tsx`). `candidates/match-card.tsx` 331→201:
  the 4 independent collapsible sections extracted to `match-points-section.tsx` (42),
  `match-gaps-section.tsx` (43), `match-legacy-scorecard-section.tsx` (71 — owns its own
  `scorecardOpen` state + lazy `useScorecard` call), and `match-dismiss-action.tsx` (73 — owns its
  own confirm-panel toggle; the actual dismiss mutation call/toast stays in the orchestrator,
  passed down as an `onConfirm` callback that never rethrows). This file had a real existing test
  suite (`match-card.test.tsx`, 4 tests) that only covered the match-points/gaps expand and the
  Screen action — the plan doc's "no dedicated test" note was stale; per Quality Gate Rule 4, added
  3 render-smoke tests covering the 2 sections the existing suite never touched (legacy-scorecard
  lazy-fetch-on-expand, dismiss confirm/cancel, dismiss confirm/call-endpoint), bringing the suite
  to 7 tests. `candidates/screening-detail.tsx`: re-diffed against its own Tier 1 split commit
  (`6f00e12`, 2026-07-29) rather than trusting the plan's "+1 line" guess — the real growth was
  +37 lines (267→304), all from the `async-pipeline-durability` D9 feature (the server-confirmed
  `question_generation_error` terminal-failure branch + "Check again" retry button), added after
  Tier 1's split closed. That whole 3-way branch (error / polling / gave-up-with-retry) was
  presentational and cleanly separable — extracted to a new `screening-question-generation-status.tsx`
  (93 lines); polling state (`pollingEnabled`) and the refetch call stay owned by the orchestrator,
  passed down as `pollingEnabled` + an `onCheckAgain` callback. File is now 304→264 lines. The
  optimistic-concurrency `version` state and `handleSaveResponses`/`handleSubmitOutcome` were not
  touched (confirmed by direct inspection, not just re-reading the header comment) — same
  constraint as Tier 1's own split of this file. All 3 files' real external importers
  (`position-detail.tsx` → `JdPanel`, `candidate-matches-section.tsx` → `MatchCard`,
  `candidates/[id]/screenings/[sid]/page.tsx` → `ScreeningDetail`) confirmed unchanged — same
  export name/props, grep-verified. Zero behavior/accessibility/ARIA change anywhere (hygiene-only
  split, no design work). Verification: all 3 affected suites green (`jd-panel.test.tsx` 13,
  `match-card.test.tsx` 7, `screening-detail.test.tsx` 5 — 25 total, 0 failed); `tsc --noEmit`
  clean repo-wide; `eslint` clean on all 11 touched/created files. Local-only per user instruction
  (GitHub Actions billing-blocked until 2026-09-01) — not pushed, no PR.
- ✅ **Tier 5 frontend catch-up, batch B (2026-08-31, branch `dev/hygiene-tier5-frontend-b`) —
  CLOSES THE ENTIRE 9-FILE TIER 5 CATCH-UP.** 3 more files, none of them in the original
  2026-07-29 plan doc — organic growth since that plan's source list was frozen, so this batch
  did fresh discovery rather than executing an existing analysis. `lib/types/positions.ts` (315
  lines) evaluated and **exempted, not split**: first write-up claimed ~46 importers and
  comment-grouped domains — `principal-reviewer` independently re-verified and found both false
  (35 real non-test importers, 48 including tests; types are interleaved across domains, not
  grouped), corrected here. The real reason to leave it alone: 6 `extends` chains cross what
  would be separate domain files (`PositionResponse`/`PositionDetailResponse`,
  `JDResponse`/`JDSummary`, `PositionCreate`+`PositionUpdate`/`BudgetInputs`,
  `StatusUpdate`/`NoShowCapture`) — a domain split would convert these into cross-file type
  dependencies between the new files, real structural coupling, not import-statement churn; see
  `docs/CODE_HYGIENE_DECOMPOSITION_PLAN.md`'s exempt list for the full reasoning.
  `interviews/interview-kit-drawer.tsx` 315→193: `getKitRetryState` +
  `COMPLEXITY_LABEL`/`COMPLEXITY_COLOR`/`TYPE_LABEL` (pure, no React import) extracted to
  `interview-kit-drawer.helpers.ts`; the two presentational sub-trees extracted to sibling
  components — `QuestionRow` → `interview-kit-question-row.tsx` (73 lines), `FocusAreaSection` →
  `interview-kit-focus-area-section.tsx` (48 lines). `getKitRetryState` is re-exported from the
  main file (`export { getKitRetryState };`) so `interview-kit-drawer-retry-state.test.ts`'s
  import path needed zero changes. `candidates/screening-start-dialog.tsx` 315→263: the position
  dedup + existing-screening-status-map logic (pure, no React import) extracted to
  `screening-start-dialog.helpers.ts` (`mergeOpenAndInProgressPositions`/
  `buildExistingScreeningStatusMap`); the position radio-list section extracted to
  `screening-position-list.tsx` (84 lines, presentational only — loading/error/empty states stay
  in the parent per the plan's established pattern). This file had **zero prior test coverage**
  (per Quality Gate Rule 4) — added `screening-start-dialog.test.tsx` (2 render-smoke tests:
  position selector renders the real seeded open/in_progress positions; select + submit actually
  POSTs to `/candidates/{id}/screenings` and surfaces the returned `screening_id` to the caller).
  The submit test's real-wiring claim was verified, not assumed: the `onScreeningCreated(...)`
  call was briefly severed, the test was confirmed to fail (0 calls, not a stale assertion), then
  reverted and reconfirmed green — closing the exact vacuous-test class principal-reviewer caught
  on the immediately prior Tier 5 batch. All 3 files' existing/external importers confirmed
  unchanged (same export names/props; `interview-kit-drawer.tsx`'s 2 real component consumers
  — `application-interview-panel.tsx`/`my-interviews-list.tsx` — and
  `screening-start-dialog.tsx`'s sole importer, `applications-in-candidate-card.tsx`, needed zero
  edits). Zero behavior/accessibility/ARIA change anywhere. Verification: `vitest run` on all 3
  affected test files — 11/11 passed (2 + 7 + 2); `tsc --noEmit` clean repo-wide; `eslint` clean
  on all 8 touched/created files. Local-only per user instruction (GitHub Actions billing-blocked
  until 2026-09-01) — not pushed, no PR.
- ✅ `backend/app/modules/candidates/_extraction_tasks.py::_do_extract` — **fully resolved
  2026-09-05, branch `dev/code-hygiene-level-kit-extraction`:** both the function cap
  (`_do_extract` was 138 lines) and the file cap (this fix's own function-level split had
  regressed the file to 347 lines) are now genuinely met. `_do_extract` split within the
  file first into 6 named async helpers — `_load_and_claim`, `_load_file_bytes`,
  `_run_agent` (re-raises `TransientProviderError`/`SoftTimeLimitExceeded` uncaught exactly
  as before — confirmed by reading how Celery's `autoretry_for` on the task decorator
  consumes the propagated exception, not just assumed), `_persist_extraction` +
  `_persist_duplicate` (split into two, not the one originally planned, because the single
  version landed at 48 lines), plus the pre-existing `_find_identity_duplicate`/
  `_persist_checked`/`_set_failed` — then ALL 8 of those helpers moved wholesale to a new
  sibling `_extraction_helpers.py` (259 lines), matching the `_extraction_mapping.py`
  precedent in the same directory; `_do_extract`/`_run_extraction` stay in
  `_extraction_tasks.py` (now 120 lines), importing the helpers back. **Judgment call:**
  `storage.read_object` and `extract_profile` are passed into `_load_file_bytes`/
  `_run_agent` as explicit parameters rather than imported directly by
  `_extraction_helpers.py`, because `test_tasks.py` mock.patches them as plain names on
  `_extraction_tasks` itself (`_extraction_tasks.storage.read_object`,
  `_extraction_tasks.extract_profile`) — a plain-name patch only rebinds the attribute in
  `_extraction_tasks`'s own module dict, so a separate import in the new file would have
  silently run the REAL `extract_profile`/`storage.read_object` instead of the test's mock
  — identified by tracing the patch-resolution mechanism before moving the code (same class
  of gap as the `_genai`/`time.sleep` constraint already solved for `level_kit_agent.py`
  above), then confirmed correct by the full suite passing after the move. Every helper in
  both files is ≤40 lines (AST-confirmed); every incident-history comment (BUG-003, D2/D3/D7,
  principal-reviewer round-1 M5) preserved verbatim. Zero behavior change:
  `_do_extract(session, candidate_id) -> str` signature confirmed byte-identical via
  `inspect`; `test_tasks.py` + the rest of `candidates/tests/` — 350 passed/109 skipped
  before and after (same counts, same 1 pre-existing unrelated warning); `ruff` and
  `mypy --strict` clean on both files. `principal-reviewer` round 1: APPROVE-WITH-NITS
  (no Major/Critical) — 2 stale "below"/"above" doc-position references in
  `_extraction_tasks.py`'s module docstring pointing at code that had moved (fixed —
  now name the actual function); 3 new lines exceeding this project's declared
  100-char line-length in `_extraction_tasks.py`/`_extraction_helpers.py` (fixed by
  wrapping — flagged separately below that `ruff` structurally cannot catch these,
  since `[tool.ruff.lint]` never `select`s E501); and a docstring rationale in both
  `_extraction_helpers.py` and `_level_kit_anthropic_bedrock.py` that conflated two
  different `mock.patch` mechanisms as if they were the same constraint (Gemini's
  `_genai`/`time.sleep` and `extract_profile` are true name bindings that must stay
  where they're patched; `storage.read_object`/`llm_gateway.complete`/`asyncio.sleep`
  are module attributes that would resolve correctly either way, passed in for
  call-site uniformity only, not because it was required) — all fixed inline, re-
  verified (350 passed/109 skipped unchanged, ruff/mypy clean, all 6 files across
  both entries still under 300 lines).
- 🔴 **`pyproject.toml`'s `[tool.ruff.lint]` declares `line-length = 100` but never
  `select`s anything that enforces it** (found 2026-09-05, `principal-reviewer`,
  reviewing `dev/code-hygiene-level-kit-extraction`) — ruff's default rule set
  (`E4`/`E7`/`E9`/`F`) does not include `E501`, so no line-length violation anywhere
  in this repo has ever been caught by CI; a "ruff clean" claim on any past or future
  PR is silently uninformative about the project's own stated 100-char limit.
  Confirmed several pre-existing lines already exceed 100 chars on `main` (left
  untouched by the PR above — only its own 3 new violations were fixed). Cheapest
  sound remediation: add `E501` to `[tool.ruff.lint] select` in its own dedicated
  pass — NOT a same-PR fix, since turning it on now would light up files far outside
  any single change's scope; needs its own scoped cleanup pass first.
- **Tier 4 (backend, highest-risk) — `applications/service.py` + `applications/_service_helpers.py`
  executed 2026-08-27, branch `dev/hygiene-tier4-applications-service`:** per the user's own
  explicit decision on the plan doc's one open design question (update all test `mock.patch`
  targets to the new split locations — no re-export shim to preserve the old single-location
  `write_audit_log` patch convention). `_service_helpers.py` 621→0 (deleted, fully redistributed
  into 3 new files per its own 3 separable groups): `_status_rules.py` 230 lines (BR-003 transition
  matrix + BR-018 mandatory-field validation), `_response_builders.py` 135 lines (pure
  response-shape builders), `_interview_sync.py` 273 lines (BR-013/BR-SYNC-005/Issue-5
  cross-module interview-outcome sync — `LEVEL_TO_APP_STATUS` landed here first, before
  `service.py`'s own split, exactly as the plan doc required). `service.py` 667→346 lines, split
  into `_service_creation.py` (147 lines, BR-014/BR-021), `_service_transitions.py` (341 lines,
  update_status/withdraw_application write paths + the 3 shared gate helpers'
  bodies + `get_valid_next_statuses`' body), `_service_recruiter.py` (110 lines, BR-022/BR-023).
  `ApplicationService`'s public + private method set and every signature confirmed
  byte-identical before/after via AST diff (no method added/removed/renamed/re-signatured).
  The 3 gate helpers (`_position_is_closed`/`_offer_pipeline_eligible`/`_no_show_gate_satisfied`)
  deliberately kept as thin, same-named wrapper methods ON the class in `service.py` — several
  existing unit tests patch them at `ApplicationService.<name>` or call them as bound methods
  directly (`svc._no_show_gate_satisfied(...)`), discovered only by reading the actual test
  files rather than assuming the plan doc's "move to one of the new files" phrasing meant "move
  the method off the class" — every split `do_*`/`compute_*` function calls back through the
  bound `svc` instance to reach them, so patch/call semantics are unchanged. Interviews module's
  cross-module import (`interviews/service.py` imports `auto_set_pending_on_interview_create`/
  `auto_sync_from_interview_outcome`/`revert_to_pending_from_interview_outcome_cleared`) and
  `applications/tasks.py`'s `TERMINAL_STATUSES` import, plus 2 backend-scripts files
  (`backfill_legacy_feedback_outcome.py` + its unit test) all repointed to `_interview_sync.py`
  — confirmed each is a 1-line import-path change only (`git diff --stat`), zero logic touched.
  9 test files' `mock.patch` targets updated (exact count matched the plan's own "~9" estimate,
  independently verified via a full-suite grep, not assumed): `test_service.py` (38 occurrences
  across create/update_status/withdraw/update_recruiter/sync_recruiter, mapped to
  `_service_creation`/`_service_transitions`/`_service_recruiter` per which method each guards),
  `test_unit_p4_1_offer_pipeline_gate.py`, `test_unit_dropped_offer_declined.py`,
  `test_unit_hiring_uniqueness_applications.py`, `test_unit_p4_1_onboarded_lockdown_recruiter.py`,
  `test_unit_p6_2_position_closed_guard.py` (3 different targets across its 6 tests, split by
  which of the 3 guarded methods each covers), plus 3 import-path-only fixes
  (`test_unit_p35d_override_reason.py`, `test_unit_p37_list_display_status_label.py`,
  `test_unit_phaseb_43_valid_next_statuses.py`). `test_unit_p4_1_offer_pipeline_gate.py`'s and
  `test_unit_dropped_offer_declined.py`'s existing `ApplicationService._offer_pipeline_eligible`
  patches needed NO changes (confirmed by reading the tests first) — exactly because that gate
  stayed a class method. BR-SYNC-005/exception-propagation behavior confirmed unchanged: the
  auto-sync functions' try/no-op-return structure and `update_status`'s exact exception-raise
  ordering were moved verbatim (git diff shows deletion+identical re-addition, no logic edits).
  417 applications+interviews unit tests, 6 backfill-script unit tests, `ruff`, and `mypy` all
  clean. 🟡 Accepted residuals (same class as the `security/service.py`/`positions/repository.py`
  residuals above): `service.py` 346 lines (46 over) and `_service_transitions.py` 341 lines (41
  over) — the 3 gate helpers + `get_valid_next_statuses` could not be fully removed from
  `service.py` without breaking existing test patch/call semantics, and their bodies (already
  lengthy, pre-existing docstrings preserved verbatim rather than trimmed) had to land somewhere;
  splitting them into a 4th new file was out of the dispatched scope (only 3 new files were
  named). `interviews/service.py` + `interviews/_service_helpers.py` (1413 + 367 lines) were the
  next, larger Tier 4 item — see the entry below (done).
- **Tier 4 (backend, highest-risk) — `interviews/service.py` + `interviews/_service_helpers.py`
  executed 2026-08-28, branch `dev/hygiene-tier4-interviews-service`:** `service.py` 1494→492
  lines (grew slightly past the file's original 1413-line count before this batch started, per
  actual `wc -l` at dispatch time — not a discrepancy in this batch's work), split by the plan
  doc's 10 responsibility groups into **7** new `_service_*.py` files (not 6 — "redo/repeat" and
  "feedback-outcome override" were combined into one file first, then re-split into
  `_service_redo.py`/`_service_outcome_override.py` once the combined file (403 lines) was found
  to exceed the cap; per the dispatch brief's own "do not force exactly 6 if 7 fits the natural
  seams better"): `_service_creation.py` (202 lines, creation+sequencing), `_service_reads.py`
  (147 lines, reads), `_service_scheduling.py` (315 lines, scheduling/status+panelists),
  `_service_feedback.py` (359 lines, feedback+BR-SYNC-005 sync), `_service_kits.py` (145 lines,
  level-kit orchestration), `_service_redo.py` (262 lines, redo/repeat), `_service_outcome_
  override.py` (175 lines, feedback-outcome override). `_service_helpers.py` 373→0 (deleted,
  redistributed into 3 new files by responsibility, not absorbed into the service split files):
  `_response_builders.py` (145 lines, pure response-shape builders), `_create_validators.py`
  (122 lines, BR-001/level-membership/BR-SEQ-001 sequencing gates), `_feedback_helpers.py` (122
  lines, panelist eligibility/BR-SYNC-006/row builders — `_check_panelist_eligibility` is the one
  symbol genuinely shared across two service-split files, `_service_feedback.py` and `_service_
  outcome_override.py`). `InterviewService`'s 20 public methods (incl. `__init__`) confirmed
  name/signature-identical before vs after via AST diff — zero drift; 6 previously-bound private
  helper methods (`_sync_feedback_outcome`/`_revert_feedback_outcome`/`_sync_interview_status`/
  `_publish_interview_scheduled_event`/`_validate_redo_eligibility`/`_apply_feedback_outcome_edit`)
  became free functions in their new files, confirmed zero test coupled to them as bound methods
  (no `patch.object(InterviewService, "_sync_..._outcome", ...)` anywhere in the suite). The 3
  interleaved exception-handling conventions named as this file's real hazard were AST-diffed
  function-by-function against `main` (try/except handler type + reraise/swallow flags compared
  programmatically, not just read): BR-SYNC-005 (`_sync_feedback_outcome`/`_revert_feedback_
  outcome` in `_service_feedback.py`) — 2 bare-`Exception` swallow blocks each, zero reraises,
  identical to `main`; BR-SEQ-001 (`do_create_interview` in `_service_creation.py`) — zero
  try/except blocks, identical to `main` (the gate's `LevelSequenceViolationError` propagates
  unguarded); the redo/edit-outcome revert path (`do_redo_interview` in `_service_redo.py`,
  `do_edit_feedback_outcome`/`_apply_feedback_outcome_edit` in `_service_outcome_override.py`) —
  zero try/except blocks around `revert_to_pending_from_interview_outcome_cleared`, identical to
  `main`. `_create_interview_row`'s/`_create_or_recover_feedback`'s pre-existing `IntegrityError`
  handling (reraise vs recover, BR-002/P36) also confirmed unchanged. Test-file migration: 12
  test files' `mock.patch` targets retargeted from `app.modules.interviews.service.<symbol>` to
  the new per-flow module (`_service_creation`/`_service_scheduling`/`_service_feedback`/
  `_service_kits`/`_service_redo`/`_service_outcome_override`), following the same per-user
  decision as the applications Tier 4 batch (update call sites, no re-export shim) —
  `test_service.py` alone had ~50 sites across its create/schedule/update_status/add_panelist/
  remove_panelist/submit_feedback sections, mapped individually by which moved function each one
  guards; `test_unit_p6_2_position_closed_guard.py`'s single shared `_AUDIT`/`_REVERT` constants
  had to split into per-guarded-method constants (`_AUDIT_SCHEDULING`/`_AUDIT_FEEDBACK`/
  `_AUDIT_OUTCOME`/`_AUDIT_REDO`, `_REVERT_OUTCOME`/`_REVERT_REDO`) since one file covers all 7
  guarded methods across 4 different destination modules. **Review round 1 caught 2 Major +
  5 Minor** (principal-reviewer, CHANGES-REQUESTED) — fixed same session: (M1) the 2
  `test_get_interview_read_only_audit_not_called`/`test_list_interviews_read_only_audit_not_
  called` tests were retargeted to patch `app.shared.audit.write_audit_log` directly, which does
  NOT intercept the module-local `from ... import write_audit_log` bindings every write-path file
  holds — the assertion could never fail; confirmed by injecting the actual regression and
  watching the test pass for the wrong reason. Fixed by patching `_service_reads.write_audit_log`
  with `create=True` (today the attribute doesn't exist so the assert holds trivially; the
  moment a regression adds the import, this patch intercepts it) plus an explicit
  `assert not hasattr(_service_reads, "write_audit_log")` documenting the real guarantee. (M2)
  `_service_scheduling.py`'s header claimed its BR-SYNC-001 swallow "mirrors BR-SYNC-005's
  logging convention" when the actual code was a silent `except: pass` — fixed by adding the
  missing `_logger.warning(...)` call (closing a pre-existing NFR §4 violation carried into a
  brand-new file, same remediation class as Tier 3 frontend batch 3's dead-`useRef` fix) and
  correcting the header to state the log call was added in this commit. (m1-m2) 2 stale
  intra-module pointers still naming `_service_redo.py` for logic that actually lives in
  `_service_outcome_override.py` (`_feedback_helpers.py`'s header, `service.py:9`) — corrected.
  `test_service.py`'s direct `_create_interview_row` import/4 `Interview`-class patches also
  repointed to `_service_creation`. 417 interviews+applications unit tests, `ruff`, and `mypy`
  all clean. 🟡 Accepted residuals (same class as prior Tier 3/4 residuals above): `service.py`
  492 lines (192 over — 20 public methods with full docstrings is inherently larger than
  applications' 13-method file), `_service_scheduling.py` 315 (15 over), `_service_feedback.py`
  359 (59 over, the single cohesive BR-SYNC-005 group) — none split further without either
  fragmenting a cohesive BR group or duplicating docstrings between the class wrapper and the
  `do_*` function (the mandate's own "do not duplicate" instinct, applied here to keep the class
  file's per-method docstrings to a 2-3 line pointer at the `do_*` function rather than the full
  BR prose, which is what brought `service.py` down from an initial 547-line first draft).
- **Tier 4 (backend, mechanical but high external-caller fan-out) —
  `applications/repository.py` + `interviews/repository.py` executed 2026-08-28, branch
  `dev/hygiene-tier4-repositories`:** `applications/repository.py` 561→444 lines — the 8 raw SQL
  string constants (`_LIST_SQL`/`_COUNT_SQL`/`_DETAIL_SQL`/`_INTERVIEW_SUMMARY_SQL`/
  `_STATUS_HISTORY_SQL`/`_HEADCOUNT_SQL`/`_ORG_REJECTION_SQL`/`_SCHEDULED_INTERVIEW_LABELS_SQL`)
  extracted to a new `_queries.py` (129 lines, byte-identical content — SHA-256 verified —
  de-underscored names, aliased back to the original private names so no method body changed);
  `ApplicationRepository`'s 15 methods
  left intact in one class, per this doc's own guidance against a riskier mixin/method split
  given how many external modules (`applications/service.py`+`tasks.py`, `offers/`, `screening/`,
  `interviews/service.py`+`_service_creation.py`) import the class by name — confirmed via grep,
  zero of them needed any change since they only ever import the class, not the module-level SQL
  constants. `interviews/repository.py` 1039→382 (core file) + 5 new mixin files split by the
  named query family (verified zero cross-method calls, matching this doc's plan-time analysis):
  `_repo_panelists.py` (115 lines), `_repo_feedback.py` (173), `_repo_sequencing.py` (193),
  `_repo_redo.py` (131), `_repo_my_interviews.py` (143) — combined into `InterviewRepository` via
  multiple inheritance (MRO confirmed conflict-free: `InterviewRepository` →
  `PanelistRepositoryMixin` → `FeedbackRepositoryMixin` → `SequencingRepositoryMixin` →
  `RedoRepositoryMixin` → `MyInterviewsRepositoryMixin` → `object`). Core file keeps CRUD/status-
  history/level-kit methods plus `_DETAIL_SQL`/`_LIST_SQL` (both depend on the shared
  `CATEGORY_RANK_SUBQUERY`); confirmed `test_category_rank_regression.py` actually imports
  `CATEGORY_RANK_SUBQUERY` from the already-promoted `app/shared/sql_fragments.py` module (this
  doc's older note describing it as a private constant of `repository.py` was stale from before
  the 2026-08-26 promotion — corrected here), so nothing needed to change for that test either
  way. Confirmed via grep — zero callers of `interviews.repository` outside the module itself
  (`service.py`, `tasks.py`, `_kit_context.py`, `_service_creation.py`, `_service_feedback.py`,
  `_service_redo.py`, and the module's own tests) — matches this doc's "repository correctly
  stays module-private" note. Both classes' public methods AST-diffed against `main` (name +
  positional/keyword-arg signature comparison): zero symmetric difference on either class, zero
  signature drift. Full unit suite (417 passed, 0 failed, 294 skipped — pre-existing DB-gated
  skips unrelated to this batch), `ruff check`, and `mypy` all clean on both modules. 🟡 Accepted
  residuals (same class as prior Tier 3/4 residuals above): `applications/repository.py` 444
  lines (144 over — 15 cohesive methods with no natural sub-grouping, exactly the plan doc's own
  prediction) and `interviews/repository.py` 382 lines (82 over — core CRUD + status-history +
  level-kit methods were not among the 5 named split families, so they stayed together;
  splitting them further wasn't in scope and risked the exact "riskier second pass" this doc
  warns against elsewhere). Remaining Tier 4 items after this batch: `position-form-drawer.tsx`
  and `application-status-drawer.tsx` (both frontend).
- **Tier 4 (frontend, `position-form-drawer.tsx` — biggest file in the sweep) executed
  2026-08-29, branch `dev/hygiene-tier4-frontend`:** 965→519 lines. Split into a pure-helpers
  file (`position-form-drawer.helpers.ts`, no React import, unit-testable without a DOM) + 4
  presentational fieldset components (`position-basics-fieldset.tsx`,
  `position-hiring-manager-fieldset.tsx`, `position-jd-upload-fieldset.tsx`,
  `position-recruiter-assignments-fieldset.tsx`) + a banner-stack component
  (`position-form-status-banners.tsx`). `handleSubmit`'s 3-step chain (position write →
  non-blocking recruiter assignments → non-blocking chained JD upload, each with its own error
  semantics) stayed intact in the orchestrator per this doc's own explicit constraint — confirmed
  by reading the function body, not the header comment's claim alone. Both external importers
  (`position-detail.tsx`, `positions-list.tsx`) confirmed unchanged (same `PositionFormDrawer`
  export name/props). `position-form-drawer.test.tsx`: 4 of 7 tests fail — independently
  confirmed identical (same count, same failing assertions, same error location) against
  unmodified `main` via a stash-and-rerun isolation, not a regression from this split. `tsc
  --noEmit` and `eslint` clean on all 7 touched/created files. **Note on how this batch ran:** the
  dispatched `ux-ui-engineer` agent stalled mid-turn (harness-reported "no progress for 600s")
  right after completing this split but before the doc-update/commit step — `ListAgents`
  confirmed no live process, the split itself (verified via `tsc`, `eslint`, and the test
  isolation above) was already complete and correct, so this doc entry and the commit were done
  directly rather than re-running the whole dispatch. 🟡 Accepted residual (same class as the
  backend residuals above): `position-form-drawer.tsx` stays 519 lines, 219 over the 300-line
  cap — the remaining size is the drawer's own state block plus the plan-mandated 3-step
  `handleSubmit` chain (~105 lines, itself over the 40-line function cap but explicitly sanctioned
  by this doc's own constraint that the chain must not be distributed into children).
- **Tier 4 (frontend, `application-status-drawer.tsx` — LAST Tier 4 item, closes out the entire
  48-file sweep) executed 2026-08-29, same branch `dev/hygiene-tier4-frontend`:** 369→290 lines.
  Split into a pure-helpers file (`application-status-drawer.helpers.ts`, no React import —
  `computeFilteredStatuses`, `buildFilteredPendingReasons`, `getRejectionFieldCopy`, plus the
  shared `selectClass`/`textareaClass`/`inputClass` field-styling constants), a banner-stack
  component (`application-status-banners.tsx`), and a presentational reason/date-fields component
  (`application-status-reason-fields.tsx` — the pending-reason/rejection-reason/hold-reason/
  tentative-DOJ/onboarded-at/offer-declined-reason fields, visibility driven entirely by the
  `needs*` flags the orchestrator already computed, zero business logic in the child). Per this
  doc's own explicit constraint, `validate()` and `handleSubmit()` stayed co-located, unmoved, in
  the orchestrator. This file's header names 4 business rules as its densest documentation of
  anything in the sweep — all 4 confirmed preserved by reading the actual code, not the header
  comment alone: **D6** (interview-status-lifecycle-phaseb) — the status dropdown's option set is
  the backend's live `valid_next_statuses`, currentStatus pinned per M-1 — moved verbatim into
  `computeFilteredStatuses`, logic unchanged. **D10** (interview-status-lifecycle-phaseb) — a 409
  `OFFER_PIPELINE_NOT_ELIGIBLE` is shown as a plain, verbatim error with no skip-recovery
  affordance — untouched, still inline in `handleSubmit`'s `catch` block. **D3/D4** (positions-
  closed-lockdown-phasec) — a closed position disables Save status and shows an explanatory
  banner, the backend's own 409 `POSITION_CLOSED` staying authoritative — untouched logic
  (`positionClosed` still computed in the orchestrator), only the banner's JSX moved into
  `ApplicationStatusBanners`. Confirmed only 2 real external importers (`application-list.tsx`,
  `applications-in-position-card.tsx`), both unaffected — same `ApplicationStatusDrawer` export
  name/props; a 3rd grep hit in `application-interview-panel.tsx` was a stale doc-comment, not an
  import. `application-status-drawer.test.tsx` (the file's own pre-existing 5-test suite, already
  covering D6/D10/D3/D4 directly) — all 5 pass unmodified against the split version; since none
  failed, no stash-and-rerun pre-existing-failure isolation was needed. `tsc --noEmit` and `eslint`
  clean on all 4 touched/created files. **This is the last remaining item in the entire 48-file
  Tier 1-4 code-hygiene decomposition sweep — Tier 4, and the sweep as a whole, is now complete.**

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
  - ✅ P5 — unbounded sub-collection endpoints. `applications`/`interviews` `/status-history` now paginate (limit/offset, default 50/max 200); spec updated same-PR. `positions/{id}/history` follow-up now DONE (`dev/optimization-positions-history-pagination`, 2026-08-31): `router.py`/`subresource_service.py`/`repository.py` thread `limit`/`offset` through to a SQL `LIMIT`/`OFFSET` (not a Python slice), same `_MAX_HISTORY_LIMIT = 200`/default 50 as applications, sort order (changed_at DESC) unchanged; `openspec/specs/positions/spec.md` updated same-commit. Frontend caller (`use-position-queries.ts`'s `usePositionHistory`) had no "load more" affordance and rendered the full list unconditionally — rather than silently truncating to 50, it now explicitly requests `limit=200` to preserve current behavior; a proper paginated UI (if a position ever exceeds 200 history rows) remains unscheduled tech debt.
  - ✅ P7/P6 — safely-parallelizable sequential awaits (~5 DB round trips/request), CLOSED. `set_rls_context` (2→1 round trip, unit-tested in `backend/tests/unit/test_core_database.py` including placeholder-to-GUC pairing so a bind-swap can't silently pass), `User.role` selectin→joined, 4 repos' count+rows → `COUNT(*) OVER()` (departments' assertion moved to the passing `test_list_without_search_omits_like` per M2), `Role.permissions` `lazy="selectin"` → `lazy="raise"` per M4 (closes the last round trip; only usage anywhere in `backend/` is a class-level `.join(Role.permissions)` in `security/repository.py`, unaffected by the loader-strategy change). N1 (unused fixture param), N2 (per-fixture post-teardown verification), N3 (idempotent teardown safety net) also closed. principal-reviewer: APPROVE after remediation round.
    - **Root cause of the pre-existing `departments` test failure, now confirmed and FIXED:** `openspec/specs/departments/spec.md` AC-004 requires the list ordered by name; `departments/repository.py` ordered by `updated_at desc` instead — spec-vs-code drift, a real defect. Corrected on `dev/tech-debt-batch1` (PR #199), merged together with this Phase 2b's `COUNT(*) OVER()` windowed-count optimization on the same query — see §5 above. `organizations`' equivalent failure had no ordering AC in its spec, so that one was a genuinely stale test — fixed alongside (test assertion only, repository untouched).
    - ✅ Follow-up (N5) investigated 2026-08-29, closed — NOT removed, for a different reason than
      first thought. `departments/repository.py`/`positions/repository.py`'s
      `set_org_scope(org_id)` looked like an extra `set_config` round trip on top of
      `set_rls_context`. First investigation pass (wrong, corrected same session by
      `principal-reviewer`'s independent re-trace) claimed it was load-bearing for internal STG
      staff — it is not: `set_rls_context` sets `app.is_internal='true'` for exactly the users
      who have no `organization_id` (`core/dependencies.py`), and every org-scoped RLS policy in
      this schema contains an `fn_is_internal()` disjunct (3 of the 8 also add a leading
      `organization_id IS NULL`; `candidate_source_details` is `USING (fn_is_internal())` alone)
      — so the disjunction is satisfied regardless of `fn_current_org()`'s value for an internal
      user, and the org GUC `set_org_scope` sets cannot affect the outcome for them.
      `app.current_org`/`fn_current_org()` are consumed nowhere outside these RLS policies (no
      trigger/view/column default/app SQL) — verified independently, only 2 `current_setting`
      call sites exist in the whole schema, both inside `fn_current_org()`/`fn_is_internal()`
      themselves. **But this is NOT "redundant-and-safe" on the other branch, and is NOT kept as
      defense-in-depth — see the new §7 item below.** For org-scoped (non-internal) users,
      `positions/_service_writes.py`'s write path passes a CALLER-SUPPLIED `organization_id` into
      `set_org_scope` before the INSERT, which (since `rls_positions_isolation` is a `FOR ALL`
      policy with no separate `WITH CHECK`, defaulting to the same `USING` predicate) actively
      MOVES the RLS check to whatever org the request claims, rather than confirming the actor's
      own — this is a latent authz weakening, not a safe no-op, on the one branch where the GUC
      is actually live. Kept in place only because untangling it is a real authz change (needs an
      actor-org check added to the write path), not a perf-cleanup decision — see §7.
  - ⏸️ P8/P9 — unrefreshed materialized views remain deferred to pre-go-live (unchanged by the
    fix below — explicitly pre-production risk per auditor, not a current bottleneck).
    - ✅ **`position_history` index-gap instance closed** (2026-09-01, `dev/nfr-pos-hist-index`,
      migration `0061_pos_hist_id_time_idx`). Found 2026-08-29 (`principal-reviewer`, reviewing
      the `positions/{id}/history` pagination fix): the only usable index,
      `idx_pos_hist_pos_type_time (position_id, change_type, changed_at DESC)`, orders rows by
      `(change_type, changed_at)` within a `position_id`, which does NOT satisfy `list_history`'s
      `ORDER BY changed_at DESC` — Postgres fetched and sorted every row for the position before
      LIMIT. Fixed with a new additive `(position_id, changed_at DESC)` index
      (`idx_pos_hist_pos_time`); the 3-column index is kept as-is (still correct for any future
      `change_type`-filtered query). Live-verified via `EXPLAIN (ANALYZE, BUFFERS)` with 30,009
      synthetic rows for one position (deleted after verification): before —
      `Limit -> Sort (top-N heapsort, 30,009 rows) -> Bitmap Heap Scan`, 732 buffer hits, 4.641ms;
      after — `Limit -> Index Scan using idx_pos_hist_pos_time`, no `Sort` node, 4 buffer hits,
      0.098ms (~47x faster, ~180x fewer buffer reads). Migration upgrade→downgrade→re-upgrade
      round-tripped cleanly against local Postgres.
- ⏸️ Phase 2c — load-testing harness + 200-250 concurrent-user capacity validation. Auditor's own view: numbers would be meaningless until P0/P2 land (now done) — still nothing built, tool choice (Locust/k6/custom) an open decision for the user.
- ✅ Watch-item resolved 2026-07-28 (see P4 above): `_pipeline_progress_sql.py`'s event CTEs now join only against the current page's positions, not the full matched-position set.
- ✅ Watch-item (originally logged 2026-07-31/08-02 against the now-deleted
  `_pipeline_progress_all_levels_sql.py`, superseded by PR #209's status-groups redesign;
  corrected + re-diagnosed + fixed 2026-09-04 against the CURRENT file,
  `_pipeline_progress_group_sql.py`, `dev/pipeline-progress-group-join-perf`). Concern (a)
  [regex vs indexable enum equality on `ash.new_status`] — CONFIRMED NON-ISSUE: live-measured
  switching it to an enum-equality changed nothing (829ms vs 845ms) — no index exists on
  `new_status` to exploit, the planner already picks the equivalent path either way. Concern
  (b) [event-CTE-to-outer join evaluated on a computed `CASE` expression, not an indexable
  equijoin] — ✅ FIXED: `build_level_group_rows_sql`'s `events` CTE (~line 137) is now
  `AS MATERIALIZED`, forcing one-time computation instead of Postgres inlining it into a
  plan that re-executed the events subplan once per outer grid row (O(size^2) — measured
  440ms at the router's max page size 100 pre-fix). `MATERIALIZED` guarantees only that
  `events` is evaluated exactly once (a distinct `CTE events` node, `Actual Loops == 1`) —
  NOT any particular outer-join node type, which is cost-model-dependent and can legitimately
  be a Nested Loop at small scale even on a correct, fixed plan (round-1 principal-reviewer
  finding, Major-1: an earlier version of `test_perf_pipeline_progress_group_join.py`
  asserted the outer join was never a Nested Loop — a stronger claim than the fix makes —
  and failed on correct code; corrected to assert only the single-evaluation invariant,
  which holds at any seed scale). Live-EXPLAIN independently re-confirmed at probe scale
  (500 positions): 237.5ms -> 22.3ms. Paired with a size cap (`router.py`'s `size` param:
  max 100 -> 25) as a defense-in-depth bound. `build_status_group_rows_sql`'s own separate
  `events` CTE (~line 190) was checked and left untouched — `MATERIALIZED` measured zero
  difference there (it already gets a good plan unaided).
  NEW tracked item (secondary finding from the same diagnostic, NOT fixed by this PR): a
  redundant `positions p` join inside `position_sub_dims` in the same file —
  `p.id IN (SELECT id FROM page_positions)` could read `il.position_id IN (...)` directly
  since the two sets are already equal via the join condition. Cheap change, but touches an
  RLS-relevant predicate (positions carries an RLS policy) — needs `principal-reviewer`'s
  explicit sign-off on the isolation argument before anyone touches it, not a drive-by edit.
  Round-2 CI failure (real, fixed same PR): the new test initially hardcoded a local-dev DSN
  password (matching 3 other pre-existing `RUN_FUNCTIONAL_TESTS`-gated functional-test files'
  convention) — but this test is `RUN_DB_TESTS`-gated, which CI actually sets and runs
  (unlike `RUN_FUNCTIONAL_TESTS`), so it broke on CI's different Postgres credentials. Fixed
  by deriving the DSN from `settings.DATABASE_ADMIN_URL` instead (matching `app/scripts/
  check_schema_definition_drift.py`'s established pattern), fail-loud if unset. The 3
  other files' hardcoded-password convention remains untouched, pre-existing debt — noted
  so it isn't "discovered" again later, but those never run in CI so they don't share this
  specific failure mode.

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
- ✅ (closed — moot, 2026-08-18) `reconcile_screenings` beat-scheduling precondition. The task (and the rest of `candidates/screening/tasks.py`) was deleted entirely by `candidate-ai-match-screen-consolidation`'s retirement of that module's match-decision write path — matching is now a single on-demand "AI Job Match" trigger (`job_matcher.py`), not a fan-out needing reconciliation. No scheduling preconditions remain to track.
- 🔴 **DPDP retention enforcement + reporting materialized-view refresh — placeholder task bodies, no owner (async-pipeline-durability, deferred from Phase 2 by principal-reviewer round 1, C3).** Both beat-schedule-worthy jobs were REMOVED from `celery_app.py`'s `beat_schedule` this phase (not added, as an earlier draft of this change incorrectly claimed) because their task bodies are still placeholders: `data_privacy.enforce_data_retention` (`app/modules/data_privacy/tasks.py`) is an empty no-op, `reporting.refresh_reporting_views` (`app/modules/reporting/tasks.py`) raises `NotImplementedError`. The DPDP one is the higher-priority gap — `proposal.md` itself calls it "a live compliance exposure" (specced as a daily storage-limitation sweep, nothing currently anonymises anything). Needs: implement the real task body for each (querying `candidates.retention_expires_at` for the DPDP one; refreshing whichever materialized views `reporting/spec.md` names for the other), THEN add both back to `beat_schedule` (`enforce-dpdp-retention` daily, `refresh-reporting-materialized-views` every 10 min — cadences already decided, just needs real bodies to schedule against).
- ✅ **`level_kit_agent.py`'s `_invoke_anthropic`/`_invoke_bedrock` now route through the
  shared `llm_gateway`** — `dev/level-kit-agent-llm-gateway`, 2026-09-03. Both paths now call
  `llm_gateway.complete()` (timeout-bound clients, circuit breaker, normalized provider
  errors) instead of building raw `anthropic.Anthropic(...)`/`boto3.client(...)` clients;
  `llm_gateway` gained an optional `schema=` kwarg (anthropic-only) to preserve the
  structured-JSON output constraint. Gemini's native path is unchanged (interim, out of
  scope). See `openspec/specs/interviews/spec.md` changelog, 2026-09-03 entry.
- ✅ **D8's circuit breaker is per-process, not cross-process** (async-pipeline-durability,
  flagged Phase 4 by principal-reviewer round 1, Major-2) — closed 2026-09-04,
  `dev/llm-circuit-breaker-redis`. `_breaker_is_open`/`_breaker_record_result` in
  `llm_gateway.py` now read/write Redis (`llm_breaker:{provider}:fail_count` /
  `:tripped` keys, via the existing `app.core.redis.redis_client` singleton) instead of a
  module-level dict, so all `docker-compose.yml` prefork worker child processes share one
  view of the breaker state. Redis's own key TTL replaces the old monotonic-timestamp
  compare; a Redis outage fails OPEN (logs a warning, proceeds with the real provider
  call) rather than blocking dispatch. Tests rewritten onto `fakeredis` in
  `test_llm_circuit_breaker.py`. **Round-1 review caught a real regression before merge**
  (Major-1): the fail_count key's cleanup TTL was set only on the FIRST failure and never
  refreshed — since a real provider retry chain runs ~240-280s (longer than the fixed
  cleanup window), the count would silently expire before reaching threshold on exactly
  the slow/hanging-provider scenario this file exists to catch. Fixed: TTL now refreshes
  on every failure (a genuine sliding window), with a new test proving failures spaced
  across real time still trip the breaker.
- 🔴 **`wrap_bedrock_error` classifies `NoCredentialsError` as `TransientProviderError`**
  (found 2026-09-04, functional-test-engineer + principal-reviewer, reviewing
  `dev/llm-circuit-breaker-redis` — confirmed pre-existing via `git log`, last touched PR
  #217, not introduced or worsened by that branch). `NoCredentialsError` is caught by
  `wrap_bedrock_error`'s `BotoCoreError` catch-all branch (not the `ClientError` branch),
  so a missing/misconfigured AWS credential — a permanent config error, retrying will
  never fix it — burns the full retry budget AND counts toward/trips the circuit breaker,
  masking the real problem as a transient provider outage and delaying diagnosis. Needs
  its own scoped fix: classify `NoCredentialsError` (and similarly non-retryable boto3
  config errors) as `PermanentProviderError` instead.
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
