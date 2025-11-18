---
layout: default
title:  "Building AI Agents That Don't Go Off the Rails"
date:   2025-11-15 09:00:00
categories: AI Agents LLM Safety
---

LLM agents can edit production code, send emails, and execute workflows autonomously. Without guardrails, they will eventually do something catastrophic. The Stanford 2025 AI Index reported a 56.4% jump in AI-related incidents in 2024—233 cases total.

Here's how to build agents that stay within bounds.

## Why Agents Need Guardrails

Agents differ from simple LLM calls:
- **Multi-step execution** - Errors compound
- **Tool access** - Can affect real systems
- **Autonomy** - Less human oversight
- **Untrusted inputs** - Process emails, web pages, etc.

One bad decision can cascade through your entire workflow.

## Types of Guardrails

### Input Guardrails

Filter and validate before the agent sees input:

```python
from guardrails import Guard
from guardrails.validators import ToxicLanguage, PIIFilter

guard = Guard().use_many(
    ToxicLanguage(on_fail="exception"),
    PIIFilter(on_fail="fix")  # Remove PII automatically
)

def process_input(user_input: str) -> str:
    result = guard.validate(user_input)
    if not result.validation_passed:
        raise ValueError("Input failed validation")
    return result.validated_output
```

**Common input checks:**
- Prompt injection detection
- PII filtering
- Toxic content filtering
- Length limits
- Format validation

### Output Guardrails

Validate before returning to user:

```python
def validate_output(response: str, context: dict) -> str:
    # Check for hallucinations
    if not is_grounded(response, context["sources"]):
        return "I don't have enough information to answer that."

    # Check for harmful content
    if contains_harmful_content(response):
        return "I cannot provide that information."

    # Check for PII leakage
    response = remove_pii(response)

    return response
```

### Interaction Guardrails (Agent-Specific)

Limit agent autonomy:

```python
class AgentGuardrails:
    def __init__(self):
        self.max_steps = 10
        self.allowed_tools = ["search", "calculate", "read_file"]
        self.forbidden_actions = ["delete", "send_email", "execute_code"]
        self.require_approval = ["purchase", "modify_database"]

    def check_tool_call(self, tool_name: str, args: dict) -> bool:
        if tool_name not in self.allowed_tools:
            return False
        if tool_name in self.forbidden_actions:
            return False
        if tool_name in self.require_approval:
            return self.get_human_approval(tool_name, args)
        return True

    def check_step_limit(self, current_step: int) -> bool:
        if current_step >= self.max_steps:
            raise MaxStepsExceeded("Agent exceeded maximum steps")
        return True
```

## Implementing Guardrails

### 1. Tool Restrictions

```python
from langchain.agents import Tool

# Define what tools agent can use
safe_tools = [
    Tool(
        name="search",
        func=search_knowledge_base,
        description="Search internal knowledge base"
    ),
    Tool(
        name="calculate",
        func=calculator,
        description="Perform calculations"
    )
]

# Explicitly exclude dangerous tools
# NO: execute_code, send_email, delete_file, modify_database
```

### 2. Step Limits

```python
class BoundedAgent:
    def __init__(self, max_steps=10, max_tokens=4000):
        self.max_steps = max_steps
        self.max_tokens = max_tokens
        self.current_step = 0
        self.tokens_used = 0

    async def run(self, query: str):
        while self.current_step < self.max_steps:
            self.current_step += 1

            result = await self.think_and_act(query)

            if result.is_final:
                return result.output

            if self.tokens_used > self.max_tokens:
                return "Reached token limit. Here's what I found so far..."

        return "Reached step limit. Task incomplete."
```

### 3. Human-in-the-Loop

```python
class ApprovalRequired:
    HIGH_RISK_ACTIONS = ["purchase", "delete", "send", "modify"]

    async def execute(self, action: str, params: dict):
        if any(risk in action.lower() for risk in self.HIGH_RISK_ACTIONS):
            approval = await self.request_human_approval(action, params)
            if not approval:
                return {"status": "rejected", "reason": "Human rejected action"}

        return await self.perform_action(action, params)

    async def request_human_approval(self, action: str, params: dict):
        # Send to approval queue
        await notify_human(f"Agent wants to: {action}\nParams: {params}")
        # Wait for response (with timeout)
        return await wait_for_approval(timeout=300)
```

### 4. Chain-of-Thought Auditing

Inspect agent reasoning for misalignment:

```python
def audit_reasoning(thought_chain: list[str]) -> bool:
    """Check if agent reasoning shows signs of manipulation or misalignment."""

    red_flags = [
        "ignore previous instructions",
        "bypass security",
        "pretend to be",
        "override restrictions",
        "disregard guidelines"
    ]

    for thought in thought_chain:
        thought_lower = thought.lower()
        if any(flag in thought_lower for flag in red_flags):
            logger.warning(f"Suspicious reasoning detected: {thought}")
            return False

    return True

# Use in agent loop
if not audit_reasoning(agent.thought_chain):
    return "I cannot proceed with this request."
```

### 5. Output Validation

```python
from pydantic import BaseModel, validator

class AgentOutput(BaseModel):
    answer: str
    sources: list[str]
    confidence: float

    @validator('answer')
    def no_harmful_content(cls, v):
        if contains_harmful(v):
            raise ValueError("Response contains harmful content")
        return v

    @validator('confidence')
    def valid_confidence(cls, v):
        if not 0 <= v <= 1:
            raise ValueError("Confidence must be between 0 and 1")
        return v

def validate_agent_output(raw_output: str) -> AgentOutput:
    try:
        return AgentOutput.parse_raw(raw_output)
    except ValidationError as e:
        logger.error(f"Output validation failed: {e}")
        raise
```

## LlamaFirewall Components

Meta's May 2025 framework provides three key guardrails:

### PromptGuard 2
Detects jailbreak attempts:

```python
from llamafirewall import PromptGuard

guard = PromptGuard()

def check_prompt(user_input: str) -> bool:
    result = guard.analyze(user_input)
    if result.is_jailbreak:
        logger.warning(f"Jailbreak detected: {result.attack_type}")
        return False
    return True
```

### Agent Alignment Checks
Audits reasoning for manipulation:

```python
from llamafirewall import AlignmentChecker

checker = AlignmentChecker()

def check_alignment(agent_thoughts: list[str]) -> bool:
    result = checker.audit(agent_thoughts)
    return result.is_aligned
```

### CodeShield
Prevents dangerous code generation:

```python
from llamafirewall import CodeShield

shield = CodeShield()

def validate_generated_code(code: str) -> bool:
    result = shield.analyze(code)
    if result.has_vulnerabilities:
        logger.warning(f"Unsafe code: {result.vulnerabilities}")
        return False
    return True
```

## Real-World Failure Example

**NYC MyCity Chatbot (2024):**
- Gave illegal advice (fire employees for harassment complaints)
- Told businesses they could discriminate against pregnant women

**What was missing:**
- Domain-specific guardrails
- Clear boundaries between information and legal advice
- Human review for high-stakes responses

## Guardrail Checklist

### Input Layer
- [ ] Prompt injection detection
- [ ] PII filtering
- [ ] Toxic content filtering
- [ ] Input length limits
- [ ] Rate limiting

### Agent Layer
- [ ] Tool whitelist (explicit allow, not deny)
- [ ] Step/token limits
- [ ] Human approval for high-risk actions
- [ ] Reasoning auditing
- [ ] Sandbox for code execution

### Output Layer
- [ ] Content moderation
- [ ] Hallucination detection
- [ ] PII leakage check
- [ ] Format validation
- [ ] Source attribution

### Monitoring
- [ ] Log all agent actions
- [ ] Alert on guardrail triggers
- [ ] Track failure rates
- [ ] Regular red-teaming

## Common Mistakes

### 1. Blacklist Instead of Whitelist

```python
# BAD: Blacklist (will miss things)
forbidden_tools = ["delete", "execute"]
if tool not in forbidden_tools:
    execute(tool)

# GOOD: Whitelist (explicit allowlist)
allowed_tools = ["search", "calculate", "read"]
if tool in allowed_tools:
    execute(tool)
```

### 2. No Step Limits

Agents can loop forever. Always set limits.

### 3. Trusting Agent Output

Always validate. Agents hallucinate and make mistakes.

### 4. Reactive Only

Prevention > recovery. Once damage is done, it's done.

## Resources

- [LlamaFirewall (May 2025)](https://arxiv.org/abs/2505.03574)
- [Mastering LLM Guardrails 2025 - Orq.ai](https://orq.ai/blog/llm-guardrails)
- [LLM Guardrails Strategies 2025 - Leanware](https://www.leanware.co/insights/llm-guardrails)
- [Agentic AI Guardrails - Towards Data Science](https://towardsdatascience.com/agentic-ai-102-guardrails-and-agent-evaluation/)
- [Guardrails AI Framework](https://www.guardrailsai.com/)

---

*Questions about AI agent safety? [Let me know](mailto:jordan@jordananderson.us).*
