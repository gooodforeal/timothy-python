# Документация кода: шаблоны и примеры

Справочник к разделу 4 `SKILL.md`. Стиль докстрингов — PEP 257 + Google style.
Весь текст в коде (шапки, докстринги, комментарии, логи) — на английском;
проза этого справочника на русском. Если в проекте принят другой язык — следуй ему.

## Содержание

1. Шапки файлов: модуль, пакет, исполняемый скрипт, тесты
2. Докстринги: функция, класс, `__init__`, метод, property, генератор, async,
   исключение, Protocol, Enum, контекстный менеджер
3. Inline-комментарии: хорошие и плохие
4. Частые ошибки

---

## 1. Шапки файлов

Шапка — модульный докстринг, первая инструкция файла (до импортов, после
shebang и `# -*- coding -*-`, если они нужны). Первая строка — краткое резюме
с точкой, дальше пустая строка и секции.

### 1.1. Обычный модуль

```python
"""Order total calculation with discounts and delivery.

Purpose:
    The single place where the amount to pay is computed. Used at checkout
    (API), in cart recalculation and in the nightly reconciliation with the
    payment provider, so the formula must not be duplicated anywhere else.

Components:
    PriceBreakdown: amount details (items, discount, delivery, total).
    calculate_order_total: computes the order total.

Calculation rules:
    1. The discount applies to items only, not to delivery.
    2. Delivery is free when the items total reaches FREE_DELIVERY_THRESHOLD_RUB.
    3. All amounts are rounded to kopecks using banker's rounding.

Example:
    >>> breakdown = calculate_order_total(order, promo_code="SPRING10")
    >>> breakdown.total_rub
    Decimal('4590.00')
"""
```

Секции подбирай по смыслу — обязательны только резюме и `Purpose:`.
Полезные дополнительные секции: `Rules:`, `Dependencies:`, `Limitations:`,
`Environment variables:`, `Data format:`.

### 1.2. Пакет (`__init__.py`)

```python
"""Billing: tariffs, invoicing and payment processing.

The package groups everything related to the user's money. External code
should use only the public API below; internal modules may change.

Public API:
    create_invoice: issues an invoice for a period.
    process_payment: processes a payment for an invoice.
    BillingError: base exception of the package.
"""

from app.billing.exceptions import BillingError
from app.billing.invoices import create_invoice
from app.billing.payments import process_payment

__all__ = ["BillingError", "create_invoice", "process_payment"]
```

### 1.3. Исполняемый скрипт / CLI

```python
#!/usr/bin/env python3
"""Exports orders for a period to CSV for the accounting department.

Purpose:
    Accounting receives a CSV with all paid orders every month. The script
    runs from cron on the 1st at 03:00 and manually during reconciliation.

Usage:
    python export_orders.py --from 2026-01-01 --to 2026-01-31 --output orders.csv

Environment variables:
    DATABASE_URL: PostgreSQL connection string (required).
    EXPORT_BATCH_SIZE: batch size when reading from the DB (default 5000).

Exit codes:
    0 - success, 1 - invalid arguments, 2 - database error.
"""
```

### 1.4. Файл тестов

```python
"""Tests for order total calculation (app.billing.pricing).

Cover: promo code application, free delivery threshold, kopeck rounding
and rejection of invalid amounts.
"""
```

---

## 2. Докстринги

Общие правила:
- Первая строка — что делает (изъявительное наклонение, с точкой):
  `"""Returns…"""`, `"""Creates…"""`, `"""Checks…"""`. Не «Function for…», не «This method…».
- Типы — в аннотациях, в докстринге не повторяем.
- Секции: `Args:`, `Returns:`, `Yields:`, `Raises:`, `Attributes:`, `Example:`,
  `Note:`. Пишем секцию, только если в ней есть содержание.
- `Raises:` — только исключения, которые функция выбрасывает намеренно и которые
  вызывающий код может/должен обработать.
- Описываем поведение и контракт, а не реализацию. Реализация — в inline-комментариях.

### 2.1. Функция

```python
def split_into_batches(items: Sequence[T], batch_size: int) -> list[list[T]]:
    """Splits a sequence into batches of a fixed size.

    The last batch may be smaller than batch_size. Element order is preserved.

    Args:
        items: Source elements.
        batch_size: Maximum size of a single batch, greater than zero.

    Returns:
        A list of batches; an empty list if items is empty.

    Raises:
        ValueError: If batch_size is less than 1.
    """
```

