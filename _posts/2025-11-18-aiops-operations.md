---
layout: default
title:  "AIOps: AI-Powered IT Operations"
date:   2025-11-18 10:00:00
categories: DevOps AIOps AI Observability
---

AIOps uses AI and machine learning to automate IT operations, reducing incident resolution time by up to 60% and preventing up to 70% of potential incidents. Here's how to implement AIOps effectively.

## What is AIOps?

AIOps combines big data and machine learning to automate:
- **Anomaly detection**: Identify issues before users notice
- **Event correlation**: Reduce alert noise by 70-90%
- **Root cause analysis**: Pinpoint issues faster
- **Automated remediation**: Fix issues without human intervention

## AIOps Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Data Sources                      │
│  Logs | Metrics | Traces | Events | Tickets         │
└─────────────────────────┬───────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────┐
│              Data Ingestion & Processing             │
└─────────────────────────┬───────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────┐
│                  AI/ML Engine                        │
│  Anomaly Detection | Correlation | Prediction       │
└─────────────────────────┬───────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────┐
│           Automation & Orchestration                 │
│  Alerting | Remediation | Runbooks                  │
└─────────────────────────────────────────────────────┘
```

## Anomaly Detection

### Time-Series Anomaly Detection

```python
from prophet import Prophet
import pandas as pd

class MetricAnomalyDetector:
    def __init__(self, sensitivity: float = 0.95):
        self.sensitivity = sensitivity
        self.model = None

    def train(self, historical_data: pd.DataFrame):
        """Train Prophet model on historical metrics."""
        # Prophet expects columns: ds (timestamp), y (value)
        df = historical_data.rename(columns={
            "timestamp": "ds",
            "value": "y"
        })

        self.model = Prophet(
            interval_width=self.sensitivity,
            daily_seasonality=True,
            weekly_seasonality=True
        )
        self.model.fit(df)

    def detect_anomalies(self, current_data: pd.DataFrame) -> list:
        """Detect anomalies in current metrics."""
        forecast = self.model.predict(current_data)

        anomalies = []
        for idx, row in current_data.iterrows():
            pred = forecast.loc[idx]

            if row["y"] < pred["yhat_lower"] or row["y"] > pred["yhat_upper"]:
                anomalies.append({
                    "timestamp": row["ds"],
                    "actual": row["y"],
                    "expected_lower": pred["yhat_lower"],
                    "expected_upper": pred["yhat_upper"],
                    "severity": self.calculate_severity(row["y"], pred)
                })

        return anomalies

    def calculate_severity(self, actual, pred) -> str:
        deviation = abs(actual - pred["yhat"]) / pred["yhat"]
        if deviation > 0.5:
            return "critical"
        elif deviation > 0.2:
            return "warning"
        return "info"
```

### Isolation Forest for Multi-dimensional Anomalies

```python
from sklearn.ensemble import IsolationForest
import numpy as np

class SystemAnomalyDetector:
    def __init__(self, contamination: float = 0.01):
        self.model = IsolationForest(
            contamination=contamination,
            random_state=42,
            n_estimators=100
        )

    def train(self, metrics: np.ndarray):
        """Train on normal system behavior."""
        # metrics: [cpu, memory, disk_io, network_io, latency]
        self.model.fit(metrics)

    def detect(self, current_metrics: np.ndarray) -> dict:
        """Detect system anomalies."""
        predictions = self.model.predict(current_metrics)
        scores = self.model.score_samples(current_metrics)

        anomalies = []
        for i, (pred, score) in enumerate(zip(predictions, scores)):
            if pred == -1:  # Anomaly
                anomalies.append({
                    "index": i,
                    "metrics": current_metrics[i].tolist(),
                    "anomaly_score": float(score)
                })

        return {
            "total_samples": len(current_metrics),
            "anomalies_detected": len(anomalies),
            "anomalies": anomalies
        }
```

## Event Correlation

### Intelligent Alert Grouping

```python
from sklearn.cluster import DBSCAN
from sentence_transformers import SentenceTransformer
import numpy as np

