# Public Interest Evaluation Specification

Status: Living Specification
Owner: MediaDotGames Platform

This document defines the criteria used to evaluate whether a news story is relevant to the public interest.

The specification governs the behavior of the Public Interest LLM Judging Service.

## 1. System Purpose

The system evaluates news **stories** to determine whether they are relevant to the public interest.

The evaluation must be:

- deterministic
- reproducible
- auditable

The LLM must function as a structured evaluator rather than a subjective commentator.

## 2. Evaluation Unit

The system evaluates **individual stories**, not topics or clusters.

Input data for evaluation includes:

- `story_id`
- `headline`
- `evaluation_text` (composed from headline + snippet or body text)
- `source_name`

The evaluation text is composed using a priority order: full body text (if > 100 chars), then headline + snippet (if snippet is valid), then headline only.

## 3. Definition of Public Interest

A story is considered relevant to the public interest if it helps the public understand:

- how power is exercised
- how institutions affect people's lives
- what new facts about those systems have emerged

Stories that do not advance those goals receive Low or Not Public Interest classifications.

**Focus on systems, not individuals.** The system prioritizes institutional behavior over individual incidents. This avoids distortion from cherry-picked crime stories, culture-war outrage content, and celebrity controversies.

**Commentary does not automatically qualify.** Opinion or analysis pieces qualify only if they analyze real institutional actions with real consequences.

**Historical or archival stories usually fail.** Stories about historical figures or anniversaries generally fail unless they reveal new evidence or documents.

Topics primarily involving entertainment, celebrity activity, lifestyle, or consumer products generally do not qualify.

## 4. Evaluation Criteria

Each story is evaluated across four binary criteria (true/false).

### Criterion 1: Real-World Impact (`material_impact`)

Does the story describe events that materially affect people's safety, health, economic conditions, civil rights, or security?

Examples:

- war
- economic shocks
- public health crises
- environmental disasters
- major policy changes

### Criterion 2: Institutional Action (`institutional_action`)

Does the story focus on actions or decisions by institutions?

Qualifying institutions:

- governments
- courts
- regulators
- legislatures
- large corporations
- security forces

This criterion intentionally filters out individual behavior stories.

Examples that fail: random crime, celebrity misconduct, interpersonal disputes.

Examples that pass: court rulings, legislation, regulatory enforcement, institutional failure.

### Criterion 3: Widespread Impact (`scope_scale`)

Does the issue affect large populations or systems, rather than a small number of individuals?

Examples: national policy, major geopolitical events, systemic regulatory changes, economic disruptions.

Examples that fail: isolated local incidents, individual disputes, small community events.

### Criterion 4: New Information (`new_information`)

Does the story report a new development, decision, event, document, or verified fact that had not previously been publicly reported?

Examples that qualify: new policy decisions, court rulings, indictments, investigative revelations, newly released reports or data.

Examples that do not qualify: commentary, historical retrospectives, reaction pieces, follow-up summaries repeating known facts.

## 5. Gate Rule

A story qualifies as public interest if:

`met_count >= 3` AND at least one **anchor criterion** is true.

Anchor criteria are `material_impact` or `institutional_action`.

## 6. Classification Labels

| Label | Definition | `is_public_interest` |
|-------|-----------|---------------------|
| High | 4 criteria met | `true` |
| Moderate | 3 criteria met (with anchor) | `true` |
| Low | 2 criteria met | `false` |
| Not Public Interest | 0-1 criteria met | `false` |

Confidence values: High = 0.7, Moderate = 0.6, Low = 0.4, Not Public Interest = 0.4.

## 7. Output Schema

The judging service produces the following result per story.

```json
{
  "story_id": "string",
  "is_public_interest": true,
  "label": "High",
  "met_count": 4,
  "confidence": 0.7,
  "material_impact": true,
  "institutional_action": true,
  "scope_scale": true,
  "new_information": true,
  "assessment_json": {
    "prompt_version": "pi_v1.0",
    "model": "openai/gpt-4o-mini",
    "raw_response": "...",
    "reasoning": "One sentence explanation."
  }
}
```

## 8. Storage

Assessment results are stored in:

`public.public_interest_assessments`

Fields:

- `id` (UUID, primary key)
- `story_id` (TEXT, unique)
- `evaluated_at` (TIMESTAMPTZ)
- `is_public_interest` (BOOLEAN)
- `label` (TEXT: 'High' | 'Moderate' | 'Low' | 'Not Public Interest')
- `met_count` (INT, 0-4)
- `confidence` (REAL)
- `material_impact` (BOOLEAN)
- `institutional_action` (BOOLEAN)
- `scope_scale` (BOOLEAN)
- `new_information` (BOOLEAN)
- `assessment_json` (JSONB: full LLM response + prompt version + model)
- `pipeline_run_id` (UUID)
- `created_at` (TIMESTAMPTZ)

Re-evaluation overwrites via UPSERT on `story_id`.

## 9. Determinism Requirements

To ensure reproducibility:

- the prompt template must be versioned
- the model version must be stored
- evaluation inputs must be stored
- temperature should be set to 0

## 10. Intended Uses

The classifier enables:

- **Media signal-to-noise measurement** — comparing outlets by share of public-interest coverage
- **Omission detection** — identifying important issues receiving little coverage
- **Saturation detection** — detecting amplification of low-value stories
