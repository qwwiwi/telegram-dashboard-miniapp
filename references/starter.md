# Каркас проекта: бот, мини-приложение, backend, сборщик

Это рабочий минимум. Все файлы согласованы между собой: цифры, имена таблиц и адреса API одни и те же. Каркас запускается и проходит свои тесты. Берите его целиком, потом меняйте под бизнес: прежде всего список метрик в `app/metrics.py` и источник в сборщике.

Файл длинный, потому что в нём весь код. Читайте только нужный раздел: список файлов ниже говорит, что где лежит.

Код: Python 3.12+, FastAPI (веб-фреймворк), SQLite, страница на чистом JavaScript с библиотекой Chart.js для графиков. Комментарии в коде на английском.

## 1. Файлы

```
dashboard/
  .env.example  .gitignore  requirements.txt  requirements-dev.txt
  app/
    settings.py     настройки из переменных окружения
    metrics.py      каталог метрик, отделы, формулы производных (одно место правды)
    db.py           схема SQLite, подключение, синхронизация метрик и allowlist
    auth.py         проверка initData, токен сессии, ключи сборщиков, лимит попыток
    main.py         API: /api/login, /api/summary, /api/department/{name}, /api/sections, /ingest
    keys.py         команда для ключей сборщиков
  static/
    index.html      мини-приложение (карточки, график, разбивка)
    vendor/chart.umd.min.js   библиотека графиков, скачать один раз и закоммитить
  bot/bot.py        бот: кнопка «Открыть дашборд», кнопка меню
  collectors/push_example.py   пример сборщика (+ демо-данные)
  tools/dev_server.py          экран в обычном браузере с подставным пользователем
  tests/test_dashboard.py  tests/test_bot.py
  deploy/           Caddyfile, systemd-службы и таймеры
```

## 2. Первый запуск на своём компьютере

```bash
python3 -m venv venv && venv/bin/pip install -r requirements-dev.txt
mkdir -p static/vendor
curl -L -o static/vendor/chart.umd.min.js https://cdn.jsdelivr.net/npm/chart.js@4.5.1/dist/chart.umd.min.js
venv/bin/python -m pytest          # all green before anything else
venv/bin/python tools/dev_server.py   # then open http://127.0.0.1:8765/dev
```

`tools/dev_server.py` создаёт временную базу `dev.db`, заливает 90 дней демо-цифр через настоящий `/ingest` и открывает страницу с подставным Telegram-пользователем. Его нельзя выкладывать на сервер.
Предупреждение pytest про `httpx` в новых версиях Starlette безвредно.

## 3. Настройки (`.env.example`)

Копия называется `.env` и в git не попадает. Приложение читает обычные переменные окружения (сам `.env` оно не открывает): локально загрузите их командой `set -a; . ./.env; set +a`, на сервере их подставляет systemd из `/etc/dashboard/*.env` (см. деплой).

```bash
# Copy to .env (never commit .env). Real values only on the server.
BOT_TOKEN=<<from BotFather>>
# Comma-separated Telegram numeric ids that may open the dashboard
ALLOWED_TG_IDS=111111111
# Long random string: python -c "import secrets; print(secrets.token_urlsafe(32))"
SESSION_SECRET=<<random string, 32+ characters>>
DB_PATH=/var/lib/dashboard/dashboard.db
# Business timezone: day boundaries of all daily numbers. Set once, never change.
BUSINESS_TZ=UTC
CURRENCY=₽
# Bot and collectors only
PUBLIC_URL=https://dashboard.example.com
INGEST_URL=http://127.0.0.1:8000
INGEST_KEY=<<from app.keys create>>
```

`.gitignore`:

```text
.env
*.env
*.db
*.db-wal
*.db-shm
venv/
__pycache__/
.pytest_cache/
```

`requirements.txt`:

```text
fastapi
uvicorn[standard]
```

`requirements-dev.txt`:

```text
-r requirements.txt
pytest
httpx
```

## 4. Backend

### `app/settings.py`

```python
"""Settings from environment variables. Values are never logged."""

from __future__ import annotations

import os
from collections.abc import Mapping
from dataclasses import dataclass
from pathlib import Path
from zoneinfo import ZoneInfo


@dataclass(frozen=True)
class Settings:
    """Runtime settings of the dashboard backend."""

    bot_token: str
    session_secret: bytes
    allowed_tg_ids: frozenset[int]
    db_path: Path
    tz: ZoneInfo
    currency: str = "₽"
    init_data_max_age_s: int = 300  # login must follow the Mini App launch within 5 minutes
    session_ttl_s: int = 12 * 3600


def load_settings(env: Mapping[str, str] | None = None) -> Settings:
    """Build Settings from a mapping (default: the process environment).

    Raises:
        RuntimeError: a required variable is missing or malformed. The message names variables, never values.
    """
    env = os.environ if env is None else env
    required = ("BOT_TOKEN", "ALLOWED_TG_IDS", "SESSION_SECRET", "DB_PATH")
    missing = [n for n in required if not env.get(n, "").strip()]
    if missing:
        raise RuntimeError("missing settings: " + ", ".join(missing))
    try:
        ids = frozenset(int(x) for x in env["ALLOWED_TG_IDS"].split(",") if x.strip())
    except ValueError as exc:
        raise RuntimeError("ALLOWED_TG_IDS must be comma-separated integers") from exc
    if len(env["SESSION_SECRET"]) < 32:
        raise RuntimeError("SESSION_SECRET must be at least 32 characters")
    return Settings(
        bot_token=env["BOT_TOKEN"].strip(),
        session_secret=env["SESSION_SECRET"].encode(),
        allowed_tg_ids=ids,
        db_path=Path(env["DB_PATH"]),
        tz=ZoneInfo(env.get("BUSINESS_TZ", "UTC")),
        currency=env.get("CURRENCY", "₽"),
    )
```

### `app/metrics.py`: каталог метрик

Сюда вносят всё, что показывает дашборд. Остальной код общий для любых метрик.

```python
"""The catalogue of metrics: the single place that says what the dashboard shows.

To add a number: add a Metric below (stored: a collector pushes it; derived: computed from totals on read).
"""

from __future__ import annotations

from collections.abc import Callable
from dataclasses import dataclass

DEPARTMENTS: dict[str, str] = {"marketing": "Маркетинг", "sales": "Продажи", "finance": "Финансы"}


@dataclass(frozen=True)
class Metric:
    """One number on the dashboard.

    kind: ``sum`` (add up the days of a period), ``balance`` (the last value of the period) or ``derived``
    (a ratio of other totals, computed on read, never stored).
    good: which direction is good news (``up``, ``down`` or None for neutral).
    """

    key: str
    department: str
    title: str
    unit: str  # count | money | percent
    kind: str  # sum | balance | derived
    good: str | None = "up"
    derive: Callable[[dict[str, float | None]], float | None] | None = None


def div(a: float | None, b: float | None, scale: float = 1.0) -> float | None:
    """a / b * scale; None when a number is missing or b is zero (an unknown stays unknown, never 0)."""
    return None if a is None or not b else a / b * scale


def sub(a: float | None, b: float | None) -> float | None:
    """a - b; None when either is missing."""
    return None if a is None or b is None else a - b


METRICS: list[Metric] = [
    Metric("marketing.visits", "marketing", "Трафик (визиты)", "count", "sum"),
    Metric("marketing.leads", "marketing", "Заявки", "count", "sum"),
    Metric("marketing.spend", "marketing", "Расходы на рекламу", "money", "sum", good=None),
    Metric("marketing.cost_per_lead", "marketing", "Стоимость заявки", "money", "derived", good="down",
           derive=lambda t: div(t.get("marketing.spend"), t.get("marketing.leads"))),
    Metric("sales.deals", "sales", "Сделки", "count", "sum"),
    Metric("sales.conversion", "sales", "Конверсия заявка → сделка", "percent", "derived",
           derive=lambda t: div(t.get("sales.deals"), t.get("marketing.leads"), 100)),
    Metric("sales.revenue", "sales", "Выручка", "money", "sum"),
    Metric("finance.income", "finance", "Доходы", "money", "sum"),
    Metric("finance.expenses", "finance", "Расходы", "money", "sum", good="down"),
    Metric("finance.profit", "finance", "Прибыль", "money", "derived",
           derive=lambda t: sub(t.get("finance.income"), t.get("finance.expenses"))),
    Metric("finance.cash_balance", "finance", "Остаток денег", "money", "balance"),
]
BY_KEY: dict[str, Metric] = {m.key: m for m in METRICS}
STORED: dict[str, Metric] = {m.key: m for m in METRICS if m.kind != "derived"}
# The «Сводка» screen: 4-6 numbers, in this order.
SUMMARY_KEYS = ["marketing.leads", "marketing.cost_per_lead", "sales.deals", "sales.revenue", "finance.profit",
                "finance.cash_balance"]
SUMMARY_CHART_KEY = "sales.revenue"
```

### `app/db.py`: схема

Те же таблицы описаны в `database.md`.

