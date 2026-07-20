# Урок 31. JWT — теория, сессии и токены

## Почему этот урок важен

На каждом собеседовании по backend-разработке звучат вопросы: «Как работает авторизация на вашем сайте?», «Чем JWT отличается от сессий?», «Когда использовать токены, а когда сессии?». Это фундаментальные знания, без которых сложно говорить о современной веб-разработке.

Мы уже реализовали авторизацию в нашем проекте — через сессии. Сейчас разберём, как она работает изнутри, что такое JWT и почему для SPA и мобильных приложений используют именно токены. А в конце напишем минимальный пример JWT на Python — без DRF, просто чтобы увидеть механику.

---

## Часть 1. Как работают сессии

### Проблема HTTP

HTTP — протокол без состояния (stateless). Каждый запрос к серверу независим: сервер не помнит, кто делал предыдущий запрос. Когда пользователь входит на сайт и переходит на следующую страницу — с точки зрения HTTP это два совершенно разных запроса от неизвестного клиента.

Чтобы сервер «помнил» пользователя между запросами, нужен механизм идентификации. Сессии — один из способов решить эту проблему.

### Механика сессий в Django

```
1. Пользователь отправляет логин и пароль
         ↓
2. Django проверяет учётные данные
         ↓
3. Django создаёт запись в таблице django_session
   (или в Redis/файле — зависит от настройки SESSION_ENGINE)
   со случайным уникальным ключом: "abc123xyz..."
         ↓
4. Django отправляет браузеру cookie: sessionid=abc123xyz...
         ↓
5. Браузер сохраняет cookie и отправляет его
   с каждым последующим запросом автоматически
         ↓
6. Django получает запрос, читает sessionid из cookie,
   находит сессию в таблице, достаёт user_id,
   загружает пользователя из auth_user
         ↓
7. request.user = пользователь
```

Ключевой момент: **данные хранятся на сервере**. Cookie содержит только идентификатор (`sessionid`), а не данные пользователя. Чтобы узнать, кто делает запрос, Django всегда обращается к хранилищу сессий.

### Посмотреть сессии в нашем проекте

```bash
python manage.py shell
```

```python
from django.contrib.sessions.models import Session
from django.utils import timezone

# Все активные сессии
sessions = Session.objects.filter(expire_date__gt=timezone.now())
for s in sessions:
    data = s.get_decoded()
    print(f'Session key: {s.session_key[:10]}...')
    print(f'Data: {data}')
    # {'_auth_user_id': '1', '_auth_user_backend': '...', '_auth_user_hash': '...'}
```

В таблице `django_session` хранится: ключ сессии (случайная строка), зашифрованные данные (содержат `user_id`), дата истечения.

### Где хранятся сессии

По умолчанию Django хранит сессии в базе данных (`django_session`). Но это можно изменить:

```python
# settings.py

# База данных (по умолчанию)
SESSION_ENGINE = 'django.contrib.sessions.backends.db'

# Redis — более быстро, не нагружает БД
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'  # наш Redis из прошлого урока

# Файловая система
SESSION_ENGINE = 'django.contrib.sessions.backends.file'

# Cookie (данные в самом cookie, не на сервере) — не рекомендуется для чувствительных данных
SESSION_ENGINE = 'django.contrib.sessions.backends.signed_cookies'
```

Для нашего проекта с Redis из урока 30 — логично перенести сессии туда:

```python
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'
```

---

## Часть 2. Что такое JWT

### Идея токена

Токен — это другой подход к той же проблеме: как сервер узнаёт, кто делает запрос? Вместо того чтобы хранить данные на сервере и выдавать клиенту только ключ, токен содержит **сами данные** в закодированном виде.

**JWT (JSON Web Token)** — стандарт (RFC 7519) для передачи данных между сторонами в виде JSON-объекта, подписанного цифровой подписью. Клиент хранит токен у себя и отправляет его с каждым запросом.

### Структура JWT

JWT выглядит как три строки, разделённые точками:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImFkbWluIiwiZXhwIjoxNzAwMDAwMDAwfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

Три части:

**1. Header (заголовок)** — алгоритм подписи и тип токена:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**2. Payload (полезная нагрузка)** — данные (claims):

```json
{
  "user_id": 1,
  "username": "admin",
  "exp": 1700000000,
  "iat": 1699996400
}
```

`exp` — время истечения токена (Unix timestamp), `iat` — время выпуска. Эти поля стандартные. Остальные (`user_id`, `username`) — произвольные, их добавляем мы.

**3. Signature (подпись)** — результат подписи header+payload секретным ключом:

```
HMACSHA256(
  base64url(header) + "." + base64url(payload),
  SECRET_KEY
)
```

Каждая часть кодируется в base64url (не шифруется — только кодируется). Подпись гарантирует, что данные не были изменены: если кто-то изменит payload, подпись перестанет совпадать, и сервер отвергнет токен.

> **Важно:** payload JWT можно прочитать без ключа — это не шифрование, а подписание. Никогда не кладите в JWT пароли или чувствительные данные. Проверить токен (убедиться, что он не изменён) можно только зная секретный ключ.

### Механика JWT-авторизации

```
1. Пользователь отправляет логин и пароль
         ↓
2. Сервер проверяет учётные данные
         ↓
3. Сервер формирует JWT с user_id и временем истечения,
   подписывает своим SECRET_KEY
         ↓
4. Сервер возвращает токен клиенту в теле ответа
         ↓
5. Клиент сохраняет токен (localStorage, sessionStorage, память)
         ↓
6. При каждом запросе клиент отправляет токен
   в заголовке: Authorization: Bearer <token>
         ↓
7. Сервер получает токен, проверяет подпись,
   проверяет срок действия, читает user_id
         ↓
8. Никакого обращения к БД или хранилищу — сервер
   доверяет подписи
```

Ключевой момент: **данные хранятся у клиента**. Сервер не хранит ни токен, ни сессию — он только проверяет подпись при каждом запросе.

---

## Часть 3. Сессии vs JWT — детальное сравнение

### Хранение состояния

| | Сессии | JWT |
|---|---|---|
| Где хранятся данные | На сервере (БД, Redis) | У клиента (токен) |
| Что у клиента | Только ID сессии (cookie) | Весь токен с данными |
| Обращение к хранилищу при каждом запросе | Да — найти сессию | Нет — только проверить подпись |

### Масштабирование

Это главное преимущество JWT в распределённых системах.

Представим: два сервера приложения за балансировщиком нагрузки.

**С сессиями:**
```
Запрос 1 → Сервер А → создаёт сессию в БД
Запрос 2 → Сервер Б → должен найти сессию → идёт в ту же БД ✅
Работает, но все серверы зависят от одного хранилища сессий
```

**С JWT:**
```
Запрос 1 → Сервер А → выдаёт JWT, подписанный SECRET_KEY
Запрос 2 → Сервер Б → получает JWT, проверяет подпись тем же SECRET_KEY ✅
Работает без общего хранилища — достаточно знать SECRET_KEY
```

### Отзыв токена (logout)

Это главный **недостаток** JWT по сравнению с сессиями.

**С сессиями:** выход — это удаление записи из таблицы сессий. Немедленно. `Session.objects.filter(pk=session_key).delete()` — и пользователь вышел.

**С JWT:** после успешной авторизации сервер создаёт JWT-токены и отправляет их клиенту. Дальше именно клиент хранит эти токены и прикладывает их к запросам. Сам сервер, как правило, не хранит информацию о выданных access-токенах, поэтому мгновенно "забрать" их обратно уже нельзя.

Чтобы «отозвать» токен, можно:
- Подождать, пока истечёт `exp` — после этого токен автоматически станет недействительным.
- Вести чёрный список отозванных токенов на сервере — но тогда теряется главное преимущество JWT (stateless).
- Использовать очень короткое время жизни access-токена (15 минут) и refresh-токен для получения нового access-токена.

Именно поэтому JWT обычно используют парами:

```
Access token  — короткоживущий (15 мин — 1 час)
                используется для авторизации каждого запроса к API

Refresh token — долгоживущий (7–30 дней)
                используется только для получения нового access token
```

