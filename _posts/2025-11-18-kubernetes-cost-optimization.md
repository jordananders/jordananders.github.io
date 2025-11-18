---
layout: default
title:  "Kubernetes Cost Optimization for AI/ML Workloads"
date:   2025-11-18 12:00:00
categories: DevOps Kubernetes FinOps CostOptimization
---

Kubernetes clusters can become expensive quickly, especially with GPU workloads for ML. Here's how to optimize costs while maintaining performance.

## Cost Visibility

### Kubecost Installation

```bash
# Install Kubecost
helm repo add kubecost https://kubecost.github.io/cost-analyzer/

helm install kubecost kubecost/cost-analyzer \
  --namespace kubecost \
  --create-namespace \
  --set kubecostToken="your-token" \
  --set prometheus.server.persistentVolume.enabled=false
```

### Custom Cost Metrics

```python
# cost_metrics.py
from prometheus_client import Gauge, start_http_server
import kubernetes

# Define metrics
namespace_cost = Gauge(
    'namespace_cost_hourly',
    'Hourly cost by namespace',
    ['namespace']
)

workload_cost = Gauge(
    'workload_cost_hourly',
    'Hourly cost by workload',
    ['namespace', 'workload', 'type']
)

gpu_cost = Gauge(
    'gpu_cost_hourly',
    'Hourly GPU cost',
    ['namespace', 'workload', 'gpu_type']
)

class CostCalculator:
    # Pricing (adjust for your cloud provider)
    PRICES = {
        'cpu_per_core_hour': 0.031,
        'memory_per_gb_hour': 0.004,
        'gpu_a100_per_hour': 2.93,
        'gpu_v100_per_hour': 2.48,
        'gpu_t4_per_hour': 0.35
    }

    def __init__(self):
        kubernetes.config.load_incluster_config()
        self.v1 = kubernetes.client.CoreV1Api()

    def calculate_pod_cost(self, pod) -> float:
        """Calculate hourly cost for a pod"""
        cost = 0

        for container in pod.spec.containers:
            resources = container.resources
            if not resources or not resources.requests:
                continue

            # CPU cost
            cpu_request = resources.requests.get('cpu', '0')
            cpu_cores = self._parse_cpu(cpu_request)
            cost += cpu_cores * self.PRICES['cpu_per_core_hour']

            # Memory cost
            memory_request = resources.requests.get('memory', '0')
            memory_gb = self._parse_memory(memory_request) / (1024**3)
            cost += memory_gb * self.PRICES['memory_per_gb_hour']

            # GPU cost
            gpu_count = int(resources.requests.get('nvidia.com/gpu', 0))
            if gpu_count > 0:
                gpu_type = pod.metadata.labels.get('gpu-type', 'a100')
                gpu_price = self.PRICES.get(
                    f'gpu_{gpu_type}_per_hour',
                    self.PRICES['gpu_a100_per_hour']
                )
                cost += gpu_count * gpu_price

        return cost

    def collect_metrics(self):
        """Collect cost metrics for all pods"""
        pods = self.v1.list_pod_for_all_namespaces()

        namespace_costs = {}
        workload_costs = {}

        for pod in pods.items:
            ns = pod.metadata.namespace
            cost = self.calculate_pod_cost(pod)

            # Aggregate by namespace
            namespace_costs[ns] = namespace_costs.get(ns, 0) + cost

            # Aggregate by workload
            workload = self._get_workload_name(pod)
            key = (ns, workload)
            workload_costs[key] = workload_costs.get(key, 0) + cost

        # Export metrics
        for ns, cost in namespace_costs.items():
            namespace_cost.labels(namespace=ns).set(cost)

        for (ns, workload), cost in workload_costs.items():
            workload_cost.labels(
                namespace=ns,
                workload=workload,
                type='deployment'
            ).set(cost)
```

## Right-Sizing

### Vertical Pod Autoscaler

```yaml
# vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: ml-inference-vpa
  namespace: ml-platform
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: ml-inference
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
      - containerName: ml-server
        minAllowed:
          cpu: 100m
          memory: 256Mi
        maxAllowed:
          cpu: 4
          memory: 8Gi
        controlledResources: ["cpu", "memory"]
```

### Goldilocks Recommendations

```bash
# Install Goldilocks
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
helm install goldilocks fairwinds-stable/goldilocks --namespace goldilocks

# Enable for namespace
kubectl label ns ml-platform goldilocks.fairwinds.com/enabled=true
```

### Resource Analysis Script

