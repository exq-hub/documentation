---
title: Admin & system endpoints
description: Session start, collection info, audit logging, and health check endpoints.
icon: shield
---

Errors return `{"detail": "..."}`, with status `404` for an unknown collection on `/exq/info/indexes`, `422` for an invalid request, and `500` otherwise.

## Session & collection discovery

### GET `/exq/init/{session}`

Logs the start of a session and returns the enabled collections. The UI, for example, calls it once when it loads, with a new random session ID.

**Path parameter**: `session`, the session ID chosen by the client.

**Response**:

```json
{
  "session": "abc123",
  "collections": ["my_collection", "photos"]
}
```

---

## Info endpoints

### POST `/exq/info/totalItems`

Returns the number of items in a collection's default index.

**Request body**:

```json
{
  "session": "abc123",
  "collection": "my_collection",
  "modelId": 1
}
```

**Response**:

```json
{ "total_items": 4143681 }
```

---

### GET `/exq/info/filters/{session}/{collection}`

Returns the collection's filters. Each filter is a tagset.

**Path parameters**:

| Parameter | Description |
|-----------|-------------|
| `session` | Session ID |
| `collection` | Collection name |

**Response**: a list.

```json
[
  { "id": 2, "name": "Categories", "tagtypeId": 1, "tagtype": "alphanumerical" },
  { "id": 6, "name": "Start (ms)", "tagtypeId": 5, "tagtype": "numerical_int" }
]
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | `integer` | Tagset ID. Used as `filter.id` in [filters](/filters/), and in `filterIds` and `sortBy` on item endpoints. |
| `name` | `string` | Tagset name |
| `tagtypeId` | `integer` | Tag type ID. Used as `filter.tagtype_id` in filters. |
| `tagtype` | `string` | Tag type name, from the database's `tag_types` table |

---

### GET `/exq/info/filters/values/{session}/{collection}/{tagtypeId}/{tagsetId}`

Returns all valid values for a specific filter. Used to populate dropdowns and multi-select lists in the UI.

**Path parameters**:

| Parameter | Description |
|-----------|-------------|
| `session` | Session ID |
| `collection` | Collection name |
| `tagtypeId` | The filter's `tagtypeId`, from the filter list above |
| `tagsetId` | The filter's `id`, from the filter list above |

**Response**: the filter's values. For example, the Categories filter above (`id` 2, `tagtypeId` 1) is requested with `GET /exq/info/filters/values/abc123/my_collection/1/2` and returns its categories (shortened here):

```json
[
  { "id": 10, "value": "documentary" },
  { "id": 65, "value": "travel" }
]
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | `integer` | The value's ID. Put it in `value_ids` to filter on this value. See [Filters](/filters/). |
| `value` | depends on the filter | The value itself, for example a category name or a number |

---

### GET `/exq/info/indexes/{session}/{collection}`

Returns the indexes configured for a collection. A client uses this to find the index names it can pass as `index_name` in search requests.

**Path parameters**:

| Parameter | Description |
|-----------|-------------|
| `session` | Session ID |
| `collection` | Collection name |

**Response**: one entry per index, as configured in `[[collections.indexes]]`:

```json
[
  { "name": "CLIP", "index_type": "faiss", "embedding_type": "CLIP", "source_type": "Image", "default": true }
]
```

| Field | Type | Description |
|-------|------|-------------|
| `name` | `string` | The index name. Pass it as `index_name` in search requests. |
| `index_type` | `string` | The index backend, such as `zarr` or `faiss` |
| `embedding_type` | `string` | The embedding family, such as `CLIP` or `Text`. It decides which search endpoint can use the index. |
| `source_type` | `string` | The media type the index covers, such as `Image` or `Text` |
| `default` | `boolean` | `true` for the index used when a request doesn't name one |

An unknown collection returns `404` with `{"detail": "Unknown collection: '...'"}`.

---

## Audit logging endpoints

These endpoints let clients record model lifecycle events and user interactions that originated on the client side.

### POST `/exq/log/addModel`

Logs that a relevance feedback model was added to the session. In the UI, each tab has its own model, so this is logged when a tab is opened.

**Request body**:

```json
{
  "session": "abc123",
  "collection": "my_collection",
  "modelId": 1
}
```

**Response**:

```json
{ "status": "Logged successfully" }
```

---

### POST `/exq/log/removeModel`

Logs that a relevance feedback model was removed from the session, for example when a tab is closed in the UI.

The request body and response are the same as for `/exq/log/addModel`.

---

### POST `/exq/log/clientEvent`

Logs a batch of client-side events, such as clicks and scrolls in the UI, to the server's audit log.

**Request body**: a list of events.

```json
[
  {
    "ts": 1761580086123,
    "action": "Click <history_model_0__scroll>",
    "element_id": "history_model_0__scroll",
    "route": "/search",
    "data": null,
    "session": "abc123"
  }
]
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ts` | `number` | Yes | When the event happened on the client, in milliseconds since the Unix epoch, as JavaScript's `Date.now()` gives it. Logged with the event. |
| `action` | `string` | Yes | What happened. Logged as `Client Event: <action>`. |
| `element_id` | `string` | No | The UI element involved |
| `route` | `string` | No | The page the event happened on |
| `data` | `string`, `number` or `boolean` | No | Extra information about the event |
| `session` | `string` | Yes | Session ID |

**Response**:

```json
{ "status": "Logged successfully" }
```

---

## System endpoints

### GET `/health`

Returns the health status of the service. It checks that the first collection's database and default index are loaded.

**Response** (healthy):

```json
{
  "status": "healthy",
  "version": "x.y.z",
  "collections": ["my_collection"],
  "models_loaded": {
    "CLIP": ["ViT-SO400M-14-SigLIP-384"],
    "Text": []
  }
}
```

`collections` lists all enabled collections. `models_loaded` lists the models that are loaded, for each `embedding_type`. A model that is only loaded when it's first used shows up after that first use.

If the check fails, the response is `{"status": "unhealthy", "reason": "..."}`, still with status `200`.

---

### GET `/`

Returns basic service information: name, version, description, and where the main endpoints are.

```json
{
  "name": "Live Services Engine",
  "version": "x.y.z",
  "description": "The Live Service Engine handles execution and logging of search requests within Exquisitor.",
  "endpoints": {
    "health": "/health",
    "search": "/exq/search/",
    "items": "/exq/item/",
    "admin": "/exq/",
    "docs": "/docs"
  }
}
```
