---
layout: default
title:  "Site Reliability Engineering: SLOs, Error Budgets, and Toil Reduction"
date:   2025-11-18 12:00:00
categories: DevOps SRE Reliability SLOs
---

SRE brings software engineering to operations. For ML systems, this means defining SLOs for model performance, managing inference error budgets, and automating ML operations toil.

## SLI/SLO/SLA Framework

### Definitions

- **SLI (Service Level Indicator)**: Quantitative measure of service behavior
- **SLO (Service Level Objective)**: Target value for SLI
- **SLA (Service Level Agreement)**: Contract with consequences
- **Error Budget**: Allowed failures (100% - SLO)

### ML-Specific SLIs

```python
# sli_definitions.py
from dataclasses import dataclass
from enum import Enum

class SLIType(Enum):
    AVAILABILITY = "availability"
    LATENCY = "latency"
    QUALITY = "quality"
    FRESHNESS = "freshness"

@dataclass
class SLI:
    name: str
    type: SLIType
    good_events_query: str
    total_events_query: str
    description: str

# Define ML service SLIs
ML_SLIS = [
    SLI(
        name="inference_availability",
        type=SLIType.AVAILABILITY,
        good_events_query="""
            sum(rate(ml_inference_requests_total{status="success"}[5m]))
        """,
        total_events_query="""
            sum(rate(ml_inference_requests_total[5m]))
        """,
        description="Percentage of successful inference requests"
    ),
    SLI(
        name="inference_latency",
        type=SLIType.LATENCY,
        good_events_query="""
            sum(rate(ml_inference_latency_bucket{le="0.1"}[5m]))
        """,
        total_events_query="""
            sum(rate(ml_inference_latency_count[5m]))
        """,
        description="Percentage of requests under 100ms"
    ),
    SLI(
        name="model_quality",
        type=SLIType.QUALITY,
        good_events_query="""
            sum(rate(ml_predictions_correct_total[5m]))
        """,
        total_events_query="""
            sum(rate(ml_predictions_total[5m]))
        """,
        description="Model prediction accuracy"
    ),
    SLI(
        name="feature_freshness",
        type=SLIType.FRESHNESS,
        good_events_query="""
            sum(rate(feature_age_seconds_bucket{le="3600"}[5m]))
        """,
        total_events_query="""
            sum(rate(feature_age_seconds_count[5m]))
        """,
        description="Features updated within last hour"
    )
]
```

## SLO Definition

### Sloth Configuration

```yaml
# slos.yaml
version: prometheus/v1
service: ml-inference
labels:
  team: ml-platform
  tier: "1"

slos:
  - name: availability
    objective: 99.9
    description: "ML inference service availability"
    sli:
      events:
        error_query: |
          sum(rate(ml_inference_requests_total{status=~"5.."}[{{.window}}]))
        total_query: |
          sum(rate(ml_inference_requests_total[{{.window}}]))
    alerting:
      name: MLInferenceAvailability
      labels:
        severity: critical
      annotations:
        summary: "ML inference availability below SLO"
      page_alert:
        labels:
          severity: critical
      ticket_alert:
        labels:
          severity: warning

  - name: latency
    objective: 99
    description: "Inference latency under 100ms"
    sli:
      events:
        error_query: |
          sum(rate(ml_inference_latency_count[{{.window}}])) -
          sum(rate(ml_inference_latency_bucket{le="0.1"}[{{.window}}]))
        total_query: |
          sum(rate(ml_inference_latency_count[{{.window}}]))
    alerting:
      name: MLInferenceLatency
      labels:
        severity: warning

  - name: model-accuracy
    objective: 95
    description: "Model prediction accuracy"
    sli:
      events:
        error_query: |
          sum(rate(ml_predictions_total[{{.window}}])) -
          sum(rate(ml_predictions_correct_total[{{.window}}]))
        total_query: |
          sum(rate(ml_predictions_total[{{.window}}]))
    alerting:
      name: MLModelAccuracy
      labels:
        severity: warning
```

