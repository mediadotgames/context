# Specifications

## Enrichment Pipeline

### Inputs
- Source: `public.newsapi_articles` (39K+ articles, 14 outlets, 27+ days)
- Columns consumed: uri, title, body, url, source->>'title', date_time_published, categories (JSONB), concepts (JSONB)

### Processing Stages

**Stage 1 — Story Selection**
- Select unprocessed articles from newsapi_articles
- Deduplication: title+source (keep earliest) and URL-normalized (strip query params)
- Cross-source syndication intentionally preserved
- Impact: ~3.3% filtered

**Stage 2 — Validation**
- Text normalization (headline_clean, story_text_clean)
- Body quality: body_clean (length>=100, no template HTML, not headline dup) → body_valid (body_clean AND cosine sim >= 0.3)
- Top category assignment: highest-weight `news/*` URI → 9-value enum
- NER: spaCy PERSON/ORG/GPE extraction
- Embedding: BAAI/bge-small-en-v1.5 (384d) on cluster_text (headline 3x + body[:250])
- Headline-body similarity: cosine sim of headline vs body[:500] embeddings
- Output: validation_outputs table

**Stage 3 — Clustering**
- Algorithm: medoid-based assign-or-create, three-stage
  1. Candidate selection: pgvector top-5 by medoid embedding similarity
  2. Filtering: 168h temporal window + entity veto (require shared NER entity)
  3. Scoring: composite = embedding_sim * 0.65 + entity_overlap * 0.35
- Threshold: composite >= 0.45 to assign, else create new cluster
- Dynamic stop entities: auto-detected at 5% corpus frequency
- Medoid recomputed every 10 assignments
- Output: topics + headline_topic_assignments tables

**Stage 4 — Cluster Labeling**
- PI-weighted medoid headline selection
- If cluster has PI stories → label = headline of medoid(PI subset)
- Else → label = headline of medoid(all stories)
- Output: topics.label updated

**Stage 5 — PI Evaluation**
- Model: gpt-4o-mini via OpenAI API (temperature=0.0)
- Prompt: PI_SYSTEM_PROMPT + PI_USER_TEMPLATE (prompt version pi_v1.0)
- Input: full body when body_valid=True, headline only otherwise, truncated to 3000 chars
- Response: JSON with 4 boolean criteria + reasoning
- Gate rule: classify_pi() → (is_pi, label, met_count, confidence)
- Concurrency: sequential (max_concurrent=1, request_delay=1.0)
- Retry: 8 attempts, exponential backoff 15-120s on 429
- Output: public_interest_assessments table

**Stage 6 — Scope Classification**
- IN_SCOPE: is_public_interest=True OR cluster_size >= threshold
- OUT_OF_SCOPE: everything else
- Output: scope_classifications table

**Stage 7 — Refresh**
- TRUNCATE + INSERT into enriched_headlines
- Joins: newsapi_articles + validation_outputs + topics + assignments + PI assessments + scope

### Outputs
- `validation_outputs` — per-story enrichment data
- `topics` — cluster metadata
- `headline_topic_assignments` — story-to-cluster mapping
- `public_interest_assessments` — PI evaluation results
- `scope_classifications` — scope status
- `enriched_headlines` — denormalized pre-joined table for Grafana/frontend

## Ingestion Pipeline

### Inputs
- NewsAPI.ai API (full body text, metadata, concepts, categories)

### Processing
1. Collect articles via API queries
2. Load into newsapi_articles (raw JSONB preserved)
3. Normalize sources → newsapi_sources
4. Normalize concepts → newsapi_concepts + newsapi_article_concepts
5. Normalize categories → newsapi_categories + newsapi_article_categories
6. Record metrics → pipeline_run_metrics

### Outputs
- `newsapi_articles` — core fact table with raw JSONB
- `newsapi_sources` — canonical source dimension
- `newsapi_concepts` — canonical concept dimension
- `newsapi_categories` — canonical category dimension
- Junction tables for article↔concept, article↔category relationships
- `pipeline_run_metrics` — per-run observability

## Bias & Coverage Analytics

### Inputs
- enriched_headlines joined with outlet_bias_scores

### Processing
- Composite bias: normalize AllSides/Ad Fontes/MBFC to [-1,1], average, map to AllSides scale
- Political grouping: Left/Center/Right
- Geographic grouping: US/Non-US
- Heatmap: topic × outlet matrix with article counts
- Asymmetry: political_skew and geo_skew per topic

### Outputs
- `outlet_bias_scores` — 14 outlets seeded
- `v_heatmap` — topic × outlet matrix (filtered to IN_SCOPE)
- `v_heatmap_asymmetry` — skew metrics per topic

## PI Benchmark Framework

### Inputs
- Stories with existing PI assessments

### Processing
1. Sample: stratified random sampling (boundary/random/errors modes)
2. Review: CSV export with LLM outputs + empty human columns
3. Import: scripts/import_labels.py → pi_benchmark_labels table
4. Measure: compare human vs LLM labels per tier

### Outputs
- CSV for human review
- `pi_benchmark_labels` table
- Accuracy metrics per label tier
