---
status: Draft
last_updated: 2026-09-02
type: spike
related: docs/SPIKE-external-seo-dashboard.md
---

# Architecture: External SEO Dashboard (spike)

Документ фиксирует архитектуру **вертикального среза spike**. Не описывает целевую систему.

## 1. Граница ответственности

```
┌──────────────────────────┐         ┌──────────────────────────┐
│  External Dashboard SPA  │  HTTP   │       OpenSEO            │
│  (отдельный origin /     │ ──────► │  (Cloudflare Worker)     │
│   отдельная репа)        │         │                          │
│                          │         │  existing internal       │
│  HTML + Tailwind +       │         │  modules (read-only)     │
│  Alpine.js + Chart.js    │         │           │              │
│  (всё через CDN)         │         │           ▼              │
└──────────────────────────┘         │      DataForSEO          │
                                     └──────────────────────────┘
```

**Жёсткое правило:** в коде Dashboard **запрещён** `import` любых модулей из `src/` OpenSEO. Связь — только HTTP. Если правило нарушено — spike провален вне зависимости от того, работает ли UI.

## 2. Frontend stack (зафиксировано)

| Слой         | Решение        | Почему                                  |
| ------------ | -------------- | --------------------------------------- |
| Layout       | Tailwind (CDN) | zero build, итеративно                 |
| State        | Alpine.js      | filters / form / table без SPA          |
| Chart        | Chart.js (CDN) | scatter, tooltips, легенда из коробки   |
| HTTP         | `fetch`        | нативно, без axios / tanstack-query     |
| Сборка       | нет            | один `index.html`                       |
| Размещение   | отдельная репа | чище изоляция, не тянет pnpm-workspace  |

**Альтернатива рассмотрена и отклонена:** React + Vite + recharts — перебор для spike, отложено до stage «после подтверждения гипотезы».

## 3. API contract (целевой для spike)

Минимальный набор, который должен закрыть vertical slice:

```
GET /api/v1/keywords/metrics
  ?q=<keyword|domain>
  &location=<code>     // optional, default из конфига OpenSEO
  &language=<code>     // optional, default из конфига OpenSEO
  &limit=<int>         // optional, default 50

Response 200:
{
  "query": "...",
  "location": 2840,
  "language": "en",
  "items": [
    {
      "keyword": "...",
      "volume": 12300,
      "keyword_difficulty": 42,
      "cpc": 1.23,
      "intent": "commercial",
      "trend": [12, 15, 19, 22, 28, 31, 30, 28]   // 8 точек или сколько отдаёт DFS
    },
    ...
  ]
}
```

**Один endpoint закрывает весь vertical slice.** Дополнительные модули (SERP, competitors) — вне spike.

**Версионирование:** префикс `/v1/`. Если в OpenSEO уже есть роуты без версии и они подходят — фиксируем в `SPIKE-RESULT`, версию добавляем отдельным шагом.

## 4. Cross-cutting constraints (проверить в день 1)

| Constraint        | Что проверяем                                            | Если нет                                  |
| ----------------- | -------------------------------------------------------- | ----------------------------------------- |
| CORS              | `Access-Control-Allow-Origin` на нужном endpoint         | spike → Variant B: открыть CORS / proxy  |
| Auth              | требуется ли токен, какая схема (cookie / bearer / API)  | spike: разрешена простая bearer-апроксим. |
| Rate limiting     | как клиент понимает 429                                  | retry с backoff в Alpine                  |
| Stable schema     | поля приходят стабильно, нет snake_case / camelCase микса | spike: документируем и пишем adapter      |
| HTTPS             | обязателен для прод                                       | dev: можно http localhost                 |

## 5. Data flow (один сценарий)

```
User types "running shoes" in input
        │
        ▼
Alpine: search() 
        │  fetch(/api/v1/keywords/metrics?q=running+shoes)
        ▼
OpenSEO: validate query, call DataForSEO
        │  map DFS response → our contract
        ▼
Dashboard: render Table + Chart.js scatter
        │  filters (volume / KD / intent) reactive через Alpine
        ▼
User видит Opportunity Map
```

## 6. Что НЕ в архитектуре spike (явно)

- plugin runtime / module loader
- shared design system / компонентная библиотека
- типизированный API client (генерируется из схемы — отдельная задача)
- SSR / SSG
- state persistence (localStorage опционально, не блокер)
- i18n
- analytics / tracking
- deployment pipeline (для spike достаточно GitHub Pages / статический хостинг)

## 7. Definition of "spike passes architectural bar"

Готово, когда одновременно:

- [ ] в `package.json` / lockfile Dashboard нет ни одной строки `openseo/*` (или пакетов, импортирующих внутренние модули OpenSEO)
- [ ] Dashboard работает на отдельном origin, без локальных хаков
- [ ] API contract зафиксирован в spike-result (хотя бы один рабочий endpoint)
- [ ] CORS / auth / rate-limit поведение задокументировано
- [ ] failure modes (network error, 4xx, 5xx, empty result) обработаны в UI

Если хотя бы один пункт не закрыт — spike не passed, переход к проектированию модулей не начинаем.
