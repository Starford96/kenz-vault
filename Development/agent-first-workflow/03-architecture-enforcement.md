---
title: Архітектурний Enforcement
tags:
  - architecture
  - linting
  - go
  - agent-first
created: 2026-02-26
---

# Архітектурний Enforcement

## Центральна ідея

> "Make bad taste impossible" — замість лекцій про якість, зроби порушення неможливими механічно.

Архітектурні правила, що існують тільки в документації чи в голові архітектора, будуть порушені. Агент не має "здорового глузду" — він потребує механічних guardrails.

## Layered Architecture

### Принцип

Фіксована послідовність залежностей. Залежності течуть тільки "вперед":

```
Types → Config → Repo → Service → Runtime → Transport
```

| Шар | Директорія | Відповідальність | Може імпортувати |
|-----|-----------|------------------|------------------|
| Types | `internal/types/` | Доменні типи, інтерфейси | Нічого (крім std) |
| Config | `internal/config/` | Конфігурація | Types |
| Repo | `internal/repo/` | Data access | Types, Config |
| Service | `internal/service/` | Бізнес-логіка | Types, Config, Repo |
| Runtime | `internal/runtime/` | App lifecycle, DI | Types, Config, Repo, Service |
| Transport | `internal/transport/` | gRPC/HTTP handlers | Types, Config, Service, Runtime |

### Заборони
- Transport **НЕ** імпортує Repo напряму (тільки через Service)
- Service **НЕ** імпортує Transport
- Types **НЕ** імпортує нічого з internal/
- Cross-cutting concerns входять через єдиний explicit interface

## Інструмент 1: go-arch-lint (Рекомендовано)

Декларативний YAML-конфіг для enforcement'у імпорт-границь.

### Інсталяція

```bash
go install github.com/fe3dback/go-arch-lint@latest
```

### Конфігурація

```yaml
# .go-arch-lint.yml
version: 3
workdir: internal

components:
  types:     { in: types/** }
  config:    { in: config/** }
  repo:      { in: repo/** }
  service:   { in: service/** }
  runtime:   { in: runtime/** }
  transport: { in: transport/** }

commonComponents:
  - types          # types доступний всім

deps:
  config:
    mayDependOn: [types]
  repo:
    mayDependOn: [types, config]
  service:
    mayDependOn: [types, config, repo]
  runtime:
    mayDependOn: [types, config, repo, service]
  transport:
    mayDependOn: [types, config, service, runtime]
```

### Запуск

```bash
# Перевірка
go-arch-lint check --project-path .

# JSON output для CI/агентів
go-arch-lint check --project-path . --output-type json
```

### Makefile target

```makefile
.PHONY: arch-check
arch-check:
	@echo "Checking architecture boundaries..."
	go-arch-lint check --project-path .
```

## Інструмент 2: ArchUnit для Go (тест-based)

Архітектурні тести що запускаються через `go test`:

```go
// internal/architecture_test.go
package internal_test

import (
    "testing"
    "github.com/kcmvp/archunit"
)

func TestLayerBoundaries(t *testing.T) {
    typesLayer := archunit.ArchLayer("Types", "myapp/internal/types/...")
    configLayer := archunit.ArchLayer("Config", "myapp/internal/config/...")
    repoLayer := archunit.ArchLayer("Repo", "myapp/internal/repo/...")
    serviceLayer := archunit.ArchLayer("Service", "myapp/internal/service/...")
    runtimeLayer := archunit.ArchLayer("Runtime", "myapp/internal/runtime/...")
    transportLayer := archunit.ArchLayer("Transport", "myapp/internal/transport/...")

    arch := archunit.ArchUnit(
        typesLayer, configLayer, repoLayer,
        serviceLayer, runtimeLayer, transportLayer,
    )

    err := arch.Validate(
        archunit.Layers("Types").ShouldNotRefer(
            archunit.Layers("Config", "Repo", "Service", "Runtime", "Transport")),
        archunit.Layers("Config").ShouldNotRefer(
            archunit.Layers("Repo", "Service", "Runtime", "Transport")),
        archunit.Layers("Repo").ShouldNotRefer(
            archunit.Layers("Service", "Runtime", "Transport")),
        archunit.Layers("Service").ShouldNotRefer(
            archunit.Layers("Runtime", "Transport")),
        archunit.Layers("Runtime").ShouldNotRefer(
            archunit.Layers("Transport")),
    )
    if err != nil {
        t.Fatal(err)
    }
}
```

## Інструмент 3: Custom Linters (go/analysis)

### Навіщо

Стандартні лінтери не знають ваші правила. Custom linters:
- Специфічні для вашого проєкту
- Помилки містять інструкції по виправленню — прямо в контекст агента
- Підтримують `SuggestedFix` — автоматичне виправлення через `--fix`

