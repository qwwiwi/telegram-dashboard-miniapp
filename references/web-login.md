# Вход с обычной страницы: одноразовая ссылка и код из Telegram

Это необязательное расширение каркаса из `starter.md`. По умолчанию дашборд открывается только внутри Telegram как мини-приложение. Здесь добавляется второй вход: владелец открывает дашборд в обычном браузере (на компьютере, в ноутбуке), нажимает «Перейти в Telegram», а бот присылает ему **одноразовую ссылку** и **6-значный код**. Пароль не нужен. Нужны ссылка или код: хватает одного.

Код ниже написан на каркасе и проверен тестами на SQLite и на PostgreSQL (в конце файла: что проверено, что нет). Берите его целиком: файлы согласованы между собой.

## 1. Когда это нужно

Берите расширение, если выполняется хотя бы одно:

- дашборд хотят открывать на компьютере в обычной вкладке, на большом экране, а не только в телефоне;
- ссылку на дашборд нужно уметь открыть без Telegram (например, в браузере рабочего ноутбука);
- мини-приложение внутри Telegram не подходит (корпоративный запрет, веб-версия Telegram не открывает его).

Если дашборд нужен только в телефоне, не добавляйте: чем меньше входов, тем меньше поверхность атаки. Вход из мини-приложения не меняется и работает рядом.

Слова, которые нужны дальше:
- **Попытка входа (attempt)** – запись в базе: «этот браузер хочет войти». У неё есть одноразовая метка (nonce), срок жизни и счётчик неверных кодов.
- **Привязка к браузеру** – куки, которая выдаётся при старте попытки и подтверждает: код вводит тот же браузер, что начал вход.
- **Одноразовая ссылка (magic link)** – адрес с длинным случайным токеном. Работает один раз и недолго.
- **Сессия** – запись в базе после входа. Браузер хранит её токен в куки, база – только его хеш.
- **Токен подтверждения (csrf)** – скрытое поле кнопки «Войти» на странице одноразовой ссылки. Оно считается от куки этого браузера и от самой ссылки ключом сервера, поэтому чужая страница не может отправить такую кнопку от имени вашего браузера.

## 2. Как это работает

```
1. Браузер открывает /login. Нажимает «Войти через Telegram».
   POST /auth/start -> сервер создаёт попытку, ставит куки-привязку (dash_login),
   отдаёт ссылку https://t.me/<your_bot>?start=login_<метка>.
2. Страница показывает кнопку «Перейти в Telegram» (эта ссылка) и поле для кода.
3. Владелец открывает бота по ссылке. Бот получает «/start login_<метка>», проверяет
   Telegram-id по списку, вызывает закрытый адрес backend /internal/login/issue (с общим ключом).
4. Backend сверяет id с allowlist, сохраняет хеши ссылки и кода и возвращает ссылку, код и
   «откуда запрос» (браузер и IP). Бот отправляет всё это владельцу в личный чат.
5a. Владелец нажимает ссылку: страница просит нажать «Войти» (так предпросмотр ссылки в
    Telegram не тратит токен). Страница ставит этому браузеру куки dash_csrf и кладёт в форму
    скрытое поле csrf. POST /auth/magic/<токен> проходит, только если (1) Origin (или Referer)
    наш и (2) поле csrf совпадает с куки и ссылкой. Тогда ссылка тратится один раз -> сессия,
    куки dash_session.
5b. Или вводит код на странице: POST /auth/code. Работает только в браузере с куки-привязкой,
    недолго; 5 неверных кодов сжигают попытку.
6. Сессия: случайный токен в куки (HttpOnly, Secure, SameSite=Lax), в базе только sha256,
   срок жизни, выход (POST /auth/logout), «выйти везде» (команда бота /logout_all).
   Куки принимается, только пока расширение включено.
```

Зачем две проверки на подтверждении. Без них чужая страница могла бы отправить форму с **чужой** (злоумышленника) действующей ссылкой из браузера жертвы: жертва оказалась бы вошедшей под чужим Telegram-id и дальше вносила бы свои данные в чужой аккаунт («login CSRF»). `SameSite=Lax` этого не останавливает: он запрещает отправлять куки, но не запрещает браузеру принять новую. Поэтому: (1) любой POST под `/auth/...` принимается только со своей страницы (заголовок Origin, а если его нет, Referer; нет обоих, `null` или чужой адрес – 403); (2) подтверждение ссылки требует токен подтверждения.

Команда `/login` в боте даёт только ссылку, без кода: код действует лишь в браузере, который начал попытку.

Что остаётся от каркаса без изменений: проверка initData для мини-приложения, allowlist, ключи сборщиков. Новый вход использует тот же allowlist: отозванный человек теряет и веб-сессию, и мини-приложение сразу.

## 3. Какие файлы меняются

| Файл | Что |
|---|---|
| `app/weblogin.py` | **новый**: вся логика (попытки, коды, сессии) и адреса `/auth/...`, `/login`, `/internal/...` |
| `app/settings.py` | правка: шесть новых настроек (в том числе ключ сервера `web_hmac_key`) |
| `app/main.py` | правка: четыре места (импорт, `current_user`, создание таблиц, подключение адресов) |
| `static/login.html` | **новый**: страница входа |
| `static/index.html` | правка: в обычном браузере брать цифры по куки и отправлять на `/login`; кнопка «Выйти» |
| `bot/bot.py` | правка: команды `/start login_...`, `/login`, `/logout_all` |
| `tests/test_web_login.py` | **новый**: тесты |
| `deploy/Caddyfile`, файлы настроек | правка: закрыть `/internal` снаружи, новые переменные (в том числе `WEB_LOGIN_HMAC_KEY`) |

Расширение включается настройками: пока их нет, адреса не подключаются, и каркас работает как раньше.

## 4. Настройки

В `.env.example` (и в файлах настроек на сервере) добавьте:

```bash
# Web login (references/web-login.md). Backend needs the first four; the bot needs the last two.
PUBLIC_URL=https://dashboard.example.com
WEB_LOGIN_BOT_USERNAME=<your_bot>
# Shared secret between the bot and the backend: python -c "import secrets; print(secrets.token_urlsafe(32))"
WEB_LOGIN_INTERNAL_KEY=<<random string, 32+ bytes>>
# Backend ONLY (not in bot.env, not in the database): key of the code hashes and of the confirm-form tokens.
# A different value from the one above: python -c "import secrets; print(secrets.token_urlsafe(32))"
WEB_LOGIN_HMAC_KEY=<<another random string, 32+ bytes>>
# Bot only: where the backend listens (loopback)
WEB_LOGIN_INTERNAL_URL=http://127.0.0.1:8000
```

Кто что читает:

| Файл настроек | Новые переменные |
|---|---|
| `backend.env` | `PUBLIC_URL`, `WEB_LOGIN_BOT_USERNAME` (без `@`), `WEB_LOGIN_INTERNAL_KEY`, `WEB_LOGIN_HMAC_KEY` |
| `bot.env` | `WEB_LOGIN_INTERNAL_URL`, `WEB_LOGIN_INTERNAL_KEY` (то же значение, что у backend) |

`PUBLIC_URL` backend нужен, чтобы собрать адрес одноразовой ссылки и чтобы знать, какой Origin считать «своим». Оба ключа – секреты: только в файлы настроек, не в чат, не в git, не в логи.

Ключ `WEB_LOGIN_HMAC_KEY` нужен потому, что кодов всего миллион: хеш без ключа перебирается за секунды, и копия базы раскрыла бы код. С ключом, которого в базе нет, одной копии базы недостаточно. Ключ длиной меньше 32 байт или пустой при включённом расширении: приложение **не запускается** (в ошибке названа переменная, значения нет). Смена ключа безопасна: текущие попытки входа перестанут подходить (их срок 5 минут), сессии остаются.

## 5. Таблицы

Две новые таблицы. Время в них – секунды Unix (UTC), числом: так срок жизни считается простым сравнением с часами приложения (`app.state.clock`), а тесты подменяют часы. Это отличие от текстовых дат в `database.md`; метрик оно не касается.

Текст SQL один и тот же для SQLite и PostgreSQL, как проверено в тестах: `BIGINT` и `DOUBLE PRECISION` SQLite принимает как `INTEGER` и `REAL` (по правилам сродства типов), а PostgreSQL получает те типы, что нужны (`postgres.md`: `tg_id` – `BIGINT`, дробные – `DOUBLE PRECISION`).

| Колонка | SQLite | PostgreSQL | Смысл |
|---|---|---|---|
| `tg_id`, `issued_tg` | `INTEGER` (из `BIGINT`) | `BIGINT` | Telegram-id бывают больше 4 байт |
| `created`, `expires`, `consumed_at`, `revoked_at` | `REAL` (из `DOUBLE PRECISION`) | `DOUBLE PRECISION` | секунды Unix, UTC |
| `failed` | `INTEGER` | `INTEGER` | неверные коды |
| хеши, `ip`, `browser` | `TEXT` | `TEXT` | – |

Схема лежит в `app/weblogin.py` (список `SCHEMA`), создаётся при старте приложения, повтор безопасен (`IF NOT EXISTS`). Что хранится:

- `login_attempts`: хеш метки, хеш привязки к браузеру, хеш ссылки, HMAC кода (ключ сервера `WEB_LOGIN_HMAC_KEY` над строкой «хеш метки:код»; ключа в базе нет), кому выдано, срок, счётчик неверных кодов, отметка «использовано». Ни метки, ни токена, ни кода в открытом виде. Токены подтверждения в базе не хранятся вообще.
- `web_sessions`: хеш токена, кому принадлежит, срок, отметка отзыва.

Если в проекте уже переведено на Postgres по `postgres.md`, отдельных правок не нужно: код пишет через те же `conn.execute(...)` с `?`.

## 6. Код

Применяйте файлы по порядку. Для правок: найдите блок «Найти» (он встречается в файле один раз) и замените его блоком «Заменить на». Правки подходят и к варианту каркаса на SQLite, и к варианту из `postgres.md`.

### 6.1. Настройки

#### `app/settings.py` (правка 1: поля)

Найти:
```python
    session_ttl_s: int = 12 * 3600
```
Заменить на:
```python
    session_ttl_s: int = 12 * 3600
    # Web login (references/web-login.md). It is switched on only when the first three are set.
    web_bot_username: str = ""  # the bot's @username without «@»
    public_url: str = ""  # https://dashboard.example.com, no trailing slash
    internal_key: str = ""  # shared secret between the bot and the backend (min 32 bytes)
    # Server key of the code hashes and of the confirm-form tokens. Lives only in backend.env, never in the database,
    # never in the bot's file. When web login is on and it is missing or shorter than 32 bytes, the app does not start.
    web_hmac_key: str = ""
    login_ttl_s: int = 300  # life of a login attempt, of a one-time link and of a code
    web_session_ttl_s: int = 7 * 86400

    @property
    def web_login_enabled(self) -> bool:
        """True when everything the web login needs is configured."""
        return bool(self.web_bot_username and self.public_url and self.internal_key)
```

#### `app/settings.py` (правка 2: чтение из окружения)

Найти:
```python
        currency=env.get("CURRENCY", "₽"),
    )
```
Заменить на:
```python
        currency=env.get("CURRENCY", "₽"),
        web_bot_username=env.get("WEB_LOGIN_BOT_USERNAME", "").strip().lstrip("@"),
        public_url=env.get("PUBLIC_URL", "").strip().rstrip("/"),
        internal_key=env.get("WEB_LOGIN_INTERNAL_KEY", "").strip(),
        web_hmac_key=env.get("WEB_LOGIN_HMAC_KEY", "").strip(),
    )
```

### 6.2. Логика и адреса

Один новый модуль. Почему адреса собираются функцией `build_router(...)`, а не лежат в `main.py`: модулю нужны `get_conn`, `get_settings` и `now_ts` из `main.py`, а `main.py` импортирует модуль. Передача их аргументами убирает круговой импорт и не дублирует код.

