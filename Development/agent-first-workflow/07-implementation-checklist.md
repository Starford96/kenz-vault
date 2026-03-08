---
title: Покроковий чекліст імплементації
tags:
  - checklist
  - agent-first
  - workflow
created: 2026-02-26
---

# Покроковий чекліст імплементації

Практичний план впровадження agent-first workflow для MythicalGames. Розбитий на фази з оцінкою зусиль.

## Фаза 0: Foundation (1 день)

Мінімальна база яку можна зробити за один день.

### 0.1 Оновити claude-setup шаблон

- [ ] Оновити settings-init команду для нової структури
- [ ] Додати еталонний CLAUDE.md (table-of-contents pattern, ~100 рядків)
- [ ] Додати стандартний набір `.claude/rules/` (code-style, testing, error-handling)
- [ ] Додати docs/ scaffolding (architecture.md, decisions/, plans/, debt.md)
- [ ] Додати Makefile з targets: build, test, lint, arch-check, docs-check, ci

### 0.2 Створити еталонні конфіги

```
claude-setup/
├── templates/
│   ├── CLAUDE.md.tmpl
│   ├── .claude/
│   │   ├── settings.json
│   │   ├── rules/
│   │   │   ├── code-style.md
│   │   │   ├── testing.md
│   │   │   ├── error-handling.md
│   │   │   └── security.md
│   │   └── agents/
│   │       ├── code-reviewer.md
│   │       ├── test-writer.md
│   │       └── doc-gardener.md
│   ├── .golangci.yml
│   ├── .go-arch-lint.yml
│   ├── .markdownlint.yml
│   ├── .markdown-link-check.json
│   ├── Makefile
│   └── docs/
│       ├── architecture.md.tmpl
│       ├── debt.md.tmpl
│       ├── ecosystem.md.tmpl
│       └── decisions/
│           └── _template.md
```

### 0.3 Install script оновлення

```bash
#!/bin/bash
# scripts/bootstrap-project.sh
# Копіює шаблони в цільовий проєкт

PROJECT_DIR=${1:-.}

# Структура
mkdir -p "$PROJECT_DIR/.claude/rules"
mkdir -p "$PROJECT_DIR/.claude/agents"
mkdir -p "$PROJECT_DIR/docs/decisions"
mkdir -p "$PROJECT_DIR/docs/plans/active"
mkdir -p "$PROJECT_DIR/docs/plans/completed"

# Конфіги (тільки якщо не існують)
cp -n templates/.golangci.yml "$PROJECT_DIR/"
cp -n templates/.go-arch-lint.yml "$PROJECT_DIR/"
cp -n templates/.markdownlint.yml "$PROJECT_DIR/"
cp -n templates/Makefile "$PROJECT_DIR/"

# Claude config
cp -n templates/.claude/settings.json "$PROJECT_DIR/.claude/"
cp -rn templates/.claude/rules/ "$PROJECT_DIR/.claude/rules/"
cp -rn templates/.claude/agents/ "$PROJECT_DIR/.claude/agents/"

echo "Done. Now customize CLAUDE.md and docs/architecture.md for this project."
```

---

## Фаза 1: Пілотний проєкт (2-3 дні)

Вибрати **один** активний проєкт і повністю налаштувати. Рекомендація: `pf3-directory` (вже має найповніший setup).

### 1.1 CLAUDE.md рефакторинг
- [ ] Скоротити CLAUDE.md до ~100 рядків (table of contents)
- [ ] Виділити деталі в `.claude/rules/`
- [ ] Створити `docs/architecture.md` з опис шарів та залежностей
- [ ] Додати `docs/ecosystem.md` з cross-service залежностями

### 1.2 Architecture enforcement
- [ ] Встановити go-arch-lint: `go install github.com/fe3dback/go-arch-lint@latest`
- [ ] Створити `.go-arch-lint.yml` для структури проєкту
- [ ] Запустити `go-arch-lint check` — виправити порушення
- [ ] Додати `make arch-check` target

### 1.3 Documentation system
- [ ] Створити `docs/decisions/` з 3-5 ключовими ADR
- [ ] Додати `.markdownlint.yml`
- [ ] Додати `make docs-check` target
- [ ] Створити `docs/debt.md` з відомим техборгом

### 1.4 CI оновлення
- [ ] Додати arch-check job в CircleCI
- [ ] Додати docs-check job
- [ ] Переконатись що total CI < 5 хв

### 1.5 Agent configs
- [ ] Створити `.claude/agents/doc-gardener.md`
- [ ] Створити `.claude/agents/code-reviewer.md` (read-only)
- [ ] Оновити `.claude/settings.json` з hooks

### 1.6 Валідація
- [ ] Запустити Claude Code в проєкті
- [ ] Дати задачу "implement feature X" і подивитись як агент навігує
- [ ] Перевірити чи агент знаходить потрібну документацію
- [ ] Перевірити чи лінтери ловлять порушення і дають зрозумілі повідомлення

---

## Фаза 2: Розгортання на активні проєкти (1 тиждень)

Застосувати шаблон до всіх активно розроблюваних проєктів.

