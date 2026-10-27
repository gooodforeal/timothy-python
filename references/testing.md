# Тесты: правила и примеры

Справочник к разделу 11 `SKILL.md`. Фреймворк — `pytest`.

## Содержание

1. Что тестировать
2. Расположение и именование
3. Структура теста: Arrange → Act → Assert
4. `parametrize` вместо копипасты
5. Фикстуры и фабрики тестовых данных
6. Фейки и моки — только на границах
7. Время, случайность, файлы, окружение
8. Исключения и логи
9. Асинхронный код
10. Покрытие и запуск

---

## 1. Что тестировать

Этот справочник — про то, **как** писать тесты. **Когда** их писать, решает
раздел 11 `SKILL.md`: тестовые файлы появляются, если пользователь просил тесты
или в проекте уже есть набор тестов. Иначе проверяем код запуском или
одноразовой проверкой во временной папке и предлагаем тесты в ответе.

Когда тесты пишем, **обязательно** покрываем:
- бизнес-правила и расчёты (деньги, скидки, статусы, права доступа);
- граничные случаи: пустой вход, ноль, один элемент, максимум, `None`,
  отрицательные значения, дубликаты;
- ветки с ошибками: функция бросает нужное исключение с понятным сообщением;
- каждый исправленный баг — тест, который падал до фикса;
- контракты между слоями: что API отдаёт на корректный и некорректный запрос.

**Не нужно:**
- тривиальные геттеры, `dataclass` без логики, константы;
- поведение самого Python, фреймворка или библиотеки;
- приватные функции напрямую — они проверяются через публичные.

Чем больше логики и чем дороже ошибка — тем подробнее тесты. Пирамида: много
быстрых юнит-тестов на сервисы, меньше интеграционных (с БД в контейнере),
единицы end-to-end.

## 2. Расположение и именование

```
src/shop_api/services/order_checkout.py
tests/services/test_order_checkout.py
tests/conftest.py                         # общие фикстуры
tests/services/conftest.py                # фикстуры только для сервисов
```

Имя теста: `test_<что>_<ожидаемый результат>_<при каком условии>`.

```python
def test_calculate_discount_rate_returns_zero_for_inactive_user() -> None: ...
def test_checkout_raises_payment_declined_error_when_card_is_rejected() -> None: ...
```

Тест-классы — только для группировки сценариев одной функции, без состояния:
`class TestCalculateDiscountRate:`.

## 3. Структура теста: Arrange → Act → Assert

Три блока, разделённые пустой строкой. Один сценарий — один тест. Действие
(Act) — обычно одна строка.

```python
def test_calculate_cart_total_applies_current_prices() -> None:
    """Сумма корзины считается по актуальным ценам, а не по ценам в момент добавления."""
    items = [CartItem(sku="ABC-000001", quantity=2), CartItem(sku="ABC-000002", quantity=1)]
    price_by_sku = {"ABC-000001": Decimal("100.00"), "ABC-000002": Decimal("50.50")}

    total_rub = calculate_cart_total(items, price_by_sku)

    assert total_rub == Decimal("250.50")
```

- Проверяй **результат и побочные эффекты**, а не порядок внутренних вызовов.
- Несколько `assert` допустимы, если они проверяют один сценарий (например, поля
  одного объекта). Если проверок много — сравни объект целиком.
- Никакой логики в тестах: без `if`, циклов и вычисления ожидаемого значения тем
  же алгоритмом, что и в коде. Ожидаемое значение — литерал.

## 4. `parametrize` вместо копипасты

```python
@pytest.mark.parametrize(
    ("total_spent_rub", "expected_rate"),
    [
        pytest.param(0, 0.0, id="no-orders"),
        pytest.param(100_000, 0.0, id="exactly-threshold"),
        pytest.param(100_001, 0.1, id="above-threshold"),
    ],
)
def test_calculate_discount_rate_depends_on_total_spent(
    total_spent_rub: int, expected_rate: float
) -> None:
    """Скидка VIP начисляется только строго выше порога."""
    user = make_user(orders=[make_order(amount_rub=total_spent_rub)])

    assert calculate_discount_rate(user) == expected_rate
```

`id=` у каждого варианта — чтобы в отчёте было видно, какой случай упал.

## 5. Фикстуры и фабрики тестовых данных

- **Фикстура** — подготовка окружения (клиент, сессия БД, временная папка,
  фейковый шлюз). Общие — в `conftest.py`.
- **Фабрика** — функция, создающая валидный объект с разумными значениями по
  умолчанию; в тесте переопределяется только то, что важно для сценария.

