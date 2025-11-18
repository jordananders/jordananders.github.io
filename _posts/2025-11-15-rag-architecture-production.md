---
layout: default
title:  "RAG Architecture: From Concept to Production"
date:   2025-11-15 06:00:00
categories: AI LLM RAG
---

RAG (Retrieval-Augmented Generation) has evolved from a clever hack into a foundational pattern for building AI systems. In 2025, it's the strategic imperative for any AI application that needs accurate, current, and auditable responses.

Here's how to implement RAG properly.

## Why RAG?

LLMs have two fundamental problems:
1. **Hallucination** - They make things up confidently
2. **Stale knowledge** - Training data has a cutoff date

RAG solves both by grounding responses in your actual data.

## Core Architecture

```
User Query → Embed Query → Vector Search → Retrieve Chunks → Augment Prompt → LLM → Response
```

### Step 1: Ingest Documents

```python
from langchain.document_loaders import DirectoryLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter

# Load documents
loader = DirectoryLoader('./docs', glob="**/*.pdf")
documents = loader.load()

# Split into chunks
splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=["\n\n", "\n", ". ", " ", ""]
)
chunks = splitter.split_documents(documents)
```

### Step 2: Create Embeddings

```python
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import Chroma

# Create embeddings
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# Store in vector database
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings,
    persist_directory="./chroma_db"
)
```

### Step 3: Retrieve Relevant Context

```python
# Query the vector store
query = "What is our refund policy?"
results = vectorstore.similarity_search(query, k=5)

# results contains the most relevant chunks
for doc in results:
    print(doc.page_content)
    print(doc.metadata)
```

### Step 4: Generate Response

```python
from langchain.chat_models import ChatOpenAI
from langchain.prompts import ChatPromptTemplate

# Build prompt with context
template = """Answer the question based on the following context.
If you cannot answer from the context, say "I don't have that information."

Context:
{context}

Question: {question}

Answer:"""

prompt = ChatPromptTemplate.from_template(template)

# Generate response
llm = ChatOpenAI(model="gpt-4", temperature=0)
context = "\n\n".join([doc.page_content for doc in results])
response = llm.invoke(prompt.format(context=context, question=query))
```

## Chunking Strategies

Chunking is critical. Bad chunking = bad retrieval.

### Fixed-Size Chunks

```python
# Simple but can split mid-sentence
splitter = CharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
```

### Recursive Splitting (Recommended)

```python
# Tries to split at natural boundaries
splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=["\n\n", "\n", ". ", " ", ""]
)
```

### Semantic Chunking

```python
# Split by semantic meaning (more expensive)
from langchain_experimental.text_splitter import SemanticChunker

splitter = SemanticChunker(embeddings, breakpoint_threshold_type="percentile")
```

### Document-Aware Chunking

For structured documents (Markdown, HTML):

```python
from langchain.text_splitter import MarkdownHeaderTextSplitter

headers_to_split_on = [
    ("#", "Header 1"),
    ("##", "Header 2"),
    ("###", "Header 3"),
]
splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers_to_split_on)
```

## Retrieval Optimization

### Hybrid Search

Combine semantic search with keyword search:

```python
from langchain.retrievers import BM25Retriever, EnsembleRetriever

# Keyword search
bm25_retriever = BM25Retriever.from_documents(chunks)
bm25_retriever.k = 5

# Semantic search
semantic_retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

# Combine
ensemble_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, semantic_retriever],
    weights=[0.3, 0.7]  # Weight semantic higher
)
```

### Query Rewriting

Improve retrieval by rewriting queries:

```python
def rewrite_query(query: str) -> str:
    prompt = f"""Rewrite this query to be more specific for document search.
    Original: {query}
    Rewritten:"""
    return llm.invoke(prompt)

# "refund policy" → "company refund and return policy terms conditions"
```

### Multi-Query Retrieval

Generate multiple perspectives:

```python
from langchain.retrievers.multi_query import MultiQueryRetriever

retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(),
    llm=llm
)
# Generates multiple queries, retrieves for each, deduplicates
```

### Re-ranking

Re-rank results for relevance:

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')

def rerank(query, documents, top_k=3):
    pairs = [[query, doc.page_content] for doc in documents]
    scores = reranker.predict(pairs)

    ranked = sorted(zip(documents, scores), key=lambda x: x[1], reverse=True)
    return [doc for doc, score in ranked[:top_k]]
