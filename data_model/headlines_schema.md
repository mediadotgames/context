# Database Schema

## Primary Source Table

Table: `public.newsapi_articles` (active feed)

Core article fields:
- uri TEXT PRIMARY KEY
- title TEXT, body TEXT, url TEXT, image TEXT
- date DATE, time TIME, date_time TIMESTAMPTZ, date_time_published TIMESTAMPTZ
- lang TEXT, sentiment DOUBLE PRECISION, data_type TEXT (news|blog|pr)
- is_duplicate BOOLEAN (not used — unreliable)
- event_uri TEXT, story_uri TEXT, relevance INTEGER, sim DOUBLE PRECISION, wgt BIGINT

Raw upstream JSONB (preserved for audit/replay/debugging):
- source JSONB — `{"uri": "bbc.com", "title": "BBC", ...}`
- categories JSONB — `[{"uri": "news/Politics", "wgt": 100}, ...]`
- concepts JSONB — `[{"uri": "...", "label": {"eng": "..."}, "type": "..."}, ...]`
- authors JSONB, links JSONB, videos JSONB, shares JSONB
- duplicate_list JSONB, extracted_dates JSONB, location JSONB
- original_article JSONB, raw_article JSONB

Relational linkage fields:
- source_uri TEXT — FK to `newsapi_sources.uri`
- ingestion_source TEXT, run_id TIMESTAMPTZ, run_type TEXT, nth_run INTEGER
- collected_at TIMESTAMPTZ, ingested_at TIMESTAMPTZ
- created_at TIMESTAMPTZ, updated_at TIMESTAMPTZ

> **Note**: Prefer normalized joins (newsapi_sources, newsapi_concepts, newsapi_categories) over JSONB extraction for analytics queries. The JSONB columns are retained for auditability and raw payload inspection.

## Normalized Ingestion Dimension Tables

The ingestion pipeline projects high-value dimensions from raw JSONB into normalized relational tables. These are populated automatically by the loaders and backfilled for historical data.

Table: `public.newsapi_sources`

Canonical source dimension. One row per upstream publisher.

- uri TEXT PRIMARY KEY — source URI/domain
- title TEXT — display name
- description TEXT, image TEXT, thumb_image TEXT
- social_media JSONB, ranking JSONB, location JSONB
- created_at TIMESTAMPTZ, updated_at TIMESTAMPTZ

Relationships: one source → many newsapi_articles (via source_uri)

Table: `public.newsapi_concepts`

Canonical concept/entity dimension. Covers people, organizations, locations, wiki topics.

- uri TEXT PRIMARY KEY
- type TEXT — person, loc, org, wiki
- label JSONB — localized labels
- image TEXT, location JSONB, synonyms JSONB
- created_at TIMESTAMPTZ, updated_at TIMESTAMPTZ

Relationships: many concepts ↔ many articles (via newsapi_article_concepts)

Table: `public.newsapi_categories`

Canonical category dimension. Hierarchical via parent_uri.

- uri TEXT PRIMARY KEY
- parent_uri TEXT — upstream parent (not FK-enforced — upstream taxonomy is incomplete)
- label TEXT
- created_at TIMESTAMPTZ, updated_at TIMESTAMPTZ

Relationships: many categories ↔ many articles (via newsapi_article_categories)

Data quality note: `news/*` categories look reasonable as broad topical labels. `dmoz/*` categories are noisy and over-attached — should be quality-tiered before influencing downstream logic.

Table: `public.newsapi_article_concepts` (junction)

- article_uri TEXT (FK → newsapi_articles.uri)
- concept_uri TEXT (FK → newsapi_concepts.uri)
- PK: (article_uri, concept_uri)

Table: `public.newsapi_article_categories` (junction)

- article_uri TEXT (FK → newsapi_articles.uri)
- category_uri TEXT (FK → newsapi_categories.uri)
- PK: (article_uri, category_uri)

Table: `public.pipeline_run_metrics`

Per-run observability for normalized pipeline outputs.

- PK: (run_id, ingestion_source, run_type, nth_run, metric_name)
- Tracks article counts, concept links, category links, and distinct dimensions per run

## Deprecated Source Table

Table: `public.headlines` (newsapi.org, deprecated)

- id TEXT PRIMARY KEY
- source_id TEXT, source_name TEXT, author TEXT
- title TEXT, description TEXT, url TEXT, url_to_image TEXT
- published_at TIMESTAMPTZ, content TEXT (99% truncated)
- created_at TIMESTAMPTZ, updated_at TIMESTAMPTZ, topic_id UUID

