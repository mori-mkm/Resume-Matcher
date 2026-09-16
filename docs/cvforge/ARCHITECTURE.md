# CVForge — Architecture

> Status: **analysis only**. No engine code exists yet. This document records the
> architecture found in the fork, the target architecture, and where the two meet.
> Upstream docs stay authoritative for current behavior: [docs/agent/README.md](../agent/README.md).

---

## 1. Current architecture (as found)

Fork of `srbhr/Resume-Matcher` at `ab2b370`. Monorepo, two apps, no service boundary
beyond HTTP.

```
apps/backend   FastAPI + Python 3.13, SQLAlchemy 2.0 async over SQLite (aiosqlite)
apps/frontend  Next.js 16 + React 19, Tailwind v4, vitest
```

### 1.1 Backend module map

| Module | LOC | Role |
|---|---|---|
| `app/main.py` | 141 | App factory, lifespan, CORS, router mounting under `/api/v1` |
| `app/config.py` | 426 | Settings + `config.json` store + provider key mapping |
| `app/llm.py` | 1718 | LiteLLM wrapper: `get_llm_config`, `get_router`, `complete`, `complete_json`, `check_llm_health`, JSON extraction, timeout/temperature policy |
| `app/models.py` | 162 | ORM: `Resume`, `Job`, `Improvement`, `TailoringPreview`, `Application`, `ApiKey` |
| `app/database.py` | 1245 | Async facade over the ORM; preserves TinyDB-era dict semantics |
| `app/db_engine.py` | 79 | Engine factories, WAL/FK pragmas, `init_models_sync` (**`create_all` + hand-rolled additive ALTERs — no Alembic**) |
| `app/crypto.py` | 129 | Fernet encryption for stored API keys |
| `app/pdf.py` | 905 | Headless Chromium (Playwright) render of the `/print/*` routes |
| `app/preview.py` | 43 | Fingerprints + preview claim/conflict errors |
| `app/ai_budget.py`, `app/ai_limits.py` | 132 | Per-operation AI budgets and error envelopes |

Routers (mounted at `/api/v1`): `health`, `config`, `resumes` (2489 LOC — upload, improve,
improve/preview, improve/confirm, PDF, cover letter, outreach, interview prep), `jobs`,
`enrichment`, `applications` (Kanban tracker), `resume_wizard`.

Services: `parser.py` (document → markdown → JSON), `improver.py` (keyword extraction,
diff generation, diff application, skill plans, resume diffing), `refiner.py` (keyword
match, gap analysis, AI-phrase removal, master-alignment validation), `resume_preservation.py`
(`finalize_ai_resume`, `validate_confirmed_resume`, `grounding_review_warnings`), `ats.py`
(`compute_ats_score`), `cover_letter.py`, `interview_prep.py`, `resume_wizard.py`.

### 1.2 Current tailoring flow

```
upload resume ─► parser.parse_document ─► parser.parse_resume_to_json ─► Resume.processed_data
upload JD ─────► Job.content
POST /resumes/improve[/preview] ─┬─ improver.extract_job_keywords        (LLM)
                                 ├─ improver.generate_skill_target_plan  (LLM)
                                 ├─ improver.generate_resume_diffs       (LLM)
                                 ├─ improver.apply_diffs                 (deterministic)
                                 ├─ improver.verify_diff_result          (deterministic guardrail)
                                 ├─ resume_preservation.finalize_ai_resume        (deterministic)
                                 ├─ resume_preservation.grounding_review_warnings (deterministic)
                                 └─ ats.compute_ats_score                (deterministic)
POST /resumes/improve/confirm ───┬─ resume_preservation.validate_confirmed_resume
                                 ├─ new Resume row
                                 └─ auto-creates an Application (tracker) card
GET /resumes/{id}/pdf ───────────► pdf.py ─► Chromium ─► /print/resumes/{id}
```

**This is already a preview → human confirm → persist pipeline.** It is the structural
ancestor of the CVForge Final Review checkpoint, and the reason we extend rather than rewrite.

### 1.3 Frontend map

Routes: `/`, `/dashboard`, `/builder`, `/tailor`, `/tracker`, `/resumes/[id]`,
`/resume-wizard`, `/settings`, plus `/print/resumes/[id]` and `/print/cover-letter/[id]`
(consumed by Chromium, not by humans).

Templates in `components/resume/`: `swiss-single`, `swiss-two-column`, `modern`,
`modern-two-column`, `latex`, `clean`, `vivid` — registered by the `TemplateType` union in
`lib/types/template-settings.ts`. Default today is `swiss-single`.

