---
layout: default
title:  "A/B Testing LLM Responses"
date:   2025-11-17 08:00:00
categories: AI LLM Testing Evaluation
---

A/B testing is your gold-standard tool for comparing LLM models and prompts in production. Unlike offline evaluation, A/B tests measure real user behavior and business outcomes. Here's how to implement them effectively.

## Why A/B Test LLMs?

### The Problem with Offline Evaluation

Offline metrics don't capture:
- User satisfaction
- Business impact
- Real-world edge cases
- Long-term engagement

### What A/B Tests Reveal

- **Causal impact**: Isolate model changes from external factors
- **User behavior**: Completion rates, follow-ups, satisfaction
- **Business metrics**: Conversion, retention, revenue

## A/B Testing Architecture

```python
import hashlib
import random
from dataclasses import dataclass
from typing import Literal

@dataclass
class Variant:
    name: str
    model: str
    prompt_template: str
    weight: float = 0.5

class LLMExperiment:
    def __init__(self, experiment_id: str, variants: list[Variant]):
        self.experiment_id = experiment_id
        self.variants = variants

    def assign_variant(self, user_id: str) -> Variant:
        """Deterministically assign user to variant."""
        # Hash for consistent assignment
        hash_input = f"{self.experiment_id}:{user_id}"
        hash_value = int(hashlib.md5(hash_input.encode()).hexdigest(), 16)

        # Weighted selection
        cumulative = 0
        normalized = (hash_value % 1000) / 1000

        for variant in self.variants:
            cumulative += variant.weight
            if normalized < cumulative:
                return variant

        return self.variants[-1]

# Usage
experiment = LLMExperiment(
    experiment_id="prompt-v2-test",
    variants=[
        Variant(
            name="control",
            model="gpt-4o-mini",
            prompt_template="Answer the question: {question}",
            weight=0.5
        ),
        Variant(
            name="treatment",
            model="gpt-4o-mini",
            prompt_template="You are an expert. Answer concisely: {question}",
            weight=0.5
        )
    ]
)

variant = experiment.assign_variant(user_id="user123")
```

## Feature Flag Integration

### Using PostHog

```python
from posthog import Posthog

posthog = Posthog(
    project_api_key="your-key",
    host="https://app.posthog.com"
)

def get_llm_config(user_id: str) -> dict:
    """Get LLM configuration based on feature flag."""
    variant = posthog.get_feature_flag(
        "llm-prompt-experiment",
        user_id,
        person_properties={"plan": "premium"}
    )

    configs = {
        "control": {
            "model": "gpt-4o-mini",
            "temperature": 0.7,
            "system_prompt": "You are a helpful assistant."
        },
        "concise": {
            "model": "gpt-4o-mini",
            "temperature": 0.5,
            "system_prompt": "You are a helpful assistant. Be concise."
        },
        "detailed": {
            "model": "gpt-4o",
            "temperature": 0.7,
            "system_prompt": "You are a helpful assistant. Provide detailed explanations."
        }
    }

    return configs.get(variant, configs["control"])
```

### Using LaunchDarkly

```python
import ldclient
from ldclient.config import Config

ldclient.set_config(Config("your-sdk-key"))
client = ldclient.get()

def get_model_variant(user: dict) -> str:
    """Get model variant from LaunchDarkly."""
    context = {
        "kind": "user",
        "key": user["id"],
        "custom": {
            "plan": user.get("plan", "free"),
            "country": user.get("country", "US")
        }
    }

    return client.variation(
        "llm-model-selection",
        context,
        "gpt-4o-mini"  # default
    )
```

## Tracking Metrics

### Core Metrics to Track

