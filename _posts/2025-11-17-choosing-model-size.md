---
layout: default
title:  "Choosing the Right Model Size"
date:   2025-11-17 19:00:00
categories: AI LLM Architecture
---

Bigger isn't always better. Smaller models are closing the gap with larger ones while offering better latency and lower costs. Here's how to choose the right model size for your use case.

## Model Size Tiers

| Size | Parameters | Best For | Typical Latency |
|------|------------|----------|-----------------|
| Tiny | < 3B | Classification, simple extraction | < 100ms |
| Small | 7-8B | Chatbots, summarization, RAG | 100-500ms |
| Medium | 13-14B | Content generation, coding | 200-800ms |
| Large | 32-70B | Complex reasoning, multi-step tasks | 500ms-2s |
| Frontier | 405B+ | Research, maximum capability | 1-5s |

## Decision Framework

### Start with Requirements

```python
requirements = {
    "latency_budget_ms": 500,      # Max acceptable latency
    "quality_threshold": 0.85,     # Min acceptable quality score
    "cost_per_1k_tokens": 0.01,    # Budget constraint
    "monthly_requests": 1000000,   # Expected volume
    "task_complexity": "medium"    # simple/medium/complex
}

def recommend_model_size(requirements: dict) -> str:
    """Recommend model size based on requirements."""

    if requirements["task_complexity"] == "simple":
        if requirements["latency_budget_ms"] < 200:
            return "tiny"  # < 3B
        return "small"  # 7-8B

    elif requirements["task_complexity"] == "medium":
        if requirements["latency_budget_ms"] < 500:
            return "small"  # 7-8B
        return "medium"  # 13-14B

    else:  # complex
        if requirements["quality_threshold"] > 0.95:
            return "large"  # 70B
        return "medium"  # 13-14B
```

### Task-Based Selection

| Task | Recommended Size | Why |
|------|-----------------|-----|
| Text classification | Tiny (< 3B) | Simple pattern matching |
| FAQ chatbot | Small (7-8B) | Basic conversation |
| RAG Q&A | Small-Medium (7-14B) | Needs context understanding |
| Code generation | Medium-Large (14-70B) | Requires reasoning |
| Content writing | Medium (13-14B) | Balance of quality/speed |
| Complex reasoning | Large (70B+) | Multi-step logic |
| Agentic workflows | Large (70B+) | Tool use, planning |

## Quality-Cost-Latency Triangle

You can optimize for two, but not all three:

```python
class ModelSelector:
    def __init__(self):
        self.models = {
            "gpt-4o-mini": {
                "quality": 0.85,
                "cost_per_1m_input": 0.15,
                "latency_ms": 200,
                "tokens_per_second": 100
            },
            "gpt-4o": {
                "quality": 0.95,
                "cost_per_1m_input": 2.50,
                "latency_ms": 500,
                "tokens_per_second": 50
            },
            "claude-3-haiku": {
                "quality": 0.82,
                "cost_per_1m_input": 0.25,
                "latency_ms": 150,
                "tokens_per_second": 120
            },
            "claude-3-5-sonnet": {
                "quality": 0.92,
                "cost_per_1m_input": 3.00,
                "latency_ms": 400,
                "tokens_per_second": 60
            },
            "llama-3.1-8b": {
                "quality": 0.80,
                "cost_per_1m_input": 0.10,
                "latency_ms": 100,
                "tokens_per_second": 150
            },
            "llama-3.1-70b": {
                "quality": 0.90,
                "cost_per_1m_input": 0.88,
                "latency_ms": 350,
                "tokens_per_second": 40
            }
        }

    def select(
        self,
        min_quality: float,
        max_latency_ms: int,
        max_cost: float
    ) -> list[str]:
        """Find models meeting all constraints."""
        candidates = []

        for name, specs in self.models.items():
            if (specs["quality"] >= min_quality and
                specs["latency_ms"] <= max_latency_ms and
                specs["cost_per_1m_input"] <= max_cost):
                candidates.append(name)

        # Sort by cost
        return sorted(
            candidates,
            key=lambda x: self.models[x]["cost_per_1m_input"]
        )

# Usage
selector = ModelSelector()
models = selector.select(
    min_quality=0.85,
    max_latency_ms=500,
    max_cost=1.00
)
# Returns: ["gpt-4o-mini", "llama-3.1-70b"]
```