Что здесь важно для безопасности (каждое свойство проверяет тест из раздела 8):
- токены – 256 случайных бит, в базе только sha256; код хранится как HMAC-SHA256 от «хеш метки:код» с ключом сервера, которого нет в базе;
- любой POST под `/auth/...` принимается только с нашего Origin (если Origin нет, с нашего Referer), иначе 403 и ничего не меняется;
- подтверждение ссылки требует токен подтверждения, привязанный к браузеру (куки `dash_csrf`) и к ссылке;
- куки сессии принимается только при включённом расширении; присутствующий заголовок `Authorization` либо верный `Bearer`, либо 401;
- сравнение хешей – постоянного времени (`hmac.compare_digest`);
- ссылка и код одноразовые: «использовано» ставится одним условным `UPDATE`, поэтому две одновременные попытки не пройдут обе;
- 5 неверных кодов сжигают попытку (в том же `UPDATE`, без гонки);
- закрытые адреса `/internal/...` отвечают 404 всем, кроме локального бота с ключом;
- в логи не попадают id, токены и коды: только слова-статусы.

#### `app/weblogin.py`

```python
"""Web login for an ordinary browser: a one-time link and a 6-digit code sent by the Telegram bot.

1. POST /auth/start makes a login attempt (a nonce) and binds it to this browser with an HttpOnly cookie; the
   answer is the deep link t.me/<bot>?start=login_<nonce>.
2. The bot calls POST /internal/login/issue (loopback only, shared key). For an allowlisted id the backend stores
   the hashes of a one-time link and a 6-digit code and returns both, plus «where the request came from».
   /login in the bot asks for a link only (nonce=None): a code is valid only in the browser that started the attempt.
3. The user taps the link (GET shows a confirm page, POST consumes it: link previews never log in) or types the
   code (POST /auth/code, only in the bound browser). 5 wrong codes burn the attempt.
4. A session is a random token in a cookie; the database keeps its sha256.
Tokens are 256-bit and stored only as sha256. The code is stored as HMAC-SHA256("<nonce hash>:<code>") under a server
key that is not in the database: a database dump alone does not give the 6 digits.
Every POST under /auth must come from our own page (Origin, else Referer), and the confirm form carries a token bound
to the browser's cookie and to the link: a foreign page cannot log a victim's browser in under the attacker's id.
No id, token or code is ever logged.
"""

from __future__ import annotations

import hashlib
import hmac
import logging
import re
import secrets
from collections.abc import Callable
from dataclasses import dataclass
from pathlib import Path
from typing import Any, Protocol
from urllib.parse import parse_qs, urlsplit

from fastapi import APIRouter, Depends, HTTPException, Request
from fastapi.responses import FileResponse, HTMLResponse, JSONResponse, RedirectResponse, Response
from pydantic import BaseModel, Field

from app import auth
from app.settings import Settings

logger = logging.getLogger("dashboard")
STATIC = Path(__file__).resolve().parent.parent / "static"  # the same folder as in app.main
SESSION_COOKIE = "dash_session"
LOGIN_COOKIE = "dash_login"  # ties an attempt to the browser that started it; only sent to /auth/...
CSRF_COOKIE = "dash_csrf"  # the browser half of the confirm-form token; only sent to /auth/magic/...
TOKEN_RE = re.compile(r"^[A-Za-z0-9_-]{43}$")  # token_urlsafe(32)
MAX_CODE_FAILS = 5
LOOPBACK = frozenset({"127.0.0.1", "::1"})
NO_STORE = {"Cache-Control": "no-store", "Referrer-Policy": "no-referrer"}
TOO_MANY = "Слишком много попыток, подождите 10 минут"

# One text for SQLite and PostgreSQL: BIGINT / DOUBLE PRECISION are INTEGER / REAL to SQLite (type affinity).
# Times are Unix seconds (UTC), compared with the app clock.
SCHEMA = [
    """CREATE TABLE IF NOT EXISTS login_attempts (
      nonce_hash TEXT PRIMARY KEY,
      binding_hash TEXT UNIQUE,
      ip TEXT NOT NULL,
      browser TEXT NOT NULL,
      created DOUBLE PRECISION NOT NULL,
      expires DOUBLE PRECISION NOT NULL,
      issued_tg BIGINT,
      code_hash TEXT,
      magic_hash TEXT UNIQUE,
      failed INTEGER NOT NULL DEFAULT 0,
      consumed_at DOUBLE PRECISION
    )""",
    """CREATE TABLE IF NOT EXISTS web_sessions (
      token_hash TEXT PRIMARY KEY,
      tg_id BIGINT NOT NULL,
      created DOUBLE PRECISION NOT NULL,
      expires DOUBLE PRECISION NOT NULL,
      revoked_at DOUBLE PRECISION
    )""",
    "CREATE INDEX IF NOT EXISTS web_sessions_tg ON web_sessions(tg_id)",
]


class Conn(Protocol):
    """What this module needs from a connection: sqlite3.Connection and app.db.Db (postgres.md) both fit."""

    def execute(self, sql: str, params: tuple[Any, ...] = (), /) -> Any: ...

    def __enter__(self) -> Any: ...

    def __exit__(self, *exc: Any) -> Any: ...


def create_tables(conn: Conn) -> None:
    """Create the web-login tables (safe to repeat)."""
    with conn:
        for statement in SCHEMA:
            conn.execute(statement)


# ---------- tokens and codes ----------

def new_token() -> str:
    """A 256-bit url-safe token (43 characters): the same generator as the ingest keys."""
    return auth.new_ingest_key()


def new_code() -> str:
    """A 6-digit code, uniformly random (leading zeros kept)."""
    return f"{secrets.randbelow(1_000_000):06d}"


def code_hash(key: str, nonce_hash: str, code: str) -> str:
    """HMAC-SHA256 of «nonce hash:code» under the server key.

    Tied to the attempt (the same 6 digits give different hashes in different attempts) and keyed: only 10**6 codes
    exist, so a plain hash would fall to a million tries for anyone holding a copy of the database.
    """
    return hmac.new(key.encode(), f"{nonce_hash}:{code}".encode(), hashlib.sha256).hexdigest()


def csrf_field(key: str, cookie: str, token: str) -> str:
    """The hidden field of the confirm form: HMAC of this browser's cookie and of this link. Only the server makes it."""
    return hmac.new(key.encode(), f"csrf:{cookie}:{token}".encode(), hashlib.sha256).hexdigest()


def csrf_ok(key: str, cookie: str | None, token: str, field: str) -> bool:
    """True when ``field`` was made for this very cookie and this very link (constant-time comparison)."""
    if not cookie or not TOKEN_RE.fullmatch(cookie) or not field.isascii():
        return False
    return hmac.compare_digest(csrf_field(key, cookie, token).encode(), field.encode())


def origin_of(url: str) -> str:
    """«scheme://host[:port]» in lower case; empty when the text has no host («null», garbage)."""
    parts = urlsplit(url)
    return f"{parts.scheme}://{parts.netloc}".lower() if parts.scheme and parts.netloc else ""


def same_origin(origin: str | None, referer: str | None, public_url: str) -> bool:
    """A POST may come only from our own page: Origin, and Referer when there is no Origin at all.

    No Origin and no Referer, «null» and every foreign address are refused. A present Origin is never overridden by Referer.
    """
    expected = origin_of(public_url)
    if not expected:
        return False
    if origin is not None:
        return origin_of(origin) == expected
    return bool(referer) and origin_of(referer or "") == expected


_BROWSERS = (("YaBrowser/", "Яндекс Браузер"), ("Edg/", "Edge"), ("OPR/", "Opera"), ("Firefox/", "Firefox"),
             ("FxiOS/", "Firefox"), ("CriOS/", "Chrome"), ("Chrome/", "Chrome"), ("Safari/", "Safari"))
_SYSTEMS = (("iPhone", "iOS"), ("iPad", "iOS"), ("Android", "Android"), ("Mac OS X", "macOS"),
            ("Windows", "Windows"), ("Linux", "Linux"))


def browser_family(user_agent: str) -> str:
    """«Chrome, macOS» from a User-Agent. Only this short label is stored and shown; the raw header never is."""
    browser = next((name for mark, name in _BROWSERS if mark in user_agent), None)
    if browser is None:
        return "неизвестный браузер"
    system = next((name for mark, name in _SYSTEMS if mark in user_agent), None)
    return f"{browser}, {system}" if system else browser


# ---------- attempts ----------

@dataclass(frozen=True)
class Started:
    """A new attempt: ``nonce`` goes into the deep link, ``binding`` into the browser's cookie."""

    nonce: str
    binding: str


@dataclass(frozen=True)
class Issued:
    """Result of issuing a link (and a code). ``status``: issued | forbidden | unknown | used | expired."""

    status: str
    token: str | None = None
    code: str | None = None
    ip: str | None = None
    browser: str | None = None


@dataclass(frozen=True)
class CodeResult:
    """Result of a code attempt. ``status``: ok | wrong | none | used | burned | expired | not_issued."""

    status: str
    tg_id: int | None = None
    left: int | None = None


def start_attempt(conn: Conn, ip: str, user_agent: str, now: float, ttl: int) -> Started:
    """Store a new attempt (only the hashes of the nonce and of the binding)."""
    nonce, binding = new_token(), new_token()
    with conn:
        conn.execute("DELETE FROM login_attempts WHERE expires < ?", (now - 86400,))
        conn.execute("DELETE FROM web_sessions WHERE expires < ?", (now - 86400,))
        conn.execute(
            "INSERT INTO login_attempts(nonce_hash, binding_hash, ip, browser, created, expires) VALUES (?,?,?,?,?,?)",
            (auth.sha256_hex(nonce), auth.sha256_hex(binding), ip, browser_family(user_agent), now, now + ttl))
    return Started(nonce, binding)


def issue(conn: Conn, tg_id: int, now: float, ttl: int, nonce: str | None, key: str) -> Issued:
    """Make a one-time link (and, for a browser attempt, a code) for an allowlisted id.

    ``nonce=None`` is the bot's /login: a link only, and a new one cancels the previous /login link.
    """
    if not auth.is_allowed(conn, tg_id):
        return Issued("forbidden")
    token = new_token()
    if nonce is None:
        with conn:
            conn.execute("UPDATE login_attempts SET consumed_at=? WHERE binding_hash IS NULL AND issued_tg=? "
                         "AND consumed_at IS NULL", (now, tg_id))
            conn.execute(
                "INSERT INTO login_attempts(nonce_hash, binding_hash, ip, browser, created, expires, issued_tg, "
                "magic_hash) VALUES (?, NULL, 'bot', '/login', ?, ?, ?, ?)",
                (auth.sha256_hex(new_token()), now, now + ttl, tg_id, auth.sha256_hex(token)))
        return Issued("issued", token)
    if not TOKEN_RE.fullmatch(nonce):
        return Issued("unknown")
    nonce_hash, code = auth.sha256_hex(nonce), new_code()
    row = conn.execute("SELECT issued_tg, consumed_at, expires, ip, browser FROM login_attempts WHERE nonce_hash=?",
                       (nonce_hash,)).fetchone()
    if row is None:
        return Issued("unknown")
    if row["issued_tg"] is not None or row["consumed_at"] is not None:
        return Issued("used")
    if row["expires"] <= now:
        return Issued("expired")
    with conn:  # the conditions repeat in WHERE: of two concurrent bot calls only one wins
        changed = conn.execute(
            "UPDATE login_attempts SET issued_tg=?, code_hash=?, magic_hash=?, expires=? "
            "WHERE nonce_hash=? AND issued_tg IS NULL AND consumed_at IS NULL AND expires>?",
            (tg_id, code_hash(key, nonce_hash, code), auth.sha256_hex(token), now + ttl, nonce_hash, now)).rowcount
    if changed != 1:
        return Issued("used")
    return Issued("issued", token, code, row["ip"], row["browser"])


def consume_magic(conn: Conn, token: str, now: float) -> int | None:
    """Use a one-time link: the id it was issued to, or None (unknown, used, expired, burned)."""
    if not TOKEN_RE.fullmatch(token or ""):
        return None
    with conn:
        rows = conn.execute(
            "UPDATE login_attempts SET consumed_at=? WHERE magic_hash=? AND consumed_at IS NULL "
            "AND issued_tg IS NOT NULL AND expires>? RETURNING issued_tg",
            (now, auth.sha256_hex(token), now)).fetchall()
    return int(rows[0][0]) if rows else None


def verify_code(conn: Conn, binding: str | None, code: str, now: float, key: str) -> CodeResult:
    """Check a code against the attempt bound to THIS browser; there is no other kind of code."""
    row = None
    if binding and TOKEN_RE.fullmatch(binding):
        row = conn.execute(
            "SELECT nonce_hash, issued_tg, code_hash, expires, failed, consumed_at FROM login_attempts "
            "WHERE binding_hash=?", (auth.sha256_hex(binding),)).fetchone()
    if row is None:
        return CodeResult("none")
    if row["consumed_at"] is not None:
        return CodeResult("burned" if row["failed"] >= MAX_CODE_FAILS else "used")
    if row["expires"] <= now:
        return CodeResult("expired")
    if row["issued_tg"] is None:
        return CodeResult("not_issued")
    nonce_hash = row["nonce_hash"]
    if hmac.compare_digest(code_hash(key, nonce_hash, code), row["code_hash"] or ""):
        with conn:
            changed = conn.execute("UPDATE login_attempts SET consumed_at=? WHERE nonce_hash=? AND consumed_at IS NULL",
                                   (now, nonce_hash)).rowcount
        return CodeResult("ok", int(row["issued_tg"])) if changed == 1 else CodeResult("used")
    with conn:  # one atomic step: the counter never loses a guess, and the 5th burns the attempt
        updated = conn.execute(
            "UPDATE login_attempts SET failed=failed+1, consumed_at=CASE WHEN failed+1>=? THEN ? ELSE consumed_at END "
            "WHERE nonce_hash=? AND consumed_at IS NULL RETURNING failed",
            (MAX_CODE_FAILS, now, nonce_hash)).fetchall()
    if not updated:
        return CodeResult("used")
    failed = int(updated[0][0])
    if failed >= MAX_CODE_FAILS:
        return CodeResult("burned")
    return CodeResult("wrong", left=MAX_CODE_FAILS - failed)


# ---------- sessions ----------

def create_session(conn: Conn, tg_id: int, now: float, ttl: int) -> str:
    """A new session: returns the token for the cookie; the database keeps only its sha256."""
    token = new_token()
    with conn:
        conn.execute("INSERT INTO web_sessions(token_hash, tg_id, created, expires) VALUES (?,?,?,?)",
                     (auth.sha256_hex(token), tg_id, now, now + ttl))
    return token


def session_tg(conn: Conn, token: str | None, now: float) -> int | None:
    """The id behind a live session cookie, else None. The allowlist is checked by the caller on every request."""
    if not token or not TOKEN_RE.fullmatch(token):
        return None
    row = conn.execute("SELECT tg_id FROM web_sessions WHERE token_hash=? AND revoked_at IS NULL AND expires>?",
                       (auth.sha256_hex(token), now)).fetchone()
    return int(row[0]) if row else None


def revoke_session(conn: Conn, token: str, now: float) -> None:
    """Close one session (logout)."""
    if TOKEN_RE.fullmatch(token or ""):
        with conn:
            conn.execute("UPDATE web_sessions SET revoked_at=? WHERE token_hash=? AND revoked_at IS NULL",
                         (now, auth.sha256_hex(token)))


def revoke_all(conn: Conn, tg_id: int, now: float) -> int:
    """Close every session of one person; returns how many were open."""
    with conn:
        return int(conn.execute("UPDATE web_sessions SET revoked_at=? WHERE tg_id=? AND revoked_at IS NULL",
                                (now, tg_id)).rowcount)


# ---------- pages ----------

_STYLE = ("<style>body{margin:0;font:15px/1.4 system-ui,sans-serif;background:#fff;color:#111}"
          "main{max-width:420px;margin:15vh auto;padding:0 16px}h1{font-size:20px}"
          "button{border:0;border-radius:16px;padding:10px 18px;background:#2481cc;color:#fff;font:inherit}"
          "@media (prefers-color-scheme:dark){body{background:#17181a;color:#eee}}</style>")


def _page(body: str) -> str:
    return ('<!doctype html><html lang="ru"><head><meta charset="utf-8">'
            '<meta name="viewport" content="width=device-width, initial-scale=1"><meta name="robots" content="noindex">'
            f'<title>Вход в дашборд</title>{_STYLE}</head><body><main>{body}</main></body></html>')


CSRF_PAGE = _page("<h1>Страница устарела</h1><p>Откройте ссылку из Telegram ещё раз и нажмите «Войти» в том же браузере. "
                  "Ссылка при этом не потрачена.</p>")
INVALID_PAGE = _page("<h1>Ссылка недействительна</h1><p>Она устарела или уже использована. Откройте страницу входа "
                     "и нажмите «Перейти в Telegram» ещё раз или отправьте боту /login.</p>")
LIMIT_PAGE = _page("<h1>Слишком много попыток</h1><p>Подождите 10 минут и попробуйте снова.</p>")


def magic_page(token: str, field: str) -> str:
    """The confirm page. The token (TOKEN_RE) and the field (hex) are URL-safe characters, so they need no escaping."""
    return _page('<h1>Вход в дашборд</h1><p>Нажмите «Войти», чтобы открыть дашборд на этом устройстве.</p>'
                 f'<form method="post" action="/auth/magic/{token}"><input type="hidden" name="csrf" value="{field}">'
                 '<button type="submit">Войти</button></form>')


# ---------- HTTP ----------

class CodeIn(BaseModel):
    code: str = Field(pattern=r"^[0-9]{6}$")  # [0-9], not \d: \d also matches non-ASCII digits


class IssueIn(BaseModel):
    tg_id: int
    nonce: str | None = Field(default=None, max_length=64)


class TgIn(BaseModel):
    tg_id: int


def _client(request: Request) -> str:
    return request.client.host if request.client else "unknown"


async def form_csrf(request: Request) -> str:
    """The «csrf» field of the urlencoded confirm form ('' when absent). Parsed by hand: no extra package is needed."""
    body = await request.body()
    if len(body) > 2048:
        return ""
    values = parse_qs(body.decode("ascii", "ignore"), max_num_fields=4).get("csrf", [])
    return values[0] if len(values) == 1 else ""


def set_session_cookie(resp: Response, token: str, settings: Settings) -> None:
    """HttpOnly (no script can read it), Secure (HTTPS only), SameSite=Lax (not sent with cross-site POSTs)."""
    resp.set_cookie(SESSION_COOKIE, token, max_age=settings.web_session_ttl_s, httponly=True, secure=True,
                    samesite="lax", path="/")


def build_router(get_conn: Callable[..., Any], get_settings: Callable[..., Settings],
                 now_ts: Callable[..., float]) -> APIRouter:
    """The web-login addresses; the three dependencies come from app.main (see the module note in the guide)."""
    start_limit = auth.FailureLimiter(limit=5, window_s=600)  # every start counts
    code_limit = auth.FailureLimiter(limit=10, window_s=600)  # failed codes only
    magic_limit = auth.FailureLimiter(limit=5, window_s=600)  # failed links only
    router = APIRouter()

    def same_origin_guard(request: Request, settings: Settings = Depends(get_settings)) -> None:
        """Every POST under /auth comes from our own page (Origin, else Referer); anything else is 403 and changes nothing."""
        if request.method == "POST" and not same_origin(request.headers.get("origin"), request.headers.get("referer"),
                                                        settings.public_url):
            logger.warning("web login: POST from a foreign or unknown origin refused")
            raise HTTPException(status_code=403)

    auth_routes = APIRouter(prefix="/auth", dependencies=[Depends(same_origin_guard)])
    internal = APIRouter(prefix="/internal")

    def bot_guard(request: Request, settings: Settings = Depends(get_settings)) -> None:
        """Only the local bot with the shared key gets in; anyone else sees a missing page (404)."""
        presented = auth.bearer(request.headers.get("authorization"))
        key_ok = presented is not None and hmac.compare_digest(auth.sha256_hex(presented),
                                                              auth.sha256_hex(settings.internal_key))
        if _client(request) not in LOOPBACK or not key_ok:
            logger.warning("internal: refused")
            raise HTTPException(status_code=404)

    @router.get("/login")
    def login_page() -> FileResponse:
        return FileResponse(STATIC / "login.html", headers={"Cache-Control": "no-cache", "Referrer-Policy": "no-referrer"})

    @auth_routes.post("/start")
    def auth_start(request: Request, conn: Conn = Depends(get_conn), s: Settings = Depends(get_settings),
                   t: float = Depends(now_ts)) -> JSONResponse:
        ip = _client(request)
        if start_limit.blocked(ip):
            raise HTTPException(status_code=429, detail=TOO_MANY)
        start_limit.record(ip)
        started = start_attempt(conn, ip, request.headers.get("user-agent", ""), t, s.login_ttl_s)
        resp = JSONResponse({"deep_link": f"https://t.me/{s.web_bot_username}?start=login_{started.nonce}",
                             "expires_in": s.login_ttl_s}, headers=NO_STORE)
        # The bot may answer up to ttl after the start and the code then lives ttl more: the cookie covers both.
        resp.set_cookie(LOGIN_COOKIE, started.binding, max_age=2 * s.login_ttl_s, httponly=True, secure=True,
                        samesite="lax", path="/auth")
        return resp

    @auth_routes.post("/code")
    def auth_code(body: CodeIn, request: Request, conn: Conn = Depends(get_conn), s: Settings = Depends(get_settings),
                  t: float = Depends(now_ts)) -> JSONResponse:
        ip = _client(request)
        if code_limit.blocked(ip):
            raise HTTPException(status_code=429, detail=TOO_MANY)
        res = verify_code(conn, request.cookies.get(LOGIN_COOKIE), body.code, t, s.web_hmac_key)
        if res.status == "ok" and (res.tg_id is None or not auth.is_allowed(conn, res.tg_id)):
            res = CodeResult("none")  # the person was removed between the issue and the typing
        if res.status != "ok" or res.tg_id is None:
            code_limit.record(ip)
            logger.info("web login code refused: %s", res.status)
            payload: dict[str, Any] = {"status": res.status}
            if res.left is not None:
                payload["left"] = res.left
            return JSONResponse(payload, status_code=400, headers=NO_STORE)
        resp = JSONResponse({"status": "ok"}, headers=NO_STORE)
        set_session_cookie(resp, create_session(conn, res.tg_id, t, s.web_session_ttl_s), s)
        resp.delete_cookie(LOGIN_COOKIE, path="/auth", secure=True, httponly=True)
        return resp

    @auth_routes.get("/magic/{token}", response_class=HTMLResponse)
    def magic_confirm(token: str, request: Request, s: Settings = Depends(get_settings)) -> HTMLResponse:
        """GET only shows a button: a link preview or a prefetch never uses the token up.

        It also hands this browser its half of the form token (an HttpOnly cookie). The other half, the hidden field, is
        an HMAC of that cookie and of the link, so a page of another site can neither read nor guess it. Referrer-Policy
        is same-origin, not no-referrer: with no-referrer a browser sends «Origin: null» with the form, and the Origin check
        would refuse the owner too. Nothing is sent to other sites either way.
        """
        if not TOKEN_RE.fullmatch(token):
            return HTMLResponse(INVALID_PAGE, status_code=400, headers=NO_STORE)
        mine = request.cookies.get(CSRF_COOKIE)
        cookie = mine if mine and TOKEN_RE.fullmatch(mine) else new_token()  # two tabs keep working
        resp = HTMLResponse(magic_page(token, csrf_field(s.web_hmac_key, cookie, token)),
                            headers={**NO_STORE, "Referrer-Policy": "same-origin"})
        resp.set_cookie(CSRF_COOKIE, cookie, max_age=s.login_ttl_s, httponly=True, secure=True, samesite="strict",
                        path="/auth/magic")
        return resp

    @auth_routes.post("/magic/{token}")
    def magic_consume(token: str, request: Request, field: str = Depends(form_csrf), conn: Conn = Depends(get_conn),
                      s: Settings = Depends(get_settings), t: float = Depends(now_ts)) -> Response:
        if not csrf_ok(s.web_hmac_key, request.cookies.get(CSRF_COOKIE), token, field):
            # Not counted by the limiter (a foreign page must not be able to lock the owner out) and the link stays unspent.
            return HTMLResponse(CSRF_PAGE, status_code=403, headers=NO_STORE)
        ip = _client(request)
        if magic_limit.blocked(ip):
            return HTMLResponse(LIMIT_PAGE, status_code=429, headers=NO_STORE)
        tg_id = consume_magic(conn, token, t)
        if tg_id is not None and not auth.is_allowed(conn, tg_id):
            tg_id = None
        if tg_id is None:
            magic_limit.record(ip)
            return HTMLResponse(INVALID_PAGE, status_code=400, headers=NO_STORE)
        resp = RedirectResponse("/", status_code=303, headers=NO_STORE)
        set_session_cookie(resp, create_session(conn, tg_id, t, s.web_session_ttl_s), s)
        resp.delete_cookie(CSRF_COOKIE, path="/auth/magic", secure=True, httponly=True)
        return resp

    @auth_routes.post("/logout")
    def logout(request: Request, conn: Conn = Depends(get_conn), t: float = Depends(now_ts)) -> JSONResponse:
        token = request.cookies.get(SESSION_COOKIE)
        if token:
            revoke_session(conn, token, t)
        resp = JSONResponse({"ok": True}, headers=NO_STORE)
        resp.delete_cookie(SESSION_COOKIE, path="/", secure=True, httponly=True)
        return resp

    @internal.post("/login/issue", dependencies=[Depends(bot_guard)])
    def internal_issue(body: IssueIn, conn: Conn = Depends(get_conn), s: Settings = Depends(get_settings),
                       t: float = Depends(now_ts)) -> dict[str, Any]:
        res = issue(conn, body.tg_id, t, s.login_ttl_s, body.nonce, s.web_hmac_key)
        if res.status != "issued":
            return {"status": res.status}
        out: dict[str, Any] = {"status": "issued", "url": f"{s.public_url}/auth/magic/{res.token}",
                               "expires_in": s.login_ttl_s}
        if res.code is not None:
            out["code"] = res.code
            out["request"] = {"browser": res.browser, "ip": res.ip}
        return out

    @internal.post("/sessions/revoke_all", dependencies=[Depends(bot_guard)])
    def internal_revoke_all(body: TgIn, conn: Conn = Depends(get_conn), t: float = Depends(now_ts)) -> dict[str, int]:
        if not auth.is_allowed(conn, body.tg_id):
            raise HTTPException(status_code=403)
        return {"revoked": revoke_all(conn, body.tg_id, t)}

    router.include_router(auth_routes)
    router.include_router(internal)
    return router
```

