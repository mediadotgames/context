# Requirements

## User Stories

### Enrichment Pipeline
- As a news analyst, I want articles clustered by topic so that I can see which stories are being covered by multiple outlets
- As a news analyst, I want public interest scoring so that I can filter noise from substantive journalism
- As a news analyst, I want a coverage heatmap so that I can see which outlets cover which topics and spot gaps
- As a news analyst, I want political and geographic asymmetry metrics so that I can identify lopsided coverage

### Ranking & Distribution (planned)
- As a user, I want to sort stories by multiple dimensions (coverage volume, trending, public interest, under-covered)
- As a user, I want visibility into what is trending on major distribution channels (Google, X, YouTube)
- As a user, I want to detect divergence between media production and public attention
- As a user, I want pluggable ranking inputs not locked to one vendor

### PI Evaluation Quality
- As a developer, I want to benchmark LLM PI accuracy against human judgment so that I can tune the prompt and thresholds
- As a developer, I want stratified sampling that oversamples boundary cases so that benchmarks focus on where classification is hardest

## Acceptance Criteria

### Clustering
- Stories about the same event cluster together regardless of headline wording
- Different events in the same category do NOT merge (e.g. different sports games)
- Singleton rate is tracked and reported
- Cluster labels are human-readable and represent the most significant story

### PI Evaluation
- Gate rule correctly classifies: 4/4→High, 3/4+anchor→Moderate, 2/4→Low, 0-1→Not PI
- Anchor requirement enforced (material_impact OR institutional_action)
- All 8 unit test cases pass

### Heatmap
- Rows ordered by cluster_size DESC (biggest stories first)
- Columns ordered by bias score ASC (left→right politically)
- Article counts are accurate per topic × outlet cell

## Test Cases

### PI Gate Rule (tests/test_pi_classification.py)
- All four criteria met → High, is_pi=True, met=4
- Three met with both anchors → Moderate, is_pi=True, met=3
- Three met with material_impact only → Moderate
- Three met with institutional_action only → Moderate
- Two met with anchors → Low, is_pi=False, met=2
- Two met non-anchor → Low, is_pi=False
- One met → Not Public Interest, is_pi=False
- Zero met → Not Public Interest, met=0

### Validation (tests/test_validation.py)
- Top category assignment (12 tests covering all 9 enum values + edge cases)
- Body quality checks
- Cluster text composition (headline 3x + body[:250])
- Evaluation text selection (body_valid → full body, else headline)

## Non-Functional Requirements

- Pipeline must handle 40K+ stories without OOM
- Embeddings generated locally (no API cost)
- PI evaluation must handle rate limiting gracefully (retry with backoff)
- All enrichment outputs stored in PostgreSQL (single system of record)
- Grafana dashboards must query in <5 seconds

## Ranking Requirements (detailed)

See `context/requirements/ranking_requirements.md` for full BR/TR breakdown:
- BR-1 through BR-5: business requirements for composite ranking
- TR-1 through TR-10: technical requirements for implementation
