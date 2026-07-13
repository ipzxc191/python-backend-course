# Урок 24. Регистрация. UserCreationForm. Авторизация через email

## Вход есть — теперь нужна регистрация

В прошлом уроке мы научили сайт принимать вошедших пользователей. Но пока единственный способ создать аккаунт — через `createsuperuser` в терминале. Сегодня добавим публичную регистрацию и расширим стандартную форму входа — чтобы пользователь мог войти не только по username, но и по email.

---

## UserCreationForm — встроенная форма регистрации

`django.contrib.auth.forms.UserCreationForm` — это готовая `ModelForm` для создания нового пользователя. Она содержит три поля:

- `username` — имя пользователя
- `password1` — пароль
- `password2` — подтверждение пароля

Форма автоматически проверяет, что оба пароля совпадают, и валидирует пароль по встроенным правилам Django (минимальная длина, не только цифры, не слишком простой).

### Простейшая регистрация через UserCreationForm

```python
# films/views.py
from django.contrib.auth.forms import UserCreationForm
from django.contrib.auth import login


class RegisterView(CreateView):
    form_class = UserCreationForm
    template_name = 'registration/register.html'
    success_url = reverse_lazy('films:index')

    def form_valid(self, form):
        response = super().form_valid(form)
        # Автоматически входим после регистрации
        login(self.request, self.object)
        return response
```

Функция `login(request, user)` из `django.contrib.auth` создаёт сессию и «входит» пользователем — после этого `request.user` будет ссылаться на зарегистрированного пользователя.

```html
<!-- templates/registration/register.html -->
{% extends 'base.html' %}

{% block title %}Регистрация{% endblock %}

{% block content %}
    <h1>Создать аккаунт</h1>

    <form method="POST">
        {% csrf_token %}
        {{ form.as_p }}
        <button type="submit">Зарегистрироваться</button>
    </form>

    <p>Уже есть аккаунт? <a href="{% url 'login' %}">Войти</a></p>
{% endblock %}
```

Добавляем маршрут — он идёт в корневой `urls.py`, так как не привязан к приложению `films`:

```python
# filmsite/urls.py
from films.views import RegisterView

urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/', include('django.contrib.auth.urls')),
    path('accounts/register/', RegisterView.as_view(), name='register'),
    path('', include('films.urls')),
]
```

---

## Расширяем форму регистрации

Стандартный `UserCreationForm` содержит только три поля — без email, имени, фамилии. Для нашего сайта добавим email как обязательное поле:

```python
# films/forms.py
from django.contrib.auth.forms import UserCreationForm
from django.contrib.auth.models import User


class CustomRegistrationForm(UserCreationForm):
    email = forms.EmailField(
        required=True,
        label='Email',
        help_text='Обязательное поле. Введите действующий адрес.'
    )

    class Meta:
        model = User
        fields = ('username', 'email', 'password1', 'password2')

    def clean_email(self):
        email = self.cleaned_data['email']
        if User.objects.filter(email=email).exists():
            raise forms.ValidationError('Пользователь с таким email уже зарегистрирован.')
        return email

    def save(self, commit=True):
        user = super().save(commit=False)
        user.email = self.cleaned_data['email']
        if commit:
            user.save()
        return user
```

Несколько важных деталей:

`clean_email()` — проверяем уникальность email. Стандартный `User` не имеет `unique=True` на поле `email`, поэтому Django не защитит от дублей автоматически. Проверяем вручную через `filter().exists()`.

`save(commit=False)` — переопределяем метод сохранения, чтобы явно сохранить email. Django не делает этого автоматически, даже если поле включено в `Meta.fields`, — потому что `email` не является обязательным полем модели `User`.

При стандартной отрисовке в форме с помощью `{{ form.as_p }}` у нас будут прописываться все подсказки, например:

```
...
Пароль не должен быть слишком похож на другую вашу личную информацию.
Ваш пароль должен содержать как минимум 8 символов.
Пароль не должен быть слишком простым и распространенным.
Пароль не может состоять только из цифр.
...
```

Что бы это избежать, можно некоторые подсказки убрать вообще, и добавить в инпуты placeholder'ы. Для это можно переорделелить `__init__` для `CustomRegistrationForm`:

```python
def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)

        # Убираем подсказки
        self.fields['username'].help_text = None
        self.fields['password1'].help_text = None
        self.fields['password2'].help_text = None

        # Красивые placeholder'ы
        self.fields['username'].widget.attrs.update({
            'placeholder': 'Введите имя пользователя'
        })

        self.fields['email'].widget.attrs.update({
            'placeholder': 'Введите email'
        })

        self.fields['password1'].widget.attrs.update({
            'placeholder': 'Введите пароль'
        })

        self.fields['password2'].widget.attrs.update({
            'placeholder': 'Повторите пароль'
        })
```

