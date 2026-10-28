---
title: Search endpoints
description: REST endpoints for searching a collection.
icon: search
---

All search endpoints are under `/exq/search/`. They take `POST` requests with a JSON body.

The running server also serves its live schema at `/docs` and `/openapi.json`. When in doubt, those are authoritative.

## `SessionInfo`

Every request has a `session_info` object:

```json
{
  "session": "abc123",
  "collection": "my_collection",
  "modelId": 1
}
```

| Field | Type | Description |
|-------|------|-------------|
| `session` | `string` | Session ID, chosen by the client |
| `collection` | `string` | Collection `name` from the configuration |
| `modelId` | `integer` | ID of the client's relevance feedback model. The UI has one per tab. Only recorded in the audit log. |

## Responses and errors

Search endpoints return the matching media IDs, best match first:

```json
{ "suggestions": [1093987, 3548930, 4134098] }
```

Temporal search returns chains instead of IDs. See [below](#post-%2Fexq%2Fsearch%2Ftemporal).

| Status | When |
|--------|------|
| `200` | Success. `suggestions` can be empty. |
| `400` | The search failed, for example an unknown `index_name`. Body: `{"detail": "..."}` |
| `422` | The request body is invalid |
| `500` | Unexpected server error. Body: `{"detail": "..."}` |

---

## POST `/exq/search/clip`

Searches a `CLIP` index with a text query. The index's CLIP model encodes the query.

See [CLIP search](/search/clip/) for how it works.

### Request body

```json
{
  "text": "a dog running on a beach",
  "n": 20,
  "seen": [101, 205, 340],
  "excluded": [],
  "filters": null,
  "index_name": null,
  "session_info": {
    "session": "abc123",
    "collection": "my_collection",
    "modelId": 1
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `text` | `string` | Yes | Text query |
| `n` | `integer` | Yes | Maximum number of results |
| `seen` | `list[int]` | No | Media IDs already shown. Skipped. |
| `excluded` | `list[int]` | No | Media IDs to leave out, together with all items in their group (e.g. the whole video) |
| `filters` | `ActiveFilters` | No | Filter tree. See [Filters](/filters/). |
| `index_name` | `string` | No | A `CLIP` index of the collection. Defaults as described in [Default index](/configuration/#default-index). |
| `session_info` | `SessionInfo` | Yes | Session context |

### Response

`{"suggestions": [...]}`. See [Responses and errors](#responses-and-errors).

---

## POST `/exq/search/text`

Searches a `Text` index with a text query, for example transcripts. The index's sentence-transformers model encodes the query.

See [Text search](/search/text/) for how it works.

### Request body

Same fields as [`/exq/search/clip`](#post-%2Fexq%2Fsearch%2Fclip). `index_name` must name a `Text` index.

If the collection has no `Text` index, the request fails with `400`.

### Response

`{"suggestions": [...]}`. See [Responses and errors](#responses-and-errors).

---

## POST `/exq/search/rf`

Relevance feedback search. A linear SVM is trained on positive and negative examples. Its hyperplane is then used as the query.

See [Relevance feedback](/search/rf/) for how it works.

### Request body

```json
{
  "pos": [42, 17],
  "neg": [830],
  "n": 20,
  "seen": [101, 205],
  "excluded": [],
  "query": null,
  "filters": null,
  "index_name": null,
  "session_info": {
    "session": "abc123",
    "collection": "my_collection",
    "modelId": 1
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `pos` | `list[int]` | Yes | Positive example media IDs. Can be empty. |
| `neg` | `list[int]` | Yes | Negative example media IDs. Can be empty. |
| `n` | `integer` | Yes | Maximum number of results |
| `seen` | `list[int]` | Yes | Media IDs already shown. Skipped. Can be empty. |
| `excluded` | `list[int]` | Yes | Media IDs to leave out, together with all items in their group. Can be empty. |
| `query` | `string` | No | Text query for pseudo relevance feedback. Its top 10 results are added to the positives. |
| `filters` | `ActiveFilters` | No | Filter tree. See [Filters](/filters/). |
| `index_name` | `string` | No | Any index of the collection. Defaults to the collection's default index. |
| `session_info` | `SessionInfo` | Yes | Session context |

### Response

`{"suggestions": [...]}`. See [Responses and errors](#responses-and-errors).

---

## POST `/exq/search/temporal`

Finds videos where an ordered sequence of queries (hops) matches segments one after another. Each hop can search its own index.

See [Temporal search](/search/temporal/) for how it works.

### Request body

```json
{
  "hops": [
    { "kind": "query", "text": "a dog on a beach" },
    { "kind": "rf", "pos": [42, 17], "neg": [830] },
    { "kind": "query", "text": "someone talks about the weather", "index_name": "Transcripts" }
  ],
  "n": 50,
  "seen": [],
  "excluded": [],
  "filters": null,
  "top_r": 1000,
  "rrf_k": 60,
  "chains_per_video": 1,
  "iou_threshold": 0.5,
  "session_info": {
    "session": "abc123",
    "collection": "my_collection",
    "modelId": 1
  }
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `hops` | `list[Hop]` | Yes | | The queries, in order. At least one. |
| `n` | `integer` | No | `50` | Number of videos to return |
| `seen` | `list[int]` | No | `[]` | Media IDs already shown. Not used as first-hop anchors. |
| `excluded` | `list[int]` | No | `[]` | Media IDs to leave out, together with all segments of their videos |
| `filters` | `ActiveFilters` | No | | Filter tree applied to every hop |
| `top_r` | `integer` | No | `1000` | Number of first-hop segments to start chains from |
| `rrf_k` | `number` | No | `60` | Smoothing constant of the RRF score |
| `chains_per_video` | `integer` | No | `1` | Maximum chains per video. `-1` returns all. |
| `iou_threshold` | `number` | No | `0.5` | Chains of one video that overlap more than this (IoU) are redundant. Only the better one is kept. |
| `session_info` | `SessionInfo` | Yes | | Session context |

### Hops

Each hop has a `kind`. Every hop can also set:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `index_name` | `string` | collection default | Index this hop searches |
| `weight` | `number` | `1.0` | Weight of this hop in the RRF score |
| `filters` | `ActiveFilters` | | Extra filters for this hop. Both these and the top-level filters must pass. |

`"kind": "query"` is a text query:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `text` | `string` | Yes | Text query. Encoded by the model of the hop's index. |

`"kind": "rf"` is a relevance feedback query:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `pos` | `list[int]` | No | Positive example media IDs |
| `neg` | `list[int]` | No | Negative example media IDs |
| `query` | `string` | No | Text query for pseudo relevance feedback |

### Response

A list of chains, best first:

```json
{
  "suggestions": [
    {
      "group": 3914996,
      "items": [3915023, 3915051],
      "length": 2,
      "score": 0.0267,
      "start_ms": 90215.0,
      "end_ms": 187145.0
    }
  ]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `group` | `integer` | Media ID of the video |
| `items` | `list[int]` | Media IDs of the matched segments, one per hop, in hop order |
| `length` | `integer` | Number of hops matched, counted from the first hop |
| `score` | `number` | RRF score over the matched hops |
| `start_ms` | `number` | Start time of the first segment |
| `end_ms` | `number` | End time of the last segment |

Chains are ranked by `length`, then by `score`. A chain can be shorter than `hops` when a later hop has no match after the previous segment.

Segment times come from the `Start (ms)`, `Keyframe (ms)`, or `Time (ms)` tagset, and the optional `End (ms)` tagset.

---

## POST `/exq/search/faceted`

Returns items that match a filter tree. No vector search is involved.

If a group matches, for example a video by its title, its items are returned instead of the group. Results are in no particular order.

See [Faceted search](/search/faceted/) for how it works.

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
    "modelId": 1
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `n` | `integer` | Yes | Maximum number of results |
| `filters` | `ActiveFilters` | Yes | Filter tree. See [Filters](/filters/). |
| `session_info` | `SessionInfo` | Yes | Session context |

### Response

`{"suggestions": [...]}`. See [Responses and errors](#responses-and-errors).
