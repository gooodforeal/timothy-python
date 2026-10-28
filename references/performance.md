# Производительность: паттерны «до → после»

Справочник к разделу 5 `SKILL.md`. Порядок действий:
1. Выбрать правильный алгоритм и структуру данных (это всегда).
2. Не делать лишнюю работу и лишний I/O (это всегда).
3. Измерить. Оптимизировать дальше только доказанное узкое место.

## Содержание

1. Структуры данных
2. Алгоритмическая сложность
3. Память и потоковая обработка
4. Строки и regex
5. Кэширование и инварианты
6. I/O: файлы, БД, сеть
7. Конкурентность
8. pandas / numpy
9. Как измерять

---

## 1. Структуры данных

| Операция | Плохо | Хорошо |
|---|---|---|
| Проверка вхождения | `x in some_list` — O(n) | `x in some_set` — O(1) |
| Поиск по ключу | цикл по списку | `dict` — O(1) |
| Добавление/удаление с начала | `list.insert(0, x)`, `list.pop(0)` — O(n) | `deque.appendleft`, `deque.popleft` — O(1) |
| Подсчёт | ручной `dict` с `if key in` | `collections.Counter` |
| Группировка | `if key not in d: d[key] = []` | `defaultdict(list)` |
| Top-N из большой коллекции | `sorted(items)[:n]` — O(n log n) | `heapq.nlargest(n, items)` — O(n log k) |
| Поиск в отсортированном | линейный проход | `bisect` — O(log n) |
| Много однотипных объектов | обычный класс | `@dataclass(slots=True)` — меньше памяти, быстрее доступ |

```python
# Before: O(n) per check -> O(n * m) in the loop
banned_emails = load_banned_emails()  # list
allowed_users = [user for user in users if user.email not in banned_emails]

# After: O(1) per check -> O(n + m)
banned_emails = set(load_banned_emails())
allowed_users = [user for user in users if user.email not in banned_emails]
```

## 2. Алгоритмическая сложность

Главный источник тормозов — вложенный поиск. Строим индекс один раз.

```python
# Before: O(orders * customers)
for order in orders:
    for customer in customers:
        if customer.id == order.customer_id:
            order.customer_name = customer.name

# After: O(orders + customers)
# Customer index by id: built once, then lookups are O(1)
customer_by_id = {customer.id: customer for customer in customers}
for order in orders:
    order.customer_name = customer_by_id[order.customer_id].name
```

Повторная сортировка внутри цикла, `list.remove` в цикле, `count()` в цикле —
тоже скрытый O(n²).

## 3. Память и потоковая обработка

```python
# Before: the whole file in memory
lines = path.read_text().splitlines()
error_count = len([line for line in lines if "ERROR" in line])

# After: line by line, O(1) memory
with path.open(encoding="utf-8") as log_file:
    error_count = sum(1 for line in log_file if "ERROR" in line)
```

- Генераторное выражение вместо списка, если результат проходится один раз.
- `itertools.islice`, `itertools.chain`, `itertools.batched` (3.12+) — для
  потоков без промежуточных списков.
- Читай из БД курсором/чанками (`yield_per`, `fetchmany`), а не `fetchall()`
  на миллион строк.
- Не копируй большие коллекции без нужды (`list(x)`, срезы `[:]`).

## 4. Строки и regex

```python
# Before: O(n²) — each concatenation creates a new string
report = ""
for row in rows:
    report += f"{row.name};{row.total}\n"

# After: O(n)
report = "".join(f"{row.name};{row.total}\n" for row in rows)
```

```python
# Before: the pattern is parsed on every call (the re cache is limited and not free)
def is_valid_sku(sku: str) -> bool:
    return re.fullmatch(r"[A-Z]{3}-\d{6}", sku) is not None

# After: compile once at module import
SKU_PATTERN = re.compile(r"[A-Z]{3}-\d{6}")  # Three capital letters, a hyphen, six digits


def is_valid_sku(sku: str) -> bool:
    """Checks that the SKU matches the ABC-123456 format."""
    return SKU_PATTERN.fullmatch(sku) is not None
```

Простые проверки — строковыми методами, без regex: `startswith`, `endswith`,
`in`, `isdigit`.

## 5. Кэширование и инварианты

```python
# Before: the same value is computed on every iteration
for product in products:
    tax_rate = get_tax_rate(settings.region)  # does not depend on product
    product.price_with_tax = product.price * (1 + tax_rate)

# After: the invariant is moved out of the loop
tax_multiplier = 1 + get_tax_rate(settings.region)
for product in products:
    product.price_with_tax = product.price * tax_multiplier
```

