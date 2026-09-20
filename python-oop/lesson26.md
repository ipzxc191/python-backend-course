# Урок 26. Коллекция `__slots__`: оптимизация памяти и поведение при наследовании

---

## Как Python хранит атрибуты: проблема `__dict__`

Каждый раз, когда вы создаёте объект в Python, интерпретатор выделяет память не только для самого объекта, но и для хранения его атрибутов. По умолчанию атрибуты хранятся в обычном словаре Python — `__dict__`:

```python
class User:
    def __init__(self, username, email):
        self.username = username
        self.email = email


user = User("alice", "alice@example.com")

print(user.__dict__)
# {'username': 'alice', 'email': 'alice@example.com'}

print(type(user.__dict__))
# <class 'dict'>
```

Словарь — мощная и гибкая структура данных. Именно благодаря `__dict__` вы можете добавлять новые атрибуты к объекту в любой момент:

```python
user.age = 28          # создаём новый атрибут — без ошибок
user.role = "admin"    # ещё один
print(user.__dict__)
# {'username': 'alice', 'email': 'alice@example.com', 'age': 28, 'role': 'admin'}
```

Но у этой гибкости есть цена. Словарь Python — сложная хеш-таблица, которая занимает значительно больше памяти, чем просто набор полей. В типичной 64-битной CPython-реализации пустой словарь занимает около 200 байт, а объект с несколькими атрибутами — ещё больше.

Измерим реальное потребление памяти:

```python
import sys


class UserWithDict:
    def __init__(self, username, email, age):
        self.username = username
        self.email = email
        self.age = age


user = UserWithDict("alice", "alice@example.com", 28)

print(f"Объект:  {sys.getsizeof(user)} байт")
print(f"__dict__: {sys.getsizeof(user.__dict__)} байт")
print(f"Итого:   {sys.getsizeof(user) + sys.getsizeof(user.__dict__)} байт")
# Объект:   48 байт
# __dict__:  232 байт
# Итого:    280 байт
```

Обратите внимание: сам объект весит около 48 байт, но словарь с атрибутами — ещё 232. Большая часть памяти уходит именно на хранение `__dict__`.

Для одного объекта это незначительно. Но представьте веб-приложение, которое обрабатывает миллион HTTP-запросов в день и создаёт объект события для каждого. Или систему кеширования, держащую в памяти 500 000 записей пользователей. В таких сценариях экономия памяти на каждом объекте напрямую влияет на стоимость инфраструктуры.

---

## Синтаксис `__slots__`

`__slots__` — это атрибут класса, который объявляет фиксированный набор атрибутов для объектов этого класса. Вместо создания словаря `__dict__` Python создаёт компактные дескрипторы на уровне класса:

```python
class UserWithSlots:
    __slots__ = ['username', 'email', 'age']

    def __init__(self, username, email, age):
        self.username = username
        self.email = email
        self.age = age


user = UserWithSlots("alice", "alice@example.com", 28)

print(user.username)   # alice — работает как обычно
print(user.email)      # alice@example.com
```

`__slots__` можно объявить как список, кортеж или любой итерируемый объект строк:

```python
# Все варианты эквивалентны:
class A:
    __slots__ = ['x', 'y']

class B:
    __slots__ = ('x', 'y')

class C:
    __slots__ = {'x': 'координата X', 'y': 'координата Y'}  # словарь: значения — документация

class D:
    __slots__ = 'x'   # одна строка для одного слота
```

Вариант со словарём интересен тем, что позволяет добавить документацию к каждому слоту — значения словаря становятся строками документации для соответствующих дескрипторов.

---

## Что меняется с `__slots__`

**Нет `__dict__` по умолчанию.** Попытка обратиться к нему вызывает `AttributeError`:

```python
user = UserWithSlots("alice", "alice@example.com", 28)

try:
    print(user.__dict__)
except AttributeError as e:
    print(e)   # 'UserWithSlots' object has no attribute '__dict__'
```

**Нельзя добавить атрибут, не объявленный в `__slots__`:**

```python
try:
    user.role = "admin"   # AttributeError!
except AttributeError as e:
    print(e)   # 'UserWithSlots' object has no attribute 'role'
```

**Нет `__weakref__` по умолчанию.** Слабые ссылки (ссылка на объект, которая **не увеличивает его счётчик ссылок и не мешает сборщику мусора удалить объект**) на объект с `__slots__` не работают — если не добавить `'__weakref__'` в список слотов явно.

**Дескрипторы вместо атрибутов.** Для каждого слота Python создаёт на уровне класса специальный объект — `member_descriptor`:

