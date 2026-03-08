---
title: Agent-First Development Workflow
tags:
  - agent-first
  - workflow
  - development
  - llm
created: 2026-02-26
---

# Agent-First Development Workflow

Гайд з побудови середовища розробки, де AI-агенти є основними виконавцями коду, а інженери — архітекторами середовища.

Базується на [Harness Engineering](https://openai.com/index/harness-engineering/) (OpenAI) та адаптований під Claude Code + Go мікросервісний стек MythicalGames.

## Центральна ідея

> "The era of writing code well is ending, and the era of designing environments well has begun."

Інженер більше не пише код — він проєктує **середовище**, в якому агент може працювати ефективно:
- Формулює інтент (промпт, план, специфікація)
- Будує feedback loops (лінтери, CI, тести, observability)
- Підтримує документацію як runtime dependency для агента

## Ключові принципи

| # | Принцип | Суть |
|---|---------|------|
| 1 | **Репозиторій = єдине джерело правди** | Все, що агент не бачить в контексті — не існує |
| 2 | **CLAUDE.md як карта, не енциклопедія** | ~100 рядків з покажчиками на docs/ |
| 3 | **Механічний enforcement** | Лінтери, structural tests, CI — не промпт-дисципліна |
| 4 | **Corrections are cheap, waiting is expensive** | Мінімум блокуючих gates, короткоживучі PR |
| 5 | **Boring tech перемагає** | Стабільні бібліотеки з хорошим training data coverage |
| 6 | **Observability для агентів** | Агент має бачити логи, метрики, трейси |
| 7 | **Doc-gardening** | Документація валідується автоматично |

## Розподіл ролей

### Людина (стратег/архітектор)
- Проєктує архітектуру та обмеження
- Формулює задачі як декларативний інтент
- Кодує "golden principles" в репозиторій
- Architectural gatekeeping
- Питання: "якої здатності не вистачає агенту?" (замість "спробуй знову")

### Агент (виконавець)
- Генерація коду за специфікацією
- Створення та виконання тестів
- PR management та ітерації
- Документація та її підтримка
- Code review (agent-to-agent)
- Автономні сесії до 6+ годин
- Очищення technical debt

## Поточний стан MythicalGames

**Що вже є:**
- CLAUDE.md шаблон високої якості (в ~8 проєктах)
- Multi-agent SDLC workflow (orchestrator → architect → developer → reviewer → tester)
- claude-setup репо з 8+ скілами та install.sh
- golangci-lint v2 (в ~12 проєктах)
- PostToolUse hook для auto-gofmt
- Conventional Commits формат

**Основні гепи:**
- ~75% проєктів без CLAUDE.md
- Немає cross-repo навігаційної карти
- Немає pre-commit hooks
- Немає doc-gardening автоматизації
- Немає observability для агентів
- Немає автоматичної пропагації шаблонів

## Документи в цій серії

| Документ | Зміст |
|----------|-------|
| [01-claude-md-structure.md](01-claude-md-structure.md) | CLAUDE.md як карта: структура, правила, приклади |
| [02-documentation-system.md](02-documentation-system.md) | Документація як runtime dependency, doc-gardening |
| [03-architecture-enforcement.md](03-architecture-enforcement.md) | Лінтери, structural tests, layering enforcement |
| [04-ci-cd-pipeline.md](04-ci-cd-pipeline.md) | CI/CD для agent-first: швидкість, merge philosophy |
| [05-observability.md](05-observability.md) | Observability stack для агентів |
| [06-repo-as-source-of-truth.md](06-repo-as-source-of-truth.md) | Репо як єдине джерело правди |
| [07-implementation-checklist.md](07-implementation-checklist.md) | Покроковий чекліст імплементації |

## Джерела

- [OpenAI: Harness Engineering](https://openai.com/index/harness-engineering/)
- [OpenAI Cookbook: ExecPlans](https://developers.openai.com/cookbook/articles/codex_exec_plans)
- [Martin Fowler / Birgitta Böckeler: Harness Engineering](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html)
- [matklad: ARCHITECTURE.md (2021)](https://matklad.github.io/2021/02/06/ARCHITECTURE.md.html)