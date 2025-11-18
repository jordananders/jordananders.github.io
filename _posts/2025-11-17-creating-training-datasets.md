---
layout: default
title:  "Creating Quality Training Datasets for LLM Fine-Tuning"
date:   2025-11-17 10:00:00
categories: AI LLM Fine-tuning Data
---

High-quality training data is the most important factor in fine-tuning LLM performance. With public datasets largely exhausted, synthetic data generation has become essential. Here's how to create effective datasets for your fine-tuning needs.

## Why Data Quality Matters

Poor data leads to:
- Hallucinations and errors
- Inconsistent outputs
- Bias amplification
- Poor generalization

Good data enables:
- Task-specific expertise
- Consistent formatting
- Domain knowledge
- Reliable outputs

## Dataset Formats

### Instruction-Response Format (SFT)

```json
{
  "messages": [
    {"role": "system", "content": "You are a helpful coding assistant."},
    {"role": "user", "content": "Write a function to reverse a string"},
    {"role": "assistant", "content": "def reverse_string(s):\n    return s[::-1]"}
  ]
}
```

### Preference Format (DPO/RLHF)

```json
{
  "prompt": "Explain quantum computing",
  "chosen": "Quantum computing uses quantum bits (qubits) that can exist in superposition...",
  "rejected": "Quantum computing is when computers use quantum physics to go really fast..."
}
```

## Synthetic Data Generation

### Basic Generation Pipeline

```python
from openai import OpenAI
import json

client = OpenAI()

class SyntheticDataGenerator:
    def __init__(self, domain: str, task: str):
        self.domain = domain
        self.task = task

    def generate_examples(self, n: int, seed_examples: list = None) -> list:
        """Generate synthetic training examples."""
        prompt = f"""Generate {n} high-quality training examples for:
Domain: {self.domain}
Task: {self.task}

Requirements:
1. Diverse inputs covering edge cases
2. Accurate, helpful responses
3. Consistent formatting
4. Varying complexity levels

{"Seed examples for style reference:\n" + json.dumps(seed_examples, indent=2) if seed_examples else ""}

Return as JSON array with 'input' and 'output' fields."""

        response = client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"},
            temperature=0.8  # Higher for diversity
        )

        return json.loads(response.choices[0].message.content)["examples"]

# Usage
generator = SyntheticDataGenerator(
    domain="customer support",
    task="answer product questions"
)

examples = generator.generate_examples(
    n=50,
    seed_examples=[
        {
            "input": "How do I reset my password?",
            "output": "To reset your password:\n1. Go to login page\n2. Click 'Forgot Password'\n3. Enter your email\n4. Check your inbox for reset link"
        }
    ]
)
```

### Three Strategies for SFT Data

```python
class SFTDataStrategies:
    def __init__(self):
        self.client = OpenAI()

    def answer_augmentation(self, question: str, base_answer: str) -> list:
        """Generate multiple answer variations for same question."""
        prompt = f"""Given this Q&A pair, generate 5 alternative answers.
        Each should be correct but vary in:
        - Level of detail
        - Tone (formal/casual)
        - Structure (bullets/prose)

        Question: {question}
        Base Answer: {base_answer}

        Return as JSON array."""

        response = self.client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"}
        )

        return json.loads(response.choices[0].message.content)["answers"]

    def question_rephrase(self, question: str, answer: str) -> list:
        """Generate question variations for same answer."""
        prompt = f"""Generate 5 different ways to ask this question.
        The answer should remain the same.

        Original: {question}
        Answer: {answer}

        Vary by:
        - Phrasing
        - Specificity
        - User expertise level

        Return as JSON array."""

        response = self.client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"}
        )

        return json.loads(response.choices[0].message.content)["questions"]

    def new_question_generation(self, topic: str, n: int) -> list:
        """Generate entirely new Q&A pairs for a topic."""
        prompt = f"""Generate {n} unique Q&A pairs about: {topic}

        Cover:
        - Basic concepts
        - Common problems
        - Edge cases
        - Best practices

        Return as JSON array with 'question' and 'answer' fields."""

        response = self.client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"}
        )

        return json.loads(response.choices[0].message.content)["pairs"]
```

### Evolutionary Data Generation

