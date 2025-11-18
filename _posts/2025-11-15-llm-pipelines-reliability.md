---
layout: default
title:  "Building Reliable LLM Pipelines: Error Handling and Retries"
date:   2025-11-15 10:00:00
categories: AI LLM Production Reliability
---

LLM APIs fail. Rate limits hit, servers timeout, networks drop. Without proper error handling, your AI application crumbles at the first sign of trouble. Here's how to build pipelines that stay up when providers go down.

## Common LLM Failure Types

### Transient Errors (Retry These)

- **Rate Limiting (429)** - Too many requests
- **Server Errors (500, 502, 503)** - Provider issues
- **Timeouts** - Slow responses
- **Network failures** - DNS, connectivity

### Permanent Errors (Don't Retry)

- **Authentication (401, 403)** - Bad API keys
- **Bad Request (400)** - Malformed input
- **Content Policy (400)** - Blocked content
- **Model Not Found (404)** - Wrong model name

## Exponential Backoff with Jitter

The gold standard for retries:

```python
import random
import time
from functools import wraps

def retry_with_backoff(
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    exponential_base: float = 2.0,
    jitter: bool = True
):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            retries = 0
            while retries <= max_retries:
                try:
                    return func(*args, **kwargs)
                except RetryableError as e:
                    if retries == max_retries:
                        raise

                    # Calculate delay with exponential backoff
                    delay = min(
                        base_delay * (exponential_base ** retries),
                        max_delay
                    )

                    # Add jitter to prevent thundering herd
                    if jitter:
                        delay = delay * (0.5 + random.random())

                    print(f"Retry {retries + 1}/{max_retries} after {delay:.2f}s")
                    time.sleep(delay)
                    retries += 1

            return None
        return wrapper
    return decorator
```

**Why jitter?** When multiple clients retry at the same time (thundering herd), they all hit the API simultaneously and get rate limited again. Jitter spreads retries randomly, reducing collisions.

## Using Tenacity for Robust Retries

Tenacity is the most popular Python retry library:

```python
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential_jitter,
    retry_if_exception_type,
    before_sleep_log
)
import logging
import openai

logger = logging.getLogger(__name__)

class RetryableError(Exception):
    pass

def is_retryable(exception):
    """Determine if an exception should trigger a retry."""
    if isinstance(exception, openai.RateLimitError):
        return True
    if isinstance(exception, openai.APITimeoutError):
        return True
    if isinstance(exception, openai.APIConnectionError):
        return True
    if isinstance(exception, openai.InternalServerError):
        return True
    return False

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential_jitter(initial=1, max=60),
    retry=retry_if_exception_type((
        openai.RateLimitError,
        openai.APITimeoutError,
        openai.APIConnectionError,
        openai.InternalServerError
    )),
    before_sleep=before_sleep_log(logger, logging.WARNING)
)
def call_openai(messages: list[dict]) -> str:
    response = openai.chat.completions.create(
        model="gpt-4",
        messages=messages,
        timeout=30
    )
    return response.choices[0].message.content
```

## Fallback Strategies

When retries fail, fall back to alternative providers:

```python
from typing import Optional

class LLMWithFallbacks:
    def __init__(self):
        self.providers = [
            {"name": "openai", "model": "gpt-4", "client": openai_client},
            {"name": "anthropic", "model": "claude-3-sonnet", "client": anthropic_client},
            {"name": "azure", "model": "gpt-4", "client": azure_client},
        ]

    async def complete(self, messages: list[dict]) -> Optional[str]:
        errors = []

        for provider in self.providers:
            try:
                response = await self._call_provider(provider, messages)
                return response
            except Exception as e:
                errors.append(f"{provider['name']}: {str(e)}")
                continue

        # All providers failed
        raise AllProvidersFailedError(
            f"All providers failed: {'; '.join(errors)}"
        )

    async def _call_provider(self, provider: dict, messages: list[dict]) -> str:
        # Each provider gets its own retry logic
        return await self._with_retries(
            lambda: provider["client"].complete(provider["model"], messages)
        )
```

### Using LiteLLM for Multi-Provider Fallbacks

```python
from litellm import completion

response = completion(
    model="gpt-4",
    messages=[{"role": "user", "content": "Hello"}],
    fallbacks=["claude-3-sonnet-20240229", "azure/gpt-4"],
    num_retries=3
)
```

## Circuit Breakers

Prevent cascading failures by stopping requests to failing services:

