# Безопасность: паттерны «плохо → хорошо»

Справочник к разделу 10 `SKILL.md`. Любые данные снаружи (HTTP-запрос, файл,
аргумент CLI, сообщение из очереди, ответ чужого API) считаются недоверенными.

## Содержание

1. Секреты и конфигурация
2. SQL-инъекции
3. Команды ОС
4. Десериализация и выполнение кода
5. Пути и файлы
6. Токены, пароли, сравнение секретов
7. Валидация внешнего ввода
8. Сеть и HTTP
9. Логи и сообщения об ошибках
10. Зависимости
11. Инструменты проверки

---

## 1. Секреты и конфигурация

```python
# Плохо: секрет в коде попадает в git навсегда
API_TOKEN = "sk_live_51H..."

# Хорошо: единственное место чтения окружения — config.py
class Settings(BaseSettings):
    """Настройки приложения из переменных окружения."""

    model_config = SettingsConfigDict(env_file=".env")

    database_url: PostgresDsn
    payment_api_token: SecretStr  # SecretStr не выводится в repr и логи


settings = Settings()
```

- `.env` — в `.gitignore`; рядом `.env.example` с именами переменных и пояснениями
  без значений.
- Секрет случайно попал в git — его нужно **отозвать и перевыпустить**, удаление
  из истории не помогает.
- Секреты не передаются в аргументах командной строки (видны в `ps`) и в URL
  (оседают в логах прокси).

## 2. SQL-инъекции

```python
# Плохо: email='; DROP TABLE users; -- выполнится
cursor.execute(f"SELECT * FROM users WHERE email = '{email}'")

# Хорошо: параметр передаётся отдельно, драйвер сам экранирует
cursor.execute("SELECT id, email FROM users WHERE email = %s", (email,))

# Хорошо: ORM / query builder
session.scalars(select(User).where(User.email == email))
```

Имена колонок/таблиц параметрами не передаются — если они приходят от
пользователя, выбираем из белого списка:

```python
SORTABLE_COLUMNS = {"created_at": Order.created_at, "total": Order.total_rub}

sort_column = SORTABLE_COLUMNS.get(requested_sort)
if sort_column is None:
    raise ValidationError(f"Сортировка по {requested_sort!r} не поддерживается")
```

## 3. Команды ОС

```python
# Плохо: filename='a.png; rm -rf ~' выполнит вторую команду
os.system(f"convert {filename} thumbnail.png")
subprocess.run(f"convert {filename} thumbnail.png", shell=True)

# Хорошо: список аргументов, без оболочки; ошибки не теряются; есть таймаут
subprocess.run(
    ["convert", str(source_path), str(thumbnail_path)],
    check=True,
    timeout=CONVERT_TIMEOUT_SECONDS,
    capture_output=True,
)
```

Аргумент, начинающийся с `-`, может быть принят за опцию — где поддерживается,
отделяй файлы `--`: `["git", "log", "--", user_path]`.

## 4. Десериализация и выполнение кода

| Опасно для недоверенных данных | Безопасная замена |
|---|---|
| `eval`, `exec`, `compile` | `ast.literal_eval` для литералов, явный парсер, `json` |
| `pickle.loads`, `shelve`, `marshal`, `joblib.load` | `json`, `msgpack`, pydantic-модели |
| `yaml.load` | `yaml.safe_load` |
| `xml.etree` на внешних XML | `defusedxml` |
| `__import__` / `importlib` по имени от пользователя | словарь-реестр разрешённых обработчиков |

`pickle` допустим только для данных, которые приложение само создало и хранит в
месте, куда никто другой не пишет.

## 5. Пути и файлы

```python
UPLOADS_DIR = Path("/srv/app/uploads").resolve()


def resolve_upload_path(requested_name: str) -> Path:
    """Возвращает путь к файлу внутри папки загрузок.

    Raises:
        ValidationError: Путь выходит за пределы папки загрузок.
    """
    # resolve() раскрывает «..» и симлинки — после этого проверка надёжна
    candidate_path = (UPLOADS_DIR / requested_name).resolve()
    if not candidate_path.is_relative_to(UPLOADS_DIR):
        raise ValidationError(f"Недопустимый путь: {requested_name!r}")
    return candidate_path
```

