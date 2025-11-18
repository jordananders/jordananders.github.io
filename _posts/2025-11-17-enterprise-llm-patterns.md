---
layout: default
title:  "Enterprise LLM Integration Patterns"
date:   2025-11-17 20:00:00
categories: AI LLM Enterprise Architecture
---

Enterprise LLM deployments require centralized control, security, compliance, and scalability. The gateway pattern has emerged as the standard architecture. Here are the key patterns for enterprise integration.

## Architecture Overview

```
┌─────────────────────────────────────────────────┐
│                 Applications                     │
└───────────────────────┬─────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────┐
│              LLM Gateway Layer                   │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌────────┐ │
│  │ Auth    │ │ Rate    │ │ Guard-  │ │ Audit  │ │
│  │         │ │ Limit   │ │ rails   │ │ Log    │ │
│  └─────────┘ └─────────┘ └─────────┘ └────────┘ │
└───────────────────────┬─────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────┐
│             Routing & Caching                    │
└───────────────────────┬─────────────────────────┘
                        │
    ┌───────────────────┼───────────────────┐
    ▼                   ▼                   ▼
┌────────┐        ┌────────┐          ┌────────┐
│ OpenAI │        │ Claude │          │ Self-  │
│        │        │        │          │ hosted │
└────────┘        └────────┘          └────────┘
```

## LLM Gateway Implementation

### Core Gateway

```python
from fastapi import FastAPI, HTTPException, Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import httpx
from datetime import datetime
import json

app = FastAPI()
security = HTTPBearer()

class LLMGateway:
    def __init__(self):
        self.providers = {
            "openai": {
                "base_url": "https://api.openai.com/v1",
                "models": ["gpt-4o", "gpt-4o-mini"]
            },
            "anthropic": {
                "base_url": "https://api.anthropic.com/v1",
                "models": ["claude-3-5-sonnet-20241022"]
            },
            "self-hosted": {
                "base_url": "http://vllm-service:8000/v1",
                "models": ["llama-3.1-70b"]
            }
        }
        self.cache = SemanticCache()
        self.rate_limiter = RateLimiter()
        self.audit_logger = AuditLogger()
        self.guardrails = Guardrails()

    async def route_request(
        self,
        request: dict,
        user_context: dict
    ) -> dict:
        """Route request through gateway."""
        # 1. Authenticate and authorize
        if not self.authorize(user_context, request.get("model")):
            raise HTTPException(403, "Unauthorized model access")

        # 2. Rate limiting
        if not await self.rate_limiter.allow(user_context["user_id"]):
            raise HTTPException(429, "Rate limit exceeded")

        # 3. Apply guardrails to input
        sanitized = await self.guardrails.check_input(request["messages"])
        if sanitized["blocked"]:
            raise HTTPException(400, f"Input blocked: {sanitized['reason']}")

        request["messages"] = sanitized["messages"]

        # 4. Check cache
        cached = await self.cache.get(request)
        if cached:
            self.audit_logger.log(request, cached, user_context, cache_hit=True)
            return cached

        # 5. Route to provider
        provider = self.select_provider(request.get("model"))
        response = await self.call_provider(provider, request)

        # 6. Apply guardrails to output
        filtered = await self.guardrails.check_output(response)
        if filtered["blocked"]:
            response["choices"][0]["message"]["content"] = filtered["replacement"]

        # 7. Cache and log
        await self.cache.set(request, response)
        self.audit_logger.log(request, response, user_context)

        return response

    def authorize(self, user_context: dict, model: str) -> bool:
        """Check if user can access model."""
        allowed_models = user_context.get("allowed_models", [])
        return model in allowed_models or "*" in allowed_models

    def select_provider(self, model: str) -> str:
        """Find provider for model."""
        for provider, config in self.providers.items():
            if model in config["models"]:
                return provider
        raise HTTPException(400, f"Unknown model: {model}")

    async def call_provider(self, provider: str, request: dict) -> dict:
        """Call the actual LLM provider."""
        config = self.providers[provider]

        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{config['base_url']}/chat/completions",
                json=request,
                headers=self.get_headers(provider),
                timeout=60.0
            )

        return response.json()

# FastAPI endpoint
gateway = LLMGateway()

@app.post("/v1/chat/completions")
async def chat_completions(
    request: dict,
    credentials: HTTPAuthorizationCredentials = Depends(security)
):
    user_context = await validate_token(credentials.credentials)
    return await gateway.route_request(request, user_context)
```

