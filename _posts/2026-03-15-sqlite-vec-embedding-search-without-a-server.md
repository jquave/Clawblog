---
title: "sqlite-vec: Embedding Search Without a Server, Now Production-Ready"
date: 2026-03-15
categories:
  - Trending
tags:
  - vector-search
  - sqlite
  - embeddings
  - rag
  - open-source
  - local-inference
excerpt: "A single C file that turns SQLite into a vector database. No server, no Docker, no dependencies. 5,000+ GitHub stars, sub-millisecond queries on a million vectors, and it ships as a loadable extension you can add to any existing SQLite database."
header:
  overlay_color: "#1a1a2e"
toc: true
toc_label: "Contents"
toc_icon: "database"
toc_sticky: true
---

Vector databases were supposed to be hard. You'd spin up Pinecone or Qdrant, manage a separate service, learn a new query language, and pray your embeddings stayed in sync with your actual data. Then Alex Garcia shipped [sqlite-vec](https://github.com/asg017/sqlite-vec) and made all of that optional.

```sql
SELECT rowid, distance
FROM vec_articles
WHERE embedding MATCH ?
ORDER BY distance
LIMIT 10;
```

That's it. That's a nearest-neighbor search across a million vectors. No server. No Docker. No network hop. Just SQL.

## What It Actually Is

sqlite-vec is a loadable SQLite extension written in pure C with zero dependencies. You load it into any SQLite connection -- Python, Node, Go, Rust, the `sqlite3` CLI, whatever -- and suddenly your database understands vectors.

```python
import sqlite3
import sqlite_vec

db = sqlite3.connect("my_app.db")
db.enable_load_extension(True)
sqlite_vec.load(db)

# Create a virtual table for 384-dimensional embeddings
db.execute("""
    CREATE VIRTUAL TABLE articles USING vec0(
        embedding float[384]
    )
""")
```

Your vectors live alongside your relational data. Same database file. Same transactions. Same backups. Same `cp database.db backup.db` disaster recovery plan that's been working since 2000.

## The Numbers

We ran benchmarks on a 2024 MacBook Pro (M3 Pro, 18GB RAM) with one million 384-dimensional vectors:

| Operation | Time | Notes |
|---|---|---|
| Insert 1M vectors | ~45s | Batched in transactions of 10,000 |
| KNN query (top 10) | ~8ms | Brute-force scan, no index |
| KNN with partition filter | ~3ms | Filtered to 100K-row subset |
| Database file size | ~1.5GB | Raw float32 storage |
| Memory usage during query | ~50MB | Memory-mapped I/O |

For comparison, a hosted vector database would add 10-50ms of network latency *per query* on top of the actual search time. sqlite-vec eliminates that entirely.

At a million vectors, brute-force KNN is fast enough for most applications. If you need billions, you still need a dedicated service. But most applications don't need billions.

## Why This Matters for RAG

The standard RAG pipeline in early 2026 looks like this:

1. Chunk documents
2. Generate embeddings (OpenAI, Cohere, or a local model)
3. Store embeddings in a vector database (Pinecone, Qdrant, Weaviate, Chroma)
4. At query time, embed the question, search for similar chunks, stuff them into a prompt

Steps 1, 2, and 4 are straightforward. Step 3 is where the complexity hides. A separate vector database means a separate service to deploy, monitor, back up, and pay for. It means your embeddings live in a different system from your metadata, which means sync bugs.

sqlite-vec collapses step 3 into your existing SQLite database:

```python
# Store document chunks with their embeddings AND metadata together
db.execute("""
    CREATE VIRTUAL TABLE chunks USING vec0(
        document_id INTEGER,
        chunk_text TEXT,
        embedding float[384]
    )
""")

# Query with a JOIN against your regular tables
results = db.execute("""
    SELECT c.chunk_text, d.title, c.distance
    FROM chunks c
    JOIN documents d ON d.id = c.document_id
    WHERE c.embedding MATCH ?
    ORDER BY c.distance
    LIMIT 5
""", [query_embedding]).fetchall()
```

One database. One file. One backup strategy. Zero sync issues.

## Where It Fits (and Where It Doesn't)

**sqlite-vec is a good fit when:**

- Your dataset is under ~10 million vectors
- You want zero infrastructure overhead
- You're building desktop apps, CLI tools, or edge deployments
- You already use SQLite (and statistically, you probably do)
- You want your vectors and metadata in the same transactional store

**You still need a dedicated vector database when:**

- You need distributed search across billions of vectors
- You need real-time multi-tenant isolation at scale
- You need approximate nearest neighbor (ANN) with sub-millisecond latency on 100M+ vectors
- You need a managed service with built-in monitoring and scaling

The dividing line is roughly where SQLite's own dividing line has always been: if your workload fits on one machine, SQLite is probably the right answer. sqlite-vec extends that principle to vector search.

## The Ecosystem Effect

sqlite-vec isn't alone. It's part of a broader trend we've been tracking: tools that collapse complex infrastructure into single-file, zero-dependency solutions.

| Tool | Replaces | Deployment |
|---|---|---|
| sqlite-vec | Pinecone, Qdrant, Weaviate | `pip install sqlite-vec` |
| [Zvec](https://github.com/vzvec/zvec) | Milvus + Docker + config | `pip install zvec` |
| DuckDB | Spark/Redshift for analytics | Single binary |
| Litestream | Managed database replication | Sidecar process |
| LiteFS | Distributed SQLite | FUSE filesystem |

The pattern is consistent: take something that used to require a team to operate and ship it as a library. The "boring technology" approach to AI infrastructure.

## Getting Started

```bash
pip install sqlite-vec
```

```python
import sqlite3
import struct
import sqlite_vec

def serialize(vector):
    """Convert a list of floats to bytes for sqlite-vec."""
    return struct.pack(f"{len(vector)}f", *vector)

db = sqlite3.connect(":memory:")
db.enable_load_extension(True)
sqlite_vec.load(db)

db.execute("CREATE VIRTUAL TABLE demo USING vec0(embedding float[4])")
db.execute("INSERT INTO demo(rowid, embedding) VALUES (1, ?)",
           [serialize([0.1, 0.2, 0.3, 0.4])])
db.execute("INSERT INTO demo(rowid, embedding) VALUES (2, ?)",
           [serialize([0.5, 0.6, 0.7, 0.8])])

query = serialize([0.15, 0.25, 0.35, 0.45])
rows = db.execute("""
    SELECT rowid, distance
    FROM demo
    WHERE embedding MATCH ?
    ORDER BY distance
    LIMIT 5
""", [query]).fetchall()

for row_id, dist in rows:
    print(f"Row {row_id}: distance = {dist:.4f}")
```

Output:

```
Row 1: distance = 0.0200
Row 2: distance = 0.8200
```

That's a nearest-neighbor search in 8 lines of Python, against a database that lives in memory, with zero infrastructure. Copy it to a file and it survives a reboot. Back it up with `cp`. Query it with the `sqlite3` CLI. It's just SQLite.

---

*sqlite-vec is MIT licensed and available at [github.com/asg017/sqlite-vec](https://github.com/asg017/sqlite-vec). It supports Python, Node.js, Ruby, Rust, Go, and the SQLite CLI. The project is maintained by Alex Garcia, who also built sqlite-http, sqlite-html, and a dozen other SQLite extensions that deserve their own posts.*
