---
layout: default
title:  "Latency Optimization for LLM Applications"
date:   2025-11-17 12:00:00
categories: AI LLM Performance Production
---

LLM latency directly impacts user experience. Time-to-first-token determines perceived responsiveness, while tokens-per-second affects overall speed. Here's how to optimize both in production.

## Understanding LLM Latency

```
Total Latency = TTFT + (Time per token × Output tokens)
```

Key metrics:
- **TTFT (Time to First Token)**: How long until streaming begins
- **TPOT (Time per Output Token)**: Speed of token generation
- **Total latency**: End-to-end response time

Typical ranges:
- TTFT: 100ms - 2s
- TPOT: 10ms - 100ms per token
- Total: 500ms - 30s depending on output length

## Streaming Responses

The most impactful UX improvement:

```python
from openai import OpenAI

client = OpenAI()

def stream_response(prompt: str):
    """Stream response for immediate feedback."""
    stream = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        stream=True
    )

    for chunk in stream:
        if chunk.choices[0].delta.content:
            yield chunk.choices[0].delta.content

# FastAPI endpoint
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

@app.get("/chat")
async def chat(prompt: str):
    return StreamingResponse(
        stream_response(prompt),
        media_type="text/event-stream"
    )
```

### Server-Sent Events (SSE)

```python
async def sse_stream(prompt: str):
    """Format as Server-Sent Events."""
    stream = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        stream=True
    )

    for chunk in stream:
        content = chunk.choices[0].delta.content
        if content:
            yield f"data: {content}\n\n"

    yield "data: [DONE]\n\n"
```

### Client-Side Handling

```javascript
const eventSource = new EventSource('/chat?prompt=' + encodeURIComponent(prompt));

let fullResponse = '';

eventSource.onmessage = (event) => {
    if (event.data === '[DONE]') {
        eventSource.close();
        return;
    }

    fullResponse += event.data;
    document.getElementById('response').textContent = fullResponse;
};
```

## Prompt Optimization

### Minimize Input Tokens

```python
def optimize_prompt(prompt: str, context: str) -> str:
    """Reduce prompt length while preserving information."""
    # Bad: Long, verbose prompt
    bad_prompt = f"""
    I have the following context information that I would like you to consider
    when answering my question. Please read through all of this carefully:

    {context}

    Now, based on the above context, please answer the following question
    in a helpful and detailed manner:

    {prompt}
    """

    # Good: Concise, focused prompt
    good_prompt = f"""Context: {context[:2000]}

    Question: {prompt}

    Answer concisely."""

    return good_prompt
```

### Limit Output Tokens

```python
def get_response(prompt: str, max_tokens: int = 500) -> str:
    """Limit output length for faster responses."""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        max_tokens=max_tokens,  # Hard limit
        temperature=0.7
    )

    return response.choices[0].message.content
```

## Caching Strategies

### Exact Match Cache

```python
import hashlib
import redis

class ResponseCache:
    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis = redis.from_url(redis_url)
        self.ttl = 3600  # 1 hour

    def _cache_key(self, model: str, messages: list) -> str:
        content = f"{model}:{json.dumps(messages)}"
        return f"llm:{hashlib.sha256(content.encode()).hexdigest()}"

    def get(self, model: str, messages: list) -> str | None:
        key = self._cache_key(model, messages)
        return self.redis.get(key)

    def set(self, model: str, messages: list, response: str):
        key = self._cache_key(model, messages)
        self.redis.setex(key, self.ttl, response)

# Usage
cache = ResponseCache()

def cached_completion(model: str, messages: list) -> str:
    # Check cache
    cached = cache.get(model, messages)
    if cached:
        return cached.decode()

    # Call API
    response = client.chat.completions.create(
        model=model,
        messages=messages
    )

    result = response.choices[0].message.content

    # Cache result
    cache.set(model, messages, result)

    return result
```

### Semantic Cache

