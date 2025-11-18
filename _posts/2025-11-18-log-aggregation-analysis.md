---
layout: default
title:  "Log Aggregation and AI-Powered Analysis"
date:   2025-11-18 12:00:00
categories: DevOps Observability Logging AI
---

Centralized logging with AI-powered analysis can automatically detect anomalies, correlate events, and surface insights from massive log volumes.

## Logging Architecture

```
┌───────────┐   ┌───────────┐   ┌───────────┐
│   Apps    │   │   K8s     │   │   Infra   │
└─────┬─────┘   └─────┬─────┘   └─────┬─────┘
      │               │               │
      └───────────────┼───────────────┘
                      ▼
            ┌─────────────────┐
            │   Fluentd/      │
            │   Fluent Bit    │
            └────────┬────────┘
                     ▼
            ┌─────────────────┐
            │   Kafka/        │
            │   Vector        │
            └────────┬────────┘
                     ▼
            ┌─────────────────┐
            │  Elasticsearch/ │
            │  Loki           │
            └────────┬────────┘
                     ▼
            ┌─────────────────┐
            │  Grafana/       │
            │  Kibana         │
            └─────────────────┘
```

## Structured Logging

### Python Application

```python
# logger.py
import structlog
import logging
from opentelemetry import trace

def configure_logging():
    """Configure structured logging with trace context"""
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            add_trace_context,
            structlog.processors.JSONRenderer()
        ],
        wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(),
    )

def add_trace_context(logger, method_name, event_dict):
    """Add OpenTelemetry trace context"""
    span = trace.get_current_span()
    ctx = span.get_span_context()

    if ctx.is_valid:
        event_dict['trace_id'] = format(ctx.trace_id, '032x')
        event_dict['span_id'] = format(ctx.span_id, '016x')

    return event_dict

# Usage
logger = structlog.get_logger()

logger.info(
    "prediction_completed",
    model="recommendation-v2",
    user_id="user_123",
    latency_ms=45,
    prediction_count=10
)
```

### ML-Specific Logging

```python
# ml_logger.py
class MLLogger:
    def __init__(self):
        self.logger = structlog.get_logger()

    def log_prediction(self, model_name: str, input_data: dict,
                       prediction: dict, latency_ms: float):
        """Log ML prediction with context"""
        self.logger.info(
            "ml_prediction",
            event_type="prediction",
            model_name=model_name,
            model_version=prediction.get('model_version'),
            input_hash=self._hash_input(input_data),
            prediction_class=prediction.get('class'),
            confidence=prediction.get('confidence'),
            latency_ms=latency_ms,
            feature_count=len(input_data.get('features', []))
        )

    def log_model_load(self, model_name: str, version: str,
                      load_time_ms: float):
        """Log model loading"""
        self.logger.info(
            "model_loaded",
            event_type="model_lifecycle",
            model_name=model_name,
            model_version=version,
            load_time_ms=load_time_ms
        )

    def log_data_drift(self, feature: str, drift_score: float):
        """Log detected data drift"""
        self.logger.warning(
            "data_drift_detected",
            event_type="monitoring",
            feature=feature,
            drift_score=drift_score,
            alert_level="warning" if drift_score < 0.3 else "critical"
        )
```

## Loki for Kubernetes

### Installation

```bash
# Install Loki Stack
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --namespace logging \
  --create-namespace \
  --set promtail.enabled=true \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=50Gi
```

### Promtail Configuration

```yaml
# promtail-config.yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    pipeline_stages:
      - cri: {}
      - json:
          expressions:
            level: level
            message: message
            trace_id: trace_id
            model_name: model_name
      - labels:
          level:
          model_name:
      - timestamp:
          source: time
          format: RFC3339Nano
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
```

### LogQL Queries

```logql
# Error rate by service
sum by (app) (rate({namespace="ml-platform"} |= "error" [5m]))

# ML prediction latency histogram
{app="ml-inference"} | json | latency_ms > 100

# Find slow predictions
{app="ml-inference"}
  | json
  | latency_ms > 200
  | line_format "{{.model_name}} - {{.latency_ms}}ms"

# Group errors by model
sum by (model_name) (
  count_over_time(
    {app="ml-inference"} |= "error" | json [1h]
  )
)

# Trace correlation
{app=~"ml-.*"} |= "trace_id=abc123"
```

## Elasticsearch/OpenSearch

### Index Template

```json
{
  "index_patterns": ["ml-logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "index.lifecycle.name": "ml-logs-policy"
    },
    "mappings": {
      "properties": {
        "@timestamp": {"type": "date"},
        "level": {"type": "keyword"},
        "message": {"type": "text"},
        "model_name": {"type": "keyword"},
        "model_version": {"type": "keyword"},
        "latency_ms": {"type": "float"},
        "trace_id": {"type": "keyword"},
        "user_id": {"type": "keyword"},
        "prediction": {
          "properties": {
            "class": {"type": "keyword"},
            "confidence": {"type": "float"}
          }
        }
      }
    }
  }
}
```

### Index Lifecycle Management

```json
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50gb",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": {"number_of_shards": 1},
          "forcemerge": {"max_num_segments": 1}
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {}
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

## AI-Powered Log Analysis

### Anomaly Detection

```python
# log_anomaly_detector.py
from sklearn.ensemble import IsolationForest
from sentence_transformers import SentenceTransformer
import numpy as np

