---
layout: default
title:  "Model Monitoring and Drift Detection in Production"
date:   2025-11-18 12:00:00
categories: DevOps MLOps Monitoring DriftDetection
---

Production ML models degrade over time due to data drift and concept drift. Here's how to monitor models and detect when retraining is needed.

## Types of Drift

- **Data drift**: Input distribution changes
- **Concept drift**: Relationship between inputs and outputs changes
- **Prediction drift**: Output distribution changes
- **Performance drift**: Model accuracy degrades

## Drift Detection Framework

```python
# drift_detector.py
from dataclasses import dataclass
from scipy import stats
import numpy as np
from typing import Optional

@dataclass
class DriftResult:
    feature: str
    drift_score: float
    p_value: float
    is_drifted: bool
    drift_type: str

class DriftDetector:
    def __init__(self, threshold: float = 0.05):
        self.threshold = threshold
        self.baseline_stats = {}

    def set_baseline(self, data: np.ndarray, feature_names: list):
        """Set baseline statistics from training data"""
        for i, name in enumerate(feature_names):
            column = data[:, i]
            self.baseline_stats[name] = {
                'mean': np.mean(column),
                'std': np.std(column),
                'distribution': column
            }

    def detect_drift(self, production_data: np.ndarray,
                     feature_names: list) -> list[DriftResult]:
        """Detect drift in production data"""
        results = []

        for i, name in enumerate(feature_names):
            column = production_data[:, i]
            baseline = self.baseline_stats[name]

            # Kolmogorov-Smirnov test
            ks_stat, p_value = stats.ks_2samp(
                baseline['distribution'],
                column
            )

            # Population Stability Index
            psi = self._calculate_psi(baseline['distribution'], column)

            # Jensen-Shannon divergence
            js_div = self._jensen_shannon(baseline['distribution'], column)

            results.append(DriftResult(
                feature=name,
                drift_score=psi,
                p_value=p_value,
                is_drifted=p_value < self.threshold or psi > 0.2,
                drift_type=self._classify_drift(ks_stat, psi)
            ))

        return results

    def _calculate_psi(self, expected: np.ndarray,
                       actual: np.ndarray, bins: int = 10) -> float:
        """Calculate Population Stability Index"""
        # Create bins from expected distribution
        breakpoints = np.percentile(expected, np.linspace(0, 100, bins + 1))
        breakpoints[0] = -np.inf
        breakpoints[-1] = np.inf

        # Calculate proportions
        expected_counts = np.histogram(expected, bins=breakpoints)[0]
        actual_counts = np.histogram(actual, bins=breakpoints)[0]

        expected_props = expected_counts / len(expected)
        actual_props = actual_counts / len(actual)

        # Avoid division by zero
        expected_props = np.clip(expected_props, 0.0001, None)
        actual_props = np.clip(actual_props, 0.0001, None)

        # PSI calculation
        psi = np.sum((actual_props - expected_props) *
                     np.log(actual_props / expected_props))

        return psi

    def _jensen_shannon(self, p: np.ndarray, q: np.ndarray) -> float:
        """Calculate Jensen-Shannon divergence"""
        # Create histograms
        bins = 50
        range_min = min(p.min(), q.min())
        range_max = max(p.max(), q.max())

        p_hist = np.histogram(p, bins=bins, range=(range_min, range_max))[0]
        q_hist = np.histogram(q, bins=bins, range=(range_min, range_max))[0]

        # Normalize
        p_hist = p_hist / p_hist.sum()
        q_hist = q_hist / q_hist.sum()

        # JS divergence
        m = (p_hist + q_hist) / 2
        js = 0.5 * stats.entropy(p_hist, m) + 0.5 * stats.entropy(q_hist, m)

        return js

    def _classify_drift(self, ks_stat: float, psi: float) -> str:
        if psi > 0.25:
            return "severe"
        elif psi > 0.1:
            return "moderate"
        elif psi > 0.05:
            return "slight"
        return "none"
```

## Feature Distribution Monitoring

### Evidently Integration

```python
# evidently_monitor.py
from evidently import ColumnMapping
from evidently.report import Report
from evidently.metric_preset import DataDriftPreset, TargetDriftPreset
from evidently.metrics import *

class EvidentlyMonitor:
    def __init__(self, reference_data, column_mapping: dict):
        self.reference = reference_data
        self.mapping = ColumnMapping(**column_mapping)

    def generate_drift_report(self, current_data) -> dict:
        """Generate comprehensive drift report"""
        report = Report(metrics=[
            DataDriftPreset(),
            TargetDriftPreset(),
            DatasetDriftMetric(),
            DatasetMissingValuesMetric(),
            ColumnSummaryMetric(column_name="prediction"),
        ])

        report.run(
            reference_data=self.reference,
            current_data=current_data,
            column_mapping=self.mapping
        )

        # Extract results
        results = report.as_dict()

        return {
            'dataset_drift': results['metrics'][2]['result']['dataset_drift'],
            'drift_share': results['metrics'][2]['result']['share_of_drifted_columns'],
            'drifted_features': self._get_drifted_features(results),
            'timestamp': datetime.utcnow().isoformat()
        }

    def _get_drifted_features(self, results: dict) -> list:
        """Extract list of drifted features"""
        drifted = []
        for col_name, col_data in results['metrics'][0]['result']['drift_by_columns'].items():
            if col_data['drift_detected']:
                drifted.append({
                    'feature': col_name,
                    'drift_score': col_data['drift_score'],
                    'stattest': col_data['stattest_name']
                })
        return drifted
```

