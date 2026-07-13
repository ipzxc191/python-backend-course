# Урок 23. Система авторизации Django. LoginView, LogoutView, login_required, LoginRequiredMixin

## Что даёт встроенная система авторизации

Пять модулей мы строили сайт, где любой пользователь может делать всё: добавлять фильмы, редактировать, удалять, оставлять рецензии. В реальном приложении это неприемлемо — разные действия доступны разным пользователям.

Django поставляется с полноценной системой авторизации: пользователи, пароли, сессии, разрешения, группы. Всё это входит в `django.contrib.auth`, который уже давно подключён в `INSTALLED_APPS` с самого первого урока и создал свои таблицы при первом `migrate`. Мы просто ещё не использовали это.

Сегодня разберём первую половину модуля: вход и выход из системы, защита представлений через декоратор и Mixin.

---

## Встроенная модель User

`django.contrib.auth.models.User` — это таблица пользователей, которую Django создал при первом `migrate`. Суперпользователь, которого мы создали в уроке 13 через `createsuperuser` — это уже объект этой модели.

Основные поля:

| Поле | Назначение |
|---|---|
| `id` | Первичный ключ |
| `username` | Уникальное имя пользователя |
| `email` | Электронная почта |
| `password` | Хэш пароля (не сам пароль) |
| `first_name`, `last_name` | Имя и фамилия |
| `is_active` | Активен ли аккаунт |
| `is_staff` | Есть ли доступ к `/admin/` |
| `is_superuser` | Суперпользователь — все права без явного назначения |
| `date_joined` | Дата регистрации |
| `last_login` | Дата последнего входа |

Важно: Django никогда не хранит пароль в открытом виде — только его хэш через PBKDF2 (по умолчанию). При проверке пароля Django хэширует введённое значение и сравнивает хэши.

### request.user

В каждом представлении текущий пользователь доступен через `request.user`. Это либо объект `User`, либо `AnonymousUser`, если пользователь не вошёл в систему:

```python
def some_view(request):
    if request.user.is_authenticated:
        print(request.user.username)  # имя авторизованного пользователя
    else:
        print('Анонимный пользователь')
```

`is_authenticated` — это атрибут, не метод. Для объекта `User` он всегда `True`, для `AnonymousUser` — всегда `False`.

---

## Настройка URL-адресов авторизации

Django предоставляет готовые представления для входа и выхода. Их можно подключить одной строкой:

```python
# filmsite/urls.py
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('django.contrib.auth.urls')),  # все маршруты авторизации
    path('', include('films.urls')),
]
```

`django.contrib.auth.urls` добавляет следующие маршруты:

| Маршрут | Имя | Назначение |
|---|---|---|
| `accounts/login/` | `login` | Форма входа |
| `accounts/logout/` | `logout` | Выход |
| `accounts/password_change/` | `password_change` | Смена пароля |
| `accounts/password_change/done/` | `password_change_done` | Подтверждение смены |
| `accounts/password_reset/` | `password_reset` | Восстановление пароля |
| `accounts/password_reset/done/` | `password_reset_done` | Письмо отправлено |
| `accounts/reset/<uidb64>/<token>/` | `password_reset_confirm` | Форма нового пароля |
| `accounts/reset/done/` | `password_reset_complete` | Пароль изменён |

Сегодня разберём `login` и `logout`, остальные — в уроках 24 и 25.

---

## Настройки в settings.py

```python
# filmsite/settings.py

# Куда перенаправлять после успешного входа
LOGIN_REDIRECT_URL = '/'  # на главную страницу

# Куда перенаправлять после выхода
LOGOUT_REDIRECT_URL = '/'

# Куда перенаправлять неавторизованных пользователей
# (используется login_required и LoginRequiredMixin)
LOGIN_URL = '/accounts/login/'
```

---

## LoginView — форма входа

`LoginView` из `django.contrib.auth.views` — это готовое CBV-представление, которое показывает форму входа (GET) и проверяет учётные данные (POST). При успешном входе перенаправляет на `LOGIN_REDIRECT_URL` или на страницу, которую пользователь пытался открыть до перенаправления (параметр `next`).

`LoginView` уже подключён через `django.contrib.auth.urls`. Осталось только создать шаблон — Django будет искать его в папке `registration/`:

```
templates/
└── registration/
    └── login.html
```

```html
<!-- templates/registration/login.html -->
{% extends 'base.html' %}

{% block title %}Вход — Сайт фильмов{% endblock %}

{% block content %}
    <h1>Вход на сайт</h1>

    {% if form.errors %}
        <div class="error">
            <p>Неверное имя пользователя или пароль. Попробуйте снова.</p>
        </div>
    {% endif %}

    <form method="POST">
        {% csrf_token %}
        {{ form.as_p }}
        <input type="hidden" name="next" value="{{ next }}">
        <button type="submit">Войти</button>
    </form>
{% endblock %}
```

Скрытое поле `next` сохраняет адрес, куда пользователь должен вернуться после входа. Если пользователь пытался открыть `/films/add/`, но был перенаправлен на страницу входа, `next` сохраняет `/films/add/`, и `LoginView` вернёт пользователя туда после успешной авторизации.

---

## LogoutView — выход

Выход реализован так же просто. Поскольку выход — это действие, изменяющее состояние (завершает сессию), оно должно выполняться через POST, а не через GET-ссылку:

```html
<!-- В шапке сайта — например, в base.html -->
{% if user.is_authenticated %}
    <span>{{ user.username }}</span>
    <form method="POST" action="{% url 'logout' %}" style="display: inline;">
        {% csrf_token %}
        <button type="submit">Выйти</button>
    </form>
{% else %}
    <a href="{% url 'login' %}">Войти</a>
{% endif %}
```

> Django 5.x требует POST для выхода — GET-запрос на `/accounts/logout/` больше не завершает сессию. Это изменение было введено в Django 5.0 по соображениям безопасности (защита от CSRF).

Переменная `user` доступна в шаблонах без явной передачи в контекст — её добавляет контекстный процессор `django.contrib.auth.context_processors.auth`, который уже подключён в `TEMPLATES` → `'OPTIONS'` → `'context_processors'` в `settings.py`.

---

## login_required — защита FBV

Декоратор `login_required` проверяет, авторизован ли пользователь, перед вызовом функции. Если нет — перенаправляет на `LOGIN_URL`:

```python
# films/views.py
from django.contrib.auth.decorators import login_required


@login_required
def add_film(request):
    # Этот код выполнится только для авторизованных пользователей
    ...


@login_required
def add_review(request, slug):
    ...
```

Применять декоратор можно и в `urls.py`, не меняя сам файл представлений:

```python
# films/urls.py
from django.contrib.auth.decorators import login_required
from . import views

urlpatterns = [
    path('films/add/', login_required(views.add_film), name='add_film'),
]
```

### Параметры login_required

```python
# Указать свой URL входа
@login_required(login_url='/my-login/')

# Указать redirect_field_name (по умолчанию 'next')
@login_required(redirect_field_name='redirect_to')
```

---

## LoginRequiredMixin — защита CBV

Для классовых представлений используется `LoginRequiredMixin`, который мы уже подключали в прошлом уроке для `FilmCreateView` и других представлений. Теперь он будет работать по-настоящему — потому что мы настроили `LOGIN_URL`:

```python
# films/views.py
from django.contrib.auth.mixins import LoginRequiredMixin


class FilmCreateView(LoginRequiredMixin, CreateView):
    model = Film
    form_class = FilmForm
    template_name = 'films/add_film.html'


class FilmUpdateView(LoginRequiredMixin, FilmEditMixin, UpdateView):
    context_object_name = 'film'

    def get_success_url(self):
        return self.object.get_absolute_url()


class FilmDeleteView(LoginRequiredMixin, PermissionRequiredMixin, DeleteView):
    model = Film
    permission_required = 'films.delete_film'
    template_name = 'films/film_confirm_delete.html'
    success_url = reverse_lazy('films:film_list')
```

Как работает `LoginRequiredMixin`: он переопределяет метод `dispatch()` — точку входа, которую `View.as_view()` вызывает для каждого запроса. До того как запрос попадёт в `get()` или `post()`, `dispatch()` проверяет `request.user.is_authenticated`. Если нет — делает редирект. Именно поэтому Mixin должен стоять первым в списке наследования.

---

## Обновляем шапку сайта