Короткий ключ (или отсутствующий `WEB_LOGIN_HMAC_KEY`) отвергает `main.py` (правка 3 ниже): приложение не стартует. Адреса `/internal/...` отвечают 404 и при неверном ключе, и при обращении не с локального адреса, чтобы снаружи было не видно, что они существуют.

### 6.3. Подключение в `main.py`

Четыре небольшие правки. Старый вход через мини-приложение (`/api/login`, заголовок `Authorization`) не меняется.

#### `app/main.py` (правка 1: импорт)

Найти:
```python
from app import auth
```
Заменить на:
```python
from app import auth, weblogin
```

#### `app/main.py` (правка 2: кто сейчас пользователь)

`current_user` принимает токен в заголовке (мини-приложение) **или**, если заголовка нет, куки сессии (обычный браузер). Если заголовок есть, он обязан быть верным `Bearer`: любой другой (`Basic ...`, пустой `Bearer`, мусор) даёт 401, а куки не пробуются. Куки сессии принимаются только пока расширение включено: выключили – старые веб-сессии не действуют (401), а таблицы `web_sessions` на свежей базе может и не быть, поэтому к ней в этом случае даже не обращаются.

Найти:
```python
    tg_id = auth.read_session(settings.session_secret, auth.bearer(request.headers.get("authorization")), t)
    if tg_id is None:
```
Заменить на:
```python
    header = request.headers.get("authorization")
    if header is not None:  # the Mini App sends a signed token; a header that is present is never ignored
        bearer = auth.bearer(header)
        tg_id = auth.read_session(settings.session_secret, bearer, t) if bearer is not None else None
    elif settings.web_login_enabled:  # an ordinary browser sends the session cookie of the web login
        tg_id = weblogin.session_tg(conn, request.cookies.get(weblogin.SESSION_COOKIE), t)
    else:  # the extension is off: a cookie is no way in (and web_sessions may not exist)
        tg_id = None
    if tg_id is None:
```

