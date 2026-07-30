---
paths:
  - "frontend/**"
  - "**/*.tsx"
  - "**/*.ts"
---

# ATS UX/UI Guardrails

**Project:** Enterprise Applicant Tracking System (ATS) Web Application  
**Purpose:** Mandatory UX/UI decision guardrails for AI agents, designers, and engineers before creating mockups, prototypes, or production UI.  
**Primary consumers:** Claude Code agents, OpenSpec workflow agents, product designers, frontend engineers, QA, accessibility reviewers, security/compliance reviewers.  
**Last updated:** 2026-06-01

---

## 1. Mandatory pre-work before any UX/UI output

Before creating a screen, wireframe, component, interaction, page, route, or production UI, the agent MUST complete this sequence:

1. Read this file completely.
2. Read the current OpenSpec project context and relevant artifacts:
   - `openspec/project.md`, if present.
   - Relevant current specs under `openspec/specs/**/spec.md`.
   - Relevant active change files under `openspec/changes/<change-id>/`, especially `proposal.md`, `design.md`, `tasks.md`, and delta specs.
3. Use Unblocked context when configured:
   - Query for related PRs, design discussions, tickets, production incidents, domain decisions, analytics notes, and support feedback.
   - Prefer current repository and team context over generic assumptions.
   - Summarize what was found, and cite or link the internal sources in the OpenSpec proposal/design when available.
4. Identify the affected ATS roles, data sensitivity, workflow stage, and success metric before designing.
5. Confirm the UI supports all required states: default, loading, empty, no results, permission denied, validation error, integration failure, partial sync, draft, submitted, success, destructive confirmation, and audit/history state.
6. Update or create OpenSpec artifacts before implementation if the UX change changes behavior, permissions, workflow, compliance posture, data exposure, automation, or integration contracts.

If any of the above cannot be completed, the agent MUST state the gap before producing UI and MUST choose the safest conservative design.

---

## 2. ATS role coverage baseline

Every UX/UI proposal MUST explicitly say which roles are affected and which roles are intentionally unaffected.

### Candidate-side roles

- External candidate
- Internal candidate / employee applicant
- Referred candidate
- Campus / early-career applicant
- Contractor / contingent worker applicant
- New hire / preboarding user

### Hiring and talent acquisition roles

- Recruiter
- Sourcer
- Recruiting coordinator
- Hiring manager
- Interviewer
- Interview panel member
- Offer approver
- HRBP
- Compensation partner
- Finance approver
- Legal / compliance reviewer
- DEI / equal opportunity reviewer
- Background check / verification operator
- Onboarding / HR operations user

### Enterprise and partner roles

- ATS administrator
- IT / security administrator
- Integration owner
- Reporting / analytics user
- TA leadership
- Executive stakeholder
- Agency recruiter
- RPO partner
- Assessment / background-check vendor user
- Job board / distribution partner

Do not design a generic one-size-fits-all UI if role-specific visibility, responsibility, or workflow differs.

---

## 3. Non-negotiable product principles

1. **Task-first, not module-first.** Users should see what needs action, who owns it, why it matters, and what happens next.
2. **Pipeline clarity.** A candidate, requisition, interview, approval, or offer must always expose current state, owner, blocker, next action, age, and history.
3. **Sensitive data minimization.** Show only the data needed for the current role and task.
4. **Human accountability.** AI, automation, parsing, matching, and ranking must support human review and never hide the reason for a recommendation.
5. **Accessibility by default.** Keyboard access, semantic markup, focus states, screen reader support, contrast, and non-color indicators are required.
6. **Enterprise scale.** Every list, dashboard, workflow, and admin screen must handle high data volume, many business units, many countries, many templates, and complex permissions.
7. **Recoverable workflows.** Long forms, scheduling flows, approvals, candidate communications, and bulk actions need draft states, previews, validation, and safe recovery from failures.
8. **Auditability.** Important user, system, integration, and automation actions must be traceable.
9. **Configurability with guardrails.** Admins should configure workflows without creating inconsistent, inaccessible, or non-compliant experiences.
10. **Candidate respect.** Candidate-facing flows must be transparent, mobile-friendly, accessible, low-friction, and privacy-conscious.

---

## 4. Required UX discovery questions

Before designing or coding, answer these in the OpenSpec proposal or design artifact:

1. What user role is performing the action?
2. What hiring stage, requisition stage, or candidate lifecycle stage is involved?
3. What is the primary task and the user's success condition?
4. What are the secondary tasks and likely edge cases?
5. What data is visible, editable, hidden, masked, or audit-only?
6. What permissions or business-unit boundaries apply?
7. What information should be candidate-visible versus internal-only?
8. What happens if the user lacks access, the record is archived, the job is closed, the link is expired, or the integration fails?
9. What must be logged for compliance, debugging, reporting, or audit?
10. What accessibility, mobile, localization, and performance requirements apply?

---

## 5. Information architecture and navigation guardrails

- Provide role-specific dashboards for candidates, recruiters, hiring managers, interviewers, coordinators, admins, agencies, and executives.
- Avoid burying daily work inside admin-style object menus.
- Global navigation should separate: Jobs/Requisitions, Candidates, Talent Pools/CRM, Interviews, Offers, Tasks, Reports, Admin, and Integrations.
- Provide a universal command/search entry point for expert users.
- Use breadcrumbs or clear context headers for deep pages such as candidate profile, requisition detail, offer detail, and admin configuration.
- Show workspace scope clearly: business unit, region, department, job, hiring team, agency, or personal queue.
- Never make users guess whether they are viewing production data, a draft, a template, an archived record, or a sandbox/test configuration.

---

## 6. Dashboard and task inbox guardrails

Every operational dashboard should answer:

- What needs my attention now?
- What is blocked?
- What is overdue?
- What changed since I last visited?
- Which candidates, requisitions, interviews, offers, or approvals are at risk?

Minimum dashboard capabilities:

- Role-specific widgets and saved views.
- Task inbox with due dates, priority, owner, and escalation state.
- Drill-down from metric to underlying records.
- Clear distinction between actionable alerts and informational notifications.
- Empty states that explain how to get started.
- Permission-aware metrics: do not count or expose inaccessible records.

---

## 7. Pipeline and workflow guardrails

For candidate, requisition, interview, approval, offer, and onboarding workflows:

- Show current stage, previous stage, next stage, owner, stage age, SLA, blockers, and available actions.
- Support configurable stages while preserving consistent UX patterns.
- Display stage transition requirements before users attempt a move.
- Require reason capture for rejection, withdrawal, offer decline, failed checks, and exceptional approvals.
- Make destructive or irreversible actions explicit, previewable, and permission-controlled.
- Support bulk movement only with preview, exclusions, validation, and undo where feasible.
- Preserve complete activity history across stage changes.

---

## 8. Candidate experience guardrails

Candidate-facing experiences MUST be:

- Mobile-first and responsive.
- Accessible without requiring unnecessary account creation.
- Clear about application status without exposing internal evaluation details.
- Transparent about required documents, time expectations, interview format, location, remote/hybrid status, and next steps.
- Capable of save-and-resume for long applications.
- Respectful of candidate privacy, consent, communication preferences, and withdrawal rights.
- Supportive of document upload from local device and common cloud locations where applicable.
- Designed to avoid duplicate entry after resume parsing.

Candidate self-service should include, where policy allows:

- Profile updates.
- Application status.
- Interview schedule and reschedule request.
- Document upload.
- Communication preferences.
- Application withdrawal.
- Offer review and acceptance flow.
- Preboarding task status.

---

## 9. Candidate profile guardrails

A candidate profile should be a single source of truth, with clear separation between internal and candidate-visible content.

Include, when permissioned:

- Identity and contact details.
- Resume/CV and original attachments.
- Parsed profile fields with source indicators.
- Applications and requisitions.
- Current pipeline stages.
- Interview feedback and scorecards.
- Internal notes.
- Candidate communications.
- Consent and privacy status.
- Duplicate warnings.
- Source/referral/agency ownership.
- Assessment/background-check status.
- Offer and onboarding status.
- Activity timeline and audit history.

Do not mix internal notes, scorecards, private deliberation, AI reasoning, or compensation discussion into candidate-visible areas.

---

## 10. Requisition and job setup guardrails

Requisition creation should be guided, not a blank administrative form.

Support:

- Job templates by role family, department, level, region, employment type, and location model.
- Intake questions for hiring manager and recruiter alignment.
- Headcount plan, replacement/new role reason, budget, compensation band, approval path, and target start date.
- Hiring team assignment and responsibility mapping.
- Job description, minimum qualifications, preferred qualifications, legal notices, and application questions.
- Interview plan and scorecard template.
- Preview of internal and external job posting.
- Localization and region-specific compliance fields.

