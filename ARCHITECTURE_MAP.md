# Architecture Map

High-level architecture of the MediaDotGames platform.

Major subsystems:

1. Story Ingestion (NewsAPI.ai → `newsapi_articles`)
2. Story Storage (RDS PostgreSQL 15 + pgvector)
3. Validation & Embedding (text normalization, body quality, category assignment, embedding generation)
4. Topic Clustering (medoid-based assign-or-create with 168h temporal window + entity veto)
5. Public Interest Evaluation (LLM 4-criteria gate rule)
6. Scope Classification (IN_SCOPE / OUT_OF_SCOPE)
7. Bias & Coverage Analytics (outlet bias scores, heatmap views, asymmetry detection)
8. Ranking & Distribution Signals (composite ranking from media production + public attention + strategic importance signals) — planned
9. Visualization (Grafana dashboards)

```mermaid
flowchart TD
A[NewsAPI.ai] --> B[Story Ingestion]
B --> C[newsapi_articles]
C --> D[Validation & Embedding]
D --> E[validation_outputs]
E --> F[Topic Clustering]
F --> G[topics + assignments]
G --> H[Public Interest Evaluation]
H --> I[public_interest_assessments]
I --> J[Scope Classification]
J --> K[Refresh enriched_headlines]
K --> L[Bias & Coverage Analytics]
L --> M[v_heatmap + v_heatmap_asymmetry]
M --> N[Ranking & Distribution Signals]
N --> O[Grafana Visualization]
P[outlet_bias_scores] --> L
Q[Google Trends / NewsWhip / X Trends] -.-> N
```
