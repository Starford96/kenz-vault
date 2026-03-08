# Controlling Concurrency in Go — Worker Pool vs Semaphore

## Проблема: Unbounded Concurrency

Наївний код:

```go
for _, job := range jobs {
    go process(job)
}
```

Якщо `jobs = 100000`, то створиться:

```
100000 goroutines
↓
memory росте
↓
scheduler overhead
↓
downstream перевантажується
↓
latency росте
```

Production правило:

```
bounded concurrency
```

---

# Worker Pool

Worker pool — це патерн, де **кількість goroutines обмежена**, а задачі стоять у черзі.

## Модель

```
tasks
 ↓
queue
 ↓
workers (goroutines)
 ↓
execution
```

## Приклад

```go
jobs := make(chan Job)

for i := 0; i < 10; i++ {
    go worker(jobs)
}

for _, job := range jobList {
    jobs <- job
}
```

Worker:

```go
func worker(jobs <-chan Job) {
    for job := range jobs {
        process(job)
    }
}
```

## Що відбувається

```
workers = 10 goroutines
jobs = N задач
jobs чекають у черзі
```

## Інтуїція

```
1000 задач
10 працівників
працівники беруть задачі з черги
```

## Коли використовувати

Worker pool добре підходить для:

```
job pipelines
queue consumers
background processing
stream processing (Kafka, SQS)
```

---

# Semaphore Pattern

Semaphore використовується для **обмеження одночасного доступу до ресурсу**.

## Модель

```
goroutines
 ↓
semaphore (limit)
 ↓
execution
```

## Реалізація через channel

```go
sem := make(chan struct{}, 10)

for _, job := range jobs {
    sem <- struct{}{}

    go func(job Job) {
        defer func() { <-sem }()
        process(job)
    }(job)
}
```

## Що відбувається

```
1000 goroutines створюються
10 виконуються
990 чекають semaphore
```

Тобто:

```
goroutines waiting
```

---

# Ключова різниця

Worker Pool:

```
tasks waiting
workers fixed
```

Semaphore:

```
goroutines waiting
execution limited
```

---

# Memory Model

Worker pool:

```
goroutines = workers
memory stable
```

Semaphore:

```
goroutines = tasks
memory grows with tasks
```

---

# Mental Models

### Worker Pool

```
100 задач
10 працівників
задачі стоять у черзі
```

### Semaphore

```
100 людей
10 дверей
всередині може бути лише 10
```

---

# Practical Use Cases

### Worker Pool

```
Kafka consumers
job queues
background workers
batch pipelines
```

### Semaphore

```
HTTP fan-out
DB connection limiting
external API calls
file operations
```

---

# Production Pattern (комбінація)

У реальних системах часто використовують обидва підходи:

```
worker pool
↓
job processing
↓
semaphore
↓
external resource access
```

Приклад:

```
Kafka → workers → HTTP calls (limited by semaphore)
```

---

# Коротке правило

Якщо:

```
tasks unbounded / streaming
```

→ Worker Pool

Якщо:

```
fan-out operations
finite list of tasks
```

→ Semaphore

---

# Mental Checklist

Перед вибором патерну запитай:

1. Чи є **черга задач**?

→ Worker Pool

2. Чи треба **обмежити доступ до ресурсу**?

→ Semaphore
