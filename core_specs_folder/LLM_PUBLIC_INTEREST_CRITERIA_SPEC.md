# Public Interest Evaluation Specification

Status: Living Specification  
Owner: MediaDotGames Platform

This document defines the deterministic criteria used to evaluate whether a news topic is relevant to the public interest.

The specification governs the behavior of the Public Interest LLM Judging Service.

## 1. System Purpose

The system evaluates news **topics** to determine whether they are relevant to the public interest.

A topic represents a cluster of stories referring to the same underlying event.

The evaluation must be:

- deterministic
- reproducible
- auditable

The LLM must function as a structured evaluator rather than a subjective commentator.

## 2. Evaluation Unit

The system evaluates **topics**, not individual stories.

Input data for evaluation includes:

- `topic_id`
- `representative_headline`
- `representative_story_summary`
- `topic_keywords`
- `cluster_size`
- `sources_reporting`

The representative story is typically the most central story in the cluster.

## 3. Definition of Public Interest

A topic is considered relevant to the public interest if it materially affects:

- governance
- public policy
- public safety
- economic systems
- civil rights
- environmental systems
- large populations

Topics primarily involving entertainment, celebrity activity, lifestyle, or consumer products generally do not qualify.

## 4. Evaluation Criteria

Each topic is evaluated across multiple criteria.

Each criterion is scored:

- `0 = not present`
- `1 = weak presence`
- `2 = moderate presence`
- `3 = strong presence`

### Criterion 1: Governance Impact

Does the topic involve government institutions or political processes?

Examples:

- legislation
- regulatory policy
- elections
- executive actions
- international diplomacy

### Criterion 2: Public Safety

Does the topic involve risks to human safety?

Examples:

- natural disasters
- war
- terrorism
- large-scale accidents
- public health emergencies

### Criterion 3: Economic Impact

Does the topic affect economic systems or major industries?

Examples:

- financial crises
- major corporate collapse
- labor strikes
- major layoffs
- systemic market disruptions

### Criterion 4: Civil Rights / Legal Precedent

Does the topic affect civil liberties or legal frameworks?

Examples:

- court rulings
- constitutional interpretation
- discrimination cases
- major legal reforms

### Criterion 5: Environmental Impact

Does the topic affect environmental systems?

Examples:

- climate policy
- environmental disasters
- ecological regulation

### Criterion 6: Population Scale

How many people are affected?

Scoring guide:

- `0 = fewer than 1,000 people`
- `1 = local community`
- `2 = regional population`
- `3 = national or global population`

## 5. Scoring Model

The total public interest score is calculated as:

`public_interest_score = sum(criteria_scores)`

Maximum possible score:

`18`

## 6. Classification Rule

- `score >= 5` → Public Interest
- `score < 5` → Not Public Interest

This threshold may be adjusted after empirical testing.

## 7. Output Schema

The judging service must produce the following JSON result.

```json
{
  "topic_id": "uuid",
  "public_interest": true,
  "score_total": 7,
  "criteria_scores": {
    "governance": 2,
    "public_safety": 1,
    "economic_impact": 2,
    "civil_rights": 0,
    "environment": 0,
    "population_scale": 2
  },
  "reasoning_summary": "Concise explanation of the evaluation",
  "model": "gpt-x",
  "evaluation_timestamp": "timestamp"
}
```

## 8. Storage

Judgment results should be stored in a database table.

Example schema:

`public_interest_judgments`

Fields:

- `id`
- `topic_id`
- `public_interest`
- `score_total`
- `criteria_scores_json`
- `reasoning_summary`
- `model`
- `evaluation_timestamp`

## 9. Determinism Requirements

To ensure reproducibility:

- the prompt template must be versioned
- the model version must be stored
- evaluation inputs must be stored
- temperature should be set to 0
