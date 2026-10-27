# Структура проектов

Справочник к разделу 6 `SKILL.md`. Выбирай **самый маленький** шаблон, который
закрывает задачу. Не создавай пустые папки и модули «на будущее» — папка
появляется, когда в ней появляется первый файл с реальным кодом.

## Содержание

1. Как выбрать структуру
2. Ответственность слоёв
3. Шаблоны: скрипт, CLI-утилита, библиотека, маленький сервис, сервис со слоями, бот, data-пайплайн
4. Когда разбивать модуль
5. Эталонный `pyproject.toml`

---

## 1. Как выбрать структуру

| Задача | Структура |
|---|---|
| Разовая автоматизация, < ~300 строк | Один файл-скрипт (3.1) |
| Утилита с командами, несколько модулей | CLI-пакет (3.2) |
| Переиспользуемый код для других проектов | Библиотека в `src/` (3.3) |
| Маленький HTTP-сервис или бот: 1–2 сущности, ~5 эндпоинтов | Плоский пакет из 3–5 модулей (3.4) |
| HTTP API с несколькими сущностями и бизнес-логикой | Сервис со слоями (3.5) |
| Telegram/Discord-бот с несколькими сценариями | Сервис со слоями, `bot/` вместо `api/` (3.6) |
| ETL, обработка данных, ML-подготовка | Пайплайн по этапам (3.7) |

Ориентир: для маленькой задачи решение по стандарту сопоставимо по объёму с
обычным — больше документации и типов, но не в разы больше файлов.
`pyproject.toml`, `.gitignore`, `.env.example`, `Dockerfile` — только если о них
просили или без них проект не запустится.

Растёт проект — переходи к следующему шаблону, когда текущий начинает мешать,
а не заранее.

## 2. Ответственность слоёв

| Слой | Что внутри | Что НЕ должно быть внутри |
|---|---|---|
| `config.py` | Чтение env/конфиг-файла, валидация настроек | Бизнес-логика |
| `models/`, `schemas/` | dataclass / pydantic-модели, Enum | I/O, запросы к БД |
| `services/` | Бизнес-правила, оркестрация шагов | HTTP-объекты (Request/Response), SQL, `print` |
| `repositories/` | Запросы к БД, маппинг строк в модели | Бизнес-решения |
| `clients/` | Обращения к внешним API, ретраи, таймауты | Бизнес-решения |
| `api/`, `cli/`, `bot/` | Разбор входа → вызов сервиса → ответ | Логика сложнее 5–10 строк |
| `exceptions.py` | Иерархия исключений приложения | — |

Направление зависимостей: `api/cli/bot → services → repositories/clients → models`.
Нижние слои ничего не знают о верхних. `config` и `models` может импортировать кто угодно.

---

## 3. Шаблоны

### 3.1. Скрипт

```
export_orders.py        # шапка, константы, функции, main()
```

Внутри: шапка → импорты → константы → функции по шагам → `main()` →
`if __name__ == "__main__": main()`. Аргументы — через `argparse`.

### 3.2. CLI-утилита

```
project-name/
├── pyproject.toml
├── README.md
├── src/
│   └── image_tools/
│       ├── __init__.py          # шапка пакета, версия
│       ├── __main__.py          # python -m image_tools → cli.main()
│       ├── cli.py               # разбор аргументов, вызов команд
│       ├── resizing.py          # логика команды resize
│       ├── watermarking.py      # логика команды watermark
│       └── exceptions.py
└── tests/
    ├── test_resizing.py
    └── test_watermarking.py
```

Модули логики называются по функции (`resizing.py`), а не по слою (`core.py`).

### 3.3. Библиотека

```
project-name/
├── pyproject.toml
├── README.md
├── src/
│   └── geo_distance/
│       ├── __init__.py          # публичный API через __all__
│       ├── py.typed             # маркер для тайпчекеров
│       ├── haversine.py
│       ├── vincenty.py
│       ├── units.py
│       └── exceptions.py
└── tests/
```

Публичный API — только то, что реэкспортировано в `__init__.py`. Всё остальное
считается внутренним.

### 3.4. Маленький сервис (FastAPI, бот)

Одна-две сущности, несколько эндпоинтов, бизнес-логики почти нет.

```
notes_service/
├── __init__.py          # шапка пакета
├── main.py              # приложение FastAPI, роуты, обработчик 404
├── db.py                # подключение к SQLite и все запросы (параметризованные)
├── schemas.py           # pydantic-модели запросов и ответов
├── config.py            # путь к БД и прочие настройки из окружения (если их больше одной-двух)
└── requirements.txt
```

- Роуты тонкие и вызывают функции из `db.py` напрямую — отдельный слой
  `services/` появится вместе с первой бизнес-логикой.
- Собственное исключение (например, `NoteNotFoundError`) можно держать в `db.py`,
  пока оно одно.
- Тесты — по правилам раздела 11 `SKILL.md` (если просили или в проекте уже есть).
- Переходи к шаблону 3.5, когда модуль перерастает ~400 строк или появляются
  независимые сущности.

### 3.5. Веб-сервис со слоями (FastAPI и аналоги)

