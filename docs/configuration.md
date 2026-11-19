---
title: Configuration
description: Complete reference for the TOML configuration file that controls server settings, collections, and their indexes.
icon: settings
---

The engine reads its configuration from `./data/config.toml` on startup. Start from the template:

```bash
cp data/config.toml.template data/config.toml
```

The file has these sections:

| Section | Purpose |
|-------|---------|
| `[default]` | Global defaults (model device) |
| `[server]` | Bind address, port, and hot-reload |
| `[logging]` | Log level and audit log directory |
| `[[collections]]` | One section per collection |
| `[[collections.indexes]]` | One section per index, nested under its collection |

Relative paths depend on the working directory. Start the server from the repository root.

## `[default]`

```toml
[default]
model_device = "auto"
```

| Key | Default | Values | Description |
|-----|---------|--------|-------------|
| `model_device` | `"auto"` | `auto`, `cpu`, `cuda`, `cuda:N`, `mps` | Device for model inference. `auto` picks CUDA, then MPS, then CPU. |

## `[server]`

```toml
[server]
host = "127.0.0.1"
port = 8000
reload = true
```

| Key | Default | Description |
|-----|---------|-------------|
| `host` | `"127.0.0.1"` | Bind address for the uvicorn server |
| `port` | `8000` | TCP port |
| `reload` | `true` | Hot-reload on code changes (development) |

