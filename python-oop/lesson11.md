# Урок 11. Управление доступом к атрибутам: `__getattribute__`, `__getattr__`, `__setattr__`, `__delattr__`

---

## Как Python работает с атрибутами объектов

До сих пор в курсе атрибуты объектов воспринимались как нечто само собой разумеющееся: мы писали `self.x = 10` в `__init__`, затем читали `obj.x` и получали значение. Это работало, и мы не задавались вопросом, что именно происходит внутри интерпретатора в момент каждой такой операции.

Этот урок — про то, что скрыто за простым синтаксисом точки. Когда Python видит выражение `obj.x`, он не просто «идёт и берёт значение». Он выполняет последовательность конкретных шагов, и каждый из этих шагов реализован через специальный метод. Именно эти методы — `__getattribute__`, `__getattr__`, `__setattr__` и `__delattr__` — мы разберём в этом уроке.

Понимание этих методов важно по двум причинам. Во-первых, они объясняют поведение Python, которое иначе выглядит магическим. Во-вторых, они открывают возможность встраивать собственную логику в операции чтения, записи и удаления атрибутов — это основа валидации данных, прокси-объектов, ORM-систем (в том числе Django ORM) и многих других паттернов.

---

## Что происходит, когда вы обращаетесь к атрибуту

Прежде чем перейти к магическим методам, необходимо понять базовую модель хранения атрибутов в Python.

Каждый объект в Python хранит свои атрибуты в специальном словаре `__dict__`. Это обычный словарь Python, и вы можете посмотреть его содержимое напрямую:

```python
class User:
    role = "user"  # атрибут класса, общий для всех экземпляров

    def __init__(self, name, email):
        self.name = name    # атрибут экземпляра
        self.email = email  # атрибут экземпляра


u = User("Alice", "alice@example.com")
print(u.__dict__)
# {'name': 'Alice', 'email': 'alice@example.com'}

print(User.__dict__.keys())
# dict_keys(['__module__', 'role', '__init__', '__dict__', '__weakref__', '__doc__'])
```

Обратите внимание: атрибут `role` есть в `__dict__` класса, но не в `__dict__` экземпляра. Когда вы пишете `u.role`, Python сначала смотрит в `__dict__` экземпляра `u`, не находит там `role`, и затем поднимается на уровень выше — в `__dict__` класса `User`. Именно поэтому `u.role` возвращает `"user"`, хотя этот атрибут принадлежит классу, а не объекту.

Здесь кроется распространённая ловушка при работе с атрибутами класса:

```python
class Config:
    DEBUG = False  # атрибут класса — общий для всех экземпляров

    def __init__(self):
        pass

    def enable_debug(self):
        self.DEBUG = True  # ВНИМАНИЕ: это создаёт атрибут ЭКЗЕМПЛЯРА, не меняет класс


c1 = Config()
c2 = Config()

c1.enable_debug()

print(c1.DEBUG)        # True  — атрибут экземпляра c1
print(c2.DEBUG)        # False — атрибут класса, c2 своего не имеет
print(Config.DEBUG)    # False — класс не изменился
```

Любое присваивание через `self` создаёт или изменяет атрибут именно этого экземпляра — никогда класса. Если нужно изменить атрибут класса, используют `cls` в методе `@classmethod` или обращаются напрямую через имя класса: `Config.DEBUG = True`.

Теперь, когда модель хранения атрибутов ясна, разберём, что именно вызывает Python при каждой операции с атрибутами.

---

## Полная схема вызовов

При любой операции с атрибутами Python вызывает соответствующий магический метод. Схема выглядит так:

```python
obj.x           →  obj.__getattribute__('x')
                   если не найден → obj.__getattr__('x')

obj.x = 10      →  obj.__setattr__('x', 10)

del obj.x       →  obj.__delattr__('x')
```

По умолчанию эти методы реализованы в базовом классе `object`, от которого неявно наследуется каждый класс в Python. Запись `class MyClass:` полностью эквивалентна записи `class MyClass(object):`. Это означает, что стандартное поведение — поиск атрибута в `__dict__` экземпляра, затем в классе — уже реализовано внутри Python и работает без каких-либо усилий с вашей стороны.

