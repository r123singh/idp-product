Here are the use-cases mapped to the endpoints actually implemented in `job_api`, ordered happy-path first.

> **Ready-to-run requests:** `use-cases.ps1` contains plain copy/paste PowerShell commands (one block per use case). Start the API (`..\.venv\Scripts\python.exe local_runner.py` from `job_api/`), then paste a command, replacing `<JOB_ID>` / `<DOC_ID>` / `<WEBHOOK_ID>` with values from earlier responses.

## A. Happy path — Share transfer processed successfully

| # | Use-case | Endpoint | Notes |
|---|----------|----------|-------|
| 1 | **Submit a job** with the share-transfer certificate + each party's passport | `POST /v1/jobs` → `202` | JSON body: `jobType: shareTransfer`, `files: [ ...paths/URLs... ]`. Returns `jobId` + `Location` header. |
| 2 | **Poll job status** until terminal | `GET /v1/jobs/{jobId}` | Returns `status`, `progress`, per-doc summaries, **and `workflowValidation`** (the cross-document result). |
| 2b | **Open job detail screen (one call)** | `GET /v1/jobs/{jobId}/detail` | Full UI payload: overview, workflow, correction plan, documents with extraction + validation, `issues`, `highlights`, `timeline`. |
| 3 | **Read the cross-document result** | (same `GET /v1/jobs/{jobId}`) | `workflowValidation.status = PASSED`, all parties have passports, nothing missing. |
| 4 | **List documents** in the job | `GET /v1/jobs/{jobId}/documents` | Each document's lifecycle: `QUEUED → EXTRACTING → VALIDATING → COMPLETED`. |
| 5 | **Get one document's full transaction** | `GET /v1/jobs/{jobId}/documents/{documentId}` | `source`, detected `documentType`, extraction + validation blocks. |
| 6 | **Get extracted data** for a document | `GET /v1/jobs/{jobId}/documents/{documentId}/extraction` | The structured `extractedData` from the upstream parser. |
| 7 | **Get per-document validation results** | `GET /v1/jobs/{jobId}/documents/{documentId}/validations` | The single-document rule outcomes (e.g. `PASSPORT_NOT_EXPIRED`). |
| — | **Job detail screen (all of the above in one call)** | `GET /v1/jobs/{jobId}/detail` | See [Job detail for UI](#job-detail-for-ui) below. |

### Happy-path request / response examples

**1. Create the job** — `POST /v1/jobs`

Request:

```json
{
  "jobType": "shareTransfer",
  "files": [
    "https://example.com/files/sample.pdf",
    "https://example.com/files/sample.pdf",
    "https://example.com/files/sample.pdf",
  ],
  "priority": "NORMAL",
  "metadata": { "requestId": "REQ-2026-001" }
}
```

Response `202 Accepted` (header `Location: /v1/jobs/job_8f3a2b1c`). At creation the documents are still `QUEUED`, so `workflowValidation.status` is `INCOMPLETE`:

```json
{
  "status": { "code": 202, "message": "Job accepted for processing" },
  "job": {
    "jobId": "job_8f3a2b1c",
    "jobType": "shareTransfer",
    "status": "ACCEPTED",
    "totalDocuments": 3,
    "completedDocuments": 0,
    "failedDocuments": 0,
    "progress": { "percentage": 0, "currentStage": "QUEUED" },
    "workflowValidation": { "status": "INCOMPLETE", "summary": { "totalChecks": 1, "passed": 0, "failed": 0, "warnings": 0 }, "expectedDocuments": 1, "presentDocuments": 0, "missingDocuments": [], "checks": [] },
    "documents": [
      { "documentId": "doc_4ce7fdc342", "fileName": "share_transfer_happy_case.pdf", "status": "QUEUED" },
      { "documentId": "doc_a1b2c3d4e5", "fileName": "Mohammad_passport_Active.jpeg", "status": "QUEUED" },
      { "documentId": "doc_b12db999da", "fileName": "chithu_passport_2025.pdf", "status": "QUEUED" }
    ],
    "createdAt": "2026-06-02T06:38:03Z",
    "metadata": { "requestId": "REQ-2026-001" }
  }
}
```

**2–3. Poll status + read cross-document result** — `GET /v1/jobs/job_8f3a2b1c`

Response `200` once terminal — `workflowValidation.status = PASSED` (certificate present, both parties matched to a passport):

```json
{
  "status": { "code": 200, "message": "Success" },
  "job": {
    "jobId": "job_8f3a2b1c",
    "jobType": "shareTransfer",
    "status": "COMPLETED",
    "totalDocuments": 3,
    "completedDocuments": 3,
    "failedDocuments": 0,
    "progress": { "percentage": 100, "currentStage": "DONE" },
    "workflowValidation": {
      "status": "PASSED",
      "summary": { "totalChecks": 3, "passed": 3, "failed": 0, "warnings": 0 },
      "expectedDocuments": 3,
      "presentDocuments": 3,
      "missingDocuments": [],
      "checks": [
        { "checkId": "PRIMARY_DOCUMENT_PRESENT", "name": "Share transfer certificate present", "passed": true, "severity": "error", "documentType": "shareTransferCertificate" },
        { "checkId": "PARTY_PASSPORT:transferor:onur_omer_emre", "name": "Passport per party", "passed": true, "severity": "error", "subject": "Onur Omer Emre", "documentType": "passport" },
        { "checkId": "PARTY_PASSPORT:transferee:onur_yegenoglu", "name": "Passport per party", "passed": true, "severity": "error", "subject": "Onur Yegenoglu", "documentType": "passport" }
      ]
    },
    "createdAt": "2026-06-02T06:38:03Z",
    "completedAt": "2026-06-02T06:38:29Z"
  }
}
```

**4. List documents** — `GET /v1/jobs/job_8f3a2b1c/documents`

```json
{
  "status": { "code": 200, "message": "Success" },
  "documents": [
    { "documentId": "doc_4ce7fdc342", "fileName": "share_transfer_happy_case.pdf", "documentType": "shareTransferCertificate", "status": "COMPLETED", "extractionStatus": "COMPLETED", "validationStatus": "COMPLETED" },
    { "documentId": "doc_a1b2c3d4e5", "fileName": "Mohammad_passport_Active.jpeg", "documentType": "passport", "status": "COMPLETED", "extractionStatus": "COMPLETED", "validationStatus": "COMPLETED" },
    { "documentId": "doc_b12db999da", "fileName": "chithu_passport_2025.pdf", "documentType": "passport", "status": "COMPLETED", "extractionStatus": "COMPLETED", "validationStatus": "COMPLETED" }
  ]
}
```

**5. Get one document transaction** — `GET /v1/jobs/job_8f3a2b1c/documents/doc_4ce7fdc342`

```json
{
  "status": { "code": 200, "message": "Success" },
  "document": {
    "documentId": "doc_4ce7fdc342",
    "jobId": "job_8f3a2b1c",
    "fileName": "share_transfer_happy_case.pdf",
    "source": "https://example.com/files/sample.pdf",
    "documentType": "shareTransferCertificate",
    "status": "COMPLETED",
    "extraction": { "status": "COMPLETED", "startedAt": "2026-06-02T06:38:05Z", "completedAt": "2026-06-02T06:38:21Z" },
    "validation": { "status": "COMPLETED", "summary": { "totalRules": 7, "passed": 7, "failed": 0, "warnings": 0 } },
    "createdAt": "2026-06-02T06:38:03Z"
  }
}
```

**6. Get extracted data** — `GET /v1/jobs/job_8f3a2b1c/documents/doc_4ce7fdc342/extraction`

```json
{
  "status": { "code": 200, "message": "Success" },
  "extraction": {
    "status": "COMPLETED",
    "data": {
      "extractedData": {
        "companyInfo": { "companyName": "EMR ENERGY Registry", "entityNumber": "ENT194063", "documentTitle": "RESOLUTIONS OF SHAREHOLDER OF EMR ENERGY Registry (SHARE TRANSFER FORM)" },
        "currentShareStructure": { "transferors": [ { "name": "Chitharanj Kachappilly Verghese", "numberOfShares": 50 } ] },
        "transferDetails": { "transaction": [ { "sellerName": "Chitharanj Kachappilly Verghese", "buyerName": "Mohammad Milhem", "numberOfSharesTransferred": 25 } ] },
        "buyerAndResolution": { "transferees": [ { "name": "Mohammad Milhem", "numberOfSharesAfterTransfer": 25 } ] },
        "signatures": { "transferDate": "2025-11-04" }
      },
      "missingFields": [],
      "statusIndicator": "success"
    }
  }
}
```

**7. Get per-document validation results** — `GET /v1/jobs/job_8f3a2b1c/documents/doc_4ce7fdc342/validations`

```json
{
  "status": { "code": 200, "message": "Success" },
  "validation": {
    "status": "COMPLETED",
    "summary": { "totalRules": 7, "passed": 7, "failed": 0, "warnings": 0 },
    "results": [
      { "ruleId": "TRANSFER_DATE_PAST_OR_TODAY", "ruleName": "TRANSFER_DATE_PAST_OR_TODAY", "passed": true },
      { "ruleId": "ENTITY_NUMBER_PATTERN", "ruleName": "ENTITY_NUMBER_PATTERN", "passed": true },
      { "ruleId": "DOCUMENT_TITLE_CHECK", "ruleName": "DOCUMENT_TITLE_CHECK", "passed": true }
    ]
  }
}
```

## Job detail for UI

`GET /v1/jobs/{jobId}/detail` returns one aggregated payload for a job review screen. Use `GET /v1/jobs/{jobId}` for lightweight polling; open **detail** when rendering the full view.

| Section | Use on UI |
|---------|-----------|
| `detail.overview` | Header: status, progress, `overallHealth` (`HEALTHY` / `ATTENTION` / `BLOCKED` / `PROCESSING`), `canCancel`, `canRetry` |
| **`detail.presentation`** | **Primary case layout** — main certificate, parties, linked passports (see below) |
| `detail.highlights` | Summary chips (documents completed, workflow status, pending corrections) |
| `detail.workflowValidation` | Cross-document checklist panel |
| `detail.correctionPlan` | Action list for retry (`add` / `replace`) |
| `detail.documents[]` | Full technical transactions (extraction + validation inline) for expand/drill-down |
| `detail.issues` | Unified issues feed (`workflow`, `validation`, `document`, `correction`) |
| `detail.timeline` | Activity / stage timeline per job and document |
| `detail.links` | Relative paths for drill-down APIs |

### `presentation` (share transfer)

| Field | Use on UI |
|-------|-----------|
| `partyCount`, `transferorCount`, `transfereeCount` | Party summary header |
| `mainDocument` | Certificate card: company, Registry number, transfer date, transactions; `document.links` → extraction/validations |
| `parties[]` | One row/card per party from the certificate |
| `parties[].certificateDetails` | Shareholding, transactions, signatures for that party |
| `parties[].personalInfo` | Merged identity fields from linked documents |
| `parties[].documents[]` | Passport / Emirates ID linked by name, each with `links` and `personalInfo` |
| `parties[].workflowStatus` | Whether required passport check passed for this party |
| `supportingDocuments.unassigned` | Supporting docs not matched to any party |

Example (abbreviated):

```json
{
  "status": { "code": 200, "message": "Success" },
  "detail": {
    "overview": {
      "jobId": "job_8f3a2b1c",
      "status": "COMPLETED",
      "overallHealth": "ATTENTION",
      "canRetry": true,
      "completedDocuments": 2,
      "totalDocuments": 2
    },
    "presentation": {
      "viewType": "shareTransfer",
      "partyCount": 2,
      "transferorCount": 1,
      "transfereeCount": 1,
      "mainDocument": {
        "displayTitle": "Share transfer certificate",
        "companyName": "EMR ENERGY Registry",
        "entityNumber": "ENT194063",
        "transferDate": "2025-11-04",
        "document": { "documentId": "doc_…", "links": { "extraction": "/v1/jobs/job_…/documents/doc_…/extraction" } }
      },
      "parties": [
        {
          "partyId": "transferor:onur_omer_emre",
          "role": "transferor",
          "name": "Onur Omer Emre",
          "certificateDetails": { "shareholding": [{ "numberOfShares": 50 }] },
          "personalInfo": { "fullName": "ONUR OMER EMRE" },
          "documents": [{ "displayTitle": "Passport — ONUR OMER EMRE", "personalInfo": { "fullName": "ONUR OMER EMRE" } }]
        }
      ],
      "supportingDocuments": { "totalCount": 1, "countByType": { "passport": 1 }, "unassigned": [] }
    },
    "workflowValidation": { "status": "FAILED", "missingDocuments": ["Passport for …"] },
    "correctionPlan": { "pendingCount": 1, "actions": [{ "action": "add", "subject": "…" }] },
    "documents": [
      {
        "displayTitle": "Share transfer certificate",
        "failedValidationCount": 0,
        "document": { "documentId": "doc_…", "extraction": { "data": { "extractedData": {} } }, "validation": {} }
      }
    ],
    "issues": [{ "category": "workflow", "severity": "error", "title": "Passport per party" }],
    "highlights": [{ "key": "workflow", "label": "Workflow validation", "value": "FAILED", "severity": "error" }],
    "timeline": [{ "kind": "job", "stage": "created", "title": "Job created", "status": "completed" }],
    "links": { "job": "/v1/jobs/job_8f3a2b1c", "documents": "/v1/jobs/job_8f3a2b1c/documents", "detail": "/v1/jobs/job_8f3a2b1c/detail" }
  }
}
```

## A2. Correction flow — invalid share transfer document, then corrected

This is the happy path's sibling: the certificate is recognised and extracts fine, but it **fails its own validation rules** (e.g. the transfer date is in the future, or the registry entity number is malformed). The operator spots it, fixes the file, and resubmits.

> **How correction works — same job, via retry.** `POST /v1/jobs/{jobId}/retry` keeps the **same `jobId`** and re-evaluates `workflowValidation` afterwards. It supports three actions (any combination):
> - `documentIds` (or empty body) — **re-run** FAILED documents against the same `source` (transient/technical failures, e.g. the DNS/timeout case in section C).
> - `replaceDocuments` — **replace** an invalid document's `source` with a corrected one and re-process it.
> - `addFiles` — **add** a missing document (closes `workflowValidation` gaps).
>
> The job must be in a terminal state (you can't retry one that's still processing). The per-job limit of 20 documents still applies.

> **Where invalidity shows up.** A failed *content* rule does **not** fail the document — it still ends `COMPLETED`. The signal is `validation.summary.failed > 0` (and the failed rule in `…/validations`). Cross-document *completeness* (missing certificate / passports) is separate and shows in `workflowValidation`.

**Step 1 — Submit the job with the (invalid) certificate + passports** — `POST /v1/jobs`

```json
{
  "jobType": "shareTransfer",
  "files": [
    "https://example.com/files/sample.pdf",
    "https://example.com/files/sample.pdf",
    "https://example.com/files/sample.pdf"
  ],
  "metadata": { "requestId": "REQ-2026-002" }
}
```

→ `202` with a new `jobId` (e.g. `job_inv12345`).

**Step 2 — Poll the job** — `GET /v1/jobs/job_inv12345`

The job finishes `COMPLETED` and `workflowValidation` is `PASSED` (the certificate *is* present and both parties matched). The certificate's content problem is **not** visible here — you must drill into the document. (Tip: `completedDocuments: 3` but you should always check each document's `validation.summary`.)

