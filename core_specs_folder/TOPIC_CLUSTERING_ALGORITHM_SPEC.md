# Topic Clustering Algorithm Specification

Status: Living Specification  
Owner: MediaDotGames Platform

This document defines the algorithm used to cluster news stories into topics.

## 1. System Purpose

The clustering system groups related stories referring to the same real-world event.

Each cluster represents a **topic**.

Clustering enables:

- topic-level analysis
- public interest evaluation
- coverage analysis

## 2. Input Dataset

Stories are stored in the table:

`public.headlines`

Relevant fields:

- `id`
- `title`
- `description`
- `content`
- `published_at`
- `source_name`

## 3. Preprocessing

Story text is normalized before embedding.

Input text for embeddings:

- `title`
- `description`
- `first 500 characters of content`

Combined format:

```text
TITLE: {title}

DESCRIPTION: {description}

CONTENT: {snippet}
```

## 4. Embedding Generation

Each story is converted into a vector embedding.

Possible embedding models:

- sentence-transformers
- OpenAI embedding models
- local transformer embedding models

Embedding dimension depends on the model used.

## 5. Similarity Calculation

Similarity between stories is computed using cosine similarity.

`similarity = cosine_similarity(vector_a, vector_b)`

Similarity range:

`-1 to 1`

## 6. Clustering Algorithm

Preferred algorithm:

`HDBSCAN`

Reasons:

- no need to predefine cluster count
- robust to noise
- handles uneven cluster sizes

Alternative:

`Agglomerative clustering`

## 7. Clustering Parameters

Initial parameter suggestions:

- `minimum_cluster_size = 3`
- `similarity_threshold ≈ 0.80`

Stories below similarity threshold remain unclustered.

## 8. Topic Creation

Each cluster becomes a topic.

Topic attributes:

- `topic_id`
- `representative_story_id`
- `cluster_size`
- `embedding_centroid`
- `created_at`

## 9. Representative Story Selection

The representative story is chosen as:

`story with smallest average distance to other cluster members`

This story becomes the cluster summary input.

## 10. Database Updates

Stories receive the assigned `topic_id`.

Example update:

```sql
UPDATE public.headlines
SET topic_id = {topic_id}
WHERE id = {story_id}
```

## 11. Re-Clustering Strategy

Clusters may evolve over time as new stories arrive.

Two strategies are possible:

- Option A: daily batch clustering
- Option B: incremental clustering

Initial implementation should use **daily batch clustering**.

## 12. Output Data

Topic table example:

`topics`

Fields:

- `id`
- `representative_story_id`
- `cluster_size`
- `embedding_centroid`
- `created_at`
