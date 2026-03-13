# Public Interest Pipeline

Purpose:

Determine whether a story is relevant to the public interest using LLM evaluation.

## Processing Steps

1. **Select** stories needing evaluation (new or content-changed since last assessment)
2. **Compose evaluation text** — full body text when `body_valid = true`, headline only otherwise
3. **Send** evaluation text to LLM (OpenAI API) with structured prompt
4. **Apply gate rule** to classify result
5. **Store** assessment (UPSERT into `public_interest_assessments`)

## Gate Rule

4 criteria evaluated: `material_impact`, `institutional_action`, `scope_scale`, `new_information`

| Label | Condition | is_public_interest |
|---|---|---|
| High | 4/4 met | true |
| Moderate | 3/4 met AND (material_impact OR institutional_action) | true |
| Low | 2/4 met | false |
| Not Public Interest | 0-1/4 met | false |

The "anchor" requirement (material_impact OR institutional_action) prevents stories from qualifying on soft criteria alone.

## Evaluation Text Selection

- `body_valid = true` → use full article body (passes both text cleanliness AND semantic consistency with headline)
- `body_valid = false` → headline only

This ensures the LLM evaluates substantive text rather than truncated or garbage body content.

## Model

- **LLM**: gpt-4o-mini via OpenAI API (temperature=0.0)
- Configurable via `LLM_MODEL` env var; Ollama supported via `LLM_BASE_URL`

## Benchmark Framework

Human review workflow for measuring LLM accuracy:

1. **Sample** (`scripts/sample_stories.py`) — Stratified random sampling from PI-assessed stories. Three focus modes: `boundary` (oversample Moderate/Low), `errors` (heavy Moderate/Low), `random` (proportional). Excludes already-labeled stories.
2. **Review** — CSV export with LLM outputs + empty `human_*` columns for manual labeling
3. **Import** (`scripts/import_labels.py`) — Import human labels into `pi_benchmark_labels` table. Re-derives label using same `classify_pi()` gate rule. UPSERT on `(story_id, reviewer_id)`.
4. **Measure** — Compare human vs LLM labels to calculate accuracy per label tier

Unit tests: `tests/test_pi_classification.py` — 8 tests covering all gate rule label/anchor combinations.