**Step 3 — Check the certificate's validations** — `GET /v1/jobs/job_inv12345/documents/doc_4ce7fdc342/validations`

```json
{
  "status": { "code": 200, "message": "Success" },
  "validation": {
    "status": "COMPLETED",
    "summary": { "totalRules": 7, "passed": 6, "failed": 1, "warnings": 0 },
    "results": [
      {
        "ruleId": "TRANSFER_DATE_PAST_OR_TODAY",
        "ruleName": "TRANSFER_DATE_PAST_OR_TODAY",
        "passed": false,
        "message": "Transfer date must be today or earlier."
      },
      { "ruleId": "ENTITY_NUMBER_PATTERN", "ruleName": "ENTITY_NUMBER_PATTERN", "passed": true },
      { "ruleId": "DOCUMENT_TITLE_CHECK", "ruleName": "DOCUMENT_TITLE_CHECK", "passed": true }
    ]
  }
}
```

`summary.failed = 1` → the share transfer document is invalid. The operator corrects the source document offline.

**Step 4 — Replace the invalid certificate in the SAME job** — `POST /v1/jobs/job_inv12345/retry`

```json
{
  "replaceDocuments": [
    {
      "documentId": "doc_4ce7fdc342",
      "source": "https://example.com/files/sample.pdf"
    }
  ]
}
```

