---
layout: default
title:  "LLM Token Optimization Techniques"
date:   2025-11-15 12:00:00
categories: AI LLM Cost Optimization
---

LLM API costs are dominated by tokens. More tokens = higher costs and slower responses. With strategic optimization, you can reduce costs by 60-80% without sacrificing quality. Here's how.

## Why Tokens Matter

Every API call bills on tokens:
- **Input tokens**: Your prompt + context
- **Output tokens**: Model's response (usually 2-3x more expensive)

A 1000-token prompt at $0.01/1K input tokens costs $0.01 per call. At 100K calls/day, that's $1,000/day just for input. Optimization directly impacts your bottom line.

## Prompt Compression

### Remove Redundancy

```python
# BEFORE: 156 tokens
"""
I would like you to help me with a task. The task that I need help with
is analyzing a piece of text. The text that I want you to analyze is a
customer review. I want you to determine if the sentiment of this customer
review is positive, negative, or neutral. Please analyze the following
customer review and tell me what the sentiment is:
"""

# AFTER: 24 tokens
"""
Classify sentiment (positive/negative/neutral):
"""
```

### Use Abbreviations in System Prompts

```python
# BEFORE: 89 tokens
system_prompt = """
You are a helpful customer service assistant for TechCorp Inc.
Your role is to help customers with their technical issues,
answer questions about products, and provide support in a
friendly and professional manner. Always be polite and thorough.
"""

# AFTER: 34 tokens
system_prompt = """
TechCorp support agent. Help with tech issues, product questions.
Be professional, thorough.
"""
```

### LLMLingua for Automatic Compression

```python
from llmlingua import PromptCompressor

compressor = PromptCompressor(
    model_name="microsoft/llmlingua-2-bert-base-multilingual-cased-meetingbank"
)

# Compress long context
compressed = compressor.compress_prompt(
    context=long_document,
    instruction="Summarize the key findings",
    question="What are the main conclusions?",
    rate=0.5  # Target 50% of original tokens
)

# Use compressed prompt
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": compressed["compressed_prompt"]}]
)
```

## Context Caching

Cache static content to avoid re-sending:

### Anthropic Prompt Caching

```python
import anthropic

client = anthropic.Anthropic()

# First call - cache the system prompt
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": large_static_context,  # 10K+ tokens of documentation
            "cache_control": {"type": "ephemeral"}
        }
    ],
    messages=[{"role": "user", "content": "How do I configure X?"}]
)

# Subsequent calls - cached content is reused
# Only pay for cache read (90% cheaper than re-processing)
```

### OpenAI Stored Completions

```python
# Store common context server-side
stored_context = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": documentation},
        {"role": "user", "content": "Acknowledge context"},
    ],
    store=True
)

# Reference stored context in future calls
response = client.chat.completions.create(
    model="gpt-4o",
    previous_response_id=stored_context.id,
    messages=[{"role": "user", "content": "How do I configure X?"}]
)
```

## Smart Model Routing

Route requests to appropriate model sizes:

```python
from enum import Enum

class TaskComplexity(Enum):
    SIMPLE = "simple"      # Classification, extraction
    MEDIUM = "medium"      # Summarization, Q&A
    COMPLEX = "complex"    # Reasoning, coding, analysis

def select_model(task: TaskComplexity) -> str:
    models = {
        TaskComplexity.SIMPLE: "gpt-4o-mini",      # $0.15/1M tokens
        TaskComplexity.MEDIUM: "gpt-4o",           # $2.50/1M tokens
        TaskComplexity.COMPLEX: "gpt-4-turbo",     # $10/1M tokens
    }
    return models[task]

def classify_task(prompt: str) -> TaskComplexity:
    """Use small model to classify task complexity."""

    classification_prompt = f"""
    Classify task complexity: simple/medium/complex

    Simple: extraction, classification, formatting
    Medium: summarization, Q&A, translation
    Complex: reasoning, coding, analysis, creative

    Task: {prompt[:200]}

    Output only: simple, medium, or complex
    """

    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": classification_prompt}],
        max_tokens=10
    )

    result = response.choices[0].message.content.lower().strip()
    return TaskComplexity(result)

# Usage
async def process_request(prompt: str) -> str:
    complexity = classify_task(prompt)
    model = select_model(complexity)

    response = await client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}]
    )

    return response.choices[0].message.content
```

## RAG for Context Efficiency

Only retrieve relevant context instead of sending everything:

```python
from sentence_transformers import SentenceTransformer
import numpy as np

class EfficientRAG:
    def __init__(self):
        self.encoder = SentenceTransformer('all-MiniLM-L6-v2')
        self.chunks = []
        self.embeddings = []

    def add_documents(self, documents: list[str], chunk_size: int = 500):
        """Split documents into chunks and embed."""
        for doc in documents:
            # Chunk the document
            doc_chunks = self._chunk_text(doc, chunk_size)
            self.chunks.extend(doc_chunks)

        # Embed all chunks
        self.embeddings = self.encoder.encode(self.chunks)

    def query(self, question: str, top_k: int = 3) -> str:
        """Retrieve only relevant chunks."""
        # Embed question
        q_embedding = self.encoder.encode([question])[0]

        # Find most similar chunks
        similarities = np.dot(self.embeddings, q_embedding)
        top_indices = np.argsort(similarities)[-top_k:][::-1]

        # Return only relevant context
        relevant_chunks = [self.chunks[i] for i in top_indices]
        return "\n\n".join(relevant_chunks)

# Usage
rag = EfficientRAG()
rag.add_documents(all_documentation)  # 100K tokens total

# Instead of sending all 100K tokens, send only relevant ~1K
relevant_context = rag.query("How do I configure authentication?")

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": f"Context:\n{relevant_context}"},
        {"role": "user", "content": "How do I configure authentication?"}
    ]
)
```