```python
print(type(UserWithSlots.username))
# <class 'member_descriptor'>

print(UserWithSlots.username)
# <member 'username' of 'UserWithSlots' objects>
```

Это дескриптор, который при обращении через экземпляр (`user.username`) читает значение из компактного буфера, выделенного прямо в теле объекта — без дополнительных словарей.

---

## Измерение экономии памяти

Посмотрим на реальные числа. Создадим 100 000 объектов двух версий и сравним потребление памяти. 

Для этого используем модуль `tracemalloc`, который будет считать реальное потребление памяти на промежутке от `tracemalloc.start()` до `tracemalloc.stop()`. Метод `get_traced_memory()` возвращает кортеж из двух элементов - `сколько памяти используется сейчас` и `максимум за всё время`.

```python
import sys
import tracemalloc


class UserWithDict:
    def __init__(self, uid, username, email):
        self.uid = uid
        self.username = username
        self.email = email


class UserWithSlots:
    __slots__ = ['uid', 'username', 'email']

    def __init__(self, uid, username, email):
        self.uid = uid
        self.username = username
        self.email = email


def measure_memory(cls, count=100_000):
    tracemalloc.start()
    objects = [cls(i, f"user_{i}", f"user_{i}@example.com") for i in range(count)]
    _, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return peak, objects


peak_dict, _ = measure_memory(UserWithDict)
peak_slots, _ = measure_memory(UserWithSlots)

print(f"Без __slots__: {peak_dict / 1024 / 1024:.1f} МБ")
print(f"С __slots__:   {peak_slots / 1024 / 1024:.1f} МБ")
print(f"Экономия:      {(1 - peak_slots / peak_dict) * 100:.0f}%")

# Типичный вывод:
# Без __slots__: 38.5 МБ
# С __slots__:   13.2 МБ
# Экономия:      66%
```

Экономия около 60–70% памяти — это существенно. Для 100 000 объектов разница составляет 25 МБ. При миллионе объектов — уже 250 МБ.

Посмотрим на размер отдельного объекта:

```python
u_dict  = UserWithDict(1, "alice", "alice@example.com")
u_slots = UserWithSlots(1, "alice", "alice@example.com")

# sys.getsizeof учитывает только «оболочку» объекта, не содержимое строк
print(f"UserWithDict:  {sys.getsizeof(u_dict)} байт (+ {sys.getsizeof(u_dict.__dict__)} для __dict__)")
print(f"UserWithSlots: {sys.getsizeof(u_slots)} байт (нет __dict__)")

# Типичный вывод:
# UserWithDict:  48 байт (+ 232 для __dict__)
# UserWithSlots: 80 байт (нет __dict__)
```

Объект с `__slots__` сам по себе немного больше (80 байт против 48) — слоты хранятся прямо в нём. Но отсутствие `__dict__` (232 байта) даёт существенную общую экономию: 80 против 280.

---

## `__slots__` и наследование: три сценария

Здесь начинается самая важная и часто непонятая часть темы. Поведение `__slots__` при наследовании зависит от того, какие классы в иерархии его объявляют.

### Сценарий 1: Родитель без `__slots__`, дочерний с `__slots__`

```python
class Parent:
    def __init__(self, name):
        self.name = name   # хранится в __dict__


class Child(Parent):
    __slots__ = ['age']   # объявляем слот

    def __init__(self, name, age):
        super().__init__(name)
        self.age = age
```

Что происходит? Поскольку `Parent` не имеет `__slots__`, у объектов `Parent` есть `__dict__`. Дочерний класс `Child` наследует `__dict__` от родителя — и хотя `Child` объявляет `__slots__`, `__dict__` никуда не исчезает:

```python
child = Child("alice", 28)
print(child.__dict__)     # {'name': 'alice'} — __dict__ есть!
print(child.age)          # 28 — из слота
child.extra = "anything"  # можно добавлять произвольные атрибуты через __dict__
print(child.__dict__)     # {'name': 'alice', 'extra': 'anything'}
```

`__slots__` в `Child` добавил слот для `age`, но не устранил `__dict__`. Экономии памяти практически нет.

### Сценарий 2: Родитель с `__slots__`, дочерний без `__slots__`

```python
class Parent:
    __slots__ = ['name']

    def __init__(self, name):
        self.name = name


class Child(Parent):
    # __slots__ не объявлен!

    def __init__(self, name, age):
        super().__init__(name)
        self.age = age
```

Дочерний класс без `__slots__` автоматически получает `__dict__`. Объекты `Child` имеют и слоты родителя, и словарь:

```python
child = Child("alice", 28)
print(child.__dict__)     # {'age': 28} — age хранится в __dict__
print(child.name)         # alice — из слота родителя
child.extra = "anything"  # произвольные атрибуты — через __dict__
```