```python
from functools import cache


@cache
def get_tax_rate(region_code: str) -> Decimal:
    """Returns the region's tax rate; the result is cached for the process lifetime."""
```

- `functools.cache` / `lru_cache(maxsize=...)` — только для **чистых** функций
  с хэшируемыми аргументами. Для данных, которые меняются, — кэш с TTL.
- `functools.cached_property` — для дорогих вычисляемых атрибутов объекта.

## 6. I/O: файлы, БД, сеть

**N+1 запросов к БД — самая частая проблема.**

```python
# Before: 1 query for orders + N queries for products
orders = session.scalars(select(Order)).all()
for order in orders:
    print(order.items)  # lazy loading: a separate query per order

# After: 2 queries in total
orders = session.scalars(select(Order).options(selectinload(Order.items))).all()
```

- Вставка/обновление пачками (`executemany`, bulk-операции), а не по строке.
- Выбирай только нужные колонки; фильтруй и агрегируй в БД, а не в Python.
- Индексы на колонки из `WHERE`/`JOIN`/`ORDER BY`.
- HTTP: один клиент/сессия на всё приложение (пул соединений), а не новый на
  каждый запрос. Всегда таймаут. Ретраи — с экспоненциальной задержкой и лимитом.

```python
# Before: a new connection per request, no timeout
for user_id in user_ids:
    response = requests.get(f"{API_URL}/users/{user_id}")

# After: reuse connections, limit the wait
with httpx.Client(base_url=API_URL, timeout=REQUEST_TIMEOUT_SECONDS) as client:
    for user_id in user_ids:
        response = client.get(f"/users/{user_id}")
```

## 7. Конкурентность

| Характер задачи | Инструмент |
|---|---|
| Много сетевых/дисковых ожиданий, async-стек | `asyncio` + `TaskGroup` + `Semaphore` |
| Много сетевых ожиданий, синхронные библиотеки | `concurrent.futures.ThreadPoolExecutor` |
| Тяжёлые вычисления на чистом Python | `ProcessPoolExecutor` |
| Численные вычисления | векторизация `numpy`/`polars` — обычно быстрее процессов |

```python
MAX_CONCURRENT_REQUESTS = 10  # The profile service limit is 10 concurrent requests


async def fetch_user_profiles(user_ids: list[int], client: httpx.AsyncClient) -> list[UserProfile]:
    """Loads profiles concurrently without exceeding the concurrency limit."""
    # The semaphore limits the number of simultaneous requests
    semaphore = asyncio.Semaphore(MAX_CONCURRENT_REQUESTS)

    async def fetch_one(user_id: int) -> UserProfile:
        """Loads a single profile, waiting for a free slot."""
        async with semaphore:
            response = await client.get(f"/profiles/{user_id}")
            response.raise_for_status()
            return UserProfile.model_validate(response.json())

    # TaskGroup cancels the remaining tasks if one fails
    async with asyncio.TaskGroup() as task_group:
        tasks = [task_group.create_task(fetch_one(user_id)) for user_id in user_ids]

    return [task.result() for task in tasks]
```

В `async def` нельзя вызывать блокирующий код (`time.sleep`, `requests`,
синхронные драйверы БД) — используй async-аналоги или `asyncio.to_thread`.

## 8. pandas / numpy

```python
# Before: a row-by-row Python loop: hundreds of times slower
for index, row in df.iterrows():
    df.at[index, "total"] = row["price"] * row["quantity"]

# After: vectorized operation
df["total"] = df["price"] * df["quantity"]
```

- Условия — `np.where`, `np.select`, `Series.where`, а не `apply` с `if`.
- Читай только нужные колонки: `pd.read_csv(path, usecols=[...], dtype={...})`.
- Категориальные строки с повторами → `dtype="category"`.
- Большие объёмы — `polars` или чтение чанками.

## 9. Как измерять

```bash
python -m cProfile -s cumulative script.py | head -30   # where the time goes
python -m timeit -s "from module import func" "func()"  # micro-benchmark
py-spy top --pid <PID>                                  # profile of a live process
```

В коде — `time.perf_counter()` для замеров участков. Память — `tracemalloc`.

Оптимизация, которая усложняет код, сопровождается комментарием с причиной и,
по возможности, цифрами: `# Индекс вместо поиска: 12 с → 0.3 с на выгрузке 200k заказов`.
