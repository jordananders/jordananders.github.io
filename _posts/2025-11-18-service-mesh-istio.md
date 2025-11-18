---
layout: default
title:  "Service Mesh with Istio for ML Services"
date:   2025-11-18 12:00:00
categories: DevOps ServiceMesh Istio Kubernetes
---

Service meshes provide traffic management, security, and observability. For ML services, they enable canary deployments, A/B testing, and model versioning with traffic splitting.

## Istio Installation

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Install with production profile
istioctl install --set profile=production -y

# Enable sidecar injection
kubectl label namespace ml-platform istio-injection=enabled
```

## Traffic Management

### Virtual Service for Model Versions

```yaml
# model-routing.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ml-inference
  namespace: ml-platform
spec:
  hosts:
    - ml-inference
  http:
    - match:
        - headers:
            x-model-version:
              exact: "v2"
      route:
        - destination:
            host: ml-inference
            subset: v2
    - route:
        - destination:
            host: ml-inference
            subset: v1
          weight: 90
        - destination:
            host: ml-inference
            subset: v2
          weight: 10

---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: ml-inference
  namespace: ml-platform
spec:
  host: ml-inference
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

### Canary Deployment

```yaml
# canary-deployment.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: recommendation-model
spec:
  hosts:
    - recommendation-model
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: recommendation-model
            subset: canary

    # Mirror traffic to canary for testing
    - route:
        - destination:
            host: recommendation-model
            subset: stable
      mirror:
        host: recommendation-model
        subset: canary
      mirrorPercentage:
        value: 10
```

### A/B Testing

```yaml
# ab-testing.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ml-inference
spec:
  hosts:
    - ml-inference
  http:
    # Route based on user segment
    - match:
        - headers:
            x-user-segment:
              exact: "premium"
      route:
        - destination:
            host: ml-inference
            subset: premium-model

    # Route based on geographic location
    - match:
        - headers:
            x-region:
              regex: "us-.*"
      route:
        - destination:
            host: ml-inference
            subset: us-model

    # Default route
    - route:
        - destination:
            host: ml-inference
            subset: default-model
```

## Circuit Breaking

### Outlier Detection

```yaml
# circuit-breaker.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: ml-inference
spec:
  host: ml-inference
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 60s
      maxEjectionPercent: 50
      minHealthPercent: 30

    connectionPool:
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
        maxRetries: 3

    loadBalancer:
      simple: LEAST_REQUEST
```

### Retry Policy

```yaml
# retry-policy.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: model-registry
spec:
  hosts:
    - model-registry
  http:
    - route:
        - destination:
            host: model-registry
      retries:
        attempts: 3
        perTryTimeout: 5s
        retryOn: 5xx,reset,connect-failure,retriable-4xx
      timeout: 30s
```

## Security

### mTLS Configuration

```yaml
# mtls-policy.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: ml-platform
spec:
  mtls:
    mode: STRICT

---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: ml-inference-auth
  namespace: ml-platform
spec:
  selector:
    matchLabels:
      app: ml-inference
  rules:
    - from:
        - source:
            principals:
              - cluster.local/ns/api-gateway/sa/api-gateway
              - cluster.local/ns/ml-platform/sa/batch-processor
      to:
        - operation:
            methods: ["POST"]
            paths: ["/predict", "/batch-predict"]
```

### JWT Authentication

```yaml
# jwt-auth.yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: ml-api-jwt
  namespace: ml-platform
spec:
  selector:
    matchLabels:
      app: ml-inference
  jwtRules:
    - issuer: "https://auth.example.com"
      jwksUri: "https://auth.example.com/.well-known/jwks.json"
      audiences:
        - "ml-api"

---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: ml-platform
spec:
  selector:
    matchLabels:
      app: ml-inference
  rules:
    - from:
        - source:
            requestPrincipals: ["*"]
      when:
        - key: request.auth.claims[groups]
          values: ["ml-users", "ml-admins"]
```

## Observability

### Distributed Tracing

```yaml
# tracing-config.yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  tracing:
    - providers:
        - name: jaeger
      randomSamplingPercentage: 10
      customTags:
        model_version:
          header:
            name: x-model-version
        user_id:
          header:
            name: x-user-id
```

### Custom Metrics

```yaml
# custom-metrics.yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: ml-metrics
  namespace: ml-platform
spec:
  metrics:
    - providers:
        - name: prometheus
      overrides:
        - match:
            metric: REQUEST_COUNT
          tagOverrides:
            model_version:
              operation: UPSERT
              value: request.headers['x-model-version'] | 'unknown'
            prediction_type:
              operation: UPSERT
              value: request.headers['x-prediction-type'] | 'unknown'
```

