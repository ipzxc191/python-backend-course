# Урок 16. Протокол последовательности: `__getitem__`, `__setitem__`, `__delitem__`

---

## Что такое протокол последовательности

В Python квадратные скобки — один из самых универсальных синтаксических элементов. Вы используете их для доступа к элементам списка по индексу, для получения значения из словаря по ключу, для получения символа строки, для среза. При этом синтаксис всегда одинаков: `obj[key]`.

```python
lst = [10, 20, 30]
print(lst[1])           # 20 — доступ по индексу

dct = {"name": "Alice"}
print(dct["name"])      # Alice — доступ по ключу

s = "hello"
print(s[0])             # h — доступ к символу

print(lst[1:3])         # [20, 30] — срез
```

За всем этим стоит единый механизм. Когда Python видит `obj[key]`, он вызывает `obj.__getitem__(key)`. Когда видит `obj[key] = value` — вызывает `obj.__setitem__(key, value)`. Когда видит `del obj[key]` — вызывает `obj.__delitem__(key)`.

Реализовав эти три метода в собственном классе, вы получаете объект, который работает со скобочной нотацией точно так же, как встроенные типы. Причём «ключ» в этой нотации может быть чем угодно: целым числом, строкой, объектом `slice`, кортежем — всё зависит от того, как вы реализуете эти методы.

Совокупность `__getitem__`, `__setitem__` и `__delitem__` называется протоколом последовательности (или протоколом контейнера). Это одна из причин, почему Python называют языком с «утиной типизацией»: объект не обязан наследоваться от `list` или `dict`, чтобы вести себя как список или словарь — достаточно реализовать нужные методы.

---

## Метод `__getitem__`: чтение по ключу или индексу

`__getitem__` вызывается при любом обращении к объекту через квадратные скобки для чтения. Метод принимает один аргумент — ключ — и должен вернуть соответствующее значение или выбросить исключение, если ключ не найден.

Начнём с минимального примера, чтобы увидеть чистую механику:

```python
class NumberSquares:
    """
    Объект, который для любого целого числа n возвращает n².
    Данные не хранятся — значение вычисляется при каждом обращении.
    """

    def __getitem__(self, n):
        # n — это то, что передано внутри квадратных скобок
        if not isinstance(n, int):
            raise TypeError(f"Индекс должен быть целым числом, получен {type(n).__name__}")
        if n < 0:
            raise IndexError("Отрицательные индексы не поддерживаются")
        return n ** 2


squares = NumberSquares()

print(squares[5])    # 25
print(squares[10])   # 100
print(squares[0])    # 0

try:
    print(squares["five"])
except TypeError as e:
    print(e)   # Индекс должен быть целым числом, получен str
```

Важно: Python не преобразует ключ автоматически. Если вы написали `squares[-1]`, в `__getitem__` придёт именно `-1` — без каких-либо корректировок. Обработка отрицательных индексов (как в списках, где `-1` означает последний элемент) — полностью ваша ответственность.

Для корректной обработки отрицательных индексов в контейнерах с известным размером используется стандартный приём:

```python
def __getitem__(self, index):
    # Преобразуем отрицательный индекс в положительный
    if index < 0:
        index = len(self._data) + index
    if not 0 <= index < len(self._data):
        raise IndexError(f"Индекс {index} выходит за пределы допустимого диапазона")
    return self._data[index]
```

Исключение при отсутствии ключа: для индексируемых по числу объектов (аналог списка) принято выбрасывать `IndexError`. Для объектов с произвольными ключами (аналог словаря) — `KeyError`. Это не синтаксическое требование Python, но следование этому соглашению делает ваш объект предсказуемым для тех, кто будет его использовать.

---

## Метод `__setitem__`: запись по ключу

`__setitem__` вызывается при операции присваивания через квадратные скобки: `obj[key] = value`. Метод принимает два аргумента: ключ и значение, и ничего не возвращает.

```python
class LoggedDict:
    """
    Словарь, который логирует каждую операцию записи.
    """

    def __init__(self):
        self._data = {}

    def __getitem__(self, key):
        return self._data[key]

    def __setitem__(self, key, value):
        # Логируем операцию перед сохранением
        print(f"[SET] {key!r} = {value!r}")
        self._data[key] = value

    def __repr__(self):
        return f"LoggedDict({self._data!r})"


d = LoggedDict()
d["name"] = "Alice"      # [SET] 'name' = 'Alice'
d["age"] = 28            # [SET] 'age' = 28
d["name"] = "Bob"        # [SET] 'name' = 'Bob' — перезапись

print(d["name"])         # Bob
```

Практически важный случай — валидация типов при записи. В веб-разработке это используется при реализации схем данных: объект принимает данные от клиента и немедленно проверяет их корректность:

