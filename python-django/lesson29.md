# Урок 29. OAuth 2.0: вход через GitHub и ВКонтакте

## Что такое OAuth 2.0 и зачем он нужен

OAuth 2.0 — это протокол делегированной авторизации. Вместо того чтобы создавать и хранить пароль на нашем сайте, пользователь входит через стороннюю платформу (GitHub, ВКонтакте), которая подтверждает его личность и передаёт нам базовую информацию: имя, email, аватар.

Это удобно для пользователя — не нужно запоминать ещё один пароль. И снижает ответственность нашего приложения — мы не храним пароли вообще, если пользователь авторизовался только через соцсеть.

Принципиальная схема работает так:

```
Пользователь нажимает «Войти через GitHub»
        ↓
Браузер переходит на GitHub — пользователь подтверждает доступ
        ↓
GitHub перенаправляет обратно на наш сайт с временным кодом
        ↓
Наш сервер обменивает код на access token
        ↓
Наш сервер запрашивает данные пользователя у GitHub по токену
        ↓
Django создаёт или находит пользователя в БД и входит от его имени
```

Весь этот механизм реализует библиотека `python-social-auth` — нам остаётся только настроить параметры.

---

## Установка python-social-auth

```bash
pip install social-auth-app-django
```

Добавляем в `INSTALLED_APPS` и подключаем маршруты:

```python
# filmsite/settings.py
INSTALLED_APPS = [
    # ... существующие приложения ...
    'social_django',
]
```

```python
# filmsite/urls.py
urlpatterns = [
    path('admin/', admin.site.urls),
    path('auth/', include('social_django.urls', namespace='social')),
    path('accounts/login/', LoginView.as_view(authentication_form=CustomAuthenticationForm), name='login'),
    path('accounts/', include('django.contrib.auth.urls')),
    path('accounts/register/', RegisterView.as_view(), name='register'),
    path('', include('films.urls')),
]
```

Применяем миграции — `social_django` создаёт свои таблицы для хранения связей между Django-пользователями и их OAuth-профилями:

```bash
python manage.py migrate
```

---

## Настройка AUTHENTICATION_BACKENDS

`python-social-auth` тоже работает через механизм бэкендов аутентификации, который мы разобрали в уроке 24. Добавляем бэкенды для GitHub и ВКонтакте в `settings.py`:

```python
# filmsite/settings.py
AUTHENTICATION_BACKENDS = [
    'social_core.backends.github.GithubOAuth2',      # GitHub
    'social_core.backends.vk.VKOAuth2',              # ВКонтакте
    'films.backends.EmailAuthBackend',                # наш бэкенд — вход по email
    'django.contrib.auth.backends.ModelBackend',      # стандартный — вход по username
]
```

---

## Часть 1. Авторизация через GitHub

GitHub OAuth работает на `localhost` — это удобно для разработки и позволяет протестировать всё прямо сейчас.

### Создание OAuth-приложения на GitHub