Добавить после класса Meta и перед переопределенными функциями.

---

Обновляем `RegisterView`:

```python
class RegisterView(CreateView):
    form_class = CustomRegistrationForm
    template_name = 'registration/register.html'
    success_url = reverse_lazy('films:index')

    def form_valid(self, form):
        response = super().form_valid(form)
        login(self.request, self.object)
        return response
```

---

## Авторизация через email

По умолчанию Django принимает вход только по `username`. Но пользователи привыкли входить по email — особенно на современных сайтах. Реализуем это через кастомный бэкенд аутентификации.

### Что такое Authentication Backend

`django.contrib.auth` делегирует проверку учётных данных через набор «бэкендов». По умолчанию используется `ModelBackend`, который ищет пользователя по `username`. Можно написать собственный бэкенд, добавить его в `settings.py`, и Django будет пробовать каждый по очереди — пока какой-нибудь не аутентифицирует пользователя.

### Пишем EmailAuthBackend

```python
# films/backends.py
from django.contrib.auth.models import User
from django.contrib.auth.backends import ModelBackend


class EmailAuthBackend(ModelBackend):
    """
    Аутентификация по email вместо username.
    Наследуем ModelBackend, чтобы переиспользовать логику проверки пароля
    и разрешений — переопределяем только метод поиска пользователя.
    """

    def authenticate(self, request, username=None, password=None, **kwargs):
        # username здесь — это то, что ввёл пользователь в поле «Имя пользователя»
        # мы трактуем это значение как email
        try:
            user = User.objects.get(email=username)
        except User.DoesNotExist:
            return None
        except User.MultipleObjectsReturned:
            # Если несколько пользователей с одинаковым email — берём первого
            # В реальном проекте стоит логировать такую ситуацию
            user = User.objects.filter(email=username).first()

        if user.check_password(password) and self.user_can_authenticate(user):
            return user
        return None
```

`user.check_password(password)` — встроенный метод, который проверяет, соответствует ли пароль сохранённому хэшу. Никогда не сравнивай пароли напрямую через `==`.

`self.user_can_authenticate(user)` — проверяет `user.is_active`. Это важно: деактивированные пользователи не должны входить, даже если пароль верный.

### Подключение в settings.py

```python
# filmsite/settings.py
AUTHENTICATION_BACKENDS = [
    'films.backends.EmailAuthBackend',      # сначала пробуем по email
    'django.contrib.auth.backends.ModelBackend',  # потом по username
]
```

Django пробует бэкенды по порядку. Если `EmailAuthBackend` вернёт `None` (email не найден) — Django попробует стандартный `ModelBackend` (вход по username). Это значит, что сайт поддерживает **оба** способа входа одновременно — и по email, и по username.

### Обновляем форму входа для подсказки пользователю

Стандартная форма `LoginView` показывает поле с лейблом «Имя пользователя». Переименуем его, чтобы пользователь понимал, что можно ввести email:

```python
# films/forms.py
from django.contrib.auth.forms import AuthenticationForm


class CustomAuthenticationForm(AuthenticationForm):
    username = forms.CharField(
        label='Имя пользователя или Email',
        widget=forms.TextInput(attrs={'autofocus': True}),
    )
```

Подключаем форму к `LoginView` через атрибут `authentication_form` — для этого нужно переопределить `LoginView` в нашем `urls.py` или создать собственный подкласс:

```python
# filmsite/urls.py
from django.contrib.auth.views import LoginView
from films.forms import CustomAuthenticationForm

urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/login/', LoginView.as_view(authentication_form=CustomAuthenticationForm), name='login'),
    path('accounts/', include('django.contrib.auth.urls')),
    path('accounts/register/', RegisterView.as_view(), name='register'),
    path('', include('films.urls')),
]
```

> Обрати внимание: маршрут `/accounts/login/` теперь прописан явно перед `include('django.contrib.auth.urls')`. Django перебирает маршруты сверху вниз и остановится на первом совпадении, поэтому наш кастомный `login` переопределит стандартный из пакета.

---

Обновляем `RegisterView`:

```python
from django.contrib.auth import authenticate, login


class RegisterView(CreateView):
    form_class = CustomRegistrationForm
    template_name = 'registration/register.html'
    success_url = reverse_lazy('films:index')

    def form_valid(self, form):
        response = super().form_valid(form)
        
        # аутентифицируем пользователя явно
        user = authenticate(
            self.request,
            username=form.cleaned_data['email'],
            password=form.cleaned_data['password1'],
        )

        if user is not None:
            login(self.request, user)
        return response
```