```python
from dataclasses import dataclass
from datetime import datetime
import json

@dataclass
class LLMInteraction:
    user_id: str
    experiment_id: str
    variant: str
    timestamp: datetime

    # Input metrics
    input_tokens: int

    # Output metrics
    output_tokens: int
    latency_ms: float

    # Quality metrics
    user_rating: int | None = None
    thumbs_up: bool | None = None

    # Business metrics
    task_completed: bool = False
    follow_up_needed: bool = False

    # Cost
    cost_usd: float = 0.0

class MetricsTracker:
    def __init__(self, analytics_client):
        self.analytics = analytics_client

    def track_interaction(self, interaction: LLMInteraction):
        """Track LLM interaction for A/B analysis."""
        self.analytics.track(
            user_id=interaction.user_id,
            event="llm_interaction",
            properties={
                "experiment_id": interaction.experiment_id,
                "variant": interaction.variant,
                "input_tokens": interaction.input_tokens,
                "output_tokens": interaction.output_tokens,
                "latency_ms": interaction.latency_ms,
                "cost_usd": interaction.cost_usd,
                "task_completed": interaction.task_completed,
                "follow_up_needed": interaction.follow_up_needed
            }
        )

    def track_feedback(self, user_id: str, interaction_id: str, rating: int):
        """Track user feedback."""
        self.analytics.track(
            user_id=user_id,
            event="llm_feedback",
            properties={
                "interaction_id": interaction_id,
                "rating": rating
            }
        )
```

### Langfuse Integration

```python
from langfuse import Langfuse
from langfuse.decorators import observe

langfuse = Langfuse()

@observe()
def generate_response(
    user_id: str,
    prompt: str,
    variant: str
) -> str:
    """Generate response with Langfuse tracking."""
    from langfuse.decorators import langfuse_context

    # Set experiment context
    langfuse_context.update_current_trace(
        user_id=user_id,
        tags=[f"variant:{variant}"],
        metadata={"experiment": "prompt-optimization-v2"}
    )

    # Get prompt from Langfuse (enables A/B testing)
    prompt_obj = langfuse.get_prompt(
        name="main-prompt",
        label=variant  # "prod-a" or "prod-b"
    )

    # Generate
    response = call_llm(prompt_obj.compile(input=prompt))

    # Score for evaluation
    langfuse_context.score_current_trace(
        name="response_length",
        value=len(response)
    )

    return response
```

## Statistical Analysis

### Sample Size Calculation

```python
from scipy import stats
import numpy as np

def calculate_sample_size(
    baseline_rate: float,
    minimum_detectable_effect: float,
    alpha: float = 0.05,
    power: float = 0.8
) -> int:
    """Calculate required sample size per variant."""
    # Effect size
    p1 = baseline_rate
    p2 = baseline_rate * (1 + minimum_detectable_effect)

    # Pooled probability
    p_pooled = (p1 + p2) / 2

    # Z-scores
    z_alpha = stats.norm.ppf(1 - alpha / 2)
    z_beta = stats.norm.ppf(power)

    # Sample size formula
    n = (
        (z_alpha * np.sqrt(2 * p_pooled * (1 - p_pooled)) +
         z_beta * np.sqrt(p1 * (1 - p1) + p2 * (1 - p2))) ** 2
    ) / ((p2 - p1) ** 2)

    return int(np.ceil(n))

# Example: 10% baseline completion, detect 5% relative improvement
sample_size = calculate_sample_size(
    baseline_rate=0.10,
    minimum_detectable_effect=0.05
)
print(f"Need {sample_size} samples per variant")
```

### Significance Testing

```python
from scipy import stats
import pandas as pd

def analyze_experiment(control_data: list, treatment_data: list) -> dict:
    """Analyze A/B test results."""
    control = pd.Series(control_data)
    treatment = pd.Series(treatment_data)

    # Basic stats
    control_mean = control.mean()
    treatment_mean = treatment.mean()

    # Relative lift
    lift = (treatment_mean - control_mean) / control_mean

    # T-test
    t_stat, p_value = stats.ttest_ind(control, treatment)

    # Confidence interval for difference
    diff = treatment_mean - control_mean
    se = np.sqrt(control.var()/len(control) + treatment.var()/len(treatment))
    ci_95 = (diff - 1.96 * se, diff + 1.96 * se)

    return {
        "control_mean": control_mean,
        "treatment_mean": treatment_mean,
        "lift": lift,
        "p_value": p_value,
        "significant": p_value < 0.05,
        "confidence_interval": ci_95
    }

# Example usage
results = analyze_experiment(
    control_data=[0.82, 0.85, 0.79, ...],  # relevance scores
    treatment_data=[0.88, 0.91, 0.87, ...]
)

if results["significant"]:
    print(f"Treatment wins with {results['lift']:.1%} lift")
```

