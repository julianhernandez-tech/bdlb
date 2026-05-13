# BDLB Build Brief

This brief is the source-of-truth for all sub-agents working on the Backward-Design Lesson Builder (BDLB) build. Every sub-agent reads from this file by path.

---

## ⚠️ FOLDER ISOLATION — HARD RULE (read first)

The BDLB system lives in **`bdlb/` ONLY**. Sub-agents may write **only** to the path they are given in their OUTPUTS section. They must **never**:

- Write, edit, rename, move, or delete any file outside `bdlb/`.
- Touch any STAAR source spec (READ-ONLY references).
- Reference STAAR runtime directories (e.g. `outputs/grade{N}/...`, `schemas/...` at repo root, `testcreation/...`, `Images/...`) in any new spec.
- Use a filename that already exists under any STAAR location.

The STAAR system at `test_creation/.claude/agents/` is **READ-ONLY for this build**. Sub-agents may read STAAR specs for inspiration only.

The only file the build orchestrator writes at repo root is **this file** (`bdlb_build_brief.md`).

---

## 1. MVP scope of BDLB

Input: a single STAAR-style math question image.

Output: a lesson made of:

- A `lesson_plan.json` produced by backward design (3 tiers: easy, medium, hard) based on the grade level (assume grade-1 mastery of topics).
- A tier spec per level with a variation matrix of **exactly 20** non-redundant variations.
- 60 authored items (3 tiers × 20 variations) passing Layer 1 + Layer 2 QC.
- Required images rendered and QC'd.
- A teaching article with a thinking process, drafted by 3 parallel models, then QC'd (deterministic + smart + render).
- A final assembled `lesson.html` and `lesson_manifest.json`.

The orchestrator must follow the same discipline as `staar-orchestrator.v2.md`:
phased state machine, mechanical routing, retry cap = 3 per side, defense-in-depth QC,
halt on systemic failure patterns, never bypass a QC layer, no subjective judgment by the orchestrator.

---

## 2. Output directory for this build

Everything goes under `bdlb/` at repo root. The exact layout is:

```
{repo_root}/
  bdlb_build_brief.md           ← the only file at repo root
  bdlb/
    agents/                      ← all new .md specs
    schemas/                     ← all new JSON schemas
    reference/                   ← human-readable notes only
    runs/                        ← runtime output (not populated at build time)
    build_state.json
    build_events.jsonl
    consistency_report.json      ← written in Phase B3
    README.md                    ← written in Phase B4
```

`build_state.json` is updated only after each phase's artifacts are written and validated.

---

## 3. Required new agent specs (clone-and-adapt list)

Create the following `.md` files under `bdlb/agents/` ONLY. Each must follow the source spec's structure: front-matter (`name`, `description`, `tools`), Role, Input contract, Output contract, Self-checks, Forbidden behaviors.

STAAR source specs live under: `test_creation/.claude/agents/` (READ-ONLY).