### Pyrra SLO Management

```yaml
# pyrra-slo.yaml
apiVersion: pyrra.dev/v1alpha1
kind: ServiceLevelObjective
metadata:
  name: ml-inference-availability
  namespace: ml-platform
  labels:
    prometheus: k8s
    role: alert-rules
spec:
  target: "99.9"
  window: 28d
  description: "ML inference service availability"
  indicator:
    ratio:
      errors:
        metric: ml_inference_requests_total{status=~"5.."}
      total:
        metric: ml_inference_requests_total
  alerting:
    disabled: false
```

## Error Budget Management

### Error Budget Calculator

```python
# error_budget.py
from datetime import datetime, timedelta
from prometheus_api_client import PrometheusConnect

class ErrorBudgetManager:
    def __init__(self, prometheus_url: str):
        self.prom = PrometheusConnect(url=prometheus_url)

    def calculate_budget(self, slo_name: str, target: float,
                        window_days: int = 28) -> dict:
        """Calculate error budget status"""
        # Get SLI value
        sli_value = self._get_sli_value(slo_name, window_days)

        # Calculate budget
        total_budget = (1 - target / 100) * window_days * 24 * 60  # minutes
        consumed = (1 - sli_value / 100) * window_days * 24 * 60
        remaining = total_budget - consumed

        return {
            'slo_name': slo_name,
            'target': target,
            'current_sli': sli_value,
            'total_budget_minutes': total_budget,
            'consumed_minutes': consumed,
            'remaining_minutes': remaining,
            'remaining_percent': (remaining / total_budget) * 100 if total_budget > 0 else 0,
            'burn_rate': consumed / (window_days * 24 * 60) if window_days > 0 else 0
        }

    def get_budget_forecast(self, slo_name: str, target: float) -> dict:
        """Forecast when error budget will be exhausted"""
        # Get current burn rate
        burn_rate_1h = self._get_burn_rate(slo_name, hours=1)
        burn_rate_6h = self._get_burn_rate(slo_name, hours=6)

        budget = self.calculate_budget(slo_name, target)
        remaining = budget['remaining_minutes']

        # Forecast exhaustion
        if burn_rate_1h > 0:
            hours_to_exhaustion = remaining / (burn_rate_1h * 60)
        else:
            hours_to_exhaustion = float('inf')

        return {
            'burn_rate_1h': burn_rate_1h,
            'burn_rate_6h': burn_rate_6h,
            'hours_to_exhaustion': hours_to_exhaustion,
            'exhaustion_time': datetime.utcnow() + timedelta(hours=hours_to_exhaustion)
                if hours_to_exhaustion != float('inf') else None
        }

    def _get_burn_rate(self, slo_name: str, hours: int) -> float:
        """Calculate current burn rate"""
        query = f"""
            1 - (
                sum(rate(ml_inference_requests_total{{status="success"}}[{hours}h])) /
                sum(rate(ml_inference_requests_total[{hours}h]))
            )
        """
        result = self.prom.custom_query(query)
        return float(result[0]['value'][1]) if result else 0
```

### Multi-Window Burn Rate Alerts

```yaml
# burn-rate-alerts.yaml
groups:
  - name: slo-burn-rate
    rules:
      # Fast burn (2% budget in 1 hour)
      - alert: ErrorBudgetFastBurn
        expr: |
          (
            ml_slo_errors:ratio_rate1h > (14.4 * (1 - 0.999))
            and
            ml_slo_errors:ratio_rate5m > (14.4 * (1 - 0.999))
          )
        for: 2m
        labels:
          severity: critical
          alert_type: burn_rate
        annotations:
          summary: "Fast error budget burn detected"

      # Medium burn (5% budget in 6 hours)
      - alert: ErrorBudgetMediumBurn
        expr: |
          (
            ml_slo_errors:ratio_rate6h > (6 * (1 - 0.999))
            and
            ml_slo_errors:ratio_rate30m > (6 * (1 - 0.999))
          )
        for: 15m
        labels:
          severity: warning
          alert_type: burn_rate

      # Slow burn (10% budget in 3 days)
      - alert: ErrorBudgetSlowBurn
        expr: |
          ml_slo_errors:ratio_rate3d > (1 * (1 - 0.999))
        for: 1h
        labels:
          severity: info
          alert_type: burn_rate
```

