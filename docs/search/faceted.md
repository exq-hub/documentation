---
title: Faceted search
description: Metadata-only search using the filter expression tree, with no vector similarity involved.
icon: filter
---

Faceted search returns items that satisfy a filter expression tree without performing any vector similarity computation. It is the right choice when the user wants to browse or count items by metadata, such as category or time, rather than by visual content.

## How it works

1. **Compile filters.** The filter tree is turned into a single SQL query. It checks each item against its own tags and the tags of its group, as described in [How filters are applied](/filters/#how-filters-are-applied).
2. **Query the database.** The query runs against the SQLite database of the collection. Of the items that match, only those in the index named by `index_name`, or in the collection's [default index](/configuration/#default-index), are kept. For a CLIP index of keyframes, that means only keyframes are returned. Up to `n` of them are returned, in no particular order.
3. **Return results.** The matching IDs are returned in the `suggestions` array in the standard search response shape.

Because no embedding lookup is involved, faceted search is very fast.

## When to use faceted search

| Use case | Recommended strategy |
|----------|---------------------|
| "Show me all videos from 2023" | Faceted |
| "Show me sports clips" | Faceted (if category is a filter) or CLIP |
| "Find more images like this one" | Relevance feedback |
| "Find images of a sunset on a beach" | CLIP |
| "Find sports clips from 2023 that look like this" | CLIP or RF + filters |

## Filter requirement

Unlike the other searches, the `filters` field is **required** for faceted search. An empty or null filter would match every item and is not useful for this strategy.

See [Filters](/filters/) for the full filter expression reference and examples.

## Request reference

See [POST /exq/search/faceted](/api/search/#post-%2Fexq%2Fsearch%2Ffaceted) in the search endpoints for the full request and response schema.
