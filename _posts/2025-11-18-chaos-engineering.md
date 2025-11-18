---
layout: default
title:  "Chaos Engineering: Building Resilient Systems"
date:   2025-11-18 12:00:00
categories: DevOps Reliability ChaosEngineering
---

Chaos engineering proactively tests system resilience by injecting failures. For ML systems, this includes testing model fallbacks, GPU failures, and data pipeline disruptions.

## Chaos Engineering Principles

1. **Hypothesize about steady state**: Define what normal looks like
2. **Vary real-world events**: Inject realistic failures
3. **Run experiments in production**: Test where it matters
4. **Automate experiments**: Make chaos continuous
5. **Minimize blast radius**: Start small, expand gradually

## Chaos Mesh for Kubernetes

### Installation

```bash
# Install Chaos Mesh
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace chaos-mesh \
  --create-namespace \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock
```

### Pod Chaos Experiments

```yaml
# pod-kill-experiment.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: ml-service-pod-kill
  namespace: chaos-mesh
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - ml-platform
    labelSelectors:
      app: ml-inference
  scheduler:
    cron: "0 */4 * * *"  # Every 4 hours
```

```yaml
# pod-failure-experiment.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: ml-service-failure
spec:
  action: pod-failure
  mode: fixed-percent
  value: "30"
  duration: "60s"
  selector:
    namespaces:
      - ml-platform
    labelSelectors:
      app: ml-inference
```

### Network Chaos

```yaml
# network-delay.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: model-registry-latency
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - ml-platform
    labelSelectors:
      app: model-registry
  delay:
    latency: "200ms"
    jitter: "50ms"
    correlation: "50"
  duration: "5m"
  direction: to
  target:
    selector:
      namespaces:
        - ml-platform
      labelSelectors:
        app: ml-inference
```

```yaml
# network-partition.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: feature-store-partition
spec:
  action: partition
  mode: all
  selector:
    namespaces:
      - ml-platform
    labelSelectors:
      app: feature-store
  direction: both
  target:
    selector:
      namespaces:
        - ml-platform
      labelSelectors:
        app: ml-inference
  duration: "2m"
```

### IO Chaos

```yaml
# io-latency.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: IOChaos
metadata:
  name: model-loading-latency
spec:
  action: latency
  mode: one
  selector:
    namespaces:
      - ml-platform
    labelSelectors:
      app: ml-inference
  volumePath: /models
  path: "*.pt"
  delay: "500ms"
  percent: 50
  duration: "5m"
```

## Litmus Chaos

### Experiment Definition

```yaml
# litmus-experiment.yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: ml-chaos-engine
  namespace: ml-platform
spec:
  appinfo:
    appns: ml-platform
    applabel: app=ml-inference
    appkind: deployment
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-memory-hog
      spec:
        components:
          env:
            - name: MEMORY_CONSUMPTION
              value: "500"
            - name: TOTAL_CHAOS_DURATION
              value: "60"
            - name: CHAOS_INJECT_COMMAND
              value: "md5sum /dev/zero"

    - name: pod-cpu-hog
      spec:
        components:
          env:
            - name: CPU_CORES
              value: "2"
            - name: TOTAL_CHAOS_DURATION
              value: "60"
```

### Custom Experiment

```yaml
# custom-ml-experiment.yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosExperiment
metadata:
  name: model-corruption
  namespace: ml-platform
spec:
  definition:
    scope: Namespaced
    permissions:
      - apiGroups: [""]
        resources: ["pods"]
        verbs: ["get", "list", "delete"]
    image: "litmuschaos/go-runner:latest"
    args:
      - -c
      - ./experiments -name model-corruption
    env:
      - name: MODEL_PATH
        value: "/models/current"
      - name: CORRUPTION_PERCENTAGE
        value: "10"
      - name: TOTAL_CHAOS_DURATION
        value: "30"
```

## ML-Specific Chaos Tests

### Model Fallback Testing

