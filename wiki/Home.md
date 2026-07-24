# Intelligent Document Processing — Wiki Home

Portfolio product artifacts for an **API-led Intelligent Document Processing (IDP)** system that automates multi-document share-transfer reviews.

## Product narrative
Share transfer requests require several documents (certificate, passports, IDs, proof of address). Manual review is slow and error-prone. This product turns that into a **job-based workflow**: upload once, extract structured data, validate per document, then cross-check parties across documents.

## Repository map
| Repository | What it showcases |
|------------|-------------------|
| [idp-product](https://github.com/r123singh/idp-product) | Specs, OpenAPI, schemas (this wiki) |
| [idp-jobs-api](https://github.com/r123singh/idp-jobs-api) | Multi-document job orchestration API |
| [idp-extraction-engine](https://github.com/r123singh/idp-extraction-engine) | Schema-driven extraction + validation |
| [idp-assistant-ui](https://github.com/r123singh/idp-assistant-ui) | Conversational UI + UX prototypes |

## Wiki index
1. [[Product-Overview]] — problem, outcome, what was established
2. [[Product-Requirements]] — field-level extraction & validation intent
3. [[Use-Cases]] — happy path and exception flows mapped to APIs
4. [[API-Design]] — capability model and design choices
5. [[Architecture]] — how jobs, extraction, and UI fit together
6. [[Document-Schemas]] — document types and schema-driven config
7. [[UX-Journey]] — chat / prototype flow states
8. [[Deployment-Notes]] — local vs cloud packaging model

> Client-identifying names are intentionally omitted. Examples use generic registry / entity identifiers.