### Authentication & Authorization

```python
import jwt
from datetime import datetime, timedelta

class AuthService:
    def __init__(self, secret_key: str):
        self.secret_key = secret_key

    def create_api_key(
        self,
        user_id: str,
        team_id: str,
        allowed_models: list[str],
        rate_limit: int,
        expires_days: int = 365
    ) -> str:
        """Create API key for user."""
        payload = {
            "user_id": user_id,
            "team_id": team_id,
            "allowed_models": allowed_models,
            "rate_limit": rate_limit,
            "exp": datetime.utcnow() + timedelta(days=expires_days)
        }

        return jwt.encode(payload, self.secret_key, algorithm="HS256")

    def validate_token(self, token: str) -> dict:
        """Validate and decode token."""
        try:
            return jwt.decode(token, self.secret_key, algorithms=["HS256"])
        except jwt.ExpiredSignatureError:
            raise HTTPException(401, "Token expired")
        except jwt.InvalidTokenError:
            raise HTTPException(401, "Invalid token")

# Role-based access control
class RBAC:
    def __init__(self):
        self.roles = {
            "developer": {
                "models": ["gpt-4o-mini", "llama-3.1-8b"],
                "rate_limit": 100  # per minute
            },
            "analyst": {
                "models": ["gpt-4o-mini", "gpt-4o"],
                "rate_limit": 200
            },
            "admin": {
                "models": ["*"],
                "rate_limit": 1000
            }
        }

    def get_permissions(self, role: str) -> dict:
        return self.roles.get(role, self.roles["developer"])
```

### PII Detection & Redaction

```python
from presidio_analyzer import AnalyzerEngine
from presidio_anonymizer import AnonymizerEngine

class PIIGuardrail:
    def __init__(self):
        self.analyzer = AnalyzerEngine()
        self.anonymizer = AnonymizerEngine()
        self.enabled_entities = [
            "PERSON", "EMAIL_ADDRESS", "PHONE_NUMBER",
            "CREDIT_CARD", "US_SSN", "IP_ADDRESS"
        ]

    def detect_and_redact(self, text: str) -> tuple[str, list]:
        """Detect and redact PII."""
        # Analyze
        results = self.analyzer.analyze(
            text=text,
            entities=self.enabled_entities,
            language="en"
        )

        if not results:
            return text, []

        # Log detected PII types (not values)
        detected = [r.entity_type for r in results]

        # Redact
        anonymized = self.anonymizer.anonymize(
            text=text,
            analyzer_results=results
        )

        return anonymized.text, detected

class Guardrails:
    def __init__(self):
        self.pii = PIIGuardrail()
        self.content_filter = ContentFilter()

    async def check_input(self, messages: list) -> dict:
        """Apply guardrails to input messages."""
        processed = []

        for msg in messages:
            # Redact PII
            content, pii_types = self.pii.detect_and_redact(msg["content"])

            # Check content policy
            if await self.content_filter.is_blocked(content):
                return {
                    "blocked": True,
                    "reason": "Content policy violation",
                    "messages": []
                }

            processed.append({**msg, "content": content})

        return {
            "blocked": False,
            "messages": processed
        }

    async def check_output(self, response: dict) -> dict:
        """Apply guardrails to output."""
        content = response["choices"][0]["message"]["content"]

        # Check for harmful content
        if await self.content_filter.is_blocked(content):
            return {
                "blocked": True,
                "replacement": "I cannot provide that response."
            }

        # Redact any leaked PII
        clean_content, _ = self.pii.detect_and_redact(content)
        response["choices"][0]["message"]["content"] = clean_content

        return {"blocked": False}
```

