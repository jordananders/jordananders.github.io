---
layout: default
title:  "Open Source LLM Comparison: Llama, Mistral, and Qwen"
date:   2025-11-17 13:00:00
categories: AI LLM Open-Source
---

Open source LLMs have reached competitive performance with proprietary models. Here's a practical comparison of Llama, Mistral, and Qwen to help you choose the right model for your use case.

## Quick Comparison

| Model | Best For | Context | Parameters | License |
|-------|----------|---------|------------|---------|
| Llama 3.1 | General purpose | 128K | 8B/70B/405B | Meta License |
| Qwen 2.5 | Code & Math | 128K | 0.5B-72B | Apache 2.0 |
| Mistral | Multilingual | 32K | 7B/8x7B/8x22B | Apache 2.0 |

## Llama 3.1/3.2

Meta's flagship open source model family.

### Strengths
- Best general knowledge and reasoning
- Strong instruction following
- Excellent multilingual support (8 languages)
- Large community and ecosystem

### Models

```python
# Model options
models = {
    "llama-3.1-8b": "Fast, resource-efficient",
    "llama-3.1-70b": "Balanced performance/cost",
    "llama-3.1-405b": "Highest capability",
    "llama-3.2-1b": "Edge deployment",
    "llama-3.2-3b": "Mobile/edge",
    "llama-3.2-11b-vision": "Multimodal",
    "llama-3.2-90b-vision": "Best multimodal"
}
```

### Usage

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

model_id = "meta-llama/Llama-3.1-8B-Instruct"

tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(
    model_id,
    torch_dtype=torch.bfloat16,
    device_map="auto"
)

messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "Explain quantum computing in simple terms."}
]

input_ids = tokenizer.apply_chat_template(
    messages,
    return_tensors="pt"
).to(model.device)

outputs = model.generate(
    input_ids,
    max_new_tokens=256,
    temperature=0.7,
    do_sample=True
)

response = tokenizer.decode(outputs[0], skip_special_tokens=True)
```

### Benchmark Performance (Llama 3.1 70B)

- MMLU: 86.0%
- HumanEval: 80.5%
- MATH: 68.0%
- GSM8K: 95.1%

## Qwen 2.5

Alibaba's leading model, excelling in code and math.

### Strengths
- Best coding performance (HumanEval 85+)
- Superior math capabilities (MATH 80+)
- Cost-efficient at smaller sizes
- 72B outperforms Llama 405B on many tasks

### Models

```python
models = {
    "qwen2.5-0.5b": "Ultra-light, 0.5GB",
    "qwen2.5-1.5b": "Mobile deployment",
    "qwen2.5-7b": "Standard tasks",
    "qwen2.5-14b": "Enhanced capability",
    "qwen2.5-32b": "Strong all-around",
    "qwen2.5-72b": "Maximum performance",
    "qwen2.5-coder": "Specialized for code",
    "qwen2.5-math": "Specialized for math"
}
```

### Usage

```python
from transformers import AutoModelForCausalLM, AutoTokenizer

model_name = "Qwen/Qwen2.5-7B-Instruct"

model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype="auto",
    device_map="auto"
)
tokenizer = AutoTokenizer.from_pretrained(model_name)

messages = [
    {"role": "system", "content": "You are a helpful coding assistant."},
    {"role": "user", "content": "Write a Python function to find prime numbers."}
]

text = tokenizer.apply_chat_template(
    messages,
    tokenize=False,
    add_generation_prompt=True
)

model_inputs = tokenizer([text], return_tensors="pt").to(model.device)

generated_ids = model.generate(
    **model_inputs,
    max_new_tokens=512
)

response = tokenizer.batch_decode(generated_ids, skip_special_tokens=True)[0]
```

### Benchmark Performance (Qwen 2.5 72B)

- MMLU: 86.6%
- HumanEval: 86.6%
- MATH: 83.1%
- GSM8K: 95.8%

## Mistral

French AI lab's efficient models with strong multilingual support.

### Strengths
- Excellent multilingual capabilities
- Efficient architecture (MoE)
- Strong European language support
- Good performance per parameter

### Models

```python
models = {
    "mistral-7b": "Efficient base model",
    "mixtral-8x7b": "MoE, 46.7B params",
    "mixtral-8x22b": "Large MoE, enterprise",
    "mistral-nemo": "12B, Apache 2.0",
    "codestral": "Code-specialized"
}
```

### Usage

```python
from transformers import AutoModelForCausalLM, AutoTokenizer

model_id = "mistralai/Mistral-7B-Instruct-v0.3"

tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(
    model_id,
    torch_dtype=torch.bfloat16,
    device_map="auto"
)

messages = [
    {"role": "user", "content": "Translate to French: The weather is nice today."}
]

inputs = tokenizer.apply_chat_template(
    messages,
    return_tensors="pt"
).to("cuda")

outputs = model.generate(inputs, max_new_tokens=100)
print(tokenizer.decode(outputs[0], skip_special_tokens=True))
```

### Benchmark Performance (Mixtral 8x7B)

- MMLU: 70.6%
- HumanEval: 40.2%
- MATH: 28.4%
- MT-Bench: 8.30

## Head-to-Head Comparisons

### Coding Tasks

```python
# Winner: Qwen 2.5

# HumanEval scores (pass@1):
# Qwen 2.5 72B:   86.6%
# Llama 3.1 70B:  80.5%
# Mixtral 8x7B:   40.2%