Когда вы определяете один из этих методов в своём классе, вы полностью перехватываете соответствующую операцию и получаете возможность реализовать любую логику перед тем, как (и если) передать управление стандартной реализации.

---

## Метод `__getattribute__`: перехват любого чтения атрибута

`__getattribute__` вызывается абсолютно при каждом обращении к любому атрибуту объекта — неважно, существует атрибут или нет. Это делает его самым мощным и одновременно самым опасным из четырёх методов.

Рассмотрим простейший пример, демонстрирующий, как часто он вызывается:

```python
class Request:
    def __init__(self, method, path):
        self.method = method
        self.path = path

    def __getattribute__(self, name):
        # Этот код выполняется при ЛЮБОМ обращении к атрибуту,
        # включая self.method и self.path внутри других методов
        print(f"[__getattribute__] обращение к '{name}'")
        return object.__getattribute__(self, name)


req = Request("GET", "/api/users")
print(req.method)
print(req.path)
```

Вывод:

```
[__getattribute__] обращение к 'method'
GET
[__getattribute__] обращение к 'path'
/api/users
```

Строка `object.__getattribute__(self, name)` — это вызов стандартной реализации из базового класса `object`. Она выполняет обычный поиск атрибута: сначала в `__dict__` экземпляра, затем в классе. Без этой строки метод вернул бы `None` (или вообще ничего) — и объект перестал бы нормально работать.

Критически важно понять, почему внутри `__getattribute__` нельзя обращаться к атрибутам через `self.name`:

```python
def __getattribute__(self, name):
    # НЕВЕРНО — вызовет бесконечную рекурсию
    return self._data[name]
    # self._data снова вызывает __getattribute__('_data')
    # тот вызывает __getattribute__('_data') снова...
    # RecursionError: maximum recursion depth exceeded
```

Каждое обращение к `self.что_угодно` внутри `__getattribute__` вновь вызывает `__getattribute__`, что немедленно приводит к переполнению стека. Единственный безопасный способ получить атрибут внутри `__getattribute__` — явно вызвать реализацию из `object`:

```python
def __getattribute__(self, name):
    # ВЕРНО — обходит наш перехватчик и идёт напрямую в стандартную реализацию
    return object.__getattribute__(self, name)
```

Практическое применение `__getattribute__` — логирование доступа к атрибутам, создание прокси-объектов и ленивые вычисления (lazy loading), когда значение атрибута вычисляется только в момент первого обращения. Рассмотрим реальный пример — класс конфигурации, который логирует каждое чтение параметра:

```python
class AppConfig:
    """
    Конфигурация веб-приложения с логированием доступа к настройкам.
    В реальном проекте вместо print использовался бы модуль logging.
    """

    def __init__(self, **settings):
        self._settings = settings

    def __getattribute__(self, name):
        # Служебные атрибуты (начинающиеся с '_') не логируем —
        # иначе внутренняя работа объекта засорила бы лог
        if not name.startswith('_'):
            print(f"[CONFIG READ] параметр '{name}' запрошен")
        return object.__getattribute__(self, name)


config = AppConfig()
# создаем параметры config
config.DEBUG = True
config.DATABASE_URL = 'postgresql://localhost/mydb'
config.SECRET_KEY = 'supersecret'

_ = config._settings

_ = config.DEBUG
# [CONFIG READ] параметр 'DEBUG' запрошен

_ = config.DATABASE_URL
# [CONFIG READ] параметр 'DATABASE_URL' запрошен
```

---

## Метод `__getattr__`: обработка отсутствующих атрибутов

В отличие от `__getattribute__`, который вызывается всегда, метод `__getattr__` вызывается только тогда, когда атрибут не был найден обычным способом. Это резервный обработчик — Python обращается к нему лишь после того, как поиск в `__dict__` экземпляра и в классе завершился неудачей.

