---
title: Filters
description: Composable filter expression trees for constraining search results by metadata.
icon: sliders-horizontal
---

Filters allow search requests to constrain results to items that match metadata conditions. They are available on every search endpoint, and faceted search requires them. They are expressed as a JSON tree of logical operators and leaf conditions.

## Overview

A filter is expressed as an `ActiveFilters` object with a single `root` node. The root can be either a **leaf** (a single condition) or a **group** (a logical combination of children).

```json
{
  "root": { ... }
}
```

## Node types

### `FilterLeaf`

A leaf node tests a single tagset against a constraint.

```json
{
  "kind": "leaf",
  "not_": false,
  "filter": {
    "id": 3,
    "tagtype_id": 1,
    "constraint": { ... }
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `kind` | `"leaf"` | Always `"leaf"` |
| `not_` | `boolean` | If `true`, negates the condition |
| `filter` | `DBFilter` | The condition to evaluate |

### `FilterGroup`

A group node combines child nodes with a logical operator.

```json
{
  "kind": "group",
  "operator": "AND",
  "not_": false,
  "children": [ ... ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `kind` | `"group"` | Always `"group"` |
| `operator` | `"AND"` \| `"OR"` | How children are combined |
| `not_` | `boolean` | If `true`, negates the entire group |
| `children` | `list[FilterLeaf \| FilterGroup]` | Child nodes |

## Constraints

### `DBValueConstraint` (categorical match)

Matches items that have one or more of the specified tag value IDs.

```json
{
  "value_ids": [10, 11],
  "operator": "OR"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `value_ids` | `list[int]` | Tag value IDs to match (retrieve from [filter values endpoint](/api/admin/)) |
| `operator` | `"AND"` \| `"OR"` | With `OR`, an item must have at least one of the tags. With `AND`, it must have all of them. Default `OR`. |

### `DBRangeConstraint` (numerical range)

Matches items whose tag value falls within an inclusive range.

```json
{
  "lower_bound": 2020,
  "upper_bound": 2024
}
```

| Field | Type | Description |
|-------|------|-------------|
| `lower_bound` | `int \| float \| string` | Lower bound (inclusive) |
| `upper_bound` | `int \| float \| string` | Upper bound (inclusive) |

### `DBFilter` (the condition wrapper)

```json
{
  "id": 3,
  "tagtype_id": 1,
  "constraint": { ... }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | `integer` | Tagset ID (from [filter definitions](/api/admin/)) |
| `tagtype_id` | `integer` | Tag-type ID (from [filter definitions](/api/admin/)) |
| `constraint` | `DBValueConstraint \| DBRangeConstraint` | The actual condition |

## How filters are applied

A filter is checked on each item. An item counts its own tags and the tags of its group, for example a keyframe counts the tags of its video. So a keyframe of a travel video matches a filter on the video's category, even though the keyframe itself has no category. This also lets one filter combine tags from different levels, such as the video's category and the keyframe's start time. A negated condition works the same way, so "not documentary" leaves out documentary videos and all their keyframes.

For searches that rank items by similarity, when 10,000 or fewer items pass the filters and haven't been seen or excluded, the engine scores those items exactly instead of searching the index. Otherwise, it searches the index and skips the items that don't pass. Either way, all results pass the filters.

Temporal search hops can also add their own filters. See [Temporal search](/search/temporal/#filters).

## Examples

The examples use a video collection. Each video has a Categories tagset, and each keyframe has a Start (ms) tagset with its time in the video. The IDs are only examples, so look up the IDs of your own collection as described in [Discovering filter IDs](#discovering-filter-ids).

### Single category filter

"Show *documentary* or *travel* videos and their keyframes". Categories is tagset 2, an alphanumerical tagset (tag type 1). The tag `documentary` has ID 10 and `travel` has ID 65.

```json
{
  "root": {
    "kind": "leaf",
    "filter": {
      "id": 2,
      "tagtype_id": 1,
      "constraint": {
        "value_ids": [10, 65],
        "operator": "OR"
      }
    }
  }
}
```

### Range filter

"Show keyframes that start between 5 and 60 seconds into their video". Start (ms) is tagset 6, a whole-number tagset (tag type 5).

```json
{
  "root": {
    "kind": "leaf",
    "filter": {
      "id": 6,
      "tagtype_id": 5,
      "constraint": {
        "lower_bound": 5000,
        "upper_bound": 60000
      }
    }
  }
}
```

### Compound AND filter

"Show keyframes of *travel* videos that start between 5 and 60 seconds into the video". The category is a tag of the video and the start time is a tag of the keyframe, so this combines both levels.

```json
{
  "root": {
    "kind": "group",
    "operator": "AND",
    "children": [
      {
        "kind": "leaf",
        "filter": {
          "id": 2,
          "tagtype_id": 1,
          "constraint": {
            "value_ids": [65],
            "operator": "OR"
          }
        }
      },
      {
        "kind": "leaf",
        "filter": {
          "id": 6,
          "tagtype_id": 5,
          "constraint": {
            "lower_bound": 5000,
            "upper_bound": 60000
          }
        }
      }
    ]
  }
}
```

### Negated filter

"Show *travel* videos that are not *documentary* videos, and their keyframes". A video tagged with both categories is left out, together with its keyframes.

```json
{
  "root": {
    "kind": "group",
    "operator": "AND",
    "children": [
      {
        "kind": "leaf",
        "filter": {
          "id": 2,
          "tagtype_id": 1,
          "constraint": {
            "value_ids": [65],
            "operator": "OR"
          }
        }
      },
      {
        "kind": "leaf",
        "not_": true,
        "filter": {
          "id": 2,
          "tagtype_id": 1,
          "constraint": {
            "value_ids": [10],
            "operator": "OR"
          }
        }
      }
    ]
  }
}
```

## Discovering filter IDs

Use the admin endpoints to look up the correct IDs before building a filter:

1. **GET** `/exq/info/filters/{session}/{collection}` lists all tagsets with their `id` and `tagtypeId`. These go into a filter's `id` and `tagtype_id`.
2. **GET** `/exq/info/filters/values/{session}/{collection}/{tagtypeId}/{tagsetId}` lists all valid `value_ids` for a categorical tagset.

See [Admin endpoints](/api/admin/) for details.
