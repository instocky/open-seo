---
status: Draft
last_updated: 2026-09-02
type: spike
---

# Spike: External SEO Dashboard поверх OpenSEO API

## Гипотеза

OpenSEO = SEO data engine / backend.
Custom Dashboard = specialist-oriented UI, который развивается независимо и общается с OpenSEO **только через HTTP API**, без импорта внутренних модулей.

Если вертикальный slice (`keyword metrics → Opportunity Map`) проходит — гипотеза технически подтверждена, и имеет смысл проектировать остальные модули. Если нет — фиксируем, какой минимальный API layer нужно добавить в OpenSEO.

## Что проверяем

1. Существует ли в OpenSEO стабильный HTTP API, достаточный для внешнего клиента:
   - `keyword`, `volume`, `keyword_difficulty`, `CPC`, `intent`, `trend`, `SERP`, `competitors`, `related_keywords`.
2. Можно ли прицепить к нему полностью независимый frontend (CORS, auth, versioning, стабильность response shape).
3. Достаточно ли данных для построения Opportunity Map (Volume × Difficulty с фильтрами по CPC/intent/trend).

## Текущее состояние API (предварительно)

Из `src/routes/api` присутствуют только `auth/`, `autumn/`, `ga4/`, `gsc/` — это **внутренние** роуты, не публичный SEO data API. Проверка на старте spike обязана подтвердить или опровергнуть наличие публичного keyword/serp/competitor API. Если его нет — spike включает в себя тонкий API adapter (Variant B из брифа).

## Vertical slice (минимальный)

```
[External SPA]  →  HTTP  →  [OpenSEO API]  →  DataForSEO
                                  ↓
                          keyword metrics
                                  ↓
                          Opportunity Map
```

Одна страница:
- input: `keyword | domain`
- output: таблица keyword metrics + Opportunity Map (scatter: Volume × KD, размер/цвет = CPC, facet = intent)

## Scope (что входит)

- read-only SEO data через HTTP
- минимальный Opportunity Map
- базовая фильтрация (volume / KD / intent / trend)
- документированный API contract (минимум — endpoint, request, response shape, auth, CORS)

## Out of scope (явно)

- billing / accounts / multi-tenancy
- write-операции (сохранение keywords/projects на стороне OpenSEO)
- plugin runtime / module system
- дизайн-система
- воспроизведение всего OpenSEO UI
- MCP / agent skills (это другой слой, не путать с frontend API)
- замена DataForSEO
- собственный crawler

## Критерии успеха

**Technical**
- external SPA подключается к OpenSEO через HTTP и получает данные без `import` внутренних модулей OpenSEO.

**Functional**
- keyword search возвращает keyword metrics: `volume`, `kd`, `cpc`, `intent`, `trend`.
- таблица + Opportunity Map рендерятся.
- работает минимум 1 фильтр (например, по volume или KD).

**Architectural**
- API контракт зафиксирован в `docs/API-external-dashboard.md` (или аналог) — endpoint, схема ответа, auth, CORS, versioning.
- явно зафиксировано, в какой из трёх веток (A: API готов / B: нужен тонкий adapter / C: API нет) мы оказались.

## Главный вопрос по итогу

> Достаточно ли API OpenSEO стабильно и полно, чтобы использовать его как SEO backend для собственного продукта?

- Да → проектировать остальные модули (SERP Explorer, Competitors, …).
- Нет → сформулировать минимальный API layer, который нужно добавить в OpenSEO, и оценить стоимость.

## Деливераблы spike

1. Работающий vertical slice в этой ветке.
2. `docs/SPIKE-RESULT-external-seo-dashboard.md` с:
   - фактическим списком endpoints, которые удалось/не удалось использовать;
   - фиксацией A/B/C;
   - решением по принципиальному вопросу (см. выше);
   - если B или C — минимальный proposal на API adapter.
3. Один-два скриншота / curl-трейсы, подтверждающие работу.

## Когда spike считается завершённым

Когда есть:
- работающий демо-проход `keyword → metrics → scatter chart`;
- зафиксированный API contract;
- явный ответ на главный вопрос и (если нужно) proposal на adapter.

Не раньше.
