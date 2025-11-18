---
layout: default
title:  "CI/CD Pipelines for Machine Learning Models"
date:   2025-11-18 12:00:00
categories: DevOps MLOps CICD MachineLearning
---

ML CI/CD extends traditional pipelines with model validation, data versioning, and automated retraining. Here's how to build production-grade ML pipelines.

## ML Pipeline Architecture

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Code      │    │   Data      │    │   Model     │
│   Commit    │───>│  Validation │───>│  Training   │
└─────────────┘    └─────────────┘    └─────────────┘
                                             │
┌─────────────┐    ┌─────────────┐    ┌──────▼──────┐
│  Deploy     │<───│  Registry   │<───│  Evaluation │
│  to Prod    │    │  Push       │    │  & Testing  │
└─────────────┘    └─────────────┘    └─────────────┘
```

## GitHub Actions for ML

### Complete ML Pipeline

```yaml
# .github/workflows/ml-pipeline.yml
name: ML Model Pipeline

on:
  push:
    branches: [main]
    paths:
      - 'models/**'
      - 'training/**'
      - 'data/**'
  schedule:
    - cron: '0 2 * * 1'  # Weekly retraining

env:
  MODEL_NAME: recommendation-model
  REGISTRY: ghcr.io/${{ github.repository }}

jobs:
  data-validation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Pull data from DVC
        run: |
          dvc pull data/training
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Validate data schema
        run: python scripts/validate_data.py

      - name: Check data quality
        run: |
          python -c "
          import great_expectations as ge
          context = ge.get_context()
          result = context.run_checkpoint('training_data_checkpoint')
          if not result.success:
              raise Exception('Data validation failed')
          "

      - name: Check for data drift
        run: python scripts/detect_drift.py

  train:
    needs: data-validation
    runs-on: [self-hosted, gpu]
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Pull data
        run: dvc pull data/training
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Train model
        run: |
          python training/train.py \
            --config configs/production.yaml \
            --output models/trained
        env:
          MLFLOW_TRACKING_URI: ${{ secrets.MLFLOW_URI }}
          MLFLOW_EXPERIMENT_NAME: ${{ env.MODEL_NAME }}

      - name: Upload model artifact
        uses: actions/upload-artifact@v3
        with:
          name: trained-model
          path: models/trained/

  evaluate:
    needs: train
    runs-on: ubuntu-latest
    outputs:
      passed: ${{ steps.evaluate.outputs.passed }}
      metrics: ${{ steps.evaluate.outputs.metrics }}
    steps:
      - uses: actions/checkout@v4

      - name: Download model
        uses: actions/download-artifact@v3
        with:
          name: trained-model
          path: models/trained/

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run evaluation
        id: evaluate
        run: |
          python evaluation/evaluate.py \
            --model models/trained \
            --test-data data/test \
            --output evaluation_results.json

          # Check if model passes thresholds
          python -c "
          import json
          with open('evaluation_results.json') as f:
              results = json.load(f)

          passed = (
              results['accuracy'] >= 0.95 and
              results['latency_p99_ms'] <= 100 and
              results['f1_score'] >= 0.90
          )

          print(f'passed={str(passed).lower()}')
          print(f'metrics={json.dumps(results)}')
          " >> $GITHUB_OUTPUT

      - name: Upload evaluation results
        uses: actions/upload-artifact@v3
        with:
          name: evaluation-results
          path: evaluation_results.json

      - name: Comment PR with metrics
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v6
        with:
          script: |
            const metrics = ${{ steps.evaluate.outputs.metrics }};
            const body = `## Model Evaluation Results

            | Metric | Value | Threshold |
            |--------|-------|-----------|
            | Accuracy | ${metrics.accuracy.toFixed(4)} | >= 0.95 |
            | F1 Score | ${metrics.f1_score.toFixed(4)} | >= 0.90 |
            | Latency (P99) | ${metrics.latency_p99_ms}ms | <= 100ms |

            **Status**: ${{ steps.evaluate.outputs.passed == 'true' && '✅ Passed' || '❌ Failed' }}`;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: body
            });

  register:
    needs: evaluate
    if: needs.evaluate.outputs.passed == 'true'
    runs-on: ubuntu-latest
    outputs:
      model_version: ${{ steps.register.outputs.version }}
    steps:
      - uses: actions/checkout@v4

      - name: Download model
        uses: actions/download-artifact@v3
        with:
          name: trained-model
          path: models/trained/

      - name: Download evaluation results
        uses: actions/download-artifact@v3
        with:
          name: evaluation-results

      - name: Register model in MLflow
        id: register
        run: |
          VERSION=$(python scripts/register_model.py \
            --model-path models/trained \
            --model-name ${{ env.MODEL_NAME }} \
            --metrics evaluation_results.json)
          echo "version=$VERSION" >> $GITHUB_OUTPUT
        env:
          MLFLOW_TRACKING_URI: ${{ secrets.MLFLOW_URI }}

      - name: Build and push container
        run: |
          docker build -t ${{ env.REGISTRY }}/${{ env.MODEL_NAME }}:${{ steps.register.outputs.version }} .
          docker push ${{ env.REGISTRY }}/${{ env.MODEL_NAME }}:${{ steps.register.outputs.version }}

  deploy-staging:
    needs: register
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to staging
        run: |
          kubectl set image deployment/ml-inference \
            ml-inference=${{ env.REGISTRY }}/${{ env.MODEL_NAME }}:${{ needs.register.outputs.model_version }} \
            -n ml-staging

      - name: Run smoke tests
        run: |
          python tests/smoke_tests.py \
            --endpoint https://ml-staging.example.com/predict

      - name: Run shadow traffic test
        run: |
          python tests/shadow_test.py \
            --duration 30m \
            --traffic-percent 10

  deploy-production:
    needs: [register, deploy-staging]
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Deploy canary
        run: |
          kubectl apply -f - <<EOF
          apiVersion: flagger.app/v1beta1
          kind: Canary
          metadata:
            name: ml-inference
            namespace: ml-production
          spec:
            targetRef:
              apiVersion: apps/v1
              kind: Deployment
              name: ml-inference
            analysis:
              interval: 1m
              threshold: 5
              maxWeight: 50
              stepWeight: 10
          EOF

      - name: Monitor rollout
        run: |
          kubectl -n ml-production wait canary/ml-inference \
            --for=condition=Promoted \
            --timeout=30m