# Recommendation: Qwen 2.5-Coder for code generation
```

### Math & Reasoning

```python
# Winner: Qwen 2.5

# MATH benchmark:
# Qwen 2.5 72B:   83.1%
# Llama 3.1 70B:  68.0%
# Mixtral 8x7B:   28.4%

# GSM8K:
# Qwen 2.5 72B:   95.8%
# Llama 3.1 70B:  95.1%
# Mixtral 8x7B:   74.4%
```

### General Knowledge

```python
# Winner: Llama 3.1 (at 405B scale)

# MMLU scores:
# Llama 3.1 405B: 88.6%
# Qwen 2.5 72B:   86.6%
# Llama 3.1 70B:  86.0%
# Mixtral 8x7B:   70.6%
```

### Multilingual

```python
# Winner: Mistral (European) / Qwen (Asian)

# Mistral: Best for French, German, Spanish, Italian
# Qwen: Best for Chinese, Japanese, Korean
# Llama: Good all-around multilingual
```

### Speed (tokens/second on H100)

```python
# With TensorRT-LLM optimization:

# 7-8B models:
# Qwen2.5-7B:    ~150 tok/s
# Llama-3.1-8B:  ~140 tok/s
# Mistral-7B:    ~145 tok/s

# 70B+ models:
# Qwen2.5-72B:   ~35 tok/s
# Llama-3.1-70B: ~30 tok/s
# Mixtral-8x7B:  ~45 tok/s (MoE efficiency)
```

## Deployment Options

### Ollama (Local)

```bash
# Install model
ollama pull llama3.1:8b
ollama pull qwen2.5:7b
ollama pull mistral:7b

# Run
ollama run llama3.1:8b "Explain REST APIs"
```

### vLLM (Production)

```python
from vllm import LLM, SamplingParams

# High-throughput serving
llm = LLM(
    model="meta-llama/Llama-3.1-8B-Instruct",
    tensor_parallel_size=1,
    gpu_memory_utilization=0.9
)

sampling_params = SamplingParams(
    temperature=0.7,
    max_tokens=256
)

outputs = llm.generate(["Write a haiku about coding"], sampling_params)
```

### API Services

```python
# Together AI
from openai import OpenAI

client = OpenAI(
    api_key="your-together-key",
    base_url="https://api.together.xyz/v1"
)

response = client.chat.completions.create(
    model="meta-llama/Llama-3.1-70B-Instruct-Turbo",
    messages=[{"role": "user", "content": "Hello!"}]
)

# Groq (fastest)
client = OpenAI(
    api_key="your-groq-key",
    base_url="https://api.groq.com/openai/v1"
)

response = client.chat.completions.create(
    model="llama-3.1-70b-versatile",
    messages=[{"role": "user", "content": "Hello!"}]
)
```

## Resource Requirements

### Memory (FP16)

| Model | VRAM Required | Consumer GPU |
|-------|--------------|--------------|
| 7-8B | 16GB | RTX 4090 |
| 13-14B | 28GB | 2x RTX 4090 |
| 32B | 64GB | A100 80GB |
| 70-72B | 140GB | 2x A100 80GB |
| 405B | 810GB | 8x H100 |

### Quantization

```python
# 4-bit quantization reduces memory by ~4x
from transformers import BitsAndBytesConfig

quantization_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16
)

# 70B model in ~35GB VRAM
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3.1-70B-Instruct",
    quantization_config=quantization_config,
    device_map="auto"
)
```

## Model Selection Guide

### Choose Llama if:
- General-purpose chatbot/assistant
- Need largest context (128K)
- Want strong community support
- Building RAG applications
- Need vision capabilities (3.2)

### Choose Qwen if:
- Building coding tools
- Math/science applications
- Need Asian language support
- Want best performance per dollar
- Enterprise applications (Apache 2.0)

### Choose Mistral if:
- European language focus
- Resource-constrained deployment
- Need MoE efficiency
- Building translation tools
- Apache 2.0 license required

## Cost Comparison (API Pricing)

| Model | Input ($/1M) | Output ($/1M) | Provider |
|-------|-------------|---------------|----------|
| Llama 3.1 8B | $0.10 | $0.10 | Together |
| Llama 3.1 70B | $0.88 | $0.88 | Together |
| Qwen 2.5 72B | $0.90 | $0.90 | Together |
| Mixtral 8x7B | $0.60 | $0.60 | Together |
| Groq Llama 70B | $0.59 | $0.79 | Groq |

## Licensing

```python
licenses = {
    "Llama": {
        "type": "Meta License",
        "commercial": True,
        "restrictions": "700M MAU limit",
        "derivatives": True
    },
    "Qwen": {
        "type": "Apache 2.0",
        "commercial": True,
        "restrictions": None,
        "derivatives": True
    },
    "Mistral": {
        "type": "Apache 2.0",
        "commercial": True,
        "restrictions": None,
        "derivatives": True
    }
}
```

## Resources

- [Llama Models](https://llama.meta.com/)
- [Qwen Models](https://github.com/QwenLM/Qwen2.5)
- [Mistral AI](https://mistral.ai/)
- [Hugging Face Open LLM Leaderboard](https://huggingface.co/spaces/open-llm-leaderboard/open_llm_leaderboard)
- [Artificial Analysis Benchmarks](https://artificialanalysis.ai/leaderboards/models)

---

*Questions about choosing an open source LLM? [Let me know](mailto:jordan@jordananderson.us).*
