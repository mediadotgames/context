# MediaDotGames Context Repository

This repository contains the **complete architecture and product context** for the MediaDotGames News Analysis Platform.

This documentation exists to provide:

- system architecture documentation
- product vision and mission
- data model specifications
- pipeline definitions
- API contracts
- infrastructure plans
- ML / LLM usage patterns

This repo is intentionally **separate from implementation repos** so that engineers and AI coding agents can reason about the system holistically.

Expected scale:

50,000+ news stories per day

Primary processing model:

Daily batch pipelines

Primary technologies:

Python
Amazon RDS PostgreSQL 15 + pgvector
LLM services (OpenAI API)
Grafana visualization

Key capabilities:

- Ingest 50K+ stories/day from NewsAPI.ai
- Cluster stories into topics using embedding similarity + concept overlap
- Evaluate public interest via LLM-based 4-criteria gate rule
- Detect coverage asymmetry across outlet political and geographic groups
- Visualize bias, coverage gaps, and news-to-noise ratios via heatmap views
- Rank stories by composite signals including distribution channel trends (planned)

This is a **living specification** and will evolve as the platform develops.