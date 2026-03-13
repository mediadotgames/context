# Open Questions

## OQ-1: Categories for clustering

**Decision needed**: Should normalized categories (`newsapi_categories`) be used as a clustering signal?
**Options**:
- Use `news/*` categories as a hard gate (category_gate=True) — risk of inflating clusters on broad categories like news/politics
- Use categories as a weighted signal in composite score — adds a third dimension but needs weight tuning
- Quality-tier categories first, then use only high-quality tiers — more work but cleaner signal
- Keep category_gate=False, rely on entity veto + embedding similarity — current approach, working reasonably
**What unblocks it**: Run clustering quality analysis on full 27-day backfill. Compare clusters with and without category signals.
**Priority**: Medium — not blocking but could improve clustering quality

## OQ-2: Enrichment pipeline refactor to normalized tables

**Decision needed**: Should `validation.py` and `source_mapping.py` switch from JSONB extraction to querying normalized tables (newsapi_sources, newsapi_concepts, newsapi_categories)?
**Options**:
- Refactor to use normalized joins — simpler Python code, cleaner data, but adds DB round-trips
- Keep JSONB extraction — works fine, no migration risk, already tested
- Hybrid: use normalized tables for new features, keep JSONB extraction for existing code
**What unblocks it**: Measure performance impact of normalized joins vs JSONB extraction.
**Priority**: Low — current approach works, refactor when convenient

## OQ-3: Concept integration into clustering

**Decision needed**: How/when to use `newsapi_concepts` in the clustering pipeline?
**Options**:
- Replace spaCy NER with NewsAPI concepts for entity overlap — concepts have broader coverage but were found noisy in earlier experiments
- Use concepts as additional signal alongside NER — more signals but more complexity
- Use concepts for post-clustering analysis only (co-occurrence, frequency) — no pipeline change
**What unblocks it**: Quality analysis of normalized concepts vs spaCy NER. Compare overlap, noise, coverage.
**Priority**: Medium — concepts are "relatively strong and useful" per data analysis

## OQ-4: PI evaluation speed / model alternatives

**Decision needed**: How to run PI evaluation across 40K+ stories without hitting rate limits?
**Options**:
- OpenAI Batch API — already built (batch.py), separate rate limit pool, 50% cheaper, async (results in hours). **Recommended first step.**
- Ollama local — zero rate limits/cost, GPU workstation available, quality unknown for this task
- Groq/Together/Fireworks — OpenAI-compatible APIs, higher rate limits, zero code changes
- Increase OpenAI concurrency — currently max_concurrent=1, could bump but hits ceiling faster
**What unblocks it**: Try Batch API for backfill. Run Ollama benchmark on sample using PI evaluation framework.
**Priority**: High — blocking full corpus PI evaluation

## OQ-5: Sports mega-cluster

**Decision needed**: How to prevent different sports games from clustering together?
**Options**:
- Max cluster size cap — simple but arbitrary
- Temporal windowing — already at 168h, could tighten for sports
- Category-based scope filtering — remove sports from clustering
- Sub-clustering with tighter thresholds for large clusters
- Solve at logic layer (user preference) rather than stacking rules
**What unblocks it**: Analyze sports cluster composition on full backfill. Understand failure modes.
**Priority**: Medium — known issue, not blocking MVP

## OQ-6: Video transcript ingestion

**Decision needed**: Is it feasible to ingest CC/subtitle transcripts from news video pages?
**Options**:
- Extract CC data from video page HTML — feasibility unknown, likely site-specific
- Use YouTube API for news channel videos — more structured but limited to YouTube
- Use a vendor/service for video transcript extraction
- Defer entirely — focus on print stories for now
**What unblocks it**: Technical spike on CC extraction from 2-3 news sites. Assess effort vs coverage.
**Priority**: Low — long-term future exploration, not blocking current work
