---
layout: default
title:  "Evaluating Fine-Tuned Models"
date:   2025-11-17 22:00:00
categories: AI LLM Fine-tuning Evaluation
---

Fine-tuned models can outperform GPT-4 on specialized tasks, but only with proper evaluation. Here's how to benchmark your fine-tuned models and compare them against baselines.

## Why Evaluation Matters

Fine-tuning without evaluation leads to:
- Overfitting to training data
- Regression on general capabilities
- No proof of improvement
- Wasted compute resources

## Evaluation Framework

### Before vs After Comparison

```python
from dataclasses import dataclass
from typing import Callable

@dataclass
class EvaluationResult:
    model: str
    accuracy: float
    precision: float
    recall: float
    f1: float
    latency_ms: float
    examples_evaluated: int

class ModelEvaluator:
    def __init__(self, test_dataset: list[dict]):
        self.test_data = test_dataset
        self.metrics = {}

    def evaluate(
        self,
        model_fn: Callable,
        model_name: str
    ) -> EvaluationResult:
        """Evaluate model on test dataset."""
        predictions = []
        latencies = []

        for item in self.test_data:
            start = time.time()
            prediction = model_fn(item["input"])
            latency = (time.time() - start) * 1000

            predictions.append({
                "predicted": prediction,
                "expected": item["expected"],
                "correct": self.is_correct(prediction, item["expected"])
            })
            latencies.append(latency)

        return EvaluationResult(
            model=model_name,
            accuracy=self.calculate_accuracy(predictions),
            precision=self.calculate_precision(predictions),
            recall=self.calculate_recall(predictions),
            f1=self.calculate_f1(predictions),
            latency_ms=sum(latencies) / len(latencies),
            examples_evaluated=len(predictions)
        )

    def compare(self, results: list[EvaluationResult]) -> dict:
        """Compare multiple models."""
        baseline = results[0]
        comparisons = []

        for result in results[1:]:
            comparisons.append({
                "model": result.model,
                "accuracy_delta": result.accuracy - baseline.accuracy,
                "latency_delta": result.latency_ms - baseline.latency_ms,
                "improvement": result.accuracy > baseline.accuracy
            })

        return {
            "baseline": baseline.model,
            "comparisons": comparisons
        }
```

## Standard Benchmarks

### Task-Specific Benchmarks

```python
class BenchmarkRunner:
    def __init__(self):
        self.benchmarks = {
            "code": {
                "humaneval": self.run_humaneval,
                "mbpp": self.run_mbpp
            },
            "math": {
                "gsm8k": self.run_gsm8k,
                "math": self.run_math_benchmark
            },
            "reasoning": {
                "arc": self.run_arc,
                "hellaswag": self.run_hellaswag
            },
            "knowledge": {
                "mmlu": self.run_mmlu,
                "triviaqa": self.run_triviaqa
            }
        }

    async def run_humaneval(self, model_fn) -> dict:
        """Run HumanEval code benchmark."""
        from human_eval.data import read_problems
        from human_eval.evaluation import evaluate_functional_correctness

        problems = read_problems()
        samples = []

        for task_id, problem in problems.items():
            completion = await model_fn(problem["prompt"])
            samples.append({
                "task_id": task_id,
                "completion": completion
            })

        results = evaluate_functional_correctness(samples)

        return {
            "benchmark": "humaneval",
            "pass@1": results["pass@1"],
            "pass@10": results.get("pass@10", None),
            "total_problems": len(problems)
        }

    async def run_mmlu(self, model_fn) -> dict:
        """Run MMLU knowledge benchmark."""
        from datasets import load_dataset

        dataset = load_dataset("cais/mmlu", "all")
        correct = 0
        total = 0

        for item in dataset["test"]:
            question = item["question"]
            choices = item["choices"]
            answer = item["answer"]

            prompt = self.format_mmlu_prompt(question, choices)
            prediction = await model_fn(prompt)

            if self.parse_mmlu_answer(prediction) == answer:
                correct += 1
            total += 1

        return {
            "benchmark": "mmlu",
            "accuracy": correct / total,
            "correct": correct,
            "total": total
        }
```

### Custom Domain Benchmark

```python
class CustomBenchmark:
    def __init__(self, name: str, test_cases: list[dict]):
        self.name = name
        self.test_cases = test_cases

    async def run(self, model_fn, evaluator_fn) -> dict:
        """Run custom benchmark with custom evaluator."""
        results = []

        for case in self.test_cases:
            output = await model_fn(case["input"])

            score = await evaluator_fn(
                input=case["input"],
                output=output,
                expected=case.get("expected"),
                criteria=case.get("criteria", [])
            )

            results.append({
                "case_id": case.get("id"),
                "score": score,
                "output": output
            })

        return {
            "benchmark": self.name,
            "average_score": sum(r["score"] for r in results) / len(results),
            "results": results
        }

# Create custom benchmark for your domain
customer_support_benchmark = CustomBenchmark(
    name="customer_support_qa",
    test_cases=[
        {
            "id": "return_policy",
            "input": "What's your return policy?",
            "expected": "30-day return policy",
            "criteria": ["accurate", "helpful", "professional"]
        },
        {
            "id": "shipping_time",
            "input": "How long does shipping take?",
            "expected": "5-7 business days",
            "criteria": ["accurate", "specific", "complete"]
        }
    ]
)
```

