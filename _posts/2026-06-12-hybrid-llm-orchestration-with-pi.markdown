---
layout: post
title: "Hybrid LLM Orchestration with Pi"
author: Mike Rodriguez
date: 2026-06-12
tags:
  [
    pi,
    coding-agent,
    local-llm,
    hybrid-workflow,
    apple-silicon,
    ollama,
    mlx,
    deepseek,
    dwarf-star,
    ai-orchestration,
  ]
---

# Introduction

As costs increase for agentic workflows with cloud hosted frontier model
providers increases, there is more interest in running local models. There are
also privacy concerns beyond cost to consider. There are obvious advantages to
frontier cloud models in terms of their ability to utilize world class hardware
resources that are not generally available to consumers. However, not all
agentic tasks demand equally high resource availability.

<!-- prettier-ignore -->
* TOC
{:toc}

The **Pi coding agent harness** (see
[https://github.com/earendil-works/pi](https://github.com/earendil-works/pi))
is a popular and awesome harness for both local and cloud hosted models. It is
also a great choice to explore this type of cloud vs local agent orchestration.
I'm going to provide a brief walk through how I've configured my environment to
have options ranging from basic local execution setups to a a hybrid workflow
using DeepSeek V4 cloud hosted model with delegation to locally running models.
I'll be demonstrating using Apple Silicon, although many of the ideas apply
regardless.

# A Note on Apple Silicon

The M-series chips are a great fit for running local LLMs due to allowing the
GPU to access the same high-speed RAM pool as the CPU.

- **Unified Memory**: With 128GB of RAM on an M5 Max, I can run 30B+ parameter
  models with 128k context windows.
- **The Hardware Scale**: While you can get started with 32GB or 64GB, the
  "sweet spot" for reasoning-capable local models starts at 128GB. Higher specs
  simply unlock larger, more capable models (like Qwen 72B or full-quant
  DeepSeek).
- **Metal Acceleration**: Using frameworks like **MLX** ensures that inference
  is fully optimized for the GPU cores, making local coding feel "snappy" rather
  than sluggish.

---

# Step 1: Starting with the Basics (Local-Only)

If you're just starting, you can run Pi entirely offline. This is perfect for
air-gapped work and toavoiding cloud API costs and latency.

### Option A: Using Ollama

Ollama is a popular and relatively easiest way to manage GGUF models. It
handles the server lifecycle and model pulling automatically.

1.  **Pull the model**: `ollama pull qwen3.6:35b`
2.  **Launch Pi**:
    ```bash
    pi --model ollama/qwen3.6:35b
    ```
3.  **Config**: See the [Ollama models.json configuration](#configuring-the-harness) below.

### Option B: Using oMLX

On Apple Silicon, **oMLX** (see
[https://github.com/jundot/omlx](https://github.com/jundot/omlx)) provides
native MLX quantization, which is often faster and more memory-efficient than
GGUF.

1.  **Configure the provider**: Ensure your `models.json` points to your local
    oMLX server (typically port 8000).
2.  **Launch Pi**:
    ```bash
    pi --model omlx/Qwen3-Coder-Next-MLX-8bit
    ```
3.  **Config**: See the [oMLX models.json configuration](#configuring-the-harness) below.

---

# Step 2: Using Cloud Hosted DeepSeek V4 (Non-Local)

Sometimes local models lack the "nuance" needed for complex architectural
decisions. In these cases, I'll demonstrate using **DeepSeek V4** via their
hosted API. It’s incredibly cost-effective while providing frontier-level
reasoning (see
[https://api-docs.deepseek.com/quick_start/pricing](https://api-docs.deepseek.com/quick_start/pricing)).

1.  **Get your API Key**: Set `DEEPSEEK_API_KEY` in your environment.
2.  **Run Pi**:
    ```bash
    pi --model deepseek/deepseek-chat
    ```
3.  **Config**: See the [DeepSeek models.json configuration](#configuring-the-harness) below.

---

# Step 3: The Hybrid Workflow (Cloud + Local Delegation)

We use the Cloud model (DeepSeek V4) for planning and decision-making, but we
delegate the "labor" to a local model via the `delegate_to_local` tool.

### How it works

Using the
[`delegate-local.ts`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/extensions/delegate-local.ts)
extension from my
[pi-agent-recipes](https://github.com/mrrodriguez/pi-agent-recipes) repo, Pi
gains a tool that allows the primary model to spin up a "sub-agent" running on
your local hardware.

1.  **Load the extension**: Copy
    [`delegate-local.ts`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/extensions/delegate-local.ts)
    to `~/.pi/agent/extensions/`.
2.  **Launch with Delegation**:
    ```bash
    pi --model deepseek/deepseek-chat --dm omlx/Qwen3-Coder-Next-MLX-8bit
    ```
3.  **The Result**: DeepSeek plans the refactor, but your local model actually
    writes the code. You save on tokens for large file operations as well as
    possibly snappier performance and no unnecessary data transmitted.

---

# Advanced: Local Reasoning with ds4 (Dwarf Star)

The **ds4 (Dwarf Star)** project is an attempt to push local inference to its
maximum potential. It’s a lightweight C-based inference engine designed
specifically for running models like DeepSeek locally with minimal overhead.

I use a custom
[`ds4.ts`](https://github.com/mrrodriguez/pi-agent-recipes/blob/main/extensions/ds4.ts)
extension to manage the lifecycle of the `ds4` binary, including an automatic
watchdog and log viewer.

- **Further Reading**:
  - A deep dive into why this matters:
    - [Armin Ronacher: Local Models for the Rest of
      Us](https://lucumr.pocoo.org/2026/5/8/local-models/)
  - The inspiration for local Dwarf Star integration:
    - [mitsuhiko/pi-ds4](https://github.com/mitsuhiko/pi-ds4)

---

# Configuring the Harness

To make this all work, you need a precise `models.json` configuration. Pi uses
this file to understand how to talk to each provider, handle authentication,
and correctly track usage costs.

One powerful feature of Pi's config is **shell expansion in keys**. For
example, `!echo ${OMLX_API_KEY:-omlx}` allows you to securely pull an API key
from your environment or fallback to a default value.

Here is an explicit look at the configuration for the providers discussed:

```json
{
  "providers": {
    "deepseek": {
      "models": [
        {
          "id": "deepseek-chat",
          "name": "DeepSeek Chat",
          "cost": {
            "input": 0.14,
            "output": 0.28,
            "cacheRead": 0.01,
            "cacheWrite": 0
          },
          "contextWindow": 128000,
          "maxTokens": 8192
        }
      ]
    },
    "ollama": {
      "name": "Ollama",
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [
        {
          "id": "qwen3.6:35b-a3b-coding-nvfp4",
          "name": "Qwen3.6 35B Coding (local)",
          "reasoning": true,
          "cost": { "input": 0, "output": 0 },
          "contextWindow": 128000,
          "maxTokens": 8192
        }
      ]
    },
    "omlx": {
      "name": "oMLX",
      "baseUrl": "http://localhost:8000/v1",
      "api": "openai-completions",
      "apiKey": "!echo ${OMLX_API_KEY:-omlx}",
      "models": [
        {
          "id": "Qwen3-Coder-Next-MLX-8bit",
          "name": "Qwen3 Coder Next 8bit (oMLX)",
          "reasoning": true,
          "cost": { "input": 0, "output": 0 },
          "contextWindow": 128000,
          "compat": {
            "thinkingFormat": "qwen-chat-template",
            "supportsStrictMode": true
          }
        }
      ]
    }
  }
}
```

### Why these details matter:

- **Cost Tracking**: By defining the `cost` object (per million tokens), Pi can
  provide real-time session cost estimates. This is important when using cloud
  hosted models, as it allows you to see exactly how much you're saving by
  delegating expensive operations to local models (where `cost` is set to `0`).
- **Cache Efficiency**: Note the `cacheRead` and `cacheWrite` fields for
  DeepSeek. Modern providers offer significant discounts for prompt caching; Pi
  uses these fields to accurately reflect your actual spend.
- **Precision Model IDs**: Using specific IDs like
  `qwen3.6:35b-a3b-coding-nvfp4` ensures you are using the exact quantization
  variant optimized for your hardware.
- **`baseUrl` & `api`**: Essential for directing Pi to your local inference
  servers.
- **`compat`**: Critical for reasoning-capable models like Qwen 3 to handle
  "thought" blocks correctly.

# Conclusion

The future of AI-assisted engineering may be move beyond bigger models in the
cloud and involve **smarter orchestration** to save cost and distribute
resource usage. By combining the reasoning power of a frontier model like
DeepSeek (or Claude, GPT, etc) with the privacy and speed of local MLX models,
you create a coding environment that is more reliable, cheaper, and more secure
than any pure-cloud solution.
