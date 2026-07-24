# API Design

## Design question
Should the product expose **generic** extract/validate APIs, or step-wise APIs per workflow stage?

## Decision
**Both layers, cleanly separated:**
1. **Extraction engine** — generic, schema-driven extract + validate for one document
2. **Jobs API** — workflow-aware orchestration for multi-document share-transfer jobs
3. **Config APIs (spec'd)** — document-type schema/config and storage contracts (OpenAPI in repo)

## Capabilities captured in specs
- CRUD-ish document-type configuration (fields + rules)
- Extract by known `documentType` **or** auto-identify then extract
- Image (PNG/JPEG) and PDF inputs
- Structured JSON responses with rule-level boolean outcomes
- Explicit error classes: bad upload, wrong type, LLM/processing failures
- Async jobs with webhooks (optional) and rich poll/detail reads

## Agent / chat readiness
OpenAPI-first contracts, stable job identifiers, and a single **job detail** resource so conversational UIs and LLM agents do not need to fan out across many calls for a screen.

See also: `docs/api-design-notes.md`, `docs/api-capabilities.md`, `openapi/`.
