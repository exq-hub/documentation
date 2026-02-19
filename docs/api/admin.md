---
title: Admin & system endpoints
description: Session initialisation, filter discovery, audit logging, and health check endpoints.
icon: shield
---

## Session & collection discovery

### GET `/exq/init/{session}`

Initialises a session and returns the list of enabled collections. Clients call this on startup to discover which collections are available before issuing search requests.

**Path parameter**: `session` — the client-generated session ID.

**Response**:

```json
{
  "session": "abc123",
  "collections": [
    {
      "name": "my_collection",
      "total_items": 150000
    }
  ]
}
```

---

## Info endpoints

### POST `/exq/info/totalItems`

Returns the total number of items in a collection.

**Request body**:

```json
{
  "session_info": {
    "session": "abc123",
    "collection": "my_collection",
    "modelId": "model_1"
  }
}
```

**Response**:

```json
{ "total": 150000 }
```

---

### GET `/exq/info/filters/{session}/{collection}`

Returns all available filter definitions for a collection. Each filter corresponds to a tagset and describes how items can be constrained.

**Path parameters**:

| Parameter | Description |
|-----------|-------------|
| `session` | Session ID |
| `collection` | Collection name |

**Response**:

```json
{
  "filters": [
    {
      "id": 1,
      "name": "Category",
      "tagtype_id": 2,
      "tagtype": "categorical"
    },
    {
      "id": 3,
      "name": "Year",
      "tagtype_id": 3,
      "tagtype": "numerical_int"
    }
  ]
}
```

---

### GET `/exq/info/filters/values/{session}/{collection}/{tagtypeId}/{tagsetId}`

Returns all valid values for a specific filter. Used to populate dropdowns and multi-select lists in the UI.

**Path parameters**:

| Parameter | Description |
|-----------|-------------|
| `session` | Session ID |
| `collection` | Collection name |
| `tagtypeId` | Tag-type ID (from the filter definition) |
| `tagsetId` | Tagset ID (from the filter definition) |

**Response** (categorical example):

```json
{
  "values": [
    { "id": 10, "value": "Sports" },
    { "id": 11, "value": "News" },
    { "id": 12, "value": "Documentary" }
  ]
}
```

---

## Audit logging endpoints

These endpoints let clients record model lifecycle events and user interactions that originated on the client side.

### POST `/exq/log/addModel`

Logs that a model was added to the current session.

**Request body**:

```json
{
  "session_info": {
    "session": "abc123",
    "collection": "my_collection",
    "modelId": "model_1"
  }
}
```

---

### POST `/exq/log/removeModel`

Logs that a model was removed from the current session.

Same request body shape as `/log/addModel`.

---

### POST `/exq/log/clientEvent`

Logs a batch of client-side events (e.g. image views, UI interactions) to the server audit log.

**Request body**:

```json
{
  "events": [
    {
      "type": "view",
      "media_id": 42,
      "timestamp": 1718000000000
    }
  ],
  "session_info": {
    "session": "abc123",
    "collection": "my_collection",
    "modelId": "model_1"
  }
}
```

---

## System endpoints

### GET `/health`

Returns the health status of the service. Validates that all enabled collections have accessible databases and indices.

**Response** (healthy):

```json
{
  "status": "healthy",
  "collections": {
    "my_collection": {
      "items": 150000,
      "index": "faiss",
      "database": "ok"
    }
  }
}
```

If any collection fails validation the `status` field will be `"degraded"` or `"unhealthy"` and the affected collection entry will contain an `"error"` field.

---

### GET `/`

Returns basic service metadata: name, version, and available collections.

```json
{
  "service": "Live Services Engine",
  "version": "1.0.0",
  "collections": ["my_collection", "archive"]
}
```
