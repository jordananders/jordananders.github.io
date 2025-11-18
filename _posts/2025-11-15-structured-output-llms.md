---
layout: default
title:  "Structured Output from LLMs: JSON Mode and Function Calling"
date:   2025-11-15 11:00:00
categories: AI LLM JSON Production
---

Getting consistent, parseable output from LLMs is critical for production applications. Free-form text is great for chat, but when you need to populate databases, call APIs, or drive UI components, you need structure. Here's how to get reliable structured output from modern LLMs.

## Three Approaches

### 1. JSON Mode

Basic JSON output guarantee—valid JSON, but no schema enforcement:

```python
# OpenAI JSON Mode
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "Return valid JSON"},
        {"role": "user", "content": "Extract name and email from: John Doe, john@example.com"}
    ],
    response_format={"type": "json_object"}
)

# Returns valid JSON, but structure not guaranteed
# Could be {"name": "John", "email": "john@example.com"}
# Or {"person": {"name": "John Doe", "contact": "john@example.com"}}
```

**Pros:** Simple, widely supported
**Cons:** No schema validation, structure can vary

### 2. Structured Outputs (JSON Schema)

Schema-enforced output—guarantees exact structure:

```python
from pydantic import BaseModel

class Person(BaseModel):
    name: str
    email: str
    age: int | None = None

# OpenAI Structured Outputs
response = client.beta.chat.completions.parse(
    model="gpt-4o-2024-08-06",
    messages=[
        {"role": "user", "content": "Extract: John Doe, 30, john@example.com"}
    ],
    response_format=Person
)

person = response.choices[0].message.parsed
print(person.name)   # "John Doe"
print(person.email)  # "john@example.com"
print(person.age)    # 30
```

**Pros:** Guaranteed schema compliance, type safety
**Cons:** Not all models support it

### 3. Function Calling

Define tools with schemas—model calls them with structured arguments:

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "extract_person",
            "description": "Extract person information",
            "parameters": {
                "type": "object",
                "properties": {
                    "name": {"type": "string"},
                    "email": {"type": "string", "format": "email"},
                    "age": {"type": "integer"}
                },
                "required": ["name", "email"]
            }
        }
    }
]

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Extract: John Doe, john@example.com"}],
    tools=tools,
    tool_choice={"type": "function", "function": {"name": "extract_person"}}
)

# Parse the function call arguments
args = json.loads(response.choices[0].message.tool_calls[0].function.arguments)
```

**Pros:** Flexible, works with external tools
**Cons:** More verbose setup

## Provider-Specific Implementations

### OpenAI

```python
from openai import OpenAI
from pydantic import BaseModel

client = OpenAI()

# Method 1: Structured Outputs (Best for extraction)
class Article(BaseModel):
    title: str
    summary: str
    tags: list[str]
    sentiment: str

response = client.beta.chat.completions.parse(
    model="gpt-4o-2024-08-06",
    messages=[
        {"role": "user", "content": f"Analyze this article: {article_text}"}
    ],
    response_format=Article
)

article = response.choices[0].message.parsed

# Method 2: JSON Mode (Simpler, less strict)
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "Return JSON with title, summary, tags, sentiment"},
        {"role": "user", "content": f"Analyze: {article_text}"}
    ],
    response_format={"type": "json_object"}
)

data = json.loads(response.choices[0].message.content)
```

### Anthropic (Claude)

Claude uses tools for structured output:

```python
import anthropic

client = anthropic.Anthropic()

# Define output schema as a tool
tools = [
    {
        "name": "extract_article",
        "description": "Extract article information",
        "input_schema": {
            "type": "object",
            "properties": {
                "title": {"type": "string"},
                "summary": {"type": "string"},
                "tags": {"type": "array", "items": {"type": "string"}},
                "sentiment": {
                    "type": "string",
                    "enum": ["positive", "negative", "neutral"]
                }
            },
            "required": ["title", "summary", "tags", "sentiment"]
        }
    }
]

