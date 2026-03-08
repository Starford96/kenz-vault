---
title: "GCP Agent Engine: Code Execution"
tags:
  - gcp
  - ai-agents
  - vertex-ai
  - code-execution
  - sandbox
created: 2026-02-26
---

# GCP Agent Engine: Code Execution

> Частина [[Research/GCP-Vertex-AI-Agent-Engine|Vertex AI Agent Engine]]

Офіційна документація: https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/code-execution/overview

## Що це

Code Execution дозволяє агенту виконувати код у безпечному ізольованому managed sandbox. Ідеально для фінансових розрахунків, data science workflows, та інших задач де потрібно генерувати і виконувати код.

## Характеристики

- Sandbox створюється і виконує код **менше ніж за секунду**
- Підтримка файлового I/O до **100MB** (request або response)
- Sandbox зберігає стан (пам'ять) до **14 днів** (TTL налаштовується)
- Працює з **будь-яким фреймворком** і **будь-якою моделлю**
- **Не потребує деплою на Agent Engine** — може працювати локально або з будь-якого середовища

## Операції

- **Create sandbox** — створює ізольоване середовище для виконання коду. Обмежена файлова система, без мережевого доступу.
- **Get sandbox** — перевірка конфігурації і статусу sandbox
- **List sandboxes** — список всіх sandbox-ів у проєкті (з фільтрами)
- **Execute code** — надсилає код + input файли → отримує stdout, stderr, output файли. Зберігає стан між викликами.

## Безпека

- Повна ізоляція від хостової системи
- Без мережевого доступу
- Обмежена файлова система
- VPC-SC, CMEK, Data Residency, HIPAA

## Use Cases

- Фінансові калькуляції
- Data science workflows
- Генерація графіків і візуалізацій
- Аналіз даних
- Тестування коду