Обновим `base.html`, чтобы навигация учитывала статус авторизации:

```html
<!-- templates/base.html -->
{% load static %}
{% load film_tags %}
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}Сайт фильмов{% endblock %}</title>
    <link rel="stylesheet" href="{% static 'css/base.css' %}">
    {% block extra_css %}{% endblock %}
</head>
<body>
    <header>
        <nav>
            <a href="{% url 'films:index' %}">Главная</a> |
            <a href="{% url 'films:film_list' %}">Каталог</a> |
            <a href="{% url 'films:about' %}">О сайте</a>

            <span class="nav-auth">
                {% if user.is_authenticated %}
                    | <a href="{% url 'films:add_film' %}">Добавить фильм</a>
                    | {{ user.username }}
                    <form method="POST" action="{% url 'logout' %}" style="display: inline;">
                        {% csrf_token %}
                        <button type="submit" class="nav-link-btn">Выйти</button>
                    </form>
                {% else %}
                    | <a href="{% url 'login' %}">Войти</a>
                {% endif %}
            </span>
        </nav>
    </header>

    <main>
        {% block content %}{% endblock %}
    </main>

    {% latest_films 3 %}

    <footer>
        <p>© {% current_year %} Сайт фильмов</p>
    </footer>

    {% block extra_js %}{% endblock %}
</body>
</html>
```

---

## Подводные камни

### GET-запрос для logout в Django 5

В Django до версии 5.0 выход можно было выполнить простой ссылкой `<a href="{% url 'logout' %}">`. Начиная с Django 5.0 это не работает — LogoutView принимает только POST. Если на проекте стоит Django 5.x, а в шаблоне осталась GET-ссылка для выхода — пользователи не смогут выйти без видимой ошибки (их просто перенаправят на главную без завершения сессии).

### login_required и CBV

Нельзя применить `@login_required` непосредственно к классу CBV — только к результату `as_view()`:

```python
# Неправильно — декоратор применяется к классу, а не к функции
@login_required
class FilmCreateView(CreateView):
    ...

# Через Mixin — рекомендуемый способ для CBV
class FilmCreateView(LoginRequiredMixin, CreateView):
    ...

# Через as_view() в urls.py — работает, но Mixin удобнее
path('films/add/', login_required(FilmCreateView.as_view()), name='add_film'),
```

### next после входа

Если неавторизованный пользователь пытается открыть `/films/add/`, Django перенаправит его на `/accounts/login/?next=/films/add/`. После успешного входа `LoginView` прочитает параметр `next` и вернёт пользователя на `/films/add/`. Но если в форме входа не передать `<input type="hidden" name="next" value="{{ next }}">` — значение `next` потеряется при отправке POST-запроса. Шаблон `registration/login.html` обязательно должен содержать это скрытое поле.

---

## Вопросы для проверки

1. Что возвращает `request.user` для неавторизованного пользователя, и как проверить статус авторизации?
2. Чем отличается `login_required` от `LoginRequiredMixin` и когда применять каждый?
3. Почему выход из системы в Django 5.x должен выполняться через POST-форму, а не через ссылку?
4. Для чего служит скрытое поле `<input type="hidden" name="next">` в шаблоне формы входа?
5. Откуда переменная `user` берётся в шаблонах без явной передачи в контекст?

---

## Практическая задача

**Тип: расширь проект**

Защити операции записи в каталоге и обнови навигацию.

**Требования:**

1. Добавь `LoginRequiredMixin` ко всем CBV, которые изменяют данные: `FilmCreateView`, `FilmUpdateView`, `FilmDeleteView`, `AddReviewView`, `DirectorCreateView`, `DirectorUpdateView`, `DirectorDeleteView`
2. Для FBV `search_film` и `catalog_stats` защита не нужна — они только читают данные
3. Обнови `base.html`: кнопки «Добавить фильм» и «Выйти» показываются только авторизованным пользователям; неавторизованным — ссылка «Войти»
4. Создай шаблон `templates/registration/login.html` со скрытым полем `next`
5. Проверь: открой `/films/add/` без входа — должно перенаправить на страницу входа; после входа — вернуть обратно на `/films/add/`

---

[Предыдущий урок](lesson22.md) | [Следующий урок](lesson24.md)