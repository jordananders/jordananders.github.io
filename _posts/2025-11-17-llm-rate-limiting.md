---
layout: default
title:  "LLM Rate Limiting and Queue Management"
date:   2025-11-17 14:00:00
categories: AI LLM Production Infrastructure
---

LLM APIs have strict rate limits on requests and tokens. Effective rate limiting and queue management prevents errors, ensures fair resource allocation, and maintains service quality. Here's how to implement them properly.

## Understanding Rate Limits

### Provider Limits

```python
RATE_LIMITS = {
    "openai": {
        "gpt-4o": {"rpm": 500, "tpm": 30000},
        "gpt-4o-mini": {"rpm": 500, "tpm": 200000},
        "gpt-4-turbo": {"rpm": 500, "tpm": 30000}
    },
    "anthropic": {
        "claude-3-opus": {"rpm": 50, "tpm": 20000},
        "claude-3-sonnet": {"rpm": 50, "tpm": 40000},
        "claude-3-haiku": {"rpm": 50, "tpm": 100000}
    },
    "together": {
        "llama-3.1-70b": {"rpm": 600, "tpm": 1000000}
    }
}
```

### Error Responses

```python
# OpenAI: 429 Too Many Requests
{
    "error": {
        "message": "Rate limit reached for gpt-4o",
        "type": "rate_limit_error",
        "code": "rate_limit_exceeded"
    }
}

# Anthropic: 429 or 529 (overloaded)
{
    "error": {
        "type": "rate_limit_error",
        "message": "Number of request tokens has exceeded..."
    }
}
```

## Token Bucket Rate Limiter

Classic algorithm for smooth rate limiting:

```python
import asyncio
import time

class TokenBucket:
    def __init__(self, rate: float, capacity: int):
        """
        Args:
            rate: Tokens added per second
            capacity: Maximum tokens in bucket
        """
        self.rate = rate
        self.capacity = capacity
        self.tokens = capacity
        self.last_update = time.time()
        self.lock = asyncio.Lock()

    async def acquire(self, tokens: int = 1) -> float:
        """Acquire tokens, waiting if necessary. Returns wait time."""
        async with self.lock:
            # Refill bucket
            now = time.time()
            elapsed = now - self.last_update
            self.tokens = min(
                self.capacity,
                self.tokens + elapsed * self.rate
            )
            self.last_update = now

            # Calculate wait time if needed
            if self.tokens < tokens:
                wait_time = (tokens - self.tokens) / self.rate
                await asyncio.sleep(wait_time)
                self.tokens = 0
                self.last_update = time.time()
                return wait_time

            self.tokens -= tokens
            return 0

# Usage
rpm_limiter = TokenBucket(rate=500/60, capacity=50)  # 500 RPM

async def rate_limited_call(client, messages):
    await rpm_limiter.acquire(1)
    return await client.chat.completions.create(
        model="gpt-4o-mini",
        messages=messages
    )
```

## Dual Rate Limiter (RPM + TPM)

Handle both request and token limits:

```python
class DualRateLimiter:
    def __init__(self, rpm: int, tpm: int):
        """Rate limiter for both requests and tokens."""
        self.request_bucket = TokenBucket(rate=rpm/60, capacity=rpm//10)
        self.token_bucket = TokenBucket(rate=tpm/60, capacity=tpm//10)

    async def acquire(self, estimated_tokens: int):
        """Acquire both request and token capacity."""
        # Acquire both limits
        await asyncio.gather(
            self.request_bucket.acquire(1),
            self.token_bucket.acquire(estimated_tokens)
        )

    def estimate_tokens(self, messages: list, max_tokens: int) -> int:
        """Estimate total tokens for a request."""
        # Rough estimate: 4 chars = 1 token
        input_tokens = sum(
            len(m.get("content", "")) // 4 for m in messages
        )
        return input_tokens + max_tokens

# Usage
limiter = DualRateLimiter(rpm=500, tpm=200000)

async def call_with_limits(messages: list, max_tokens: int = 500):
    estimated = limiter.estimate_tokens(messages, max_tokens)
    await limiter.acquire(estimated)

    response = await client.chat.completions.create(
        model="gpt-4o-mini",
        messages=messages,
        max_tokens=max_tokens
    )

    return response
```

