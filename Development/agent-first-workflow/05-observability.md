---
title: Observability для агентів
tags:
  - observability
  - monitoring
  - agent-first
  - opentelemetry
created: 2026-02-26
---

# Observability для агентів

## Центральна ідея

> Агент не може виправити те, чого не бачить. Observability робить додаток "legible" для агента.

OpenAI інтегрували повний observability stack в рантайм агента: логи (Victoria Logs / LogQL), метрики (Victoria Metrics / PromQL), трейси (OpenTelemetry), та навіть браузерне середовище (Chrome DevTools Protocol). Агент сам пише запити для дебагу.

## Рівні Observability

### Рівень 1: Structured Logging (Базовий)

**Що:** Структуровані логи (JSON) з контекстом, що агент може фільтрувати.

**Go реалізація (slog):**

```go
// internal/runtime/logger.go
package runtime

import (
    "log/slog"
    "os"
)

func NewLogger(env string) *slog.Logger {
    var handler slog.Handler
    if env == "development" {
        handler = slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
            Level: slog.LevelDebug,
        })
    } else {
        handler = slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
            Level: slog.LevelInfo,
        })
    }
    return slog.New(handler)
}
```

**Чому JSON:** Агент може grep'ати по полях:
```bash
# Агент шукає помилки в конкретному сервісі
cat app.log | jq 'select(.level == "ERROR" and .service == "user")'
```

### Рівень 2: OpenTelemetry Tracing (Рекомендовано)

**Що:** Distributed tracing дозволяє агенту бачити повний шлях запиту через мікросервіси.

**Go реалізація:**

```go
// internal/runtime/telemetry.go
package runtime

import (
    "context"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
    "go.opentelemetry.io/otel/sdk/resource"
    "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
)

func InitTracer(ctx context.Context, serviceName string) (*trace.TracerProvider, error) {
    exporter, err := otlptracehttp.New(ctx)
    if err != nil {
        return nil, fmt.Errorf("create OTLP exporter: %w", err)
    }

    tp := trace.NewTracerProvider(
        trace.WithBatcher(exporter),
        trace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName(serviceName),
        )),
    )
    otel.SetTracerProvider(tp)
    return tp, nil
}
```

### Рівень 3: Метрики (PromQL)

**Що:** Кількісні показники що агент може запитувати.

```go
// internal/runtime/metrics.go
package runtime

import (
    "go.opentelemetry.io/otel/metric"
)

type Metrics struct {
    RequestDuration metric.Float64Histogram
    RequestCount    metric.Int64Counter
    ErrorCount      metric.Int64Counter
    ActiveConns     metric.Int64UpDownCounter
}

func NewMetrics(meter metric.Meter) (*Metrics, error) {
    reqDuration, err := meter.Float64Histogram("http_request_duration_seconds",
        metric.WithDescription("HTTP request duration in seconds"),
    )
    if err != nil {
        return nil, err
    }
    // ... інші метрики
    return &Metrics{RequestDuration: reqDuration}, nil
}
```

**Агент-кейс:** Замість "додаток гальмує" → агент запитує:
```promql
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

### Рівень 4: Local Observability Stack (Per-Worktree)

**Що:** Ефемерний observability stack що піднімається per git worktree і зникає коли задача завершена.

**Docker Compose:**

```yaml
# docker-compose.observability.yml
services:
  victoriametrics:
    image: victoriametrics/victoria-metrics:latest
    ports: ["8428:8428"]
    volumes: ["./data/vm:/victoria-metrics-data"]

  victorialogs:
    image: victoriametrics/victoria-logs:latest
    ports: ["9428:9428"]
    volumes: ["./data/vl:/victoria-logs-data"]

  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
```

**Makefile targets:**

```makefile
.PHONY: obs-up obs-down obs-logs obs-metrics

obs-up:
	docker compose -f docker-compose.observability.yml up -d

obs-down:
	docker compose -f docker-compose.observability.yml down -v

obs-logs:
	@echo "Victoria Logs: http://localhost:9428/select/logsql/query"

obs-metrics:
	@echo "Victoria Metrics: http://localhost:8428/vmui"
```

### Рівень 5: CDP Integration (Для UI-проєктів)

**Що:** Chrome DevTools Protocol дозволяє агенту робити DOM snapshots, screenshots, навігацію.

**Релевантно для:** pf3-studio-portal та інших React/UI проєктів.

**Playwright приклад:**

```javascript
// scripts/cdp-snapshot.js
const { chromium } = require('playwright');

async function snapshot(url) {
    const browser = await chromium.launch({ headless: true });
    const page = await browser.newPage();
    await page.goto(url);

    // DOM snapshot
    const dom = await page.content();

    // Screenshot
    await page.screenshot({ path: 'screenshot.png', fullPage: true });

    // Console errors
    const errors = [];
    page.on('console', msg => {
        if (msg.type() === 'error') errors.push(msg.text());
    });

    await browser.close();
    return { dom, errors };
}
```

## Observability для Claude Code

### Хук для збору результатів

```json
// .claude/settings.json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo \"Session completed at $(date)\" >> ~/.claude/agent-logs/sessions.log"
          }
        ]
      }
    ]
  }
}
```

### MCP Server для метрик

Можна підключити Victoria Metrics через MCP:

```json
// .mcp.json
{
  "mcpServers": {
    "metrics": {
      "command": "npx",
      "args": ["-y", "@anthropic/mcp-server-http"],
      "env": {
        "BASE_URL": "http://localhost:8428"
      }
    }
  }
}
```

## Від вимог до тестабельних інваріантів

Ключовий інсайт з Harness Engineering:

| Вимога (людська) | Інваріант (тестабельний) |
|-----------------|------------------------|
| "Додаток гальмує" | Startup < 800ms, p99 latency < 200ms |
| "Щось зламалось" | Error rate < 0.1% за останні 5 хв |
| "Повільні запити" | `histogram_quantile(0.99, ...) < 0.5` |
| "Memory leak" | RSS growth < 10MB/hour under load |

Агент перетворює нечіткі вимоги на конкретні метричні запити.

## Що робити зараз

### Крок 1: Structured logging (1 год)
1. Перевести логування на `slog` з JSON output (якщо ще не)
2. Додати контекстні поля: service, method, request_id, duration
3. Логи мають бути grep-able агентом

### Крок 2: OpenTelemetry (2-4 год)
1. Додати OTel SDK в Go сервіси
2. Інструментувати gRPC/HTTP handlers
3. Експортувати в OTLP endpoint

### Крок 3: Local observability (1 год)
1. Створити `docker-compose.observability.yml`
2. Додати Makefile targets: obs-up, obs-down
3. Агент може піднімати стек per-задачу

### Крок 4: Dashboards (2 год)
1. Provisioned Grafana dashboards (JSON as code)
2. Ключові панелі: latency, error rate, throughput
3. Агент може дивитись dashboards через URL