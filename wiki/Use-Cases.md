# Use Cases

Happy-path first, then exception paths — mapped to the Jobs API.

## A. Happy path — share transfer processed successfully
1. `POST /v1/jobs` — submit certificate + party passports → `202` + `jobId`
2. `GET /v1/jobs/{jobId}` — poll until terminal; read `workflowValidation`
3. `GET /v1/jobs/{jobId}/detail` — one-call UI payload (overview, issues, timeline)
4. `GET /v1/jobs/{jobId}/documents` — per-document lifecycle
5. `GET .../extraction` and `.../validations` — structured results

**Success signal:** `workflowValidation.status = PASSED` (certificate present; each party matched to a passport).

## B. Incomplete package
Job completes extraction where possible, but workflow status is `INCOMPLETE` / failed checks for missing party documents. Correction plan tells the user what to upload next.

## C. Validation failures
Per-document rules fail (e.g. expired passport). Job detail exposes issues + highlights for operator review; retry endpoints allow reprocessing after a new file is supplied.

## D. Operator controls
- Cancel an in-flight job
- Retry failed documents
- Inspect timeline / audit-friendly history

Full request/response examples live in the product repo: `docs/use-cases.md` and `postman/`.