## Request Queue with Priority

```python
import heapq
from dataclasses import dataclass, field
from typing import Any
import asyncio

@dataclass(order=True)
class PrioritizedRequest:
    priority: int
    timestamp: float = field(compare=False)
    request: Any = field(compare=False)
    future: asyncio.Future = field(compare=False)

class PriorityQueue:
    def __init__(self, max_size: int = 1000):
        self.queue = []
        self.max_size = max_size
        self.lock = asyncio.Lock()

    async def put(self, request: Any, priority: int = 5) -> asyncio.Future:
        """Add request to queue. Returns future with result."""
        async with self.lock:
            if len(self.queue) >= self.max_size:
                raise Exception("Queue full")

            future = asyncio.Future()
            item = PrioritizedRequest(
                priority=priority,
                timestamp=time.time(),
                request=request,
                future=future
            )
            heapq.heappush(self.queue, item)
            return future

    async def get(self) -> PrioritizedRequest | None:
        """Get highest priority request."""
        async with self.lock:
            if self.queue:
                return heapq.heappop(self.queue)
            return None

    @property
    def size(self) -> int:
        return len(self.queue)

class QueuedRateLimiter:
    def __init__(self, rpm: int, tpm: int):
        self.limiter = DualRateLimiter(rpm, tpm)
        self.queue = PriorityQueue()
        self.running = False

    async def start(self):
        """Start processing queue."""
        self.running = True
        while self.running:
            item = await self.queue.get()
            if item:
                try:
                    # Rate limit
                    await self.limiter.acquire(item.request["estimated_tokens"])

                    # Process request
                    result = await self._process(item.request)
                    item.future.set_result(result)
                except Exception as e:
                    item.future.set_exception(e)
            else:
                await asyncio.sleep(0.1)

    async def _process(self, request: dict):
        """Process the actual LLM request."""
        return await client.chat.completions.create(**request["params"])

    async def submit(self, params: dict, priority: int = 5) -> Any:
        """Submit request to queue."""
        request = {
            "params": params,
            "estimated_tokens": self._estimate_tokens(params)
        }
        future = await self.queue.put(request, priority)
        return await future

    def _estimate_tokens(self, params: dict) -> int:
        messages = params.get("messages", [])
        max_tokens = params.get("max_tokens", 500)
        input_tokens = sum(len(m.get("content", "")) // 4 for m in messages)
        return input_tokens + max_tokens
```

## Exponential Backoff with Retry

```python
import random
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type
)
from openai import RateLimitError, APIError

class RetryableClient:
    def __init__(self):
        self.client = OpenAI()

    @retry(
        stop=stop_after_attempt(5),
        wait=wait_exponential(multiplier=1, min=1, max=60),
        retry=retry_if_exception_type((RateLimitError, APIError))
    )
    async def complete(self, **kwargs):
        """Completion with automatic retry on rate limits."""
        return await self.client.chat.completions.create(**kwargs)

# Custom backoff with jitter
async def backoff_retry(func, max_retries: int = 5):
    """Retry with exponential backoff and jitter."""
    for attempt in range(max_retries):
        try:
            return await func()
        except RateLimitError as e:
            if attempt == max_retries - 1:
                raise

            # Exponential backoff with jitter
            wait = min(60, (2 ** attempt) + random.random())
            print(f"Rate limited, waiting {wait:.1f}s...")
            await asyncio.sleep(wait)

# Usage
result = await backoff_retry(
    lambda: client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": "Hello"}]
    )
)
```

## Multi-Provider Load Balancing