Слоты родителя работают — `name` хранится компактно. Но `Child` добавил `__dict__` для своих атрибутов.

### Сценарий 3: Оба класса с `__slots__` — правильный подход

```python
class Parent:
    __slots__ = ['name']

    def __init__(self, name):
        self.name = name


class Child(Parent):
    __slots__ = ['age']   # только НОВЫЕ атрибуты — name уже в родителе

    def __init__(self, name, age):
        super().__init__(name)
        self.age = age
```

Теперь всё работает как надо:

```python
child = Child("alice", 28)

print(child.name)    # alice — из слота Parent
print(child.age)     # 28 — из слота Child

try:
    print(child.__dict__)   # AttributeError — нет __dict__!
except AttributeError as e:
    print(e)

try:
    child.extra = "anything"   # AttributeError — нет произвольных атрибутов
except AttributeError as e:
    print(e)

# Объект занимает минимум памяти
print(sys.getsizeof(child))   # ~80 байт без __dict__
```

**Правило**: для полной экономии памяти `__slots__` должны быть объявлены во всех классах иерархии. Один класс без `__slots__` «пробивает» экономию.

---

## Что объявлять в `__slots__` дочернего класса

Дочерний класс объявляет в `__slots__` только свои **новые** атрибуты — те, которых нет у родителя:

```python
class BaseModel:
    __slots__ = ['id', 'created_at']

    def __init__(self, id, created_at=None):
        import datetime
        self.id = id
        self.created_at = created_at or datetime.datetime.now()


class UserModel(BaseModel):
    # Объявляем ТОЛЬКО новые атрибуты — id и created_at уже в родителе
    __slots__ = ['username', 'email', 'is_active']

    def __init__(self, id, username, email):
        super().__init__(id)
        self.username = username
        self.email = email
        self.is_active = True


class AdminModel(UserModel):
    # Объявляем только то, чего нет в UserModel и BaseModel
    __slots__ = ['admin_level', 'permissions']

    def __init__(self, id, username, email, admin_level=1):
        super().__init__(id, username, email)
        self.admin_level = admin_level
        self.permissions = set()
```

Что происходит, если повторить атрибут родителя в `__slots__` дочернего класса?

```python
class Bad(BaseModel):
    __slots__ = ['id', 'username']  # id уже объявлен в BaseModel!

    def __init__(self):
        pass
```

Python не выдаст ошибку, но создаст два дескриптора для `id` — один в `BaseModel`, один в `Bad`. Дескриптор дочернего класса «затеняет» родительский, что приводит к непредсказуемому поведению. Дублирование слотов — ошибка в проектировании, не следует так делать.

---

## `__slots__` и множественное наследование

При множественном наследовании `__slots__` требует особого внимания. Если несколько родителей определяют **непустые** `__slots__`, Python не может корректно организовать память и выбрасывает `TypeError`:

```python
class A:
    __slots__ = ['x']

class B:
    __slots__ = ['y']

try:
    class C(A, B):   # TypeError!
        __slots__ = ['z']
except TypeError as e:
    print(e)
# multiple bases have instance lay-out conflict
```

Это ограничение связано с тем, как CPython размещает слоты в памяти: каждый класс с непустыми слотами определяет «схему» расположения данных в памяти объекта, и два конкурирующих родителя создают конфликт.

**Правильный паттерн для множественного наследования с `__slots__`**: только один из родителей имеет непустые слоты, все остальные (Mixin-классы) объявляют пустой `__slots__ = ()`:

```python
class TimestampMixin:
    __slots__ = ()   # пустой __slots__ — не добавляет атрибуты, не создаёт __dict__

    def get_age(self):
        import datetime
        return (datetime.datetime.now() - self.created_at).total_seconds()


class JSONMixin:
    __slots__ = ()   # пустой — Mixin не должен иметь своих атрибутов

    def to_dict(self):
        return {slot: getattr(self, slot)
                for cls in type(self).__mro__
                for slot in getattr(cls, '__slots__', [])
                if slot not in ('__dict__', '__weakref__')}


class BaseRecord:
    __slots__ = ['id', 'created_at', 'updated_at']

    def __init__(self, id):
        import datetime
        self.id = id
        self.created_at = datetime.datetime.now()
        self.updated_at = self.created_at


class UserRecord(TimestampMixin, JSONMixin, BaseRecord):
    __slots__ = ['username', 'email']

    def __init__(self, id, username, email):
        super().__init__(id)
        self.username = username
        self.email = email


user = UserRecord(1, "alice", "alice@example.com")
print(user.to_dict())
# {'id': 1, 'created_at': ..., 'updated_at': ..., 'username': 'alice', 'email': 'alice@example.com'}

# Нет __dict__ — полная экономия памяти
try:
    user.__dict__
except AttributeError:
    print("Нет __dict__")

# Нет произвольных атрибутов
try:
    user.extra = "test"
except AttributeError as e:
    print(e)
```