## Toil Reduction

### Toil Identification

```python
# toil_tracker.py
from dataclasses import dataclass
from datetime import datetime

@dataclass
class ToilEntry:
    task_name: str
    time_spent_minutes: int
    frequency: str  # daily, weekly, monthly
    manual_steps: int
    automatable: bool
    engineer: str
    timestamp: datetime

class ToilTracker:
    def __init__(self, db_client):
        self.db = db_client

    async def log_toil(self, entry: ToilEntry):
        """Log toil activity"""
        await self.db.insert('toil_entries', {
            'task_name': entry.task_name,
            'time_spent': entry.time_spent_minutes,
            'frequency': entry.frequency,
            'manual_steps': entry.manual_steps,
            'automatable': entry.automatable,
            'engineer': entry.engineer,
            'timestamp': entry.timestamp
        })

    async def get_toil_report(self, days: int = 30) -> dict:
        """Generate toil report"""
        entries = await self.db.query("""
            SELECT task_name, SUM(time_spent) as total_time,
                   COUNT(*) as occurrences, MAX(automatable) as automatable
            FROM toil_entries
            WHERE timestamp > NOW() - INTERVAL '$1 days'
            GROUP BY task_name
            ORDER BY total_time DESC
        """, days)

        total_toil = sum(e['total_time'] for e in entries)
        automatable_toil = sum(
            e['total_time'] for e in entries if e['automatable']
        )

        return {
            'total_toil_hours': total_toil / 60,
            'automatable_hours': automatable_toil / 60,
            'automation_opportunity': automatable_toil / total_toil * 100 if total_toil > 0 else 0,
            'top_toil_tasks': entries[:10]
        }
```

### Automation Opportunities

```python
# automation_engine.py
class AutomationEngine:
    def __init__(self):
        self.automations = {}

    def register_automation(self, task_name: str, handler):
        """Register automation handler for a task"""
        self.automations[task_name] = handler

    async def automate_task(self, task_name: str, context: dict) -> bool:
        """Execute automation for a task"""
        if task_name not in self.automations:
            return False

        handler = self.automations[task_name]
        result = await handler(context)

        # Log automation execution
        await self._log_automation(task_name, result)

        return result.success

# Example automations
engine = AutomationEngine()

@engine.register_automation("model_rollback")
async def automate_model_rollback(context):
    """Automate model rollback on accuracy drop"""
    model_name = context['model_name']

    # Get previous version
    prev_version = await get_previous_model_version(model_name)

    # Update deployment
    await update_model_deployment(model_name, prev_version)

    # Verify rollback
    health = await check_model_health(model_name)

    return AutomationResult(success=health.healthy, version=prev_version)

@engine.register_automation("gpu_node_recovery")
async def automate_gpu_recovery(context):
    """Automate GPU node recovery"""
    node_name = context['node_name']

    # Drain node
    await drain_node(node_name)

    # Reset GPU
    await reset_gpu(node_name)

    # Uncordon node
    await uncordon_node(node_name)

    return AutomationResult(success=True)
```

## On-Call Management

### Escalation Policy

```yaml
# escalation-policy.yaml
policy:
  name: ml-platform-oncall
  description: "ML Platform on-call escalation"

  levels:
    - level: 1
      targets:
        - type: schedule
          id: ml-primary-oncall
      delay_minutes: 0

    - level: 2
      targets:
        - type: schedule
          id: ml-secondary-oncall
        - type: user
          id: ml-team-lead
      delay_minutes: 15

    - level: 3
      targets:
        - type: user
          id: engineering-manager
      delay_minutes: 30

  repeat:
    enabled: true
    delay_minutes: 30
    max_repeats: 3
```

