# Service Specification Template (v0.1)

> **Scope**: Concise statement of what this service does (metadata CRUD, queue management, etc.) and what it explicitly does not do in v0.1.
>
> **Deliverable**: This spec is the authoritative source of truth for contracts, models, DB schema, and test plan. Implementation is complete when all tests defined here pass and golden snapshots match.

---

## 1. Context & Goals

* Purpose of the service in one or two sentences.
* Key goals: e.g., deterministic CRUD API, persistence in Postgres, stable error model, pagination, etc.
* Non-goals (to prevent scope creep).
* Assumptions (ID generation, external auth handled elsewhere, etc.).

---

## 2. Versioning & Conventions

* Base path (e.g., `/api/v1`).
* Content-Type (usually `application/json; charset=utf-8`).
* Timestamps (ISO-8601 UTC).
* IDs (UUID v4).
* Pagination (simple page/limit; deterministic sort).

---

## 3. Data Model (API Contracts)

### 3.1 Resource Schema

```json
{
  "id": "uuid",
  "field1": "string",
  "field2": "integer",
  "created_at": "ISO-8601 UTC",
  "updated_at": "ISO-8601 UTC"
}
```

### 3.2 Create Schema

```json
{
  "field1": "required string",
  "field2": "required integer"
}
```

### 3.3 Update Schema

```json
{
  "field1": "optional string",
  "field2": "optional integer"
}
```

### 3.4 Error Envelope

```json
{
  "error": {
    "code": "string (e.g., not-found, validation-error)",
    "message": "human-friendly message",
    "details": { "fieldErrors?": [{"field": "string", "reason": "string"}] }
  }
}
```

---

## 4. Endpoints

* **Health/Meta**: `/healthz`, `/readyz`, `/openapi.json`.
* **Create Resource**: `POST /resources`.
* **Get Resource by ID**: `GET /resources/{id}`.
* **List Resources**: `GET /resources` with `q`, `page`, `limit`, `sort`, `order`.
* **Update Resource**: `PATCH /resources/{id}`.
* **Delete Resource**: `DELETE /resources/{id}`.

Each endpoint should define:

* Request schema
* Response schema(s)
* Example request/response (happy path and at least one error)

---

## 5. Behavior Details

* Validation & normalization rules.
* Error model (standard codes, envelopes).
* Idempotency semantics.
* Pagination rules.
* Sorting rules.

---

## 6. Persistence (PostgreSQL)

### 6.1 Table Definition (DDL reference)

```sql
CREATE TABLE resources (
  id UUID PRIMARY KEY,
  field1 VARCHAR(200) NOT NULL,
  field2 INTEGER NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 6.2 Uniqueness Policy

* Document any uniqueness constraints (or explicitly state none).

### 6.3 Migrations

* Migration file name and summary.

### 6.4 Repository Contract

```ts
interface ResourceRepository {
  create(data: CreateIn): Resource;
  get(id: UUID): Resource | None;
  list(params: ListParams): Page<Resource>;
  update(id: UUID, patch: UpdateIn): Resource | NotFound;
  delete(id: UUID): Deleted | NotFound;
}
```

#### Invariants & Semantics

* Normalization rules
* Validation rules
* Timestamp behavior
* Sorting & filtering
* Pagination math
* Transactions & concurrency

#### Error Mapping

| DB Condition / Error Source  | Service Error Code | HTTP Status | Notes |
| ---------------------------- | ------------------ | ----------- | ----- |
| CHECK constraint violation   | validation-error   | 400         | …     |
| NOT NULL violation           | validation-error   | 400         | …     |
| Unique violation             | conflict           | 409         | …     |
| Missing row (get/update/del) | not-found          | 404         | …     |
| Other DB error               | internal-error     | 500         | …     |

---

## 7. OpenAPI (normative excerpts)

* Show minimal YAML with paths, schemas, and standard error components.

---

## 8. Test Plan (Definition of Done)

### 8.1 Testing Strategy

* Unit tests (validation, happy path, edge cases).
* Golden snapshot tests (responses, OpenAPI snapshot).
* Repository tests (real Postgres).
* Thin end-to-end smoke tests.

### 8.2 Test Matrix (per endpoint)

* POST: happy path, validation errors, conflict.
* GET: existing, not found.
* LIST: paging, sorting, invalid params.
* PATCH: partial updates, validation errors, not found.
* DELETE: existing, not found.
* Healthz/readyz.

### 8.3 Snapshot Rules

* Strip or freeze non-deterministic fields (timestamps, UUIDs).
* Keep snapshots representative and minimal.

### 8.4 Exit Criteria

* All tests pass
* OpenAPI snapshot matches baseline
* Spec updated for any intentional changes

---

## 9. Operational Notes

* Readiness behavior
* Observability requirements (structured logging, request IDs)
* External dependencies (auth, rate limiting, etc.)

---

## 10. Change Control

* Versioning rules (minor vs major bumps)
* Sunset/deprecation process

---

## 11. Open Questions / Future Enhancements

* Placeholder for things out of scope in v0.1