**Правило для Mixin-классов**: если Mixin не добавляет новых атрибутов (что правильно для Mixin), объявляйте `__slots__ = ()`. Это говорит Python: «Я знаю о `__slots__`, не создавай `__dict__` для этого класса».

---

## `__slots__` с `__dict__` и `__weakref__`

Если вам нужны слоты для оптимизации, но при этом иногда нужна возможность добавлять произвольные атрибуты — можно явно включить `'__dict__'` в список слотов:

```python
class HybridUser:
    __slots__ = ['username', 'email', '__dict__']   # явно добавляем __dict__

    def __init__(self, username, email):
        self.username = username
        self.email = email


user = HybridUser("alice", "alice@example.com")

# Слоты работают как обычно
print(user.username)   # alice

# __dict__ тоже есть — можно добавлять произвольные атрибуты
user.role = "admin"
print(user.__dict__)   # {'role': 'admin'}
```

Этот гибридный подход даёт частичную экономию: `username` и `email` хранятся в слотах, а произвольные атрибуты — в `__dict__`. Экономия меньше, чем при полных слотах, но гибкость сохраняется.

Для поддержки слабых ссылок (`weakref`) нужно явно добавить `'__weakref__'`:

```python
import weakref


class CachableUser:
    __slots__ = ['username', 'email', '__weakref__']   # поддержка weakref

    def __init__(self, username, email):
        self.username = username
        self.email = email


user = CachableUser("alice", "alice@example.com")
weak = weakref.ref(user)   # работает!
print(weak().username)     # alice
```

Без `'__weakref__'` в `__slots__` создание слабой ссылки вызвало бы `TypeError`. Это важно если объекты используются в кешах с `weakref.WeakValueDictionary`.

---

## `__slots__` и `@property`

`@property` и `__slots__` прекрасно работают вместе. `property` является дескриптором уровня класса — он не конфликтует с `__slots__`:

```python
class Temperature:
    __slots__ = ['_celsius']   # только "сырой" атрибут в слоте

    def __init__(self, celsius):
        self._celsius = celsius

    @property
    def celsius(self):
        return self._celsius

    @celsius.setter
    def celsius(self, value):
        if value < -273.15:
            raise ValueError(f"Температура ниже абсолютного нуля: {value}")
        self._celsius = value

    @property
    def fahrenheit(self):
        return self._celsius * 9 / 5 + 32

    @property
    def kelvin(self):
        return self._celsius + 273.15

    def __repr__(self):
        return f"Temperature({self._celsius}°C)"


t = Temperature(100)
print(t.celsius)      # 100
print(t.fahrenheit)   # 212.0
print(t.kelvin)       # 373.15

t.celsius = 0
print(t)              # Temperature(0°C)

try:
    t.celsius = -300   # ValueError
except ValueError as e:
    print(e)

# Нет __dict__
try:
    t.__dict__
except AttributeError:
    print("Нет __dict__ — объект компактный")
```

Обратите внимание: в `__slots__` объявлен `_celsius` (приватный атрибут), а `celsius`, `fahrenheit`, `kelvin` — это `@property` на уровне класса. Они не участвуют в `__slots__` и не занимают дополнительной памяти в каждом экземпляре.

---

## Практический пример: система обработки событий

Рассмотрим реальный сценарий: система логирования HTTP-запросов. Каждый запрос порождает объект события. При высокой нагрузке — тысячи событий в секунду, которые нужно хранить в памяти для батчевой обработки.