```

## Data Versioning with DVC

### DVC Configuration

```yaml
# dvc.yaml
stages:
  prepare:
    cmd: python src/prepare.py
    deps:
      - src/prepare.py
      - data/raw
    outs:
      - data/processed

  train:
    cmd: python src/train.py
    deps:
      - src/train.py
      - data/processed
    params:
      - train.epochs
      - train.learning_rate
      - train.batch_size
    outs:
      - models/model.pt
    metrics:
      - metrics.json:
          cache: false

  evaluate:
    cmd: python src/evaluate.py
    deps:
      - src/evaluate.py
      - models/model.pt
      - data/test
    metrics:
      - evaluation.json:
          cache: false
```

### Pipeline Commands

```bash
# Initialize DVC
dvc init
dvc remote add -d myremote s3://my-bucket/dvc

# Add data
dvc add data/training
git add data/training.dvc .gitignore
git commit -m "Add training data v1"

# Run pipeline
dvc repro

# Push data and models
dvc push

# Compare experiments
dvc metrics diff

# Create experiment
dvc exp run -n exp-lr-0.001 --set-param train.learning_rate=0.001
```

## Model Testing

### Unit Tests

```python
# tests/test_model.py
import pytest
import torch
from models.recommendation import RecommendationModel

class TestRecommendationModel:
    @pytest.fixture
    def model(self):
        return RecommendationModel.load("models/trained")

    def test_model_output_shape(self, model):
        """Test model produces correct output shape"""
        batch_size = 32
        input_features = torch.randn(batch_size, 128)

        output = model(input_features)

        assert output.shape == (batch_size, 10)

    def test_model_deterministic(self, model):
        """Test model produces deterministic outputs"""
        input_features = torch.randn(1, 128)

        output1 = model(input_features)
        output2 = model(input_features)

        assert torch.allclose(output1, output2)

    def test_model_handles_edge_cases(self, model):
        """Test model handles edge case inputs"""
        # Empty features
        with pytest.raises(ValueError):
            model(torch.tensor([]))

        # Wrong dimensions
        with pytest.raises(ValueError):
            model(torch.randn(32, 64))

    @pytest.mark.parametrize("batch_size", [1, 16, 64, 128])
    def test_various_batch_sizes(self, model, batch_size):
        """Test model handles various batch sizes"""
        input_features = torch.randn(batch_size, 128)
        output = model(input_features)
        assert output.shape[0] == batch_size
