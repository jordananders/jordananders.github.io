---
layout: default
title:  "Vector Database Comparison: Pinecone vs Weaviate vs Chroma vs pgvector"
date:   2025-11-15 07:00:00
categories: AI VectorDatabase RAG
---

Choosing a vector database is one of the first decisions in any RAG or semantic search project. I've used all the major options. Here's when to use each.

## Quick Decision Guide

- **Prototyping/Learning:** Chroma
- **Already using PostgreSQL:** pgvector
- **Production at scale:** Pinecone
- **Hybrid search + open source:** Weaviate
- **Maximum performance:** Qdrant

## Detailed Comparison

### Pinecone

**Best for:** Production systems with strict SLAs

**Pros:**
- Fully managed (zero ops)
- Consistent sub-50ms latency at billion scale
- SOC 2 Type II certified
- Real-time updates with immediate consistency

**Cons:**
- Expensive at scale (~$675/month for 10M vectors)
- Vendor lock-in
- No self-hosted option

**Use when:**
- Customer-facing AI applications
- Need guaranteed uptime
- Don't want to manage infrastructure

```python
import pinecone

pinecone.init(api_key="your-key", environment="us-east1-gcp")
index = pinecone.Index("my-index")

# Upsert vectors
index.upsert(vectors=[
    ("id1", [0.1, 0.2, ...], {"metadata": "value"}),
])

# Query
results = index.query(vector=[0.1, 0.2, ...], top_k=5)
```

### Weaviate

**Best for:** Hybrid search and open source deployments

**Pros:**
- Combines vector + keyword search
- Built-in integrations (OpenAI, Cohere)
- Self-hosted or managed cloud
- GraphQL API
- Multi-modal support

**Cons:**
- More complex setup
- Eventual consistency (configurable)
- Steeper learning curve

**Use when:**
- Need hybrid (semantic + keyword) search
- Want open source with commercial support
- Building knowledge graphs

```python
import weaviate

client = weaviate.Client("http://localhost:8080")

# Create schema
client.schema.create_class({
    "class": "Document",
    "vectorizer": "text2vec-openai",
    "properties": [{"name": "content", "dataType": ["text"]}]
})

# Add data
client.data_object.create(
    data_object={"content": "Document text"},
    class_name="Document"
)

# Hybrid search
result = client.query.get("Document", ["content"]).with_hybrid(
    query="search term",
    alpha=0.5  # Balance between vector and keyword
).do()
```

### Chroma

**Best for:** Prototyping and learning

**Pros:**
- Dead simple API
- Python-native
- Zero configuration
- Great for notebooks
- In-memory or persistent

**Cons:**
- Limited scale (< 1M vectors realistic)
- Fewer features
- Manual reindexing for updates

**Use when:**
- Building prototypes
- Learning vector databases
- Small datasets (< 100k documents)

```python
import chromadb

client = chromadb.Client()
collection = client.create_collection("my_collection")

# Add documents
collection.add(
    documents=["doc1 text", "doc2 text"],
    metadatas=[{"source": "a"}, {"source": "b"}],
    ids=["id1", "id2"]
)

# Query
results = collection.query(
    query_texts=["search query"],
    n_results=5
)
```

### pgvector

**Best for:** PostgreSQL users adding vector search

**Pros:**
- Use existing Postgres infrastructure
- SQL interface
- Familiar tooling (backups, monitoring)
- Transactional consistency
- Low ops overhead if already using Postgres

**Cons:**
- Performance ceiling (~10-100M vectors)
- Limited to PostgreSQL
- Manual index tuning required

**Use when:**
- Already using PostgreSQL
- Need transactional consistency
- Want to keep stack simple
- Dataset < 100M vectors

```sql
-- Enable extension
CREATE EXTENSION vector;

-- Create table with vector column
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding vector(1536)
);

-- Create index
CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

-- Insert
INSERT INTO documents (content, embedding)
VALUES ('doc text', '[0.1, 0.2, ...]');

-- Query
SELECT * FROM documents
ORDER BY embedding <=> '[0.1, 0.2, ...]'
LIMIT 5;
```

## Performance Benchmarks (2025)

**1M vectors, 1536 dimensions:**

| Database | Insertions/sec | Queries/sec | p50 Latency |
|----------|---------------|-------------|-------------|
| Pinecone | 50,000 | 5,000 | 23ms |
| Qdrant | 45,000 | 4,500 | 28ms |
| Weaviate | 35,000 | 3,500 | 34ms |
| pgvector | 30,000 | 3,000 | 45ms |
| Chroma | 25,000 | 2,000 | 65ms |

## Cost Comparison

**10M vectors, 1M queries/month:**

| Database | Monthly Cost | Notes |
|----------|-------------|-------|
| Pinecone | ~$675 | Serverless pricing |
| Weaviate (self-hosted) | ~$200 | Infrastructure only |
| pgvector (Supabase) | ~$250 | Pro plan |
| Chroma | ~$100 | Self-hosted |
| Qdrant Cloud | ~$400 | Managed |

## Migration Path

Most successful projects follow this pattern:

1. **Prototype:** Chroma (fast setup)
2. **MVP:** pgvector (if using Postgres) or Weaviate
3. **Production:** Pinecone or Weaviate Cloud

## Feature Comparison

| Feature | Pinecone | Weaviate | Chroma | pgvector |
|---------|----------|----------|--------|----------|
| Hybrid Search | No | Yes | No | No |
| Self-hosted | No | Yes | Yes | Yes |
| Managed Cloud | Yes | Yes | Limited | Via Supabase |
| Metadata Filtering | Yes | Yes | Yes | Yes |
| Multi-tenancy | Yes | Yes | Limited | Via schemas |
| ACID Transactions | No | No | No | Yes |

## Common Mistakes

### 1. Starting with Pinecone for Prototypes

Overkill for learning. Start with Chroma.

### 2. Using pgvector at Massive Scale

Performance degrades past 100M vectors. Plan migration early.

### 3. Ignoring Hybrid Search

For documents, hybrid (vector + keyword) often beats pure vector search.

### 4. Not Testing at Scale

Performance characteristics change dramatically at scale. Test with realistic data volumes.

## My Recommendations

**Just learning?** → Chroma

**Building a prototype?** → Chroma or pgvector

**Production with < 50M vectors?** → pgvector or Weaviate

**Production at scale?** → Pinecone or Weaviate Cloud

**Need hybrid search?** → Weaviate

## Resources

- [Vector Database Comparison 2025 - DataCamp](https://www.datacamp.com/blog/the-top-5-vector-databases)
- [Pinecone vs Weaviate vs Chroma 2025](https://aloa.co/ai/comparisons/vector-database-comparison/pinecone-vs-weaviate-vs-chroma)
- [Complete Guide 2025 - System Debug](https://sysdebug.com/posts/vector-database-comparison-guide-2025/)
- [Best Vector Databases 2025 - lakeFS](https://lakefs.io/blog/12-vector-databases-2023/)

---

*Questions about vector databases? [Let me know](mailto:jordan@jordananderson.us).*
