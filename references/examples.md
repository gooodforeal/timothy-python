# Пример рефакторинга «до → после»

Справочник к `SKILL.md`: как типичный «рабочий, но плохой» скрипт приводится
к стандарту. Используй как образец при рефакторинге и ревью.

## Задача

Скрипт читает CSV с заказами, пересчитывает суммы из долларов в рубли по курсу
ЦБ, считает выручку по клиентам и сохраняет топ-10 клиентов в JSON.

---

## Before

```python
import requests, json, csv, os

def main(f, out, debug=False, mode="json"):
    d = []
    with open(f) as file:
        r = csv.reader(file)
        next(r)
        for row in r:
            d.append(row)
    res = requests.get("https://www.cbr-xml-daily.ru/daily_json.js")
    rate = res.json()["Valute"]["USD"]["Value"]
    clients = []
    totals = []
    for row in d:
        if row[3] == "paid":
            amount = float(row[2]) * rate
            if row[1] in clients:
                totals[clients.index(row[1])] += amount
            else:
                clients.append(row[1])
                totals.append(amount)
    result = []
    for i in range(len(clients)):
        result.append({"client": clients[i], "total": round(totals[i], 2)})
    result = sorted(result, key=lambda x: x["total"], reverse=True)[:10]
    try:
        with open(out, "w") as o:
            json.dump(result, o)
    except:
        pass
    if debug:
        print(result)

main("orders.csv", "top.json")
```

### Что не так

| Проблема | Правило |
|---|---|
| Нет шапки, докстрингов, комментариев | Документация (4) |
| Нет аннотаций типов | Типизация (8) |
| `d`, `r`, `res`, `f`, `o`, `main` делает всё | Именование (3), SRP (1) |
| Параметры `debug`, `mode` — `mode` не используется | YAGNI (2) |
| `row[3]`, `row[1]` — магические индексы колонок | DRY / понятность |
| `clients.index(...)` в цикле — O(n²) | Производительность (5) |
| Весь CSV в памяти без нужды | Производительность (5) |
| `float` для денег | Корректность |
| Нет таймаута у запроса, нет проверки статуса | Ошибки (9), Безопасность (10) |
| `except: pass` — ошибка записи молча теряется | Ошибки (9) |
| URL и `10` захардкожены в теле функции | Константы (12) |
| Вызов `main()` при импорте модуля | Структура модуля (6) |
| `print` для отладки | Логирование (9) |

---

## After

Объём задачи небольшой — это **один скрипт**, а не пакет из десяти модулей
(KISS). Если бы появились другие отчёты по тем же заказам, загрузку курса и
чтение заказов стоило бы вынести в отдельные модули.

