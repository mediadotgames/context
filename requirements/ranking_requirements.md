# Ranking & Distribution Signals — Requirements

## Business Requirements

### BR-1: Multi-dimensional story ranking

Sort and surface stories by multiple independent dimensions — not just article count. Users should be able to view stories ranked by coverage volume, trending velocity, public interest, and divergence (media vs public attention).

### BR-2: Distribution channel visibility

Provide visibility into what is trending across major distribution channels (Google Search, X, YouTube, TikTok, Facebook/Meta, Reddit). The product should reflect what people are talking about, not just what newsrooms are publishing.

### BR-3: Media vs public attention divergence detection

Detect and surface stories where media production and public attention are misaligned:
- High media coverage, low public attention (over-covered)
- Low media coverage, high public attention (under-covered but rising)
- High social circulation, low newsroom attention

### BR-4: Multiple rank views in the product

Support distinct ranking modes (tabs or views) in the frontend:
- Most Covered
- Trending Now
- Breaking Out
- High Public Interest
- Under-covered but Rising
- High Social / Low Newsroom

### BR-5: Pluggable ranking inputs

Ranking inputs must not be locked to a single vendor or platform. The system should support adding, removing, or reweighting signal sources without redesigning the ranking logic.

### BR-6: Defensible ranking logic

Vendors provide signal inputs; we own and control the composite ranking formula. The ranking logic is a core product differentiator and must remain in-house.

---

## Technical Requirements

### TR-1: Normalized trend subject concept

Create a normalized "trend subject" abstraction for attaching external signals at three levels:
- **Cluster/topic level** — the event or story cluster
- **Concept/entity level** — people, organizations, locations (from NewsAPI concepts / spaCy NER)
- **Keyword/topic level** — search terms, hashtags, trending phrases

This is a prerequisite for any external signal integration.

### TR-2: Trend signal ingestion pipeline

Build a pipeline that periodically fetches trend data from external APIs, normalizes it, and stores it in PostgreSQL. Must support:
- Multiple sources with different polling intervals
- Normalization to a common schema
- Historical retention for trend analysis
- Graceful handling of API rate limits and outages

### TR-3: Composite ranking function

Implement a scoring function that combines internal signals (article velocity, outlet count, PI score) with external signals (search trends, social engagement) using tunable weights. Must support:
- Per-view weight profiles (e.g., "Trending Now" weights social signals higher)
- Staleness decay (time-sensitive ranking)
- Easy addition of new signal terms

### TR-4: trend_signals table schema

Design and implement a `trend_signals` table (or equivalent) with at minimum:
- `source` — signal provider (google_trends, newswhip, x_trends, etc.)
- `subject_type` — topic, entity, keyword
- `subject_id` — FK to topic_id, concept label, or keyword string
- `score` — normalized signal value
- `raw_data` — JSONB for source-specific metadata
- `observed_at` — timestamp of observation
- `expires_at` — optional TTL for stale signals

### TR-5: Google Trends API integration (P1)

Integrate Google Trends API to fetch:
- Trending searches (real-time and daily)
- Interest-over-time for topic cluster keywords/entities
- Regional breakdowns where useful
- Store as normalized trend_signals

### TR-6: NewsWhip evaluation and potential integration (P1)

Evaluate NewsWhip Spike API for:
- Cross-platform engagement data (Facebook, X, TikTok, Instagram, Reddit, YouTube)
- Prediction scores for story virality
- API access model, pricing, rate limits
- Data freshness and coverage completeness
- Integration feasibility with our schema

### TR-7: X Trends API integration (P2)

Integrate X Trends endpoints:
- Trending topics by WOEID (location-based)
- Match trending topics to existing topic clusters / entities
- Store as normalized trend_signals
- Handle noise filtering (brigading, irony, non-news trends)

### TR-8: Staleness decay function

Implement time-based decay for ranking scores:
- Recent signals weighted more heavily than older ones
- Configurable half-life per signal source
- Prevents stale trends from persisting in "Trending Now" views

### TR-9: Divergence metrics

Compute divergence between media production and public attention per topic:
- `media_attention` — article count, outlet count, velocity (internal signals)
- `public_attention` — search trends, social engagement (external signals)
- `divergence = public_attention - media_attention` (normalized)
- Expose as a rankable/filterable dimension

### TR-10: Ranked view API endpoints

Expose ranked topic/story lists through the backend API:
- Parameterized by view type (most_covered, trending_now, breaking_out, etc.)
- Support pagination, filtering by category/date/scope
- Return composite scores and component signal breakdowns

---

## Traceability

| Business Req | Technical Reqs | Phase |
|---|---|---|
| BR-1 Multi-dimensional ranking | TR-3, TR-8 | P1 |
| BR-2 Channel visibility | TR-2, TR-5, TR-6, TR-7 | P1-P2 |
| BR-3 Divergence detection | TR-9 | P2 |
| BR-4 Multiple rank views | TR-3, TR-10 | P1-P2 |
| BR-5 Pluggable inputs | TR-1, TR-4 | P1 (schema) |
| BR-6 Defensible logic | TR-3 | P1 |

## Priority Order

1. **TR-1 + TR-4** — Schema readiness (trend subject + trend_signals table). Prerequisite for everything else.
2. **TR-3 + TR-8** — Composite ranking function with staleness decay. Enables multi-dimensional ranking using internal signals immediately.
3. **TR-5** — Google Trends integration. First external signal.
4. **TR-6** — NewsWhip evaluation. Potentially the biggest single unlock for cross-platform signals.
5. **TR-2** — Trend signal ingestion pipeline (generalized, once we have 2+ sources).
6. **TR-7** — X Trends integration.
7. **TR-9** — Divergence metrics.
8. **TR-10** — Ranked view API endpoints.