### Grafana Dashboard for ML Services

```json
{
  "dashboard": {
    "title": "ML Service Mesh",
    "panels": [
      {
        "title": "Request Rate by Model Version",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(istio_requests_total{destination_service=\"ml-inference\"}[5m])) by (model_version)",
            "legendFormat": "{{model_version}}"
          }
        ]
      },
      {
        "title": "P99 Latency by Model",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, sum(rate(istio_request_duration_milliseconds_bucket{destination_service=\"ml-inference\"}[5m])) by (le, model_version))",
            "legendFormat": "{{model_version}}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(rate(istio_requests_total{destination_service=\"ml-inference\", response_code=~\"5.*\"}[5m])) / sum(rate(istio_requests_total{destination_service=\"ml-inference\"}[5m])) * 100"
          }
        ]
      }
    ]
  }
}
```

## Progressive Delivery

### Flagger Integration

```yaml
# flagger-canary.yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: ml-inference
  namespace: ml-platform
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ml-inference
  service:
    port: 8080
    targetPort: 8080
    gateways:
      - ml-gateway
    hosts:
      - ml.example.com
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99
        interval: 1m
      - name: request-duration
        thresholdRange:
          max: 500
        interval: 1m
    webhooks:
      - name: model-accuracy-check
        type: rollout
        url: http://ml-validator.ml-platform/validate
        timeout: 30s
        metadata:
          type: accuracy
          threshold: "0.95"
```

### Custom Metrics for ML

```yaml
# ml-metrics-template.yaml
apiVersion: flagger.app/v1beta1
kind: MetricTemplate
metadata:
  name: model-accuracy
  namespace: ml-platform
spec:
  provider:
    type: prometheus
    address: http://prometheus:9090
  query: |
    avg(ml_model_accuracy{
      deployment="{{ target }}",
      namespace="{{ namespace }}"
    })
```

## Rate Limiting

### Global Rate Limit

```yaml
# rate-limit.yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: ml-rate-limit
  namespace: istio-system
spec:
  workloadSelector:
    labels:
      app: ml-inference
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_INBOUND
        listener:
          filterChain:
            filter:
              name: envoy.filters.network.http_connection_manager
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.local_ratelimit
          typed_config:
            "@type": type.googleapis.com/udpa.type.v1.TypedStruct
            type_url: type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
            value:
              stat_prefix: http_local_rate_limiter
              token_bucket:
                max_tokens: 1000
                tokens_per_fill: 100
                fill_interval: 1s
              filter_enabled:
                runtime_key: local_rate_limit_enabled
                default_value:
                  numerator: 100
                  denominator: HUNDRED
```

## Service Entry for External Services

```yaml
# external-model-registry.yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: huggingface-hub
  namespace: ml-platform
spec:
  hosts:
    - huggingface.co
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL

---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: huggingface-hub
spec:
  host: huggingface.co
  trafficPolicy:
    tls:
      mode: SIMPLE
    connectionPool:
      tcp:
        maxConnections: 10
```

## Troubleshooting

### Debug Configuration

```bash
# Check proxy configuration
istioctl proxy-config routes deploy/ml-inference -n ml-platform

# Check cluster endpoints
istioctl proxy-config endpoints deploy/ml-inference -n ml-platform

# Analyze configuration issues
istioctl analyze -n ml-platform

# View proxy logs
kubectl logs -l app=ml-inference -c istio-proxy -n ml-platform
```

## Best Practices

1. **Progressive rollouts**: Use Flagger for automated canary
2. **Strict mTLS**: Encrypt all service-to-service traffic
3. **Fine-grained auth**: Use JWT and RBAC
4. **Circuit breakers**: Protect against cascading failures
5. **Observability**: Custom metrics for ML-specific KPIs
6. **Rate limiting**: Protect expensive inference endpoints
7. **Traffic mirroring**: Test new models safely
8. **Timeout tuning**: Account for model inference time

## Resources

- [Istio Documentation](https://istio.io/latest/docs/)
- [Flagger Progressive Delivery](https://flagger.app/)
- [Istio Security Best Practices](https://istio.io/latest/docs/ops/best-practices/security/)
- [Envoy Proxy](https://www.envoyproxy.io/)

---

*Questions about service mesh? [Let me know](mailto:jordan@jordananderson.us).*