Guardrail: do not let a job be published with missing required compliance, accessibility, location, compensation, approval, or ownership information.

---

## 11. Search, filters, and data table guardrails

High-volume enterprise ATS work depends on strong search and table UX.

Minimum capabilities:

- Global search across candidates, jobs, requisitions, offers, notes where permissioned, and IDs.
- Advanced search with saved views.
- Boolean or structured candidate search where relevant.
- Filters for stage, status, owner, source, location, skills, tags, business unit, agency, date range, aging, and custom fields.
- Sort, group, resize, pin, and show/hide columns.
- Bulk actions with previews and permission checks.
- Inline quick actions for common tasks.
- Clear no-results and filtered-empty states.
- Large dataset handling through pagination, virtualization, or progressive loading.

Search relevance must explain why a record matched when ranking, AI, or semantic search is used.

---

## 12. Forms, validation, and recovery guardrails

Forms must be fast, forgiving, and auditable.

Required patterns:

- Autosave or explicit draft state for long forms.
- Inline validation and summary validation for long forms.
- Clear required and optional fields.
- Smart defaults from templates, job, region, user role, or previous records.
- Conditional fields with visible reasoning.
- Error messages that explain how to fix the problem.
- File upload progress, validation, retry, and failure recovery.
- Change summary before submission for approvals, offers, job publishing, and bulk actions.
- Confirmation for destructive, external-facing, or compliance-sensitive submissions.

Do not make users restart long workflows after timeout, failed upload, or integration error.

---

## 13. Collaboration and communication guardrails

Hiring is collaborative. UI must support clean handoffs.

Include:

- Assigned owner for every task.
- Mentions and comments where appropriate.
- Internal notes separated from candidate communications.
- Candidate email/SMS/message templates with preview.
- Merge-field validation before sending.
- Communication history, delivery status, and reply threading.
- Interviewer reminders and feedback nudges.
- Escalation rules for overdue approvals, feedback, or scheduling.
- Shared decision records and debrief notes.

Never allow bulk messages to send without recipient preview, template preview, and broken-field validation.

---

## 14. Interview scheduling and feedback guardrails

Scheduling UX must handle real enterprise complexity:

- Candidate availability collection.
- Interviewer availability.
- Time zones.
- Rooms, video links, travel/location details, buffers, and panels.
- Rescheduling, cancellations, no-shows, and replacement interviewers.
- Calendar integration status and sync failures.
- Interview kits and structured scorecards.
- Feedback reminders and overdue states.

Feedback UX must:

- Use role-specific scorecards and competencies.
- Ask for evidence, not only numeric ratings.
- Allow private interviewer notes only if policy permits.
- Prevent accidental visibility to candidates.
- Support debrief comparison and decision rationale.
- Consider hiding others' feedback until the interviewer submits their own, if product policy supports independent feedback.

---

## 15. Offer, approval, and onboarding guardrails

Offer UX must support:

- Compensation band visibility based on permissions.
- Salary, bonus, equity, benefits, start date, location, employment type, and contingencies.
- Exception justification when outside policy.
- Approval chain visibility.
- Version control and change comparison.
- Offer letter generation and e-signature.
- Negotiation history.
- Candidate-visible offer package.
- Final acceptance/decline state.

Onboarding and preboarding handoff must:

- Avoid asking candidates/new hires to re-enter data already collected.
- Show pending HR, IT, payroll, background-check, document, and hiring-manager tasks.
- Distinguish ATS-owned tasks from downstream HRIS/onboarding tasks.
- Show integration status and failure recovery.

---

## 16. Admin and configuration guardrails

Admin UX must enable configuration without chaos.

Support:

- Configurable pipelines, stages, approval chains, custom fields, forms, templates, scorecards, rejection reasons, source lists, brands, notifications, and automation rules.
- Preview before publishing configuration changes.
- Sandbox/test mode for configuration changes.
- Version history and rollback for critical configurations.
- Warnings for changes that affect active requisitions, active candidates, templates, or reports.
- Dependency visibility: where a field, template, stage, or rule is used.
- Permission-aware admin screens.
- Audit trail for configuration changes.

Do not create admin screens that require database knowledge or developer intervention for routine changes.

---

## 17. Permissions, privacy, security, and compliance guardrails

The ATS handles sensitive personal, employment, compensation, and evaluation data.

Every UI change must account for:

- Role-based access control.
- Business-unit, region, job, agency, ownership, and stage-based visibility.
- PII minimization and masking.
- Candidate consent and privacy notice states.
- Retention, anonymization, deletion, legal hold, and export workflows.
- Audit logs for sensitive reads/writes, exports, permission changes, automation actions, and data syncs.
- Clear distinction between internal-only and candidate-visible information.
- Region-specific compliance text and required fields.
- Secure handling of attachments and generated documents.
- SSO/MFA and session timeout UX where relevant.

Guardrail: never expose demographic, compensation, assessment, background-check, disability, immigration, or internal evaluation data unless the role, region, and workflow explicitly require it.

---

## 18. AI, automation, and resume parsing guardrails

Any AI-assisted ATS feature must be transparent, reviewable, and overrideable.

Applies to:

- Resume parsing.
- Candidate matching.
- Candidate ranking.
- AI summaries.
- Screening recommendations.
- Knockout question automation.
- Interview question generation.
- Offer or compensation recommendations.
- Communication drafting.

Required UX:

- Label AI-generated or system-inferred content.
- Show source evidence where possible.
- Allow human edit, override, and correction.
- Avoid presenting AI output as final hiring judgment.
- Log automated decisions and human overrides.
- Explain limitations and confidence where relevant.
- Provide bias, compliance, and adverse-impact review hooks for selection-related automation.

Resume parsing must show original resume beside parsed fields and distinguish extracted, user-entered, recruiter-edited, and system-inferred data.

---

## 19. Accessibility guardrails

All ATS UI must target WCAG 2.2 AA or the project's stricter standard.

Required:

- Keyboard-only operation for all workflows.
- Visible focus indicators.
- Semantic headings, landmarks, labels, and form relationships.
- Screen-reader-friendly status changes and validation errors.
- Sufficient contrast.
- No color-only status communication.
- Accessible data tables, menus, modals, calendars, filters, uploaders, charts, and kanban/pipeline views.
- Reduced motion support for nonessential animation.
- Error messages that are programmatically associated with fields.
- Touch targets suitable for mobile use.

Complex components such as drag-and-drop pipelines, scheduling calendars, and data grids must provide accessible alternatives.

---

## 20. Localization and global enterprise guardrails

Global ATS designs must account for:

- Language expansion and right-to-left readiness where relevant.
- Time zones and daylight saving changes.
- Date, time, currency, phone, address, name, and tax/ID format differences.
- Country-specific application questions and notices.
- Region-specific retention and consent rules.
- Remote, hybrid, onsite, multi-location, and cross-border roles.
- Local compensation display rules.
- Localized templates and candidate communications.

Never hardcode country-specific assumptions into reusable ATS components.

---

## 21. Analytics and reporting guardrails

Reporting must be actionable, permission-aware, and explainable.

Support dashboards for:

- Funnel conversion.
- Time-to-fill.
- Time-in-stage.
- Source quality.
- Candidate drop-off.
- Interview feedback delays.
- Offer acceptance and decline reasons.
- Aging requisitions.
- Recruiter workload.
- Hiring-manager responsiveness.
- Compliance and audit reporting.
- Data quality issues.

Every metric should have:

- Clear definition.
- Date range and filters.
- Data freshness indicator.
- Drill-down to source records where permissioned.
- Export controls based on permissions.
- Empty/error states.

Do not show sensitive demographic or compliance metrics to unauthorized users.

---

## 22. Integration and sync guardrails

Enterprise ATS integrations often include HRIS, SSO, calendars, email, SMS, job boards, assessment vendors, background-check vendors, payroll, onboarding, analytics, and document signing.

Integration UX must show:

- Connection status.
- Last successful sync.
- Failed records.
- Retry options.
- Field mapping and transformation errors.
- Permission or token expiration issues.
- Partial success states.
- Ownership of the integration.
- Impacted candidates/jobs/offers.

Do not hide sync failures behind generic errors. Recruiters and admins need to know whether an action was completed, delayed, partially completed, or failed.

---

## 23. Data quality and duplicate management guardrails

Support data quality as a first-class workflow.

Required patterns:

- Duplicate candidate detection with match evidence.
- Controlled merge/unmerge workflow.
- Preservation of source, referral, agency, application, and communication history.
- Stale requisition and stale candidate indicators.
- Missing required data warnings.
- Inconsistent source and tag cleanup.
- Data health dashboard for admins and reporting owners.

Never merge candidate records silently or overwrite candidate-provided data without a visible record of what changed.

---

## 24. Notifications and attention management guardrails

Notifications should reduce work, not create noise.

