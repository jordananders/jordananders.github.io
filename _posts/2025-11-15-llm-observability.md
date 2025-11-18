---
layout: default
title:  "LLM Observability: Logging, Tracing, and Monitoring"
date:   2025-11-15 08:00:00
categories: AI LLM MLOps Observability
---

LLMs are non-deterministic black boxes. Without proper observability, debugging production issues becomes guesswork. Here's how to instrument LLM applications for production.

## Why LLM Observability is Different

Traditional observability focuses on latency, errors, and throughput. LLM observability adds:

- **Token usage** (cost tracking)
- **Prompt/response quality**
- **Hallucination detection**
- **Semantic drift**
- **Chain/agent step tracing**

## Core Components

### 1. Request/Response Logging

Log every LLM call with metadata:

```python
import json
import time
from datetime import datetime

def log_llm_call(prompt, response, metadata=None):
    log_entry = {
        "timestamp": datetime.utcnow().isoformat(),
        "prompt": prompt,
        "response": response,
        "model": metadata.get("model"),
        "tokens_in": metadata.get("prompt_tokens"),
        "tokens_out": metadata.get("completion_tokens"),
        "latency_ms": metadata.get("latency_ms"),
        "user_id": metadata.get("user_id"),
        "request_id": metadata.get("request_id"),
        "temperature": metadata.get("temperature"),
        "cost": calculate_cost(metadata)
    }
    logger.info(json.dumps(log_entry))

# Usage
start = time.time()
response = openai.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": prompt}],
    temperature=0.3
)
latency = (time.time() - start) * 1000

log_llm_call(prompt, response.choices[0].message.content, {
    "model": "gpt-4",
    "prompt_tokens": response.usage.prompt_tokens,
    "completion_tokens": response.usage.completion_tokens,
    "latency_ms": latency,
    "user_id": user_id,
    "request_id": request_id,
    "temperature": 0.3
})
```

### 2. Distributed Tracing

Trace multi-step LLM workflows:

```python
from opentelemetry import trace
from opentelemetry.trace import SpanKind

tracer = trace.get_tracer(__name__)

async def rag_query(query: str):
    with tracer.start_as_current_span("rag_query") as span:
        span.set_attribute("query", query)

        # Step 1: Embed query
        with tracer.start_as_current_span("embed_query"):
            embedding = await embed(query)

        # Step 2: Retrieve documents
        with tracer.start_as_current_span("retrieve_docs") as retrieve_span:
            docs = await vectorstore.search(embedding, k=5)
            retrieve_span.set_attribute("docs_retrieved", len(docs))

        # Step 3: Generate response
        with tracer.start_as_current_span("generate_response") as gen_span:
            response = await llm.generate(query, docs)
            gen_span.set_attribute("tokens_used", response.usage.total_tokens)

        return response
```

### 3. Key Metrics

```python
from prometheus_client import Counter, Histogram, Gauge

# Request metrics
llm_requests = Counter(
    'llm_requests_total',
    'Total LLM requests',
    ['model', 'status']
)

llm_latency = Histogram(
    'llm_latency_seconds',
    'LLM request latency',
    ['model'],
    buckets=[0.1, 0.5, 1, 2, 5, 10, 30]
)

# Token metrics
tokens_used = Counter(
    'llm_tokens_total',
    'Total tokens used',
    ['model', 'type']  # type: prompt/completion
)

# Cost tracking
llm_cost = Counter(
    'llm_cost_dollars',
    'LLM cost in dollars',
    ['model']
)

# Active requests
active_requests = Gauge(
    'llm_active_requests',
    'Currently active LLM requests'
)

# Usage
active_requests.inc()
with llm_latency.labels(model='gpt-4').time():
    response = await llm.generate(prompt)
active_requests.dec()

llm_requests.labels(model='gpt-4', status='success').inc()
tokens_used.labels(model='gpt-4', type='prompt').inc(response.usage.prompt_tokens)
tokens_used.labels(model='gpt-4', type='completion').inc(response.usage.completion_tokens)
```

## Open Source Tools

### Langfuse

Most popular open source LLM observability:

```python
from langfuse import Langfuse

langfuse = Langfuse()

@langfuse.observe()
def my_llm_function(query):
    # Automatic tracing
    response = openai.chat.completions.create(
        model="gpt-4",
        messages=[{"role": "user", "content": query}]
    )
    return response.choices[0].message.content

# Or manual tracing
trace = langfuse.trace(name="rag-query", user_id="user-123")
generation = trace.generation(
    name="llm-call",
    model="gpt-4",
    input=prompt,
    output=response,
    usage={"input": 100, "output": 50}
)
```

