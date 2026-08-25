# Урок 15. Контекстные менеджеры: методы `__enter__` и `__exit__`

---

## Проблема, которую решают контекстные менеджеры

Представьте типичную задачу: открыть файл, записать в него данные, закрыть файл. Код выглядит просто:

```python
f = open("log.txt", "w")
f.write("Запрос обработан успешно")
f.close()
```

Но у этого кода есть скрытая проблема. Если строка `f.write(...)` выбросит исключение — из-за нехватки места на диске, из-за ошибки кодировки или по любой другой причине — строка `f.close()` никогда не выполнится. Файл останется открытым. 

В масштабах веб-сервера, который обрабатывает сотни запросов одновременно, накопление незакрытых файловых дескрипторов — это реальная проблема, приводящая к исчерпанию системных ресурсов.

Правильное решение без контекстных менеджеров — конструкция `try/finally`. `finally` - блок выполняется всегда, независимо от того, возникло исключение или нет:

```python
f = open("log.txt", "w")
try:
    f.write("Запрос обработан успешно")
finally:
    f.close()   # выполнится даже если f.write() выбросит исключение
```

Это работает корректно. Но представьте, что таких ресурсов несколько: соединение с базой данных, файл лога, сетевое соединение. Код превращается в пирамиду вложенных `try/finally`, которую трудно читать и легко написать с ошибкой.

**Контекстный менеджер** — это способ зафиксировать паттерн «подготовить ресурс — сделать работу — гарантированно освободить ресурс» в одном объекте и использовать его через лаконичную конструкцию `with`:

```python
with open("log.txt", "w") as f:
    f.write("Запрос обработан успешно")
# f.close() вызывается автоматически — всегда, даже при исключении
```

Здесь `open()` возвращает файловый объект, который является контекстным менеджером. Python вызывает его метод `__exit__` при выходе из блока `with` — и именно там происходит закрытие файла. Теперь разберём, как это устроено изнутри.

---

## Как работает `with` под капотом

Конструкция `with obj as x:` — это синтаксический сахар. Python разворачивает её в строго определённую последовательность вызовов:

```python
with obj as x:
    # тело блока
    do_something(x)
```

Эквивалентно следующему коду:

```python
x = obj.__enter__()       # шаг 1: вход в контекст, результат присваивается x

try:
    do_something(x)       # шаг 2: выполнение тела блока
except:
    # шаг 3а: если возникло исключение — __exit__ вызывается с информацией о нём
    if not obj.__exit__(*sys.exc_info()):
        raise             # если __exit__ не подавил исключение — пробрасываем дальше
else:
    # шаг 3б: если исключений не было — __exit__ вызывается с тремя None
    obj.__exit__(None, None, None)
```

Три ключевых момента, которые необходимо зафиксировать.

**Первый**: `__enter__` возвращает объект, который присваивается переменной после `as`. Это не обязательно тот же объект `obj` — `__enter__` может вернуть что угодно: `self`, другой объект, простое значение или даже `None`.

**Второй**: часть `as x` необязательна. Если вам нужен только эффект входа и выхода, а возвращаемый `__enter__` объект не нужен — пишите просто `with obj:`.

**Третий**: `__exit__` вызывается всегда — и при нормальном завершении блока, и при любом исключении. Именно это свойство делает контекстные менеджеры надёжным инструментом управления ресурсами.

---

## Сигнатура `__exit__` и её аргументы

Сигнатура `__exit__` выглядит так:

```python
def __exit__(self, exc_type, exc_val, exc_tb):
    ...
```

Три аргумента помимо `self` — это информация об исключении, если оно возникло. Разберём каждый подробно.

`exc_type` — это класс исключения. Например, `ValueError`, `FileNotFoundError`, `ZeroDivisionError`. Если исключения не было — `None`.

`exc_val` — это сам объект исключения. Через него можно получить сообщение об ошибке (`str(exc_val)`) и любые атрибуты, которые определены в классе исключения. Если исключения не было — `None`.

`exc_tb` — это объект трейсбека (traceback). Он содержит информацию о том, в каком файле и на какой строке возникло исключение. Используется при логировании ошибок. Если исключения не было — `None`.

