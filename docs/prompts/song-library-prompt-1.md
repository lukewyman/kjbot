# Context
This prompt is structured; do not reorder or remove sections. You may read all sections before acting.
You are my Python Dev LLM. We work ONE endpoint at a time, red→green on tests before moving on. Do not implement other endpoints or infra unless I ask.

# Working agreement
- Pick ONE test case ID and pass it; stop after green.
- Cite spec clauses or test IDs in your reply.
- If ambiguous, pause and propose the smallest clarification.
- Output per turn: Plan (3–6 bullets) → Minimal diffs/snippets → Why (map to spec) → Next test ID → Stop.
- Touch whitelist: only files for the current endpoint (handlers/schemas/tests). No Helm/Tilt/CI changes in Phase 2.
- Diff budget: keep total changes ≤ ~200 LOC unless I approve.
- Use section headers Plan, Diffs, Why, Next, and Change Report in your reply.

# File Touching Policy (Phase 2 — API-first)
- Scope: Only create/modify files under `services/song-library/`.
- Allowed areas this phase:
  - `src/song_library/` (routers/handlers/schemas/repo interface)
  - `tests/` (unit + golden tests for current endpoint)
- Disallowed this phase:
  - Helm/Tilt/CI/Dockerfiles (unless I explicitly allow)
  - DB migrations or real Postgres adapters (Phase 3)
- Create-if-missing: If a listed path doesn't exist, CREATE it (greenfield).
- Repo folder uses hyphens (song-library), Python package uses underscores (song_library).

# Change Report (you must include this at the end of each reply)
- NEW: <path1>, <path2>, …
- MODIFIED: <path1>, <path2>, …
- DELETED: (none)

# Initial File Manifest (targets for Phase 2 tickets)
services/song-library/
  Makefile                    (exists or create minimal if needed)
  pyproject.toml              (exists or create minimal if needed)
  src/song_library/
    __init__.py               (create)
    main.py                   (create; FastAPI app factory + /healthz, /readyz)
    routers/                  (create dir)
      __init__.py             (create)
      songs.py                (create; POST/GET/LIST/PATCH/DELETE handlers)
    schemas/                  (create dir)
      __init__.py             (create)
      song.py                 (create; pydantic models mirroring OpenAPI)
    repo/                     (create dir)
      __init__.py             (create)
      contract.py             (create; repository interface only, no DB)
      fake.py                 (create; in-memory fake used in Phase 2 tests)
  tests/
    unit/
      test_post_songs.py      (create for POST-01/02)
    golden/
      conftest.py             (create; fixtures: fixed clock/uuid, golden loader)
      test_api_golden.py      (create; HTTP-level golden tests)
      test_openapi_snapshot.py (create later; snapshot once API stabilizes)

# Per-ticket file whitelist
- May create/modify:
  - `src/song_library/routers/songs.py`
  - `src/song_library/schemas/song.py`
  - `src/song_library/repo/fake.py`
  - `tests/golden/*` and `tests/unit/test_post_songs.py` (or the test for this endpoint)
- Do NOT touch other files.
- If a file doesn’t exist, CREATE it following the Initial File Manifest.

# Show changes as:
1) Path header (e.g., services/song-library/src/song_library/routers/songs.py)
2) Minimal snippet(s) around the change
3) Brief note why each change is needed (link to test ID)
4) End with the Change Report list (NEW/MODIFIED/DELETED)

# Phase & Acceptance
Current Phase: **2 — API-first (repository mocked)**
Service: **song-library**
Acceptance: tests pass, goldens match byte-for-byte, spec clauses satisfied.
Tests are executed via make test; assume pytest is configured and goldens live under docs/services/song-library/spec/golden/responses

REPO SHAPE (reference & guardrails)

```
Makefile
Tiltfile
docs/
  services/
    song-library/
      spec/
        song-library.md
        openapi.v1.yaml
        test-matrix.md
        build-sequence.md
        changelog.md
        golden/
          responses/
services/
  song-library/        ← new microservice under development
    Dockerfile.dev
    Dockerfile.prod
    Makefile
    pyproject.toml
    src/
      svc_song_library/
        main.py
        services/
          __init__.py
        contracts/
          __init__.py
        adapters/
          __init__.py
    tests/
      unit/
        test_api.py
        test_post_songs.py
  svc-1/                   ← reference service (layout & conventions)
  svc-2/                   ← reference service (layout & conventions)
helm/
  svc-1/
  svc-2/
k8s/
  dev/
    kustomization.yaml
    namespace.yaml

```

===== SPEC: song-library.md (reference) =====
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

===== END SPEC =====

