# KJBot — Docs / Spec Navigation Guide

This `docs/` folder is the **source of truth** for KJBot service specifications. It’s designed to serve two audiences equally:

- **Humans**: contributors, reviewers, and maintainers  
- **LLMs**: acting in a Python Dev role, able to “help themselves” to structured spec artifacts

---

## Structure

```bash
docs/
README.md ← You are here (navigation guide)
templates/ ← Reusable patterns for new services
spec-template.md
services/
<service-name>/
spec/
<service>.md ← Narrative spec (contracts, behavior, DB, ops)
openapi.v1.yaml ← Machine-readable contract (OpenAPI)
test-matrix.md ← Normative test cases (stable IDs)
golden/ ← Golden snapshots (expected responses)
changelog.md ← Spec-visible change log
```

---

## Roles of Each File

- **`<service>.md`** — The narrative “why and how.” Contracts, field rules, repo contract, behavior, and operational notes.  
- **`openapi.v1.yaml`** — Canonical machine-readable API schema. Must match the narrative spec and golden snapshots.  
- **`test-matrix.md`** — Enumerates normative test cases (POST-01, GET-02, etc.). Each case links to a golden snapshot file.  
- **`golden/`** — JSON files with deterministic example inputs/outputs. These act as truth for golden tests.  
- **`changelog.md`** — Append-only log of spec-visible changes, with references to test case IDs.  

---

## Conventions

- **Stable IDs**: every test case has a fixed ID (`POST-01`, `PATCH-04`, etc.) for traceability.  
- **Deterministic goldens**: freeze UUIDs and timestamps in snapshots so test comparisons are stable.  
- **Split pattern**: author specs initially as a single *bundle* doc; then split into these four artifacts.  
- **LLM-ergonomics**: services share the same folder layout and spec headings so an LLM can always find what it needs.  

---

## How to Use

- **Humans**:  
  - Start from `templates/spec-template.md` to draft new service specs.  
  - Use `test-matrix.md` + goldens as the acceptance criteria.  
  - Update `changelog.md` for any visible spec change.  

- **LLMs**:  
  - Retrieve `<service>.md` for narrative rules.  
  - Retrieve `openapi.v1.yaml` for schemas and endpoints.  
  - Retrieve `test-matrix.md` + `golden/` to know exactly what test cases must pass.  

---

## Definition of Done

A service spec is **done** when:  
- All cases in `test-matrix.md` are implemented and passing.  
- Golden snapshots match actual responses.  
- `openapi.v1.yaml` is in sync with the narrative spec.  
- `changelog.md` records the changes.  