```python
class StrictDict:
    """
    Словарь с проверкой типов при записи.
    Схема передаётся при создании: {ключ: ожидаемый_тип}.
    """

    def __init__(self, schema):
        self._schema = schema   # {'name': str, 'age': int, 'is_active': bool}
        self._data = {}

    def __getitem__(self, key):
        if key not in self._data:
            raise KeyError(f"Ключ {key!r} не найден")
        return self._data[key]

    def __setitem__(self, key, value):
        if key not in self._schema:
            raise KeyError(f"Ключ {key!r} не определён в схеме")

        expected_type = self._schema[key]
        if not isinstance(value, expected_type):
            raise TypeError(
                f"Поле {key!r} ожидает {expected_type.__name__}, "
                f"получен {type(value).__name__}"
            )
        self._data[key] = value

    def __repr__(self):
        return f"StrictDict({self._data!r})"


user_schema = StrictDict({
    "username": str,
    "age": int,
    "is_active": bool,
})

user_schema["username"] = "alice"
user_schema["age"] = 28
user_schema["is_active"] = True

print(user_schema["username"])   # alice

try:
    user_schema["age"] = "двадцать восемь"
except TypeError as e:
    print(e)   # Поле 'age' ожидает int, получен str

try:
    user_schema["email"] = "alice@example.com"
except KeyError as e:
    print(e)   # Ключ 'email' не определён в схеме
```

---

## Метод `__delitem__`: удаление по ключу

`__delitem__` вызывается оператором `del` при удалении элемента по ключу: `del obj[key]`. Метод принимает ключ и ничего не возвращает.

```python
class ProtectedDict:
    """
    Словарь, в котором некоторые ключи защищены от удаления.
    """

    def __init__(self, protected_keys=None):
        self._data = {}
        self._protected = set(protected_keys or [])

    def __getitem__(self, key):
        return self._data[key]

    def __setitem__(self, key, value):
        self._data[key] = value

    def __delitem__(self, key):
        if key in self._protected:
            raise KeyError(
                f"Ключ {key!r} защищён от удаления"
            )
        if key not in self._data:
            raise KeyError(f"Ключ {key!r} не найден")
        del self._data[key]

    def __repr__(self):
        return f"ProtectedDict(data={self._data!r}, protected={self._protected!r})"


cfg = ProtectedDict(protected_keys=["SECRET_KEY", "DATABASE_URL"])

cfg["SECRET_KEY"] = "super-secret"
cfg["DEBUG"] = True
cfg["PORT"] = 8080

del cfg["PORT"]         # успешно — PORT не защищён
print(cfg["DEBUG"])     # True

try:
    del cfg["SECRET_KEY"]
except KeyError as e:
    print(e)   # Ключ 'SECRET_KEY' защищён от удаления
```

---

## Срезы и объект `slice`

Один из самых неочевидных аспектов протокола: когда вы пишете `obj[1:3]` или `obj[::2]`, Python не передаёт в `__getitem__` два отдельных числа. Он создаёт специальный объект типа `slice` и передаёт его как единственный аргумент.

```python
class SliceDemo:
    def __getitem__(self, key):
        print(f"Тип ключа: {type(key).__name__}")
        print(f"Значение ключа: {key!r}")


demo = SliceDemo()

demo[5]       # Тип ключа: int,   Значение ключа: 5
demo[1:3]     # Тип ключа: slice, Значение ключа: slice(1, 3, None)
demo[::2]     # Тип ключа: slice, Значение ключа: slice(None, None, 2)
demo[1:10:3]  # Тип ключа: slice, Значение ключа: slice(1, 10, 3)
```

Объект `slice` имеет три атрибута: `start`, `stop` и `step`. Любой из них может быть `None`, если соответствующая часть среза не указана. Для корректной обработки среза используется метод `slice.indices(length)`, который возвращает реальные значения `start`, `stop`, `step` с учётом длины коллекции:

```python
s = slice(1, 10, 2)
# Для коллекции длиной 6:
start, stop, step = s.indices(6)
print(start, stop, step)   # 1 6 2
# Метод автоматически ограничивает stop длиной коллекции
```

**Реализуем класс, который корректно обрабатывает и индексы, и срезы**:

```python
class CustomList:
    """
    Список с поддержкой индексов (включая отрицательные) и срезов.
    """

    def __init__(self, data):
        self._data = list(data)

    def __getitem__(self, key):
        if isinstance(key, int):
            # Обычный индекс — делегируем встроенному списку,
            # который сам обработает отрицательные значения и IndexError
            return self._data[key]

        elif isinstance(key, slice):
            # Срез — возвращаем новый объект того же типа
            return CustomList(self._data[key])

        else:
            raise TypeError(
                f"Индекс должен быть int или slice, получен {type(key).__name__}"
            )

    def __setitem__(self, key, value):
        if isinstance(key, (int, slice)):
            self._data[key] = value
        else:
            raise TypeError(f"Недопустимый тип индекса: {type(key).__name__}")

    def __delitem__(self, key):
        if isinstance(key, (int, slice)):
            del self._data[key]
        else:
            raise TypeError(f"Недопустимый тип индекса: {type(key).__name__}")

    def __len__(self):
        return len(self._data)

    def __repr__(self):
        return f"CustomList({self._data!r})"


lst = CustomList([10, 20, 30, 40, 50])

print(lst[0])        # 10
print(lst[-1])       # 50  — обрабатывается встроенным списком
print(lst[1:3])      # CustomList([20, 30])
print(lst[::2])      # CustomList([10, 30, 50])

lst[1] = 99
print(lst)           # CustomList([10, 99, 30, 40, 50])

del lst[0]
print(lst)           # CustomList([99, 30, 40, 50])
```

Обратите внимание: `lst[1:3]` возвращает не список, а снова `CustomList`. Это важное архитектурное решение: операции над объектом должны возвращать объекты того же типа, сохраняя поведение и методы. Это называется **принципом закрытости под операциями**.

