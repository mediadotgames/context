# LLM Judging

LLMs evaluate whether a story is relevant to the public interest.

## Criteria

Four binary criteria (true/false):

1. Real-World Impact (`material_impact`)
2. Institutional Action (`institutional_action`)
3. Widespread Impact (`scope_scale`)
4. New Information (`new_information`)

## Inputs

- `story_id`
- `headline`
- `evaluation_text` (headline + snippet or body text)
- `source_name`

## Outputs

- `is_public_interest` (boolean)
- `label` (High / Moderate / Low / Not Public Interest)
- `met_count` (0-4)
- `confidence` (float)
- 4 individual criterion booleans
- `assessment_json` (full LLM response + prompt version + model)

## Gate Rule

`met_count >= 3` AND (`material_impact` OR `institutional_action`)

Judgments must be stored with reproducible prompts.