```python
# resource_analyzer.py
from kubernetes import client, config
import pandas as pd

class ResourceAnalyzer:
    def __init__(self):
        config.load_incluster_config()
        self.v1 = client.CoreV1Api()
        self.metrics = client.CustomObjectsApi()

    def analyze_utilization(self, namespace: str) -> pd.DataFrame:
        """Analyze resource utilization vs requests"""
        pods = self.v1.list_namespaced_pod(namespace)
        data = []

        for pod in pods.items:
            # Get current usage from metrics API
            try:
                metrics = self.metrics.get_namespaced_custom_object(
                    "metrics.k8s.io", "v1beta1", namespace,
                    "pods", pod.metadata.name
                )
            except:
                continue

            for container in pod.spec.containers:
                requests = container.resources.requests or {}

                # Find matching container metrics
                container_metrics = next(
                    (c for c in metrics['containers']
                     if c['name'] == container.name),
                    None
                )

                if not container_metrics:
                    continue

                cpu_request = self._parse_cpu(requests.get('cpu', '0'))
                cpu_usage = self._parse_cpu(container_metrics['usage']['cpu'])

                memory_request = self._parse_memory(requests.get('memory', '0'))
                memory_usage = self._parse_memory(
                    container_metrics['usage']['memory']
                )

                data.append({
                    'pod': pod.metadata.name,
                    'container': container.name,
                    'cpu_request': cpu_request,
                    'cpu_usage': cpu_usage,
                    'cpu_utilization': cpu_usage / cpu_request if cpu_request else 0,
                    'memory_request': memory_request,
                    'memory_usage': memory_usage,
                    'memory_utilization': memory_usage / memory_request if memory_request else 0
                })

        return pd.DataFrame(data)

    def get_recommendations(self, df: pd.DataFrame) -> list:
        """Generate right-sizing recommendations"""
        recommendations = []

        for _, row in df.iterrows():
            if row['cpu_utilization'] < 0.3:
                recommendations.append({
                    'pod': row['pod'],
                    'type': 'cpu',
                    'current': row['cpu_request'],
                    'recommended': row['cpu_usage'] * 1.5,
                    'savings_percent': (1 - row['cpu_utilization']) * 100
                })

            if row['memory_utilization'] < 0.3:
                recommendations.append({
                    'pod': row['pod'],
                    'type': 'memory',
                    'current': row['memory_request'],
                    'recommended': row['memory_usage'] * 1.5,
                    'savings_percent': (1 - row['memory_utilization']) * 100
                })

        return recommendations
```

## Spot Instances

### Karpenter Configuration

```yaml
# karpenter-provisioner.yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: ml-training-spot
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
    - key: node.kubernetes.io/instance-type
      operator: In
      values: ["p3.2xlarge", "p3.8xlarge", "g4dn.xlarge"]
    - key: topology.kubernetes.io/zone
      operator: In
      values: ["us-east-1a", "us-east-1b", "us-east-1c"]

  limits:
    resources:
      cpu: 1000
      nvidia.com/gpu: 100

  providerRef:
    name: default

  ttlSecondsAfterEmpty: 30
  ttlSecondsUntilExpired: 2592000  # 30 days

  # Consolidation
  consolidation:
    enabled: true

---
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: ml-inference-ondemand
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["on-demand"]
    - key: node.kubernetes.io/instance-type
      operator: In
      values: ["g4dn.xlarge", "g4dn.2xlarge"]

  taints:
    - key: nvidia.com/gpu
      value: "true"
      effect: NoSchedule

  labels:
    workload-type: inference
```

### Spot Instance Handling

```yaml
# spot-tolerant-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-training
  namespace: ml-platform
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: ml-training
    spec:
      nodeSelector:
        karpenter.sh/capacity-type: spot

      tolerations:
        - key: "karpenter.sh/spot"
          operator: "Exists"
          effect: "NoSchedule"

      # Handle spot interruption
      terminationGracePeriodSeconds: 120

      containers:
        - name: trainer
          image: myregistry/ml-trainer:v1
          lifecycle:
            preStop:
              exec:
                command:
                  - /bin/sh
                  - -c
                  - |
                    # Save checkpoint before termination
                    python save_checkpoint.py
          resources:
            requests:
              nvidia.com/gpu: "1"
            limits:
              nvidia.com/gpu: "1"
```

## Cluster Autoscaling

### Horizontal Pod Autoscaler

