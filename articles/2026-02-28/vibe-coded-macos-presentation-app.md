---
title: Я написав macOS-застосунок для презентацій у стилі «вайб-кодингу»
tags:
  - ai-coding
  - swift
  - macos
  - vibe-coding
  - claude-code
created: 2026-02-28
---

# Я написав macOS-застосунок для презентацій у стилі «вайб-кодингу»

> **Оригінал:** https://simonwillison.net/2026/Feb/25/present/
> **Джерело:** Simon Willison
> **Дата:** 2026-02-25
> **Оцінка:** 8/10
> **Дайджест:** [[articles/2026-02-28/digest|Дайджест 2026-02-28]]

---

25 лютого 2026

Simon Willison виступав на Social Science FOO Camp у Mountain View. Він узяв слот для доповіді «Стан LLM, лютий 2026» і напередодні написав власний macOS-застосунок для презентації — за ~45 хвилин через Claude Code.

У доповіді два «гіміки»: аналіз даних про сезон розмноження какапо через coding agent, і фінальне розкриття — вся презентація велася через новий застосунок, написаний вайб-кодингом.

## Present.app

Написаний на Swift і SwiftUI, важить 355 КБ (76 КБ стиснутий).

Ідея: зазвичай Simon веде презентації через послідовність вебсторінок у браузері. Недолік — якщо браузер крашнеться, вся презентація пропала.

Початковий промпт:

```
Build a SwiftUI app for giving presentations where every slide is a URL.
The app starts as a window with a webview on the right and a UI on the left
for adding, removing and reordering the sequence of URLs. Then you click Play
in a menu and the app goes full screen and the left and right keys switch between URLs
```

Це породило план, Claude Code реалізував.

У Present доповідь — впорядкована послідовність URL-адрес. Cmd+Shift+P — повноекранний режим. Стрілки ліво/право — перемикають слайди. Escape — вийти. **Автоматичне збереження при кожній зміні** — якщо застосунок крашнеться, перезапустіть і все відновиться.

## Дистанційне керування з телефону

Наступний промпт:

```
Add a web server which listens on 0.0.0.0:9123 — the web server serves a single
mobile-friendly page with prominent left and right buttons — clicking those buttons
switches the slide left and right — there is also a button to start presentation mode
or stop depending on the mode it is in.
```

Tailscale на ноутбуці і телефоні — керування презентацією через `http://100.122.231.116:9123/` з будь-якої точки світу.

## Вивчення коду

Claude Code реалізував вебсервер **через raw сокети без жодної бібліотеки**:

```swift
private func route(_ raw: String) -> String {
    let firstLine = raw.components(separatedBy: "\r\n").first ?? ""
    let parts = firstLine.split(separator: " ")
    let path = parts.count >= 2 ? String(parts[1]) : "/"

    switch path {
    case "/next":
        state?.goToNext()
        return jsonResponse("ok")
    case "/prev":
        state?.goToPrevious()
        return jsonResponse("ok")
    }
}
```

GET-запити для зміни стану — CSRF-вразливість, але для цього застосунку не критично.

## Висновки

- Swift — правильний вибір для повноекранного застосунку з вебконтентом і мережевим керуванням
- Код простий і прямолінійний
- Xcode жодного разу не відкривав
- Хтось, хто знає Swift, написав би краще — але це ілюстрація розширення горизонтів через AI-агентів