```python
class Response:
    def __init__(self, status_code, data):
        self.status_code = status_code
        self.data = data

    def __getattr__(self, name):
        # Этот метод вызовется только для атрибутов,
        # которых нет в __dict__ экземпляра и нет в классе.
        # status_code и data сюда никогда не попадут.
        print(f"[__getattr__] атрибут '{name}' не существует")
        return None


resp = Response(200, {"id": 1})

print(resp.status_code)  # 200  — найден в __dict__, __getattr__ не вызывается
print(resp.headers)      # None — не найден, вызывается __getattr__
# [__getattr__] атрибут 'headers' не существует
```

Разница между двумя методами принципиальна и важна:

| Метод | Когда вызывается |
|---|---|
| `__getattribute__` | При каждом обращении к любому атрибуту |
| `__getattr__` | Только если атрибут не найден стандартным способом |

`__getattr__` значительно безопаснее в использовании: он не требует вызова `object.__getattribute__` и не создаёт риска рекурсии. Его типичное применение — возвращать значение по умолчанию для несуществующих атрибутов и реализовывать «динамические» атрибуты, которые вычисляются на лету.

Практический пример — объект HTTP-ответа, который позволяет обращаться к заголовкам как к атрибутам, не требуя явного задания каждого из них:

```python
class HTTPResponse:
    """
    Объект HTTP-ответа. Стандартные атрибуты (status_code, body)
    хранятся явно. Заголовки доступны через __getattr__ из словаря headers.
    """

    def __init__(self, status_code, body, headers=None):
        self.status_code = status_code
        self.body = body
        # Нормализуем ключи заголовков к нижнему регистру
        self.headers = {k.lower(): v for k, v in headers.items()}

    def __getattr__(self, name):
        # Если атрибут не найден обычным способом, ищем его среди заголовков.
        # Нормализуем имя: content_type → content-type
        header_name = name.replace('_', '-')
        if header_name in self.headers:
            return self.headers[header_name]
        # Если и в заголовках нет — возвращаем None без исключения
        return None


resp = HTTPResponse(
    status_code=200,
    body='{"id": 1, "name": "Alice"}',
    headers={
        "Content-Type": "application/json",
        "X-Request-Id": "abc-123",
        "Cache-Control": "no-cache"
    }
)

# Стандартные атрибуты работают через __dict__
print(resp.status_code)     # 200

# Заголовки доступны как атрибуты через __getattr__
print(resp.content_type)    # application/json
print(resp.x_request_id)    # abc-123
print(resp.cache_control)   # no-cache

# Несуществующий заголовок возвращает None, а не исключение
print(resp.authorization)   # None
```

---

## Метод `__setattr__`: перехват любой записи атрибута

Метод `__setattr__` вызывается при каждой операции присваивания атрибуту объекта — включая те, что происходят внутри `__init__`. Это означает, что если вы переопределите `__setattr__`, он будет работать уже при создании объекта.

```python
class Point:
    def __init__(self, x, y):
        # Эти строки вызовут __setattr__('x', ...) и __setattr__('y', ...)
        self.x = x
        self.y = y

    def __setattr__(self, name, value):
        print(f"[__setattr__] {name} = {value}")
        object.__setattr__(self, name, value)
        # Без этой строки атрибут НЕ будет сохранён


pt = Point(10, 20)
# [__setattr__] x = 10
# [__setattr__] y = 20
```

Как и в случае с `__getattribute__`, здесь существует опасность рекурсии. Если внутри `__setattr__` попытаться присвоить значение через `self.name = value`, это немедленно вызовет тот же `__setattr__` снова:

```python
def __setattr__(self, name, value):
    self.name = value  # ОШИБКА: рекурсия, RecursionError
```

Есть два правильных способа сохранить атрибут внутри `__setattr__`:

```python
# Способ 1: через базовый класс object (предпочтительный)
object.__setattr__(self, name, value)

# Способ 2: напрямую в словарь __dict__ экземпляра
self.__dict__[name] = value
```

Оба способа обходят наш перехватчик и записывают значение напрямую. Способ через `object.__setattr__` предпочтителен, поскольку он корректно работает с дескрипторами (которые разбирались в уроках по инкапсуляции) и другими механизмами Python.