#### `app/main.py` (правка 3: таблицы при старте)

Найти:
```python
        migrate(conn, settings.allowed_tg_ids)
    finally:
```
Заменить на:
```python
        migrate(conn, settings.allowed_tg_ids)
        if settings.web_login_enabled:
            for name, value in (("WEB_LOGIN_INTERNAL_KEY", settings.internal_key),
                                ("WEB_LOGIN_HMAC_KEY", settings.web_hmac_key)):
                if len(value.encode()) < 32:  # the message names the variable, never the value
                    raise RuntimeError(f"{name} must be at least 32 bytes")
            weblogin.create_tables(conn)
    finally:
```

#### `app/main.py` (правка 4: адреса)

Найти:
```python
    app.state.limiter = auth.FailureLimiter()
```
Заменить на:
```python
    app.state.limiter = auth.FailureLimiter()
    if settings.web_login_enabled:  # optional extension: references/web-login.md
        app.include_router(weblogin.build_router(get_conn, get_settings, now_ts))
```

### 6.4. Страница входа

#### `static/login.html`

Тот же стиль, что у `static/index.html`: чистый JavaScript, ответы сервера вставляются как текст, не как HTML.

```html
<!doctype html>
<html lang="ru">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <meta name="robots" content="noindex, nofollow">
  <meta name="referrer" content="no-referrer">
  <title>Вход в дашборд</title>
  <style>
    :root { --bg: #fff; --text: #111; --hint: #777; --card: #f2f2f4; --accent: #2481cc; --bad: #d03a3a; }
    @media (prefers-color-scheme: dark) { :root { --bg: #17181a; --text: #eee; --hint: #999; --card: #26282b; } }
    * { box-sizing: border-box; }
    body { margin: 0; background: var(--bg); color: var(--text); font: 15px/1.4 system-ui, sans-serif; }
    main { max-width: 420px; margin: 0 auto; padding: 12vh 16px 32px; }
    h1 { font-size: 20px; }
    p, .hint { color: var(--hint); overflow-wrap: anywhere; }
    button, a.btn { display: inline-block; border: 0; border-radius: 16px; padding: 10px 18px; background: var(--accent);
                    color: #fff; font: inherit; text-decoration: none; cursor: pointer; }
    button:disabled { opacity: .6; cursor: default; }
    form { display: flex; gap: 8px; margin-top: 8px; }
    input { flex: 1; min-width: 0; border: 1px solid var(--card); border-radius: 12px; padding: 10px 12px; background: var(--card);
            color: var(--text); font: inherit; font-size: 18px; letter-spacing: 4px; }
    #msg { color: var(--bad); min-height: 1.4em; margin-top: 12px; }
  </style>
</head>
<body>
<main>
  <h1>Вход в дашборд</h1>
  <p>Вход подтверждает бот в Telegram. Пароль не нужен.</p>
  <button id="go" type="button">Войти через Telegram</button>
  <div id="step2" hidden>
    <p>1. Откройте бота и нажмите «Запустить» (Start):</p>
    <a id="tg" class="btn" target="_blank" rel="noopener">Перейти в Telegram</a>
    <p>2. Бот пришлёт одноразовую ссылку и 6-значный код. Нажмите ссылку или введите код здесь:</p>
    <form id="codeform">
      <input id="code" inputmode="numeric" autocomplete="one-time-code" maxlength="6" pattern="[0-9]{6}" required
             aria-label="Код из Telegram">
      <button type="submit">Войти</button>
    </form>
  </div>
  <div id="msg" role="status"></div>
</main>
<script>
"use strict";
const $ = (id) => document.getElementById(id);
const say = (text) => { $("msg").textContent = text; };
const WHY = {                                        // the server answers with a status word; the words are ours
  wrong: "Неверный код.", none: "Код не подходит для этого браузера. Нажмите «Войти через Telegram» ещё раз.",
  expired: "Код устарел. Начните вход заново.", used: "Этот вход уже использован. Начните заново.",
  burned: "Слишком много неверных кодов. Начните вход заново.", not_issued: "Бот ещё не прислал код. Откройте бота по ссылке.",
};

$("go").onclick = async () => {
  say("");
  $("go").disabled = true;
  try {
    const r = await fetch("/auth/start", { method: "POST" });
    if (r.status === 429) throw new Error("Слишком много попыток. Подождите 10 минут.");
    if (!r.ok) throw new Error("Не удалось начать вход. Обновите страницу.");
    const d = await r.json();
    if (typeof d.deep_link !== "string" || !d.deep_link.startsWith("https://t.me/")) throw new Error("Неожиданный ответ сервера.");
    $("tg").href = d.deep_link;                      // checked above: only a t.me address is ever put into the link
    $("step2").hidden = false;
    $("code").focus();
  } catch (e) { say(e.message); }
  $("go").disabled = false;
};

$("codeform").onsubmit = async (ev) => {
  ev.preventDefault();
  say("");
  const r = await fetch("/auth/code", { method: "POST", headers: { "Content-Type": "application/json" },
                                        body: JSON.stringify({ code: $("code").value.trim() }) });
  if (r.ok) { location.replace("/"); return; }
  if (r.status === 429) { say("Слишком много попыток. Подождите 10 минут."); return; }
  if (r.status === 422) { say("Код состоит из шести цифр."); return; }
  const d = await r.json().catch(() => ({}));
  say((WHY[d.status] || "Не удалось войти.") + (d.left ? " Осталось попыток: " + d.left + "." : ""));
};
</script>
</body>
</html>
```

#### `static/index.html` (правка 1: кнопка «Выйти»)

Найти:
```html
  <div id="extra"></div>
</main>
```
Заменить на:
```html
  <div id="extra"></div>
  <button id="logout" type="button" hidden style="margin-top: 24px">Выйти</button>
</main>
```

#### `static/index.html` (правка 2: запросы без токена)

В обычном браузере токена нет, цифры отдаёт куки. Нет куки (ответ 401) – страница уводит на `/login`.

Найти:
```javascript
  const r = await fetch(path, { headers: { Authorization: "Bearer " + state.token, Accept: "application/json" } });
  if (r.status === 401 && retry) { await login(); return api(path, false); }
```
Заменить на:
```javascript
  const headers = { Accept: "application/json" };
  if (state.token) headers.Authorization = "Bearer " + state.token;   // no token = an ordinary browser: the cookie speaks
  const r = await fetch(path, { headers });
  if (r.status === 401 && !state.token) { location.href = "/login"; throw new Error("Нужен вход."); }
  if (r.status === 401 && retry) { await login(); return api(path, false); }
```

#### `static/index.html` (правка 3: запуск вне Telegram)

Найти:
```javascript
  if (!tg || !tg.initData) { say("Откройте дашборд через бота в Telegram."); return; }
```
Заменить на:
```javascript
  if (!tg || !tg.initData) {                         // an ordinary browser: the session cookie decides, else /login
    $("logout").hidden = false;
    $("logout").onclick = async () => { await fetch("/auth/logout", { method: "POST" }); location.href = "/login"; };
    try { drawControls(await api("/api/sections")); await load(); } catch (e) { say(e.message); }
    return;
  }
```

### 6.5. Бот

Бот не хранит ни ссылок, ни кодов: он спрашивает их у backend по закрытому адресу и пересылает владельцу. Без `WEB_LOGIN_INTERNAL_KEY` бот ведёт себя как раньше.

#### `bot/bot.py` (правка 1: импорты и константы)

