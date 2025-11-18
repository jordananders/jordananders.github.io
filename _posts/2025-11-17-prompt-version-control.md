---
layout: default
title:  "Prompt Version Control and Testing"
date:   2025-11-17 15:00:00
categories: AI LLM Testing DevOps
---

Prompts are code. They contain logic, affect outputs, and evolve through iteration. Treat them with the same version control, testing, and deployment practices you use for application code. Here's how.

## Why Version Control Prompts?

Without versioning, you'll face:
- Regressions from undocumented changes
- Multiple versions floating around
- Bugs you can't reproduce
- No audit trail for debugging

## Basic Version Control

### File-Based Versioning

```yaml
# prompts/customer-support/v2.yaml
metadata:
  version: "2.1.0"
  author: "team@company.com"
  created: "2024-03-15"
  description: "Improved tone and added product context"
  changelog:
    - "2.1.0: Added product catalog context"
    - "2.0.0: Restructured for clarity"
    - "1.0.0: Initial version"

prompt:
  system: |
    You are a helpful customer support agent for TechCorp.

    Guidelines:
    - Be friendly but professional
    - Always verify customer identity first
    - Escalate billing issues to human agents

    Product catalog:
    {product_context}

  user_template: |
    Customer query: {query}
    Customer tier: {tier}
    Previous interactions: {history}

variables:
  product_context: "retrieval"
  query: "required"
  tier: "default:standard"
  history: "optional"
```

### Git-Based Workflow

```bash
prompts/
├── customer-support/
│   ├── main.yaml           # Production prompt
│   ├── variants/
│   │   ├── concise.yaml    # A/B test variant
│   │   └── detailed.yaml   # A/B test variant
│   └── tests/
│       ├── test_cases.yaml
│       └── golden_responses.yaml
├── code-assistant/
│   └── main.yaml
└── README.md
```

### Semantic Versioning for Prompts

```python
from dataclasses import dataclass
from enum import Enum

class ChangeType(Enum):
    MAJOR = "major"  # Breaking changes to output format
    MINOR = "minor"  # New capabilities, backward compatible
    PATCH = "patch"  # Bug fixes, wording tweaks

@dataclass
class PromptVersion:
    major: int
    minor: int
    patch: int

    def bump(self, change_type: ChangeType) -> "PromptVersion":
        if change_type == ChangeType.MAJOR:
            return PromptVersion(self.major + 1, 0, 0)
        elif change_type == ChangeType.MINOR:
            return PromptVersion(self.major, self.minor + 1, 0)
        else:
            return PromptVersion(self.major, self.minor, self.patch + 1)

    def __str__(self):
        return f"{self.major}.{self.minor}.{self.patch}"
```

## Prompt Registry

Centralized prompt management:

```python
import yaml
from pathlib import Path
from functools import lru_cache

class PromptRegistry:
    def __init__(self, prompts_dir: str = "prompts"):
        self.prompts_dir = Path(prompts_dir)
        self._cache = {}

    def get(self, name: str, version: str = "latest") -> dict:
        """Get prompt by name and version."""
        cache_key = f"{name}:{version}"

        if cache_key not in self._cache:
            prompt_data = self._load_prompt(name, version)
            self._cache[cache_key] = prompt_data

        return self._cache[cache_key]

    def _load_prompt(self, name: str, version: str) -> dict:
        """Load prompt from file system."""
        if version == "latest":
            # Find latest version
            prompt_dir = self.prompts_dir / name
            versions = sorted(prompt_dir.glob("v*.yaml"), reverse=True)
            if not versions:
                raise ValueError(f"No versions found for {name}")
            path = versions[0]
        else:
            path = self.prompts_dir / name / f"v{version}.yaml"

        with open(path) as f:
            return yaml.safe_load(f)

    def render(self, name: str, version: str = "latest", **variables) -> dict:
        """Get prompt with variables filled in."""
        prompt_data = self.get(name, version)

        system = prompt_data["prompt"]["system"].format(**variables)
        user_template = prompt_data["prompt"]["user_template"]

        return {
            "system": system,
            "user_template": user_template,
            "version": prompt_data["metadata"]["version"]
        }

# Usage
registry = PromptRegistry()
prompt = registry.render(
    "customer-support",
    version="2.1.0",
    product_context="..."
)
```

## Testing Prompts

### Unit Tests with Assertions

