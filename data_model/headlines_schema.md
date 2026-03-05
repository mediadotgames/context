# Headlines Table Schema

Table: public.headlines

id text PRIMARY KEY
source_id text
source_name text
author text
title text
description text
url text
url_to_image text
published_at timestamptz
content text
created_at timestamptz
updated_at timestamptz
topic_id uuid