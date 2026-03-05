# Ingestion Pipeline

Purpose:

Collect news stories from external sources.

Processing Steps:

1. Fetch stories from APIs
2. Normalize story schema
3. Deduplicate based on URL and title similarity
4. Insert stories into Postgres

Output:

Rows inserted into public.headlines