Короткая очевидная функция — однострочный докстринг:

```python
def is_weekend(day: date) -> bool:
    """Checks whether the date falls on a Saturday or Sunday."""
    return day.weekday() >= 5
```

### 2.2. Класс и `__init__`

В докстринге класса — назначение и публичные атрибуты. В `__init__` — аргументы
конструктора.

```python
class RateLimiter:
    """Limits the call rate of an external API using the token bucket algorithm.

    A single instance is shared by all requests to one API, so the class
    is thread-safe.

    Attributes:
        max_calls_per_second: Allowed average number of calls per second.
    """

    def __init__(self, max_calls_per_second: float, *, burst_size: int = 1) -> None:
        """Creates a limiter with a full token supply.

        Args:
            max_calls_per_second: Average allowed call rate.
            burst_size: How many calls can be made in a row without waiting.
        """
```

Для `dataclass` / pydantic-модели — только докстринг класса с `Attributes:`.

```python
@dataclass(frozen=True, slots=True)
class CurrencyRate:
    """Official exchange rate of one currency to the ruble on a given date.

    Attributes:
        currency_code: ISO 4217 alphabetic code, e.g. ``"USD"``.
        rate_rub: Price of one currency unit in rubles.
        on_date: Date the rate is effective for.
    """

    currency_code: str
    rate_rub: Decimal
    on_date: date
```

### 2.3. Метод

```python
    def acquire(self, *, timeout_seconds: float | None = None) -> bool:
        """Waits for a free token and takes it.

        Args:
            timeout_seconds: Maximum wait time; None means wait indefinitely.

        Returns:
            True if a token was acquired; False if the timeout expired.
        """
```

Приватный метод — тоже с докстрингом, можно короче:

```python
    def _refill_tokens(self) -> None:
        """Refills the token supply in proportion to the elapsed time."""
```

### 2.4. Property

Описываем как атрибут (существительным), без «Returns»:

```python
    @property
    def remaining_days(self) -> int:
        """Number of days until the subscription ends; 0 if it has expired."""
```

### 2.5. Генератор

```python
def read_csv_rows(path: Path, *, encoding: str = "utf-8") -> Iterator[dict[str, str]]:
    """Reads a CSV file row by row without loading it into memory at once.

    Args:
        path: Path to a CSV file with a header in the first row.
        encoding: File encoding.

    Yields:
        A "column name -> value" dict for each data row.

    Raises:
        FileNotFoundError: If the file does not exist.
    """
```

### 2.6. Асинхронная функция

```python
async def fetch_user_profiles(user_ids: Iterable[int], client: httpx.AsyncClient) -> list[UserProfile]:
    """Loads user profiles from the profile service concurrently.

    Requests run concurrently, but no more than MAX_CONCURRENT_REQUESTS at
    a time, to stay within the service rate limit.

    Args:
        user_ids: User identifiers.
        client: Shared HTTP client with configured timeouts.

    Returns:
        Profiles in the same order as user_ids.

    Raises:
        ProfileServiceError: The service returned an error for at least one request.
    """
```

### 2.7. Исключение

```python
class AppError(Exception):
    """Base application exception; catch it to handle any of our errors."""


class PaymentDeclinedError(AppError):
    """The payment provider declined the payment.

    Attributes:
        decline_code: Decline code from the provider, e.g. ``"insufficient_funds"``.
    """

    def __init__(self, decline_code: str) -> None:
        """Stores the decline code and builds a readable message.

        Args:
            decline_code: Decline code from the payment provider.
        """
        super().__init__(f"Payment declined: {decline_code}")
        self.decline_code = decline_code
```

### 2.8. Protocol

```python
class NotificationSender(Protocol):
    """User notification delivery channel (email, Telegram, push)."""

    def send(self, user_id: int, message: str) -> None:
        """Sends a message to the user through this channel.

        Raises:
            DeliveryError: The message could not be delivered.
        """
        ...
```

### 2.9. Enum