```
project-name/
├── pyproject.toml
├── .env.example                 # все переменные окружения с пояснениями, без секретов
├── src/
│   └── shop_api/
│       ├── __init__.py
│       ├── main.py              # создание приложения, подключение роутеров
│       ├── config.py            # Settings(BaseSettings)
│       ├── exceptions.py
│       ├── dependencies.py      # провайдеры зависимостей (сессия БД, клиенты)
│       ├── api/
│       │   ├── __init__.py
│       │   ├── orders.py        # роутер /orders — тонкий
│       │   └── users.py
│       ├── schemas/             # pydantic-модели запросов/ответов
│       │   ├── orders.py
│       │   └── users.py
│       ├── models/              # ORM-модели
│       │   ├── order.py
│       │   └── user.py
│       ├── services/            # бизнес-логика
│       │   ├── order_checkout.py
│       │   └── user_registration.py
│       ├── repositories/
│       │   ├── order_repository.py
│       │   └── user_repository.py
│       └── clients/
│           └── payment_gateway.py
├── migrations/
└── tests/
    ├── conftest.py
    ├── api/
    ├── services/
    └── repositories/
```

Пока в сервисе одна-две сущности, `schemas/`, `models/`, `services/` могут быть
одиночными файлами (`schemas.py`, `services.py`); папкой они становятся, когда
файл перерастает ~400 строк или в нём появляются независимые темы.

Альтернатива для крупных сервисов — **разбиение по фичам** (domain-first):

```
src/shop_api/
├── orders/
│   ├── api.py
│   ├── schemas.py
│   ├── models.py
│   ├── service.py
│   └── repository.py
└── users/
    └── ...
```

Выбери один подход на проект и не смешивай.

### 3.6. Бот с несколькими сценариями

```
src/support_bot/
├── __init__.py
├── main.py                      # запуск polling/webhook
├── config.py
├── exceptions.py
├── bot/
│   ├── handlers/                # тонкие обработчики по темам
│   │   ├── start.py
│   │   └── tickets.py
│   ├── keyboards.py
│   └── texts.py                 # все тексты сообщений в одном месте (DRY)
├── services/
│   └── ticket_routing.py
└── repositories/
    └── ticket_repository.py
```

### 3.7. Data-пайплайн / ETL

```
src/sales_pipeline/
├── __init__.py
├── config.py
├── run_pipeline.py              # оркестрация: extract → transform → load
├── extract/
│   ├── crm_export.py
│   └── warehouse_api.py
├── transform/
│   ├── cleaning.py
│   └── aggregation.py
├── load/
│   └── analytics_db.py
└── schemas.py                   # контракты данных между этапами
```

Каждый этап — чистые функции «вход → выход», которые легко тестировать отдельно.

---

## 4. Когда разбивать модуль

Разбивай, если выполняется хотя бы одно:
- модуль больше ~400 строк;
- в модуле есть две темы, которые меняются по разным причинам;
- в имени модуля просится «и» или он называется `utils`/`common`/`misc`;
- для чтения одной функции приходится прыгать по всему файлу.

Не разбивай, если:
- получится модуль из одной функции на 5 строк без собственной темы;
- части всегда меняются вместе.

Разбивать функцию — если она длиннее ~40 строк, вложенность больше 3 уровней,
или в ней видны отдельные смысловые шаги.

---

## 5. Эталонный `pyproject.toml`

```toml
[project]
name = "shop-api"
version = "0.1.0"
description = "HTTP API интернет-магазина"
requires-python = ">=3.12"
dependencies = []

[dependency-groups]
dev = ["pytest", "pytest-cov", "ruff", "mypy"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

# --- Линтер и форматтер ---
[tool.ruff]
line-length = 100
target-version = "py312"
src = ["src", "tests"]

[tool.ruff.lint]
select = [
    "E", "W",   # pycodestyle
    "F",        # pyflakes: неиспользуемые импорты/переменные
    "I",        # isort: порядок импортов
    "N",        # pep8-naming
    "UP",       # современный синтаксис
    "B",        # bugbear: типичные ошибки
    "C4",       # comprehensions
    "SIM",      # упрощения
    "PERF",     # антипаттерны производительности
    "RET",      # return
    "PTH",      # pathlib вместо os.path
    "ERA",      # закомментированный код
    "PL",       # pylint
    "RUF",
    "D",        # pydocstyle: докстринги везде
    "ANN",      # аннотации типов везде
    "S",        # bandit: проверки безопасности
]
ignore = [
    "RUF001", "RUF002", "RUF003",  # разрешаем кириллицу в строках, докстрингах и комментариях
]

[tool.ruff.lint.pydocstyle]
convention = "google"

[tool.ruff.lint.per-file-ignores]
# Тесты где угодно: в tests/ и рядом со скриптом (test_*.py, conftest.py).
# assert и литералы в проверках — норма для тестов; докстринг у теста не обязателен,
# если сценарий ясен из имени (см. testing.md), поэтому D103 тоже отключён.
"**/tests/**" = ["S101", "PLR2004", "D103"]
"**/test_*.py" = ["S101", "PLR2004", "D103"]
"**/conftest.py" = ["S101", "PLR2004", "D103"]

# --- Типы ---
[tool.mypy]
python_version = "3.12"
strict = true
warn_unreachable = true

# --- Тесты ---
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra --strict-markers"
```

Команды проверки:

```bash
ruff format . && ruff check --fix . && mypy src && pytest
```
