---
title: Search endpoints
description: REST endpoints for CLIP text search, SVM relevance feedback, and faceted filtering.
icon: search
---

All search endpoints are mounted under `/exq/search/` and accept `POST` requests with a JSON body. Every request requires a `session_info` object that identifies the session, collection, and active model.

## `SessionInfo`

All three endpoints share this object:

```json
{
  "session": "string",
  "collection": "string",
  "modelId": "string"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `session` | `string` | Unique session identifier |
| `collection` | `string` | Collection name as defined in `config.ini` |
| `modelId` | `string` | Identifier for the model being used (logged for audit) |

---

## POST `/exq/search/clip`

Performs a text-to-image search using CLIP embeddings. The query text is tokenised and encoded by the server-side CLIP model, then used to retrieve the most similar items from the vector index.

See [CLIP search strategy](/search/clip/) for a detailed explanation of how the search works.

### Request body

```json
{
  "text": "a dog running on a beach",
  "n": 20,
  "seen": [101, 205, 340],
  "filters": null,
  "excluded": [],
  "session_info": {
    "session": "abc123",
    "collection": "my_collection",
    "modelId": "model_1"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `text` | `string` | Yes | Natural language query |
| `n` | `integer` | Yes | Number of results to return |
| `seen` | `list[int]` | Yes | Media IDs already shown to the user — excluded from results |
| `filters` | `ActiveFilters \| null` | No | Filter tree to constrain results. See [Filters](/filters/) |
| `excluded` | `list[int]` | No | Additional media IDs to blacklist |
| `session_info` | `SessionInfo` | Yes | Session context |

### Response

```json
{
  "suggestions": [42, 17, 830, 12],
  "request_timestamp": 1718000000000,
  "completion_time": 1718000000082,
  "strategy": "clip"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `suggestions` | `list[int]` | Ordered list of media IDs (most relevant first) |
| `request_timestamp` | `integer` | Unix ms timestamp when request was received |
| `completion_time` | `integer` | Unix ms timestamp when results were ready |
| `strategy` | `string` | Always `"clip"` |

---

## POST `/exq/search/rf`

Performs relevance feedback search using a linear SVM trained on user-selected positive and negative examples. Optionally, a text query can seed pseudo-positives via CLIP.

See [Relevance feedback strategy](/search/rf/) for a detailed explanation.

### Request body

```json
{
  "pos": [42, 17],
  "neg": [830],
  "n": 20,
  "seen": [101, 205],
  "query": "sunny outdoor scenes",
  "filters": null,
  "excluded": [],
  "session_info": {
    "session": "abc123",
    "collection": "my_collection",
    "modelId": "model_1"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `pos` | `list[int]` | Yes | Positive example media IDs |
| `neg` | `list[int]` | Yes | Negative example media IDs |
| `n` | `integer` | Yes | Number of results to return |
| `seen` | `list[int]` | Yes | Media IDs already shown — excluded from results |
| `query` | `string \| null` | No | Optional text query; top-10 CLIP results become pseudo-positives if `pos` is empty |
| `filters` | `ActiveFilters \| null` | No | Filter tree. See [Filters](/filters/) |
| `excluded` | `list[int]` | No | Additional IDs to blacklist |
| `session_info` | `SessionInfo` | Yes | Session context |

### Response

Same shape as CLIP search, with `"strategy": "rf"`.

---

## POST `/exq/search/faceted`

Returns items matching a filter tree without performing any vector search. Useful for browsing by metadata alone.

See [Faceted search strategy](/search/faceted/) for details.

### Request body

```json
{
  "n": 50,
  "filters": {
    "root": {
      "kind": "group",
      "operator": "AND",
      "children": [
        {
          "kind": "leaf",
          "filter": {
            "id": 3,
            "tagtype_id": 1,
            "constraint": {
              "value_ids": [10, 11],
              "operator": "OR"
            }
          }
        }
      ]
    }
  },
  "session_info": {
    "session": "abc123",
    "collection": "my_collection",
    "modelId": "model_1"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `n` | `integer` | Yes | Number of results to return |
| `filters` | `ActiveFilters` | Yes | Filter tree (required for this strategy) |
| `session_info` | `SessionInfo` | Yes | Session context |

### Response

Same shape as CLIP search, with `"strategy": "faceted"`.