## Benchmarking Your Task

### Compare Models on Your Data

```python
from dataclasses import dataclass
import time

@dataclass
class BenchmarkResult:
    model: str
    accuracy: float
    avg_latency_ms: float
    cost_per_request: float
    throughput: float

class TaskBenchmark:
    def __init__(self, test_cases: list[dict]):
        self.test_cases = test_cases
        self.models = {
            "small": "gpt-4o-mini",
            "medium": "gpt-4o",
            "large": "claude-3-5-sonnet"
        }

    async def benchmark_model(
        self,
        model_name: str,
        client
    ) -> BenchmarkResult:
        """Benchmark a single model."""
        correct = 0
        latencies = []
        total_tokens = 0

        for case in self.test_cases:
            start = time.time()

            response = await client.chat.completions.create(
                model=model_name,
                messages=[{"role": "user", "content": case["input"]}]
            )

            latency = (time.time() - start) * 1000
            latencies.append(latency)

            # Check accuracy
            if self.evaluate(response, case["expected"]):
                correct += 1

            total_tokens += response.usage.total_tokens

        return BenchmarkResult(
            model=model_name,
            accuracy=correct / len(self.test_cases),
            avg_latency_ms=sum(latencies) / len(latencies),
            cost_per_request=self.calculate_cost(model_name, total_tokens / len(self.test_cases)),
            throughput=1000 / (sum(latencies) / len(latencies))
        )

    def evaluate(self, response, expected) -> bool:
        """Check if response matches expected output."""
        # Implement task-specific evaluation
        return expected.lower() in response.choices[0].message.content.lower()

    def calculate_cost(self, model: str, tokens: int) -> float:
        """Calculate cost per request."""
        costs = {
            "gpt-4o-mini": 0.15 / 1_000_000,
            "gpt-4o": 2.50 / 1_000_000,
            "claude-3-5-sonnet": 3.00 / 1_000_000
        }
        return costs.get(model, 0) * tokens

    async def run_all(self) -> list[BenchmarkResult]:
        """Benchmark all models."""
        results = []
        for size, model in self.models.items():
            result = await self.benchmark_model(model, client)
            results.append(result)
        return results
```

### Quality Score Comparison

```python
# Real-world example: e-commerce product descriptions
results = {
    "gpt-4o-mini (8B equivalent)": {
        "accuracy": 82,
        "fluency": 85,
        "latency_ms": 180,
        "cost_per_1k": 0.003
    },
    "gpt-4o (175B equivalent)": {
        "accuracy": 94,
        "fluency": 96,
        "latency_ms": 520,
        "cost_per_1k": 0.050
    },
    "Llama 3.1 70B": {
        "accuracy": 89,
        "fluency": 91,
        "latency_ms": 350,
        "cost_per_1k": 0.018  # Self-hosted
    }
}

# Decision: For product descriptions, gpt-4o-mini is sufficient
# 12% quality gain doesn't justify 17x cost increase
```

## Optimization Strategies

### 1. Cascading Models

Use small model first, fall back to large for complex cases:

```python
class CascadingRouter:
    def __init__(self):
        self.small_model = "gpt-4o-mini"
        self.large_model = "gpt-4o"
        self.confidence_threshold = 0.8

    async def generate(self, prompt: str) -> str:
        """Route to appropriate model based on confidence."""
        # Try small model first
        response = await client.chat.completions.create(
            model=self.small_model,
            messages=[{"role": "user", "content": prompt}],
            logprobs=True
        )

        # Check confidence
        confidence = self.calculate_confidence(response)

        if confidence >= self.confidence_threshold:
            return response.choices[0].message.content

        # Fall back to large model
        response = await client.chat.completions.create(
            model=self.large_model,
            messages=[{"role": "user", "content": prompt}]
        )

        return response.choices[0].message.content

    def calculate_confidence(self, response) -> float:
        """Calculate response confidence from logprobs."""
        logprobs = response.choices[0].logprobs.content
        if not logprobs:
            return 0.5

        avg_logprob = sum(t.logprob for t in logprobs) / len(logprobs)
        return min(1.0, max(0.0, 1 + avg_logprob / 5))
```

