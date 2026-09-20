# Урок 24. Собственные исключения: наследование от `Exception`

---

## Зачем создавать собственные исключения

Когда в коде что-то идёт не так, Python выбрасывает исключение. Стандартная библиотека предоставляет около 70 встроенных типов исключений: `ValueError`, `TypeError`, `FileNotFoundError`, `KeyError` и другие. Может возникнуть вопрос: зачем создавать собственные, если стандартных так много?

Рассмотрим реальный сценарий. Функция обработки платежа падает с ошибкой. В первом варианте:

```python
raise RuntimeError("Payment declined: insufficient funds")
```

Во втором:

```python
raise InsufficientFundsError(
    amount=1500,
    available=300,
    currency="RUB"
)
```

Первый вариант — просто строка с текстом. Код, который перехватывает это исключение, вынужден парсить строку, чтобы понять причину. Это хрупко и неудобно.

Второй вариант — объект с данными. Код-перехватчик может обратиться к `exc.amount`, `exc.available` и принять решение на основе структурированных данных: предложить пополнить баланс, предложить рассрочку, вернуть конкретный HTTP-статус.

Собственные исключения дают три ключевых преимущества.

**Первое — точная диагностика.** `except InsufficientFundsError` поймает именно недостаток средств, а не любую ошибку при оплате. Разные типы ошибок обрабатываются разными блоками `except`.

**Второе — структурированные данные.** Исключение может нести любую информацию: поле с ошибкой, код ошибки, HTTP-статус, детали для лога. Это не просто строка.

**Третье — иерархия.** `except PaymentError` поймает и `InsufficientFundsError`, и `CardDeclinedError`, и `PaymentTimeoutError` — одним блоком. Это удобно, когда нужна общая обработка для категории ошибок.

---

## Минимальный синтаксис и иерархия встроенных исключений

Создать собственное исключение предельно просто:

```python
class MyError(Exception):
    pass
```

Это полноценное, рабочее исключение. Можно выбрасывать, ловить, передавать сообщение:

```python
raise MyError("Что-то пошло не так")
```

```python
try:
    raise MyError("Произошла ошибка")
except MyError as e:
    print(e)            # Произошла ошибка
    print(repr(e))      # MyError('Произошла ошибка')
    print(e.args)       # ('Произошла ошибка',)
```

Атрибут `args` — кортеж аргументов, переданных при создании исключения. `__str__` по умолчанию возвращает `str(args[0])` если аргумент один, или строку кортежа если их несколько. Всё это наследуется от `Exception` автоматически.

Теперь разберём, почему наследоваться нужно именно от `Exception`, а не от `BaseException`.

В Python иерархия базовых классов исключений выглядит так:

```
BaseException
├── SystemExit          ← sys.exit() — завершение программы
├── KeyboardInterrupt   ← Ctrl+C — прерывание пользователем
├── GeneratorExit       ← закрытие генератора
└── Exception           ← ВСЕ обычные исключения
    ├── StopIteration
    ├── ArithmeticError
    │   ├── ZeroDivisionError
    │   └── OverflowError
    ├── LookupError
    │   ├── KeyError
    │   └── IndexError
    ├── ValueError
    ├── TypeError
    ├── RuntimeError
    └── ... (около 60 других)
```

`SystemExit`, `KeyboardInterrupt` и `GeneratorExit` наследуют от `BaseException` напрямую — они не являются «ошибками» в обычном смысле, это системные события. Поэтому `except Exception` не поймает `KeyboardInterrupt` — и это правильно: если пользователь нажал Ctrl+C, программа должна завершиться, а не поглотить это событие в `except Exception`.

Именно поэтому кастомные исключения должны наследоваться от `Exception`: они являются обычными ошибками приложения, которые должны перехватываться стандартными `except` блоками.

Никогда не наследуйтесь от `BaseException` напрямую, если только вы не создаёте системное событие — что в прикладном коде практически никогда не нужно.

---

## Добавление атрибутов: структурированная информация

Настоящая сила собственных исключений — в дополнительных атрибутах. Исключение становится объектом данных, несущим всю необходимую информацию об ошибке:

```python
class ValidationError(Exception):
    """
    Ошибка валидации входных данных.
    Несёт информацию о конкретных полях с ошибками.
    """

    def __init__(self, message, field=None, errors=None):
        # Передаём message в родительский Exception через super().__init__()
        # Это гарантирует, что str(exc) и exc.args работают корректно
        super().__init__(message)

        self.message = message
        self.field = field          # конкретное поле с ошибкой (если одно)
        self.errors = errors or {}  # словарь {поле: список_ошибок} (если несколько)

    def __str__(self):
        if self.errors:
            error_list = "; ".join(
                f"{field}: {', '.join(msgs)}"
                for field, msgs in self.errors.items()
            )
            return f"Ошибки валидации: {error_list}"
        if self.field:
            return f"Ошибка поля '{self.field}': {self.message}"
        return self.message

    def __repr__(self):
        return (f"ValidationError(message={self.message!r}, "
                f"field={self.field!r}, errors={self.errors!r})")
```

Демонстрируем разные способы использования:

```python
# Простая ошибка с сообщением
try:
    raise ValidationError("Email обязателен")
except ValidationError as e:
    print(e)        # Email обязателен
    print(e.field)  # None

# Ошибка конкретного поля
try:
    raise ValidationError("Некорректный формат", field="email")
except ValidationError as e:
    print(e)        # Ошибка поля 'email': Некорректный формат
    print(e.field)  # email

# Ошибки нескольких полей (результат полной валидации формы)
try:
    raise ValidationError(
        "Форма содержит ошибки",
        errors={
            "username": ["Слишком короткое (минимум 3 символа)"],
            "email":    ["Некорректный формат", "Домен не существует"],
            "age":      ["Должно быть числом"],
        }
    )
except ValidationError as e:
    print(e)
    # Ошибки валидации: username: Слишком короткое (минимум 3 символа);
    #                    email: Некорректный формат, Домен не существует;
    #                    age: Должно быть числом

    # Машинная обработка — работаем с атрибутами, а не парсим строку
    for field, msgs in e.errors.items():
        print(f"  Поле '{field}': {msgs}")
```

Обратите внимание на вызов `super().__init__(message)`. Это важно: он устанавливает `self.args = (message,)`, что обеспечивает корректную работу встроенного `__str__` и корректное отображение исключения в трейсбеке. Если не вызвать `super().__init__()`, атрибут `args` останется пустым кортежем, и некоторые инструменты логирования не смогут правильно отобразить исключение.

---

## Иерархии собственных исключений

Реальные приложения организуют исключения в иерархии. Это позволяет ловить ошибки как точечно (конкретный тип), так и широко (вся категория):

```python
# Уровень 1: базовое исключение всего приложения
class AppError(Exception):
    """Базовое исключение приложения. Все кастомные ошибки наследуют от него."""
    pass


# Уровень 2: категории ошибок
class DatabaseError(AppError):
    """Ошибки при работе с базой данных."""
    pass


class AuthError(AppError):
    """Ошибки аутентификации и авторизации."""
    pass


class ValidationError(AppError):
    """Ошибки валидации данных."""
    def __init__(self, message, errors=None):
        super().__init__(message)
        self.errors = errors or {}


# Уровень 3: конкретные ошибки
class DatabaseConnectionError(DatabaseError):
    """Не удалось подключиться к базе данных."""
    def __init__(self, host, port, original_error=None):
        super().__init__(f"Не удалось подключиться к {host}:{port}")
        self.host = host
        self.port = port
        self.original_error = original_error


class DatabaseQueryError(DatabaseError):
    """Ошибка выполнения запроса."""
    def __init__(self, query, message):
        super().__init__(f"Ошибка запроса: {message}")
        self.query = query


class AuthenticationFailedError(AuthError):
    """Неверные учётные данные."""
    def __init__(self, username):
        super().__init__(f"Аутентификация не удалась для пользователя {username!r}")
        self.username = username


class PermissionDeniedError(AuthError):
    """Доступ запрещён."""
    def __init__(self, user, action, resource):
        super().__init__(
            f"Пользователю {user!r} запрещено действие {action!r} "
            f"над ресурсом {resource!r}"
        )
        self.user = user
        self.action = action
        self.resource = resource
```

Теперь код-перехватчик может реагировать на разных уровнях детализации:

```python
def handle_request(user, action, resource):
    try:
        # Имитация бизнес-логики
        if user == "anonymous":
            raise AuthenticationFailedError(user)
        if resource == "admin_panel" and user != "admin":
            raise PermissionDeniedError(user, action, resource)
        if resource == "broken_db":
            raise DatabaseConnectionError("db.example.com", 5432)

        return {"success": True, "data": f"Результат для {resource}"}

    except AuthenticationFailedError as e:
        # Точный перехват — знаем, что это ошибка аутентификации
        return {"error": "auth_failed", "message": str(e), "status": 401}

    except PermissionDeniedError as e:
        # Знаем конкретно: запрещённое действие
        return {"error": "forbidden", "user": e.user,
                "action": e.action, "status": 403}

    except DatabaseError as e:
        # Широкий перехват — любая ошибка БД
        return {"error": "database_error", "message": str(e), "status": 503}

    except AppError as e:
        # Страховочная сетка — любая ошибка приложения
        return {"error": "app_error", "message": str(e), "status": 500}


print(handle_request("anonymous", "read", "users"))
# {'error': 'auth_failed', 'message': "Аутентификация не удалась...", 'status': 401}

print(handle_request("bob", "delete", "admin_panel"))
# {'error': 'forbidden', 'user': 'bob', 'action': 'delete', 'status': 403}

print(handle_request("admin", "read", "broken_db"))
# {'error': 'database_error', 'message': 'Не удалось подключиться...', 'status': 503}
```

Порядок блоков `except` важен: более специфичные типы должны идти раньше более общих. Если поставить `except AppError` первым — он поймает всё, и до `except DatabaseError` очередь никогда не дойдёт.

---

## Цепочка исключений: `raise ... from ...`

Часто новое исключение возникает в результате другого. Python предоставляет механизм явной цепочки исключений через `raise ... from ...`:

```python
class ServiceError(Exception):
    pass


class DatabaseError(Exception):
    pass


def load_user_from_db(user_id):
    # Имитируем ошибку на уровне БД
    raise DatabaseError(f"Connection timeout при запросе пользователя {user_id}")


def get_user(user_id):
    try:
        return load_user_from_db(user_id)
    except DatabaseError as db_error:
        # Оборачиваем ошибку БД в ошибку сервисного уровня
        # from db_error создаёт явную цепочку: __cause__ = db_error
        raise ServiceError(f"Не удалось загрузить пользователя {user_id}") from db_error


try:
    get_user(42)
except ServiceError as e:
    print(f"Ошибка сервиса: {e}")
    print(f"Причина: {e.__cause__}")
```

Здесь происходит сразу несколько вещей. Исходное исключение `DatabaseError` становится **прямой причиной** (`__cause__`) нового исключения `ServiceError`:

```text
ServiceError.__cause__ → DatabaseError
```

При этом исходное исключение также сохраняется в `__context__`. Это связано с тем, что новое исключение создаётся внутри `except`, то есть Python знает, какое исключение обрабатывалось в этот момент:

```text
ServiceError.__context__ → DatabaseError
ServiceError.__cause__   → DatabaseError
```

Разница между ними заключается не в том, какое исключение они содержат в данном примере, а в **способе построения цепочки**:

* `__context__` — исключение, которое уже обрабатывалось, когда возникло новое исключение. Оно появляется **неявно**.
* `__cause__` — исключение, которое мы **явно указали** через `raise ... from ...`.

Когда используется `from`, Python устанавливает `__cause__` и подавляет автоматическое отображение `__context__` в traceback. Поэтому при `raise ... from ...` пользователю показывается именно явная причина:

```
DatabaseError: Connection timeout при запросе пользователя 42

The above exception was the direct cause of the following exception:

ServiceError: Не удалось загрузить пользователя 42
```

Это очень удобно для отладки: трейсбек показывает полную цепочку причин.

Разберём три варианта поведения:

**Вариант 1: `raise NewError from original`** — явная цепочка. `NewError.__cause__ = original`. В трейсбеке: «The above exception was the direct cause of the following exception».

**Вариант 2: `raise NewError` внутри `except` блока** — неявная цепочка. `NewError.__context__ = original`. В трейсбеке: «During handling of the above exception, another exception occurred».

**Вариант 3: `raise NewError from None`** — скрытие причины. Полезно, когда детали реализации не должны «протекать» наружу:

```python
class APIError(Exception):
    pass


def call_external_api(endpoint):
    try:
        # Имитируем внутреннее исключение с деталями реализации
        raise ConnectionError(f"SSL handshake failed for {endpoint}")
    except ConnectionError:
        # Скрываем детали реализации от внешнего кода
        # from None обрывает цепочку — __cause__ и __context__ будут None
        raise APIError(f"Внешний сервис недоступен") from None


try:
    call_external_api("https://payments.example.com/charge")
except APIError as e:
    print(e)                  # Внешний сервис недоступен
    print(e.__cause__)        # None — детали реализации скрыты
    print(e.__context__)      # None
```

`from None` используется, когда вы намеренно создаёте абстракцию: пользователь API должен знать, что «внешний сервис недоступен», но не должен видеть внутренние детали вроде «SSL handshake failed».

---

## Исключение как объект данных: паттерн для API

В веб-разработке исключения часто должны конвертироваться в HTTP-ответы. Наиболее удобный подход — хранить всю необходимую информацию прямо в объекте исключения:

```python
class HTTPException(Exception):
    """
    Базовое HTTP-исключение. Аналог django.core.exceptions или
    fastapi.HTTPException.

    Хранит всё, что нужно для формирования HTTP-ответа:
    статус-код, код ошибки, сообщение, детали.
    """

    def __init__(self, message, status_code=500, error_code=None, details=None):
        super().__init__(message)
        self.message = message
        self.status_code = status_code
        self.error_code = error_code or f"HTTP_{status_code}"
        self.details = details or {}

    def to_dict(self) -> dict:
        """Конвертирует исключение в словарь для API-ответа."""
        result = {
            "error": self.error_code,
            "message": self.message,
            "status_code": self.status_code,
        }
        if self.details:
            result["details"] = self.details
        return result

    def __str__(self):
        return f"[{self.status_code}] {self.message}"

    def __repr__(self):
        return (f"{self.__class__.__name__}("
                f"status={self.status_code}, "
                f"code={self.error_code!r}, "
                f"message={self.message!r})")


class BadRequestError(HTTPException):
    """400 Bad Request — некорректный запрос клиента."""
    def __init__(self, message="Некорректный запрос", details=None):
        super().__init__(message, status_code=400,
                         error_code="BAD_REQUEST", details=details)


class UnauthorizedError(HTTPException):
    """401 Unauthorized — требуется аутентификация."""
    def __init__(self, message="Требуется аутентификация"):
        super().__init__(message, status_code=401, error_code="UNAUTHORIZED")


class ForbiddenError(HTTPException):
    """403 Forbidden — доступ запрещён."""
    def __init__(self, message="Доступ запрещён", required_permission=None):
        details = {}
        if required_permission:
            details["required_permission"] = required_permission
        super().__init__(message, status_code=403,
                         error_code="FORBIDDEN", details=details)
        self.required_permission = required_permission


class NotFoundError(HTTPException):
    """404 Not Found — ресурс не найден."""
    def __init__(self, resource_type, resource_id=None):
        message = f"{resource_type} не найден"
        if resource_id:
            message = f"{resource_type} с id={resource_id!r} не найден"
        super().__init__(message, status_code=404, error_code="NOT_FOUND")
        self.resource_type = resource_type
        self.resource_id = resource_id


class ValidationHTTPError(HTTPException):
    """422 Unprocessable Entity — ошибки валидации."""
    def __init__(self, errors: dict):
        super().__init__(
            "Ошибки валидации входных данных",
            status_code=422,
            error_code="VALIDATION_ERROR",
            details={"errors": errors}
        )
        self.validation_errors = errors


class InternalServerError(HTTPException):
    """500 Internal Server Error."""
    def __init__(self, message="Внутренняя ошибка сервера"):
        super().__init__(message, status_code=500,
                         error_code="INTERNAL_ERROR")
```

