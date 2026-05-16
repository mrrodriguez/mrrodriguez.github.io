---
layout: post
title: "Local LLM Delegation MCP"
author: Mike Rodriguez
date: 2026-05-15
tags:
  [
    mcp,
    llm,
    local-llm,
    delegation,
    automation,
    local-llm-delegation-mcp,
    claude,
    gemini,
    ollama,
    omlx,
    qwen,
    ai,
    agents,
  ]
---

# Introduction

I have recently been exploring running local LLMs, primarily to see how they perform in coding and complex reasoning tasks compared to frontier cloud-hosted models. I am currently using a MacBook Pro M5 Max with 128GB of memory and a 40-core GPU. This gives me plenty of room to explore relatively large local models, though this approach still applies even with more modest hardware.

I found local models to be fairly capable, but they can be limiting when it comes to larger, more complicated architectural tasks. However, it occurred to me that I could still try using them for fit-for-purpose tasks—similar to how tools like Claude Code or Gemini CLI use model choice orchestration to delegate simpler tasks to cheaper or lighter models.

# Local LLM Delegation MCP

To explore this further, I created a new repository: [local-llm-delegation-mcp](https://github.com/mrrodriguez/local-llm-delegation-mcp).

This is a basic Python MCP server that can be configured for use by another agentic coding harness, particularly one running a larger, more complex model. With the correct MCP configuration and instructions, a harness like Claude Code or Gemini CLI can delegate specific tasks to your local LLM whenever it deems appropriate. The primary model then acts as a prompter and validator for those tasks.

In theory, this should save tokens and possibly even improve efficiency by leveraging your local hardware and the capabilities of freely available models.

The setup instructions can be found in the [repo README](https://github.com/mrrodriguez/local-llm-delegation-mcp/blob/main/README.md).

# Conclusion

By delegating routine tasks to local models, we can leverage our own hardware to reduce costs and latency without sacrificing the power of frontier models for complex architectural work. This hybrid approach represents a practical path forward for building more efficient and sustainable AI-powered development workflows.