```yaml
# hpa-ml-inference.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ml-inference-hpa
  namespace: ml-platform
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ml-inference
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70

    # Custom metric for queue depth
    - type: External
      external:
        metric:
          name: inference_queue_depth
          selector:
            matchLabels:
              service: ml-inference
        target:
          type: AverageValue
          averageValue: "30"

  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
        - type: Pods
          value: 4
          periodSeconds: 15
      selectPolicy: Max
```

### KEDA for Event-Driven Scaling

```yaml
# keda-scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: ml-batch-processor
  namespace: ml-platform
spec:
  scaleTargetRef:
    name: ml-batch-processor
  minReplicaCount: 0
  maxReplicaCount: 50
  pollingInterval: 15
  cooldownPeriod: 300

  triggers:
    # Scale on SQS queue
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123456789/ml-jobs
        queueLength: "5"
        awsRegion: us-east-1

    # Scale on Prometheus metric
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: pending_ml_jobs
        threshold: "10"
        query: sum(ml_jobs_pending{status="queued"})
```

## GPU Optimization

### Time-Slicing

```yaml
# gpu-time-slicing.yaml
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
        resources:
          - name: nvidia.com/gpu
            replicas: 4
```

### Multi-Instance GPU (MIG)

```yaml
# mig-config.yaml
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
            "3g.20gb": 2
            "2g.10gb": 2
```

### GPU Scheduling Optimization

```python
# gpu_scheduler.py
class GPUScheduler:
    def __init__(self):
        self.cost_per_hour = {
            'a100': 2.93,
            'v100': 2.48,
            't4': 0.35
        }

    def optimize_gpu_allocation(self, jobs: list) -> dict:
        """Optimize GPU allocation for cost"""
        allocations = {}

        for job in sorted(jobs, key=lambda j: j['priority'], reverse=True):
            # Determine minimum GPU requirement
            min_gpu = self._get_min_gpu(job)

            # Find cheapest suitable GPU
            suitable = [
                gpu for gpu, cost in self.cost_per_hour.items()
                if self._can_run(job, gpu)
            ]

            if suitable:
                cheapest = min(suitable, key=lambda g: self.cost_per_hour[g])
                allocations[job['id']] = {
                    'gpu_type': cheapest,
                    'count': min_gpu,
                    'estimated_cost': self.cost_per_hour[cheapest] * min_gpu * job['estimated_hours']
                }

        return allocations

    def _can_run(self, job: dict, gpu_type: str) -> bool:
        """Check if job can run on GPU type"""
        requirements = job.get('gpu_requirements', {})

        if requirements.get('min_memory_gb', 0) > self._gpu_memory(gpu_type):
            return False

        return True
```

## Cost Policies

### Resource Quotas

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ml-team-quota
  namespace: ml-team
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 500Gi
    limits.cpu: "200"
    limits.memory: 1Ti
    requests.nvidia.com/gpu: "20"
    persistentvolumeclaims: "50"
    pods: "200"
```

### Limit Ranges

```yaml
# limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: ml-limits
  namespace: ml-platform
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 1Gi
      defaultRequest:
        cpu: 100m
        memory: 256Mi
      max:
        cpu: "8"
        memory: 32Gi
        nvidia.com/gpu: "4"
      min:
        cpu: 50m
        memory: 64Mi
```

## Best Practices

1. **Right-size resources**: Use VPA recommendations
2. **Use spot instances**: For fault-tolerant workloads
3. **Implement autoscaling**: HPA and cluster autoscaler
4. **GPU time-slicing**: Share GPUs for dev/test
5. **Set quotas**: Prevent runaway costs
6. **Monitor costs**: Use Kubecost or native tools
7. **Review regularly**: Monthly cost reviews
8. **Cleanup unused**: Delete idle resources

## Cost Optimization Checklist

- [ ] VPA installed and configured
- [ ] HPA for all production workloads
- [ ] Spot instances for training workloads
- [ ] Resource quotas per namespace
- [ ] GPU sharing for non-production
- [ ] Cost monitoring dashboards
- [ ] Automated cleanup of idle resources
- [ ] Monthly cost review process

## Resources

- [Kubecost Documentation](https://docs.kubecost.com/)
- [Karpenter Best Practices](https://karpenter.sh/docs/)
- [AWS EKS Cost Optimization](https://aws.amazon.com/blogs/containers/cost-optimization-for-kubernetes-on-aws/)
- [FinOps Foundation](https://www.finops.org/)

---

*Questions about Kubernetes cost optimization? [Let me know](mailto:jordan@jordananderson.us).*