```python
"""SQLite access: one short-lived connection per request, WAL mode."""

from __future__ import annotations

import sqlite3
from collections.abc import Iterable
from pathlib import Path

from app.metrics import STORED

SCHEMA = """
CREATE TABLE IF NOT EXISTS allowed_users (
  tg_id INTEGER PRIMARY KEY,
  name TEXT NOT NULL DEFAULT '',
  added_at TEXT NOT NULL DEFAULT (datetime('now')),
  revoked_at TEXT
);
CREATE TABLE IF NOT EXISTS metric_defs (
  key TEXT PRIMARY KEY,
  department TEXT NOT NULL,
  title TEXT NOT NULL,
  unit TEXT NOT NULL CHECK (unit IN ('count', 'money', 'percent')),
  kind TEXT NOT NULL CHECK (kind IN ('sum', 'balance'))
);
CREATE TABLE IF NOT EXISTS metric_values (
  key TEXT NOT NULL REFERENCES metric_defs(key),
  day TEXT NOT NULL,
  dim TEXT NOT NULL DEFAULT '',
  value REAL NOT NULL,
  source TEXT NOT NULL,
  updated_at TEXT NOT NULL DEFAULT (datetime('now')),
  PRIMARY KEY (key, day, dim)
);
CREATE INDEX IF NOT EXISTS metric_values_day ON metric_values(day);
CREATE TABLE IF NOT EXISTS ingest_keys (
  id INTEGER PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,
  key_hash TEXT NOT NULL UNIQUE,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  revoked_at TEXT,
  last_used_at TEXT
);
CREATE TABLE IF NOT EXISTS ingest_log (
  id INTEGER PRIMARY KEY,
  at TEXT NOT NULL DEFAULT (datetime('now')),
  key_name TEXT NOT NULL,
  source TEXT NOT NULL,
  rows INTEGER NOT NULL
);
"""


def connect(path: Path) -> sqlite3.Connection:
    """Open the database (creating its folder) with WAL, foreign keys and a busy timeout."""
    path.parent.mkdir(parents=True, exist_ok=True)
    # FastAPI may open and close a request's connection on different threads; one request uses it at a time.
    conn = sqlite3.connect(path, timeout=5.0, check_same_thread=False)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA journal_mode=WAL")
    conn.execute("PRAGMA foreign_keys=ON")
    return conn


def migrate(conn: sqlite3.Connection, allowed_ids: Iterable[int]) -> None:
    """Create tables, sync metric_defs with app.metrics, add allowlist ids (a revoked id stays revoked)."""
    conn.executescript(SCHEMA)
    with conn:
        for m in STORED.values():
            conn.execute(
                "INSERT INTO metric_defs(key, department, title, unit, kind) VALUES (?,?,?,?,?) "
                "ON CONFLICT(key) DO UPDATE SET department=excluded.department, title=excluded.title, "
                "unit=excluded.unit, kind=excluded.kind", (m.key, m.department, m.title, m.unit, m.kind))
        for tg_id in allowed_ids:
            conn.execute("INSERT OR IGNORE INTO allowed_users(tg_id) VALUES (?)", (tg_id,))
```

### `app/auth.py`: проверка initData, сессии, ключи

`validate_init_data` реализует официальный алгоритм проверки по токену бота. Если захотите проверять подпись без токена на сервере (сторонний сервис), у Telegram есть второй способ с открытым ключом Ed25519: он описан на той же странице, поле `signature`. Для личного дашборда этого не нужно.

```python
"""Telegram Mini App login: initData check, signed session token, ingest keys, a tiny rate limiter."""

from __future__ import annotations

import hashlib
import hmac
import json
import secrets
import sqlite3
import time
from collections import defaultdict
from typing import Any
from urllib.parse import parse_qsl

MAX_INIT_DATA = 4096


class InitDataError(ValueError):
    """initData is malformed, forged or too old. The message never contains the data itself."""


def validate_init_data(init_data: str, bot_token: str, now: float, max_age_s: int = 300) -> dict[str, Any]:
    """Check Telegram Mini App initData (official HMAC algorithm) and return the user object.

    Steps (core.telegram.org/bots/webapps, «Validating data received via the Mini App»):
      1. split the query string into fields, take ``hash`` out (it is not part of the signed text);
      2. data-check-string = the other fields as ``key=value`` (values URL-decoded), sorted by key, joined by ``\\n``
         (the ``signature`` field, if present, stays in: only ``hash`` is removed);
      3. secret = HMAC_SHA256(key="WebAppData", message=bot_token);
      4. expected = hex(HMAC_SHA256(key=secret, message=data-check-string)), compared in constant time;
      5. ``auth_date`` must be fresh (and not from the future).

    Raises:
        InitDataError: any check failed.
    """
    if not init_data or len(init_data) > MAX_INIT_DATA:
        raise InitDataError("bad size")
    try:
        pairs = parse_qsl(init_data, keep_blank_values=True, strict_parsing=True)
    except ValueError as exc:
        raise InitDataError("malformed") from exc
    fields = dict(pairs)
    if len(fields) != len(pairs):  # a repeated key makes «which value was signed» ambiguous
        raise InitDataError("duplicate field")
    received = fields.pop("hash", "")
    if not received:
        raise InitDataError("no hash")
    check_string = "\n".join(sorted(f"{k}={v}" for k, v in fields.items()))  # sort the lines, as Telegram's sample does
    secret = hmac.new(b"WebAppData", bot_token.encode(), hashlib.sha256).digest()
    expected = hmac.new(secret, check_string.encode(), hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expected, received):
        raise InitDataError("bad signature")
    # Only signed data below this line.
    auth_date = fields.get("auth_date", "")
    if not (auth_date.isascii() and auth_date.isdigit()):
        raise InitDataError("bad auth_date")
    age = now - int(auth_date)
    if age > max_age_s or age < -60:
        raise InitDataError("expired")
    try:
        user = json.loads(fields.get("user", ""))
    except ValueError as exc:
        raise InitDataError("no user") from exc
    if not isinstance(user, dict) or not isinstance(user.get("id"), int) or isinstance(user.get("id"), bool):
        raise InitDataError("no user id")
    return user


def _sign(secret: bytes, payload: str) -> str:
    return hmac.new(secret, payload.encode(), hashlib.sha256).hexdigest()


def make_session(secret: bytes, tg_id: int, expires: int) -> str:
    """A signed token ``<tg_id>.<expires>.<signature>``; the server keeps no session table."""
    payload = f"{tg_id}.{expires}"
    return f"{payload}.{_sign(secret, payload)}"


def read_session(secret: bytes, token: str | None, now: float) -> int | None:
    """The Telegram id of a valid, unexpired token, else None."""
    if not token:
        return None
    parts = token.split(".")
    if len(parts) != 3 or not (parts[0].isdigit() and parts[1].isdigit()):
        return None
    if not hmac.compare_digest(_sign(secret, f"{parts[0]}.{parts[1]}"), parts[2]):
        return None
    return int(parts[0]) if int(parts[1]) > now else None


def is_allowed(conn: sqlite3.Connection, tg_id: int) -> bool:
    """True when the id is on the allowlist and not revoked. Checked on every request, so a revoke is instant."""
    row = conn.execute("SELECT 1 FROM allowed_users WHERE tg_id=? AND revoked_at IS NULL", (tg_id,)).fetchone()
    return row is not None


def sha256_hex(value: str) -> str:
    """Hex sha256 of a string (ingest keys are stored only in this form)."""
    return hashlib.sha256(value.encode()).hexdigest()


def new_ingest_key() -> str:
    """A 256-bit random key; show it once, store only its hash."""
    return secrets.token_urlsafe(32)


def ingest_key_name(conn: sqlite3.Connection, presented: str | None, now_iso: str) -> str | None:
    """The name of the active ingest key behind ``presented`` (and stamp its last use), else None."""
    if not presented:
        return None
    row = conn.execute("SELECT id, name FROM ingest_keys WHERE key_hash=? AND revoked_at IS NULL",
                       (sha256_hex(presented),)).fetchone()
    if row is None:
        return None
    with conn:
        conn.execute("UPDATE ingest_keys SET last_used_at=? WHERE id=?", (now_iso, row["id"]))
    return str(row["name"])


def bearer(header: str | None) -> str | None:
    """The token of an ``Authorization: Bearer <token>`` header."""
    scheme, _, value = (header or "").partition(" ")
    if scheme.lower() != "bearer":
        return None
    return value.strip() or None


class FailureLimiter:
    """At most ``limit`` failed logins per client per ``window_s`` seconds (in memory, enough for one process)."""

    def __init__(self, limit: int = 10, window_s: int = 600) -> None:
        self.limit, self.window_s = limit, window_s
        self._fails: dict[str, list[float]] = defaultdict(list)

    def blocked(self, client: str) -> bool:
        cutoff = time.time() - self.window_s
        self._fails[client] = [t for t in self._fails[client] if t > cutoff]
        return len(self._fails[client]) >= self.limit

    def record(self, client: str) -> None:
        self._fails[client].append(time.time())
```

### `app/main.py`: API

Адреса:
- `POST /api/login` – тело `{"init_data": "..."}`, ответ `{"token": "...", "expires_in": ...}`;
- `GET /api/sections` – меню;
- `GET /api/summary?days=7|30|90` – карточки «Сводки» и график;
- `GET /api/department/{name}?days=7|30|90` – карточки, ряды по дням и разбивка отдела;
- `POST /ingest` – для сборщиков, `Authorization: Bearer <ключ>`;
- `GET /healthz`.

Все `/api/...` кроме `login` требуют `Authorization: Bearer <токен сессии>`.