Главная практическая задача `__setattr__` — валидация данных в момент присваивания. Рассмотрим класс, моделирующий запись пользователя, который должен попасть в базу данных:

```python
class UserRecord:
    """
    Запись пользователя с валидацией типов при установке атрибутов.
    Имитирует поведение, которое позже будет реализовано Django ORM
    автоматически через поля модели.
    """

    # Описываем допустимые атрибуты и их ожидаемые типы
    _FIELD_TYPES = {
        'username': str,
        'email': str,
        'age': int,
        'is_active': bool,
    }

    def __init__(self, username, email, age, is_active=True):
        self.username = username
        self.email = email
        self.age = age
        self.is_active = is_active

    def __setattr__(self, name, value):
        # Проверяем тип только для известных полей модели
        if name in self._FIELD_TYPES:
            expected_type = self._FIELD_TYPES[name]
            if not isinstance(value, expected_type):
                raise TypeError(
                    f"Поле '{name}' ожидает тип {expected_type.__name__}, "
                    f"получен {type(value).__name__}"
                )
        # Для служебных атрибутов (начинающихся с '_') проверку пропускаем
        object.__setattr__(self, name, value)

    def __repr__(self):
        return (
            f"UserRecord(username={self.username!r}, "
            f"email={self.email!r}, "
            f"age={self.age!r}, "
            f"is_active={self.is_active!r})"
        )
```

Проверим работу валидации:

```python
# Корректные данные — всё работает
user = UserRecord("alice", "alice@example.com", 28)
print(user)
# UserRecord(username='alice', email='alice@example.com', age=28, is_active=True)

# Изменение с корректным типом — работает
user.age = 29
print(user.age)   # 29

# Попытка передать строку вместо числа
user.age = "двадцать девять"
# TypeError: Поле 'age' ожидает тип int, получен str

# Попытка передать число вместо строки
user.username = 42
# TypeError: Поле 'username' ожидает тип str, получен int

# Попытка передать неверный тип флага
user.is_active = 1
# TypeError: Поле 'is_active' ожидает тип bool, получен int
```

Обратите внимание на последний пример: `1` не является `bool`, хотя в Python `bool` является подклассом `int`, и `True == 1`. Функция `isinstance(1, bool)` вернёт `False`, потому что `1` — это именно `int`, а не `bool`. Это именно то поведение, которое нам нужно в строгой валидации.

---

## Метод `__delattr__`: перехват удаления атрибута

Метод `__delattr__` вызывается при выполнении оператора `del` применительно к атрибуту объекта. Он реже используется на практике, но важен для полноты картины и применяется в тех случаях, когда нужно контролировать, какие атрибуты допустимо удалять.

```python
class Config:
    def __init__(self, **settings):
        for key, value in settings.items():
            object.__setattr__(self, key, value)

    def __delattr__(self, name):
        print(f"[__delattr__] удаление атрибута '{name}'")
        # Вызываем стандартную реализацию — без этой строки атрибут не удалится
        object.__delattr__(self, name)


cfg = Config(DEBUG=True, PORT=8080, HOST="localhost")
print(cfg.DEBUG)    # True

del cfg.DEBUG
# [__delattr__] удаление атрибута 'DEBUG'

print(cfg.__dict__)
# {'PORT': 8080, 'HOST': 'localhost'}
```

Если в `__delattr__` не вызвать `object.__delattr__(self, name)`, атрибут не будет удалён — метод перехватит операцию и просто ничего не сделает. Это может использоваться намеренно — для запрета удаления определённых атрибутов:

```python
class DatabaseConnection:
    """
    Объект соединения с базой данных.
    Некоторые атрибуты нельзя удалять после установки соединения.
    """

    _PROTECTED = {'host', 'port', 'database'}

    def __init__(self, host, port, database):
        self.host = host
        self.port = port
        self.database = database
        self._connected = False

    def __delattr__(self, name):
        if name in self._PROTECTED:
            raise AttributeError(
                f"Атрибут '{name}' защищён от удаления. "
                f"Закройте соединение перед изменением параметров подключения."
            )
        object.__delattr__(self, name)


conn = DatabaseConnection("localhost", 5432, "myapp_db")

del conn._connected    # разрешено — не в списке защищённых
print(conn.__dict__)
# {'host': 'localhost', 'port': 5432, 'database': 'myapp_db'}

del conn.host
# AttributeError: Атрибут 'host' защищён от удаления.
#                 Закройте соединение перед изменением параметров подключения.
```

