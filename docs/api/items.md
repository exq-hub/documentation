---
title: Item endpoints
description: REST endpoints for retrieving item metadata, related items, and exclusion checks.
icon: image
---

Item endpoints are under `/exq/item/`. They take `POST` requests with a JSON body, and read from the collection's database.

Every request has a [`session_info`](/api/search/#sessioninfo) object.

| Status | When |
|--------|------|
| `200` | Success |
| `404` | The item is not found, or the database query failed. Body: `{"detail": "..."}` |
| `422` | The request body is invalid |
| `500` | Unexpected server error. Body: `{"detail": "..."}` |

---

## POST `/exq/item/info`

Returns what a client needs to show an item. Optionally includes metadata values.

### Request body

```json
{
  "mediaId": 1093987,
  "filterIds": [6, 7],
  "session_info": {
    "session": "abc123",
    "collection": "my_collection",
    "modelId": 1
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mediaId` | `integer` | Yes | Media ID of the item |
| `filterIds` | `list[int]` | No | Tagset IDs to include under `metadata`. See [filter definitions](/api/admin/#get-%2Fexq%2Finfo%2Ffilters%2F%7Bsession%7D%2F%7Bcollection%7D). |
| `session_info` | `SessionInfo` | Yes | Session context |

### Response

```json
{
  "id": 1093987,
  "name": 1093987,
  "mediaId": 1093987,
  "mediaType": 1,
  "thumbPath": "https://media.example.com/my_collection/keyframes/video_0042/keyframe_58.jpg",
  "srcPath": "https://media.example.com/my_collection/keyframes/video_0042/keyframe_58.jpg",
  "groupId": 1093929,
  "metadata": { "Start (ms)": 144177, "End (ms)": 144577 }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | `integer` | Media ID |
| `name` | `integer` | Currently the media ID |
| `mediaId` | `integer` | Media ID |
| `mediaType` | `integer` | ID of the item's source type in the database's `source_types` table (e.g. `1` Image, `2` Video) |
| `thumbPath` | `string \| null` | Full thumbnail URL, built from the collection's `thumbnail_media_url` and `thumbnail_media_params`. `null` when the item has no thumbnail, for example a whole video. |
| `srcPath` | `string` | Full media URL, built from the collection's `original_media_url` and `original_media_params` |
| `groupId` | `integer \| null` | Media ID of the item's group, e.g. its video. `null` for a group item itself. |
| `metadata` | `object` | Only when `filterIds` is given. Tagset name to value. Multiple values come as a list. Tagsets without a value for this item are left out. |

---

## POST `/exq/item/info/batch`

Same as [`/exq/item/info`](#post-%2Fexq%2Fitem%2Finfo), for many items in one call.

### Request body

```json
{
  "mediaIds": [1093987, 999999999],
  "filterIds": [],
  "session_info": {
    "session": "abc123",
    "collection": "my_collection",
    "modelId": 1
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mediaIds` | `list[int]` | Yes | Media IDs of the items. Duplicates are collapsed. |
| `filterIds` | `list[int]` | No | Tagset IDs to include under each item's `metadata` |
| `session_info` | `SessionInfo` | Yes | Session context |

### Response

```json
{
  "items": [
    { "id": 1093987, "name": 1093987, "mediaId": 1093987, "mediaType": 1, "thumbPath": "...", "srcPath": "...", "groupId": 1093929 }
  ],
  "missing": [999999999]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `items` | `list[object]` | One `/exq/item/info` result per found item, in request order |
| `missing` | `list[int]` | Requested media IDs that were not found. Unlike `/exq/item/info`, these don't cause a `404`. |

---

## POST `/exq/item/base`

**Deprecated.** Alias of [`/exq/item/info`](#post-%2Fexq%2Fitem%2Finfo), kept for existing clients. Same request and response.

---

## POST `/exq/item/details`

**Deprecated.** Use [`/exq/item/info`](#post-%2Fexq%2Fitem%2Finfo) with `filterIds` instead.

Same request as `/exq/item/info`. Returns only the `metadata` object:

```json
{ "Categories": ["documentary", "travel"] }
```

---

## POST `/exq/item/related`

Returns the items of a group, for example all keyframes of a video. Pass the group item's media ID, such as an item's `groupId`.

### Request body

```json
{
  "mediaId": 1093929,
  "sortBy": 6,
  "sortOrder": "asc",
  "session_info": {
    "session": "abc123",
    "collection": "my_collection",
    "modelId": 1
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mediaId` | `integer` | Yes | Media ID of the group item |
| `sortBy` | `integer` | No | Tagset ID to sort by, e.g. `Start (ms)`. Unsorted if omitted. |
| `sortOrder` | `"asc" \| "desc"` | No | Sort direction. Default `"asc"`. Items without a value come last. |
| `session_info` | `SessionInfo` | Yes | Session context |

If an item has several values in the `sortBy` tagset, the smallest is used for `asc` and the largest for `desc`.

### Response

```json
{ "related": [1093930, 1093931, 1093932] }
```

An item that isn't a group returns an empty list.

---

## POST `/exq/item/excluded`

Checks whether an item belongs to one of the given groups. A client can use it, for example, to hide items from videos the user has excluded.

### Request body

```json
{
  "itemId": 1093987,
  "excluded_ids": [1093929],
  "session_info": {
    "session": "abc123",
    "collection": "my_collection",
    "modelId": 1
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `itemId` | `integer` | Yes | Media ID of the item to check |
| `excluded_ids` | `list[int]` | Yes | Media IDs of the excluded group items |
| `session_info` | `SessionInfo` | Yes | Session context |

### Response

```json
{ "excludedOrNot": true }
```