Response `202` — the job is re-processing the replaced document and keeps its `jobId`:

```json
{
  "status": { "code": 202, "message": "Retry initiated: 0 re-run, 0 added, 1 replaced" },
  "retriedDocuments": [],
  "replacedDocuments": [ "doc_4ce7fdc342" ],
  "job": { "jobId": "job_inv12345", "status": "PROCESSING", "totalDocuments": 3 }
}
```

**Step 5 — Re-poll the same job + recheck the certificate** — `GET /v1/jobs/job_inv12345` then `…/documents/doc_4ce7fdc342/validations`

The replaced certificate now passes (`summary.failed = 0`) and the job is clean — same `jobId` throughout:

```json
{
  "job": {
    "jobId": "job_inv12345",
    "status": "COMPLETED",
    "workflowValidation": { "status": "PASSED", "expectedDocuments": 3, "presentDocuments": 3, "missingDocuments": [] }
  }
}
```

**Variant A — a missing document (incomplete business case).** If `workflowValidation` reports a missing party passport, **add** it to the same job instead of replacing:

`POST /v1/jobs/job_inv12345/retry`
```json
{ "addFiles": [ "https://example.com/files/sample.pdf" ] }
```
→ `202` (`addedDocuments: ["doc_99aa88bb77"]`); re-poll → `workflowValidation.status = PASSED`.

