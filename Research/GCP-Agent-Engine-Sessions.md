---
title: "GCP Agent Engine: Sessions"
tags:
  - gcp
  - ai-agents
  - vertex-ai
  - sessions
created: 2026-02-26
---

# GCP Agent Engine: Sessions

> Частина [[Research/GCP-Vertex-AI-Agent-Engine|Vertex AI Agent Engine]]

Офіційна документація: https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/sessions/overview

## Що це

Vertex AI Agent Engine Sessions зберігає історію взаємодій між користувачем і агентами. Це definitive source для контексту розмови і довгострокової пам'яті.

## Основні концепти

- **Session** — хронологічна послідовність повідомлень і дій (events) для однієї взаємодії між користувачем і агентом
- **Event** — зберігає контент розмови + дії агентів (function calls тощо)
- **State** — тимчасові дані, релевантні лише для поточної розмови
- **Memory** — персоналізована інформація, доступна між різними сесіями (див. [[Research/GCP-Agent-Engine-Memory-Bank|Memory Bank]])

## Функціональність

- Створення нових розмов (create session)
- Відновлення існуючих розмов (get session)
- Збереження прогресу (append events)
- Список розмов для користувача (list sessions)
- Видалення сесій (delete session)

## Два способи використання

1. **ADK (Agent Development Kit)** — сесії управляються автоматично після деплою агента
2. **API calls** — прямі виклики до Agent Engine Sessions API (без ADK)

## Зв'язок з іншими сервісами

- Sessions + [[Research/GCP-Agent-Engine-Memory-Bank|Memory Bank]] = повна система пам'яті агента
- Sessions дає short-term context, Memory Bank дає long-term personalization
- Sessions підтримує VPC-SC, CMEK, Data Residency, HIPAA