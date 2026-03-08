# Go Concurrency — Scheduler & Goroutines Notes

## 1. Goroutine

**Goroutine** — це легка одиниця виконання, якою керує **Go runtime**, а не операційна система.

Запуск:

```go
go processEvent(e)
```

Це означає:

```
runtime створює goroutine
↓
scheduler планує її виконання
```

### Чому goroutines дешеві

| Thread (OS)  | Goroutine            |
| ------------ | -------------------- |
| ~1 MB stack  | ~2 KB stack          |
| керується OS | керується Go runtime |

Stack goroutine **динамічно росте і зменшується**.

Тому можна запускати **десятки тисяч goroutines**.

---

## 2. Go Scheduler

**Scheduler** — це механізм runtime, який вирішує:

* яка goroutine виконується
* яка чекає
* коли переключити виконання

### Основна задача scheduler

```
багато goroutines
↓
менше потоків
↓
CPU
```

---

## 3. Основні сутності scheduler

```
G — goroutine
M — machine (OS thread)
P — processor
```

### Інтуїтивна модель

```
goroutine = задача
thread = робітник
P = дозвіл виконувати Go код
scheduler = диспетчер
```

### Схема виконання

```
goroutines
   ↓
run queue
   ↓
scheduler
   ↓
threads
   ↓
CPU
```

---

## 4. GOMAXPROCS

`GOMAXPROCS` визначає:

```
максимальну кількість goroutines,
які можуть реально виконуватись одночасно
```

Наприклад:

```
GOMAXPROCS = 8
```

→ максимум **8 goroutines паралельно на CPU**.

---

## 5. Concurrency vs Parallelism

Це дві різні концепції, які часто плутають.

### Concurrency

**Concurrency** означає, що система може працювати з багатьма задачами, перемикаючись між ними.

```
задачі чергуються
```

CPU швидко перемикається між goroutines, створюючи ілюзію одночасної роботи.

Приклад:

```go
func main() {
    go taskA()
    go taskB()
    go taskC()

    time.Sleep(time.Second)
}
```

Якщо:

```
GOMAXPROCS = 1
```

тоді:

```
CPU → taskA
CPU → taskB
CPU → taskC
CPU → taskA
...
```

Задачі виконуються **по черзі**, але дуже швидко перемикаються.

---

### Parallelism

**Parallelism** означає, що задачі реально виконуються одночасно на різних ядрах CPU.

Наприклад:

```
CPU core 1 → taskA
CPU core 2 → taskB
CPU core 3 → taskC
```

Це можливо тільки якщо:

```
GOMAXPROCS > 1
```

---

### Як це виглядає у Go

Go спочатку забезпечує **concurrency**, а **parallelism** з'являється автоматично, коли є кілька CPU ядер.

```
many goroutines
↓
scheduler
↓
threads
↓
CPU cores
```

---

### Приклад

```go
for i := 0; i < 1000; i++ {
    go work()
}
```

Ми створюємо **1000 concurrent задач**, але реально одночасно виконуються тільки:

```
GOMAXPROCS goroutines
```

інші чекають у scheduler.

---

### Mental model

Concurrency:

```
один кухар
багато замовлень
він перемикається між ними
```

Parallelism:

```
багато кухарів
кожен готує свою страву
одночасно
```

---

### Важливий висновок

Go concurrency модель дозволяє:

* писати код так, ніби задач багато
* runtime сам розподіляє їх по CPU

Тобто розробник думає про:

```
concurrency
```

а runtime займається:

```
scheduling + parallel execution
```

---

## 6. Що відбувається коли goroutine блокується

Приклад:

```go
resp, _ := http.Get(url)
```

goroutine чекає I/O.

Scheduler робить:

```
goroutine блокується
↓
thread звільняється
↓
scheduler запускає іншу goroutine
```

Це робить Go ефективним для:

* HTTP серверів
* network services
* I/O workloads