```python
import datetime
import sys
import tracemalloc
from collections import deque


class HTTPEvent:
    """
    Событие HTTP-запроса с оптимизацией памяти через __slots__.
    При 10 000 запросов в секунду и 60-секундном окне —
    600 000 объектов одновременно в памяти.
    """
    __slots__ = [
        'timestamp',
        'method',
        'path',
        'status_code',
        'response_time_ms',
        'user_id',
        'ip_address',
    ]

    def __init__(self, method, path, status_code,
                 response_time_ms, user_id=None, ip_address=None):
        self.timestamp = datetime.datetime.now()
        self.method = method
        self.path = path
        self.status_code = status_code
        self.response_time_ms = response_time_ms
        self.user_id = user_id
        self.ip_address = ip_address

    @property
    def is_error(self):
        return self.status_code >= 400

    @property
    def is_slow(self):
        return self.response_time_ms > 1000

    def to_dict(self):
        return {slot: getattr(self, slot) for slot in self.__slots__}

    def __repr__(self):
        return f"HTTPEvent({self.method} {self.path} {self.status_code})"


class HTTPEventWithDict:
    """Та же структура, но без __slots__ — для сравнения."""

    def __init__(self, method, path, status_code,
                 response_time_ms, user_id=None, ip_address=None):
        self.timestamp = datetime.datetime.now()
        self.method = method
        self.path = path
        self.status_code = status_code
        self.response_time_ms = response_time_ms
        self.user_id = user_id
        self.ip_address = ip_address


class EventQueue:
    """
    Очередь событий с ограниченным размером.
    Хранит последние N событий для анализа.
    """

    def __init__(self, max_size=10_000):
        self._queue = deque(maxlen=max_size)
        self._total_processed = 0

    def push(self, event: HTTPEvent):
        self._queue.append(event)
        self._total_processed += 1

    def get_errors(self):
        return [e for e in self._queue if e.is_error]

    def get_slow_requests(self):
        return [e for e in self._queue if e.is_slow]

    def get_stats(self):
        if not self._queue:
            return {}
        times = [e.response_time_ms for e in self._queue]
        return {
            "count": len(self._queue),
            "avg_response_ms": sum(times) / len(times),
            "max_response_ms": max(times),
            "error_rate": len(self.get_errors()) / len(self._queue) * 100,
        }


# Сравнение потребления памяти
def benchmark(cls, n=50_000):
    tracemalloc.start()
    events = [
        cls(
            method="GET",
            path=f"/api/users/{i}",
            status_code=200 if i % 10 != 0 else 404,
            response_time_ms=100 + (i % 500),
            user_id=i % 1000,
            ip_address="192.168.1.1"
        )
        for i in range(n)
    ]
    _, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    return peak, events


peak_dict, _ = benchmark(HTTPEventWithDict)
peak_slots, _ = benchmark(HTTPEvent)

print(f"50 000 событий без __slots__: {peak_dict / 1024 / 1024:.1f} МБ")
print(f"50 000 событий с __slots__:   {peak_slots / 1024 / 1024:.1f} МБ")
print(f"Экономия: {(peak_dict - peak_slots) / 1024 / 1024:.1f} МБ "
      f"({(1 - peak_slots / peak_dict) * 100:.0f}%)")

# Размер одного объекта
e_dict  = HTTPEventWithDict("GET", "/api", 200, 150)
e_slots = HTTPEvent("GET", "/api", 200, 150)

print(f"\nОдин объект без __slots__: "
      f"{sys.getsizeof(e_dict) + sys.getsizeof(e_dict.__dict__)} байт")
print(f"Один объект с __slots__:   {sys.getsizeof(e_slots)} байт")

# Демонстрация работы очереди
queue = EventQueue(max_size=1000)
import random

for i in range(500):
    queue.push(HTTPEvent(
        method=random.choice(["GET", "POST", "PUT"]),
        path=f"/api/endpoint/{i % 20}",
        status_code=random.choice([200, 200, 200, 404, 500]),
        response_time_ms=random.randint(50, 2000),
    ))

stats = queue.get_stats()
print(f"\nСтатистика очереди:")
print(f"  Запросов: {stats['count']}")
print(f"  Среднее время: {stats['avg_response_ms']:.0f}мс")
print(f"  Ошибок: {stats['error_rate']:.1f}%")
```

---

## Практический пример: модели данных с иерархией `__slots__`

Покажем полноценную иерархию моделей с `__slots__` на нескольких уровнях:

```python
import datetime


class BaseRecord:
    """Базовая запись — всегда имеет id и временные метки."""
    __slots__ = ['id', 'created_at', 'updated_at']

    def __init__(self, id, **kwargs):
        self.id = id
        now = datetime.datetime.now()
        self.created_at = now
        self.updated_at = now

    def touch(self):
        """Обновляет временную метку."""
        self.updated_at = datetime.datetime.now()

    def __repr__(self):
        return f"{self.__class__.__name__}(id={self.id})"


class UserRecord(BaseRecord):
    """Запись пользователя."""
    __slots__ = ['username', 'email', '_is_active']

    def __init__(self, id, username, email, **kwargs):
        super().__init__(id, **kwargs)
        self.username = username
        self.email = email
        self._is_active = True

    @property
    def is_active(self):
        return self._is_active

    @is_active.setter
    def is_active(self, value):
        if not isinstance(value, bool):
            raise TypeError("is_active должен быть bool")
        self._is_active = value
        self.touch()

    def deactivate(self):
        self.is_active = False


class AdminRecord(UserRecord):
    """Запись администратора."""
    __slots__ = ['admin_level', '_permissions']

    def __init__(self, id, username, email, admin_level=1, **kwargs):
        super().__init__(id, username, email, **kwargs)
        self.admin_level = admin_level
        self._permissions = frozenset()

    @property
    def permissions(self):
        return self._permissions

    def grant_permission(self, permission):
        self._permissions = self._permissions | {permission}
        self.touch()


# Демонстрация иерархии
user = UserRecord(1, "alice", "alice@example.com")
admin = AdminRecord(2, "bob", "bob@example.com", admin_level=2)

print(user)         # UserRecord(id=1)
print(admin)        # AdminRecord(id=2)

# Все слоты работают
print(admin.username)      # bob (из UserRecord)
print(admin.id)            # 2 (из BaseRecord)
print(admin.admin_level)   # 2 (из AdminRecord)

admin.grant_permission("delete_users")
print(admin.permissions)   # frozenset({'delete_users'})

# Нет __dict__
for obj in [user, admin]:
    try:
        obj.__dict__
    except AttributeError:
        print(f"{obj}: нет __dict__")

# isinstance работает через всю иерархию
print(isinstance(admin, BaseRecord))   # True
print(isinstance(admin, UserRecord))   # True
print(isinstance(admin, AdminRecord))  # True

# Все слоты в иерархии
def get_all_slots(cls):
    return [slot for c in cls.__mro__
            for slot in getattr(c, '__slots__', [])
            if slot not in ('__dict__', '__weakref__')]

print(get_all_slots(AdminRecord))
# ['admin_level', '_permissions', 'username', 'email', '_is_active',
#  'id', 'created_at', 'updated_at']
```

---

## Pickle и `__slots__`: нюанс сериализации

Стандартный механизм pickle (`pickle.dumps()` / `pickle.loads()`) использует `__dict__` для сохранения состояния объекта. Объекты с `__slots__` и без `__dict__` требуют явной реализации `__getstate__` и `__setstate__`:

```python
import pickle


class SlottedPoint:
    __slots__ = ['x', 'y']

    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __getstate__(self):
        """Возвращает состояние для сериализации."""
        return {slot: getattr(self, slot) for slot in self.__slots__}

    def __setstate__(self, state):
        """Восстанавливает состояние при десериализации."""
        for slot, value in state.items():
            setattr(self, slot, value)

    def __repr__(self):
        return f"SlottedPoint({self.x}, {self.y})"


p = SlottedPoint(1, 2)

# Без __getstate__/__setstate__ — ошибка
# data = pickle.dumps(p)  # TypeError или AttributeError

# С реализованными методами — работает
data = pickle.dumps(p)
p2 = pickle.loads(data)
print(p2)         # SlottedPoint(1, 2)
print(p2.x, p2.y) # 1 2
```

Если у объекта есть и `__slots__`, и `__dict__` (через гибридный подход или через родителя), то pickle может работать без явных методов — но только для содержимого `__dict__`. Слоты в таком случае всё равно нужно обрабатывать вручную.

---

## Когда использовать `__slots__`, а когда нет

**Используйте `__slots__` когда:**

- Создаёте сотни тысяч или миллионы объектов одного класса (события, записи, точки данных).
- Набор атрибутов класса стабилен и не изменяется динамически.
- Объекты служат «структурами данных» — хранят значения без динамического расширения.
- Профилирование показало, что память является узким местом.

**Не используйте `__slots__` когда:**

- Нужно динамически добавлять атрибуты к объектам.
- Используете библиотеки, которые работают через `__dict__` (некоторые ORM, фреймворки).
- Класс участвует в сложном множественном наследовании с несколькими «настоящими» родителями.
- Используете `pickle` без готовности реализовывать `__getstate__`/`__setstate__`.
- Создаёте небольшое количество объектов — экономия будет незначительной, но сложность возрастёт.

Правило: **сначала измерьте, потом оптимизируйте**. Не добавляйте `__slots__` превентивно — только когда профилирование показало реальную проблему с памятью.

---

## Неочевидные моменты и типичные ошибки

**`__slots__` не предотвращает изменение атрибутов.** Он только ограничивает набор допустимых имён. Установленное значение можно изменить свободно:

```python
user = UserRecord(1, "alice", "alice@example.com")
user.username = "bob"   # OK — изменяем существующий слот
print(user.username)    # bob
```