Токены всегда создаются **сервером** и отправляются **клиенту**, где затем хранятся. Для хранения могут использоваться:
- память приложения (временное хранилище до перезагрузки страницы);
- `localStorage` — постоянное хранилище браузера;
- `sessionStorage` — хранилище до закрытия вкладки;
- `httpOnly Cookie` — специальная Cookie, недоступная JavaScript.

На практике чаще всего **access token** хранится в памяти приложения (или реже в `localStorage`), а **refresh token** — в `httpOnly Cookie`, так как это считается наиболее безопасным вариантом.

При выходе пользователя **refresh token** обычно аннулируется на сервере (небольшой чёрный список только для refresh-токенов), а **access token** просто перестаёт работать после истечения срока действия.

### Когда что использовать

| Критерий | Сессии | JWT |
|---|---|---|
| Веб-приложение с шаблонами | ✅ Идеально | Избыточно |
| REST API для SPA или мобильного приложения | Неудобно (CORS, cookie) | ✅ Стандарт |
| Микросервисная архитектура | Сложно (общее хранилище) | ✅ Stateless |
| Нужен мгновенный logout | ✅ Просто | Сложно |
| Горизонтальное масштабирование | Нужно общее хранилище | ✅ Проще |
| Браузерный клиент | ✅ Cookie безопаснее | localStorage уязвим к XSS |

**Вывод для нашего проекта:** Django-сайт с шаблонами — сессии правильный выбор. JWT понадобится в следующем курсе, когда мы будем делать REST API с DRF, и к нему подключать фронтенд или мобильное приложение.

---

## Часть 4. JWT на практике — минимальный пример

Покажем механику JWT на Python, не подключая DRF.

```bash
pip install PyJWT
```

### Создание и проверка токена

```python
# Запустить в shell: python manage.py shell
import jwt
import datetime
from django.conf import settings


# Создаём токен
def create_access_token(user_id: int, username: str) -> str:
    payload = {
        'user_id': user_id,
        'username': username,
        'exp': datetime.datetime.utcnow() + datetime.timedelta(minutes=15),
        'iat': datetime.datetime.utcnow(),
        'type': 'access',
    }
    token = jwt.encode(payload, settings.SECRET_KEY, algorithm='HS256')
    return token


# Проверяем и декодируем токен
def decode_access_token(token: str) -> dict:
    try:
        payload = jwt.decode(token, settings.SECRET_KEY, algorithms=['HS256'])
        return payload
    except jwt.ExpiredSignatureError:
        raise ValueError('Токен истёк')
    except jwt.InvalidTokenError:
        raise ValueError('Токен недействителен')


# Пример использования
from django.contrib.auth.models import User

user = User.objects.first()
token = create_access_token(user.id, user.username)
print(f'Token: {token[:50]}...')

# Декодируем
payload = decode_access_token(token)
print(f'Payload: {payload}')
# {'user_id': 1, 'username': 'admin', 'exp': ..., 'iat': ..., 'type': 'access'}
```

### Смотрим на структуру токена вручную

```python
import base64, json

parts = token.split('.')

# Декодируем header
header = base64.b64decode(parts[0] + '==')
print('Header:', json.loads(header))
# {'alg': 'HS256', 'typ': 'JWT'}

# Декодируем payload — без знания SECRET_KEY!
payload_raw = base64.b64decode(parts[1] + '==')
print('Payload:', json.loads(payload_raw))
# {'user_id': 1, 'username': 'admin', 'exp': ..., 'iat': ...}

# Третья часть — подпись, бинарные данные
print('Signature (bytes):', parts[2])
```

Это наглядно показывает: payload читается без ключа. Подпись нужна только для **проверки целостности**, а не для скрытия данных.

### Имитация API-запроса с JWT

Напишем простейший FBV-endpoint, который принимает JWT в заголовке:

```python
# films/views.py
import jwt
from django.conf import settings
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt
from django.contrib.auth.models import User


def jwt_protected_api(request):
    """
    Пример API-представления с JWT-авторизацией.
    Реальный вариант будет реализован через DRF в следующем курсе.
    """
    auth_header = request.headers.get('Authorization', '')

    if not auth_header.startswith('Bearer '):
        return JsonResponse({'error': 'Токен не предоставлен'}, status=401)

    token = auth_header.split(' ')[1]

    try:
        payload = jwt.decode(token, settings.SECRET_KEY, algorithms=['HS256'])
    except jwt.ExpiredSignatureError:
        return JsonResponse({'error': 'Токен истёк'}, status=401)
    except jwt.InvalidTokenError:
        return JsonResponse({'error': 'Токен недействителен'}, status=401)

    user_id = payload.get('user_id')
    try:
        user = User.objects.get(pk=user_id)
    except User.DoesNotExist:
        return JsonResponse({'error': 'Пользователь не найден'}, status=401)

    # Возвращаем данные — как настоящий API
    return JsonResponse({
        'user_id': user.id,
        'username': user.username,
        'message': f'Привет, {user.username}! Это защищённый endpoint.'
    })
```

```python
# films/urls.py
path('api/me/', jwt_protected_api, name='api_me'),
```

Проверить через curl:

```bash
# Получаем токен в shell, копируем

# Запрос с токеном
curl -H "Authorization: Bearer <ваш_токен>" http://localhost:8000/api/me/

# Запрос без токена — получим 401
curl http://localhost:8000/api/me/
```

---

## Часть 5. Затравка для курса по DRF

Пример выше — это ручная реализация JWT без инструментов. В реальном API-проекте так не делают. Django REST Framework предоставляет полноценную инфраструктуру:

```python
# Так выглядит то же самое в DRF + simplejwt
# (в следующем курсе)

from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework_simplejwt.views import TokenObtainPairView

# Получение токена
path('api/token/', TokenObtainPairView.as_view()),

# Защищённый endpoint
@api_view(['GET'])
@permission_classes([IsAuthenticated])
def me(request):
    return Response({
        'user_id': request.user.id,
        'username': request.user.username,
    })
```

DRF берёт на себя: парсинг запроса, сериализацию ответа, документацию через Swagger, throttling, pagination, permissions — всё то, что мы делали вручную в нашем примере. JWT-авторизация через `simplejwt` подключается в несколько строк настроек.

Наш проект на Django с шаблонами и DRF-API — это не противоречие. Многие продакшн-проекты сочетают оба подхода: шаблоны для SEO-страниц, API для мобильного приложения или React/Vue-фронтенда.

---

## Подводные камни

### JWT в localStorage — уязвимость к XSS

Хранить JWT в `localStorage` браузера — распространённая практика, но небезопасная: любой JS-скрипт на странице (в том числе инжектированный через XSS) может прочитать `localStorage`. Более безопасный вариант — `httpOnly` cookie (недоступен JS), но тогда теряется простота CORS. Это компромисс, который каждый проект решает по-своему.

### Секретный ключ — это всё

Если `SECRET_KEY` из Django `settings.py` утечёт — злоумышленник сможет создавать произвольные токены для любого пользователя. Для JWT в продакшне часто используют отдельный ключ, а не `SECRET_KEY` Django. В `djangorestframework-simplejwt` для этого есть настройка `SIGNING_KEY`.

### Размер токена

JWT с полезной нагрузкой весит 200–400 байт и передаётся с каждым запросом в заголовке. Для большинства API это незначительно, но при большом payload (не кладите туда лишних данных) или высоком трафике это становится заметным.

---

## Вопросы для проверки

1. Объясни механику сессий: что хранится на сервере, что у клиента и как сервер идентифицирует пользователя при каждом запросе?
2. Из каких трёх частей состоит JWT и почему payload можно прочитать без знания секретного ключа?
3. Почему сессии лучше подходят для немедленного logout, а с JWT это сложнее?
4. В каких архитектурных ситуациях JWT предпочтительнее сессий?
5. Почему в нашем текущем Django-проекте с шаблонами правильно использовать сессии, а не JWT?

---

[Предыдущий урок](lesson30.md) | [Следующий урок](lesson32.md)