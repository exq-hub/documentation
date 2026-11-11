---
title: Text search
description: Text-to-text retrieval using the index's text-embedding model.
icon: file-text
---

Text search finds items whose text is close in meaning to a natural language query, for example transcripts. It searches `Text` indexes, while [CLIP search](/search/clip/) searches `CLIP` indexes.

## How it works

The query is encoded with the index's [sentence-transformers](https://www.sbert.net/) model, and the embedding is L2-normalised. From there, the search works like [CLIP search](/search/clip/#how-it-works). Seen and excluded items are skipped, the [filters](/filters/) apply, and the index is searched for the top-`n` results.

The search runs on the `Text` index named by `index_name`, or the [default Text index](/configuration/#default-index) when it isn't set.

## Model

Each Text index sets its model with `model_name`. See [Configuration](/configuration/) for how to use a model from the Hugging Face Hub or a local folder.

## Request reference

See [Search endpoints — POST /exq/search/text](/api/search/#post-%2Fexq%2Fsearch%2Ftext) for the full request and response schema.