### Prometheus Metrics

```python
# prometheus_metrics.py
from prometheus_client import Gauge, Counter, Histogram

# Define metrics
model_accuracy = Gauge(
    'ml_model_accuracy',
    'Current model accuracy',
    ['model_name', 'version']
)

prediction_drift = Gauge(
    'ml_prediction_drift',
    'Prediction distribution drift score',
    ['model_name']
)

feature_drift = Gauge(
    'ml_feature_drift',
    'Feature drift score',
    ['model_name', 'feature']
)

prediction_latency = Histogram(
    'ml_prediction_latency_seconds',
    'Model prediction latency',
    ['model_name'],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0]
)

predictions_total = Counter(
    'ml_predictions_total',
    'Total predictions made',
    ['model_name', 'outcome']
)

class MetricsCollector:
    def __init__(self, model_name: str, version: str):
        self.model_name = model_name
        self.version = version

    def record_prediction(self, latency: float, outcome: str):
        """Record prediction metrics"""
        prediction_latency.labels(model_name=self.model_name).observe(latency)
        predictions_total.labels(
            model_name=self.model_name,
            outcome=outcome
        ).inc()

    def update_accuracy(self, accuracy: float):
        """Update model accuracy metric"""
        model_accuracy.labels(
            model_name=self.model_name,
            version=self.version
        ).set(accuracy)

    def update_drift_scores(self, drift_results: list):
        """Update drift metrics"""
        for result in drift_results:
            feature_drift.labels(
                model_name=self.model_name,
                feature=result.feature
            ).set(result.drift_score)
```

## Performance Monitoring

### Ground Truth Collection

```python
# ground_truth_collector.py
import asyncio
from datetime import datetime, timedelta

class GroundTruthCollector:
    def __init__(self, db_client, delay_hours: int = 24):
        self.db = db_client
        self.delay = timedelta(hours=delay_hours)

    async def collect_labels(self, prediction_ids: list) -> dict:
        """Collect ground truth labels for predictions"""
        labels = {}

        for pred_id in prediction_ids:
            # Query outcome after delay
            outcome = await self.db.query(
                """
                SELECT outcome, outcome_timestamp
                FROM outcomes
                WHERE prediction_id = $1
                AND outcome_timestamp > $2
                """,
                pred_id,
                datetime.utcnow() - self.delay
            )

            if outcome:
                labels[pred_id] = outcome['outcome']

        return labels

    async def calculate_performance(self, model_name: str,
                                    window_hours: int = 24) -> dict:
        """Calculate model performance from ground truth"""
        results = await self.db.query(
            """
            SELECT
                p.prediction,
                o.outcome,
                p.created_at
            FROM predictions p
            JOIN outcomes o ON p.id = o.prediction_id
            WHERE p.model_name = $1
            AND p.created_at > $2
            """,
            model_name,
            datetime.utcnow() - timedelta(hours=window_hours)
        )

        if not results:
            return None

        predictions = [r['prediction'] for r in results]
        outcomes = [r['outcome'] for r in results]

        return {
            'accuracy': sum(p == o for p, o in zip(predictions, outcomes)) / len(predictions),
            'sample_size': len(predictions),
            'window_hours': window_hours
        }
```

### Alerting Rules

```yaml
# prometheus-alerts.yaml
groups:
  - name: ml-model-alerts
    rules:
      - alert: ModelAccuracyDegraded
        expr: ml_model_accuracy < 0.90
        for: 30m
        labels:
          severity: warning
          team: ml-platform
        annotations:
          summary: "Model {{ $labels.model_name }} accuracy below threshold"
          description: "Accuracy is {{ $value | printf \"%.2f\" }}"

      - alert: SignificantDataDrift
        expr: ml_feature_drift > 0.2
        for: 15m
        labels:
          severity: warning
          team: ml-platform
        annotations:
          summary: "Data drift detected for {{ $labels.model_name }}"
          description: "Feature {{ $labels.feature }} drift score: {{ $value }}"

      - alert: PredictionLatencyHigh
        expr: histogram_quantile(0.99, rate(ml_prediction_latency_seconds_bucket[5m])) > 0.5
        for: 10m
        labels:
          severity: critical
          team: ml-platform
        annotations:
          summary: "High prediction latency for {{ $labels.model_name }}"

      - alert: ModelNeedsRetraining
        expr: |
          ml_model_accuracy < 0.85
          and increase(ml_predictions_total[24h]) > 1000
        for: 1h
        labels:
          severity: warning
          team: ml-platform
        annotations:
          summary: "Model {{ $labels.model_name }} needs retraining"
```

