---
title: Install
description: Prerequisites, setup, and running the Live Services Engine.
icon: package
---

## Prerequisites

- **Python 3.13+**
- **[uv](https://docs.astral.sh/uv/)**, a fast Python package manager
- A GPU (CUDA or Apple Silicon) is recommended. CPU also works.
- For each collection: a metadata database, embedding files, and index files. See [Database](/database/) and [Vector indices](/indices/).

## Setup

1. Clone the repository:
    ```bash
    git clone https://github.com/exq-hub/live-services-engine.git
    cd live-services-engine
    ```
2. Install dependencies:
    ```bash
    uv sync
    ```
3. Copy the configuration template:
    ```bash
    cp data/config.toml.template data/config.toml
    ```
4. Edit `data/config.toml` to point at your collection databases, embeddings, and indexes. See the [Configuration](/configuration/) guide for all options.

Start the server from the repository root. Relative paths in the config, the model caches, and the logs all depend on the working directory.

## Run

Start the server directly:

```bash
uv run python main.py
```

This uses the `[server]` and `[logging]` settings in `data/config.toml`.

Or via the task runner:

```bash
uv run invoke run --host=0.0.0.0 --port=8000

# Stop the server
uv run invoke stop
```

The task runner does **not** read `[server]`. Its defaults are `127.0.0.1`, port `8000`, and reload on. Override them with `--host`, `--port`, or `--reload`.

Interactive API docs are at `http://127.0.0.1:8000/docs`.

## Verify

```bash
curl http://127.0.0.1:8000/health
```

A healthy server responds with:

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

`collections` lists the enabled collections. `models_loaded` lists the loaded models for each `embedding_type`. See [GET /health](/api/admin/#get-%2Fhealth) for details. The check covers the first collection only. It verifies that its database and default index are loaded. If not, it returns `{"status": "unhealthy", "reason": "..."}`.

## Development commands

| Command | Description |
|---------|-------------|
| `uv sync` | Install / update dependencies |
| `uv sync --group dev` | Also install dev tools (pytest, ruff) |
| `uv run python main.py` | Start the server |
| `uv run invoke run` | Start via task runner |
| `uv run invoke stop` | Stop a running server |
| `uv run pytest` | Run the test suite |
| `uv run ruff check .` | Lint the codebase |
| `uv run ruff format .` | Format the codebase |

## Model cache and running offline

The first time a CLIP model is loaded, [open_clip](https://github.com/mlfoundations/open_clip) downloads the full model to the Hugging Face cache (`~/.cache/huggingface/hub`, or `HF_HUB_CACHE` if set). The engine then keeps only what it needs for text queries:

- the text encoder in `data/models/<model_name>_text_open_clip-<version>.pth`
- the tokenizer in `data/tokenizers/<model_name>_open_clip-<version>.pth`

Here `<version>` is the installed open_clip version.

Later startups load only the text encoder from these files, without the need of internet. The copy in the Hugging Face cache is no longer used and can be deleted.

At startup, only the default index's model loads. Other models load on first use, so query those indexes once while online. With `preload_all_indexes = true`, all models load at startup.

After upgrading `open_clip_torch`, the files are rebuilt on the next start, because their names include the open_clip version. They are rebuilt from the Hugging Face cache if the model is still there, or by downloading it again. The files of the old version are no longer used and can be deleted. After upgrading `transformers`, delete the model's files in `data/models/` and `data/tokenizers/` yourself, because their names don't change.

Text (sentence-transformers) models are not stored in `data/`. They download to the Hugging Face cache (`~/.cache/huggingface/hub`, or `HF_HUB_CACHE` if set). To stop them checking online for updates at startup, set `HF_HUB_OFFLINE=1`.