## Example Queries (Normalized Tables)

Top sources:
```sql
SELECT s.title, a.source_uri, COUNT(*) AS articles
FROM public.newsapi_articles a
LEFT JOIN public.newsapi_sources s ON s.uri = a.source_uri
GROUP BY s.title, a.source_uri
ORDER BY articles DESC LIMIT 25;
```

Recent articles for a concept:
```sql
SELECT a.uri, a.title, a.date_time_published
FROM public.newsapi_articles a
JOIN public.newsapi_article_concepts ac ON ac.article_uri = a.uri
WHERE ac.concept_uri = 'http://en.wikipedia.org/wiki/United_States'
ORDER BY a.date_time_published DESC LIMIT 50;
```

Top categories by article count:
```sql
SELECT c.uri, c.label, COUNT(*) AS article_count
FROM public.newsapi_article_categories ac
JOIN public.newsapi_categories c ON c.uri = ac.category_uri
GROUP BY c.uri, c.label
ORDER BY article_count DESC LIMIT 25;
```

## Enrichment Tables

Table: `public.validation_outputs`

- story_id TEXT PRIMARY KEY (FK → newsapi_articles.uri)
- headline_clean TEXT, story_text_clean TEXT
- body_valid BOOLEAN, body_headline_similarity REAL
- embedding VECTOR(384)
- top_category TEXT (sports|politics|business|entertainment|technology|health|environment|science|general)
- concept_labels JSONB, category_labels JSONB, named_entities JSONB
- dedup_status TEXT ('keep' | 'duplicate'), dedup_reason TEXT, dedup_kept_story_id TEXT
- validation_version TEXT

Table: `public.topics`

- topic_id UUID PRIMARY KEY
- label TEXT, centroid VECTOR(384)
- story_count INTEGER, dominant_category TEXT
- created_at TIMESTAMPTZ

Table: `public.headline_topic_assignments`

- story_id TEXT (FK), topic_id UUID (FK)
- composite_score REAL

Table: `public.public_interest_assessments`

- story_id TEXT PRIMARY KEY (FK)
- is_public_interest BOOLEAN, label TEXT, met_count INTEGER, confidence REAL
- material_impact BOOLEAN, institutional_action BOOLEAN, scope_scale BOOLEAN, new_information BOOLEAN
- assessment_json JSONB, evaluated_at TIMESTAMPTZ, pipeline_run_id UUID

Table: `public.scope_classifications`

- story_id TEXT PRIMARY KEY (FK)
- scope_status TEXT ('IN_SCOPE' | 'OUT_OF_SCOPE')

## Denormalized Output Table

Table: `public.enriched_headlines`

Pre-joined table refreshed via TRUNCATE + INSERT. Single table for Grafana and frontend.

Key columns: story_id, title, description, content, url, source_name, published_at, headline_clean, body_valid, top_category, topic_id, topic_label, cluster_size, composite_score, dominant_category, is_public_interest, pi_label, met_count, material_impact, institutional_action, scope_scale, new_information, scope_status, source_feed, enriched_at

## Analytics Tables

Table: `public.outlet_bias_scores`

- outlet_domain TEXT PRIMARY KEY
- source_name TEXT UNIQUE — join key to enriched_headlines.source_name
- allsides_bias_score DOUBLE PRECISION, adfontes_bias_score DOUBLE PRECISION
- adfontes_reliability_score DOUBLE PRECISION
- mbfc_bias_score DOUBLE PRECISION, mbfc_factual_rating TEXT
- composite_bias_score DOUBLE PRECISION, composite_bias_label TEXT
- political_group TEXT (Left|Center|Right), geo_group TEXT (US|Non-US)
- bias_last_verified_date DATE

## Analytics Views

View: `v_heatmap`

Topic x outlet matrix. Rows = topics by cluster_size DESC. Columns = outlets by adfontes_bias_score ASC (left→right politically). Cells: article_count, pi_count, noise_count, news_to_noise_ratio. Filtered to scope_status = 'IN_SCOPE'.

View: `v_heatmap_asymmetry`

Per-topic asymmetry metrics. political_skew = normalized (left - right) difference [-1, +1]. geo_skew = normalized (US - non-US) difference [-1, +1]. Center outlets excluded from political skew. Filtered to scope_status = 'IN_SCOPE'.