```python
class DebugContext:
    def __enter__(self):
        print("Входим в блок")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is None:
            # Исключения не было — все три аргумента равны None
            print("Блок завершён успешно")
        else:
            # Исключение есть — выводим информацию о нём
            print(f"Тип исключения: {exc_type.__name__}")
            print(f"Сообщение: {exc_val}")
            print(f"Трейсбек: {exc_tb}")
        # Возвращаем None (False) — исключение не подавляется
```

Проверим:

```python
with DebugContext():
    print("Выполняем работу")
# Входим в блок
# Выполняем работу
# Блок завершён успешно

with DebugContext():
    raise ValueError("Что-то пошло не так")
# Входим в блок
# Тип исключения: ValueError
# Сообщение: Что-то пошло не так
# Трейсбек: <traceback object at 0x...>
# Traceback (most recent call last): ...   ← исключение распространяется дальше
```

---

## Что возвращает `__exit__`: механизм подавления исключений

Возвращаемое значение `__exit__` определяет, что произойдёт с исключением после того, как метод завершится.

Если `__exit__` возвращает истинное значение (`True` или любой другой truthy-объект) — Python считает, что исключение обработано, и подавляет его. Выполнение продолжается с первой строки после блока `with`.

Если `__exit__` возвращает ложное значение (`False`, `None`, `0`) — исключение продолжает распространяться дальше. Именно поэтому `__exit__` без явного `return` ведёт себя как без подавления: функция без `return` возвращает `None`, что является ложным значением.

```python
class SuppressValueError:
    """Контекстный менеджер, который подавляет ValueError, но пропускает остальные."""

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is ValueError:
            print(f"Перехвачен ValueError: {exc_val} — продолжаем работу")
            return True    # подавляем исключение
        return False       # все остальные исключения распространяются дальше


with SuppressValueError():
    raise ValueError("Некорректные данные")
print("Эта строка выполнится — ValueError подавлен")
# Перехвачен ValueError: Некорректные данные — продолжаем работу
# Эта строка выполнится — ValueError подавлен

with SuppressValueError():
    raise TypeError("Неверный тип")
# TypeError НЕ подавлен — исключение распространяется дальше
# Traceback (most recent call last): ...
```

Подавление исключений — инструмент, которым нужно пользоваться осторожно. Подавляйте только те типы исключений, которые вы ожидаете и умеете обработать. Глобальное подавление всех исключений (`return True` без проверки типа) — это антипаттерн: он скрывает реальные ошибки и делает отладку невозможной.

---

## Простейший пример: класс `Timer`

Реализуем первый собственный контекстный менеджер — измерение времени выполнения блока кода. Это классический учебный пример, потому что он чистый: никакой сложной логики, только механика `__enter__` и `__exit__`.

```python
import time


class Timer:
    """
    Контекстный менеджер для измерения времени выполнения блока кода.
    Фиксирует момент входа в блок и вычисляет прошедшее время при выходе.
    """

    def __init__(self, label=""):
        self.label = label        # необязательная метка для вывода
        self.elapsed = 0.0        # время выполнения в секундах — доступно после блока

    def __enter__(self):
        self._start = time.perf_counter()   # фиксируем время входа
        return self   # возвращаем себя, чтобы через as можно было получить elapsed

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.elapsed = time.perf_counter() - self._start   # вычисляем прошедшее время

        prefix = f"[{self.label}] " if self.label else ""
        print(f"{prefix}Время выполнения: {self.elapsed:.4f} сек.")

        # Не подавляем исключения — возвращаем None (False)
        # Нам важно замерить время даже при ошибке, но саму ошибку не скрывать

    def __repr__(self):
        return f"Timer(label={self.label!r}, elapsed={self.elapsed:.4f})"
```

Использование:

```python
# Без as — просто замеряем время
with Timer("Загрузка данных"):
    time.sleep(0.1)   # имитируем задержку
# [Загрузка данных] Время выполнения: 0.1003 сек.

# С as — получаем доступ к результату после блока
with Timer("Обработка запроса") as t:
    total = sum(range(1_000_000))

print(f"Сумма: {total}, заняло: {t.elapsed:.4f} сек.")
# [Обработка запроса] Время выполнения: 0.0312 сек.
# Сумма: 499999500000, заняло: 0.0312 сек.

# Timer работает даже при исключении — фиксирует время до момента ошибки
try:
    with Timer("Падающий запрос") as t:
        time.sleep(0.05)
        raise RuntimeError("Ошибка в обработчике")
except RuntimeError:
    print(f"Запрос упал через {t.elapsed:.4f} сек.")
# [Падающий запрос] Время выполнения: 0.0501 сек.
# Запрос упал через 0.0501 сек.
```