```

### Integration Tests

```python
# tests/test_integration.py
import pytest
import requests
import time

class TestModelServing:
    BASE_URL = "http://localhost:8080"

    @pytest.fixture(scope="class")
    def server(self):
        """Start model server for testing"""
        import subprocess
        proc = subprocess.Popen(["python", "serve.py"])
        time.sleep(5)  # Wait for server
        yield
        proc.terminate()

    def test_health_endpoint(self, server):
        """Test health check endpoint"""
        response = requests.get(f"{self.BASE_URL}/health")
        assert response.status_code == 200
        assert response.json()["status"] == "healthy"

    def test_prediction_endpoint(self, server):
        """Test prediction endpoint"""
        payload = {
            "user_id": "user_123",
            "context": {"time_of_day": "morning"}
        }

        response = requests.post(
            f"{self.BASE_URL}/predict",
            json=payload
        )

        assert response.status_code == 200
        result = response.json()
        assert "predictions" in result
        assert len(result["predictions"]) == 10

    def test_latency_requirement(self, server):
        """Test latency meets requirements"""
        payload = {"user_id": "user_123", "context": {}}

        latencies = []
        for _ in range(100):
            start = time.time()
            requests.post(f"{self.BASE_URL}/predict", json=payload)
            latencies.append((time.time() - start) * 1000)

        p99 = sorted(latencies)[98]
        assert p99 < 100, f"P99 latency {p99}ms exceeds 100ms"

    def test_concurrent_requests(self, server):
        """Test handling concurrent requests"""
        import concurrent.futures

        def make_request():
            payload = {"user_id": f"user_{time.time()}", "context": {}}
            return requests.post(f"{self.BASE_URL}/predict", json=payload)

        with concurrent.futures.ThreadPoolExecutor(max_workers=50) as executor:
            futures = [executor.submit(make_request) for _ in range(100)]
            results = [f.result() for f in futures]

        success_rate = sum(1 for r in results if r.status_code == 200) / len(results)
        assert success_rate >= 0.99
```

### Performance Benchmarks

```python
# tests/benchmark.py
import pytest
from pytest_benchmark.fixture import BenchmarkFixture
import torch
from models.recommendation import RecommendationModel

class TestModelPerformance:
    @pytest.fixture
    def model(self):
        return RecommendationModel.load("models/trained")

    def test_inference_latency(self, benchmark: BenchmarkFixture, model):
        """Benchmark inference latency"""
        input_features = torch.randn(1, 128)

        result = benchmark(model, input_features)

        # Assert p99 latency
        assert benchmark.stats['mean'] < 0.01  # 10ms

    def test_batch_throughput(self, benchmark: BenchmarkFixture, model):
        """Benchmark batch throughput"""
        batch = torch.randn(64, 128)

        benchmark(model, batch)

        # Calculate throughput
        items_per_second = 64 / benchmark.stats['mean']
        assert items_per_second > 1000  # >1000 items/sec
