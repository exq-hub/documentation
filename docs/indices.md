---
title: Vector indices
description: How Exquisitor searches embeddings, how to choose an index type, and how to build the files.
icon: database-zap
---

An index is one searchable set of embeddings in a collection, for example of keyframes or of transcripts. Each index has its own entry under `[[collections.indexes]]` in `config.toml`, and its `index_type` decides how it is searched (see [Configuration](/configuration/#%5B%5Bcollections.indexes%5D%5D)). Indexes in one collection can use different types.

## Embeddings and positions

Every index has an embeddings file, which holds one embedding per row. A row's number is the item's position in the index, and the collection's database tells Exquisitor which media is at which position, as described in [Database](/database/#fields-exquisitor-looks-for). An index of type `faiss` also has an index file built from the same embeddings, in the same order.

Exquisitor needs the embeddings file for every index type, because some searches score items on their embeddings directly. Relevance feedback trains on the embeddings of the items a user marks, temporal search scores the segments of a few groups, and a filtered search with few matching items scores just those items.

Queries are L2-normalised, so the embeddings must be L2-normalised too. Search then ranks items by cosine similarity. If the embeddings are not normalised, items whose vectors happen to be longer rank higher than they should.

## Choosing an index type

### `zarr`: exact search

```toml
  [[collections.indexes]]
  name = "CLIP"
  index_type = "zarr"
  embeddings_file = "./data/my_collection/embeddings.zip"
```

A `zarr` index searches the embeddings file itself, so there is nothing else to build. Each search compares the query with every embedding and returns the closest items, which are always the exact nearest neighbours. The embeddings are read from disk in large blocks and scored in parallel on all CPU threads but one.

Use `zarr` for collections of up to a few hundred thousand items, or whenever you want exact results and the simplest setup.

### `faiss`: approximate search

```toml
  [[collections.indexes]]
  name = "CLIP"
  index_type = "faiss"
  index_file = "./data/my_collection/index.faiss"
  embeddings_file = "./data/my_collection/embeddings.zip"
```

A `faiss` index searches a [FAISS](https://github.com/facebookresearch/faiss) index file that you build beforehand from the embeddings. It looks at only part of the collection, so it is much faster on large collections but can miss some of the nearest neighbours. Exquisitor recognises IVF and HNSW index files and tunes how much of the index they search, as described in [How a search runs](#how-a-search-runs).

Use `faiss` for collections of millions of items, where comparing the query with every embedding is too slow.

## How a search runs

A search asks for `n` items but skips the items in `seen` and `excluded`. If the request has filters, it also skips every item that doesn't pass them.

A `zarr` index scores every item, so it simply leaves out the skipped ones and returns the best `n` of the rest.

A `faiss` index only returns the best items within the part of the index it searches. The scope is set by `nprobe` for IVF index files, which Exquisitor sets to 64 because FAISS's default of 1 is too narrow, and by `efSearch` for HNSW index files, which Exquisitor raises to at least 100. If too many of the returned items are skipped, Exquisitor asks for twice as many and searches again, and it doubles the scope whenever the index can't return as many items as asked for. This repeats until there are `n` items left or the whole index has been searched.

If a request has filters and 10,000 or fewer items of the index pass them, Exquisitor instead scores just those items on their embeddings. This is faster than skipping almost everything else, and the result is exact for every index type.

Before searching, Exquisitor checks that the query has as many dimensions as the index. If it doesn't, the error names the index's `model_name`, which is then usually not the model that produced the embeddings.

## Building the embeddings file

The embeddings file is a [Zarr](https://zarr.dev/) store holding a 2-D array of `float32` or `float16` values, with one row per item and one column per dimension. Exquisitor reads Zarr format 2 and 3, from a zip file (`.zip` or `.zipstore`) or a `.zarr` directory. The array can be the store itself or an array named `embeddings` inside it.

Put the rows in the order of the positions in the database, so that row 0 is the media whose `Index ID` is 0. To write L2-normalised embeddings from a NumPy array into a zip file:

```python
import numpy as np
import zarr
from zarr.storage import ZipStore

# One embedding per row, in the order of the positions in the database
embeddings = np.load("embeddings.npy").astype(np.float32)
embeddings /= np.linalg.norm(embeddings, axis=1, keepdims=True)

store = ZipStore("embeddings.zip", mode="w")
zarr.create_array(store, name="embeddings", data=embeddings)
store.close()
```

## Building a FAISS index

You build a FAISS index beforehand with the FAISS Python library, from the same normalised embeddings and in the same order. Use the inner product metric, which equals cosine similarity on normalised embeddings. For example, an IVF index:

```python
import faiss

d = embeddings.shape[1]  # embedding dimension

# IVF flat index
nlist = 1024  # number of Voronoi cells
quantiser = faiss.IndexFlatIP(d)
index = faiss.IndexIVFFlat(quantiser, d, nlist, faiss.METRIC_INNER_PRODUCT)
index.train(embeddings)
index.add(embeddings)

faiss.write_index(index, "index.faiss")
```

For an HNSW index, use `faiss.IndexHNSWFlat(d, 32, faiss.METRIC_INNER_PRODUCT)` instead, which needs no training. For IVF, the FAISS guidelines suggest an `nlist` of 4 to 16 times the square root of the number of items, and training needs at least 39 items per list.
