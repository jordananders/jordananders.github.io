---
layout: default
title:  "MLOps: Building Production ML Pipelines"
date:   2025-11-18 09:00:00
categories: DevOps MLOps AI Kubernetes
---

MLOps bridges the gap between data science experimentation and production deployment. Organizations using proper MLOps practices report 32% lower deployment time and 40% faster experimentation cycles. Here's how to build production ML pipelines.

## MLOps Architecture

```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│   Feature   │──▶│   Model     │──▶│   Model     │──▶│   Model     │
│   Store     │   │   Training  │   │   Registry  │   │   Serving   │
└─────────────┘   └─────────────┘   └─────────────┘   └─────────────┘
       │                 │                 │                 │
       └─────────────────┴────────┬────────┴─────────────────┘
                                  │
                         ┌────────▼────────┐
                         │   Monitoring    │
                         │   & Feedback    │
                         └─────────────────┘
```

## Tool Selection: Kubeflow vs MLflow

### When to Use Kubeflow

- Kubernetes-native environments
- Large-scale distributed training
- Complex multi-step pipelines
- Need for GPU orchestration

### When to Use MLflow

- Experiment tracking focus
- Simpler setup requirements
- Smaller teams
- Model registry and versioning

### Using Both Together

```python
# Kubeflow pipeline with MLflow tracking
from kfp import dsl
import mlflow

@dsl.component
def train_model(data_path: str, model_name: str):
    import mlflow
    import mlflow.sklearn
    from sklearn.ensemble import RandomForestClassifier

    mlflow.set_tracking_uri("http://mlflow-server:5000")
    mlflow.set_experiment("my-experiment")

    with mlflow.start_run():
        # Load data and train
        model = RandomForestClassifier(n_estimators=100)
        model.fit(X_train, y_train)

        # Log metrics
        mlflow.log_metric("accuracy", accuracy)
        mlflow.log_param("n_estimators", 100)

        # Register model
        mlflow.sklearn.log_model(
            model,
            "model",
            registered_model_name=model_name
        )
```

## Kubeflow Pipelines

### Pipeline Definition

```python
from kfp import dsl, compiler
from kfp.dsl import Input, Output, Dataset, Model

@dsl.component(base_image="python:3.11")
def preprocess_data(
    raw_data: Input[Dataset],
    processed_data: Output[Dataset]
):
    import pandas as pd

    df = pd.read_csv(raw_data.path)
    # Preprocessing logic
    df_processed = df.dropna()
    df_processed.to_csv(processed_data.path, index=False)

@dsl.component(base_image="python:3.11")
def train_model(
    training_data: Input[Dataset],
    model: Output[Model],
    epochs: int = 10
):
    import joblib
    from sklearn.ensemble import GradientBoostingClassifier

    # Train model
    clf = GradientBoostingClassifier(n_estimators=epochs)
    clf.fit(X, y)

    joblib.dump(clf, model.path)

@dsl.component(base_image="python:3.11")
def evaluate_model(
    model: Input[Model],
    test_data: Input[Dataset]
) -> float:
    import joblib

    clf = joblib.load(model.path)
    accuracy = clf.score(X_test, y_test)
    return accuracy

@dsl.pipeline(name="ml-training-pipeline")
def ml_pipeline(epochs: int = 10):
    preprocess_task = preprocess_data(raw_data=raw_dataset)

    train_task = train_model(
        training_data=preprocess_task.outputs["processed_data"],
        epochs=epochs
    )

    evaluate_task = evaluate_model(
        model=train_task.outputs["model"],
        test_data=test_dataset
    )

# Compile pipeline
compiler.Compiler().compile(ml_pipeline, "pipeline.yaml")
```

### Running on Kubeflow

```python
from kfp.client import Client

client = Client(host="https://kubeflow.example.com")

# Submit pipeline run
run = client.create_run_from_pipeline_func(
    ml_pipeline,
    arguments={"epochs": 20},
    experiment_name="training-experiments"
)
```

## MLflow Tracking

### Experiment Tracking

```python
import mlflow
from mlflow.tracking import MlflowClient

mlflow.set_tracking_uri("http://mlflow:5000")
mlflow.set_experiment("recommendation-model")

with mlflow.start_run(run_name="v1.2.0"):
    # Log parameters
    mlflow.log_params({
        "learning_rate": 0.01,
        "batch_size": 32,
        "epochs": 100
    })

    # Train model
    for epoch in range(100):
        loss = train_epoch(model, data)
        mlflow.log_metric("loss", loss, step=epoch)

    # Log model
    mlflow.pytorch.log_model(model, "model")

    # Log artifacts
    mlflow.log_artifact("confusion_matrix.png")
```

### Model Registry

```python
from mlflow.tracking import MlflowClient

client = MlflowClient()

# Register model
model_uri = f"runs:/{run_id}/model"
mv = client.create_model_version(
    name="recommendation-model",
    source=model_uri,
    run_id=run_id
)

# Transition to production
client.transition_model_version_stage(
    name="recommendation-model",
    version=mv.version,
    stage="Production"
)

# Load production model
model = mlflow.pyfunc.load_model(
    "models:/recommendation-model/Production"
)
```

