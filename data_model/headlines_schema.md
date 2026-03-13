# Database Schema

## Primary Source Table

Table: `public.newsapi_articles` (active feed)

- uri TEXT PRIMARY KEY
- title TEXT
- body TEXT
- url TEXT
- source JSONB — `{"uri": "bbc.com", "title": "BBC", ...}`
- categories JSONB — `[{"uri": "news/Politics", "wgt": 100}, ...]`
- concepts JSONB — `[{"uri": "...", "label": {"eng": "..."}, "type": "..."}, ...]`
- date_time_published TIMESTAMPTZ
- date_time TIMESTAMPTZ
- is_duplicate BOOLEAN (not used — unreliable)

## Deprecated Source Table

Table: `public.headlines` (newsapi.org, deprecated)

- id TEXT PRIMARY KEY
- source_id TEXT, source_name TEXT, author TEXT
- title TEXT, description TEXT, url TEXT, url_to_image TEXT
- published_at TIMESTAMPTZ, content TEXT (99% truncated)
- created_at TIMESTAMPTZ, updated_at TIMESTAMPTZ, topic_id UUID

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
