# Ranking & Distribution Signals

## Problem

The platform currently ranks stories by article counts only. This measures what newsrooms are producing, but not what the public is paying attention to. Distribution is a huge part of the media game — stories don't win just by being covered, they win by getting distributed. The platform needs a **distribution-aware ranking layer** that combines media production signals with public attention signals from channels like Google Search, X, TikTok, YouTube, and Facebook.

## Three Signal Families

Stories should be rankable across three independent dimensions:

### 1. Media Production Signals (current)

What newsrooms are covering.

- Story count per topic cluster
- Number of unique outlets covering a topic
- Source diversity (political/geographic spread)
- Recency / article velocity / acceleration
- Public interest score (LLM-evaluated)

### 2. Public Attention Signals (new — not yet implemented)

What the public is paying attention to.

- Search spikes (Google Trends)
- Social mentions and engagement velocity
- Cross-platform spread (story appears on multiple channels)
- Platform-specific trending status (X, YouTube, TikTok, Reddit)

### 3. Strategic Importance Signals (partially current)

What matters even if not yet huge.

- Public interest score (existing)
- Outlet influence weighting
- Novelty / newness of information
- "Under-covered but rising" — low media attention, growing public attention
- Coverage asymmetry (political/geographic skew)

## Ranking Views

These three signal families enable multiple product views, not one master list:

| View | Primary Signals | Description |
|---|---|---|
| Most Covered | Media production | Highest article count, most outlets |
| Trending Now | Public attention | Highest real-time social/search velocity |
| Breaking Out | Public attention + media acceleration | Rising fast across both channels |
| High Public Interest | Strategic importance | LLM-scored public interest regardless of volume |
| Under-covered but Rising | Divergence (low media, high public) | Stories the public cares about but newsrooms haven't caught up to |
| High Social / Low Newsroom | Divergence (high social, low media) | Distribution outrunning editorial attention |

The last two divergence views are especially valuable — they surface stories where media production and public attention are misaligned.

## External Signal Sources

### P1 — First integration targets (post-MVP)

**Google Trends**

- Official Google Trends API (alpha) with up to 5 years of data, regional breakdowns, interval aggregation
- Also exposes "Trending Now" surface with current trends
- Measures attention/curiosity, not just media output — good proxy for "what ordinary people are suddenly trying to understand"
- Less tied to any single social platform's culture
- Useful for both ranking and retrospective analysis
- **Recommended as the first external signal source**

**NewsWhip (vendor evaluation)**

- Spike product combines real-time web and social content with public engagement data
- Predicts which stories/topics will matter
- Tracks engagement across Facebook, X, TikTok, Instagram, Reddit, YouTube
- API available for integrating into custom dashboards and workflows
- **Strongest candidate for cross-platform distribution signal shortcut** — analogous to how NewsAPI.ai is the ingestion shortcut
- Avoids building and maintaining a brittle, fragmented social trend ingestion layer across five platforms with inconsistent access rules
- If budget allows, this is one of the first commercial tools to seriously evaluate after MVP

### P2 — Second wave

**X Trends**

- Official trends endpoints including Trends by WOEID (location-based)
- Fast-moving, high-news-density platform — often where news breaks and narratives crystallize
- Good for politics, media, sports, live events
- Caveats: can be noisy, brigaded, irony-heavy, not representative of broad public attention
- Should be **a signal, not the signal**

### P3 — Deeper channel coverage

**YouTube**

- Data API supports pulling most popular videos by region with category filters
- Underrated signal: reflects mainstream audience attention, creator-driven news commentary, story persistence beyond the first headline cycle
- Good for detecting when a story moves from "newsroom topic" to "broader public conversation"

**TikTok**

- Important conceptually, but official access is constrained
- Research Tools require non-commercial eligibility in specific regions
- More of a strategy problem than an engineering problem
- Likely accessed indirectly first: via NewsWhip data, manual exports, or compliant vendor solutions

**Facebook / Instagram / Meta**

- CrowdTangle is gone; replacement is Meta Content Library / API
- Application-based access for researchers, not straightforward commercial API access
- Same indirect-access approach as TikTok: vendor (NewsWhip) or constrained official access

## Buy vs Build

### Buy (signal inputs)

- **NewsWhip** — cross-platform engagement/distribution layer (evaluate post-MVP)
- **Google Trends** — direct first-party attention signal (own it yourself)
- This combination is attractive because NewsWhip gives broad cross-platform coverage while Google Trends gives a durable, interpretable attention signal

### Build (ranking logic)

- **Composite ranking function** — vendors provide inputs; we own the ranking logic
- This keeps the product defensible

## Composite Trend Score (conceptual)

```
trend_score =
    w1 * article_velocity_in_cluster
  + w2 * unique_outlet_acceleration
  + w3 * google_search_spike
  + w4 * x_trend_presence
  + w5 * youtube_popularity_proxy
  + w6 * cross_platform_presence_bonus
  + w7 * public_interest_multiplier
  - w8 * stale_age_decay
  - w9 * duplicate_low_information_penalty
```

Weights are tunable. Channels are pluggable inputs — adding a new source means adding a new weighted term, not redesigning the formula.

## Schema Design Principle

Even before external signals are integrated, the schema should support attaching trend signals at three levels:

- **Cluster/topic level** — trend signals for the event/story
- **Concept/entity level** — trend signals for people, organizations, locations
- **Keyword/topic level** — trend signals for search terms and hashtags

This means a normalized "trend subject" concept is needed, even if crude at first.

## Phased Approach

### Now (MVP)

- Keep article-count-based ranking
- Design schema so external signals can be attached at cluster, concept, and keyword levels
- No external API integrations yet

### P1 (post-MVP)

- Google Trends API integration
- NewsWhip vendor evaluation
- Ranking schema that supports external signals (`trend_signals` table)
- Composite ranking function with tunable weights

### P2

- X Trends API integration
- Divergence dashboards: media attention vs public attention
- "Under-covered but rising" and "High social / low newsroom" views

### P3

- Deeper channel coverage: YouTube, Reddit, TikTok, Meta
- Dependent on access constraints, cost, and vendor relationships

## Key Design Decision

> Do not make "trending" a single raw feed. Make it a composite ranking framework where channels are pluggable inputs. That will age much better.