class AlertCorrelator:
    def __init__(self):
        self.encoder = SentenceTransformer('all-MiniLM-L6-v2')

    def correlate_alerts(
        self,
        alerts: list[dict],
        time_window_minutes: int = 5
    ) -> list[dict]:
        """Group related alerts together."""
        if not alerts:
            return []

        # Extract alert messages
        messages = [a["message"] for a in alerts]

        # Encode alerts
        embeddings = self.encoder.encode(messages)

        # Cluster similar alerts
        clustering = DBSCAN(eps=0.3, min_samples=2).fit(embeddings)

        # Group alerts by cluster
        groups = {}
        for i, label in enumerate(clustering.labels_):
            if label == -1:
                # Unclustered alert
                groups[f"single_{i}"] = [alerts[i]]
            else:
                if label not in groups:
                    groups[label] = []
                groups[label].append(alerts[i])

        # Create correlated incidents
        incidents = []
        for group_id, group_alerts in groups.items():
            incidents.append({
                "incident_id": f"INC-{group_id}",
                "alert_count": len(group_alerts),
                "severity": max(a["severity"] for a in group_alerts),
                "first_seen": min(a["timestamp"] for a in group_alerts),
                "alerts": group_alerts,
                "summary": self.generate_summary(group_alerts)
            })

        return incidents

    def generate_summary(self, alerts: list[dict]) -> str:
        """Generate incident summary from alerts."""
        services = set(a.get("service", "unknown") for a in alerts)
        return f"{len(alerts)} alerts from {', '.join(services)}"
```

## Root Cause Analysis

### Causal Analysis with Graph

```python
import networkx as nx

class RootCauseAnalyzer:
    def __init__(self):
        self.dependency_graph = nx.DiGraph()

    def build_dependency_graph(self, services: list[dict]):
        """Build service dependency graph."""
        for service in services:
            self.dependency_graph.add_node(
                service["name"],
                type=service["type"]
            )

            for dep in service.get("dependencies", []):
                self.dependency_graph.add_edge(dep, service["name"])

    def analyze_incident(self, affected_services: list[str]) -> dict:
        """Find root cause of incident."""
        # Find common ancestors
        if not affected_services:
            return {"root_cause": None, "confidence": 0}

        # Get all ancestors for each affected service
        all_ancestors = []
        for service in affected_services:
            ancestors = nx.ancestors(self.dependency_graph, service)
            all_ancestors.append(ancestors | {service})

        # Find intersection
        common = set.intersection(*all_ancestors) if all_ancestors else set()

        # Root cause is the service with no healthy dependencies
        root_causes = []
        for candidate in common:
            # Check if all dependencies are healthy
            deps = list(self.dependency_graph.predecessors(candidate))
            if candidate in affected_services:
                root_causes.append({
                    "service": candidate,
                    "affected_downstream": len(list(
                        nx.descendants(self.dependency_graph, candidate)
                    ))
                })

        # Sort by impact
        root_causes.sort(key=lambda x: x["affected_downstream"], reverse=True)

        return {
            "root_cause": root_causes[0] if root_causes else None,
            "candidates": root_causes,
            "affected_services": affected_services
        }
```

## Automated Remediation

### Runbook Automation

```python
import asyncio
from typing import Callable

class RemediationEngine:
    def __init__(self):
        self.runbooks = {}

    def register_runbook(
        self,
        incident_type: str,
        actions: list[Callable]
    ):
        """Register automated runbook."""
        self.runbooks[incident_type] = actions

    async def execute_remediation(
        self,
        incident: dict
    ) -> dict:
        """Execute automated remediation."""
        incident_type = incident.get("type")

        if incident_type not in self.runbooks:
            return {
                "status": "manual_required",
                "reason": f"No runbook for {incident_type}"
            }

        results = []
        for action in self.runbooks[incident_type]:
            try:
                result = await action(incident)
                results.append({
                    "action": action.__name__,
                    "status": "success",
                    "result": result
                })

                if result.get("resolved"):
                    break

            except Exception as e:
                results.append({
                    "action": action.__name__,
                    "status": "failed",
                    "error": str(e)
                })

        return {
            "incident_id": incident["id"],
            "actions_executed": len(results),
            "results": results,
            "resolved": any(r.get("result", {}).get("resolved") for r in results)
        }