### Bayesian Analysis

```python
import numpy as np
from scipy import stats

def bayesian_analysis(
    control_successes: int,
    control_total: int,
    treatment_successes: int,
    treatment_total: int,
    samples: int = 10000
) -> dict:
    """Bayesian A/B test analysis."""
    # Beta posteriors (uninformative prior)
    control_posterior = stats.beta(
        control_successes + 1,
        control_total - control_successes + 1
    )
    treatment_posterior = stats.beta(
        treatment_successes + 1,
        treatment_total - treatment_successes + 1
    )

    # Sample from posteriors
    control_samples = control_posterior.rvs(samples)
    treatment_samples = treatment_posterior.rvs(samples)

    # Probability treatment is better
    prob_treatment_better = np.mean(treatment_samples > control_samples)

    # Expected lift
    lift_samples = (treatment_samples - control_samples) / control_samples
    expected_lift = np.mean(lift_samples)

    return {
        "prob_treatment_better": prob_treatment_better,
        "expected_lift": expected_lift,
        "lift_95_ci": (np.percentile(lift_samples, 2.5), np.percentile(lift_samples, 97.5))
    }
```

## Shadow Testing

For high-stakes applications, test without user impact:

```python
import asyncio
from typing import Any

class ShadowTest:
    def __init__(self, production_model: str, shadow_model: str):
        self.production = production_model
        self.shadow = shadow_model
        self.comparisons = []

    async def run(self, prompt: str) -> str:
        """Run production and shadow models in parallel."""
        # Run both models
        prod_task = self.call_model(self.production, prompt)
        shadow_task = self.call_model(self.shadow, prompt)

        prod_response, shadow_response = await asyncio.gather(
            prod_task, shadow_task
        )

        # Log comparison (async, non-blocking)
        asyncio.create_task(self.log_comparison(
            prompt, prod_response, shadow_response
        ))

        # Return production response
        return prod_response

    async def call_model(self, model: str, prompt: str) -> dict:
        """Call LLM and return response with metadata."""
        start = time.time()
        response = await llm_client.generate(model=model, prompt=prompt)
        latency = time.time() - start

        return {
            "model": model,
            "response": response,
            "latency": latency
        }

    async def log_comparison(self, prompt: str, prod: dict, shadow: dict):
        """Log comparison for analysis."""
        comparison = {
            "prompt": prompt,
            "production": prod,
            "shadow": shadow,
            "latency_diff": shadow["latency"] - prod["latency"]
        }

        # Evaluate quality difference
        comparison["quality_comparison"] = await self.evaluate_quality(
            prod["response"], shadow["response"]
        )

        await self.analytics.log(comparison)
```

## LLM-as-Judge Evaluation

Use an LLM to evaluate responses:

```python
from openai import OpenAI

client = OpenAI()

def evaluate_responses(
    prompt: str,
    response_a: str,
    response_b: str
) -> dict:
    """Use LLM to compare two responses."""
    evaluation_prompt = f"""Compare these two responses to the user's question.

User Question: {prompt}

Response A:
{response_a}

Response B:
{response_b}

Evaluate on these criteria (1-5 scale):
1. Relevance: How well does it address the question?
2. Accuracy: Is the information correct?
3. Clarity: Is it easy to understand?
4. Completeness: Does it fully answer the question?

Return JSON with scores for each response and a winner."""

    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": evaluation_prompt}],
        response_format={"type": "json_object"}
    )

    return json.loads(response.choices[0].message.content)

# Batch evaluation
async def batch_evaluate(test_cases: list) -> dict:
    """Evaluate multiple test cases."""
    results = {"A_wins": 0, "B_wins": 0, "ties": 0}

    for case in test_cases:
        eval_result = evaluate_responses(
            case["prompt"],
            case["response_a"],
            case["response_b"]
        )

        if eval_result["winner"] == "A":
            results["A_wins"] += 1
        elif eval_result["winner"] == "B":
            results["B_wins"] += 1
        else:
            results["ties"] += 1

    return results
```

