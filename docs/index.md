---
title: Introduction
description: An overview of the Exquisitor Live Services Engine and its core capabilities.
icon: info
---

## What is the Live Services Engine?

The **Live Services Engine (LSE)** is the Python/FastAPI backend for [Exquisitor](https://github.com/exq-hub/exquisitor), a multimedia search and exploration system. It exposes a REST API that powers three distinct search strategies over large image collections:

- **CLIP search** — text-to-image retrieval using the `ViT-SO400M-14-SigLIP-384` CLIP model
- **Relevance feedback** — iterative search refinement using a linear SVM trained on user-selected positive and negative examples
- **Faceted filtering** — metadata-only queries using a composable filter expression tree

The engine is designed to serve multiple independent collections simultaneously, each with its own vector index and SQLite metadata database.

## Key features

- **Three search modes**: CLIP text search, SVM relevance feedback, and filter-only faceted search work independently or in combination.
- **Multi-collection support**: Each collection has its own configuration, index, and database. All are loaded at startup and served from a single process.
- **Flexible vector indices**: Supports both brute-force [Zarr](https://zarr.dev) arrays and approximate nearest-neighbour [FAISS](https://faiss.ai) indices (IVF or HNSW).
- **Composable filters**: A tree-based filter expression system allows arbitrary AND/OR/NOT combinations of categorical and numerical constraints.
- **Audit logging**: All search requests, item views, and client events are written to a structured MessagePack log.
- **Hardware-aware**: Automatically selects CUDA, Apple MPS, or CPU for model inference. The CLIP text model is cached to disk after first download.

## Architecture at a glance

```
Client
  │
  ▼
FastAPI (main.py)
  │
  ├── /exq/search/   →  SearchService  →  Strategy (CLIP / RF / Faceted)
  ├── /exq/item/     →  ItemService
  ├── /exq/          →  Admin / Init / Filters
  └── /health        →  Health check
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
      IndexRepository             DatabaseRepository
      (FAISS / Zarr)              (SQLite per collection)
```

## What's next?

- [Install and run the server](/install/)
- [Configure collections](/configuration/)
- [Explore the API reference](/api/search/)