### Audit Logging

```python
import json
from datetime import datetime

class AuditLogger:
    def __init__(self, storage_backend):
        self.storage = storage_backend

    def log(
        self,
        request: dict,
        response: dict,
        user_context: dict,
        cache_hit: bool = False
    ):
        """Log request for audit trail."""
        log_entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "user_id": user_context["user_id"],
            "team_id": user_context.get("team_id"),
            "model": request.get("model"),
            "cache_hit": cache_hit,

            # Token usage
            "input_tokens": response.get("usage", {}).get("prompt_tokens", 0),
            "output_tokens": response.get("usage", {}).get("completion_tokens", 0),

            # Request metadata (not content for privacy)
            "message_count": len(request.get("messages", [])),
            "has_system_prompt": any(
                m["role"] == "system" for m in request.get("messages", [])
            ),

            # Response metadata
            "response_id": response.get("id"),
            "finish_reason": response.get("choices", [{}])[0].get("finish_reason")
        }

        self.storage.write(log_entry)

    def query(
        self,
        user_id: str = None,
        team_id: str = None,
        start_date: datetime = None,
        end_date: datetime = None
    ) -> list:
        """Query audit logs."""
        return self.storage.query(
            user_id=user_id,
            team_id=team_id,
            start_date=start_date,
            end_date=end_date
        )
```

## Multi-Provider Routing

### Intelligent Router

```python
class IntelligentRouter:
    def __init__(self):
        self.providers = {
            "openai": Provider("openai", priority=1),
            "anthropic": Provider("anthropic", priority=2),
            "self-hosted": Provider("self-hosted", priority=3)
        }
        self.model_mapping = {
            "gpt-4o": ["openai"],
            "gpt-4o-mini": ["openai", "self-hosted"],
            "claude-3-5-sonnet": ["anthropic"],
            "llama-3.1-70b": ["self-hosted"]
        }

    async def route(self, model: str, request: dict) -> dict:
        """Route request to best available provider."""
        providers = self.model_mapping.get(model, [])

        for provider_name in providers:
            provider = self.providers[provider_name]

            if await provider.is_healthy():
                try:
                    return await provider.call(request)
                except Exception as e:
                    # Try next provider
                    continue

        raise HTTPException(503, "All providers unavailable")

class Provider:
    def __init__(self, name: str, priority: int):
        self.name = name
        self.priority = priority
        self.failures = 0
        self.last_failure = None

    async def is_healthy(self) -> bool:
        """Check if provider is healthy."""
        if self.failures > 5:
            # Circuit breaker
            if (datetime.utcnow() - self.last_failure).seconds < 60:
                return False
            # Reset after cooldown
            self.failures = 0

        return True

    async def call(self, request: dict) -> dict:
        """Call provider API."""
        try:
            response = await self._make_request(request)
            self.failures = 0
            return response
        except Exception as e:
            self.failures += 1
            self.last_failure = datetime.utcnow()
            raise
```

### Cost-Based Routing

```python
class CostOptimizedRouter:
    def __init__(self):
        self.costs = {
            "gpt-4o": {"input": 2.50, "output": 10.00},
            "gpt-4o-mini": {"input": 0.15, "output": 0.60},
            "claude-3-5-sonnet": {"input": 3.00, "output": 15.00},
            "llama-3.1-70b-self": {"input": 0.00, "output": 0.00}  # Self-hosted
        }

    def select_model(
        self,
        task_type: str,
        budget_per_request: float,
        quality_requirement: str
    ) -> str:
        """Select most cost-effective model."""
        candidates = []

        for model, cost in self.costs.items():
            estimated_cost = self.estimate_cost(model, task_type)

            if estimated_cost <= budget_per_request:
                if self.meets_quality(model, quality_requirement):
                    candidates.append((model, estimated_cost))

        if not candidates:
            raise ValueError("No model meets requirements")

        # Return cheapest
        return min(candidates, key=lambda x: x[1])[0]

    def estimate_cost(self, model: str, task_type: str) -> float:
        """Estimate cost for task."""
        token_estimates = {
            "classification": (100, 10),
            "summarization": (500, 100),
            "generation": (200, 500)
        }

        input_tokens, output_tokens = token_estimates.get(task_type, (200, 200))
        cost = self.costs[model]

        return (
            (input_tokens / 1_000_000 * cost["input"]) +
            (output_tokens / 1_000_000 * cost["output"])
        )
```

