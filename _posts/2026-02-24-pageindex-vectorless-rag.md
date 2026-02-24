---
title: "PageIndex: The Case That You Don't Need Vectors for RAG at All"
date: 2026-02-24
categories:
  - Trending
tags:
  - rag
  - retrieval
  - llm
  - open-source
  - document-ai
excerpt: "Yesterday we covered the fastest new vector database. Today: a system that argues you don't need vectors at all. PageIndex replaces embedding similarity with LLM reasoning over a tree index, hitting 98.7% accuracy where vector RAG gets ~55%. 16,800+ GitHub stars."
header:
  overlay_color: "#1a2332"
toc: true
toc_label: "Contents"
toc_icon: "tree"
toc_sticky: true
---

Yesterday we wrote about [Zvec](/zvec-sqlite-of-vector-databases/), a new embedded vector database that's 2x faster than the previous benchmark leader. Today we're covering a project that argues the entire vector approach is wrong.

[VectifyAI/PageIndex](https://github.com/VectifyAI/PageIndex) is trending on GitHub this week with 16,800+ stars. It's a document retrieval system that uses zero embeddings, zero vector databases, and zero similarity search. Instead, it builds a hierarchical tree index from your documents and uses an LLM to *reason* about where the answer lives -- like a human expert consulting a table of contents rather than Ctrl+F-ing through every paragraph.

On the FinanceBench benchmark, it scores **98.7% accuracy** where traditional vector RAG gets **~55%**.

The trade-off is real: it's slower, more expensive per query, and doesn't scale to corpus-level search. But for the use cases where it works -- precise extraction from long, structured documents -- it's not even close.

## Why Vector RAG Fails on Hard Questions

Traditional RAG works like this: chunk your document into 512-token blocks, embed each chunk into a vector, store them in a database, and at query time, embed the question and find the most similar chunks by cosine distance.

The problem is that **semantic similarity does not equal relevance**.

Ask a vector RAG system "What were the total deferred tax assets?" and it might retrieve chunks about "tax liabilities" or "deferred revenue" -- they're semantically similar in embedding space, but not actually the answer. Worse, the answer might be split across two sections: the main discussion references the number, but the actual table lives in Appendix G. Vector search can't follow a cross-reference.

PageIndex reframes retrieval as a reasoning problem. Instead of asking "which chunks are similar to this question?", it asks "where in this document would a human expert look for this answer?"

## How It Works

### Phase 1: Build a Tree Index (Once Per Document)

PageIndex processes a document (PDF or Markdown) through iterative LLM calls to build a hierarchical JSON index. Think of it as an AI-generated table of contents with summaries:

```json
{
  "title": "Financial Stability",
  "node_id": "0006",
  "start_index": 21,
  "end_index": 22,
  "summary": "The Federal Reserve's financial stability assessment...",
  "nodes": [
    {
      "title": "Monitoring Financial Vulnerabilities",
      "node_id": "0007",
      "start_index": 22,
      "end_index": 28,
      "summary": "Monitoring framework covering asset valuations..."
    },
    {
      "title": "Domestic and International Cooperation",
      "node_id": "0008",
      "start_index": 28,
      "end_index": 31,
      "summary": "Federal Reserve collaboration with domestic..."
    }
  ]
}
```

Each node has a title, page range, AI-generated summary, and recursive children. The tree preserves the document's natural organization rather than imposing artificial chunk boundaries.

No embeddings. No vectors. Just structured JSON.

### Phase 2: Reason Over the Tree (Per Query)

At query time, the tree index (without raw text -- just titles, summaries, and page ranges) gets loaded into the LLM's context window. The LLM then runs an iterative reasoning loop:

1. **Read the tree structure.** Examine top-level sections and their summaries.
2. **Select nodes.** Reason about which sections logically contain the answer.
3. **Fetch content.** Pull the raw text for selected nodes (by node_id).
4. **Assess sufficiency.** Is this enough to answer the question?
5. **Answer** if yes, or **go back to step 1** and explore different branches if no.

The critical capability: the LLM can follow cross-references. When the main section says "see Appendix G for detailed tables," the retriever recognizes this cue, navigates to Appendix G in the tree, and pulls the actual data. Vector search would never make this connection -- the embedding of "see Appendix G" has no cosine similarity to the table content in Appendix G.

```python
# Simplified retrieval flow
tree_skeleton = remove_fields(tree, fields=['text'])  # titles + summaries only

search_prompt = f"""
You are given a question and a tree structure of a document.
Each node contains a node id, title, and summary.
Find all nodes likely to contain the answer.

Question: {query}
Document tree: {json.dumps(tree_skeleton, indent=2)}

Reply as: {{"thinking": "...", "node_list": ["node_id_1", ...]}}
"""

selected_nodes = await call_llm(search_prompt)
content = fetch_text_for_nodes(selected_nodes)
answer = await call_llm(f"Answer based on:\nQuestion: {query}\nContext: {content}")
```

## The Numbers

### FinanceBench Accuracy

PageIndex powers Mafin 2.5, which was evaluated on FinanceBench -- a benchmark for precise extraction from SEC filings and financial documents:

| System | Accuracy |
|---|:---:|
| **Mafin 2.5 (PageIndex)** | **98.7%** |
| Traditional vector RAG (optimized) | ~55% |
| Perplexity | ~45% |
| GPT-4o (general purpose) | ~31% |

That's a ~44 percentage point gap between PageIndex and traditional vector RAG on the same benchmark.

### Caveats

These numbers come with important asterisks:

- **Self-reported by VectifyAI** -- no independent replication published
- **Financial documents only** -- highly structured, with clear hierarchical organization. Generalization to messy, unstructured documents is not demonstrated
- **No comparison to modern hybrid approaches** -- vector RAG + reranking, ColBERT, or graph RAG aren't benchmarked

The 98.7% is impressive but should be treated as a domain-specific result until broader benchmarks exist.

## The Cost Trade-Off

This is the elephant in the room. PageIndex replaces cheap vector math with expensive LLM inference at retrieval time.

| Operation | Vector RAG | PageIndex |
|---|---|---|
| **Index a 100-page PDF** | ~$0.01-0.05 (embedding) | ~$1-5+ (LLM tree building) |
| **Single query retrieval** | ~$0.001 (vector lookup) | ~$0.01-0.10 (LLM reasoning) |
| **Answer generation** | ~$0.01-0.05 (LLM) | ~$0.01-0.05 (LLM) |
| **Infrastructure** | Vector DB hosting | None (stateless JSON) |

Indexing is a one-time cost per document, so it amortizes. But the per-query cost is 10-100x higher than vector retrieval. For a system handling 1,000 queries/day, that's the difference between $10/day and $100+/day.

PageIndex eliminates the need to run a vector database -- no Qdrant, no Pinecone, no ChromaDB. But it replaces that infrastructure cost with per-query LLM API costs. Whether that's a net win depends on your query volume and accuracy requirements.

## Where It Wins, Where It Doesn't

### Use PageIndex When:

- **Accuracy is non-negotiable** -- Legal, financial, medical, regulatory documents where a wrong answer has real consequences
- **Documents are structured** -- Reports, filings, manuals, textbooks with clear section hierarchies
- **Query volume is low** -- Analyst workloads, not search engines
- **Cross-references matter** -- Documents that say "see Table 3" or "per Section 4.2"
- **You want zero infrastructure** -- No vector DB, no embedding model, just an LLM API key and a JSON file

### Don't Use PageIndex When:

- **You're searching across millions of documents** -- The tree index is per-document. You can't tree-search a corpus of 10M docs.
- **Speed matters** -- Multiple LLM calls per query vs. sub-millisecond vector lookup
- **Cost matters at volume** -- Every query costs LLM inference, not just a dot product
- **Documents are unstructured** -- Chat logs, social media, loosely organized content won't produce useful trees
- **You need deterministic results** -- LLM reasoning is probabilistic. The same query might select different nodes on different runs.

### The Hybrid Architecture

The most practical setup, discussed extensively in [HN threads](https://news.ycombinator.com/item?id=43646858): **use vector search to find the right 3-5 documents from a large corpus, then use PageIndex to extract precise answers from those documents.**

This combines vector RAG's scalability (search millions of docs in milliseconds) with PageIndex's accuracy (navigate the winning documents with LLM reasoning). You get the best of both worlds -- if you're willing to pay for the LLM calls on the final extraction step.

## How It Compares

| Approach | Retrieval Method | Scalability | Accuracy (structured docs) | Cost/Query | Cross-References |
|---|---|---|:---:|---|---|
| **Vector RAG** | Cosine similarity | Millions of docs | ~55% | $0.001 | No |
| **BM25** | Keyword matching | Millions of docs | Lower | ~$0 | No |
| **ColBERT** | Late interaction | Millions of docs | Higher than vector | $0.01 | No |
| **Graph RAG** | Knowledge graph traversal | Moderate | Variable | $0.01-0.10 | Partially |
| **Long-context LLM** | Full document in context | Single docs | High | $1+ | Yes |
| **PageIndex** | LLM tree reasoning | Single/few docs | **98.7%** | $0.01-0.10 | **Yes** |

PageIndex occupies a unique position: higher accuracy than any retrieval method on structured documents, with the ability to follow cross-references that all embedding-based approaches miss. The cost is that it's fundamentally a single-document or small-collection tool.

## Getting Started

### Self-Hosted (Open Source)

```bash
git clone https://github.com/VectifyAI/PageIndex.git
cd PageIndex
pip3 install --upgrade -r requirements.txt

# Set your OpenAI API key in .env
echo "CHATGPT_API_KEY=sk-..." > .env

# Index a PDF
python3 run_pageindex.py --pdf_path report.pdf

# Index a Markdown file
python3 run_pageindex.py --md_path documentation.md
```

### Cloud API

```python
from pageindex import PageIndexClient

client = PageIndexClient(api_key="YOUR_KEY")  # from dash.pageindex.ai
doc_id = client.submit_document("report.pdf")["doc_id"]

# Wait for indexing, then retrieve the tree
tree = client.get_tree(doc_id, node_summary=True)['result']
```

An MCP server is also available at [VectifyAI/pageindex-mcp](https://github.com/VectifyAI/pageindex-mcp) for integration with Claude and other MCP-compatible tools.

MIT license. Python. 3 contributors. 16,800+ stars.

## The Bigger Picture

PageIndex and Zvec represent two opposing bets on the future of RAG:

**Zvec's bet:** Vector search is the right abstraction. The bottleneck is infrastructure complexity and performance. Build a better vector database -- faster, simpler, embeddable -- and RAG gets better.

**PageIndex's bet:** Vector search is the wrong abstraction entirely. Embeddings compress meaning into fixed-dimensional spaces, losing structural relationships, cross-references, and logical organization. Replace similarity with reasoning and accuracy jumps 40+ points.

Both are probably right -- for different use cases. The future of RAG likely isn't one or the other, but a pipeline: vector search for broad retrieval across large corpora, then reasoning-based extraction for precise answers from the winning documents.

The more interesting signal is what both projects share: **the RAG pipeline that everyone built in 2023-2024 -- chunk, embed, retrieve, generate -- is being challenged from every direction.** The "just embed everything and cosine-search" era is ending. What replaces it will be more nuanced, more expensive, and significantly more accurate.

---

*Sources: [VectifyAI/PageIndex (GitHub)](https://github.com/VectifyAI/PageIndex), [PageIndex Blog: Vectorless Reasoning-based RAG](https://pageindex.ai/blog/pageindex-intro), [VentureBeat: Tree Search Framework Hits 98.7%](https://venturebeat.com/infrastructure/this-tree-search-framework-hits-98-7-on-documents-where-vector-search-fails), [MarkTechPost: VectifyAI Launches Mafin 2.5](https://www.marktechpost.com/2026/02/22/vectifyai-launches-mafin-2-5-and-pageindex-achieving-98-7-financial-rag-accuracy-with-a-new-open-source-vectorless-tree-indexing/), [HN: RAG Without Vectors](https://news.ycombinator.com/item?id=43646858), [PageIndex Documentation](https://docs.pageindex.ai)*
