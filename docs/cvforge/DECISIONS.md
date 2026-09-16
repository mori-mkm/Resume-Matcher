# CVForge — Decision Log

Append-only. Each entry: context, decision, consequence. Newest last.

---

## D-001 — Fork and extend, do not rewrite
**Date:** 2026-09-16 · **Status:** accepted

**Context.** The upstream Resume-Matcher already carries a preview → confirm → persist
tailoring pipeline, a multi-provider LLM layer with working Ollama support, a Playwright PDF
renderer, seven resume templates, a Kanban tracker, ~1067 backend tests, 500 frontend tests
and golden evals.

**Decision.** CVForge is built by extension. New engine code lands in new modules; existing
modules gain functions, never changed signatures.

**Consequence.** Slower initial progress, near-zero regression risk, and the whole existing
test suite keeps guarding us. Dead-looking upstream code stays until the replacement is
proven in production use.

---

## D-002 — LLM is the last rung, never the first
**Date:** 2026-09-16 · **Status:** accepted

**Context.** The current pipeline spends three LLM calls per tailoring run
(`extract_job_keywords`, `generate_skill_target_plan`, `generate_resume_diffs`). Two of
those are arguably deterministic work.

**Decision.** Ladder: deterministic Python → exact matching → local embeddings → rules →
cache → LLM. An LLM call must be justified by genuine interpretation or rewriting.

**Consequence.** Only Job Analyzer and Writer keep LLM calls. Every score must be
reproducible for a fixed input, which makes deterministic testing possible and makes
"why did it pick this?" answerable.

---

## D-003 — Ollama is the reference runtime
**Date:** 2026-09-16 · **Status:** accepted

**Context.** Operating cost must be zero. `ollama` is already a first-class provider and
one of two providers allowed to run without an API key (`app/llm.py:855`).

**Decision.** Every engine feature must work on Ollama alone. Other providers stay
supported and optional.

**Consequence.** Prompts stay small and schemas flat (local models are weaker at strict
JSON). Embeddings must come from a local model. Nothing may hard-require a paid provider.

---

## D-004 — Evidence Store is the only factual source
**Date:** 2026-09-16 · **Status:** accepted

**Context.** Today the LLM rewrites resume content directly and
`resume_preservation.grounding_review_warnings` only *warns* about ungrounded claims.

**Decision.** No content reaches a rendered resume without resolving to at least one
allowed evidence record. `restrictions` are hard gates, not advisories — a violated
restriction rejects the claim.

**Consequence.** The Writer stops being a rewriter and becomes an assembler over approved
facts. `grounding_review_warnings` stays untouched for the legacy `/tailor` path; the
blocking validator is a new, separate function.

---

## D-005 — Both human checkpoints are mandatory
**Date:** 2026-09-16 · **Status:** accepted

**Context.** The existing `improve/preview` → `improve/confirm` flow already proves users
accept a review step.

**Decision.** Evidence Review (`requirement → evidence → score → reason`; approve / remove /
add / block) and Final Review (PDF, ATS, coverage, factual warnings, layout warnings) are
both required. `approved` is reachable only through Final Review.

**Consequence.** No fire-and-forget generation path will exist, including for automation.

---

## D-006 — Cover Letter, Outreach and Interview Prep: hidden, not deleted
**Date:** 2026-09-16 · **Status:** accepted

**Context.** All three are already gated by `enable_cover_letter`,
`enable_outreach_message` and `enable_interview_prep`, which **default to `False`**
(`app/routers/config.py:258-296`).

**Decision.** Deactivate in the UI by leaving the flags off and hiding the settings toggles.
Keep the services, columns, print routes, components and tests.

**Consequence.** Zero deletion risk, zero test churn, and re-enabling is a flag flip. The
DB columns stay populated for existing rows.

---

## D-007 — New tables only; no schema migration framework
**Date:** 2026-09-16 · **Status:** accepted

**Context.** There is no Alembic. Schema comes from `Base.metadata.create_all` plus
hand-rolled idempotent `PRAGMA table_info` ALTERs in `app/db_engine.py:63-79`.

**Decision.** Evidence work adds **new tables**, which `create_all` handles for free. Adding
a column to an existing table requires an idempotent guard in `init_models_sync` plus a test
against a pre-existing database file.

**Consequence.** Alembic is not introduced yet. Revisit if the engine ever needs a
destructive schema change.

---

## D-008 — Six project subagents, tool-restricted
**Date:** 2026-09-16 · **Status:** accepted

**Decision.** `.claude/agents/`: `architect` and `reviewer` (read-only), plus
`evidence-engineer`, `matching-engineer`, `frontend-engineer` and `quality-engineer`
(read/write within a named scope). Descriptions stay one line to keep context cheap.

**Consequence.** Exploration is not duplicated across agents; each returns only findings,
files, risks, recommendations and decisions needed.

---

## D-009 — Windows backend test failure is preexisting and out of scope
**Date:** 2026-09-16 · **Status:** accepted

**Context.** All 1056 backend tests error on this machine. The autouse
`deny_external_network` fixture (`tests/conftest.py:56-73`) patches `socket.socket.connect`;
on Windows `socket.socketpair()` falls back to a loopback connect, which asyncio's
`ProactorEventLoop` needs for its self-pipe. Platform-specific, not caused by CVForge.

**Decision.** Do not fix it inside Phase 1 — modifying existing tests is out of scope per
`.claude/CLAUDE.md`. Record it, and fix it in a dedicated change before Phase 2 backend work.

**Consequence.** Until then, backend changes must be verified in WSL or Docker. Recorded as
risk R1 in [MIGRATION_PLAN.md](./MIGRATION_PLAN.md).

---

## Open — decisions needed before Phase 2

| # | Question | Options |
|---|---|---|
| O-1 | Is evidence global, or scoped to a master resume? | Global store (one truth, cross-resume reuse) vs. per-resume (simpler, duplicated facts) |
| O-2 | How is evidence first populated? | Deterministic extraction from `Resume.processed_data` · manual entry · LLM-assisted extraction with human confirmation |
| O-3 | Is `strength` an enum or a number? | Enum (`weak` / `moderate` / `strong`) reads better in review; a float ranks better |
| O-4 | Are `restrictions` a closed vocabulary or free-form? | Closed is enforceable and testable; free-form is flexible but unvalidatable |
| O-5 | Which Ollama embedding model is the reference? | e.g. `nomic-embed-text` or `mxbai-embed-large` — fixes the vector dimension and the cache format |
| O-6 | Where do evidence vectors live? | Blob column in SQLite (no dependency) vs. a vector extension (new dependency, contradicts D-001) |
| O-7 | Does Phase 2 ship UI, or API only? | API-only is faster; a thin Swiss list view makes it usable immediately |