```python
import pytest
from openai import OpenAI

client = OpenAI()

class PromptTester:
    def __init__(self, prompt: dict, model: str = "gpt-4o-mini"):
        self.prompt = prompt
        self.model = model

    def test(self, user_input: str, assertions: list) -> dict:
        """Test prompt with assertions."""
        response = client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": self.prompt["system"]},
                {"role": "user", "content": user_input}
            ]
        )

        output = response.choices[0].message.content

        results = {
            "passed": True,
            "output": output,
            "assertions": []
        }

        for assertion in assertions:
            passed = assertion["check"](output)
            results["assertions"].append({
                "name": assertion["name"],
                "passed": passed
            })
            if not passed:
                results["passed"] = False

        return results

# Define test cases
test_cases = [
    {
        "input": "What's your return policy?",
        "assertions": [
            {
                "name": "mentions_return_window",
                "check": lambda x: "30 days" in x.lower() or "30-day" in x.lower()
            },
            {
                "name": "professional_tone",
                "check": lambda x: not any(word in x.lower() for word in ["dunno", "idk", "lol"])
            }
        ]
    },
    {
        "input": "I want to cancel my subscription",
        "assertions": [
            {
                "name": "offers_retention",
                "check": lambda x: "discount" in x.lower() or "offer" in x.lower()
            },
            {
                "name": "provides_cancel_option",
                "check": lambda x: "cancel" in x.lower()
            }
        ]
    }
]

# Run tests
def test_customer_support_prompt():
    registry = PromptRegistry()
    prompt = registry.get("customer-support", "2.1.0")
    tester = PromptTester(prompt)

    for case in test_cases:
        result = tester.test(case["input"], case["assertions"])
        assert result["passed"], f"Failed: {case['input']}"
```

### LLM-as-Judge Evaluation

```python
class LLMJudge:
    def __init__(self):
        self.client = OpenAI()

    def evaluate(
        self,
        prompt: str,
        response: str,
        criteria: list[str]
    ) -> dict:
        """Evaluate response using LLM as judge."""
        criteria_text = "\n".join([f"- {c}" for c in criteria])

        judge_prompt = f"""Evaluate this AI response.

User prompt: {prompt}

AI response: {response}

Evaluate on these criteria (score 1-5 for each):
{criteria_text}

Return JSON with scores and brief explanations for each criterion."""

        result = self.client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": judge_prompt}],
            response_format={"type": "json_object"}
        )

        return json.loads(result.choices[0].message.content)

# Usage
judge = LLMJudge()
evaluation = judge.evaluate(
    prompt="What's your return policy?",
    response="We offer a 30-day return policy...",
    criteria=[
        "Accuracy: Information is factually correct",
        "Helpfulness: Fully answers the question",
        "Tone: Professional and friendly",
        "Conciseness: No unnecessary information"
    ]
)
```

### Golden Response Comparison

```python
from sentence_transformers import SentenceTransformer
import numpy as np

class GoldenResponseTester:
    def __init__(self, threshold: float = 0.85):
        self.model = SentenceTransformer('all-MiniLM-L6-v2')
        self.threshold = threshold

    def compare(self, actual: str, expected: str) -> dict:
        """Compare response to golden response."""
        embeddings = self.model.encode([actual, expected])
        similarity = np.dot(embeddings[0], embeddings[1]) / (
            np.linalg.norm(embeddings[0]) * np.linalg.norm(embeddings[1])
        )

        return {
            "similarity": float(similarity),
            "passed": similarity >= self.threshold
        }

# Load golden responses
golden_responses = {
    "return_policy": "We offer a 30-day return policy for all unused items...",
    "shipping_info": "Standard shipping takes 5-7 business days..."
}
```

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/prompt-tests.yml
name: Prompt Tests

on:
  push:
    paths:
      - 'prompts/**'
  pull_request:
    paths:
      - 'prompts/**'

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install pytest openai pyyaml sentence-transformers

      - name: Run prompt tests
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: pytest tests/prompts/ -v

      - name: Run promptfoo eval
        run: |
          npx promptfoo eval --config prompts/eval-config.yaml
          npx promptfoo eval --output results.json

      - name: Check eval results
        run: |
          python scripts/check_eval_results.py results.json --threshold 0.9

      - name: Upload results
        uses: actions/upload-artifact@v4
        with:
          name: prompt-eval-results
          path: results.json
```

### Promptfoo Configuration

```yaml
# prompts/eval-config.yaml
prompts:
  - file://prompts/customer-support/v2.yaml

providers:
  - openai:gpt-4o-mini

