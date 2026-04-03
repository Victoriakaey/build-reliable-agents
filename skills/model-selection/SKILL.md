---
name: model-selection
description: "Use when choosing a model for an LLM application, deciding between API and local deployment, or evaluating whether fine-tuning is needed. Guides through deployment decision (API vs local), model selection based on task requirements and constraints, configuration setup, and the prompt engineering vs fine-tuning decision. Records user's setup for future reference."
---

# Model Selection

## The Core Problem

Choosing a model is not just a technical decision — it's a cost, latency, privacy, and capability tradeoff. The most common mistakes are: using the most powerful model everywhere (expensive and slow), not considering local models when privacy matters, and jumping to fine-tuning when better prompting would have worked.

**The discipline:** Understand your constraints first. The right model is the weakest model that reliably solves your problem.

---

## Step 1: Understand Your Constraints

Before looking at any specific model, answer these questions:

```
What is the task?
(text generation, classification, summarization, code, reasoning, etc.)

What is the acceptable latency?
(< 1s for real-time UX, 1-10s is acceptable, > 10s is background/batch)

What is the cost sensitivity?
(prototype: cost doesn't matter, production: minimize cost per query)

Does the data contain sensitive or private information?
(PII, proprietary data, medical records, legal documents)

Do you need the model to run offline or on-device?
(air-gapped environment, no internet, edge deployment)

What is the deployment environment?
(cloud server, local laptop, edge device, mobile)

What context window size do you need?
(short queries, long documents, entire codebases)
```

---

## Step 2: API vs Local Model

This is the first and most important decision.

### Use API (cloud-hosted) when:
- You need the best possible capability
- Latency of 1-5s is acceptable
- Data is not sensitive or you trust the provider's privacy policy
- You don't want to manage infrastructure
- You're prototyping or in early stages
- Usage is unpredictable (API scales automatically)

### Use local model when:
- Data is sensitive, private, or regulated (PII, medical, legal, financial)
- You need offline capability or air-gapped environment
- You want zero per-query cost after setup
- Latency requirements are strict and you have good hardware
- You want full control over the model and its outputs
- You're building for edge or on-device deployment

### The hybrid pattern
Many production systems use both:
- Local model for sensitive or high-volume operations
- API model for complex reasoning that requires maximum capability
- Local model as fallback when API is unavailable

---

## Step 3A: If Using API Models

### Provider comparison

| Provider | Flagship Model | Best For | Context Window |
|---|---|---|---|
| Anthropic | Claude Sonnet 4.6 / Opus 4.6 | Reasoning, long context, instruction following | 200K tokens |
| OpenAI | GPT-4o / o3 / o4-mini | Code, structured output, tool use, reasoning | 128K-200K tokens |
| Google | Gemini 2.5 Pro | Multimodal, very long context | 1M tokens |
| Mistral | Mistral Large | European data residency, cost-efficient | 128K tokens |
| Groq | Llama 3.3 70B | Ultra-low latency API, open weights | 128K tokens |

### Model tier selection

**Use the smallest model that works.** Start with the medium tier as your baseline, then try smaller if it works or larger if it doesn't.

| Tier | Examples | Use When |
|---|---|---|
| Small / Fast | Claude Haiku 4.5, GPT-4o-mini, Gemini Flash | Classification, simple extraction, high volume, cost-sensitive |
| Medium | Claude Sonnet 4.6, GPT-4o, Gemini Pro | Most production tasks, good balance of capability and cost |
| Large / Best | Claude Opus 4.6, o3, o4-mini (reasoning) | Complex reasoning, difficult tasks where quality matters most |

**Practical approach:**
1. Start with a medium-tier model
2. If it works reliably → try the small tier (cheaper)
3. If it doesn't work reliably → try the large tier (more capable)
4. If large tier still fails → the problem may need fine-tuning or a different approach

### API configuration to record

```
Provider: [Anthropic / OpenAI / Google / Mistral / other]
Model: [exact model name and version]
Temperature: [0 for deterministic, 0.7 for creative]
Max tokens: [set explicitly — never leave unlimited]
Top-p: [usually leave at default]
System prompt: [yes/no — stored where?]
API key storage: [environment variable name]
Rate limits: [requests per minute, tokens per minute]
Estimated cost per query: [input tokens × rate + output tokens × rate]
```

---

## Step 3B: If Using Local Models

### Hardware requirements by model size

| Model Size | Min RAM (CPU) | Recommended RAM | Min VRAM (GPU) | Recommended VRAM |
|---|---|---|---|---|
| 1-3B params | 4GB | 8GB | 2GB | 4GB |
| 7-8B params | 8GB | 16GB | 4GB | 8GB |
| 13-14B params | 16GB | 32GB | 8GB | 12GB |
| 30-34B params | 32GB | 64GB | 16GB | 24GB |
| 70B params | 64GB | 128GB | 32GB | 48GB+ |

**GPU vs CPU:** GPU is 5-20x faster for inference. CPU-only is fine for development and low-latency-tolerant production use.

### Local model options by use case

**General purpose / instruction following:**
- Llama 3.2 3B — fast, good for simple tasks, fits on any hardware
- Llama 3.1 8B — good balance, recommended starting point
- Llama 3.3 70B — near-API quality, needs good hardware
- Mistral 7B / Mixtral 8x7B — strong for its size, good instruction following

**Code:**
- Qwen2.5-Coder 7B / 14B — best open-source code model at its size
- DeepSeek-Coder V2 — strong code capabilities
- CodeLlama 13B — solid, well-tested

**Reasoning:**
- Qwen3 8B / 14B — strong reasoning, supports thinking mode
- DeepSeek-R1 (distilled) — reasoning-focused, various sizes