## Feature Store with Feast

```python
from feast import FeatureStore, Entity, FeatureView, Field
from feast.types import Float32, Int64

# Define entity
user = Entity(
    name="user_id",
    join_keys=["user_id"]
)

# Define feature view
user_features = FeatureView(
    name="user_features",
    entities=[user],
    schema=[
        Field(name="age", dtype=Int64),
        Field(name="total_purchases", dtype=Float32),
        Field(name="avg_order_value", dtype=Float32)
    ],
    source=BigQuerySource(
        table="project.dataset.user_features"
    )
)

# Get features for training
store = FeatureStore(repo_path=".")

training_df = store.get_historical_features(
    entity_df=entity_df,
    features=[
        "user_features:age",
        "user_features:total_purchases",
        "user_features:avg_order_value"
    ]
).to_df()

# Get features for inference
feature_vector = store.get_online_features(
    features=[
        "user_features:age",
        "user_features:total_purchases"
    ],
    entity_rows=[{"user_id": 123}]
).to_dict()
```

## Model Serving with Seldon

```yaml
apiVersion: machinelearning.seldon.io/v1
kind: SeldonDeployment
metadata:
  name: recommendation-model
spec:
  predictors:
    - name: default
      replicas: 3
      graph:
        name: classifier
        implementation: SKLEARN_SERVER
        modelUri: s3://models/recommendation/v1
        envSecretRefName: s3-credentials
      componentSpecs:
        - spec:
            containers:
              - name: classifier
                resources:
                  requests:
                    memory: "1Gi"
                    cpu: "1"
                  limits:
                    memory: "2Gi"
                    cpu: "2"
```

### Canary Deployment

```yaml
apiVersion: machinelearning.seldon.io/v1
kind: SeldonDeployment
metadata:
  name: recommendation-model
spec:
  predictors:
    - name: default
      replicas: 3
      traffic: 90
      graph:
        name: classifier
        modelUri: s3://models/v1
    - name: canary
      replicas: 1
      traffic: 10
      graph:
        name: classifier
        modelUri: s3://models/v2
```

## CI/CD for ML

### GitHub Actions Pipeline

```yaml
name: ML Pipeline

on:
  push:
    paths:
      - 'models/**'
      - 'training/**'

jobs:
  train-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest tests/

      - name: Train model
        env:
          MLFLOW_TRACKING_URI: ${{ secrets.MLFLOW_URI }}
        run: python training/train.py

      - name: Evaluate model
        run: python training/evaluate.py

      - name: Register model
        if: success()
        run: |
          python -c "
          import mlflow
          client = mlflow.tracking.MlflowClient()
          client.transition_model_version_stage(
              name='my-model',
              version='${{ github.run_number }}',
              stage='Staging'
          )
          "

      - name: Deploy to staging
        run: kubectl apply -f k8s/staging/
```

## Monitoring with Evidently

```python
from evidently.report import Report
from evidently.metric_preset import DataDriftPreset, TargetDriftPreset

# Create drift report
report = Report(metrics=[
    DataDriftPreset(),
    TargetDriftPreset()
])

report.run(
    reference_data=training_data,
    current_data=production_data
)

# Save report
report.save_html("drift_report.html")

# Check for drift
drift_detected = report.as_dict()["metrics"][0]["result"]["dataset_drift"]

if drift_detected:
    # Trigger retraining
    trigger_pipeline("retrain")
```

## Best Practices

### 1. Model-as-Code

```python
# Version everything
model_config = {
    "version": "1.2.0",
    "architecture": "transformer",
    "hyperparameters": {
        "learning_rate": 0.001,
        "batch_size": 32
    },
    "training_data": "s3://data/v3/",
    "commit": os.getenv("GIT_COMMIT")
}

mlflow.log_dict(model_config, "model_config.json")
```

### 2. Automated Testing

```python
def test_model_accuracy():
    model = load_model("models:/my-model/Production")
    accuracy = evaluate(model, test_data)
    assert accuracy > 0.85, f"Accuracy {accuracy} below threshold"

def test_model_latency():
    model = load_model("models:/my-model/Production")
    start = time.time()
    model.predict(sample_input)
    latency = time.time() - start
    assert latency < 0.1, f"Latency {latency}s exceeds 100ms"

def test_no_data_drift():
    report = generate_drift_report(reference, current)
    assert not report.drift_detected
```

### 3. Pipeline Triggers

- **On schedule**: Daily/weekly retraining
- **On data availability**: New data arrives
- **On drift detection**: Model performance degrades
- **On demand**: Manual trigger

## Resources

- [Kubeflow Documentation](https://www.kubeflow.org/docs/)
- [MLflow Documentation](https://mlflow.org/docs/latest/index.html)
- [Feast Feature Store](https://docs.feast.dev/)
- [Seldon Core](https://docs.seldon.io/)
- [MLOps Roadmap 2024](https://www.marvelousmlops.io/p/mlops-roadmap-2024)

---

*Questions about MLOps pipelines? [Let me know](mailto:jordan@jordananderson.us).*