Функция-обработчик, превращающая исключения в HTTP-ответы — аналог middleware в Django/FastAPI:

```python
def exception_handler(func):
    """Декоратор: перехватывает исключения и конвертирует их в ответы."""
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except HTTPException as e:
            return {"success": False, **e.to_dict()}
        except Exception as e:
            # Неожиданная ошибка — логируем и возвращаем 500
            print(f"[ERROR] Неожиданное исключение: {type(e).__name__}: {e}")
            server_error = InternalServerError()
            return {"success": False, **server_error.to_dict()}
    return wrapper


@exception_handler
def get_user(user_id, current_user):
    if not current_user:
        raise UnauthorizedError()
    if user_id == 0:
        raise NotFoundError("User", user_id)
    if current_user.get("role") != "admin" and current_user.get("id") != user_id:
        raise ForbiddenError(required_permission="admin")
    return {"success": True, "user": {"id": user_id, "name": "Alice"}}


@exception_handler
def create_user(data, current_user):
    errors = {}
    if not data.get("username"):
        errors["username"] = ["Поле обязательно"]
    if not data.get("email") or "@" not in data.get("email", ""):
        errors["email"] = ["Некорректный email"]
    if errors:
        raise ValidationHTTPError(errors)
    return {"success": True, "user": {"id": 42, **data}}


# Тестируем
print(get_user(1, None))
# {'success': False, 'error': 'UNAUTHORIZED', 'message': 'Требуется аутентификация', 'status_code': 401}

print(get_user(0, {"id": 1, "role": "admin"}))
# {'success': False, 'error': 'NOT_FOUND', 'message': "User с id=0 не найден", 'status_code': 404}

print(get_user(99, {"id": 1, "role": "user"}))
# {'success': False, 'error': 'FORBIDDEN', ..., 'status_code': 403}

print(get_user(1, {"id": 1, "role": "admin"}))
# {'success': True, 'user': {'id': 1, 'name': 'Alice'}}

print(create_user({"username": "", "email": "bad"}, {"id": 1}))
# {'success': False, 'error': 'VALIDATION_ERROR', ..., 'status_code': 422}
```

---

## Практический пример: исключения в бизнес-логике

Реализуем систему обработки заказов с полноценной иерархией бизнес-исключений:

```python
from decimal import Decimal


# Иерархия бизнес-исключений
class OrderError(Exception):
    """Базовое исключение для ошибок в системе заказов."""

    def __init__(self, message, order_id=None):
        super().__init__(message)
        self.message = message
        self.order_id = order_id

    def to_dict(self) -> dict:
        result = {"error_type": self.__class__.__name__, "message": self.message}
        if self.order_id:
            result["order_id"] = self.order_id
        return result


class InsufficientFundsError(OrderError):
    """Недостаточно средств на счёте."""

    def __init__(self, required: Decimal, available: Decimal,
                 currency: str, order_id=None):
        super().__init__(
            f"Недостаточно средств: требуется {required} {currency}, "
            f"доступно {available} {currency}",
            order_id=order_id
        )
        self.required = required
        self.available = available
        self.currency = currency
        self.shortage = required - available

    def to_dict(self) -> dict:
        d = super().to_dict()
        d.update({
            "required": str(self.required),
            "available": str(self.available),
            "shortage": str(self.shortage),
            "currency": self.currency,
        })
        return d


class ProductUnavailableError(OrderError):
    """Товар недоступен или закончился на складе."""

    def __init__(self, product_id, product_name, requested_qty,
                 available_qty=0, order_id=None):
        if available_qty == 0:
            message = f"Товар '{product_name}' (id={product_id}) отсутствует на складе"
        else:
            message = (f"Товар '{product_name}' (id={product_id}): "
                       f"запрошено {requested_qty}, доступно {available_qty}")
        super().__init__(message, order_id=order_id)
        self.product_id = product_id
        self.product_name = product_name
        self.requested_qty = requested_qty
        self.available_qty = available_qty


class OrderLimitExceededError(OrderError):
    """Превышен лимит заказов для пользователя."""

    def __init__(self, user_id, current_count, max_allowed, order_id=None):
        super().__init__(
            f"Превышен лимит заказов: у пользователя {user_id} уже {current_count} "
            f"активных заказов (максимум {max_allowed})",
            order_id=order_id
        )
        self.user_id = user_id
        self.current_count = current_count
        self.max_allowed = max_allowed


class OrderService:
    """Сервисный слой для работы с заказами."""

    MAX_ORDERS_PER_USER = 5

    def __init__(self):
        # Имитация данных
        self._inventory = {
            "prod_1": {"name": "Ноутбук", "qty": 3, "price": Decimal("89990")},
            "prod_2": {"name": "Мышь", "qty": 0, "price": Decimal("1500")},
            "prod_3": {"name": "Клавиатура", "qty": 10, "price": Decimal("3500")},
        }
        self._user_balances = {
            "user_1": Decimal("150000"),
            "user_2": Decimal("500"),
        }
        self._user_order_counts = {
            "user_1": 2,
            "user_2": 5,
        }

    def place_order(self, user_id: str, product_id: str,
                    quantity: int, order_id: str) -> dict:
        """
        Оформляет заказ. Может выбросить:
        - OrderLimitExceededError
        - ProductUnavailableError
        - InsufficientFundsError
        """
        # Проверка лимита заказов
        current_count = self._user_order_counts.get(user_id, 0)
        if current_count >= self.MAX_ORDERS_PER_USER:
            raise OrderLimitExceededError(
                user_id=user_id,
                current_count=current_count,
                max_allowed=self.MAX_ORDERS_PER_USER,
                order_id=order_id,
            )

        # Проверка наличия товара
        product = self._inventory.get(product_id)
        if not product:
            raise ProductUnavailableError(
                product_id=product_id,
                product_name="(неизвестный)",
                requested_qty=quantity,
                order_id=order_id,
            )
        if product["qty"] < quantity:
            raise ProductUnavailableError(
                product_id=product_id,
                product_name=product["name"],
                requested_qty=quantity,
                available_qty=product["qty"],
                order_id=order_id,
            )

        # Проверка баланса
        total_price = product["price"] * quantity
        balance = self._user_balances.get(user_id, Decimal("0"))
        if balance < total_price:
            raise InsufficientFundsError(
                required=total_price,
                available=balance,
                currency="RUB",
                order_id=order_id,
            )

        # Всё проверено — оформляем заказ
        self._inventory[product_id]["qty"] -= quantity
        self._user_balances[user_id] -= total_price
        self._user_order_counts[user_id] = current_count + 1

        return {
            "order_id": order_id,
            "user_id": user_id,
            "product": product["name"],
            "quantity": quantity,
            "total": str(total_price),
            "status": "confirmed",
        }


def process_order_request(service: OrderService, user_id: str,
                           product_id: str, quantity: int) -> dict:
    """
    API-уровень: вызывает сервис и конвертирует бизнес-исключения в HTTP-ответы.
    """
    import uuid
    order_id = str(uuid.uuid4())[:8]

    try:
        result = service.place_order(user_id, product_id, quantity, order_id)
        return {"success": True, "status": 200, **result}

    except InsufficientFundsError as e:
        # Специфический ответ: предлагаем пополнить баланс
        return {
            "success": False,
            "status": 402,
            **e.to_dict(),
            "suggestion": f"Пополните баланс минимум на {e.shortage} {e.currency}",
        }

    except ProductUnavailableError as e:
        return {"success": False, "status": 409, **e.to_dict()}

    except OrderLimitExceededError as e:
        return {
            "success": False,
            "status": 429,
            **e.to_dict(),
            "suggestion": "Дождитесь завершения активных заказов",
        }

    except OrderError as e:
        # Страховочный блок для любой ошибки заказа
        return {"success": False, "status": 400, **e.to_dict()}


service = OrderService()

# Успешный заказ
print(process_order_request(service, "user_1", "prod_1", 1))
# {'success': True, 'status': 200, 'order_id': ..., 'product': 'Ноутбук', ...}

print()

# Товар закончился
print(process_order_request(service, "user_1", "prod_2", 1))
# {'success': False, 'status': 409, ..., 'message': "Товар 'Мышь' отсутствует на складе"}

print()

# Недостаточно средств
print(process_order_request(service, "user_2", "prod_3", 1))
# {'success': False, 'status': 402, ..., 'suggestion': 'Пополните баланс...'}

print()

# Превышен лимит заказов
print(process_order_request(service, "user_2", "prod_1", 1))
# Сначала выбросит InsufficientFundsError (баланс 500 < 89990)
# Если бы баланс был достаточный — OrderLimitExceededError (5 заказов)
```

---

## Исключения и логирование

В продакшне исключения должны не только обрабатываться, но и логироваться. Python предоставляет специальный метод для логирования с трейсбеком:

```python
import logging

logging.basicConfig(level=logging.INFO,
                    format='%(asctime)s [%(levelname)s] %(message)s')
logger = logging.getLogger(__name__)


def process_payment(amount, user_id):
    try:
        # Имитируем ошибку
        raise InsufficientFundsError(
            required=Decimal("1500"),
            available=Decimal("300"),
            currency="RUB"
        )
    except InsufficientFundsError as e:
        # logging.exception() — логирует сообщение + полный трейсбек
        logger.exception(f"Ошибка оплаты для пользователя {user_id}: {e}")
        return None
    except Exception as e:
        # Для неожиданных ошибок — тоже exception() с трейсбеком
        logger.exception(f"Неожиданная ошибка при оплате: {e}")
        raise   # повторно выбрасываем — дальше решает вызывающий код


# logging.exception() эквивалентно logging.error(exc_info=True)
# exc_info=True добавляет полный трейсбек в лог
# Это критически важно для диагностики проблем в продакшне
```

Разница между методами логирования:
- `logger.error(msg)` — только сообщение, без трейсбека
- `logger.exception(msg)` — сообщение + трейсбек (только внутри `except`)
- `logger.error(msg, exc_info=True)` — то же самое, можно использовать вне `except`

Всегда используйте `logger.exception()` или `exc_info=True` при логировании ошибок — трейсбек критически важен для диагностики в продакшне.

---

## Неочевидные моменты и типичные ошибки

**`except Exception as e: pass` — антипаттерн.** Это «поглощает» все исключения без следа. Если произошла ошибка, она исчезнет молча — невозможно будет понять, что пошло не так:

```python
# ПЛОХО: все ошибки исчезают
try:
    result = process_payment(amount)
except Exception:
    pass   # ошибка съедена, программа продолжает с неверным состоянием

# ХОРОШО: логируем и решаем, что делать дальше
try:
    result = process_payment(amount)
except InsufficientFundsError as e:
    logger.warning(f"Недостаточно средств: {e}")
    return None
except Exception as e:
    logger.exception(f"Неожиданная ошибка: {e}")
    raise   # или возвращаем ошибку пользователю
```

**`raise` без аргументов повторно выбрасывает текущее исключение.** Это полезно, когда нужно что-то сделать с исключением (залогировать) и продолжить его распространение:

```python
try:
    risky_operation()
except SomeError as e:
    logger.exception("Произошла ошибка")
    raise   # повторно выбрасывает SomeError без изменений
```

**Исключения — объекты, их можно хранить и передавать.** Это позволяет собирать список ошибок для валидации:

```python
def validate_form(data):
    errors = []

    try:
        validate_username(data.get("username"))
    except ValidationError as e:
        errors.append(e)

    try:
        validate_email(data.get("email"))
    except ValidationError as e:
        errors.append(e)

    if errors:
        # Создаём одно исключение со списком всех ошибок
        raise ValidationError(
            "Форма содержит ошибки",
            errors={str(i): str(e) for i, e in enumerate(errors)}
        )
```

**Несколько типов в одном `except`.** Вместо дублирования кода:

```python
# Нечитаемо и дублирует код
try:
    result = parse_data(raw)
except ValueError:
    logger.error("Ошибка значения")
    return None
except TypeError:
    logger.error("Ошибка типа")
    return None

# Читаемо и лаконично
try:
    result = parse_data(raw)
except (ValueError, TypeError) as e:
    logger.error(f"Ошибка парсинга: {type(e).__name__}: {e}")
    return None
```

---

## Итоги урока

Собственные исключения — это не просто удобство, а инструмент проектирования. Они позволяют выразить семантику ошибок предметной области, нести структурированные данные об ошибке и строить гибкие иерархии для точечной обработки.

Минимальный синтаксис — `class MyError(Exception): pass`. Для нетривиальных случаев добавляйте `__init__` с атрибутами и вызывайте `super().__init__(message)` — это обеспечивает корректную работу `args` и `__str__`.

Строите иерархии от общего к частному: `AppError` → `DatabaseError` → `ConnectionTimeoutError`. `except DatabaseError` поймёт и `ConnectionTimeoutError` — благодаря наследованию.

`raise ... from original` создаёт явную цепочку исключений, сохраняя контекст в `__cause__`. `raise ... from None` скрывает детали реализации. Оба механизма важны при проектировании API и сервисных слоёв.

В следующем уроке мы рассмотрим множественное наследование и алгоритм MRO — то, как Python разрешает конфликты имён при наследовании от нескольких родителей.

---

## Вопросы

1. Почему кастомные исключения следует наследовать от `Exception`, а не от `BaseException`? Какие исключения наследуют от `BaseException` напрямую и почему?
2. Зачем вызывать `super().__init__(message)` в `__init__` кастомного исключения? Что произойдёт, если не вызвать?
3. В иерархии исключений `AppError → DatabaseError → ConnectionTimeoutError`: что именно поймёт `except DatabaseError`? Что поймёт `except AppError`?
4. Чем `raise MyError from original_error` отличается от `raise MyError` внутри блока `except`?
5. Что означает `raise` без аргументов? В каком контексте это используется?§
6. Почему `except Exception as e: pass` является антипаттерном? Как правильно поступать в ситуации, когда нужно перехватить «любую ошибку»?
7. Как `raise ... from None` изменяет поведение цепочки исключений? Когда это полезно?
8. Можно ли хранить объект исключения в переменной и работать с ним после блока `try/except`? Что при этом важно учитывать?