```python
"""The dashboard backend: login, read API for the Mini App, ingest for collectors, static files."""

from __future__ import annotations

import logging
import sqlite3
import time
from collections.abc import Iterator
from datetime import date, datetime, timedelta
from pathlib import Path
from typing import Any

from fastapi import Depends, FastAPI, HTTPException, Query, Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import FileResponse, JSONResponse
from fastapi.staticfiles import StaticFiles
from pydantic import BaseModel, Field, field_validator

from app import auth
from app.db import connect, migrate
from app.metrics import BY_KEY, DEPARTMENTS, METRICS, STORED, SUMMARY_CHART_KEY, SUMMARY_KEYS, Metric
from app.settings import Settings, load_settings

logger = logging.getLogger("dashboard")
STATIC = Path(__file__).resolve().parent.parent / "static"
PERIODS = (7, 30, 90)
MAX_PAST_DAYS = 400


# ---------- request models ----------

class LoginIn(BaseModel):
    init_data: str = Field(min_length=1, max_length=auth.MAX_INIT_DATA)


class RowIn(BaseModel):
    key: str = Field(max_length=64)
    day: str = Field(min_length=10, max_length=10)  # YYYY-MM-DD in the business timezone
    value: float = Field(allow_inf_nan=False)
    dim: str = Field(default="", max_length=64)  # '' = the total; else a breakdown value, e.g. a channel

    @field_validator("day")
    @classmethod
    def _real_date(cls, v: str) -> str:
        date.fromisoformat(v)
        return v


class IngestIn(BaseModel):
    source: str = Field(pattern=r"^[a-z0-9_-]{1,40}$")
    rows: list[RowIn] = Field(max_length=2000)


# ---------- dependencies ----------

def get_settings(request: Request) -> Settings:
    return request.app.state.settings


def get_conn(request: Request) -> Iterator[sqlite3.Connection]:
    conn = connect(request.app.state.settings.db_path)
    try:
        yield conn
    finally:
        conn.close()


def now_ts(request: Request) -> float:
    return float(request.app.state.clock())


def current_user(request: Request, conn: sqlite3.Connection = Depends(get_conn),
                 settings: Settings = Depends(get_settings), t: float = Depends(now_ts)) -> int:
    """The Telegram id behind a valid session token that is STILL on the allowlist; 401 / 403 otherwise."""
    tg_id = auth.read_session(settings.session_secret, auth.bearer(request.headers.get("authorization")), t)
    if tg_id is None:
        raise HTTPException(status_code=401)
    if not auth.is_allowed(conn, tg_id):
        raise HTTPException(status_code=403)
    return tg_id


def ingest_key(request: Request, conn: sqlite3.Connection = Depends(get_conn), t: float = Depends(now_ts),
               settings: Settings = Depends(get_settings)) -> str:
    """The name of the ingest key presented as a Bearer token; 401 when unknown or revoked."""
    stamp = datetime.fromtimestamp(t, settings.tz).strftime("%Y-%m-%d %H:%M:%S")
    name = auth.ingest_key_name(conn, auth.bearer(request.headers.get("authorization")), stamp)
    if name is None:
        logger.warning("ingest: bad key")
        raise HTTPException(status_code=401)
    return name


# ---------- numbers ----------

def local_today(settings: Settings, t: float) -> date:
    return datetime.fromtimestamp(t, settings.tz).date()


def windows(today: date, days: int) -> tuple[tuple[str, str], tuple[str, str]]:
    """(current, previous) windows of ``days`` FULL days ending yesterday (today is a partial day, never compared)."""
    end = today - timedelta(days=1)
    start = end - timedelta(days=days - 1)
    prev_end = start - timedelta(days=1)
    prev_start = prev_end - timedelta(days=days - 1)
    return (start.isoformat(), end.isoformat()), (prev_start.isoformat(), prev_end.isoformat())


def totals(conn: sqlite3.Connection, start: str, end: str) -> dict[str, float | None]:
    """Total per metric over [start, end]: sum / last balance for stored ones, then the derived ratios.

    A metric with no rows in the window is None («no data»), never 0.
    """
    out: dict[str, float | None] = {k: None for k in BY_KEY}
    for r in conn.execute("SELECT v.key, SUM(v.value) AS s FROM metric_values v JOIN metric_defs d ON d.key=v.key "
                          "WHERE v.dim='' AND d.kind='sum' AND v.day BETWEEN ? AND ? GROUP BY v.key", (start, end)):
        out[r["key"]] = r["s"]
    for r in conn.execute(
            "SELECT v.key, v.value FROM metric_values v JOIN metric_defs d ON d.key=v.key "
            "JOIN (SELECT key, MAX(day) AS day FROM metric_values WHERE dim='' AND day BETWEEN ? AND ? GROUP BY key) m "
            "ON m.key=v.key AND m.day=v.day WHERE v.dim='' AND d.kind='balance'", (start, end)):
        out[r["key"]] = r["value"]
    for m in METRICS:
        if m.derive is not None:
            out[m.key] = m.derive(out)
    return out


def card(m: Metric, cur: float | None, prev: float | None) -> dict[str, Any]:
    """One number with its change against the previous period (None when either side is unknown or prev is 0)."""
    delta = None if cur is None or not prev else round((cur - prev) / abs(prev) * 100, 1)
    return {"key": m.key, "title": m.title, "unit": m.unit, "good": m.good, "value": cur, "prev": prev,
            "delta_pct": delta, "plot": m.kind != "derived"}


def series(conn: sqlite3.Connection, key: str, start: str, end: str) -> list[dict[str, Any]]:
    """One point per day of [start, end]; a day without data is null (a gap in the chart), never 0."""
    have = {r["day"]: r["value"] for r in conn.execute(
        "SELECT day, value FROM metric_values WHERE key=? AND dim='' AND day BETWEEN ? AND ?", (key, start, end))}
    d, last, points = date.fromisoformat(start), date.fromisoformat(end), []
    while d <= last:
        points.append({"day": d.isoformat(), "value": have.get(d.isoformat())})
        d += timedelta(days=1)
    return points


def period_param(days: int = Query(7)) -> int:
    if days not in PERIODS:
        raise HTTPException(status_code=422, detail="days: 7, 30 or 90")
    return days


# ---------- app ----------

def create_app(settings: Settings | None = None) -> FastAPI:
    """Application factory (uvicorn ``--factory``); tests pass their own settings."""
    settings = settings or load_settings()
    conn = connect(settings.db_path)
    try:
        migrate(conn, settings.allowed_tg_ids)
    finally:
        conn.close()
    app = FastAPI(title="dashboard", docs_url=None, redoc_url=None, openapi_url=None)  # no public API docs
    app.state.settings = settings
    app.state.clock = time.time
    app.state.limiter = auth.FailureLimiter()

    @app.exception_handler(RequestValidationError)
    async def validation_error(request: Request, exc: RequestValidationError) -> JSONResponse:
        """422 without the rejected input: a login body carries signed user data and must never be echoed."""
        detail = [{"loc": [str(x) for x in e.get("loc", ())], "msg": str(e.get("msg", ""))} for e in exc.errors()]
        return JSONResponse({"detail": detail}, status_code=422)

    @app.middleware("http")
    async def no_store(request: Request, call_next: Any) -> Any:
        response = await call_next(request)
        if request.url.path.startswith(("/api/", "/ingest")):
            response.headers["Cache-Control"] = "no-store"
        return response

    @app.get("/healthz")
    def healthz() -> dict[str, bool]:
        return {"ok": True}

    @app.post("/api/login")
    def login(body: LoginIn, request: Request, conn: sqlite3.Connection = Depends(get_conn),
              s: Settings = Depends(get_settings), t: float = Depends(now_ts)) -> JSONResponse:
        """Exchange Telegram initData for a session token. The browser's own claims are never trusted."""
        client = request.client.host if request.client else "unknown"
        limiter: auth.FailureLimiter = request.app.state.limiter
        if limiter.blocked(client):
            raise HTTPException(status_code=429, detail="Слишком много попыток, подождите 10 минут")
        try:
            user = auth.validate_init_data(body.init_data, s.bot_token, t, s.init_data_max_age_s)
        except auth.InitDataError as exc:
            limiter.record(client)
            logger.info("login refused: %s", exc)
            raise HTTPException(status_code=401, detail="Подпись Telegram не прошла проверку") from None
        if not auth.is_allowed(conn, user["id"]):
            limiter.record(client)
            raise HTTPException(status_code=403, detail="Нет доступа")
        token = auth.make_session(s.session_secret, user["id"], int(t) + s.session_ttl_s)
        name = str(user.get("first_name", ""))[:40]
        return JSONResponse({"token": token, "expires_in": s.session_ttl_s, "name": name})

    @app.get("/api/summary")
    def summary(days: int = Depends(period_param), _: int = Depends(current_user),
                conn: sqlite3.Connection = Depends(get_conn), s: Settings = Depends(get_settings),
                t: float = Depends(now_ts)) -> dict[str, Any]:
        (start, end), (pstart, pend) = windows(local_today(s, t), days)
        cur, prev = totals(conn, start, end), totals(conn, pstart, pend)
        chart = BY_KEY[SUMMARY_CHART_KEY]
        return {"period": {"from": start, "to": end, "days": days}, "currency": s.currency,
                "cards": [card(BY_KEY[k], cur[k], prev[k]) for k in SUMMARY_KEYS],
                "chart": {"key": chart.key, "title": chart.title, "unit": chart.unit,
                          "points": series(conn, chart.key, start, end)}}

    @app.get("/api/department/{name}")
    def department(name: str, days: int = Depends(period_param), _: int = Depends(current_user),
                   conn: sqlite3.Connection = Depends(get_conn), s: Settings = Depends(get_settings),
                   t: float = Depends(now_ts)) -> dict[str, Any]:
        if name not in DEPARTMENTS:
            raise HTTPException(status_code=404)
        (start, end), (pstart, pend) = windows(local_today(s, t), days)
        cur, prev = totals(conn, start, end), totals(conn, pstart, pend)
        mine = [m for m in METRICS if m.department == name]
        breakdown: dict[str, list[dict[str, Any]]] = {}
        for m in mine:
            if m.kind == "sum":
                rows = conn.execute("SELECT dim, SUM(value) AS s FROM metric_values WHERE key=? AND dim!='' "
                                    "AND day BETWEEN ? AND ? GROUP BY dim ORDER BY s DESC LIMIT 20",
                                    (m.key, start, end)).fetchall()
                if rows:
                    breakdown[m.key] = [{"dim": r["dim"], "value": r["s"]} for r in rows]
        return {"department": name, "title": DEPARTMENTS[name],
                "period": {"from": start, "to": end, "days": days}, "currency": s.currency,
                "cards": [card(m, cur[m.key], prev[m.key]) for m in mine],
                "series": {m.key: series(conn, m.key, start, end) for m in mine if m.key in STORED},
                "breakdown": breakdown}

    @app.get("/api/sections")
    def sections(_: int = Depends(current_user)) -> list[dict[str, str]]:
        """The menu: the frontend builds its tabs from this list."""
        return [{"id": "summary", "title": "Сводка"}] + [{"id": k, "title": v} for k, v in DEPARTMENTS.items()]

    @app.post("/ingest")
    def ingest(body: IngestIn, conn: sqlite3.Connection = Depends(get_conn), key_name: str = Depends(ingest_key),
               s: Settings = Depends(get_settings), t: float = Depends(now_ts)) -> dict[str, int]:
        """Collectors push daily numbers. Re-sending a day replaces its value (idempotent)."""
        today = local_today(s, t)
        for r in body.rows:
            m = STORED.get(r.key)
            if m is None:
                raise HTTPException(status_code=422, detail=f"unknown metric: {r.key[:64]}")
            d = date.fromisoformat(r.day)
            if d > today or (today - d).days > MAX_PAST_DAYS:
                raise HTTPException(status_code=422, detail="day out of range")
            if r.value < 0 and m.kind != "balance":
                raise HTTPException(status_code=422, detail="negative value")
        with conn:
            for r in body.rows:
                conn.execute(
                    "INSERT INTO metric_values(key, day, dim, value, source) VALUES (?,?,?,?,?) "
                    "ON CONFLICT(key, day, dim) DO UPDATE SET value=excluded.value, source=excluded.source, "
                    "updated_at=datetime('now')", (r.key, r.day, r.dim, r.value, body.source))
            conn.execute("INSERT INTO ingest_log(key_name, source, rows) VALUES (?,?,?)",
                         (key_name, body.source, len(body.rows)))
        return {"stored": len(body.rows)}

    @app.get("/")
    def index() -> FileResponse:
        return FileResponse(STATIC / "index.html", headers={"Cache-Control": "no-cache"})

    app.mount("/static", StaticFiles(directory=STATIC), name="static")
    return app
```