```python
#!/usr/bin/env python3
""""Top customers by revenue" report in rubles for paid orders.

Purpose:
    Reads an orders export from CSV (amounts in dollars), converts them to rubles
    at the current Central Bank of Russia rate and saves the top customers by
    revenue to JSON. Used by the sales department for the weekly review.

Input CSV format:
    order_id,customer_email,amount_usd,status
    Only rows with status "paid" are counted.

Usage:
    python top_customers_report.py orders.csv --output top_customers.json --limit 10
"""

import argparse
import csv
import json
import logging
from collections import Counter
from collections.abc import Iterable, Iterator
from decimal import ROUND_HALF_EVEN, Decimal
from pathlib import Path

import httpx

logger = logging.getLogger(__name__)

CBR_DAILY_RATES_URL = "https://www.cbr-xml-daily.ru/daily_json.js"
REQUEST_TIMEOUT_SECONDS = 10.0
PAID_STATUS = "paid"
DEFAULT_TOP_LIMIT = 10
KOPECK = Decimal("0.01")


def fetch_usd_rate_rub() -> Decimal:
    """Loads the current dollar-to-ruble rate from the Central Bank of Russia API.

    Returns:
        Price of one dollar in rubles.

    Raises:
        httpx.HTTPError: The API is unavailable or returned an error status.
    """
    response = httpx.get(CBR_DAILY_RATES_URL, timeout=REQUEST_TIMEOUT_SECONDS)
    response.raise_for_status()

    # The rate arrives as a JSON number; go through str into Decimal to avoid float error
    return Decimal(str(response.json()["Valute"]["USD"]["Value"]))


def read_paid_orders(orders_path: Path) -> Iterator[tuple[str, Decimal]]:
    """Reads paid orders from the CSV export row by row.

    Args:
        orders_path: Path to a CSV with columns customer_email, amount_usd, status.

    Yields:
        (customer email, order amount in dollars) pairs for paid orders.
    """
    with orders_path.open(encoding="utf-8", newline="") as orders_file:
        # DictReader: access columns by name, not by fragile indexes
        for row in csv.DictReader(orders_file):
            if row["status"] == PAID_STATUS:
                yield row["customer_email"], Decimal(row["amount_usd"])


def calculate_top_customers(
    paid_orders: Iterable[tuple[str, Decimal]],
    usd_rate_rub: Decimal,
    limit: int,
) -> list[dict[str, str]]:
    """Computes revenue per customer in rubles and returns the leaders.

    Args:
        paid_orders: (customer email, amount in dollars) pairs.
        usd_rate_rub: Dollar-to-ruble rate.
        limit: How many customers to return.

    Returns:
        Customers by descending revenue: ``{"customer_email": ..., "revenue_rub": ...}``.
        The amount is a string so JSON does not lose kopecks.
    """
    # Counter sums by key in O(1) and can return the top N directly
    revenue_usd_by_customer: Counter[str] = Counter()
    for customer_email, amount_usd in paid_orders:
        revenue_usd_by_customer[customer_email] += amount_usd

    # Apply the rate to the totals, not to every order: fewer operations
    return [
        {
            "customer_email": customer_email,
            "revenue_rub": str((revenue_usd * usd_rate_rub).quantize(KOPECK, ROUND_HALF_EVEN)),
        }
        for customer_email, revenue_usd in revenue_usd_by_customer.most_common(limit)
    ]


def parse_arguments() -> argparse.Namespace:
    """Parses command-line arguments."""
    parser = argparse.ArgumentParser(description="Top customers by revenue in rubles.")
    parser.add_argument("orders_path", type=Path, help="CSV orders export")
    parser.add_argument("--output", type=Path, default=Path("top_customers.json"))
    parser.add_argument("--limit", type=int, default=DEFAULT_TOP_LIMIT)
    return parser.parse_args()


def main() -> None:
    """Builds the report from command-line arguments and saves it to JSON."""
    logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
    arguments = parse_arguments()

    # Fetch the rate once per report so all amounts are computed consistently
    usd_rate_rub = fetch_usd_rate_rub()
    top_customers = calculate_top_customers(
        read_paid_orders(arguments.orders_path), usd_rate_rub, arguments.limit
    )

    # ensure_ascii=False keeps non-ASCII text in the report readable without \u escapes
    arguments.output.write_text(
        json.dumps(top_customers, ensure_ascii=False, indent=2), encoding="utf-8"
    )
    logger.info("Report saved: %s (%d customers)", arguments.output, len(top_customers))


if __name__ == "__main__":
    main()
```

### Что изменилось и почему

- **Структура:** одна большая функция → 4 функции по шагам (загрузить курс,
  прочитать заказы, посчитать, сохранить). Каждая тестируется отдельно, у
  `calculate_top_customers` нет I/O — её можно проверить без сети и файлов.
- **YAGNI:** удалены неиспользуемые `mode` и `debug`; не появилось ни классов,
  ни отдельных модулей — для одного отчёта они не нужны.
- **Имена:** `d`, `r`, `res` → `paid_orders`, `usd_rate_rub`,
  `revenue_usd_by_customer`; единицы (`_usd`, `_rub`) — в именах.
