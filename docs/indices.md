---
title: Vector indices
description: FAISS and Zarr index backends — how they work, how to choose, and how incremental search operates.
icon: database-zap
---

The engine supports two vector index backends. The choice is made per collection via the `IndexType` key in `config.ini`.

## Zarr — brute-force search

```ini
IndexType = zarr
EmbeddingsFile = ./data/my_collection/embeddings.zarr.zip
```

The `EmbeddingsFile` Zarr archive is loaded entirely into memory on startup. At query time, the engine computes the dot product between the query vector and every item embedding, then returns the top-`n` by similarity.

**Implementation details:**

- Parallelised across CPU threads via `ThreadPoolExecutor`
- Always returns **exact** nearest neighbours
- `CLIPIndexFile` is not used — the Zarr archive serves as both embeddings store and search index

**Use Zarr when:**

- The collection has up to a few hundred thousand items
- Exact nearest-neighbour accuracy is required
- You want a simpler setup with no separate index file to build

## FAISS — approximate nearest-neighbour search

```ini
IndexType = faiss
EmbeddingsFile  = ./data/my_collection/embeddings.zarr.zip
CLIPIndexFile   = ./data/my_collection/index.faiss
```

A pre-built FAISS index (IVF or HNSW) is loaded from `CLIPIndexFile`. The Zarr embeddings file is still required — it is used to fetch raw vectors for relevance feedback (the SVM needs the actual embeddings, not the ANN index).

**Implementation details:**

- Supported index types: IVF (inverted file with flat quantiser) and HNSW
- IVF has a tunable `nprobe` parameter; HNSW has `efSearch`
- Returns **approximate** nearest neighbours

**Use FAISS when:**

- The collection has millions of items and brute-force is too slow
- You are willing to trade a small accuracy drop for much lower query latency

## Incremental search

Both backends implement an **incremental search** interface designed to handle large skip sets efficiently.

The problem: after removing `seen` and `excluded` items from the index results, fewer than `n` usable items may remain. Incremental search solves this by requesting progressively more candidates until the skip-filtered result set reaches `n`.

For FAISS:

1. Issue an initial search for `n × k₀` candidates (small multiplier).
2. Apply the skip set; if the result count < `n`, increase `nprobe` / `efSearch` and search again.
3. Repeat until `n` results are found or the index is exhausted.

For Zarr:

- The full brute-force scan inherently covers all items, so the skip set is applied post-scan and the top-`n` non-skipped items are returned directly.

## Building a FAISS index

Index building is done offline (not by the engine at runtime). A typical workflow using the FAISS Python library:

```python
import faiss
import numpy as np

# embeddings: np.ndarray of shape (N, D), float32, L2-normalised
d = embeddings.shape[1]  # embedding dimension

# IVF flat index
nlist = 1024  # number of Voronoi cells
quantiser = faiss.IndexFlatIP(d)
index = faiss.IndexIVFFlat(quantiser, d, nlist, faiss.METRIC_INNER_PRODUCT)
index.train(embeddings)
index.add(embeddings)

faiss.write_index(index, "index.faiss")
```

Ensure embeddings are **L2-normalised** before adding them to a FAISS inner-product index so that dot-product similarity equals cosine similarity — matching how the CLIP query vectors are normalised at inference time.

## Zarr archive format

The embeddings Zarr archive is expected to be a single Zarr v2 zip store containing a 2-D array of shape `(N, D)` in `float32` or `float16`.

```python
import zarr
import numpy as np

embeddings = np.load("embeddings.npy").astype(np.float32)
zarr.save("embeddings.zarr.zip", embeddings)
```
