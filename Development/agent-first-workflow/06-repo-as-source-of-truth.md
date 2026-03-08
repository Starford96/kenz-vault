---
title: Репозиторій як єдине джерело правди
tags:
  - documentation
  - adr
  - workflow
  - agent-first
created: 2026-02-26
---

# Репозиторій як єдине джерело правди

## Центральна ідея

> Все, що агент не бачить в контексті — для нього не існує. Як і для нового працівника через 3 місяці.

Обговорення в Slack, рішення в Google Docs, домовленості на мітингах — якщо це не в репо, агент про це не знає. І не зможе дізнатись.

## Що має бути в репо

| Категорія | Де зберігати | Формат |
|-----------|-------------|--------|
| Архітектурні рішення | `docs/decisions/NNN-*.md` | ADR (Architecture Decision Record) |
| Конвенції кодування | `.claude/rules/code-style.md` | Markdown з прикладами |
| API контракти | `proto/` або `api/` | Protobuf / OpenAPI |
| Deployment конфігурація | `deploy/` або `k8s/` | Kustomize / Helm |
| Тестові стратегії | `.claude/rules/testing.md` | Markdown |
| Runbooks | `docs/runbooks/` | Markdown |
| Відомий техборг | `docs/debt.md` | Markdown з пріоритетами |
| Плани виконання | `docs/plans/active/` | ExecPlan формат |
| Секрети (посилання) | `docs/runbooks/secrets.md` | Де знайти, не самі значення |

## Що НЕ має бути тільки в Slack/Docs

### Anti-patterns

**Anti-pattern 1: "Ми домовились в Slack"**
```
# ПОГАНО — рішення в Slack
@team: давайте використовувати UUID v7 замість v4 для всіх нових entity IDs
👍 5 реакцій
```

```markdown
# ДОБРЕ — рішення в ADR
# docs/decisions/015-uuid-v7.md

# ADR-015: UUID v7 для entity IDs

**Статус:** Accepted
**Дата:** 2026-02-20

## Контекст
UUID v4 не дає природного ordering. Це ускладнює пагінацію та індексацію.

## Рішення
Використовуємо UUID v7 (time-sortable) для всіх нових entity IDs.
Існуючі UUID v4 не мігруємо.

## Наслідки
- Нові таблиці використовують UUID v7
- Ordering за ID == ordering за часом створення
- go.uuid.NewV7() замість go.uuid.New()
```

**Anti-pattern 2: "Є Google Doc"**
```
# ПОГАНО
"Архітектура описана в цьому Google Doc: <link>"
```

```markdown
# ДОБРЕ
# docs/architecture.md — версіонований, в репо, доступний агенту
```

**Anti-pattern 3: "Запитай Васю"**
```
# ПОГАНО
"Як працює автентифікація? Запитай Васю, він це писав"
```

```markdown
# ДОБРЕ
# docs/architecture.md#authentication
## Authentication Flow
1. Client sends JWT token in Authorization header
2. Gateway validates token via Auth service gRPC call
3. Auth service checks token signature + expiry
4. User context propagated via gRPC metadata
```

## Cross-Repo навігація

### Проблема

MythicalGames має 31+ мікросервіс. Агент працює в одному репо і не знає про інші.

### Рішення: Ecosystem Map

```markdown
# docs/ecosystem.md (в кожному репо)

## Залежності цього сервісу

| Сервіс | Репо | Протокол | Що надає |
|--------|------|----------|----------|
| pf3-directory | github.com/MythicalGames/pf3-directory | gRPC | User profiles, auth |
| pf3-catalog | github.com/MythicalGames/pf3-catalog | gRPC | Item definitions |
| pf3-inventory | github.com/MythicalGames/pf3-inventory | gRPC | User item ownership |

## Хто залежить від цього сервісу

| Сервіс | Репо | Що використовує |
|--------|------|-----------------|
| pf3-web-gtw | github.com/MythicalGames/pf3-web-gtw | REST → gRPC proxy |
| pf3-studio-portal | github.com/MythicalGames/pf3-studio-portal | Admin UI |

## Proto контракти

Shared protos: `github.com/MythicalGames/pf3-contract`
```

### Рішення: Monorepository docs (опціонально)

Централізований репо з описом всієї системи:

```
platform-docs/
├── services/
│   ├── pf3-directory.md
│   ├── pf3-catalog.md
│   └── ...
├── architecture/
│   ├── overview.md          # System-level architecture
│   ├── data-flow.md         # Як дані течуть між сервісами
│   └── auth-flow.md         # Автентифікація cross-service
└── decisions/
    ├── platform-001-grpc.md # Platform-level ADRs
    └── ...
```

## Процес: Slack → Repo Pipeline

### Правило

Кожне технічне рішення з Slack має бути задокументоване в репо протягом 24 годин.

### Workflow

1. Рішення приймається в Slack (обговорення, голосування)
2. Відповідальний створює ADR або оновлює docs/
3. PR з документацією (може бути створений агентом)
4. Лінк на PR поститься назад в Slack тред

### Автоматизація через Claude Code

Скіл який створює ADR:

```markdown
# .claude/skills/create-adr/SKILL.md
---
name: create-adr
description: Create an Architecture Decision Record
---

Створи ADR файл в docs/decisions/:

1. Знайди найвищий номер ADR в docs/decisions/
2. Створи файл з наступним номером: docs/decisions/NNN-<slug>.md
3. Використай шаблон:
   - Status: Accepted
   - Date: сьогоднішня дата
   - Context: $ARGUMENTS
   - Decision: (потрібно заповнити)
   - Alternatives: (потрібно заповнити)
   - Consequences: (потрібно заповнити)

Запитай у користувача деталі рішення та альтернативи.
```

## .claude/rules/ як живі конвенції

### Чому rules/ а не довгий CLAUDE.md

- **Auto-discovered** — Claude Code знаходить їх автоматично
- **Scoped** — можна обмежити правила до конкретних шляхів
- **Модульні** — легко додати/видалити/оновити одне правило
- **Reviewable** — кожне правило в окремому PR

### Приклади

```markdown
# .claude/rules/error-handling.md
---
paths: ["internal/**/*.go"]
---
# Error Handling

- Завжди обгортай помилки контекстом: `fmt.Errorf("operation: %w", err)`
- Ніколи не ігноруй помилки (навіть defer Close())
- Кастомні error types тільки коли потрібен type assertion в caller'і
- Для sentinel errors: `var ErrNotFound = errors.New("not found")`
```

```markdown
# .claude/rules/proto.md
---
paths: ["proto/**/*.proto", "internal/transport/**/*.go"]
---
# Protocol Buffers

- Proto файли в proto/ директорії
- Генерація: `make generate`
- Не редагуй згенерований код (*.pb.go)
- Handler'и в transport/ приймають proto types, конвертують в domain types
```

## Що робити зараз

### Крок 1: Аудит знань (2 год)
1. Пройтись по останнім 100 повідомленням в tech-каналах Slack
2. Виписати всі технічні рішення
3. Для кожного: чи є це в репо? Якщо ні — створити ADR

### Крок 2: docs/decisions/ (1 год)
1. Створити директорію та шаблон
2. Задокументувати ключові рішення
3. Додати create-adr скіл

### Крок 3: Ecosystem map (1 год)
1. Створити docs/ecosystem.md в кожному активному сервісі
2. Описати залежності та хто залежить від цього сервісу
3. Посилання на proto контракти

### Крок 4: Процес (ongoing)
1. Впровадити "Slack → ADR" правило
2. Кожен PR що змінює архітектуру повинен оновити docs/
3. doc-gardener агент перевіряє актуальність