```python
from sentence_transformers import SentenceTransformer
import numpy as np

class SemanticCache:
    def __init__(self, similarity_threshold: float = 0.95):
        self.model = SentenceTransformer('all-MiniLM-L6-v2')
        self.threshold = similarity_threshold
        self.cache = []  # (embedding, query, response)

    def get(self, query: str) -> str | None:
        if not self.cache:
            return None

        query_embedding = self.model.encode(query)

        for emb, cached_query, response in self.cache:
            similarity = np.dot(query_embedding, emb) / (
                np.linalg.norm(query_embedding) * np.linalg.norm(emb)
            )

            if similarity > self.threshold:
                return response

        return None

    def set(self, query: str, response: str):
        embedding = self.model.encode(query)
        self.cache.append((embedding, query, response))
```

### KV Cache for Self-Hosted Models

```python
# For vLLM or similar
from vllm import LLM, SamplingParams

llm = LLM(
    model="meta-llama/Llama-2-7b-hf",
    # Enable KV cache
    gpu_memory_utilization=0.9,
    max_model_len=4096
)

# Prefix caching for repeated context
llm = LLM(
    model="meta-llama/Llama-2-7b-hf",
    enable_prefix_caching=True  # Cache shared prefixes
)
```

## Model Selection

### Speed vs Quality Trade-offs

```python
class ModelRouter:
    def __init__(self):
        self.models = {
            "fast": "gpt-4o-mini",      # ~50-100 tok/s
            "balanced": "gpt-4o",        # ~30-50 tok/s
            "quality": "gpt-4-turbo",    # ~20-30 tok/s
        }

    def select_model(self, task: str, latency_budget_ms: int) -> str:
        """Select model based on task and latency requirements."""
        if latency_budget_ms < 1000:
            return self.models["fast"]
        elif task in ["analysis", "coding", "reasoning"]:
            return self.models["quality"]
        else:
            return self.models["balanced"]
```

### Speculative Decoding

Use fast model to generate candidates, verify with slow model:

```python
async def speculative_decode(prompt: str) -> str:
    """Use fast model to speed up slow model."""
    # Generate candidates with fast model
    fast_response = await client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        n=3  # Multiple candidates
    )

    candidates = [c.message.content for c in fast_response.choices]

    # Verify best candidate with slow model
    verification_prompt = f"""Original question: {prompt}

    Candidate answer: {candidates[0]}

    Is this answer correct and complete? If not, provide the correct answer."""

    verified = await client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": verification_prompt}]
    )

    return verified.choices[0].message.content
```

## Parallel Processing

### Concurrent API Calls

```python
import asyncio
from openai import AsyncOpenAI

async_client = AsyncOpenAI()

async def parallel_completions(prompts: list[str]) -> list[str]:
    """Process multiple prompts in parallel."""
    tasks = [
        async_client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}]
        )
        for prompt in prompts
    ]

    responses = await asyncio.gather(*tasks)

    return [r.choices[0].message.content for r in responses]
```

### Batch with Map-Reduce

```python
async def map_reduce_summarize(documents: list[str]) -> str:
    """Parallel map, sequential reduce."""
    # Map: Summarize each document in parallel
    summaries = await parallel_completions([
        f"Summarize this document in 2 sentences:\n\n{doc}"
        for doc in documents
    ])

    # Reduce: Combine summaries
    final = await async_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{
            "role": "user",
            "content": f"Combine these summaries:\n\n" + "\n\n".join(summaries)
        }]
    )

    return final.choices[0].message.content
```

## Infrastructure Optimization

### Connection Pooling

```python
import httpx
from openai import OpenAI

# Configure connection pool
transport = httpx.HTTPTransport(
    retries=3,
    http2=True,  # Enable HTTP/2
    limits=httpx.Limits(
        max_connections=100,
        max_keepalive_connections=20
    )
)

client = OpenAI(
    http_client=httpx.Client(transport=transport)
)
```

### Regional Deployment

