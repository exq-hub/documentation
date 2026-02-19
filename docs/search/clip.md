---
title: CLIP search
description: Text-to-image retrieval using the ViT-SO400M-14-SigLIP-384 CLIP model.
icon: type
---

CLIP search converts a natural language query into a dense embedding and finds the most visually similar items in the vector index. It is the primary discovery mode for users who want to describe what they are looking for in words.

## How it works

1. **Tokenise** — The query string is tokenised using the CLIP tokenizer paired with the `ViT-SO400M-14-SigLIP-384` model from [open_clip](https://github.com/mlfoundations/open_clip).
2. **Encode** — The token sequence is passed through the CLIP text encoder. On CUDA the model runs in FP16 for speed; on CPU and MPS it uses FP32.
3. **Normalise** — The resulting embedding vector is L2-normalised so that dot-product similarity equals cosine similarity.
4. **Build skip set** — The union of `seen`, `excluded`, and any IDs excluded by the active `filters` is computed. These IDs are skipped during retrieval.
5. **Search index** — The normalised query vector is passed to `IndexRepository.search_clip()`, which queries the Zarr or FAISS index for the top-`n` results that are not in the skip set. See [Vector indices](/indices/) for backend details.
6. **Map IDs** — Index positions are translated back to media IDs via the `numerical_int_tags` mapping in the SQLite database.

## Model details

| Property | Value |
|----------|-------|
| Model family | `ViT-SO400M-14-SigLIP-384` |
| Source | `open_clip` |
| Text encoder cache | `./data/model_text.pth` |
| Image resolution | 384 × 384 (visual side only, not used here) |
| Embedding dimension | 1152 |

The model is loaded once at startup and reused for the lifetime of the process. The text encoder is serialised to disk after first load so subsequent starts skip the download.

## Device selection

The CLIP model respects the `ModelDevice` setting in `[DEFAULT]`. When set to `auto`, the engine selects:

1. CUDA (first available GPU)
2. Apple MPS
3. CPU

## Query tips for users

- Describe the **visual content**, not concepts: *"a red car parked outside a house"* works better than *"transportation"*.
- Include **lighting and style** cues: *"low light indoor portrait"* or *"aerial drone shot of a city"*.
- CLIP is language-agnostic at inference time; multilingual queries may work but English generally performs best.

## Combining with filters

The `filters` field in the request applies a metadata constraint on top of the vector results. Items that don't satisfy the filter tree are added to the skip set before index search begins. See [Filters](/filters/) for the full filter expression reference.
