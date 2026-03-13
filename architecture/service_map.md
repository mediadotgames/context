# Service Map

Logical services in the platform.

## Story Ingestion Service

Collect stories from NewsAPI.ai. Writes to `public.newsapi_articles`.

## Enrichment Pipeline

Python-based 7-stage pipeline (`headlines-enrichment` repo):

1. **Story Selection** — Select unprocessed articles
2. **Validation** — Text normalization, body quality check (cleanliness + semantic consistency), top category assignment, embedding generation
3. **Clustering** — Multi-signal assign-or-create with pgvector
4. **Labeling** — Assign human-readable labels to clusters
5. **PI Evaluation** — LLM-based public interest scoring
6. **Scope Classification** — IN_SCOPE / OUT_OF_SCOPE determination
7. **Refresh** — Materialize `enriched_headlines` pre-joined table

## Bias & Coverage Analytics Service

Compute coverage metrics across outlets:

- **outlet_bias_scores** — Composite bias scores from AllSides, Ad Fontes Media, MBFC for 14 tracked outlets
- **v_heatmap** — Topic x outlet matrix with article counts and news-to-noise ratio
- **v_heatmap_asymmetry** — Political skew (left vs right) and geographic skew (US vs non-US) per topic

## Ranking & Distribution Signals Service (planned)

Composite ranking layer combining three signal families:

- **Media production signals** (current) — article velocity, outlet count, source diversity, PI score
- **Public attention signals** (planned) — Google Trends search spikes, social engagement via NewsWhip, X Trends, YouTube popularity
- **Strategic importance signals** (partial) — PI score, novelty, coverage asymmetry, divergence between media and public attention

External signal sources (phased):
- P1: Google Trends API, NewsWhip evaluation
- P2: X Trends API, divergence dashboards
- P3: YouTube, Reddit, TikTok/Meta (vendor-dependent)

Key design principle: "Trending" is a composite ranking framework where channels are pluggable inputs, not a single raw feed.

See `context/pipelines/ranking_distribution_signals.md` for full strategy and `context/requirements/ranking_requirements.md` for requirements.

## Visualization Service

Grafana dashboards querying `enriched_headlines`, `v_heatmap`, and `v_heatmap_asymmetry`.