**Пустой `__slots__ = ()` — важный паттерн.** Он говорит Python: «Этот класс участвует в `__slots__`-системе, не создавай для него `__dict__`». Особенно важен для Mixin-классов в иерархиях со слотами.

**Классовые атрибуты не конфликтуют со `__slots__`.** Константы, методы класса, статические методы — всё это остаётся на уровне класса и не занимает место в экземплярах:

```python
class Config:
    __slots__ = ['value']
    DEFAULT_VALUE = 42   # атрибут класса — не слот

    def __init__(self, value=None):
        self.value = value or self.DEFAULT_VALUE


c = Config()
print(c.value)           # 42
print(Config.DEFAULT_VALUE)  # 42
# DEFAULT_VALUE не попадает в __slots__ — это атрибут класса
```

**`__slots__` как строка** (не список) допустим для одного слота, но может вызвать неочевидное поведение:

```python
class A:
    __slots__ = 'x'   # строка — итерируется как ['x', 'x'... нет, просто 'x']

a = A()
a.x = 1   # OK
```

Лучше всегда использовать список или кортеж — это явнее и безопаснее.

---

## Итоги урока

`__slots__` — инструмент оптимизации памяти, который заменяет словарь `__dict__` компактными дескрипторами на уровне класса. Это даёт экономию 50–70% памяти для каждого объекта при наличии фиксированного набора атрибутов.

Ключевые правила: для полной экономии все классы в иерархии должны объявлять `__slots__`. Дочерний класс объявляет только новые атрибуты. При множественном наследовании Mixin-классы должны иметь `__slots__ = ()`. Если хотя бы один класс в иерархии не имеет `__slots__` — `__dict__` появляется у объектов дочернего класса.

`@property` и `__slots__` совместимы — property определяется на уровне класса и не занимает место в экземплярах. Pickle требует явной реализации `__getstate__` и `__setstate__`. `'__dict__'` и `'__weakref__'` можно явно добавить в `__slots__` при необходимости.

Используйте `__slots__` только после профилирования, когда память является реальным узким местом — не превентивно.

---

## Вопросы

1. Где Python по умолчанию хранит атрибуты объекта? Почему это удобно, но затратно по памяти?
2. Что именно создаёт Python для каждого слота в `__slots__`? Чем это отличается от обычного атрибута?
3. Опишите три сценария `__slots__` при наследовании. Какой из них даёт полную экономию памяти?
4. Почему при множественном наследовании несколько родителей с непустыми `__slots__` вызывают `TypeError`? Как правильно организовать Mixin-классы?
5. Что произойдёт, если дочерний класс повторит в `__slots__` атрибут, уже объявленный в родительском классе?
6. Как `__slots__` взаимодействует с `@property`? Нужно ли включать имя свойства в `__slots__`?
7. Почему объекты с `__slots__` по умолчанию не поддерживают pickle? Как это исправить?
8. Почему `__slots__` не следует добавлять превентивно ко всем классам? В каких случаях это оправдано?

---

## Задачи

### **Задача 1**. 

Класс `Point3D` с `__slots__`

Создайте класс `Point3D` для трёхмерной точки с `__slots__ = ['x', 'y', 'z']`. Реализуйте `__init__`, `__repr__` в формате `Point3D(1, 2, 3)`, `__add__` (сложение двух точек), `__sub__`, `__mul__` (умножение на скаляр), `__abs__` (расстояние от начала координат). Добавьте `@property` `magnitude` (то же, что `abs()`). 

Убедитесь, что `__dict__` отсутствует и добавить произвольный атрибут нельзя. 

Сравните размер объекта с аналогичным классом без `__slots__`(например `Point3DNoSlots(1, 2, 3)`).

**Пример использования**:

```python
p1 = Point3D(1, 2, 3)
p2 = Point3D(4, 5, 6)

print(p1 + p2)    # Point3D(5, 7, 9)
print(p2 - p1)    # Point3D(3, 3, 3)
print(p1 * 2)     # Point3D(2, 4, 6)
print(abs(p1))    # 3.7416...
print(p1.magnitude)  # 3.7416...

try:
    p1.color = "red"   # AttributeError
except AttributeError as e:
    print(e)

import sys
print(sys.getsizeof(p1))  # меньше, чем без __slots__
```

---

### **Задача 2**. 

Иерархия `__slots__` для системы уведомлений

Создайте иерархию классов с `__slots__` на каждом уровне. Базовый класс `BaseNotification` со слотами `['id', 'created_at', 'status']` и методами `mark_sent()` (устанавливает `status = "sent"`), `mark_failed(reason)` (`status = "failed"`).
Дочерний `EmailNotification(BaseNotification)` со слотами `['recipient', 'subject', 'body']`. 