| New BDLB spec (write to `bdlb/agents/{name}`) | Source STAAR spec (READ-ONLY) | Adaptation summary |
|---|---|---|
| `bdlb-orchestrator.md` | `staar-orchestrator.v2.md` | Replace phases with BDLB phases 0–7. Replace agent table with BDLB roster. Keep dispatch protocol, retry semantics, halt rules, defense-in-depth, artifact-first writes verbatim. All paths rewritten to `bdlb/`. |
| `seed-question-extractor.md` | base on input-contract pattern in `source-extractor.md` | Reads a PNG of a STAAR question and emits `seed_question.parsed.json` with `stem_text`, `options`, `image_descriptions`, `inferred_tek`, `inferred_grade`, `solution_path`, `cognitive_demand_estimate`. Uses vision input. Returns `error: true` on illegible input. |
| `backward-design-analyst.md` | `grade5-blueprint-analyst.v2.md` | Replace test-blueprint planning with lesson-tier planning. Output `lesson_plan.json`: `lesson_goal`, `tek_alignment`, `rigor_anchor: hard`, `tiers.{easy,medium,hard}` (`description`, `item_type`, `cognitive_demand`), `scaffolding_logic`, `thinking_process`. Hard tier rigor matches seed; medium one rigor level below; easy one below medium. |
| `tier-specifier.md` | `grade5-blueprint-analyst.v2.md` + `question-generator.v2.md` (constraint half) | Dispatched 3× parallel (one per tier). Output `tier_spec.{tier}.json` with: `tier`, `item_type`, `must_include`, `must_exclude`, `image_required`, `image_spec_template`, `variation_matrix` of **exactly 20** entries, `uniqueness_rule`. Computes variations programmatically via Bash+Python; verifies constraints (e.g., easy "no carry" → every column sum < 10) and no duplicate `(numbers, context_seed)` pairs. Schema enforces `minItems=20, maxItems=20`. |
| `tier-explanation-enricher.md` | `smart-qc.v2.md` (documentation-grounded reasoning) | Dispatched 3× parallel. Reads `tier_spec.{tier}.json` + `lesson_plan.thinking_process` + documentation corpus. Outputs `tier_explanation.{tier}.json` with thinking steps, common misconceptions, worked example, vocabulary notes. |
| `image-spec-generator.md` | `image-generator.md` (input-contract half) | Per variation when `image_required: true`. Emits a fully populated imagespec, no nulls, every label/value present. |
| `image-renderer.md` | `image-generator.md` | Reuse Pillow-only deterministic rendering. PNG + `.report.json` with `render_status`. Forbid generative image APIs. |
| `image-qc.md` | `deterministic-qc.v2.md` (image checks) | Validates image presence, dimensions, label margins ≥ 20px, no answer-leak in image. |
| `question-author.md` | `question-generator.v2.md` | Inputs: one `variation_matrix` entry, `tier_spec`, `tier_explanation`, prior accepted item summaries (frozen at wave start), optional image PNG path. Output `bdlb/runs/{run_id}/items/{tier}/{var_id}.json` validating against `bdlb/schemas/item_schema.json`. Inherit all phrasing rules. |
| `lesson-deterministic-qc.md` | `deterministic-qc.v2.md` | Reuse Sections A–N. Add `VARIATION_UNIQUENESS` (no two variations within a tier share both numeric pair and context_seed; 4-gram Jaccard < 0.30 stem-to-stem within tier). |
| `lesson-smart-qc.md` | `smart-qc.v2.md` | Reuse all gates. Add `TIER_RIGOR_MATCH`, `SCAFFOLD_INTEGRITY`, `HARD_TIER_MATCHES_SEED`. |
| `article-author.md` | `question-generator.v2.md` (authoring discipline) + `grade5-blueprint-analyst.v2.md` (long-form structure) | Dispatched 3× parallel using 3 different models. Inputs: `lesson_plan.json`, all 60 canonical items, all 3 `tier_explanation` files. Output `articles/draft_{model_id}.json` with: `title`, `intro`, `thinking_process`, `tier_walkthroughs` (each linking a real `var_id`), `summary`, `html_render`. |
| `article-deterministic-qc.md` | `deterministic-qc.v2.md` | Schema validation, link integrity to `var_id`s, vocabulary whitelist, length bands, heading hierarchy, no forbidden phrases, no raw `</>`/`$`. |
| `article-smart-qc.md` | `smart-qc.v2.md` | Pedagogical gates: `THINKING_PROCESS_COHERENT`, `SCAFFOLD_VISIBLE`, `READING_LEVEL_APPROPRIATE`, `WORKED_EXAMPLES_REFERENCE_REAL_ITEMS`, `NO_ANSWER_LEAK_TO_BANK`. |
| `article-render-qc.md` | `alphatest-render-qc.md` + `render-harness-builder.md` | Renders article HTML in headless Chromium. Math glyphs, no overflow, no broken images, all referenced `var_id`s resolve, accessibility checks. |
| `lesson-assembler.md` | `qti-assembler.md` | Bundles canonical items + selected article + manifest into `lesson.html` and `lesson_manifest.json`. **No QTI**; HTML output. |

---

## 4. Required new schemas (under `bdlb/schemas/` ONLY)

- `lesson_plan_schema.json`
- `tier_spec_schema.json` (with `variation_matrix.minItems: 20, maxItems: 20, uniqueItems: true`)
- `tier_explanation_schema.json`
- `item_schema.json` (extends STAAR's; adds `tier`, `var_id`, `variation_basis`)
- `image_spec_schema.json`
- `qc_report_schema.json` (reuse STAAR's 4 variants; add new gates from §3)
- `article_draft_schema.json`
- `state_schema.json`
- `agent_invocation_schema.json` (`spec_version`, `spec_sha256`, `model`, `temperature`, `input_hash`, `output_hash`)

All schemas MUST:

- Use JSON Schema draft 2020-12 (`"$schema": "https://json-schema.org/draft/2020-12/schema"`).
- Set `"$id"` to `https://bdlb.local/schemas/{filename}`.
- Be self-contained: no external `$ref` to STAAR schemas at repo root.
- Reject unknown fields with `"additionalProperties": false` at every object level.
- Use `"type"`, `"required"`, and explicit constraints on every field.

---

## 5. STAAR source spec locations (READ-ONLY)

All STAAR source specs live at: `test_creation/.claude/agents/`

Confirmed present:

- `staar-orchestrator.v2.md`
- `question-generator.v2.md`
- `deterministic-qc.v2.md`
- `smart-qc.v2.md`
- `test-level-qc.md`
- `image-generator.md`
- `render-harness-builder.md`
- `alphatest-render-qc.md`
- `qti-assembler.md`
- `source-extractor.md`
- `grade3-blueprint-analyst.v2.md`
- `grade4-blueprint-analyst.v2.md`
- `grade5-blueprint-analyst.v2.md`
- `alphatest-creation-walkthrough.md`
- `README.md`

Sub-agents MUST NOT edit any of these. They may only read them.

---

## 6. Forbidden behaviors (all sub-agents)

- Writing any file outside the OUTPUT path declared in their dispatch.
- Editing, renaming, moving, or deleting any file outside `bdlb/`.
- Referencing STAAR runtime directories (`outputs/grade{N}/...`, repo-root `schemas/...`, `testcreation/...`, `Images/...`) in any new spec.
- Reusing a STAAR filename for a new BDLB spec.
- Inventing agents not in the §3 roster.
- Adding `$ref` from BDLB schemas to STAAR schemas.
- Writing prose claims about behavior they cannot enforce.

---

End of brief.
