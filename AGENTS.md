# AGENTS.md

This repository contains the **system context** for the MediaDotGames platform.

AI coding agents should read the following files first:

1. architecture/system_architecture.md
2. data_model/entities.md
3. pipelines/ingestion_pipeline.md
4. pipelines/clustering_pipeline.md
5. pipelines/public_interest_pipeline.md

The system processes news stories through the following pipeline:

Story Ingestion → Topic Clustering → Public Interest Evaluation → Coverage Analysis → Visualization

Important principles:

- Deterministic processing
- Reproducibility
- Auditable LLM decisions
- Batch-oriented pipelines