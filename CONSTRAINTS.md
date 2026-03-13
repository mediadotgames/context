# Constraints & Risks

## Technical Constraints

### OpenAI rate limits
PI evaluation via gpt-4o-mini hits rate limits when processing at scale. Current pipeline runs sequentially (max_concurrent=1, request_delay=1.0) which is ~1 req/sec. At 40K stories, this is ~11 hours before rate limit delays. Batch API (already built) is the recommended path for bulk processing.

### pgvector sequential queries
Clustering issues one pgvector similarity query per story. No batch similarity search. Could use HNSW index for faster approximate search at scale.

### Embedding model lock-in
All embeddings stored as VECTOR(384) from bge-small-en-v1.5. Changing to a different model (e.g. bge-base 768d) requires re-embedding the entire corpus and updating the vector column dimension.

### Single-threaded pipeline
Pipeline runs stages sequentially within a single process. No parallelism across stages or within stages (except embedding generation which uses sentence-transformers batching).

### PostgreSQL as sole datastore
All data lives in one RDS PostgreSQL 15 instance (db.t3.micro). No read replicas, no OLAP engine. Analytics queries may slow as data grows beyond 100K+ articles.

### Category taxonomy noise
`dmoz/*` categories from NewsAPI.ai are noisy and over-attached. Only `news/*` categories are reliable for downstream logic. Any category-based feature needs quality tiering first.

## Known Risks

### Sports mega-cluster
Structural similarity in sports game body text ("scored", "victory", "tournament") causes different games to cluster together regardless of threshold settings. Entity veto helps but doesn't fully solve it. Observed across all clustering experiments.

### High singleton rate
~81% of stories remain unclustered (singletons) regardless of threshold. Body text and concept overlap help but many stories are genuinely unique or have insufficient overlap with existing clusters.

### PI evaluation quality unknown at scale
LLM accuracy on PI classification has not been benchmarked against human judgment yet. The benchmark framework is built but no labels have been collected. Quality could vary by topic, outlet, or article length.

### Rate limit fragility
No circuit breaker or adaptive rate limiting. Pipeline retries 8 times with exponential backoff on 429s but will eventually fail and halt. No partial recovery — must restart from the beginning of the stage.

### Centroid column naming
topics.centroid actually stores the medoid embedding, not a true centroid. Column name is misleading but kept for pgvector index compatibility. Could cause confusion for new developers.

## Tech Debt

### P1 — Should fix soon
1. **Sports mega-cluster** — needs structural solution, not just threshold tuning
2. **No error recovery/retry** — pipeline halts on failures, no partial restart
3. **Pipeline run stats** — no per-stage timing, threshold, or model version tracking

### P2 — Address when scaling
4. **CLUSTER_BODY_CHAR_LIMIT hardcoded** — 500 chars, should be in Settings
5. **Sequential clustering** — one pgvector query per story
6. **`public.headlines` deprecation** — remove source mapping abstraction
7. **`snippet_valid` column** — soft-deprecated, safe to DROP

### P3 — Nice to have
8. **Embedding model upgrade** — bge-small (384d) works; bge-base (768d) may improve quality
9. **LLM provider abstraction** — only OpenAI implemented; Anthropic stub exists
10. **Enrichment pipeline → normalized tables** — still uses JSONB extraction in Python
