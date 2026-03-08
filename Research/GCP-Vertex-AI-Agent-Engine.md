---
title: GCP Vertex AI Agent Engine — Ресерч
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

## Що вміє Agent Engine

| Сервіс | Опис |
|--------|------|
| **Runtime** | Запуск і масштабування агентів у managed середовищі |
| **Sessions** | Зберігання розмов між користувачем і агентом |
| **Memory Bank** | Довгострокова пам'ять між сесіями |
| **Code Execution** | Виконання коду в ізольованому sandbox |
| **Example Store** _(Preview)_ | Зберігання few-shot прикладів для покращення агента |
| **Evaluation** _(Preview)_ | Оцінка якості агента з Gen AI Evaluation Service |
| **Observability** | Cloud Trace, Cloud Monitoring, Cloud Logging (OpenTelemetry) |

---

## Підтримувані фреймворки

| Рівень підтримки | Фреймворки |
|------------------|-----------|
| **Full integration** | ADK (Agent Development Kit), LangChain, LangGraph |
| **Vertex AI SDK integration** | AG2, LlamaIndex (Query Pipeline) |
| **Custom template** | CrewAI, будь-який власний фреймворк |

---

## Які моделі можна підключати

Agent Engine сам по собі **не прив'язаний до конкретних моделей** — він запускає Python-код агента, який сам звертається до моделей через Vertex AI API.

Офіційно в документації (у контексті ADK) використовується:
-  (як приклад за замовчуванням)
- Будь-яка модель з **Vertex AI Model Garden** (Gemini 2.0, 1.5 Pro/Flash, Claude через Anthropic vertex endpoint, тощо)
- Зовнішні моделі — через власний API у коді агента

> Модель визначається в коді агента, а не в конфігурації Agent Engine.

---

## Як деплоїти агента

### 3 способи деплою

1. **З об'єкта агента** () — ідеально для Colab/інтерактивної розробки
2. **З вихідних файлів** — підходить для CI/CD, Terraform, IaC
3. **З Developer Connect** (Git) — для проєктів у GitHub/GitLab, з підтримкою version control

### Базовий приклад (Python SDK)



### Приклад ADK-агента (мінімум)



---

## Управління задеплоєними агентами

### Список агентів (REST API)
<!DOCTYPE html>
<html lang=en>
  <meta charset=utf-8>
  <meta name=viewport content="initial-scale=1, minimum-scale=1, width=device-width">
  <title>Error 404 (Not Found)!!1</title>
  <style>
    *{margin:0;padding:0}html,code{font:15px/22px arial,sans-serif}html{background:#fff;color:#222;padding:15px}body{margin:7% auto 0;max-width:390px;min-height:180px;padding:30px 0 15px}* > body{background:url(//www.google.com/images/errors/robot.png) 100% 5px no-repeat;padding-right:205px}p{margin:11px 0 22px;overflow:hidden}ins{color:#777;text-decoration:none}a img{border:0}@media screen and (max-width:772px){body{background:none;margin-top:0;max-width:none;padding-right:0}}#logo{background:url(//www.google.com/images/branding/googlelogo/1x/googlelogo_color_150x54dp.png) no-repeat;margin-left:-5px}@media only screen and (min-resolution:192dpi){#logo{background:url(//www.google.com/images/branding/googlelogo/2x/googlelogo_color_150x54dp.png) no-repeat 0% 0%/100% 100%;-moz-border-image:url(//www.google.com/images/branding/googlelogo/2x/googlelogo_color_150x54dp.png) 0}}@media only screen and (-webkit-min-device-pixel-ratio:2){#logo{background:url(//www.google.com/images/branding/googlelogo/2x/googlelogo_color_150x54dp.png) no-repeat;-webkit-background-size:100% 100%}}#logo{display:inline-block;height:54px;width:150px}
  </style>
  <a href=//www.google.com/><span id=logo aria-label=Google></span></a>
  <p><b>404.</b> <ins>That’s an error.</ins>
  <p>The requested URL <code>/v1/projects/PROJECT_ID/locations/LOCATION/reasoningEngines</code> was not found on this server.  <ins>That’s all we know.</ins>

### Python SDK


### REST ідентифікатор ресурсу


---

## Ресурсні ліміти

| Параметр | Діапазон | За замовчуванням |
|----------|---------|-----------------|
|  | 0–10 | 1 |
|  | 1–1000 (100 з VPC-SC) | 100 |
|  | 1, 2, 4, 6, 8 | 4 |
|  | 1Gi–32Gi | 4Gi |
|  | — | 9 |

---

## Безпека

- **VPC Service Controls** — повна ізоляція від інтернету
- **Private Service Connect (PSC-I)** — доступ до приватних сервісів у VPC
- **Customer-Managed Encryption Keys (CMEK)** — власне шифрування at rest
- **Data Residency (DRZ)** — дані залишаються в обраному регіоні
- **HIPAA compliant**
- **IAM + Service Account** — кожен агент може мати окрему ідентичність

---

## Сесії та пам'ять (ADK)



---

## Agent Starter Pack

Готові production-ready шаблони від Google:
- https://github.com/GoogleCloudPlatform/agent-starter-pack
- Включає: ReAct, RAG, multi-agent шаблони
- Terraform для інфраструктури
- CI/CD через Cloud Build
- Вбудована observability

---

## Ціноутворення

- **Free tier** є для Agent Engine Runtime
- Детальне ціноутворення: https://cloud.google.com/vertex-ai/pricing#vertex-ai-agent-engine

---

## Корисні посилання

- [Overview](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview)
- [Deploy an agent](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/deploy)
- [ADK development](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/develop/adk)
- [Manage deployed agents](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/manage/overview)
- [Agent Starter Pack (GitHub)](https://github.com/GoogleCloudPlatform/agent-starter-pack)
- [ADK Docs](https://google.github.io/adk-docs/)
- [A2A Protocol](https://a2a-protocol.org/)