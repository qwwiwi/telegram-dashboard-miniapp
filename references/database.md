# База данных для дашборда

Для личного дашборда хватает SQLite: это один файл, отдельный сервер базы не нужен, резервная копия – копия файла. Цифр немного: 50 метрик за год по 10 разрезов – это сотни тысяч строк, для SQLite мелочь.

Идея в одной фразе: **в базе лежат сырые числа по дням, а проценты, доли и «стоимость чего-то» считаются при чтении.**

## Таблицы

Эти таблицы создаёт `app/db.py` из starter.md. Если меняете схему, меняйте и там, и здесь.

```sql
-- Who may open the dashboard. A removed person gets revoked_at, the row stays (history of who had access).
CREATE TABLE IF NOT EXISTS allowed_users (
  tg_id INTEGER PRIMARY KEY,
  name TEXT NOT NULL DEFAULT '',
  added_at TEXT NOT NULL DEFAULT (datetime('now')),
  revoked_at TEXT
);

-- What each metric is. Filled from app/metrics.py at every start; derived ratios are NOT stored here.
CREATE TABLE IF NOT EXISTS metric_defs (
  key TEXT PRIMARY KEY,                 -- 'department.name', e.g. 'sales.revenue'
  department TEXT NOT NULL,
  title TEXT NOT NULL,
  unit TEXT NOT NULL CHECK (unit IN ('count', 'money', 'percent')),
  kind TEXT NOT NULL CHECK (kind IN ('sum', 'balance'))   -- sum: add the days up; balance: take the last day
);

-- The numbers: one row per metric, day and breakdown value.
CREATE TABLE IF NOT EXISTS metric_values (
  key TEXT NOT NULL REFERENCES metric_defs(key),
  day TEXT NOT NULL,                    -- 'YYYY-MM-DD', a calendar day in the business timezone
  dim TEXT NOT NULL DEFAULT '',         -- '' = the total; otherwise a breakdown value, e.g. a channel
  value REAL NOT NULL,
  source TEXT NOT NULL,                 -- which collector wrote it ('crm', 'ads', 'demo')
  updated_at TEXT NOT NULL DEFAULT (datetime('now')),
  PRIMARY KEY (key, day, dim)
);
CREATE INDEX IF NOT EXISTS metric_values_day ON metric_values(day);

-- Keys of the collectors. Only the sha256 of a key is stored.
CREATE TABLE IF NOT EXISTS ingest_keys (
  id INTEGER PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,
  key_hash TEXT NOT NULL UNIQUE,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  revoked_at TEXT,
  last_used_at TEXT
);

-- Who pushed what and when (to answer «why is this day empty?»).
CREATE TABLE IF NOT EXISTS ingest_log (
  id INTEGER PRIMARY KEY,
  at TEXT NOT NULL DEFAULT (datetime('now')),
  key_name TEXT NOT NULL,
  source TEXT NOT NULL,
  rows INTEGER NOT NULL
);
```

Сырые события нужны не всегда. Заведите таблицу, когда числа за день надо уметь пересчитать или проверить по отдельным записям (сделка, платёж, заявка), или когда границу дня могут сдвинуть.

```sql
-- Raw events: one row per real-world record. Do not store personal data you do not need.
CREATE TABLE IF NOT EXISTS events (
  id INTEGER PRIMARY KEY,
  source TEXT NOT NULL,
  external_id TEXT NOT NULL,            -- the id in the source system: makes a re-push a no-op
  occurred_at TEXT NOT NULL,            -- UTC, ISO 8601: '2026-01-15T09:30:00Z'
  kind TEXT NOT NULL,                   -- 'lead', 'deal_won', 'payment'
  amount REAL,
  payload TEXT NOT NULL DEFAULT '{}',   -- extra JSON, no secrets
  received_at TEXT NOT NULL DEFAULT (datetime('now')),
  UNIQUE (source, external_id)
);
CREATE INDEX IF NOT EXISTS events_kind_time ON events(kind, occurred_at);
```

Из событий сборщик (или ночная задача) считает дневные числа и кладёт в `metric_values`. Дневные числа остаются тем, что читает дашборд: быстро и просто.

## Запись: вставить или заменить

Любая запись числа – это «вставить, а при совпадении ключа заменить». Тогда повторная отправка безвредна, а поздние поправки источника просто обновляют день.

```sql
INSERT INTO metric_values(key, day, dim, value, source) VALUES (?, ?, ?, ?, ?)
ON CONFLICT(key, day, dim) DO UPDATE SET value = excluded.value, source = excluded.source,
                                         updated_at = datetime('now');
```

То же для событий: `INSERT ... ON CONFLICT(source, external_id) DO UPDATE SET amount = excluded.amount, kind = excluded.kind`.

## Четыре запроса, которые нужны почти всегда

Параметры `:today`, `:yesterday`, `:from`, `:to`, `:key` подставляет приложение. «Сегодня» берут в часовом поясе бизнеса, а не из `date('now')`: оно даёт UTC.

