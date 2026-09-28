# Домашнее задание № 4 · План запроса и индексы

## Цель

Научиться читать план выполнения запроса (`explain`), выявлять узкие места (COLLSCAN) и подбирать составные индексы по правилу **E–S–R** (Equality → Sort → Range).

Работа выполнялась на базе `logs`, коллекция `events` — **1200 документов**.

---

## Стенд

```bash
git clone https://github.com/MaximBytecamp/mongodb-practice.git
cd mongodb-practice
docker compose up -d
docker compose run --rm reset   # сброс к исходному состоянию
docker compose exec mongo mongosh logs
```

---

## Что такое план выполнения

`explain("executionStats")` возвращает документ с тремя ключевыми числами:

| Поле | Смысл |
|---|---|
| `nReturned` | Сколько документов возвращено клиенту |
| `totalDocsExamined` | Сколько документов сервер прочитал для этого |
| `totalKeysExamined` | Сколько записей индекса просмотрено |

Когда `totalDocsExamined` равно числу документов в коллекции — сервер прочитал всё (шаг **COLLSCAN**). После создания правильного индекса появляется **IXSCAN**, и `totalDocsExamined` приближается к `nReturned`.

---

## Задача 1 — Медленные ошибки payments, свежие сверху

```js
db.events.find({
  service: "payments",
  level: "error",
  duration_ms: { $gt: 500 }
}).sort({ ts: -1 })
```

### 1.1 Без индекса — `plan-1.json`

| Метрика | Значение |
|---|---|
| `nReturned` | **33** |
| `totalDocsExamined` | **1200** |
| `totalKeysExamined` | **0** |
| Шаги | `SORT ← COLLSCAN` |

**Вывод**: сервер прочитал все 1200 документов, чтобы вернуть 33. Коэффициент эффективности — 1 к 36. Шаг COLLSCAN — полный перебор коллекции, шаг SORT — сортировка в памяти после фильтрации.

---

## Задача 2 — Индекс по правилу ESR

Разбор запроса по правилу **E–S–R**:
- **E** (Equality): `service = "payments"`, `level = "error"` → поля точного совпадения
- **S** (Sort): `ts: -1` → поле сортировки
- **R** (Range): `duration_ms: { $gt: 500 }` → диапазонный фильтр

Оптимальный порядок полей в индексе: `{ service: 1, level: 1, ts: -1, duration_ms: 1 }`

```js
db.events.createIndex({ service: 1, level: 1, ts: -1, duration_ms: 1 })
```

### 2.1 С индексом ESR — `plan-2.json`

| Метрика | Значение |
|---|---|
| `nReturned` | **33** |
| `totalDocsExamined` | **33** |
| `totalKeysExamined` | **43** |
| Шаги | `FETCH ← IXSCAN` |
| Индекс | `service_1_level_1_ts_-1_duration_ms_1` |

**Вывод**: `totalDocsExamined` = `nReturned` = 33 — сервер читает ровно нужные документы. Сортировка не нужна в памяти, т.к. индекс уже упорядочен по `ts`. Ключей просмотрено 43 (чуть больше из-за структуры B-дерева при Range). Это **оптимальный** план.

```js
db.events.dropIndex({ service: 1, level: 1, ts: -1, duration_ms: 1 })
```

---

## Задача 3 — Порядок полей имеет значение

Сравниваем три варианта индекса с теми же полями:

| Файл | Индекс | nReturned | DocsExamined | KeysExamined | Шаги |
|---|---|---|---|---|---|
| plan-1.json | *(нет)* | 33 | 1200 | 0 | SORT ← COLLSCAN |
| plan-2.json | `{service,level,ts,duration_ms}` | 33 | 33 | 43 | FETCH ← IXSCAN |
| plan-3a.json | `{duration_ms,service,level,ts}` | 33 | 33 | 378 | FETCH ← SORT ← IXSCAN |
| plan-3b.json | `{service,level,duration_ms,ts}` | 33 | 33 | 33 | FETCH ← SORT ← IXSCAN |

### Анализ разницы

**plan-3a** (`{ duration_ms, service, level, ts }`):
- `duration_ms` идёт первым — Range-условие. Индекс не может сузить набор до точного совпадения `service`/`level`, сканирование идёт по большому диапазону (378 ключей!)
- После IXSCAN нужна дополнительная SORT в памяти для `ts`
- ❌ Нарушен порядок E–S–R: Range-поле поставлено первым

