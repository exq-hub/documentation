---
title: Install
description: Prerequisites, setup, and running the Live Services Engine.
icon: package
---

## Prerequisites

- **Python 3.13+**
- **[uv](https://docs.astral.sh/uv/)** — fast Python package manager
- A GPU with CUDA or Apple Silicon (MPS) is recommended for CLIP inference, but CPU works too.

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
3. Create the data directory and add your configuration file:
    ```bash
    mkdir -p data
    cp config.example.ini data/config.ini
    ```
4. Edit `data/config.ini` to point at your collection databases and indices. See the [Configuration](/configuration/) guide for all options.

## Run

Start the server directly:

```bash
uv run python main.py
```

Or via the task runner:

```bash
uv run invoke run
```

The server starts on `http://127.0.0.1:8000` by default (configurable in `[SERVER]`).

## Verify

Check that the service is healthy and all collections loaded correctly:

```bash
curl http://127.0.0.1:8000/health
```

A successful response lists each collection with its item count and index status:

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

## Development commands

| Command | Description |
|---------|-------------|
| `uv sync` | Install / update all dependencies |
| `uv run python main.py` | Start the server |
| `uv run invoke run` | Start via task runner |
| `uv run ruff check .` | Lint the codebase |
| `uv run ruff format .` | Format the codebase |

## First-time model download

On first start the engine downloads and caches the `ViT-SO400M-14-SigLIP-384` CLIP model from [open_clip](https://github.com/mlfoundations/open_clip). The text encoder is then serialised to `./data/model_text.pth` so subsequent starts are instant.

Ensure the server has internet access on first run, or pre-download the model and place it at `./data/model_text.pth`.
