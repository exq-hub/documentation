---
title: Architecture
description: Component overview, dependency injection, and request lifecycle for the Live Services Engine.
icon: layers
---

## Component map

```
┌──────────────────────────────────────────────────────┐
│                    FastAPI (main.py)                  │
│                                                       │
│  /exq/search/  ──► SearchService ──► Strategy        │
│  /exq/item/    ──► ItemService                       │
│  /exq/         ──► Admin handlers                    │
│  /health       ──► Health check                      │
└───────────────────────┬──────────────────────────────┘
                        │ ApplicationContainer (DI)
          ┌─────────────┼─────────────────┐
          ▼             ▼                 ▼
   ModelManager   IndexRepository   DatabaseRepository
   (CLIP model)   (FAISS / Zarr)    (SQLite per collection)
```

## Layers

### API layer (`app/api/`)

FastAPI route handlers. Each router module groups related endpoints:

| Module | Prefix | Responsibility |
|--------|--------|----------------|
| `routes/search.py` | `/exq/search/` | Dispatch to search strategies |
| `routes/items.py` | `/exq/item/` | Item metadata retrieval |
| `routes/admin.py` | `/exq/` | Session init, filter discovery, audit logging |

Route handlers are thin: they validate the incoming request (via Pydantic schemas), call the appropriate service, schedule audit log entries as background tasks, and return the response.

### Service layer (`app/services/`)

Business logic lives here, decoupled from HTTP concerns.

| Service | Responsibility |
|---------|----------------|
| `SearchService` | Strategy selection, wall-clock timing, result assembly |
| `ItemService` | Item metadata queries |
| `LoggingService` | Async audit log writes via FastAPI `BackgroundTasks` |

### Strategy layer (`app/strategies/`)

Each search mode is implemented as an independent strategy class that inherits from `BaseSearchStrategy`.

| Strategy | Class | Description |
|----------|-------|-------------|
| CLIP | `CLIPSearchStrategy` | Text encoding + index search |
| RF | `RFSearchStrategy` | SVM training + index search |
| Faceted | `FacetedSearchStrategy` | SQL filter compilation + database query |

`SearchService` owns one instance of each strategy and routes requests by name.

### Repository layer (`app/repositories/`)

Stateful wrappers around external data stores. Initialised once and held for the process lifetime.

| Repository | Responsibility |
|-----------|----------------|
| `IndexRepository` | Load and query FAISS / Zarr vector indices |
| `DatabaseRepository` | Open and query SQLite databases |

### Core layer (`app/core/`)

Infrastructure and cross-cutting concerns.

| Module | Responsibility |
|--------|----------------|
| `config.py` | INI file loading and Pydantic validation |
| `models.py` | `ModelManager` (CLIP) and `ApplicationContainer` (DI) |
| `indexes.py` | Abstract index interfaces (`BaseIndex`, `FaissIndex`, `ZarrIndex`) |
| `exceptions.py` | Custom exception hierarchy |

## Dependency injection

`ApplicationContainer` acts as a singleton DI container. It is created once during the FastAPI lifespan and injected into routes via `app/api/dependencies.py`.

```python
# Simplified lifespan (main.py)
@asynccontextmanager
async def lifespan(app: FastAPI):
    container = ApplicationContainer()
    container.initialize()   # loads config, model, DBs, indices
    app.state.container = container
    yield
    container.shutdown()
```

Route handlers receive the container (or individual services) via `Depends()`:

```python
@router.post("/clip")
async def clip_search(
    request: TextSearchRequest,
    service: SearchService = Depends(get_search_service),
):
    return await service.search("clip", request)
```

## Request lifecycle

For a CLIP search request the flow is:

```
POST /exq/search/clip
  │
  ├─ Pydantic validates TextSearchRequest
  │
  ├─ SearchService.search("clip", request)
  │     ├─ Record request_timestamp
  │     ├─ CLIPSearchStrategy.execute(request)
  │     │     ├─ Tokenise + encode text → embedding
  │     │     ├─ Build skip set (seen ∪ excluded ∪ filtered)
  │     │     ├─ IndexRepository.search_clip(embedding, n, skip)
  │     │     └─ DatabaseRepository.get_media_ids(index_ids)
  │     └─ Record completion_time
  │
  ├─ BackgroundTask: LoggingService.log_search_request(...)
  │
  └─ Return SearchResponse
```

## Exception handling

All custom exceptions inherit from `LSEException`. A global FastAPI exception handler catches them, returns HTTP 400 with a structured JSON body, and logs the event to the audit log:

```json
{
  "error": "CollectionNotFound",
  "message": "Collection 'unknown' is not enabled"
}
```

See `app/core/exceptions.py` for the full hierarchy.

## Startup sequence

1. Parse `./data/config.ini`
2. Validate all collection configs (Pydantic)
3. Select compute device (CUDA → MPS → CPU)
4. Load CLIP model (from disk cache or download)
5. For each enabled collection:
   - Open SQLite database
   - Load Zarr embeddings or FAISS index
6. Initialise `LoggingService` (if `LogDirectory` is set)
7. Server begins accepting requests

## Shutdown sequence

1. Signal `LoggingService` to flush and stop
2. Close all SQLite connections
3. Release GPU memory