**plan-3b** (`{ service, level, duration_ms, ts }`):
- `service` и `level` идут первыми (Equality) — отлично
- Затем `duration_ms` (Range), потом `ts` (Sort) — E–R–S вместо E–S–R
- Индекс читает только 33 ключа, но сортировка в памяти всё равно нужна
- ⚠️ Лучше, чем 3a, но хуже, чем plan-2

**plan-2** (`{ service, level, ts, duration_ms }`):
- E–S–R: Equality (`service, level`) → Sort (`ts`) → Range (`duration_ms`)
- Индекс упорядочен так, что и фильтр, и сортировка покрываются без дополнительных шагов
- Читает 43 ключа без SORT в памяти
- ✅ Оптимальный порядок по правилу ESR

**Вывод**: Порядок полей в составном индексе критически важен. Правило E–S–R позволяет MongoDB использовать индекс и для фильтрации, и для сортировки без дополнительных операций.

---

## Задача 4 — Самостоятельный подбор индекса

```js
db.events.find({ level: "error" }).sort({ duration_ms: -1 }).limit(10)
```

Разбор по ESR:
- **E** (Equality): `level = "error"`
- **S** (Sort): `duration_ms: -1`
- Range нет

Оптимальный индекс: `{ level: 1, duration_ms: -1 }`

### 4.1 Без индекса — `plan-4-bez.json`

| Метрика | Значение |
|---|---|
| `nReturned` | **10** |
| `totalDocsExamined` | **1200** |
| `totalKeysExamined` | **0** |
| Шаги | `SORT ← COLLSCAN` |

### 4.2 С индексом `{ level, duration_ms }` — `plan-4.json`

```js
db.events.createIndex({ level: 1, duration_ms: -1 })
```

| Метрика | Значение |
|---|---|
| `nReturned` | **10** |
| `totalDocsExamined` | **10** |
| `totalKeysExamined` | **10** |
| Шаги | `LIMIT ← FETCH ← IXSCAN` |
| Индекс | `level_1_duration_ms_-1` |

**Вывод**: благодаря индексу MongoDB читает ровно 10 документов. Шаг LIMIT применяется сразу после IXSCAN, не нужно читать все 180 ошибок. Идеальный план — `nReturned = totalDocsExamined = totalKeysExamined = 10`.

---

## Сводная таблица

| Файл | Запрос | Индекс | nReturned | DocsExamined | KeysExamined | Шаги |
|---|---|---|---|---|---|---|
| plan-1.json | Q1 | *нет* | 33 | 1200 | 0 | SORT ← COLLSCAN |
| plan-2.json | Q1 | `{service,level,ts,duration_ms}` | 33 | 33 | 43 | FETCH ← IXSCAN |
| plan-3a.json | Q1 | `{duration_ms,service,level,ts}` | 33 | 33 | 378 | FETCH ← SORT ← IXSCAN |
| plan-3b.json | Q1 | `{service,level,duration_ms,ts}` | 33 | 33 | 33 | FETCH ← SORT ← IXSCAN |
| plan-4-bez.json | Q2 | *нет* | 10 | 1200 | 0 | SORT ← COLLSCAN |
| plan-4.json | Q2 | `{level,duration_ms}` | 10 | 10 | 10 | LIMIT ← FETCH ← IXSCAN |

---

## Выводы

1. **COLLSCAN** = сервер читает всю коллекцию. На 1200 документах это незаметно, но на миллионах — критично.
2. **Правило ESR**: Equality → Sort → Range — золотое правило для составных индексов.
3. **Порядок полей** в индексе определяет эффективность: неправильный порядок может увеличить количество просмотренных ключей в 10× и принудить к дополнительной SORT в памяти.
4. **`totalDocsExamined ≈ nReturned`** — признак хорошего индекса. Если они равны — индекс работает идеально.

---

## Файлы

```
lesson_04/
├── README.md         ← этот файл
├── plan-1.json       ← Q1 без индекса (COLLSCAN)
├── plan-2.json       ← Q1 с ESR-индексом (оптимально)
├── plan-3a.json      ← Q1: {duration_ms,service,level,ts} (Range первый)
├── plan-3b.json      ← Q1: {service,level,duration_ms,ts} (E-R-S)
├── plan-4-bez.json   ← Q2 без индекса (COLLSCAN)
└── plan-4.json       ← Q2 с {level,duration_ms} (оптимально)
```