- Имена загружаемых файлов не используем как есть — генерируем свои (`uuid4`) и
  храним исходное имя в БД.
- Ограничиваем размер загрузки и проверяем тип по содержимому, а не по расширению.
- Временные файлы — `tempfile`, не самодельные пути в `/tmp`.

## 6. Токены, пароли, сравнение секретов

```python
# Плохо: random предсказуем
reset_token = "".join(random.choices(string.ascii_letters, k=32))

# Хорошо: криптостойкий генератор
reset_token = secrets.token_urlsafe(32)
```

```python
# Плохо: обычное сравнение уязвимо к атаке по времени
if provided_signature == expected_signature: ...

# Хорошо: сравнение за постоянное время
if hmac.compare_digest(provided_signature, expected_signature): ...
```

- Пароли — только хэш медленной функцией: `argon2-cffi` или `bcrypt`. Не `md5`,
  не `sha256`, не обратимое шифрование.
- Хэш для целостности/подписи — `hashlib.sha256` / `hmac`; `md5`/`sha1` —
  только для некриптографических задач (`usedforsecurity=False`).

## 7. Валидация внешнего ввода

Валидируем **на границе** по схеме, внутри — доверяем типам.

```python
class CreateOrderRequest(BaseModel):
    """Тело запроса на создание заказа."""

    model_config = ConfigDict(extra="forbid")  # неизвестные поля — ошибка, а не тихий пропуск

    customer_email: EmailStr
    items: list[OrderItemRequest] = Field(min_length=1, max_length=100)
    comment: str = Field(default="", max_length=500)
```

- Ограничения длины строк, размеров списков, диапазонов чисел — всегда; иначе
  один запрос может съесть память.
- `extra="forbid"` для входящих моделей — защита от mass assignment
  (`{"is_admin": true}` в теле запроса).
- Права доступа проверяются на сервере для **каждого** объекта: пользователь
  может запросить чужой `order_id`.

## 8. Сеть и HTTP

- Таймаут у каждого запроса; `verify=False` запрещён.
- URL, полученный от пользователя, перед запросом проверяется по белому списку
  хостов (защита от SSRF — запросов к внутренним адресам `169.254.169.254`,
  `localhost`, `10.0.0.0/8`).
- Редиректы на чужие хосты — не следовать автоматически, если URL от пользователя.
- Ответ чужого API валидируется так же, как пользовательский ввод.

## 9. Логи и сообщения об ошибках

- Не логируем: пароли, токены, ключи, полные номера карт, паспортные данные,
  полные тела запросов с персональными данными.
- Пользователю — общее сообщение («Не удалось обработать платёж»), в лог —
  подробности. Трейсбеки и SQL наружу не отдаются.
- Пользовательский ввод в лог — через параметры `%s`/`%r`, чтобы переносы строк
  не подделывали записи.

```python
logger.info("Вход пользователя %s", user.email)  # не логируем пароль и токен сессии
```

## 10. Зависимости

- Версии зафиксированы lock-файлом (`uv.lock`), обновляются осознанно.
- Новую библиотеку проверяй: поддерживается ли, сколько у неё пользователей,
  нет ли опечатки в названии (typosquatting: `reqeusts`).
- Регулярно: `pip-audit` / `uv pip audit` на известные уязвимости.

## 11. Инструменты проверки

- `ruff` с правилами `S` (flake8-bandit) — ловит `shell=True`, `eval`, `pickle`,
  хардкод паролей, `verify=False`. Добавь `"S"` в `select` и игнорируй `S101`
  (assert) в тестах.
- `pip-audit` — уязвимые версии зависимостей.
- `gitleaks` / `detect-secrets` в pre-commit — секреты не попадают в коммит.