---

## Полная картина: жизненный цикл атрибута

Теперь, разобрав все четыре метода, можно собрать полную картину того, что происходит при каждой операции с атрибутом:

```
Запись:      obj.x = 10   →  __setattr__('x', 10)
                               → внутри: object.__setattr__ или self.__dict__[key] = value

Чтение:      obj.x        →  __getattribute__('x')
                               → если найден: вернуть значение
                               → если НЕ найден: __getattr__('x')

Удаление:    del obj.x    →  __delattr__('x')
                               → внутри: object.__delattr__ или del self.__dict__[key]
```

Связь между методами не симметрична. `__getattribute__` и `__setattr__` вызываются всегда и без исключений — они являются основным слоем перехвата. `__getattr__` вызывается только при неудаче поиска — он является запасным слоем. `__delattr__` вызывается при удалении и независим от остальных.

---

## Практический пример: класс `RequestValidator`

Объединим все четыре метода в одном реалистичном примере. Представим класс, который моделирует объект входящего HTTP-запроса с жёсткой схемой полей — аналог того, с чем студент встретится при работе с сериализаторами Django REST Framework:

```python
class RequestValidator:
    """
    Объект входящего запроса с контролем доступа к атрибутам.

    Разрешённые поля и их типы описаны в _SCHEMA.
    Поля из _READONLY нельзя изменить после установки.
    Поля из _REQUIRED нельзя удалить.
    При обращении к неизвестному полю возвращается None.
    """

    _SCHEMA = {
        'method': str,
        'path': str,
        'body': dict,
        'user_id': int,
        'is_authenticated': bool,
    }

    _READONLY = {'method', 'path'}  # нельзя изменить после установки
    _REQUIRED = {'method', 'path', 'is_authenticated'}  # нельзя удалить

    def __init__(self, method, path, body=None, user_id=None, is_authenticated=False):
        # Используем object.__setattr__, чтобы первая запись не прошла
        # через наш __setattr__ с проверкой READONLY (атрибуты ещё не установлены)
        object.__setattr__(self, '_initialized', False)
        self.method = method
        self.path = path
        self.body = body or {}
        self.user_id = user_id
        self.is_authenticated = is_authenticated
        object.__setattr__(self, '_initialized', True)

    def __getattribute__(self, name):
        # Логируем только обращения к полям схемы, не к служебным атрибутам
        schema = object.__getattribute__(self, '_SCHEMA')
        if name in schema:
            print(f"[READ] {name}")
        return object.__getattribute__(self, name)

    def __getattr__(self, name):
        # Обращение к полю, которого нет ни в __dict__, ни в классе
        print(f"[WARN] поле '{name}' не определено в схеме, возвращаем None")
        return None

    def __setattr__(self, name, value):
        # Проверяем попытку изменить readonly-поле после инициализации
        try:
            initialized = object.__getattribute__(self, '_initialized')
        except AttributeError:
            initialized = False

        readonly = object.__getattribute__(self, '_READONLY')
        if initialized and name in readonly:
            raise AttributeError(
                f"Поле '{name}' доступно только для чтения после инициализации запроса"
            )

        # Проверяем тип, если поле описано в схеме
        schema = object.__getattribute__(self, '_SCHEMA')
        if name in schema and value is not None:
            expected = schema[name]
            if not isinstance(value, expected):
                raise TypeError(
                    f"Поле '{name}' ожидает {expected.__name__}, "
                    f"получен {type(value).__name__}"
                )

        object.__setattr__(self, name, value)

    def __delattr__(self, name):
        required = object.__getattribute__(self, '_REQUIRED')
        if name in required:
            raise AttributeError(
                f"Поле '{name}' является обязательным и не может быть удалено"
            )
        object.__delattr__(self, name)

    def __repr__(self):
        return (
            f"RequestValidator(method={self.method!r}, path={self.path!r}, "
            f"user_id={self.user_id!r}, is_authenticated={self.is_authenticated!r})"
        )
```

