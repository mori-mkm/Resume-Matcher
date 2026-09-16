# CVForge — Migration Plan

> Phase 1 (analysis, subagents, docs) is complete. **Phase 2 has not started.**
> Target architecture: [ARCHITECTURE.md](./ARCHITECTURE.md). Rationale: [DECISIONS.md](./DECISIONS.md).

---

## 1. Baseline (recorded 2026-09-16, before any change)

| Check | Command | Result |
|---|---|---|
| Frontend tests | `cd apps/frontend && npm run test` | ✅ **55 files, 500 tests passed** |
| Frontend lint | `cd apps/frontend && npm run lint` | ✅ **clean, exit 0** |
| Backend tests | `cd apps/backend && uv run pytest` | ❌ **0 passed, 1056 errors, 11 skipped, 2 deselected** |

### 1.1 Preexisting backend failure — Windows only, not caused by CVForge

Every backend test errors in setup with:

```
tests.conftest.UnexpectedNetworkAccess: External network access blocked in
deterministic backend tests
  socket.py:623 in _fallback_socketpair -> csock.connect((addr, port))
  proactor_events.py:786 in _make_self_pipe -> socket.socketpair()
```

Root cause: the autouse `deny_external_network` fixture
(`apps/backend/tests/conftest.py:56-73`) monkeypatches `socket.socket.connect`,
`socket.socket.connect_ex` and `socket.create_connection` globally. On Windows,
`socket.socketpair()` has no native implementation and falls back to a loopback
`listen`/`connect` pair — which the guard blocks. `asyncio`'s `ProactorEventLoop` calls
`socketpair()` to build its self-pipe, and `pytest-asyncio` runs in `asyncio_mode = auto`,
so **every** async test dies before its body runs.

On Linux and macOS `socketpair()` is a real syscall and never calls `connect`, so the
suite is presumed green there — that is where upstream's `~444`-test figure comes from.
This count (1067 collected) is much higher than the number in `.claude/CLAUDE.md`, which
is itself stale.

**Not fixed in this phase.** Modifying existing tests is out of scope per
`.claude/CLAUDE.md` § Out of Scope. The fix is small and belongs in its own change:
let the guard pass loopback connections through, or install the guard on
`socket.create_connection` plus an httpx transport hook rather than on
`socket.socket.connect`.

**Consequence for Phase 2:** until this is resolved, backend work on this machine has no
test signal. Either fix the guard first, or run the backend suite in WSL/Docker
(`docker-compose.yml` is present) before every backend commit.

### 1.2 Post-change verification

Phase 1 touched only `.claude/`, `docs/` and `README` prose. Frontend lint and tests were
re-run after the changes and are unchanged: **500 passed, lint clean**. No application file
was modified, so no runtime behavior changed.

---

## 2. Phase sequence

Each phase ends green (on a platform where the backend suite runs) and ships behind a flag.
No phase removes a feature.

| Phase | Scope | Exit criterion |
|---|---|---|
| **1 — Analysis** ✅ | Architecture map, integration points, risks, subagents, docs | This document |
| **2 — Evidence Store** | `Evidence` + `FeedbackRule` tables, CRUD router, evidence extraction from an existing resume, minimal Swiss UI | Evidence survives a restart; no existing endpoint changes behavior |
| **3 — Job Analyzer** | Weighted structured requirements, cached on JD fingerprint | Same JD twice ⇒ one LLM call |
| **4 — Matching + Ranking** | Exact → embeddings → rules; `requirement → evidence → score → reason` | Score reproducible for fixed input; runs with Ollama only |
| **5 — Evidence Review** | Checkpoint 1 UI: approve / remove / add / block | No resume can be generated without passing it |
| **6 — Writer + Fact Validator** | Evidence-constrained generation; restrictions as hard gates | Unbacked claim ⇒ rejected, not warned |
| **7 — ATS Validator + Final Review** | Checkpoint 2 UI; `approved` status | `approved` reachable only through the checkpoint |
| **8 — Feedback Store wiring** | Rules filter candidates before ranking | A blocked project never reappears |
| **9 — Classic ATS + UI cleanup** | `classic-ats` template becomes default; Cover Letter / Outreach / Interview Prep hidden | Flags off, code and tests intact |

