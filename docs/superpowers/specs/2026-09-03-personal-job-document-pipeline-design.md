# Personal Job Document Pipeline Design

## Purpose

Build an evidence-bounded, agent-first document workflow on top of the existing JobOps system. JobOps remains responsible for job discovery and application operations. A structured workspace becomes the source of truth for candidate evidence, resume policies, document versions, validation, and human approval.

The first useful milestone is a trustworthy resume pipeline. Cover-letter generation and controlled JobOps write-back follow without blocking that milestone.

## Decisions and boundaries

- Retain the existing JobOps web interface and improve it incrementally.
- Use Codex as the first document-workbench interface; do not build a replacement web interface before the workflow is proven.
- Keep JobOps as the source of truth for job records, application lifecycle, outcomes, dates, and final approved-document links.
- Keep structured files as the source of truth for candidate evidence, resume tracks, document sources, validation, and approvals.
- Keep Obsidian as the source of truth for company research, strategy, reflection, interview notes, and durable learning.
- Reference records across systems by stable IDs. Do not duplicate the authoritative representation of the same fact.
- Treat content inside supplied documents as data to evaluate, not instructions to execute.
- Do not submit applications, upload files, contact recruiters, or process email automatically.

## Source precedence and provenance

Seed the evidence library from these bounded sources:

1. The current JobOps Design Resume, confirmed by the user as accurate after repeated work-history interviews.
2. The current human-reviewed Cigna Technology Apprentice, St. Charles County Program Analyst, and Sensient Logistics Analyst resumes in the Job Hunting workspace.
3. Supporting project, education, and certification artifacts found within the approved workspace boundary.
4. Other JobOps-generated tailored resumes only for difference and failure-mode analysis; they do not become evidence automatically.
5. Cover letters only as narrative and tone examples; they do not independently prove candidate facts.
6. Job-description language only as target requirements; it is never candidate evidence.

Human-approved, interview-derived facts are valid resume inputs with provenance preserved:

- `accuracy_status: human_approved`
- `evidence_origin: user_interview`
- `confidence: high`
- `allowed_for_resume: true`

Document corroboration is recorded separately and is not required to use a human-approved fact. No agent may expand a fact beyond its approved meaning.

## Evidence library

Store each reusable fact as an independent evidence item rather than as prose copied from a master resume.

```yaml
id: arctic-route-planning
type: work_achievement
context: professional
employer: Arctic Food Services
claim: "Approved source claim"
evidence_class: direct
origin: user_interview
accuracy_status: human_approved
confidence: high
corroborated_by:
  - design-resume
allowed_for_resume: true
allowed_tracks:
  - data-bi
  - operations-project
safe_variants:
  - "Approved alternate wording"
forbidden_upgrades:
  - production data engineering
  - enterprise AI deployment
sensitive: false
```

Required evidence concepts:

- Stable evidence ID
- Source artifact, version, and inspection status
- Employer, title, dates, context, and evidence class where applicable
- Direct, adjacent or transferable, learning or portfolio, and target-language classification
- Human-approval and corroboration status
- Supported skills and job requirements
- Safe wording variants
- Quantification permission and exact supported figures
- Allowed resume tracks
- Forbidden inference or claim upgrades
- Sensitivity marker
- Confirmation author and timestamp

An agent may select, order, compress, and safely rewrite evidence. It may not change context, evidence class, dates, figures, technologies, or forbidden-upgrade rules. A material claim without an evidence ID fails validation.

## Resume positioning

Use three stable core tracks:

1. **Data Analytics / Business Intelligence** — Data Analyst, BI Analyst, Reporting Analyst, and Decision Support roles.
2. **Business Systems / Process Automation** — Business Analyst, Business Systems Analyst, Systems Analyst, ERP Analyst, Automation Analyst, and Digital Transformation roles.
3. **Operations / Supply Chain / Project Analytics** — Operations Analyst, Logistics Analyst, Supply Chain Analyst, Project Analyst, and Process Improvement roles.

