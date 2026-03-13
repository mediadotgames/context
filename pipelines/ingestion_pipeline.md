# Ingestion Pipeline

Purpose:

Collect news stories from NewsAPI.ai and store them in a hybrid model: raw JSONB preserved on `newsapi_articles` plus normalized relational dimension tables for efficient querying.

## Processing Steps

1. **Collect** — Fetch stories from NewsAPI.ai (full body text, metadata, concepts, categories)
2. **Load articles** — Insert/upsert into `newsapi_articles` with raw JSONB columns intact
3. **Normalize sources** — Upsert canonical source records into `newsapi_sources`
4. **Normalize concepts** — Upsert canonical concept records into `newsapi_concepts`, link via `newsapi_article_concepts`
5. **Normalize categories** — Upsert canonical category records into `newsapi_categories`, link via `newsapi_article_categories`
6. **Record metrics** — Write run-level observability to `pipeline_run_metrics`

## Output Tables

- `newsapi_articles` — core article fact table (raw JSONB preserved)
- `newsapi_sources` — canonical source dimension (one row per publisher)
- `newsapi_concepts` — canonical concept dimension (person, org, loc, wiki)
- `newsapi_categories` — canonical category dimension (hierarchical via parent_uri)
- `newsapi_article_concepts` — article ↔ concept junction
- `newsapi_article_categories` — article ↔ category junction
- `pipeline_run_metrics` — per-run counts and observability

## Design Decisions

- **Hybrid model**: Raw JSONB preserved for auditability, replay, and debugging. Normalized tables enable fast indexed joins for analytics.
- **Foreign keys enforced** between articles and dimension tables
- **Historical data backfilled** into normalized tables
- **Loaders updated** so all new runs populate normalized tables automatically
- **Category quality**: `news/*` categories look reasonable as broad topical labels. `dmoz/*` categories appear noisy and over-attached — should be quality-tiered before influencing downstream logic.
- **Concepts**: Look relatively strong and useful as semantic features for clustering, entity linking, and co-occurrence analysis.

## Query Guidance

Prefer normalized table joins over JSONB extraction:
```sql
-- Use this (normalized join):
SELECT s.title, COUNT(*) FROM newsapi_articles a
JOIN newsapi_sources s ON s.uri = a.source_uri GROUP BY s.title;

-- Not this (JSONB extraction):
SELECT a.source->>'title', COUNT(*) FROM newsapi_articles a GROUP BY a.source->>'title';
```

## Notes

- `public.headlines` (newsapi.org) is deprecated — 99% truncated content, ingestion outages, strictly inferior coverage
- All enrichment pipeline stages use `newsapi_articles` exclusively
- The enrichment pipeline currently still extracts concepts/categories from JSONB in Python (via `source_mapping.py` and `validation.py`) — switching to normalized table queries is a potential future optimization