- **Производительность:** `list.index` в цикле (O(n²)) → `Counter` (O(n));
  CSV читается потоком генератором; курс умножается на итог клиента, а не на
  каждый заказ.
- **Корректность:** `Decimal` вместо `float`, банковское округление до копеек,
  таймаут и `raise_for_status()` у запроса.
- **Ошибки:** `except: pass` удалён — если файл не записался, скрипт падает с
  понятным трейсбеком, а не делает вид, что всё хорошо.
- **Документация:** шапка с назначением, форматом входа и командой запуска;
  докстринг у каждой функции; комментарии объясняют неочевидные решения.
- **Модуль безопасен для импорта:** запуск только через `if __name__ == "__main__"`.

---

## Мини-примеры частых правок

### Пустые имена → говорящие имена

```python
# Before
def proc(d, f=False):
    r = []
    for i in d:
        if i["s"] == 1 or f:
            r.append(i)
    return r

# After
def filter_active_orders(orders: list[Order], *, include_inactive: bool = False) -> list[Order]:
    """Returns active orders (or all of them if include_inactive=True)."""
    return [order for order in orders if include_inactive or order.is_active]
```

### Вложенные `if` и ручной аккумулятор → ранний выход и `sum()`

```python
# Before
def get_discount(user):
    if user is not None:
        if user.is_active:
            total = 0
            for order in user.orders:
                total = total + order.amount
            if total > 100_000:
                return 0.1
    return 0

# After
VIP_THRESHOLD_RUB = 100_000
VIP_DISCOUNT_RATE = 0.1


def calculate_discount_rate(user: User | None) -> float:
    """Returns the user's discount rate: 10% for VIP, otherwise 0."""
    # Inactive and anonymous users get no discount
    if user is None or not user.is_active:
        return 0.0

    # VIP is a user whose total spend exceeds the threshold
    total_spent_rub = sum(order.amount_rub for order in user.orders)
    return VIP_DISCOUNT_RATE if total_spent_rub > VIP_THRESHOLD_RUB else 0.0
```

### Класс без состояния → функция

```python
# Before
class PriceFormatter:
    def __init__(self):
        pass

    def format(self, price):
        return f"{price:,.2f} ₽".replace(",", " ")

# After
def format_price_rub(price_rub: Decimal) -> str:
    """Formats a ruble amount with a thousands separator: "12 345.00 ₽"."""
    return f"{price_rub:,.2f} ₽".replace(",", " ")
```

### Цепочка `if/elif` → словарь-диспетчер

```python
# Before
def export(report, fmt):
    if fmt == "csv":
        return export_csv(report)
    elif fmt == "json":
        return export_json(report)
    elif fmt == "xlsx":
        return export_xlsx(report)
    else:
        raise ValueError("bad format")

# After
EXPORTER_BY_FORMAT: dict[str, Callable[[Report], bytes]] = {
    "csv": export_csv,
    "json": export_json,
    "xlsx": export_xlsx,
}


def export_report(report: Report, export_format: str) -> bytes:
    """Serializes the report into the chosen format.

    Raises:
        ValueError: The format is not supported.
    """
    exporter = EXPORTER_BY_FORMAT.get(export_format)
    if exporter is None:
        supported = ", ".join(EXPORTER_BY_FORMAT)
        raise ValueError(f"Format {export_format!r} is not supported; available: {supported}")
    return exporter(report)
```

Новый формат добавляется одной строкой в словаре — без правки функции (OCP).

### Зависимость создаётся внутри → передаётся снаружи

```python
# Before: impossible to test without a real DB and SMTP
def register_user(email: str) -> None:
    db = Database(os.environ["DATABASE_URL"])
    mailer = SmtpMailer()
    db.insert_user(email)
    mailer.send_welcome(email)

# After: dependencies are passed explicitly (DIP) and can be replaced in tests
def register_user(email: str, *, user_repository: UserRepository, mailer: Mailer) -> None:
    """Creates a user and sends them a welcome email."""
    user_repository.add(email)
    mailer.send_welcome(email)
```