### `app/keys.py`: ключи для сборщиков

```python
"""Manage ingest keys: ``python -m app.keys create <name>`` | ``revoke <name>`` | ``list``.

A new key is printed once. Only its sha256 is stored, so a lost key cannot be recovered: revoke and create another.
"""

from __future__ import annotations

import sys

from app.auth import new_ingest_key, sha256_hex
from app.db import connect, migrate
from app.settings import load_settings


def main(argv: list[str]) -> int:
    """Run the command; returns a process exit code."""
    settings = load_settings()
    conn = connect(settings.db_path)
    migrate(conn, settings.allowed_tg_ids)
    cmd = argv[0] if argv else ""
    if cmd == "create" and len(argv) == 2:
        key = new_ingest_key()
        with conn:
            conn.execute("INSERT INTO ingest_keys(name, key_hash) VALUES (?, ?)", (argv[1], sha256_hex(key)))
        print(f"Key for «{argv[1]}» (shown once, put it into the collector's env file):\n{key}")
    elif cmd == "revoke" and len(argv) == 2:
        with conn:
            n = conn.execute("UPDATE ingest_keys SET revoked_at=datetime('now') WHERE name=? AND revoked_at IS NULL",
                             (argv[1],)).rowcount
        print("revoked" if n else "no active key with this name")
    elif cmd == "list":
        for r in conn.execute("SELECT name, created_at, revoked_at, last_used_at FROM ingest_keys ORDER BY id"):
            print(r["name"], r["created_at"], r["revoked_at"] or "active", r["last_used_at"] or "never used")
    else:
        print(__doc__)
        return 2
    return 0


if __name__ == "__main__":
    sys.exit(main(sys.argv[1:]))
```

## 5. Мини-приложение: `static/index.html`

Одна страница. Меню, периоды и карточки строятся из ответов API: новый отдел появляется без правок страницы. Страница подключает официальный скрипт Telegram и загружает токен через `/api/login`. Вне Telegram она просит открыть дашборд через бота.

