---
inclusion: always
---

# API Design Best Practices

Standard guidelines for designing RESTful APIs. Use these as guardrails when creating or reviewing any API specification. Sections are ordered by criticality — security and correctness first, extensibility patterns last.

---

# Tier 1 — Security & Correctness

---

## 1. Security

- All endpoints MUST use OAuth 2.0 (Authorization Code flow) via the `oauth2` security scheme.
- Define granular scopes following the pattern `{domain}:{action}` or `{domain}:{subdomain}:{action}`:
  - e.g., `orders:read`, `orders:write`, `orders:customer:view`
- Apply the principle of least privilege — each endpoint declares only the scopes it needs.
- Read-only endpoints use read scopes; mutating endpoints use write/action scopes.
- Never expose internal IDs or secrets in URLs or response bodies unnecessarily.
- Always use HTTPS. Never allow plaintext HTTP for any environment.

---

## 2. Input Sanitization & Injection Prevention

- Apply `maxLength` on all string fields — no free-text field should accept unbounded input.
- Use `pattern` (regex) to whitelist allowed characters on structured string fields:
  - Codes/identifiers: `^[A-Z0-9\-]{1,50}$`
  - Names: `^[\w\s\-\.]{1,100}$`
  - This blocks script tags, SQL fragments, and special characters at the schema level.
- Use `enum` for any field with a fixed set of values — the schema rejects invalid input before business logic runs.
- Use `format` constraints (`date`, `date-time`, `email`, `uri`) so the framework validates structure automatically.
- Apply `minimum` / `maximum` on all numeric fields to prevent overflow or nonsensical values.
- Apply `minItems` / `maxItems` on array fields to cap bulk input sizes.
- On the server side:
  - Never interpolate user input directly into SQL — always use parameterized queries.
  - Never pass user input to shell commands or template engines unescaped.
  - Strip or reject HTML/script content in free-text fields unless rich text is explicitly required.
  - Use output encoding when rendering user-supplied data in any downstream context (HTML, email, PDF).
- Log sanitized versions of input — never log raw PII or unescaped user content.
- The OpenAPI schema is your first line of defense. The more you constrain at the spec level, the less sanitization you need in code.

---

## 3. Response Envelope & Error Handling

All API responses MUST use a uniform envelope with a `status` object at the top level. This ensures clients always parse the same shape for both success and error cases.

```json
// Success response (single resource — key name matches the entity)
{
  "status": { "code": 200, "message": "Success" },
  "order": { ... }
}

// Success response (list — key name is the plural entity)
{
  "status": { "code": 200, "message": "Success" },
  "orders": [ ... ],
  "resultInfo": { "numberOfRecords": 20, "offset": 0, "totalRecords": 100 }
}

// Error response
{
  "status": {
    "code": 400,
    "message": "Missing required field.",
    "errorCode": "INVALID_REQUEST"
  }
}
```

**Envelope rules:**
- `status.code` — HTTP status code (mirrored for client convenience).
- `status.message` — human-readable description (success or error).
- `status.errorCode` — present only on errors; machine-readable, uppercase, underscore-separated.
- The payload key MUST be named after the key entity, not a generic `data`. For a single resource use the singular entity name (e.g., `order`, `customer`, `product`); for a list use the plural (e.g., `orders`, `customers`, `products`). Do NOT use a generic `data` key.
- On error responses, omit the entity key entirely — only `status` is returned.
- `resultInfo` — present only on list responses for pagination metadata.
- Clients check `status.code` or the presence of `status.errorCode` to determine success vs. failure.
- Define the `status` and `resultInfo` objects as reusable schemas in `components/schemas`. The entity key varies per endpoint, so each operation defines its own response schema composing the shared `status` object with the entity-specific key.

**Error code rules:**
- `errorCode` values MUST be stable, uppercase, underscore-separated strings (e.g., `ORDER_NOT_FOUND`, `DUPLICATE_ENTRY`).
- Every non-2xx response in the spec MUST document the error schema.
- Group error codes by domain for discoverability.
- Use the following common error codes as a baseline across all APIs:

| HTTP Status | errorCode                  | When to Use                                                        |
|-------------|----------------------------|--------------------------------------------------------------------|
| 400         | `INVALID_REQUEST`          | Malformed JSON, missing required fields, bad syntax                |
| 400         | `INVALID_PARAMETER`        | Query/path parameter fails validation (type, range, format)        |
| 400         | `INVALID_STATUS_TRANSITION`| Requested state change violates lifecycle rules                    |
| 400         | `DUPLICATE_ENTRY`          | Resource already exists (e.g., duplicate key or identifier)        |
| 400         | `VALIDATION_FAILED`        | Business rule violation (e.g., start date after end date)          |
| 401         | `UNAUTHORIZED`             | Missing or invalid authentication token                            |
| 403         | `FORBIDDEN`                | Valid token but insufficient scopes or permissions                 |
| 404         | `RESOURCE_NOT_FOUND`       | Requested resource does not exist                                  |
| 404         | `{DOMAIN}_NOT_FOUND`       | Domain-specific not-found (e.g., `ORDER_NOT_FOUND`, `USER_NOT_FOUND`) |
| 409         | `CONFLICT`                 | Concurrent modification or already-processed request               |
| 409         | `ALREADY_EXISTS`           | Attempt to create a resource that already exists                   |
| 409         | `STATE_CONFLICT`           | Resource is in a state that prevents the requested operation       |
| 410         | `RESOURCE_GONE`            | Resource existed but is no longer available (expired, deleted)     |
| 422         | `UNPROCESSABLE_ENTITY`     | Request is well-formed but semantically invalid                    |
| 429         | `RATE_LIMIT_EXCEEDED`      | Too many requests — client should back off and retry               |
| 500         | `INTERNAL_ERROR`           | Unexpected server-side failure                                     |
| 502         | `BAD_GATEWAY`              | Invalid response from an upstream service                          |
| 503         | `SERVICE_UNAVAILABLE`      | Downstream dependency is unreachable or under maintenance          |
| 504         | `GATEWAY_TIMEOUT`          | Upstream service did not respond in time                           |

---

## 4. HTTP Methods and Status Codes

| Operation           | Method  | Success Code | Notes                              |
|---------------------|---------|--------------|-------------------------------------|
| Create resource     | POST    | 201          | Return the created resource         |
| List resources      | GET     | 200          | Include pagination metadata         |
| Get single resource | GET     | 200          | Return full resource representation |
| Full update         | PUT     | 200          | Replace entire resource             |
| Partial update      | PATCH   | 200          | Return updated resource             |
| Delete resource     | DELETE  | 204          | No content in response body         |
| Status transition   | PATCH   | 200          | Use for state machine transitions   |

- Use `400` for validation errors and malformed requests.
- Use `404` when a referenced resource doesn't exist.
- Never use `200` for creation — always `201`.
- Use `204` for successful deletions with no response body.

---

# Tier 2 — Structure & Contracts

---

## 5. Resource Naming and URL Structure

- Use plural nouns for resource collections: `/orders`, `/products`, `/customers`.
- Use kebab-case for multi-word path segments: `/order-items`, `/payment-methods`.
- Nest sub-resources under their parent: `/orders/{orderId}/items`.
- Use path parameters for resource identifiers, query parameters for filtering/pagination.
- Action endpoints (non-CRUD) use a verb suffix: `/orders/{orderId}/cancel`.
- Keep URLs max 2-3 levels deep. Avoid deeply nested paths.
- For APIs that return data scoped to a particular customer, use one of two URL forms depending on the identifier:
  - Default (primary ID): `/customers/customerkey/{customer_identifier}/...` — used when looking up by the system's primary customer ID.
    - `/customers/C-98765/orders`
    - `/customers/C-98765/loyalty-points`
  - Alternate key lookup: `/customers/{customerkey-name}/{customerkey-value}/...` — used when looking up by a non-primary identifier. `{customerkey-name}` is the identifier type (e.g., `email`, `mobile`, `loyaltyId`) and `{customerKey-value}` is the actual value.
    - `/customers/email/user@example.com/orders`
    - `/customers/mobile/+1234567890/loyalty-points`
    - `/customers/loyaltyId/LY-12345/vouchers`
- The alternate key pattern makes the lookup key explicit in the URL and avoids ambiguity when multiple identifier types exist.

---

## 6. Request and Response Design