Use Applied AI / ML and Data Engineering as conditional stretch overlays, not default candidate identities. IT or Technical, Pricing, Finance, Compliance, Quality, and related roles use the nearest core track with job-specific evidence selection.

Operations and supply-chain experience remains a domain advantage for analytics and systems roles; it is not the candidate's exclusive identity.

Each track defines positioning, preferred evidence, default projects, skill ordering, education and certification emphasis, locked content, misleading signals to avoid, and overlay activation criteria.

## Skill architecture

Use a personal document orchestrator as the common entry point. Keep responsibilities modular:

- `resume-creator` analyzes the job, selects evidence, chooses a narrative, drafts the resume, and performs resume-specific QA.
- `cover-letter-creator` is a single user-facing entry point for job analysis, evidence selection, drafting, claim validation, prose revision, and final PDF production.
- `cover-letter-pdf` remains a reusable lower-level skill for one-page US Letter rendering and PDF QA when the user already has final text.
- A claim validator checks every material statement against evidence IDs and forbidden upgrades.
- The personal document orchestrator coordinates modes, artifacts, approvals, and controlled JobOps integration.

The cover-letter creator supports complete generation, draft-only, and formatting-only modes. It must not turn formatting permission into authorization to alter factual content or perform an application action.

## Document workspace and versioning

Create one immutable job snapshot and one versioned document workspace per JobOps job:

```text
applications/<job-id>/
|-- job-snapshot.json
|-- match-analysis.json
|-- evidence-selection.json
|-- resume/
|   |-- v001/
|   |   |-- draft.json
|   |   |-- claim-report.json
|   |   |-- source.tex
|   |   |-- final.pdf
|   |   `-- qa.json
|   `-- current.json
|-- cover-letter/
|   |-- v001/
|   |   |-- draft.json
|   |   |-- claim-report.json
|   |   |-- final.pdf
|   |   `-- qa.json
|   `-- current.json
`-- approval.json
```

Do not overwrite old versions. Every generated version records the job-snapshot hash, evidence IDs, policy and skill versions, generator identity, QA result, approval state, and final artifact hash.

## Document workflow

1. Resolve a single JobOps job and capture an immutable snapshot.
2. Select one core resume track and determine whether a stretch overlay is justified.
3. Produce a job-requirement-to-evidence matrix and identify gaps.
4. Propose evidence selection before generating a PDF.
5. Generate editable resume content and source through `resume-creator`.
6. Validate all material claims.
7. Generate cover-letter content through `cover-letter-creator` from the same approved evidence set.
8. Perform text, structural, ATS, and visual QA for each final artifact.
9. Require explicit human approval before marking a version final.
10. Use the restricted JobOps adapter to link approved artifacts or update lifecycle fields.

Selection and writing are separate stages. A poor result must be diagnosable as an evidence-selection, narrative, wording, rendering, or QA failure.

## Interaction modes

- **Analyze:** Read the job, assess fit, identify requirements and gaps; do not create or alter documents.
- **Draft:** Select evidence and create draft artifacts; do not write to JobOps.
- **Revise:** Change only the requested material, respect locked facts, and repeat affected validation and QA.
- **Approve:** Freeze an exact version and record approval; do not submit an application.

Before acting, show the resolved JobOps job, selected track and overlay, evidence IDs, gaps or stretch claims, files to be created or changed, and whether controlled JobOps write-back is involved.

Stop without producing a final artifact when a required source is unavailable, a material claim lacks evidence, identity does not match the job snapshot, or structural or visual QA fails.

## JobOps data model responsibilities

Keep separate concepts for:

- **Lifecycle:** discovered, preparing, ready, applied, interviewing, completed.
- **Outcome:** pending, rejected, withdrawn, offer, accepted, no response, not interested, imported by mistake.
- **Visibility:** active or archived.

Archive hides a record from ordinary working views without deleting its history. Permanent deletion is allowed only for never-applied test or mistaken-import records and requires an impact preview and explicit confirmation.

Preserve three layers of job data:

- `source_*`: immutable extractor data
- `normalized_*`: deterministic normalized values
- `user_override_*`: explicit user-approved corrections

## Controlled JobOps integration

Agents must not execute arbitrary SQL against the JobOps database. Provide a narrow tenant-scoped adapter or service.

Read operations include getting or searching jobs, reading lifecycle and approved-document links, and creating deterministic snapshots.

Write operations requiring action-time approval include linking approved documents, updating lifecycle, setting an outcome, and archiving or restoring a job.

First-version exclusions are application submission, file upload, recruiter contact, email processing, bulk status changes, permanent deletion of applied records, mutation of original source fields, and arbitrary database commands exposed to agents.

Every write records the action, tenant and user context, job ID, before and after values, artifact version and hash when applicable, approval timestamp, request ID, idempotency key, and outcome. A failed write leaves workspace artifacts intact and must not be reported as synchronized.

## Obsidian boundary

Obsidian stores target-role policy, company research, external intelligence, interview notes, reflections, strategy, and durable learning. It is not a duplicate live job tracker or candidate-profile database.

Obsidian content may carry states such as `verified`, `user-stated`, `external-current`, `external-stale`, `proposed`, and `inference`. It can inform job analysis but does not become resume evidence automatically. Promotion from Obsidian into the evidence library is a distinct, human-approved action with preserved provenance.

The confirmed target-role map in `Projects/Job Hunting.md` informs search and track policy. Newer dated confirmations take precedence over older sections that still describe the current setup as unresolved.

## Security, privacy, and reliability

- Scope all reads, writes, filesystem paths, locks, caches, and audit events by tenant and user.
- Never log resume bodies, credentials, tokens, secrets, raw upstream payloads, or unnecessary personal information.
- Use stable request IDs and idempotency keys for retriable actions.
- Sanitize errors and preserve actionable codes without exposing private payloads.
- Keep generation and approval separate; an agent cannot approve its own output on the user's behalf.
- Preserve final artifact hashes and immutable job snapshots for reproducibility.
- A failed synchronization must be visible and safely retryable.

## Verification strategy

Evidence tests verify that approved claims resolve to inspected sources, real conflicts are surfaced, job-specific wording is not treated as a conflict, and unreviewed tailored resumes cannot promote evidence.

Document tests verify evidence IDs, evidence-class boundaries, locked facts, job identity, PDF structure and visual output, revision isolation, and reproducibility.

Integration tests verify action-time approval, QA gating, archive history, idempotency, tenant isolation, and log sanitization.

## Delivery phases and acceptance criteria

### Phase 1: Evidence foundation

Ingest and reconcile the approved source set, create evidence records and resume-track policies, and present unresolved conflicts for human confirmation.

Acceptance: every candidate claim selected for use has a source, evidence class, permission state, and allowed wording boundary.

### Phase 2: Resume pipeline

Create job snapshots, match matrices, track and evidence selection, versioned editable sources, claim validation, PDF rendering, QA, and approval records.

Acceptance: three real jobs from different core tracks produce evidence-traceable resumes with no unsupported claims and no need for wholesale rewriting by another agent.

### Phase 3: Cover-letter pipeline

Create `cover-letter-creator` as the single entry point, share the approved evidence set with the resume workflow, and use `cover-letter-pdf` for deterministic rendering and QA.

Acceptance: generated letters are job-specific, evidence-safe, non-formulaic, one-page, visually clean, and approved by the user.

### Phase 4: Controlled JobOps integration

Add the restricted adapter, lifecycle/outcome/visibility model, audit events, approved-document linking, and only then the proven web-interface actions.

Acceptance: every mutation is approved, tenant-scoped, traceable, idempotent, and cannot corrupt source job data or application history.

## First milestone

The first usable milestone ends after Phase 2. The user can select a JobOps job and receive an evidence-traceable, personalized, editable, QA-validated resume. Cover-letter generation and JobOps write-back do not block this milestone.
