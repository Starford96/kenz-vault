---
title: Документація як Runtime Dependency
tags:
  - documentation
  - adr
  - agent-first
  - doc-gardening
created: 2026-02-26
---

# Документація як Runtime Dependency

## Центральна ідея

> "Docs aren't documentation anymore — they're runtime dependencies for agents."

Документація для агентів — це не довідковий матеріал, а **оперативна інформація**, від якої залежить якість виконання задачі. Якщо документ застарів — агент зробить помилку.

## Структура docs/

```
docs/
├── architecture.md              # Архітектура: шари, залежності, інваріанти
├── conventions.md               # Конвенції кодування (якщо не влазить в rules/)
├── decisions/                   # Architecture Decision Records
│   ├── _template.md             # Шаблон ADR
│   ├── 001-use-temporal.md
│   ├── 002-grpc-over-rest.md
│   └── 003-event-sourcing.md
├── plans/                       # Плани виконання
│   ├── active/                  # Поточні плани
│   │   └── migrate-to-v2.md
│   └── completed/               # Завершені (для контексту)
│       └── initial-setup.md
├── debt.md                      # Відомий технічний борг
├── runbooks/                    # Операційні runbooks
│   ├── deploy.md
│   └── incident-response.md
└── api/                         # API документація (якщо не proto)
    └── endpoints.md
```

## Architecture Decision Records (ADR)

### Навіщо

Кожне архітектурне рішення, яке не зафіксоване в репо, для агента **не існує**. Через 3 місяці воно не існуватиме й для нового члена команди.

### Шаблон

```markdown
# ADR-NNN: <Назва рішення>

**Статус:** Accepted | Deprecated | Superseded by ADR-XXX
**Дата:** YYYY-MM-DD

## Контекст

Яка проблема? Які обмеження?

## Рішення

Що вирішили і чому.

## Альтернативи

Що розглядали і чому відхилили.

## Наслідки

Що зміниться. Які trade-offs.
```

### Правила
- Один ADR = одне рішення
- ADR ніколи не видаляють, тільки позначають Deprecated/Superseded
- Агент **повинен** перевірити decisions/ перед пропозицією архітектурних змін

## ExecPlan (Плани виконання)

Для складних задач, що потребують багатогодинної автономної роботи агента.

### Обов'язкові секції

```markdown
# Plan: <Назва>

## Мета
Що отримає користувач після виконання. Як побачити що це працює.

## Контекст
Поточний стан. Припускаємо нульове попереднє знання.

## Кроки
1. Конкретний крок з вказівкою файлів і очікуваного результату
2. Кожен крок ідемпотентний (безпечно повторити)
3. Команди з очікуваним output'ом

## Валідація
Як перевірити що все працює:
- Навігація на <URL> повертає HTTP 200
- `go test ./...` проходить без помилок
- `make lint` без warnings

## Прогрес
- [x] Крок 1 (timestamp)
- [ ] Крок 2
- [ ] Крок 3

## Рішення (Decision Log)
- Обрали X замість Y тому що Z

## Сюрпризи
- Виявлено: <неочікувана поведінка> з доказами
```

### Принципи ExecPlan
- **Self-contained** — все необхідне в одному документі
- **Novice-accessible** — кожен спеціалізований термін пояснено
- **Demonstrable outcomes** — валідація через спостережувані результати
- **Idempotent steps** — безпечно повторити будь-який крок

## Doc-Gardening: Автоматична підтримка

### Що це

Фоновий процес, який періодично сканує документацію на предмет:
- Застарілих посилань на код (файли/функції що більше не існують)
- Протиріч з реальним кодом
- Зламаних markdown-посилань
- Документів що не оновлювались > N днів

### Реалізація через Claude Code

#### Варіант 1: Doc-gardener агент (.claude/agents/doc-gardener.md)

```markdown
---
name: doc-gardener
description: Scans and fixes stale documentation
tools: Read, Grep, Glob, Bash, Edit
model: sonnet
---

# Doc-Gardener

Ти — агент підтримки документації. Твоя задача:

1. Проскануй всі файли в docs/
2. Для кожного .md файлу:
   - Перевір чи існують файли/функції на які посилається документ
   - Перевір чи відповідає опис архітектури реальній структурі коду
   - Перевір чи зламані markdown-посилання
3. Для кожної знайденої проблеми:
   - Виправ якщо зміна очевидна
   - Додай TODO коментар якщо потрібне рішення людини
4. Виведи звіт по знайденим та виправленим проблемам
```

#### Варіант 2: CI перевірка (Makefile + скрипт)

```makefile
.PHONY: docs-check
docs-check:
	@echo "Checking documentation..."
	@markdownlint docs/ --config .markdownlint.yml
	@markdown-link-check docs/**/*.md --config .markdown-link-check.json
	@./scripts/check-doc-refs.sh
```

```bash
#!/bin/bash
# scripts/check-doc-refs.sh — Перевіряє що посилання на код в доках актуальні
set -euo pipefail
errors=0

# Знаходимо всі посилання на Go файли в документації
grep -rn '`[a-zA-Z_/]*\.go`' docs/ | while IFS= read -r match; do
    file=$(echo "$match" | grep -oP '`\K[^`]+\.go')
    if [ ! -f "$file" ] && [ ! -f "internal/$file" ]; then
        echo "WARNING: $(echo "$match" | cut -d: -f1-2) references '$file' which may not exist"
        ((errors++)) || true
    fi
done

if [ "$errors" -gt 0 ]; then
    echo "Found $errors potentially stale code references in docs"
    exit 1
fi
echo "All doc references look good"
```

### Валідація документації в CI

```yaml
# .markdownlint.yml
default: true
MD013:
  line_length: 120
  code_blocks: false
MD033: false              # Allow inline HTML
MD041: true               # First line = heading
```

```json
// .markdown-link-check.json
{
  "ignorePatterns": [
    { "pattern": "^https://internal" },
    { "pattern": "^#" }
  ],
  "aliveStatusCodes": [200, 206]
}
```

## Що робити зараз

1. **Створити docs/ структуру** — architecture.md, decisions/, plans/
2. **Перенести знання з Slack/Docs** — кожне рішення має бути ADR в репо
3. **Додати markdownlint + markdown-link-check** — в Makefile та CI
4. **Створити doc-gardener агент** — для періодичного сканування
5. **Зробити docs/ частиною PR review** — зміна коду = оновлення доків