## Response Caching

Cache common responses:

```python
import hashlib
import redis
import json

class ResponseCache:
    def __init__(self):
        self.redis = redis.Redis()
        self.ttl = 3600  # 1 hour

    def _hash_request(self, model: str, messages: list[dict]) -> str:
        """Create deterministic hash of request."""
        content = json.dumps({"model": model, "messages": messages}, sort_keys=True)
        return hashlib.sha256(content.encode()).hexdigest()

    async def get_or_create(self, model: str, messages: list[dict]) -> str:
        cache_key = self._hash_request(model, messages)

        # Check cache
        cached = self.redis.get(cache_key)
        if cached:
            return cached.decode()

        # Call API
        response = await client.chat.completions.create(
            model=model,
            messages=messages
        )
        result = response.choices[0].message.content

        # Cache response
        self.redis.setex(cache_key, self.ttl, result)

        return result

# High cache hit rate for common queries
cache = ResponseCache()
response = await cache.get_or_create(
    "gpt-4o-mini",
    [{"role": "user", "content": "What are your business hours?"}]
)
```

## Chat History Management

Summarize long conversations:

```python
class ConversationManager:
    def __init__(self, max_tokens: int = 2000, summarize_after: int = 10):
        self.messages = []
        self.max_tokens = max_tokens
        self.summarize_after = summarize_after
        self.summary = ""

    def add_message(self, role: str, content: str):
        self.messages.append({"role": role, "content": content})

        # Summarize if too many messages
        if len(self.messages) >= self.summarize_after:
            self._summarize()

    def _summarize(self):
        """Compress conversation history into summary."""
        conversation = "\n".join(
            f"{m['role']}: {m['content']}" for m in self.messages[:-2]
        )

        summary_response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{
                "role": "user",
                "content": f"Summarize this conversation in 2-3 sentences:\n{conversation}"
            }],
            max_tokens=150
        )

        self.summary = summary_response.choices[0].message.content

        # Keep only last 2 messages + summary
        self.messages = self.messages[-2:]

    def get_messages(self) -> list[dict]:
        """Get messages for API call."""
        result = []

        if self.summary:
            result.append({
                "role": "system",
                "content": f"Previous conversation summary: {self.summary}"
            })

        result.extend(self.messages)
        return result
```

## Output Token Control

### Set Max Tokens

```python
# Don't let model ramble
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": prompt}],
    max_tokens=500  # Limit response length
)
```

### Request Concise Output

```python
# Be explicit about brevity
system_prompt = """
Provide concise responses. Use bullet points for lists.
Limit explanations to 2-3 sentences unless asked for detail.
"""
```

### Use Structured Output

```python
from pydantic import BaseModel

class BriefAnswer(BaseModel):
    answer: str  # Max 100 chars
    confidence: float

# Structured output is naturally concise
response = client.beta.chat.completions.parse(
    model="gpt-4o-2024-08-06",
    messages=[{"role": "user", "content": question}],
    response_format=BriefAnswer
)
```

## Measuring Optimization Impact

```python
from dataclasses import dataclass, field

@dataclass
class TokenMetrics:
    endpoint: str
    input_tokens: list[int] = field(default_factory=list)
    output_tokens: list[int] = field(default_factory=list)
    costs: list[float] = field(default_factory=list)

    def add_usage(self, input_tokens: int, output_tokens: int, cost: float):
        self.input_tokens.append(input_tokens)
        self.output_tokens.append(output_tokens)
        self.costs.append(cost)

    def report(self) -> dict:
        return {
            "endpoint": self.endpoint,
            "avg_input_tokens": sum(self.input_tokens) / len(self.input_tokens),
            "avg_output_tokens": sum(self.output_tokens) / len(self.output_tokens),
            "total_cost": sum(self.costs),
            "avg_cost": sum(self.costs) / len(self.costs),
            "total_calls": len(self.costs)
        }

# Track before and after optimization
before_metrics = TokenMetrics("customer_service")
after_metrics = TokenMetrics("customer_service_optimized")

# Compare reports to measure savings
```

## Optimization Checklist

### Quick Wins (Day 1)
- [ ] Audit prompts for redundancy
- [ ] Set appropriate max_tokens
- [ ] Enable provider caching (if available)
- [ ] Implement response caching for common queries

### Medium Effort (Week 1)
- [ ] Implement model routing (small/medium/large)
- [ ] Add RAG for dynamic context
- [ ] Compress chat history
- [ ] Use structured outputs

### Advanced (Month 1)
- [ ] Train custom small model for specific tasks
- [ ] Implement prompt compression (LLMLingua)
- [ ] Consider self-hosting for high volume
- [ ] Fine-tune for domain-specific efficiency

## Cost Comparison Example

**Before Optimization:**
- 100K requests/day
- Average 2000 input tokens, 500 output tokens
- Model: GPT-4 Turbo
- Daily cost: ~$3,500

**After Optimization:**
- Model routing: 70% to GPT-4o-mini
- RAG reduces context: 2000 → 500 tokens
- Response caching: 30% hit rate
- Chat summarization: 40% reduction
- Daily cost: ~$450

**Savings: 87%**

## Resources

- [OpenAI Token Usage](https://platform.openai.com/docs/guides/production-best-practices)
- [Anthropic Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [LLMLingua Paper](https://arxiv.org/abs/2310.05736)
- [Tiktoken (Token Counter)](https://github.com/openai/tiktoken)

---

*Questions about optimizing LLM costs? [Let me know](mailto:jordan@jordananderson.us).*
