---
title: Introduction
description: An overview of the Exquisitor Live Services Engine and its core capabilities.
icon: info
---

## What is the Live Services Engine?

The **Live Services Engine (LSE)** is the Python/FastAPI backend for [Exquisitor](https://exquisitor.org), a multimedia search and exploration system. It exposes a REST API for searching and exploring large image and video collections, including:

- **CLIP search** — text-to-image retrieval with a CLIP model (by default `ViT-SO400M-14-SigLIP-384`)
- **Text search** — semantic search over text embeddings, such as video transcripts, using sentence-transformers
- **Relevance feedback** — iterative refinement with a linear SVM trained on positive and negative examples
- **Temporal search** — an ordered sequence of queries matched to segments that occur one after another in the same video
- **Faceted filtering** — metadata-only queries using a composable filter expression tree

The engine serves multiple independent collections from one process. Each collection has its own SQLite metadata database and one or more vector indexes.

## Key features

- **Multiple indexes**: A collection can have several indexes. Each has its own model and index type.
- **Combinable search**: Filters work with every search. Relevance feedback supports pseudo relevance feedback from a text query. Temporal search runs a sequence of queries, which can be of different types.
- **Exact or approximate search**: Chosen per index.
- **Composable filters**: Combine filters with AND, OR, and NOT.
- **Audit logging**: Searches, item requests, and client events are logged.
- **Hardware-aware**: Uses CUDA, Apple MPS, or CPU automatically. Models are cached on disk after the first download, so the server can run offline. See [Install](/install/#model-cache-and-running-offline).

## Architecture at a glance

```
Client
  │
  ▼
FastAPI (main.py)
  │
  ├── /exq/search/   →  SearchService  →  Search strategies
  ├── /exq/item/     →  ItemService
  ├── /exq/          →  Session, collection info, logging
  └── /health        →  Health check
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
  Model managers     IndexRepository     DatabaseRepository
  (per embedding     (vector indexes)    (SQLite per collection)
   type)
```

See [Architecture](/architecture/) for details.

## What's next?

- [Install and run the server](/install/)
- [Configure collections and indexes](/configuration/)
- [Explore the API reference](/api/search/)