## Evaluation Metrics

### Classification Metrics

```python
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score,
    f1_score, confusion_matrix
)

def evaluate_classification(predictions: list, labels: list) -> dict:
    """Evaluate classification task."""
    return {
        "accuracy": accuracy_score(labels, predictions),
        "precision": precision_score(labels, predictions, average="weighted"),
        "recall": recall_score(labels, predictions, average="weighted"),
        "f1": f1_score(labels, predictions, average="weighted"),
        "confusion_matrix": confusion_matrix(labels, predictions).tolist()
    }
```

### Generation Metrics

```python
from rouge_score import rouge_scorer
from sacrebleu.metrics import BLEU

class GenerationMetrics:
    def __init__(self):
        self.rouge_scorer = rouge_scorer.RougeScorer(
            ['rouge1', 'rouge2', 'rougeL'],
            use_stemmer=True
        )
        self.bleu = BLEU()

    def calculate_rouge(self, prediction: str, reference: str) -> dict:
        """Calculate ROUGE scores."""
        scores = self.rouge_scorer.score(reference, prediction)
        return {
            "rouge1": scores["rouge1"].fmeasure,
            "rouge2": scores["rouge2"].fmeasure,
            "rougeL": scores["rougeL"].fmeasure
        }

    def calculate_bleu(self, predictions: list[str], references: list[list[str]]) -> float:
        """Calculate corpus BLEU score."""
        result = self.bleu.corpus_score(predictions, references)
        return result.score

    def calculate_semantic_similarity(self, pred: str, ref: str) -> float:
        """Calculate semantic similarity using embeddings."""
        from sentence_transformers import SentenceTransformer

        model = SentenceTransformer('all-MiniLM-L6-v2')
        embeddings = model.encode([pred, ref])

        similarity = np.dot(embeddings[0], embeddings[1]) / (
            np.linalg.norm(embeddings[0]) * np.linalg.norm(embeddings[1])
        )
        return float(similarity)
```

### LLM-as-Judge

```python
class LLMJudge:
    def __init__(self):
        self.client = OpenAI()

    async def evaluate(
        self,
        input_text: str,
        output: str,
        criteria: list[str],
        reference: str = None
    ) -> dict:
        """Use LLM to evaluate output quality."""
        criteria_text = "\n".join([f"- {c}" for c in criteria])

        prompt = f"""Evaluate this AI response.

Input: {input_text}

Response: {output}

{f"Reference answer: {reference}" if reference else ""}

Evaluate on these criteria (score 1-5 each):
{criteria_text}

Also provide an overall score (1-5) and brief explanation.

Return JSON with scores for each criterion, overall_score, and explanation."""

        response = self.client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"}
        )

        return json.loads(response.choices[0].message.content)

# Usage
judge = LLMJudge()
evaluation = await judge.evaluate(
    input_text="What is machine learning?",
    output=model_response,
    criteria=["accuracy", "clarity", "completeness"],
    reference="Machine learning is..."
)
```

## Holdout Test Sets

### Creating Test Sets

```python
from sklearn.model_selection import train_test_split

def create_evaluation_splits(dataset: list[dict], seed: int = 42) -> dict:
    """Create train/val/test splits."""
    # 80/10/10 split
    train, temp = train_test_split(dataset, test_size=0.2, random_state=seed)
    val, test = train_test_split(temp, test_size=0.5, random_state=seed)

    return {
        "train": train,
        "validation": val,
        "test": test
    }

def create_stratified_test(
    dataset: list[dict],
    stratify_key: str,
    test_size: int = 100
) -> list[dict]:
    """Create stratified test set."""
    from collections import defaultdict

    # Group by category
    groups = defaultdict(list)
    for item in dataset:
        groups[item[stratify_key]].append(item)

    # Sample from each group
    test_set = []
    per_group = test_size // len(groups)

    for category, items in groups.items():
        sampled = random.sample(items, min(per_group, len(items)))
        test_set.extend(sampled)

    return test_set
```

## A/B Evaluation

```python
class ABEvaluator:
    def __init__(self):
        self.judge = LLMJudge()

    async def pairwise_comparison(
        self,
        test_cases: list[dict],
        model_a_fn,
        model_b_fn
    ) -> dict:
        """Compare two models head-to-head."""
        results = {"A_wins": 0, "B_wins": 0, "ties": 0}

        for case in test_cases:
            response_a = await model_a_fn(case["input"])
            response_b = await model_b_fn(case["input"])

            # Randomize order to avoid position bias
            if random.random() > 0.5:
                first, second = response_a, response_b
                first_label, second_label = "A", "B"
            else:
                first, second = response_b, response_a
                first_label, second_label = "B", "A"

            winner = await self.judge_comparison(
                case["input"], first, second
            )

            if winner == "first":
                results[f"{first_label}_wins"] += 1
            elif winner == "second":
                results[f"{second_label}_wins"] += 1
            else:
                results["ties"] += 1

        # Calculate win rate
        total = sum(results.values())
        results["A_win_rate"] = results["A_wins"] / total
        results["B_win_rate"] = results["B_wins"] / total

        return results

    async def judge_comparison(self, input_text: str, response_1: str, response_2: str) -> str:
        """Judge which response is better."""
        prompt = f"""Compare these two responses to the same input.

Input: {input_text}

Response 1:
{response_1}

Response 2:
{response_2}

Which response is better? Consider accuracy, helpfulness, and clarity.
Return only: "first", "second", or "tie"."""

        response = self.client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": prompt}]
        )

        return response.choices[0].message.content.strip().lower()
```