```python
# chaos_ml_tests.py
import asyncio
from chaos_toolkit import ChaosExperiment

class MLChaosExperiment:
    def __init__(self, config):
        self.config = config

    async def test_model_fallback(self):
        """Test graceful degradation when primary model fails"""
        experiment = ChaosExperiment(
            title="Model Fallback Test",
            description="Verify fallback model activates when primary fails"
        )

        # Define steady state
        experiment.steady_state_hypothesis(
            title="Service responds with predictions",
            probes=[
                {
                    "name": "prediction-available",
                    "type": "http",
                    "tolerance": 200,
                    "provider": {
                        "type": "http",
                        "url": f"{self.config.service_url}/predict",
                        "method": "POST"
                    }
                }
            ]
        )

        # Define chaos action
        experiment.method([
            {
                "name": "kill-primary-model",
                "type": "action",
                "provider": {
                    "type": "python",
                    "module": "ml_chaos.actions",
                    "func": "corrupt_model_file",
                    "arguments": {
                        "model_path": "/models/primary"
                    }
                }
            }
        ])

        # Verify fallback
        experiment.rollbacks([
            {
                "name": "restore-primary-model",
                "type": "action",
                "provider": {
                    "type": "python",
                    "module": "ml_chaos.actions",
                    "func": "restore_model_file",
                    "arguments": {
                        "model_path": "/models/primary"
                    }
                }
            }
        ])

        return await experiment.run()

    async def test_feature_store_outage(self):
        """Test behavior when feature store is unavailable"""
        # Kill feature store pods
        await self._kill_pods("app=feature-store")

        # Verify service uses cached features
        response = await self._make_prediction_request()
        assert response.status_code == 200
        assert response.json()["using_cached_features"] == True

        # Verify degraded accuracy is acceptable
        accuracy = await self._measure_accuracy()
        assert accuracy > 0.85, f"Accuracy {accuracy} below threshold"

    async def test_gpu_failure(self):
        """Test graceful handling of GPU failures"""
        # Simulate GPU memory exhaustion
        await self._inject_gpu_fault()

        # Verify CPU fallback
        response = await self._make_prediction_request()
        assert response.status_code == 200

        # Verify latency is within acceptable range
        latency = response.elapsed.total_seconds()
        assert latency < 5.0, f"Latency {latency}s exceeds threshold"
```

### Data Pipeline Chaos

```python
# data_pipeline_chaos.py
class DataPipelineChaos:
    def __init__(self, kafka_config, redis_config):
        self.kafka = kafka_config
        self.redis = redis_config

    async def test_kafka_partition_loss(self):
        """Test handling of Kafka partition unavailability"""
        # Get current consumer lag
        initial_lag = await self._get_consumer_lag()

        # Kill Kafka broker
        await self._kill_broker(broker_id=1)

        # Wait for rebalance
        await asyncio.sleep(30)

        # Verify consumers rebalanced
        new_lag = await self._get_consumer_lag()

        # Lag should recover
        assert new_lag < initial_lag * 2

        # Restore broker
        await self._start_broker(broker_id=1)

    async def test_redis_failover(self):
        """Test Redis Sentinel failover"""
        # Get current master
        original_master = await self._get_redis_master()

        # Kill master
        await self._kill_redis_node(original_master)

        # Wait for failover
        await asyncio.sleep(10)

        # Verify new master elected
        new_master = await self._get_redis_master()
        assert new_master != original_master

        # Verify feature cache still works
        features = await self._get_features("user_123")
        assert features is not None

    async def test_data_corruption(self):
        """Test detection of corrupted training data"""
        # Inject corrupted records
        await self._inject_bad_records(count=100)

        # Trigger training pipeline
        job_id = await self._start_training()

        # Verify pipeline detects corruption
        status = await self._wait_for_job(job_id)
        assert status == "failed"
        assert "data_validation_error" in status.error
```

## GameDay Automation

### GameDay Framework

