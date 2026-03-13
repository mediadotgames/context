# Ingestion Pipeline

Purpose:

Collect news stories from NewsAPI.ai into `public.newsapi_articles`.

Processing Steps:

1. Fetch stories from NewsAPI.ai (full body text, metadata, concepts, categories)
2. Normalize story schema
3. Insert stories into PostgreSQL

Output:

Rows inserted into `public.newsapi_articles`. This is the sole active ingestion feed.

Note: `public.headlines` (newsapi.org) is deprecated — 99% truncated content, ingestion outages, strictly inferior coverage. All enrichment pipeline stages use `newsapi_articles` exclusively.
