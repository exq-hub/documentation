---
title: Relevance feedback
description: Iterative search refinement using a linear SVM trained on positive and negative examples.
icon: refresh-cw
---

Relevance feedback (RF) search refines results by learning from user interactions. The user marks items as relevant (positive) or irrelevant (negative), and the engine trains a linear SVM to find more items like the positives and fewer like the negatives.

## How it works

### 1. Collect examples

The engine builds its positive and negative example sets from multiple sources:

| Situation | Behaviour |
|-----------|-----------|
| `pos` IDs provided | Use them directly |
| `query` provided | Pseudo-relevance feedback (see below) |
| No `pos` IDs and no `query` | Sample 5 random items from the collection |
| `neg` IDs provided | Use them directly |
| `neg` is empty | Sample 5 random items from the collection |

This fallback chain ensures the SVM always has training data, even at the start of a session when the user hasn't made any selections yet.

### 2. Fetch embeddings

The engine retrieves the raw embedding vectors for all positive and negative examples from the index's `embeddings_file`.

### 3. Train the SVM

A `SGDClassifier` (scikit-learn), which is a linear SVM with its default settings, is trained with:

- Positive examples labelled `+1`
- Negative examples labelled `−1`

After training, the learned weight vector (the SVM hyperplane normal) becomes the new query vector. This vector points in embedding space toward the positives and away from the negatives.

### 4. Search

The SVM weight vector is used to query the index named by `index_name`, or the collection's [default index](/configuration/#default-index) when it isn't set. It retrieves the top-`n` items closest to the hyperplane's positive side, after applying the skip set and any active filters.

## Pseudo-relevance feedback (PRF)

When `query` is set, the engine runs the query on the same index and adds the top 10 results to the positive set, even when `pos` IDs are given.

PRF takes a text search one step forward as an RF session. The top results act as if the user had marked them relevant, so the user gets RF results without selecting anything first.

## Session workflow example

```
1. User enters "news anchor at a desk"
   → CLIP search returns 20 results

2. User marks items 42 and 17 as relevant, and item 830 as not relevant
   → RF request: pos=[42, 17], neg=[830]
   → SVM trains, returns refined results

3. User marks 2 more positives, 1 more negative
   → RF request: pos=[42, 17, 56, 91], neg=[830, 204]
   → Results converge on the target visual concept
```

## Combining with filters

Like CLIP search, RF supports the `filters` field. Items that don't match the filter tree are added to the skip set before the SVM search, ensuring results are always within the filtered subset.

## Request reference

See [POST /exq/search/rf](/api/search/#post-%2Fexq%2Fsearch%2Frf) in the search endpoints for the full request and response schema.