Найти:
```python
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
```
Заменить на:
```python
import functools
import json
import logging
import os
import re
import sys
import time
import urllib.error
import urllib.request
from collections.abc import Callable
from typing import Any

logger = logging.getLogger("dashboard-bot")
NO_ACCESS = "Нет доступа."
BUTTON_TEXT = "Дашборд"
NONCE_RE = re.compile(r"^login_([A-Za-z0-9_-]{43})$")  # the deep-link parameter made by POST /auth/start
LOGIN_PROBLEMS = {
    "expired": "Ссылка устарела. Нажмите «Перейти в Telegram» на странице входа ещё раз.",
    "used": "Для этого входа код уже выдан. Нажмите «Перейти в Telegram» на странице входа ещё раз.",
    "unknown": "Попытка входа не найдена. Нажмите «Перейти в Telegram» на странице входа или отправьте /login.",
    "forbidden": NO_ACCESS,
}
# (path, body) -> JSON answer of the backend's loopback-only /internal/... endpoint
Internal = Callable[[str, dict[str, Any]], dict[str, Any]]
```

#### `bot/bot.py` (правка 2: обращение к backend и тексты, новый параметр `handle`)

Найти:
```python
def handle(token: str, update: dict[str, Any], allowed: frozenset[int], public_url: str) -> None:
```
Заменить на:
```python
class _NoRedirect(urllib.request.HTTPRedirectHandler):
    """Never follow a redirect: urllib would resend the Authorization header (the shared key) to the new address."""

    def redirect_request(self, *args: Any, **kwargs: Any) -> None:
        return None


def internal_call(base_url: str, key: str, path: str, body: dict[str, Any]) -> dict[str, Any]:
    """POST to the backend's /internal/... endpoint; raises RuntimeError (the message has no URL and no key)."""
    req = urllib.request.Request(base_url.rstrip("/") + path, data=json.dumps(body).encode(),
                                 headers={"Content-Type": "application/json", "Authorization": f"Bearer {key}"})
    try:
        with urllib.request.build_opener(_NoRedirect).open(req, timeout=15) as resp:
            data = json.load(resp)
    except urllib.error.HTTPError as exc:
        raise RuntimeError(f"internal {path}: http {exc.code}") from None
    except (urllib.error.URLError, TimeoutError, OSError, ValueError) as exc:
        raise RuntimeError(f"internal {path}: {type(exc).__name__}") from None
    if not isinstance(data, dict):
        raise RuntimeError(f"internal {path}: unexpected answer")
    return data


def login_text(res: dict[str, Any]) -> str:
    """The message with the one-time link and (for a browser attempt) the code."""
    if res.get("status") != "issued":
        return LOGIN_PROBLEMS.get(str(res.get("status")), LOGIN_PROBLEMS["unknown"])
    minutes = int(res.get("expires_in", 300)) // 60
    head = f"Вход в дашборд\n\nОдноразовая ссылка (действует {minutes} мин.):\n{res['url']}\n\n"
    if not res.get("code"):
        return head + ("Код для ввода на странице бот присылает только после кнопки «Перейти в Telegram» "
                       "на странице входа.\n\nЕсли вы не входили – ничего не делайте.")
    req = res.get("request") or {}
    return head + (f"Или введите код на странице входа: {res['code']}\n\n"
                   f"Запрос с: {req.get('browser', 'неизвестный браузер')}, {req.get('ip', 'неизвестный IP')}\n\n"
                   "Никому не сообщайте код. Если вы не входили – ничего не делайте.")


def web_login_reply(text: str, tg_id: int, internal: Internal) -> str | None:
    """The answer to /start login_<nonce>, /login or /logout_all; None for any other text."""
    match = NONCE_RE.fullmatch(text[len("/start"):].strip()) if text.startswith("/start") else None
    try:
        if match:
            return login_text(internal("/internal/login/issue", {"tg_id": tg_id, "nonce": match.group(1)}))
        if text == "/login":
            return login_text(internal("/internal/login/issue", {"tg_id": tg_id, "nonce": None}))
        if text == "/logout_all":
            closed = int(internal("/internal/sessions/revoke_all", {"tg_id": tg_id}).get("revoked", 0))
            return f"Готово: закрыто сессий – {closed}."
    except RuntimeError as exc:
        logger.warning("internal call failed: %s", exc)
        return "Сервис входа недоступен, попробуйте через минуту."
    return None


def handle(token: str, update: dict[str, Any], allowed: frozenset[int], public_url: str,
           internal: Internal | None = None) -> None:
```

#### `bot/bot.py` (правка 3: команды входа в `handle`)

Найти:
```python
    if (msg.get("from") or {}).get("id") not in allowed:
        call(token, "sendMessage", {"chat_id": chat["id"], "text": NO_ACCESS})
        return
```
Заменить на:
```python
    uid = (msg.get("from") or {}).get("id")
    if uid not in allowed:
        call(token, "sendMessage", {"chat_id": chat["id"], "text": NO_ACCESS})
        return
    if internal is not None:  # web login is on
        reply = web_login_reply((msg.get("text") or "").strip(), int(uid), internal)
        if reply is not None:
            call(token, "sendMessage", {"chat_id": chat["id"], "text": reply,
                                        "link_preview_options": {"is_disabled": True}})
            return
```

#### `bot/bot.py` (правка 4: связь с backend при запуске)

Найти:
```python
    if sys.argv[1:] == ["set-menu"]:
        set_menu_buttons(token, allowed, public_url)
        return 0
    offset = 0
```
Заменить на:
```python
    if sys.argv[1:] == ["set-menu"]:
        set_menu_buttons(token, allowed, public_url)
        return 0
    internal: Internal | None = None
    key = os.environ.get("WEB_LOGIN_INTERNAL_KEY", "")
    if key:  # web login is on: the bot may ask the backend for links and codes
        internal = functools.partial(internal_call, os.environ.get("WEB_LOGIN_INTERNAL_URL", "http://127.0.0.1:8000"), key)
    offset = 0
```

#### `bot/bot.py` (правка 5: передать связь в обработчик)

Найти:
```python
                handle(token, update, allowed, public_url)
```
Заменить на:
```python
                handle(token, update, allowed, public_url, internal)
```

## 7. Деплой

Только после «да» владельца. Службы и порядок те же, что в `starter.md`, раздел 10. Добавляется три вещи.

1. **Настройки.** Допишите переменные из раздела 4 в `/etc/dashboard/backend.env` и `/etc/dashboard/bot.env`. `WEB_LOGIN_INTERNAL_KEY` создайте один раз и положите одинаковым в оба файла, не печатая на экран. `WEB_LOGIN_HMAC_KEY` – другое значение и **только** в `backend.env` (права 640, как у остальных секретов). `PUBLIC_URL` должен совпадать с адресом, с которого владелец открывает страницу входа, (схема, домен и порт): иначе проверка Origin откажет и ему.
2. **Caddy закрывает `/internal` снаружи.** Эти адреса нужны только боту на этом же сервере. Правка в `deploy/Caddyfile`: блок для `/ingest` становится общим.

#### `deploy/Caddyfile` (правка)

Найти:
```caddyfile
	@ingest path /ingest /ingest/*
	respond @ingest 404
```
Заменить на:
```caddyfile
	@closed path /ingest /ingest/* /internal /internal/*
	respond @closed 404
```

   Это второй рубеж: сам backend тоже отвечает 404 всем, кроме локального бота с ключом (служба слушает `127.0.0.1`, а Caddy передаёт настоящий адрес клиента через `--proxy-headers`, как в службе из `starter.md`).
3. **Перезапуск.** `sudo systemctl restart dashboard dashboard-bot`. Таблицы создаются при старте. Если до этого стоял вариант расширения без `WEB_LOGIN_HMAC_KEY`: текущие попытки входа (5 минут) перестанут подходить, начать вход заново можно сразу; сессии не затронуты.

Проверка с другого компьютера:

```bash
curl -s -o /dev/null -w '%{http_code}\n' -X POST https://<домен>/internal/login/issue   # must be 404
curl -s -o /dev/null -w '%{http_code}\n' https://<домен>/login                          # 200
curl -s -o /dev/null -w '%{http_code}\n' https://<домен>/api/summary                    # 401 (no cookie)
curl -s -o /dev/null -w '%{http_code}\n' -X POST https://<домен>/auth/start             # 403 (no Origin, no Referer)
curl -s -o /dev/null -w '%{http_code}\n' -X POST -H 'Origin: https://evil.example' https://<домен>/auth/start   # 403
```

Затем владелец открывает `https://<домен>/login` в обычном браузере, проходит вход ссылкой и кодом; чужой Telegram-аккаунт должен получить «Нет доступа.». После входа цифры открываются, «Выйти» закрывает сессию.

## 8. Тесты

Тест-файл целиком. Он берёт из каркаса `create_app`, `Settings`, `auth`, `bot.bot` и работает на SQLite всегда, а на PostgreSQL – когда задана `TEST_DATABASE_URL` и применён `postgres.md` (иначе строки Postgres помечены «skipped»). Бот в тестах подключён прямо к приложению, поэтому сквозная проверка идёт по настоящему коду обеих сторон.

Что проверяется: старт, выдача и вход по ссылке; вход по коду в привязанном браузере; отказ в чужом браузере; повторное использование; срок жизни; сжигание после 5 неверных кодов; чужой Telegram-id; лимит попыток; выход; закрытые адреса; хеши в базе (код – HMAC с ключом, перебор миллиона простых sha256 ничего не находит); отсутствие секретов в логах; чужой Origin и отсутствие Origin отвергаются и не создают сессию; подтверждение без верного токена подтверждения отвергается (нет поля, чужое поле, нет куки, поле от другой ссылки); при выключенном расширении куки сессии не работает (401, не 500, в том числе на свежей базе); любой неверный `Authorization` даёт 401 и не уступает место куки; короткий или отсутствующий ключ – приложение не запускается.

Перед тем как верить тестам, сломайте что-нибудь на пробу (например, уберите условие `consumed_at IS NULL` в `consume_magic`): хотя бы один тест обязан упасть.

#### `tests/test_web_login.py`

