# Token-Optimized AI-Assisted Development — Playbook

**Living document — updated whenever a new technique is identified or refined.**
Version history in the [Changelog](#changelog) at the bottom.

> **How to use this:** Read Part 1 before writing the first line of code.
> Follow Part 2 throughout development. Check Part 4 when costs spike unexpectedly.
> Update Part 3 benchmarks and the Changelog when new data or techniques emerge.

---

## Who this is for

Any team using Claude Code (or a similar AI coding assistant) on a production project.
The practices here were derived from a real enterprise build (ATS Platform, STG Labs, Bengaluru)
and are generic enough to apply to any stack or domain. The reference implementation examples
are drawn from that project; adapt to your own.

**The payoff:** a disciplined team running these practices consistently sees 50–65% lower
compute cost vs. an undisciplined team on the same build — with no reduction in code quality,
test coverage, or review rigor.

---

## The core insight

AI assistant compute costs scale with output tokens (roughly 5× the cost of input tokens).
Undisciplined AI development produces output tokens in two ways:

1. **Wrong agent for the task** — a broad agent reads more context and produces more prose than a specific one.
2. **Rework cycles** — a preventable `CHANGES-REQUESTED` from a reviewer costs as much as the original build.

Discipline eliminates both. The practices below target these two root causes, in that order.

---

## Part 1: Project setup (before the first line of code)

Do this once when a project starts. Every item here affects every session from that point on.

### 1.1 Working agreement file (CLAUDE.md or equivalent)

Create `.claude/CLAUDE.md` at the project root. This file is read by Claude Code before every task.
It is the binding contract for the project — what every agent must follow.

**Minimum sections to include:**
- Project stack and purpose (2–3 lines)
- Architecture rules (what each layer owns — no mixing)
- Code quality non-negotiables (type hints, docstrings, test coverage, no N+1, soft-delete)
- Engineering mandate: write minimum code that satisfies acceptance criteria; no speculative features
- Agent routing rules (see Part 2, Discipline 1)
- Spec pre-read mandate (see Part 2, Discipline 2)
- Skills for mechanical work (see Part 2, Discipline 3)
- Cost alert thresholds (see Part 2, Discipline 4)
- Audit scope discipline (see Part 2, Discipline 5)
- Progress capture protocol (see Part 2, Discipline 6)
- Git rules: branch per feature, PR for every change, never push direct to main

**What NOT to put in CLAUDE.md:** narrative history, session notes, task status. Those belong
in the memory file (§1.6) and commit messages.

### 1.2 Specialized agent definitions

Create `.claude/agents/<role>.md` for each distinct domain. Each file has YAML frontmatter
and a body that defines the agent's behavior.

**Standard agents for a full-stack project:**
| File | Role |
|---|---|
| `backend-engineer.md` | API / service / DB layer implementation |
| `ux-ui-engineer.md` | Frontend / UI implementation |
| `unit-test-engineer.md` | Isolated unit tests (mocked dependencies) |
| `integration-test-engineer.md` | End-to-end and functional tests (real dependencies) |
| `principal-reviewer.md` | Senior review gate — correctness, security, architecture (opus/high) |
| `principal-performance-auditor.md` | On-demand: query plans, indexing, load testing (opus/xhigh) |
| `principal-reliability-engineer.md` | On-demand: failure modes, retries, DR, runbooks (opus/xhigh) |

**Critical YAML gotcha (will silently break agent registration):**
Any colon followed by a space inside an unquoted YAML value is parsed as a key-value separator.

```yaml
# BROKEN — "any UI: screens" causes YAML parse failure; agent never registers
description: Use for any UI: screens, forms, tables.

# CORRECT — quote the description
description: "Use for any UI: screens, forms, tables."
```

Verify every agent registers by checking the available agent types list after a fresh session start.
If a file exists but the agent doesn't appear in the list, the YAML frontmatter is the first suspect.

### 1.3 Skills for mechanical work

Create `.claude/skills/<name>/SKILL.md` for repeatable, structured tasks that should never
be done free-form in the main conversation loop.

**Minimum skill set:**
| Skill | Purpose |
|---|---|
| `commit-message` | Conventional commit + provenance trailers |
| `summarize-changes` | PR descriptions, change summaries |
| `changelog` | Release notes |
| `fix-bug` | Structured diagnosis → hypothesis → fix |
| `implement-feature` | Task decomposition before agent spawn |
| `review-pr` | Structured diff review |
| `architecture-review` | Deep architectural analysis |
| `security-review` | Security audit of a change |
| `database-migration` | Schema change review and migration guidance |

**Model routing note:** skills can declare `model: haiku` in frontmatter, but if
`disable-model-invocation: true` is set, the skill runs on the current model (not Haiku).
In that case, savings come from structured, compact output discipline — not model-tier switching.
Don't assume model routing works without verifying the flag.

### 1.4 Cavecrew agents (lookup + bounded edit + audit)

Install the cavecrew plugin to get three compressed-output agents:

| Agent | Use case | Saving vs. alternative |
|---|---|---|
| `cavecrew-investigator` | "Where is X defined / what calls Y / map dir" | ~60% fewer tokens vs. Explore or general-purpose |
| `cavecrew-builder` | Bounded 1–2 file edits (refuses broader scope) | Prevents scope creep on mechanical edits |
| `cavecrew-reviewer` | Diff / PR audits (severity-tagged, no filler) | Faster, denser review output |

Use `cavecrew-investigator` as the first tool for any lookup question. Never use Explore
or general-purpose as the default for "find where X is."

### 1.5 Prose compression

AI assistant responses contain significant filler: pleasantries, hedging, restatements.
These consume output tokens (expensive) without adding information.

**Options, in order of effectiveness:**
1. **Caveman plugin** (Claude Code) — enforces terse prose automatically, persists across sessions.
   Install once; configure `full` mode during build, `ultra` on pure discussion turns.
2. **CLAUDE.md rule** — add a line: "Drop filler (articles, pleasantries, hedging). Fragments
   acceptable. Technical substance unchanged." Lower enforcement than a plugin but zero setup cost.

Compression achieved: ~15–25% reduction in all prose output. Code, commands, structured tables:
never compressed.

### 1.6 Memory and progress capture

Replace expensive documentation protocols with a single lightweight file:

**`memory/resume-pointer.md`** — the "you are here" file. Updated after each PR merges to main.
Contains: what's live (2–3 bullet points), Alembic/schema head, how to run locally, what's next.

**Delivery tracker** (e.g. `docs/GO_LIVE_CHECKLIST.md`) — flip the relevant row inline in the
same PR as the code. Never in a separate docs PR.

**What to drop:**
- Separate phase-log documents per milestone (expensive docs PR each time)
- Narrative "journey doc" updates per feature (high cost, low restore-point value)
- Standalone "session summaries" (git log + PR descriptions are the record)

**Context restore after a session gap:** read resume-pointer → read delivery tracker →
read the relevant spec. Three files, < 2 minutes. No transcript replay needed.

### 1.7 Pre-code setup checklist

Before writing the first line of project code, verify:

```
[ ] CLAUDE.md created with all mandatory sections (§1.1)
[ ] Specialized agent .md files in .claude/agents/ (§1.2)
    [ ] YAML description values quoted if they contain ": "
    [ ] All agents appear in session's available agent types list after fresh start
[ ] Skills installed in .claude/skills/ — at minimum: commit-message, summarize-changes,
    changelog, fix-bug, implement-feature, review-pr (§1.3)
[ ] Cavecrew agents available: investigator, builder, reviewer (§1.4)
[ ] Prose compression configured (plugin or CLAUDE.md rule) (§1.5)
[ ] memory/resume-pointer.md created (even if empty — "nothing built yet") (§1.6)
[ ] Delivery tracker created with status column (§1.6)
[ ] Team briefed: cost alert protocol, agent routing rules, spec pre-read discipline
```

A project that clears this checklist before the first feature build is configured for
cost-optimized development from day one.

---

## Part 2: During development — the disciplines

These rules apply to every task, every session, for the life of the project.

### D1. Agent routing — use the most specific agent, always

**The rule:** before spawning an agent, ask "is there a more specific agent for this task?"
If yes, use that one. Never use `general-purpose` as a fallback when a specialized agent covers the work.

**Routing table:**
| Task type | Agent |
|---|---|
| Backend API / service / DB | `backend-engineer` |
| Frontend / UI | `ux-ui-engineer` |
| Unit tests (mocked) | `unit-test-engineer` |
| Integration / E2E tests | `integration-test-engineer` |
| Senior review gate | `principal-reviewer` |
| "Where is X / what calls Y / map dir" | `cavecrew-investigator` FIRST |
| 1–2 file bounded edit | `cavecrew-builder` |
| Diff / PR audit | `cavecrew-reviewer` |
| Perf deep-dive | `principal-performance-auditor` |
| Failure modes / DR | `principal-reliability-engineer` |

**Never:** route UI work to `general-purpose`. It loads UX guardrails inline, produces
broader context reads, and is less consistent than a dedicated UI agent.

### D2. Spec pre-read before implementation agents (prevents ~$10–15 CHANGES-REQUESTED cycle)

The most expensive event in AI-assisted development is a preventable CHANGES-REQUESTED cycle.
Every cycle costs as much as the original build. Three sections cause the majority of failures:

| Spec section | What to check | Common failure mode |
|---|---|---|
| §audit / §audit-trail | Exact audit action strings | Agent writes `"create"` instead of spec's `"entity_created"` |
| §permissions / §authorization | Exact roles per endpoint | Agent uses wrong role set |
| §errors / §error-codes | Exact codes + HTTP status | Agent returns 422 where spec says 409 + specific code |

**The rule:** before spawning any implementation agent, the main loop explicitly reads these three
sections of the spec. This takes 2 minutes. It prevents a full review cycle.

Add this to `backend-engineer.md` agent definition as a mandatory first step.

### D3. Skills for mechanical work — invoke via Skill tool, never inline

Invoke structured skills instead of asking the main loop to produce these free-form:

| Task | Invoke | Never do |
|---|---|---|
| Git commit | `/commit-message` | Write commits inline |
| PR description | `/summarize-changes` | Summarize changes inline |
| Release notes | `/changelog` | Write changelog inline |
| Bug analysis | `/fix-bug` | Debug free-form in main loop |

Free-form generation in the main loop produces verbose, inconsistent output and buries
project conventions (commit format, provenance trailers) in the conversation history
where they erode over time.

### D4. Cost alerts — mandatory BEFORE and AFTER every task, no size carve-out (binding, updated 2026-07-24)

**Superseded 2026-07-24:** the "Simple (1–2 files): no alert" carve-out below is gone. Every task
now gets a before-estimate AND an after-actual, scaled to size — see CLAUDE.md's "Cost alerts —
BEFORE and AFTER every task" section (enforced under the no-override Model tier & CI independence
mandate, not the softer 3-request rule). This entry exists so this playbook doesn't drift from
that binding rule the way it briefly did on 2026-07-24 itself (a review caught this exact section
still stating the old carve-out one commit after CLAUDE.md superseded it).

