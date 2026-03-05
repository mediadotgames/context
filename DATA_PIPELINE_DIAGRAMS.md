# Data Pipeline Diagrams

## Story Ingestion Pipeline

News APIs
→ Ingestion workers
→ Normalize article schema
→ Deduplicate stories
→ Store in Postgres

## Topic Clustering Pipeline

Stories
→ Generate embeddings
→ Similarity clustering
→ Assign topic_id
→ Store topic clusters

## Public Interest Evaluation Pipeline

Topic cluster
→ Generate topic summary
→ LLM evaluation
→ Store public interest judgment

## Coverage Analysis Pipeline

Judgments + Topics
→ Compute outlet coverage
→ Aggregate coverage metrics
→ Export visualization dataset