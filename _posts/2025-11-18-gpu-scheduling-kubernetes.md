---
layout: default
title:  "GPU Scheduling in Kubernetes for AI Workloads"
date:   2025-11-18 11:00:00
categories: DevOps Kubernetes GPU AI
---

Over two-thirds of organizations say Kubernetes is key to their AI strategy. GPU scheduling is critical for ML workloads - here's how to configure NVIDIA GPUs in Kubernetes effectively.

## NVIDIA GPU Operator

The GPU Operator automates GPU management across any infrastructure:

```bash
# Add NVIDIA Helm repo
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

# Install GPU Operator
helm install gpu-operator nvidia/gpu-operator \
    --namespace gpu-operator \
    --create-namespace \
    --set driver.enabled=true \
    --set toolkit.enabled=true \
    --set devicePlugin.enabled=true \
    --set dcgmExporter.enabled=true
```

### Components Installed

- **NVIDIA drivers**: GPU kernel drivers
- **Container toolkit**: Runtime support for GPU containers
- **Device plugin**: Exposes GPUs to Kubernetes scheduler
- **DCGM exporter**: GPU metrics for Prometheus

## Basic GPU Scheduling

### Request GPU Resources

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-training
spec:
  containers:
    - name: trainer
      image: pytorch/pytorch:latest
      resources:
        limits:
          nvidia.com/gpu: 1  # Request 1 GPU
      command: ["python", "train.py"]
  nodeSelector:
    nvidia.com/gpu.present: "true"
```

### Multi-GPU Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: distributed-training
spec:
  containers:
    - name: trainer
      image: pytorch/pytorch:latest
      resources:
        limits:
          nvidia.com/gpu: 4  # Request 4 GPUs
      env:
        - name: CUDA_VISIBLE_DEVICES
          value: "0,1,2,3"
```

## GPU Sharing Strategies

### Problem

By default, GPUs cannot be overcommitted in Kubernetes. One GPU = one pod. This leads to poor utilization for:
- Low-batch inference
- Jupyter notebooks
- CI/CD testing
- Small model training

### Solution 1: Time-Slicing

Multiple pods share one GPU by time-slicing:

```yaml
# ConfigMap for time-slicing
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config
  namespace: gpu-operator
data:
  any: |-
    version: v1
    sharing:
      timeSlicing:
        renameByDefault: false
        failRequestsGreaterThanOne: false
        resources:
          - name: nvidia.com/gpu
            replicas: 4
```

```bash
# Apply to GPU Operator
helm upgrade gpu-operator nvidia/gpu-operator \
    --namespace gpu-operator \
    --set devicePlugin.config.name=time-slicing-config
```

Now each physical GPU appears as 4 virtual GPUs:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: inference-1
spec:
  containers:
    - name: model
      image: my-inference:latest
      resources:
        limits:
          nvidia.com/gpu: 1  # Gets 1/4 of physical GPU
```

**Limitations**:
- No memory isolation
- No fault isolation
- Context switching overhead

**Best for**: Inference, notebooks, CI/CD

### Solution 2: Multi-Instance GPU (MIG)

Partition GPU into isolated instances (A100, H100 only):

```yaml
# MIG configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: mig-config
  namespace: gpu-operator
data:
  config.yaml: |
    version: v1
    mig-configs:
      all-balanced:
        - devices: all
          mig-enabled: true
          mig-devices:
            "1g.5gb": 7
```

Request specific MIG slice:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: inference-pod
spec:
  containers:
    - name: model
      image: my-model:latest
      resources:
        limits:
          nvidia.com/mig-1g.5gb: 1  # 1 MIG slice
```

**MIG Profiles** (A100 80GB):

| Profile | Memory | Instances |
|---------|--------|-----------|
| 1g.10gb | 10GB | 7 |
| 2g.20gb | 20GB | 3 |
| 3g.40gb | 40GB | 2 |
| 7g.80gb | 80GB | 1 |

**Best for**: Production inference, multi-tenant clusters

### Solution 3: NVIDIA vGPU

Requires NVIDIA AI Enterprise license:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vgpu-workload
spec:
  containers:
    - name: app
      resources:
        limits:
          nvidia.com/vgpu: 1
