---
layout: default
title:  "Modern Observability with OpenTelemetry and AI"
date:   2025-11-18 12:00:00
categories: DevOps Observability OpenTelemetry AI
---

OpenTelemetry has become the standard for collecting telemetry data. Combined with AI-powered analysis, it enables predictive monitoring and intelligent alerting.

## OpenTelemetry Architecture

### Instrumentation

```python
# Auto-instrumentation for Python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

# Setup tracer
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

# Configure OTLP exporter
otlp_exporter = OTLPSpanExporter(
    endpoint="localhost:4317",
    insecure=True
)

span_processor = BatchSpanProcessor(otlp_exporter)
trace.get_tracer_provider().add_span_processor(span_processor)

# Auto-instrument Flask
from flask import Flask
app = Flask(__name__)
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()
```

### Custom Spans and Attributes

```python
from opentelemetry import trace
from opentelemetry.trace import Status, StatusCode

tracer = trace.get_tracer(__name__)

@app.route('/api/predict')
def predict():
    with tracer.start_as_current_span("ml_prediction") as span:
        # Add custom attributes
        span.set_attribute("model.name", "recommendation-v2")
        span.set_attribute("model.version", "2.1.0")
        span.set_attribute("request.batch_size", len(request.json))

        try:
            result = model.predict(request.json)
            span.set_attribute("prediction.confidence", result.confidence)
            span.set_status(Status(StatusCode.OK))
            return jsonify(result)
        except Exception as e:
            span.set_status(Status(StatusCode.ERROR, str(e)))
            span.record_exception(e)
            raise
```

### Metrics Collection

```python
from opentelemetry import metrics
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter

# Setup metrics
metric_reader = PeriodicExportingMetricReader(
    OTLPMetricExporter(endpoint="localhost:4317"),
    export_interval_millis=60000
)
metrics.set_meter_provider(MeterProvider(metric_readers=[metric_reader]))

meter = metrics.get_meter(__name__)

# Create instruments
request_counter = meter.create_counter(
    "api_requests_total",
    description="Total API requests",
    unit="1"
)

latency_histogram = meter.create_histogram(
    "api_latency_ms",
    description="API latency in milliseconds",
    unit="ms"
)

model_inference_time = meter.create_histogram(
    "model_inference_ms",
    description="ML model inference time",
    unit="ms"
)

# Record metrics
@app.route('/api/predict')
def predict():
    start = time.time()
    request_counter.add(1, {"endpoint": "/api/predict", "model": "v2"})

    result = model.predict(request.json)

    latency = (time.time() - start) * 1000
    latency_histogram.record(latency, {"endpoint": "/api/predict"})
    model_inference_time.record(result.inference_time_ms)

    return jsonify(result)
```

## OpenTelemetry Collector

### Collector Configuration

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

  prometheus:
    config:
      scrape_configs:
        - job_name: 'kubernetes-pods'
          kubernetes_sd_configs:
            - role: pod

processors:
  batch:
    timeout: 10s
    send_batch_size: 1024

  memory_limiter:
    check_interval: 1s
    limit_mib: 1000
    spike_limit_mib: 200

  attributes:
    actions:
      - key: environment
        value: production
        action: insert

  # AI-powered sampling
  probabilistic_sampler:
    sampling_percentage: 10

exporters:
  otlp:
    endpoint: "tempo:4317"
    tls:
      insecure: true

  prometheus:
    endpoint: "0.0.0.0:8889"

  loki:
    endpoint: "http://loki:3100/loki/api/v1/push"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, attributes]
      exporters: [otlp]

    metrics:
      receivers: [otlp, prometheus]
      processors: [memory_limiter, batch]
      exporters: [prometheus]

    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [loki]
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
spec:
  replicas: 3
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
        - name: collector
          image: otel/opentelemetry-collector-contrib:0.88.0
          args:
            - --config=/etc/otel/config.yaml
          ports:
            - containerPort: 4317  # OTLP gRPC
            - containerPort: 4318  # OTLP HTTP
            - containerPort: 8889  # Prometheus metrics
          resources:
            limits:
              memory: 2Gi
              cpu: 1000m
          volumeMounts:
            - name: config
              mountPath: /etc/otel
      volumes:
        - name: config
          configMap:
            name: otel-collector-config
```

## AI-Powered Anomaly Detection

### Intelligent Alerting

```python
import numpy as np
from sklearn.ensemble import IsolationForest
from prometheus_api_client import PrometheusConnect