```

## Model Registry

### MLflow Registration

```python
# scripts/register_model.py
import mlflow
from mlflow.tracking import MlflowClient
import json
import argparse

def register_model(model_path: str, model_name: str,
                   metrics_path: str) -> str:
    """Register model in MLflow"""
    client = MlflowClient()

    # Load metrics
    with open(metrics_path) as f:
        metrics = json.load(f)

    # Log model with metrics
    with mlflow.start_run() as run:
        # Log metrics
        for key, value in metrics.items():
            mlflow.log_metric(key, value)

        # Log model
        mlflow.pytorch.log_model(
            pytorch_model=model_path,
            artifact_path="model",
            registered_model_name=model_name
        )

    # Get latest version
    versions = client.get_latest_versions(model_name)
    latest_version = max(v.version for v in versions)

    # Transition to staging
    client.transition_model_version_stage(
        name=model_name,
        version=latest_version,
        stage="Staging"
    )

    return latest_version

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--model-path", required=True)
    parser.add_argument("--model-name", required=True)
    parser.add_argument("--metrics", required=True)
    args = parser.parse_args()

    version = register_model(
        args.model_path,
        args.model_name,
        args.metrics
    )
    print(version)
```

## Automated Retraining

### Trigger Conditions

```python
# monitoring/retrain_trigger.py
from dataclasses import dataclass
from datetime import datetime, timedelta

@dataclass
class RetrainTrigger:
    accuracy_threshold: float = 0.90
    drift_threshold: float = 0.1
    max_age_days: int = 30

class RetrainMonitor:
    def __init__(self, trigger: RetrainTrigger):
        self.trigger = trigger

    def should_retrain(self, metrics: dict) -> tuple[bool, str]:
        """Determine if model should be retrained"""
        reasons = []

        # Check accuracy degradation
        if metrics['current_accuracy'] < self.trigger.accuracy_threshold:
            reasons.append(
                f"Accuracy {metrics['current_accuracy']:.2f} below "
                f"threshold {self.trigger.accuracy_threshold}"
            )

        # Check data drift
        if metrics['drift_score'] > self.trigger.drift_threshold:
            reasons.append(
                f"Data drift {metrics['drift_score']:.2f} exceeds "
                f"threshold {self.trigger.drift_threshold}"
            )

        # Check model age
        model_age = datetime.now() - metrics['model_trained_at']
        if model_age > timedelta(days=self.trigger.max_age_days):
            reasons.append(
                f"Model age {model_age.days} days exceeds "
                f"max {self.trigger.max_age_days} days"
            )

        return len(reasons) > 0, "; ".join(reasons)

    def trigger_retrain(self, reason: str):
        """Trigger retraining pipeline"""
        # GitHub Actions workflow dispatch
        import requests
        requests.post(
            "https://api.github.com/repos/org/repo/dispatches",
            headers={
                "Authorization": f"token {GITHUB_TOKEN}",
                "Accept": "application/vnd.github.v3+json"
            },
            json={
                "event_type": "retrain",
                "client_payload": {"reason": reason}
            }
        )
```

## Best Practices

1. **Version everything**: Code, data, models, configs
2. **Automate validation**: Schema, quality, drift checks
3. **Test thoroughly**: Unit, integration, performance
4. **Progressive rollout**: Canary deployments
5. **Monitor continuously**: Trigger retraining on drift
6. **Reproducibility**: Lock dependencies, log params
7. **Artifact management**: Use model registry
8. **Security**: Scan containers, sign artifacts

## Resources

- [MLOps Principles](https://ml-ops.org/)
- [DVC Documentation](https://dvc.org/doc)
- [MLflow Model Registry](https://mlflow.org/docs/latest/model-registry.html)
- [Google MLOps Whitepaper](https://cloud.google.com/resources/mlops-whitepaper)

---

*Questions about ML CI/CD? [Let me know](mailto:jordan@jordananderson.us).*
