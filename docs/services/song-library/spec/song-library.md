# KJBot — song-library Service Specification (v1.0.0)

> **Scope**: Metadata-only CRUD for karaoke songs (no media upload/streaming). Stores references to media in S3 via `{ bucket, key }`.
>
> **Deliverables**: This narrative spec is authoritative for contracts, models, DB schema, and behavior. Implementation is complete when the accompanying **test-matrix** passes and **golden snapshots** match. Machine contract lives in `openapi.v1.yaml`.

---

## 1. Context & Goals

* Provide a deterministic REST API for CRUD and listing of song metadata.
* Persistence: PostgreSQL with explicit migrations and indices for search/sort.
* Stable error model and consistent pagination across endpoints.

### Non-Goals (v1.0.0)

* Uploading/transcoding media; presigned URL orchestration.
* AuthZ policies (assumed upstream). Service may echo Request ID; does not enforce scopes in v1.
* Full-text fuzzy search; v1 ships simple `ILIKE` contains filter.

### Assumptions

* `song_id`: UUID v4 owned by this service.
* `duration`: integer seconds, 0–86,400.
* `media_file`: reference only `{ bucket, key }` (both required on create).

---

## 2. Versioning & Conventions

* Base path: `/api/v1`.
* Content-Type: `application/json; charset=utf-8`.
* Timestamps: ISO-8601 UTC (`...Z`).
* IDs: UUID v4 strings.
* Pagination: `page`/`limit` with stable sort; see §5.4.

---

## 3. Data Model (API Contracts)

### 3.1 JSON Schemas (code-like)

```json
// Song (response model)
{
  "song_id": "uuid",
  "title": "string (1..200)",
  "artist": "string (1..200)",
  "duration": 0,
  "media_file": { "bucket": "string", "key": "string" },
  "created_at": "ISO-8601 UTC",
  "updated_at": "ISO-8601 UTC"
}
```

```json
// SongCreate (request)
{
  "title": "required string",
  "artist": "required string",
  "duration": "required integer >= 0",
  "media_file": { "bucket": "required", "key": "required" }
}
```

```json
// SongUpdate (request; PATCH semantics)
{
  "title": "optional string",
  "artist": "optional string",
  "duration": "optional integer >= 0",
  "media_file": { "bucket": "required with key if present", "key": "required with bucket if present" }
}
```

```json
// Error envelope
{ "error": { "code": "not-found|validation-error|conflict|internal-error", "message": "...", "details": { "fieldErrors?": [{"field":"...","reason":"..."}] } } }
```

### 3.2 Field Rules

* `title`, `artist`: trim and collapse internal whitespace; 1–200 chars; must contain at least one non-space.
* `duration`: 0–86,400.
* `media_file.bucket` 1–200; `media_file.key` 1–1024.

---

## 4. Endpoints

### 4.1 Health/Meta

* `GET /api/v1/healthz` → `200 {"status":"ok"}` (no DB access)
* `GET /api/v1/readyz` → `200 {"status":"ready"}` when DB reachable & migrations applied; else `503`.

### 4.2 Create Song — `POST /api/v1/songs`

* **Request**: `SongCreate`
* **Responses**: `201 Song` with `Location: /api/v1/songs/{song_id}`; `400 validation-error`; `409 conflict` (future uniqueness only)

### 4.3 Get Song — `GET /api/v1/songs/{song_id}`

* `200 Song`; `404 not-found`

### 4.4 List Songs — `GET /api/v1/songs`

* Query: `q?`, `page=1`, `limit=25 (≤100)`, `sort={title,artist,created_at,updated_at,duration}`, `order={asc,desc}`
* Returns envelope: `{ items, page, limit, total, has_next }`

### 4.5 Update Song — `PATCH /api/v1/songs/{song_id}`

* **Request**: `SongUpdate`
* `200 Song`; `400 validation-error`; `404 not-found`; `409 conflict` (future uniqueness)

### 4.6 Delete Song — `DELETE /api/v1/songs/{song_id}`

* `204 No Content`; `404 not-found`

---

## 5. Behavior Details

### 5.1 Validation & Normalization

* Trim/collapse spaces in `title`/`artist`; reject empty-after-trim.
* Reject `duration` outside 0–86,400.
* `media_file` must be both-or-none on update.

### 5.2 Error Model

* Codes: `validation-error`, `not-found`, `conflict`, `internal-error`.
* Always return the error envelope for non-2xx; include `fieldErrors` where applicable.

### 5.3 Idempotency

* `PATCH` idempotent for identical body. `DELETE` idempotent; repeated delete → `404`. `POST` non-idempotent (future `Idempotency-Key` TBD).

### 5.4 Pagination

* `page≥1`; `limit∈[1,100]`; `has_next = page*limit < total`.
* Stable sort; tie-break by `song_id ASC`.

---

## 6. Persistence (PostgreSQL)

### 6.1 DDL (reference)

```sql
CREATE TABLE songs (
  song_id      UUID PRIMARY KEY,
  title        VARCHAR(200) NOT NULL,
  artist       VARCHAR(200) NOT NULL,
  duration     INTEGER NOT NULL CHECK (duration >= 0 AND duration <= 86400),
  media_bucket VARCHAR(200) NOT NULL,
  media_key    VARCHAR(1024) NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_songs_title_ci  ON songs (LOWER(title));
CREATE INDEX idx_songs_artist_ci ON songs (LOWER(artist));
CREATE INDEX idx_songs_created_at ON songs (created_at DESC, song_id ASC);
CREATE INDEX idx_songs_updated_at ON songs (updated_at DESC, song_id ASC);
```

### 6.2 Uniqueness Policy

* **None in v1.0.0**. If enabled later (e.g., `(LOWER(title), LOWER(artist), duration)`), endpoints may return `409 conflict`.

### 6.3 Repository Contract (service-internal)

* See invariants & test matrix in repo (contract summarized): normalization, timestamp updates, ILIKE filter on `title|artist`, stable sort + `song_id` tie-break, pagination math, transactional writes, last-write-wins.

### 6.4 Error Mapping (DB → Service → HTTP)

| DB Condition / Error Source                           | Service Error Code | HTTP Status | Notes                                                               |
| ----------------------------------------------------- | ------------------ | ----------- | ------------------------------------------------------------------- |
| CHECK constraint violation (e.g., `duration` range)   | `validation-error` | 400         | Include `fieldErrors` for the offending field(s).                   |
| NOT NULL violation                                    | `validation-error` | 400         | Same envelope as above.                                             |
| Unique violation (future optional `(title, artist…)`) | `conflict`         | 409         | Only if uniqueness constraint is enabled in schema.                 |
| Missing row on `get`, `update`, or `delete`           | `not-found`        | 404         | Repo returns None/NotFound; handler wraps into error envelope.      |
| Any other DB error (connection, unexpected failure)   | `internal-error`   | 500         | Do not leak DB internals; log details internally with `request_id`. |

---

## 7. OpenAPI & Tests

* Machine contract: see `openapi.v1.yaml`.
* Tests & golden snapshots: see `test-matrix.md` and `golden/`.

---

## 8. Change Control

* Backward-compatible additions: bump minor version in `openapi.v1.yaml` `info.version`.
* Breaking changes: introduce `/api/v2`, deprecate v1 with Sunset header; update changelog.

---

## 9. Future Enhancements

* Idempotency-Key for POST; extended search (tsvector); soft deletes; media existence checks.

---