При такой реализации пользователь после регистрации сразу становится залогиненым. Без дополнительного использование `authenticate` у нас могла возникнуть ошибка. Django должен явно знать, с помощью какого бэкэнда авторизации авторизовался пользователь.

## Обновляем ссылку на регистрацию в шаблоне входа

В уроке 23 мы добавили в `registration/login.html` ссылку `{% url 'register' %}`. Теперь этот маршрут реально существует — ссылка заработает.

```html
<p>Ещё нет аккаунта? <a href="{% url 'register' %}">Зарегистрироваться</a></p>
```

---

## Подводные камни

### save() в CustomRegistrationForm — обязателен

`UserCreationForm` сохраняет пользователя с хэшированным паролем через специальный механизм (`set_password()`). Если переопределить `save()` и забыть вызвать `super().save(commit=False)` — пользователь сохранится с незахэшированным паролем в виде открытого текста:

```python
# Пароль сохранится в открытом виде — критическая уязвимость
def save(self, commit=True):
    user = User(username=self.cleaned_data['username'])
    user.password = self.cleaned_data['password1']
    user.email = self.cleaned_data['email']
    user.save()
    return user

# super().save() вызывает set_password() — пароль хэшируется
def save(self, commit=True):
    user = super().save(commit=False)
    user.email = self.cleaned_data['email']
    if commit:
        user.save()
    return user
```

### MultipleObjectsReturned в EmailAuthBackend

Поле `email` в стандартной модели `User` не уникально — в базе могут быть несколько пользователей с одинаковым email (если проверку не добавили в форму регистрации или обошли её программно). `User.objects.get(email=...)` выбросит `MultipleObjectsReturned`. Обрабатываем явно, как показано выше.

### Порядок AUTHENTICATION_BACKENDS влияет на производительность

Django проверяет бэкенды по порядку. Если `EmailAuthBackend` стоит первым, а большинство пользователей входит по username — каждый вход будет делать дополнительный запрос к БД (поиск по email) перед тем, как попасть в `ModelBackend`. Для небольшого учебного проекта не критично, но для высоконагруженных систем порядок имеет значение.

---

## Итоговые изменения

```python
# filmsite/settings.py — добавляем
LOGIN_REDIRECT_URL = '/'
LOGOUT_REDIRECT_URL = '/'
LOGIN_URL = '/accounts/login/'

AUTHENTICATION_BACKENDS = [
    'films.backends.EmailAuthBackend',
    'django.contrib.auth.backends.ModelBackend',
]
```

```python
# filmsite/urls.py
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static
from django.contrib.auth.views import LoginView
from films.views import RegisterView
from films.forms import CustomAuthenticationForm

admin.site.site_header = 'Управление каталогом фильмов'
admin.site.site_title = 'Админка — Сайт фильмов'
admin.site.index_title = 'Панель администратора'

urlpatterns = [
    path('admin/', admin.site.urls),
    path('accounts/login/', LoginView.as_view(authentication_form=CustomAuthenticationForm), name='login'),
    path('accounts/', include('django.contrib.auth.urls')),
    path('accounts/register/', RegisterView.as_view(), name='register'),
    path('', include('films.urls')),
]

if settings.DEBUG:
    import debug_toolbar
    urlpatterns += [path('__debug__/', include(debug_toolbar.urls))]
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

---

## Вопросы для проверки

1. Зачем переопределять `save()` в `CustomRegistrationForm`, если `email` уже добавлен в `Meta.fields`?
2. Что такое Authentication Backend и зачем нужен кастомный?
3. Почему нельзя сравнивать пароли напрямую через `==` и нужно использовать `user.check_password()`?
4. Почему явный маршрут для `/accounts/login/` нужно ставить перед `include('django.contrib.auth.urls')`?
5. Что происходит, если оба бэкенда в `AUTHENTICATION_BACKENDS` не могут аутентифицировать пользователя?

---

## Практическая задача

**Тип: расширь проект**

Расширь форму регистрации, добавив поля имени и фамилии.

**Требования:**

1. Добавь в `CustomRegistrationForm` поля `first_name` и `last_name` — оба необязательные
2. Обнови `Meta.fields`, включив новые поля в нужном порядке: `username`, `first_name`, `last_name`, `email`, `password1`, `password2`
3. Обнови метод `save()` — явно сохраняй `first_name` и `last_name` в объект пользователя до `user.save()`
4. Убедись, что `clean_email()` по-прежнему проверяет уникальность email

---

[Предыдущий урок](lesson23.md) | [Следующий урок](lesson25.md)