```python
"""Web login: start -> bot issues a link and a code -> link or code logs in. Everything runs against the real app."""

from __future__ import annotations

import hashlib
import hmac
import logging
import os
import re
import uuid
from collections.abc import Iterator
from dataclasses import dataclass, replace
from pathlib import Path
from typing import Any
from zoneinfo import ZoneInfo

import pytest
from fastapi.testclient import TestClient

from app import auth, weblogin
from app.db import connect
from app.main import create_app
from app.settings import Settings, load_settings
from bot import bot as bot_module

OWNER = 5_000_000_011  # above 2**31 on purpose: BIGINT columns must hold it
STRANGER = 42
BOT_USERNAME = "your_dash_bot"
PUBLIC = "https://dashboard.example.com"
KEY = "k" * 40
HMAC_KEY = "h" * 40  # the server key of the code hashes and of the confirm-form tokens (never in the database)
EVIL = "https://evil.example"
T0 = 1_790_730_000.0
KEY_HEADERS = {"Authorization": f"Bearer {KEY}"}
CHROME_MAC = ("Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) "
              "Chrome/129.0.0.0 Safari/537.36")


@dataclass
class World:
    """One app, a client for the local bot and a factory of separate browsers (each with its own cookies)."""

    app: Any
    settings: Settings
    bot: TestClient

    def browser(self, ip: str = "203.0.113.7", origin: str | None = PUBLIC) -> TestClient:
        """A browser: it sends the Origin of the page it is on (origin=None: a client that sends no Origin)."""
        headers = {"User-Agent": CHROME_MAC, **({"Origin": origin} if origin else {})}
        return TestClient(self.app, base_url=PUBLIC, client=(ip, 50000), headers=headers)

    def at(self, seconds_after_t0: float) -> None:
        self.app.state.clock = lambda: T0 + seconds_after_t0

    def db(self) -> Any:
        return connect(self.settings.db_target if hasattr(self.settings, "db_target") else self.settings.db_path)


def build(settings: Settings) -> World:
    app = create_app(settings)
    app.state.clock = lambda: T0
    return World(app, settings, TestClient(app, base_url="http://127.0.0.1:8000", client=("127.0.0.1", 50000)))


@pytest.fixture(params=["sqlite", "postgres"])
def world(request: pytest.FixtureRequest, tmp_path: Path) -> Iterator[World]:
    """A fresh app on each backend. PostgreSQL runs only with TEST_DATABASE_URL (and the postgres.md variant)."""
    common: dict[str, Any] = {
        "bot_token": "123456:TEST-TOKEN-NOT-REAL", "session_secret": b"s" * 40, "allowed_tg_ids": frozenset({OWNER}),
        "tz": ZoneInfo("UTC"), "web_bot_username": BOT_USERNAME, "public_url": PUBLIC, "internal_key": KEY,
        "web_hmac_key": HMAC_KEY}
    if request.param == "sqlite":
        yield build(Settings(db_path=tmp_path / "w.db", **common))
        return
    base = os.environ.get("TEST_DATABASE_URL")
    if not base or "database_url" not in Settings.__dataclass_fields__:
        pytest.skip("TEST_DATABASE_URL is not set, or the postgres.md variant is not applied")
    import psycopg

    schema = "t_" + uuid.uuid4().hex[:10]
    with psycopg.connect(base, autocommit=True) as admin:
        admin.execute(f"CREATE SCHEMA {schema}")
        url = base + ("&" if "?" in base else "?") + f"options=-csearch_path%3D{schema}&application_name={schema}"
        try:
            yield build(Settings(db_path=None, database_url=url, **common))  # type: ignore[call-arg]
        finally:
            admin.execute("SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE application_name = %s", (schema,))
            admin.execute(f"DROP SCHEMA {schema} CASCADE")


def cookies_of(resp: Any) -> str:
    return "; ".join(resp.headers.get_list("set-cookie")).lower()


def start(browser: TestClient) -> str:
    """Press «Войти через Telegram»: returns the nonce from the deep link."""
    r = browser.post("/auth/start")
    assert r.status_code == 200, r.text
    return str(r.json()["deep_link"].split("start=login_")[1])


def issue(w: World, nonce: str | None, tg_id: int = OWNER) -> dict[str, Any]:
    """What the bot does: ask the backend for a link and a code."""
    r = w.bot.post("/internal/login/issue", json={"tg_id": tg_id, "nonce": nonce}, headers=KEY_HEADERS)
    assert r.status_code == 200, r.text
    return dict(r.json())


def path_of(issued: dict[str, Any]) -> str:
    return str(issued["url"]).removeprefix(PUBLIC)


def wrong(code: str) -> str:
    return "000000" if code != "000000" else "111111"


def confirm_form(browser: TestClient, path: str) -> str:
    """Open the link (GET) like a browser does: returns the hidden token of the «Войти» form."""
    page = browser.get(path)
    assert page.status_code == 200 and 'method="post"' in page.text
    found = re.search(r'name="csrf" value="([0-9a-f]{64})"', page.text)
    assert found, page.text
    return found.group(1)


def confirm(browser: TestClient, path: str) -> Any:
    """Open the link and press «Войти»: the POST carries the page's hidden token."""
    return browser.post(path, data={"csrf": confirm_form(browser, path)}, follow_redirects=False)


def test_start_gives_deep_link_and_bound_cookie(world: World) -> None:
    r = world.browser().post("/auth/start")
    body = r.json()
    assert set(body) == {"deep_link", "expires_in"} and body["expires_in"] == 300
    assert body["deep_link"].startswith(f"https://t.me/{BOT_USERNAME}?start=login_")
    assert len(body["deep_link"].split("start=")[1]) <= 64  # Telegram's limit for the start parameter
    cookie = cookies_of(r)
    for part in ("dash_login=", "httponly", "secure", "samesite=lax", "path=/auth", "max-age=600"):
        assert part in cookie
    assert "no-store" in r.headers["cache-control"]


def test_magic_link_logs_in_once(world: World) -> None:
    browser = world.browser()
    issued = issue(world, start(browser))
    assert issued["status"] == "issued" and issued["url"].startswith(f"{PUBLIC}/auth/magic/")
    assert browser.get("/api/summary").status_code == 401  # no session yet
    page = browser.get(path_of(issued))  # a link preview does this: it must not use the token up
    assert page.status_code == 200 and 'method="post"' in page.text
    assert page.headers["referrer-policy"] == "same-origin" and "no-store" in page.headers["cache-control"]
    assert "dash_csrf=" in cookies_of(page) and "httponly" in cookies_of(page) and "samesite=strict" in cookies_of(page)
    assert browser.get(path_of(issued)).status_code == 200
    ok = confirm(browser, path_of(issued))
    assert ok.status_code == 303 and ok.headers["location"] == "/"
    cookie = cookies_of(ok)
    for part in ("dash_session=", "httponly", "secure", "samesite=lax", "path=/", "max-age=604800"):
        assert part in cookie
    assert browser.get("/api/summary").status_code == 200  # the cookie opens the dashboard API
    assert "dash_csrf=" in cookies_of(ok)  # the confirm cookie is deleted after use
    assert confirm(world.browser(), path_of(issued)).status_code == 400  # one use only


def test_code_in_the_bound_browser_logs_in_once(world: World) -> None:
    browser = world.browser()
    issued = issue(world, start(browser))
    assert len(issued["code"]) == 6 and issued["code"].isdigit() and issued["expires_in"] == 300
    r = browser.post("/auth/code", json={"code": issued["code"]})
    assert r.status_code == 200 and r.json() == {"status": "ok"}
    assert "dash_session=" in cookies_of(r)
    assert browser.get("/api/summary").status_code == 200
    again = browser.post("/auth/code", json={"code": issued["code"]})
    assert again.status_code == 400  # the binding cookie was deleted and the attempt is used


def test_code_is_refused_in_another_browser(world: World) -> None:
    browser = world.browser()
    issued = issue(world, start(browser))
    other = world.browser()  # no binding cookie
    r = other.post("/auth/code", json={"code": issued["code"]})
    assert r.status_code == 400 and r.json()["status"] == "none"
    assert other.get("/api/summary").status_code == 401
    assert browser.post("/auth/code", json={"code": issued["code"]}).status_code == 200  # the right browser still can


def test_link_and_code_cannot_be_reused(world: World) -> None:
    browser = world.browser()
    issued = issue(world, start(browser))
    assert confirm(browser, path_of(issued)).status_code == 303
    used = browser.post("/auth/code", json={"code": issued["code"]})  # same attempt, already used by the link
    assert used.status_code == 400 and used.json()["status"] == "used"
    browser2 = world.browser()
    issued2 = issue(world, start(browser2))
    assert browser2.post("/auth/code", json={"code": issued2["code"]}).status_code == 200
    assert confirm(world.browser(), path_of(issued2)).status_code == 400  # link after code


def test_deep_link_gives_a_code_only_once(world: World) -> None:
    nonce = start(world.browser())
    assert issue(world, nonce)["status"] == "issued"
    assert issue(world, nonce) == {"status": "used"}
    assert issue(world, "x" * 43) == {"status": "unknown"}
    assert issue(world, "../etc") == {"status": "unknown"}


def test_expiry(world: World) -> None:
    browser = world.browser()
    nonce = start(browser)
    assert browser.post("/auth/code", json={"code": "123456"}).json()["status"] == "not_issued"
    world.at(240)  # the bot is opened 4 minutes after the start: still fine
    issued = issue(world, nonce)
    world.at(500)  # the code lives ttl from the issue (until 540) and the cookie lives 600
    assert browser.post("/auth/code", json={"code": wrong(issued["code"])}).json()["status"] == "wrong"
    world.at(541)
    assert browser.post("/auth/code", json={"code": issued["code"]}).json()["status"] == "expired"
    assert confirm(browser, path_of(issued)).status_code == 400  # the link expired together with it
    late = start(world.browser())
    world.at(541 + 301)
    assert issue(world, late) == {"status": "expired"}  # the bot is opened after the attempt's life


def test_code_works_late_but_before_its_own_expiry(world: World) -> None:
    browser = world.browser()
    nonce = start(browser)
    world.at(240)
    issued = issue(world, nonce)
    world.at(500)
    assert browser.post("/auth/code", json={"code": issued["code"]}).json() == {"status": "ok"}


def test_session_expires(world: World) -> None:
    browser = world.browser()
    issued = issue(world, start(browser))
    assert browser.post("/auth/code", json={"code": issued["code"]}).status_code == 200
    world.at(7 * 86400 - 10)
    assert browser.get("/api/summary").status_code == 200
    world.at(7 * 86400 + 10)
    assert browser.get("/api/summary").status_code == 401


def test_five_wrong_codes_burn_the_attempt(world: World) -> None:
    browser = world.browser()
    issued = issue(world, start(browser))
    answers = [browser.post("/auth/code", json={"code": wrong(issued["code"])}).json() for _ in range(4)]
    assert [a["left"] for a in answers] == [4, 3, 2, 1] and {a["status"] for a in answers} == {"wrong"}
    assert browser.post("/auth/code", json={"code": wrong(issued["code"])}).json()["status"] == "burned"
    assert browser.post("/auth/code", json={"code": issued["code"]}).json()["status"] == "burned"  # the right code is too late
    assert confirm(browser, path_of(issued)).status_code == 400  # and so is the link
    assert browser.get("/api/summary").status_code == 401


def test_not_allowed_id_gets_nothing(world: World) -> None:
    nonce = start(world.browser())
    assert issue(world, nonce, tg_id=STRANGER) == {"status": "forbidden"}
    assert issue(world, None, tg_id=STRANGER) == {"status": "forbidden"}
    conn = world.db()
    assert conn.execute("SELECT COUNT(*) FROM login_attempts WHERE issued_tg IS NOT NULL").fetchone()[0] == 0
    conn.close()


def test_removed_person_loses_the_web_session_at_once(world: World) -> None:
    browser = world.browser()
    issued = issue(world, start(browser))
    assert browser.post("/auth/code", json={"code": issued["code"]}).status_code == 200
    conn = world.db()
    with conn:
        conn.execute("UPDATE allowed_users SET revoked_at=datetime('now') WHERE tg_id=?", (OWNER,))
    conn.close()
    assert browser.get("/api/summary").status_code == 403
    assert issue(world, None) == {"status": "forbidden"}


def test_rate_limits(world: World) -> None:
    for _ in range(5):
        assert world.browser().post("/auth/start").status_code == 200
    blocked = world.browser().post("/auth/start")
    assert blocked.status_code == 429 and "10 минут" in blocked.json()["detail"]
    assert world.browser(ip="198.51.100.9").post("/auth/start").status_code == 200  # another address is not blocked
    guesser = world.browser(ip="198.51.100.10")
    for _ in range(10):
        assert guesser.post("/auth/code", json={"code": "123456"}).status_code == 400
    assert guesser.post("/auth/code", json={"code": "123456"}).status_code == 429
    bad = "/auth/magic/" + "a" * 43
    for _ in range(5):
        assert confirm(guesser, bad).status_code == 400
    assert confirm(guesser, bad).status_code == 429
    assert world.browser(ip="198.51.100.11").post("/auth/code", json={"code": "12ab34"}).status_code == 422
    assert world.browser(ip="198.51.100.11").post("/auth/code", json={"code": "١٢٣٤٥٦"}).status_code == 422  # not ASCII digits


def test_logout_and_logout_everywhere(world: World) -> None:
    first, second = world.browser(), world.browser()
    for browser in (first, second):
        issued = issue(world, start(browser))
        assert browser.post("/auth/code", json={"code": issued["code"]}).status_code == 200
    out = first.post("/auth/logout")
    assert out.status_code == 200 and out.json() == {"ok": True} and "dash_session=" in cookies_of(out)
    assert first.get("/api/summary").status_code == 401
    assert second.get("/api/summary").status_code == 200  # logout closes only this session
    conn = world.db()
    assert conn.execute("SELECT COUNT(*) FROM web_sessions WHERE revoked_at IS NULL").fetchone()[0] == 1
    conn.close()
    r = world.bot.post("/internal/sessions/revoke_all", json={"tg_id": OWNER}, headers=KEY_HEADERS)
    assert r.json() == {"revoked": 1}
    assert second.get("/api/summary").status_code == 401
    assert world.bot.post("/internal/sessions/revoke_all", json={"tg_id": STRANGER}, headers=KEY_HEADERS).status_code == 403


def test_bot_login_gives_a_link_only_and_a_new_one_cancels_the_old(world: World) -> None:
    first, second = issue(world, None), issue(world, None)
    assert first["status"] == "issued" and "code" not in first and "request" not in first
    assert confirm(world.browser(), path_of(first)).status_code == 400  # replaced by the second
    assert confirm(world.browser(), path_of(second)).status_code == 303
    conn = world.db()
    assert conn.execute("SELECT COUNT(*) FROM login_attempts WHERE code_hash IS NOT NULL").fetchone()[0] == 0
    conn.close()


def test_issue_reports_where_the_request_came_from(world: World) -> None:
    issued = issue(world, start(world.browser(ip="203.0.113.77")))
    assert issued["request"] == {"browser": "Chrome, macOS", "ip": "203.0.113.77"}
    assert weblogin.browser_family("curl/8.7.1") == "неизвестный браузер"
    assert weblogin.browser_family("Mozilla/5.0 (iPhone; CPU iPhone OS 17_5 like Mac OS X) AppleWebKit/605.1.15 "
                                   "(KHTML, like Gecko) Version/17.5 Mobile/15E148 Safari/604.1") == "Safari, iOS"
    conn = world.db()
    stored = " ".join(r["browser"] for r in conn.execute("SELECT browser FROM login_attempts").fetchall())
    conn.close()
    assert "Mozilla" not in stored  # only the family is kept, never the raw header


def test_internal_endpoints_are_closed_to_everyone_but_the_local_bot(world: World) -> None:
    body = {"tg_id": OWNER, "nonce": None}
    assert world.bot.post("/internal/login/issue", json=body).status_code == 404  # no key
    assert world.bot.post("/internal/login/issue", json=body, headers={"Authorization": "Bearer nope"}).status_code == 404
    outside = world.browser(ip="203.0.113.50")  # the right key, but not from the loopback address
    assert outside.post("/internal/login/issue", json=body, headers=KEY_HEADERS).status_code == 404
    assert outside.post("/internal/sessions/revoke_all", json={"tg_id": OWNER}, headers=KEY_HEADERS).status_code == 404


def test_secrets_are_stored_only_as_hashes(world: World) -> None:
    browser = world.browser()
    nonce = start(browser)
    issued = issue(world, nonce)
    binding = browser.cookies.get("dash_login")
    assert browser.post("/auth/code", json={"code": issued["code"]}).status_code == 200
    session = browser.cookies.get("dash_session")
    token = path_of(issued).rsplit("/", 1)[1]
    conn = world.db()
    attempt = conn.execute("SELECT * FROM login_attempts").fetchone()
    stored = conn.execute("SELECT token_hash FROM web_sessions").fetchone()[0]
    everything = " ".join(str(v) for v in tuple(attempt)) + " " + stored
    conn.close()
    for secret in (nonce, binding, token, session, issued["code"]):
        assert secret and secret not in everything
    assert attempt["nonce_hash"] == auth.sha256_hex(nonce) and attempt["magic_hash"] == auth.sha256_hex(token)
    keyed = hmac.new(HMAC_KEY.encode(), f"{auth.sha256_hex(nonce)}:{issued['code']}".encode(), hashlib.sha256)
    assert attempt["code_hash"] == keyed.hexdigest()  # keyed with the server key and tied to the attempt
    assert stored == auth.sha256_hex(session)
    # a stolen dump (nonce hash + code hash) does not give the code: no plain sha256 of the 10**6 codes matches
    leaked = attempt["code_hash"]
    assert not any(hashlib.sha256(f"{attempt['nonce_hash']}:{n:06d}".encode()).hexdigest() == leaked for n in range(10**6))
    assert len(nonce) == len(token) == len(session) == 43  # 256 bits, url-safe


def test_no_ids_tokens_or_codes_in_logs(world: World, caplog: pytest.LogCaptureFixture) -> None:
    caplog.set_level(logging.DEBUG)
    browser = world.browser()
    nonce = start(browser)
    issued = issue(world, nonce)
    binding = browser.cookies.get("dash_login")
    browser.post("/auth/code", json={"code": wrong(issued["code"])})
    world.bot.post("/internal/login/issue", json={"tg_id": OWNER, "nonce": None})  # a refused call is logged too
    browser.post("/auth/code", json={"code": issued["code"]})
    for secret in (str(OWNER), nonce, binding, path_of(issued).rsplit("/", 1)[1], issued["code"], KEY, HMAC_KEY):
        assert secret and secret not in caplog.text


@pytest.mark.parametrize("header", ["Bearer garbage", "Bearer", "Bearer ", "bearer   ", "Basic dXNlcjpwYXNz", "Token abc", "x", ""])
def test_cookie_never_overrides_an_authorization_header(world: World, header: str) -> None:
    """A header that is present must be a correct Bearer token; otherwise 401, the cookie is not tried."""
    browser = world.browser()
    issued = issue(world, start(browser))
    assert browser.post("/auth/code", json={"code": issued["code"]}).status_code == 200
    assert browser.get("/api/summary", headers={"Authorization": header}).status_code == 401
    assert browser.get("/api/sections", headers={"Authorization": header}).status_code == 401
    assert browser.get("/api/sections").status_code == 200  # without the header the cookie speaks


def test_login_page_is_served(world: World) -> None:
    r = world.browser().get("/login")
    assert r.status_code == 200 and "Перейти в Telegram" in r.text and "/auth/start" in r.text
    assert r.headers["referrer-policy"] == "no-referrer"


def test_disabled_without_settings(tmp_path: Path) -> None:
    env = {"BOT_TOKEN": "1:x", "ALLOWED_TG_IDS": "1", "SESSION_SECRET": "s" * 40, "DB_PATH": str(tmp_path / "d.db")}
    off = load_settings(env)
    assert not off.web_login_enabled
    client = TestClient(create_app(off))
    assert client.post("/auth/start").status_code == 404 and client.get("/login").status_code == 404
    full = {**env, "PUBLIC_URL": PUBLIC + "/", "WEB_LOGIN_BOT_USERNAME": "@" + BOT_USERNAME,
            "WEB_LOGIN_INTERNAL_KEY": KEY, "WEB_LOGIN_HMAC_KEY": HMAC_KEY}
    on = load_settings(full)
    assert on.web_login_enabled and on.public_url == PUBLIC and on.web_bot_username == BOT_USERNAME
    assert on.web_hmac_key == HMAC_KEY
    create_app(on)  # both secrets are long enough
    for name in ("WEB_LOGIN_INTERNAL_KEY", "WEB_LOGIN_HMAC_KEY"):
        for bad in ("short", "é" * 15):  # too short: 30 bytes is still short
            with pytest.raises(RuntimeError, match=name):  # web login is configured, a key is weak: the app does not start
                create_app(load_settings({**full, name: bad}))
    with pytest.raises(RuntimeError, match="WEB_LOGIN_HMAC_KEY"):  # configured, but the code key is missing
        create_app(load_settings({**full, "WEB_LOGIN_HMAC_KEY": ""}))
    assert not load_settings({**full, "WEB_LOGIN_INTERNAL_KEY": ""}).web_login_enabled  # no shared key: stays off


def test_session_cookie_is_ignored_when_the_extension_is_off(world: World, tmp_path: Path) -> None:
    browser = world.browser()
    issued = issue(world, start(browser))
    assert browser.post("/auth/code", json={"code": issued["code"]}).status_code == 200
    cookie = {"Cookie": f"dash_session={browser.cookies.get('dash_session')}"}
    assert browser.get("/api/summary").status_code == 200  # on: the session works
    off = replace(world.settings, web_bot_username="", public_url="", internal_key="", web_hmac_key="")
    assert not off.web_login_enabled
    app = create_app(off)  # the same database, the extension switched off: the old session is still stored there
    app.state.clock = lambda: T0
    client = TestClient(app, base_url=PUBLIC, raise_server_exceptions=False)
    assert client.get("/api/summary", headers=cookie).status_code == 401
    assert client.get("/api/sections", headers=cookie).status_code == 401
    # a fresh database has no web_sessions table at all: a well-formed cookie is 401, never a 500
    fresh = Settings(db_path=tmp_path / "fresh.db", bot_token="1:x", session_secret=b"s" * 40,
                     allowed_tg_ids=frozenset({OWNER}), tz=ZoneInfo("UTC"))
    app = create_app(fresh)
    app.state.clock = lambda: T0
    client = TestClient(app, base_url=PUBLIC, raise_server_exceptions=False)
    assert client.get("/api/summary", headers={"Cookie": "dash_session=" + "a" * 43}).status_code == 401
    assert client.post("/auth/logout").status_code == 404  # and the web-login addresses do not exist


# ---------- CSRF: who may press the buttons ----------

def test_cross_origin_posts_are_rejected_and_make_no_session(world: World) -> None:
    attacker, victim = world.browser(), world.browser()
    issued = issue(world, start(attacker))  # the attacker's own valid link and code
    path = path_of(issued)
    field = confirm_form(attacker, path)  # a page the attacker can fetch for himself
    foreign = ({"Origin": EVIL}, {"Origin": "null"}, {"Origin": "http://dashboard.example.com"},
               {"Origin": "https://dashboard.example.com.evil.example"}, {"Referer": EVIL + "/page"},
               {"Origin": EVIL, "Referer": PUBLIC + "/login"}, {})  # a present Origin is never overridden by Referer
    for extra in foreign:
        quiet = world.browser(origin=None)  # no default Origin: only what this attempt sends
        bodies = ((path, {"data": {"csrf": field}}), ("/auth/start", {}), ("/auth/code", {"json": {"code": "123456"}}),
                  ("/auth/logout", {}))
        for url, kw in bodies:
            r = quiet.post(url, headers=extra, follow_redirects=False, **kw)
            assert r.status_code == 403, (url, extra, r.status_code)
            assert "dash_session" not in cookies_of(r) and "dash_login" not in cookies_of(r)
        assert quiet.get("/api/summary").status_code == 401
    # even the right token, posted from a foreign page by the victim's browser, logs nobody in
    r = victim.post(path, data={"csrf": field}, headers={"Origin": EVIL}, follow_redirects=False)
    assert r.status_code == 403 and victim.get("/api/summary").status_code == 401
    conn = world.db()
    assert conn.execute("SELECT COUNT(*) FROM web_sessions").fetchone()[0] == 0
    assert conn.execute("SELECT COUNT(*) FROM login_attempts WHERE consumed_at IS NOT NULL").fetchone()[0] == 0
    conn.close()
    assert attacker.post(path, data={"csrf": field}, follow_redirects=False).status_code == 303  # the link was not spent
    # Referer is the fallback for a client without Origin; the same address passes
    assert world.browser(origin=None).post("/auth/start", headers={"Referer": PUBLIC + "/login"}).status_code == 200


def test_confirm_needs_the_csrf_token_of_this_browser(world: World) -> None:
    issued = issue(world, start(world.browser()))
    path = path_of(issued)
    browser, other = world.browser(), world.browser()
    field, other_field = confirm_form(browser, path), confirm_form(other, path)
    assert field != other_field  # every opened page has its own token

    def refused(r: Any) -> None:
        assert r.status_code == 403 and "dash_session" not in cookies_of(r), r.status_code

    refused(browser.post(path, follow_redirects=False))  # no field
    refused(browser.post(path, data={"csrf": ""}, follow_redirects=False))
    refused(browser.post(path, data={"csrf": "0" * 64}, follow_redirects=False))  # wrong field
    refused(browser.post(path, data={"csrf": other_field}, follow_redirects=False))  # a field of another browser
    refused(browser.post(path, data={"csrf": field.upper()}, follow_redirects=False))
    refused(world.browser().post(path, data={"csrf": field}, follow_redirects=False))  # same Origin, but no cookie
    other.cookies.clear()
    refused(other.post(path, data={"csrf": other_field}, follow_redirects=False))  # cookie missing in its own browser
    assert browser.get("/api/summary").status_code == 401
    other_link = path.rsplit("/", 1)[0] + "/" + "b" * 43
    refused(browser.post(other_link, data={"csrf": field}, follow_redirects=False))  # the token is tied to its link
    ok = browser.post(path, data={"csrf": field}, follow_redirects=False)  # the refusals did not spend the link
    assert ok.status_code == 303 and browser.get("/api/summary").status_code == 200


# ---------- the bot, wired straight to the app ----------

@pytest.fixture()
def sent(monkeypatch: pytest.MonkeyPatch) -> list[tuple[str, dict[str, Any]]]:
    calls: list[tuple[str, dict[str, Any]]] = []
    monkeypatch.setattr(bot_module, "call", lambda token, method, body, timeout=40: calls.append((method, body)))
    return calls


def message(user_id: int, text: str) -> dict[str, Any]:
    return {"update_id": 1, "message": {"chat": {"id": user_id, "type": "private"}, "from": {"id": user_id}, "text": text}}


def test_bot_flow_end_to_end(world: World, sent: list[tuple[str, dict[str, Any]]]) -> None:
    def internal(path: str, body: dict[str, Any]) -> dict[str, Any]:
        return dict(world.bot.post(path, json=body, headers=KEY_HEADERS).json())

    browser = world.browser()
    nonce = start(browser)
    allowed = frozenset({OWNER})
    bot_module.handle("t", message(OWNER, f"/start login_{nonce}"), allowed, PUBLIC + "/", internal)
    method, body = sent[-1]
    text = body["text"]
    assert method == "sendMessage" and body["link_preview_options"] == {"is_disabled": True}
    code = re.search(r"код на странице входа: (\d{6})", text)
    link = re.search(r"https://\S+/auth/magic/\S+", text)
    assert code and link and "Запрос с: Chrome, macOS" in text
    assert browser.post("/auth/code", json={"code": code.group(1)}).status_code == 200
    bot_module.handle("t", message(OWNER, "/login"), allowed, PUBLIC + "/", internal)
    assert "Или введите код" not in sent[-1][1]["text"] and "/auth/magic/" in sent[-1][1]["text"]  # /login: no code
    bot_module.handle("t", message(OWNER, "/logout_all"), allowed, PUBLIC + "/", internal)
    assert sent[-1][1]["text"] == "Готово: закрыто сессий – 1."
    assert browser.get("/api/summary").status_code == 401
    bot_module.handle("t", message(OWNER, "/start"), allowed, PUBLIC + "/", internal)  # plain /start: the Mini App button
    assert "reply_markup" in sent[-1][1]


def test_bot_refuses_strangers_without_calling_the_backend(sent: list[tuple[str, dict[str, Any]]]) -> None:
    def internal(path: str, body: dict[str, Any]) -> dict[str, Any]:
        raise AssertionError("a stranger must never reach the backend")

    bot_module.handle("t", message(STRANGER, "/start login_" + "a" * 43), frozenset({OWNER}), PUBLIC + "/", internal)
    assert sent[-1][1]["text"] == bot_module.NO_ACCESS


def test_bot_survives_a_backend_failure(sent: list[tuple[str, dict[str, Any]]]) -> None:
    def internal(path: str, body: dict[str, Any]) -> dict[str, Any]:
        raise RuntimeError("internal /login: URLError")

    bot_module.handle("t", message(OWNER, "/login"), frozenset({OWNER}), PUBLIC + "/", internal)
    assert "недоступен" in sent[-1][1]["text"] and "internal" not in sent[-1][1]["text"]


def test_bot_without_web_login_behaves_as_before(sent: list[tuple[str, dict[str, Any]]]) -> None:
    bot_module.handle("t", message(OWNER, "/start login_" + "a" * 43), frozenset({OWNER}), PUBLIC + "/")
    assert "reply_markup" in sent[-1][1]
```