**Variant B — the wrong file entirely.** If the uploaded "certificate" isn't recognised as a `shareTransferCertificate`, `workflowValidation` fails at the job level (no need to drill into a document):

```json
"workflowValidation": {
  "status": "FAILED",
  "expectedDocuments": 1,
  "presentDocuments": 0,
  "missingDocuments": [ "Share transfer certificate has not been uploaded yet." ],
  "checks": [ { "checkId": "PRIMARY_DOCUMENT_PRESENT", "name": "Share transfer certificate present", "passed": false, "severity": "error", "message": "Share transfer certificate has not been uploaded yet." } ]
}
```

Recovery is the same `retry` call — `replaceDocuments` (if a wrong file was uploaded as the certificate) or `addFiles` (if it was never uploaded).

## Complex cases — invalid file + missing file workflow handling

Real run (`job_4d8a445477`, `REQ-2026-003`): the job was created with the **share-transfer certificate** and **only Mohammad’s passport** (transferor **Chitharanj**’s passport was never submitted). After the first pass:

- **Workflow:** transferor passport for **Chitharanj Kachappilly** is missing.
- **Per-document:** Mohammad’s passport slot is `COMPLETED` but **invalid** (five failed MRZ / name rules).
- **`correctionPlan`** (from first-pass extraction + workflow, no storage API) lists exactly what to fix:
  - **replace** `doc_531b3c7408` — bad passport file already in the job.
  - **add** passport for **Chitharanj Kachappilly** (transferor).

