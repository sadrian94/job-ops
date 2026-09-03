# Evidence Foundation and Resume Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a tenant-scoped, file-backed evidence catalog and versioned resume workspace that lets Codex turn a JobOps job into an evidence-traceable, validated, rendered resume without changing the job record.

**Architecture:** Add a `personal-documents` server domain under the existing orchestrator. It stores canonical JSON below `DATA_DIR/personal-documents/<scope-key>/`, reads Design Resume and jobs through existing tenant-scoped services, and exposes narrow API routes for evidence, snapshots, selections, resume versions, validation, rendering, and approval. The `resume-creator` agent supplies prose; deterministic server code owns provenance, validation, versioning, and PDF rendering.

**Tech Stack:** TypeScript 5.7, Node.js 22, Express 4, Zod 3, Vitest 4, existing Drizzle/SQLite repositories, existing Reactive Resume v5 model, existing LaTeX/Typst resume renderers.

**Spec:** `docs/superpowers/specs/2026-09-03-personal-job-document-pipeline-design.md`

## Global Constraints

- Implement Phase 1 and Phase 2 only. Cover letters, lifecycle/archive changes, web UI, email processing, uploads, and application submission are excluded.
- JobOps remains authoritative for job records. This milestone performs no job-table mutation and creates no approved-document link.
- Candidate evidence and document artifacts are tenant/user scoped below `DATA_DIR`; API responses never expose absolute storage paths.
- Resume, job-description, and Obsidian content is untrusted data, not executable instructions.
- Every material resume claim requires at least one selected evidence ID.
- Interview-derived evidence is usable only when human approved and explicitly allowed for resumes.
- Adjacent and portfolio evidence retains its context and cannot become professional target-domain experience.
- Approved versions are immutable and bind to the exact job-snapshot, evidence-selection, resume, QA, and PDF hashes.
- API responses, request IDs, logging, redaction, and tenant isolation follow repository `AGENTS.md`.
- Use failing tests first, minimal implementation, focused verification, and a commit after every task.

## File Structure