```python
# gameday.py
import asyncio
from datetime import datetime
from dataclasses import dataclass

@dataclass
class GameDayScenario:
    name: str
    description: str
    chaos_actions: list
    success_criteria: list
    duration_minutes: int

class GameDayOrchestrator:
    def __init__(self, slack_webhook, pagerduty_key):
        self.slack = slack_webhook
        self.pagerduty = pagerduty_key
        self.results = []

    async def run_gameday(self, scenarios: list[GameDayScenario]):
        """Run a series of chaos scenarios"""
        await self._notify_start()

        for scenario in scenarios:
            result = await self._run_scenario(scenario)
            self.results.append(result)

            if not result.passed:
                await self._notify_failure(scenario, result)

        await self._notify_completion()
        return self._generate_report()

    async def _run_scenario(self, scenario: GameDayScenario):
        """Execute a single scenario"""
        start_time = datetime.now()

        # Collect baseline metrics
        baseline = await self._collect_metrics()

        # Execute chaos actions
        for action in scenario.chaos_actions:
            await self._execute_action(action)

        # Wait for scenario duration
        await asyncio.sleep(scenario.duration_minutes * 60)

        # Collect post-chaos metrics
        post_chaos = await self._collect_metrics()

        # Evaluate success criteria
        passed = all(
            self._evaluate_criterion(criterion, baseline, post_chaos)
            for criterion in scenario.success_criteria
        )

        # Cleanup
        await self._cleanup_chaos()

        return GameDayResult(
            scenario=scenario.name,
            passed=passed,
            duration=datetime.now() - start_time,
            baseline_metrics=baseline,
            post_chaos_metrics=post_chaos
        )

    def _generate_report(self) -> str:
        """Generate GameDay report"""
        report = ["# GameDay Report", ""]

        for result in self.results:
            status = "PASS" if result.passed else "FAIL"
            report.append(f"## {result.scenario}: {status}")
            report.append(f"Duration: {result.duration}")
            report.append("")

            # Metrics comparison
            report.append("### Metrics")
            for metric in result.baseline_metrics:
                baseline = result.baseline_metrics[metric]
                post = result.post_chaos_metrics[metric]
                change = ((post - baseline) / baseline) * 100
                report.append(f"- {metric}: {baseline:.2f} -> {post:.2f} ({change:+.1f}%)")

        return "\n".join(report)
```

### Automated GameDay Schedule

```yaml
# gameday-schedule.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: weekly-gameday
  namespace: chaos-mesh
spec:
  schedule: "0 14 * * 3"  # Wednesday 2 PM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: gameday-runner
              image: myregistry/gameday-runner:latest
              env:
                - name: SCENARIOS
                  value: "pod-kill,network-latency,disk-full"
                - name: SLACK_WEBHOOK
                  valueFrom:
                    secretKeyRef:
                      name: chaos-secrets
                      key: slack-webhook
          restartPolicy: Never
```

## Observability During Chaos

### Chaos Dashboard

```python
# chaos_dashboard.py
from prometheus_client import Gauge, Counter
import grafana_api

chaos_experiment_running = Gauge(
    'chaos_experiment_running',
    'Currently running chaos experiment',
    ['experiment_name', 'target']
)

chaos_experiment_total = Counter(
    'chaos_experiment_total',
    'Total chaos experiments run',
    ['experiment_name', 'result']
)

class ChaosDashboard:
    def __init__(self, grafana_url, api_key):
        self.grafana = grafana_api.GrafanaApi(
            auth=(api_key,),
            host=grafana_url
        )

    def create_annotations(self, experiment_name: str,
                          start_time: int, end_time: int):
        """Add Grafana annotations for chaos window"""
        self.grafana.annotations.post_annotation({
            "time": start_time * 1000,
            "timeEnd": end_time * 1000,
            "tags": ["chaos", experiment_name],
            "text": f"Chaos experiment: {experiment_name}"
        })
```

## Best Practices

1. **Start small**: Begin with non-production, expand gradually
2. **Define success criteria**: Know what "working" means
3. **Automate rollback**: Always have a way to stop
4. **Monitor everything**: Observe during chaos
5. **Document learnings**: Share findings broadly
6. **Regular cadence**: Make chaos routine
7. **Include on-call**: Test incident response
8. **Test ML specifics**: Model fallbacks, feature stores

## Chaos Testing Checklist

- [ ] Define steady state hypothesis
- [ ] Identify blast radius and containment
- [ ] Set up monitoring dashboards
- [ ] Configure automatic rollback
- [ ] Notify relevant teams
- [ ] Document expected behavior
- [ ] Plan for escalation
- [ ] Schedule post-mortem

## Resources

- [Principles of Chaos Engineering](https://principlesofchaos.org/)
- [Chaos Mesh Documentation](https://chaos-mesh.org/docs/)
- [Litmus Chaos](https://litmuschaos.io/)
- [Netflix Chaos Engineering](https://netflixtechblog.com/tagged/chaos-engineering)

---

*Questions about chaos engineering? [Let me know](mailto:jordan@jordananderson.us).*