class AIAnomalyDetector:
    def __init__(self, prometheus_url: str):
        self.prom = PrometheusConnect(url=prometheus_url)
        self.models = {}

    def train_baseline(self, metric_name: str, label_config: dict,
                       duration: str = "7d"):
        """Train anomaly detection model on historical data"""
        query = f'{metric_name}{{{self._format_labels(label_config)}}}'

        # Fetch historical data
        data = self.prom.custom_query_range(
            query=query,
            start_time=self._parse_duration(duration),
            end_time=datetime.now(),
            step="5m"
        )

        if not data:
            return

        # Extract values
        values = np.array([float(v[1]) for v in data[0]['values']])

        # Train Isolation Forest
        model = IsolationForest(
            contamination=0.01,
            random_state=42
        )
        model.fit(values.reshape(-1, 1))

        self.models[metric_name] = {
            'model': model,
            'mean': np.mean(values),
            'std': np.std(values)
        }

    def detect_anomaly(self, metric_name: str, value: float) -> dict:
        """Detect if a value is anomalous"""
        if metric_name not in self.models:
            return {'is_anomaly': False, 'confidence': 0}

        model_data = self.models[metric_name]
        model = model_data['model']

        # Predict
        prediction = model.predict([[value]])[0]
        score = model.score_samples([[value]])[0]

        # Calculate z-score
        z_score = abs((value - model_data['mean']) / model_data['std'])

        return {
            'is_anomaly': prediction == -1,
            'confidence': abs(score),
            'z_score': z_score,
            'expected_range': (
                model_data['mean'] - 3 * model_data['std'],
                model_data['mean'] + 3 * model_data['std']
            )
        }

    def _format_labels(self, labels: dict) -> str:
        return ','.join([f'{k}="{v}"' for k, v in labels.items()])
```

### Predictive Alerting

```python
from prophet import Prophet
import pandas as pd

class PredictiveAlerting:
    def __init__(self):
        self.forecast_models = {}

    def train_forecast(self, metric_name: str, historical_data: pd.DataFrame):
        """Train forecasting model"""
        df = historical_data.rename(columns={
            'timestamp': 'ds',
            'value': 'y'
        })

        model = Prophet(
            interval_width=0.95,
            daily_seasonality=True,
            weekly_seasonality=True
        )
        model.fit(df)

        self.forecast_models[metric_name] = model

    def predict_breach(self, metric_name: str, threshold: float,
                       hours_ahead: int = 24) -> dict:
        """Predict if metric will breach threshold"""
        if metric_name not in self.forecast_models:
            return None

        model = self.forecast_models[metric_name]

        # Create future dataframe
        future = model.make_future_dataframe(
            periods=hours_ahead,
            freq='H'
        )

        forecast = model.predict(future)

        # Check for predicted breaches
        future_predictions = forecast.tail(hours_ahead)
        breaches = future_predictions[
            future_predictions['yhat'] > threshold
        ]

        if not breaches.empty:
            first_breach = breaches.iloc[0]
            return {
                'will_breach': True,
                'predicted_time': first_breach['ds'],
                'predicted_value': first_breach['yhat'],
                'confidence_interval': (
                    first_breach['yhat_lower'],
                    first_breach['yhat_upper']
                )
            }

        return {'will_breach': False}
```

## Distributed Tracing

### Trace Context Propagation

```python
from opentelemetry import trace
from opentelemetry.propagate import inject, extract
import requests

tracer = trace.get_tracer(__name__)

def call_downstream_service(url: str, data: dict):
    """Make HTTP call with trace context propagation"""
    with tracer.start_as_current_span("downstream_call") as span:
        span.set_attribute("http.url", url)

        # Inject trace context into headers
        headers = {}
        inject(headers)

        response = requests.post(url, json=data, headers=headers)

        span.set_attribute("http.status_code", response.status_code)
        return response.json()

# Receiving service extracts context
@app.route('/api/process')
def process():
    # Extract trace context from incoming request
    context = extract(request.headers)

    with tracer.start_as_current_span(
        "process_request",
        context=context
    ) as span:
        # Process continues with same trace
        result = do_processing()
        return jsonify(result)
```

### Service Map Generation

```python
from collections import defaultdict
import networkx as nx