Storage/upload stays outside this API and only returns URLs; use **`correctionPlan`** to know what to upload, then **`sources`** on retry so each URL is extracted once and matched to the right action.

### Step 1 — Create job (certificate + one passport; transferor passport omitted)

`POST /v1/jobs`

Request:

```json
{
  "jobType": "shareTransfer",
  "files": [
    "https://example.com/files/sample.pdf",
    "https://example.com/files/sample.pdf"
  ],
  "metadata": { "requestId": "REQ-2026-003" }
}
```

Response `202`:

```json
{
  "status": { "code": 202, "message": "Job accepted for processing" },
  "job": {
    "jobId": "job_4d8a445477",
    "jobType": "shareTransfer",
    "status": "ACCEPTED",
    "totalDocuments": 2,
    "completedDocuments": 0,
    "failedDocuments": 0,
    "progress": { "percentage": 0, "currentStage": "QUEUED" },
    "documents": [
      {
        "documentId": "doc_80878e1eb5",
        "fileName": "share_transfer_happy_case.pdf",
        "status": "QUEUED"
      },
      {
        "documentId": "doc_531b3c7408",
        "fileName": "Mohammad+passport+-+Active.jpeg",
        "status": "QUEUED"
      }
    ],
    "createdAt": "2026-06-04T08:54:29Z",
    "metadata": { "requestId": "REQ-2026-003" }
  }
}
```

### Step 2 — Poll job; read `workflowValidation` and `correctionPlan`

`GET /v1/jobs/job_4d8a445477`