# Example runbook actions
async def restart_service(incident: dict) -> dict:
    service = incident["service"]
    # kubectl rollout restart deployment/{service}
    return {"resolved": True, "action": f"Restarted {service}"}

async def scale_up(incident: dict) -> dict:
    service = incident["service"]
    # kubectl scale deployment/{service} --replicas=5
    return {"resolved": True, "action": f"Scaled {service} to 5 replicas"}

async def clear_cache(incident: dict) -> dict:
    # redis-cli FLUSHDB
    return {"resolved": True, "action": "Cleared cache"}

# Register runbook
engine = RemediationEngine()
engine.register_runbook(
    "high_latency",
    [clear_cache, scale_up, restart_service]
)
```

## Predictive Analytics

```python
from sklearn.ensemble import GradientBoostingRegressor
import pandas as pd

class CapacityPredictor:
    def __init__(self):
        self.model = GradientBoostingRegressor(n_estimators=100)

    def train(self, historical_usage: pd.DataFrame):
        """Train on historical resource usage."""
        # Features: hour, day_of_week, month, lag features
        X = self.extract_features(historical_usage)
        y = historical_usage["usage"]

        self.model.fit(X, y)

    def predict_capacity_exhaustion(
        self,
        current_usage: float,
        capacity: float,
        days_ahead: int = 7
    ) -> dict:
        """Predict when capacity will be exhausted."""
        predictions = []

        for day in range(days_ahead):
            # Generate future features
            future_features = self.generate_future_features(day)
            predicted_usage = self.model.predict([future_features])[0]
            predictions.append(predicted_usage)

        # Find exhaustion point
        for i, pred in enumerate(predictions):
            if pred > capacity * 0.9:
                return {
                    "warning": True,
                    "days_until_threshold": i,
                    "predicted_usage": pred,
                    "threshold": capacity * 0.9,
                    "recommendation": f"Scale up within {i} days"
                }

        return {
            "warning": False,
            "max_predicted": max(predictions),
            "capacity": capacity
        }
```

## Integration Example

### Complete AIOps Pipeline

```python
class AIOPsPlatform:
    def __init__(self):
        self.anomaly_detector = MetricAnomalyDetector()
        self.alert_correlator = AlertCorrelator()
        self.rca_analyzer = RootCauseAnalyzer()
        self.remediation_engine = RemediationEngine()

    async def process_metrics(self, metrics: dict):
        """Process incoming metrics through AIOps pipeline."""
        # 1. Detect anomalies
        anomalies = self.anomaly_detector.detect_anomalies(metrics)

        if not anomalies:
            return {"status": "healthy"}

        # 2. Generate alerts
        alerts = [self.create_alert(a) for a in anomalies]

        # 3. Correlate alerts into incidents
        incidents = self.alert_correlator.correlate_alerts(alerts)

        # 4. Analyze root cause for each incident
        for incident in incidents:
            affected = [a["service"] for a in incident["alerts"]]
            rca = self.rca_analyzer.analyze_incident(affected)
            incident["root_cause"] = rca

        # 5. Execute automated remediation
        for incident in incidents:
            if incident["severity"] in ["critical", "high"]:
                result = await self.remediation_engine.execute_remediation(incident)
                incident["remediation"] = result

        return {
            "status": "incidents_detected",
            "incidents": incidents
        }
```

## Best Practices

1. **Start with observability**: Ensure comprehensive data collection
2. **Tune thresholds gradually**: Reduce false positives over time
3. **Human-in-the-loop**: Keep humans involved for critical actions
4. **Version your models**: Track ML model performance
5. **Feedback loops**: Use incident outcomes to improve detection

## Key Metrics

- **MTTR reduction**: Target 50-60% improvement
- **Alert noise reduction**: Target 70-90%
- **False positive rate**: Keep below 10%
- **Automation rate**: Percentage of incidents auto-resolved

## Resources

- [Gartner AIOps Market Guide](https://www.gartner.com/en/documents/aiops)
- [Splunk State of AIOps](https://www.splunk.com/en_us/campaigns/state-of-observability.html)
- [Moogsoft AIOps Platform](https://www.moogsoft.com/)
- [BigPanda](https://www.bigpanda.io/)

---

*Questions about AIOps implementation? [Let me know](mailto:jordan@jordananderson.us).*