## 9. Проверка руками

Тесты не заменяют одного прогона глазами. Куки с `Secure` сохраняются только по HTTPS (исключение – адрес `127.0.0.1` и `localhost` в Chromium), поэтому полную проверку делайте на сервере после деплоя:

1. Откройте `https://<домен>/login` в обычном браузере, нажмите «Войти через Telegram», затем «Перейти в Telegram».
2. В боте должны прийти ссылка, код и строка «Запрос с: <браузер>, <ваш IP>». Если браузер или адрес не ваши – не вводите код.
3. Введите код на странице: откроется дашборд. Выйдите кнопкой «Выйти» и повторите вход ссылкой.
4. Откройте другой браузер, нажмите ссылку из старого сообщения: «Ссылка недействительна».
5. Пошлите боту `/logout_all` из другого аккаунта: ответ «Нет доступа.».

## 10. Чек-лист безопасности

- [ ] Токены (метка, привязка, ссылка, сессия) – 256 случайных бит, в базе только sha256.
- [ ] Код хранится как HMAC от «хеш метки:код» с ключом сервера (`WEB_LOGIN_HMAC_KEY`, 32+ байт, не в базе, только в `backend.env`); сравнение постоянного времени.
- [ ] Любой POST под `/auth/...` проверяет Origin (или Referer); нет обоих, `null` и чужой адрес – 403.
- [ ] Подтверждение ссылки требует токен подтверждения (куки `dash_csrf` + скрытое поле, HMAC над куки и ссылкой).
- [ ] Куки сессии принимается только при включённом расширении; `Authorization` либо верный `Bearer`, либо 401.
- [ ] Ссылка и код одноразовые; ссылка открывается подтверждением (GET ничего не тратит).
- [ ] Срок жизни попытки и кода короткий (5 минут), сессии – 7 дней; настройки `login_ttl_s`, `web_session_ttl_s`.
- [ ] Код работает только в браузере с куки-привязкой; 5 неверных кодов сжигают попытку вместе со ссылкой.
- [ ] Лимит попыток по адресу клиента на старт, на неверные коды и на неверные ссылки.
- [ ] Куки `HttpOnly`, `Secure`, `SameSite=Lax`; куки привязки видна только адресам `/auth/...`.
- [ ] `/internal/...` – только локальный бот и ключ (проверка и в приложении, и в Caddy); снаружи 404.
- [ ] Ключ бота и backend – 32+ символов, только в файлах настроек с правами 640.
- [ ] Allowlist проверяется на каждом запросе и при выдаче: отзыв действует сразу, в том числе на веб-сессии.
- [ ] В бот приходит «Запрос с: <браузер>, <IP>», чтобы заметить чужую ссылку входа.
- [ ] В логах нет id, токенов, кодов, ключа (тест это проверяет).
- [ ] Предпросмотр ссылки в Telegram отключён (`link_preview_options`), страница ссылки отдаёт `Referrer-Policy: no-referrer`.

