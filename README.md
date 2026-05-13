# BDLB — Backward-Design Lesson Builder

> **BDLB and STAAR are independent systems.** They share the same repository for
> convenience, but they never share files, never reference each other's runtime
> paths, and never modify each other's artifacts. BDLB lives under `bdlb/`
> exclusively. STAAR lives under `test_creation/` exclusively. The only file
> outside `bdlb/` that belongs to the BDLB build is `bdlb_build_brief.md` at
> repo root, which sub-agents read during the build.

---

## What BDLB does

Given **one** STAAR-style math question image as the seed, BDLB produces a
backward-designed lesson:

- A `lesson_plan.json` with three tiers (easy, medium, hard), where the **hard
  tier matches the seed question's rigor**, medium is one cognitive-demand
  level below, and easy is one below medium. The plan assumes the student has
  mastery at (grade − 1).
- For each tier: a `tier_spec.{tier}.json` with a **variation matrix of
  exactly 20 entries**, deterministically enumerated and deduped on
  `(numbers, context_seed)`.
- For each tier: a `tier_explanation.{tier}.json` grounded in a documentation
  corpus — thinking steps, common misconceptions, worked example, vocabulary
  notes.
- **60 authored items** (3 tiers × 20 variations), each passing Layer 1
  (deterministic) + Layer 2 (smart) QC.
- Required images rendered deterministically with Pillow, each passing
  image-QC.
- **One teaching article** with a visible thinking process, drafted by **three
  parallel models**, each draft passing deterministic + smart + render QC.
- A final assembled `lesson.html` and `lesson_manifest.json` packaged
  deterministically.

---

## Runtime phase map

```
USER provides PNG → bdlb-orchestrator
   Phase 0   Preflight                         (verify run_id, schemas, state)
   Phase 1   Seed extraction                   → seed-question-extractor
   Phase 2   Backward design                   → backward-design-analyst         → lesson_plan.json
   Phase 3   Tier specification                → tier-specifier ×3 parallel       → tier_spec.{easy,medium,hard}.json
   Phase 3b  Tier explanation                  → tier-explanation-enricher ×3     → tier_explanation.{tier}.json
   Phase 4   Item authoring + image + QC       → per variation:
                                                  question-author
                                                  → [if image_required]
                                                       image-spec-generator
                                                       → image-renderer
                                                       → image-qc
                                                  → lesson-deterministic-qc
                                                  → lesson-smart-qc
   Phase 5   Article authoring                 → article-author ×3 (3 models)
   Phase 6   Article QC                        → article-deterministic-qc
                                                → article-smart-qc
                                                → article-render-qc
   Phase 7   Lesson assembly                   → lesson-assembler                 → lesson.html + lesson_manifest.json
```

The orchestrator's runtime discipline (cloned verbatim from STAAR's
`staar-orchestrator.v2.md`):

- One agent per dispatch. Inputs are file paths; outputs are files; never
  trust stdout. Parallel dispatches batched into one tool block, then wait.
- **Retry cap = 3 per side.** Beyond that, mark `spec_failed: true` and halt
  per source rules.
- **Defense in depth.** Every item passes deterministic → smart QC; every
  image passes image-QC; every article passes deterministic → smart → render
  QC. Never bypass a QC layer.
- **Artifact-first writes.** `state.json.phase` advances only after artifacts
  are validated on disk.
- The orchestrator never authors content; it only routes.

---

## Agent roster (16 agents)

| Agent (in `bdlb/agents/`)         | Role                                                                  | Source STAAR spec (READ-ONLY reference)                                |
| --------------------------------- | --------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `bdlb-orchestrator.md`            | Plans and routes only. State machine, retry, halt, defense-in-depth.  | `test_creation/.claude/agents/staar-orchestrator.v2.md`                |
| `seed-question-extractor.md`      | Vision-only: PNG → structured seed question JSON.                     | `test_creation/.claude/agents/source-extractor.md` (input-contract)    |
| `backward-design-analyst.md`      | Builds the 3-tier lesson plan with rigor anchored on the seed.        | `test_creation/.claude/agents/grade5-blueprint-analyst.v2.md`          |
| `tier-specifier.md`               | Emits a 20-entry variation matrix per tier (3× parallel).             | grade5-blueprint-analyst + question-generator.v2 (constraint half)     |
| `tier-explanation-enricher.md`    | Documentation-grounded pedagogical notes per tier (3× parallel).      | `test_creation/.claude/agents/smart-qc.v2.md` (documentation grounding)|
| `image-spec-generator.md`         | Per variation → fully populated imagespec.                            | `image-generator.md` (input-contract half)                             |
| `image-renderer.md`               | imagespec → PNG via Pillow. No generative APIs.                       | `image-generator.md` (render half)                                     |
| `image-qc.md`                     | Validates rendered images (margins, dims, no answer leak).            | `deterministic-qc.v2.md` (image checks)                                |
| `question-author.md`              | Writes one item per variation. Inherits all phrasing rules.           | `question-generator.v2.md`                                             |
| `lesson-deterministic-qc.md`      | Mechanical Layer-1 QC + `VARIATION_UNIQUENESS` gate.                  | `deterministic-qc.v2.md`                                               |
| `lesson-smart-qc.md`              | Semantic Layer-2 QC + tier-rigor / scaffold / hard-tier-vs-seed gates.| `smart-qc.v2.md`                                                       |
| `article-author.md`               | Writes the teaching article (3× parallel, 3 models).                  | `question-generator.v2.md` + `grade5-blueprint-analyst.v2.md`          |
| `article-deterministic-qc.md`     | Mechanical QC of one article draft.                                   | `deterministic-qc.v2.md`                                               |
| `article-smart-qc.md`             | Pedagogical QC of one article draft.                                  | `smart-qc.v2.md`                                                       |
| `article-render-qc.md`            | Headless-Chromium render + visual gates of the article.               | `alphatest-render-qc.md` + `render-harness-builder.md`                 |
| `lesson-assembler.md`             | Deterministic bundle → `lesson.html` + `lesson_manifest.json`. No QTI.| `qti-assembler.md`                                                     |

