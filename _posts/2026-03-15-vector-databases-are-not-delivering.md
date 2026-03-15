---
title: "Vector Databases Aren't Delivering: Full-Text Search Was Right All Along"
date: 2026-03-15
categories:
  - Analysis
tags:
  - vector-search
  - full-text-search
  - rag
  - bm25
  - retrieval
  - benchmarks
excerpt: "Microsoft's own tests found vector search alone scored 2.79/5 on groundedness while full-text search hit 4.87/5. GitHub chose BM25 over vectors for 100 billion documents. The benchmarks are in, and they're not kind to the vector database hype cycle."
header:
  overlay_color: "#1a0a0a"
toc: true
toc_label: "Contents"
toc_icon: "search"
toc_sticky: true
---

Every RAG tutorial in 2024 started the same way: chunk your documents, generate embeddings, store them in a vector database, do a similarity search. Pinecone, Qdrant, Weaviate, Chroma -- pick your poison. The implicit promise was that semantic similarity would find what keyword search couldn't.

The benchmarks are in. That promise largely hasn't held up.

## The Microsoft Numbers

The most damning data comes from Microsoft's own Azure team. They tested vector search, full-text search, and hybrid approaches against ground-truth citations. The results were, in the researcher's words, "shocking":

| Retrieval Method | Groundedness (out of 5) | Citation Match Rate |
|---|---|---|
| Vector search alone | 2.79 | 2% |
| Full-text search alone | 4.87 | 89% |
| Hybrid + semantic ranker | 4.89 | 92% |

Read that again. Vector search alone matched the correct citation **2% of the time**. Full-text search hit **89%**. The gap isn't marginal. It's a chasm.

## GitHub Chose BM25

GitHub has 100+ billion documents to search. They evaluated vector search. They chose BM25 -- the same algorithm that's powered search engines since the 1990s. Their reasoning: computational efficiency, zero-shot capability, and reliable handling of diverse query types.

When the company that hosts most of the world's code decides that keyword search is better than embeddings for code search at scale, that's a signal worth paying attention to.

## Why Vectors Fail in Practice

In controlled demos, vector search looks magical. You search for "how to deploy a container" and it finds documents about "Docker orchestration" even though the words don't overlap. Impressive.

But real-world RAG workloads aren't controlled demos. They involve:

- **Exact identifiers**: product SKUs, error codes, API endpoint names, IP addresses, legal clause numbers. Vector embeddings are terrible at exact matching. If a user searches for `ERR_CONNECTION_REFUSED` and your retrieval returns a document about "network connectivity issues" instead of the one containing that exact error code, your RAG app just hallucinated.

- **Short, literal tokens**: config flags, version numbers, function names. Embeddings compress these into a dense space where `v2.3.1` and `v4.0.0` look similar. They aren't.

- **Technical documentation**: the February 2026 AWS study found that agentic keyword search achieved over 90% of traditional RAG performance on technical docs and financial documents -- without any vector database at all.

The failure mode is consistent: vector search returns results that *sound* plausible but are *factually wrong*, because embeddings capture vibes, not precision.

## The Keyword Search Comeback

A striking finding from early 2026: tool-augmented LLM agents using plain keyword search can match or exceed vector-based RAG systems. The approach is simple -- give an agent a search tool, let it formulate its own queries, and iterate. No embeddings. No vector database. No chunking pipeline.

This works because the LLM itself handles the "semantic understanding" part. It doesn't need embeddings to bridge the vocabulary gap -- it can just rephrase and re-search. The retrieval layer only needs to be precise, not clever.

## What Actually Works: Hybrid Search (But Mostly BM25)

The production consensus in 2026 isn't "vector search is useless." It's "vector search is a recall booster on top of a full-text search foundation." The winning architecture looks like this:

1. **BM25 full-text search** for precision (exact matches, identifiers, technical terms)
2. **Vector search** for recall (catching semantically related documents that keyword search misses)
3. **Re-ranking** with a cross-encoder to sort the merged results

Benchmarks from OpenAI and Qdrant show hybrid search pushing recall from ~0.72 (BM25 alone) to ~0.91, and precision from ~0.68 to ~0.87. But notice: BM25 alone already gets you most of the way there. The vector component is additive, not foundational.

Databricks testing found that re-ranked hybrid results reduce LLM hallucinations by 35% compared to raw embedding similarity. The re-ranker matters more than the vector search.

## The Infrastructure Question

Here's the part nobody talks about: full-text search is free infrastructure. PostgreSQL has it built in (`tsvector`, `tsquery`). SQLite has it (`FTS5`). Elasticsearch has been doing it for a decade. You don't need a new database, a new service, or a new vendor.

Vector databases, by contrast, are additional infrastructure. You need to:

- Generate and store embeddings (compute cost + storage)
- Keep embeddings in sync with source data (sync bugs)
- Manage a separate service (ops overhead)
- Pay a vendor or self-host (financial cost)

All of that for a retrieval method that, on its own, matches the right citation 2% of the time.

## Where Vectors Still Make Sense

To be fair, there are legitimate use cases:

- **Image/audio/multimodal search**: when there's no text to keyword-search, embeddings are your only option
- **Cross-language retrieval**: finding English documents from a Japanese query
- **Exploratory discovery**: "show me things similar to this" when you don't know what you're looking for
- **Semantic deduplication**: finding near-duplicate content across large corpora

But for the dominant RAG use case -- "find the right document to answer this question" -- full-text search with a good re-ranker does the job better, faster, and cheaper.

## The Takeaway

The vector database hype cycle followed a familiar pattern: a genuinely useful technology got marketed as a silver bullet, adopted uncritically, and is now facing a correction. Embeddings aren't useless. But they were never the retrieval breakthrough they were sold as.

If you're building a RAG system today, start with full-text search. Add vectors later if your recall metrics demand it. And put a re-ranker on top regardless.

The boring technology was right all along.

---

*Sources: [Microsoft Azure: Vector Search Is Not Enough](https://techcommunity.microsoft.com/blog/azuredevcommunityblog/doing-rag-vector-search-is-not-enough/4161073), [Meilisearch: Why You Shouldn't Use Vector Databases for RAG](https://www.meilisearch.com/blog/vector-dbs-rag), [Keyword Search Is All You Need](https://signals.aktagon.com/articles/2026/02/keyword-search-is-all-you-need-achieving-rag-level-performance-without-vector-databases-using-agentic-tool-use/), [Redis: Full-Text Search for RAG](https://redis.io/blog/full-text-search-for-rag-the-precision-layer/), [ZenML: BM25 vs Vector Search at GitHub Scale](https://www.zenml.io/llmops-database/bm25-vs-vector-search-for-large-scale-code-repository-search), [VentureBeat: Six Data Predictions for 2026](https://venturebeat.com/data/six-data-shifts-that-will-shape-enterprise-ai-in-2026/)*