## Multi-Armed Bandit Approach

Automatically optimize traffic allocation:

```python
import numpy as np

class ThompsonSampling:
    def __init__(self, variants: list[str]):
        self.variants = variants
        # Beta distribution parameters (successes, failures)
        self.alpha = {v: 1 for v in variants}
        self.beta = {v: 1 for v in variants}

    def select_variant(self) -> str:
        """Select variant using Thompson Sampling."""
        samples = {
            v: np.random.beta(self.alpha[v], self.beta[v])
            for v in self.variants
        }
        return max(samples, key=samples.get)

    def update(self, variant: str, success: bool):
        """Update beliefs based on outcome."""
        if success:
            self.alpha[variant] += 1
        else:
            self.beta[variant] += 1

    def get_stats(self) -> dict:
        """Get current beliefs about each variant."""
        return {
            v: {
                "estimated_rate": self.alpha[v] / (self.alpha[v] + self.beta[v]),
                "samples": self.alpha[v] + self.beta[v] - 2
            }
            for v in self.variants
        }

# Usage
bandit = ThompsonSampling(["gpt-4o-mini", "gpt-4o", "claude-3-haiku"])

for request in requests:
    # Select best variant
    variant = bandit.select_variant()

    # Get response
    response = get_llm_response(variant, request)

    # Update based on user feedback
    success = get_user_feedback(response)
    bandit.update(variant, success)
```

## Production Checklist

### Before Launch
- [ ] Define primary metric (e.g., task completion)
- [ ] Define guardrail metrics (latency, cost, errors)
- [ ] Calculate required sample size
- [ ] Set up tracking and logging
- [ ] Implement gradual rollout (1% -> 10% -> 50%)

### During Experiment
- [ ] Monitor guardrail metrics hourly
- [ ] Check for segment imbalances
- [ ] Watch for novelty effects
- [ ] Log qualitative feedback

### Analysis
- [ ] Wait for statistical significance
- [ ] Check for Simpson's paradox (segment-level differences)
- [ ] Analyze by user segments
- [ ] Consider long-term effects

## Common Pitfalls

### 1. Peeking at Results

```python
# Bad: Stopping early when you see significance
if p_value < 0.05:
    stop_experiment()  # Inflates false positive rate

# Good: Pre-register sample size and stick to it
def should_stop(current_samples: int, target_samples: int) -> bool:
    return current_samples >= target_samples
```

### 2. Not Accounting for Multiple Comparisons

```python
# Bad: Testing many metrics without correction
for metric in metrics:
    if test_significance(metric) < 0.05:
        print(f"{metric} is significant!")

# Good: Bonferroni correction
alpha = 0.05 / len(metrics)
for metric in metrics:
    if test_significance(metric) < alpha:
        print(f"{metric} is significant!")
```

### 3. Ignoring Variance

```python
# Bad: Only comparing means
print(f"Treatment mean: {treatment.mean()}")

# Good: Consider variance and confidence intervals
print(f"Treatment: {treatment.mean():.3f} +/- {1.96 * treatment.sem():.3f}")
```

## Resources

- [PostHog LLM A/B Testing Tutorial](https://posthog.com/tutorials/llm-ab-tests)
- [Langfuse Prompt A/B Testing](https://langfuse.com/docs/prompts/a-b-testing)
- [Eppo LLM Comparison Guide](https://www.geteppo.com/blog/how-to-compare-llm-with-an-a-b-test)
- [Confident AI LLM Testing Methods](https://www.confident-ai.com/blog/llm-testing-in-2024-top-methods-and-strategies)

---

*Questions about A/B testing LLMs? [Let me know](mailto:jordan@jordananderson.us).*