Посмотрим на работу объекта:

```python
req = RequestValidator(
    method="POST",
    path="/api/orders",
    body={"product_id": 42, "quantity": 3},
    user_id=7,
    is_authenticated=True
)

# Чтение поля — логируется через __getattribute__
print(req.method)
# [READ] method
# POST

# Обращение к несуществующему полю — через __getattr__
print(req.ip_address)
# [WARN] поле 'ip_address' не определено в схеме, возвращаем None
# None

# Изменение разрешённого поля с корректным типом
req.user_id = 15
print(req.user_id)
# [READ] user_id
# [READ] user_id  ← второй раз при print
# 15

# Попытка изменить readonly-поле
req.method = "GET"
# AttributeError: Поле 'method' доступно только для чтения после инициализации запроса

# Попытка передать неверный тип
req.user_id = "admin"
# TypeError: Поле 'user_id' ожидает int, получен str

# Попытка удалить обязательное поле
del req.is_authenticated
# AttributeError: Поле 'is_authenticated' является обязательным и не может быть удалено
```

Этот пример намеренно сложнее предыдущих — он демонстрирует, как четыре метода работают совместно и как избежать рекурсии при их совместном использовании. 

Ключевой приём: на протяжении всего класса для чтения атрибутов внутри методов используется `object.__getattribute__(self, name)`, а не `self.name`, потому что в классе переопределён `__getattribute__`, который выводит логирующее сообщение при обращении к полям схемы. 

Если внутри `__setattr__` или `__delattr__` читать атрибуты через `self.name`, это: во-первых, создаст нежелательные записи в лог при каждой внутренней проверке, во-вторых — в случае ошибки в `__getattribute__` приведёт к рекурсии. Использование `object.__getattribute__` обходит наш перехватчик и обращается к значению напрямую.

---

## Когда использовать эти методы

Все четыре метода — инструменты низкого уровня, и их следует применять осознанно. Неправильное использование `__getattribute__` и `__setattr__` легко приводит к рекурсии или к тому, что объект начинает работать непредсказуемо. Это не означает, что их нужно избегать — но к ним нужно прибегать только тогда, когда более простые инструменты (`@property`, дескрипторы) не решают задачу.

Типичные задачи, где эти методы действительно нужны:

`__getattribute__` — логирование доступа к атрибутам в целях аудита, прокси-объекты, перенаправляющие вызовы к другому объекту, ленивая загрузка данных (атрибут вычисляется только при первом обращении).

`__getattr__` — возврат значений по умолчанию для несуществующих атрибутов, динамические атрибуты (как в примере с HTTP-заголовками), объекты-заглушки в тестах.

`__setattr__` — централизованная валидация типов при записи, защита атрибутов от изменения, синхронизация атрибутов с внешним хранилищем (база данных, кеш).

`__delattr__` — защита критически важных атрибутов от случайного удаления, синхронизация удаления с внешним хранилищем.

---

## Итоги урока

В этом уроке мы разобрали механизм, который стоит за простым синтаксисом точки в Python. Четыре метода — `__getattribute__`, `__getattr__`, `__setattr__`, `__delattr__` — дают полный контроль над жизненным циклом атрибутов объекта.

`__getattribute__` перехватывает любое чтение и требует особой осторожности из-за риска рекурсии: внутри него нельзя обращаться к атрибутам через `self`, только через `object.__getattribute__(self, name)`.

`__getattr__` — безопасный резервный обработчик, вызывается только для несуществующих атрибутов и не требует специальных мер против рекурсии.

`__setattr__` перехватывает любую запись, включая операции внутри `__init__`. Для сохранения значения внутри него необходимо явно вызывать `object.__setattr__(self, name, value)`.

`__delattr__` перехватывает удаление и, как и предыдущий метод, требует явного вызова `object.__delattr__(self, name)` для фактического удаления атрибута.