Обратите внимание: `__enter__` возвращает `self`, и именно поэтому `t` в конструкции `with Timer(...) as t` — это сам объект `Timer`. После блока мы можем обратиться к `t.elapsed` и получить измеренное время. Если бы `__enter__` возвращал `None`, конструкция `as t` всё равно работала бы, но `t` был бы `None` и доступ к `t.elapsed` вызвал бы исключение.

---

## Управление ресурсами: класс `DatabaseConnection`

Главная роль контекстных менеджеров в веб-разработке — управление ресурсами, которые необходимо освобождать. 

Соединение с базой данных — самый частый пример. Оно должно быть открыто перед выполнением запросов и закрыто после — вне зависимости от того, завершились ли запросы успешно.

```python
class DatabaseConnection:
    """
    Контекстный менеджер для управления соединением с базой данных.
    Гарантирует закрытие соединения при любом исходе: успех или исключение.
    """

    def __init__(self, host, port, database):
        self.host = host
        self.port = port
        self.database = database
        self._connection = None
        self._is_open = False

    def _connect(self):
        """Имитация установки соединения с БД."""
        print(f"Подключение к {self.host}:{self.port}/{self.database}...")
        self._is_open = True
        print("Соединение установлено.")

    def _disconnect(self):
        """Имитация закрытия соединения."""
        self._is_open = False
        print("Соединение закрыто.")

    def execute(self, query):
        """Имитация выполнения SQL-запроса."""
        if not self._is_open:
            raise RuntimeError("Соединение не установлено")
        print(f"Выполняем запрос: {query}")
        return [{"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob"}]

    def __enter__(self):
        # При входе в блок with — открываем соединение
        self._connect()
        return self   # возвращаем себя, чтобы можно было делать conn.execute(...)

    def __exit__(self, exc_type, exc_val, exc_tb):
        # При выходе из блока with — закрываем соединение ВСЕГДА
        self._disconnect()

        if exc_type is not None:
            # Если было исключение — логируем его, но не подавляем
            print(f"Соединение закрыто после ошибки: {exc_type.__name__}: {exc_val}")

        # Возвращаем None (False) — исключения не подавляем

    def __bool__(self):
        return self._is_open

    def __repr__(self):
        status = "open" if self._is_open else "closed"
        return f"DatabaseConnection({self.host}/{self.database}, {status})"
```

Проверим поведение в обоих сценариях:

```python
# Сценарий 1: успешное выполнение
with DatabaseConnection("localhost", 5432, "myapp_db") as conn:
    users = conn.execute("SELECT * FROM users WHERE is_active = TRUE")
    print(f"Получено {len(users)} записей")

# Подключение к localhost:5432/myapp_db...
# Соединение установлено.
# Выполняем запрос: SELECT * FROM users WHERE is_active = TRUE
# Получено 2 записей
# Соединение закрыто.

print()

# Сценарий 2: исключение внутри блока
try:
    with DatabaseConnection("localhost", 5432, "myapp_db") as conn:
        conn.execute("SELECT * FROM users")
        raise ValueError("Ошибка валидации данных")
except ValueError as e:
    print(f"Поймали во внешнем коде: {e}")

# Подключение к localhost:5432/myapp_db...
# Соединение установлено.
# Выполняем запрос: SELECT * FROM users
# Соединение закрыто.
# Соединение закрыто после ошибки: ValueError: Ошибка валидации данных
# Поймали во внешнем коде: Ошибка валидации данных
```

Обратите внимание на второй сценарий: `_disconnect()` вызвался до того, как `ValueError` достиг внешнего `try/except`. Соединение закрыто гарантированно — именно это и нужно.

---

## Обработка исключений внутри `__exit__`: транзакции