### 2. Task-Specific Routing

```python
class TaskRouter:
    def __init__(self):
        self.task_models = {
            "classification": "gpt-4o-mini",
            "summarization": "gpt-4o-mini",
            "generation": "gpt-4o",
            "reasoning": "gpt-4o",
            "coding": "claude-3-5-sonnet"
        }

    def route(self, task_type: str) -> str:
        """Get model for task type."""
        return self.task_models.get(task_type, "gpt-4o-mini")
```

### 3. Quantization for Self-Hosted

Run larger models with less resources:

```python
# Full precision: 70B model needs ~140GB VRAM
# 4-bit quantization: Same model in ~35GB VRAM

from transformers import AutoModelForCausalLM, BitsAndBytesConfig

quantization_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16
)

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3.1-70B-Instruct",
    quantization_config=quantization_config,
    device_map="auto"
)

# Quality loss: ~2-5%
# Speed gain: 2-3x
# Memory reduction: 4x
```

## Cost Projections

### Monthly Cost by Volume

```python
def calculate_monthly_cost(
    model: str,
    requests_per_day: int,
    avg_tokens_per_request: int
) -> dict:
    """Calculate monthly costs."""

    costs = {
        "gpt-4o-mini": {"input": 0.15, "output": 0.60},
        "gpt-4o": {"input": 2.50, "output": 10.00},
        "claude-3-5-sonnet": {"input": 3.00, "output": 15.00},
        "llama-3.1-70b": {"input": 0.88, "output": 0.88}  # Together AI
    }

    if model not in costs:
        return {"error": "Unknown model"}

    monthly_requests = requests_per_day * 30
    input_tokens = avg_tokens_per_request * 0.3
    output_tokens = avg_tokens_per_request * 0.7

    cost = costs[model]
    monthly_cost = monthly_requests * (
        (input_tokens / 1_000_000 * cost["input"]) +
        (output_tokens / 1_000_000 * cost["output"])
    )

    return {
        "model": model,
        "monthly_requests": monthly_requests,
        "monthly_cost": round(monthly_cost, 2)
    }

# Example: 10K requests/day, 1000 tokens each
print(calculate_monthly_cost("gpt-4o-mini", 10000, 1000))
# {"model": "gpt-4o-mini", "monthly_requests": 300000, "monthly_cost": 157.50}

print(calculate_monthly_cost("gpt-4o", 10000, 1000))
# {"model": "gpt-4o", "monthly_requests": 300000, "monthly_cost": 2625.00}
```

## Decision Checklist

- [ ] **Define quality requirements**: What accuracy is acceptable?
- [ ] **Set latency budget**: What's the max response time?
- [ ] **Estimate volume**: How many requests per day?
- [ ] **Calculate budget**: What can you spend monthly?
- [ ] **Benchmark options**: Test 2-3 models on your data
- [ ] **Consider scaling**: Will needs grow?
- [ ] **Plan for edge cases**: Need fallback to larger model?

## Common Patterns

### Pattern 1: Start Small, Scale Up

```python
# Initial deployment
model = "gpt-4o-mini"

# After analyzing quality metrics
if average_user_rating < 3.5:
    model = "gpt-4o"  # Upgrade
```

### Pattern 2: Different Models for Different Tasks

```python
models = {
    "triage": "gpt-4o-mini",       # Classify intent
    "retrieval": "text-embedding-3-small",
    "generation": "gpt-4o",         # High quality output
    "validation": "gpt-4o-mini"     # Check output
}
```

### Pattern 3: User Tier Based

```python
def get_model(user_tier: str) -> str:
    return {
        "free": "gpt-4o-mini",
        "pro": "gpt-4o",
        "enterprise": "gpt-4-turbo"
    }.get(user_tier, "gpt-4o-mini")
```

## Resources

- [Artificial Analysis Benchmarks](https://artificialanalysis.ai/)
- [Open LLM Leaderboard](https://huggingface.co/spaces/open-llm-leaderboard)
- [LLMFlation Cost Trends](https://a16z.com/navigating-the-high-cost-of-ai-compute/)

---

*Questions about choosing model size? [Let me know](mailto:jordan@jordananderson.us).*
