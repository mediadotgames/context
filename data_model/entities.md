# Core Entities

## Story (newsapi_articles)

Represents a single news article from NewsAPI.ai.

Attributes:

- uri (PK) — NewsAPI.ai unique identifier
- title — Article headline
- body — Full article body text
- url — Source article URL
- source — JSONB with publisher metadata (uri, title, domain)
- categories — JSONB array of hierarchical categories (e.g. `news/Politics`)
- concepts — JSONB array of entities (people, orgs, locations)
- date_time_published — Original publication timestamp
- date_time — NewsAPI.ai indexing timestamp

Relationships:

- Story → belongs_to → Topic (via headline_topic_assignments)
- Story → has → ValidationOutput
- Story → has → PublicInterestAssessment
- Story → produced_by → NewsSource (via source JSONB)

## ValidationOutput

Enrichment output from the validation stage.

Attributes:

- story_id (PK, FK → newsapi_articles.uri)
- headline_clean — Normalized headline text
- story_text_clean — Normalized body text
- body_valid — Boolean: passes both text cleanliness AND semantic consistency (headline-body cosine sim >= 0.3)
- body_headline_similarity — Cosine similarity between headline and body[:500] embeddings
- embedding — 384-dim vector (BAAI/bge-small-en-v1.5)
- top_category — One of: sports, politics, business, entertainment, technology, health, environment, science, general
- concept_labels — JSONB array of normalized concept labels
- category_labels — JSONB array of category URI paths
- named_entities — JSONB array of spaCy NER entities
- dedup_status — 'keep' or 'duplicate'
- dedup_reason — NULL or comma-separated rule names
- validation_version — Schema version (currently v2.0)

## Topic

Represents a cluster of related stories.

Attributes:

- topic_id (PK, UUID)
- label — Human-readable label (medoid headline)
- centroid — Cluster centroid embedding vector
- story_count — Number of stories in cluster
- dominant_category — Most common top_category in cluster
- created_at

Relationships:

- Topic → contains → Stories (via headline_topic_assignments)

## PublicInterestAssessment

LLM-based public interest evaluation result.

Attributes:

- story_id (PK, FK)
- is_public_interest — Boolean gate result
- label — High / Moderate / Low / Not Public Interest
- met_count — Number of criteria met (0-4)
- confidence — Confidence score
- material_impact — Boolean criterion
- institutional_action — Boolean criterion
- scope_scale — Boolean criterion
- new_information — Boolean criterion
- assessment_json — JSONB with prompt version, model, raw response, reasoning
- evaluated_at — Timestamp
- pipeline_run_id — FK to pipeline run

## OutletBiasScore

Outlet-level bias scores from multiple rating services.

Attributes:

- outlet_domain (PK) — e.g. 'reuters.com'
- source_name (UNIQUE) — Display name matching enriched_headlines.source_name
- allsides_bias_score — AllSides scale (-6 to +6)
- adfontes_bias_score — Ad Fontes Media scale (-42 to +42)
- adfontes_reliability_score — Ad Fontes reliability (0-64)
- mbfc_bias_score — MBFC scale (-10 to +10)
- mbfc_factual_rating — MBFC factual rating (Very High, High, Mostly Factual, Mixed, Low)
- composite_bias_score — Normalized average on AllSides scale
- composite_bias_label — Left / Lean Left / Center / Lean Right / Right
- political_group — Left / Center / Right (for asymmetry analysis)
- geo_group — US / Non-US (for asymmetry analysis)
- bias_last_verified_date

## EnrichedHeadline

Denormalized pre-joined table for Grafana and frontend consumption. Refreshed via TRUNCATE + INSERT.

Key columns: story_id, title, content, source_name, published_at, headline_clean, body_valid, top_category, topic_id, topic_label, cluster_size, is_public_interest, pi_label, scope_status, source_feed.

## TrendSignal (planned — not yet implemented)

External distribution/attention signal attached to a trend subject.

Planned attributes:

- id (PK)
- source — signal provider (google_trends, newswhip, x_trends, youtube, etc.)
- subject_type — 'topic', 'entity', or 'keyword'
- subject_id — FK to topic_id, concept label, or keyword string
- score — normalized signal value
- raw_data — JSONB for source-specific metadata
- observed_at — timestamp of observation
- expires_at — optional TTL for stale signals

Relationships:

- TrendSignal → attached_to → Topic (when subject_type = 'topic')
- TrendSignal → attached_to → Concept/Entity (when subject_type = 'entity')
- TrendSignal → attached_to → Keyword (when subject_type = 'keyword')

See `context/pipelines/ranking_distribution_signals.md` and `context/requirements/ranking_requirements.md` for full strategy.