```python
import time
from enum import Enum
from dataclasses import dataclass

class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Blocking requests
    HALF_OPEN = "half_open"  # Testing recovery

@dataclass
class CircuitBreaker:
    failure_threshold: int = 5
    recovery_timeout: int = 60
    half_open_max_calls: int = 3

    def __post_init__(self):
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time = None
        self.half_open_calls = 0

    def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if self._should_attempt_reset():
                self.state = CircuitState.HALF_OPEN
                self.half_open_calls = 0
            else:
                raise CircuitOpenError("Circuit breaker is open")

        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _should_attempt_reset(self) -> bool:
        return (
            self.last_failure_time and
            time.time() - self.last_failure_time >= self.recovery_timeout
        )

    def _on_success(self):
        if self.state == CircuitState.HALF_OPEN:
            self.half_open_calls += 1
            if self.half_open_calls >= self.half_open_max_calls:
                self.state = CircuitState.CLOSED
                self.failure_count = 0

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()

        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
```

### Combining Retries and Circuit Breakers

```python
class ResilientLLMClient:
    def __init__(self):
        self.circuit_breaker = CircuitBreaker(
            failure_threshold=5,
            recovery_timeout=60
        )

    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential_jitter(initial=1, max=10)
    )
    def complete(self, messages: list[dict]) -> str:
        return self.circuit_breaker.call(
            self._make_request,
            messages
        )
```

## Timeout Management

Set appropriate timeouts at multiple levels:

```python
import httpx

# Connection timeout: How long to wait for connection
# Read timeout: How long to wait for response

client = openai.OpenAI(
    timeout=httpx.Timeout(
        connect=5.0,    # Connection timeout
        read=30.0,      # Read timeout
        write=10.0,     # Write timeout
        pool=10.0       # Pool timeout
    )
)

# Or simpler
client = openai.OpenAI(timeout=30.0)  # Total timeout
```

### Streaming Timeouts

```python
async def stream_with_timeout(messages: list[dict], timeout: int = 60):
    start_time = time.time()

    async for chunk in client.chat.completions.create(
        model="gpt-4",
        messages=messages,
        stream=True
    ):
        if time.time() - start_time > timeout:
            raise TimeoutError("Stream exceeded timeout")

        yield chunk.choices[0].delta.content
```

## Error Classification

Classify errors to handle them appropriately:

```python
from enum import Enum

class ErrorCategory(Enum):
    RETRYABLE = "retryable"
    FALLBACK = "fallback"
    FATAL = "fatal"

def classify_error(error: Exception) -> ErrorCategory:
    """Classify an error for appropriate handling."""

    # Rate limits - retry with backoff
    if isinstance(error, openai.RateLimitError):
        return ErrorCategory.RETRYABLE

    # Server errors - retry then fallback
    if isinstance(error, openai.InternalServerError):
        return ErrorCategory.RETRYABLE

    # Timeouts - retry with longer timeout
    if isinstance(error, openai.APITimeoutError):
        return ErrorCategory.RETRYABLE

    # Content policy - try different provider
    if "content_policy" in str(error).lower():
        return ErrorCategory.FALLBACK

    # Auth errors - fatal, fix configuration
    if isinstance(error, openai.AuthenticationError):
        return ErrorCategory.FATAL

    # Unknown - treat as fatal
    return ErrorCategory.FATAL

async def handle_with_classification(func, *args, **kwargs):
    try:
        return await func(*args, **kwargs)
    except Exception as e:
        category = classify_error(e)

        if category == ErrorCategory.RETRYABLE:
            # Retry logic here
            pass
        elif category == ErrorCategory.FALLBACK:
            # Use fallback provider
            pass
        else:
            # Log and raise
            raise
```

## Request Queuing

For high-volume applications, queue requests to manage load:

```python
import asyncio
from collections import deque

class RequestQueue:
    def __init__(self, max_concurrent: int = 10, requests_per_minute: int = 60):
        self.semaphore = asyncio.Semaphore(max_concurrent)
        self.rate_limiter = RateLimiter(requests_per_minute)
        self.queue = deque()

    async def enqueue(self, request_func):
        """Add request to queue and process."""
        async with self.semaphore:
            await self.rate_limiter.acquire()
            return await request_func()

class RateLimiter:
    def __init__(self, requests_per_minute: int):
        self.requests_per_minute = requests_per_minute
        self.tokens = requests_per_minute
        self.last_update = time.time()

    async def acquire(self):
        while True:
            self._refill()
            if self.tokens >= 1:
                self.tokens -= 1
                return
            await asyncio.sleep(0.1)

    def _refill(self):
        now = time.time()
        elapsed = now - self.last_update
        self.tokens = min(
            self.requests_per_minute,
            self.tokens + elapsed * (self.requests_per_minute / 60)
        )
        self.last_update = now
```