- Request bodies MUST clearly mark `required` fields at the schema level.
- Use `enum` constraints for fields with a fixed set of values (e.g., status, type, category).
- Use `nullable: true` for optional fields that may be absent vs. explicitly null.
- Timestamps MUST use ISO 8601 `date-time` format (UTC): `"2025-06-01T08:30:00Z"`.
- Date-only fields use `date` format: `"2025-06-01"`.
- Provide `example` values for every field to support documentation and mocking.
- Responses for create/update operations should return the full resource representation.
- Use consistent field naming across all endpoints (camelCase recommended).

---

## 7. Schema Design

- Define reusable schemas in `components/schemas` and reference via `$ref`.
- Use separate schemas for list (summary) vs. detail views:
  - Summary schema for list endpoints (lighter payload, includes key fields and stats)
  - Detail schema for single-resource endpoints (full representation with nested data)
- Embed aggregate stats as nested objects rather than requiring separate endpoints.
- Apply `minimum`/`maximum` constraints on numeric fields where applicable.
- Use `pattern` for string fields with specific formats (e.g., hex colors, codes).

---

## 8. Versioning

- Use URL path versioning as the primary strategy: `/v1/orders`, `/v2/orders`.
- Alternatively, use header-based versioning via `Accept` header for minor revisions.
- Never introduce breaking changes without a version bump.
- Deprecate old versions with clear timelines and migration guides.

---

## 9. Pagination

- All list endpoints MUST support offset-based pagination using `offset` and `limit` query parameters.
- Responses MUST include a `resultInfo` metadata object with:
  - `numberOfRecords` — count of records in the current page
  - `offset` — starting position of the current page
  - `totalRecords` — total available records across all pages
- Default values: `offset=0`, `limit=20` (configurable per API).
- Clients should never assume they've received all records unless `offset + numberOfRecords >= totalRecords`.

```yaml
# Example resultInfo
resultInfo:
  numberOfRecords: 20
  offset: 0
  totalRecords: 100
```

---

# Tier 3 — Behavior & Resilience

---

## 10. Rate Limiting & Throttling

- All APIs MUST include rate limiting to protect backend services from abuse and overload.
- Return standard rate limit headers in every response:
  - `X-RateLimit-Limit` — maximum requests allowed in the current window
  - `X-RateLimit-Remaining` — requests remaining in the current window
  - `X-RateLimit-Reset` — UTC epoch timestamp when the window resets
- When the limit is exceeded, return `429 Too Many Requests` with a `Retry-After` header (seconds until the client can retry).
- Define rate limits per scope: per-client, per-user, per-endpoint, or per-API-key.
- Document rate limits in the API spec description and in response headers so clients can self-throttle.
- Consider tiered quotas for different API consumers (e.g., free vs. premium).

```yaml
# Example 429 response headers
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1719500400
Retry-After: 60
```

---

## 11. Idempotency

- All non-idempotent operations (POST, PATCH) SHOULD support an `Idempotency-Key` request header to allow safe retries.
- The server MUST store the result of the first request for a given key and return the same response for duplicate requests.
- Idempotency keys should be client-generated UUIDs (e.g., `Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000`).
- Keys should expire after a reasonable TTL (e.g., 24-48 hours).
- If a duplicate request is received while the original is still processing, return `409 Conflict` or `423 Locked`.
- GET, PUT, and DELETE are inherently idempotent and do not require this header.
- Document idempotency support clearly in the API spec for each endpoint.

---

## 12. State Machine and Lifecycle

- Resources with lifecycle states should define explicit status enums.
- Document valid state transitions clearly (e.g., `draft → active → completed → archived`).
- Status transitions should be performed via PATCH with a `status` field.
- Include timestamp tracking for each state transition (e.g., `createdAt`, `updatedAt`, `completedAt`).
- Validate transitions server-side — reject invalid state changes with `400` and `INVALID_STATUS_TRANSITION`.

---

## 13. Filtering and Querying

- List endpoints should support filtering via query parameters, not request bodies.
- Common filter patterns:
  - `status` — filter by resource state (use enum values)
  - `dateFrom` / `dateTo` — date range filtering
  - Named lookups by key business fields (e.g., `name`, `email`, `type`)