```python
def make_user(*, is_active: bool = True, orders: list[Order] | None = None) -> User:
    """Создаёт валидного пользователя для тестов; поля по умолчанию не важны для сценария."""
    return User(id=UserId(1), email="user@example.com", is_active=is_active, orders=orders or [])


@pytest.fixture
def payment_gateway() -> FakePaymentGateway:
    """Фейковый шлюз, который по умолчанию одобряет все платежи."""
    return FakePaymentGateway()
```

Область фикстуры (`scope="session"`) расширяй, только если создание дорогое и
объект не изменяется тестами — иначе тесты начнут влиять друг на друга.

## 6. Фейки и моки — только на границах

Подменяем то, что медленно, недетерминировано или находится вне нашего кода:
сеть, БД, время, файловую систему, случайность, внешние API.
Внутреннюю бизнес-логику **не мокаем** — иначе тест проверяет моки.

**Предпочтение — фейк через аргумент** (это возможно благодаря DIP):

```python
class FakePaymentGateway:
    """Шлюз в памяти: запоминает списания и умеет имитировать отказ."""

    def __init__(self, *, decline_code: str | None = None) -> None:
        """Создаёт шлюз.

        Args:
            decline_code: Если задан, каждый платёж отклоняется с этим кодом.
        """
        self.decline_code = decline_code
        self.charged_amounts: list[Decimal] = []

    def charge(self, amount_rub: Decimal, card_token: str) -> PaymentResult:
        """Имитирует списание: отклоняет или запоминает сумму."""
        if self.decline_code is not None:
            raise PaymentDeclinedError(self.decline_code)
        self.charged_amounts.append(amount_rub)
        return PaymentResult(is_successful=True)


def test_checkout_charges_order_total() -> None:
    """При оформлении списывается итоговая сумма заказа."""
    gateway = FakePaymentGateway()
    order = make_order(amount_rub=1500)

    checkout(order, gateway)

    assert gateway.charged_amounts == [Decimal("1500")]
```

**`monkeypatch` / `unittest.mock`** — когда зависимость нельзя передать
аргументом (сторонний код, модульный клиент). Патчим там, где имя
**используется**, а не где определено: `monkeypatch.setattr("app.billing.httpx.get", ...)`.
Для HTTP удобнее `respx` (httpx) или `responses` (requests).

## 7. Время, случайность, файлы, окружение

- **Время:** передавай текущее время аргументом (`now: datetime`) или используй
  `time-machine`/`freezegun`. Тест не должен зависеть от того, когда он запущен.
- **Случайность:** передавай `random.Random(seed)` или фиксируй seed.
- **Файлы:** встроенная фикстура `tmp_path` — каждый тест в своей папке.
- **Окружение:** `monkeypatch.setenv` / `delenv`, никогда не трогай реальный `.env`.
- **Сеть:** реальные запросы в юнит-тестах запрещены (`pytest-socket` поможет это
  гарантировать).

## 8. Исключения и логи

```python
def test_find_order_raises_not_found_for_unknown_id(order_repository: FakeOrderRepository) -> None:
    """Запрос несуществующего заказа даёт NotFoundError с id в сообщении."""
    with pytest.raises(NotFoundError, match="Заказ 404"):
        find_order(OrderId(404), order_repository)


def test_import_logs_skipped_rows(caplog: pytest.LogCaptureFixture) -> None:
    """Строки без артикула пропускаются с предупреждением в логе."""
    caplog.set_level(logging.WARNING)

    import_price_rows([{"sku": "", "price": "10"}])

    assert "пропущена строка без артикула" in caplog.text
```

`pytest.raises` оборачивает **только** вызов, который должен упасть, — не
подготовку.

## 9. Асинхронный код

```python
@pytest.mark.anyio
async def test_fetch_user_profiles_preserves_order(fake_profile_client: FakeProfileClient) -> None:
    """Профили возвращаются в порядке переданных id, даже если ответы пришли вразнобой."""
    profiles = await fetch_user_profiles([3, 1, 2], fake_profile_client)

    assert [profile.user_id for profile in profiles] == [3, 1, 2]
```

Плагин — `anyio` (идёт с httpx) или `pytest-asyncio`; выбери один на проект.

## 10. Покрытие и запуск

```bash
pytest                                 # все тесты
pytest tests/services -k discount      # подмножество по пути и имени
pytest --cov=src --cov-report=term-missing
```

- Покрытие — индикатор пропусков, а не цель. Важнее, что покрыты ветки
  бизнес-логики и ошибок, чем общий процент.
- Тесты быстрые: юнит-тесты всего проекта — секунды. Медленные помечай
  `@pytest.mark.slow` и запускай отдельно.
- Упавший тест чинят или удаляют осознанно; `skip`/`xfail` — только с причиной:
  `@pytest.mark.xfail(reason="API v2 ещё не поддерживает фильтр, PAY-201")`.