i18n: `messages/{en,es,fr,ja,ko,pt-BR,zh}.json`, parity enforced by
`apps/backend/tests/unit/test_check_locale_parity.py`.

### 1.4 Provider / Ollama posture

`ollama` is a first-class provider (`app/config.py:233,262,288`; `app/llm.py:231,573,686,855,1349`).
LiteLLM routes it as `ollama_chat/` (`/api/chat`, message array). `ollama` and
`openai_compatible` are the two providers allowed to run without an API key
(`app/llm.py:855`, `app/routers/health.py:47`). The zero-cost runtime requirement is
therefore already satisfiable for **chat**. It is **not** satisfiable for embeddings today:
there is no embedding call anywhere in `apps/backend/app/`.

---

## 2. Target architecture (CVForge Engine)

```
Job Description
  └─► Job Analyzer          requirements, weights, must-have vs nice-to-have
        └─► Evidence Store          the single factual source of truth
              └─► Exact + Semantic Matching
                    └─► Evidence Ranking          requirement → evidence → score → reason
                          └─► HUMAN: Evidence Review     approve / remove / add / block
                                └─► Writer                  (LLM, evidence-constrained)
                                      └─► Fact Validator    restrictions are hard gates
                                            └─► ATS Validator
                                                  └─► Renderer ─► PDF
                                                        └─► HUMAN: Final Review ─► approved
                                                              └─► Feedback Store
```

### 2.1 Central principle

> An LLM is never used for work that deterministic code can do.

Ladder — stop at the first rung that holds:

1. deterministic Python
2. exact / normalized matching
3. local embeddings (Ollama)
4. rules
5. cache
6. LLM — only for interpretation or rewriting

Stage-by-stage intended rung:

| Stage | Rung | Note |
|---|---|---|
| Job Analyzer | 6, cached | LLM extraction is genuinely interpretive; cache on JD hash (`preview.job_fingerprint` already exists) |
| Evidence Store | 1 | pure CRUD |
| Exact matching | 2 | reuse `refiner._keyword_in_text`, `improver._normalize_skill_key` |
| Semantic matching | 3 | Ollama embeddings via `litellm.aembedding` — **no new dependency** |
| Ranking | 1 + 4 | reproducible score, human-readable reason |
| Evidence Review | human | — |
| Writer | 6 | the only unavoidable generation step |
| Fact Validator | 1 | extend `resume_preservation.py` |
| ATS Validator | 1 | extend `ats.compute_ats_score` |
| Renderer / PDF | 1 | unchanged `app/pdf.py` |
| Final Review | human | — |
| Feedback Store | 1 | CRUD + a rule filter applied before ranking |

### 2.2 Evidence Store shape

```
id            str   stable identifier
source        str   where the fact comes from (resume entry, project, certificate, …)
source_type   str   experience | project | education | certification | manual | …
fact          str   the claim, in the words of the person it belongs to
tags          list  free-form domain tags
technologies  list  normalized technology tokens (exact-matching key)
metrics       list  numbers the fact is allowed to state
strength      str   how well-supported the fact is
status        str   lifecycle (draft / active / retired / blocked)
restrictions  list  hard gates, see below
```

Restrictions are **gates, not warnings**. A claim that violates one is rejected:

- `do_not_claim_production`
- `do_not_claim_architecture_ownership`
- `do_not_modify_metric`
- `do_not_modify_job_title`

**Invariant:** no content reaches a rendered resume without resolving to at least one
allowed evidence record. This is the Fact Validator contract, and it is stricter than the
existing `grounding_review_warnings`, which warns rather than blocks.

### 2.3 Human-in-the-loop

| Checkpoint | Shows | User can |
|---|---|---|
| Evidence Review | `requirement → evidence → score → reason` | approve, remove, add, block |
| Final Review | PDF, ATS analysis, requirement coverage, factual warnings, layout warnings | approve — the only path to `approved` |

### 2.4 Feedback Store

Human decisions persist as reusable rules — for example *"never use project X in
applications"* — and are applied as a filter **before** ranking, so a blocked item never
reaches the review table again.

---

## 3. Integration points

Where engine code attaches, ordered by how little of the existing surface it disturbs.

