---
title: "GCP Agent Engine: Безпека та Enterprise"
tags:
  - gcp
  - ai-agents
  - vertex-ai
  - security
  - enterprise
created: 2026-02-26
---

# GCP Agent Engine: Безпека та Enterprise

> Частина [[Research/GCP-Vertex-AI-Agent-Engine|Vertex AI Agent Engine]]

## Enterprise Security Features

### VPC Service Controls
- Посилює захист даних, мінімізує ризики data exfiltration
- Блокує **весь публічний інтернет-доступ**
- Агент зберігає доступ до Google APIs (BigQuery, Cloud SQL, Vertex AI)
- Підтримується: Runtime, [[Research/GCP-Agent-Engine-Sessions|Sessions]], [[Research/GCP-Agent-Engine-Memory-Bank|Memory Bank]], [[Research/GCP-Agent-Engine-Code-Execution|Code Execution]]

### Private Service Connect Interface (PSC-I)
- Дозволяє агентам взаємодіяти з приватними сервісами у VPC
- Налаштовується через network_attachment при деплої
- Можна використовувати shared або dedicated network attachments

### Customer-Managed Encryption Keys (CMEK)
- Власні ключі шифрування для даних at rest через Cloud KMS
- Підтримується: Runtime, Sessions, Memory Bank, Code Execution

### Data Residency (DRZ)
- Гарантує що дані at rest зберігаються в обраному регіоні
- Підтримується: Runtime, Sessions, Memory Bank, Code Execution

### HIPAA
- Повна HIPAA compliance для всіх сервісів Agent Engine

### Access Transparency
- Логи дій персоналу Google при доступі до контенту
- Підтримується: Runtime, Sessions, Memory Bank

### Agent Identity (Preview)
- Кожен агент може мати унікальну IAM identity
- Прив'язана до resource ID агента, незалежна від фреймворку
- Дозволяє гранулярний контроль доступу per agent

### Custom Service Account
- Замість default identity можна задати custom service account
- Окремі permissions для кожного агента

## Threat Detection

Agent Engine Threat Detection (Preview) — вбудований сервіс Security Command Center:
- Виявляє потенційні атаки на задеплоєних агентів
- Автоматичний моніторинг без додаткового налаштування

## Матриця підтримки security features

| Feature | Runtime | Sessions | Memory Bank | Code Execution | Example Store |
|---------|---------|----------|-------------|----------------|---------------|
| VPC-SC | ✅ | ✅ | ✅ | ✅ | ❌ |
| CMEK | ✅ | ✅ | ✅ | ✅ | ❌ |
| DRZ | ✅ | ✅ | ✅ | ✅ | ❌ |
| HIPAA | ✅ | ✅ | ✅ | ✅ | ✅ |
| Access Transparency | ✅ | ✅ | ✅ | ❌ | ❌ |

## Secrets Management

Environment variables можуть посилатися на Secret Manager:

```python
env_vars = {
    "DB_PASSWORD": {"secret": "my-secret-id", "version": "1"},
}
```

Service Agent автоматично отримує секрети при деплої.

## Observability

- **Cloud Trace** (OpenTelemetry) — distributed tracing
- **Cloud Monitoring** — метрики (latency, error rate, token usage)
- **Cloud Logging** — логи агента

Метрики доступні через Console:
- Overview: latency, request count, error rate
- Models: model calls, model error rate, token usage
- Tools: tool calls, tool error rate, tool latency
- Usage: token usage, CPU/memory allocation