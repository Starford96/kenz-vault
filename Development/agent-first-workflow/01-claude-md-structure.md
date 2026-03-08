---
title: CLAUDE.md як карта, не енциклопедія
tags:
  - claude-md
  - agent-first
  - documentation
created: 2026-02-26
---

# CLAUDE.md як карта, не енциклопедія

## Проблема

Один великий CLAUDE.md файл не працює:
- **Контекст — дефіцитний ресурс.** Гігантський instruction file витісняє задачу, код і релевантні доки з контекстного вікна агента.
- **Миттєво застаріває.** Монолітний мануал стає "кладовищем застарілих правил" — агент не може відрізнити актуальне від неактуального.
- **Люди перестають підтримувати.** Файл стає "attractive nuisance."

## Рішення: Table of Contents Pattern

### Короткий CLAUDE.md (~100 рядків)

CLAUDE.md — це **зміст**, що вказує на структуровану `docs/` директорію. Він дає агенту:
1. Що це за проєкт (2-3 речення)
2. Як побудувати, запустити, протестувати (таблиця команд)
3. Де шукати глибший контекст (покажчики на docs/)
4. Ключові конвенції (стисло)

### Структура файлів

```
project/
├── CLAUDE.md                    # ~100 рядків — карта
├── .claude/
│   ├── settings.json            # Дозволи, хуки
│   ├── rules/                   # Модульні правила (auto-discovered)
│   │   ├── code-style.md        # Go стиль
│   │   ├── testing.md           # Тестові патерни
│   │   ├── security.md          # Безпека
│   │   └── project-ops.md       # Proto/build операції
│   ├── agents/                  # Спеціалізовані агенти
│   │   ├── code-reviewer.md
│   │   ├── test-writer.md
│   │   └── doc-gardener.md
│   └── skills/                  # Скіли (capabilities)
│       └── commit/SKILL.md
├── docs/
│   ├── architecture.md          # Архітектура: шари, залежності
│   ├── decisions/               # ADR (Architecture Decision Records)
│   │   ├── 001-use-temporal.md
│   │   └── 002-grpc-over-rest.md
│   ├── plans/                   # Активні та завершені плани
│   │   ├── active/
│   │   └── completed/
│   └── debt.md                  # Відомий технічний борг
```

## Шаблон CLAUDE.md

```markdown
# <Project Name>

<1-2 речення: що це за сервіс, основна відповідальність>

## Команди

| Дія | Команда |
|-----|---------|
| Build | `make build` |
| Test (all) | `make test` |
| Test (one) | `go test -run TestName ./path/to/pkg/...` |
| Lint | `make lint` |
| Generate | `make generate` |
| Run | `make run` |

## Архітектура

Детальна архітектура: [docs/architecture.md](docs/architecture.md)

### Шари (dependency flow →)
Types → Config → Repo → Service → Runtime → Transport

### Ключові директорії
- `internal/types/` — доменні типи, інтерфейси
- `internal/config/` — конфігурація
- `internal/repo/` — data access
- `internal/service/` — бізнес-логіка
- `internal/runtime/` — app lifecycle, DI
- `internal/transport/` — gRPC/HTTP handlers

## Конвенції

- Go: early returns, error wrapping з контекстом, маленькі функції
- Коміти: `<type>[scope]: description` (Conventional Commits)
- Тести: table-driven, мінімум моків, тест на файл `_test.go`

## Документація

- Рішення: [docs/decisions/](docs/decisions/)
- Технічний борг: [docs/debt.md](docs/debt.md)
- Плани: [docs/plans/](docs/plans/)
```

## Ієрархія CLAUDE.md в Claude Code

Claude Code автоматично завантажує CLAUDE.md в правильному порядку:

| Рівень | Шлях | Shared? | Призначення |
|--------|------|---------|-------------|
| User | `~/.claude/CLAUDE.md` | Ні | Глобальні преференції |
| Project | `./CLAUDE.md` | Так (git) | Проєктний контекст |
| Project Local | `./CLAUDE.local.md` | Ні (.gitignore) | Локальні override'и |
| Rules | `./.claude/rules/*.md` | Так (git) | Модульні правила |
| Memory | `~/.claude/projects/<proj>/memory/MEMORY.md` | Ні | Накопичені знання |

### Модульні правила (.claude/rules/)

Правила автоматично виявляються рекурсивно. Підтримують glob-based scope:

```markdown
---
paths: ["internal/service/**/*.go"]
---
# Service Layer Rules

- Сервіси приймають інтерфейси, повертають конкретні типи
- Кожен метод сервісу обгортає помилки з контекстом
- Не імпортувати transport layer напряму
```

### Імпорт через @-синтаксис

CLAUDE.md може імпортувати інші файли:

```markdown
# Project X

@docs/architecture.md
@docs/conventions.md
```

Максимальна глибина вкладеності: 5 рівнів.

## Progressive Disclosure

Агент починає з малого стабільного entry point і завантажує глибший контекст тільки коли потрібно:

1. **Завжди в контексті:** CLAUDE.md (~100 рядків) + .claude/rules/
2. **Завантажується при потребі:** docs/ файли (коли агент читає відповідну директорію)
3. **Child CLAUDE.md:** файли CLAUDE.md в піддиректоріях завантажуються on-demand коли агент працює з тими файлами

## ARCHITECTURE.md (додатково)

Базується на [matklad's ARCHITECTURE.md](https://matklad.github.io/2021/02/06/ARCHITECTURE.md.html):
- Короткий bird's-eye view проєкту
- Тільки рідко-змінювана інформація
- Архітектурні інваріанти часто формулюються як "щось НЕ існує"
- Наприклад: "Handlers ніколи не звертаються до БД напряму"

## Що робити зараз

1. **Для проєктів з CLAUDE.md** — скоротити до ~100 рядків, виділити деталі в docs/
2. **Для проєктів без CLAUDE.md** — створити за шаблоном вище
3. **Додати .claude/rules/** — виділити стильові та архітектурні правила
4. **Створити docs/architecture.md** — опис шарів та залежностей
5. **Стандартизувати через settings-init** — оновити команду для нового формату