response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=tools,
    tool_choice={"type": "tool", "name": "extract_article"},
    messages=[
        {"role": "user", "content": f"Analyze this article: {article_text}"}
    ]
)

# Extract structured data from tool use
for block in response.content:
    if block.type == "tool_use":
        article_data = block.input
        print(article_data["title"])
        print(article_data["sentiment"])
```

### Using LangChain for Unified Interface

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic
from pydantic import BaseModel, Field

class ExtractedData(BaseModel):
    """Extracted information from text."""
    name: str = Field(description="Person's full name")
    email: str = Field(description="Email address")
    company: str = Field(description="Company name")
    role: str = Field(description="Job title or role")

# Works with OpenAI
openai_llm = ChatOpenAI(model="gpt-4o")
structured_openai = openai_llm.with_structured_output(ExtractedData)

# Works with Anthropic
anthropic_llm = ChatAnthropic(model="claude-sonnet-4-20250514")
structured_anthropic = anthropic_llm.with_structured_output(ExtractedData)

# Same interface for both
result = structured_openai.invoke("Extract: Jane Smith, CTO at TechCorp, jane@techcorp.com")
print(result.name)     # "Jane Smith"
print(result.company)  # "TechCorp"
```

## Advanced Patterns

### Nested Structures

```python
from pydantic import BaseModel
from typing import Optional

class Address(BaseModel):
    street: str
    city: str
    state: str
    zip_code: str
    country: str = "USA"

class Contact(BaseModel):
    email: str
    phone: Optional[str] = None

class Person(BaseModel):
    name: str
    contact: Contact
    addresses: list[Address]
    metadata: dict[str, str] = {}

# Use with structured outputs
response = client.beta.chat.completions.parse(
    model="gpt-4o-2024-08-06",
    messages=[{"role": "user", "content": complex_text}],
    response_format=Person
)
```

### Enums and Constraints

```python
from pydantic import BaseModel, Field
from enum import Enum

class Priority(str, Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"

class Status(str, Enum):
    OPEN = "open"
    IN_PROGRESS = "in_progress"
    RESOLVED = "resolved"
    CLOSED = "closed"

class Ticket(BaseModel):
    title: str = Field(max_length=100)
    description: str
    priority: Priority
    status: Status
    assignee: Optional[str] = None
    tags: list[str] = Field(max_items=5)
```

### Multiple Extraction

```python
class Entity(BaseModel):
    name: str
    type: str
    description: str

class ExtractionResult(BaseModel):
    entities: list[Entity]
    relationships: list[dict[str, str]]
    summary: str

# Extract multiple entities from text
response = client.beta.chat.completions.parse(
    model="gpt-4o-2024-08-06",
    messages=[
        {"role": "system", "content": "Extract all entities and their relationships"},
        {"role": "user", "content": document_text}
    ],
    response_format=ExtractionResult
)
```

## Validation and Error Handling

### Pydantic Validation

```python
from pydantic import BaseModel, Field, validator, field_validator
import re

class UserInput(BaseModel):
    email: str
    phone: str
    age: int = Field(ge=0, le=150)

    @field_validator('email')
    @classmethod
    def validate_email(cls, v):
        if not re.match(r'^[\w\.-]+@[\w\.-]+\.\w+$', v):
            raise ValueError('Invalid email format')
        return v

    @field_validator('phone')
    @classmethod
    def validate_phone(cls, v):
        # Remove non-digits
        digits = re.sub(r'\D', '', v)
        if len(digits) < 10:
            raise ValueError('Phone must have at least 10 digits')
        return digits
```

### Handling Partial Failures

```python
from pydantic import BaseModel, ValidationError

def extract_with_retry(text: str, schema: type[BaseModel], max_retries: int = 3):
    """Extract structured data with validation retry."""

    for attempt in range(max_retries):
        try:
            response = client.beta.chat.completions.parse(
                model="gpt-4o-2024-08-06",
                messages=[{"role": "user", "content": text}],
                response_format=schema
            )

            # Validate again (paranoia check)
            validated = schema.model_validate(response.choices[0].message.parsed)
            return validated

        except ValidationError as e:
            if attempt == max_retries - 1:
                raise

            # Add error context for retry
            error_context = f"Previous attempt failed validation: {e}"
            text = f"{text}\n\nNote: {error_context}"

    return None
```