**Thresholds (scale the format, never skip the substance):**
| Task type | Before | After |
|---|---|---|
| Simple (docs, 1–2 files) | One-line estimate ("Est. <$1, 1 file") | One-line actual |
| Medium (1 agent, scoped feature) | Alert if estimated >$10 | State actual, flag >2x overrun |
| Full module build (backend + tests + UI + review) | Alert at start; check-in at ~$25 milestone | Full actual breakdown |
| Broad sweep (audit, platform review) | Define scope + emit estimate; wait for confirmation | Full actual breakdown |

**Alert format:**
```
[COST ALERT] Est. ~$X. Scope: [Y agents, Z files / modules / dimensions]. Proceed?
```

Never start a broad task and discover the cost after. Estimate before; confirm scope. Never finish
any task — including the smallest one — without stating what it actually cost.

### D5. Audit scope discipline — define before running

Broad "audit the codebase" requests are the highest-cost category of work.
Platform-wide audits with no scope constraint routinely cost $30–40 and produce
findings too diffuse to prioritize.

**Before any audit task, specify:**
- Which modules or subsystems (not "everything")
- Which quality dimensions: security? schema evolution? layering? test coverage? performance?
- Expected output: finding list? severity-tagged? summary only?

A narrowly scoped audit costs 50–70% less and produces more actionable output.
Default to a targeted slice. Only go broad when explicitly requested — and then alert first.