`uv run python main.py` uses these settings. `invoke run` does not. See [Install](/install/#run).

## `[logging]`

```toml
[logging]
level = "INFO"
directory = "./logs/"
```

| Key | Default | Description |
|-----|---------|-------------|
| `level` | `"INFO"` | Python log level (`DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`) |
| `directory` | `"./logs/"` | Directory for the audit log. Created if missing. See [Logging](/logging/). |

Each server start writes a new audit log file, `audit_<YYYY-MM-DD_HH-MM-SS>.log`. All collections log to this file. With `reload = true`, every reload also starts a new file.

## `[[collections]]`

Each `[[collections]]` section defines one collection. API requests refer to it by `name`, in `session_info.collection`.

```toml
[[collections]]
name = "my_collection"
enabled = true
database_file = "./data/my_collection/my_collection.db"
thumbnail_media_url = "https://media.example.com/my_collection/thumbs"
original_media_url = "https://media.example.com/my_collection/originals"
```

| Key | Required | Default | Description |
|-----|----------|---------|-------------|
| `name` | Yes | | Collection identifier |
| `enabled` | No | `false` | Collections without `enabled = true` are skipped. |
| `database_file` | Yes | | Path to the collection's database file. Must exist. See [Database](/database/). |
| `thumbnail_media_url` | Yes | | Base URL for thumbnails |
| `original_media_url` | Yes | | Base URL for original media |
| `thumbnail_media_params` | No | `{}` | Query parameters appended to every thumbnail URL |
| `original_media_params` | No | `{}` | Query parameters appended to every original media URL |
| `preload_all_indexes` | No | `false` | Load all indexes and their models at startup, not on first use |

### Media URLs

Item URLs are built as `<base_url>/<uri>`, where `<uri>` comes from the database. Leave the trailing slash off the base URL.

If the media server needs a token or other query parameters, add them like this:

```toml
thumbnail_media_params = { token = "your-token" }
original_media_params = { token = "your-token" }
```

This produces URLs like `https://media.example.com/my_collection/thumbs/<uri>?token=your-token`.

## `[[collections.indexes]]`

Each collection needs at least one index. An index is one searchable set of embeddings, for example of keyframes or of transcripts. Indexes in a collection can use different index types and embedding types.

```toml
  [[collections.indexes]]
  name = "CLIP"
  index_type = "zarr"
  embeddings_file = "./data/my_collection/embeddings.zarr.zip"
```

| Key | Required | Default | Description |
|-----|----------|---------|-------------|
| `name` | Yes | | Index name, unique within the collection. Used as `index_name` in search requests. |
| `index_type` | Yes | | Index backend. Currently `zarr` or `faiss`. See [Vector indices](/indices/). |
| `embeddings_file` | Yes | | Path to the embeddings, as a Zarr store (`.zip`, `.zipstore`, or `.zarr`). Relevance feedback reads vectors from it. Must exist. |
| `index_file` | Depends on `index_type` | | Path to the index file. Required for `faiss`. Not allowed for `zarr`, which searches `embeddings_file` directly. |
| `embedding_type` | No | `"CLIP"` | Embedding family. Currently `CLIP` or `Text`. Sets which model encodes queries. `/exq/search/clip` searches `CLIP` indexes, and `/exq/search/text` searches `Text` indexes. Relevance feedback and temporal search accept either. |
| `model_name` | No | `"ViT-SO400M-14-SigLIP-384"` | Model that produced the embeddings. See [Model names](#model-names). |
| `source_type` | No | `"Image"` | Media type of the indexed items. Currently `Image`, `Video`, `Audio`, `Text`, or `Other`. Must exist in the database's `source_types` table. |
| `tagset` | No | `"<name> Index ID"` | Database tagset that maps media IDs to index positions. See [Sharing an id space](#sharing-an-id-space). |
| `default` | See below | `false` | Marks the index used when a request doesn't name one |

### Model names

`CLIP` indexes are loaded with open_clip. A plain open_clip model name, such as the default `ViT-SO400M-14-SigLIP-384`, works when open_clip has one set of weights for that model. For a model with several, such as `ViT-B-32`, use the `hf-hub:` name of the weights you want, such as `hf-hub:timm/vit_base_patch32_clip_224.openai`. If you give the plain name of such a model, the engine stops at startup and lists the `hf-hub:` names to choose from. Any other model on the Hugging Face Hub can also be used with `model_name = "hf-hub:organization/repository"`. For a model from anywhere else, download it into a folder with an `open_clip_config.json` and a weights file, and set `model_name = "local-dir:/path/to/folder"`.

`Text` indexes are loaded with sentence-transformers. To use a model from the Hugging Face Hub, set `model_name` to its name, such as `"sentence-transformers/all-MiniLM-L6-v2"`. For a model from anywhere else, download it into a folder as saved by sentence-transformers, and set `model_name = "/path/to/folder"`.

A model already in the Hugging Face cache is not downloaded again.

`hf-hub:` and `local-dir:` names also work on Windows, because the engine replaces characters such as `:` when it names its cache files.

See [Install](/install/#model-cache-and-running-offline) for where models are downloaded and cached.

### Default index

A collection with one index uses it as the default. A collection with more than one index must mark **exactly one** with `default = true`.

When a request has no `index_name`:

- `/exq/search/rf` and temporal search hops use the default index.
- `/exq/search/clip` and `/exq/search/text` use the default index if it has the right `embedding_type`. Otherwise, they use the first index of that type.

At startup, only the default index loads. Other indexes load on first use. Set `preload_all_indexes = true` to load all of them at startup.

### Sharing an id space

Each index maps media IDs to index positions through a database tagset. By default it is named `"<name> Index ID"`.

Sometimes several indexes are built from the same embeddings, in the same order. Give them the same `tagset`. They then share one mapping, so the database needs only one copy:

```toml
  [[collections.indexes]]
  name = "CLIP_hnsw"
  index_type = "faiss"
  index_file = "./data/my_collection/clip_hnsw.faiss"
  embeddings_file = "./data/my_collection/clip_embeddings.zipstore"
  tagset = "SigLIP ID"
  default = true

  [[collections.indexes]]
  name = "CLIP_ivfpq"
  index_type = "faiss"
  index_file = "./data/my_collection/clip_ivfpq.faiss"
  embeddings_file = "./data/my_collection/clip_embeddings.zipstore"
  tagset = "SigLIP ID"
```

## Validation

The server checks the configuration at startup. It refuses to start if:

- the file is missing or isn't `.toml`
- `database_file`, `embeddings_file`, or a required `index_file` doesn't exist
- `index_file` is set on a `zarr` index, or missing on a `faiss` index
- `embedding_type`, `source_type`, or `model_device` has an unsupported value
- two indexes in a collection share a name
- a collection with several indexes doesn't mark exactly one as `default`

## Full example

```toml
[default]
model_device = "auto"

[server]
host = "0.0.0.0"
port = 8000
reload = false

[logging]
level = "INFO"
directory = "./logs/"

# Keyframe search plus transcript search over the same videos.
[[collections]]
name = "videos_2024"
enabled = true
database_file = "./data/videos_2024/videos_2024.db"
thumbnail_media_url = "https://media.example.com/videos_2024/thumbs"
original_media_url = "https://media.example.com/videos_2024/originals"
original_media_params = { token = "your-token" }

  [[collections.indexes]]
  name = "CLIP"
  index_type = "faiss"
  index_file = "./data/videos_2024/clip_hnsw.faiss"
  embeddings_file = "./data/videos_2024/clip_embeddings.zipstore"
  default = true

  [[collections.indexes]]
  name = "Transcripts"
  index_type = "zarr"
  embeddings_file = "./data/videos_2024/transcript_embeddings.zarr"
  embedding_type = "Text"
  model_name = "your-sentence-transformers-model"
  source_type = "Text"

# A small photo collection with a single brute-force index.
[[collections]]
name = "photos"
enabled = true
database_file = "./data/photos/photos.db"
thumbnail_media_url = "https://media.example.com/photos/thumbs"
original_media_url = "https://media.example.com/photos/originals"

  [[collections.indexes]]
  name = "CLIP"
  index_type = "zarr"
  embeddings_file = "./data/photos/embeddings.zarr.zip"
```