### Приклад: заборона panic()

```go
// tools/linters/nopanic/analyzer.go
package nopanic

import (
    "go/ast"
    "golang.org/x/tools/go/analysis"
)

var Analyzer = &analysis.Analyzer{
    Name: "nopanic",
    Doc:  "disallows bare panic() calls outside init()",
    Run:  run,
}

func run(pass *analysis.Pass) (interface{}, error) {
    for _, file := range pass.Files {
        ast.Inspect(file, func(n ast.Node) bool {
            call, ok := n.(*ast.CallExpr)
            if !ok {
                return true
            }
            ident, ok := call.Fun.(*ast.Ident)
            if !ok || ident.Name != "panic" {
                return true
            }

            pass.Report(analysis.Diagnostic{
                Pos: call.Pos(),
                Message: "bare panic() found. " +
                    "Fix: return an error instead. " +
                    "Use fmt.Errorf(\"unexpected: %v\", val) or errors.New(). " +
                    "See docs/conventions.md#error-handling",
            })
            return true
        })
    }
    return nil, nil
}
```

### Патерн agent-friendly помилки

```
error internal/handler/user.go:15:2 [layer-violation]
  Package "myapp/internal/repo" imported in transport layer.
  Rule: transport must not import repo directly. Use service layer.
  Fix: Remove direct repo import. Inject through service constructor.
  Docs: docs/architecture.md#layer-rules
```

**Формула:**
1. `file:line:column` — машинно-парсабле розташування
2. **Що порушено** — конкретне порушення
3. **Яке правило** — посилання на правило
4. **Як виправити** — конкретна дія
5. **Де прочитати** — посилання на документацію

## golangci-lint: Agent-Friendly Output

### Конфігурація для структурованого output

```yaml
# .golangci.yml
output:
  formats:
    - format: text
      path: stdout
    - format: json
      path: ./lint-results.json

linters:
  enable:
    - errcheck
    - staticcheck
    - govet
    - ineffassign
    - unused
    - revive
    - gocyclo
    - gocognit
    - gosec
    - gocritic

linters-settings:
  revive:
    rules:
      - name: exported
        arguments: [checkPrivateReceivers]
      - name: error-return
      - name: error-naming
      - name: increment-decrement
```

## Claude Code Hooks для Enforcement

### Auto-lint при збереженні

```json
// .claude/settings.json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "filepath=\"$CLAUDE_FILE_PATH\"; if [[ \"$filepath\" == *.go ]]; then gofmt -w \"$filepath\" && goimports -w \"$filepath\"; fi"
          }
        ]
      }
    ]
  }
}
```

### Блокування небезпечних операцій

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "prompt",
            "prompt": "Check if this bash command is destructive (rm -rf, DROP TABLE, git push --force). If destructive, output JSON: {\"decision\": \"block\", \"reason\": \"Destructive operation blocked\"}. If safe, output: {\"decision\": \"allow\"}"
          }
        ]
      }
    ]
  }
}
```

## "Parse, Don't Validate" Pattern

Принцип з Harness Engineering: валідація на границях системи, strict types всередині.

```go
// internal/types/user.go — strict types
type UserID string
type Email string

// internal/transport/handler.go — parsing at boundary
func parseCreateUserRequest(req *pb.CreateUserRequest) (types.CreateUserParams, error) {
    if req.Email == "" {
        return types.CreateUserParams{}, fmt.Errorf("email is required")
    }
    email, err := types.ParseEmail(req.Email)
    if err != nil {
        return types.CreateUserParams{}, fmt.Errorf("invalid email: %w", err)
    }
    return types.CreateUserParams{
        Email: email,
        Name:  req.Name,
    }, nil
}

// internal/service/user.go — no validation needed, types are already parsed
func (s *UserService) Create(ctx context.Context, params types.CreateUserParams) (*types.User, error) {
    // params.Email is already validated — just use it
    return s.repo.Create(ctx, params)
}
```

## Що робити зараз

### Крок 1: go-arch-lint (30 хв)
1. `go install github.com/fe3dback/go-arch-lint@latest`
2. Створити `.go-arch-lint.yml` для кожного Go проєкту
3. Додати `make arch-check` target
4. Додати в CI pipeline

### Крок 2: golangci-lint стандартизація (1 год)
1. Створити еталонний `.golangci.yml` в claude-setup
2. Скопіювати в усі Go проєкти (або symlink)
3. Переконатись що вивід включає fix instructions

### Крок 3: Custom linters (за потреби)
1. Почати з простих правил (no-panic, layer-violations)
2. Використовувати `go/analysis` framework
3. Error messages завжди включають інструкцію по виправленню

### Крок 4: Hooks (15 хв)
1. Оновити `.claude/settings.json` — auto-gofmt/goimports при Edit/Write
2. Додати в шаблон settings-init