### D6. Progress capture — lightweight, per-merge

After each PR merges to main, and only after:
1. Update `resume-pointer.md` with what landed (2–3 lines).
2. Flip the relevant delivery tracker row inline — this should have been in the same PR as the code.

That's the complete protocol. No separate docs PRs. No phase logs. No narrative updates.

Run `/compact` when context grows large. The resume-pointer is the restore point.

### D7. Prose compression — enforce in every session

Caveman mode (or equivalent) should be active in every build session. If it resets, re-enable
before proceeding. Verbose responses waste output tokens on every interaction.

**Verify compression is active** at the start of each session. A brief response to a simple
question is the positive signal. A paragraph where a sentence would do is the negative signal.

### D8. Pin subagent model tier explicitly — never let it silently inherit (binding)

The Agent tool defaults to inheriting the main-loop model when no `model:` override is given.
If the main loop's model tier changes (e.g. an orchestrator upgrade), every subagent that
silently inherits pays whatever that tier costs — including cavecrew-style lookup/grep work
that never needed the upgraded tier's reasoning depth.

**Rule:** every `.claude/agents/*.md` subagent definition pins `model: sonnet` explicitly in
its frontmatter (not omitted). This gets the current orchestrator tier today while decoupling
each named agent from future orchestrator-tier changes — a later switch to a cheaper or more
expensive default model does not silently drag these 8 specialist agents along with it.
Cavecrew-tier agents (investigator/builder/reviewer) stay on their own cheaper routing (D1) —
this rule does not push them onto the pinned tier.