Принципом закрытости под операциями: если операция выполняется над объектом некоторого типа, результат этой операции по возможности тоже должен оставаться объектом этого типа. При таком подходе все предусмотренные классом возможности остаются доступными.

Закрытость особенно полезна, когда у нас есть наследники. Наследование подробно мы будем разбирать в следующем модуле.

Если операция представляет преобразование объекта, которое концептуально сохраняет его абстракцию, результат должен по возможности сохранять тот же тип и его поведение. Закрытость под операциями защищает не только тип, но и контракт класса.

Если пользователь работает с `CustomList`, он должен иметь возможность рассчитывать, что типичные операции над ним не заставят его внезапно переходить на работу с обычным `list`.

---

## Связь с протоколом итерации: `__getitem__` как fallback

Это один из самых неочевидных моментов темы. Если у объекта определён `__getitem__`, но не определён `__iter__`, Python способен итерироваться по нему автоматически. Интерпретатор начинает вызывать `__getitem__(0)`, `__getitem__(1)`, `__getitem__(2)` и так далее, пока не получит `IndexError` — это сигнал об окончании последовательности.

```python
class LegacySequence:
    """
    Класс без __iter__, но с __getitem__.
    Python всё равно сможет итерироваться по нему.
    """

    def __init__(self, data):
        self._data = data

    def __getitem__(self, index):
        return self._data[index]   # IndexError при выходе за границы


seq = LegacySequence([10, 20, 30])

# for работает через __getitem__, даже без __iter__
for item in seq:
    print(item)
# 10
# 20
# 30

# list() тоже работает
print(list(seq))   # [10, 20, 30]

# И оператор in (через последовательный перебор)
print(20 in seq)   # True
```

Этот механизм существует по историческим причинам: в ранних версиях Python `__iter__` не было, и итерация реализовывалась только через `__getitem__`. Сегодня этот fallback сохраняется для совместимости.

Почему важно знать об этом? По двум причинам. 

Первая — ваш класс может стать итерируемым непреднамеренно, если вы реализовали `__getitem__` для другой цели. 

Вторая — если ваш объект логически является последовательностью, явная реализация `__iter__` предпочтительнее: она даёт контроль над поведением, эффективнее при вложенных циклах и делает намерение явным.

---

## Оператор `in` и метод `__contains__`

Когда Python встречает `3 in container` он пытается определить, является ли `3` элементом `container`. Для пользовательского класса Python в первую очередь ищет специальный метод `__contains__`.

Если метода `__contains__` нет, Python всё равно может выполнить `in`. Для последовательностей существует **резервный механизм**: Python пытается получать элементы через `__getitem__`, начиная с индекса `0`.

Если метод `__getitem__` так же не переопределен, и нет метода `__contains__`, Python может использовать итерацию для последовательного поиска. За это отвечает метод `__iter__`.

Оператор `in` (`3 in container`) работающий через последовательный перебор с помощью `__getitem__` или `__iter__` обходит весь контейнер. В худшем случае, что бы найти элемент, оператору `in` понадобится пройти по всем элементам объекта. Это означает линейную сложность O(n), где n - это количество элементов в объекте.

Если вы хотите уменьшить сложность поиска, то вы можете явно продублировать основные данные в структуру, в которой сложность поиска изначально равна O(1), например словарь или множество. И переопределить метод `__contains__` в котором явно написать логику поиска элемента сразу в свойстве, которое хранит данные в словаре или множестве.

```python
class FastContainer:
    def __init__(self, items):
        self._data = list(items)
        self._lookup = set(items)   # для O(1) проверки

    def __getitem__(self, index):
        return self._data[index]

    def __contains__(self, item):
        # O(1) вместо O(n) — используем множество для поиска
        return item in self._lookup

    def __len__(self):
        return len(self._data)


container = FastContainer([1, 2, 3, 4, 5])

print(3 in container)    # True  — использует __contains__
print(99 in container)   # False — O(1), не перебирает все элементы
```

Если `__contains__` не определён, но есть `__iter__` — Python использует его для последовательного поиска. Если нет ни `__contains__`, ни `__iter__`, но есть `__getitem__` — Python использует fallback через `__getitem__`.

У такого подхода есть свои подводные камни, например появляется **стоимость поддержки двух структур**. Оптимизация через дополнительную структуру данных требует поддерживать её актуальность.

---

## Практический пример: класс `Headers`

В HTTP каждый запрос и ответ несут набор заголовков — пар «имя: значение». Имена заголовков по стандарту нечувствительны к регистру: `Content-Type`, `content-type` и `CONTENT-TYPE` — это один и тот же заголовок.

В Django и DRF объект запроса предоставляет заголовки именно с таким поведением. Реализуем его:

```python
class Headers:
    """
    Коллекция HTTP-заголовков с нечувствительным к регистру доступом.
    Имена заголовков нормализуются к нижнему регистру при хранении.

    Пример:
        h["Content-Type"] и h["content-type"] — одно и то же.
    """

    def __init__(self, initial=None):
        # Храним заголовки в нижнем регистре
        self._headers = {}
        if initial:
            for key, value in initial.items():
                self[key] = value   # используем __setitem__ для нормализации

    def _normalize(self, key):
        """Приводим имя заголовка к нижнему регистру."""
        if not isinstance(key, str):
            raise TypeError(f"Имя заголовка должно быть строкой, получен {type(key).__name__}")
        return key.lower()

    def __getitem__(self, key):
        normalized = self._normalize(key)
        if normalized not in self._headers:
            raise KeyError(f"Заголовок {key!r} не найден")
        return self._headers[normalized]

    def __setitem__(self, key, value):
        normalized = self._normalize(key)
        self._headers[normalized] = str(value)   # значение всегда строка

    def __delitem__(self, key):
        normalized = self._normalize(key)
        if normalized not in self._headers:
            raise KeyError(f"Заголовок {key!r} не найден")
        del self._headers[normalized]

    def __contains__(self, key):
        return self._normalize(key) in self._headers

    def __iter__(self):
        return iter(self._headers)

    def __len__(self):
        return len(self._headers)

    def get(self, key, default=None):
        """Безопасное получение заголовка без исключения."""
        try:
            return self[key]
        except KeyError:
            return default

    def items(self):
        return self._headers.items()

    def __repr__(self):
        return f"Headers({self._headers!r})"
```

Проверим работу:

```python
headers = Headers({
    "Content-Type": "application/json",
    "Authorization": "Bearer abc123",
    "X-Request-Id": "req-001",
})

# Нечувствительность к регистру при чтении
print(headers["Content-Type"])    # application/json
print(headers["content-type"])    # application/json
print(headers["CONTENT-TYPE"])    # application/json

# Проверка наличия тоже нечувствительна к регистру
print("authorization" in headers)    # True
print("Authorization" in headers)    # True
print("X-Api-Key" in headers)        # False

# Запись
headers["Cache-Control"] = "no-cache"
print(headers["cache-control"])       # no-cache

# Безопасное получение
print(headers.get("Accept", "application/json"))   # application/json (default)

# Удаление
del headers["X-Request-Id"]
print("x-request-id" in headers)   # False

# Итерация (по нормализованным именам)
for name in headers:
    print(f"{name}: {headers[name]}")
# content-type: application/json
# authorization: Bearer abc123
# cache-control: no-cache

print(repr(headers))
```

Этот класс демонстрирует важный архитектурный принцип: нормализация данных при записи (а не при чтении) гарантирует, что внутри объекта всегда хранится единообразное представление, независимо от того, как данные были переданы снаружи.

---

## Практический пример: класс `Config`

Конфигурация веб-приложения — ещё один классический случай для протокола контейнера. Реализуем объект конфигурации, который поддерживает доступ по ключу, вложенные пути через точку и защиту обязательных ключей:

```python
class Config:
    """
    Конфигурация веб-приложения с доступом по ключу.

    Особенности:
    - Поддержка вложенных путей: config["database.host"]
    - Защита обязательных ключей от удаления
    - Типизированные значения через схему

    Пример:
        config["database.host"] эквивалентно config["database"]["host"]
    """

    def __init__(self, data=None, required_keys=None):
        self._data = data or {}
        self._required = set(required_keys or [])

    def __getitem__(self, key):
        # Поддерживаем вложенные пути через точку
        if "." in key:
            parts = key.split(".", 1)   # разбиваем только по первой точке
            top, rest = parts[0], parts[1]
            if top not in self._data:
                raise KeyError(f"Ключ {top!r} не найден в конфигурации")
            nested = self._data[top]
            if isinstance(nested, dict):
                # Рекурсивно создаём Config для вложенного словаря
                return Config(nested)[rest]
            raise KeyError(f"Значение по ключу {top!r} не является вложенной конфигурацией")

        if key not in self._data:
            raise KeyError(f"Параметр {key!r} не найден в конфигурации")
        return self._data[key]

    def __setitem__(self, key, value):
        # Вложенные пути при записи — тоже поддерживаем
        if "." in key:
            parts = key.split(".", 1)
            top, rest = parts[0], parts[1]
            if top not in self._data:
                self._data[top] = {}
            if isinstance(self._data[top], dict):
                Config(self._data[top])[rest] = value
                return
        self._data[key] = value

    def __delitem__(self, key):
        if key in self._required:
            raise KeyError(
                f"Параметр {key!r} является обязательным и не может быть удалён"
            )
        if key not in self._data:
            raise KeyError(f"Параметр {key!r} не найден")
        del self._data[key]

    def __contains__(self, key):
        if "." in key:
            try:
                self[key]
                return True
            except KeyError:
                return False
        return key in self._data

    def get(self, key, default=None):
        try:
            return self[key]
        except KeyError:
            return default

    def __repr__(self):
        return f"Config({self._data!r})"


# Создаём конфигурацию приложения
config = Config(
    data={
        "DEBUG": False,
        "SECRET_KEY": "a-very-long-secret-key",
        "database": {
            "host": "localhost",
            "port": 5432,
            "name": "myapp_db",
        },
        "redis": {
            "host": "localhost",
            "port": 6379,
        }
    },
    required_keys=["SECRET_KEY", "database"]
)

# Доступ к простым ключам
print(config["DEBUG"])        # False
print(config["SECRET_KEY"])   # a-very-long-secret-key

# Доступ через вложенные пути
print(config["database.host"])   # localhost
print(config["database.port"])   # 5432
print(config["redis.host"])      # localhost

# Проверка наличия
print("DEBUG" in config)             # True
print("database.host" in config)     # True
print("database.password" in config) # False

# Запись вложенного значения
config["database.password"] = "secret"
print(config["database.password"])   # secret

# Безопасное получение с дефолтом
print(config.get("EMAIL_HOST", "smtp.gmail.com"))   # smtp.gmail.com

# Удаление обычного ключа
config["DEBUG"] = True
del config["DEBUG"]
print("DEBUG" in config)   # False

# Попытка удалить обязательный ключ
try:
    del config["SECRET_KEY"]
except KeyError as e:
    print(e)   # Параметр 'SECRET_KEY' является обязательным и не может быть удалён
```