Более сложный и реалистичный сценарий — управление транзакцией базы данных. Транзакция — это группа операций, которая должна выполниться либо целиком, либо не выполниться вовсе. Если все операции прошли успешно — делаем `COMMIT`. Если хоть одна операция упала — делаем `ROLLBACK`.

Именно так работает `django.db.transaction.atomic()`. Реализуем упрощённый аналог:

```python
class ManagedTransaction:
    """
    Контекстный менеджер для управления транзакцией базы данных.

    Поведение:
    - Успешное завершение блока → COMMIT
    - Любое исключение → ROLLBACK, исключение не подавляется
    - Ожидаемые ошибки валидации (ValueError) → ROLLBACK + подавление исключения
    """

    def __init__(self, connection, suppress_validation_errors=False):
        self.connection = connection
        self.suppress_validation_errors = suppress_validation_errors
        self._transaction_id = id(self)

    def __enter__(self):
        print(f"[TXN {self._transaction_id}] BEGIN TRANSACTION")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is None:
            print(f"[TXN {self._transaction_id}] COMMIT")
            return False

        print(f"[TXN {self._transaction_id}] ROLLBACK — причина: {exc_type.__name__}: {exc_val}")

        if self.suppress_validation_errors and exc_type is ValueError:
            print(f"[TXN {self._transaction_id}] ValueError подавлен, продолжаем")
            return True

        return False

    def execute(self, query):
        print(f"[TXN {self._transaction_id}] Выполняем: {query}")


# Сценарий 1: успешная транзакция
with ManagedTransaction(connection=None) as txn:
    txn.execute("INSERT INTO orders (user_id, total) VALUES (1, 1500)")
    txn.execute("UPDATE users SET order_count = order_count + 1 WHERE id = 1")

print()

# Сценарий 2: ошибка валидации — ROLLBACK + подавление
with ManagedTransaction(connection=None, suppress_validation_errors=True) as txn:
    txn.execute("INSERT INTO orders (user_id, total) VALUES (2, 0)")
    raise ValueError("Сумма заказа должна быть больше нуля")

print("Выполнение продолжается — ValueError был подавлен")

print()

# Сценарий 3: неожиданная ошибка — ROLLBACK без подавления
try:
    with ManagedTransaction(connection=None) as txn:
        txn.execute("DELETE FROM users WHERE id = 99")
        raise RuntimeError("Потеряно соединение с базой данных")
except RuntimeError as e:
    print(f"Критическая ошибка поймана снаружи: {e}")
```

---

## Вложенные контекстные менеджеры

Python позволяет вкладывать несколько контекстных менеджеров. Существуют два синтаксических варианта:

```python
# Вариант 1: последовательный — более явный
with DatabaseConnection("localhost", 5432, "myapp") as conn:
    with Timer("Запрос") as t:
        result = conn.execute("SELECT * FROM users")

# Вариант 2: компактный — все менеджеры в одной строке
with DatabaseConnection("localhost", 5432, "myapp") as conn, Timer("Запрос") as t:
    result = conn.execute("SELECT * FROM users")
```

Оба варианта полностью эквивалентны. Порядок вызовов при вложении строго определён: `__enter__` вызываются слева направо, `__exit__` — в обратном порядке (LIFO).

```python
class Marker:
    def __init__(self, name):
        self.name = name

    def __enter__(self):
        print(f"[{self.name}] __enter__")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        print(f"[{self.name}] __exit__")


with Marker("A") as a, Marker("B") as b, Marker("C") as c:
    print("Тело блока")

# [A] __enter__
# [B] __enter__
# [C] __enter__
# Тело блока
# [C] __exit__
# [B] __exit__
# [A] __exit__
```

Если при вложении один из `__enter__` выбросит исключение, будут вызваны `__exit__` только тех менеджеров, чьи `__enter__` уже успешно завершились.

---

## `contextlib.contextmanager`: генераторный способ

Стандартная библиотека Python предоставляет декоратор `@contextmanager` из модуля `contextlib`. Он позволяет создать контекстный менеджер из обычной генераторной функции без написания класса.

Механика проста: код до `yield` выполняется как `__enter__`, значение, переданное в `yield`, становится результатом `as`, код после `yield` выполняется как `__exit__`.

