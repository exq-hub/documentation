---
title: Configuration
description: Complete reference for the INI configuration file that controls collections, server settings, and model behaviour.
icon: settings
---

The engine reads its configuration from `./data/config.ini` on startup. The file uses standard INI syntax and is divided into three tiers:

| Section | Purpose |
|---------|---------|
| `[DEFAULT]` | Shared defaults inherited by all sections |
| `[SERVER]` and `[LOGGING]` | Reserved system sections |
| Any other section name | Defines a collection |

## `[DEFAULT]`

```ini
[DEFAULT]
ModelDevice = auto
```

| Key | Values | Description |
|-----|--------|-------------|
| `ModelDevice` | `auto`, `cpu`, `cuda`, `mps`, `cuda:N` | Device for CLIP inference. `auto` selects CUDA → MPS → CPU in that order. |

## `[SERVER]`

```ini
[SERVER]
Host = 127.0.0.1
Port = 8000
Reload = False
```

| Key | Default | Description |
|-----|---------|-------------|
| `Host` | `127.0.0.1` | Bind address for the uvicorn server |
| `Port` | `8000` | TCP port |
| `Reload` | `False` | Enable hot-reload for development (`True` / `False`) |

## `[LOGGING]`

```ini
[LOGGING]
Level = INFO
```

| Key | Default | Description |
|-----|---------|-------------|
| `Level` | `INFO` | Python logging level (`DEBUG`, `INFO`, `WARNING`, `ERROR`) |

## Collection sections

Each named section (other than `DEFAULT`, `SERVER`, `LOGGING`) defines one collection. The section name becomes the collection identifier used in API requests.

```ini
[my_collection]
Enabled = True
IndexType = faiss
EmbeddingsFile = ./data/my_collection/embeddings.zarr.zip
CLIPIndexFile  = ./data/my_collection/index.faiss
DatabaseFile   = ./data/my_collection/collection.db
ThumbnailMediaURL = https://cdn.example.com/thumbs/
OriginalMediaURL  = https://cdn.example.com/originals/
LogDirectory   = ./logs/my_collection/
```

### Collection keys

| Key | Required | Description |
|-----|----------|-------------|
| `Enabled` | Yes | `True` to load the collection at startup; `False` to skip it |
| `IndexType` | Yes | `zarr` or `faiss` — selects the vector index backend |
| `EmbeddingsFile` | Yes | Path to the Zarr archive (`.zarr.zip`) containing item embeddings |
| `DatabaseFile` | Yes | Path to the SQLite database for this collection |
| `ThumbnailMediaURL` | Yes | Base URL prepended to thumbnail URIs returned by the API |
| `OriginalMediaURL` | Yes | Base URL prepended to full-resolution URIs |
| `CLIPIndexFile` | FAISS only | Path to the `.faiss` index file — required when `IndexType = faiss` |
| `LogDirectory` | No | Directory for audit log files. Audit logging is disabled if omitted |

## Index types

### `zarr` — brute-force search

The `EmbeddingsFile` Zarr archive is loaded into memory and searched exhaustively via dot-product similarity, parallelised across CPU threads. No separate index file is needed.

Use `zarr` for smaller collections (up to a few hundred thousand items) or when you want exact nearest-neighbour results.

```ini
[small_collection]
Enabled = True
IndexType = zarr
EmbeddingsFile = ./data/small/embeddings.zarr.zip
DatabaseFile   = ./data/small/collection.db
ThumbnailMediaURL = https://cdn.example.com/small/thumbs/
OriginalMediaURL  = https://cdn.example.com/small/originals/
```

### `faiss` — approximate nearest-neighbour search

A pre-built FAISS index (IVF or HNSW) is loaded alongside the embeddings file. Searches are approximate but orders of magnitude faster for large collections. The index supports incremental search, automatically expanding the probe range when fewer than `n` results survive the skip and filter steps.

```ini
[large_collection]
Enabled = True
IndexType = faiss
EmbeddingsFile = ./data/large/embeddings.zarr.zip
CLIPIndexFile  = ./data/large/index.faiss
DatabaseFile   = ./data/large/collection.db
ThumbnailMediaURL = https://cdn.example.com/large/thumbs/
OriginalMediaURL  = https://cdn.example.com/large/originals/
```

## Full example

```ini
[DEFAULT]
ModelDevice = auto

[SERVER]
Host = 0.0.0.0
Port = 8000
Reload = False

[LOGGING]
Level = INFO

[videos_2024]
Enabled = True
IndexType = faiss
EmbeddingsFile = ./data/videos_2024/embeddings.zarr.zip
CLIPIndexFile  = ./data/videos_2024/index.faiss
DatabaseFile   = ./data/videos_2024/metadata.db
ThumbnailMediaURL = https://cdn.example.com/videos_2024/thumbs/
OriginalMediaURL  = https://cdn.example.com/videos_2024/originals/
LogDirectory   = ./logs/videos_2024/

[photos_archive]
Enabled = True
IndexType = zarr
EmbeddingsFile = ./data/photos/embeddings.zarr.zip
DatabaseFile   = ./data/photos/metadata.db
ThumbnailMediaURL = https://cdn.example.com/photos/thumbs/
OriginalMediaURL  = https://cdn.example.com/photos/originals/
```
