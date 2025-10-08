# Song-Library Service — Test Matrix (v0.1)

> This file enumerates normative test cases for the song-library service. Each case corresponds to an expected behavior in the spec. Goldens (example responses) live under `golden/`.

---

## 1. POST /api/v1/songs

| Case ID | Description                           | Input                             | Expected Status | Golden Snapshot                 |
| ------- | ------------------------------------- | --------------------------------- | --------------- | ------------------------------- |
| POST-01 | Happy path create                     | Valid SongCreate                  | 201 Created     | `golden/responses/post_01.json` |
| POST-02 | Validation: empty title               | `{ title: "" }`                   | 400             | `post_02.json`                  |
| POST-03 | Validation: negative duration         | `{ duration: -5 }`                | 400             | `post_03.json`                  |
| POST-04 | Validation: missing media_file fields | `{ media_file: { bucket: "x" } }` | 400             | `post_04.json`                  |
| POST-05 | (Future) Conflict on uniqueness       | Duplicate title+artist+duration   | 409             | `post_05.json`                  |

---

## 2. GET /api/v1/songs/{id}

| Case ID | Description      | Input ID         | Expected Status | Golden Snapshot |
| ------- | ---------------- | ---------------- | --------------- | --------------- |
| GET-01  | Happy path fetch | Existing song_id | 200 OK          | `get_01.json`   |
| GET-02  | Not found        | Random UUID      | 404             | `get_02.json`   |

---

## 3. GET /api/v1/songs (list)

| Case ID | Description           | Params                 | Expected Status | Golden Snapshot |
| ------- | --------------------- | ---------------------- | --------------- | --------------- |
| LIST-01 | Default paging        | none                   | 200 OK          | `list_01.json`  |
| LIST-02 | With filter `q`       | `q=journey`            | 200 OK          | `list_02.json`  |
| LIST-03 | With sort + order     | `sort=title&order=asc` | 200 OK          | `list_03.json`  |
| LIST-04 | Invalid sort param    | `sort=bogus`           | 400             | `list_04.json`  |
| LIST-05 | Pagination beyond end | `page=99&limit=50`     | 200 OK          | `list_05.json`  |

---

## 4. PATCH /api/v1/songs/{id}

| Case ID  | Description                      | Patch Body                           | Expected Status | Golden Snapshot |
| -------- | -------------------------------- | ------------------------------------ | --------------- | --------------- |
| PATCH-01 | Happy path update                | `{ title: "New Title" }`             | 200 OK          | `patch_01.json` |
| PATCH-02 | Validation: invalid duration     | `{ duration: 90000 }`                | 400             | `patch_02.json` |
| PATCH-03 | Validation: half-specified media | `{ media_file: { bucket: "only" } }` | 400             | `patch_03.json` |
| PATCH-04 | Not found                        | Random UUID                          | 404             | `patch_04.json` |
| PATCH-05 | (Future) Conflict on uniqueness  | Duplicate title+artist+duration      | 409             | `patch_05.json` |

---

## 5. DELETE /api/v1/songs/{id}

| Case ID | Description       | Input ID         | Expected Status | Golden Snapshot |
| ------- | ----------------- | ---------------- | --------------- | --------------- |
| DEL-01  | Happy path delete | Existing song_id | 204 No Content  | N/A             |
| DEL-02  | Not found         | Random UUID      | 404             | `del_02.json`   |

---

## 6. Health/Ready Endpoints

| Case ID | Endpoint     | Condition                 | Expected Status         | Golden Snapshot |
| ------- | ------------ | ------------------------- | ----------------------- | --------------- |
| HLTH-01 | GET /healthz | Always                    | 200 OK                  | `hlth_01.json`  |
| RDY-01  | GET /readyz  | DB connected, migrations  | 200 OK                  | `rdy_01.json`   |
| RDY-02  | GET /readyz  | DB down or migration miss | 503 Service Unavailable | `rdy_02.json`   |

---

## Notes

* Goldens should strip or freeze non-deterministic fields (UUIDs, timestamps).
* Each case ID is stable for traceability in changelogs and test code.
* Future enhancements (idempotency, uniqueness) get provisional cases now (`POST-05`, `PATCH-05`).