## Compliance Features

### GDPR Compliance

```python
class GDPRCompliance:
    def __init__(self, gateway):
        self.gateway = gateway

    async def handle_data_deletion(self, user_id: str):
        """Handle GDPR deletion request."""
        # 1. Delete cached responses
        await self.gateway.cache.delete_user_data(user_id)

        # 2. Anonymize audit logs
        await self.gateway.audit_logger.anonymize_user(user_id)

        # 3. Notify downstream systems
        await self.notify_deletion(user_id)

    async def handle_data_export(self, user_id: str) -> dict:
        """Export user data for GDPR request."""
        return {
            "audit_logs": await self.gateway.audit_logger.export_user(user_id),
            "usage_stats": await self.get_usage_stats(user_id)
        }
```

### SOC 2 Logging

```python
class SOC2AuditLogger(AuditLogger):
    def log(self, request, response, user_context, **kwargs):
        """Enhanced logging for SOC 2 compliance."""
        entry = super().create_entry(request, response, user_context)

        # Add SOC 2 required fields
        entry.update({
            "access_type": "api",
            "authentication_method": user_context.get("auth_method"),
            "ip_address": user_context.get("ip_address"),
            "user_agent": user_context.get("user_agent"),
            "request_id": str(uuid.uuid4()),
            "data_classification": self.classify_data(request)
        })

        # Write to immutable storage
        self.storage.write_immutable(entry)
```

## Monitoring & Observability

```python
from prometheus_client import Counter, Histogram, Gauge

# Metrics
requests_total = Counter(
    'llm_gateway_requests_total',
    'Total requests',
    ['model', 'status', 'team']
)

latency_histogram = Histogram(
    'llm_gateway_latency_seconds',
    'Request latency',
    ['model']
)

tokens_used = Counter(
    'llm_gateway_tokens_total',
    'Tokens used',
    ['model', 'type']
)

cost_total = Counter(
    'llm_gateway_cost_usd_total',
    'Total cost in USD',
    ['model', 'team']
)

# Instrument gateway
class InstrumentedGateway(LLMGateway):
    async def route_request(self, request, user_context):
        start = time.time()

        try:
            response = await super().route_request(request, user_context)

            # Record metrics
            requests_total.labels(
                model=request["model"],
                status="success",
                team=user_context["team_id"]
            ).inc()

            latency_histogram.labels(
                model=request["model"]
            ).observe(time.time() - start)

            return response

        except Exception as e:
            requests_total.labels(
                model=request.get("model", "unknown"),
                status="error",
                team=user_context.get("team_id", "unknown")
            ).inc()
            raise
```

## Production Checklist

- [ ] API gateway with authentication
- [ ] Rate limiting per user/team
- [ ] PII detection and redaction
- [ ] Content moderation
- [ ] Audit logging
- [ ] Multi-provider failover
- [ ] Cost tracking per team
- [ ] GDPR/compliance features
- [ ] Monitoring and alerting

## Resources

- [Wealthsimple LLM Gateway Case Study](https://www.zenml.io/llmops-database/building-a-secure-and-scalable-llm-gateway-for-enterprise-genai-adoption)
- [Kong AI Gateway](https://konghq.com/products/kong-ai-gateway)
- [TrueFoundry LLM Gateway](https://www.truefoundry.com/blog/llm-gateway)
- [Portkey AI Gateway](https://portkey.ai/)

---

*Questions about enterprise LLM integration? [Let me know](mailto:jordan@jordananderson.us).*
