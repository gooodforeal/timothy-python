# Ошибки и логирование

Справочник к разделу 9 `SKILL.md`.

## Содержание

1. Иерархия исключений приложения
2. Где ловить исключения
3. Как правильно ловить и перебрасывать
4. Ретраи для внешних сервисов
5. Исключение или возвращаемое значение
6. Настройка логирования
7. Уровни логов
8. Что и как писать в лог
9. Антипаттерны

---

## 1. Иерархия исключений приложения

Одно базовое исключение и несколько наследников **по смыслу реакции**, а не по
месту возникновения. Хватает 3–7 классов на приложение.

```python
"""Application exceptions of shop_api.

Each exception maps to one reaction at the top level: NotFoundError ->
HTTP 404, ValidationError -> 422, ExternalServiceError -> 502 and retry later.
"""


class AppError(Exception):
    """Base application exception; catch it to handle any of our errors."""


class NotFoundError(AppError):
    """The requested object does not exist or is not accessible to the user."""


class ValidationError(AppError):
    """The data violates a business rule (the format is already validated by the input schema)."""


class ConflictError(AppError):
    """The operation conflicts with the object's current state, e.g. a repeated payment."""


class ExternalServiceError(AppError):
    """An external service is unavailable or returned an error; the operation can be retried later."""
```

Отдельный класс оправдан, если вызывающему коду нужно ловить именно его или
он несёт данные (`PaymentDeclinedError.decline_code`). Иначе — существующий класс
с понятным сообщением.

## 2. Где ловить исключения

```
api/cli/bot (boundary)   ← catch AppError: log + a clear response to the user
   services             ← catch only if we can handle it (fallback, compensation)
   clients/repositories ← translate foreign exceptions (httpx, DB driver) into ours
```

- **Нижний слой** переводит исключения библиотек в исключения приложения —
  выше никто не должен знать про `httpx.HTTPStatusError` или `psycopg.Error`.
- **Средний слой** обычно ничего не ловит.
- **Граница** — одно место обработки: обработчик исключений фреймворка, `main()`
  CLI, middleware бота.

```python
# clients/payment_gateway.py — translating a foreign error into ours
def charge(self, amount_rub: Decimal, card_token: str) -> PaymentResult:
    """Charges the card through the payment provider API.

    Raises:
        PaymentDeclinedError: The payment provider declined the payment.
        ExternalServiceError: The payment provider is unavailable.
    """
    try:
        response = self._client.post("/charges", json={"amount": str(amount_rub), "token": card_token})
        response.raise_for_status()
    except httpx.HTTPStatusError as error:
        # 402 is a meaningful bank decline shown to the user; anything else is a service failure
        if error.response.status_code == HTTPStatus.PAYMENT_REQUIRED:
            raise PaymentDeclinedError(error.response.json()["decline_code"]) from error
        raise ExternalServiceError(f"Payment gateway responded with {error.response.status_code}") from error
    except httpx.TransportError as error:
        raise ExternalServiceError("Payment gateway is unavailable") from error

    return PaymentResult.model_validate(response.json())
```

```python
# main.py CLI — boundary: the one place where errors become an exit code
def main() -> None:
    """Runs the export and translates errors into exit codes."""
    configure_logging()
    try:
        run_export(parse_arguments())
    except AppError as error:
        # Expected error: a short message without a traceback
        logger.error("Export failed: %s", error)
        sys.exit(1)
```

Неожиданные исключения (не `AppError`) на границе **не ловим широким
`except Exception`** без логирования трейсбека: пусть фреймворк/интерпретатор
покажет полный трейсбек, или ловим и вызываем `logger.exception(...)`.

## 3. Как правильно ловить и перебрасывать

```python
# try only around the line that may raise the expected exception
try:
    raw_config = config_path.read_text(encoding="utf-8")
except FileNotFoundError as error:
    raise ConfigError(f"Config file not found: {config_path}") from error

# Further processing is outside try so a foreign error is not caught by accident
config = parse_config(raw_config)
```

- `from error` — сохраняет цепочку причин в трейсбеке. `from None` — только когда
  причина действительно не нужна (и с комментарием).
- Просто `raise` (без аргументов) — перебросить пойманное как есть, например
  после логирования или отката транзакции.
- Очистка ресурсов — `with` или `finally`, не дублирование кода в каждом `except`.
- Несколько независимых ошибок (параллельные задачи) — `ExceptionGroup` и `except*`.
- Игнорирование ошибки — только явно и с причиной:

```python
# The cache file may have been removed by a parallel process - that is fine
with contextlib.suppress(FileNotFoundError):
    cache_path.unlink()
```

## 4. Ретраи для внешних сервисов

