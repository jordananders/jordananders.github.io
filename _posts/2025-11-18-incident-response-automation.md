---
layout: default
title:  "Incident Response Automation with AI-Powered Analysis"
date:   2025-11-18 12:00:00
categories: DevOps SRE IncidentResponse AI
---

Automated incident response reduces MTTR and on-call burden. AI-powered analysis can identify root causes, suggest remediation, and even auto-resolve common issues.

## Incident Response Architecture

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Alerting   │───>│   Triage    │───>│  Response   │
│  (PagerDuty)│    │   (AI)      │    │  Automation │
└─────────────┘    └─────────────┘    └─────────────┘
       │                  │                  │
       ▼                  ▼                  ▼
┌─────────────────────────────────────────────────┐
│              Incident Management                 │
│    (Slack, Jira, Runbooks, Communication)       │
└─────────────────────────────────────────────────┘
```

## PagerDuty Integration

### Webhook Handler

```python
# incident_handler.py
from fastapi import FastAPI, Request, BackgroundTasks
from pydantic import BaseModel
import httpx

app = FastAPI()

class PagerDutyWebhook(BaseModel):
    messages: list
    routing_key: str

@app.post("/webhook/pagerduty")
async def handle_pagerduty_webhook(
    request: Request,
    background_tasks: BackgroundTasks
):
    payload = await request.json()

    for message in payload.get('messages', []):
        event = message.get('event')
        incident = message.get('incident', {})

        if event == 'incident.triggered':
            background_tasks.add_task(
                handle_new_incident,
                incident
            )
        elif event == 'incident.acknowledged':
            background_tasks.add_task(
                update_incident_status,
                incident,
                'acknowledged'
            )
        elif event == 'incident.resolved':
            background_tasks.add_task(
                handle_resolution,
                incident
            )

    return {"status": "received"}

async def handle_new_incident(incident: dict):
    """Process new incident with AI triage"""
    incident_id = incident['id']
    title = incident['title']
    service = incident['service']['name']

    # Gather context
    context = await gather_incident_context(incident)

    # AI-powered analysis
    analysis = await analyze_incident(title, context)

    # Create response channel
    channel = await create_incident_channel(incident_id)

    # Post initial analysis
    await post_to_channel(channel, format_analysis(analysis))

    # Auto-remediate if confident
    if analysis['confidence'] > 0.9 and analysis['can_auto_remediate']:
        await execute_remediation(incident, analysis['remediation'])
```

### Event Orchestration

```python
# event_orchestration.py
from dataclasses import dataclass
from enum import Enum

class Severity(Enum):
    SEV1 = "sev1"
    SEV2 = "sev2"
    SEV3 = "sev3"

@dataclass
class IncidentRule:
    name: str
    conditions: dict
    actions: list

class EventOrchestrator:
    def __init__(self):
        self.rules = []

    def add_rule(self, rule: IncidentRule):
        self.rules.append(rule)

    async def process_event(self, event: dict):
        """Process event through orchestration rules"""
        for rule in self.rules:
            if self._matches(event, rule.conditions):
                for action in rule.actions:
                    await self._execute_action(action, event)

    def _matches(self, event: dict, conditions: dict) -> bool:
        for key, value in conditions.items():
            if key == 'severity':
                if event.get('severity') != value:
                    return False
            elif key == 'service_pattern':
                if not re.match(value, event.get('service', '')):
                    return False
            elif key == 'time_range':
                current_hour = datetime.now().hour
                if not (value['start'] <= current_hour <= value['end']):
                    return False
        return True

    async def _execute_action(self, action: dict, event: dict):
        if action['type'] == 'page':
            await self._page_team(action['team'], event)
        elif action['type'] == 'slack':
            await self._notify_slack(action['channel'], event)
        elif action['type'] == 'runbook':
            await self._execute_runbook(action['runbook_id'], event)
        elif action['type'] == 'auto_resolve':
            await self._auto_resolve(event, action['conditions'])

# Configure rules
orchestrator = EventOrchestrator()

# SEV1 ML model degradation
orchestrator.add_rule(IncidentRule(
    name="ml-model-degradation-sev1",
    conditions={
        "severity": Severity.SEV1,
        "service_pattern": r"ml-.*"
    },
    actions=[
        {"type": "page", "team": "ml-platform"},
        {"type": "slack", "channel": "#incident-war-room"},
        {"type": "runbook", "runbook_id": "ml-model-failover"}
    ]
))
```

## AI-Powered Analysis

### Incident Analyzer

```python
# ai_analyzer.py
from openai import OpenAI
import json

