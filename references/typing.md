# Типизация: правила и примеры

Справочник к разделу 8 `SKILL.md`. Синтаксис — Python 3.12+. Проверка —
`mypy --strict` или `pyright` в strict-режиме.

## Содержание

1. Базовые аннотации
2. Коллекции: вход абстрактный, выход конкретный
3. `None` и опциональные значения
4. Структурированные данные вместо `dict[str, Any]`
5. Фиксированные наборы значений: `Literal`, `Enum`
6. `NewType` и алиасы
7. Обобщения (generics)
8. `Protocol` и `Callable`
9. `*args`, `**kwargs`, `overload`, `Self`
10. Внешние данные и `Any`
11. `cast` и `type: ignore`
12. Частые ошибки тайпчекера

---

## 1. Базовые аннотации

Аннотируются **все** параметры и возвращаемые значения, включая `-> None`.

```python
MAX_RETRY_ATTEMPTS: Final = 3  # Final: the constant cannot be reassigned


def send_reminder(user: User, *, delay_seconds: float = 0.0) -> None:
    """Puts a reminder for the user into the send queue."""


class ReportBuilder:
    """Builds a report from sections in the given order."""

    def __init__(self, title: str) -> None:
        """Creates an empty report with a title.

        Args:
            title: Report title.
        """
        self.title = title  # the type is inferred from the parameter annotation
        self._sections: list[Section] = []  # an empty collection: specify the type explicitly
```

Локальные переменные аннотируем, только когда тип не выводится: пустые
коллекции, начальное `None`, объявление переменной перед ветвлением.

```python
# The type is inferred, no annotation needed
total_rub = sum(order.amount_rub for order in orders)

# The type is not inferred, annotation needed
latest_order: Order | None = None
orders_by_status: defaultdict[OrderStatus, list[Order]] = defaultdict(list)
```

## 2. Коллекции: вход абстрактный, выход конкретный

Принимаем самый общий тип, который реально нужен, — так функцию можно вызвать
со списком, кортежем, генератором. Возвращаем конкретный тип — так вызывающий
знает, что у него в руках.

| Что делаем с аргументом | Тип параметра |
|---|---|
| Один раз проходим циклом | `Iterable[T]` |
| Нужен `len()`, индекс, повторный проход | `Sequence[T]` |
| Только читаем словарь | `Mapping[K, V]` |
| Изменяем на месте | `list[T]`, `dict[K, V]`, `MutableMapping[K, V]` |
| Проверяем вхождение | `Collection[T]` или `AbstractSet[T]` |

```python
from collections.abc import Iterable, Mapping


def calculate_cart_total(items: Iterable[CartItem], price_by_sku: Mapping[str, Decimal]) -> Decimal:
    """Computes the cart total using current prices."""
    return sum((price_by_sku[item.sku] * item.quantity for item in items), start=Decimal("0"))
```

Кортежи: `tuple[int, str]` — фиксированная длина и типы; `tuple[int, ...]` —
произвольная длина. Кортеж из 3+ полей со смыслом → `NamedTuple` или `dataclass`.

## 3. `None` и опциональные значения

- Пишем `X | None`, не `Optional[X]`. Значение по умолчанию `None` не делает
  тип опциональным автоматически — пиши явно.
- Перед использованием сужаем тип проверкой (guard clause).

```python
def get_delivery_address(user: User) -> Address | None:
    """Returns the default delivery address or None if no address is set."""


def build_shipping_label(user: User) -> ShippingLabel:
    """Builds a shipping label for the user's order.

    Raises:
        MissingAddressError: The user has no delivery address.
    """
    address = get_delivery_address(user)
    # After the check the type checker knows address is an Address, not None
    if address is None:
        raise MissingAddressError(user.id)
    return ShippingLabel(recipient=user.full_name, address=address)
```

Функция, которая «может не найти», возвращает `X | None` **или** бросает
исключение — выбери одно по смыслу и задокументируй, не делай оба варианта.

## 4. Структурированные данные вместо `dict[str, Any]`

```python
# Bad: neither the type checker nor the reader knows which fields exist
def create_invoice(data: dict[str, Any]) -> dict[str, Any]: ...

# Good: internal data is a dataclass
@dataclass(frozen=True, slots=True)
class InvoiceRequest:
    """Invoice creation request.

    Attributes:
        customer_id: Customer being invoiced.
        amount_rub: Amount to pay.
        due_date: Payment deadline.
    """

    customer_id: CustomerId
    amount_rub: Decimal
    due_date: date
```

| Ситуация | Инструмент |
|---|---|
| Внутренняя модель с поведением/неизменяемая | `@dataclass(frozen=True, slots=True)` |
| Данные снаружи, нужна валидация (API, конфиг, файлы) | `pydantic.BaseModel` |
| Уже существующий словарь (JSON, `**kwargs`), который не превращаем в объект | `TypedDict` |
| Небольшой неизменяемый кортеж с именами | `NamedTuple` |

```python
class RateResponse(TypedDict):
    """The part of the rates API response that we use."""

    CharCode: str
    Value: float
    Nominal: int
```

## 5. Фиксированные наборы значений: `Literal`, `Enum`

```python
# Literal: for a small set of parameter values, without behavior
def export_report(report: Report, export_format: Literal["csv", "json"]) -> bytes: ...

# StrEnum: when values are used in several places, stored in the DB,
# or have behavior
class OrderStatus(StrEnum):
    """Order status."""

    CREATED = "created"
    PAID = "paid"
    CANCELLED = "cancelled"
```

Перебор вариантов с проверкой полноты — через `match` и `assert_never`:

```python
def get_status_title(status: OrderStatus) -> str:
    """Returns a human-readable status label."""
    match status:
        case OrderStatus.CREATED:
            return "Создан"
        case OrderStatus.PAID:
            return "Оплачен"
        case OrderStatus.CANCELLED:
            return "Отменён"
        case _:
            # The type checker fails here if a status is added to the Enum and left unhandled
            assert_never(status)
```

## 6. `NewType` и алиасы

```python
# Alias: just a short name for a complex type, interchangeable with the original
type PriceTable = dict[str, dict[date, Decimal]]

# NewType: a distinct type, UserId and OrderId cannot be mixed up
UserId = NewType("UserId", int)
OrderId = NewType("OrderId", int)


def cancel_order(order_id: OrderId, cancelled_by: UserId) -> None:
    """Cancels an order on behalf of the user."""


cancel_order(user.id, order.id)  # the type checker catches swapped arguments
```

## 7. Обобщения (generics)

Синтаксис PEP 695 — без `TypeVar`:

```python
def first_or_default[T](items: Iterable[T], default: T) -> T:
    """Returns the first element or default if there are none."""
    return next(iter(items), default)


def group_by[K, V](items: Iterable[V], key: Callable[[V], K]) -> dict[K, list[V]]:
    """Groups elements by key, preserving order within a group."""
    groups: defaultdict[K, list[V]] = defaultdict(list)
    for item in items:
        groups[key(item)].append(item)
    return dict(groups)


class Page[T]:
    """A single page of a paginated result."""

    def __init__(self, items: list[T], total_count: int) -> None:
        """Creates a page.

        Args:
            items: Items of the current page.
            total_count: Total number of items in the whole result.
        """
        self.items = items
        self.total_count = total_count
```

Ограничение: `def max_by[T: Comparable](...)` — T должен поддерживать протокол.

## 8. `Protocol` и `Callable`

`Protocol` описывает зависимость по поведению: подходит любой объект с нужными
методами, наследоваться не обязательно. Используй, когда реализаций больше одной
или нужна подмена в тестах (см. YAGNI в `SKILL.md`).

```python
class PaymentGateway(Protocol):
    """Payment gateway through which money is charged."""

    def charge(self, amount_rub: Decimal, card_token: str) -> PaymentResult:
        """Charges the card."""
        ...


def checkout(order: Order, gateway: PaymentGateway) -> PaymentResult:
    """Pays for an order through the given gateway."""
    return gateway.charge(order.total_rub, order.card_token)
```

`Callable` — для функций-параметров. Если сигнатура сложная (именованные
аргументы) — `Protocol` с `__call__`.

```python
OrderFilter = Callable[[Order], bool]


def select_orders(orders: Iterable[Order], is_selected: OrderFilter) -> list[Order]:
    """Returns orders for which the predicate returned True."""
    return [order for order in orders if is_selected(order)]
```

## 9. `*args`, `**kwargs`, `overload`, `Self`

```python
# The annotation describes the type of ONE element
def log_values(*values: float, **labels: str) -> None: ...


# Strictly typed kwargs via TypedDict + Unpack
class RequestOptions(TypedDict, total=False):
    """Optional HTTP request parameters."""

    timeout_seconds: float
    retries: int


def fetch(url: str, **options: Unpack[RequestOptions]) -> bytes: ...
```

`Self` — для методов, возвращающих экземпляр своего класса:

```python
class QueryBuilder:
    """Query builder with call chaining."""

    def where(self, condition: str) -> Self:
        """Adds a condition and returns the same builder."""
        self._conditions.append(condition)
        return self
```

`@overload` — только когда тип результата действительно зависит от типа/значения
аргумента и это важно вызывающему коду. Обычно проще две функции с понятными
именами.

## 10. Внешние данные и `Any`

`Any` выключает проверку типов — ошибки становятся невидимыми. Правила:

- JSON из API, файлы, сообщения очереди → сразу валидируем в модель на границе
  (`RateResponse.model_validate(payload)` / `TypedDict` + проверка).
- Значение неизвестного типа — `object` и сужение через `isinstance`.
- Нетипизированная библиотека — ставим стабы (`types-requests` и т. п.), иначе
  оборачиваем вызов в одну типизированную функцию, и `Any` не расползается.

```python
def parse_port(raw_value: object) -> int:
    """Converts a config value to a port number.

    Raises:
        ConfigError: The value is not a valid port number.
    """
    # isinstance narrows object to int, so the type checker is satisfied
    if isinstance(raw_value, int) and 1 <= raw_value <= 65_535:
        return raw_value
    raise ConfigError(f"Invalid port: {raw_value!r}")
```

## 11. `cast` и `type: ignore`

- `cast()` ничего не проверяет в рантайме — только после реальной проверки или
  гарантии, которую тайпчекер не видит. С комментарием.
- `# type: ignore` — всегда с кодом ошибки и причиной:
  `# type: ignore[attr-defined]  # атрибут добавляется плагином SQLAlchemy`.
- Много `ignore` в одном месте — сигнал, что типы спроектированы неправильно.

## 12. Частые ошибки тайпчекера

| Ошибка | Причина | Исправление |
|---|---|---|
| `Item "None" of "X \| None" has no attribute` | Нет проверки на `None` | Guard clause `if x is None: ...` |
| `Need type annotation for "items"` | Пустая коллекция | `items: list[Order] = []` |
| `Incompatible types in assignment` | Переменная переиспользуется под другой тип | Отдельная переменная с отдельным именем |
| `Returning Any from function declared to return "X"` | Значение из нетипизированного источника | Валидировать в модель / сузить `isinstance` |
| `Argument has incompatible type "list[Child]"; expected "list[Parent]"` | `list` инвариантен | Параметр типа `Sequence[Parent]` |
| `Missing return statement` | Не все ветки возвращают значение | Добавить `return`/`raise` или `assert_never` |
