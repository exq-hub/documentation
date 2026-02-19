---
title: Faceted search
description: Metadata-only search using the filter expression tree, with no vector similarity involved.
icon: filter
---

Faceted search returns items that satisfy a filter expression tree without performing any vector similarity computation. It is the right choice when the user wants to browse or count items by metadata — category, date range, source type, etc. — rather than by visual content.

## How it works

1. **Compile filters** — The `ActiveFilters` tree is recursively compiled into a SQL `WHERE` clause by `db_helper.py`. Each leaf becomes a join on the tag tables; groups become parenthesised AND/OR blocks; `not_` flags add `NOT` operators.
2. **Query the database** — The compiled query runs against the SQLite database for the active collection, returning up to `n` matching media IDs.
3. **Return results** — The matching IDs are returned in the `suggestions` array in the standard search response shape.

Because no embedding lookup is involved, faceted search is very fast regardless of collection size.

## When to use faceted search

| Use case | Recommended strategy |
|----------|---------------------|
| "Show me all videos from 2023" | Faceted |
| "Show me sports clips" | Faceted (if category is a filter) or CLIP |
| "Find more images like this one" | Relevance feedback |
| "Find images of a sunset on a beach" | CLIP |
| "Find sports clips from 2023 that look like this" | CLIP or RF + filters |

## Filter requirement

Unlike CLIP and RF search, the `filters` field is **required** for faceted search. An empty or null filter would match every item and is not useful for this strategy.

See [Filters](/filters/) for the full filter expression reference and examples.

## Request reference

See [Search endpoints — POST /exq/search/faceted](/api/search/#post-exqsearchfaceted) for the full request and response schema.