```

## Advanced RAG Patterns

### Self-RAG

The model decides when to retrieve:

```python
def self_rag(query):
    # First, ask if retrieval is needed
    needs_retrieval = llm.invoke(
        f"Does this question require external knowledge? {query}\nAnswer yes/no:"
    )

    if "yes" in needs_retrieval.lower():
        context = retrieve(query)
        return generate_with_context(query, context)
    else:
        return llm.invoke(query)
```

### Corrective RAG

Validate and correct retrieved documents:

```python
def corrective_rag(query):
    docs = retrieve(query)

    # Check relevance of each doc
    relevant_docs = []
    for doc in docs:
        relevance = llm.invoke(
            f"Is this document relevant to '{query}'?\nDoc: {doc.page_content}\nAnswer yes/no:"
        )
        if "yes" in relevance.lower():
            relevant_docs.append(doc)

    # If not enough relevant docs, search web
    if len(relevant_docs) < 2:
        web_results = web_search(query)
        relevant_docs.extend(web_results)

    return generate_with_context(query, relevant_docs)
```

### Agentic RAG

Combine RAG with autonomous agents:

```python
from langchain.agents import create_react_agent

tools = [
    search_knowledge_base,
    search_web,
    calculate,
    lookup_database
]

agent = create_react_agent(llm, tools, prompt)

# Agent decides which tools to use based on query
response = agent.invoke({"input": "What were our Q3 sales compared to Q2?"})
```

## Production Considerations

### Metadata Filtering

```python
# Store metadata with chunks
chunk.metadata = {
    "source": "policy_handbook.pdf",
    "page": 15,
    "department": "HR",
    "updated": "2025-01-15"
}

# Filter during retrieval
results = vectorstore.similarity_search(
    query,
    k=5,
    filter={"department": "HR"}
)
```

### Source Attribution

Always show sources:

```python
def generate_with_sources(query, docs):
    context = "\n\n".join([
        f"[Source: {doc.metadata['source']}, Page {doc.metadata['page']}]\n{doc.page_content}"
        for doc in docs
    ])

    response = llm.invoke(prompt.format(context=context, question=query))

    sources = [{"source": doc.metadata['source'], "page": doc.metadata['page']}
               for doc in docs]

    return {"answer": response, "sources": sources}
```

### Evaluation

```python
# Test retrieval quality
def evaluate_retrieval(test_queries, ground_truth):
    scores = []
    for query, expected_docs in zip(test_queries, ground_truth):
        retrieved = retriever.get_relevant_documents(query)
        retrieved_ids = [doc.metadata['id'] for doc in retrieved]

        # Calculate recall
        hits = len(set(retrieved_ids) & set(expected_docs))
        recall = hits / len(expected_docs)
        scores.append(recall)

    return sum(scores) / len(scores)
```

### Caching

```python
import hashlib
from functools import lru_cache

@lru_cache(maxsize=1000)
def cached_retrieve(query_hash):
    return vectorstore.similarity_search(query, k=5)

def retrieve(query):
    query_hash = hashlib.md5(query.encode()).hexdigest()
    return cached_retrieve(query_hash)
```

## Vector Database Options

| Database | Best For | Notes |
|----------|----------|-------|
| Chroma | Prototyping | Simple, local |
| Pinecone | Production | Managed, scalable |
| Weaviate | Hybrid search | Built-in BM25 |
| pgvector | Existing Postgres | Add to existing DB |
| Qdrant | Performance | Fast, Rust-based |

## Common Mistakes

### 1. Chunks Too Large or Small

**Too large:** Context gets diluted, irrelevant info included
**Too small:** Context fragmented, loses meaning

**Solution:** Test with 500-1500 tokens, adjust based on your content.

### 2. No Overlap Between Chunks

```python
# BAD
splitter = CharacterTextSplitter(chunk_size=1000, chunk_overlap=0)

# GOOD
splitter = CharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
```

### 3. Ignoring Metadata

Store and filter by metadata—department, date, document type.

### 4. Not Testing Retrieval Quality

Build evaluation sets. Measure recall@k.

## Resources

- [RAG Architecture Explained 2025 - Orq.ai](https://orq.ai/blog/rag-architecture)
- [2025 Guide to RAG - EdenAI](https://www.edenai.co/post/the-2025-guide-to-retrieval-augmented-generation-rag)
- [RAG: The Definitive Guide 2025](https://www.chitika.com/retrieval-augmented-generation-rag-the-definitive-guide-2025/)
- [RAG in 2025 - Glean](https://www.glean.com/blog/rag-retrieval-augmented-generation)

---

*Questions about RAG implementation? [Let me know](mailto:jordan@jordananderson.us).*
