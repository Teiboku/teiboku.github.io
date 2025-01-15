---
title: "Two Phase Neural Sparse Search On AWS OpenSearch"
date: 2025-01-15
draft: false
description: "a description"
tags: ["example", "tag"]
summary: "A speed up algorithm for AWS OpenSearch Neural Sparse Search"
---

## Some link for your reference
[RFC by me](https://github.com/opensearch-project/neural-search/issues/646)

[PR by me](https://github.com/opensearch-project/neural-search/pull/695/files)

[Blog post by me](https://opensearch.org/blog/Introducing-a-neural-sparse-two-phase-algorithm/)


## Introduction
Neural sparse search is an efficient method for semantic retrieval introduced in OpenSearch 2.11. It uses semantic techniques to interpret queries, handling terms that traditional search may miss. While dense models find similar results, they can overlook exact matches. Neural sparse search uses sparse representations to capture both semantic similarities and specific terms, improving result explanation and presentation by providing a more comprehensive retrieval solution.

## Background
Neural sparse search first expands text (either a query or a document) into a larger set of terms, each weighted by its semantic relevance. It then uses Lucene’s efficient term vector computation to identify the highest-scoring results. This approach leads to reduced index and memory costs as well as lower computational expenses. For example, while dense encoding using k-NN retrieval increases RAM costs by 7.9% at search time, neural sparse search uses a native Lucene index, avoiding any increase in RAM cost at search time. Moreover, neural sparse search leads to a much smaller index size compared to dense encoding. A document-only model generates an index that is only 10.4% the size of a dense encoding index, and for a bi-encoder, the index size is 7.2% of a dense encoding index.


Given these advantages, we’ve continued to refine neural sparse retrieval to make it even more efficient. OpenSearch 2.15 introduced a new feature: the two-phase search pipeline. This pipeline splits the neural sparse query terms into two categories: high-scoring tokens that are more relevant to the search and low-scoring tokens that are less relevant. Initially, the algorithm selects documents using the high-scoring tokens and then recalculates the score for those documents by including both high- and low-scoring tokens. This process significantly reduces computational load while maintaining the quality of the final ranking.

## The two-phase algorithm
The two-phase search algorithm operates in two stages:

### Initial Phase
The algorithm uses model inference to quickly select a set of candidate documents using high-scoring tokens from the query. These high-scoring tokens, which constitute a small portion of the total number of tokens, have significant weight—or relevance—allowing for a rapid identification of potentially relevant documents. This process significantly reduces the number of documents that need to be processed, thereby lowering computational costs.

### Recalculation Phase
The algorithm then recalculates the scores for the candidate documents selected in the first phase, this time including both high-scoring and low-scoring tokens from the query. Although low-scoring tokens carry less weight individually, they provide valuable information as part of a comprehensive evaluation, particularly when long-tail terms contribute significantly to the overall score. This allows the algorithm to determine final document scores with greater accuracy.

By processing documents in stages, this approach reduces computational overhead while mainitaining accuracy. The rapid selection in the first phase enhances efficiency, while the more detailed scoring in the second phase ensures accuracy. Even when handling a large number of long-tail terms, the results remain of high quality, with a notable improvement in computational efficiency.

## Performance metrics
Depending on the data distribution, the two-phase processor achieved an increase in speed ranging from 1.22x to 1.78x in doc only model.
![two-phase-doc-model-p99-latency](https://opensearch.org/assets/media/blog-images/2024-08-07-Introducing-a-neural-sparse-two-phase-algorithm/two-phase-doc-model-p99-latency.jpg)
And achieved an increase in speed ranging from 4.15x to 6.87x in bi-encoder model.
![two-phase-doc-model-p99-latency](https://opensearch.org/assets/media/blog-images/2024-08-07-Introducing-a-neural-sparse-two-phase-algorithm/two-phase-doc-model-p99-latency.jpg)