```html
<!doctype html>
<html lang="ru">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
  <meta name="robots" content="noindex, nofollow">
  <title>Моя компания</title>
  <!-- Telegram Mini App SDK: official file, must load before our script -->
  <script src="https://telegram.org/js/telegram-web-app.js"></script>
  <!-- Chart library, saved once into static/vendor (do not hotlink a foreign CDN: it stalls on some mobile networks) -->
  <script src="/static/vendor/chart.umd.min.js" defer></script>
  <style>
    :root { --bg: var(--tg-theme-bg-color, #fff); --text: var(--tg-theme-text-color, #111);
            --hint: var(--tg-theme-hint-color, #777); --card: var(--tg-theme-secondary-bg-color, #f2f2f4);
            --accent: var(--tg-theme-button-color, #2481cc); --good: #1a8f4c; --bad: #d03a3a; }
    * { box-sizing: border-box; }
    body { margin: 0; background: var(--bg); color: var(--text); font: 15px/1.4 system-ui, sans-serif; }
    main { padding: 12px 12px 32px; max-width: 760px; margin: 0 auto; }
    nav, .periods { display: flex; gap: 8px; overflow-x: auto; margin-bottom: 12px; }
    button { border: 0; border-radius: 16px; padding: 8px 14px; background: var(--card); color: var(--text);
             font: inherit; white-space: nowrap; }
    button.on { background: var(--accent); color: var(--tg-theme-button-text-color, #fff); }
    .grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(150px, 1fr)); gap: 10px; }
    .card { background: var(--card); border-radius: 12px; padding: 12px; min-width: 0; }
    .card.pick { cursor: pointer; }
    .card.sel { outline: 2px solid var(--accent); }
    .title { color: var(--hint); font-size: 13px; overflow-wrap: anywhere; }
    .value { font-size: 22px; font-weight: 650; margin: 4px 0; overflow-wrap: anywhere; }
    .delta.good { color: var(--good); } .delta.bad { color: var(--bad); } .delta.none { color: var(--hint); }
    .chart { background: var(--card); border-radius: 12px; padding: 12px; margin-top: 12px; height: 260px; }
    h2 { font-size: 15px; margin: 16px 0 6px; }
    table { width: 100%; border-collapse: collapse; }
    td { padding: 6px 0; border-bottom: 1px solid var(--card); }
    td:last-child { text-align: right; }
    #msg { color: var(--hint); padding: 24px 0; text-align: center; }
  </style>
</head>
<body>
<main>
  <nav id="tabs" aria-label="Разделы"></nav>
  <div class="periods" id="periods" aria-label="Период"></div>
  <div id="msg">Загрузка…</div>
  <div class="grid" id="cards"></div>
  <div class="chart" id="chartbox" hidden><canvas id="chart"></canvas></div>
  <div id="extra"></div>
</main>
<script>
"use strict";
const tg = window.Telegram && window.Telegram.WebApp;
const $ = (id) => document.getElementById(id);
const state = { section: "summary", days: 7, plot: null, data: null, chart: null, token: null, currency: "" };

function el(tag, cls, text) {                       // build DOM with textContent only: server data never becomes HTML
  const e = document.createElement(tag);
  if (cls) e.className = cls;
  if (text !== undefined) e.textContent = text;
  return e;
}
function fmt(value, unit) {
  if (value === null || value === undefined) return "нет данных";
  if (unit === "percent") return new Intl.NumberFormat("ru-RU", { maximumFractionDigits: 1 }).format(value) + " %";
  const n = new Intl.NumberFormat("ru-RU", { maximumFractionDigits: unit === "money" ? 0 : 1 }).format(value);
  return unit === "money" ? n + " " + state.currency : n;
}
function say(text) { $("msg").textContent = text; $("msg").hidden = !text; }

async function login() {                            // the ONLY place that sends initData; the server decides who you are
  const r = await fetch("/api/login", { method: "POST", headers: { "Content-Type": "application/json" },
                                        body: JSON.stringify({ init_data: tg.initData }) });
  if (r.status === 403) throw new Error("У этого аккаунта нет доступа к дашборду.");
  if (!r.ok) throw new Error("Не удалось войти. Закройте окно и откройте дашборд снова из бота.");
  state.token = (await r.json()).token;
  sessionStorage.setItem("token", state.token);
}
async function api(path, retry = true) {
  const r = await fetch(path, { headers: { Authorization: "Bearer " + state.token, Accept: "application/json" } });
  if (r.status === 401 && retry) { await login(); return api(path, false); }
  if (r.status === 403) throw new Error("У этого аккаунта нет доступа к дашборду.");
  if (!r.ok) throw new Error("Сервер ответил " + r.status);
  return r.json();
}

function deltaNode(c) {
  if (c.delta_pct === null) return el("div", "delta none", "нет сравнения");
  const up = c.delta_pct >= 0;
  const good = c.good === null ? null : (c.good === "up") === up;
  const cls = c.delta_pct === 0 || good === null ? "none" : good ? "good" : "bad";
  return el("div", "delta " + cls, (up ? "+" : "") + c.delta_pct + " % к прошлому периоду");
}
function drawChart(title, unit, points) {
  const canvas = $("chart");
  $("chartbox").hidden = !window.Chart;
  if (!window.Chart) return;                         // library missing: cards still work
  if (state.chart) state.chart.destroy();
  state.chart = new Chart(canvas, {
    type: "line",
    data: { labels: points.map((p) => p.day.slice(5)),
            datasets: [{ label: title, data: points.map((p) => p.value), borderColor: "#2481cc", tension: 0.25,
                         spanGaps: false, pointRadius: 2 }] },
    options: { responsive: true, maintainAspectRatio: false, animation: false,
               plugins: { legend: { display: true }, tooltip: { callbacks: { label: (c) => fmt(c.parsed.y, unit) } } },
               scales: { y: { beginAtZero: true, ticks: { callback: (v) => new Intl.NumberFormat("ru-RU").format(v) } } } },
  });
}
function render() {
  const d = state.data;
  state.currency = d.currency;
  const grid = $("cards");
  grid.replaceChildren();
  for (const c of d.cards) {
    const node = el("div", "card" + (c.plot && d.series ? " pick" : "") + (state.plot === c.key ? " sel" : ""));
    node.append(el("div", "title", c.title), el("div", "value", fmt(c.value, c.unit)), deltaNode(c));
    if (c.plot && d.series) node.onclick = () => { state.plot = c.key; render(); };
    grid.append(node);
  }
  if (d.chart) drawChart(d.chart.title, d.chart.unit, d.chart.points);
  else {
    const c = d.cards.find((x) => x.key === state.plot) || d.cards.find((x) => x.plot);
    state.plot = c.key;
    drawChart(c.title, c.unit, d.series[c.key]);
  }
  const extra = $("extra");
  extra.replaceChildren();
  for (const [key, rows] of Object.entries(d.breakdown || {})) {   // e.g. leads by channel
    const title = d.cards.find((x) => x.key === key).title;
    const table = el("table");
    for (const r of rows) { const tr = el("tr"); tr.append(el("td", "", r.dim), el("td", "", fmt(r.value, "count"))); table.append(tr); }
    extra.append(el("h2", "", title + ": по каналам"), table);
  }
  say("");
}
async function load() {
  say("Загрузка…");
  try {
    const path = state.section === "summary" ? "/api/summary" : "/api/department/" + state.section;
    state.data = await api(path + "?days=" + state.days);
    render();
  } catch (e) { say(e.message); }
}
function buttons(box, items, current, onPick) {
  box.replaceChildren();
  for (const [id, label] of items) {
    const b = el("button", id === current ? "on" : "", label);
    b.type = "button";
    b.onclick = () => onPick(id);
    box.append(b);
  }
}
function drawControls(sections) {
  buttons($("tabs"), sections.map((s) => [s.id, s.title]), state.section, (id) => { state.section = id; state.plot = null; drawControls(sections); load(); });
  buttons($("periods"), [[7, "7 дней"], [30, "30 дней"], [90, "90 дней"]], state.days, (d) => { state.days = d; drawControls(sections); load(); });
}
async function start() {
  if (!tg || !tg.initData) { say("Откройте дашборд через бота в Telegram."); return; }
  tg.ready(); tg.expand();
  try {
    state.token = sessionStorage.getItem("token");
    if (!state.token) await login();
    drawControls(await api("/api/sections"));
    await load();
  } catch (e) { say(e.message); }
}
start();
</script>
</body>
</html>
```

Дальше по желанию: вынести скрипт и стили в отдельные файлы и добавить заголовок `Content-Security-Policy` (разрешить свой домен и `https://telegram.org`); после этого проверить, что приложение открывается в Telegram.

## 6. Бот: `bot/bot.py`

Мини-приложение открывается двумя способами, и оба нужны:
- **кнопка под сообщением** (`web_app` в inline-клавиатуре): бот присылает её на `/start`;
- **кнопка меню** рядом с полем ввода: `python bot/bot.py set-menu` вызывает метод Bot API `setChatMenuButton` для каждого id из списка. То же можно сделать в BotFather (раздел настроек бота, пункт про кнопку меню; названия пунктов в BotFather могут отличаться).

Адрес мини-приложения должен быть HTTPS.

```python
"""The bot: answers /start with a button that opens the Mini App, for allowlisted people only.

Run:  python bot/bot.py            (long polling, keep it running as a service)
      python bot/bot.py set-menu   (one-off: puts the «Дашборд» button next to the message field of each allowed chat)
Environment: BOT_TOKEN, ALLOWED_TG_IDS, PUBLIC_URL (https://..., the address where the dashboard is served).
Only the standard library is used. The token is part of the Bot API URL and is never logged.
"""

from __future__ import annotations

import json
import logging
import os
import sys
import time
import urllib.error
import urllib.request
from typing import Any

logger = logging.getLogger("dashboard-bot")
NO_ACCESS = "Нет доступа."
BUTTON_TEXT = "Дашборд"


def call(token: str, method: str, body: dict[str, Any], timeout: float = 40) -> Any:
    """Call a Bot API method; returns ``result`` or raises RuntimeError (the message has no URL and no token)."""
    req = urllib.request.Request(f"https://api.telegram.org/bot{token}/{method}", data=json.dumps(body).encode(),
                                 headers={"Content-Type": "application/json"})
    try:
        with urllib.request.urlopen(req, timeout=timeout) as resp:
            data = json.load(resp)
    except (urllib.error.URLError, TimeoutError, OSError, ValueError) as exc:
        raise RuntimeError(f"{method}: {type(exc).__name__}") from None
    if not data.get("ok"):
        raise RuntimeError(f"{method}: not ok")
    return data.get("result")


def open_button(public_url: str) -> dict[str, Any]:
    """An inline button that opens the Mini App inside Telegram (a web_app button, not a plain link)."""
    return {"inline_keyboard": [[{"text": "Открыть дашборд", "web_app": {"url": public_url}}]]}


def set_menu_buttons(token: str, allowed: frozenset[int], public_url: str) -> None:
    """Put the «Дашборд» menu button into each allowed private chat."""
    for chat_id in sorted(allowed):
        call(token, "setChatMenuButton", {"chat_id": chat_id, "menu_button": {
            "type": "web_app", "text": BUTTON_TEXT, "web_app": {"url": public_url}}})
        logger.info("menu button set for %s", chat_id)


def handle(token: str, update: dict[str, Any], allowed: frozenset[int], public_url: str) -> None:
    """Answer one message. Strangers get a refusal and no button."""
    msg = update.get("message") or {}
    chat = msg.get("chat") or {}
    if chat.get("type") != "private":
        return
    if (msg.get("from") or {}).get("id") not in allowed:
        call(token, "sendMessage", {"chat_id": chat["id"], "text": NO_ACCESS})
        return
    call(token, "sendMessage", {"chat_id": chat["id"], "text": "Дашборд компании. Нажмите кнопку ниже.",
                                "reply_markup": open_button(public_url)})


def main() -> int:
    """Entry point; returns a process exit code."""
    logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(name)s %(message)s")
    try:
        token, public_url = os.environ["BOT_TOKEN"], os.environ["PUBLIC_URL"].rstrip("/") + "/"
        allowed = frozenset(int(x) for x in os.environ["ALLOWED_TG_IDS"].split(",") if x.strip())
    except (KeyError, ValueError) as exc:
        logger.error("bad configuration: %s", type(exc).__name__)
        return 2
    if sys.argv[1:] == ["set-menu"]:
        set_menu_buttons(token, allowed, public_url)
        return 0
    offset = 0
    while True:
        try:
            updates = call(token, "getUpdates", {"offset": offset, "timeout": 30, "allowed_updates": ["message"]}, 40)
        except RuntimeError as exc:
            logger.warning("getUpdates failed: %s", exc)
            time.sleep(5)
            continue
        for update in updates:
            offset = max(offset, int(update["update_id"]) + 1)
            try:
                handle(token, update, allowed, public_url)
            except Exception:  # noqa: BLE001 - one bad message must not stop the bot
                logger.exception("update failed")


if __name__ == "__main__":
    sys.exit(main())
```