- Filter parameter names should match the field names in the response schema.
- Support sorting via `sortBy` and `sortOrder` (asc/desc) query parameters where applicable.

---

# Tier 4 — Patterns & Extensibility

---

## 14. Bulk Operations

- Support bulk creation by accepting arrays in POST request bodies.
- Return processing summary stats in the response: `successCount`, `duplicateCount`, `failedCount`.
- This gives callers immediate feedback without needing a separate status-check endpoint.
- For large bulk operations, consider async processing with a status polling endpoint.

---

## 15. Free-Form Fields with Optional Structure

When a field needs to be solution-agnostic (content varies by implementation) but still secure, split it into a free-text component and a structured metadata component.

**Schema pattern:**

```yaml
additionalInfo:
  type: object
  nullable: true
  properties:
    notes:
      type: string
      maxLength: 500
      description: Free-form text notes. HTML and script content will be stripped server-side.
    metadata:
      type: object
      maxProperties: 20
      additionalProperties:
        type: string
        maxLength: 200
      description: >
        Arbitrary key-value pairs for structured data.
        Keys must be alphanumeric with underscores.
        Values are strings with a 200-char limit.
```

**Why this works:**
- `notes` — completely free-form, capped by `maxLength`, sanitized server-side.
- `metadata` — open-ended key-value map. Any key is allowed, but values are always flat strings with a size cap. No nested objects, no arrays, no injection via deep structures.
- `maxProperties: 20` prevents payload bloat.
- `additionalProperties: { type: string, maxLength: 200 }` constrains every value at the schema level.

**Usage examples:**

```json
// Free-text only
{
  "additionalInfo": {
    "notes": "Campaign for Summer 2025 customers"
  }
}

// Structured data only
{
  "additionalInfo": {
    "metadata": {
      "campaign": "summer_2025",
      "channel": "email",
      "segment": "premium"
    }
  }
}

// Both combined
{
  "additionalInfo": {
    "notes": "Targeting premium users in APAC region",
    "metadata": {
      "region": "APAC",
      "tier": "premium",
      "budget_code": "MKT-2025-Q3"
    }
  }
}
```

**Server-side validation rules:**
- Strip HTML/script tags from `notes` (e.g., `<script>`, `<img onerror=...>`).
- Validate `metadata` keys match `^[a-zA-Z0-9_]{1,50}$` — reject keys with special characters.
- Reject or sanitize `metadata` values containing `<`, `>`, backticks, or `${`.
- Return `400 VALIDATION_FAILED` if constraints are violated.

**When to use this pattern:**
- Any field labelled "additional info", "notes", "custom data", or "metadata".
- Fields where the consuming solution decides what goes in, not the API spec.
- Avoid using a plain `type: string` for structured data — it forces consumers to JSON-encode into a string, losing schema validation entirely.

---

## 16. Optional Features via Query Parameters

- Use boolean query parameters for optional response enrichment (e.g., `includeDetails=true`, `expand=metadata`).
- Conditional response fields should be marked `nullable: true` and documented as present only when the flag is set.
- Keep the default response lean — enrich only when explicitly requested.

---

## 17. Deprecation Strategy

- Use the `deprecated: true` flag in OpenAPI on endpoints, parameters, or schemas being phased out.
- Include a `Sunset` HTTP header in responses from deprecated endpoints with the removal date:
  - `Sunset: Sat, 01 Mar 2026 00:00:00 GMT`
- Add a `Deprecation` header with the date the endpoint was deprecated:
  - `Deprecation: Wed, 01 Jan 2025 00:00:00 GMT`
- Include a `Link` header pointing to the replacement endpoint or migration guide:
  - `Link: <https://api.example.com/v2/orders>; rel="successor-version"`
- Document deprecation timelines in the API changelog and notify consumers proactively.
- Maintain deprecated endpoints for at least one full version cycle (minimum 6 months recommended).
- Return a warning in the response body or via a `Warning` header for deprecated endpoints still in use.

---

## 18. Webhooks & Async Patterns

- For long-running operations, return `202 Accepted` with a status polling URL:

```yaml
# Example 202 response
statusCode: 202
headers:
  Location: /jobs/{jobId}
body:
  jobId: "abc-123"
  status: "processing"
  statusUrl: "/jobs/abc-123"
```