```

**Best for**: VMs, maximum isolation, enterprise

## Node Labeling and Affinity

### Label GPU Nodes

```bash
# Add custom labels
kubectl label nodes gpu-node-1 gpu-type=a100
kubectl label nodes gpu-node-2 gpu-type=t4
```

### Schedule by GPU Type

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: training-job
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: nvidia.com/gpu.product
                operator: In
                values:
                  - NVIDIA-A100-SXM4-80GB
                  - NVIDIA-A100-SXM4-40GB
  containers:
    - name: trainer
      resources:
        limits:
          nvidia.com/gpu: 8
```

## Priority and Preemption

### Priority Classes

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-training
value: 1000000
globalDefault: false
description: "High priority for production training jobs"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority-experiment
value: 100
preemptionPolicy: Never  # Don't preempt others
description: "Low priority for experiments"
```

### Use in Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: important-training
spec:
  priorityClassName: high-priority-training
  containers:
    - name: trainer
      resources:
        limits:
          nvidia.com/gpu: 4
```

## Resource Quotas

### Limit GPU Usage per Namespace

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: gpu-quota
  namespace: ml-team
spec:
  hard:
    requests.nvidia.com/gpu: "8"
    limits.nvidia.com/gpu: "8"
```

### Limit per User (with LimitRange)

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: gpu-limits
  namespace: ml-team
spec:
  limits:
    - type: Container
      max:
        nvidia.com/gpu: "2"  # Max 2 GPUs per container
      default:
        nvidia.com/gpu: "1"
```

## Monitoring GPU Usage

### DCGM Exporter Metrics

```yaml
# ServiceMonitor for Prometheus
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: dcgm-exporter
spec:
  selector:
    matchLabels:
      app: nvidia-dcgm-exporter
  endpoints:
    - port: metrics
      interval: 15s
```

### Key Metrics

```promql
# GPU utilization
DCGM_FI_DEV_GPU_UTIL

# Memory usage
DCGM_FI_DEV_FB_USED / DCGM_FI_DEV_FB_TOTAL

# Temperature
DCGM_FI_DEV_GPU_TEMP

# Power usage
DCGM_FI_DEV_POWER_USAGE
```

### Grafana Dashboard

```json
{
  "panels": [
    {
      "title": "GPU Utilization",
      "targets": [{
        "expr": "avg(DCGM_FI_DEV_GPU_UTIL) by (gpu, kubernetes_node)"
      }]
    },
    {
      "title": "GPU Memory Used",
      "targets": [{
        "expr": "DCGM_FI_DEV_FB_USED / 1024"
      }]
    }
  ]
}
```

## Autoscaling GPU Workloads

### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: inference-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: inference-service
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Pods
      pods:
        metric:
          name: gpu_utilization
        target:
          type: AverageValue
          averageValue: "80"
```

### Cluster Autoscaler for GPU Nodes

```yaml
# Node pool configuration (GKE example)
apiVersion: container.gke.io/v1
kind: NodePool
metadata:
  name: gpu-pool
spec:
  autoscaling:
    enabled: true
    minNodeCount: 0
    maxNodeCount: 10
  config:
    machineType: n1-standard-8
    accelerators:
      - acceleratorType: nvidia-tesla-t4
        acceleratorCount: 1
```

## Best Practices

### 1. Right-Size GPU Requests

```yaml
# Bad: Over-provisioning
resources:
  limits:
    nvidia.com/gpu: 4  # Only using 1

# Good: Match to workload
resources:
  limits:
    nvidia.com/gpu: 1
```

### 2. Use Spot/Preemptible for Training

```yaml
spec:
  nodeSelector:
    cloud.google.com/gke-spot: "true"
  tolerations:
    - key: "cloud.google.com/gke-spot"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
```

### 3. Set Resource Limits

```yaml
resources:
  requests:
    nvidia.com/gpu: 1
    memory: "16Gi"
    cpu: "4"
  limits:
    nvidia.com/gpu: 1
    memory: "32Gi"
    cpu: "8"
```

### 4. Implement Checkpointing

```python
# Save checkpoints for preemptible workloads
if epoch % 10 == 0:
    torch.save({
        'epoch': epoch,
        'model_state_dict': model.state_dict(),
        'optimizer_state_dict': optimizer.state_dict(),
    }, f'/checkpoints/epoch_{epoch}.pt')
```

## Resources

- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html)
- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [NVIDIA Time-Slicing Guide](https://developer.nvidia.com/blog/improving-gpu-utilization-in-kubernetes/)
- [Run:ai Platform](https://www.run.ai/)

---

*Questions about GPU scheduling in Kubernetes? [Let me know](mailto:jordan@jordananderson.us).*
