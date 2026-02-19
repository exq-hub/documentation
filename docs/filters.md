---
title: Filters
description: Composable filter expression trees for constraining search results by metadata.
icon: sliders-horizontal
---

Filters allow search requests to constrain results to items that match metadata conditions. They are available on all three search strategies and are expressed as a JSON tree of logical operators and leaf conditions.

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

### `DBValueConstraint` — categorical match

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
| `operator` | `"AND"` \| `"OR"` | `OR` = item must have at least one; `AND` = item must have all |

### `DBRangeConstraint` — numerical range

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

### `DBFilter` — the condition wrapper

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

## Examples

### Single category filter

"Show items tagged as *Sports* or *News*":

```json
{
  "root": {
    "kind": "leaf",
    "filter": {
      "id": 1,
      "tagtype_id": 2,
      "constraint": {
        "value_ids": [10, 11],
        "operator": "OR"
      }
    }
  }
}
```

### Date range filter

"Show items from 2020 to 2024":

```json
{
  "root": {
    "kind": "leaf",
    "filter": {
      "id": 3,
      "tagtype_id": 3,
      "constraint": {
        "lower_bound": 2020,
        "upper_bound": 2024
      }
    }
  }
}
```

### Compound AND filter

"Show *Sports* items from 2022 to 2024":

```json
{
  "root": {
    "kind": "group",
    "operator": "AND",
    "children": [
      {
        "kind": "leaf",
        "filter": {
          "id": 1,
          "tagtype_id": 2,
          "constraint": {
            "value_ids": [10],
            "operator": "OR"
          }
        }
      },
      {
        "kind": "leaf",
        "filter": {
          "id": 3,
          "tagtype_id": 3,
          "constraint": {
            "lower_bound": 2022,
            "upper_bound": 2024
          }
        }
      }
    ]
  }
}
```

### Negated filter

"Show everything *except* Documentary":

```json
{
  "root": {
    "kind": "leaf",
    "not_": true,
    "filter": {
      "id": 1,
      "tagtype_id": 2,
      "constraint": {
        "value_ids": [12],
        "operator": "OR"
      }
    }
  }
}
```

## Discovering filter IDs

Use the admin endpoints to look up the correct IDs before building a filter:

1. **GET** `/exq/info/filters/{session}/{collection}` — lists all tagsets with their `id` and `tagtype_id`.
2. **GET** `/exq/info/filters/values/{session}/{collection}/{tagtypeId}/{tagsetId}` — lists all valid `value_ids` for a categorical tagset.

See [Admin endpoints](/api/admin/) for details.