## 7. Сборщик: `collectors/push_example.py`

Меняют одну функцию: `read_source`. Она возвращает строки одного законченного дня или ничего. `--demo` нужен только для первого просмотра.

```python
"""Example collector: reads numbers from a source and pushes them to the dashboard's /ingest.

Run:  python collectors/push_example.py             (push the last 3 days; safe to repeat, the server replaces them)
      python collectors/push_example.py --dry-run   (print what would be sent)
      python collectors/push_example.py --demo      (push 60 days of FAKE numbers, source «demo», to try the screens)
Environment: INGEST_URL (e.g. http://127.0.0.1:8000), INGEST_KEY (from ``python -m app.keys create``).
Real collectors replace ``read_source`` only: it must return the FINISHED day's numbers, or nothing when unsure.
"""

from __future__ import annotations

import argparse
import json
import logging
import os
import random
import sys
import urllib.error
import urllib.request
from datetime import date, datetime, timedelta
from typing import Any
from zoneinfo import ZoneInfo

logger = logging.getLogger("collector")
SOURCE = "example"
BUSINESS_TZ = ZoneInfo(os.environ.get("BUSINESS_TZ", "UTC"))  # the same value as the server's, set once


class _NoRedirect(urllib.request.HTTPRedirectHandler):
    """Never follow a redirect: urllib would resend the Authorization header to the new address."""

    def redirect_request(self, *args: Any, **kwargs: Any) -> None:
        return None


def read_source(day: date) -> list[dict[str, Any]]:
    """Return the rows of one finished day. REPLACE THIS with a real read (CRM, ad cabinet, bank export, sheet).

    A row is ``{"key", "day", "value"}`` plus an optional ``"dim"`` (a breakdown value such as a channel).
    Return [] when the source has no answer: a missing day stays empty, it is never sent as 0.
    """
    return []


def demo_rows(day: date, rng: random.Random) -> list[dict[str, Any]]:
    """Fake numbers for one day (only for ``--demo``)."""
    d = day.isoformat()
    leads = rng.randint(15, 40)
    deals = max(1, leads // rng.randint(4, 8))
    revenue = deals * rng.randint(9000, 15000)
    expenses = rng.randint(4000, 9000) + int(revenue * 0.3)
    rows = [("marketing.visits", leads * rng.randint(25, 40)), ("marketing.leads", leads),
            ("marketing.spend", leads * rng.randint(150, 300)), ("sales.deals", deals), ("sales.revenue", revenue),
            ("finance.income", revenue), ("finance.expenses", expenses),
            ("finance.cash_balance", 500000 + (day.toordinal() % 60) * 3000)]
    out: list[dict[str, Any]] = [{"key": k, "day": d, "value": v} for k, v in rows]
    for channel, share in (("telegram", 0.5), ("сайт", 0.3), ("реклама", 0.2)):
        out.append({"key": "marketing.leads", "day": d, "dim": channel, "value": round(leads * share)})
    return out


def push(url: str, key: str, source: str, rows: list[dict[str, Any]]) -> int:
    """POST rows in chunks; returns how many the server stored. The key is never logged."""
    opener = urllib.request.build_opener(_NoRedirect)
    stored = 0
    for i in range(0, len(rows), 1000):
        req = urllib.request.Request(url.rstrip("/") + "/ingest", method="POST",
                                     data=json.dumps({"source": source, "rows": rows[i:i + 1000]}).encode(),
                                     headers={"Content-Type": "application/json", "Authorization": f"Bearer {key}"})
        try:
            with opener.open(req, timeout=30) as resp:
                stored += int(json.load(resp)["stored"])
        except urllib.error.HTTPError as exc:
            raise RuntimeError(f"server answered {exc.code}: {exc.read(300).decode(errors='replace')}") from None
        except (urllib.error.URLError, TimeoutError, OSError, ValueError) as exc:
            raise RuntimeError(type(exc).__name__) from None
    return stored


def main() -> int:
    """Entry point; a non-zero exit code makes the scheduler (systemd) show the run as failed."""
    logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(name)s %(message)s")
    parser = argparse.ArgumentParser()
    parser.add_argument("--dry-run", action="store_true")
    parser.add_argument("--demo", action="store_true")
    args = parser.parse_args()
    today = datetime.now(BUSINESS_TZ).date()
    if args.demo:
        rng, source = random.Random(1), "demo"
        rows = [r for i in range(1, 61) for r in demo_rows(today - timedelta(days=i), rng)]
    else:
        source = SOURCE
        rows = [r for i in (1, 2, 3) for r in read_source(today - timedelta(days=i))]
    if args.dry_run:
        print(json.dumps({"source": source, "rows": rows[:5], "count": len(rows)}, ensure_ascii=False, indent=1))
        return 0
    try:
        url, key = os.environ["INGEST_URL"], os.environ["INGEST_KEY"]
    except KeyError as exc:
        logger.error("missing environment variable %s", exc.args[0])
        return 2
    if not rows:
        logger.warning("nothing to send")
        return 0
    try:
        logger.info("stored %d rows", push(url, key, source, rows))
    except RuntimeError as exc:
        logger.error("push failed: %s", exc)
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

## 8. Экран без Telegram: `tools/dev_server.py`

```python
"""DEV ONLY: open the dashboard in an ordinary browser with a FAKE Telegram user and demo numbers.

Run from the project folder:  python tools/dev_server.py   then open http://127.0.0.1:8765/dev
It uses a throwaway database (dev.db) and a fake bot token, listens on 127.0.0.1 only, and must never be deployed.
"""

from __future__ import annotations

import random
import sys
import time
from datetime import date, timedelta
from pathlib import Path
from zoneinfo import ZoneInfo

import uvicorn
from fastapi.responses import HTMLResponse
from fastapi.testclient import TestClient

ROOT = Path(__file__).resolve().parent.parent
sys.path[:0] = [str(ROOT), str(ROOT / "collectors")]

from app import auth  # noqa: E402
from app.db import connect, migrate  # noqa: E402
from app.main import STATIC, create_app  # noqa: E402
from app.settings import Settings  # noqa: E402
from push_example import demo_rows  # noqa: E402
from tests.test_dashboard import sign_init_data  # noqa: E402

FAKE_TOKEN, FAKE_USER = "123456:DEV-ONLY-FAKE-TOKEN", 1


def main() -> None:
    """Create the app, fill it with demo data through the real /ingest, serve /dev."""
    db = ROOT / "dev.db"
    db.unlink(missing_ok=True)
    settings = Settings(bot_token=FAKE_TOKEN, session_secret=b"d" * 40, allowed_tg_ids=frozenset({FAKE_USER}),
                        db_path=db, tz=ZoneInfo("UTC"), init_data_max_age_s=86400)
    app = create_app(settings)
    key = auth.new_ingest_key()
    conn = connect(db)
    migrate(conn, settings.allowed_tg_ids)
    with conn:
        conn.execute("INSERT INTO ingest_keys(name, key_hash) VALUES ('dev', ?)", (auth.sha256_hex(key),))
    conn.close()
    rng, today = random.Random(1), date.today()
    rows = [r for i in range(1, 91) for r in demo_rows(today - timedelta(days=i), rng)]
    reply = TestClient(app).post("/ingest", json={"source": "demo", "rows": rows},
                                 headers={"Authorization": f"Bearer {key}"})
    assert reply.status_code == 200, reply.text

    @app.get("/dev")
    def dev_page() -> HTMLResponse:
        stub = ("<script>window.Telegram = {WebApp: {initData: %r, ready() {}, expand() {}}};</script>"
                % sign_init_data(FAKE_USER, int(time.time()), FAKE_TOKEN))
        html = (STATIC / "index.html").read_text(encoding="utf-8")
        return HTMLResponse(html.replace('<script src="https://telegram.org/js/telegram-web-app.js"></script>', stub))

    uvicorn.run(app, host="127.0.0.1", port=8765, log_level="warning")


if __name__ == "__main__":
    main()
```

## 9. Тесты

Подпись initData в тестах собрана вручную по описанию Telegram, независимо от кода проверки. Перед тем как верить тестам, один раз сломайте проверку подписи (например, замените условие на `if False:`): хотя бы один тест обязан упасть.

### `tests/test_dashboard.py`

```python
"""Checks for the dashboard skeleton. initData is signed here by hand, following the Telegram documentation."""

from __future__ import annotations

import hashlib
import hmac
import json
import time
from pathlib import Path
from typing import Any
from urllib.parse import urlencode
from zoneinfo import ZoneInfo

import pytest
from fastapi.testclient import TestClient

from app import auth
from app.db import connect
from app.main import create_app
from app.settings import Settings

BOT_TOKEN = "123456:TEST-TOKEN-NOT-REAL"
OWNER = 111
STRANGER = 222


def sign_init_data(user_id: int, auth_date: int, token: str = BOT_TOKEN, extra: dict[str, str] | None = None) -> str:
    """Build a signed initData string exactly as the documentation describes."""
    fields = {"auth_date": str(auth_date), "query_id": "AAH-test", "user": json.dumps({"id": user_id, "first_name": "Test"}),
              **(extra or {})}
    check_string = "\n".join(sorted(f"{k}={v}" for k, v in fields.items()))
    secret = hmac.new(b"WebAppData", token.encode(), hashlib.sha256).digest()
    fields["hash"] = hmac.new(secret, check_string.encode(), hashlib.sha256).hexdigest()
    return urlencode(fields)


