---
title: CLIP search
description: Text-to-image retrieval using the index's CLIP model.
icon: type
---

CLIP search converts a natural language query into a dense embedding and finds the most visually similar items in the vector index. It is the primary discovery mode for users who want to describe what they are looking for in words.

## How it works

1. **Tokenise.** The query string is tokenised using the CLIP tokenizer paired with the index's CLIP model from [open_clip](https://github.com/mlfoundations/open_clip).
2. **Encode.** The token sequence is passed through the CLIP text encoder. On CUDA the model runs in FP16 for speed. On CPU and MPS it uses FP32.
3. **Normalise.** The resulting embedding vector is L2-normalised so that dot-product similarity equals cosine similarity.
4. **Build skip set.** The union of `seen`, `excluded` together with all items in their group (e.g. the whole video), and any IDs excluded by the active `filters` is computed. These IDs are skipped during retrieval.
5. **Search index.** The normalised query vector is searched against the CLIP index named by `index_name`, or the [default CLIP index](/configuration/#default-index) when it isn't set, for the top-`n` results that are not in the skip set. See [Vector indices](/indices/) for backend details.
6. **Map IDs.** Index positions are translated back to media IDs via the `numerical_int_tags` mapping in the SQLite database.

## Model

Each CLIP index sets its model with `model_name`. The model runs on the device set by `model_device` in the `[default]` section. See [Configuration](/configuration/) for which models can be used, and [Install](/install/#model-cache-and-running-offline) for how models are loaded and cached.

## Query tips for users

- Describe the **visual content**, not concepts: *"a red car parked outside a house"* works better than *"transportation"*.
- Include **lighting and style** cues: *"low light indoor portrait"* or *"aerial drone shot of a city"*.
- Most CLIP models are trained mainly on English text, so English queries generally work best. Queries in other languages may work, depending on the model.

## Combining with filters

The `filters` field in the request applies a metadata constraint on top of the vector results. Items that don't satisfy the filter tree are added to the skip set before index search begins. See [Filters](/filters/) for the full filter expression reference.