## Regression Testing

Ensure fine-tuning doesn't hurt general capabilities:

```python
class RegressionTester:
    def __init__(self, general_benchmarks: list[str]):
        self.benchmarks = general_benchmarks
        self.baseline_scores = {}

    async def establish_baseline(self, model_fn, model_name: str):
        """Run benchmarks on base model."""
        scores = {}

        for benchmark in self.benchmarks:
            score = await self.run_benchmark(model_fn, benchmark)
            scores[benchmark] = score

        self.baseline_scores[model_name] = scores
        return scores

    async def check_regression(
        self,
        finetuned_fn,
        base_model: str,
        tolerance: float = 0.05
    ) -> dict:
        """Check for regression vs base model."""
        baseline = self.baseline_scores.get(base_model)
        if not baseline:
            raise ValueError(f"No baseline for {base_model}")

        regressions = []

        for benchmark in self.benchmarks:
            new_score = await self.run_benchmark(finetuned_fn, benchmark)
            base_score = baseline[benchmark]

            delta = new_score - base_score

            if delta < -tolerance:
                regressions.append({
                    "benchmark": benchmark,
                    "baseline": base_score,
                    "new": new_score,
                    "delta": delta
                })

        return {
            "has_regression": len(regressions) > 0,
            "regressions": regressions,
            "tolerance": tolerance
        }
```

## Evaluation Pipeline

```python
class FineTuneEvaluationPipeline:
    def __init__(self, config: dict):
        self.config = config
        self.judge = LLMJudge()
        self.metrics = GenerationMetrics()

    async def run_full_evaluation(
        self,
        base_model_fn,
        finetuned_model_fn,
        test_dataset: list[dict]
    ) -> dict:
        """Run complete evaluation pipeline."""
        results = {
            "timestamp": datetime.utcnow().isoformat(),
            "test_size": len(test_dataset)
        }

        # 1. Task-specific evaluation
        print("Running task evaluation...")
        results["task_performance"] = await self.evaluate_task(
            finetuned_model_fn,
            test_dataset
        )

        # 2. Compare to baseline
        print("Comparing to baseline...")
        results["vs_baseline"] = await self.compare_to_baseline(
            base_model_fn,
            finetuned_model_fn,
            test_dataset
        )

        # 3. Check for regression
        print("Checking for regression...")
        results["regression"] = await self.check_regression(
            finetuned_model_fn,
            self.config["base_model"]
        )

        # 4. Quality metrics
        print("Calculating quality metrics...")
        results["quality"] = await self.evaluate_quality(
            finetuned_model_fn,
            test_dataset
        )

        # 5. Generate summary
        results["summary"] = self.generate_summary(results)

        return results

    def generate_summary(self, results: dict) -> dict:
        """Generate evaluation summary."""
        task_score = results["task_performance"]["score"]
        improvement = results["vs_baseline"]["improvement"]
        has_regression = results["regression"]["has_regression"]

        recommendation = "DEPLOY"
        if has_regression:
            recommendation = "REVIEW - Regression detected"
        elif improvement < 0.05:
            recommendation = "REVIEW - Minimal improvement"

        return {
            "task_score": task_score,
            "improvement_vs_baseline": improvement,
            "has_regression": has_regression,
            "recommendation": recommendation
        }

# Usage
pipeline = FineTuneEvaluationPipeline({
    "base_model": "llama-3.1-8b",
    "task": "customer_support"
})

results = await pipeline.run_full_evaluation(
    base_model_fn=base_model,
    finetuned_model_fn=finetuned_model,
    test_dataset=test_set
)

print(f"Recommendation: {results['summary']['recommendation']}")
```

## Best Practices

1. **Hold out test data**: Never evaluate on training data
2. **Use multiple metrics**: Single metric can be misleading
3. **Compare to baseline**: Always measure improvement
4. **Check for regression**: Test general capabilities
5. **Use LLM judges**: For subjective quality assessment
6. **Document evaluation**: Track methodology and results

## Resources

- [Predibase Fine-tuning Index](https://predibase.com/fine-tuning-index)
- [Confident AI Evaluation Guide](https://www.confident-ai.com/blog/llm-evaluation-metrics-everything-you-need-for-llm-evaluation)
- [HumanEval Benchmark](https://github.com/openai/human-eval)
- [lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness)

---

*Questions about evaluating fine-tuned models? [Let me know](mailto:jordan@jordananderson.us).*