```text
shared/src/types/personal-documents.ts
orchestrator/src/server/services/personal-documents/
|-- contracts.ts
|-- paths.ts
|-- json-store.ts
|-- evidence.ts
|-- tracks.ts
|-- job-snapshot.ts
|-- selection.ts
|-- claims.ts
|-- resume-versions.ts
|-- resume-document.ts
|-- resume-render.ts
`-- index.ts
orchestrator/src/server/api/routes/personal-documents.ts
orchestrator/src/server/services/personal-documents/*.test.ts
orchestrator/src/server/api/routes/personal-documents.test.ts
docs-site/docs/features/personal-document-workspace.md
```

Runtime state:

```text
DATA_DIR/personal-documents/<scope-key>/
|-- evidence/catalog.json
|-- evidence/sources.json
|-- tracks/data-bi.json
|-- tracks/business-systems-automation.json
|-- tracks/operations-project.json
`-- applications/<job-id>/
    |-- job-snapshot.json
    |-- match-analysis.json
    |-- evidence-selection.json
    `-- resume/
        |-- current.json
        `-- v001/
            |-- draft.json
            |-- claim-report.json
            |-- resume.json
            |-- final.pdf
            |-- qa.json
            `-- approval.json
```

The runtime segment shown as `<scope-key>` is derived only from `getPrivateDataScope().scopeKey`; callers never supply it.

---

### Task 1: Shared Contracts and Strict Schemas

**Files:**
- Create: `shared/src/types/personal-documents.ts`
- Modify: `shared/src/types/index.ts`
- Create: `orchestrator/src/server/services/personal-documents/contracts.ts`
- Test: `orchestrator/src/server/services/personal-documents/contracts.test.ts`

**Interfaces:**
- Consumes: existing `Job` and Reactive Resume v5 JSON types.
- Produces: `EvidenceItem`, `EvidenceCatalog`, `ResumeTrack`, `JobSnapshot`, `EvidenceSelection`, `ResumeDraft`, `ClaimReport`, `ResumeQaReport`, `ResumeApproval`, and `ResumeVersionManifest` plus matching Zod schemas.

- [ ] **Step 1: Write the failing schema test**

```ts
import { describe, expect, it } from "vitest";
import { evidenceCatalogSchema, resumeDraftSchema } from "./contracts";

describe("personal document contracts", () => {
  it("accepts approved interview evidence", () => {
    const result = evidenceCatalogSchema.parse({
      schemaVersion: 1,
      items: [{
        id: "arctic-route-planning",
        type: "work_achievement",
        context: "professional",
        claim: "Sequenced delivery routes using operational constraints.",
        evidenceClass: "direct",
        origin: "user_interview",
        accuracyStatus: "human_approved",
        confidence: "high",
        allowedForResume: true,
        corroboratedBy: ["design-resume"],
        allowedTracks: ["data-bi", "operations-project"],
        safeVariants: ["Sequenced delivery routes using operational constraints."],
        forbiddenUpgrades: ["production route-optimization platform"],
        sensitive: false,
        sources: [{ artifactId: "design-resume", inspected: true }],
        confirmedBy: "user",
        confirmedAt: "2026-09-03T17:00:00.000Z",
      }],
    });
    expect(result.items).toHaveLength(1);
  });

  it("rejects a draft with no claims", () => {
    expect(() => resumeDraftSchema.parse({ schemaVersion: 1, claims: [] }))
      .toThrow();
  });
});
```

- [ ] **Step 2: Run the test and confirm the missing-module failure**

Run: `npm --workspace orchestrator run test:run -- src/server/services/personal-documents/contracts.test.ts`

Expected: FAIL because `./contracts` does not exist.

- [ ] **Step 3: Define shared unions and interfaces**

```ts
export type EvidenceClass = "direct" | "adjacent" | "learning_portfolio";
export type EvidenceOrigin =
  | "user_interview"
  | "design_resume"
  | "human_reviewed_resume"
  | "supporting_artifact"
  | "obsidian_promoted";
export type AccuracyStatus = "human_approved" | "needs_review" | "rejected";
export type ResumeTrackId =
  | "data-bi"
  | "business-systems-automation"
  | "operations-project";
export type StretchOverlayId = "applied-ai-ml" | "data-engineering";
export type ResumeVersionStatus =
  | "draft"
  | "validation_failed"
  | "qa_failed"
  | "ready_for_review"
  | "approved";
```

Every persisted interface has `schemaVersion: 1`. Every `ResumeDraftClaim` has `claimId`, `text`, `evidenceIds`, `section`, and `contextLabel`. Hash fields are lowercase 64-character SHA-256 values; timestamps are ISO strings.

- [ ] **Step 4: Implement strict Zod schemas**

Use `.strict()` on persisted objects. Refine approved evidence to require at least one inspected source; refine resume-enabled evidence to require `human_approved`; require non-empty evidence IDs on material claims; require job-snapshot and evidence-selection hashes on version manifests.

- [ ] **Step 5: Export shared types and rerun the test**

Add `export * from "./personal-documents";` to `shared/src/types/index.ts`.

Run: `npm --workspace orchestrator run test:run -- src/server/services/personal-documents/contracts.test.ts`

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add shared/src/types/personal-documents.ts shared/src/types/index.ts orchestrator/src/server/services/personal-documents/contracts.ts orchestrator/src/server/services/personal-documents/contracts.test.ts
git commit -m "feat: define personal document contracts"
```

---

### Task 2: Tenant-Scoped Atomic JSON Storage

**Files:**
- Create: `orchestrator/src/server/services/personal-documents/paths.ts`
- Create: `orchestrator/src/server/services/personal-documents/json-store.ts`
- Test: `orchestrator/src/server/services/personal-documents/json-store.test.ts`

**Interfaces:**
- Consumes: `getDataDir()` and `getPrivateDataScope()`.
- Produces: `getPersonalDocumentsRoot(): string`, `resolvePersonalDocumentPath(...segments: string[]): string`, `readJsonArtifact<T>(path, schema): Promise<T | null>`, and `writeJsonArtifactAtomic<T>(path, value, schema): Promise<void>`.

- [ ] **Step 1: Write traversal and atomicity tests**

```ts
it("keeps artifacts inside the active private scope", () => {
  const path = resolvePersonalDocumentPath("evidence", "catalog.json");
  expect(path).toContain(join(
    tempDir,
    "personal-documents",
    Buffer.from("tenant_default", "utf8").toString("base64url"),
  ));
  expect(() => resolvePersonalDocumentPath("..", "outside.json"))
    .toThrow("Invalid personal document path segment");
});

it("validates before replacing an existing artifact", async () => {
  await writeJsonArtifactAtomic(path, validCatalog, evidenceCatalogSchema);
  await expect(
    writeJsonArtifactAtomic(path, { schemaVersion: 1 }, evidenceCatalogSchema),
  ).rejects.toThrow();
  expect(await readJsonArtifact(path, evidenceCatalogSchema)).toEqual(validCatalog);
});
```

- [ ] **Step 2: Run the tests and confirm failure**

Run: `npm --workspace orchestrator run test:run -- src/server/services/personal-documents/json-store.test.ts`

Expected: FAIL because the storage functions are missing.

- [ ] **Step 3: Implement safe path resolution**

Encode `getPrivateDataScope().scopeKey` with `Buffer.from(scopeKey, "utf8").toString("base64url")` and build the root with `join(getDataDir(), "personal-documents", encodedScopeKey)`. Permit later path segments matching `/^[A-Za-z0-9._-]+$/`, resolve the result, and reject results outside the resolved scope root. Tests cover distinct tenant/user scope keys to prevent path collisions.

- [ ] **Step 4: Implement atomic JSON writes**

Validate first, serialize as `JSON.stringify(value, null, 2) + "\n"`, create the parent directory, write a same-directory temporary file with mode `0o600`, rename it over the destination, and remove the temporary file on failure. Never log artifact bodies.

- [ ] **Step 5: Run tests and commit**

Run: `npm --workspace orchestrator run test:run -- src/server/services/personal-documents/json-store.test.ts`

Expected: PASS.

```bash
git add orchestrator/src/server/services/personal-documents/paths.ts orchestrator/src/server/services/personal-documents/json-store.ts orchestrator/src/server/services/personal-documents/json-store.test.ts
git commit -m "feat: add scoped personal document storage"
```

---

### Task 3: Approved Evidence Catalog

**Files:**
- Create: `orchestrator/src/server/services/personal-documents/evidence.ts`
- Test: `orchestrator/src/server/services/personal-documents/evidence.test.ts`

**Interfaces:**
- Consumes: structured source records and evidence proposals prepared by the agent workflow.
- Produces: `getEvidenceCatalog()`, `replaceEvidenceCatalog(input)`, `listEvidenceForTrack(trackId)`, and `validateEvidencePromotion(input)`.

- [ ] **Step 1: Write evidence-boundary tests**

```ts
it("permits human-approved interview evidence", () => {
  expect(validateEvidencePromotion(approvedInterviewEvidence)).toEqual({ ok: true });
});

it("blocks unreviewed generated content", () => {
  expect(() => validateEvidencePromotion({
    ...approvedInterviewEvidence,
    origin: "human_reviewed_resume",
    accuracyStatus: "needs_review",
  })).toThrow("Evidence must be human approved before resume use");
});
```

- [ ] **Step 2: Run the tests and confirm failure**

Run: `npm --workspace orchestrator run test:run -- src/server/services/personal-documents/evidence.test.ts`

Expected: FAIL because the evidence service is missing.

- [ ] **Step 3: Implement catalog validation**

Reject duplicate IDs, uninspected sources, unknown tracks, empty safe variants, unsupported origins, and `allowedForResume: true` unless status is `human_approved`. Sort items by ID before persistence for stable hashes and diffs.

- [ ] **Step 4: Persist source metadata separately**

Each source record contains `artifactId`, display name, artifact kind, private source-path label, SHA-256, inspection timestamp, human-review state, and optional Design Resume revision. Public responses omit the private source path.

- [ ] **Step 5: Run tests and commit**

Run: `npm --workspace orchestrator run test:run -- src/server/services/personal-documents/evidence.test.ts`

Expected: PASS.

```bash
git add orchestrator/src/server/services/personal-documents/evidence.ts orchestrator/src/server/services/personal-documents/evidence.test.ts
git commit -m "feat: add approved evidence catalog"
```

---

### Task 4: Resume Track Policies

**Files:**
- Create: `orchestrator/src/server/services/personal-documents/tracks.ts`
- Test: `orchestrator/src/server/services/personal-documents/tracks.test.ts`

**Interfaces:**
- Consumes: job title and description.
- Produces: `getDefaultResumeTracks(): ResumeTrack[]` and `recommendResumeTrack(job): TrackRecommendation`.

- [ ] **Step 1: Write deterministic policy tests**

```ts
it.each([
  ["Business Intelligence Analyst", "data-bi"],
  ["ERP Analyst", "business-systems-automation"],
  ["Logistics Analyst - Warehouse and Inventory", "operations-project"],
])("maps %s to %s", (title, expectedTrack) => {
  expect(recommendResumeTrack({ title, jobDescription: title }).trackId)
    .toBe(expectedTrack);
});

it("requires review for a data-engineering overlay", () => {
  const result = recommendResumeTrack({
    title: "Junior Data Engineer",
    jobDescription: "Build ETL pipelines and SQL data models.",
  });
  expect(result.overlayIds).toEqual(["data-engineering"]);
  expect(result.requiresHumanReview).toBe(true);
});
```

- [ ] **Step 2: Run the tests and confirm failure**

Run: `npm --workspace orchestrator run test:run -- src/server/services/personal-documents/tracks.test.ts`

Expected: FAIL because track policies are missing.

- [ ] **Step 3: Implement the approved tracks**

Encode Data Analytics/BI, Business Systems/Process Automation, and Operations/Supply Chain/Project Analytics. Each policy includes narrative purpose, role families, evidence priorities, project categories, skill ordering, education emphasis, locked facts, misleading signals, and overlay rules.

- [ ] **Step 4: Keep selection advisory**

Return matched signals and reasons. Require a separately persisted human-approved selection. Applied AI/ML and Data Engineering always return `requiresHumanReview: true`.

- [ ] **Step 5: Run tests and commit**

Run: `npm --workspace orchestrator run test:run -- src/server/services/personal-documents/tracks.test.ts`

Expected: PASS.

```bash
git add orchestrator/src/server/services/personal-documents/tracks.ts orchestrator/src/server/services/personal-documents/tracks.test.ts
git commit -m "feat: define personal resume tracks"
```

---

### Task 5: Immutable Job Snapshots

**Files:**
- Create: `orchestrator/src/server/services/personal-documents/job-snapshot.ts`
- Test: `orchestrator/src/server/services/personal-documents/job-snapshot.test.ts`

**Interfaces:**
- Consumes: `getJobById(jobId)` and scoped JSON storage.
- Produces: `createJobSnapshot(jobId): Promise<JobSnapshot>` and `getJobSnapshot(jobId): Promise<JobSnapshot | null>`.

- [ ] **Step 1: Write identity and privacy tests**

```ts
it("returns the existing snapshot for unchanged job identity", async () => {
  const first = await createJobSnapshot(job.id);
  const second = await createJobSnapshot(job.id);
  expect(second.sha256).toBe(first.sha256);
  expect(second.capturedAt).toBe(first.capturedAt);
});

it("excludes mutable tailoring and storage fields", async () => {
  const snapshot = await createJobSnapshot(job.id);
  expect(snapshot.job).not.toHaveProperty("pdfPath");
  expect(snapshot.job).not.toHaveProperty("tailoredSummary");
  expect(snapshot.job).not.toHaveProperty("tailoredSkills");
});
```

- [ ] **Step 2: Run the tests and confirm failure**

Run: `npm --workspace orchestrator run test:run -- src/server/services/personal-documents/job-snapshot.test.ts`

Expected: FAIL because snapshot creation is missing.

- [ ] **Step 3: Implement canonical hashing**

Snapshot source identity, title, employer, URLs, posting date, location, compensation fields, job type, level, function, description, skills, experience range, deadline, and discovery time. Canonically sort object keys before hashing. If a snapshot exists, return it without replacement.

- [ ] **Step 4: Add missing-job and isolation coverage**

Return a typed `NOT_FOUND` error for an unavailable job. Verify another tenant/user scope cannot read the first scope's snapshot.

- [ ] **Step 5: Run tests and commit**

Run: `npm --workspace orchestrator run test:run -- src/server/services/personal-documents/job-snapshot.test.ts`

Expected: PASS.

```bash
git add orchestrator/src/server/services/personal-documents/job-snapshot.ts orchestrator/src/server/services/personal-documents/job-snapshot.test.ts
git commit -m "feat: add immutable job snapshots"
```

---

### Task 6: Evidence Selection and Claim Validation

**Files:**
- Create: `orchestrator/src/server/services/personal-documents/selection.ts`
- Create: `orchestrator/src/server/services/personal-documents/claims.ts`
- Test: `orchestrator/src/server/services/personal-documents/selection.test.ts`
- Test: `orchestrator/src/server/services/personal-documents/claims.test.ts`

**Interfaces:**
- Consumes: `JobSnapshot`, selected track, `EvidenceCatalog`, `EvidenceSelection`, and `ResumeDraft`.
- Produces: `saveMatchAnalysis(input)`, `saveEvidenceSelection(input)`, and `validateResumeClaims(input): ClaimReport`.

- [ ] **Step 1: Write selection-integrity tests**

```ts
it("rejects evidence outside the selected track", async () => {
  await expect(saveEvidenceSelection({
    jobId: "job-1",
    jobSnapshotHash: "a".repeat(64),
    trackId: "data-bi",
    overlayIds: [],
    evidenceIds: ["operations-only-item"],
    requirementMappings: [],
    confirmedByUser: true,
    approvedAt: "2026-09-03T17:00:00.000Z",
  })).rejects.toThrow("Evidence is not allowed for track data-bi");
});
```

- [ ] **Step 2: Write claim-boundary tests**

```ts
it("fails professional wording backed only by portfolio evidence", () => {
  const report = validateResumeClaims({
    draft: draftWithProfessionalProductionClaim,
    selection: approvedSelection,
    catalog: catalogWithPortfolioEvidence,
  });
  expect(report.status).toBe("failed");
  expect(report.issues[0]?.code).toBe("CONTEXT_UPGRADE");
});

it("fails a forbidden upgrade", () => {
  const report = validateResumeClaims({
    draft: draftClaimingEnterpriseAiDeployment,
    selection: approvedSelection,
    catalog,
  });
  expect(report.issues.map((issue) => issue.code))
    .toContain("FORBIDDEN_UPGRADE");
});
```

- [ ] **Step 3: Run both tests and confirm failure**

Run: `npm --workspace orchestrator run test:run -- src/server/services/personal-documents/selection.test.ts src/server/services/personal-documents/claims.test.ts`

Expected: FAIL because the services are missing.

- [ ] **Step 4: Implement approved selection persistence**

Verify snapshot hash, track, overlays, catalog membership, allowed tracks, resume permission, human approval, and explicit gap handling. Persist requirement mappings with `matched`, `adjacent`, `learning_portfolio`, or `gap` status.

- [ ] **Step 5: Implement deterministic claim validation**

For every material claim, verify that evidence exists and was selected, context labels do not upgrade evidence, numeric tokens occur in approved evidence, named tools occur in approved claims or safe variants, and forbidden-upgrade phrases do not occur. Emit `MISSING_EVIDENCE`, `UNSELECTED_EVIDENCE`, `CONTEXT_UPGRADE`, `UNSUPPORTED_NUMBER`, `UNSUPPORTED_TOOL`, and `FORBIDDEN_UPGRADE` with claim IDs, without logging full claim text.

- [ ] **Step 6: Run tests and commit**

Run: `npm --workspace orchestrator run test:run -- src/server/services/personal-documents/selection.test.ts src/server/services/personal-documents/claims.test.ts`

Expected: PASS.

```bash
git add orchestrator/src/server/services/personal-documents/selection.ts orchestrator/src/server/services/personal-documents/selection.test.ts orchestrator/src/server/services/personal-documents/claims.ts orchestrator/src/server/services/personal-documents/claims.test.ts
git commit -m "feat: validate resume evidence selections"
```

---

### Task 7: Versioned Resume Assembly

**Files:**
- Create: `orchestrator/src/server/services/personal-documents/resume-versions.ts`
- Create: `orchestrator/src/server/services/personal-documents/resume-document.ts`
- Test: `orchestrator/src/server/services/personal-documents/resume-versions.test.ts`
- Test: `orchestrator/src/server/services/personal-documents/resume-document.test.ts`

**Interfaces:**
- Consumes: current Design Resume JSON, `ResumeDraft`, approved selection, and passing claim report.
- Produces: `createResumeVersion(input): Promise<ResumeVersionManifest>`, `assembleResumeDocument(input): Record<string, unknown>`, and `getResumeVersion(jobId, version): Promise<ResumeVersionBundle>`.

- [ ] **Step 1: Write immutability tests**

```ts
it("allocates v001 then v002 without replacing v001", async () => {
  const first = await createResumeVersion(validVersionInput);
  const firstBundle = await getResumeVersion("job-1", first.version);
  const second = await createResumeVersion(validVersionInput);
  expect(first.version).toBe("v001");
  expect(second.version).toBe("v002");
  expect(await getResumeVersion("job-1", first.version)).toEqual(firstBundle);
});
```

- [ ] **Step 2: Write locked-fact preservation tests**

```ts
it("preserves identity, employer, title, date, degree, and certification facts", () => {
  const result = assembleResumeDocument({
    baseResume: designResume,
    draft: approvedDraft,
    selection: approvedSelection,
  });
  expect(getLockedFacts(result)).toEqual(getLockedFacts(designResume));
});
```

- [ ] **Step 3: Run the tests and confirm failure**

Run: `npm --workspace orchestrator run test:run -- src/server/services/personal-documents/resume-versions.test.ts src/server/services/personal-documents/resume-document.test.ts`

Expected: FAIL because versioning and assembly are missing.

- [ ] **Step 4: Implement immutable version allocation**

Under a per-job lock, read `current.json`, allocate the next zero-padded version, validate inputs, atomically write the draft and claim report, then update `current.json`. Public manifests contain relative artifact names only.

- [ ] **Step 5: Implement constrained Reactive Resume assembly**

Clone the current Design Resume. Apply only operations whose claim IDs passed validation. Permit summary replacement, evidence-backed bullet selection and order, skill grouping, education compression, certification order, and project visibility. Reject changes to contact details, employer names, official titles, dates, degrees, certification names, and evidence context. Reuse the existing Reactive Resume v5 model and renderer normalization.

- [ ] **Step 6: Run tests and commit**

Run: `npm --workspace orchestrator run test:run -- src/server/services/personal-documents/resume-versions.test.ts src/server/services/personal-documents/resume-document.test.ts`

Expected: PASS.

```bash
git add orchestrator/src/server/services/personal-documents/resume-versions.ts orchestrator/src/server/services/personal-documents/resume-versions.test.ts orchestrator/src/server/services/personal-documents/resume-document.ts orchestrator/src/server/services/personal-documents/resume-document.test.ts
git commit -m "feat: create evidence-bound resume versions"
```

---

### Task 8: Rendering, QA, and Approval Freeze

**Files:**
- Create: `orchestrator/src/server/services/personal-documents/resume-render.ts`
- Modify: `orchestrator/src/server/services/resume-renderer/index.ts`
- Modify: `orchestrator/src/server/services/resume-renderer/latex.ts`
- Modify: `orchestrator/src/server/services/resume-renderer/typst.ts`
- Test: `orchestrator/src/server/services/personal-documents/resume-render.test.ts`

**Interfaces:**
- Consumes: a validated resume version and existing `renderResumePdf(args)`.
- Produces: `renderResumeVersion(jobId, version): Promise<ResumeQaReport>`, `recordVisualInspection(input)`, and `approveResumeVersion(input): Promise<ResumeApproval>`.

- [ ] **Step 1: Write render and approval gate tests**

```ts
it("does not render a failed claim report", async () => {
  await expect(renderResumeVersion("job-1", "v001"))
    .rejects.toThrow("Resume claims must pass before rendering");
  expect(renderResumePdf).not.toHaveBeenCalled();
});

it("does not approve a version before visual QA passes", async () => {
  await expect(approveResumeVersion({
    jobId: "job-1",
    version: "v001",
    approvedBy: "user",
    confirmedByUser: true,
  })).rejects.toThrow("Resume QA must pass before approval");
});
```

- [ ] **Step 2: Run the test and confirm failure**

Run: `npm --workspace orchestrator run test:run -- src/server/services/personal-documents/resume-render.test.ts`

Expected: FAIL because render and approval gates are missing.

- [ ] **Step 3: Preserve renderer source beside the PDF**

Extend `renderResumePdf`, `renderLatexPdf`, and `renderTypstPdf` with an optional `sourceOutputPath`. The personal-document service resolves both output paths inside the immutable version directory. After generating the LaTeX or Typst source and before deleting the renderer temporary directory, copy it to `source.tex` or `source.typ`. Existing callers omit the option and retain current behavior.

- [ ] **Step 4: Persist structural QA**

Verify a non-empty PDF, selectable extracted text, expected headings and locked identity text, absence of configured forbidden phrases, and the assembled-resume hash. Store results and PDF SHA-256 in `qa.json`. Structural success sets visual status to `pending`, not overall success.

- [ ] **Step 5: Freeze exact approved versions**

Require the exact job ID, snapshot hash, version, passing claim report, passing structural QA, passed visual inspection with inspector and time, and matching final PDF hash. Write `approval.json` once with `approvedBy: "user"`; reject replacement or mutation.

- [ ] **Step 6: Run renderer regressions and commit**

Run: `npm --workspace orchestrator run test:run -- src/server/services/personal-documents/resume-render.test.ts src/server/services/resume-renderer/latex.test.ts src/server/services/resume-renderer/typst.test.ts`

Expected: PASS without changing current PDF behavior.

```bash
git add orchestrator/src/server/services/personal-documents/resume-render.ts orchestrator/src/server/services/personal-documents/resume-render.test.ts orchestrator/src/server/services/resume-renderer/index.ts orchestrator/src/server/services/resume-renderer/latex.ts orchestrator/src/server/services/resume-renderer/typst.ts
git commit -m "feat: gate resume rendering and approval"
```

---

### Task 9: Narrow API, Documentation, and Acceptance Verification

**Files:**
- Create: `orchestrator/src/server/services/personal-documents/index.ts`
- Create: `orchestrator/src/server/api/routes/personal-documents.ts`
- Create: `orchestrator/src/server/api/routes/personal-documents.test.ts`
- Modify: `orchestrator/src/server/api/routes.ts`
- Create: `orchestrator/src/server/services/personal-documents/acceptance.test.ts`
- Create: `docs-site/docs/features/personal-document-workspace.md`
- Modify: `docs-site/sidebars.ts`

**Interfaces:**
- Consumes: Tasks 2-8 services and existing API response/error/request-context helpers.
- Produces: tenant-scoped `/api/personal-documents/*` endpoints and a documented agent-first workflow.

- [ ] **Step 1: Write API contract and privacy tests**

```ts
it("creates a request-scoped snapshot", async () => {
  const response = await api.post(
    "/api/personal-documents/jobs/b4d2e820-c3db-47f1-88f1-f03e20ec33d9/snapshot",
  );
  expect(response.status).toBe(201);
  expect(response.body).toMatchObject({
    ok: true,
    data: { jobId: "b4d2e820-c3db-47f1-88f1-f03e20ec33d9" },
    meta: { requestId: expect.any(String) },
  });
});

it("does not expose absolute storage paths", async () => {
  const response = await api.get("/api/personal-documents/evidence");
  expect(JSON.stringify(response.body)).not.toContain(tempDir);
  expect(response.headers["x-request-id"]).toBeTruthy();
});
```

- [ ] **Step 2: Run the API tests and confirm route-not-found failure**

Run: `npm --workspace orchestrator run test:run -- src/server/api/routes/personal-documents.test.ts`

Expected: FAIL because the router is not registered.

- [ ] **Step 3: Implement and register exact endpoints**

```text
GET  /api/personal-documents/evidence
PUT  /api/personal-documents/evidence
GET  /api/personal-documents/tracks
POST /api/personal-documents/jobs/:jobId/snapshot
PUT  /api/personal-documents/jobs/:jobId/match-analysis
PUT  /api/personal-documents/jobs/:jobId/evidence-selection
POST /api/personal-documents/jobs/:jobId/resumes
GET  /api/personal-documents/jobs/:jobId/resumes/:version
POST /api/personal-documents/jobs/:jobId/resumes/:version/validate
POST /api/personal-documents/jobs/:jobId/resumes/:version/render
POST /api/personal-documents/jobs/:jobId/resumes/:version/visual-inspection
POST /api/personal-documents/jobs/:jobId/resumes/:version/approve
```

Use strict bodies, shared response helpers, request IDs, sanitized structured logs, and existing request context. Evidence replacement, evidence selection, visual inspection, and approval require `confirmedByUser: true`; the assertion authorizes only that exact operation.

- [ ] **Step 4: Add negative and tenant-isolation tests**

Verify 400 for malformed bodies, 401/403 according to app mode, 404 for unavailable jobs and versions, 409 for immutable-version conflicts, 422 for claim or QA failure, and no cross-tenant artifact visibility.

- [ ] **Step 5: Add three-track acceptance tests**

Use synthetic, non-personal fixtures for Data/BI, Business Systems/Automation, and Operations/Project. Each includes direct, adjacent, and portfolio evidence plus one explicit forbidden upgrade.

```ts
it.each([
  ["data-bi", "Analyst Data Reporting"],
  ["business-systems-automation", "ERP Analyst"],
  ["operations-project", "Logistics Analyst"],
])("builds a validated %s resume workspace", async (trackId, title) => {
  const result = await runResumeWorkspaceFixture({ trackId, title });
  expect(result.claimReport.status).toBe("passed");
  expect(result.qa.structuralStatus).toBe("passed");
  expect(result.version).toBe("v001");
});
```

- [ ] **Step 6: Write user-facing documentation**

Use the required sections: What it is, Why it exists, How to use it, Common problems, and Related pages. Document source priority, evidence classes, three tracks, stretch overlays, API-only milestone, visual inspection, approval boundary, and the absence of JobOps mutation or application submission.

- [ ] **Step 7: Run focused API and acceptance tests**

Run: `npm --workspace orchestrator run test:run -- src/server/api/routes/personal-documents.test.ts src/server/services/personal-documents/acceptance.test.ts src/server/api/routes/api-contract.test.ts src/server/api/routes/tenant-isolation.test.ts`

Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add orchestrator/src/server/services/personal-documents/index.ts orchestrator/src/server/api/routes/personal-documents.ts orchestrator/src/server/api/routes/personal-documents.test.ts orchestrator/src/server/api/routes.ts orchestrator/src/server/services/personal-documents/acceptance.test.ts docs-site/docs/features/personal-document-workspace.md docs-site/sidebars.ts
git commit -m "feat: expose personal resume workspace"
```

---

### Task 10: CI-Parity and Real-Job Dry Runs

**Files:**
- No planned source changes. Any discovered failure is corrected in the owning task before repeating this task.

**Interfaces:**
- Consumes: the completed Phase 1-2 domain.
- Produces: verified build/test results and three user-reviewed dry-run workspaces without JobOps mutation.

- [ ] **Step 1: Run repository checks**

```bash
./orchestrator/node_modules/.bin/biome ci .
npm run check:types:shared
npm --workspace orchestrator run check:types
npm --workspace gradcracker-extractor run check:types
npm --workspace ukvisajobs-extractor run check:types
npm --workspace orchestrator run build:client
npm --workspace orchestrator run test:run
```

Expected: every command exits 0. If `better-sqlite3` reports a Node ABI mismatch, run `npm --workspace orchestrator rebuild better-sqlite3` under Node 22 and repeat the failed checks.

- [ ] **Step 2: Import the human-reviewed evidence seed**

Use the Design Resume plus the approved Cigna Technology Apprentice, St. Charles County Program Analyst, and Sensient Logistics Analyst source files. Before submitting the evidence catalog, show source hashes, evidence counts, unresolved conflicts, and forbidden upgrades to the user. Submit only after exact approval.

- [ ] **Step 3: Dry-run one existing job per core track**

```text
Data/BI: 22652ce4-07c2-4466-baf3-296aa712e5b0
Business Systems/Automation: 477ba09e-2c9b-426c-b7f7-97e4b0d25c1c
Operations/Project: 5f5812a9-1d68-424d-832b-e6886a7bf290
```

For each job, create a snapshot, approve evidence selection, draft through `resume-creator`, validate claims, assemble and render the resume, inspect every page, record visual QA, and present the exact version for user review. Do not approve on the user's behalf.

- [ ] **Step 4: Prove JobOps remained unchanged**

Compare each job's status, outcome, tailored fields, PDF path, and updated timestamp before and after the dry run. Expected: identical values. Confirm new runtime state exists only under the active private scope's `personal-documents` directory.

- [ ] **Step 5: Record the verification handoff**

Report exact commands, exit codes, dry-run version IDs, artifact hashes, visual-inspection results, unresolved evidence issues, and confirmation that no JobOps job mutation or application action occurred. Do not create an empty commit.
