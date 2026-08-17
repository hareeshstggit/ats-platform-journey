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

**`candidate-ai-match-screen-consolidation` (CR#1) — executes second, after CR#1.A.** User CR,
confirmed + fully documented 2026-08-08 — full OpenSpec change at
`openspec/changes/candidate-ai-match-screen-consolidation/` (proposal/design/specs
delta/tasks all complete, `openspec status` confirms 4/4). User's explicit instruction: this is
the FIRST thing built when the dev freeze lifts, ahead of everything else in this backlog —
spec documented first (done), then build → test → review → merge, full Gate 5 pipeline, no
shortcuts. Summary: stop auto-firing matching/screening on upload (extraction-only); collapse
"Trigger Job Matching"/"Trigger AI Screening" into one LLM-primary "AI Job Match" trigger
(offline scorer demoted to fail-safe-only fallback); hard-gate results at a configurable ≥75%
match (default), each result carrying a reworked, pair-specific 5+5 (match points / gaps to
verify); unify screening-question generation onto one mechanism (`candidate_screenings/`)
used by both a new per-position "Screen" action and the existing top-right entry point;
retire `candidates/screening/`'s match-decision+scorecard write path (table stays, per the
additive-migration rule). Read the OpenSpec change's own design.md for full rationale,
decisions, and open verification items before starting task 1.1.