## Continuous Monitoring Pipeline

### Monitoring Service

```python
# monitoring_service.py
import asyncio
from datetime import datetime
from apscheduler.schedulers.asyncio import AsyncIOScheduler

class ModelMonitoringService:
    def __init__(self, config):
        self.config = config
        self.drift_detector = DriftDetector()
        self.evidently = EvidentlyMonitor(
            reference_data=self._load_reference_data(),
            column_mapping=config.column_mapping
        )
        self.metrics = MetricsCollector(config.model_name, config.version)
        self.scheduler = AsyncIOScheduler()

    async def start(self):
        """Start monitoring service"""
        # Schedule periodic checks
        self.scheduler.add_job(
            self.check_drift,
            'interval',
            minutes=15,
            id='drift_check'
        )

        self.scheduler.add_job(
            self.calculate_performance,
            'interval',
            hours=1,
            id='performance_check'
        )

        self.scheduler.add_job(
            self.generate_report,
            'cron',
            hour=6,
            id='daily_report'
        )

        self.scheduler.start()

    async def check_drift(self):
        """Check for data drift"""
        # Get recent predictions
        recent_data = await self._get_recent_predictions(hours=1)

        if len(recent_data) < 100:
            return

        # Statistical drift detection
        drift_results = self.drift_detector.detect_drift(
            recent_data,
            self.config.feature_names
        )

        # Update metrics
        self.metrics.update_drift_scores(drift_results)

        # Alert if significant drift
        drifted = [r for r in drift_results if r.is_drifted]
        if drifted:
            await self._alert_drift(drifted)

        # Evidently report
        report = self.evidently.generate_drift_report(
            pd.DataFrame(recent_data, columns=self.config.feature_names)
        )

        # Store report
        await self._store_report(report)

    async def calculate_performance(self):
        """Calculate model performance with ground truth"""
        performance = await self.ground_truth.calculate_performance(
            self.config.model_name
        )

        if performance:
            self.metrics.update_accuracy(performance['accuracy'])

            if performance['accuracy'] < self.config.accuracy_threshold:
                await self._trigger_retraining(
                    reason=f"Accuracy dropped to {performance['accuracy']:.2f}"
                )

    async def _alert_drift(self, drifted_features: list):
        """Send drift alert"""
        message = f"Data drift detected in model {self.config.model_name}:\n"
        for feature in drifted_features:
            message += f"- {feature.feature}: {feature.drift_type} drift "
            message += f"(score: {feature.drift_score:.3f})\n"

        await self.notifier.send_alert(
            channel="#ml-alerts",
            message=message,
            severity="warning"
        )

    async def _trigger_retraining(self, reason: str):
        """Trigger model retraining pipeline"""
        await self.pipeline_client.trigger(
            pipeline="ml-training-pipeline",
            parameters={
                "model_name": self.config.model_name,
                "reason": reason,
                "triggered_by": "monitoring"
            }
        )
```

## Grafana Dashboard

```json
{
  "dashboard": {
    "title": "ML Model Monitoring",
    "panels": [
      {
        "title": "Model Accuracy Over Time",
        "type": "graph",
        "targets": [
          {
            "expr": "ml_model_accuracy",
            "legendFormat": "{{model_name}}"
          }
        ],
        "yaxes": [
          {
            "min": 0,
            "max": 1
          }
        ]
      },
      {
        "title": "Feature Drift Heatmap",
        "type": "heatmap",
        "targets": [
          {
            "expr": "ml_feature_drift",
            "legendFormat": "{{feature}}"
          }
        ]
      },
      {
        "title": "Prediction Volume",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(ml_predictions_total[5m])) by (model_name)",
            "legendFormat": "{{model_name}}"
          }
        ]
      },
      {
        "title": "P99 Latency",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, sum(rate(ml_prediction_latency_seconds_bucket[5m])) by (le, model_name))",
            "legendFormat": "{{model_name}}"
          }
        ]
      }
    ]
  }
}
```

## Best Practices

1. **Establish baselines**: Know normal behavior
2. **Multiple detection methods**: Statistical + ML-based
3. **Delayed ground truth**: Handle label delay
4. **Contextual alerts**: Avoid alert fatigue
5. **Automated retraining**: Trigger when needed
6. **A/B test models**: Compare performance
7. **Feature importance tracking**: Monitor key features
8. **Version everything**: Track data and models

## Resources

- [Evidently AI Documentation](https://docs.evidentlyai.com/)
- [Google ML Testing Guide](https://developers.google.com/machine-learning/testing-debugging)
- [NannyML for Performance Monitoring](https://nannyml.readthedocs.io/)
- [Alibi Detect](https://docs.seldon.io/projects/alibi-detect/)

---

*Questions about model monitoring? [Let me know](mailto:jordan@jordananderson.us).*