Дочерний `SMSNotification(BaseNotification)` со слотами `['phone', 'text']`. 

Дочерний `PushNotification(BaseNotification)` со слотами `['device_token', 'title', 'payload']`.

Убедитесь, что нет `__dict__` ни у одного класса. Напишите функцию `get_all_slots(obj)`, возвращающую все слоты объекта с учётом всей иерархии.

**Пример использования**:

```python
email = EmailNotification(1, "alice@example.com", "Привет", "Тело письма")
sms = SMSNotification(2, "+79001234567", "Код: 1234")

email.mark_sent()
print(email.status)   # sent

print(get_all_slots(email))
# ['recipient', 'subject', 'body', 'id', 'created_at', 'status']

try:
    email.__dict__
except AttributeError:
    print("Нет __dict__")
```

---

### **Задача 3**. 

Сравнение памяти: `__slots__` vs `__dict__`

Создайте два варианта класса `LogEntry` — с `__slots__` и без. Каждый объект хранит: `timestamp` (datetime), `level` (str), `message` (str), `source` (str), `request_id` (str). 

Напишите функцию `benchmark_memory(cls, count)`, которая создаёт `count` объектов и возвращает пиковое потребление памяти в МБ. 

Запустите для 10 000, 100 000 и 500 000 объектов. Выведите сравнительную таблицу с экономией в процентах.

**Пример вывода**:

```
Объектов   Без __slots__   С __slots__    Экономия
10 000         7.8 МБ         2.7 МБ        65%
100 000       77.6 МБ        26.8 МБ        65%
500 000      387.9 МБ       134.1 МБ        65%
```

---

### **Задача 4**. 

Mixin с пустым `__slots__` в иерархии

Создайте систему объектов с `__slots__` и Mixin-классами. Базовый класс `BaseEntity` со слотами `['id', 'name']`. 

Mixin-классы с пустым `__slots__ = ()`:
- `SerializableMixin` — метод `to_dict()` собирает все слоты через MRO.
- `ValidatableMixin` — метод `validate()` проверяет, что `name` не пустое.
- `ComparableMixin` — реализует `__eq__` и `__lt__` по `id`.

Конкретные классы:
- `Product(SerializableMixin, ValidatableMixin, ComparableMixin, BaseEntity)` — добавляет слоты `['price', 'category']`.
- `Category(SerializableMixin, ComparableMixin, BaseEntity)` — добавляет слот `['parent_id']`.

Убедитесь, что у объектов нет `__dict__`. Убедитесь, что `isinstance(product, SerializableMixin)` возвращает `True`.

**Пример использования**:

```python
p1 = Product(1, "Ноутбук", price=89990, category="electronics")
p2 = Product(2, "Мышь", price=1500, category="electronics")

print(p1.to_dict())   # {'id': 1, 'name': 'Ноутбук', 'price': 89990, 'category': 'electronics'}
print(p1.validate())  # True
print(p1 < p2)        # True (id 1 < 2)

products = sorted([p2, p1])
print([p.name for p in products])  # ['Ноутбук', 'Мышь']

print(isinstance(p1, SerializableMixin))   # True
try:
    p1.__dict__
except AttributeError:
    print("Нет __dict__")
```

---

### **Задача 5**. 

`__slots__` с pickle

Создайте класс `CacheEntry` с `__slots__ = ['key', 'value', 'expires_at', 'hit_count']`. Реализуйте `__getstate__` и `__setstate__` для поддержки pickle. 

Добавьте метод `is_expired()` (сравнивает `expires_at` с текущим временем), `increment_hits()` (увеличивает `hit_count`). 

Создайте `PersistentCache` — обычный класс (без `__slots__`), хранящий список `CacheEntry` объектов. 

Реализуйте для него `save(filepath)` через pickle и `@classmethod load(filepath)`. 

Убедитесь, что после сохранения и загрузки все `CacheEntry` сохранили свои данные.

**Пример использования**:

```python
import datetime, pickle

cache = PersistentCache()
cache.add("user:1", {"name": "Alice"}, ttl_seconds=3600)
cache.add("user:2", {"name": "Bob"}, ttl_seconds=60)

entry = cache.get("user:1")
entry.increment_hits()
print(entry.hit_count)    # 1
print(entry.is_expired()) # False

cache.save("cache.pkl")
loaded_cache = PersistentCache.load("cache.pkl")

loaded_entry = loaded_cache.get("user:1")
print(loaded_entry.value)     # {'name': 'Alice'}
print(loaded_entry.hit_count) # 1
```

---

[Предыдущий урок](lesson25.md) | [Следующий урок](lesson27.md)