```python
import random

class EvolutionaryGenerator:
    """Generate increasingly complex examples through evolution."""

    def __init__(self):
        self.client = OpenAI()

    def evolve_dataset(
        self,
        seed_examples: list,
        generations: int = 5,
        population_size: int = 20
    ) -> list:
        """Evolve dataset through multiple generations."""
        population = seed_examples.copy()
        all_examples = seed_examples.copy()

        for gen in range(generations):
            # Generate new examples inspired by current population
            new_examples = self.generate_offspring(
                population,
                population_size
            )

            # Score and filter
            scored = self.score_examples(new_examples)
            top_examples = sorted(scored, key=lambda x: x["score"], reverse=True)
            top_examples = top_examples[:population_size // 2]

            # Add to population
            population = [e["example"] for e in top_examples]
            all_examples.extend(population)

            print(f"Generation {gen + 1}: {len(population)} examples, avg score: {sum(e['score'] for e in top_examples) / len(top_examples):.2f}")

        return all_examples

    def generate_offspring(self, parents: list, n: int) -> list:
        """Generate new examples inspired by parents."""
        prompt = f"""Based on these example Q&A pairs, generate {n} new ones.

        Make the new examples:
        1. More complex or nuanced
        2. Cover related but different scenarios
        3. Include edge cases

        Parent examples:
        {json.dumps(random.sample(parents, min(5, len(parents))), indent=2)}

        Return as JSON array."""

        response = self.client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"},
            temperature=0.9
        )

        return json.loads(response.choices[0].message.content)["examples"]

    def score_examples(self, examples: list) -> list:
        """Score examples for quality."""
        scored = []
        for example in examples:
            score = self.evaluate_example(example)
            scored.append({"example": example, "score": score})
        return scored

    def evaluate_example(self, example: dict) -> float:
        """Evaluate single example quality."""
        prompt = f"""Rate this training example quality (0-10):

        Question: {example.get('question', example.get('input'))}
        Answer: {example.get('answer', example.get('output'))}

        Score based on:
        - Clarity of question
        - Accuracy of answer
        - Helpfulness
        - Completeness

        Return just the numeric score."""

        response = self.client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            temperature=0
        )

        try:
            return float(response.choices[0].message.content.strip())
        except:
            return 5.0
```

## Quality Assurance

### LLM-as-Judge Validation

```python
class DataValidator:
    def __init__(self):
        self.client = OpenAI()

    def validate_example(self, example: dict) -> dict:
        """Validate a single training example."""
        prompt = f"""Evaluate this training example:

        Input: {example['input']}
        Output: {example['output']}

        Check for:
        1. Factual accuracy
        2. Completeness
        3. Appropriate tone
        4. Formatting consistency
        5. Potential bias

        Return JSON with:
        - valid: boolean
        - score: 0-10
        - issues: list of problems
        - suggestions: improvements"""

        response = self.client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"}
        )

        return json.loads(response.choices[0].message.content)

    def validate_dataset(self, dataset: list, sample_size: int = 100) -> dict:
        """Validate a dataset with sampling."""
        import random

        sample = random.sample(dataset, min(sample_size, len(dataset)))
        results = []

        for example in sample:
            validation = self.validate_example(example)
            results.append(validation)

        # Aggregate results
        valid_count = sum(1 for r in results if r["valid"])
        avg_score = sum(r["score"] for r in results) / len(results)

        # Common issues
        all_issues = []
        for r in results:
            all_issues.extend(r.get("issues", []))

        from collections import Counter
        issue_counts = Counter(all_issues)

        return {
            "total_sampled": len(sample),
            "valid_count": valid_count,
            "valid_rate": valid_count / len(sample),
            "average_score": avg_score,
            "common_issues": issue_counts.most_common(10)
        }
```

### Deduplication

```python
from sentence_transformers import SentenceTransformer
import numpy as np

class Deduplicator:
    def __init__(self, threshold: float = 0.9):
        self.model = SentenceTransformer('all-MiniLM-L6-v2')
        self.threshold = threshold

    def deduplicate(self, examples: list) -> list:
        """Remove near-duplicate examples."""
        # Get embeddings for inputs
        texts = [e.get('input', e.get('question', '')) for e in examples]
        embeddings = self.model.encode(texts)

        # Find duplicates
        keep_indices = []
        for i in range(len(examples)):
            is_duplicate = False

            for j in keep_indices:
                similarity = np.dot(embeddings[i], embeddings[j]) / (
                    np.linalg.norm(embeddings[i]) * np.linalg.norm(embeddings[j])
                )

                if similarity > self.threshold:
                    is_duplicate = True
                    break

            if not is_duplicate:
                keep_indices.append(i)

        deduplicated = [examples[i] for i in keep_indices]
        print(f"Removed {len(examples) - len(deduplicated)} duplicates")

        return deduplicated
```

### Diversity Scoring

```python
class DiversityScorer:
    def __init__(self):
        self.model = SentenceTransformer('all-MiniLM-L6-v2')

    def score_diversity(self, examples: list) -> float:
        """Score dataset diversity (0-1)."""
        texts = [e.get('input', e.get('question', '')) for e in examples]
        embeddings = self.model.encode(texts)

        # Calculate pairwise distances
        from scipy.spatial.distance import pdist
        distances = pdist(embeddings, metric='cosine')

        # Average distance = diversity score
        return float(np.mean(distances))

    def find_gaps(self, examples: list, target_topics: list) -> list:
        """Find topic gaps in dataset."""
        # Embed examples and topics
        example_texts = [e.get('input', '') for e in examples]
        example_embeddings = self.model.encode(example_texts)
        topic_embeddings = self.model.encode(target_topics)

        gaps = []
        for i, topic in enumerate(target_topics):
            # Find max similarity to any example
            similarities = np.dot(example_embeddings, topic_embeddings[i])
            max_sim = np.max(similarities)

            if max_sim < 0.7:  # Topic not well covered
                gaps.append({
                    "topic": topic,
                    "coverage": float(max_sim)
                })

        return sorted(gaps, key=lambda x: x["coverage"])
```