### 2.1 Пріоритетні проєкти (вже мають CLAUDE.md)
- [ ] pf3-directory — пілот з Фази 1
- [ ] pf3-studio-portal
- [ ] proof-of-human
- [ ] wallet-hook-receiver
- [ ] pf3-metadata-hub
- [ ] pf3-webhook-notifier

Для кожного:
1. Запустити `bootstrap-project.sh`
2. Адаптувати CLAUDE.md під проєкт
3. Створити `.go-arch-lint.yml` під структуру проєкту
4. Написати `docs/architecture.md`
5. Додати CI jobs

### 2.2 Автоматизація пропагації
- [ ] Створити скрипт що перевіряє які проєкти мають/не мають конфіги
- [ ] Alarm для проєктів де CLAUDE.md застарів (> 30 днів без змін)

---

## Фаза 3: Advanced capabilities (2-4 тижні)

### 3.1 Custom linters
- [ ] Написати перший custom linter (no-panic або layer-violation)
- [ ] Інтегрувати в golangci-lint або окремий `make custom-lint`
- [ ] Error messages містять fix instructions + docs link

### 3.2 Observability
- [ ] Structured logging (slog + JSON) в нових сервісах
- [ ] OpenTelemetry tracing для gRPC handlers
- [ ] docker-compose.observability.yml з Victoria Metrics/Logs
- [ ] Makefile targets: obs-up, obs-down

### 3.3 Doc-gardening automation
- [ ] doc-gardener агент з розкладом (щотижня)
- [ ] CI перевірка посилань в документації
- [ ] Автоматичне відкриття PR при знаходженні stale docs

### 3.4 ExecPlan workflow
- [ ] Шаблон ExecPlan для складних задач
- [ ] Скіл `/create-plan` що генерує ExecPlan
- [ ] `docs/plans/active/` для поточних планів

### 3.5 Cross-repo coordination
- [ ] `docs/ecosystem.md` в кожному сервісі
- [ ] Platform-level decisions в окремому репо або shared docs
- [ ] Спільний glossary для всієї платформи

---

## Фаза 4: Scale & Optimize (ongoing)

### 4.1 Метрики ефективності
- [ ] Відстежувати: PRs/day, CI pass rate, time-to-merge
- [ ] Порівнювати: до і після agent-first workflow
- [ ] Agent success rate по типах задач

### 4.2 Покриття проєктів
- [ ] Розширити на менш активні проєкти за потребою
- [ ] Стандартизувати CircleCI pipeline через shared-ci-config
- [ ] Template propagation автоматизація

### 4.3 Continuous improvement
- [ ] Ретроспектива: що працює, що ні
- [ ] Оновлення шаблонів на основі learnings
- [ ] Memory files з інсайтами агентів

---

## Quick Reference: Файли та інструменти

### Файли для створення/оновлення

| Файл | Що робить | Де шаблон |
|------|-----------|-----------|
| `CLAUDE.md` | Карта проєкту для агента | claude-setup/templates/ |
| `.claude/settings.json` | Дозволи, хуки, preferences | claude-setup/templates/ |
| `.claude/rules/*.md` | Модульні правила | claude-setup/templates/ |
| `.claude/agents/*.md` | Спеціалізовані агенти | claude-setup/templates/ |
| `.golangci.yml` | Linter конфігурація | claude-setup/templates/ |
| `.go-arch-lint.yml` | Architecture enforcement | claude-setup/templates/ |
| `.markdownlint.yml` | Doc linting | claude-setup/templates/ |
| `Makefile` | Стандартні targets | claude-setup/templates/ |
| `docs/architecture.md` | Архітектура проєкту | Унікальний per project |
| `docs/decisions/*.md` | Architecture Decision Records | Шаблон _template.md |
| `docs/ecosystem.md` | Cross-service залежності | Унікальний per project |
| `docs/debt.md` | Технічний борг | Шаблон |

### Інструменти для встановлення

| Інструмент | Команда | Призначення |
|-----------|---------|-------------|
| go-arch-lint | `go install github.com/fe3dback/go-arch-lint@latest` | Architecture enforcement |
| markdownlint-cli2 | `npm install -g markdownlint-cli2` | Doc linting |
| markdown-link-check | `npm install -g markdown-link-check` | Doc link validation |
| golangci-lint | Вже встановлено | Code linting |
| gofumpt | `go install mvdan.cc/gofumpt@latest` | Strict formatting |

### Claude Code конфігурація

| Конфіг | Файл | Scope |
|--------|------|-------|
| Global settings | `~/.claude/settings.json` | Всі проєкти |
| Project settings | `.claude/settings.json` | Один проєкт |
| Local overrides | `.claude/settings.local.json` | Особисті |
| Global skills | `~/.claude/skills/` | Всі проєкти |
| Project skills | `.claude/skills/` | Один проєкт |
| Global agents | `~/.claude/agents/` | Всі проєкти |
| Project agents | `.claude/agents/` | Один проєкт |
| Rules | `.claude/rules/` | Один проєкт |
| Memory | `~/.claude/projects/<proj>/memory/` | Per session |

---

## Принцип прогресу

> Start small. Validate. Expand.

Не намагайся впровадити все одразу. Фаза 0 + Фаза 1 дадуть 80% результату за 20% зусиль. Решта — інкрементальні покращення на основі реального досвіду.