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
MAX_RETRY_ATTEMPTS: Final = 3  # Final — константу нельзя переопределить


def send_reminder(user: User, *, delay_seconds: float = 0.0) -> None:
    """Ставит напоминание пользователю в очередь отправки."""


class ReportBuilder:
    """Собирает отчёт из секций в заданном порядке."""

    def __init__(self, title: str) -> None:
        """Создаёт пустой отчёт с заголовком.

        Args:
            title: Заголовок отчёта.
        """
        self.title = title  # тип выводится из аннотации параметра
        self._sections: list[Section] = []  # пустая коллекция — тип указываем явно
```

Локальные переменные аннотируем, только когда тип не выводится: пустые
коллекции, начальное `None`, объявление переменной перед ветвлением.

```python
# Тип выводится — аннотация не нужна
total_rub = sum(order.amount_rub for order in orders)

# Тип не выводится — аннотация нужна
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
    """Считает сумму корзины по актуальным ценам."""
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
    """Возвращает адрес доставки по умолчанию или None, если адрес не задан."""


def build_shipping_label(user: User) -> ShippingLabel:
    """Формирует этикетку для отправки заказа пользователю.

    Raises:
        MissingAddressError: У пользователя не указан адрес доставки.
    """
    address = get_delivery_address(user)
    # После проверки тайпчекер знает, что address — Address, а не None
    if address is None:
        raise MissingAddressError(user.id)
    return ShippingLabel(recipient=user.full_name, address=address)
```

Функция, которая «может не найти», возвращает `X | None` **или** бросает
исключение — выбери одно по смыслу и задокументируй, не делай оба варианта.

## 4. Структурированные данные вместо `dict[str, Any]`

```python
# Плохо: ни тайпчекер, ни читатель не знают, какие поля есть
def create_invoice(data: dict[str, Any]) -> dict[str, Any]: ...

# Хорошо: внутренние данные — dataclass
@dataclass(frozen=True, slots=True)
class InvoiceRequest:
    """Запрос на выставление счёта.

    Attributes:
        customer_id: Клиент, которому выставляется счёт.
        amount_rub: Сумма к оплате.
        due_date: Крайний срок оплаты.
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
    """Фрагмент ответа API курсов, который мы используем."""

    CharCode: str
    Value: float
    Nominal: int
```

## 5. Фиксированные наборы значений: `Literal`, `Enum`

```python
# Literal — для небольшого набора значений параметра, без поведения
def export_report(report: Report, export_format: Literal["csv", "json"]) -> bytes: ...

# StrEnum — когда значения используются в нескольких местах, хранятся в БД
# или у них есть поведение
class OrderStatus(StrEnum):
    """Статус заказа."""

    CREATED = "created"
    PAID = "paid"
    CANCELLED = "cancelled"
```

Перебор вариантов с проверкой полноты — через `match` и `assert_never`:

```python
def get_status_title(status: OrderStatus) -> str:
    """Возвращает человекочитаемое название статуса."""
    match status:
        case OrderStatus.CREATED:
            return "Создан"
        case OrderStatus.PAID:
            return "Оплачен"
        case OrderStatus.CANCELLED:
            return "Отменён"
        case _:
            # Тайпчекер упадёт здесь, если в Enum добавят статус и забудут обработать
            assert_never(status)
```

## 6. `NewType` и алиасы

```python
# Алиас — просто короткое имя для сложного типа, взаимозаменяем с исходным
type PriceTable = dict[str, dict[date, Decimal]]

# NewType — отдельный тип: UserId и OrderId нельзя перепутать
UserId = NewType("UserId", int)
OrderId = NewType("OrderId", int)


def cancel_order(order_id: OrderId, cancelled_by: UserId) -> None:
    """Отменяет заказ от имени пользователя."""


cancel_order(user.id, order.id)  # тайпчекер поймает перепутанные аргументы
```

## 7. Обобщения (generics)

Синтаксис PEP 695 — без `TypeVar`:

```python
def first_or_default[T](items: Iterable[T], default: T) -> T:
    """Возвращает первый элемент или default, если элементов нет."""
    return next(iter(items), default)


def group_by[K, V](items: Iterable[V], key: Callable[[V], K]) -> dict[K, list[V]]:
    """Группирует элементы по ключу, сохраняя порядок внутри группы."""
    groups: defaultdict[K, list[V]] = defaultdict(list)
    for item in items:
        groups[key(item)].append(item)
    return dict(groups)


class Page[T]:
    """Одна страница результата постраничной выборки."""

    def __init__(self, items: list[T], total_count: int) -> None:
        """Создаёт страницу.

        Args:
            items: Элементы текущей страницы.
            total_count: Общее число элементов во всей выборке.
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
    """Платёжный шлюз, через который списываются деньги."""

    def charge(self, amount_rub: Decimal, card_token: str) -> PaymentResult:
        """Списывает сумму с карты."""
        ...


def checkout(order: Order, gateway: PaymentGateway) -> PaymentResult:
    """Оплачивает заказ через переданный шлюз."""
    return gateway.charge(order.total_rub, order.card_token)
```

`Callable` — для функций-параметров. Если сигнатура сложная (именованные
аргументы) — `Protocol` с `__call__`.

```python
OrderFilter = Callable[[Order], bool]


def select_orders(orders: Iterable[Order], is_selected: OrderFilter) -> list[Order]:
    """Возвращает заказы, для которых фильтр вернул True."""
    return [order for order in orders if is_selected(order)]
```

## 9. `*args`, `**kwargs`, `overload`, `Self`

```python
# Аннотация описывает тип ОДНОГО элемента
def log_values(*values: float, **labels: str) -> None: ...


# Строго типизированные kwargs — через TypedDict + Unpack
class RequestOptions(TypedDict, total=False):
    """Необязательные параметры HTTP-запроса."""

    timeout_seconds: float
    retries: int


def fetch(url: str, **options: Unpack[RequestOptions]) -> bytes: ...
```

`Self` — для методов, возвращающих экземпляр своего класса:

```python
class QueryBuilder:
    """Построитель запроса с цепочкой вызовов."""

    def where(self, condition: str) -> Self:
        """Добавляет условие и возвращает этот же построитель."""
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
    """Преобразует значение из конфига в номер порта.

    Raises:
        ConfigError: Значение не является допустимым номером порта.
    """
    # isinstance сужает object до int — дальше тайпчекер доволен
    if isinstance(raw_value, int) and 1 <= raw_value <= 65_535:
        return raw_value
    raise ConfigError(f"Некорректный порт: {raw_value!r}")
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