class IncidentAnalyzer:
    def __init__(self, openai_key: str):
        self.client = OpenAI(api_key=openai_key)

    async def analyze(self, incident: dict, context: dict) -> dict:
        """Analyze incident with LLM"""
        prompt = self._build_analysis_prompt(incident, context)

        response = self.client.chat.completions.create(
            model="gpt-4",
            messages=[
                {
                    "role": "system",
                    "content": """You are an SRE expert analyzing production incidents.
                    Provide:
                    1. Root cause analysis
                    2. Impact assessment
                    3. Remediation steps
                    4. Similar past incidents
                    Respond in JSON format."""
                },
                {"role": "user", "content": prompt}
            ],
            response_format={"type": "json_object"}
        )

        return json.loads(response.choices[0].message.content)

    def _build_analysis_prompt(self, incident: dict, context: dict) -> str:
        return f"""
        Incident: {incident['title']}
        Service: {incident['service']}
        Severity: {incident['severity']}

        Recent Logs:
        {context['logs'][-50:]}

        Metrics (last 30 min):
        - Error Rate: {context['metrics']['error_rate']}
        - Latency P99: {context['metrics']['latency_p99']}
        - CPU Usage: {context['metrics']['cpu']}

        Recent Changes:
        {json.dumps(context['recent_deployments'], indent=2)}

        Similar Past Incidents:
        {json.dumps(context['similar_incidents'], indent=2)}

        Analyze this incident and provide remediation steps.
        """

    async def suggest_runbook(self, analysis: dict) -> str:
        """Suggest appropriate runbook based on analysis"""
        response = self.client.chat.completions.create(
            model="gpt-4",
            messages=[
                {
                    "role": "system",
                    "content": "Based on the incident analysis, suggest the most appropriate runbook from the available runbooks."
                },
                {
                    "role": "user",
                    "content": f"Analysis: {json.dumps(analysis)}\n\nAvailable Runbooks: {self._get_runbook_list()}"
                }
            ]
        )

        return response.choices[0].message.content
```

### Context Gathering

```python
# context_gatherer.py
from prometheus_api_client import PrometheusConnect
from elasticsearch import Elasticsearch
import asyncio

class ContextGatherer:
    def __init__(self, config):
        self.prometheus = PrometheusConnect(url=config.prometheus_url)
        self.elasticsearch = Elasticsearch([config.elasticsearch_url])
        self.deployment_api = config.deployment_api

    async def gather(self, incident: dict) -> dict:
        """Gather all context for incident analysis"""
        service = incident['service']['name']
        start_time = incident['created_at']

        # Gather in parallel
        logs, metrics, deployments, similar = await asyncio.gather(
            self._get_logs(service, start_time),
            self._get_metrics(service, start_time),
            self._get_recent_deployments(service),
            self._find_similar_incidents(incident)
        )

        return {
            'logs': logs,
            'metrics': metrics,
            'recent_deployments': deployments,
            'similar_incidents': similar
        }

    async def _get_logs(self, service: str, start_time: str) -> list:
        """Get recent error logs"""
        query = {
            "query": {
                "bool": {
                    "must": [
                        {"match": {"service": service}},
                        {"match": {"level": "error"}},
                        {"range": {"@timestamp": {"gte": start_time}}}
                    ]
                }
            },
            "sort": [{"@timestamp": "desc"}],
            "size": 100
        }

        result = self.elasticsearch.search(index="logs-*", body=query)
        return [hit['_source']['message'] for hit in result['hits']['hits']]

    async def _get_metrics(self, service: str, start_time: str) -> dict:
        """Get key metrics"""
        queries = {
            'error_rate': f'sum(rate(http_requests_total{{service="{service}",status=~"5.."}}[5m])) / sum(rate(http_requests_total{{service="{service}"}}[5m]))',
            'latency_p99': f'histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{{service="{service}"}}[5m])) by (le))',
            'cpu': f'sum(rate(container_cpu_usage_seconds_total{{pod=~"{service}.*"}}[5m]))'
        }

        metrics = {}
        for name, query in queries.items():
            result = self.prometheus.custom_query(query)
            metrics[name] = result[0]['value'][1] if result else 'N/A'

        return metrics

    async def _find_similar_incidents(self, incident: dict) -> list:
        """Find similar past incidents using embeddings"""
        # Use vector similarity to find similar incidents
        # Implementation depends on your incident database
        pass
