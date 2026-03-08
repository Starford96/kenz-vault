---
title: CI/CD для Agent-First Workflow
tags:
  - ci-cd
  - agent-first
  - workflow
  - go
created: 2026-02-26
---

# CI/CD для Agent-First Workflow

## Центральна ідея

> "Corrections are cheap, waiting is expensive."

В agent-first середовищі throughput — головна метрика. Агент може генерувати 3.5+ PR на день. CI повинен давати feedback за хвилини, не за десятки хвилин.

## Merge Philosophy

### Принципи

| Принцип | Опис |
|---------|------|
| **Мінімум блокуючих gates** | Тільки критичні перевірки блокують merge |
| **Короткоживучі PR** | Більшість reviewable за < 1 хвилину |
| **Flaky tests → follow-up** | Flaky тест фіксять follow-up ранами, не блокують прогрес |
| **Agent-to-agent review** | Nitpicking робить агент-рев'юер, людина — architectural decisions |
| **Auto-merge для рефакторингу** | Drift-correction PR мержаться автоматично якщо CI зелений |

### Коли людина рев'юїть

- Архітектурні зміни (нові шари, нові залежності)
- Security-sensitive код
- Public API зміни
- Зміни в CI/CD pipeline

### Коли агент рев'юїть

- Стилістичні зміни
- Bug fixes з тестами
- Рефакторинг без зміни поведінки
- Документація

## CI Pipeline: Цільова архітектура

### Вимоги до швидкості

| Стадія | Цільовий час | Блокує merge? |
|--------|-------------|---------------|
| Lint | < 2 хв | Так |
| Architecture check | < 30с | Так |
| Unit tests | < 5 хв | Так |
| Doc validation | < 30с | Ні (warning) |
| Integration tests | < 10 хв | Так (для main) |
| **Total (parallel)** | **< 5 хв** | |

### GitHub Actions приклад

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
  push:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true        # Відміняє старі runs для того ж PR

jobs:
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true
      - uses: golangci/golangci-lint-action@v6
        with:
          version: v2.1
          args: --timeout=3m

  arch:
    runs-on: ubuntu-latest
    timeout-minutes: 2
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true
      - name: Architecture check
        run: |
          go install github.com/fe3dback/go-arch-lint@latest
          go-arch-lint check --project-path .

  test:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true
      - name: Unit tests
        run: go test -race -count=1 -timeout=5m ./...
      - name: Architecture tests
        run: go test -run TestArch -timeout=2m ./...

  docs:
    runs-on: ubuntu-latest
    timeout-minutes: 2
    steps:
      - uses: actions/checkout@v4
      - uses: DavidAnson/markdownlint-cli2-action@v19
        with:
          globs: "docs/**/*.md"
      - name: Check doc links
        uses: gaurav-nelson/github-action-markdown-link-check@v1
        with:
          folder-path: docs/

  # Merge gate — всі обов'язкові checks пройшли
  ready:
    needs: [lint, arch, test]
    runs-on: ubuntu-latest
    steps:
      - run: echo "All required checks passed"
```

### CircleCI адаптація (для поточного стеку)

Оскільки MythicalGames використовує CircleCI, аналогічна структура:

```yaml
# .circleci/config.yml
version: 2.1

executors:
  go:
    docker:
      - image: cimg/go:1.23
    resource_class: medium

jobs:
  lint:
    executor: go
    steps:
      - checkout
      - restore_cache:
          keys: [go-mod-{{ checksum "go.sum" }}]
      - run:
          name: Lint
          command: |
            curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/HEAD/install.sh | sh -s -- -b $(go env GOPATH)/bin v2.1.0
            golangci-lint run --timeout=3m
      - save_cache:
          key: go-mod-{{ checksum "go.sum" }}
          paths: [~/go/pkg/mod]

  arch-check:
    executor: go
    steps:
      - checkout
      - run:
          name: Architecture boundaries
          command: |
            go install github.com/fe3dback/go-arch-lint@latest
            go-arch-lint check --project-path .

  test:
    executor: go
    steps:
      - checkout
      - restore_cache:
          keys: [go-mod-{{ checksum "go.sum" }}]
      - run:
          name: Unit tests
          command: go test -race -count=1 -timeout=5m ./...

  docs-check:
    docker:
      - image: node:22-alpine
    steps:
      - checkout
      - run:
          name: Markdown lint
          command: |
            npm install -g markdownlint-cli2
            markdownlint-cli2 "docs/**/*.md"

workflows:
  ci:
    jobs:
      - lint
      - arch-check
      - test
      - docs-check
```

## Makefile як Interface

Всі CI команди доступні локально через Makefile:

```makefile
.PHONY: lint test arch-check docs-check ci

lint:
	golangci-lint run --timeout=3m

test:
	go test -race -count=1 -timeout=5m ./...

test-arch:
	go test -run TestArch -timeout=2m ./...

arch-check:
	go-arch-lint check --project-path .

docs-check:
	markdownlint-cli2 "docs/**/*.md"
	markdown-link-check docs/**/*.md --config .markdown-link-check.json || true

# Всі перевірки разом
ci: lint arch-check test test-arch docs-check
```

Агент запускає `make ci` для повної перевірки перед PR.

## Agent-Friendly Error Output

### GitHub Actions Annotations

CI помилки відображаються як inline annotations на PR:

```bash
# В CI скрипті
echo "::error file=internal/handler/user.go,line=15,title=Layer Violation::Transport layer imports repo directly. Use service layer instead."
echo "::warning file=docs/architecture.md,line=42,title=Stale Reference::Referenced function CreateUser() no longer exists in service/user.go"
```

### Problem Matchers

```json
// .github/matchers/go.json
{
  "problemMatcher": [{
    "owner": "go-test",
    "severity": "error",
    "pattern": [{
      "regexp": "^\\s+(.+\\.go):(\\d+):\\s+(.+)$",
      "file": 1,
      "line": 2,
      "message": 3
    }]
  }]
}
```

## Оптимізації швидкості

1. **Паралельні jobs** — lint, arch, test, docs одночасно
2. **`concurrency` + `cancel-in-progress`** — вбиває застарілі runs
3. **Go module caching** — `~/go/pkg/mod` кешується між runs
4. **`-count=1`** — вимикає Go test caching в CI (свіжі runs)
5. **`-race` тільки в CI** — локально пропускаємо для швидкості
6. **Timeout enforcement** — запобігає зависанню jobs

## Що робити зараз

### Крок 1: Makefile стандартизація (1 год)
1. Створити еталонний Makefile в claude-setup з targets: lint, test, arch-check, docs-check, ci
2. Скопіювати/адаптувати в кожен Go проєкт
3. Оновити CLAUDE.md — таблиця команд посилається на make targets

### Крок 2: CI pipeline (2 год)
1. Додати arch-check та docs-check до CircleCI workflows
2. Паралелізувати jobs
3. Переконатись що total wall-clock < 5 хв

### Крок 3: Merge rules (30 хв)
1. Налаштувати branch protection: require lint + arch + test
2. Docs-check як optional (warning only)
3. Дозволити auto-merge для PR з лейблом `refactor`