---

## Задачи

### **Задача 1**. 

Иерархия исключений для интернет-магазина

Создайте иерархию исключений для системы интернет-магазина. Базовое исключение `ShopError(Exception)` с атрибутами `message` и `code` (строка-идентификатор ошибки). Метод `to_dict()` возвращает `{"code": code, "message": message}`. Дочерние категории (классы без реализации):
- `InventoryError(ShopError)` — ошибки склада.
- `PaymentError(ShopError)` — ошибки оплаты.
- `ShippingError(ShopError)` — ошибки доставки.

Конкретные классы:
- `OutOfStockError(InventoryError)` — принимает `product_name`, `requested`, `available`. `code = "OUT_OF_STOCK"`.
- `CardDeclinedError(PaymentError)` — принимает `reason` (строка). `code = "CARD_DECLINED"`.
- `InvalidAddressError(ShippingError)` — принимает `address`. `code = "INVALID_ADDRESS"`.

Напишите функцию `process_purchase(product, quantity, card, address)`, которая имитирует покупку и выбрасывает соответствующие исключения при: `quantity > 5` (нет на складе), `card == "declined"`, `address == ""`. Обработайте все исключения в одном блоке через иерархию.

**Пример использования**:

```python
try:
    process_purchase("Ноутбук", 10, "valid_card", "Москва")
except OutOfStockError as e:
    print(e.to_dict())   # {'code': 'OUT_OF_STOCK', 'message': '...', ...}

try:
    process_purchase("Мышь", 1, "declined", "Москва")
except PaymentError as e:   # ловит CardDeclinedError
    print(e.code)   # CARD_DECLINED

# Перехват любой ошибки магазина
try:
    process_purchase("Клавиатура", 1, "valid", "")
except ShopError as e:
    print(e.to_dict())
```

---

### **Задача 2**. 

Исключение как объект данных для API

Создайте класс `APIException(Exception)` с атрибутами `status_code` (int), `error_code` (str), `message` (str), `details` (dict, по умолчанию `{}`). Метод `to_response()` возвращает словарь `{"status": status_code, "error": error_code, "message": message, "details": details}`. Переопределите `__str__` в формате `"[status_code] error_code: message"`.

Создайте конкретные исключения:
- `NotFound(APIException)` — принимает `resource` и `id`. status=404, error="NOT_FOUND". Из `resource` нужно сформировать `message`: `"{resource} с id={id} не найден"`.
- `Conflict(APIException)` — принимает `resource` и `conflict_field`. status=409, error="CONFLICT". Из `resource` и `conflict_field` нужно сформировать `message`: `"{resource} с таким {conflict_field} уже существует"`.
- `RateLimitExceeded(APIException)` — принимает `limit` и `reset_in_seconds`. status=429, error="RATE_LIMIT".

Напишите декоратор `api_endpoint`, который оборачивает функцию: при `APIException` возвращает `to_response()`, при других исключениях логирует и возвращает 500-ответ (в формате метода `to_response`).

**Пример использования**:

```python
@api_endpoint
def get_user(user_id):
    if user_id == 0:
        raise NotFound("User", user_id)
    return {"id": user_id, "name": "Alice"}

@api_endpoint
def create_user(username):
    if username == "alice":
        raise Conflict("User", "username")
    return {"created": True}

print(get_user(0))
# {'status': 404, 'error': 'NOT_FOUND', 'message': "User с id=0 не найден", 'details': {}}

print(get_user(1))
# {'id': 1, 'name': 'Alice'}

print(create_user("alice"))
# {'status': 409, 'error': 'CONFLICT', ...}
```

---

### **Задача 3**. 

Цепочка исключений при работе с базой данных

Создайте три уровня исключений: низкоуровневое `DriverError(Exception)` (имитирует ошибку драйвера БД), среднеуровневое `RepositoryError(Exception)` (ошибка репозитория), высокоуровневое `ServiceError(Exception)` (ошибка сервисного слоя). Все три уровня исключений только повторяют поведение `Exception`.

Реализуйте три функции-имитации:
- `db_query(query)` — выбрасывает `DriverError` если query содержит "broken".
- `user_repository_find(user_id)` — вызывает `db_query`, при `DriverError` выбрасывает `RepositoryError("Не удалось получить пользователя")` через явную цепочку (`from`).
- `user_service_get(user_id)` — вызывает `user_repository_find`, при `RepositoryError` выбрасывает `ServiceError` через `from`, а при `user_id == 0` — `ServiceError from None` (скрывает детали).

Продемонстрируйте: при обычной ошибке трейсбек показывает цепочку, при `user_id=0` — только `ServiceError`.

**Пример использования**:

```python
try:
    user_service_get(999)    # broken query
except ServiceError as e:
    print(e)
    print(f"__cause__: {e.__cause__}")   # RepositoryError
    print(f"__cause__.__cause__: {e.__cause__.__cause__}")  # DriverError

try:
    user_service_get(0)    # from None
except ServiceError as e:
    print(e)
    print(f"__cause__: {e.__cause__}")   # None — скрыто
```