## Monitoring and Alerting

Track reliability metrics:

```python
from dataclasses import dataclass, field
from collections import defaultdict
import time

@dataclass
class ReliabilityMetrics:
    requests: int = 0
    successes: int = 0
    failures: int = 0
    retries: int = 0
    fallbacks: int = 0
    circuit_breaks: int = 0
    latencies: list = field(default_factory=list)
    errors_by_type: dict = field(default_factory=lambda: defaultdict(int))

    @property
    def success_rate(self) -> float:
        if self.requests == 0:
            return 0
        return self.successes / self.requests

    @property
    def avg_latency(self) -> float:
        if not self.latencies:
            return 0
        return sum(self.latencies) / len(self.latencies)

    @property
    def p99_latency(self) -> float:
        if not self.latencies:
            return 0
        sorted_latencies = sorted(self.latencies)
        idx = int(len(sorted_latencies) * 0.99)
        return sorted_latencies[idx]

class MonitoredLLMClient:
    def __init__(self):
        self.metrics = ReliabilityMetrics()

    async def complete(self, messages: list[dict]) -> str:
        self.metrics.requests += 1
        start_time = time.time()

        try:
            result = await self._complete_with_retries(messages)
            self.metrics.successes += 1
            return result
        except Exception as e:
            self.metrics.failures += 1
            self.metrics.errors_by_type[type(e).__name__] += 1
            raise
        finally:
            latency = time.time() - start_time
            self.metrics.latencies.append(latency)

            # Alert on degradation
            if self.metrics.success_rate < 0.95:
                self._alert("Success rate below 95%")
            if self.metrics.p99_latency > 10:
                self._alert("P99 latency above 10s")
```

## Production Checklist

### Retry Configuration
- [ ] Exponential backoff with jitter
- [ ] Max retry limit (3-5 attempts)
- [ ] Only retry transient errors
- [ ] Log all retry attempts

### Fallback Setup
- [ ] Multiple provider accounts
- [ ] Provider-specific prompts if needed
- [ ] Test fallbacks regularly
- [ ] Monitor fallback usage

### Circuit Breakers
- [ ] Configure failure threshold
- [ ] Set recovery timeout
- [ ] Alert on circuit open
- [ ] Log state transitions

### Timeouts
- [ ] Connection timeout (5-10s)
- [ ] Read timeout (30-60s)
- [ ] Total request timeout
- [ ] Streaming chunk timeout

### Monitoring
- [ ] Success/failure rates
- [ ] Latency percentiles
- [ ] Retry counts
- [ ] Error type breakdown

## Common Mistakes

### 1. Retrying Non-Retryable Errors

```python
# BAD: Retrying auth errors wastes time
@retry(stop=stop_after_attempt(3))
def bad_retry(messages):
    return client.complete(messages)  # Will retry 401 errors

# GOOD: Only retry transient errors
@retry(
    stop=stop_after_attempt(3),
    retry=retry_if_exception_type(openai.RateLimitError)
)
def good_retry(messages):
    return client.complete(messages)
```

### 2. No Jitter

```python
# BAD: All clients retry at exactly 1s, 2s, 4s
wait=wait_exponential(multiplier=1)

# GOOD: Randomized delays
wait=wait_exponential_jitter(initial=1, max=60)
```

### 3. Infinite Retries

Always set a maximum:

```python
# BAD: Could retry forever
@retry(wait=wait_exponential())
def infinite_retry():
    pass

# GOOD: Limited attempts
@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential_jitter()
)
def bounded_retry():
    pass
```

## Resources

- [Tenacity Documentation](https://tenacity.readthedocs.io/)
- [LiteLLM Reliability Guide](https://docs.litellm.ai/docs/completion/reliable_completions)
- [Portkey Circuit Breakers](https://portkey.ai/blog/retries-fallbacks-and-circuit-breakers-in-llm-apps/)
- [LangChain Fallbacks](https://python.langchain.com/v0.1/docs/guides/productionization/fallbacks/)
- [OpenAI Error Handling](https://platform.openai.com/docs/guides/error-codes)

---

*Building reliable LLM applications? [Let me know](mailto:jordan@jordananderson.us) what patterns work for you.*
