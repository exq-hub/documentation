---
title: Database
description: SQLite schema used to store item metadata, tags, and the CLIP index ID mapping.
icon: database
---

Each collection has its own SQLite database file, configured via `DatabaseFile` in `config.ini`. The schema is read-only at runtime — the engine never writes to the database.

## Tables

### `medias`

The master table. One row per media item.

| Column | Type | Description |
|--------|------|-------------|
| `id` | `INTEGER PRIMARY KEY` | Unique media ID used throughout the API |
| `uri` | `TEXT` | Relative path or identifier for the original media |
| `thumbnail_uri` | `TEXT` | Relative path for the thumbnail |
| `group_id` | `INTEGER` | Groups related items (e.g. frames from the same video shot). `NULL` if ungrouped |
| `source_type` | `TEXT` | Optional provenance label |

### `tags`

Defines individual tag instances.

| Column | Type | Description |
|--------|------|-------------|
| `id` | `INTEGER PRIMARY KEY` | Tag ID |
| `tagset_id` | `INTEGER` | Foreign key to `tagsets` |
| … | | Additional descriptor columns |

### `tagsets`

Groups tags into logical filters (e.g. "Category", "Year", "Source").

| Column | Type | Description |
|--------|------|-------------|
| `id` | `INTEGER PRIMARY KEY` | Tagset ID — used in filter expressions |
| `name` | `TEXT` | Human-readable filter name |
| `tagtype_id` | `INTEGER` | Foreign key to `tag_types` |

### `tag_types`

Describes the data type of a tagset's values.

| Column | Type | Description |
|--------|------|-------------|
| `id` | `INTEGER PRIMARY KEY` | Tag-type ID — used in filter expressions |
| `name` | `TEXT` | e.g. `categorical`, `numerical_int` |

### `taggings`

Join table linking media items to tags.

| Column | Type | Description |
|--------|------|-------------|
| `media_id` | `INTEGER` | Foreign key to `medias.id` |
| `tag_id` | `INTEGER` | Foreign key to `tags.id` |

### `categorical_tags`

Stores string values for categorical tag types.

| Column | Type | Description |
|--------|------|-------------|
| `id` | `INTEGER PRIMARY KEY` | |
| `tag_id` | `INTEGER` | Foreign key to `tags.id` |
| `value` | `TEXT` | The string value (e.g. `"Sports"`) |

### `numerical_int_tags`

Stores integer values for numerical tag types. Also used for the special **CLIP Index ID** mapping.

| Column | Type | Description |
|--------|------|-------------|
| `id` | `INTEGER PRIMARY KEY` | |
| `tag_id` | `INTEGER` | Foreign key to `tags.id` |
| `value` | `INTEGER` | The integer value |

## CLIP index ID mapping

Vector indices address items by their position in the index (0-based integer), while the rest of the system uses `medias.id`. The mapping between these two spaces is stored as a special tagset in `numerical_int_tags`.

At query time:

1. The index returns a list of **index positions**.
2. `DatabaseRepository.get_media_ids(collection, index_ids)` looks up the `numerical_int_tags` rows for the "CLIP Index ID" tagset to translate positions → media IDs.

Conversely, to fetch embeddings for specific media items (e.g. for relevance feedback), the engine resolves media IDs → index positions via the same table.

## Key database methods

| Method | Description |
|--------|-------------|
| `load_database(collection, path)` | Open the SQLite connection |
| `get_total_items(collection)` | `SELECT COUNT(*) FROM medias` |
| `get_filters(collection)` | Return all tagsets for filter discovery |
| `get_filter_values(collection, tagsetId, tagtypeId)` | Return all values for a tagset |
| `get_media_ids(collection, index_ids)` | Translate index positions to media IDs |
| `get_item_base_info(collection, media_ids)` | URI + thumbnail + original URL |
| `get_item_detailed_info(collection, media_ids, tagset_ids)` | Base info + tag values |
| `get_related_items(collection, media_id)` | Items with the same `group_id` |
| `is_item_excluded(collection, media_id, excluded_group_ids)` | Check group exclusion |