```python
class OrderStatus(StrEnum):
    """Order lifecycle; transitions are allowed only forward through the list."""

    CREATED = "created"  # Created, awaiting payment
    PAID = "paid"  # Paid, handed over to the warehouse
    SHIPPED = "shipped"  # Handed over to delivery
    DELIVERED = "delivered"  # Received by the customer
```

### 2.10. Контекстный менеджер

```python
@contextmanager
def temporary_directory(prefix: str) -> Iterator[Path]:
    """Creates a temporary directory and removes it with its contents on exit.

    Args:
        prefix: Directory name prefix, helps find leftovers when debugging.

    Yields:
        Path to the created directory.
    """
```

---

## 3. Inline-комментарии

Комментарий объясняет **намерение, причину или неочевидное следствие**. Если
комментарий можно удалить, ничего не потеряв, — его не должно быть. Если код
нельзя понять без комментария — сначала попробуй переименовать, потом комментируй.

### 3.1. Структура длинной функции — комментарий на каждый шаг

```python
def import_price_list(path: Path, repository: ProductRepository) -> ImportReport:
    """Imports a supplier price list and updates product prices."""
    # 1. Stream the file: supplier price lists can have 2+ million rows
    rows = read_csv_rows(path, encoding="cp1251")

    # 2. Skip rows without a SKU: the supplier puts section headings there
    valid_rows = (row for row in rows if row["sku"].strip())

    # 3. Update in batches of 1000: one UPDATE per row is too slow
    report = ImportReport()
    for batch in itertools.batched(valid_rows, PRICE_UPDATE_BATCH_SIZE):
        updated_count = repository.bulk_update_prices(parse_price_rows(batch))
        report.updated_count += updated_count

    return report
```

Короткая функция — комментарий на каждый неочевидный шаг и на выбор структуры данных:

```python
def deduplicate_events(events: list[Event]) -> list[Event]:
    """Removes duplicate events, keeping the earliest one."""
    # Sort by time so the earliest duplicate comes first
    sorted_events = sorted(events, key=lambda event: event.created_at)

    # A set gives O(1) for the "already seen" check instead of O(n) for a list
    seen_event_ids: set[str] = set()
    unique_events: list[Event] = []

    for event in sorted_events:
        # Broker redelivery arrives with the same event_id, so skip it
        if event.event_id in seen_event_ids:
            continue
        seen_event_ids.add(event.event_id)
        unique_events.append(event)

    return unique_events
```

### 3.2. Хорошие комментарии

```python
# The central bank publishes next-day rates after 15:00 MSK; before that use today's
rates_date = today + timedelta(days=1) if now_moscow.hour >= 15 else today

# Number format: +7 and 10 digits; spaces/brackets/hyphens are already stripped by normalization
PHONE_PATTERN = re.compile(r"^\+7\d{10}$")

# The API returns 500 for an empty list instead of 200 [] - work around it by not sending the request
if not order_ids:
    return []

# Decimal, not float: money must not be computed with binary rounding error
total_rub = sum((item.price_rub for item in items), start=Decimal("0"))

# dict index instead of a list search: O(n + m) instead of O(n * m) for 50k products
product_by_sku = {product.sku: product for product in products}
```

### 3.3. Плохие комментарии

```python
i += 1  # increment i                            ← tautology
users = get_users()  # get the users             ← restates the code
# result = old_algorithm(data)                  ← dead code, delete it
x = y * 1.2  # multiply by 1.2                   ← why 1.2? needs a VAT_MULTIPLIER constant
# TODO: fix this                                ← what, why, when?
```

---

## 4. Частые ошибки

| Ошибка | Как правильно |
|---|---|
| Комментарии, логи или докстринги не на английском | Весь текст в коде — на английском |
| Нет шапки у файла | Шапка есть в каждом `.py`, включая `__init__.py` и тесты |
| "Function for getting a user" | `"""Returns the user by email."""` |
| Типы продублированы: `user_id (int): ...` | `user_id: User identifier.` |
| Докстринг описывает реализацию по шагам | Докстринг — контракт; шаги — inline-комментариями |
| `Raises: Exception` | Конкретные классы исключений и условия |
| Комментарий не обновлён после правки кода | Меняешь код — проверяешь комментарии рядом |
| Нет докстринга у приватного метода | Докстринг у каждого метода, хотя бы однострочный |
