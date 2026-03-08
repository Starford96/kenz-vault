---
title: "GCP Agent Engine: Фреймворки та моделі"
tags:
  - gcp
  - ai-agents
  - vertex-ai
  - langchain
  - langgraph
  - adk
created: 2026-02-26
---

# GCP Agent Engine: Фреймворки та моделі

> Частина [[Research/GCP-Vertex-AI-Agent-Engine|Vertex AI Agent Engine]]

## Підтримувані фреймворки

### Full Integration (повна інтеграція)

#### ADK (Agent Development Kit)
- Рідний фреймворк Google для побудови агентів
- Найтісніша інтеграція з Agent Engine
- Автоматичне управління [[Research/GCP-Agent-Engine-Sessions|сесіями]] та [[Research/GCP-Agent-Engine-Memory-Bank|пам'яттю]]
- Документація: https://google.github.io/adk-docs/
- Клас: `AdkApp` у Vertex AI SDK

#### LangChain
- Класичний фреймворк для побудови LLM-ланцюжків
- Клас: `LangchainAgent` у Vertex AI SDK
- Підтримує function calling, RAG, structured output

#### LangGraph
- Фреймворк для графів станів (stateful orchestration)
- Клас: `LanggraphAgent` у Vertex AI SDK
- Підтримує checkpointing, human-in-the-loop, branching
- Ідеальний для складних multi-step workflows

### Vertex AI SDK Integration

#### AG2
- Мультиагентний фреймворк (раніше AutoGen)
- Документація: https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/develop/ag2

#### LlamaIndex (Query Pipeline)
- Фреймворк для RAG та data-aware агентів
- Документація: https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/develop/llama-index/query-pipeline

### Custom Template
- **CrewAI** — через custom template
- **Будь-який Python-фреймворк** — через custom agent class

## Підтримувані моделі

Agent Engine **не обмежує моделі** — він запускає Python-код, який сам вирішує яку модель викликати. Ось що підтримується "з коробки":

### Через Vertex AI (ChatVertexAI)
- **Gemini 2.5 Flash** — gemini-2.5-flash
- **Gemini 2.0 Flash** — gemini-2.0-flash (за замовчуванням у прикладах)
- **Gemini 1.5 Pro** — gemini-1.5-pro
- **Gemini 1.5 Flash** — gemini-1.5-flash
- Будь-яка модель з Vertex AI Model Garden

### Через ChatAnthropic (LangGraph/LangChain)
- Claude 3 Opus, Sonnet, Haiku
- Claude 3.5 Sonnet
- Потрібен API key Anthropic

### Через ChatOpenAI + Gemini ChatCompletions API
- Gemini моделі через OpenAI-compatible endpoint
- Meta Llama через Vertex AI (наприклад, `meta/llama3-405b-instruct-maas`)

### Через custom model_builder
- Будь-яка модель з LangChain Chat Models
- Самостійний HTTP-клієнт до будь-якого API

## Приклад: LangGraph з Anthropic

```python
def model_builder(*, model_name, model_kwargs=None, **kwargs):
    from langchain_anthropic import ChatAnthropic
    return ChatAnthropic(model_name=model_name, **model_kwargs)

agent = agent_engines.LanggraphAgent(
    model="claude-3-opus-20240229",
    model_builder=model_builder,
    model_kwargs={"api_key": "...", "temperature": 0.28},
)
```

## Приклад: ADK з Gemini

```python
from google.adk.agents import Agent

agent = Agent(
    model="gemini-2.0-flash",
    name="my_agent",
    tools=[my_tool],
)
```