- Provide a polling endpoint (`GET /jobs/{jobId}`) that returns progress and final result.
- For event-driven integrations, support webhook callbacks:
  - Allow consumers to register callback URLs via a subscription endpoint.
  - POST event payloads to the registered URL when the event occurs.
  - Sign webhook payloads using HMAC-SHA256 with a shared secret — include the signature in a header (e.g., `X-Webhook-Signature`).
  - Include a timestamp in the signature to prevent replay attacks.
- Implement retry logic with exponential backoff for failed webhook deliveries.
- Provide a webhook event log endpoint so consumers can replay missed events.
- Document all event types, payload schemas, and retry behavior in the API spec.

---

## 19. OpenAPI Spec Hygiene

- Always include a `servers` block with at least one environment URL.
- Add `description` to all tags.
- Include `license` and `contact` in the `info` object.
- Every operation needs an `operationId` (camelCase, unique across the spec).
- Use `tags` to group related endpoints.
- Provide `examples` for complex response shapes (named examples for polymorphic responses).
- Validate specs against linting rules (e.g., Spectral) before publishing.

---

# Appendix: API Review Report Format

When evaluating an OpenAPI spec against this steering doc, generate a compliance report using the following structure.

---

## Report Header

```markdown
# {API Name} — Steering Compliance Report

Evaluated against: `.kiro/steering/api-design-best-practices.md` ({N} sections)
Evaluation date: {YYYY-MM-DD}
Spec file: `{path/to/spec.yaml}`
```

---

## Summary Table (placed immediately after the header)

```markdown
## Summary

| Category | Pass | Partial | Fail | N/A |
|----------|------|---------|------|-----|
| 1. Security | {n} | {n} | {n} | {n} |
| 2. Input Sanitization | {n} | {n} | {n} | {n} |
| ... | ... | ... | ... | ... |
| **Total** | **{n}** | **{n}** | **{n}** | **{n}** |

**Overall compliance: {pass} of {applicable} rules pass ({percentage}%).**
```

- `applicable` = total rules minus N/A count
- Percentage = `pass / applicable * 100`, rounded to nearest integer

---

## Priority Fixes (placed right after the summary)

```markdown
### Priority fixes (high impact):
1. {Most critical fix — security or correctness}
2. {Next most critical}
...
```

**Ranking criteria (in order):**
1. Tier 1 failures (Security, Input Sanitization, Response Envelope, HTTP Methods)
2. Tier 2 failures (Naming, Request/Response Design, Schema, Versioning, Pagination)
3. Tier 3 failures (Rate Limiting, Idempotency, State Machine, Filtering)
4. Tier 4 failures (Bulk Ops, Free-Form Fields, Optional Features, Deprecation, Webhooks, Hygiene)
5. Within the same tier, prioritize by number of failing rules

---

## Per-Section Evaluation (detailed breakdown follows)

For each of the 19 steering sections, create a subsection with a rule-by-rule table:

```markdown
## Section {N}: {Section Title}

| Rule | Status | Detail |
|------|--------|--------|
| {Rule description} | ✅ Pass | {Evidence from the spec} |
| {Rule description} | ⚠️ Partial | {What's present and what's missing} |
| {Rule description} | ❌ Fail | {What's missing or incorrect} |

**Fix:** {Specific, actionable remediation steps — only include if there are failures or partials.}
```

**Status values:**
- ✅ Pass — rule is fully satisfied
- ⚠️ Partial — rule is partially met (some fields/endpoints comply, others don't)
- ❌ Fail — rule is not met
- N/A — rule does not apply to this API (e.g., no deprecated endpoints, no long-running operations)

**Rules for writing the Detail column:**
- Reference specific endpoints, fields, or schemas by name
- Quote actual values from the spec where relevant
- For failures, state what's expected vs. what's found

**Rules for the Fix section:**
- Only include when there are ❌ Fail or ⚠️ Partial results
- Be specific: name the fields, endpoints, or schemas to change
- Provide the exact property/value to add where possible
- Number multiple fixes for clarity

---

## File Naming Convention

Save the report as: `specs/{API-Name}-Review-Report.md`

Example: `specs/VoucherManagement-Review-Report.md`
