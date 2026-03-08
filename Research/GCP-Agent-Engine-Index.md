---
title: GCP Agent Engine — Index
tags:
  - gcp
  - ai-agents
  - vertex-ai
  - cloud
  - research
created: 2026-02-26
---

# GCP Vertex AI Agent Engine

> Офіційна документація: https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview

## Що це таке

**Vertex AI Agent Engine** (раніше — Reasoning Engine) — це managed-платформа від Google Cloud для розгортання, управління та масштабування AI-агентів у продакшені. Не потрібно думати про інфраструктуру — все береться на себе Agent Engine.

Це частина ширшого продукту **Vertex AI Agent Builder**.

---

## Детальна документація

- [[Research/GCP-Agent-Engine-Frameworks-Models|Фреймворки та моделі]] — ADK, LangChain, LangGraph, AG2, LlamaIndex; Gemini, Claude, Llama
- [[Research/GCP-Agent-Engine-Deploy-Scale|Деплой та масштабування]] — 3 методи деплою, ресурсні ліміти, cold start optimization
- [[Research/GCP-Agent-Engine-Sessions|Sessions]] — управління розмовами та контекстом
- [[Research/GCP-Agent-Engine-Memory-Bank|Memory Bank]] — довгострокова пам'ять між сесіями
- [[Research/GCP-Agent-Engine-Code-Execution|Code Execution]] — виконання коду в ізольованому sandbox
- [[Research/GCP-Agent-Engine-A2A-Protocol|A2A Protocol]] — мультиагентна комунікація через відкритий протокол
- [[Research/GCP-Agent-Engine-Evaluation|Evaluation]] — оцінка якості агентів
- [[Research/GCP-Agent-Engine-Security|Безпека та Enterprise]] — VPC-SC, CMEK, DRZ, HIPAA, IAM, observability

---

## Коротко по сервісах

| Сервіс | Опис | Нотатка |
|--------|------|---------|
| **Runtime** | Запуск і масштабування агентів | [[Research/GCP-Agent-Engine-Deploy-Scale|Deploy]] |
| **Sessions** | Зберігання розмов | [[Research/GCP-Agent-Engine-Sessions|Sessions]] |
| **Memory Bank** | Довгострокова пам'ять | [[Research/GCP-Agent-Engine-Memory-Bank|Memory]] |
| **Code Execution** | Sandbox для коду | [[Research/GCP-Agent-Engine-Code-Execution|Code]] |
| **Example Store** (Preview) | Few-shot приклади | — |
| **Evaluation** (Preview) | Оцінка якості | [[Research/GCP-Agent-Engine-Evaluation|Eval]] |
| **Observability** | Trace, Monitoring, Logging | [[Research/GCP-Agent-Engine-Security|Security]] |

---

## Ціноутворення

- **Free tier** є для Agent Engine Runtime
- Детальне ціноутворення: https://cloud.google.com/vertex-ai/pricing#vertex-ai-agent-engine
- Runtime оплачується per instance/hour (CPU + Memory)
- Sessions, Memory Bank, Code Execution — окремі тарифи

---

## Корисні посилання

- Overview: https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview
- Deploy: https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/deploy
- ADK Docs: https://google.github.io/adk-docs/
- Agent Starter Pack: https://github.com/GoogleCloudPlatform/agent-starter-pack
- A2A Protocol: https://a2a-protocol.org/
- A2A Samples: https://github.com/a2aproject/a2a-samples