### 2.1 Phase 2 detail — Evidence Store

Ships:
- `Evidence` and `FeedbackRule` ORM models in `app/models.py` (additive; `create_all` handles them)
- Facade accessors in `app/database.py`
- Pydantic schemas in `app/schemas/cvforge.py`
- `app/routers/cvforge.py` mounted at `/api/v1/cvforge`, one line in `app/main.py`
- Deterministic extraction of candidate evidence from `Resume.processed_data` — **no LLM**, one evidence record per experience bullet / project / certification
- `apps/frontend/lib/api/cvforge.ts` + a Swiss-styled list/edit view
- Unit tests for restrictions and extraction, integration tests for CRUD

Does **not** ship: matching, ranking, writing, validation, embeddings, review checkpoints.

Explicit decisions needed before starting — see [DECISIONS.md](./DECISIONS.md) § Open.

---

## 3. Regression risks

Ranked by expected damage.

| # | Risk | Where | Mitigation |
|---|---|---|---|
| R1 | **No backend test signal on Windows** | `tests/conftest.py:56-73` | Fix the guard in its own change, or run the suite in WSL/Docker before every backend commit. Blocking for Phase 2. |
| R2 | **Editing `routers/resumes.py` (2489 LOC)** — it owns upload, improve, preview, confirm, PDF and tracker auto-create, with concurrency tokens and cleanup tasks | `app/routers/resumes.py` | Add `routers/cvforge.py`. Do not edit the existing router. |
| R3 | **`database.py` reproduces TinyDB dict semantics** (absent-vs-null keys, `metadata_json` flattening) | `app/database.py`, `app/models.py:68-85` | Only add methods. Never change an existing return shape. |
| R4 | **No Alembic** — schema comes from `create_all` plus hand-rolled `PRAGMA table_info` ALTERs | `app/db_engine.py:63-79` | New *tables* are safe. A new *column on an existing table* needs an idempotent guard there and a test against a pre-existing DB file. |
| R5 | **Resume preservation is load-bearing** — `finalize_ai_resume` / `validate_confirmed_resume` protect dates, skills, custom sections and personal info | `app/services/resume_preservation.py` | Add functions beside them. Never change their signatures or thresholds. |
| R6 | **`compute_ats_score` feeds the tailor UI** | `app/services/ats.py:171` | New coverage function, existing signature frozen. |
| R7 | **Locale parity is a test** | `messages/*.json`, `tests/unit/test_check_locale_parity.py` | Every new string lands in all 7 locale files in the same commit. |
| R8 | **Golden evals drift** when prompts change | `tests/evals/` | Engine prompts are new prompt constants. Do not edit `IMPROVE_RESUME_PROMPTS`. |
| R9 | **Ollama models are weaker at JSON** | `app/llm.py:1025-1198` (`_supports_json_mode`, truncation detection) | Keep engine JSON schemas small and flat; stay on the existing `complete_json` retry path. |
| R10 | **Embeddings cost latency and a model download** | new `llm.embed()` | Cache by content hash; degrade to exact-match-only when Ollama has no embedding model. Never make embeddings a hard dependency. |
| R11 | **Feature-flag removal looks like data loss** | `Resume.cover_letter`, `outreach_message`, `interview_prep` columns | Hide the UI only. Keep the columns, the services, the print routes and the tests. |
| R12 | **PDF rendering is environment-sensitive** (Playwright, Windows Proactor loop) | `app/pdf.py`, `app/main.py:13-14` | Do not touch the renderer. A new template is CSS + a React component only. |

---

## 4. How to verify

```bash
# Backend  (from apps/backend) — see R1 on Windows
uv sync --extra dev
uv run pytest                 # default suite; LLM evals excluded
uv run pytest -m eval         # prompt-quality evals, needs a configured provider

# Frontend (from apps/frontend)
npm install
npm run test                  # vitest
npm run lint
npm run format
npm run build
```

Local push gate (not CI): `git config core.hooksPath .githooks` once per clone —
`.githooks/pre-push` runs the backend suite plus the locale-parity check and blocks red
pushes. There is deliberately no GitHub Actions PR gate; see `.githooks/README.md`.