===== OPENAPI: openapi.v1.yaml (reference) =====
```yaml
openapi: 3.1.0
info:
  title: KJBot song-library API
  version: 1.0.0
servers:
  - url: /api/v1
paths:
  /healthz:
    get:
      summary: Liveness probe
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  status: { type: string, enum: [ok] }
  /readyz:
    get:
      summary: Readiness probe
      responses:
        '200':
          description: Ready
          content:
            application/json:
              schema:
                type: object
                properties:
                  status: { type: string, enum: [ready] }
        '503':
          description: Not Ready
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
  /songs:
    post:
      summary: Create a song
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/SongCreate' }
      responses:
        '201':
          description: Created
          headers:
            Location:
              description: URL of created resource
              schema: { type: string }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Song' }
        '400': { $ref: '#/components/responses/ValidationError' }
        '409': { $ref: '#/components/responses/ConflictError' }
    get:
      summary: List songs
      parameters:
        - { name: q, in: query, schema: { type: string } }
        - { name: page, in: query, schema: { type: integer, minimum: 1, default: 1 } }
        - { name: limit, in: query, schema: { type: integer, minimum: 1, maximum: 100, default: 25 } }
        - { name: sort, in: query, schema: { type: string, enum: [title, artist, created_at, updated_at, duration] } }
        - { name: order, in: query, schema: { type: string, enum: [asc, desc] } }
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema: { $ref: '#/components/schemas/SongList' }
  /songs/{song_id}:
    parameters:
      - name: song_id
        in: path
        required: true
        schema: { type: string, format: uuid }
    get:
      summary: Get song by id
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Song' }
        '404': { $ref: '#/components/responses/NotFoundError' }
    patch:
      summary: Update song
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/SongUpdate' }
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Song' }
        '400': { $ref: '#/components/responses/ValidationError' }
        '404': { $ref: '#/components/responses/NotFoundError' }
        '409': { $ref: '#/components/responses/ConflictError' }
    delete:
      summary: Delete song
      responses:
        '204': { description: No Content }
        '404': { $ref: '#/components/responses/NotFoundError' }
components:
  schemas:
    MediaRef:
      type: object
      required: [bucket, key]
      properties:
        bucket: { type: string, minLength: 1, maxLength: 200 }
        key: { type: string, minLength: 1, maxLength: 1024 }
    Song:
      type: object
      required: [song_id, title, artist, duration, media_file, created_at, updated_at]
      properties:
        song_id: { type: string, format: uuid }
        title: { type: string, minLength: 1, maxLength: 200 }
        artist: { type: string, minLength: 1, maxLength: 200 }
        duration: { type: integer, minimum: 0, maximum: 86400 }
        media_file: { $ref: '#/components/schemas/MediaRef' }
        created_at: { type: string, format: date-time }
        updated_at: { type: string, format: date-time }
    SongCreate:
      type: object
      required: [title, artist, duration, media_file]
      properties:
        title: { type: string, minLength: 1, maxLength: 200 }
        artist: { type: string, minLength: 1, maxLength: 200 }
        duration: { type: integer, minimum: 0, maximum: 86400 }
        media_file: { $ref: '#/components/schemas/MediaRef' }
    SongUpdate:
      type: object
      properties:
        title: { type: string, minLength: 1, maxLength: 200 }
        artist: { type: string, minLength: 1, maxLength: 200 }
        duration: { type: integer, minimum: 0, maximum: 86400 }
        media_file: { $ref: '#/components/schemas/MediaRef' }
    SongList:
      type: object
      required: [items, page, limit, total, has_next]
      properties:
        items:
          type: array
          items: { $ref: '#/components/schemas/Song' }
        page: { type: integer }
        limit: { type: integer }
        total: { type: integer }
        has_next: { type: boolean }
    Error:
      type: object
      properties:
        error:
          type: object
          required: [code, message]
          properties:
            code: { type: string }
            message: { type: string }
            details: { type: object }
  responses:
    ValidationError:
      description: Validation error
      content:
        application/json:
          schema: { $ref: '#/components/schemas/Error' }
    NotFoundError:
      description: Resource not found
      content:
        application/json:
          schema: { $ref: '#/components/schemas/Error' }
    ConflictError:
      description: Conflict
      content:
        application/json:
          schema: { $ref: '#/components/schemas/Error' }
```

===== END OPENAPI =====

===== BUILD SEQUENCE ======

# Dev Sequence — song-library Microservice Build

## Phase 0 — Spec ready
- Finish `docs/services/song-library/spec/*`:
  - `song-library.md`
  - `openapi.v1.yaml`
  - `test-matrix.md`
  - `golden/`
- Confirm contracts, validations, and error model are reviewed and stable.
- ✅ **Gate:** spec + matrix approved.

