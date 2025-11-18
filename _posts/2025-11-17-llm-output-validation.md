---
layout: default
title:  "LLM Output Validation and Parsing"
date:   2025-11-17 21:00:00
categories: AI LLM Validation Pydantic
---

Getting reliable structured output from LLMs is challenging. The Instructor library with Pydantic provides automatic validation, retries, and type safety. Here's how to use it effectively.

## The Problem

LLMs return strings, but applications need structured data. Common issues:
- Invalid JSON syntax
- Missing required fields
- Wrong data types
- Values outside valid ranges

## Instructor + Pydantic Solution

```python
import instructor
from openai import OpenAI
from pydantic import BaseModel, Field
from typing import Literal

# Patch OpenAI client
client = instructor.from_openai(OpenAI())

# Define schema with Pydantic
class User(BaseModel):
    name: str
    age: int = Field(ge=0, le=150)
    email: str

# Get validated output
user = client.chat.completions.create(
    model="gpt-4o-mini",
    response_model=User,
    messages=[
        {"role": "user", "content": "Extract: John Doe, 30, john@example.com"}
    ]
)

print(user)
# User(name='John Doe', age=30, email='john@example.com')
print(type(user))
# <class '__main__.User'>
```

## Complex Schemas

### Nested Objects

```python
from pydantic import BaseModel
from typing import Optional

class Address(BaseModel):
    street: str
    city: str
    country: str
    postal_code: Optional[str] = None

class Company(BaseModel):
    name: str
    industry: str

class Person(BaseModel):
    name: str
    age: int
    address: Address
    company: Optional[Company] = None
    skills: list[str] = []

# Extract complex data
person = client.chat.completions.create(
    model="gpt-4o-mini",
    response_model=Person,
    messages=[{
        "role": "user",
        "content": """
        John Smith is 35 years old, living at 123 Main St, New York, USA.
        He works at TechCorp in the software industry.
        Skills: Python, Machine Learning, DevOps
        """
    }]
)
```

### Enums and Literals

```python
from enum import Enum
from typing import Literal

class Priority(str, Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"

class Ticket(BaseModel):
    title: str
    description: str
    priority: Priority
    category: Literal["bug", "feature", "question"]
    tags: list[str] = []

ticket = client.chat.completions.create(
    model="gpt-4o-mini",
    response_model=Ticket,
    messages=[{
        "role": "user",
        "content": "Login button broken on mobile. Urgent fix needed!"
    }]
)

print(ticket.priority)
# Priority.HIGH
print(ticket.category)
# "bug"
```

## Custom Validators

### Field Validation

```python
from pydantic import BaseModel, field_validator, ValidationInfo
import re

class Contact(BaseModel):
    name: str
    email: str
    phone: str

    @field_validator("email")
    @classmethod
    def validate_email(cls, v: str) -> str:
        if not re.match(r"[^@]+@[^@]+\.[^@]+", v):
            raise ValueError("Invalid email format")
        return v.lower()

    @field_validator("phone")
    @classmethod
    def validate_phone(cls, v: str) -> str:
        # Remove non-digits
        digits = re.sub(r"\D", "", v)
        if len(digits) < 10:
            raise ValueError("Phone must have at least 10 digits")
        return digits

# Instructor will retry on validation failure
contact = client.chat.completions.create(
    model="gpt-4o-mini",
    response_model=Contact,
    max_retries=3,  # Retry on validation failure
    messages=[{
        "role": "user",
        "content": "John at john@example.com, phone: (555) 123-4567"
    }]
)
```

### Model-Level Validation

```python
from pydantic import model_validator

class DateRange(BaseModel):
    start_date: str
    end_date: str

    @model_validator(mode="after")
    def validate_dates(self):
        from datetime import datetime

        start = datetime.fromisoformat(self.start_date)
        end = datetime.fromisoformat(self.end_date)

        if end < start:
            raise ValueError("end_date must be after start_date")

        return self
```

### LLM-Based Validation

```python
from instructor import llm_validator

class ProductReview(BaseModel):
    product_name: str
    rating: int = Field(ge=1, le=5)
    review_text: str
    sentiment: str

    @field_validator("review_text")
    @classmethod
    def validate_review(cls, v: str) -> str:
        # Use LLM to validate content quality
        return llm_validator(
            statement="Review must be constructive and non-offensive",
            allow_override=False
        )(v)
```

## Automatic Retries

```python
# Instructor automatically retries on validation failure
result = client.chat.completions.create(
    model="gpt-4o-mini",
    response_model=User,
    max_retries=3,  # Retry up to 3 times
    messages=[{"role": "user", "content": "..."}]
)

# Custom retry logic
from tenacity import Retrying, stop_after_attempt, wait_fixed

result = client.chat.completions.create(
    model="gpt-4o-mini",
    response_model=User,
    max_retries=Retrying(
        stop=stop_after_attempt(3),
        wait=wait_fixed(1)
    ),
    messages=[{"role": "user", "content": "..."}]
)
```

## Streaming Validation

```python
from instructor import Partial

# Stream with partial validation
class Report(BaseModel):
    title: str
    summary: str
    key_findings: list[str]
    recommendations: list[str]

# Get partial results as they stream
for partial_report in client.chat.completions.create_partial(
    model="gpt-4o",
    response_model=Report,
    messages=[{
        "role": "user",
        "content": "Write a market analysis report for Q4 2024"
    }]
):
    # Fields are available as they're validated
    if partial_report.title:
        print(f"Title: {partial_report.title}")
    if partial_report.key_findings:
        print(f"Findings so far: {len(partial_report.key_findings)}")
```