tests:
  - description: "Return policy query"
    vars:
      query: "What's your return policy?"
    assert:
      - type: contains
        value: "30 days"
      - type: llm-rubric
        value: "Response is helpful and professional"

  - description: "Billing escalation"
    vars:
      query: "I was charged twice"
    assert:
      - type: contains
        value: "escalate"
      - type: not-contains
        value: "refund processed"  # Should escalate, not resolve

  - description: "Product question"
    vars:
      query: "Does product X work with Y?"
    assert:
      - type: llm-rubric
        value: "Response uses product catalog context accurately"
```

## Deployment Pipeline

### Multi-Environment Deployment

```python
class PromptDeployer:
    def __init__(self):
        self.environments = {
            "dev": "dev-prompts",
            "staging": "staging-prompts",
            "prod": "prod-prompts"
        }

    def deploy(self, name: str, version: str, env: str):
        """Deploy prompt version to environment."""
        if env == "prod":
            # Require approval for production
            if not self.check_approval(name, version):
                raise Exception("Production deployment requires approval")

            # Check tests passed
            if not self.check_tests_passed(name, version):
                raise Exception("All tests must pass for production")

        # Deploy to environment
        registry = self.environments[env]
        self._upload_prompt(name, version, registry)

        # Update routing
        self._update_routing(name, version, env)

        print(f"Deployed {name} v{version} to {env}")

    def rollback(self, name: str, env: str):
        """Rollback to previous version."""
        current = self._get_current_version(name, env)
        previous = self._get_previous_version(name, env)

        self._update_routing(name, previous, env)
        print(f"Rolled back {name} from {current} to {previous}")

    def _upload_prompt(self, name: str, version: str, registry: str):
        # Upload to S3, database, or other storage
        pass

    def _update_routing(self, name: str, version: str, env: str):
        # Update configuration to use new version
        pass
```

### Feature Flags for Prompts

```python
class PromptFeatureFlags:
    def __init__(self, ld_client):
        self.ld = ld_client

    def get_prompt_version(
        self,
        prompt_name: str,
        user_id: str,
        default: str = "v1"
    ) -> str:
        """Get prompt version for user based on feature flags."""
        flag_key = f"prompt-{prompt_name}-version"

        version = self.ld.variation(
            flag_key,
            {"key": user_id},
            default
        )

        return version

# Usage
flags = PromptFeatureFlags(ld_client)

# Get version for user (enables gradual rollout)
version = flags.get_prompt_version("customer-support", user_id)
prompt = registry.get("customer-support", version)
```

## Monitoring in Production

```python
class PromptMonitor:
    def __init__(self):
        self.metrics = defaultdict(lambda: {
            "calls": 0,
            "errors": 0,
            "ratings": [],
            "latencies": []
        })

    def track(
        self,
        prompt_name: str,
        version: str,
        latency: float,
        success: bool,
        user_rating: int = None
    ):
        """Track prompt usage metrics."""
        key = f"{prompt_name}:{version}"

        self.metrics[key]["calls"] += 1
        self.metrics[key]["latencies"].append(latency)

        if not success:
            self.metrics[key]["errors"] += 1

        if user_rating:
            self.metrics[key]["ratings"].append(user_rating)

    def get_health(self, prompt_name: str, version: str) -> dict:
        """Get health metrics for prompt version."""
        key = f"{prompt_name}:{version}"
        data = self.metrics[key]

        return {
            "total_calls": data["calls"],
            "error_rate": data["errors"] / max(1, data["calls"]),
            "avg_latency": sum(data["latencies"]) / max(1, len(data["latencies"])),
            "avg_rating": sum(data["ratings"]) / max(1, len(data["ratings"])),
            "needs_review": self._needs_review(data)
        }

    def _needs_review(self, data: dict) -> bool:
        """Check if prompt needs review."""
        error_rate = data["errors"] / max(1, data["calls"])
        avg_rating = sum(data["ratings"]) / max(1, len(data["ratings"]))

        return error_rate > 0.05 or avg_rating < 3.5
```

## Best Practices

1. **Treat prompts as code**: Version control, review, test, deploy
2. **Use semantic versioning**: Major for breaking changes, minor for features
3. **Test at multiple levels**: Unit, integration, golden responses, LLM-judge
4. **Deploy through environments**: Dev -> Staging -> Production
5. **Monitor in production**: Track metrics, set alerts, enable rollback
6. **Document changes**: Maintain changelog for each prompt

## Tools

- [Promptfoo](https://www.promptfoo.dev/) - Testing and evaluation
- [PromptLayer](https://www.promptlayer.com/) - Version control and observability
- [Langfuse](https://langfuse.com/) - Prompt management and monitoring
- [Humanloop](https://humanloop.com/) - Prompt optimization

---

*Questions about prompt version control? [Let me know](mailto:jordan@jordananderson.us).*
