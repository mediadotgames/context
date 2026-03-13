# Clustering Pipeline

Purpose:

Group related stories into topics using multi-signal scoring.

## Deduplication (Pre-clustering)

Applied during story selection to prevent same-source duplicates from biasing cluster centroids.

Two rules, both keep the earliest record by `date_time_published`:

1. **title_and_source** — Same `title` + same `source->>'uri'`. Catches BBC .co.uk/.com variants, same-source reposts.
2. **url_normalized** — Same URL after stripping query parameters. Catches tracking-param URL variants.

Cross-source syndication (AP wire picked up by WaPo) is intentionally preserved — multiple outlets covering one story is a legitimate signal.

NewsAPI's `is_duplicate` field is not used. QA found it unreliable: misses same-source dupes, flags articles whose original isn't in our feed, inconsistent about which copy is the duplicate.

## Clustering Algorithm

Multi-signal assign-or-create with pgvector:

1. **Candidate retrieval** — pgvector returns top-5 nearest cluster centroids by embedding cosine distance
2. **Temporal gate** — Story must be within 72h of cluster's publication time range (hard cutoff)
3. **Category gate** (optional) — If enabled, story must share at least one category with the cluster (hard cutoff, not a weighted signal)
4. **Composite scoring** — `score = 0.65 × embedding_similarity + 0.35 × concept_overlap`
5. **Threshold check** — If best composite score >= 0.45, assign to that cluster. Otherwise create a new cluster.

### Signals

| Signal | Weight | Type | Description |
|--------|--------|------|-------------|
| Embedding similarity | 0.65 | Scored | Cosine similarity between story embedding and cluster centroid (384-dim, BAAI/bge-small-en-v1.5) |
| Concept overlap | 0.35 | Scored | Frequency-weighted overlap of NewsAPI concepts (entities: people, orgs, locations). Stop concepts filtered at 95th percentile. |
| Category match | n/a | Hard gate (optional) | NewsAPI hierarchical categories (e.g. `news/Politics`). When enabled, blocks cross-category clustering entirely. Not part of score. |
| Temporal window | n/a | Hard gate | 72h window. Stories outside the cluster's time range are rejected. |

### Concept Overlap Detail

- Concepts are normalized entity labels from NewsAPI (e.g. "donald trump", "iran", "boeing")
- Stop concepts (appearing in >95% of clusters) are excluded: "united states", "reuters", "associated press", etc.
- Overlap score: for each story concept found in the cluster, weight = cluster_count / story_count. Average across all non-stop concepts.

### Key Design Notes

- Category was originally a weighted signal (0.15 weight) but QA showed it inflated clusters — any articles sharing a broad category like `news/politics` would clear the threshold. Moved to hard gate.
- `event_uri` from NewsAPI is stored but not used in clustering logic. Reserved for future QA/validation.
- Category divergence within a cluster is expected to be a bug indicator. May be promoted from optional gate to hard invariant after further QA.