---

## Phase 1 — Service skeleton (minimal scaffolding)
- Copy `services/svc-1 → services/svc-song-library`.
- Implement FastAPI app scaffold with `/healthz` and `/readyz`.
- Add minimal `Makefile`, `Dockerfile.dev`, and `pyproject.toml`.
- Reuse top-level `Tiltfile`; Helm chart may remain a stub.
- ✅ **Gate:** `make test` runs locally; health endpoints pass.

---

## Phase 2 — API-first TDD (repository mocked)
- Implement endpoints one test case at a time, starting with **POST /songs**.
- Use an **in-memory fake repository** or monkeypatch to simulate persistence.
- Drive implementation from the test matrix and goldens.
- Produce and lock an **OpenAPI snapshot** after the first endpoint goes green.
- ✅ **Gate:** All golden HTTP tests for implemented endpoints pass with fake repo.

---

## Phase 3 — Repository & DB
- Add migration `0001_create_songs.sql` from spec DDL.
- Introduce **Testcontainers Postgres** (or docker-compose) for repo tests.
- Implement repository behavior per spec:
  - Normalization, timestamp updates.
  - Pagination, sort, filters.
  - Error mapping (constraint → code → HTTP).
- ✅ **Gate:** 5–8 repository tests pass; `/readyz` checks DB/migrations.

---

## Phase 4 — Thin integration
- Run app ↔ repo ↔ Postgres.
- Add **1 happy-path integration test per endpoint**.
- Verify runtime OpenAPI = stored snapshot.
- ✅ **Gate:** All integration tests green; OpenAPI verified.

---

## Phase 5 — Packaging & deployables
- Complete Helm chart (service, deployment, ingress, config placeholders).
- Add `values.dev.yaml` for local cluster.
- Ensure Tilt can bring up service + Postgres.
- ✅ **Gate:** Tilt smoke passes (`/healthz`, `/readyz`, one happy path).

---

## Phase 6 — Ops polish
- Add structured logging (`request_id`).
- Add metrics and readiness refinements.
- Implement CI jobs:
  - pre-commit hooks.
  - golden drift checks.
  - OpenAPI validation (Spectral).
- ✅ **Gate:** CI runs green; Helm & Tilt usable across dev/stg/prod.


===== END BUILD SEQUENCE ======


===== TICKET ========
# Phase & Spec
Phase: 2 — API-first (repository mocked)
Service: song-library

# Ticket
Test case ID: POST-01 (see test-matrix.md §1).
Implement **POST-01 (happy path)** for `POST /api/v1/songs`
Golden: `docs/services/song-library/spec/golden/responses/post_01.json`
{
  "song_id": "00000000-0000-4000-8000-000000000001",
  "title": "Don't Stop Believin'",
  "artist": "Journey",
  "duration": 251,
  "media_file": { "bucket": "kjbot-media", "key": "tracks/journey/dont-stop-believin.mp3" },
  "created_at": "2025-09-30T00:00:00Z",
  "updated_at": "2025-09-30T00:00:00Z"
}

Acceptance: status 201; `Location: /api/v1/songs/{song_id}`; body matches golden byte-for-byte.

# Spec excerpts (authoritative)
- Create request (`SongCreate`): requires `title`, `artist`, `duration` (0..86400), and `media_file: {bucket, key}`. (Spec §3.1)
- Response on success: `201 Song` with `Location` header `/api/v1/songs/{song_id}`. (Spec §4.2)
- Validation & normalization: trim/collapse spaces in `title`/`artist`; must be non-empty after trim. (Spec §5.1)
- Error envelope exists but not expected for happy path. (Spec §3.1 error, §5.2)

# OpenAPI slice (for this path)
paths:
  /songs:
    post:
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/SongCreate' }
      responses:
        '201':
          headers:
            Location: { schema: { type: string } }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Song' }

# Test Matrix (rows in scope)
| Case ID | Description           | Expected |
|---------|-----------------------|----------|
| POST-01 | Happy path create     | 201 + Location + Song body = golden |

# Constraints for this ticket
- Use a **fake repository** (monkeypatch/in-memory) to satisfy Phase 2.
- Generate deterministic UUID/time if tests require it (fixtures).
- No DB/Helm/Tilt work in this phase.

# Your turn
Use headings Plan, Diffs, Why, Change Report, and Next in your reply.
A) Confirm POST-01.  
B) Plan (3–6 bullets tied to POST-01).  
C) Minimal diffs/snippets only for POST-01.  
D) Why (map to spec).  
E) Stop after green.

When POST-01 passes and goldens match, return control to me for the next ticket (POST-02).