Response `200` (after processing completes — two problems at once):

```json
{
  "status": { "code": 200, "message": "Success" },
  "job": {
    "jobId": "job_4d8a445477",
    "jobType": "shareTransfer",
    "status": "COMPLETED",
    "totalDocuments": 2,
    "completedDocuments": 2,
    "failedDocuments": 0,
    "progress": { "percentage": 100, "currentStage": "DONE" },
    "workflowValidation": {
      "status": "FAILED",
      "summary": { "totalChecks": 3, "passed": 2, "failed": 1, "warnings": 0 },
      "expectedDocuments": 3,
      "presentDocuments": 2,
      "missingDocuments": [
        "Passport for Chitharanj Kachappilly (transferor) has not been uploaded."
      ],
      "checks": [
        {
          "checkId": "PRIMARY_DOCUMENT_PRESENT",
          "name": "Share transfer certificate present",
          "passed": true,
          "severity": "error",
          "documentType": "shareTransferCertificate"
        },
        {
          "checkId": "PARTY_PASSPORT:transferor:chitharanj_kachappilly",
          "name": "Passport per party",
          "passed": false,
          "severity": "error",
          "subject": "Chitharanj Kachappilly",
          "documentType": "passport",
          "message": "Passport for Chitharanj Kachappilly (transferor) has not been uploaded."
        },
        {
          "checkId": "PARTY_PASSPORT:transferee:mohammad_milhem",
          "name": "Passport per party",
          "passed": true,
          "severity": "error",
          "subject": "Mohammad Milhem",
          "documentType": "passport"
        }
      ]
    },
    "correctionPlan": {
      "pendingCount": 2,
      "actions": [
        {
          "actionId": "replace:doc_531b3c7408",
          "action": "replace",
          "documentId": "doc_531b3c7408",
          "documentType": "passport",
          "subject": "Mohammad Milhem",
          "role": "transferee",
          "reason": "validation_failed",
          "message": "Document has 5 failed validation rule(s).",
          "requiresNewSource": true,
          "failedRuleIds": [
            "MRZ_LINE1_FORMAT",
            "MRZ_LINE2_FORMAT",
            "MRZ_LINE2_COMPOSITE_CHECK_DIGIT",
            "PASSPORT_FULL_NAME_MRZ_CHECK",
            "PASSPORT_SURNAME_MRZ_CHECK"
          ]
        },
        {
          "actionId": "PARTY_PASSPORT:transferor:chitharanj_kachappilly",
          "action": "add",
          "documentType": "passport",
          "subject": "Chitharanj Kachappilly",
          "role": "transferor",
          "reason": "missing_party_document",
          "message": "Passport for Chitharanj Kachappilly (transferor) has not been uploaded.",
          "requiresNewSource": true
        }
      ]
    },
    "documents": [
      {
        "documentId": "doc_80878e1eb5",
        "fileName": "share_transfer_happy_case.pdf",
        "documentType": "shareTransferCertificate",
        "status": "COMPLETED",
        "extractionStatus": "COMPLETED",
        "validationStatus": "COMPLETED"
      },
      {
        "documentId": "doc_531b3c7408",
        "fileName": "Mohammad+passport+-+Active.jpeg",
        "documentType": "passport",
        "status": "COMPLETED",
        "extractionStatus": "COMPLETED",
        "validationStatus": "COMPLETED"
      }
    ],
    "createdAt": "2026-06-04T08:54:29Z",
    "updatedAt": "2026-06-04T09:01:03Z",
    "completedAt": "2026-06-04T09:01:03Z",
    "metadata": { "requestId": "REQ-2026-003" }
  }
}
```

**How to read this:** `job.status` is `COMPLETED` but **`workflowValidation.status` is `FAILED`** — completeness failed (missing transferor passport). Mohammad’s file is present for workflow (transferee match) but **`correctionPlan`** still flags that same `documentId` for **replace** because per-document validation failed. The **replace** action includes **`subject`** / **`role`** from first-pass extraction so retry URL matching does not overwrite Mohammad’s slot with another party’s passport. You need **two** fixes, not one.

### Step 3 — Confirm invalid passport (per-document rules)

`GET /v1/jobs/job_4d8a445477/documents/doc_531b3c7408/validations`

Response `200` (abbreviated — `summary.failed = 5`):

