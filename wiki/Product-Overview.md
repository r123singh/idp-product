# Product Overview

## Problem
Share-transfer reviews require multiple documents (certificate, party passports / IDs, sometimes proof of address). Single-document extraction APIs help, but they do not manage **multi-document context**, cross-checks, corrections, or audit-friendly job history.

## Product outcome
An **Intelligent Document Processing (IDP)** framework that:
1. Accepts a share-transfer request as one **job** with many files
2. Extracts structured fields per document type using schemas + LLM
3. Validates each document against configurable rules
4. Runs **workflow validation** across documents (e.g. every named party has a matching passport)
5. Surfaces correction plans and UI-ready detail for human review
6. Integrates with a conversational assistant UI

## What was established
- A working single-document extraction engine (schema + natural-language rules + OCR/LLM)
- Limits of single-document APIs for multi-step registry workflows
- Need for job orchestration, cross-validation, retry/correction, and audit listing
- OpenAPI-first design so chat UIs and agents can integrate cleanly

## Role highlighted (PM)
- Turned operational share-transfer checks into an API product surface
- Defined document field catalogs and validation intent
- Mapped happy-path and exception use cases to concrete endpoints
- Separated **product specs**, **orchestration API**, **extraction engine**, and **UI** so each can evolve independently
