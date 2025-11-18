---
layout: default
title:  "Conversational AI: Context Management"
date:   2025-11-17 11:00:00
categories: AI LLM Chatbot Memory
---

LLMs are stateless by default - each request is processed independently. Building conversational AI requires managing context across turns while respecting token limits and costs. Here's how to implement effective context management.

## The Problem

LLMs don't remember previous interactions. You must:
- Pass conversation history with each request
- Stay within context window limits (4K - 128K+ tokens)
- Manage costs (you pay for every token)
- Preserve relevant context while discarding noise

## Basic Conversation Buffer

The simplest approach - keep everything:

```python
from openai import OpenAI

client = OpenAI()

class ConversationBuffer:
    def __init__(self, system_prompt: str = ""):
        self.messages = []
        if system_prompt:
            self.messages.append({
                "role": "system",
                "content": system_prompt
            })

    def add_user_message(self, content: str):
        self.messages.append({"role": "user", "content": content})

    def add_assistant_message(self, content: str):
        self.messages.append({"role": "assistant", "content": content})

    def get_response(self, user_input: str) -> str:
        self.add_user_message(user_input)

        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=self.messages
        )

        assistant_message = response.choices[0].message.content
        self.add_assistant_message(assistant_message)

        return assistant_message

# Usage
chat = ConversationBuffer("You are a helpful assistant.")
print(chat.get_response("What's the capital of France?"))
print(chat.get_response("What's its population?"))  # Remembers "France"
```

**Pros**: Simple, preserves all context
**Cons**: Unbounded growth, will hit token limits

## Sliding Window Memory

Keep only the K most recent exchanges:

```python
class SlidingWindowMemory:
    def __init__(self, system_prompt: str, window_size: int = 10):
        self.system_prompt = system_prompt
        self.window_size = window_size
        self.messages = []

    def add_exchange(self, user_msg: str, assistant_msg: str):
        self.messages.append({"role": "user", "content": user_msg})
        self.messages.append({"role": "assistant", "content": assistant_msg})

        # Trim to window size (keeping pairs)
        if len(self.messages) > self.window_size * 2:
            self.messages = self.messages[-(self.window_size * 2):]

    def get_messages(self) -> list:
        return [
            {"role": "system", "content": self.system_prompt},
            *self.messages
        ]

    def get_response(self, user_input: str) -> str:
        messages = self.get_messages()
        messages.append({"role": "user", "content": user_input})

        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=messages
        )

        assistant_msg = response.choices[0].message.content
        self.add_exchange(user_input, assistant_msg)

        return assistant_msg
```

**Pros**: Bounded memory, simple
**Cons**: Loses early context abruptly

## Token-Based Window

More precise - limit by actual tokens:

```python
import tiktoken

class TokenWindowMemory:
    def __init__(
        self,
        system_prompt: str,
        max_tokens: int = 4000,
        model: str = "gpt-4o-mini"
    ):
        self.system_prompt = system_prompt
        self.max_tokens = max_tokens
        self.messages = []
        self.encoder = tiktoken.encoding_for_model(model)

    def count_tokens(self, messages: list) -> int:
        """Count tokens in message list."""
        total = 0
        for msg in messages:
            # ~4 tokens overhead per message
            total += 4
            total += len(self.encoder.encode(msg["content"]))
        return total

    def trim_to_token_limit(self):
        """Remove oldest messages to fit token limit."""
        system_msg = [{"role": "system", "content": self.system_prompt}]
        system_tokens = self.count_tokens(system_msg)

        while self.messages:
            total = system_tokens + self.count_tokens(self.messages)
            if total <= self.max_tokens:
                break

            # Remove oldest pair (user + assistant)
            if len(self.messages) >= 2:
                self.messages = self.messages[2:]
            else:
                self.messages = []

    def add_exchange(self, user_msg: str, assistant_msg: str):
        self.messages.append({"role": "user", "content": user_msg})
        self.messages.append({"role": "assistant", "content": assistant_msg})
        self.trim_to_token_limit()

    def get_messages(self) -> list:
        return [
            {"role": "system", "content": self.system_prompt},
            *self.messages
        ]
```