1. Открой [github.com/settings/developers](https://github.com/settings/developers)
2. Нажми **«New OAuth App»**
3. Заполни поля:

| Поле | Значение |
|---|---|
| Application name | filmsite-local |
| Homepage URL | `http://localhost:8000` или `http://127.0.0.1:8000/` |
| Authorization callback URL | `http://localhost:8000/auth/complete/github/` или `http://127.0.0.1:8000/auth/complete/github/` |

4. Нажми **«Register application»**
5. На следующей странице скопируй **Client ID**
6. Нажми **«Generate a new client secret»** и скопируй секрет — он показывается **один раз**

### Настройка в settings.py

```python
# filmsite/settings.py
SOCIAL_AUTH_GITHUB_KEY = 'твой_client_id'
SOCIAL_AUTH_GITHUB_SECRET = 'твой_client_secret'
```

Учётные данные через переменные окружения — по тому же принципу, что DB и SMTP:

```python
import os

SOCIAL_AUTH_GITHUB_KEY = os.environ.get('GITHUB_KEY', '')
SOCIAL_AUTH_GITHUB_SECRET = os.environ.get('GITHUB_SECRET', '')
```

Можно дополнительно запросить email:

```python
SOCIAL_AUTH_GITHUB_SCOPE = ['user:email']
```

### Куда перенаправлять после входа

```python
# filmsite/settings.py
LOGIN_REDIRECT_URL = '/'       # уже настроено в уроке 23
```

`python-social-auth` использует `LOGIN_REDIRECT_URL` после успешной OAuth-авторизации — дополнительных настроек не нужно.

### Добавляем кнопку входа в шаблон

```html
<!-- templates/registration/login.html -->
{% extends 'base.html' %}

{% block title %}Вход — Сайт фильмов{% endblock %}

{% block content %}
    <h1>Вход на сайт</h1>

    {% if form.errors %}
        <p class="error">Неверное имя пользователя/email или пароль.</p>
    {% endif %}

    <form method="POST">
        {% csrf_token %}
        {{ form.as_p }}
        <input type="hidden" name="next" value="{{ next }}">
        <button type="submit">Войти</button>
    </form>

    <hr>

    <p>Войти через:</p>
    <form action="{% url 'social:begin' 'github' %}" method="post">
        {% csrf_token %}
        <button type="submit">
            Войти через GitHub
        </button>
    </form>
    <form action="{% url 'social:begin' 'vk-oauth2' %}" method="post">
        {% csrf_token %}
        <button type="submit">
            Войти через ВКонтакте
        </button>
    </form>

    <p>Ещё нет аккаунта? <a href="{% url 'register' %}">Зарегистрироваться</a></p>
    <p><a href="{% url 'password_reset' %}">Забыли пароль?</a></p>
{% endblock %}
```

Тег `{% url 'social:begin' 'github' %}` строит URL вида `/auth/login/github/` — это точка входа, после которой `python-social-auth` сам перенаправляет пользователя на GitHub.

### Проверка

Запусти сервер и открой `http://localhost:8000/accounts/login/`. Нажми «GitHub» — должен открыться экран авторизации GitHub. После подтверждения пользователь вернётся на главную страницу. В административной панели в разделе «Social Auth» → «User social auths» появится запись с привязкой.

---

## Часть 2. Авторизация через ВКонтакте

> **Важное ограничение:** ВКонтакте не принимает `localhost` как доверенный redirect URI — при попытке получишь ошибку `redirect_uri is incorrect`. Настройку ВК выполним полностью, но проверить её можно будет только после деплоя сайта на реальный сервер с доменным именем — это разберём в модуле 9.
>
> Если хочешь протестировать OAuth ВКонтакте локально прямо сейчас — посмотри в сторону **ngrok**: он создаёт публичный HTTPS-туннель к localhost за несколько минут. Официальная документация: [ngrok.com/docs](https://ngrok.com/docs).

### Создание приложения ВКонтакте

1. Перейди на [dev.vk.com](https://dev.vk.com) и нажми **«Создать приложение»**
2. Укажи:
   - Название: `filmsite`
   - Платформа: **Сайт**
   - Нажми **«Подключить сайт»**
3. На следующем шаге укажи **URL сайта** — адрес твоего домена (например, `https://filmsite.ru`). На локальном адресе `http://localhost:8000` ВК вернёт ошибку.
4. После создания перейди в **Настройки** → скопируй **ID приложения** и **Защищённый ключ**

### Настройка в settings.py

```python
# filmsite/settings.py
SOCIAL_AUTH_VK_OAUTH2_KEY = os.environ.get('VK_KEY', '')       # ID приложения
SOCIAL_AUTH_VK_OAUTH2_SECRET = os.environ.get('VK_SECRET', '') # Защищённый ключ

# Запрашиваем email при авторизации
SOCIAL_AUTH_VK_OAUTH2_SCOPE = ['email']
```

`SOCIAL_AUTH_VK_OAUTH2_SCOPE` — список прав, которые мы запрашиваем у пользователя при входе. ВКонтакте по умолчанию не передаёт email — его нужно явно запросить через `scope`. Без этого поле `email` у созданного пользователя окажется пустым.

Ссылка на полную документацию по бэкенду ВКонтакте для `python-social-auth`: [python-social-auth.readthedocs.io/en/latest/backends/vk.html](https://python-social-auth.readthedocs.io/en/latest/backends/vk.html)

Кнопка входа через ВКонтакте уже добавлена в шаблон выше — `{% url 'social:begin' 'vk-oauth2' %}` готова и заработает автоматически после деплоя.

---

## Общие настройки social-auth

Несколько параметров, которые стоит добавить в `settings.py` для корректной работы:

```python
# filmsite/settings.py

# Если пользователь с таким email уже существует — связать аккаунты
SOCIAL_AUTH_ASSOCIATE_BY_EMAIL = True

# Pipeline — последовательность шагов при OAuth-входе
# Стандартный pipeline подходит для большинства случаев
SOCIAL_AUTH_PIPELINE = (
    'social_core.pipeline.social_auth.social_details',
    'social_core.pipeline.social_auth.social_uid',
    'social_core.pipeline.social_auth.auth_allowed',
    'social_core.pipeline.social_auth.social_user',
    'social_core.pipeline.user.get_username',
    'social_core.pipeline.user.create_user',
    'social_core.pipeline.social_auth.associate_user',
    'social_core.pipeline.social_auth.load_extra_data',
    'social_core.pipeline.user.user_details',
)
```

**`SOCIAL_AUTH_ASSOCIATE_BY_EMAIL = True`** — важная настройка: если пользователь уже зарегистрирован через форму с тем же email, что и его GitHub/ВК аккаунт, Django свяжет эти аккаунты вместо создания дубля.

**Pipeline** — это цепочка функций, которые выполняются при каждом OAuth-входе. Стандартный pipeline: получить данные от провайдера → найти или создать пользователя → связать OAuth-профиль с Django-пользователем → обновить детали. При необходимости в pipeline можно вставить собственный шаг — например, автоматическое создание `UserProfile` при первом входе через соцсеть.

### Автоматическое создание UserProfile при OAuth-входе

В уроке 26 мы настроили автосоздание `UserProfile` через `RegisterView`. Но при входе через OAuth `RegisterView` не вызывается — пользователь создаётся внутри pipeline. Добавим свой шаг:

```python
# films/pipeline.py
from .models import UserProfile


def create_user_profile(backend, user, response, *args, **kwargs):
    """Создаёт UserProfile при первом входе через OAuth, если его нет."""
    UserProfile.objects.get_or_create(user=user)
```

Добавляем в конец pipeline:

```python
SOCIAL_AUTH_PIPELINE = (
    'social_core.pipeline.social_auth.social_details',
    'social_core.pipeline.social_auth.social_uid',
    'social_core.pipeline.social_auth.auth_allowed',
    'social_core.pipeline.social_auth.social_user',
    'social_core.pipeline.user.get_username',
    'social_core.pipeline.user.create_user',
    'social_core.pipeline.social_auth.associate_user',
    'social_core.pipeline.social_auth.load_extra_data',
    'social_core.pipeline.user.user_details',
    'films.pipeline.create_user_profile',           # наш шаг — последним
)
```

---

## Итоговая конфигурация settings.py

```python
# filmsite/settings.py

INSTALLED_APPS = [
    # ... существующие приложения ...
    'social_django',
]

AUTHENTICATION_BACKENDS = [
    'social_core.backends.github.GithubOAuth2',
    'social_core.backends.vk.VKOAuth2',
    'films.backends.EmailAuthBackend',
    'django.contrib.auth.backends.ModelBackend',
]

# GitHub OAuth (работает на localhost)
SOCIAL_AUTH_GITHUB_KEY = os.environ.get('GITHUB_KEY', '')
SOCIAL_AUTH_GITHUB_SECRET = os.environ.get('GITHUB_SECRET', '')
SOCIAL_AUTH_GITHUB_SCOPE = ['user:email']

# ВКонтакте OAuth (требует реальный домен)
SOCIAL_AUTH_VK_OAUTH2_KEY = os.environ.get('VK_KEY', '')
SOCIAL_AUTH_VK_OAUTH2_SECRET = os.environ.get('VK_SECRET', '')
SOCIAL_AUTH_VK_OAUTH2_SCOPE = ['email']

SOCIAL_AUTH_ASSOCIATE_BY_EMAIL = True

SOCIAL_AUTH_PIPELINE = (
    'social_core.pipeline.social_auth.social_details',
    'social_core.pipeline.social_auth.social_uid',
    'social_core.pipeline.social_auth.auth_allowed',
    'social_core.pipeline.social_auth.social_user',
    'social_core.pipeline.user.get_username',
    'social_core.pipeline.user.create_user',
    'social_core.pipeline.social_auth.associate_user',
    'social_core.pipeline.social_auth.load_extra_data',
    'social_core.pipeline.user.user_details',
    'films.pipeline.create_user_profile',
)
```

---

## Подводные камни

### ВКонтакте и localhost

Ещё раз явно: ВК отклоняет redirect URI с `localhost` или `127.0.0.1`. Даже указав в настройках приложения ВК адрес `http://localhost:8000`, при попытке авторизации получишь ошибку `redirect_uri is incorrect`. Это ограничение платформы, не наш код. Единственные варианты для локального тестирования — ngrok или развёрнутый сервер с доменом.

### Кнопка «ВКонтакте» до деплоя

Ссылка `{% url 'social:begin' 'vk-oauth2' %}` будет генерироваться корректно и до деплоя — кнопка будет видна и кликабельна. Но переход по ней завершится ошибкой со стороны ВКонтакте (`redirect_uri is incorrect`). Это не ошибка в нашем коде — просто ограничение платформы.

### Дубли пользователей без SOCIAL_AUTH_ASSOCIATE_BY_EMAIL

Без `SOCIAL_AUTH_ASSOCIATE_BY_EMAIL = True` пользователь, зарегистрировавшийся через форму с email `ivan@example.com`, а потом вошедший через GitHub с тем же email, получит **второй** Django-аккаунт. Включённая настройка объединяет их. Но стоит знать: это создаёт потенциальную уязвимость — злоумышленник теоретически может захватить аккаунт, создав OAuth-профиль с чужим email. В реальных проектах вместе с этой настройкой часто включают подтверждение email при регистрации.

### ClientId в HTML-исходнике

Ссылка `{% url 'social:begin' 'github' %}` не содержит `client_id` в явном виде — это просто URL нашего сервера, который уже знает `client_id` из `settings.py`. Так и должно быть: `client_id` не является секретом (он виден в redirect URL), но `client_secret` никогда не должен попадать во фронтенд.

---

## Вопросы для проверки

1. Зачем нужна библиотека `python-social-auth` при реализации OAuth 2.0 в Django?
2. Почему авторизация ВКонтакте не работает на `localhost` и как это обойти?
3. Что такое pipeline в `python-social-auth` и зачем он нужен?
4. Зачем нужна настройка `SOCIAL_AUTH_ASSOCIATE_BY_EMAIL = True` и какой риск она несёт?
5. Зачем нужен параметр `SOCIAL_AUTH_VK_OAUTH2_SCOPE = ['email']`?

---

## Практическая задача

**Тип: расширь проект**

**Часть 1 (выполняется сейчас — GitHub).** Создай OAuth-приложение на GitHub, добавь настройки в `settings.py` через переменные окружения. Добавь кнопки «GitHub» и «ВКонтакте» на страницу входа. Проверь, что авторизация через GitHub работает локально — после входа пользователь должен появиться в разделе «Social Auth» в административной панели.

**Часть 2 (выполняется после деплоя — ВКонтакте).** Создай приложение на [dev.vk.com](https://dev.vk.com), укажи реальный домен сайта, сохрани ключи в переменных окружения на сервере. После деплоя в модуле 9 проверь авторизацию через ВКонтакте.

**Для обеих частей:** добавь файл `films/pipeline.py` с функцией `create_user_profile` и включи её в `SOCIAL_AUTH_PIPELINE`. Убедись, что при первом OAuth-входе для пользователя автоматически создаётся `UserProfile`.

---

[Предыдущий урок](lesson28.md) | [Следующий урок](lesson30.md)