> **Note:** Local model options evolve rapidly. Check Ollama's library or Hugging Face's Open LLM Leaderboard for current rankings before committing to a model.

**Embeddings:**
- nomic-embed-text — good general embeddings, small and fast
- snowflake-arctic-embed — strong retrieval performance
- mxbai-embed-large — high quality, larger

**Multimodal (vision + text):**
- Llama 3.2 Vision 11B — good vision capability, reasonable size
- Qwen2.5-VL 7B — strong multimodal performance

### Local deployment options

| Tool | Best For | Notes |
|---|---|---|
| **Ollama** | Development, easy setup, Mac/Linux/Windows | Simple CLI, automatic model management, OpenAI-compatible API at `localhost:11434` |
| **llama.cpp** | CPU inference, edge deployment, maximum control | Low-level, fastest CPU inference |
| **vLLM** | Production API server, high throughput, GPU | PagedAttention for efficiency, OpenAI-compatible |
| **LM Studio** | GUI-based, non-technical users | Easy model management, good for exploration |
| **Hugging Face Transformers** | Research, custom pipelines, fine-tuning | Most flexible, highest overhead |

**Recommended for most cases:** Ollama for development, vLLM for production GPU serving.

### Local model configuration to record

```
Model: [model name and version]
Model file: [path or Ollama tag]
Quantization: [Q4_K_M / Q5_K_M / Q8_0 / fp16 — higher = better quality, more memory]
Context window: [num_ctx in Ollama]
Deployment tool: [Ollama / llama.cpp / vLLM / other]
API endpoint: [http://localhost:11434 for Ollama]
Temperature: [0 for deterministic, higher for creative]
Hardware: [CPU only / GPU model / RAM available]
Inference speed: [tokens per second — measure this]
```

---

## Step 4: Prompt Engineering vs Fine-Tuning

**Always try prompt engineering first.** Fine-tuning is expensive, time-consuming, and introduces a maintenance burden. Most problems that seem to need fine-tuning can be solved with better prompting.

### Try prompt engineering first when:
- The base model knows how to do the task but produces inconsistent output format
- You need the model to follow specific style or tone guidelines
- The task requires few-shot examples to guide behavior
- You haven't yet tried chain-of-thought, system prompts, or structured output

### Consider fine-tuning when:
- Prompt engineering has been thoroughly tried and still fails
- You need highly consistent structured output and the model keeps deviating
- The task requires domain-specific knowledge not in the base model
- You have 100+ high-quality labeled examples (see minimum data requirements below)
- Latency is critical and a smaller fine-tuned model can replace a larger base model
- You need to reduce prompt token usage significantly (system prompt baked into weights)

### Fine-tuning options

| Method | Use When | Tools |
|---|---|---|
| **Full fine-tuning** | You have significant compute and want maximum performance | HuggingFace Trainer, Axolotl |
| **LoRA / QLoRA** | Limited compute, most common approach | PEFT library, Unsloth (faster) |
| **RLHF / DPO** | You have preference data (chosen vs rejected) | TRL library |
| **API fine-tuning** | You want fine-tuning without managing infrastructure | OpenAI fine-tuning API, Together AI |

**Minimum data requirements:**
- Classification / structured output: 50-200 examples
- Style / tone: 100-500 examples
- Domain knowledge: 500-2000 examples
- Complex reasoning: 1000+ examples

---

## Step 5: Record Your Setup

Every model configuration decision should be recorded. This prevents "I forgot which model we were using" and makes it easy to reproduce results.

```markdown
## Model Configuration — [Project Name]

**Date:** YYYY-MM-DD
**Task:** [what this model is doing]

### Deployment
- Type: API / Local
- Provider/Tool: [name]
- Model: [exact name and version]

### Configuration
- Temperature: [value]
- Max tokens: [value]
- Context window used: [value]
- System prompt: [yes/no — location if yes]

### Hardware (if local)
- CPU/GPU: [model]
- RAM: [amount]
- VRAM: [amount if GPU]
- Quantization: [value]
- Inference speed: [tokens/second]

### Cost (if API)
- Input cost: [$X per 1M tokens]
- Output cost: [$X per 1M tokens]
- Estimated cost per query: [$X]
- Estimated monthly cost at [N] queries/day: [$X]

### Decision Log
- Why this model over alternatives: [reason]
- What was tried before: [if anything]
- Known limitations: [what it struggles with]

### Fine-tuning (if applicable)
- Base model: [name]
- Method: [LoRA / full / API]
- Training data: [size, source]
- Evaluation metric: [what improved and by how much]
```

---

## Quick Decision Tree

```
Do you have sensitive/private data, or need offline?
├── Yes → Local model
│   ├── Have GPU with 8GB+ VRAM? → Llama 3.1 8B via Ollama (start here)
│   ├── CPU only? → Llama 3.2 3B via Ollama (fastest on CPU)
│   └── Need best quality? → Llama 3.3 70B (needs 32GB+ RAM)
└── No → API model
    ├── Prototyping / cost doesn't matter → Claude Sonnet or GPT-4o
    ├── Production, cost-sensitive, simple task → Claude Haiku or GPT-4o-mini
    ├── Production, quality matters → Claude Sonnet or GPT-4o
    └── Complex reasoning, quality critical → Claude Opus or o3

Is output format inconsistent or task-specific?
├── Haven't tried few-shot examples yet → Try prompt engineering first (see prompt-design skill)
├── Tried prompt engineering, still inconsistent → Consider fine-tuning
└── Need domain knowledge not in base model → Fine-tuning with domain data
```