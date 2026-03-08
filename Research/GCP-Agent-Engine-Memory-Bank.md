---
title: "GCP Agent Engine: Memory Bank"
tags:
  - gcp
  - ai-agents
  - vertex-ai
  - memory
created: 2026-02-26
---

# GCP Agent Engine: Memory Bank

> Частина [[Research/GCP-Vertex-AI-Agent-Engine|Vertex AI Agent Engine]]

Офіційна документація: https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/memory-bank/overview

## Що це

Memory Bank дозволяє динамічно генерувати довгострокові спогади на основі розмов користувачів з агентом. Спогади — це персоналізована інформація, доступна між різними [[Research/GCP-Agent-Engine-Sessions|сесіями]].

## Як працює

Memory Bank зберігає ізольовану колекцію "memories" для кожного scope (agent + user). Кожен memory — це незалежний факт:

```json
{
  "scope": {"agent_name": "My agent", "user": "user-123"},
  "fact": "Користувач віддає перевагу темній темі."
}
```

## Ключові можливості

### Генерація пам'яті (LLM-driven)
- **Memory Extraction** — LLM витягує найважливішу інформацію з розмов
- **Memory Consolidation** — нова інформація об'єднується з існуючими спогадами (спогади еволюціонують)
- **Async Generation** — генерація у фоновому режимі, агент не чекає
- **Customizable Extraction** — можна налаштувати які теми (topics) вважати важливими + few-shot examples
- **Multimodal Understanding** — обробляє мультимодальний контент

### Зберігання та пошук
- **Managed Storage** — повністю керований persistent store
- **Data Isolation** — спогади ізольовані per identity (scope)
- **Similarity Search** — семантичний пошук по спогадах, scoped до конкретної identity
- **Auto Expiration (TTL)** — можна задати TTL, старі спогади автоматично видаляються
- **Memory Revisions** — автоматичний трекінг як спогади змінюються з часом

### Безпека
- IAM conditions для обмеження доступу до конкретних scopes
- VPC-SC, CMEK, Data Residency, HIPAA

## Основний workflow

1. **CreateSession** — нова розмова, прив'язана до user_id
2. **AppendEvent** — зберігаємо events під час розмови
3. **GenerateMemories** — витягуємо факти з розмови (або CreateMemory для ручного запису)
4. **RetrieveMemories** — отримуємо релевантні спогади для нової розмови

## ADK Integration

В ADK використовується PreloadMemoryTool:

```python
from google.adk.tools.preload_memory_tool import PreloadMemoryTool

agent = Agent(
    model="gemini-2.0-flash",
    name="stateful_agent",
    tools=[PreloadMemoryTool()],
)
```

PreloadMemoryTool автоматично підтягує релевантні спогади на початку кожного turn.

## Use Cases

- **Long-Term Personalization** — агент пам'ятає вподобання користувача між сесіями
- **LLM-driven Knowledge Extraction** — автоматичне витягування ключових фактів
- **Dynamic Context** — на відміну від RAG (статична база знань), Memory Bank еволюціонує на основі взаємодій

## Безпека: Prompt Injection та Memory Poisoning

Memory Bank вразливий до memory poisoning — збереження фальшивої інформації. Заходи:
- Model Armor для інспекції промптів
- Adversarial testing (red teaming)
- Sandbox execution для критичних дій

Детальніше: https://research.google/pubs/an-introduction-to-googles-approach-for-secure-ai-agents/