### D9. Verify model availability once per session before spawning overridden agents (binding)

An agent spawned with an unavailable/misconfigured `model` override fails mid-run after
already consuming tokens on setup and partial work — the failure is not free. One documented
incident: a principal-reviewer run died ~110K tokens in on a bad model id, then had to be
rerun from scratch on the correct model for another ~110K tokens (~$15–20 wasted, 100%
avoidable).

**Rule:** before the first `Agent`/`Workflow` call in a session that passes an explicit
`model` override, confirm that model resolves (e.g. one `/model` check or one small
low-cost agent call). Do this once per session, not once per call.

### D10. Front-load ambiguity resolution before AskUserQuestion (binding)

Destructive or broad-scope actions (bulk deletes, dataset rebuilds, schema-affecting cleanup)
routinely surface new edge cases only after each AskUserQuestion round — orphaned FK references,
duplicate names, linked entities — forcing another round of investigation + another question.
One documented incident: a dev-data cleanup task took 5 separate SQL investigation round-trips
interleaved with 3 AskUserQuestion rounds because each answer revealed a complication that a
single upfront query would have surfaced.

**Rule:** before asking the user to resolve scope on a destructive/ambiguous action, run ALL
foreseeable edge-case checks in one batched investigation pass first (linkage, duplicates,
orphans, counts) — not one check per question round. Present the full picture once; ask once.

### D11. Reasoning-effort discipline — a lever separate from model choice (binding)

Model tier and reasoning effort are independent knobs. Pinning a capable model (D8) does not
mean every tool call under it should run at that model's highest effort setting — mechanical
work (counts, greps, file reads, simple lookups) gets no quality benefit from high effort and
pays its full cost anyway.

**Rule:**
- Default the main loop's own mechanical tool calls (grep/glob/read/count-style investigation)
  to low effort. Reserve high/max effort for design decisions, synthesis, and adversarial review.
- **Max effort is reserved for cases where it is absolutely required to protect quality,
  correctness, or a hard review gate — not a default.** Using max by default measurably slows
  throughput without a proportional quality gain on routine work.
- Each named subagent (`.claude/agents/*.md`) declares an explicit `effort:` default matched to
  its actual judgment load — not uniformly high, not uniformly low:

| Agent | Model | Effort | Why |
|---|---|---|---|
| backend-engineer | sonnet | high | Layering, business-logic, and spec-compliance judgment calls |
| unit-test-engineer | sonnet | medium | Pattern-following against known code, but must reason about semantics |
| functional-test-engineer | sonnet | medium | Mostly running/reading real output; diagnosis needs some depth |
| integration-test-engineer | sonnet | medium | Broad but largely mechanical coverage once scope is set |
| principal-reviewer | **opus** | high | Single merge-readiness gate — correctness/security/mandate judgment |
| principal-reliability-engineer | **opus** | xhigh | Failure-mode and SRE analysis is genuinely hard reasoning |
| principal-performance-auditor | **opus** | xhigh | Query-plan/profiling analysis requires deep reasoning |
| ux-ui-engineer | sonnet | high | Accessibility/UX guardrail tradeoffs need real judgment |

