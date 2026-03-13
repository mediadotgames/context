# System Architecture

The MediaDotGames platform transforms raw news data into structured analytical signals.

## Major Components

1. **News ingestion** — NewsAPI.ai → `public.newsapi_articles`
2. **Story storage** — Amazon RDS PostgreSQL 15 + pgvector 0.8.0
3. **Validation & embedding** — Text normalization, two-phase body quality check, top category assignment, embedding generation (BAAI/bge-small-en-v1.5, 384-dim)
4. **Topic clustering** — Medoid-based assign-or-create: three-stage (pgvector candidates → temporal/entity veto filtering → composite scoring). 168h temporal window, entity veto requires shared NER entity, dynamic stop entity detection
5. **Public interest evaluation** — LLM-based 4-criteria gate rule (OpenAI API)
6. **Scope classification** — IN_SCOPE / OUT_OF_SCOPE based on PI result + cluster size
7. **Bias & coverage analytics** — Outlet bias scores (AllSides, Ad Fontes, MBFC), heatmap views, political/geographic asymmetry detection
8. **Ranking & distribution signals** (planned) — Composite ranking from media production signals (article velocity, outlet count), public attention signals (Google Trends, social engagement), and strategic importance signals (PI score, novelty, divergence). See `context/pipelines/ranking_distribution_signals.md`
9. **Visualization** — Grafana dashboards, `enriched_headlines` as single pre-joined table

## Data Flow

```
NewsAPI.ai
↓
Ingestion Pipeline → newsapi_articles
↓
Validation & Embedding → validation_outputs
↓
Deduplication (title+source, URL-normalized)
↓
Topic Clustering → topics + headline_topic_assignments
↓
Cluster Labeling → topic labels
↓
Public Interest Evaluation → public_interest_assessments
↓
Scope Classification → scope_classifications
↓
Refresh → enriched_headlines (pre-joined, denormalized)
↓
Bias & Coverage Analytics (joined with outlet_bias_scores)
↓
v_heatmap + v_heatmap_asymmetry
↓
Ranking & Distribution Signals (planned: Google Trends, NewsWhip, X Trends → trend_signals → composite ranking)
↓
Grafana + Frontend
```
