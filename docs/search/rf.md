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
| `pos` is empty + `query` provided | Run CLIP search, take top 10 results as pseudo-positives |
| `pos` still empty | Sample 5 random items from the collection |
| `neg` IDs provided | Use them directly |
| `neg` is empty | Sample 5 random items from the collection |

This fallback chain ensures the SVM always has training data, even at the start of a session when the user hasn't made any selections yet.

### 2. Fetch embeddings

The engine retrieves the raw embedding vectors for all positive and negative examples from the `EmbeddingsFile` Zarr archive.

### 3. Train the SVM

A `SGDClassifier` (scikit-learn) with a linear kernel is trained with:

- Positive examples labelled `+1`
- Negative examples labelled `−1`

After training, the learned weight vector (the SVM hyperplane normal) becomes the new query vector. This vector points in embedding space toward the positives and away from the negatives.

### 4. Search

The SVM weight vector is used to query the vector index — exactly as a CLIP embedding would be — retrieving the top-`n` items closest to the hyperplane's positive side, after applying the skip set and any active filters.

## Pseudo-relevance feedback (PRF)

When `query` is set but `pos` is empty, the engine automatically runs CLIP search on the query string and uses the top 10 results as pseudo-positive examples. This seeds the SVM with a meaningful starting direction without requiring the user to manually select anything.

PRF is useful for workflows where the user first describes what they want in text, then refines from there by marking additional examples good or bad.

## Session workflow example

```
1. User enters "news anchor at a desk"
   → CLIP search returns 20 results

2. User marks items 42, 17 as relevant; item 830 as not relevant
   → RF request: pos=[42, 17], neg=[830]
   → SVM trains, returns refined results

3. User marks 2 more positives, 1 more negative
   → RF request: pos=[42, 17, 56, 91], neg=[830, 204]
   → Results converge on the target visual concept
```

## Combining with filters

Like CLIP search, RF supports the `filters` field. Items that don't match the filter tree are added to the skip set before the SVM search, ensuring results are always within the filtered subset.

## Request reference

See [Search endpoints — POST /exq/search/rf](/api/search/#post-exqsearchrf) for the full request and response schema.