None of the 8 named agents default to `max` — max stays a situational override, invoked
explicitly when a specific run demands it (e.g. a security-sensitive review), never baked in
as a standing default.

### D12. Cost-alert dollar thresholds are fixed, not tier-relative (binding)

Cost-alert thresholds (D4) are defined in dollars, not tokens or model-tier multiples.
**Rule:** do not recalibrate the $10 / $25 alert thresholds when the underlying model tier
changes. The thresholds represent a fixed spend tolerance the team has already agreed to —
they hold regardless of which model tier is doing the work.

### D13. Two-tier model roster — the 3 chokepoint/specialist roles run Opus (binding, added 2026-07-24)

**Why this exists:** a 2026-07-23/24 status review found principal-reviewer needed 3 rounds
(Sonnet/high) to close a single change — round 1 missed a 22nd endpoint the original audit never
considered, round 2 missed a spec-sync update the fix itself needed. Both were genuine
reasoning-depth misses, not execution-visibility gaps (contrast with the same review's CI-ordering
finding in D-adjacent CI docs, which was structural, not a model-capability question). The fix:
tier up the model at the one chokepoint every change passes through, and at the two genuinely
on-demand deep-dive specialists — not the whole roster, which stays Sonnet (see D8-D12 above;
building-to-spec and test-writing are bounded tasks the existing pipeline already backstops
structurally).

**Cost delta, worked (per-1M pricing, 2026-07-24 — Sonnet 5 is in its intro-pricing window through
2026-08-31, so the multiplier shrinks after that date even though nothing else changes):**

| | Sonnet 5 (intro, today) | Sonnet 5 (standard, after 2026-08-31) | Opus 4.8 |
|---|---|---|---|
| Input / output per 1M | $2.00 / $10.00 | $3.00 / $15.00 | $5.00 / $25.00 |
| Multiplier vs Sonnet | — | — | **2.5x today, 1.67x after Aug 31** |
| Per-invocation cost (~70K tokens, review-shaped 80/20 in/out mix) | ~$0.25 | ~$0.38 | ~$0.63 |

At a realistic 20–30 `principal-reviewer` invocations/month, incremental cost is **~$5–11/month**.
`principal-reliability-engineer`/`principal-performance-auditor` are on-demand (2–5 calls/month),
so their incremental cost is smaller still, likely <$3/month combined. Total incremental spend for
this entire mandate: **roughly $10–15/month** — trivial against the cost of a single production
defect at 800–1200-hires/year scale that either role exists to catch. This is a governance
decision, not a budget one; don't re-litigate the dollar amount, it's a solved question.

**Binding, no override — see CLAUDE.md's "Model tier & CI independence mandate."** This is not a
per-task judgment call the main loop makes; the 3 agent files' `model:` frontmatter is fixed until
CLAUDE.md itself is edited to say otherwise.

### D14. Trace data-matching bugs to their authoritative source upfront — never fix incrementally (binding, mandatory)

A data-matching, mapping, or derivation bug (frontend value matched against a backend-computed
string/enum, one field standing in for another, a lookup keyed on the wrong identifier) has a
"chain of custody" from where the value is authoritatively produced to where it's consumed.
Stopping the pre-fix investigation at the first plausible-looking field in that chain — instead
of tracing it all the way to the code that actually PRODUCES the value — turns one review round
into several, because each round only has visibility into the layer the previous fix touched.