---

## 7. Runtime Preemption

**Preemption** — це механізм, який дозволяє Go runtime **перервати виконання goroutine**, навіть якщо вона сама не блокується.

Це потрібно для того, щоб **одна goroutine не могла захопити CPU надовго**.

### Чому це важливо

Без preemption могла б виникнути ситуація:

```
G1 виконує довгий цикл
↓
вона ніколи не блокується
↓
інші goroutines чекають
```

Тобто одна goroutine могла б **монополізувати CPU**.

### Що означає "перервати goroutine"

Переривання **не означає знищення goroutine**. Runtime просто:

1. зупиняє її виконання
2. зберігає її стан
3. повертає її у чергу scheduler

Пізніше scheduler може **продовжити виконання з того ж місця**.

### Де зберігається стан goroutine

Стан goroutine зберігається у структурі runtime, яка називається **G (goroutine descriptor)**.

Вона містить:

* stack goroutine
* program counter (де саме виконання зупинилось)
* register state
* інформацію про статус goroutine

Спрощено:

```
G struct
 ├─ stack
 ├─ registers
 ├─ program counter
 └─ scheduler metadata
```

### Як виглядає переривання

Коли runtime вирішує зробити preemption:

```
CPU виконує G1
↓
runtime ставить preemption flag
↓
G1 призупиняється
↓
стан G1 лишається у її stack + runtime структурі
↓
scheduler запускає іншу goroutine
```

Пізніше G1 може бути знову запланована:

```
run queue
↓
scheduler
↓
thread
↓
CPU
```

І виконання продовжується **з того ж місця**.

### Приклад

```go
func busy() {
    for {
        // довга робота
    }
}

func main() {
    go busy()
    go worker()
}
```

Без preemption `busy()` могла б **ніколи не дати виконатись `worker()`**.

З preemption runtime може примусово переключити виконання.

### Як це працює (спрощено)

Go runtime періодично перевіряє goroutines у так званих **safe points** (наприклад під час функціональних викликів або runtime перевірок).

Якщо goroutine виконується занадто довго:

```
runtime ставить preemption flag
↓
goroutine призупиняється
↓
scheduler запускає іншу
```

### До Go 1.14

До Go 1.14 preemption працював гірше, бо переключення часто відбувалося тільки коли goroutine:

* блокувалась
* викликала syscall
* чекала channel

Після Go 1.14 з’явився **асинхронний preemption**, і scheduler став значно справедливішим.

### Mental model

```
preemption = "зупинити goroutine, зберегти її стан і дати CPU іншій"
```

Це допомагає:

* уникати starvation
* робити scheduler більш справедливим
* забезпечувати плавну concurrency

---

## 8. Небезпека необмежених goroutines

Приклад поганого коду:

```go
for i := 0; i < 1_000_000; i++ {
    go work()
}
```

### Що відбувається

```
1M goroutines
↓
memory usage росте
↓
scheduler overhead
↓
latency деградує
```

---

## 9. Причинно-наслідковий ланцюжок (slow external API)

Ситуація:

```
HTTP server
↓
go callExternalAPI()
↓
external API slow
```

### Ланцюжок проблем

```
slow API
↓
goroutines живуть довше
↓
goroutines накопичуються
↓
росте пам’ять
↓
runtime overhead
↓
росте latency
```

---

## 10. Production правило

❌ Погано

```
unbounded goroutines
```

✔ Добре

```
bounded concurrency
```

Наприклад:

* worker pools
* semaphore
* rate limiting

---

## Короткий mental model

```
goroutine = lightweight task
scheduler = dispatcher
threads = workers
CPU = execution
```

---

## Головний висновок

Повільний I/O не блокує весь процес, але:

```
подовжує життя goroutines
↓
goroutines накопичуються
↓
росте пам’ять і latency
```

Тому **контроль concurrency критично важливий** у production Go сервісах.