---

## Полная реализация: класс `TypedList`

Объединим все три метода в одном классе, демонстрирующем их совместную работу. `TypedList` — это список с проверкой типов при добавлении элементов, поддержкой срезов и защитой от некорректных операций:

```python
class TypedList:
    """
    Список с ограничением типов хранимых элементов.
    Поддерживает индексный доступ (включая отрицательные индексы),
    срезы, запись и удаление.
    """

    def __init__(self, item_type, initial=None):
        self._type = item_type   # тип элементов, который разрешён
        self._data = []

        if initial:
            for item in initial:
                self._validate(item)
                self._data.append(item)

    def _validate(self, value):
        """Проверяем, что значение соответствует разрешённому типу."""
        if not isinstance(value, self._type):
            raise TypeError(
                f"TypedList[{self._type.__name__}] не принимает "
                f"{type(value).__name__}: {value!r}"
            )

    def __getitem__(self, key):
        if isinstance(key, (int, slice)):
            result = self._data[key]
            # При срезе возвращаем TypedList, а не список
            if isinstance(key, slice):
                return TypedList(self._type, result)
            return result
        raise TypeError(f"Недопустимый тип индекса: {type(key).__name__}")

    def __setitem__(self, key, value):
        if isinstance(key, slice):
            # При срезовой записи проверяем каждый элемент
            for item in value:
                self._validate(item)
            self._data[key] = value
        elif isinstance(key, int):
            self._validate(value)
            self._data[key] = value
        else:
            raise TypeError(f"Недопустимый тип индекса: {type(key).__name__}")

    def __delitem__(self, key):
        if isinstance(key, (int, slice)):
            del self._data[key]
        else:
            raise TypeError(f"Недопустимый тип индекса: {type(key).__name__}")

    def append(self, value):
        self._validate(value)
        self._data.append(value)

    def __contains__(self, item):
        return item in self._data

    def __iter__(self):
        return iter(self._data)

    def __len__(self):
        return len(self._data)

    def __repr__(self):
        return f"TypedList[{self._type.__name__}]({self._data!r})"
```

Демонстрация работы:

```python
# Список только из строк
string_list = TypedList(str, ["alice", "bob", "carol"])

print(string_list[0])         # alice
print(string_list[-1])        # carol
print(string_list[0:2])       # TypedList[str](['alice', 'bob'])

string_list[1] = "dave"
print(string_list)            # TypedList[str](['alice', 'dave', 'carol'])

# Типизированная проверка при записи
try:
    string_list[0] = 42
except TypeError as e:
    print(e)   # TypedList[str] не принимает int: 42

# Добавление корректного и некорректного элемента
string_list.append("eve")
try:
    string_list.append(3.14)
except TypeError as e:
    print(e)   # TypedList[str] не принимает float: 3.14

# Удаление
del string_list[0]
print(string_list)   # TypedList[str](['dave', 'carol', 'eve'])

# Срез возвращает TypedList, а не список
subset = string_list[0:2]
print(type(subset).__name__)   # TypedList
print(subset)                  # TypedList[str](['dave', 'carol'])

# Оператор in
print("carol" in string_list)   # True
print("alice" in string_list)   # False
```

---

## Неочевидные моменты и типичные ошибки

**Отрицательные индексы не обрабатываются автоматически.** Если вы реализуете `__getitem__` самостоятельно (без делегирования встроенному списку), вы обязаны обработать отрицательные индексы сами. Python не преобразует `-1` в `len - 1` за вас:

```python
class BadSequence:
    def __init__(self):
        self._data = [10, 20, 30]

    def __getitem__(self, index):
        # Не обрабатываем отрицательные индексы
        if index >= len(self._data):
            raise IndexError("out of range")
        return self._data[index]   # -1 попадёт прямо сюда и вернёт _data[-1] случайно

seq = BadSequence()
print(seq[-1])   # 30 — работает случайно, потому что list сам понимает -1
# Но если хранить данные иначе — поведение будет непредсказуемым
```

Правильный подход при ручной реализации:

```python
def __getitem__(self, index):
    if isinstance(index, int):
        if index < 0:
            index += len(self._data)   # преобразуем отрицательный
        if not 0 <= index < len(self._data):
            raise IndexError(f"index {index} out of range")
    return self._data[index]
```

**`KeyError` vs `IndexError` — не перепутайте.** Соглашение Python: контейнеры с целочисленными индексами (как список) при выходе за границы выбрасывают `IndexError`. Контейнеры с произвольными ключами (как словарь) при отсутствующем ключе выбрасывают `KeyError`. Нарушение этого соглашения сломает совместимость с кодом, который ожидает стандартного поведения:

```python
# Если ваш объект похож на список — используйте IndexError
def __getitem__(self, index):
    if index >= len(self._data):
        raise IndexError(f"index {index} out of range")   # правильно

# Если ваш объект похож на словарь — используйте KeyError
def __getitem__(self, key):
    if key not in self._data:
        raise KeyError(key)   # правильно
```

**Возврат `None` вместо исключения нарушает контракт.** Иногда возникает соблазн вернуть `None` при отсутствующем ключе — это «мягче» и не прерывает работу программы. Но это скрывает ошибки: код, ожидающий реальное значение, получит `None` и сломается позже, в совершенно другом месте. Если хотите «мягкое» поведение — добавьте отдельный метод `get(key, default=None)`, как это сделано в `dict`.

**`__setitem__` не вызывается при инициализации объекта.** Если в `__init__` вы пишете `self._data = {}` — это обычное присваивание атрибута, не вызов `__setitem__`. `__setitem__` вызывается только при `obj[key] = value` после создания объекта.

---

## Итоги урока

Три метода — `__getitem__`, `__setitem__`, `__delitem__` — реализуют протокол контейнера: они стоят за синтаксисом квадратных скобок в Python. Реализовав их, объект получает возможность поддерживать индексный доступ, запись и удаление элементов — аналогично встроенным спискам и словарям.

Ключ в `__getitem__` может быть не только целым числом, но и строкой, объектом `slice` или любым другим объектом. При срезах `obj[1:3]` Python автоматически создаёт объект `slice` и передаёт его методу. Для правильной обработки используется `slice.indices(length)`.

Если `__iter__` не определён, но определён `__getitem__` — Python автоматически использует его для итерации, вызывая `__getitem__(0)`, `__getitem__(1)` и т.д. до `IndexError`. Это legacy-механизм; явная реализация `__iter__` предпочтительнее.

Оператор `in` работает через `__contains__` (если определён), иначе — через `__iter__` или `__getitem__`. Для контейнеров с быстрым поиском (на основе словаря или множества) явная реализация `__contains__` критически важна для производительности.

---

## Вопросы

1. Какие методы вызывает Python при выполнении операций `obj[key]`, `obj[key] = value` и `del obj[key]`? Есть ли у этих методов возвращаемое значение?
2. Что именно Python передаёт в `__getitem__` при выполнении `obj[1:5:2]`? Как получить конкретные значения из этого аргумента?
3. Какое исключение принято выбрасывать в `__getitem__` при отсутствии ключа: `KeyError` или `IndexError`? От чего это зависит?
4. Что произойдёт, если в классе определён `__getitem__`, но не определён `__iter__`? Будет ли работать цикл `for item in obj:`?
5. Как работает оператор `in` для объекта, у которого определены `__getitem__` и `__contains__`? Какой метод будет использован?
6. Обрабатывает ли Python отрицательные индексы автоматически перед передачей в `__getitem__`? Что произойдёт, если не обработать `-1` в собственной реализации?
7. В примере с классом `Headers` имена заголовков нормализуются при записи, а не при чтении. Почему это лучше, чем нормализация при чтении?
8. Почему возврат `None` вместо исключения при отсутствующем ключе в `__getitem__` считается плохой практикой? Какой правильный способ реализовать «мягкое» поведение?

---

## Задачи

### **Задача 1**. 

Класс `SquaresMap`

Создайте класс `SquaresMap`, который реализует только `__getitem__` и ведёт себя как отображение: `obj[n]` возвращает `n²` для любого целого числа `n`.

Отрицательные числа разрешены. 

Для нецелых ключей выбрасывайте `TypeError` с сообщением `"Ключ должен быть целым числом"`. 

Реализуйте `__contains__`, который возвращает `True` для любого целого числа (поскольку квадрат определён для любого целого). 

Реализуйте `__repr__` в формате `SquaresMap()`.

**Пример использования**:

```python
sm = SquaresMap()

print(sm[5])     # 25
print(sm[-3])    # 9
print(sm[0])     # 0
print(sm[10])    # 100

print(5 in sm)   # True
print(0 in sm)   # True

try:
    sm["five"]
except TypeError as e:
    print(e)   # Ключ должен быть целым числом

# Использование в цикле через явные вызовы
for i in range(5):
    print(f"{i}² = {sm[i]}")
```

---

### **Задача 2**. 

Класс `RingBuffer`

Создайте класс `RingBuffer` — кольцевой буфер фиксированного размера. Принимает один аргумент: `capacity` (ёмкость буфера, целое число). 

Реализуйте метод `push(value)`, который добавляет элемент: если буфер заполнен, самый старый элемент перезаписывается. 

Реализуйте `__getitem__`, поддерживающий индексный доступ (включая отрицательные индексы) и срезы. 

При выходе за границы выбрасывайте `IndexError`. 

Реализуйте `__len__`, возвращающий текущее количество элементов (не ёмкость).

Реализуйте `__repr__` в формате `RingBuffer(capacity=5, data=[...])`.

**Пример использования**:

```python
buf = RingBuffer(capacity=4)

buf.push(10)
buf.push(20)
buf.push(30)
buf.push(40)

print(len(buf))    # 4
print(buf[0])      # 10
print(buf[-1])     # 40
print(buf[1:3])    # [20, 30]

# Буфер заполнен — следующий push перезаписывает старейший элемент
buf.push(50)

print(len(buf))    # 4 — ёмкость не изменилась
print(buf[0])      # 20  — 10 перезаписан
print(buf[-1])     # 50  — новый элемент
print(list(buf[::1]))  # [20, 30, 40, 50]
```

---

### **Задача 3**. 

Класс `HttpHeaders`

Создайте класс `HttpHeaders` — улучшенный вариант класса `Headers` из лекции. Принимает необязательный словарь `initial` при создании.

При инициализации должны создаваться следующие атрибуты объекта класса:
* `self._data = {}` - должен будет хранить пары: `ключ в нижнем регистре` → `значение`
* `self._original = {}` - должен будет хранить пары: `ключ в нижнем регистре` → `оригинальное имя`
* Если передан словарь `initial`, то в объекте должны создаваться атрибуты из пары переданного словаря.

Реализуйте:
* `__setitem__` записать в `_data`: `ключ в нижнем регистре` → `значение`; в `_original`: `ключ в нижнем регистре` → `оригинальное имя`. Запись производить с проверкой: имя заголовка должно быть непустой строкой, значение — строкой или числом (при записи числа конвертируются в строки). 
* `__getitem__` возвращать значения из `_data` по ключу с нечувствительностью к регистру.
* `__delitem__` удаляет пары из `_data` и `_original`. Должен выбрасывать `KeyError` при попытке удалить несуществующий заголовок. 

Реализуйте:
* `__contains__` (нечувствительно к регистру), 
* `__iter__` (возвращает итератор с оригинальными именами в порядке добавления),
* `__len__`. 

Добавьте метод `get(key, default=None)`. 

Реализуйте `__repr__` в формате `HttpHeaders({'content-type': 'application/json', ...})`.

**Пример использования**:

```python
h = HttpHeaders({
    "Content-Type": "application/json",
    "X-Request-Id": "abc-123",
})

print(h["content-type"])       # application/json
print(h["CONTENT-TYPE"])       # application/json
print(h["X-Request-Id"])       # abc-123

h["Cache-Control"] = "no-cache"
h["X-Retry-Count"] = 3         # число конвертируется в строку
print(h["x-retry-count"])      # 3

print("content-type" in h)     # True
print("Authorization" in h)    # False

print(len(h))                  # 4

for name in h:
    print(f"{name}: {h[name]}")

del h["X-Request-Id"]
print("x-request-id" in h)    # False

print(h.get("Authorization", "Bearer default"))   # Bearer default
```

---

### **Задача 4**. 

Класс `QueryParams`

Создайте класс `QueryParams` — объект для работы с параметрами URL-запроса вида `?page=1&limit=20&sort=name`.

Объект можно создать двумя способами:

```python
params = QueryParams("page=1&limit=20&sort=name")  # из строки
params = QueryParams({"page": "1", "limit": "20"}) # из словаря
params = QueryParams()                              # пустой объект
```

При инициализации должен создаваться следующий атрибут:

* `self._data = {}` — хранит пары `ключ (str) → значение (str)`. Все значения хранятся как строки.

Если передана строка — разбейте её по символу `&`, затем каждую часть по символу `=` (только по первому вхождению). Пары `ключ=значение` сохраните в `_data`. Если передан словарь — скопируйте его в `_data`, приведя все значения к строке.

Реализуйте:

* `__getitem__` — возвращает значение из `_data` по ключу. Если ключ не найден — выбрасывает `KeyError`.
* `__setitem__` — записывает пару в `_data`. Ключ и значение приводятся к строке (`str(key)`, `str(value)`).
* `__delitem__` — удаляет пару из `_data`. Если ключ не найден — выбрасывает `KeyError`.
* `__contains__` — возвращает `True`, если ключ присутствует в `_data`.
* `__len__` — возвращает количество параметров в `_data`.

Добавьте методы:

* `get(key, default=None)` — возвращает значение по ключу или `default`, если ключ не найден (без выброса исключения).
* `to_string()` — возвращает строку в формате URL-параметров. Параметры должны быть отсортированы по имени ключа и разделены символом `&`. Формат: `"filter=active&limit=20&page=2"`.

Реализуйте `__repr__` в формате `QueryParams(page=2, limit=20, sort=name)` — перечисление пар `ключ=значение` через запятую.

**Пример использования**:

```python
params = QueryParams("page=1&limit=20&sort=name&order=asc")

print(params["page"])    # 1
print(params["limit"])   # 20
print(params["sort"])    # name

print("page" in params)   # True
print("token" in params)  # False
print(len(params))        # 4

params["page"] = 2
params["filter"] = "active"
del params["order"]

print(params.to_string())
# filter=active&limit=20&page=2&sort=name

print(params.get("offset", "0"))   # 0

print(repr(params))
# QueryParams(filter=active, limit=20, page=2, sort=name)
```

---

### **Задача 5**. 

Класс `Matrix`

Создайте класс `Matrix` — двумерная матрица целых чисел.

Объект создаётся следующей командой:

```python
m = Matrix(rows, cols)
```

где `rows` — количество строк, `cols` — количество столбцов (оба аргумента — положительные целые числа).

При инициализации должен создаваться следующий атрибут:

* `self._data` — список из `rows` строк, каждая строка — список из `cols` нулей.

Доступ к элементам матрицы осуществляется через кортеж `(row, col)`:

```python
m[0, 0] = 5    # запись
print(m[1, 2]) # чтение
```

Реализуйте:

* `__getitem__` — принимает ключ-кортеж `(row, col)`. Извлекает и возвращает элемент `self._data[row][col]`. Перед обращением к данным вызывайте вспомогательный метод проверки индексов.
* `__setitem__` — принимает ключ-кортеж `(row, col)` и значение. Записывает значение в `self._data[row][col]`. Перед записью вызывайте вспомогательный метод проверки индексов.
* `__delitem__` — удаление элемента матрицы не имеет смысла. При любом вызове `del m[row, col]` выбрасывать `TypeError("Удаление элементов матрицы не поддерживается")`.

Реализуйте вспомогательный метод `_check_index(self, row, col)`, который проверяет, что `row` находится в диапазоне `[0, rows)`, а `col` — в диапазоне `[0, cols)`. Если любой из индексов некорректен — выбрасывать `IndexError` с сообщением, указывающим какой именно индекс вышел за пределы и каков допустимый размер.

Добавьте методы:

* `row(n)` — возвращает список элементов строки с номером `n` (копию, не ссылку).
* `col(n)` — возвращает список элементов столбца с номером `n`.

Реализуйте `__repr__` в следующем многострочном формате:

```
Matrix 3x3:
[1, 0, 3]
[0, 5, 0]
[0, 0, 9]
```

**Пример использования**:

```python
m = Matrix(3, 3)

m[0, 0] = 1
m[1, 1] = 5
m[2, 2] = 9
m[0, 2] = 3

print(m[1, 1])    # 5
print(m[0, 2])    # 3

print(m.row(0))   # [1, 0, 3]
print(m.col(2))   # [3, 0, 9]

try:
    m[5, 0] = 10
except IndexError as e:
    print(e)   # Индекс строки 5 выходит за пределы (размер: 3)

try:
    del m[0, 0]
except TypeError as e:
    print(e)   # Удаление элементов матрицы не поддерживается

print(m)
# Matrix 3x3:
# [1, 0, 3]
# [0, 5, 0]
# [0, 0, 9]
```

---

### **Задача 6**. 

Класс `JSONPath`

Создайте класс `JSONPath` — обёртка над вложенным словарём Python, которая позволяет читать, записывать и удалять глубоко вложенные значения через путь с точкой.

Объект создаётся следующей командой:

```python
data = JSONPath({"user": {"name": "Alice", "address": {"city": "Moscow"}}})
data = JSONPath()   # пустой объект
```

При инициализации должен создаваться следующий атрибут:

* `self._data` — обычный словарь Python, переданный в аргументе (или пустой словарь, если аргумент не передан).

Путь к вложенному значению записывается через точку: `"user.address.city"` означает `self._data["user"]["address"]["city"]`. Если точки в строке нет — это обычный ключ верхнего уровня: `"api_version"` означает `self._data["api_version"]`.

Реализуйте три вспомогательных метода для работы с `self._data` по списку ключей:

* `_get_nested(data, keys)` — рекурсивно обходит словарь по списку ключей и возвращает значение. Если любой ключ из пути не найден — выбрасывает `KeyError`.
* `_set_nested(data, keys, value)` — рекурсивно устанавливает значение. Если промежуточного словаря не существует — создаёт его автоматически.
* `_del_nested(data, keys)` — рекурсивно удаляет ключ. Если любой ключ из пути не найден — выбрасывает `KeyError`.

Реализуйте магические методы, которые разбивают путь на список ключей и вызывают соответствующий вспомогательный метод:

* `__getitem__(path)` — разбивает `path` по точке, вызывает `_get_nested`. При `KeyError` внутри — перевыбрасывает `KeyError(path)` с полным путём.
* `__setitem__(path, value)` — разбивает `path` по точке, вызывает `_set_nested`.
* `__delitem__(path)` — разбивает `path` по точке, вызывает `_del_nested`. При `KeyError` внутри — перевыбрасывает `KeyError(path)` с полным путём.
* `__contains__(path)` — пытается вызвать `self[path]`; если успешно — возвращает `True`, если `KeyError` — возвращает `False`.

Реализуйте `__repr__` в формате `JSONPath({...})`, где `{...}` — стандартное строковое представление `self._data`.

**Пример использования**:

```python
data = JSONPath({
    "user": {
        "name": "Alice",
        "address": {
            "city": "Moscow",
            "zip": "101000"
        }
    },
    "api_version": "2.0"
})

print(data["api_version"])           # 2.0
print(data["user.name"])             # Alice
print(data["user.address.city"])     # Moscow

data["user.age"] = 28
print(data["user.age"])              # 28

# Если промежуточного ключа нет — он создаётся автоматически
data["server.host"] = "localhost"
data["server.port"] = 8080
print(data["server.host"])           # localhost

print("user.name" in data)           # True
print("user.phone" in data)          # False

del data["user.address.zip"]
print("user.address.zip" in data)    # False

try:
    _ = data["user.address.country"]
except KeyError as e:
    print(e)   # 'user.address.country'
```

---

[Предыдущий урок](lesson15.md) | [Следующий урок](lesson17.md)