**`interview-kit-candidate-aware-scheduled-generation` (CR#2) — executes third, after CR#1.**
User CR, confirmed + fully documented 2026-08-08 — full OpenSpec change at
`openspec/changes/interview-kit-candidate-aware-scheduled-generation/` (4/4 artifacts complete).
No schema change needed — `InterviewLevelKit`
already stores `candidate_id`; the actual gap is that generation never used it
(`LevelKitAgentContext`'s own docstring says "no candidate PII," prompt says "never invent facts
about a specific candidate" — confirmed via code read, not assumed). Summary: the 5 questions
per focus area become candidate-experience-aware (non-PII signal only), the 10 focus areas stay
position-driven (user's own ≤10% cross-candidate variation estimate); kit generation drops its
create-time trigger (`router.py:131`), keeping only the schedule-time trigger
(`router.py:212`); "Schedule Interview" action renamed to "Schedule Interview & Generate
Interview Kit"; `local_kit` offline path retained as fail-safe-only, same pattern as the CR
above. Confirmed unrelated to that CR's retired `candidates/screening/` scorecard — `interviews/`
has its own independent `scorecard_template` (BR-P20-007), untouched by either change.

### 0.1 Hard blockers (not scope choices — must happen regardless of scope decisions below)

| # | Blocker | Status | Notes |
|---|---|---|---|
| G1 | Terraform `iam` module is a skeleton (`task_role_arn`/`execution_role_arn` both `null`) | 🔴 | Blocks `terraform apply` outright — nothing else in AWS can start until real. |
| G2 | No AWS account / OIDC deploy role / Secrets Manager populated | 🔴 | `deploy.yml` fails immediately without `AWS_DEPLOY_ROLE_ARN`. User-owned action item (account provisioning), not a code fix. |
| G3 | `deploy.yml` has never run successfully end-to-end, even once | 🔴 | Needs a real dry run against a live AWS env before trusting it for go-live. |
| G4 | RDS not provisioned in cloud | 🔴 | Only local Podman Postgres exists today; migrations never run against a cloud DB. |
| G5 | Domain, TLS, WAF, CDN not applied | 🔴 | IaC authored, never applied. |
| G6 | Security pen-test + DPDP DPIA not scheduled | 🔴 | External/compliance-owned; needs lead time to schedule a vendor/reviewer. |
| G7 | Performance/SLO load testing (NFR Phase 2c) | 🟡 | **First real baseline recorded 2026-08-17** (`docs/ARCHITECTURE.md`'s SLO section, GH Actions run [32053886296](https://github.com/hareeshstggit/ats-platform-project/actions/runs/32053886296)) — all 4 read endpoints missed the <150ms p95 target (218-231ms measured), login missed <300ms badly (4874ms p95, likely single-worker queueing not bcrypt cost). **This is a local dev-stack/CI-runner baseline, not production** — real production validation is G11, still 🔴. Not yet 🟡→✅ since the harness has run only once, against a topology (2-vCPU runner, single uvicorn worker, co-located generator) known to differ from intended production sizing. |
| G8 | UAT + runbooks + backup/DR + on-call readiness | 🔴 | None started. |
| G9 | **Bedrock Claude models are NOT natively hosted in `ap-south-1`** — access from India works only via Bedrock's Global Cross-Region Inference (verified live 2026-08-08, see sources in conversation history). Real candidate/resume PII would leave India during 4 AI-feature inference calls even though app/DB/S3 stay in `ap-south-1`. | ❓ Awaiting legal/DPO | Feeds directly into D1 (DPDP scope) and D3 (Bedrock-vs-Gemini sequencing) below — get this confirmed by legal/DPO before committing to Bedrock as the AI backend; could reshape the AI-provider decision entirely. |
| G10 | API Fargate service (the actual user-facing HTTP compute) doesn't exist in Terraform yet | 🔴 | Only worker/beat/cwagent services are authored (`ecs` module). No ALB, no API service, no sizing decided. Recommended starting point: 2-3 tasks, 1 vCPU/2GB each, autoscale to 4-5 under peak (see conversation history 2026-08-08 for full NFR-grounded sizing rationale — RDS `db.r6g.large` Multi-AZ+replica, ElastiCache `cache.t4g.small` already right-sized, Celery worker's `desired_count=1` cap needs its own metrics fix before scaling). **G7's first real baseline (2026-08-17) directly supports this multi-task recommendation** — a single-worker uvicorn process under load showed a strongly bimodal login latency (median 29.67ms, p95 4874.5ms), the signature of requests queueing behind one saturated worker, not a fixed per-call cost — treat sizing as a starting request informed by real data, still not a locked number until G11's production run. |
| G11 | **Post-go-live production load-test validation** (added 2026-08-10, follows CR#1.A `nfr-response-time-slo-validation`) — re-run the same k6 harness against real deployed AWS infra once it exists; validate ECS autoscaling triggers fire before the SLO breaches (~350-400ms, not at 500ms); validate CloudFront's real effect on LCP once the CDN skeleton is applied (G5); reconcile local-dev-stack baseline numbers vs. real production numbers in `docs/ARCHITECTURE.md`. | 🔴 Blocked on go-live | **Explicitly deferred until after deployment to real AWS infra** — not Bedrock-specific (AI-feature latency is exempt from these targets by design, D4), this is about needing Multi-AZ RDS/ElastiCache/ECS autoscaling to actually exist before measuring against them. The k6 harness itself, and the FIRST (local) baseline, are built and run now as part of CR#1.A — this gate is only the second, production run. |
| G12 | **Frontend LCP/INP measurement + generator/SUT co-location caveat** (added 2026-08-17, principal-reviewer Majors 4+5 on `nfr-response-time-slo-validation`) — k6 cannot measure browser paint/interaction metrics (it is not a browser); LCP ≤2.0s/INP ≤200ms (`docs/ARCHITECTURE.md` SLO section) need a separate browser-based tool (e.g. Lighthouse CI) as a follow-up, not built as part of this harness. Separately: `load-test.yml`'s k6 generator, the FastAPI backend, Postgres, and Redis all run co-located on ONE 2-vCPU/7GB GH Actions runner (single-worker uvicorn) — any VU count run there measures runner CPU contention as much as application latency; `docs/PERFORMANCE_TESTING.md` caps the workflow's default VU count accordingly for THIS topology. Reaching the spec's real 200-250-concurrent-user target needs either a larger runner or separate generator/SUT infra. | 🔴 | Cross-ref G11 (post-go-live production re-run) — this row is the PRE-go-live local/CI-topology caveat; G11 remains the separate, later, real-infra validation. |

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
| 4 | Frontend test debt — nav-items.test.ts, position-schema.test.ts, others (6+ pre-existing failures) | 🔴 Queued (4th) |
| 5 | e2e CI job-design gap — MSW can't intercept proxied backend calls | ✅ Closed (PR #221, 2026-08-08) — real backend + Postgres/Redis boot before Playwright's `webServer`; `BACKEND_ORIGIN` feeds `next.config.mjs`'s proxy rewrite, closing the server-side ECONNREFUSED gap. `auth.spec.ts`'s stale "welcome back" assertion updated in the same PR. |
| 6 | `terraform-plan.yml` CI check has no path to ever pass — no AWS-credentials step exists anywhere in the workflow (confirmed via `git log`: file untouched since the original project-scaffold commit `537b06d`), so `terraform init -backend-config=environments/<env>/backend.hcl` cannot authenticate to the real S3 backend. **Discovered 2026-08-07 (PR #219, async-pipeline-durability Phase 6)** — this is the FIRST PR in the project's history to touch `infrastructure/terraform/**`, so the workflow's `on.pull_request.paths` filter never triggered it before now. Not caused by Phase 6; not fixable within any single PR's scope — needs a real AWS account, GitHub Actions secrets (`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` or an OIDC role), and an `aws-actions/configure-aws-credentials` step added to the workflow, which is an infra/credentials decision requiring explicit user approval, not a code fix. Every future PR touching Terraform will show this same 3-way (`dev`/`staging`/`prod`) failure until it's addressed. | 🔴 Queued (6th) |
| 7 | Local dev-stack watchdog (`scripts/dev-stack-watchdog.ps1`) — 2 minor environment quirks found 2026-08-08, neither blocking: (a) a Startup-shortcut `-Watch` loop was found still running from a prior session, silently racing any manual process kill/restart by immediately re-healing whatever was stopped — worth a note in `docs/LOCAL_DEV.md` that `-Watch` mode processes must be stopped explicitly (`Stop-Process` on the `powershell.exe -File ...-Watch` PID) before manually restarting the stack, not just the child services; (b) `uvicorn --reload`'s own internal reloader subprocess sometimes re-execs on system Python (`AppData\Local\Programs\Python\Python312\python.exe`) instead of the venv interpreter the watchdog script explicitly launched it from (`$VenvPython`), even though the script's `Start-Process -FilePath $VenvPython` call is correct — likely a Windows-specific `sys.executable`/PATH resolution quirk inside uvicorn's `WatchFiles` reloader, not a script bug. Functionally harmless observed so far (health checks and Celery task processing both succeeded on the resulting process), but worth a root-cause pass if it ever causes a real dependency mismatch (system Python312 may not have every backend dependency installed). Not investigated further this session — out of scope for the task in progress. | 🟡 Queued, low priority |
| 8 | **Prompt caching for the 4 AI-calling agents** (JD extraction, screening-question generation, candidate matching, interview-kit generation) — none of the 4 currently use `cache_control` (confirmed via repo-wide grep, zero matches), so the large, mostly-static system-instruction block each agent sends gets paid for in full on every single call, even though the same instructions repeat across hundreds/thousands of calls with only the job-description/resume/candidate content actually varying. **Scope (2026-08-08):** (a) split each agent's prompt into a static "instructions" segment (rubric, output-schema instructions, few-shot examples if any) + a dynamic "content" segment (the actual JD/resume/candidate text) — this restructuring is the real work, not the caching call itself; (b) mark the static segment with `cache_control: {type: "ephemeral"}` in `llm_gateway_providers.py`'s Anthropic/Bedrock call sites (Gemini's `google-genai` SDK has an analogous context-caching API, different shape — check `google.genai.types` for the equivalent before wiring it, don't assume the Anthropic shape carries over); (c) verify empirically (per the binding live-verification mandate) that a cache hit actually reduces billed input tokens for at least one real call, not just that the API accepts the parameter — a silently-ignored cache_control that still bills full price would be worse than not building this, since it would look optimized without being optimized; (d) Bedrock-specific note: prompt caching support and minimum cacheable-prompt-length vary by model within Bedrock (check the specific Claude model version's documented cache support before assuming parity with direct Anthropic API). **Where this lands:** JD extraction and interview-kit generation are the best first targets (longest, most static system prompts per the go-live checklist's own feature descriptions — 1-5 rubric + 10 screening Qs + 10 focus areas × 5 Q&A for kits); screening-question generation and matching likely have shorter/more-variable prompts, lower priority. Not blocking — do this as its own scoped phase, ideally timed alongside or just after the AWS Bedrock go-live migration (Bedrock's actual model+pricing will be known by then, informing exact savings estimate). | 🟡 Queued, scoped, high value |
| 9 | **Live AI cost/token-usage tracking + daily admin email digest.** User's ask: consolidate token/cost usage across all AI-calling features into a live view, and email ALL admin-role users a daily digest at 22:00 IST, timed for just before the AWS Bedrock go-live migration. **Feasibility: YES**, and every piece the mechanism needs already exists in this codebase, confirmed 2026-08-08: (a) per-call token counts are already captured in code (`llm_gateway_providers.py`, `level_kit_agent.py`) — currently discarded after the call, not persisted anywhere; (b) real email fan-out via AWS SES already exists and is live (BACKLOG #2, PR #196) — this is a new notification TYPE on an already-built delivery mechanism, not a new integration; (c) Celery beat + timezone-aware crontab scheduling already exists (Phase 2 of `async-pipeline-durability`, PR #215) — a `22:00 Asia/Kolkata` (IST has no DST, fixed UTC+5:30 year-round, so a plain UTC crontab offset is safe with no seasonal-drift risk) beat entry is the same pattern already in `celery_app.py`'s `beat_schedule`; (d) admin-user enumeration is a straightforward query against the existing RBAC roles in `app/modules/security/` (`super_admin`/`hr_admin` or whichever role set counts as "Admin" — confirm the exact role list with the user when this is actually scheduled for build, don't guess it now). **Design decision to make when building (not now):** self-tracked token-count-based cost ESTIMATE (persist per-call token counts + known per-model $/token pricing, compute an estimate — simpler, no AWS API dependency, but an estimate can drift from the real bill if pricing changes or a call type is missed) vs. a live pull from the AWS Cost Explorer API (`ce:GetCostAndUsage`, authoritative real billed $ broken down by service/model if cost-allocation tags are set up on the Bedrock resources — but Cost Explorer data has a documented ~24h refresh lag, which is actually FINE for a "yesterday's spend" daily digest at 22:00, just not usable for true real-time). Recommend Cost Explorer as the authoritative source for the EMAIL digest (real $ the user actually cares about) plus the self-tracked token counters as a supplementary real-time signal for the "live usage" view mentioned in the ask (a dashboard/metric, not the email) — this two-source design avoids the self-tracked estimate silently drifting from the real bill while still giving a live (not 24h-stale) number somewhere. **Not blocking now** — explicitly scoped by the user to be picked up just before AWS Bedrock go-live, once the real model/pricing/IAM setup is known. | 🟡 Queued, scoped, deferred to pre-Bedrock-go-live |
| 10 | **Downstream AI calls (matching, screening) fire unconditionally even when extraction just flagged the row a duplicate.** Verified 2026-08-08 (user's own concern, confirmed via direct code read, not speculation): `backend/app/modules/candidates/_extraction_tasks.py`'s `_run_extraction` sets `duplicate_of_candidate_id` when `_find_identity_duplicate()` finds a match (lines ~157-166), but the SAME function unconditionally calls `celery_app.send_task("candidates.match_candidate_to_positions", ...)` and `celery_app.send_task("candidates.screen_candidate", ...)` a few lines later (lines ~194-200) regardless of whether `duplicate_id is not None` — so a candidate already known to be a duplicate still burns a full AI matching call (scored against every open position) and a full AI screening-question-generation call, for a record that will presumably be merged/dismissed. **Two dedup layers already correctly gate the AI-cost-relevant part of the pipeline that CAN be gated pre-call**: file-hash dedup (`resume_sha256`, `check_file_dedup()` in `_upload_helpers.py`) runs BEFORE any extraction/AI call at all, for byte-identical re-uploads — this part is already optimized, no work needed. Identity-level dedup (name/email/mobile match) is structurally UNAVOIDABLE pre-extraction — those fields don't exist until extraction produces them, so extraction itself cannot be skipped for a not-yet-known duplicate; this is a genuine architectural constraint, not a gap. **Scope for the actual fix:** in `_run_extraction`, skip both `send_task` calls when `duplicate_id is not None` (the one extraction call already ran and is sunk cost either way; matching+screening are the avoidable ones). Confirm with the eventual reviewer/spec whether a duplicate candidate should still get SOME downstream processing (e.g., does the recruiter workflow ever want to see match/screening results on a flagged-duplicate row before deciding to merge vs. keep separate — check `openspec/specs/candidates/spec.md` for any existing business rule on this before assuming "always skip" is correct). | 🔴 Queued, scoped |
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
- ✅ `tests/unit/test_seed_dev.py::test_run_seed_fresh_creates_users_grants_and_audits` (2026-08-08, `chore/ci-real-db-e2e-fix` round 3) — root-caused: the test's hardcoded `== 7` was stale (2026-07-23 mypy-cleanup pass) AND it compared `summary["users"]` (covers both `TEST_USERS` + `NAMED_RECRUITER_USERS`) against `len(TEST_USERS)` alone, an existing scope mismatch independent of the stale count. Fixed to compute `total_users = len(TEST_USERS) + len(NAMED_RECRUITER_USERS)` instead of a magic number, so it self-adjusts as either list grows.
- ✅ `departments/tests/test_repository.py::test_list_scopes_to_org_searches_and_orders` (2026-07-29, `dev/tech-debt-batch1`) — root-caused as a real code defect (spec AC-004 requires name-order, code ordered by updated_at desc). Fixed the repository, not the test. Independently confirmed by NFR Phase 2b's own review (same root cause, same conclusion) — merged together with PR #197's `COUNT(*) OVER()` windowed-query optimization on the same query.
- ✅ `organizations/tests/test_repository.py::test_list_applies_search_and_active_filters_and_orders_by_updated_at` (renamed from `..._orders_by_name`, 2026-07-29, `dev/tech-debt-batch1`) — root-caused as a genuinely stale test (organizations spec has no ordering AC). Fixed the test's assertion, repository untouched.
- 🔴 Pagination edge case inherited from PR #197's `COUNT(*) OVER()` conversion (departments/organizations + 4 other repos): a page request past the end of the result set (`offset >= total`) returns `total = 0` (from `rows[0].total_count if rows else 0`) instead of the true total — the old separate `SELECT count(*)` didn't have this gap. A UI landing on an out-of-range page (e.g. after deletions) would show "0 results" and lose the ability to page back. Found during PR #199's merge-conflict review (2026-08-02); not fixed there (out of scope for that change) — needs its own fix across all 6 converted repos.
- 🔴 3 pre-existing `positions` module test failures: `test_service.py::test_change_status_stale_version_raises_409` (test itself is stale — needs a rewrite to a real reachable scenario), `test_tasks.py::test_extract_storage_miss_persists_failed` + `test_extract_success_persists_result` (mock/fixture drift, not chased to root cause).
- 🔴 No frontend component test for `positions-ageing-report.tsx` / `positions-ageing-bucket-strip.tsx`.
- ✅ = Execution Queue item 3: backend CI provisions real Postgres/Redis (`chore/ci-real-db-e2e-fix`, PR #221, 2026-08-08) — `test` job now gets `postgres:18`/`redis:7` services + `docs/ci_schema_snapshot.sql` + `alembic stamp head` + `RUN_DB_TESTS=1`. The ~18 tests that failed for lacking real services now pass. See the new needs_db entry below for what surfaced once `RUN_DB_TESTS=1` actually ran those tests for the first time. **Separately, the `test` job's own coverage gate still fails** (`Coverage failure: total of 66 is less than fail-under=80`) — pre-existing, unchanged across this PR's commits, same class of pre-existing CI blocker as the `typecheck`/`component-test` entries below (needs its own coverage-raising pass, not caused by or fixed by this PR).
- 🔴 = Execution Queue item 4: frontend `nav-items.test.ts` (2 failures, stale expected nav list) + `position-schema.test.ts` (3 failures) — both pre-existing, confirmed multiple times, never fixed.
- 🔴 `frontend/src/components/positions/interview-org-labels.test.tsx` fails on `main` (pre-existing, root-caused during `dev/interview-level-multi-panelist` review — unrelated to CR-002). The read-only interviewer view in `interview-levels-editor.tsx`'s `!canConfig` branch (~line 175-193) renders only `l.level_label` (e.g. "Bar raiser") and never the org-prefixed `"({Org} : Level-N)"` format the test expects. Needs a fix to either the component (add the org-prefix label to the read-only row) or the test's expectation — flag both `interview-org-labels.test.tsx` and `interview-levels-editor.tsx` when picked up.
- ✅ = Execution Queue item 5: `e2e` Playwright job now provisions a real backend + Postgres/Redis before Playwright's `webServer` boots (`chore/ci-real-db-e2e-fix`, PR #221, 2026-08-08) — `BACKEND_ORIGIN` feeds `next.config.mjs`'s `/api/v1/*` rewrite so Next's server-side proxy calls reach something real instead of ECONNREFUSED. `frontend/e2e/auth.spec.ts`'s stale "welcome back" assertion updated to the current universal `/reports` landing in the same PR. The systematic ECONNREFUSED/MSW-vs-real-backend failure is closed; 3 `mobile`-project (Pixel 5) tests hit genuine issues newly reachable now that mobile e2e exercises real interactive UI, all 3 quarantined + logged separately below: `auth.spec.ts`'s logout (topbar overlap), `organizations.spec.ts`'s create-org submit (drawer overlap), `pipeline-retry-badge.spec.ts` (locator-strategy gap, NOT the `page.goto()` bug first suspected -- see its own entry, which WAS real and independently fixed via nav-link navigation matching `gotoOrganizations`/`gotoPositions`). **Confirmed green via real CI (gh run 31250303308):** 43 passed / 60, 2 flaky (retry-masked, see the webkit entry directly below), 15 skipped (the quarantines + pre-existing skips), 0 hard failures.
- 🟡 **2 flaky `webkit`-project e2e tests, retry-masked (real CI, gh run 31250303308, PR #221 round 4, 2026-08-08)** — `organizations.spec.ts:46` (write-role create-an-organization) and `pipeline-retry-badge.spec.ts:76` (Failed/Failed-after-3-attempts). Both time out on `frontend/e2e/helpers/auth.ts:31`'s `page.waitForResponse` for `POST /auth/login` (Playwright's 30s test timeout), then pass on the automatic retry — originally a DETERMINISTIC every-run pattern. **Structural root cause fixed (`fix/e2e-webkit-flake-prod-build`, commit `1fad9ac`, PR #222, 2026-08-08)** — switched CI's Playwright `webServer` from `next dev` to a production build+start, removing the Next.js on-demand-route-compile race (confirmed via code read: `auth.ts` already used the correct `Promise.all([waitForResponse, click])` pattern, so this was never a test-code race). **Result across 4 independent real CI runs on this fix (run 31253719568 x2, run 31255151924, run 31256030051):** 3 runs clean (`45 passed, 15 skipped`, 0 flaky), 1 run showed `organizations.spec.ts:46` flaky again (1-in-4, vs. the prior 2-every-run deterministic pattern) — a large reduction, not full elimination. Residual is consistent with ordinary webkit-on-Linux-CI network-stack timing variance, not a compile race, not a code defect in the test or app — exactly the class of thing `retries: 1` exists to absorb, and did (job still reports overall success). **User decision (2026-08-08): merge as-is, log residual here rather than chase further** — not investigating deeper unless the residual rate climbs meaningfully above 1-in-4 in future runs.
- 🔴 **CI's e2e job runs `next start`, not production's actual deployment artifact (found via principal-reviewer round 2, `fix/e2e-webkit-flake-prod-build`, 2026-08-08).** `next.config.mjs` sets `output: "standalone"` and `frontend/Dockerfile` ships `CMD ["node", "server.js"]` — production runs the standalone server, but CI's `webServer.command` now runs `npm run build && npm start` (`next start` over the regular `.next` build), confirmed harmless for THIS PR's purpose (route-precompile, proven by 45 passing tests through real logins) but a residual environment-parity gap. **Also flagged, unverified — needs a live check before anyone relies on it either way:** whether `next.config.mjs`'s `rewrites()` closure resolves `BACKEND_ORIGIN` at build time (baked into the standalone bundle) or at runtime (`next start`/`node server.js` both re-read `process.env` per request) — if build-time-baked, prod's actual runtime env var value wouldn't behave the way CI's `next start` path does today. Not chased in that PR (out of scope, minimalism floor) — needs its own investigation.
- 🔴 **`[mobile] auth.spec.ts::logout returns to login` QUARANTINED for the `mobile` project only (PR #221 round 3, 2026-08-08)** — live-verified via real CI (gh run 31249164374): the "Log out" button's click is intercepted by the sibling role-label `<span>` ("Recruiter", `user-menu.tsx`) on the Pixel 5 viewport — a genuine topbar responsive-layout overlap (the `UserMenu` flex row doesn't fit the banner's other content at this width). Newly surfaced because mobile e2e never reached real interactive UI before this PR. Needs a UX pass (`.claude/rules/ats-ux-ui-guardrails.md`), not a blind CSS patch from a CI-infra change.
- 🔴 **`[mobile] organizations.spec.ts::create an organization → it persists and appears in the list` QUARANTINED for the `mobile` project only (PR #221 round 3, 2026-08-08)** — live-verified via real CI (gh run 31249164374): the "Create organization" submit click is intercepted by a sibling `<div class="space-y-2 flex-1">` field-group element inside the create drawer's fieldset, on the Pixel 5 viewport. A genuine drawer/fieldset responsive-layout overlap, same class of newly-surfaced mobile gap as the logout entry above. Needs its own UX pass.
- 🔴 **`[mobile] pipeline-retry-badge.spec.ts` QUARANTINED for the `mobile` project only (PR #221 round 3, 2026-08-08)** — real CI (gh run 31249164374) showed the `/candidates` heading never appearing; first suspected to be the same `page.goto()` session-drop bug found and fixed elsewhere in this PR (this spec's own `loginAsHrAdmin` + `page.goto("/candidates")` did in fact have that bug, independently fixed via a `gotoCandidates` nav-link helper matching `gotoOrganizations`/`gotoPositions`) — but round 3's own local live-verification (isolated `--project=mobile` run, retried) found the failure PERSISTS after that fix, for a genuinely different reason: on the Pixel 5 viewport the candidates table renders as a definition-list card layout (`list`/`listitem`/`term`/`definition`) below the table breakpoint, with no `role="row"` element at all, so this test's `page.getByRole("row", { name: ... })` locators match zero elements and every row-scoped assertion fails. A locator-strategy gap this spec never had to handle before real mobile e2e existed (previously ran only against MSW-mocked auth, never reaching this render at all). Needs a mobile-aware locator (e.g. scope by the enclosing `listitem` instead of `role="row"`), not chased here.
- 🔴 **`needs_db`/`RUN_DB_TESTS=1` integration tests actually ran in CI for the first time ever (PR #221, 2026-08-08)** — surfaced 72 pre-existing failures across 7 files, entirely unrelated to the CI-provisioning change itself (never caught before because these tests had never executed against a live DB in CI). Root-caused live against a real snapshot-loaded Postgres: **fixed** 24 of the 72 — `tests/integration/test_positions_flow.py` (24→6), `test_positions_defects_flow.py` (9→4) — root cause was 2 shared test fixtures (`_create_position`) missing the required `PositionCreate.approved_at` field ("Change-1", no default), added after these tests were last exercised; fix is test-only (added the field to the fixture payloads) — plus `tests/unit/test_seed_dev.py`'s own stale `== 7` assertion (round 3, see its own ✅ entry above), which is why this section no longer lists it under "remaining". **48 remain, NOT fixed here, each its own root cause:** (a) 27 in `test_interview_panelists.py` + 5 in `test_interview_levels_panelist.py` — `category="internal"` panelist creation requires an explicit `org_name` (`interview_panelists/service.py::_resolve_org_name` raises `OrgNameRequiredError` rather than auto-resolving "STG Labs") — a product/spec decision, not a mechanical fix; (b) 5 in `app/modules/interviews/tests/test_category_rank_regression.py` — FK violation inserting a fixture `interview_levels` row against a `position_id` that was never actually persisted, a separate fixture defect; (c) 6 in `test_positions_flow.py` + 4 in `test_positions_defects_flow.py` — budget/status-transition/history assertions stale against current behavior, e.g. `test_status_limited_to_three_settable_values` expects `open→in_progress` to succeed but the current status state machine returns 409 `INVALID_STATUS_TRANSITION` (needs a spec-informed fix per Gate 3, not a quick patch); (d) 1 in `test_candidates_flow.py::test_resume_download_returns_302_with_location_header` (endpoint now returns 200+JSON with a signed URL, not a 302 redirect). Each needs its own Gate-5 pipeline pass to root-cause + fix; not attempted here beyond (a)-(d)'s classification. **Re-verified against the FULL `pytest -q` run (not just these 7 files), real CI (gh run 31250303340):** `48 failed, 1836 passed, 656 skipped` — these counts are exact and confirmed at the full-suite level, not just the curated 7-file subset.
- 🔴 **`frontend/e2e/a11y.spec.ts::authenticated shell has no serious/critical a11y violations` QUARANTINED (`test.skip`, PR #221 round 2, 2026-08-08 — principal-reviewer M4)** — fails on real data: `color-contrast` (serious), the sidebar's nav-group section labels ("Sourcing"/"Evaluation"/"Pipeline"/"Admin", `text-muted-foreground/60` on white) measure 2.87:1, below WCAG AA's 4.5:1. Genuinely pre-existing (pure CSS, data-independent) — masked until now because this spec could never reach the authenticated shell in CI before this PR's login/MSW fix (docs/BACKLOG.md's `pipeline-progress` chart-ink entry, §3, is the same class of pre-existing contrast gap in a different component). Fix: raise the label's opacity/color in the sidebar-nav component (exact file not yet located — needs its own pass), then un-quarantine.
- 🔴 **`frontend/e2e/positions.spec.ts`'s interviewer read-only test QUARANTINED (`test.skip`, PR #221, 2026-08-08)** — `src/lib/navigation/nav-items.ts`'s `ROLE_NAV_OVERRIDES` restricts `interviewer`'s nav to `["Interviews"]` only, an intentional, documented restriction ("Interviewers only work their interview queue") that predates this PR. The test's own premise (interviewer navigates to Positions via the nav and sees it read-only) no longer holds — never caught before because this spec could never reach real auth against the real backend under MSW's fixture-based mocking. Needs a product decision: (a) rewrite to `page.goto("/positions")` directly (tests route-level access, bypassing nav-discoverability) or (b) drop the test as testing an unsupported scenario. **Coverage note (principal-reviewer, PR #221 round 1):** with this test quarantined AND the write-role test already skipped (Phase 15 fast-follow), `positions.spec.ts` currently has ZERO active e2e coverage — flagging the cliff explicitly rather than letting it go unnoticed.
- 🔴 **Backend `typecheck` (mypy) CI job has 133 pre-existing errors in exactly 3 files, confirmed unrelated to `chore/ci-real-db-e2e-fix` (PR #221, 2026-08-08)** — real CI (gh run 31250303340, verbatim `Found 133 errors in 3 files (checked 288 source files)`): `backend/app/scripts/seed_legal_transaction_demo.py` (68), `backend/app/scripts/seed_uat_recruitment_funnel.py` (64), `backend/alembic/versions/0054_pipeline_retry_count.py` (1) — 2 seed scripts + 1 migration, none touched by this PR (already red identically across main's last 3 backend-CI runs). `tests/unit/test_seed_dev.py`'s own pre-existing mypy debt (an `attr-defined` on `seed_dev.audit` + stale `# type: ignore` comments, confirmed via `mypy --shadow-file` against `origin/main`'s unmodified content) is real locally but sits OUTSIDE CI's mypy target set (`tests/` is excluded from the CI job's scope) — a separate observation, not folded into the 133/3-file count above. This is the same class of pre-existing debt as `component-test`'s MSW-unmatched-handler errors on unrelated frontend component tests (also confirmed pre-existing on main, zero `frontend/src` files touched by this PR) — both block backend-ci.yml/frontend-ci.yml from ever going fully green on ANY PR until fixed, independent of what that PR actually changes. Needs its own scoped pass (likely a `dev/mypy-cleanup` or similar) — not chased here.

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
