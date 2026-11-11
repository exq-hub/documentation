---
title: Temporal search
description: Finding videos where an ordered sequence of queries matches segments one after another.
icon: clock
---

Temporal search finds videos where a sequence of queries, called hops, matches segments one after another. For example, "a dog on a beach", followed by someone talking about the weather. Each hop can search its own index, so one hop can search keyframes through a CLIP index and the next can search transcripts through a Text index.

The method is described in [Exquisitor at the Video Browser Showdown 2026: Temporal Queries Revisited](https://doi.org/10.1007/978-981-95-6963-2_27).

## How it works

1. **Anchors** — The first hop retrieves `top_r` segments. These are the anchors that chains start from. Items in `seen` are skipped. Items in `excluded` are left out, together with all segments of their video.
2. **Pool** — For each later hop, the candidates are the segments of the anchors' videos that start after that video's earliest anchor. Only segments of the `source_type` of the hop's index count, so a hop on a Text index of transcripts matches transcript segments. Segments without a start time are left out.
3. **Scoring** — Each later hop scores its pool exactly and ranks it, best first.
4. **Chains** — From each anchor, each later hop picks its best ranked segment of the same video that starts after the previously picked one. A chain ends at the first hop without such a segment, so it can be shorter than the sequence.
5. **Ranking** — Chains rank by length, then by their RRF score, then by their anchor's rank. The RRF score is the sum of `weight / (rrf_k + rank)` over the chain's hops.
6. **Selection** — Within a video, a chain whose time span overlaps a better ranked chain by more than `iou_threshold` (IoU) is dropped. Up to `chains_per_video` chains are kept for each of the `n` best videos.

## Filters

Each hop can set its own `filters`. They are combined with the request's top-level `filters`, and a segment must pass both. See [Filters](/filters/).

## RF hops

An RF hop trains its SVM the same way as [relevance feedback](/search/rf/) search. If the hop has no `pos` IDs and its `query` finds nothing that passes the filters, the request returns a `400` error instead of no results. See [Responses and errors](/api/search/#responses-and-errors).

## Request reference

See [Search endpoints — POST /exq/search/temporal](/api/search/#post-%2Fexq%2Fsearch%2Ftemporal) for the request fields, the hop fields, the response, and where segment times come from.