```sql
-- 1. Today vs yesterday, every additive metric (today is a partial day: show it as «so far»)
SELECT d.department, d.title,
       SUM(CASE WHEN v.day = :today THEN v.value END) AS today,
       SUM(CASE WHEN v.day = :yesterday THEN v.value END) AS yesterday
FROM metric_values v
JOIN metric_defs d ON d.key = v.key
WHERE v.dim = '' AND d.kind = 'sum' AND v.day IN (:today, :yesterday)
GROUP BY d.key
ORDER BY d.department, d.key;

-- 2. Last 30 days of one metric (days without a row are gaps: fill them with null in code, never with 0)
SELECT day, value
FROM metric_values
WHERE key = :key AND dim = '' AND day BETWEEN :from AND :to
ORDER BY day;

-- 3. Month totals per department (additive metrics only)
SELECT substr(v.day, 1, 7) AS month, d.department, d.title, SUM(v.value) AS total
FROM metric_values v
JOIN metric_defs d ON d.key = v.key
WHERE v.dim = '' AND d.kind = 'sum'
GROUP BY month, d.key
ORDER BY month DESC, d.department, d.key;

-- 4. Cost per lead per month: a ratio of TOTALS, computed on read (NULLIF avoids division by zero)
SELECT substr(day, 1, 7) AS month,
       SUM(CASE WHEN key = 'marketing.spend' THEN value END)
         / NULLIF(SUM(CASE WHEN key = 'marketing.leads' THEN value END), 0) AS cost_per_lead
FROM metric_values
WHERE dim = '' AND key IN ('marketing.spend', 'marketing.leads')
GROUP BY month
ORDER BY month DESC;
```

Остаток денег (`kind = 'balance'`) за период – это значение последнего дня периода, поэтому в запросах 1 и 3 он не участвует. Читать его надо так: `SELECT value FROM metric_values WHERE key = 'finance.cash_balance' AND dim = '' AND day <= :to ORDER BY day DESC LIMIT 1`.

## Правила

1. **Одна строка на метрику, день и разрез.** Это первичный ключ, а не договорённость.
2. **Запись идемпотентна:** «вставить или заменить». Сборщик можно запускать сколько угодно раз.
3. **Хранить сырые числа, считать при чтении.** Хранить «стоимость заявки» нельзя: при исправлении расходов или заявок она тихо разойдётся. Доля считается от итогов (сумма расходов ÷ сумма заявок), а не как среднее дневных долей.
4. **Нет данных – нет строки.** Ноль значит «было ноль». Если источник не ответил, строку не пишут, и дашборд показывает «нет данных».
5. **Время и дни.** Моменты времени (`occurred_at`, `received_at`) – всегда в UTC. Поле `day` – календарная дата в часовом поясе бизнеса, он задан в настройках один раз. Дневные итоги нельзя пересчитать в другой пояс, поэтому поясов не смешивают, а если очень нужно, итоги пересобирают из сырых событий. Показывают даты в том же поясе.
6. **Сегодня – неполный день.** Сравнения и «за период» делают по законченным дням.
7. **Валюты не смешивать.** Разные валюты – разные метрики (`finance.income_usd`), не пересчитываются молча.
8. **Деньги в `REAL`** подходят для дашборда итогов. Если нужна сверка до копейки, храните целые числа в минимальных единицах (копейки, центы).
9. **Определения метрик живут в коде** (`app/metrics.py`) и синхронизируются в `metric_defs` при старте. Вручную в базе их не правят.
10. **Ключи хранят хешем, доступ отзывают меткой.** Строки `ingest_keys` и `allowed_users` не удаляют.
11. **Изменение схемы:** только добавление колонок и таблиц, выполняется при старте и безопасно при повторе (`CREATE ... IF NOT EXISTS`, `ALTER TABLE ... ADD COLUMN` после проверки, что колонки нет). Перед правкой – копия файла.
12. **Резервная копия:** раз в сутки `sqlite3 /var/lib/dashboard/dashboard.db ".backup '/путь/к/копии.db'"` (безопасно при работающем приложении), копию увозят с сервера. Сырые события старше года можно удалять, дневные числа хранят всегда.

## Когда переходить на Postgres

Не раньше, чем что-то из этого стало правдой:
- пишут одновременно несколько программ с разных машин, а не один сервер;
- к базе нужен доступ у нескольких людей или сервисов напрямую;
- файл вырос до нескольких гигабайт или запросы заметно тормозят;
- нужны управляемые резервные копии и восстановление на момент времени.

Полная «чистая» схема Postgres выглядела бы так: `INTEGER PRIMARY KEY` → `BIGINT GENERATED ALWAYS AS IDENTITY`, `datetime('now')` → `now()`, `day TEXT` → `DATE`, `value REAL` → `NUMERIC(18,2)` для денег, `occurred_at TEXT` → `TIMESTAMPTZ`. Конструкция `INSERT ... ON CONFLICT ... DO UPDATE` в Postgres записывается так же. Запросы выше работают после замены `substr(day, 1, 7)` на `to_char(day, 'YYYY-MM')`.

Каркас из `starter.md` работает на Postgres и без такой переделки: те же таблицы и колонки, тот же SQL, и четыре запроса выше выполняются как есть (проверено на PostgreSQL 17). Дни и время остаются текстом в том же формате, поэтому `substr(day, 1, 7)` тоже работает. Различаются только типы. Код, который это скрывает, и пошаговый переход – в `postgres.md`.

| SQLite | PostgreSQL в каркасе | Зачем |
|---|---|---|
| `INTEGER PRIMARY KEY` (id) | `BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY` | автонумерация |
| `tg_id INTEGER` | `BIGINT` | Telegram-id бывают больше 4 байт |
| `REAL` | `DOUBLE PRECISION` | `REAL` в Postgres – 4 байта и теряет точность |
| `TEXT` для дня и времени | `TEXT` в том же формате | запросы не меняются |
| `datetime('now')` | `to_char(now() AT TIME ZONE 'UTC', 'YYYY-MM-DD HH24:MI:SS')` | тот же текст, UTC |
| `?` | `%s` | другой стиль параметров |

Резервная копия Postgres – `pg_dump` по таймеру и проверка восстановления в пустую базу (см. `postgres.md`), а не копия файла.