В следующем уроке мы рассмотрим метод `__call__`, который позволяет объекту вести себя как функция — вызываться с помощью скобок и принимать аргументы.

---

## Вопросы

1. В каком порядке Python ищет атрибут при обращении `obj.x`? Опишите последовательность шагов.
2. Чем принципиально отличается `__getattribute__` от `__getattr__`? В каком случае каждый из них уместен?
3. Почему внутри `__getattribute__` нельзя писать `self.some_attr`? Что произойдёт?
4. Метод `__setattr__` вызывается в том числе при выполнении `self.x = value` внутри `__init__`. Что это означает на практике при переопределении `__setattr__`?
5. Назовите два способа сохранить атрибут внутри `__setattr__`, не вызвав рекурсию. Есть ли между ними разница?
6. Что произойдёт, если в методе `__delattr__` не вызвать `object.__delattr__(self, name)`?
7. Почему в финальном примере (`RequestValidator`) для чтения атрибутов внутри методов используется `object.__getattribute__(self, name)`, а не `self.name`?
8. В чём разница между записью `self.MIN_COORD = value` и `ClassName.MIN_COORD = value`? Как это связано с `__setattr__`?

## Задачи

### **Задача 1**. 

Класс `Book`

Создайте класс `Book`, описывающий книгу. Объект должен создаваться двумя способами: 
* `Book()` без аргументов 
* `Book(title, author, pages, year)` с четырьмя аргументами. 

При создании без аргументов атрибуты должны принимать значения по умолчанию: 
* `title` и `author` — пустые строки, 
* `pages` и `year` — целые числа `0`. 

Реализуйте `__setattr__` так, чтобы при попытке присвоить атрибуту значение неверного типа выбрасывалось исключение `TypeError("Неверный тип присваиваемых данных.")`. 

Допустимые типы: 
* `title` и `author` — строки (`str`), 
* `pages` и `year` — целые числа (`int`). 

Реализуйте также `__str__` в формате `"Название" — Автор (год, N стр.)` и `__repr__` в формате `Book(title='...', author='...', pages=..., year=...)`.

**Пример использования**:

```python
book = Book("Python ООП", "Олег Сергеев", 320, 2025)

print(book)
# "Python ООП" — Олег Сергеев (2025, 320 стр.)

print(repr(book))
# Book(title='Python ООП', author='Олег Сергеев', pages=320, year=2025)

book.pages = 350      # корректно
print(book.pages)     # 350

book.pages = "много"  # TypeError: Неверный тип присваиваемых данных.

empty_book = Book()
print(empty_book)
# "" —  (0, 0 стр.)
```

---

### **Задача 2**. 

Класс `User`

Создайте класс `User`, описывающий пользователя системы. 

Объект создаётся командой `User(name, age)`, где `name` — строка, `age` — целое число не меньше нуля. 

Реализуйте `__setattr__` с двумя видами проверок: 
при несоответствии типа выбрасывать `TypeError("Неверный тип данных")`, 
при отрицательном значении возраста — `ValueError("Возраст не может быть отрицательным")`. 

Реализуйте `__getattr__` так, чтобы при обращении к любому несуществующему атрибуту возвращалась строка `"Атрибут не найден"` без выброса исключения. 

Добавьте `__str__` в формате `User(name, age лет)` и `__repr__` в формате `User(name='...', age=...)`.

**Пример использования**:

```python
user = User("Ivan", 25)
print(user)
# User(Ivan, 25 лет)

user.age = 30
print(user.name, user.age)  # Ivan 30

print(user.email)      # Атрибут не найден
print(user.phone)      # Атрибут не найден

try:
    user.age = -5
except ValueError as e:
    print(e)           # Возраст не может быть отрицательным

try:
    user.name = 42
except TypeError as e:
    print(e)           # Неверный тип данных
```

---

### **Задача 3**. 

Классы `Shop` и `Product`

Реализуйте два класса: `Shop` и `Product`. 