```python
class LoadBalancer:
    def __init__(self):
        self.providers = {
            "openai": {
                "client": OpenAI(),
                "limiter": DualRateLimiter(rpm=500, tpm=200000),
                "weight": 0.5,
                "failures": 0
            },
            "anthropic": {
                "client": Anthropic(),
                "limiter": DualRateLimiter(rpm=50, tpm=100000),
                "weight": 0.3,
                "failures": 0
            },
            "together": {
                "client": OpenAI(
                    api_key=os.getenv("TOGETHER_API_KEY"),
                    base_url="https://api.together.xyz/v1"
                ),
                "limiter": DualRateLimiter(rpm=600, tpm=1000000),
                "weight": 0.2,
                "failures": 0
            }
        }

    def select_provider(self) -> str:
        """Select provider based on weights and health."""
        available = [
            (name, p["weight"] * (1 / (1 + p["failures"])))
            for name, p in self.providers.items()
        ]

        total = sum(w for _, w in available)
        r = random.random() * total

        cumulative = 0
        for name, weight in available:
            cumulative += weight
            if r <= cumulative:
                return name

        return available[0][0]

    async def complete(self, messages: list, **kwargs):
        """Complete with automatic failover."""
        tried = set()

        while len(tried) < len(self.providers):
            provider_name = self.select_provider()
            if provider_name in tried:
                continue

            tried.add(provider_name)
            provider = self.providers[provider_name]

            try:
                await provider["limiter"].acquire(
                    self._estimate_tokens(messages, kwargs.get("max_tokens", 500))
                )

                response = await provider["client"].chat.completions.create(
                    messages=messages,
                    **self._adapt_params(provider_name, kwargs)
                )

                # Reset failure count on success
                provider["failures"] = max(0, provider["failures"] - 1)
                return response

            except Exception as e:
                provider["failures"] += 1
                print(f"{provider_name} failed: {e}")
                continue

        raise Exception("All providers failed")

    def _adapt_params(self, provider: str, params: dict) -> dict:
        """Adapt parameters for different providers."""
        if provider == "anthropic":
            return {
                "model": "claude-3-haiku-20240307",
                **params
            }
        elif provider == "together":
            return {
                "model": "meta-llama/Llama-3.1-70B-Instruct-Turbo",
                **params
            }
        return {
            "model": "gpt-4o-mini",
            **params
        }

    def _estimate_tokens(self, messages: list, max_tokens: int) -> int:
        input_tokens = sum(len(m.get("content", "")) // 4 for m in messages)
        return input_tokens + max_tokens
```

## Semaphore-Based Concurrency Control

```python
class ConcurrencyLimiter:
    def __init__(self, max_concurrent: int = 50):
        self.semaphore = asyncio.Semaphore(max_concurrent)
        self.active = 0

    async def __aenter__(self):
        await self.semaphore.acquire()
        self.active += 1
        return self

    async def __aexit__(self, *args):
        self.active -= 1
        self.semaphore.release()

# Usage
limiter = ConcurrencyLimiter(max_concurrent=50)

async def process_batch(prompts: list):
    """Process batch with concurrency limit."""
    async def process_one(prompt):
        async with limiter:
            return await client.chat.completions.create(
                model="gpt-4o-mini",
                messages=[{"role": "user", "content": prompt}]
            )

    return await asyncio.gather(*[process_one(p) for p in prompts])
```

## Request Batching

```python
class BatchProcessor:
    def __init__(self, batch_size: int = 10, wait_time: float = 0.5):
        self.batch_size = batch_size
        self.wait_time = wait_time
        self.pending = []
        self.lock = asyncio.Lock()

    async def add(self, request: dict) -> asyncio.Future:
        """Add request to batch."""
        future = asyncio.Future()

        async with self.lock:
            self.pending.append((request, future))

            if len(self.pending) >= self.batch_size:
                await self._process_batch()

        return future

    async def _process_batch(self):
        """Process accumulated requests."""
        if not self.pending:
            return

        batch = self.pending[:self.batch_size]
        self.pending = self.pending[self.batch_size:]

        # Process batch
        try:
            results = await asyncio.gather(*[
                self._call(req) for req, _ in batch
            ], return_exceptions=True)

            for (_, future), result in zip(batch, results):
                if isinstance(result, Exception):
                    future.set_exception(result)
                else:
                    future.set_result(result)
        except Exception as e:
            for _, future in batch:
                future.set_exception(e)

    async def _call(self, request: dict):
        return await client.chat.completions.create(**request)

    async def flush(self):
        """Process remaining requests."""
        async with self.lock:
            while self.pending:
                await self._process_batch()
```