@pytest.fixture()
def env(tmp_path: Path) -> tuple[TestClient, Settings]:
    settings = Settings(bot_token=BOT_TOKEN, session_secret=b"s" * 40, allowed_tg_ids=frozenset({OWNER}),
                        db_path=tmp_path / "t.db", tz=ZoneInfo("UTC"))
    app = create_app(settings)
    return TestClient(app), settings


def login(client: TestClient, user_id: int = OWNER) -> str:
    r = client.post("/api/login", json={"init_data": sign_init_data(user_id, int(time.time()))})
    assert r.status_code == 200, r.text
    return str(r.json()["token"])


def make_key(settings: Settings, name: str = "test") -> str:
    key = auth.new_ingest_key()
    conn = connect(settings.db_path)
    with conn:
        conn.execute("INSERT INTO ingest_keys(name, key_hash) VALUES (?,?)", (name, auth.sha256_hex(key)))
    conn.close()
    return key


def test_valid_init_data_accepted() -> None:
    now = int(time.time())
    user = auth.validate_init_data(sign_init_data(OWNER, now), BOT_TOKEN, now)
    assert user["id"] == OWNER


def test_signature_field_stays_in_check_string() -> None:
    now = int(time.time())
    data = sign_init_data(OWNER, now, extra={"signature": "abc_DEF-123"})
    assert auth.validate_init_data(data, BOT_TOKEN, now)["id"] == OWNER


@pytest.mark.parametrize("mutate", ["token", "tamper_user", "no_hash", "expired", "future", "empty", "duplicate"])
def test_bad_init_data_refused(mutate: str) -> None:
    now = int(time.time())
    data = sign_init_data(OWNER, now)
    token = BOT_TOKEN
    if mutate == "token":
        token = "999:OTHER"
    elif mutate == "tamper_user":
        data = data.replace("%22id%22%3A+111", "%22id%22%3A+222")
        assert "222" in data
    elif mutate == "no_hash":
        data = "&".join(p for p in data.split("&") if not p.startswith("hash="))
    elif mutate == "expired":
        data = sign_init_data(OWNER, now - 3600)
    elif mutate == "future":
        data = sign_init_data(OWNER, now + 3600)
    elif mutate == "empty":
        data = ""
    elif mutate == "duplicate":
        data = data + "&auth_date=1"
    with pytest.raises(auth.InitDataError):
        auth.validate_init_data(data, token, now)


def test_login_flow_and_allowlist(env: tuple[TestClient, Settings]) -> None:
    client, _ = env
    assert client.get("/api/summary").status_code == 401  # no token
    token = login(client)
    assert client.get("/api/summary", headers={"Authorization": f"Bearer {token}"}).status_code == 200
    stranger = client.post("/api/login", json={"init_data": sign_init_data(STRANGER, int(time.time()))})
    assert stranger.status_code == 403  # genuine Telegram signature, but not on the allowlist
    forged = client.post("/api/login", json={"init_data": sign_init_data(OWNER, int(time.time()), token="1:x")})
    assert forged.status_code == 401


def test_revoked_user_loses_access_at_once(env: tuple[TestClient, Settings]) -> None:
    client, settings = env
    token = login(client)
    conn = connect(settings.db_path)
    with conn:
        conn.execute("UPDATE allowed_users SET revoked_at=datetime('now') WHERE tg_id=?", (OWNER,))
    conn.close()
    assert client.get("/api/summary", headers={"Authorization": f"Bearer {token}"}).status_code == 403


def test_bad_session_tokens_refused(env: tuple[TestClient, Settings]) -> None:
    client, settings = env
    now = time.time()
    good = auth.make_session(settings.session_secret, OWNER, int(now) + 100)
    assert auth.read_session(settings.session_secret, good, now) == OWNER
    assert auth.read_session(settings.session_secret, good, now + 1000) is None  # expired
    flipped = good[:-1] + ("1" if good[-1] == "0" else "0")  # always a different last character
    assert auth.read_session(settings.session_secret, flipped, now) is None  # tampered
    assert auth.read_session(b"x" * 40, good, now) is None  # other secret
    assert auth.read_session(settings.session_secret, "garbage", now) is None


def test_ingest_needs_a_known_key(env: tuple[TestClient, Settings]) -> None:
    client, settings = env
    body: dict[str, Any] = {"source": "test", "rows": [{"key": "sales.deals", "day": "2020-01-01", "value": 3}]}
    assert client.post("/ingest", json=body).status_code == 401
    assert client.post("/ingest", json=body, headers={"Authorization": "Bearer nope"}).status_code == 401
    key = make_key(settings)
    day = time.strftime("%Y-%m-%d", time.gmtime(time.time() - 86400 * 3))
    body["rows"][0]["day"] = day
    ok = client.post("/ingest", json=body, headers={"Authorization": f"Bearer {key}"})
    assert ok.status_code == 200 and ok.json() == {"stored": 1}
    conn = connect(settings.db_path)
    stored = conn.execute("SELECT key_hash FROM ingest_keys").fetchone()[0]
    assert stored == auth.sha256_hex(key) and key not in stored  # only the hash is kept


def test_ingest_is_idempotent_and_validated(env: tuple[TestClient, Settings]) -> None:
    client, settings = env
    h = {"Authorization": f"Bearer {make_key(settings)}"}
    day = time.strftime("%Y-%m-%d", time.gmtime(time.time() - 86400))

    def push(rows: list[dict[str, object]]) -> int:
        return client.post("/ingest", json={"source": "t", "rows": rows}, headers=h).status_code

    assert push([{"key": "sales.deals", "day": day, "value": 3}]) == 200
    assert push([{"key": "sales.deals", "day": day, "value": 5}]) == 200  # same day again: replaced
    conn = connect(settings.db_path)
    rows = conn.execute("SELECT value FROM metric_values WHERE key='sales.deals'").fetchall()
    assert [r[0] for r in rows] == [5.0]
    assert push([{"key": "sales.nope", "day": day, "value": 1}]) == 422  # unknown metric
    assert push([{"key": "sales.deals", "day": "2999-01-01", "value": 1}]) == 422  # future day
    assert push([{"key": "sales.deals", "day": "2020-02-30", "value": 1}]) == 422  # not a date
    assert push([{"key": "sales.deals", "day": day, "value": -1}]) == 422  # negative on a sum metric
    assert push([{"key": "finance.cash_balance", "day": day, "value": -50}]) == 200  # a balance may be negative
    assert client.post("/ingest", json={"source": "t", "rows": [{"key": "sales.deals", "day": day, "value": "NaN"}]},
                       headers=h).status_code == 422
    assert conn.execute("SELECT COUNT(*) FROM ingest_log").fetchone()[0] == 3


def test_summary_and_department_numbers(env: tuple[TestClient, Settings]) -> None:
    client, settings = env
    h = {"Authorization": f"Bearer {make_key(settings)}"}
    today = time.time()
    rows: list[dict[str, object]] = []
    for i in range(1, 15):  # 14 full days back from yesterday: 7 current + 7 previous
        day = time.strftime("%Y-%m-%d", time.gmtime(today - 86400 * i))
        current = i <= 7
        rows += [{"key": "marketing.leads", "day": day, "value": 20 if current else 10},
                 {"key": "marketing.spend", "day": day, "value": 1000},
                 {"key": "sales.deals", "day": day, "value": 2},
                 {"key": "sales.revenue", "day": day, "value": 500},
                 {"key": "finance.income", "day": day, "value": 500},
                 {"key": "finance.expenses", "day": day, "value": 300},
                 {"key": "finance.cash_balance", "day": day, "value": 10000 + (100 - i)},
                 {"key": "marketing.leads", "day": day, "value": 12, "dim": "telegram"}]
    assert client.post("/ingest", json={"source": "t", "rows": rows}, headers=h).status_code == 200
    s = {"Authorization": f"Bearer {login(client)}"}
    cards = {c["key"]: c for c in client.get("/api/summary?days=7", headers=s).json()["cards"]}
    assert cards["marketing.leads"]["value"] == 140 and cards["marketing.leads"]["prev"] == 70
    assert cards["marketing.leads"]["delta_pct"] == 100.0
    assert cards["marketing.cost_per_lead"]["value"] == pytest.approx(7000 / 140)  # ratio of totals, not of daily ratios
    assert cards["finance.profit"]["value"] == 7 * 200
    assert cards["finance.cash_balance"]["value"] == 10000 + 99  # the last day of the window (yesterday), not a sum
    dep = client.get("/api/department/marketing?days=7", headers=s).json()
    assert dep["breakdown"]["marketing.leads"] == [{"dim": "telegram", "value": 84.0}]
    assert len(dep["series"]["marketing.leads"]) == 7
    assert client.get("/api/department/nope", headers=s).status_code == 404
    assert client.get("/api/summary?days=5", headers=s).status_code == 422


def test_missing_data_is_null_not_zero(env: tuple[TestClient, Settings]) -> None:
    client, _ = env
    s = {"Authorization": f"Bearer {login(client)}"}
    body = client.get("/api/summary?days=7", headers=s).json()
    assert all(c["value"] is None and c["delta_pct"] is None for c in body["cards"])
    assert all(p["value"] is None for p in body["chart"]["points"])