Объект `Product` создаётся командой `Product(name, weight, price)`, где `name` — строка, `weight` и `price` — положительные числа (`int` или `float`). 

Каждый продукт должен автоматически получать уникальный целочисленный идентификатор `id`, начиная с `1` (используйте атрибут класса-счётчика). 

Реализуйте `__setattr__` с проверкой типов: 
* `id` должен быть `int`, 
* `name` — `str`, 
* `weight` и `price` — `int` или `float`; 

При нарушении выбрасывать `TypeError("Неверный тип присваиваемых данных.")`. 

Реализуйте `__delattr__` так, чтобы удаление атрибута `id` вызывало `AttributeError("Атрибут id удалять запрещено.")`. 

Класс `Shop` принимает `name` (строку), хранит список товаров в атрибуте `goods` и предоставляет методы `add_product(product)` и `remove_product(product)`.

**Пример использования**:

```python
shop = Shop("Мой магазин")
p1 = Product("Ноутбук", 2.5, 89990)
p2 = Product("Мышь", 0.15, 1500)

shop.add_product(p1)
shop.add_product(p2)

print(p1.id, p1.name, p1.price)   # 1 Ноутбук 89990
print(p2.id, p2.name, p2.price)   # 2 Мышь 1500
print(len(shop.goods))             # 2

try:
    p1.price = "дорого"
except TypeError as e:
    print(e)                       # Неверный тип присваиваемых данных.

try:
    del p1.id
except AttributeError as e:
    print(e)                       # Атрибут id удалять запрещено.
```

---

### **Задача 4**. 

Класс `SafeDict`

Реализуйте класс `SafeDict`, который ведёт себя как словарь, но доступ к данным осуществляется через атрибуты объекта. 

Все данные должны храниться внутри приватного атрибута `__data` (обычный словарь Python). 

Операция `obj.key = value` должна сохранять значение в `__data`. 

Операция `obj.key` — возвращать значение из `__data`. 

Операция `del obj.key` — удалять ключ из `__data`. 

При обращении к несуществующему ключу метод должен возвращать `None`, не выбрасывая исключения. 

Прямой доступ к `__data` через `obj._SafeDict__data` должен быть запрещён: при такой попытке необходимо выбросить `AttributeError("Прямой доступ запрещён")`. 

Все операции должны работать только через магические методы, без дополнительных вспомогательных методов `get`, `set` и т.д.

**Пример использования**:

```python
data = SafeDict()

data.name = "Ivan"
data.city = "Moscow"
data.age = 30

print(data.name)     # Ivan
print(data.city)     # Moscow
print(data.email)    # None — ключа нет, но исключения нет

del data.city
print(data.city)     # None — удалён

try:
    print(data._SafeDict__data)
except AttributeError as e:
    print(e)         # Прямой доступ запрещён
```

---

### **Задача 5**. 

Класс `AccessLogger`

Реализуйте класс `AccessLogger` — обёртку (proxy) над произвольным объектом, которая логирует все обращения к его атрибутам. 

Класс принимает любой объект и сохраняет его внутри атрибута `_obj`. 

При чтении атрибута через `AccessLogger` необходимо выводить строку `Доступ к атрибуту: <имя>` и возвращать значение из оригинального объекта. 

При записи атрибута — выводить `Изменение атрибута: <имя> = <значение>` и устанавливать значение в оригинальном объекте. 

Логирование не должно срабатывать на служебные атрибуты самого `AccessLogger` (в первую очередь на `_obj`). 

Для проверки определите простой класс `Person(name, age)` и используйте его как оборачиваемый объект.

**Пример использования**:

```python
person = Person("Bob", 25)
logger = AccessLogger(person)

print(logger.name)
# Доступ к атрибуту: name
# Bob

logger.age = 40
# Изменение атрибута: age = 40

print(person.age)    # 40 — изменился в оригинальном объекте

logger.city = "Nalchik"
# Изменение атрибута: city = Nalchik

print(person.city)   # Nalchik
print(logger.city)
# Доступ к атрибуту: city
# Nalchik
```

---

[Предыдущий урок](lesson10.md) | [Следующий урок](lesson12.md)