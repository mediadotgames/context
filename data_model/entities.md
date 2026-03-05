# Core Entities

## Story

Represents a single news article.

Attributes

id
source_id
source_name
title
description
content
url
published_at
topic_id

Relationships

Story → belongs_to → Topic
Story → produced_by → NewsSource

## Topic

Represents a cluster of related stories.

Attributes

id
representative_story_id
embedding
created_at

Relationships

Topic → contains → Stories

## PublicInterestJudgment

Attributes

id
topic_id
judgment_boolean
criteria_scores_json
reasoning_summary
evaluation_timestamp