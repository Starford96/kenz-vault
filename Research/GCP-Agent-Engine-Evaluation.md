---
title: "GCP Agent Engine: Evaluation"
tags:
  - gcp
  - ai-agents
  - vertex-ai
  - evaluation
  - testing
created: 2026-02-26
---

# GCP Agent Engine: Evaluation

> Частина [[Research/GCP-Vertex-AI-Agent-Engine|Vertex AI Agent Engine]]

Офіційна документація: https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/evaluate

## Що це

Gen AI Evaluation Service дозволяє оцінювати здатність агента виконувати задачі. Інтегрований з Agent Engine — можна тестувати задеплоєних агентів напряму.

## Workflow

### 1. Develop agent
Створити ADK-агента з моделлю, інструкціями та інструментами.

### 2. Deploy agent
Задеплоїти на Agent Engine Runtime (див. [[Research/GCP-Agent-Engine-Deploy-Scale|Деплой та масштабування]]).

### 3. Prepare evaluation dataset

```python
import pandas as pd
from vertexai import types

session_inputs = types.evals.SessionInput(user_id="user_123", state={})

agent_dataset = pd.DataFrame({
    "prompt": [
        "Search for 'noise-cancelling headphones'.",
        "Show me details for product 'B08H8H8H8H'.",
        "Add one pair to my cart.",
    ],
    "session_inputs": [session_inputs] * 3,
})
```

### 4. Run inference

```python
results = client.evals.run_inference(
    agent=agent_engine_resource_name,
    src=agent_dataset,
)
results.show()  # візуалізація
```

### 5. Run evaluation

```python
agent_info = types.evals.AgentInfo(...)
eval_run = client.evals.create_evaluation_run(
    agent_info=agent_info,
    dataset=results,
    ...
)
```

## Що оцінюється

- Здатність агента виконувати задачі (task completion)
- Проміжні events (intermediate_events) — function calls, reasoning steps
- Якість фінальних відповідей (responses)
- Коректність використання інструментів

## Зв'язок з іншими сервісами

- Використовує задеплоєних агентів з [[Research/GCP-Vertex-AI-Agent-Engine|Agent Engine]] Runtime
- Зберігає результати в GCS bucket
- Підтримує ADK, [[Research/GCP-Agent-Engine-Frameworks-Models|LangChain, LangGraph]] та інші [[Research/GCP-Agent-Engine-Frameworks-Models|фреймворки]]