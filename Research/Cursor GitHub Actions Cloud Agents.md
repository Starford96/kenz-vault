---
title: "Cursor + GitHub Actions: повний гайд по хмарних AI-агентах для PR"
tags:
  - cursor
  - github-actions
  - ai-agents
  - ci-cd
  - code-review
  - cloud-agents
  - bugbot
  - research
created: 2026-02-27
---

# Cursor + GitHub Actions: повний гайд по хмарних AI-агентах для PR

> Дослідження на основі офіційної документації Cursor (cursor.com/docs) станом на 2026-02-27.
> Покриває: Cloud Agents API, CLI headless, Bugbot, GitHub інтеграцію, Subagents, Hooks, Skills, безпеку, ціноутворення.

---

## Зміст

1. [[#Огляд екосистеми]]
2. [[#Cloud Agents — концепція та запуск]]
3. [[#Cloud Agents REST API]]
4. [[#Cursor CLI в GitHub Actions]]
5. [[#Bugbot — автоматичний код-рев'юер]]
6. [[#GitHub інтеграція (@cursor)]]
7. [[#Subagents — делегування задач]]
8. [[#Hooks — контроль агентного циклу]]
9. [[#Agent Skills — розширення можливостей]]
10. [[#Cookbook — готові GitHub Actions workflows]]
11. [[#Безпека та мережевий доступ]]
12. [[#Ціноутворення]]
13. [[#Best Practices]]
14. [[#Порівняння підходів]]
15. [[#Джерела]]

---

## Огляд екосистеми

Cursor надає **5 основних інтерфейсів** для запуску AI-агентів:

| Інтерфейс | Опис | Тип доступу |
|------------|------|-------------|
| **cursor.com/agents** | Веб-інтерфейс для запуску Cloud Agents | Браузер / PWA |
| **Cursor Desktop** | Dropdown "Cloud" в агентному інпуті | Десктопна аппка |
| **Cursor CLI** | `agent` команда для headless/CI | Термінал / GitHub Actions |
| **Cloud Agents API** | REST API (`api.cursor.com/v0/`) | Програмний |
| **@cursor** | Коментар на PR/issue в GitHub, Slack, Linear | GitHub / Slack |

Додатково: **Bugbot** — окремий продукт для автоматичного code review.

---

## Cloud Agents — концепція та запуск

### Що таке Cloud Agent?

Хмарний агент, що працює в **ізольованій Ubuntu VM** з повним середовищем:
- Власна файлова система з клонованим репо
- Термінал з правами на виконання команд
- Браузер та десктоп (Computer Use)
- Доступ до MCP-інструментів
- Мережевий доступ (налаштовується)

### Як працює

1. Агент **клонує репо** з GitHub/GitLab
2. Працює на **окремій гілці**
3. Може білдити, тестувати, запускати dev server
4. **Пушить зміни** та створює PR для handoff

### Канали запуску

```
cursor.com/agents  →  Веб/PWA (мобільний теж)
Cursor Desktop     →  Agent input → Cloud dropdown
Slack             →  @cursor [prompt]
GitHub            →  @cursor [prompt] на PR/issue
Linear            →  @cursor [prompt]
API               →  POST api.cursor.com/v0/agents
```

### Налаштування середовища (Setup)

Два варіанти:

**Варіант 1: Agent-driven setup (рекомендовано)**
1. Перейти на cursor.com/onboard
2. Підключити GitHub/GitLab, обрати репо
3. Додати секрети та env-змінні
4. Агент сам інсталює залежності
5. Зберегти snapshot VM для повторного використання

**Варіант 2: Manual Dockerfile**

Створити `.cursor/environment.json`:

```json
{
  "build": {
    "dockerfile": "Dockerfile",
    "context": ".."
  },
  "install": "npm install",
  "terminals": [
    {
      "name": "Run Next.js",
      "command": "npm run dev"
    }
  ]
}
```

- `dockerfile` і `context` — відносно `.cursor/`
- `install` — запускається з кореня проєкту
- `terminals` — процеси в tmux-сесії, доступній і агенту, і тобі

Повна JSON Schema: `https://www.cursor.com/schemas/environment.schema.json`

### Update та Startup команди

```
install  →  "npm install" / "bazel build"  →  кешується на диску
start    →  запускається після install (наприклад "sudo service docker start")
terminals →  tmux-сесія з app-процесами
```

### Секрети

Налаштовуються через cursor.com/dashboard → Cloud Agents:
- Шифруються at rest (KMS) та in transit
- Передаються як env-змінні
- **Redacted** секрети — додатковий захист:
  - Скануються в комітах (reject якщо знайдено)
  - Редагуються з tool calls (не потрапляють в транскрипт)

### Computer Use

Агенти можуть контролювати десктоп і браузер:
- Мишка, клавіатура, скріншоти
- Відкривати dev server, клікати по UI
- Верифікувати зміни візуально
- Відеозапис сесії

> ⚠️ Computer Use поки не підтримується для репо з Dockerfile або snapshot через environment.json.

### Long-Running Agents

Для масштабних задач (Ultra/Teams/Enterprise):
- Працюють **годинами або днями**
- Мульти-агентна архітектура: planner → executor → workers
- Пропонують план → чекають затвердження → виконують

---

## Cloud Agents REST API

Base URL: `https://api.cursor.com/v0/`

### Аутентифікація

Basic Auth з API-ключем (Cursor Dashboard → Settings → Advanced → New Admin API Key):

```bash
# Перевірка доступу
curl --request GET \
  --url https://api.cursor.com/v0/me \
  -u YOUR_API_KEY:
```

### Запуск агента

```bash
curl --request POST \
  --url https://api.cursor.com/v0/agents \
  -u YOUR_API_KEY: \
  --header "Content-Type: application/json" \
  --data '{
    "prompt": {
      "text": "Review the PR and fix all linting issues. Run tests to verify."
    },
    "source": {
      "repository": "https://github.com/your-org/your-repo",
      "ref": "main"
    },
    "target": {
      "autoCreatePr": true,
      "branchName": "cursor/fix-lint-issues"
    }
  }'
```

**Відповідь:**

```json
{
  "id": "bc_abc123",
  "status": "CREATING",
  "source": { "repository": "...", "ref": "main" },
  "target": {
    "branchName": "cursor/fix-lint-issues",
    "url": "https://cursor.com/agents?id=bc_abc123",
    "prUrl": "https://github.com/your-org/your-repo/pull/123",
    "autoCreatePr": true
  }
}
```

### Запуск на існуючому PR

```bash
curl --request POST \
  --url https://api.cursor.com/v0/agents \
  -u YOUR_API_KEY: \
  --header "Content-Type: application/json" \
  --data '{
    "prompt": { "text": "Fix all failing tests in this PR" },
    "source": {
      "prUrl": "https://github.com/your-org/repo/pull/42"
    },
    "target": { "autoBranch": false }
  }'
```

`autoBranch: false` — пушить прямо в бранч PR.

### Follow-up інструкції

```bash
curl --request POST \
  --url https://api.cursor.com/v0/agents/bc_abc123/followup \
  -u YOUR_API_KEY: \
  --header "Content-Type: application/json" \
  --data '{ "prompt": { "text": "Also add unit tests for the new function" } }'
```

### Перевірка статусу

```bash
curl --request GET \
  --url https://api.cursor.com/v0/agents/bc_abc123 \
  -u YOUR_API_KEY:
```

Lifecycle: `CREATING` → `RUNNING` → `FINISHED`

### Отримання розмови (conversation)

```bash
curl --request GET \
  --url https://api.cursor.com/v0/agents/bc_abc123/conversation \
  -u YOUR_API_KEY:
```

### Зупинка та видалення

```bash
# Зупинка (можна відновити через followup)
curl --request POST \
  --url https://api.cursor.com/v0/agents/bc_abc123/stop \
  -u YOUR_API_KEY:

# Видалення (безповоротне)
curl --request DELETE \
  --url https://api.cursor.com/v0/agents/bc_abc123 \
  -u YOUR_API_KEY:
```

### Webhook нотифікації

```json
{
  "prompt": { "text": "..." },
  "source": { "repository": "..." },
  "webhook": {
    "url": "https://your-server.com/cursor-webhook",
    "secret": "your-secret-key-min-32-chars-long-here"
  }
}
```

### Доступні моделі

```bash
curl --request GET \
  --url https://api.cursor.com/v0/models \
  -u YOUR_API_KEY:
```

Приклад: `["claude-4-sonnet-thinking", "gpt-5.2", "claude-4.5-sonnet-thinking"]`

---

## Cursor CLI в GitHub Actions

### Встановлення CLI

```yaml
- name: Install Cursor CLI
  run: |
    curl https://cursor.com/install -fsS | bash
    echo "$HOME/.cursor/bin" >> $GITHUB_PATH
```

### Аутентифікація

**Через env-змінну (рекомендовано для CI):**

```bash
export CURSOR_API_KEY=your_api_key_here
agent "implement user authentication"
```

**Через прапорець:**

```bash
agent --api-key your_api_key_here "implement user authentication"
```

**Генерація ключа:**
Dashboard → Integrations → User API Keys

**Збереження в GitHub:**

```bash
gh secret set CURSOR_API_KEY --repo OWNER/REPO --body "$CURSOR_API_KEY"
```

### Команда `agent`

```bash
agent [options] "prompt"
```

**Ключові прапорці:**

| Прапорець | Опис |
|-----------|------|
| `-p` / `--print` | Headless режим (без інтерактиву) |
| `--force` / `--yolo` | Дозволяє модифікацію файлів |
| `--model` | Вибір моделі (gpt-5.2, claude-4-sonnet-thinking тощо) |
| `--output-format` | `text`, `json`, `stream-json` |
| `--api-key` | API ключ |
| `--insecure` | Ігнорувати SSL помилки |
| `--endpoint` | Custom API endpoint |

**Інтерактивні slash-команди:**

```bash
/model auto          # авто-вибір
/model gpt-5.2       # конкретна модель
/model sonnet-4.5-thinking
```

### Конфігурація CLI

Файли конфігу:

| Тип | Платформа | Шлях |
|-----|-----------|------|
| Global | macOS/Linux | `~/.cursor/cli-config.json` |
| Global | Windows | `$env:USERPROFILE\.cursor\cli-config.json` |
| Project | All | `<project>/.cursor/cli.json` |

**Мінімальний конфіг:**

```json
{
  "version": 1,
  "editor": { "vimMode": false },
  "permissions": { "allow": ["Shell(ls)"], "deny": [] }
}
```

**Повний конфіг:**

```json
{
  "version": 1,
  "editor": { "vimMode": false },
  "permissions": {
    "allow": [
      "Shell(ls)", "Shell(git)", "Shell(npm)",
      "Read(src/**/*.ts)", "Read(**/*.md)",
      "Write(src/**)", "Write(package.json)",
      "WebFetch(docs.github.com)", "WebFetch(*.github.com)",
      "Mcp(datadog:*)"
    ],
    "deny": [
      "Shell(rm)", "Read(.env*)",
      "Write(**/*.key)", "Write(**/.env*)",
      "WebFetch(malicious-site.com)"
    ]
  },
  "network": { "useHttp1ForAgent": false },
  "attribution": {
    "attributeCommitsToAgent": true,
    "attributePRsToAgent": true
  }
}
```

### Система дозволів (Permissions)

5 типів permission tokens:

**Shell — команди:**
```
Shell(ls)         — дозволити ls
Shell(git)        — будь-яка git підкоманда
Shell(curl:*)     — curl з будь-якими аргументами
Shell(rm)         — заборонити (в deny)
```

**Read — читання файлів:**
```
Read(src/**/*.ts)  — TypeScript файли в src
Read(**/*.md)      — markdown скрізь
Read(.env*)        — заборонити env файли
```

**Write — запис файлів:**
```
Write(src/**)       — будь-що в src
Write(package.json) — конкретний файл
Write(**/*.key)     — заборонити ключі
```

**WebFetch — HTTP запити:**
```
WebFetch(docs.github.com)  — конкретний домен
WebFetch(*.example.com)    — піддомени
WebFetch(*)                — всі (обережно!)
```

**Mcp — MCP інструменти:**
```
Mcp(datadog:*)  — всі інструменти Datadog
Mcp(*:search)   — search з будь-якого сервера
Mcp(*:*)        — всі MCP (обережно!)
```

> Deny rules мають пріоритет над allow rules.

### Рівні автономності в CI

**Повна автономність:**
```yaml
agent --force -p "Handle the entire workflow including commits, pushes, and PR comments."
```

**Обмежена автономність (рекомендовано для продакшену):**
```yaml
# Агент тільки змінює файли
- name: Generate fixes
  run: |
    agent -p "IMPORTANT: Do NOT commit, push, or post comments.
    Only modify files." --force

# CI робить git/publish (детерміновано)
- name: Publish
  run: |
    git checkout -B "fix/${{ github.head_ref }}"
    git add -A && git commit -m "fix: automated"
    git push origin "fix/${{ github.head_ref }}"
```

---

## Bugbot — автоматичний код-рев'юер

Окремий продукт від Cursor. Рев'юїть PR і знаходить баги, security-проблеми та code quality issues.

### Як працює

1. Автоматичний рев'ю на **кожен PR update**
2. Або ручний тригер: `cursor review` / `bugbot run`
3. **Читає існуючі PR коментарі** як контекст (уникає дублів)
4. Залишає inline коментарі з поясненнями та пропозиціями
5. Кнопки: "Fix in Cursor" / "Fix in Web"

### Підтримка SCM

- GitHub.com ✅
- GitHub Enterprise Server (v3.8+) ✅
- GitLab.com (Premium/Ultimate) ✅
- GitLab Self-Hosted (Premium/Ultimate) ✅

### Setup (GitHub.com)

1. cursor.com/dashboard → Integrations
2. Connect GitHub → Install
3. Увімкнути Bugbot на конкретних репо

### Правила (.cursor/BUGBOT.md)

Каскадна система правил:

```
project/
  .cursor/BUGBOT.md          # Завжди включається (project-wide)
  backend/
    .cursor/BUGBOT.md        # Включається при рев'ю backend файлів
    api/
      .cursor/BUGBOT.md      # Включається при рев'ю API файлів
  frontend/
    .cursor/BUGBOT.md        # Включається при рев'ю frontend файлів
```

Порядок застосування: **Team Rules → project BUGBOT.md (з вкладеними) → User Rules**

**Приклад: безпека**
```text
If any changed file contains the string pattern /\beval\s*\(|\bexec\s*\(/i, then:
- Add a blocking Bug with title "Dangerous dynamic execution"
- Body: "Replace with safe alternatives or justify with tests."
- Apply label "security".
```

**Приклад: обов'язкові тести**
```text
If the PR modifies files in {server/**, api/**, backend/**} and there are no changes
in {**/*.test.*, **/__tests__/**, tests/**}, then:
- Add a blocking Bug titled "Missing tests for backend changes"
- Apply label "quality"
```

**Приклад: DB міграції (реальний Cursor внутрішній)**
```text
## Database migrations
- New tables, nullable/default columns: OK
- New indices on existing tables: MUST use CONCURRENTLY, isolate to own migration file
- Adding foreign keys: NOT OK
- Dropping columns: NOT OK (use @map("original_name") deprecation)
- Renaming/changing types: NOT OK (exceptions: increase VARCHAR, make nullable)
- Dropping tables: NOT OK
- New tables: prefer BIGINTEGER over INTEGER, no foreign keys, no cascading deletes
- Migrations isolated to own PR when possible
```

### Bugbot Autofix

Автоматично спавнить Cloud Agent для виправлення знайдених багів:

1. Bugbot знаходить баги під час рев'ю
2. Спавнить Cloud Agent для аналізу та фіксу
3. Пушить в існуючу або нову гілку
4. Постить коментар з результатами

**Налаштування autofix:**
- **Off** — вимкнено
- **Create New Branch** (рекомендовано)
- **Commit to Existing Branch** (макс. 3 спроби на PR)

### Bugbot Admin API

```bash
# Увімкнути Bugbot на репо
curl -X POST https://api.cursor.com/bugbot/repo/update \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"repoUrl": "https://github.com/org/repo", "enabled": true}'

# Список репо
curl https://api.cursor.com/bugbot/repos \
  -H "Authorization: Bearer $API_KEY"

# Управління доступом
curl -X POST https://api.cursor.com/bugbot/user/update \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"username": "octocat", "allow": true}'
```

### Bugbot Pricing

- **Pro** ($40/mo): до 200 PR/міс, unlimited Cursor Ask
- **Teams** ($40/user/mo): unlimited PR, pooled usage
- **Enterprise**: custom

---

## GitHub інтеграція (@cursor)

### Встановлення

1. cursor.com/dashboard → Integrations → Connect GitHub
2. Обрати All repositories або Selected
3. GitHub App: github.com/apps/cursor

### Дозволи GitHub App

| Permission | Призначення |
|------------|-------------|
| Repository access | Клонування та створення бранчів |
| Pull requests | Створення PR з агентними змінами |
| Issues | Трекінг багів та задач |
| Checks and statuses | Звіти про якість коду |
| Actions and workflows | Моніторинг CI/CD |

### Використання

```
@cursor fix the failing tests          ← запуск Cloud Agent
@cursor fix                            ← фікс Bugbot коментаря
@cursor autofix off                    ← вимкнути autofix на PR
@cursor please fix the CI failures     ← фікс CI
```

Cloud Agents автоматично виправляють GitHub Actions failures на PR, які вони створили.

### IP Allow List

Для організацій з IP allowlist:

```text
184.73.225.134
3.209.66.12
52.44.113.131
```

Рекомендовано: GitHub → Security → IP allow list → "Allow access by GitHub Apps"

### Egress IP Ranges

Доступні через API:

```bash
curl https://cursor.com/docs/ips.json
```

Відповідь містить IP ranges по кластерах (us3p, us4p, us5p) у CIDR нотації.

---

## Subagents — делегування задач

Subagent = ізольований AI-помічник з власним контекстним вікном. Агент може делегувати задачі субагентам.

### Вбудовані субагенти

| Субагент | Призначення | Чому окремий |
|----------|-------------|--------------|
| **Explore** | Пошук по кодовій базі | Генерує великий проміжний output |
| **Bash** | Серія shell-команд | Verbose command output |
| **Browser** | Контроль браузера через MCP | Noisy DOM snapshots |

### Foreground vs Background

| Режим | Поведінка | Для чого |
|-------|-----------|----------|
| **Foreground** | Блокує до завершення | Послідовні задачі |
| **Background** | Повертає одразу | Довгі задачі, паралельна робота |

### Кастомні субагенти

**Розташування:**

| Тип | Шлях | Скоп |
|-----|------|------|
| Project | `.cursor/agents/` | Поточний проєкт |
| Project | `.claude/agents/` | Сумісність з Claude |
| Project | `.codex/agents/` | Сумісність з Codex |
| User | `~/.cursor/agents/` | Глобально |

**Формат файлу:**

```markdown
---
name: security-auditor
description: Security specialist. Use when implementing auth, payments, or handling sensitive data.
model: inherit
readonly: false
is_background: false
---

You are a security expert auditing code for vulnerabilities.

When invoked:
1. Identify security-sensitive code paths
2. Check for common vulnerabilities (injection, XSS, auth bypass)
3. Verify secrets are not hardcoded
4. Review input validation and sanitization

Report findings by severity:
- Critical (must fix before deploy)
- High (fix soon)
- Medium (address when possible)
```

**Поля конфігу:**

| Поле | Обов'язкове | Опис |
|------|------------|------|
| `name` | Ні | Ідентифікатор (lowercase + hyphens, default = filename) |
| `description` | Ні | Коли використовувати |
| `model` | Ні | `fast`, `inherit`, або ID моделі |
| `readonly` | Ні | Обмежити write permissions |
| `is_background` | Ні | Запускати у фоні |

**Виклик:**

```text
> /verifier confirm the auth flow is complete
> /debugger investigate this error
> /security-auditor review the payment module
```

---

## Hooks — контроль агентного циклу

Hooks = кастомні скрипти для спостереження, блокування та модифікації поведінки агента.

### Hook Events

**Agent (Cmd+K / Agent Chat):**

| Event | Коли |
|-------|------|
| `sessionStart` / `sessionEnd` | Lifecycle сесії |
| `preToolUse` / `postToolUse` | Перед/після будь-якого tool |
| `subagentStart` / `subagentStop` | Lifecycle субагентів |
| `beforeShellExecution` / `afterShellExecution` | Shell-команди |
| `beforeMCPExecution` / `afterMCPExecution` | MCP-виклики |
| `beforeReadFile` / `afterFileEdit` | Файлові операції |
| `beforeSubmitPrompt` | Валідація промптів |
| `preCompact` | Компактифікація контексту |
| `stop` | Завершення агента |
| `afterAgentResponse` / `afterAgentThought` | Відповіді агента |

**Tab (inline completions):**
- `beforeTabFileRead` — контроль файлового доступу
- `afterTabFileEdit` — пост-обробка Tab edits

### Конфігурація

`.cursor/hooks.json` (project) або `~/.cursor/hooks.json` (global):

```json
{
  "version": 1,
  "hooks": {
    "afterFileEdit": [
      { "command": ".cursor/hooks/format.sh" }
    ],
    "beforeShellExecution": [
      {
        "command": ".cursor/hooks/approve-network.sh",
        "timeout": 30,
        "matcher": "curl|wget|nc"
      }
    ]
  }
}
```

### Типи hooks

**Command-based (default):**
- Отримує JSON через stdin
- Повертає JSON через stdout
- Exit code 0 = success, 2 = block, інше = fail-open

**Prompt-based (LLM-evaluated):**

```json
{
  "hooks": {
    "beforeShellExecution": [
      {
        "type": "prompt",
        "prompt": "Does this command look safe? Only allow read-only operations.",
        "timeout": 10
      }
    ]
  }
}
```

### Приклад: блокування git-команд

```bash
#!/bin/bash
input=$(cat)
command=$(echo "$input" | jq -r '.command // empty')

if [[ "$command" =~ git[[:space:]] ]]; then
    cat << EOF
{
  "continue": true,
  "permission": "deny",
  "user_message": "Git command blocked. Use gh CLI instead.",
  "agent_message": "Use gh repo clone, gh pr create, etc."
}
EOF
else
    cat << EOF
{ "continue": true, "permission": "allow" }
EOF
fi
```

---

## Agent Skills — розширення можливостей

Skills = портативні пакети, що вчать агента виконувати domain-specific задачі.

### Директорії

| Шлях | Скоп |
|------|------|
| `.agents/skills/` | Project |
| `.cursor/skills/` | Project |
| `~/.cursor/skills/` | User (global) |
| `.claude/skills/`, `.codex/skills/` | Сумісність |

### Структура

```
.agents/skills/deploy-app/
├── SKILL.md
├── scripts/
│   ├── deploy.sh
│   └── validate.py
├── references/
│   └── REFERENCE.md
└── assets/
    └── config-template.json
```

### SKILL.md формат

```markdown
---
name: deploy-app
description: Deploy the application. Use when deploying code or mentioning deployment.
license: MIT
disable-model-invocation: false
---

# Deploy App

Deploy using the provided scripts.

## Usage
Run: `scripts/deploy.sh <environment>`

## Pre-deployment Validation
Run: `python scripts/validate.py`
```

### Автоматичне використання

Cursor автоматично виявляє skills і використовує їх за description. Для виключно ручного виклику: `disable-model-invocation: true`.

Ручний виклик: `/skill-name` в чаті.

---

## Cookbook — готові GitHub Actions workflows

### 1. Автоматичний Code Review

`.github/workflows/cursor-code-review.yml`:

```yaml
name: Code Review
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]

permissions:
  pull-requests: write
  contents: read
  issues: write

jobs:
  code-review:
    runs-on: ubuntu-latest
    if: github.event.pull_request.draft == false
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          ref: ${{ github.event.pull_request.head.sha }}

      - name: Install Cursor CLI
        run: |
          curl https://cursor.com/install -fsS | bash
          echo "$HOME/.cursor/bin" >> $GITHUB_PATH

      - name: Configure git
        run: |
          git config user.name "Cursor Agent"
          git config user.email "cursoragent@cursor.com"

      - name: Code review
        env:
          CURSOR_API_KEY: ${{ secrets.CURSOR_API_KEY }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          agent --force --model "gpt-5.2" --output-format=text --print \
            "You are in a GitHub Actions runner.
            The gh CLI is available. Repo: ${{ github.repository }},
            PR: ${{ github.event.pull_request.number }}.
            Review the diff. Flag high-severity issues with inline comments.
            Use: gh pr diff, gh api for inline review comments.
            Max 10 comments. Emojis: 🚨 Critical 🔒 Security ⚡ Perf."
```

### 2. Автофікс CI Failures

`.github/workflows/fix-ci.yml`:

```yaml
name: Fix CI Failures
on:
  workflow_run:
    workflows: [Test]
    types: [completed]

permissions:
  contents: write
  pull-requests: write
  actions: read

jobs:
  attempt-fix:
    if: github.event.workflow_run.conclusion == 'failure'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }

      - name: Install Cursor CLI
        run: |
          curl https://cursor.com/install -fsS | bash
          echo "$HOME/.cursor/bin" >> $GITHUB_PATH

      - name: Configure git
        run: |
          git config user.name "Cursor Agent"
          git config user.email "cursoragent@cursor.com"

      - name: Fix CI
        env:
          CURSOR_API_KEY: ${{ secrets.CURSOR_API_KEY }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          agent -p "Workflow run ${{ github.event.workflow_run.id }} failed.
          Repo: ${{ github.repository }}.
          Use gh run view, gh pr diff to find the issue.
          Make minimal targeted fixes. Push to a fix branch.
          Post a PR comment with compare link." \
          --force --model "gpt-5.2" --output-format=text
```

### 3. Автоматичне оновлення документації

`.github/workflows/update-docs.yml`:

```yaml
name: Update Docs
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]

permissions:
  contents: write
  pull-requests: write

jobs:
  docs:
    if: ${{ !startsWith(github.head_ref, 'docs/') }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }

      - name: Install Cursor CLI
        run: |
          curl https://cursor.com/install -fsS | bash
          echo "$HOME/.cursor/bin" >> $GITHUB_PATH

      - name: Configure git
        run: |
          git config user.name "Cursor Agent"
          git config user.email "cursoragent@cursor.com"

      - name: Update docs
        env:
          CURSOR_API_KEY: ${{ secrets.CURSOR_API_KEY }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          agent -p "Repo: ${{ github.repository }},
          PR: ${{ github.event.pull_request.number }}.
          Detect changed code, update only relevant docs.
          Maintain persistent docs branch (prefix: docs/).
          Push to origin. Post PR comment with compare link.
          Do NOT create PRs directly." \
          --force --model "gpt-5.2" --output-format=text
```

### 4. Security Audit

`.github/workflows/secret-audit.yml`:

```yaml
name: Secrets Audit
on:
  schedule:
    - cron: "0 4 * * *"
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write
  actions: read

jobs:
  secrets-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }

      - name: Install Cursor CLI
        run: |
          curl https://cursor.com/install -fsS | bash
          echo "$HOME/.cursor/bin" >> $GITHUB_PATH

      - name: Configure git
        run: |
          git config user.name "Cursor Agent"
          git config user.email "cursoragent@cursor.com"

      - name: Scan and harden
        env:
          CURSOR_API_KEY: ${{ secrets.CURSOR_API_KEY }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          agent -p "Repo: ${{ github.repository }}.
          Scan for secrets in tracked files and history.
          Detect risky workflow patterns: unpinned actions, overbroad permissions.
          Maintain persistent audit branch. Pin actions to SHA.
          Reduce permissions. Add SECURITY_LOG.md.
          Post PR comment with compare link if open PRs exist." \
          --force --model "gpt-5.2" --output-format=text
```

### 5. Переклад i18n ключів

`.github/workflows/translate-keys.yml`:

```yaml
name: Translate Keys
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]

permissions:
  contents: write
  pull-requests: write

jobs:
  i18n:
    if: ${{ !startsWith(github.head_ref, 'translate/') }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }

      - name: Install Cursor CLI
        run: |
          curl https://cursor.com/install -fsS | bash
          echo "$HOME/.cursor/bin" >> $GITHUB_PATH

      - name: Configure git
        run: |
          git config user.name "Cursor Agent"
          git config user.email "cursoragent@cursor.com"

      - name: Translate
        env:
          CURSOR_API_KEY: ${{ secrets.CURSOR_API_KEY }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          agent -p "PR: ${{ github.event.pull_request.number }}.
          Detect i18n keys added/changed in diff.
          Fill ONLY missing locales. Never overwrite existing translations.
          Validate JSON. Push to translate/ branch.
          Post PR comment with compare link." \
          --force --model "gpt-5.2" --output-format=text
```

---

## Безпека та мережевий доступ

### Ізоляція

- Кожен Cloud Agent працює в **ізольованій VM** на AWS
- Код зберігається тільки на диску VM поки агент доступний
- Privacy Mode — код не використовується для тренування

### Секрети

- Шифрування at rest (KMS) та in transit
- Redacted секрети: скануються в комітах, редагуються в tool calls
- Не видимі нікому крім Cloud Agent user

### Мережевий доступ

3 режими:

| Режим | Поведінка |
|-------|-----------|
| **Allow all** | Без обмежень |
| **Default + allowlist** | Стандартні домени (npm, pip, Docker Hub тощо) + кастомний список |
| **Allowlist only** | Тільки ваші домени |

Налаштування: Dashboard → Cloud Agents → Security → Network Access Settings.

Enterprise може **заблокувати** налаштування для всієї команди (Lock Network Access Policy).

### Ризики

> ⚠️ Агент **автоматично виконує** terminal-команди (на відміну від foreground agent). Це створює ризик data exfiltration через prompt injection.

### Team Follow-ups

| Режим | Хто може |
|-------|----------|
| Disabled | Тільки creator |
| Service accounts only | + service accounts |
| All | Будь-хто в команді |

> ⚠️ Team follow-ups = lateral movement risk. Користувач може інструктувати агент з секретами іншого користувача.

---

## Ціноутворення

### Індивідуальні плани

| Plan | Ціна | Включений API usage |
|------|------|---------------------|
| **Pro** | $20/mo | $20 API agent usage |
| **Pro Plus** | $50/mo | $70 API agent usage |
| **Ultra** | $200/mo | $400 API agent usage |

Всі включають: unlimited tab completions, Cloud Agents, Bugbot.

### Типовий usage

| Тип користувача | Usage/mo |
|-----------------|----------|
| Daily Tab | < $20 |
| Limited Agent | ~$20 |
| Daily Agent | $60–100 |
| Power user | $200+ |

### Auto Mode pricing

| Тип | Ціна за 1M tokens |
|-----|-------------------|
| Input + Cache Write | $1.25 |
| Output | $6.00 |
| Cache Read | $0.25 |

### Max Mode

Розширений контекст. API rate + 20% upcharge.

### Cloud Agent

Чарджиться за API pricing обраної моделі. VM compute — безкоштовно поки що (буде тарифікуватись у майбутньому).

### Teams

- **Teams**: $40/user/mo
- **Enterprise**: Custom (SAML/OIDC SSO, SCIM, pooled usage, invoicing)

### Bugbot

- **Pro**: $40/mo (до 200 PR)
- **Teams**: $40/user/mo (unlimited)
- **Enterprise**: Custom

---

## Best Practices

### 1. Налаштуй середовище

Використовуй Cloud agent setup (cursor.com/onboard). Агент працює краще якщо середовище налаштоване правильно — як людина-розробник.

### 2. Забезпеч доступ

- **Секрети**: Secrets tab в dashboard
- **Egress**: Whitelist потрібні URL якщо є обмеження
- **Локальна тестованість**: Якщо людині важко тестувати локально — агенту теж буде важко

### 3. Використовуй Skills та agents.md

Агент = розумний, але low-context розробник. Дай йому контекст:
- `agents.md` — загальні інструкції (як запускати/дебажити сервіси)
- Skills — глибокі інструкції для конкретних задач

### 4. Дай інструменти

- MCP-сервери для доступу до зовнішніх систем
- Кастомні CLI для специфічних задач

### 5. Адаптуй інструменти

Створюй tools, які агент добре використовує:
- Cursor створив кастомний CLI для мікросервісів
- Моделі можуть забувати аргументи або відволікатися на noisy logs

---

## Порівняння підходів

| Підхід | Контроль | Складність | Вартість | Use case |
|--------|----------|-----------|----------|----------|
| **Bugbot** | Мінімальний | Нульова | $40/mo або free tier | Стандартний code review |
| **@cursor на PR** | Prompt-based | Низька | Cloud credits | Ad-hoc задачі на PR |
| **CLI в GitHub Actions** | Повний | Середня | API credits | Кастомні CI/CD пайплайни |
| **Cloud Agents API** | Програмний | Висока | Cloud credits | Оркестрація, автоматизація |
| **Slack @cursor** | Prompt-based | Низька | Cloud credits | Team collaboration |

### Коли що обирати

- **Просто code review** → Bugbot (zero-config)
- **Фікс CI + auto-review** → CLI в GitHub Actions
- **Оркестрація агентів з коду** → Cloud Agents API
- **Quick fixes на PR** → @cursor коментар
- **Team workflow** → Slack інтеграція

---

## Dashboard Settings

### Defaults

- **Default model** — модель без явного вибору
- **Default repository** — репо за замовчуванням
- **Base branch** — гілка для fork (default: main)

### Security

- **Display agent summary** — показувати diff images та code snippets
- **Display in external channels** — для Slack Connect
- **Team follow-ups** — дозволити follow-ups від тіммейтів

### Team Features

- **Long running agents** — дозволити/заборонити довгі агенти
- **Computer use** — дозволити/заборонити (тільки Enterprise)

---

## Slack інтеграція

### Setup

1. Dashboard → Integrations → Connect Slack
2. Install Cursor app → Slack workspace
3. Connect GitHub + enable usage-based pricing

### Використання

```
@Cursor fix the login bug                       ← auto repo selection
@Cursor in backend-api, fix the auth issue       ← specific repo
@Cursor with opus, fix the login bug             ← specific model
@Cursor branch=dev autopr=false Fix the bug      ← inline options
@Cursor settings                                 ← configure defaults
@Cursor list my agents                           ← running agents
@Cursor agent [prompt]                           ← force new agent in thread
```

### Routing Rules

Dashboard → Cloud Agents → Routing Rules:

```
"frontend" → acme/web-app
"mobile"   → acme/mobile-app
"api"      → acme/backend-services
```

---

## Джерела

### Офіційна документація
- [Cloud Agents overview](https://cursor.com/docs/cloud-agent)
- [Cloud Agents setup](https://cursor.com/docs/cloud-agent/setup)
- [Cloud Agents capabilities](https://cursor.com/docs/cloud-agent/capabilities)
- [Cloud Agents API](https://cursor.com/docs/cloud-agent/api/endpoints)
- [Cloud Agents security](https://cursor.com/docs/cloud-agent/security)
- [Cloud Agents settings](https://cursor.com/docs/cloud-agent/settings)
- [Cloud Agents network access](https://cursor.com/docs/cloud-agent/network-access)
- [Cloud Agents egress IPs](https://cursor.com/docs/cloud-agent/egress-ip-ranges)
- [CLI headless](https://cursor.com/docs/cli/headless)
- [CLI GitHub Actions](https://cursor.com/docs/cli/github-actions)
- [CLI authentication](https://cursor.com/docs/cli/reference/authentication)
- [CLI configuration](https://cursor.com/docs/cli/reference/configuration)
- [CLI permissions](https://cursor.com/docs/cli/reference/permissions)
- [Bugbot](https://cursor.com/docs/bugbot)
- [GitHub integration](https://cursor.com/docs/integrations/github)
- [Slack integration](https://cursor.com/docs/integrations/slack)
- [Agent Skills](https://cursor.com/docs/context/skills)
- [Subagents](https://cursor.com/docs/context/subagents)
- [Hooks](https://cursor.com/docs/agent/hooks)
- [Pricing](https://cursor.com/docs/account/pricing)

### Cookbook workflows
- [Code Review](https://cursor.com/docs/cli/cookbook/code-review)
- [Fix CI](https://cursor.com/docs/cli/cookbook/fix-ci)
- [Update Docs](https://cursor.com/docs/cli/cookbook/update-docs)
- [Secret Audit](https://cursor.com/docs/cli/cookbook/secret-audit)
- [Translate Keys](https://cursor.com/docs/cli/cookbook/translate-keys)

### Блог-пости
- [Third Era of AI Software Development](https://cursor.com/blog/third-era)
- [Bugbot Autofix](https://cursor.com/blog/bugbot-autofix)
- [Agent Computer Use](https://cursor.com/blog/agent-computer-use)
- [Long-Running Agents](https://cursor.com/blog/long-running-agents)
- [Self-Driving Codebases](https://cursor.com/blog/self-driving-codebases)

### Bugbot Rules Cookbook
- [Reviewing DB Migrations](https://cursor.com/docs/cookbook/bugbot-rules)

### Sitemap
- [All docs pages](https://cursor.com/llms.txt)