```python
from contextlib import contextmanager
import time


@contextmanager
def timer(label=""):
    start = time.perf_counter()
    try:
        yield   # здесь выполняется тело блока with
    finally:
        elapsed = time.perf_counter() - start
        prefix = f"[{label}] " if label else ""
        print(f"{prefix}Время выполнения: {elapsed:.4f} сек.")


with timer("Загрузка"):
    time.sleep(0.05)
# [Загрузка] Время выполнения: 0.0502 сек.
```

`try/finally` вокруг `yield` обязателен: без него код после `yield` не выполнится при исключении в теле блока.

Для обработки исключений используется `except` вокруг `yield`:

```python
@contextmanager
def managed_transaction():
    print("BEGIN TRANSACTION")
    try:
        yield
        print("COMMIT")
    except ValueError as e:
        print(f"ROLLBACK — ошибка валидации: {e}")
        # Не перевыбрасываем — исключение подавлено
    except Exception as e:
        print(f"ROLLBACK — критическая ошибка: {e}")
        raise   # перевыбрасываем — исключение не подавлено
```

**Когда использовать класс, а когда `@contextmanager`:** декоратор предпочтителен для простых случаев с минимальной логикой. Класс предпочтителен, когда менеджер хранит состояние между входом и выходом, предоставляет дополнительные методы или сложно конфигурируется.

---

## Практический пример: `TempDirectory`

В веб-разработке часто нужны временные директории — при обработке загруженных файлов или в тестах. Контекстный менеджер идеально подходит: создаём при входе, удаляем при выходе гарантированно.

```python
import os
import shutil
import tempfile


class TempDirectory:
    """
    Контекстный менеджер для работы с временной директорией.

    При входе в with:
    1. Создаёт временную директорию в текущей папке проекта.
    2. Возвращает объект TempDirectory.

    При выходе из with:
    1. Удаляет директорию со всем её содержимым.
    2. Если внутри блока произошла ошибка — всё равно удаляет директорию.
    """

    def __init__(self, prefix="tmp_", cleanup=True):
        self.prefix = prefix
        self.cleanup = cleanup
        self.path = None

    def __enter__(self):
        # Получаем путь к текущей папке проекта
        project_dir = os.getcwd()

        # Создаём временную директорию внутри проекта
        self.path = tempfile.mkdtemp(
            prefix=self.prefix,
            dir=project_dir
        )

        print(f"Создана временная директория: {self.path}")

        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        # Удаляем временную директорию со всем содержимым
        if self.cleanup and self.path and os.path.exists(self.path):
            shutil.rmtree(self.path)

            print(f"Временная директория удалена: {self.path}")

        # Если внутри with произошла ошибка
        if exc_type is not None:
            print(
                f"Директория удалена после ошибки: "
                f"{exc_type.__name__}: {exc_val}"
            )

    def create_file(self, filename, content=""):
        filepath = os.path.join(self.path, filename)

        with open(filepath, "w", encoding="utf-8") as f:
            f.write(content)

        return filepath

    def __repr__(self):
        return (
            f"TempDirectory("
            f"path={self.path!r}, "
            f"cleanup={self.cleanup}"
            f")"
        )


with TempDirectory(prefix="upload_") as tmp:
    avatar_path = tmp.create_file(
        "avatar.jpg",
        content="<binary data>"
    )

    report_path = tmp.create_file(
        "report.csv",
        content="id,name\n1,Alice"
    )

    print(f"Файлов в директории: {len(os.listdir(tmp.path))}")

    # Здесь можно открыть папку проекта
    # и увидеть временную директорию upload_xxxxxxxx

print(f"Директория существует: {os.path.exists(tmp.path)}")
```

---

## Неочевидные моменты и типичные ошибки

**`__enter__` не обязан возвращать `self`.** Он может вернуть любой объект или `None`. Например, менеджер соединения может вернуть курсор, а не само соединение:

```python
class ConnectionManager:
    def __enter__(self):
        self._conn = self._open_connection()
        return self._conn.cursor()   # возвращаем не self, а курсор

    def __exit__(self, exc_type, exc_val, exc_tb):
        self._conn.close()
```

**Если `__enter__` выбросил исключение — `__exit__` не вызывается.** Гарантия вызова `__exit__` действует только при успешном завершении `__enter__`. Если соединение не удалось установить — нечего закрывать:

```python
class FailingContext:
    def __enter__(self):
        raise RuntimeError("Не удалось инициализироваться")
        # __exit__ при этом НЕ вызывается

    def __exit__(self, exc_type, exc_val, exc_tb):
        print("Этот код не выполнится при ошибке в __enter__")
```

**`return` внутри блока `with` не обходит `__exit__`.** Python вызовет `__exit__` перед тем, как функция вернёт управление:

```python
def process_data():
    with Timer("Обработка"):
        result = sum(range(1000))
        return result   # __exit__ Timer'а будет вызван ПЕРЕД возвратом
```

**`__exit__` без явного `return` не подавляет исключения.** Это следствие того, что Python-функция без `return` возвращает `None`, что является ложным значением. Подавление требует явного `return True`.

---

## Итоги урока

Контекстный менеджер — это объект с методами `__enter__` и `__exit__`, который используется в конструкции `with`. `__enter__` выполняется при входе в блок и возвращает объект, доступный через `as`. `__exit__` выполняется при выходе из блока всегда — независимо от того, возникло исключение или нет.

`__exit__` принимает три аргумента: тип исключения, объект исключения и трейсбек. Если исключений не было — все три равны `None`. Если `__exit__` возвращает истинное значение — исключение подавляется. Если возвращает ложное или `None` — исключение распространяется дальше.

При вложении нескольких менеджеров `__enter__` вызывается слева направо, `__exit__` — в обратном порядке. Если `__enter__` одного из менеджеров выбросил исключение — `__exit__` вызывается только у тех, чьи `__enter__` уже завершились успешно.

`@contextmanager` из `contextlib` позволяет создать контекстный менеджер из генераторной функции: код до `yield` — это `__enter__`, код после — `__exit__`. Для корректной обработки исключений `yield` необходимо обернуть в `try/finally`.

В следующем уроке мы рассмотрим методы `__getitem__`, `__setitem__` и `__delitem__` — протокол, который позволяет объекту работать со скобочной нотацией `obj[key]`, как словарь или список.

---

## Вопросы

1. Разверните конструкцию `with obj as x: do_something(x)` в явные вызовы методов. Что происходит при исключении внутри блока?
2. Что означают три аргумента `exc_type`, `exc_val`, `exc_tb` в методе `__exit__`? Какими они будут, если исключения не возникло?
3. Как `__exit__` подавляет исключение? Что произойдёт, если метод не содержит явного `return`?
4. Обязан ли `__enter__` возвращать `self`? Что станет значением переменной после `as`, если `__enter__` вернёт `None`?
5. Что произойдёт, если исключение возникнет в самом методе `__enter__`? Будет ли вызван `__exit__`?
6. В каком порядке вызываются `__enter__` и `__exit__` при вложении: `with A() as a, B() as b, C() as c:`?
7. Как работает `@contextmanager`? Как реализуются логика `__enter__` и `__exit__` в генераторной функции?
8. Гарантирует ли `return` внутри блока `with`, что `__exit__` не будет вызван?

## Задачи

### **Задача 1**. 

Класс `ManagedFile`

Создайте контекстный менеджер `ManagedFile`, который принимает два аргумента: `filepath` (путь к файлу, строка) и `mode` (режим открытия, строка, по умолчанию `'r'`). 

При входе в блок `with` класс должен открывать файл и возвращать файловый объект через `as`. 

При выходе — закрывать файл. Если при работе с файлом возникло исключение — вывести сообщение `"Файл закрыт после ошибки: <тип>: <сообщение>"` и не подавлять исключение. 

Реализуйте `__repr__` в формате `ManagedFile(filepath='log.txt', mode='w')`.

**Пример использования**:

```python
with ManagedFile("test.txt", "w") as f:
    f.write("Строка 1\n")
    f.write("Строка 2\n")

with ManagedFile("test.txt", "r") as f:
    content = f.read()
    print(content)

try:
    with ManagedFile("test.txt", "r") as f:
        data = f.read()
        raise ValueError("Некорректные данные в файле")
except ValueError:
    print("Ошибка поймана снаружи")
# Файл закрыт после ошибки: ValueError: Некорректные данные в файле
# Ошибка поймана снаружи
```

---

### **Задача 2**. 

