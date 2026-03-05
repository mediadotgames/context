# Architecture Map

High-level architecture of the MediaDotGames platform.

Major subsystems:

1. Story Ingestion
2. Story Storage
3. Topic Clustering
4. Public Interest Evaluation
5. Coverage Analytics
6. Visualization

Mermaid diagram:

```mermaid
flowchart TD
A[News APIs] --> B[Story Ingestion]
B --> C[Postgres Story Database]
C --> D[Topic Clustering Service]
D --> E[Topic Database]
E --> F[Public Interest Evaluation]
F --> G[Public Interest Store]
G --> H[Coverage Analytics]
H --> I[Grafana Visualization]
```