```

## Auto-Remediation

### Remediation Engine

```python
# remediation_engine.py
from kubernetes import client, config
import asyncio

class RemediationEngine:
    def __init__(self):
        config.load_incluster_config()
        self.k8s = client.AppsV1Api()
        self.core = client.CoreV1Api()

    async def execute(self, incident: dict, remediation: dict):
        """Execute remediation action"""
        action = remediation['action']

        if action == 'restart_pods':
            await self._restart_pods(remediation['deployment'])
        elif action == 'scale_up':
            await self._scale_deployment(
                remediation['deployment'],
                remediation['replicas']
            )
        elif action == 'rollback':
            await self._rollback_deployment(remediation['deployment'])
        elif action == 'failover_model':
            await self._failover_model(remediation['model'])
        elif action == 'clear_cache':
            await self._clear_cache(remediation['cache'])

        # Log action
        await self._log_remediation(incident, remediation)

    async def _restart_pods(self, deployment: str):
        """Rolling restart of deployment"""
        namespace = "ml-platform"

        # Get deployment
        dep = self.k8s.read_namespaced_deployment(deployment, namespace)

        # Update annotation to trigger restart
        if not dep.spec.template.metadata.annotations:
            dep.spec.template.metadata.annotations = {}

        dep.spec.template.metadata.annotations['kubectl.kubernetes.io/restartedAt'] = \
            datetime.utcnow().isoformat()

        self.k8s.patch_namespaced_deployment(deployment, namespace, dep)

    async def _rollback_deployment(self, deployment: str):
        """Rollback to previous version"""
        namespace = "ml-platform"

        # Get deployment history
        history = self.k8s.read_namespaced_deployment(
            deployment, namespace
        ).status.conditions

        # Rollback using kubectl equivalent
        # This uses the revision history
        body = {
            "apiVersion": "apps/v1",
            "kind": "Deployment",
            "spec": {
                "rollbackTo": {
                    "revision": 0  # Previous revision
                }
            }
        }

        self.k8s.patch_namespaced_deployment(
            deployment, namespace, body
        )

    async def _failover_model(self, model_config: dict):
        """Failover to backup model"""
        # Update ConfigMap with backup model
        config_map = self.core.read_namespaced_config_map(
            "model-config", "ml-platform"
        )

        config_map.data['active_model'] = model_config['backup']
        config_map.data['fallback_reason'] = 'auto-remediation'

        self.core.patch_namespaced_config_map(
            "model-config", "ml-platform", config_map
        )

        # Trigger rolling update
        await self._restart_pods(model_config['deployment'])
```

### Runbook Automation

```yaml
# runbooks/ml-model-degradation.yaml
name: ML Model Degradation
description: Handle ML model performance degradation
triggers:
  - alert: MLModelAccuracyLow
  - alert: MLModelLatencyHigh

steps:
  - name: Check model metrics
    action: prometheus_query
    query: 'ml_model_accuracy{model="${model_name}"}'
    store_as: current_accuracy

  - name: Check if below threshold
    condition: '${current_accuracy} < 0.90'
    on_true:
      - name: Notify team
        action: slack_message
        channel: '#ml-platform'
        message: 'Model ${model_name} accuracy dropped to ${current_accuracy}'

      - name: Check recent deployments
        action: query_deployments
        service: '${service_name}'
        hours: 24

      - name: Decide action
        condition: '${has_recent_deployment}'
        on_true:
          - name: Rollback deployment
            action: rollback
            deployment: '${service_name}'
        on_false:
          - name: Failover to backup model
            action: update_config
            config_map: model-config
            key: active_model
            value: '${backup_model}'

  - name: Verify recovery
    action: wait_for_metric
    query: 'ml_model_accuracy{model="${model_name}"}'
    condition: '> 0.95'
    timeout: 10m
```

## Incident Communication

### Slack Bot

```python
# slack_bot.py
from slack_sdk import WebClient
from slack_sdk.socket_mode import SocketModeClient
from slack_sdk.socket_mode.request import SocketModeRequest