```json
{
  "status": { "code": 200, "message": "Success" },
  "validation": {
    "status": "COMPLETED",
    "summary": { "totalRules": 12, "passed": 7, "failed": 5, "warnings": 0 },
    "results": [
      {
        "ruleId": "MRZ_LINE1_FORMAT",
        "ruleName": "MRZ_LINE1_FORMAT",
        "passed": false,
        "message": "MRZ Line 1 must be 44 characters..."
      },
      {
        "ruleId": "MRZ_LINE2_FORMAT",
        "ruleName": "MRZ_LINE2_FORMAT",
        "passed": false,
        "message": "MRZ Line 2 must be 44 characters..."
      },
      {
        "ruleId": "MRZ_LINE2_COMPOSITE_CHECK_DIGIT",
        "ruleName": "MRZ_LINE2_COMPOSITE_CHECK_DIGIT",
        "passed": false
      },
      {
        "ruleId": "PASSPORT_FULL_NAME_MRZ_CHECK",
        "ruleName": "PASSPORT_FULL_NAME_MRZ_CHECK",
        "passed": false
      },
      {
        "ruleId": "PASSPORT_SURNAME_MRZ_CHECK",
        "ruleName": "PASSPORT_SURNAME_MRZ_CHECK",
        "passed": false
      }
    ]
  }
}
```

Certificate validations on `doc_80878e1eb5` can be checked the same way; in this run the certificate passed.

### Step 4 — Correct on the same job (replace invalid + add missing)

Upload corrected files elsewhere (storage returns **URLs only**). Then either map explicitly or use auto-assignment.

**Option A — `sources` (recommended when you only have URLs)**

Each URL is extracted once; assignment prefers **add** slots whose **`subject`** matches the file’s name (score 100) over a generic **replace** on the same `documentType` (score 10). A **replace** slot only wins when the URL’s name matches that slot’s **`subject`** (e.g. corrected Mohammad file → `replace:doc_531b3c7408`), so Chitharanj’s passport is **added** as a third document instead of replacing Mohammad’s slot.

`POST /v1/jobs/job_4d8a445477/retry`

Request:

```json
{
  "sources": [
    "https://example.com/files/sample.pdf",
    "https://example.com/files/sample.pdf"
  ]
}
```

Response `202` (each URL is extracted; matched to `correctionPlan` actions):

```json
{
  "status": {
    "code": 202,
    "message": "Retry initiated: 0 re-run, 1 added, 1 replaced"
  },
  "retriedDocuments": [],
  "addedDocuments": ["doc_xxxxxxxxxx"],
  "replacedDocuments": ["doc_531b3c7408"],
  "sourceAssignments": [
    {
      "source": "https://example.com/files/sample.pdf",
      "actionId": "replace:doc_531b3c7408",
      "action": "replace",
      "documentType": "passport"
    },
    {
      "source": "https://example.com/files/sample.pdf",
      "actionId": "PARTY_PASSPORT:transferor:chitharanj_kachappilly",
      "action": "add",
      "documentType": "passport",
      "matchedSubject": "Chitharanj Kachappilly"
    }
  ],
  "job": {
    "jobId": "job_4d8a445477",
    "status": "PROCESSING",
    "totalDocuments": 3
  }
}
```

**Option B — explicit mapping (when you already know `documentId`s)**

```json
{
  "replaceDocuments": [
    {
      "documentId": "doc_531b3c7408",
      "source": "https://example.com/files/sample.pdf"
    }
  ],
  "addFiles": [
    "https://example.com/files/sample.pdf"
  ]
}
```

### Step 5 — Re-poll job until clean

`GET /v1/jobs/job_4d8a445477`

Response `200` (target end state after successful correction):

```json
{
  "status": { "code": 200, "message": "Success" },
  "job": {
    "jobId": "job_4d8a445477",
    "status": "COMPLETED",
    "totalDocuments": 3,
    "completedDocuments": 3,
    "failedDocuments": 0,
    "workflowValidation": {
      "status": "PASSED",
      "summary": { "totalChecks": 3, "passed": 3, "failed": 0, "warnings": 0 },
      "expectedDocuments": 3,
      "presentDocuments": 3,
      "missingDocuments": []
    },
    "correctionPlan": null,
    "documents": [
      {
        "documentId": "doc_80878e1eb5",
        "fileName": "share_transfer_happy_case.pdf",
        "documentType": "shareTransferCertificate",
        "status": "COMPLETED"
      },
      {
        "documentId": "doc_531b3c7408",
        "fileName": "Mohammad+passport+-+Active_CORRECTED.jpeg",
        "documentType": "passport",
        "status": "COMPLETED"
      },
      {
        "documentId": "doc_xxxxxxxxxx",
        "fileName": "chithu+passport+2025.pdf",
        "documentType": "passport",
        "status": "COMPLETED"
      }
    ]
  }
}
```