## Monitoring and Metrics

```python
from dataclasses import dataclass
from datetime import datetime
import statistics

@dataclass
class RateLimitMetrics:
    requests_total: int = 0
    requests_queued: int = 0
    requests_rate_limited: int = 0
    tokens_used: int = 0
    wait_times: list = None

    def __post_init__(self):
        if self.wait_times is None:
            self.wait_times = []

    def record_request(self, tokens: int, wait_time: float):
        self.requests_total += 1
        self.tokens_used += tokens
        if wait_time > 0:
            self.requests_rate_limited += 1
            self.wait_times.append(wait_time)

    def report(self) -> dict:
        return {
            "total_requests": self.requests_total,
            "rate_limited_requests": self.requests_rate_limited,
            "rate_limit_ratio": self.requests_rate_limited / max(1, self.requests_total),
            "total_tokens": self.tokens_used,
            "avg_wait_time": statistics.mean(self.wait_times) if self.wait_times else 0,
            "p99_wait_time": (
                sorted(self.wait_times)[int(len(self.wait_times) * 0.99)]
                if self.wait_times else 0
            )
        }

# Instrumented limiter
class InstrumentedLimiter(DualRateLimiter):
    def __init__(self, rpm: int, tpm: int):
        super().__init__(rpm, tpm)
        self.metrics = RateLimitMetrics()

    async def acquire(self, estimated_tokens: int):
        start = time.time()
        await super().acquire(estimated_tokens)
        wait_time = time.time() - start

        self.metrics.record_request(estimated_tokens, wait_time)
```

## Production Configuration

```python
class ProductionRateLimiter:
    def __init__(self, config: dict):
        self.config = config
        self.limiters = {}
        self.queues = {}
        self.metrics = {}

        for model, limits in config["models"].items():
            self.limiters[model] = InstrumentedLimiter(
                rpm=limits["rpm"],
                tpm=limits["tpm"]
            )
            self.queues[model] = PriorityQueue(
                max_size=config.get("queue_size", 1000)
            )

    async def call(
        self,
        model: str,
        messages: list,
        priority: int = 5,
        **kwargs
    ):
        """Make rate-limited call with queuing."""
        limiter = self.limiters.get(model)
        if not limiter:
            raise ValueError(f"Unknown model: {model}")

        estimated_tokens = self._estimate(messages, kwargs.get("max_tokens", 500))

        # Acquire rate limit
        await limiter.acquire(estimated_tokens)

        # Make call
        return await client.chat.completions.create(
            model=model,
            messages=messages,
            **kwargs
        )

    def _estimate(self, messages: list, max_tokens: int) -> int:
        input_tokens = sum(len(m.get("content", "")) // 4 for m in messages)
        return input_tokens + max_tokens

    def get_metrics(self) -> dict:
        return {
            model: limiter.metrics.report()
            for model, limiter in self.limiters.items()
        }

# Usage
config = {
    "models": {
        "gpt-4o-mini": {"rpm": 500, "tpm": 200000},
        "gpt-4o": {"rpm": 500, "tpm": 30000}
    },
    "queue_size": 1000
}

limiter = ProductionRateLimiter(config)
response = await limiter.call("gpt-4o-mini", messages)
```

## Resources

- [Portkey Rate Limiting Guide](https://portkey.ai/blog/tackling-rate-limiting-for-llm-apps/)
- [OpenAI Rate Limits](https://platform.openai.com/docs/guides/rate-limits)
- [Anthropic Rate Limits](https://docs.anthropic.com/claude/reference/rate-limits)
- [Tenacity Library](https://tenacity.readthedocs.io/)

---

*Questions about LLM rate limiting? [Let me know](mailto:jordan@jordananderson.us).*
