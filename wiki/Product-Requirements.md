# Product Requirements

Source narrative distilled from the product requirements workbook (client names removed).

## Objective
Build an API-led system that automates share-transfer document review:
- Initiate a request and upload required documents (chat or API)
- Extract required fields at each step
- Validate fields and **cross-validate** against earlier documents
- Produce a final reviewable report / job detail package

## Core APIs
1. **Extraction** — return structured fields for a document type (or auto-detect type)
2. **Validation** — per-rule pass/fail with messages; plus cross-document workflow checks
3. **Jobs orchestration** — batch documents into an async job with status, detail, retry, cancel

## Primary document: Share Transfer Certificate
Illustrative field groups:
- **Company info** — company name, entity number (`ENT` + digits), document title
- **Current share structure** — shareholder, share type, counts, value per share
- **Transfer details** — seller/buyer, shares transferred, passport/registration refs
- **Post-transfer** — new holdings consistency checks
- **Signatures** — transfer date, party names, signature presence

## Supporting documents
- **Passport** — MRZ + visual zone fields; expiry checks; name match to certificate parties
- **Resident / Emirates ID** — identity fields for local parties when required
- **Proof of address** — address evidence when the workflow demands it

## Product rules of engagement
- Missing critical fields → ask for re-upload or correction
- Wrong document type → prompt for the correct document
- Multiple buyers/sellers → extract and validate each party record
- Cross-document name / ID consistency is a first-class outcome, not a UI afterthought