### Fallback Parsing

```python
import json
import re

def robust_json_parse(text: str) -> dict:
    """Parse JSON from LLM output with multiple fallbacks."""

    # Try direct parse
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        pass

    # Try extracting JSON from markdown code block
    json_match = re.search(r'```(?:json)?\s*([\s\S]*?)\s*```', text)
    if json_match:
        try:
            return json.loads(json_match.group(1))
        except json.JSONDecodeError:
            pass

    # Try finding JSON object in text
    brace_match = re.search(r'\{[\s\S]*\}', text)
    if brace_match:
        try:
            return json.loads(brace_match.group(0))
        except json.JSONDecodeError:
            pass

    raise ValueError(f"Could not parse JSON from: {text[:100]}...")
```

## When to Use What

### Use Structured Outputs When:
- You need guaranteed schema compliance
- Data goes directly into database/API
- Type safety is critical
- Schema is complex with nested objects

### Use JSON Mode When:
- Schema is simple or flexible
- You'll validate separately
- Model doesn't support structured outputs
- Quick prototyping

### Use Function Calling When:
- Integrating with external tools/APIs
- Need conditional tool selection
- Building agents that take actions
- Multiple possible outputs

## Common Mistakes

### 1. Not Specifying Format in Prompt

```python
# BAD: Model might not return JSON
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "List 3 colors"}],
    response_format={"type": "json_object"}
)
# Could fail - model needs to know to return JSON

# GOOD: Explicit instruction
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "Return a JSON object with a 'colors' array"},
        {"role": "user", "content": "List 3 colors"}
    ],
    response_format={"type": "json_object"}
)
```

### 2. Missing Field Descriptions

```python
# BAD: Model has to guess what you want
class User(BaseModel):
    name: str
    status: str
    score: int

# GOOD: Clear descriptions guide extraction
class User(BaseModel):
    name: str = Field(description="User's full legal name")
    status: str = Field(description="Account status: active, suspended, or deleted")
    score: int = Field(description="Reputation score from 0-1000")
```

### 3. Not Handling Optional Fields

```python
# BAD: Fails if field missing
class Strict(BaseModel):
    required_field: str
    also_required: int  # Will fail if not in text

# GOOD: Graceful handling of missing data
class Flexible(BaseModel):
    required_field: str
    optional_field: int | None = None
    with_default: str = "unknown"
```

## Performance Considerations

### Token Efficiency

Complex schemas cost more tokens:

```python
# Verbose schema (more tokens)
class Verbose(BaseModel):
    user_full_name_including_middle: str
    user_primary_email_address: str
    user_phone_number_with_country_code: str

# Concise schema (fewer tokens)
class Concise(BaseModel):
    name: str
    email: str
    phone: str
```

### Batch Processing

```python
async def batch_extract(texts: list[str], schema: type[BaseModel]):
    """Extract from multiple texts concurrently."""

    async def extract_one(text: str):
        response = await client.beta.chat.completions.parse(
            model="gpt-4o-2024-08-06",
            messages=[{"role": "user", "content": text}],
            response_format=schema
        )
        return response.choices[0].message.parsed

    results = await asyncio.gather(*[extract_one(t) for t in texts])
    return results
```

## Resources

- [OpenAI Structured Outputs Guide](https://platform.openai.com/docs/guides/structured-outputs)
- [Anthropic Tool Use Documentation](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)
- [LangChain Structured Output](https://python.langchain.com/docs/how_to/structured_output/)
- [Pydantic Documentation](https://docs.pydantic.dev/)
- [LiteLLM JSON Mode](https://docs.litellm.ai/docs/completion/json_mode)

---

*Questions about structured LLM output? [Let me know](mailto:jordan@jordananderson.us).*