**One documented incident (PR #159, Issue 1 — status label upgrade):** the fix needed to match
an application's `pending_reason` against the correct interview record. Three fix attempts were
needed, each caught by a *separate* principal-reviewer round (~$15–20 apiece):
1. Match by free-text `level_label` — wrong; org levels use org-specific display names
   (e.g. "Infosys L1"), not a literal "Org L1".
2. Match by `level_number` — wrong; it's a fixed, admin-configured slot that diverges from the
   value actually embedded in `pending_reason` once an earlier same-category level is deleted.
3. Match by `category_rank` — correct; it's the exact value the backend uses when it *constructs*
   `pending_reason` strings in the first place.
Each of the first two fixes looked reasonable in isolation. The investigation before attempt #1
stopped at "what field looks like it should work" instead of reading the backend code that
actually builds `pending_reason` — which would have surfaced `category_rank` as the only correct
match key on the first pass.

**Rule (mandatory, not a suggestion):** before dispatching a fix for any data-matching/derivation
bug, the pre-fix investigation (`cavecrew-investigator` or a main-loop read) MUST trace the value
back to the code that authoritatively produces it — not stop at the schema field or type that
merely carries it. If a frontend value will be matched against a backend-computed string/enum,
read the backend function that constructs that string BEFORE writing the match logic, not after
a reviewer flags a mismatch. This applies to `cavecrew-investigator`'s own investigation prompts
too: an investigation that reports "field X looks like the right match" without confirming X is
literally the same value used to construct the thing being matched against has not done its job.

### D15. Token-optimization practice showcase — name the practice, name the saving (binding, added 2026-07-24)

**Rule (mandatory, no override — see CLAUDE.md's Model tier & CI independence mandate):** after
completing any task, name which specific practice from this playbook (or which CLAUDE.md
agent-routing/gate rule) was applied, and state concretely how it reduced *this task's* actual
spend — not a generic restatement of the practice ("used cavecrew-investigator, it's cheaper"),
a specific claim about the run just completed. If no practice meaningfully applied (a trivial
task), say so explicitly — an omitted showcase section is not compliant, an explicit "N/A" is.

**Why this is separate from D1-D14 above:** those are practices to *apply*; this is the
requirement to *report* having applied them, so the discipline stays visible and auditable per
task rather than something only checkable by re-reading the whole session after the fact.

**Worked example (from the session that introduced this rule):** dispatching
`cavecrew-investigator` to find every repo-wide reference to `performance-auditor`/
`reliability-engineer` before renaming them (instead of grepping in the main loop) kept an
estimated ~15-20K tokens of raw multi-file grep output out of main-loop context — the agent
returned a compressed, structured file:line list instead. Using `principal-reviewer` itself
(post-upgrade) to review its own model-tier change, rather than a second ad-hoc main-loop
self-check, meant the 4 real findings it caught (a duplicate D-number, a contradicted rule, a
missing playbook entry, a sequential-step regression) were caught by the SAME mechanism this
mandate exists to strengthen — not a parallel, redundant verification pass.

---

## Part 3: Cost benchmarks (living — update with each project's data)

Label each row with the project and date so benchmarks stay comparable over time.

| Activity | Without discipline | With discipline | Saving | Source |
|---|---|---|---|---|
| Full module build (backend + tests + UI + review) | ~$80–100 | ~$30–45 | ~55–60% | ATS Platform, Jun 2026 |
| Broad platform audit (no scope defined) | ~$30–40 | ~$12–18 (scoped) | ~55–65% | ATS Platform, Jun 2026 |
| Phase docs rollup in separate PR | ~$5–7 | ~$0.15 (resume-pointer) | ~97% | ATS Platform, Jun 2026 |
| Wrong-agent fallback (general-purpose for UI) | ~$20–25 | ~$12–15 (ux-ui-engineer) | ~35–40% | ATS Platform, Jun 2026 |
| Preventable CHANGES-REQUESTED cycle | ~$10–15 | ~$0 (spec pre-read) | ~100% | ATS Platform, Jun 2026 |
| Commit/summary inline vs. skill | ~$1–2/commit | ~$0.20 (skill) | ~80–90% | ATS Platform, Jun 2026 |

**Note:** these are mechanism-based estimates. Authoritative numbers from the Anthropic console
→ Usage (per-model token split, filtered by date range). Update this table with real console data
when available — label it as "measured" vs. "estimated."

---

## Part 4: Anti-patterns (living — add new ones as they're discovered)

| Anti-pattern | Cost impact | Fix |
|---|---|---|
| Using `general-purpose` as default agent | High — broad context load | Route to specialized agent |
| Writing commit messages free-form | Medium — verbose, inconsistent | Use `/commit-message` skill |
| Starting a platform audit with no scope | Very high | Define modules + dimensions first (D5) |
| Separate docs PR after every milestone | Medium (~$5–7 each) | Inline checklist flip + resume-pointer (D6) |
| Skipping spec pre-read before implementation | Very high — CHANGES-REQUESTED cycle | Mandatory §audit/§permissions/§errors read (D2) |
| YAML frontmatter with unquoted colon in description | Agent registration silently breaks | Quote description values containing `: ` |
| Verbose AI responses with filler prose | Medium (~15–25% overhead) | Prose compression (D7) |
| Invoking Explore/general-purpose for lookups | Medium | Use cavecrew-investigator first (D1) |
| Assuming `model: haiku` in skill = Haiku billing | Misleading cost model | Check `disable-model-invocation` flag; savings may be discipline-only |
| Broad "review everything" before defining scope | Very high | Alert + scope first; narrow default (D5) |
| **Unit tests mock the per-item function in bulk ops** | **Very high — hides session-state bugs; each CHANGES-REQUESTED cycle costs as much as the original build (~$10–15)** | **Failure-cascade test mandatory: item N fails → verify item N+1 still runs (unit-test-engineer §failure-cascade)** |
| **No functional test before commit on new endpoint** | **High — session corruption, constraint cascades, phantom enqueue only visible against real DB** | **functional-test-engineer is hard gate; run before integration-test-engineer on every new endpoint** |
| **Celery enqueue inside savepoint (begin_nested)** | **High — RELEASE failure leaves orphan Celery task; review blocks merge** | **Enqueue AFTER savepoint context exits cleanly; use `_enqueue=False` in inner call** |
| **Bug fixes done entirely in the main loop during live testing** | **High — no cavecrew-investigator/builder, no principal-reviewer gate; cost escalates with zero auditability** | **CLAUDE.md Gate 5 (binding, no override): every bug fix routes through cavecrew-investigator → cavecrew-builder/ux-ui-engineer/backend-engineer → principal-reviewer, regardless of size** |
| **Data-matching fix based on "field looks right" instead of tracing to the authoritative source** | **Very high — each wrong layer costs a full principal-reviewer round (~$15–20); PR #159 needed 3 rounds for one bug** | **D14 (binding, mandatory): trace the value to the code that produces it before writing match logic, not after a reviewer flags the mismatch** |
| **New column backfills to NULL with no check whether existing rows had the value via another source** | **High — shipped feature appears broken on real user-tested data post-merge, forcing an emergency root-cause + backfill-script cycle (~$15–20) discovered by the user, not caught pre-merge** | **CLAUDE.md schema-evolution item 5 (binding): backfill from the authoritative source in the same PR, or explicitly justify why none is possible — "no rows affected" is not sufficient on its own** |

---

## Part 5: How this playbook evolves

This document is a living artifact, not a snapshot. It must stay current as the project
progresses and new techniques are discovered or invalidated.

**When to update:**
- A new optimization technique is identified during development → add to Part 2 or Part 4
- A benchmark is measured (real console data) → update Part 3 with "measured" label
- An anti-pattern is discovered through a cost spike → add to Part 4
- A technique is found to be less effective than stated → correct or remove it
- A new tool, agent type, or skill becomes available → add to Part 1 setup checklist

**How to update:**
Update this file in the **same PR as the code change that triggered the finding**.
Add a dated entry to the Changelog below. Never let the playbook drift from actual practice —
a stale playbook is worse than no playbook (teams follow it and get wrong results).

**Binding rule (in CLAUDE.md):** when a new technique is identified or refined during
development, update `docs/TOKEN_OPTIMIZATION_PRACTICE.md` in the same PR with a dated
Changelog entry. The playbook ships with the code, not later.

**Ownership:** the team that runs the project owns this file. It is not owned by a
separate tools/platform team — the people incurring the costs are best positioned to
keep it accurate.

---

## Changelog

| Date | Version | Change | Trigger |
|---|---|---|---|
| 2026-06-18 | 1.0 | Initial playbook — 7 disciplines, setup checklist, benchmarks, anti-patterns | ATS Platform Phase 16 post-build cost analysis (~$100 session) |
| 2026-06-18 | 1.1 | Restructured as living pre-code guide; added Part 5 evolution protocol; corrected Haiku routing assumption (`disable-model-invocation: true` = inline Sonnet, not Haiku billing); added YAML frontmatter gotcha to §1.2; expanded Part 3 with source labels | Binding mandate + org-sharing requirement |
| 2026-06-19 | 1.2 | Added 3 new anti-patterns from PR #52 bulk upload bug post-mortem: (1) mocking per-item function in bulk op tests hides session-state bugs; (2) no functional test pre-commit lets constraint/session bugs reach review; (3) Celery enqueue inside savepoint creates phantom tasks. Each caused a CHANGES-REQUESTED cycle (~$10–15 each = ~$20–30 preventable spend). Mitigations: failure-cascade test rule + functional-test-engineer hard gate (PR #54). | PR #52 post-mortem |
| 2026-07-08 | 1.3 | Added D8–D12 for the Sonnet 5 orchestrator switch: (D8) pin `model:` explicitly on all named subagents rather than inherit; (D9) verify model availability once per session before overriding (avoided a repeat ~$15–20 failed-run waste); (D10) batch edge-case investigation before AskUserQuestion rounds on destructive/ambiguous actions; (D11) reasoning-effort table per named agent, max effort reserved for genuinely-required cases only, never a default; (D12) cost-alert dollar thresholds stay fixed across model-tier changes. All 5 are binding. | Orchestrator switch to Sonnet 5 |
| 2026-07-08 | 1.4 | Added CLAUDE.md Gate 5 (Regression prevention gates): 3 consecutive bug fixes during a live UI-testing session ran entirely in the main loop — zero cavecrew-investigator/builder dispatch, zero principal-reviewer pass before merge. Unlike Gates 1–4, Gate 5 has NO 3-request override; it can only be changed by editing CLAUDE.md directly. Added matching anti-pattern row to Part 4. | Cost-escalation review during 2026-07-08 live testing session |
| 2026-07-08 | 1.5 | Added D13 (binding, mandatory): trace data-matching/derivation bugs to the code that authoritatively produces the value before writing match logic — don't fix on "field looks right" and let review find the next layer. PR #159's Issue 1 needed 3 principal-reviewer rounds (`level_label` → `level_number` → `category_rank`) for one bug because the pre-fix investigation stopped at the first plausible field instead of reading the code that constructs `pending_reason`. Added matching anti-pattern row to Part 4. | PR #159 5-round review cycle post-mortem |
| 2026-07-10 | 1.6 | Added CLAUDE.md schema-evolution item 5 (binding backfill mandate): a new column that starts persisting a value existing rows implicitly already had via another authoritative source must backfill from it in the same PR, or explicitly justify why not — "no rows affected" alone is insufficient. Migration `0047_feedback_outcome_col` shipped with that exact insufficient justification; a real pre-existing row's decided outcome was already in `application_status_history` but never backfilled, surfacing as a shipped feature (BR-SYNC-006) looking broken on user-tested data after merge — found by the user, not caught in review. Fixed via `backfill_legacy_feedback_outcome.py`. Added matching anti-pattern row to Part 4. | Live BR-SYNC-006 incident, user-reported post-merge |
| 2026-08-19 | 1.7 | Added CLAUDE.md's "CI-cost-under-real-billing mandate" (binding, no override): local-verification-first before any code/config/test/CI-workflow push (full `ruff`/`mypy`/`pytest`+`RUN_DB_TESTS=1`/`tsc`/`eslint`/`vitest`, plus a local Playwright run for new e2e coverage); review the diff before the push it belongs to, not after a CI failure surfaces what review would have caught; consolidate to the minimum CI-triggering pushes a change needs; state real dollar cost (not just minutes) in cost alerts once over the included allotment. CR#1's build session pushed 9 times to one branch (2 rounds discovering CI-only-visible defects a local Windows run structurally couldn't catch — a Windows-only venv path, a missing Celery worker in `frontend-ci.yml` — plus 3 review-driven fix rounds each re-triggering full CI) and took the account from ~1,381 to 1,991 of the 2,000-minute August allotment with no spending limit configured, meaning further usage is now real per-minute billing, not just quota risk. | CR#1 (`candidate-ai-match-screen-consolidation`) build session CI-usage review, 2026-08-19, ahead of CR#2 |