Класс `SuppressErrors`

Создайте контекстный менеджер `SuppressErrors`, который принимает произвольное количество типов исключений через `*exceptions`. 

При выходе из блока он должен подавлять исключения только переданных типов и выводить `"Подавлено исключение <тип>: <сообщение>"`. 

Исключения других типов распространяются. 

Реализуйте `__repr__` в формате `SuppressErrors(ValueError, KeyError)`.

**Пример использования:**

```python
suppress = SuppressErrors(ValueError, KeyError)

with suppress:
    raise ValueError("Некорректное значение")
print("Продолжаем после ValueError")
# Подавлено исключение ValueError: Некорректное значение

with suppress:
    raise KeyError("user_id")
print("Продолжаем после KeyError")
# Подавлено исключение KeyError: 'user_id'

try:
    with suppress:
        raise TypeError("Неверный тип")
except TypeError:
    print("TypeError не подавлен")

print(suppress)   # SuppressErrors(ValueError, KeyError)
```

---

### **Задача 3**. 

Класс `LoggedOperation`

Создайте контекстный менеджер `LoggedOperation`, который принимает `operation_name` (строка). 

При входе выводит `"[START] <название>"` и фиксирует время. 

При успешном выходе фиксирует затраченное время и выводит `"[OK] <название> — <время> сек."`. 

При исключении — `"[FAIL] <название> — <время> сек. | <тип>: <сообщение>"`, не подавляя его. 

Реализуйте `__repr__` в формате `LoggedOperation(name='Обработка запроса')`.

**Пример использования**:

```python
with LoggedOperation("Загрузка пользователей"):
    users = list(range(1000))
# [START] Загрузка пользователей
# [OK] Загрузка пользователей — 0.0001 сек.

try:
    with LoggedOperation("Обработка платежа"):
        result = 100 / 0
except ZeroDivisionError:
    pass
# [START] Обработка платежа
# [FAIL] Обработка платежа — 0.0000 сек. | ZeroDivisionError: division by zero
```

---

### **Задача 4**. 

Класс `AtomicWriter`

Создайте контекстный менеджер `AtomicWriter`, реализующий атомарную запись в целевой файл. Принимает `filepath` - путь до создаваемого целевого файла. 

При входе создаёт временный файл `filepath + ".tmp"` и возвращает его через `as`. 

При успешном завершении переименовывает временный файл в целевой. Для переименования можно использовать `os.replace`. 

При исключении удаляет временный файл, не трогая целевой. Удалить временный файл можно с помощью метода `os.remove`.

Принцип работы: либо файл перезаписан полностью, либо остался без изменений. 

Реализуйте `__repr__` в формате `AtomicWriter(filepath='config.json')`.

**Пример использования**:

```python
import os

with AtomicWriter("output.txt") as f:
    f.write("Строка 1\n")
    f.write("Строка 2\n")

with open("output.txt") as f:
    print(f.read())   # Строка 1 / Строка 2

try:
    with AtomicWriter("output.txt") as f:
        f.write("Новые данные\n")
        raise IOError("Диск заполнен")
except IOError:
    pass

with open("output.txt") as f:
    print(f.read())   # Строка 1 / Строка 2 — не изменился

os.remove("output.txt")
```

---

### **Задача 5**. 

Класс `MultiContext`

Создайте контекстный менеджер `MultiContext`, принимающий произвольное количество менеджеров (`*managers`). 

При входе вызывает `__enter__` у каждого в порядке передачи, результаты собирает в список и возвращает через `as`. 

При выходе вызывает `__exit__` в обратном порядке. 

Если один из `__exit__` выбросит исключение — остальные всё равно должны быть вызваны. 

Реализуйте `__repr__` в формате `MultiContext(3 managers)`.

**Пример использования**:

```python
with MultiContext(
    LoggedOperation("Операция A"),
    Timer("Общее время"),
    LoggedOperation("Операция B")
) as (op_a, t, op_b):
    print("Тело блока")

# [START] Операция A
# [START] Операция B
# Тело блока
# [OK] Операция B — ...
# [Общее время] Время выполнения: ...
# [OK] Операция A — ...
```

---

[Предыдущий урок](lesson14.md) | [Следующий урок](lesson16.md)