## 11. Типичные ошибки

- **Ссылка тратится сама.** Telegram и антивирусы открывают ссылки из сообщений для предпросмотра. Если вход выполняет сам GET, ссылка сгорает до владельца. Вход делает только POST после нажатия «Войти».
- **Код действует в любом браузере.** Тогда тому, кто увидел код (или переслал чужую ссылку входа), хватит одного кода. Код проверяется только вместе с куки-привязкой.
- **Счётчик неверных кодов в два шага** (прочитать, потом записать): два одновременных подбора теряют счёт. Счёт и сжигание – один `UPDATE`.
- **`\d` в проверке кода.** В регулярных выражениях Pydantic он принимает и нецифры других алфавитов. Пишите `[0-9]`.
- **Подтверждение без проверки Origin и токена (login CSRF).** Чужая страница отправляет форму с действующей ссылкой злоумышленника из вашего браузера, и вы входите под чужим id. `SameSite=Lax` тут не помогает: он не запрещает принять куки. Нужны обе проверки: Origin на каждом POST и токен подтверждения.
- **`Referrer-Policy: no-referrer` на странице подтверждения.** Браузер тогда шлёт с формой `Origin: null`, и проверка Origin отказывает самому владельцу (проверено в Chromium). Для страницы с кнопкой стоит `same-origin`: на чужие сайты по-прежнему ничего не уходит. Запросы `fetch` со страниц входа этим не затронуты.
- **Куки сессии принимается при выключенном расширении.** Старые сессии остаются живыми, а на свежей базе нет таблицы `web_sessions`, и запрос с обычным куки даёт 500. Куки читается, только когда `web_login_enabled`. При повторном включении сессии, которые не истекли, снова заработают: перед выключением выполните `/logout_all`, если это важно.
- **Неверный `Authorization` уступает куки.** Тогда заголовок `Basic ...` или пустой `Bearer` превращается в вход по куки, вопреки тексту. Присутствующий заголовок проверяется один раз и никогда не заменяется куки.
- **Код хешируется без ключа.** Кодов миллион, и копия базы раскрывает код за секунды. Хеш делается HMAC с ключом, которого в базе нет.
- **Закрытые адреса не закрыты.** Caddy по умолчанию передаёт наружу всё. Добавьте `/internal` в список 404 и проверьте `curl`-ом с другого компьютера.
- **Секреты в логах.** Id, токен ссылки и код легко попадают в сообщение об ошибке или журнал доступа. В службе из `starter.md` журнал доступа отключён (`--no-access-log`): не включайте его обратно, адрес ссылки содержит токен.
- **Куки без `Secure` или `HttpOnly`.** Первую читает любой перехватчик по обычному HTTP, вторую – любой скрипт на странице.
- **Токен в адресе после входа.** После входа страница переходит на `/` (ответ 303), а не остаётся на адресе с токеном: токен не должен попасть в историю браузера и в `Referer`.
- **Один лимит на всех.** Если общий счётчик попыток исчерпает посторонний, владелец не войдёт. Лимиты ведутся по адресу клиента.
- **Старая сессия после отзыва доступа.** Сессия сама не знает про allowlist: проверка в `current_user` на каждый запрос – обязательная.
- **Лимиты в памяти и несколько процессов.** Счётчики лежат в памяти одного процесса, как в `starter.md`. Если запускаете несколько процессов backend, лимиты нужно хранить общими (в базе).
- **Вход по обычному HTTP.** Куки с `Secure` не сохранится (кроме `127.0.0.1` и `localhost` в Chromium), вход «не работает»: проверяйте по HTTPS.

## 12. Готовая просьба агенту

> Добавь вход по одноразовой ссылке и коду из Telegram по `references/web-login.md`. Сначала примени правки, потом запусти тесты на SQLite (и на Postgres, если он у нас есть). На сервер не выкладывай: покажи, что нужно выложить, и жди моего «да».

## 13. Что проверено, что нет

Проверено запуском. Каркас из `starter.md` собран из этого репозитория во временную папку, правки применены из этого файла программой, не вручную; тесты каркаса и новые тесты зелёные:

- SQLite: тесты каркаса и тесты входа;
- PostgreSQL 17 (временный кластер, вариант каркаса из `postgres.md`): те же тесты, ни один не пропущен;
- настоящий браузер (Chromium через Playwright, адрес `http://127.0.0.1`, Secure-куки там сохраняются): вход кодом, вход ссылкой с настоящей отправкой формы, выход; форма с другого источника (другой порт) на ту же ссылку получает 403 и сессию не создаёт; со страницей подтверждения `Referrer-Policy: no-referrer` форма владельца тоже отвергается (`Origin: null`), с `same-origin` проходит;
- ломать код на пробу: подмена условий (ссылка без «использовано», сжигание отключено, проверка локального адреса убрана, проверка allowlist убрана, проверка Origin убрана, проверка токена подтверждения убрана, чтение куки при выключенном расширении, откат к куки при неверном `Authorization`, код снова обычным sha256 и другие) роняет хотя бы один тест.

Не проверено:
- настоящий бот в Telegram и настоящий домен: бот в тестах подключён к приложению напрямую, сеть Telegram не использовалась;
- как Telegram открывает ссылку входа на разных устройствах (приложение, веб-версия, встроенный браузер);
- изменённый `deploy/Caddyfile` не прогонялся через `caddy validate`;
- несколько процессов backend: счётчики лимитов в памяти одного процесса;
- вариант без вашего `PUBLIC_URL` за прокси без `--proxy-headers`: тогда все клиенты выглядят как один адрес, и лимиты общие;
- другие браузеры: проверен только Chromium. Что Firefox и Safari шлют `Origin` на POST формы со страницей `Referrer-Policy: same-origin` (а при его отсутствии `Referer` того же адреса), следует из спецификации, но запуском не проверено;
- настоящий домен с HTTPS: `PUBLIC_URL` и Origin сравнивались только в тестах и на `http://127.0.0.1`.