class LogAnomalyDetector:
    def __init__(self):
        self.encoder = SentenceTransformer('all-MiniLM-L6-v2')
        self.model = IsolationForest(contamination=0.01)
        self.baseline_embeddings = None

    def train(self, normal_logs: list[str]):
        """Train on normal log patterns"""
        embeddings = self.encoder.encode(normal_logs)
        self.baseline_embeddings = embeddings
        self.model.fit(embeddings)

    def detect_anomaly(self, log_message: str) -> dict:
        """Detect if log message is anomalous"""
        embedding = self.encoder.encode([log_message])
        score = self.model.score_samples(embedding)[0]
        is_anomaly = self.model.predict(embedding)[0] == -1

        return {
            'message': log_message,
            'is_anomaly': is_anomaly,
            'anomaly_score': abs(score),
            'similar_normal': self._find_similar(embedding)
        }

    def _find_similar(self, embedding) -> str:
        """Find most similar normal log"""
        similarities = np.dot(self.baseline_embeddings, embedding.T)
        most_similar_idx = np.argmax(similarities)
        return self.baseline_logs[most_similar_idx]
```

### LLM-Powered Analysis

```python
# llm_log_analyzer.py
from openai import OpenAI

class LLMLogAnalyzer:
    def __init__(self, api_key: str):
        self.client = OpenAI(api_key=api_key)

    async def analyze_errors(self, error_logs: list[str]) -> dict:
        """Analyze error logs with LLM"""
        prompt = f"""Analyze these error logs and provide:
1. Root cause analysis
2. Affected components
3. Recommended actions
4. Severity assessment

Logs:
{chr(10).join(error_logs[:50])}
"""

        response = self.client.chat.completions.create(
            model="gpt-4",
            messages=[
                {
                    "role": "system",
                    "content": "You are an SRE expert analyzing application logs."
                },
                {"role": "user", "content": prompt}
            ]
        )

        return {
            'analysis': response.choices[0].message.content,
            'log_count': len(error_logs)
        }

    async def summarize_logs(self, logs: list[str],
                             time_window: str) -> str:
        """Generate summary of log activity"""
        prompt = f"""Summarize the key events from these logs in {time_window}:

{chr(10).join(logs[:100])}

Provide:
- Main activities
- Any issues or warnings
- Notable patterns
"""

        response = self.client.chat.completions.create(
            model="gpt-4",
            messages=[
                {"role": "user", "content": prompt}
            ]
        )

        return response.choices[0].message.content
```

### Pattern Recognition

```python
# pattern_detector.py
from drain3 import TemplateMiner
from drain3.template_miner_config import TemplateMinerConfig

class LogPatternDetector:
    def __init__(self):
        config = TemplateMinerConfig()
        config.drain_sim_th = 0.4
        config.drain_depth = 4
        self.miner = TemplateMiner(config=config)

    def extract_patterns(self, logs: list[str]) -> list[dict]:
        """Extract log patterns using Drain algorithm"""
        patterns = {}

        for log in logs:
            result = self.miner.add_log_message(log)
            cluster_id = result['cluster_id']

            if cluster_id not in patterns:
                patterns[cluster_id] = {
                    'template': result['template_mined'],
                    'count': 0,
                    'examples': []
                }

            patterns[cluster_id]['count'] += 1
            if len(patterns[cluster_id]['examples']) < 3:
                patterns[cluster_id]['examples'].append(log)

        return sorted(
            patterns.values(),
            key=lambda x: x['count'],
            reverse=True
        )

    def detect_new_patterns(self, log: str) -> bool:
        """Detect if log represents a new pattern"""
        result = self.miner.add_log_message(log)
        return result['change_type'] == 'cluster_created'
```

## Alerting on Logs

### Loki Alert Rules

```yaml
# loki-alerts.yaml
groups:
  - name: ml-log-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate({namespace="ml-platform"} |= "error" [5m])) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate in ML platform"

      - alert: ModelLoadFailure
        expr: |
          count_over_time(
            {app="ml-inference"} |= "model_load_failed" [5m]
          ) > 0
        labels:
          severity: critical
        annotations:
          summary: "Model failed to load"

      - alert: SlowPredictions
        expr: |
          avg(
            avg_over_time(
              {app="ml-inference"} | json | unwrap latency_ms [5m]
            )
          ) > 200
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Prediction latency above threshold"
```

## Grafana Dashboard

```json
{
  "dashboard": {
    "title": "ML Logs Analysis",
    "panels": [
      {
        "title": "Log Volume by Level",
        "type": "timeseries",
        "targets": [
          {
            "expr": "sum by (level) (rate({namespace=\"ml-platform\"} | json [5m]))",
            "legendFormat": "{{level}}"
          }
        ]
      },
      {
        "title": "Errors by Model",
        "type": "piechart",
        "targets": [
          {
            "expr": "sum by (model_name) (count_over_time({app=\"ml-inference\"} |= \"error\" | json [1h]))"
          }
        ]
      },
      {
        "title": "Log Stream",
        "type": "logs",
        "targets": [
          {
            "expr": "{namespace=\"ml-platform\"} |= \"error\" or |= \"warning\""
          }
        ]
      }
    ]
  }
}
```

## Best Practices

1. **Structure logs**: JSON format with consistent fields
2. **Include context**: Trace IDs, user IDs, model versions
3. **Use log levels**: Appropriate severity classification
4. **Aggregate centrally**: Single source of truth
5. **Retention policies**: Balance cost and compliance
6. **AI analysis**: Automate pattern detection
7. **Alert thoughtfully**: Avoid alert fatigue
8. **Correlate with traces**: Connect logs to requests

## Resources

- [Grafana Loki Documentation](https://grafana.com/docs/loki/)
- [Elasticsearch Guide](https://www.elastic.co/guide/)
- [Structured Logging](https://www.structlog.org/)
- [Drain Log Parsing](https://github.com/logpai/Drain3)

---

*Questions about log aggregation? [Let me know](mailto:jordan@jordananderson.us).*