Required:

- In-app task center.
- Configurable notification preferences.
- Digest options for lower-priority updates.
- Escalation for overdue feedback, approvals, scheduling, or candidate responses.
- Clear priority and due date.
- Deep link to the exact record/action.
- Candidate-safe communication boundaries.

Avoid duplicate notifications across email, in-app, Slack/Teams, and task inbox unless the event is urgent.

---

## 25. Bulk action guardrails

Bulk actions are high-risk and high-value.

For bulk email, reject, move stage, tag, assign, export, archive, or status change:

- Show selected count and excluded count.
- Explain why records are excluded.
- Preview affected records.
- Preview candidate-facing content.
- Validate merge fields and permissions.
- Require confirmation for irreversible or external-facing actions.
- Provide progress state for long-running actions.
- Provide success, partial success, and failure results.
- Log the action with actor, timestamp, criteria, and affected records.

---

## 26. Visual design and design system guardrails

Use the project design system before inventing new UI.

Components must align on:

- Status tags and colors.
- Stage badges.
- Buttons and destructive actions.
- Form fields.
- Tables and filters.
- Modals, drawers, popovers, and toasts.
- Navigation and breadcrumbs.
- Empty/loading/error states.
- Accessibility patterns.
- Responsive behavior.
- Icon semantics.

Do not create visually similar components with different behavior or different names.

---

## 27. Performance and scalability guardrails

Design for enterprise volume:

- Large candidate databases.
- Many open requisitions.
- High-volume seasonal hiring.
- Many custom fields.
- Many business units and countries.
- Many integrations and sync events.
- Long audit histories.
- Large reports.

Required UX patterns:

- Progressive loading.
- Pagination or virtualization for large lists.
- Background processing for expensive actions.
- Clear progress and completion indicators.
- Cached/saved views where appropriate.
- Non-blocking interactions for long-running tasks.
- Graceful degradation if analytics or integrations are slow.

---

## 28. Screen state checklist

Every new or changed screen must define these states before implementation:

- First-time empty state.
- Filtered no-results state.
- Loading state.
- Slow-loading state.
- Success state.
- Validation error state.
- System error state.
- Integration error state.
- Permission denied state.
- Archived/closed/expired record state.
- Offline or interrupted network state, if applicable.
- Partial success state for bulk or integration flows.
- Unsaved changes state.
- Read-only state.
- Audit/history state.

---

## 29. Production UI definition of done

A UX/UI change is not production-ready until the OpenSpec change or PR documents:

1. Affected roles and permissions.
2. Primary workflow and alternate flows.
3. Data shown, hidden, editable, masked, and audited.
4. Candidate-visible versus internal-only content.
5. Accessibility acceptance criteria.
6. Responsive behavior.
7. Empty/loading/error/permission/integration states.
8. Localization and time-zone considerations.
9. Analytics events and reporting impacts.
10. Security/privacy/compliance impacts.
11. Bulk action safeguards, if any.
12. AI/automation transparency, if any.
13. Integration sync behavior, if any.
14. Test cases for critical flows and edge cases.
15. Rollback or recovery plan for risky UI changes.

---

## 30. OpenSpec artifact rules for ATS UX/UI work

When a change involves UX/UI, OpenSpec artifacts MUST include the following.

### `proposal.md`

- Problem statement by role.
- Affected ATS roles and workflows.
- Candidate impact, if any.
- Compliance/privacy/security impact.
- Success metrics.
- Out of scope items.
- Relevant Unblocked findings, if available.

### `design.md`

- Screen inventory.
- User flow or state transition model.
- Permission and visibility model.
- Data model at UI level.
- Component reuse plan.
- Accessibility plan.
- Responsive behavior.
- Error, empty, loading, and integration states.
- Analytics events.
- AI/automation transparency approach, if applicable.

### `tasks.md`

- UX review task.
- Accessibility review task.
- Security/privacy review task for sensitive data.
- Integration error-state task, if applicable.
- Analytics instrumentation task, if applicable.
- Unit/component/e2e test tasks for critical paths.
- Manual QA task covering role-based access.

### `specs/**/spec.md`

- Use clear requirements with observable behavior.
- Include role, permission, and state-specific scenarios.
- Include candidate-visible and internal-only expectations.
- Include audit and notification behavior where applicable.

---

## 31. Claude Code usage rule

Claude Code agents MUST NOT start ATS UI generation or production UI implementation until they have loaded or read this file and summarized:

- Affected roles.
- Primary user task.
- Sensitive data involved.
- Required states.
- OpenSpec artifacts checked.
- Unblocked context checked, if configured.
- Known gaps or assumptions.

If the current task asks for mockups only, the same rule applies. Mockups often become implementation plans; do not skip guardrails.

---

## 32. Prohibited patterns

Do not introduce these patterns:

- One universal dashboard for all roles.
- Candidate-visible pages that expose internal notes, scorecards, compensation deliberation, or AI/private reasoning.
- AI ranking or recommendation without explanation, review, and override.
- Bulk actions without preview and confirmation.
- Color-only status indicators.
- Drag-and-drop-only pipeline movement without keyboard alternative.
- Silent integration failures.
- Silent duplicate merges.
- Destructive actions without confirmation and audit trail.
- Long forms without draft or recovery strategy.
- Admin configuration without preview, dependency visibility, or rollback for critical settings.
- Reports that expose unauthorized or sensitive demographic/compliance data.
- UI that assumes a single country, language, timezone, currency, or employment model.

---

## 33. Quick pre-flight checklist for agents

Before submitting any UX/UI output, answer yes/no:

- Did I identify all affected ATS roles?
- Did I read the relevant OpenSpec artifacts?
- Did I check Unblocked context if available?
- Did I define candidate-visible versus internal-only content?
- Did I define permissions and data sensitivity?
- Did I include all major screen states?
- Did I account for accessibility?
- Did I account for mobile/responsive behavior?
- Did I account for localization/time zones where relevant?
- Did I account for audit logs and compliance needs?
- Did I account for integration failures?
- Did I account for analytics/reporting impacts?
- Did I avoid prohibited patterns?

If any answer is no, do not proceed silently. Update the OpenSpec artifact, revise the mockup, or state the assumption clearly.

---

## 34. Recommended repository wiring

Recommended canonical placement for this file:

```text
.claude/rules/ats-ux-ui-guardrails.md
```

Use this as the single source of truth for ATS UX/UI rules. Keep the file in `.claude/rules/` without YAML `paths:` frontmatter so it applies globally to Claude Code sessions rather than only after a matching source file is opened.

Do not also import this same file from `CLAUDE.md` if it already lives in `.claude/rules/`; that can duplicate context. Instead, add a short pointer in `CLAUDE.md` so humans and agents know the rule exists.

Recommended `CLAUDE.md` pointer:

```md
## ATS UX/UI guardrail

Before creating or changing any ATS UX/UI mockup, screen, component, page, route, design artifact, or production UI, follow the project rule in `.claude/rules/ats-ux-ui-guardrails.md`.
```

If your team prefers to keep canonical product documentation under `docs/`, use this alternative instead:

```text
docs/ux/ats-ux-ui-guardrails.md
```

Then import it from the root `CLAUDE.md`:

```md
@docs/ux/ats-ux-ui-guardrails.md
```

Recommended `AGENTS.md` snippet:

```md
## ATS UX/UI guardrail

Before creating or changing any ATS UX/UI mockup, screen, component, page, route, design artifact, or production UI, read and follow `.claude/rules/ats-ux-ui-guardrails.md`.
```

Recommended `openspec/config.yaml` addition:

```yaml
context: |
  This is an enterprise Applicant Tracking System (ATS). All UX/UI changes must follow `.claude/rules/ats-ux-ui-guardrails.md` before mockup, prototype, or production UI work begins.

rules:
  proposal:
    - For UX/UI changes, identify affected ATS roles, candidate impact, compliance/privacy/security impact, success metrics, and out-of-scope items.
  design:
    - For UX/UI changes, include screen inventory, user flows, permission model, state model, accessibility plan, responsive behavior, analytics events, and integration failure behavior.
  specs:
    - For UX/UI changes, include role-specific behavior, permission-specific behavior, candidate-visible/internal-only boundaries, audit behavior, and required screen states.
  tasks:
    - For UX/UI changes, include UX review, accessibility review, security/privacy review, analytics, integration error-state coverage, and role-based QA.
```

---

## 35. Maintenance rule

Update this guardrail whenever the ATS adds or materially changes:

- A user role.
- Candidate-facing workflow.
- Hiring workflow.
- Compliance requirement.
- AI or automation feature.
- Integration category.
- Admin configuration model.
- Reporting model.
- Design system standard.
- Accessibility standard.

Changes to this file should be reviewed by product, design, engineering, accessibility, security, and compliance owners.