```python
class RegionalRouter:
    def __init__(self):
        self.endpoints = {
            "us-east": "https://api.openai.com/v1",
            "eu-west": "https://api.openai.com/v1",
            # Azure OpenAI regional endpoints
            "us-east-azure": "https://your-resource.openai.azure.com"
        }

    def get_client(self, user_region: str) -> OpenAI:
        """Get client for nearest region."""
        region = self.get_nearest_region(user_region)
        return OpenAI(base_url=self.endpoints[region])

    def get_nearest_region(self, user_region: str) -> str:
        # Implement region mapping logic
        region_map = {
            "NA": "us-east",
            "EU": "eu-west",
            "APAC": "us-east"  # Fallback
        }
        return region_map.get(user_region, "us-east")
```

## Self-Hosted Optimization

### Quantization

```python
from transformers import AutoModelForCausalLM, BitsAndBytesConfig

# 4-bit quantization
quantization_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_compute_dtype=torch.float16,
    bnb_4bit_quant_type="nf4"
)

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf",
    quantization_config=quantization_config,
    device_map="auto"
)
```

### vLLM for High Throughput

```python
from vllm import LLM, SamplingParams

# Optimized inference engine
llm = LLM(
    model="meta-llama/Llama-2-7b-hf",
    tensor_parallel_size=2,  # Use 2 GPUs
    gpu_memory_utilization=0.9,
    enforce_eager=False  # Enable CUDA graphs
)

sampling_params = SamplingParams(
    temperature=0.7,
    max_tokens=500
)

outputs = llm.generate(prompts, sampling_params)
```

### TensorRT-LLM

```python
# Build optimized engine (CLI)
# trtllm-build --checkpoint_dir ./checkpoint --output_dir ./engine

from tensorrt_llm import LLM

llm = LLM(model="./engine")
outputs = llm.generate(["Hello, world!"])
```

## Monitoring & Benchmarking

```python
import time
from dataclasses import dataclass

@dataclass
class LatencyMetrics:
    ttft: float
    total_time: float
    tokens_generated: int
    tokens_per_second: float

def benchmark_completion(prompt: str) -> LatencyMetrics:
    """Measure latency metrics for a completion."""
    start = time.time()
    first_token_time = None
    tokens = 0

    stream = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        stream=True
    )

    for chunk in stream:
        if chunk.choices[0].delta.content:
            if first_token_time is None:
                first_token_time = time.time()
            tokens += 1

    end = time.time()

    ttft = first_token_time - start if first_token_time else 0
    total = end - start
    tps = tokens / (end - first_token_time) if first_token_time else 0

    return LatencyMetrics(
        ttft=ttft,
        total_time=total,
        tokens_generated=tokens,
        tokens_per_second=tps
    )

# Run benchmark
metrics = benchmark_completion("Write a short poem about coding")
print(f"TTFT: {metrics.ttft*1000:.0f}ms")
print(f"Tokens/sec: {metrics.tokens_per_second:.1f}")
```

## Production Checklist

- [ ] Enable streaming for all user-facing responses
- [ ] Implement response caching (exact + semantic)
- [ ] Use faster models for simple tasks
- [ ] Set appropriate max_tokens limits
- [ ] Configure connection pooling
- [ ] Deploy to regional endpoints
- [ ] Monitor TTFT and throughput
- [ ] Set up latency alerts

## Optimization Priority

1. **Streaming** - Immediate UX improvement
2. **Caching** - Eliminates repeat calls
3. **Model selection** - Right tool for job
4. **Prompt optimization** - Reduces input processing
5. **Infrastructure** - Connection pooling, regions
6. **Quantization/vLLM** - For self-hosted

## Resources

- [AWS Bedrock Latency Guide](https://aws.amazon.com/blogs/machine-learning/optimizing-ai-responsiveness-a-practical-guide-to-amazon-bedrock-latency-optimized-inference/)
- [Databricks Inference Best Practices](https://www.databricks.com/blog/llm-inference-performance-engineering-best-practices)
- [vLLM Documentation](https://docs.vllm.ai/)
- [TensorRT-LLM](https://github.com/NVIDIA/TensorRT-LLM)

---

*Questions about LLM latency optimization? [Let me know](mailto:jordan@jordananderson.us).*
