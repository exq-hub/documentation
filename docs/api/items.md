---
title: Item endpoints
description: REST endpoints for retrieving item metadata, related items, and exclusion checks.
icon: image
---

Item endpoints are mounted under `/exq/item/` and return metadata about individual media items stored in a collection's SQLite database.

---

## POST `/exq/item/base`

Returns the basic information needed to display an item: its URI, thumbnail URL, and original media URL.

### Request body

```json
{
  "media_ids": [42, 17, 830],
  "session_info": {
    "session": "abc123",
    "collection": "my_collection",
    "modelId": "model_1"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `media_ids` | `list[int]` | Yes | Media IDs to retrieve |
| `session_info` | `SessionInfo` | Yes | Session context |

### Response

```json
{
  "items": [
    {
      "id": 42,
      "uri": "video/segment_042.mp4",
      "thumbnail_url": "https://cdn.example.com/thumbs/segment_042.jpg",
      "original_url": "https://cdn.example.com/originals/segment_042.mp4"
    }
  ]
}
```

---

## POST `/exq/item/details`

Returns extended metadata for one or more items, including all tag values relevant to the active filters. Used to populate detail panels in the UI.

### Request body

```json
{
  "media_ids": [42],
  "tagset_ids": [1, 3, 5],
  "session_info": {
    "session": "abc123",
    "collection": "my_collection",
    "modelId": "model_1"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `media_ids` | `list[int]` | Yes | Media IDs to retrieve |
| `tagset_ids` | `list[int]` | Yes | IDs of the tagsets whose values to include |
| `session_info` | `SessionInfo` | Yes | Session context |

### Response

Returns base item info plus a `tags` map keyed by tagset ID:

```json
{
  "items": [
    {
      "id": 42,
      "uri": "video/segment_042.mp4",
      "thumbnail_url": "https://cdn.example.com/thumbs/segment_042.jpg",
      "original_url": "https://cdn.example.com/originals/segment_042.mp4",
      "tags": {
        "1": ["outdoor", "daylight"],
        "3": [2024],
        "5": ["sports"]
      }
    }
  ]
}
```

---

## POST `/exq/item/related`

Returns items that share the same `group_id` as the requested item. Useful for surfacing video segments from the same shot or sequence.

### Request body

```json
{
  "media_ids": [42],
  "session_info": {
    "session": "abc123",
    "collection": "my_collection",
    "modelId": "model_1"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `media_ids` | `list[int]` | Yes | Media IDs to find related items for |
| `session_info` | `SessionInfo` | Yes | Session context |

### Response

```json
{
  "related": {
    "42": [43, 44, 45]
  }
}
```

The response maps each requested media ID to a list of related media IDs. Items without group membership return an empty list.

---

## POST `/exq/item/excluded`

Checks whether a media item belongs to a group that has been fully excluded by the user. Used by the UI to suppress items that belong to a dismissed sequence.

### Request body

```json
{
  "media_id": 42,
  "excluded_group_ids": [7, 12, 33],
  "session_info": {
    "session": "abc123",
    "collection": "my_collection",
    "modelId": "model_1"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `media_id` | `integer` | Yes | The media ID to check |
| `excluded_group_ids` | `list[int]` | Yes | Group IDs that the user has excluded |
| `session_info` | `SessionInfo` | Yes | Session context |

### Response

```json
{
  "excluded": true
}
```
