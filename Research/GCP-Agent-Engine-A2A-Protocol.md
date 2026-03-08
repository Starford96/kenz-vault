---
title: "GCP Agent Engine: A2A Protocol"
tags:
  - gcp
  - ai-agents
  - vertex-ai
  - a2a
  - multi-agent
created: 2026-02-26
---

# GCP Agent Engine: A2A Protocol

> Частина [[Research/GCP-Vertex-AI-Agent-Engine|Vertex AI Agent Engine]]

Офіційна документація: https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/develop/a2a
Специфікація протоколу: https://a2a-protocol.org/

## Що це

Agent2Agent (A2A) — відкритий протокол для комунікації та колаборації між AI-агентами, незалежно від фреймворку. Дозволяє будувати мультиагентні системи, де агенти на різних платформах можуть взаємодіяти.

## Ключові компоненти

### AgentCard
"Візитка" агента — метадата-документ, що описує можливості агента. Інші агенти використовують AgentCard для discovery.

```python
from a2a.types import AgentCard, AgentSkill

currency_skill = AgentSkill(
    id="get_exchange_rate",
    name="Get Currency Exchange Rate",
    description="Retrieves the exchange rate between two currencies.",
    tags=["Finance", "Currency"],
    examples=["What is the exchange rate from USD to EUR?"],
)

agent_card = create_agent_card(
    agent_name="Currency Exchange Agent",
    description="An agent that can provide currency exchange rates",
    skills=[currency_skill],
)
```

### AgentExecutor
Основна логіка агента — як він обробляє tasks. Обгортає ADK Runner або будь-яку іншу логіку.

### LlmAgent (опціонально)
ADK-агент з системними інструкціями, моделлю та інструментами.

## Методи A2A-агента

- `handle_authenticated_agent_card` — повертає authenticated card з описом можливостей
- `on_message_send` — обробляє вхідне повідомлення, створює task
- `on_get_task` — повертає статус і результат task

## SDK

A2A Protocol доступний для багатьох мов:
- Python: https://github.com/a2aproject/a2a-python
- Go: https://github.com/a2aproject/a2a-go
- JavaScript: https://github.com/a2aproject/a2a-js
- Java: https://github.com/a2aproject/a2a-java
- C#/.NET: https://github.com/a2aproject/a2a-dotnet
- Samples: https://github.com/a2aproject/a2a-samples

## Зв'язок з Agent Engine

A2A-агент деплоїться на [[Research/GCP-Vertex-AI-Agent-Engine|Agent Engine]] як звичайний агент. Вимоги:

```
pip install google-cloud-aiplatform[agent_engines] a2a-sdk>=0.3.4
```

A2A-агент може взаємодіяти з іншими A2A-агентами на будь-якій платформі (не лише GCP).