### Summary

| Issue | Detected by | Fixed with |
|--------|-------------|------------|
| Missing transferor passport (Chitharanj) | `workflowValidation` + `correctionPlan` **add** | `addFiles` or matching URL in `sources` |
| Invalid Mohammad passport (MRZ rules) | `correctionPlan` **replace** + `…/validations` | `replaceDocuments` on `doc_531b3c7408` or matching URL in `sources` |
| Certificate | Passed in this run | No action |

**Sequence:** `POST /v1/jobs` → `GET /v1/jobs/{id}` (`workflowValidation` + `correctionPlan`) → optional `GET …/validations` → `POST /v1/jobs/{id}/retry` → `GET /v1/jobs/{id}` until `workflowValidation.status = PASSED`.

## B. Workflow-incomplete (business) cases — job runs fine, but documents don't satisfy the rules
Same endpoints as the happy path; only the `workflowValidation` payload differs (`GET /v1/jobs/{jobId}`):

| Use-case | Result |
|----------|--------|
| Share-transfer certificate not submitted | `workflowValidation.status = FAILED`, missing: *"Share transfer certificate has not been uploaded yet."* — fix with `retry` + `addFiles`/`replaceDocuments` |
| A party's passport missing | `FAILED`, missing: *"Passport for Onur Yegenoglu (transferee) has not been uploaded."* — fix with `retry` + `addFiles` |
| One or two of N expected docs missing | `expectedDocuments` vs `presentDocuments` + `missingDocuments[]` list |
| Job still processing | `status = INCOMPLETE` until all docs reach a terminal state |

> To resolve any of the above on the **same job**, call `POST /v1/jobs/{jobId}/retry` with `addFiles` (missing docs) and/or `replaceDocuments` (invalid docs). See section A2.

## C. Failure & recovery cases (technical)

| Use-case | Endpoint | Notes |
|----------|----------|-------|
| Extraction service unreachable / DNS / timeout (your `EXTRACTION_TIMEOUT`) | surfaced in `GET .../documents/{documentId}` | doc `status = FAILED`, `error.retryable = true` |
| **Re-run / replace / add documents (same job)** | `POST /v1/jobs/{jobId}/retry` → `202` | Empty body or `documentIds` re-runs FAILED docs; `replaceDocuments` swaps an invalid source; `addFiles` adds missing docs. Job must be terminal; re-evaluates `workflowValidation`. |
| **Cancel a running job** | `POST /v1/jobs/{jobId}/cancel` | Stops further processing |
| Unauthorized / not-found | any endpoint | `401` when `API_KEY` set and `x-api-key` missing; `404` for unknown ids |

## D. Operational/browsing cases

| Use-case | Endpoint |
|----------|----------|
| **List/filter jobs** (by `status`, `jobType`, paginated) | `GET /v1/jobs` |

## E. Webhook (event-driven) cases — optional, avoids polling

| Use-case | Endpoint | Notes |
|----------|----------|-------|
| **Register a webhook** for job/document events | `POST /v1/webhooks` → `201` | Events: `job.completed`, `job.failed`, `job.cancelled`, `document.extracted`, `document.validated`, `document.failed` |
| **List webhooks** | `GET /v1/webhooks` | |
| **Delete a webhook** | `DELETE /v1/webhooks/{webhookId}` → `204` | |
| Receive callback on completion | (your `callbackUrl` / registered URL) | Best-effort, HMAC-SHA256 signed |

---

### Minimal happy-path sequence
`POST /v1/jobs` → (poll) `GET /v1/jobs/{jobId}` until `COMPLETED` → read `workflowValidation` → for a detail screen use `GET /v1/jobs/{jobId}/detail`, or drill in via `GET .../documents`, `.../extraction`, `.../validations`.
