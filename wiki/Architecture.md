# Architecture

```text
┌─────────────────────┐
│  Assistant UI       │  chat + file attach + progress
│  (idp-assistant-ui) │
└──────────┬──────────┘
           │ HTTPS /v1/jobs*
           ▼
┌─────────────────────┐
│  Jobs API           │  accept job, persist, orchestrate
│  (idp-jobs-api)     │  workflow validation, detail DTO
└──────────┬──────────┘
           │ extractInfo
           ▼
┌─────────────────────┐
│  Extraction Engine  │  schema load → LLM/OCR → rules
│  (idp-extraction-   │  typed JSON + summary
│   engine)           │
└─────────────────────┘
```

## Persistence
Jobs and document transactions can run in-memory (local demo) or DynamoDB (deployed).

## Why separate repos
- Extraction evolves with schemas/models without coupling UI release trains
- Jobs API owns workflow product semantics
- UI/prototypes iterate on UX independently
- Product repo + wiki own the PM artifact surface