Повторяем только **временные** ошибки (таймаут, 502/503/504, 429) и только
**идемпотентные** операции (или с ключом идемпотентности). Задержка —
экспоненциальная с джиттером, число попыток ограничено.

```python
from tenacity import retry, retry_if_exception_type, stop_after_attempt, wait_exponential_jitter


@retry(
    retry=retry_if_exception_type(ExternalServiceError),
    stop=stop_after_attempt(MAX_RETRY_ATTEMPTS),
    wait=wait_exponential_jitter(initial=0.5, max=10),
    reraise=True,  # after the last attempt raise the original exception, not RetryError
)
def fetch_daily_rates(on_date: date) -> dict[str, Decimal]:
    """Loads central bank rates; transient failures are retried up to MAX_RETRY_ATTEMPTS times."""
```

Без библиотеки — простой цикл с `time.sleep`, но не самодельный «фреймворк ретраев».

## 5. Исключение или возвращаемое значение

| Ситуация | Что делать |
|---|---|
| «Не найдено» — нормальный исход (поиск, кэш) | вернуть `X \| None` |
| «Не найдено» — нарушение ожиданий (объект по id из URL) | `NotFoundError` |
| Нарушение бизнес-правила | исключение приложения |
| Ошибка программиста (невозможное состояние) | `assert` / `assert_never` / `RuntimeError` |
| Проверка «можно ли» до действия | предикат `can_cancel(order) -> bool` |

Не возвращай коды ошибок, `False` или `-1` вместо исключений, и не возвращай
`(result, error)` кортежами — это не идиоматично для Python.

## 6. Настройка логирования

В модулях — только получение логгера:

```python
logger = logging.getLogger(__name__)
```

Настройка — **один раз** в точке входа:

```python
def configure_logging(level: str = "INFO") -> None:
    """Configures the log format and level for the whole application."""
    logging.basicConfig(
        level=level,
        format="%(asctime)s %(levelname)-8s %(name)s: %(message)s",
    )
    # Third-party libraries are noisy at INFO, so leave them warnings only
    logging.getLogger("httpx").setLevel(logging.WARNING)
```

Для сервисов в проде — JSON-логи (`structlog` или `python-json-logger`), чтобы
их можно было искать и фильтровать по полям. Библиотеки не настраивают
логирование и не добавляют хендлеры — это решает приложение.

## 7. Уровни логов

| Уровень | Когда | Пример |
|---|---|---|
| `DEBUG` | Детали для отладки, в проде выключены | `"Ответ API: %d байт за %.2f с"` |
| `INFO` | Значимое событие нормальной работы | `"Импорт завершён: %d товаров обновлено"` |
| `WARNING` | Нештатно, но работа продолжается | `"Курс на %s не найден, взят предыдущий день"` |
| `ERROR` | Операция не выполнена | `"Не удалось отправить счёт %s"` |
| `CRITICAL` | Приложение не может продолжать работу | `"Нет подключения к БД, остановка"` |

`logger.exception("...")` — внутри `except`, пишет `ERROR` + трейсбек.

## 8. Что и как писать в лог

```python
# Good: lazy formatting, context, no secrets
logger.info("Order %s paid: %s RUB, attempt %d", order.id, order.total_rub, attempt)

# Bad: the f-string is formatted even if the level is disabled; no context
logger.debug(f"data: {data}")
logger.error("Error")
```

- В сообщении — идентификаторы (`order_id`, `user_id`, `request_id`), чтобы по
  логу можно было найти конкретный случай.
- Одно событие — одна запись. Не логируй в цикле по миллиону элементов на `INFO`
  — пиши итог.
- Не логируй секреты и персональные данные (см. `security.md`).

## 9. Антипаттерны

| Антипаттерн | Почему плохо | Как правильно |
|---|---|---|
| `except: pass` / `except Exception: pass` | Ошибка исчезает, баг невозможно найти | Ловить конкретное; игнор — `suppress` с комментарием |
| `try` вокруг всей функции | Ловит и чужие ошибки, маскирует баги | `try` вокруг одной опасной строки |
| Лог + `raise` на каждом уровне | Одна ошибка — пять записей в логе | Логировать один раз на границе |
| `raise Exception("...")` | Нельзя поймать избирательно | Исключение приложения |
| `raise NewError(...)` без `from` | Теряется исходная причина | `raise NewError(...) from error` |
| `print(error)` | Не попадает в систему логов, нет уровня и трейсбека | `logger.exception(...)` |
| `except Exception as e: return None` | Вызывающий не знает, что произошла ошибка | Пробросить или вернуть осмысленный результат |
| Сообщение «Ошибка» / «Что-то пошло не так» в логе | Невозможно диагностировать | Что, с каким значением, почему |