---

### **Задача 4**. 

Сборщик ошибок валидации

Создайте класс `ValidationError(Exception)` с атрибутами `errors` (словарь `{поле: список_строк}`) и методом `add_error(field, message)`. Метод `is_empty` (property) — True если ошибок нет. Метод `raise_if_errors()` — выбрасывает себя если есть ошибки. Метод `to_dict()` возвращает `{"valid": False, "errors": errors}` или `{"valid": True}`.

Создайте функцию `validate_user_registration(data: dict) -> ValidationError`, которая проверяет:
- `username`: обязательно, 3–20 символов, только `[a-zA-Z0-9_]`.
- `email`: обязательно, должен содержать `@` и `.` после `@`.
- `password`: обязательно, минимум 8 символов, должен содержать хотя бы одну цифру.
- `age`: если передан, должен быть числом от 0 до 150.

Функция собирает ВСЕ ошибки (не останавливается на первой) и возвращает объект `ValidationError`.

**Пример использования**:

```python
result = validate_user_registration({
    "username": "ab",           # слишком короткий
    "email": "notanemail",      # нет @
    "password": "nodigits",     # нет цифр
    "age": 200,                 # вне диапазона
})

print(result.is_empty)   # False
print(result.to_dict())  # {'valid': False, 'errors': {'username': [...], ...}}

try:
    result.raise_if_errors()
except ValidationError as e:
    for field, msgs in e.errors.items():
        print(f"{field}: {msgs}")

# Валидные данные
valid = validate_user_registration({
    "username": "alice_123",
    "email": "alice@example.com",
    "password": "secure123",
})
print(valid.is_empty)    # True
print(valid.to_dict())   # {'valid': True}
```

---

### **Задача 5**. 

Система повторных попыток с исключениями

Создайте иерархию: `RetryableError(Exception)` (ошибку можно повторить), `FatalError(Exception)` (повтор бессмысленен), `MaxRetriesExceededError(Exception)` (принимает `attempts` и `last_error`, сообщение включает оба значения).

Напишите декоратор `retry(max_attempts=3, exceptions=(RetryableError,))`, который:
- вызывает функцию до `max_attempts` раз,
- при `RetryableError` повторяет попытку, выводя `"[Retry N/max] ошибка: message"`,
- при других исключениях — немедленно пробрасывает их без повтора,
- если все попытки исчерпаны — выбрасывает `MaxRetriesExceededError`.

Создайте `NetworkError(RetryableError)` и `AuthError(FatalError)`. Продемонстрируйте работу декоратора.

**Пример использования**:

```python
attempt_counter = 0

@retry(max_attempts=3)
def fetch_data(url):
    global attempt_counter
    attempt_counter += 1
    if attempt_counter < 3:
        raise NetworkError(f"Connection timeout for {url}")
    return {"data": "success"}

result = fetch_data("https://api.example.com")
print(result)   # {'data': 'success'} — успех на 3-й попытке

@retry(max_attempts=2)
def restricted_resource():
    raise AuthError("Invalid token")

try:
    restricted_resource()   # немедленно выбрасывает AuthError без повтора
except AuthError as e:
    print(f"Фатальная ошибка: {e}")
```

---

### **Задача 6**. 

Исключения в pipeline обработки данных

Создайте систему исключений для pipeline обработки данных (продолжение задачи из урока 22). Базовое `PipelineError(Exception)` с атрибутами `step_name` (имя шага, где возникла ошибка) и `input_data`. Метод `to_dict()`. Дочерние: `StepValidationError(PipelineError)` — данные не прошли проверку, `StepTransformError(PipelineError)` — ошибка трансформации, `PipelineAbortedError(PipelineError)` — принимает список `collected_errors` (накопленные ошибки из всех шагов).

Доработайте класс `Pipeline` из урока 22: добавьте параметр `fail_fast=True`. При `fail_fast=True` — останавливается на первой ошибке и выбрасывает `StepValidationError` или `StepTransformError`. При `fail_fast=False` — собирает все ошибки и в конце выбрасывает `PipelineAbortedError` со списком всех ошибок.

**Пример использования**:

```python
# fail_fast=True (по умолчанию)
pipeline = Pipeline([
    Validator([lambda d: len(d) > 0]),
    Transformer(lambda d: 1 / 0),   # ZeroDivisionError → StepTransformError
], fail_fast=True)

try:
    pipeline.run([1, 2, 3])
except StepTransformError as e:
    print(e.step_name)   # 'Transformer'
    print(e.to_dict())

# fail_fast=False
pipeline2 = Pipeline([
    Validator([lambda d: isinstance(d, list)]),
    Transformer(lambda d: 1 / 0),
    Validator([lambda d: len(d) > 100]),   # тоже упадёт
], fail_fast=False)

try:
    pipeline2.run([1, 2, 3])
except PipelineAbortedError as e:
    print(f"Собрано ошибок: {len(e.collected_errors)}")
    for err in e.collected_errors:
        print(f"  - {err.step_name}: {err}")
```

---

[Предыдущий урок](lesson23.md) | [Следующий урок](lesson25.md)