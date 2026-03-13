# Decisions Log

## 2026-03-09 — Source table selection

**What**: Use `newsapi_articles` as sole ingestion feed; deprecate `public.headlines`.
**Why**: newsapi_articles has full body text (avg 3,661 chars vs 213), no ingestion gaps, 95% URL overlap. Headlines feed is 99% truncated content.
**Impact**: Source mapping abstraction can be simplified. All enrichment stages use newsapi_articles exclusively.

## 2026-03-09 — Embedding model selection

**What**: BAAI/bge-small-en-v1.5 (384 dimensions), run locally via sentence-transformers.
**Why**: Good quality for clustering, fast inference on local GPU, no API cost. bge-base (768d) is a potential upgrade if quality needs improvement.
**Impact**: All embeddings stored as VECTOR(384) in pgvector. Changing model requires re-embedding entire corpus.

## 2026-03-09 — Embedding input composition

**What**: Headline repeated 3x + body[:250] when body_clean=True. Falls back to headline-only.
**Why**: Headline 3x gives ~40-45% of embedding token weight, ensuring headline semantics dominate while body text differentiates similar headlines (e.g. different sports games). body[:250] chosen over full body to limit noise.
**Impact**: Stored in validation_outputs.embedding. Used by clustering stage.

## 2026-03-09 — Clustering algorithm: medoid-based, not centroid

**What**: Use medoid (most representative member) instead of averaged centroid for cluster comparison.
**Why**: Centroid averaging causes +0.15 inflation in similarity scores — proved via test_centroid_drift experiments. Medoid stays anchored to a real story.
**Impact**: topics.centroid column stores the medoid's embedding (reusing pgvector index). Medoid recomputed every 10 assignments.

## 2026-03-09 — Named entities replace NewsAPI concepts for clustering

**What**: Use spaCy NER (PERSON/ORG/GPE) instead of NewsAPI concept URIs for entity overlap scoring.
**Why**: NewsAPI concepts were noisy — included wiki topics, party names, media outlets that caused false merges. spaCy NER is cleaner and more diagnostic.
**Impact**: Entity veto and entity overlap scoring both use named_entities from validation_outputs.

## 2026-03-10 — Two-phase body quality check

**What**: body_clean (text cleanliness) + body_valid (body_clean AND cosine similarity >= 0.3).
**Why**: Separates text quality from semantic consistency. Some articles have clean body text that doesn't match the headline (e.g. wrong article body, template pages). The 0.3 threshold catches these.
**Impact**: body_valid gates PI evaluation text selection. body_clean gates cluster text composition.

## 2026-03-10 — Top category: 9-value immutable enum

**What**: Map highest-weight `news/*` category URI to one of: sports, politics, business, entertainment, technology, health, environment, science, general.
**Why**: Provides clean, stable category labels for frontend filtering and heatmap aggregation. Enum is immutable — no new values added without migration.
**Impact**: Stored in validation_outputs.top_category and enriched_headlines.top_category.

## 2026-03-10 — PI gate rule with anchor requirement

**What**: 4 criteria (material_impact, institutional_action, scope_scale, new_information). High=4/4, Moderate=3/4+anchor, Low=2/4, Not PI=0-1. Anchor = material_impact OR institutional_action.
**Why**: Anchor requirement prevents stories from qualifying as public interest on soft criteria alone (scope_scale + new_information without any real-world impact).
**Impact**: classify_pi() in pi_evaluation.py. 8 unit tests cover all combinations.

## 2026-03-10 — LLM model: gpt-4o-mini

**What**: Use gpt-4o-mini via OpenAI API for PI evaluation (temperature=0.0).
**Why**: Cheapest model that handles structured JSON output reliably. PI prompt is simple (4 booleans + reasoning). More expensive models have lower rate limits.
**Impact**: Configurable via LLM_MODEL env var. Ollama supported via LLM_BASE_URL override.

## 2026-03-10 — Bias scoring: composite from three sources

**What**: AllSides, Ad Fontes Media, MBFC scores normalized to [-1,1], averaged, mapped back to AllSides scale.
**Why**: No single bias rating is authoritative. Composite smooths individual service biases. 14 outlets seeded.
**Impact**: outlet_bias_scores table, v_heatmap and v_heatmap_asymmetry views.

## 2026-03-11 — Deduplication: title+source and URL-normalized

**What**: Two dedup rules at story selection, before validation. Rule 1: same title + same source URI, keep earliest. Rule 2: same URL after stripping query params, keep earliest.
**Why**: Prevents same-source duplicates from biasing cluster centroids. Cross-source syndication intentionally NOT deduped (multiple outlets covering same story is a legitimate signal).
**Impact**: ~3.3% of stories filtered. NewsAPI is_duplicate field found unreliable, not used.

## 2026-03-12 — Clustering config: 168h window, entity veto ON

**What**: Temporal window widened to 168h (7 days), entity veto enabled, category gate OFF.
**Why**: 168h accommodates multi-week backfill across 27 days of data. Entity veto prevents false merges by requiring at least one shared named entity. Category gate found to inflate clusters.
**Impact**: Clustering config in config.py. composite_threshold=0.45, weights: embedding=0.65, entities=0.35.

## 2026-03-12 — PI evaluation framework: stratified sampling

**What**: Three-mode sampling (boundary/random/errors) for PI benchmark labeling, import via scripts, pi_benchmark_labels table.
**Why**: Need to measure LLM accuracy against human judgment. Boundary mode oversamples Moderate/Low where classification is hardest.
**Impact**: scripts/sample_stories.py, scripts/import_labels.py, tests/test_pi_classification.py.

## 2026-03-12 — Cluster labels: PI-weighted medoid headline

**What**: If cluster has PI stories, label = headline of medoid(PI subset). Otherwise label = headline of medoid(all stories).
**Why**: PI stories are more representative of what matters in the cluster. Non-PI stories may be noise.
**Impact**: cluster_labeling.py update_cluster_labels().

## 2026-03-13 — Ingestion pipeline normalization

**What**: Hybrid model — raw JSONB preserved on newsapi_articles, normalized relational tables (newsapi_sources, newsapi_concepts, newsapi_categories + junction tables) populated by loaders.
**Why**: Enables fast indexed joins for analytics instead of JSONB extraction. Canonical dimensions reduce repetition. Run-level observability via pipeline_run_metrics.
**Impact**: Prefer normalized joins over JSONB for queries. Enrichment pipeline still uses JSONB in Python — potential future refactor.

## 2026-03-13 — Context maintenance protocol

**What**: 5-file protocol (DECISIONS, SPECS, REQUIREMENTS, OPEN_QUESTIONS, CONSTRAINTS) with "sync docs" command.
**Why**: Maintain living documentation that stays current with implementation. Prevents context loss across sessions.
**Impact**: CONTEXT_PROTOCOL.md in context repo root. CLAUDE.md snippet in headlines-enrichment.