def test_validation_errors_do_not_echo_input(env: tuple[TestClient, Settings]) -> None:
    client, _ = env
    r = client.post("/api/login", json={"init_data": "", "secret_marker": "LEAK-ME"})
    assert r.status_code == 422 and "LEAK-ME" not in r.text and "input" not in r.text


def test_index_and_headers(env: tuple[TestClient, Settings]) -> None:
    client, _ = env
    assert client.get("/healthz").json() == {"ok": True}
    assert "Cache-Control" in client.get("/").headers
    assert client.get("/api/summary").headers["cache-control"] == "no-store"
```

### `tests/test_bot.py`

```python
"""The bot answers allowlisted people with a Mini App button and everyone else with a refusal (no network)."""

from __future__ import annotations

from typing import Any

import pytest

from bot import bot as bot_module

URL = "https://dashboard.example.com/"


@pytest.fixture()
def sent(monkeypatch: pytest.MonkeyPatch) -> list[tuple[str, dict[str, Any]]]:
    calls: list[tuple[str, dict[str, Any]]] = []
    monkeypatch.setattr(bot_module, "call", lambda token, method, body, timeout=40: calls.append((method, body)))
    return calls


def update(user_id: int, chat_type: str = "private") -> dict[str, Any]:
    return {"update_id": 1, "message": {"chat": {"id": user_id, "type": chat_type}, "from": {"id": user_id},
                                        "text": "/start"}}


def test_allowed_user_gets_web_app_button(sent: list[tuple[str, dict[str, Any]]]) -> None:
    bot_module.handle("t", update(111), frozenset({111}), URL)
    method, body = sent[0]
    button = body["reply_markup"]["inline_keyboard"][0][0]
    assert method == "sendMessage" and button["web_app"] == {"url": URL}


def test_stranger_is_refused_and_groups_ignored(sent: list[tuple[str, dict[str, Any]]]) -> None:
    bot_module.handle("t", update(222), frozenset({111}), URL)
    assert sent[0][1]["text"] == bot_module.NO_ACCESS and "reply_markup" not in sent[0][1]
    bot_module.handle("t", update(111, chat_type="group"), frozenset({111}), URL)
    assert len(sent) == 1


def test_menu_button_is_a_web_app(sent: list[tuple[str, dict[str, Any]]]) -> None:
    bot_module.set_menu_buttons("t", frozenset({111}), URL)
    method, body = sent[0]
    assert method == "setChatMenuButton" and body["chat_id"] == 111
    assert body["menu_button"]["type"] == "web_app" and body["menu_button"]["web_app"]["url"] == URL
```

## 10. Деплой: службы и HTTPS

Нужны: сервер на Linux с systemd (Ubuntu или Debian), домен с записью A на адрес сервера, открытые порты 80 и 443. Выполняйте только после «да» владельца.

**Пользователь, код, окружение**

```bash
sudo useradd --system --home /opt/dashboard --shell /usr/sbin/nologin dashboard
sudo mkdir -p /opt/dashboard /etc/dashboard
sudo rsync -a --exclude venv --exclude '*.db' --exclude __pycache__ ./ /opt/dashboard/
cd /opt/dashboard && sudo python3 -m venv venv && sudo venv/bin/pip install -r requirements.txt
```

Код принадлежит root и доступен службе только на чтение. База и копии пишутся в `/var/lib/dashboard`.

**Файлы настроек** (создать пустыми, права `640`, владелец root и группа dashboard; заполнять на сервере, не через чат):

```bash
for f in backend bot collector; do
  sudo install -m 640 -o root -g dashboard /dev/null /etc/dashboard/$f.env
done
```

| Файл | Что внутри |
|---|---|
| `backend.env` | `BOT_TOKEN`, `ALLOWED_TG_IDS`, `SESSION_SECRET`, `DB_PATH=/var/lib/dashboard/dashboard.db`, `BUSINESS_TZ`, `CURRENCY` |
| `bot.env` | `BOT_TOKEN`, `ALLOWED_TG_IDS`, `PUBLIC_URL=https://<домен>` |
| `collector.env` | `INGEST_URL=http://127.0.0.1:8000`, `INGEST_KEY`, `BUSINESS_TZ` |

`SESSION_SECRET` создают так: `python3 -c "import secrets; print(secrets.token_urlsafe(32))"`. Не выводите значения на экран и в логи.

**Ключ для сборщика** (печатается один раз; сразу записать в `collector.env`):

```bash
sudo -u dashboard bash -c 'set -a; . /etc/dashboard/backend.env; set +a; cd /opt/dashboard && venv/bin/python -m app.keys create collector'
```

Список и отзыв: `... -m app.keys list`, `... -m app.keys revoke collector`.

**Службы**

```bash
sudo cp deploy/dashboard.service deploy/dashboard-bot.service deploy/dashboard-collector.service \
        deploy/dashboard-collector.timer deploy/dashboard-backup.service deploy/dashboard-backup.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now dashboard dashboard-bot dashboard-collector.timer dashboard-backup.timer
```

Файлы `deploy/*`:

`dashboard.service` (backend, слушает только `127.0.0.1:8000`):

```ini
[Unit]
Description=Business dashboard backend
After=network-online.target
Wants=network-online.target

[Service]
User=dashboard
Group=dashboard
WorkingDirectory=/opt/dashboard
EnvironmentFile=/etc/dashboard/backend.env
ExecStart=/opt/dashboard/venv/bin/uvicorn app.main:create_app --factory --host 127.0.0.1 --port 8000 --proxy-headers --forwarded-allow-ips 127.0.0.1 --no-access-log
Restart=always
RestartSec=3
# /var/lib/dashboard is created for this user and is the only writable place
StateDirectory=dashboard
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

`dashboard-bot.service`:

```ini
[Unit]
Description=Business dashboard Telegram bot
After=network-online.target
Wants=network-online.target

[Service]
User=dashboard
Group=dashboard
WorkingDirectory=/opt/dashboard
EnvironmentFile=/etc/dashboard/bot.env
ExecStart=/opt/dashboard/venv/bin/python bot/bot.py
Restart=always
RestartSec=5
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

`dashboard-collector.service` и `dashboard-collector.timer` (раз в час, на 10-й минуте):

```ini
[Unit]
Description=Business dashboard collector (pushes daily numbers)
After=network-online.target dashboard.service
Wants=network-online.target

[Service]
Type=oneshot
User=dashboard
Group=dashboard
WorkingDirectory=/opt/dashboard
EnvironmentFile=/etc/dashboard/collector.env
ExecStart=/opt/dashboard/venv/bin/python collectors/push_example.py
TimeoutStartSec=600
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
```

```ini
[Unit]
Description=Run the dashboard collector every hour

[Timer]
OnCalendar=*-*-* *:10:00
Persistent=true

[Install]
WantedBy=timers.target
```

`dashboard-backup.service` и `dashboard-backup.timer` (копия базы раз в сутки; увезти её с сервера – решение владельца, например `rsync` или `restic`; нужен пакет `sqlite3`):

```ini
[Unit]
Description=Business dashboard database backup (a consistent copy while the app runs)

[Service]
Type=oneshot
User=dashboard
Group=dashboard
# /var/lib/dashboard/backup is created for this user; copy its content off the server with your own tool
StateDirectory=dashboard dashboard/backup
ExecStart=/usr/bin/sqlite3 /var/lib/dashboard/dashboard.db ".backup '/var/lib/dashboard/backup/dashboard.db'"
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
```

```ini
[Unit]
Description=Daily dashboard database backup

[Timer]
OnCalendar=*-*-* 03:10:00
Persistent=true

[Install]
WantedBy=timers.target
```

**HTTPS через Caddy.** Caddy сам получает и продлевает сертификат, когда домен смотрит на сервер, а порты 80 и 443 открыты. Установите Caddy по инструкции с его сайта, замените `dashboard.example.com` на свой домен:

```caddyfile
dashboard.example.com {
	encode zstd gzip
	header {
		Strict-Transport-Security "max-age=31536000"
		X-Content-Type-Options "nosniff"
		Referrer-Policy "no-referrer"
		-Server
	}
	# Collectors run on this machine and call 127.0.0.1:8000 directly, so the public site does not serve /ingest.
	@ingest path /ingest /ingest/*
	respond @ingest 404
	reverse_proxy 127.0.0.1:8000
}
```

```bash
sudo cp deploy/Caddyfile /etc/caddy/Caddyfile      # edit the domain first
sudo caddy validate --config /etc/caddy/Caddyfile && sudo systemctl reload caddy
```

**Кнопка меню бота** (после того, как адрес открывается):

```bash
sudo -u dashboard bash -c 'set -a; . /etc/dashboard/bot.env; set +a; cd /opt/dashboard && venv/bin/python bot/bot.py set-menu'
```

**Проверка с другого компьютера:**

```bash
curl -s https://<домен>/healthz                                  # {"ok":true}
curl -s -o /dev/null -w '%{http_code}\n' https://<домен>/ingest  # must be 404
curl -s -o /dev/null -w '%{http_code}\n' https://<домен>/api/summary   # must be 401
```

Затем владелец открывает бота с телефона, нажимает «Дашборд» и видит экран. Чужой аккаунт должен получить отказ.

**Обновление:** скопировать код (`rsync` как выше), `sudo systemctl restart dashboard dashboard-bot`. Схема базы обновляется сама при старте.