class IncidentBot:
    def __init__(self, bot_token: str, app_token: str):
        self.client = WebClient(token=bot_token)
        self.socket = SocketModeClient(
            app_token=app_token,
            web_client=self.client
        )

    async def create_incident_channel(self, incident_id: str) -> str:
        """Create dedicated incident channel"""
        channel_name = f"inc-{incident_id}"

        response = self.client.conversations_create(
            name=channel_name,
            is_private=False
        )

        channel_id = response['channel']['id']

        # Set topic
        self.client.conversations_setTopic(
            channel=channel_id,
            topic=f"Incident {incident_id} - Active"
        )

        # Pin useful links
        await self._pin_incident_resources(channel_id, incident_id)

        return channel_id

    async def post_update(self, channel: str, update: dict):
        """Post incident update with formatting"""
        blocks = [
            {
                "type": "header",
                "text": {
                    "type": "plain_text",
                    "text": f"🚨 {update['title']}"
                }
            },
            {
                "type": "section",
                "fields": [
                    {"type": "mrkdwn", "text": f"*Status:*\n{update['status']}"},
                    {"type": "mrkdwn", "text": f"*Severity:*\n{update['severity']}"}
                ]
            },
            {
                "type": "section",
                "text": {
                    "type": "mrkdwn",
                    "text": f"*Summary:*\n{update['summary']}"
                }
            },
            {
                "type": "actions",
                "elements": [
                    {
                        "type": "button",
                        "text": {"type": "plain_text", "text": "View Dashboard"},
                        "url": update['dashboard_url']
                    },
                    {
                        "type": "button",
                        "text": {"type": "plain_text", "text": "View Runbook"},
                        "url": update['runbook_url']
                    }
                ]
            }
        ]

        self.client.chat_postMessage(
            channel=channel,
            blocks=blocks
        )

    async def generate_timeline(self, channel: str) -> str:
        """Generate incident timeline from channel history"""
        history = self.client.conversations_history(
            channel=channel,
            limit=200
        )

        timeline = []
        for message in history['messages']:
            if message.get('subtype') != 'bot_message':
                timeline.append({
                    'time': datetime.fromtimestamp(float(message['ts'])),
                    'user': message.get('user', 'bot'),
                    'text': message['text'][:100]
                })

        return timeline
```

## Post-Incident Analysis

### Automated Post-Mortem

```python
# postmortem_generator.py
class PostMortemGenerator:
    def __init__(self, analyzer: IncidentAnalyzer):
        self.analyzer = analyzer

    async def generate(self, incident: dict, timeline: list) -> str:
        """Generate post-mortem document"""
        prompt = f"""
        Generate a blameless post-mortem for this incident:

        Incident: {incident['title']}
        Duration: {incident['duration_minutes']} minutes
        Impact: {incident['impact']}

        Timeline:
        {self._format_timeline(timeline)}

        Include:
        1. Executive Summary
        2. Impact Analysis
        3. Root Cause
        4. Timeline
        5. Action Items
        6. Lessons Learned
        """

        response = await self.analyzer.client.chat.completions.create(
            model="gpt-4",
            messages=[
                {
                    "role": "system",
                    "content": "Generate a comprehensive, blameless post-mortem document in markdown format."
                },
                {"role": "user", "content": prompt}
            ]
        )

        return response.choices[0].message.content

    async def extract_action_items(self, postmortem: str) -> list:
        """Extract and create Jira tickets for action items"""
        # Parse action items from post-mortem
        # Create Jira tickets
        # Return list of ticket IDs
        pass
```

## Best Practices

1. **Automate first response**: Reduce human intervention
2. **Gather context proactively**: Logs, metrics, changes
3. **AI-assisted analysis**: Suggest root causes
4. **Safe auto-remediation**: Start with low-risk actions
5. **Clear communication**: Automated updates
6. **Learn from incidents**: Generate and track post-mortems
7. **Runbook everything**: Document procedures
8. **Test regularly**: Practice incident response

## Resources

- [PagerDuty Incident Response](https://response.pagerduty.com/)
- [Google SRE Book - Emergency Response](https://sre.google/sre-book/emergency-response/)
- [Incident Management Best Practices](https://www.atlassian.com/incident-management)
- [Blameless Post-Mortems](https://www.blameless.com/)

---

*Questions about incident response? [Let me know](mailto:jordan@jordananderson.us).*