## Preference Data Generation

For DPO/RLHF training:

```python
class PreferenceDataGenerator:
    def __init__(self):
        self.client = OpenAI()

    def generate_pairs(self, prompts: list) -> list:
        """Generate chosen/rejected pairs for DPO."""
        pairs = []

        for prompt in prompts:
            # Generate good response
            chosen = self.generate_response(prompt, quality="high")

            # Generate worse response
            rejected = self.generate_response(prompt, quality="low")

            pairs.append({
                "prompt": prompt,
                "chosen": chosen,
                "rejected": rejected
            })

        return pairs

    def generate_response(self, prompt: str, quality: str) -> str:
        """Generate response of specified quality."""
        if quality == "high":
            instruction = """Provide an excellent response that is:
            - Accurate and complete
            - Well-structured
            - Helpful and clear
            - Professional tone"""
        else:
            instruction = """Provide a mediocre response that has some issues:
            - Missing some details
            - Less clear structure
            - Slightly verbose
            - Could be more helpful
            (Don't make it terrible, just noticeably worse)"""

        response = self.client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": instruction},
                {"role": "user", "content": prompt}
            ]
        )

        return response.choices[0].message.content

    def validate_pair(self, pair: dict) -> bool:
        """Verify chosen is actually better than rejected."""
        prompt = f"""Compare these two responses:

        Prompt: {pair['prompt']}

        Response A:
        {pair['chosen']}

        Response B:
        {pair['rejected']}

        Which is better? Return "A" or "B"."""

        response = self.client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}],
            temperature=0
        )

        return "A" in response.choices[0].message.content
```

## Complete Pipeline

```python
class DatasetPipeline:
    def __init__(self, config: dict):
        self.generator = SyntheticDataGenerator(
            domain=config["domain"],
            task=config["task"]
        )
        self.validator = DataValidator()
        self.deduplicator = Deduplicator()
        self.diversity_scorer = DiversityScorer()

    def create_dataset(
        self,
        target_size: int,
        seed_examples: list = None,
        min_quality_score: float = 7.0
    ) -> list:
        """Create a high-quality dataset."""
        dataset = []
        attempts = 0
        max_attempts = target_size * 3

        while len(dataset) < target_size and attempts < max_attempts:
            # Generate batch
            batch_size = min(50, target_size - len(dataset))
            batch = self.generator.generate_examples(batch_size, seed_examples)

            # Validate
            for example in batch:
                validation = self.validator.validate_example(example)

                if validation["valid"] and validation["score"] >= min_quality_score:
                    dataset.append(example)

            attempts += batch_size
            print(f"Progress: {len(dataset)}/{target_size} examples")

        # Deduplicate
        dataset = self.deduplicator.deduplicate(dataset)

        # Check diversity
        diversity = self.diversity_scorer.score_diversity(dataset)
        print(f"Dataset diversity score: {diversity:.2f}")

        return dataset

    def save_dataset(self, dataset: list, path: str, format: str = "jsonl"):
        """Save dataset to file."""
        if format == "jsonl":
            with open(path, "w") as f:
                for example in dataset:
                    f.write(json.dumps(example) + "\n")
        elif format == "json":
            with open(path, "w") as f:
                json.dump(dataset, f, indent=2)

        print(f"Saved {len(dataset)} examples to {path}")

# Usage
pipeline = DatasetPipeline({
    "domain": "Python programming",
    "task": "answer coding questions"
})

dataset = pipeline.create_dataset(
    target_size=1000,
    seed_examples=[
        {"input": "How do I read a file?", "output": "..."},
        {"input": "What is a list comprehension?", "output": "..."}
    ]
)

pipeline.save_dataset(dataset, "training_data.jsonl")
```

## Best Practices

1. **Start with real examples**: Use actual user queries as seeds
2. **Validate thoroughly**: LLM-as-judge + human spot checks
3. **Ensure diversity**: Cover edge cases and varying complexity
4. **Deduplicate**: Near-duplicates hurt training
5. **Balance dataset**: Don't over-represent any category
6. **Version control**: Track dataset changes
7. **Document sources**: Note what's synthetic vs real

## Resources

- [Scale AI Synthetic Data Strategies](https://scale.com/blog/synthetic-data-fine-tuning-llms)
- [Hugging Face Synthetic Data Guide](https://huggingface.co/blog/synthetic-data-save-costs)
- [NVIDIA Nemotron Pipeline](https://blogs.nvidia.com/blog/nemotron-4-synthetic-data-generation-llm-training/)
- [mlabonne/llm-datasets](https://github.com/mlabonne/llm-datasets)
- [Gretel Navigator](https://gretel.ai/blog/how-to-create-high-quality-synthetic-data-for-fine-tuning-llms)

---

*Questions about creating training datasets? [Let me know](mailto:jordan@jordananderson.us).*