---

## Schemas (9 schemas in `bdlb/schemas/`)

| Schema                                | Purpose                                                                  |
| ------------------------------------- | ------------------------------------------------------------------------ |
| `lesson_plan_schema.json`             | Validates `lesson_plan.json` (rigor anchor, tiers, thinking process).    |
| `tier_spec_schema.json`               | Validates `tier_spec.{tier}.json`. `variation_matrix` is exactly 20.     |
| `tier_explanation_schema.json`        | Validates `tier_explanation.{tier}.json` (documentation-grounded notes). |
| `item_schema.json`                    | Validates one authored BDLB item. Adds `tier`, `var_id`, `variation_basis`.|
| `image_spec_schema.json`              | Validates an imagespec. No nulls except `caption`. `must_not_leak_answer: true` required.|
| `qc_report_schema.json`               | Single schema with 6 variants (lesson/article × det/smart, image_qc, article_render).|
| `article_draft_schema.json`           | Validates one of the 3 article drafts. Requires `tier_walkthroughs` cover all 3 tiers.|
| `state_schema.json`                   | Runtime state file (`bdlb/runs/{run_id}/state.json`) the orchestrator updates.|
| `agent_invocation_schema.json`        | Audit record per dispatch (`spec_sha256`, model, temperature, hashes).   |

All schemas use JSON Schema draft 2020-12, are self-contained (no `$ref` to
STAAR or repo-root schemas), and reject unknown fields.

---

## Directory layout

```
{repo_root}/
  bdlb_build_brief.md          ← the only repo-root file owned by this build
  bdlb/
    README.md                   ← this file (build manifest)
    build_state.json            ← orchestrator-write phase tracker (build-time)
    build_events.jsonl          ← append-only build event log
    consistency_report.json     ← Phase B3 invariant check results
    existing_specs_summary.json ← frozen-at-wave-start snapshot (build-time)
    agents/                     ← 16 BDLB agent specs (this build)
    schemas/                    ← 9 BDLB JSON Schemas (this build)
    reference/                  ← human-readable notes only (no STAAR copies)
    runs/                       ← runtime output per lesson (one subdir per run_id)
        {run_id}/
          state.json
          events.jsonl
          seed/
            seed.png
            seed_question.parsed.json
          lesson_plan.json
          tier_spec.{easy,medium,hard}.json
          tier_explanation.{easy,medium,hard}.json
          items/{easy,medium,hard}/v{01..20}.json
          images/{tier}__v{01..20}.imagespec.json
          images/{tier}__v{01..20}.png
          images/{tier}__v{01..20}.report.json
          articles/draft_{model_id}.json
          qc_reports/lesson_det__{tier}__*.json
          qc_reports/lesson_smart__{tier}__*.json
          qc_reports/image_qc__{tier}__v{01..20}.json
          qc_reports/article_det__{model_id}.json
          qc_reports/article_smart__{model_id}.json
          qc_reports/article_render__{model_id}.json
          scratch/              ← intermediate ad-hoc artifacts (Python helpers, harness HTML)
          lesson.html
          lesson_manifest.json
```

---

## Entry point

To run BDLB on a seed question, dispatch the orchestrator at
[`bdlb/agents/bdlb-orchestrator.md`](agents/bdlb-orchestrator.md) with a fresh
`run_id` and a seed PNG. The orchestrator drives all 7 phases mechanically.

The orchestrator never writes content. It only writes state and dispatches
sub-agents. All content is produced by the specialists.

---

## Independence from STAAR

| Concern                       | STAAR                                            | BDLB                                       |
| ----------------------------- | ------------------------------------------------ | ------------------------------------------ |
| Agent specs root              | `test_creation/.claude/agents/`                  | `bdlb/agents/`                             |
| Schemas root                  | `test_creation/schemas/`                         | `bdlb/schemas/`                            |
| Runtime artifacts root        | `test_creation/outputs/grade{N}/...`             | `bdlb/runs/{run_id}/...`                   |
| Image staging                 | `test_creation/Images/` (manual upload)          | `bdlb/runs/{run_id}/images/` (Pillow only) |
| Final output format           | QTI 3.0 XML + image manifest for AlphaTest API   | `lesson.html` + `lesson_manifest.json`     |
| External publishing target    | AlphaTest API                                    | None — local lesson bundle                 |

A BDLB sub-agent that writes anywhere under `test_creation/` is a hard build
failure. A STAAR sub-agent that writes anywhere under `bdlb/` is a hard
STAAR failure. The two systems are **independent** by design.

---

## Build provenance

This build was produced from the brief at `../bdlb_build_brief.md` using a
five-phase build process (B0–B4) following the dispatch protocol in
`staar-orchestrator.v2.md`:

- **B0** Preflight — created `bdlb/` tree and `bdlb_build_brief.md`.
- **B1** Schemas — 9 schemas authored in a single parallel batch.
- **B2** Specs — 16 agent specs cloned-and-adapted in 4 waves (5+5+5+1) parallel.
- **B3** Consistency — single cross-spec invariant check (`consistency_report.json`).
- **B4** Manifest — this README.

Build state and event log are at `build_state.json` and `build_events.jsonl`.
