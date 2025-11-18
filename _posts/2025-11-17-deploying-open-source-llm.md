---
layout: default
title:  "Deploying Open Source LLMs in Production"
date:   2025-11-17 16:00:00
categories: AI LLM Production Kubernetes
---

Deploying open source LLMs requires careful attention to inference performance, scaling, and resource management. vLLM and TGI have emerged as the leading serving frameworks. Here's how to deploy them in production.

## Framework Comparison

| Feature | vLLM | TGI | Ollama |
|---------|------|-----|--------|
| Throughput | Highest (24x HF) | High (3.5x HF) | Good |
| PagedAttention | Yes | Yes | No |
| Continuous Batching | Yes | Yes | No |
| Production Ready | Yes | Yes | Limited |
| Best For | High throughput | HF ecosystem | Local dev |

## vLLM Deployment

### Basic Docker Deployment

```bash
# Pull and run
docker run --runtime nvidia --gpus all \
    -v ~/.cache/huggingface:/root/.cache/huggingface \
    -p 8000:8000 \
    vllm/vllm-openai:latest \
    --model meta-llama/Llama-3.1-8B-Instruct \
    --max-model-len 8192
```

### vLLM Configuration

```python
from vllm import LLM, SamplingParams

# Initialize with performance tuning
llm = LLM(
    model="meta-llama/Llama-3.1-8B-Instruct",
    tensor_parallel_size=1,        # GPUs for tensor parallelism
    gpu_memory_utilization=0.9,    # GPU memory usage
    max_model_len=8192,            # Max context length
    enable_prefix_caching=True,    # Cache common prefixes
    enforce_eager=False,           # Enable CUDA graphs
    dtype="auto"                   # Auto-detect dtype
)

# Sampling parameters
sampling_params = SamplingParams(
    temperature=0.7,
    top_p=0.95,
    max_tokens=512
)

# Generate
outputs = llm.generate(
    ["Write a hello world in Python"],
    sampling_params
)
```

### OpenAI-Compatible Server

```bash
# Start server with OpenAI API compatibility
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-8B-Instruct \
    --host 0.0.0.0 \
    --port 8000 \
    --max-model-len 8192 \
    --gpu-memory-utilization 0.9 \
    --enable-prefix-caching
```

```python
# Client usage (same as OpenAI)
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-needed"
)

response = client.chat.completions.create(
    model="meta-llama/Llama-3.1-8B-Instruct",
    messages=[
        {"role": "user", "content": "Hello!"}
    ]
)
```

## Kubernetes Deployment

### Basic Deployment

```yaml
# vllm-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-llama
  labels:
    app: vllm-llama
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vllm-llama
  template:
    metadata:
      labels:
        app: vllm-llama
    spec:
      containers:
      - name: vllm
        image: vllm/vllm-openai:latest
        args:
          - "--model"
          - "meta-llama/Llama-3.1-8B-Instruct"
          - "--max-model-len"
          - "8192"
          - "--gpu-memory-utilization"
          - "0.9"
        ports:
        - containerPort: 8000
        resources:
          limits:
            nvidia.com/gpu: 1
          requests:
            memory: "32Gi"
            cpu: "8"
        volumeMounts:
        - name: model-cache
          mountPath: /root/.cache/huggingface
        env:
        - name: HUGGING_FACE_HUB_TOKEN
          valueFrom:
            secretKeyRef:
              name: hf-secret
              key: token
      volumes:
      - name: model-cache
        persistentVolumeClaim:
          claimName: model-cache-pvc

---
apiVersion: v1
kind: Service
metadata:
  name: vllm-service
spec:
  selector:
    app: vllm-llama
  ports:
  - port: 8000
    targetPort: 8000
  type: ClusterIP
```

### Horizontal Pod Autoscaler

```yaml
# vllm-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: vllm-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vllm-llama
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: gpu_utilization
      target:
        type: AverageValue
        averageValue: "80"
```

### KEDA Autoscaling

```yaml
# vllm-keda.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: vllm-scaledobject
spec:
  scaleTargetRef:
    name: vllm-llama
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus:9090
      metricName: vllm_request_queue_size
      query: sum(vllm_num_requests_waiting{model="llama"})
      threshold: "100"
```

## Multi-GPU Deployment

### Tensor Parallelism

```yaml
# For 70B model across 4 GPUs
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-llama-70b
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: vllm
        image: vllm/vllm-openai:latest
        args:
          - "--model"
          - "meta-llama/Llama-3.1-70B-Instruct"
          - "--tensor-parallel-size"
          - "4"
          - "--max-model-len"
          - "8192"
        resources:
          limits:
            nvidia.com/gpu: 4
          requests:
            memory: "160Gi"
            cpu: "32"
```

### Pipeline Parallelism

```bash
# For even larger models
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-405B-Instruct \
    --tensor-parallel-size 4 \
    --pipeline-parallel-size 2 \
    --max-model-len 4096
```

## Text Generation Inference (TGI)

### Docker Deployment