| # | Point | File | How |
|---|---|---|---|
| 1 | New ORM tables | `app/models.py` | Additive `Base` subclasses; `init_models_sync` creates them. Zero migration risk. |
| 2 | New DB accessors | `app/database.py` | New methods on the existing facade. Existing ones untouched. |
| 3 | New router | `app/routers/cvforge.py` (new) + `app/main.py` | One `include_router` line under `/api/v1`. |
| 4 | Engine services | `app/services/cvforge/` (new package) | Imports existing services; existing services import nothing new. |
| 5 | Embeddings | `app/llm.py` | New `embed()` alongside `complete()`, reusing `get_llm_config` and the provider prefix map. |
| 6 | Fact Validator | `app/services/resume_preservation.py` | New evidence-aware function next to `grounding_review_warnings`. Existing warn-only behavior untouched. |
| 7 | ATS Validator | `app/services/ats.py` | New coverage function; `compute_ats_score` keeps its signature — the tailor UI depends on it. |
| 8 | Renderer | `app/pdf.py`, `/print/resumes/[id]` | No change. A `classic-ats` template is additive. |
| 9 | Job Analyzer cache | `app/preview.py`, `Job.metadata_json` | `job_fingerprint` already exists; `metadata_json` already stores `job_keywords_hash`. |
| 10 | Frontend API client | `apps/frontend/lib/api/cvforge.ts` (new) | Mirrors `tracker.ts`. |
| 11 | Review UI | `apps/frontend/app/(default)/…` | New routes. `/tailor` stays until the engine replaces it. |
| 12 | Feature flags | `app/routers/config.py:258-296` | `enable_cover_letter` / `enable_outreach_message` / `enable_interview_prep` **already default to `False`** — UI deactivation needs no new mechanism. |

---

## 4. File disposition

### Kept as-is
`app/main.py`, `app/db_engine.py`, `app/crypto.py`, `app/pdf.py`, `app/preview.py`,
`app/ai_budget.py`, `app/ai_limits.py`, `app/config_cache.py`, `app/database.py`,
`app/routers/health.py`, `app/routers/jobs.py`, `app/routers/config.py`,
`app/routers/applications.py`, `app/services/parser.py`, `app/services/resume_wizard*.py`,
`app/scripts/`, `e2e_monitor/`, `components/resume/*`, `components/preview/*`,
`components/tracker/*`, `components/ui/*`, `lib/api/client.ts`, `lib/i18n/*`, `app/print/*`,
and every existing test.

### Extended
| File | Extension |
|---|---|
| `app/models.py` | `Evidence`, `EvidenceLink`, `FeedbackRule`, `JobRequirement` tables |
| `app/database.py` | accessors for the above |
| `app/llm.py` | `embed()` for local embeddings |
| `app/services/ats.py` | requirement-coverage scoring |
| `app/services/resume_preservation.py` | evidence-backed hard validation |
| `app/services/refiner.py` | expose matching primitives to the engine |
| `app/services/improver.py` | reuse `extract_job_keywords` as Job Analyzer v0 |
| `app/prompts/templates.py` | Job Analyzer and Writer prompts |
| `lib/types/template-settings.ts` | add `classic-ats` to `TemplateType` |
| `messages/*.json` | strings for the two review checkpoints |

### Replaced (superseded; the old path stays until the engine is proven)
| Current | Superseded by |
|---|---|
| `improver.generate_resume_diffs` (LLM rewrites from the resume) | Writer constrained by approved evidence |
| `improver.generate_skill_target_plan` | Evidence Ranking |
| `refiner.analyze_keyword_gaps` | Job Analyzer + Matching |
| `grounding_review_warnings` (warn) | Fact Validator (block) |
| `/tailor` page | Evidence Review + Final Review flow |

### Deactivated in the UI only (code stays, fully tested)
Cover Letter (`services/cover_letter.py`, `components/builder/cover-letter-*.tsx`,
`/print/cover-letter/[id]`), Outreach Message, and Interview Preparation
(`services/interview_prep.py`, `components/builder/interview-prep-view.tsx`).
All three are already gated by config flags that default to `False`.

### Removed
Nothing.

---

## 5. Known gaps

| Gap | Impact |
|---|---|
| No embeddings anywhere in the backend | Semantic matching needs `embed()` first; `litellm` already ships Ollama embedding support, so no new dependency |
| No `classic-ats` template | The intended future default does not exist |
| No structured evidence model | `Resume.processed_data` is a document blob, not addressable facts |
| Grounding is advisory | `grounding_review_warnings` warns; CVForge needs a blocking gate |
| No feedback persistence | Human decisions are lost between applications |
| Requirements are unstructured | `extract_job_keywords` returns keyword lists, not weighted requirements |
| Backend suite cannot run on Windows | See [MIGRATION_PLAN.md](./MIGRATION_PLAN.md) §1 |
| `routers/resumes.py` is 2489 LOC | Highest-risk file to touch; the engine should add a router rather than edit it |