## Multiple Extractions

```python
from typing import Iterable

class Entity(BaseModel):
    name: str
    type: Literal["person", "company", "location"]
    description: str

# Extract multiple items
entities = client.chat.completions.create(
    model="gpt-4o-mini",
    response_model=Iterable[Entity],
    messages=[{
        "role": "user",
        "content": """
        Apple CEO Tim Cook announced new products at the
        Cupertino headquarters yesterday.
        """
    }]
)

for entity in entities:
    print(f"{entity.name} ({entity.type}): {entity.description}")
```

## Self-Hosted Models

### Ollama

```python
from openai import OpenAI
import instructor

# Connect to Ollama
client = instructor.from_openai(
    OpenAI(
        base_url="http://localhost:11434/v1",
        api_key="ollama"
    ),
    mode=instructor.Mode.JSON
)

result = client.chat.completions.create(
    model="llama3.1:8b",
    response_model=User,
    messages=[{"role": "user", "content": "..."}]
)
```

### llama-cpp-python

```python
from llama_cpp import Llama
import instructor

llm = Llama(
    model_path="./models/llama-3.1-8b.gguf",
    n_ctx=4096,
    n_gpu_layers=-1
)

client = instructor.from_llamacpp(llm)

result = client.create_completion(
    response_model=User,
    messages=[{"role": "user", "content": "..."}]
)
```

## Without Instructor

### OpenAI Structured Output

```python
from openai import OpenAI

client = OpenAI()

class CalendarEvent(BaseModel):
    name: str
    date: str
    participants: list[str]

response = client.beta.chat.completions.parse(
    model="gpt-4o-2024-08-06",
    messages=[
        {"role": "user", "content": "Schedule team meeting for Friday at 2pm"}
    ],
    response_format=CalendarEvent
)

event = response.choices[0].message.parsed
print(event.name)  # "Team Meeting"
```

### Manual JSON Parsing

```python
import json
from pydantic import BaseModel, ValidationError

def parse_llm_response(response: str, model: type[BaseModel]) -> BaseModel:
    """Parse and validate LLM response."""
    # Try to extract JSON
    try:
        # Handle markdown code blocks
        if "```json" in response:
            json_str = response.split("```json")[1].split("```")[0]
        elif "```" in response:
            json_str = response.split("```")[1].split("```")[0]
        else:
            json_str = response

        data = json.loads(json_str.strip())
        return model.model_validate(data)

    except json.JSONDecodeError as e:
        raise ValueError(f"Invalid JSON: {e}")
    except ValidationError as e:
        raise ValueError(f"Validation failed: {e}")

# Usage
response_text = """```json
{"name": "John", "age": 30, "email": "john@example.com"}
```"""

user = parse_llm_response(response_text, User)
```

## Error Handling

```python
from pydantic import ValidationError
from instructor.exceptions import InstructorRetryException

async def safe_extract(prompt: str, model: type[BaseModel]) -> dict:
    """Extract with comprehensive error handling."""
    try:
        result = client.chat.completions.create(
            model="gpt-4o-mini",
            response_model=model,
            max_retries=3,
            messages=[{"role": "user", "content": prompt}]
        )
        return {"success": True, "data": result}

    except ValidationError as e:
        return {
            "success": False,
            "error": "validation_error",
            "details": e.errors()
        }

    except InstructorRetryException as e:
        return {
            "success": False,
            "error": "max_retries_exceeded",
            "details": str(e)
        }

    except Exception as e:
        return {
            "success": False,
            "error": "unknown_error",
            "details": str(e)
        }
```

## Common Patterns

### Optional with Defaults

```python
class Config(BaseModel):
    name: str
    timeout: int = 30
    retries: int = 3
    debug: bool = False
    tags: list[str] = []
```

### Union Types

```python
from typing import Union

class TextContent(BaseModel):
    type: Literal["text"] = "text"
    text: str

class ImageContent(BaseModel):
    type: Literal["image"] = "image"
    url: str
    alt_text: str

class Message(BaseModel):
    content: Union[TextContent, ImageContent]
```

### Recursive Structures

```python
from __future__ import annotations

class TreeNode(BaseModel):
    value: str
    children: list[TreeNode] = []
```

## Best Practices

1. **Start with simple schemas**: Add complexity as needed
2. **Use descriptive field names**: LLM uses them for context
3. **Add Field descriptions**: Help LLM understand requirements
4. **Set reasonable max_retries**: 2-3 usually sufficient
5. **Handle validation errors**: Don't assume success
6. **Test with edge cases**: Empty strings, extreme values

```python
class Product(BaseModel):
    name: str = Field(description="Product name, max 100 chars")
    price: float = Field(ge=0, description="Price in USD")
    in_stock: bool = Field(description="Whether item is available")
    sku: str = Field(pattern=r"^[A-Z]{3}-\d{4}$", description="Format: XXX-0000")
```

## Resources

- [Instructor Documentation](https://python.useinstructor.com/)
- [Pydantic LLM Guide](https://pydantic.dev/articles/llm-intro)
- [OpenAI Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs)

---

*Questions about LLM output validation? [Let me know](mailto:jordan@jordananderson.us).*