## Summary Memory

Compress old conversations into summaries:

```python
class SummaryMemory:
    def __init__(
        self,
        system_prompt: str,
        summary_threshold: int = 10,
        keep_recent: int = 4
    ):
        self.system_prompt = system_prompt
        self.summary_threshold = summary_threshold
        self.keep_recent = keep_recent
        self.summary = ""
        self.messages = []

    async def summarize_messages(self, messages: list) -> str:
        """Create summary of conversation."""
        conversation = "\n".join([
            f"{m['role']}: {m['content']}"
            for m in messages
        ])

        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{
                "role": "user",
                "content": f"""Summarize this conversation, preserving key facts,
                decisions, and context needed for continuation:

                {conversation}

                Summary:"""
            }],
            max_tokens=500
        )

        return response.choices[0].message.content

    async def add_exchange(self, user_msg: str, assistant_msg: str):
        self.messages.append({"role": "user", "content": user_msg})
        self.messages.append({"role": "assistant", "content": assistant_msg})

        # Check if we need to summarize
        if len(self.messages) > self.summary_threshold * 2:
            # Messages to summarize
            to_summarize = self.messages[:-(self.keep_recent * 2)]

            # Create new summary
            if self.summary:
                to_summarize.insert(0, {
                    "role": "system",
                    "content": f"Previous summary: {self.summary}"
                })

            self.summary = await self.summarize_messages(to_summarize)

            # Keep only recent messages
            self.messages = self.messages[-(self.keep_recent * 2):]

    def get_messages(self) -> list:
        messages = [{"role": "system", "content": self.system_prompt}]

        if self.summary:
            messages.append({
                "role": "system",
                "content": f"Conversation summary: {self.summary}"
            })

        messages.extend(self.messages)
        return messages
```

## Hybrid: Summary + Buffer

Best of both worlds:

```python
class HybridMemory:
    def __init__(
        self,
        system_prompt: str,
        max_tokens: int = 4000,
        summary_tokens: int = 500
    ):
        self.system_prompt = system_prompt
        self.max_tokens = max_tokens
        self.summary_tokens = summary_tokens
        self.summary = ""
        self.messages = []
        self.encoder = tiktoken.encoding_for_model("gpt-4o-mini")

    def count_tokens(self, text: str) -> int:
        return len(self.encoder.encode(text))

    async def maybe_summarize(self):
        """Summarize if approaching token limit."""
        current_tokens = sum(
            self.count_tokens(m["content"]) for m in self.messages
        )

        if current_tokens > self.max_tokens - self.summary_tokens:
            # Take first half of messages for summary
            midpoint = len(self.messages) // 2
            to_summarize = self.messages[:midpoint]

            if self.summary:
                summary_text = f"Previous context: {self.summary}\n\n"
            else:
                summary_text = ""

            summary_text += "\n".join([
                f"{m['role']}: {m['content']}"
                for m in to_summarize
            ])

            response = client.chat.completions.create(
                model="gpt-4o-mini",
                messages=[{
                    "role": "user",
                    "content": f"Summarize this conversation context:\n\n{summary_text}"
                }],
                max_tokens=self.summary_tokens
            )

            self.summary = response.choices[0].message.content
            self.messages = self.messages[midpoint:]

    def get_messages(self) -> list:
        messages = [{"role": "system", "content": self.system_prompt}]

        if self.summary:
            messages.append({
                "role": "system",
                "content": f"Previous conversation context:\n{self.summary}"
            })

        messages.extend(self.messages)
        return messages
```

## Semantic Memory with RAG

Retrieve relevant past context on demand:

```python
from sentence_transformers import SentenceTransformer
import numpy as np

class SemanticMemory:
    def __init__(self, system_prompt: str, top_k: int = 5):
        self.system_prompt = system_prompt
        self.top_k = top_k
        self.model = SentenceTransformer('all-MiniLM-L6-v2')
        self.exchanges = []  # (embedding, user_msg, assistant_msg)
        self.recent_messages = []

    def add_exchange(self, user_msg: str, assistant_msg: str):
        # Embed the exchange
        combined = f"User: {user_msg}\nAssistant: {assistant_msg}"
        embedding = self.model.encode(combined)

        self.exchanges.append((embedding, user_msg, assistant_msg))

        # Keep recent for immediate context
        self.recent_messages.append({"role": "user", "content": user_msg})
        self.recent_messages.append({"role": "assistant", "content": assistant_msg})

        # Limit recent
        if len(self.recent_messages) > 6:
            self.recent_messages = self.recent_messages[-6:]

    def retrieve_relevant(self, query: str) -> list:
        """Find most relevant past exchanges."""
        if not self.exchanges:
            return []

        query_embedding = self.model.encode(query)

        # Calculate similarities
        similarities = []
        for emb, user_msg, assistant_msg in self.exchanges:
            sim = np.dot(query_embedding, emb) / (
                np.linalg.norm(query_embedding) * np.linalg.norm(emb)
            )
            similarities.append((sim, user_msg, assistant_msg))

        # Get top-k
        similarities.sort(reverse=True)
        top = similarities[:self.top_k]

        messages = []
        for _, user_msg, assistant_msg in top:
            messages.append({"role": "user", "content": user_msg})
            messages.append({"role": "assistant", "content": assistant_msg})

        return messages

    def get_messages(self, current_query: str) -> list:
        messages = [{"role": "system", "content": self.system_prompt}]

        # Add relevant past context
        relevant = self.retrieve_relevant(current_query)
        if relevant:
            messages.append({
                "role": "system",
                "content": "Relevant past conversation:"
            })
            messages.extend(relevant)
            messages.append({
                "role": "system",
                "content": "Recent conversation:"
            })

        # Add recent messages
        messages.extend(self.recent_messages)

        return messages
```

## Entity Memory

Track specific entities mentioned in conversation:

```python
class EntityMemory:
    def __init__(self, system_prompt: str):
        self.system_prompt = system_prompt
        self.entities = {}  # entity_name -> info
        self.recent_messages = []

    async def extract_entities(self, text: str) -> dict:
        """Extract entities from text using LLM."""
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{
                "role": "user",
                "content": f"""Extract entities and their attributes from this text.
                Return as JSON with entity names as keys and attributes as values.

                Text: {text}

                Example output:
                {{"John": {{"role": "developer", "location": "NYC"}},
                 "Project X": {{"status": "in progress", "deadline": "next week"}}}}"""
            }],
            response_format={"type": "json_object"}
        )

        return json.loads(response.choices[0].message.content)

    async def add_exchange(self, user_msg: str, assistant_msg: str):
        # Extract entities from both messages
        user_entities = await self.extract_entities(user_msg)
        assistant_entities = await self.extract_entities(assistant_msg)

        # Merge into memory
        for entity, info in {**user_entities, **assistant_entities}.items():
            if entity in self.entities:
                self.entities[entity].update(info)
            else:
                self.entities[entity] = info

        # Store recent
        self.recent_messages.append({"role": "user", "content": user_msg})
        self.recent_messages.append({"role": "assistant", "content": assistant_msg})

        if len(self.recent_messages) > 10:
            self.recent_messages = self.recent_messages[-10:]

    def get_messages(self) -> list:
        messages = [{"role": "system", "content": self.system_prompt}]

        # Add entity context
        if self.entities:
            entity_context = "Known entities:\n"
            for name, info in self.entities.items():
                entity_context += f"- {name}: {json.dumps(info)}\n"

            messages.append({
                "role": "system",
                "content": entity_context
            })

        messages.extend(self.recent_messages)
        return messages
```

## Production Implementation

Complete context manager with multiple strategies:

```python
from enum import Enum

class MemoryStrategy(Enum):
    BUFFER = "buffer"
    SLIDING_WINDOW = "sliding_window"
    SUMMARY = "summary"
    SEMANTIC = "semantic"

class ProductionContextManager:
    def __init__(
        self,
        system_prompt: str,
        strategy: MemoryStrategy = MemoryStrategy.SUMMARY,
        max_tokens: int = 4000,
        model: str = "gpt-4o-mini"
    ):
        self.system_prompt = system_prompt
        self.strategy = strategy
        self.max_tokens = max_tokens
        self.model = model

        # Initialize appropriate memory
        if strategy == MemoryStrategy.BUFFER:
            self.memory = ConversationBuffer(system_prompt)
        elif strategy == MemoryStrategy.SLIDING_WINDOW:
            self.memory = SlidingWindowMemory(system_prompt, window_size=10)
        elif strategy == MemoryStrategy.SUMMARY:
            self.memory = HybridMemory(system_prompt, max_tokens)
        elif strategy == MemoryStrategy.SEMANTIC:
            self.memory = SemanticMemory(system_prompt)

    async def chat(self, user_input: str) -> str:
        """Process user input and return response."""
        # Get context-aware messages
        if self.strategy == MemoryStrategy.SEMANTIC:
            messages = self.memory.get_messages(user_input)
        else:
            messages = self.memory.get_messages()

        messages.append({"role": "user", "content": user_input})

        # Get response
        response = client.chat.completions.create(
            model=self.model,
            messages=messages
        )

        assistant_msg = response.choices[0].message.content

        # Update memory
        if hasattr(self.memory, 'add_exchange'):
            await self.memory.add_exchange(user_input, assistant_msg)
        else:
            self.memory.add_user_message(user_input)
            self.memory.add_assistant_message(assistant_msg)

        return assistant_msg

    def get_token_usage(self) -> int:
        """Get current token usage."""
        messages = self.memory.get_messages()
        encoder = tiktoken.encoding_for_model(self.model)

        total = 0
        for msg in messages:
            total += len(encoder.encode(msg["content"])) + 4

        return total
```

## Persistent Storage

Save context across sessions:

```python
import redis
import pickle

class PersistentMemory:
    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis = redis.from_url(redis_url)

    def save_session(self, session_id: str, memory: dict):
        """Save memory state to Redis."""
        self.redis.setex(
            f"memory:{session_id}",
            86400,  # 24 hour TTL
            pickle.dumps(memory)
        )

    def load_session(self, session_id: str) -> dict | None:
        """Load memory state from Redis."""
        data = self.redis.get(f"memory:{session_id}")
        if data:
            return pickle.loads(data)
        return None

    def delete_session(self, session_id: str):
        """Clear session memory."""
        self.redis.delete(f"memory:{session_id}")
```

## Choosing a Strategy

| Strategy | Use Case | Token Efficiency | Context Preservation |
|----------|----------|------------------|---------------------|
| Buffer | Short conversations | Low | Perfect |
| Sliding Window | Real-time chat | Medium | Recent only |
| Summary | Long conversations | High | Good for facts |
| Semantic | Knowledge queries | High | Relevant only |
| Hybrid | General purpose | High | Good balance |

## Best Practices

1. **Match strategy to use case**: Support chat needs different memory than document Q&A
2. **Monitor token usage**: Track and alert on high usage
3. **Test context loss**: Verify important info survives summarization
4. **Use session IDs**: Enable persistence and debugging
5. **Implement fallbacks**: Handle memory corruption gracefully

## Resources

- [LangChain Conversational Memory](https://www.pinecone.io/learn/series/langchain/langchain-conversational-memory/)
- [Vellum Memory Guide](https://www.vellum.ai/blog/how-should-i-manage-memory-for-my-llm-chatbot)
- [Mem0 History Summarization](https://mem0.ai/blog/llm-chat-history-summarization-guide-2025)
- [Spring AI Chat Memory](https://www.danvega.dev/blog/2024/10/11/spring-ai-chat-memory)

---

*Questions about context management? [Let me know](mailto:jordan@jordananderson.us).*