### On-Call Handoff

```python
# oncall_handoff.py
class OnCallHandoff:
    def __init__(self, slack_client, pagerduty_client):
        self.slack = slack_client
        self.pd = pagerduty_client

    async def generate_handoff_report(self) -> str:
        """Generate on-call handoff report"""
        # Get incidents from past week
        incidents = await self.pd.get_recent_incidents(days=7)

        # Get open issues
        open_issues = await self._get_open_issues()

        # Get scheduled maintenance
        maintenance = await self._get_scheduled_maintenance()

        report = f"""
## On-Call Handoff Report

### Incidents This Week
{self._format_incidents(incidents)}

### Open Issues Requiring Attention
{self._format_issues(open_issues)}

### Scheduled Maintenance
{self._format_maintenance(maintenance)}

### Key Metrics
- SLO Status: {await self._get_slo_status()}
- Error Budget Remaining: {await self._get_error_budget()}%

### Notes from Outgoing On-Call
{await self._get_oncall_notes()}
        """

        return report

    async def send_handoff(self):
        """Send handoff to next on-call"""
        report = await self.generate_handoff_report()

        # Get incoming on-call
        incoming = await self.pd.get_current_oncall('ml-platform')

        # Send to Slack
        await self.slack.send_dm(
            user=incoming,
            message=report
        )

        # Post to team channel
        await self.slack.post_message(
            channel='#ml-oncall',
            message=report
        )
```

## Capacity Planning

### Forecast Resource Needs

```python
# capacity_planning.py
from prophet import Prophet
import pandas as pd

class CapacityPlanner:
    def __init__(self, prometheus_client):
        self.prom = prometheus_client

    def forecast_capacity(self, metric: str, days_ahead: int = 30) -> dict:
        """Forecast future capacity needs"""
        # Get historical data
        history = self._get_metric_history(metric, days=90)

        # Prepare for Prophet
        df = pd.DataFrame({
            'ds': history['timestamps'],
            'y': history['values']
        })

        # Fit model
        model = Prophet(
            daily_seasonality=True,
            weekly_seasonality=True
        )
        model.fit(df)

        # Forecast
        future = model.make_future_dataframe(periods=days_ahead)
        forecast = model.predict(future)

        # Get peak prediction
        future_only = forecast.tail(days_ahead)
        peak = future_only['yhat'].max()

        return {
            'metric': metric,
            'current': history['values'][-1],
            'predicted_peak': peak,
            'growth_rate': (peak - history['values'][-1]) / history['values'][-1] * 100,
            'forecast': forecast[['ds', 'yhat', 'yhat_lower', 'yhat_upper']].to_dict()
        }

    def recommend_scaling(self, forecasts: list) -> list:
        """Generate scaling recommendations"""
        recommendations = []

        for forecast in forecasts:
            if forecast['growth_rate'] > 20:
                recommendations.append({
                    'metric': forecast['metric'],
                    'action': 'scale_up',
                    'urgency': 'high' if forecast['growth_rate'] > 50 else 'medium',
                    'recommendation': f"Increase capacity by {int(forecast['growth_rate'])}%"
                })

        return recommendations
```

## Best Practices

1. **Define meaningful SLOs**: Based on user experience
2. **Use error budgets**: Balance reliability and velocity
3. **Automate toil**: Free engineers for innovation
4. **Multi-window alerts**: Detect different burn rates
5. **Blameless post-mortems**: Learn from incidents
6. **Capacity planning**: Forecast and scale proactively
7. **On-call health**: Prevent burnout
8. **Document everything**: Runbooks and playbooks

## Resources

- [Google SRE Books](https://sre.google/books/)
- [SLO Concepts](https://sre.google/workbook/implementing-slos/)
- [Sloth SLO Generator](https://sloth.dev/)
- [Pyrra SLO Management](https://pyrra.dev/)

---

*Questions about SRE practices? [Let me know](mailto:jordan@jordananderson.us).*
