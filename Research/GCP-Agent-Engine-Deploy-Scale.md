---
title: "GCP Agent Engine: Деплой та масштабування"
tags:
  - gcp
  - ai-agents
  - vertex-ai
  - deployment
  - scaling
created: 2026-02-26
---

# GCP Agent Engine: Деплой та масштабування

> Частина [[Research/GCP-Vertex-AI-Agent-Engine|Vertex AI Agent Engine]]

Документація деплою: https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/deploy
Оптимізація: https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/optimize-runtime

## Три методи деплою

### 1. З об'єкта агента
Найпростіший — передаєш Python-об'єкт агента. Ідеально для Colab/інтерактивної розробки.

```python
remote_agent = client.agent_engines.create(
    agent=local_agent,
    config={
        "requirements": ["google-cloud-aiplatform[agent_engines,adk]"],
        "display_name": "My Agent",
        "min_instances": 1,
        "max_instances": 10,
    },
)
```

### 2. З вихідних файлів (source files)
Для CI/CD, Terraform, IaC. Не потрібен Cloud Storage bucket.

```python
remote_agent = client.agent_engines.create(
    config={
        "source_packages": ["agent_directory"],
        "entrypoint_module": "agent_directory.agent",
        "entrypoint_object": "root_agent",
        "class_methods": [{"name": "ask", "api_mode": ""}],
    },
)
```

Обмеження: source_packages не має перевищувати **8MB**.

### 3. З Developer Connect (Git)
Найкращий для командної роботи — деплой напряму з GitHub/GitLab через Developer Connect.

```python
remote_agent = client.agent_engines.create(
    config={
        "developer_connect_source": {
            "git_repository_link": "projects/.../gitRepositoryLinks/REPO_ID",
            "revision": "main",
            "dir": "path/to/dir",
        },
        "entrypoint_module": "agent",
        "entrypoint_object": "root_agent",
    },
)
```

## Ресурсні ліміти

| Параметр | Діапазон | Default |
|----------|---------|---------|
| min_instances | 0–10 | 1 |
| max_instances | 1–1000 (100 з VPC-SC) | 100 |
| CPU | 1, 2, 4, 6, 8 | 4 |
| Memory | 1Gi–32Gi | 4Gi |
| container_concurrency | — | 9 |

Рекомендований concurrency: `2 * cpu + 1`.

## Оптимізація cold starts

**Проблема:** при першому запиті (або після простою) нові інстанси запускаються, що додає ~4.7с латенсі.

**Рішення:**
1. Збільшити `min_instances` до рівня базового трафіку (напр. 10) → зменшує cold start до ~1.4с
2. Стабільне навантаження через queue — тримає сервіс "теплим"

## Оптимізація async workers

**Проблема:** за замовчуванням concurrency=9, що означає 1 запит на процес. Для async агентів (ADK) це неефективно — max latency може сягати 60с.

**Рішення:** збільшити `container_concurrency` до кратного 9 (напр. 36) для async агентів.
- З concurrency=9: max latency ~60с
- З concurrency=36: max latency ~7с

**Увага:** надто високий concurrency може спричинити OOM.

## Build Options

Можна кастомізувати container image при збірці:
- Встановлення системних залежностей (Node.js, gcloud CLI, uv/uvx)
- Installation scripts виконуються з root правами
- Корисно для MCP tools, npx тощо

## Agent Starter Pack

Готові production-ready шаблони: https://github.com/GoogleCloudPlatform/agent-starter-pack
- ReAct, RAG, multi-agent templates
- Terraform для інфраструктури
- CI/CD через Cloud Build
- Вбудована observability