```bash
docker run --gpus all -p 8080:80 \
    -v $PWD/data:/data \
    ghcr.io/huggingface/text-generation-inference:latest \
    --model-id meta-llama/Llama-3.1-8B-Instruct \
    --max-total-tokens 8192 \
    --max-input-length 4096 \
    --max-batch-prefill-tokens 4096
```

### Kubernetes with TGI

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tgi-llama
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: tgi
        image: ghcr.io/huggingface/text-generation-inference:latest
        args:
          - "--model-id"
          - "meta-llama/Llama-3.1-8B-Instruct"
          - "--max-total-tokens"
          - "8192"
          - "--quantize"
          - "bitsandbytes-nf4"  # 4-bit quantization
        ports:
        - containerPort: 80
        resources:
          limits:
            nvidia.com/gpu: 1
```

## Performance Optimization

### vLLM Tuning

```bash
# Optimize for throughput
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-8B-Instruct \
    --max-model-len 8192 \
    --gpu-memory-utilization 0.95 \
    --enable-prefix-caching \
    --max-num-batched-tokens 32768 \
    --max-num-seqs 256

# Optimize for latency
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-8B-Instruct \
    --max-model-len 4096 \
    --gpu-memory-utilization 0.8 \
    --max-num-seqs 64
```

### Quantization

```bash
# AWQ quantization for vLLM
python -m vllm.entrypoints.openai.api_server \
    --model TheBloke/Llama-2-70B-Chat-AWQ \
    --quantization awq \
    --tensor-parallel-size 2

# GPTQ quantization
python -m vllm.entrypoints.openai.api_server \
    --model TheBloke/Llama-2-70B-GPTQ \
    --quantization gptq \
    --tensor-parallel-size 2
```

## Load Balancing

### Nginx Configuration

```nginx
upstream vllm_servers {
    least_conn;
    server vllm-1:8000;
    server vllm-2:8000;
    server vllm-3:8000;
}

server {
    listen 80;

    location /v1 {
        proxy_pass http://vllm_servers;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_connect_timeout 300s;
        proxy_read_timeout 300s;
    }
}
```

### Kubernetes Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vllm-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
spec:
  rules:
  - host: llm.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: vllm-service
            port:
              number: 8000
```

## Monitoring

### Prometheus Metrics

```yaml
# prometheus-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: vllm-monitor
spec:
  selector:
    matchLabels:
      app: vllm-llama
  endpoints:
  - port: metrics
    interval: 15s
```

### Grafana Dashboard

```json
{
  "panels": [
    {
      "title": "Request Throughput",
      "targets": [
        {
          "expr": "rate(vllm_request_success_total[5m])"
        }
      ]
    },
    {
      "title": "GPU Utilization",
      "targets": [
        {
          "expr": "avg(vllm_gpu_cache_usage_perc)"
        }
      ]
    },
    {
      "title": "Queue Size",
      "targets": [
        {
          "expr": "vllm_num_requests_waiting"
        }
      ]
    },
    {
      "title": "Token Throughput",
      "targets": [
        {
          "expr": "rate(vllm_generation_tokens_total[5m])"
        }
      ]
    }
  ]
}
```

## Health Checks

```yaml
# Liveness and readiness probes
containers:
- name: vllm
  livenessProbe:
    httpGet:
      path: /health
      port: 8000
    initialDelaySeconds: 60
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 3
  readinessProbe:
    httpGet:
      path: /health
      port: 8000
    initialDelaySeconds: 30
    periodSeconds: 5
    timeoutSeconds: 3
```

## Cost Optimization

### GPU Selection

| Model Size | Recommended GPU | VRAM Needed | AWS Instance |
|------------|----------------|-------------|--------------|
| 7-8B | 1x A10G | 24GB | g5.xlarge |
| 13B | 1x A10G (4-bit) | 24GB | g5.xlarge |
| 70B | 2x A100 80GB | 140GB | p4d.24xlarge |
| 70B (4-bit) | 1x A100 80GB | 40GB | p4de.24xlarge |

### Spot Instances

```yaml
# EKS node pool with spot instances
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: gpu-spot
spec:
  requirements:
    - key: node.kubernetes.io/instance-type
      operator: In
      values: ["g5.xlarge", "g5.2xlarge"]
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
  limits:
    resources:
      nvidia.com/gpu: 20
```

## Production Checklist

- [ ] Model fits in GPU memory (or use quantization)
- [ ] Health checks configured
- [ ] Autoscaling enabled
- [ ] Monitoring and alerting set up
- [ ] Load balancer configured
- [ ] Model cache persistent volume
- [ ] Request timeouts set appropriately
- [ ] Rate limiting in place
- [ ] Graceful shutdown handling

## Resources

- [vLLM Documentation](https://docs.vllm.ai/)
- [TGI Documentation](https://huggingface.co/docs/text-generation-inference)
- [vLLM Production Stack](https://github.com/vllm-project/vllm-production-stack)
- [KServe LLM Guide](https://kserve.github.io/website/)

---

*Questions about deploying open source LLMs? [Let me know](mailto:jordan@jordananderson.us).*