### Phoenix (Arize)

Built for complex LLM pipelines:

```python
import phoenix as px

px.launch_app()

# Automatic instrumentation
from openinference.instrumentation.openai import OpenAIInstrumentor
OpenAIInstrumentor().instrument()

# Now all OpenAI calls are traced
```

### Traceloop (OpenLLMetry)

OpenTelemetry-native:

```python
from traceloop.sdk import Traceloop

Traceloop.init()

# Integrates with existing OTel setup
# Export to Jaeger, Grafana, Datadog, etc.
```

## What to Monitor

### Performance Metrics

```python
# Dashboard essentials
metrics = {
    "latency_p50": "50th percentile response time",
    "latency_p95": "95th percentile response time",
    "latency_p99": "99th percentile response time",
    "throughput": "Requests per minute",
    "error_rate": "Failed requests / total requests",
    "timeout_rate": "Timeouts / total requests"
}
```

### Cost Metrics

```python
def calculate_cost(model, prompt_tokens, completion_tokens):
    pricing = {
        "gpt-4": {"input": 0.03, "output": 0.06},
        "gpt-4-turbo": {"input": 0.01, "output": 0.03},
        "gpt-3.5-turbo": {"input": 0.0005, "output": 0.0015},
        "claude-3-opus": {"input": 0.015, "output": 0.075},
        "claude-3-sonnet": {"input": 0.003, "output": 0.015}
    }
    rates = pricing.get(model, {"input": 0, "output": 0})
    return (prompt_tokens * rates["input"] + completion_tokens * rates["output"]) / 1000
```

### Quality Metrics

```python
# Track response quality
quality_metrics = {
    "user_feedback": "Thumbs up/down",
    "relevance_score": "Retrieved doc relevance",
    "factuality": "Grounded in sources",
    "coherence": "Response makes sense",
    "completion_rate": "User finished task"
}
```

## Alerting

```yaml
# Prometheus alerting rules
groups:
  - name: llm_alerts
    rules:
      - alert: HighLLMLatency
        expr: histogram_quantile(0.95, llm_latency_seconds) > 10
        for: 5m
        annotations:
          summary: "LLM p95 latency > 10s"

      - alert: HighErrorRate
        expr: rate(llm_requests_total{status="error"}[5m]) / rate(llm_requests_total[5m]) > 0.05
        for: 5m
        annotations:
          summary: "LLM error rate > 5%"

      - alert: HighCost
        expr: increase(llm_cost_dollars[1h]) > 100
        annotations:
          summary: "LLM cost > $100/hour"

      - alert: TokenBudgetExceeded
        expr: sum(increase(llm_tokens_total[1d])) > 1000000
        annotations:
          summary: "Daily token budget exceeded"
```

## Debugging LLM Issues

### Common Problems

**1. Slow responses**
- Check model choice (GPT-4 > GPT-3.5 latency)
- Check prompt length
- Check concurrent requests

**2. High error rates**
- Rate limiting
- Invalid prompts
- Context length exceeded

**3. Poor quality outputs**
- Check retrieved context relevance
- Check prompt template
- Check temperature/parameters

### Debug Workflow

```python
# Find problematic requests
SELECT
    request_id,
    prompt,
    response,
    latency_ms,
    tokens_in + tokens_out as total_tokens
FROM llm_logs
WHERE timestamp > NOW() - INTERVAL '1 hour'
    AND (latency_ms > 5000 OR status = 'error')
ORDER BY timestamp DESC
LIMIT 100;
```

## Production Checklist

- [ ] Log all prompts and responses
- [ ] Track token usage and costs
- [ ] Implement distributed tracing
- [ ] Set up latency/error dashboards
- [ ] Configure alerts for anomalies
- [ ] Track user feedback
- [ ] Monitor for prompt injection attempts
- [ ] Implement cost controls

## Resources

- [LLM Observability Tools 2025 - lakeFS](https://lakefs.io/blog/llm-observability-tools/)
- [Top 10 LLM Observability Tools - Coralogix](https://coralogix.com/guides/llm-observability-tools/)
- [Datadog LLM Observability](https://www.datadoghq.com/product/llm-observability/)
- [Top 10 Tools 2025 - Braintrust](https://www.braintrust.dev/articles/top-10-llm-observability-tools-2025)
- [Langfuse Documentation](https://langfuse.com/docs)

---

*Questions about LLM observability? [Let me know](mailto:jordan@jordananderson.us).*