class ServiceMapBuilder:
    def __init__(self):
        self.graph = nx.DiGraph()

    def process_traces(self, traces: list):
        """Build service map from traces"""
        for trace in traces:
            spans = sorted(trace['spans'], key=lambda s: s['startTime'])

            for span in spans:
                service = span['serviceName']

                # Add node with metrics
                if not self.graph.has_node(service):
                    self.graph.add_node(service,
                        request_count=0,
                        error_count=0,
                        latency_sum=0
                    )

                self.graph.nodes[service]['request_count'] += 1
                self.graph.nodes[service]['latency_sum'] += span['duration']

                if span.get('status', {}).get('code') == 'ERROR':
                    self.graph.nodes[service]['error_count'] += 1

                # Add edges for parent-child relationships
                if span.get('parentSpanId'):
                    parent_span = self._find_span(spans, span['parentSpanId'])
                    if parent_span:
                        parent_service = parent_span['serviceName']
                        if not self.graph.has_edge(parent_service, service):
                            self.graph.add_edge(parent_service, service,
                                call_count=0)
                        self.graph.edges[parent_service, service]['call_count'] += 1

    def get_critical_path(self) -> list:
        """Identify critical path in service mesh"""
        # Find path with highest latency
        paths = []
        for source in self.graph.nodes():
            for target in self.graph.nodes():
                if source != target:
                    try:
                        path = nx.shortest_path(self.graph, source, target)
                        latency = sum(
                            self.graph.nodes[n]['latency_sum'] /
                            self.graph.nodes[n]['request_count']
                            for n in path
                        )
                        paths.append((path, latency))
                    except nx.NetworkXNoPath:
                        continue

        return max(paths, key=lambda x: x[1]) if paths else None
```

## Logs Correlation

### Structured Logging with Trace Context

```python
import logging
import json
from opentelemetry import trace

class OTelLogHandler(logging.Handler):
    def emit(self, record):
        # Get current span context
        span = trace.get_current_span()
        ctx = span.get_span_context()

        log_entry = {
            'timestamp': self.format_time(record),
            'level': record.levelname,
            'message': record.getMessage(),
            'logger': record.name,
            'trace_id': format(ctx.trace_id, '032x') if ctx.is_valid else None,
            'span_id': format(ctx.span_id, '016x') if ctx.is_valid else None,
        }

        # Add exception info if present
        if record.exc_info:
            log_entry['exception'] = self.format_exception(record.exc_info)

        print(json.dumps(log_entry))

# Configure logging
logger = logging.getLogger(__name__)
logger.addHandler(OTelLogHandler())
logger.setLevel(logging.INFO)

@app.route('/api/process')
def process():
    with tracer.start_as_current_span("process") as span:
        logger.info("Starting processing", extra={'user_id': user_id})

        try:
            result = do_work()
            logger.info("Processing complete", extra={'result_size': len(result)})
            return jsonify(result)
        except Exception as e:
            logger.error(f"Processing failed: {e}", exc_info=True)
            raise
```

## Grafana Dashboard

### Dashboard as Code

```json
{
  "dashboard": {
    "title": "Service Observability",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(api_requests_total[5m])) by (service)",
            "legendFormat": "{{service}}"
          }
        ]
      },
      {
        "title": "P99 Latency",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, sum(rate(api_latency_ms_bucket[5m])) by (le, service))",
            "legendFormat": "{{service}}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(rate(api_requests_total{status=~\"5..\"}[5m])) / sum(rate(api_requests_total[5m])) * 100"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 1},
                {"color": "red", "value": 5}
              ]
            }
          }
        }
      },
      {
        "title": "Trace Explorer",
        "type": "traces",
        "datasource": "Tempo",
        "targets": [
          {
            "query": "{service.name=\"api-gateway\"}"
          }
        ]
      }
    ]
  }
}
```

## Best Practices

### Instrumentation Guidelines

1. **Semantic conventions**: Follow OpenTelemetry semantic conventions
2. **Meaningful span names**: Use operation names, not URLs
3. **Appropriate granularity**: Don't over-instrument
4. **Error handling**: Always record exceptions
5. **Sampling strategy**: Balance detail vs. cost

### Alert Design

```yaml
# Prometheus alerting rules
groups:
  - name: slo-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(api_requests_total{status=~"5.."}[5m]))
          / sum(rate(api_requests_total[5m])) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Error rate exceeds 1%"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            sum(rate(api_latency_ms_bucket[5m])) by (le)
          ) > 500
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency exceeds 500ms"
```

## Resources

- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)
- [Grafana LGTM Stack](https://grafana.com/oss/lgtm-stack/)
- [Prometheus Best Practices](https://prometheus.io/docs/practices/naming/)

---

*Questions about observability? [Let me know](mailto:jordan@jordananderson.us).*
