# Урок 26. AbstractUser. Профиль пользователя. Контекстные процессоры

## Три связанные темы одного урока

Стандартная модель `User` покрывает базовые поля — имя, email, пароль. Но реальные проекты почти всегда требуют большего: аватар, биография, город, дата рождения. Есть два пути расширить пользователя в Django, и мы разберём оба. Третья тема — контекстные процессоры — логично появляется здесь: когда данные профиля нужны на каждой странице, правильное место для них именно там.

---

## Способ 1: OneToOneField — профиль отдельной таблицей

Этот подход мы уже анонсировали в схеме БД в самом начале курса. Создаём модель `UserProfile`, связанную с `User` через `OneToOneField`:

```python
# films/models.py
from django.contrib.auth.models import User


class UserProfile(models.Model):
    user = models.OneToOneField(
        User,
        on_delete=models.CASCADE,
        related_name='profile',
        verbose_name='Пользователь'
    )
    avatar = models.ImageField(
        upload_to='avatars/',
        blank=True,
        verbose_name='Аватар'
    )
    bio = models.TextField(blank=True, verbose_name='О себе')

    class Meta:
        verbose_name = 'Профиль пользователя'
        verbose_name_plural = 'Профили пользователей'

    def __str__(self):
        return f'Профиль: {self.user.username}'
```

Создаём и применяем миграцию:

```bash
python manage.py makemigrations
python manage.py migrate
```

### Автоматическое создание профиля при регистрации

Профиль должен создаваться вместе с пользователем — обновим `RegisterView`:

```python
# films/views.py
from .models import UserProfile


class RegisterView(CreateView):
    form_class = CustomRegistrationForm
    template_name = 'registration/register.html'
    success_url = reverse_lazy('films:index')

    def form_valid(self, form):
        response = super().form_valid(form)
        
        UserProfile.objects.create(user=self.object)  # создаем профиль пользователя
        
        user = authenticate(
            self.request,
            username=form.cleaned_data['email'],
            password=form.cleaned_data['password1'],
        )

        if user is not None:
            login(self.request, user)
        return response
```

### Доступ к профилю

```python
# В представлении
profile = request.user.profile  # через related_name

# В шаблоне
{{ request.user.profile.bio }}
{{ request.user.profile.avatar.url }}
```

### Когда выбирать этот подход

`OneToOneField` хорош, когда дополнительные данные нужны не всегда — например, заполнение профиля опционально. Таблица `User` остаётся чистой, профиль подключается по требованию. Минус: при каждом обращении к данным профиля нужен дополнительный JOIN или обращение к связанному объекту.

---

## Способ 2: AbstractUser — расширение самой модели User

`AbstractUser` — это базовый класс, от которого наследуется стандартная модель `User`. Если унаследовать от него самому — получим полноценную кастомную модель пользователя со всеми стандартными полями плюс собственными.

### Важное ограничение

`AbstractUser` нужно подключать **до первой миграции** проекта. Если базу данных уже применили — смена модели пользователя требует либо сброса БД, либо сложных манипуляций. Поэтому Django-разработчики рекомендуют всегда создавать кастомную модель пользователя с самого начала нового проекта — даже если сейчас лишних полей нет.

> В нашем проекте мы уже применили миграции — поэтому разберём `AbstractUser` как концепцию и покажем, как это делается в новых проектах. Для текущего проекта используем подход с `UserProfile`.

### Как выглядит AbstractUser в новом проекте

```python
# users/models.py (в новом проекте — до первой миграции)
from django.contrib.auth.models import AbstractUser
from django.db import models


class User(AbstractUser):
    avatar = models.ImageField(upload_to='avatars/', blank=True)
    bio = models.TextField(blank=True)

    class Meta:
        verbose_name = 'Пользователь'
        verbose_name_plural = 'Пользователи'
```

```python
# settings.py
AUTH_USER_MODEL = 'users.User'
```

```bash
python manage.py makemigrations
python manage.py migrate
```

После этого `request.user` будет объектом кастомной модели, и `request.user.avatar` работает напрямую — без `.profile`.

### Чем AbstractUser отличается от AbstractBaseUser

| | `AbstractUser` | `AbstractBaseUser` |
|---|---|---|
| Что даёт | Все стандартные поля (`username`, `email`, `is_staff` и т.д.) + возможность добавить свои | Только базовый механизм аутентификации (пароль, last_login) |
| Когда использовать | Нужно добавить поля, сохранив стандартную структуру | Нужна полностью кастомная логика входа (например, без username) |
| Сложность | Простой | Требует описать все поля и менеджер вручную |

Для большинства проектов `AbstractUser` — правильный выбор. `AbstractBaseUser` — для нестандартных случаев, например когда username не нужен вовсе, а вход только по email и номеру телефона.

---

## Форма редактирования профиля

Добавим форму для страницы профиля — редактирование аватара и биографии:

```python
# films/forms.py
from .models import UserProfile
from django.contrib.auth.models import User


class UserUpdateForm(forms.ModelForm):
    """Редактирование базовых полей пользователя."""
    class Meta:
        model = User
        fields = ['first_name', 'last_name', 'email']


class ProfileUpdateForm(forms.ModelForm):
    """Редактирование профиля."""
    class Meta:
        model = UserProfile
        fields = ['avatar', 'bio']
        widgets = {
            'bio': forms.Textarea(attrs={'rows': 4}),
        }
```

Две формы на одной странице — стандартная ситуация для страниц профиля: одна форма редактирует `User`, другая — `UserProfile`.

### Представление профиля

```python
# films/views.py
from django.contrib.auth.mixins import LoginRequiredMixin
from django.views.generic import View
from .forms import UserUpdateForm, ProfileUpdateForm


class ProfileView(LoginRequiredMixin, View):
    template_name = 'films/profile.html'

    def get(self, request):
        user_form = UserUpdateForm(instance=request.user)
        profile_form = ProfileUpdateForm(instance=request.user.profile)
        return render(request, self.template_name, {
            'user_form': user_form,
            'profile_form': profile_form,
        })

    def post(self, request):
        user_form = UserUpdateForm(request.POST, instance=request.user)
        profile_form = ProfileUpdateForm(
            request.POST,
            request.FILES,
            instance=request.user.profile
        )
        if user_form.is_valid() and profile_form.is_valid():
            user_form.save()
            profile_form.save()
            return redirect('films:profile')

        return render(request, self.template_name, {
            'user_form': user_form,
            'profile_form': profile_form,
        })
```

Обрати внимание: `if user_form.is_valid() and profile_form.is_valid()` — обе формы валидируются вместе. Если хотя бы одна невалидна — ни одна не сохраняется, и обе возвращаются в шаблон с ошибками.

```html
<!-- films/templates/films/profile.html -->
{% extends 'base.html' %}

{% block title %}Профиль — {{ user.username }}{% endblock %}

{% block content %}
    <h1>Профиль: {{ user.username }}</h1>

    {% if user.profile.avatar %}
        <img src="{{ user.profile.avatar.url }}" alt="Аватар" width="150">
    {% endif %}

    <form method="POST" enctype="multipart/form-data">
        {% csrf_token %}

        <h2>Личные данные</h2>
        {{ user_form.as_p }}

        <h2>О себе</h2>
        {{ profile_form.as_p }}

        <button type="submit">Сохранить</button>
    </form>
{% endblock %}
```

```python
# films/urls.py
path('profile/', ProfileView.as_view(), name='profile'),
```

---

Пока не у всех пользователей есть профиль, поэтому переход по адресу `profile/` выдаст ошибку.

Что бы ее исправить обратитесь к [Подводные камни: RelatedObjectDoesNotExist при обращении к профилю](#relatedobjectdoesnotexist-при-обращении-к-профилю).

## Контекстные процессоры

В уроке 7 мы упоминали, что переменная `user` доступна во всех шаблонах без явной передачи — через контекстный процессор `auth`. Теперь разберём этот механизм и напишем собственный.

### Что такое контекстный процессор

Контекстный процессор — это функция, которая принимает объект `request` и возвращает словарь. Этот словарь автоматически добавляется в контекст **каждого шаблона** при каждом запросе — без явной передачи через `render()`.

Стандартные процессоры в `settings.py`:

```python
TEMPLATES = [
    {
        ...
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',    # request во всех шаблонах
                'django.contrib.auth.context_processors.auth',   # user и perms
                'django.contrib.messages.context_processors.messages',  # messages
            ],
        },
    },
]
```

### Пишем собственный контекстный процессор

Хороший пример для нашего проекта — статистика для шапки сайта: количество фильмов и жанров, которые показываются на каждой странице.

```python
# films/context_processors.py
from .models import Film, Genre


def catalog_stats(request):
    """
    Добавляет базовую статистику каталога в контекст каждого шаблона.
    Вызывается при каждом запросе — запросы должны быть лёгкими.
    """
    return {
        'total_films': Film.objects.count(),
        'total_genres': Genre.objects.count(),
    }
```

Регистрируем в `settings.py`:

```python
TEMPLATES = [
    {
        ...
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
                'films.context_processors.catalog_stats',   # наш процессор
            ],
        },
    },
]
```

Теперь в любом шаблоне доступны переменные `total_films` и `total_genres` без явной передачи:

```html
<!-- templates/base.html -->
<footer>
    <p>В каталоге {{ total_films }} фильмов и {{ total_genres }} жанров</p>
    <p>© {% current_year %} Сайт фильмов</p>
</footer>
```

### Контекстный процессор для профиля

Если данные профиля нужны в шапке каждой страницы (аватар пользователя, например), это тоже хорошее место для процессора:

```python
# films/context_processors.py
def user_profile(request):
    """
    Добавляет профиль авторизованного пользователя в контекст.
    Для анонимных пользователей — None.
    """
    if request.user.is_authenticated:
        try:
            profile = request.user.profile
        except Exception:
            profile = None
        return {'user_profile': profile}
    return {'user_profile': None}
```

```python
# settings.py — добавляем
'films.context_processors.user_profile',
```

```html
<!-- templates/base.html — аватар в шапке -->
{% if user.is_authenticated and user_profile and user_profile.avatar %}
    <img src="{{ user_profile.avatar.url }}" alt="{{ user.username }}" width="32">
{% endif %}
```

---

## Подводные камни

### AbstractUser — только до первой миграции

Это самое критическое ограничение. Если менять `AUTH_USER_MODEL` после применённых миграций — Django выбросит ошибку, и исправить это без сброса базы очень сложно. Профессиональный совет: в каждом новом Django-проекте первым делом создать `CustomUser(AbstractUser)` с `AUTH_USER_MODEL`, даже если сейчас лишних полей нет — это страховка от боли в будущем.

### RelatedObjectDoesNotExist при обращении к профилю

Если пользователь создан до добавления `UserProfile` (например, суперпользователь через `createsuperuser`), у него не будет профиля. Обращение к `user.profile` выбросит `RelatedObjectDoesNotExist`:

```python
# Упадёт, если профиля нет
profile = request.user.profile

# Безопасное обращение
profile = getattr(request.user, 'profile', None)

# Или через get_or_create
profile, created = UserProfile.objects.get_or_create(user=request.user)
```

Создать профили для существующих пользователей вручную через shell:

```python
from django.contrib.auth.models import User
from films.models import UserProfile

for user in User.objects.filter(profile__isnull=True):
    UserProfile.objects.create(user=user)
```

### Производительность контекстных процессоров

Контекстный процессор вызывается при **каждом** запросе к любой странице с HTML-шаблоном. Если процессор делает тяжёлые запросы к БД — это замедлит весь сайт. Держи запросы в процессорах максимально простыми: `count()`, получение одного объекта, никаких сложных JOIN. Для тяжёлых данных используй кэширование (разберём в модуле 8).

### Две формы и enctype

Когда на одной странице две формы объединены в один `<form>`, нужен `enctype="multipart/form-data"`, если хотя бы одна из них содержит `ImageField` — иначе аватар не загрузится. Проверяем: в нашем случае `ProfileUpdateForm` содержит `avatar` — значит `enctype` обязателен.

---

## Итоговые изменения

```python
# films/models.py — добавляем UserProfile
class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='profile')
    avatar = models.ImageField(upload_to='avatars/', blank=True)
    bio = models.TextField(blank=True)

    class Meta:
        verbose_name = 'Профиль пользователя'
        verbose_name_plural = 'Профили пользователей'

    def __str__(self):
        return f'Профиль: {self.user.username}'
```

```python
# films/context_processors.py
from .models import Film, Genre


def catalog_stats(request):
    return {
        'total_films': Film.objects.count(),
        'total_genres': Genre.objects.count(),
    }


def user_profile(request):
    if request.user.is_authenticated:
        profile = getattr(request.user, 'profile', None)
        return {'user_profile': profile}
    return {'user_profile': None}
```

```python
# films/admin.py — регистрируем UserProfile
from .models import UserProfile

@admin.register(UserProfile)
class UserProfileAdmin(admin.ModelAdmin):
    list_display = ('user', 'bio')
```

---

## Вопросы для проверки

1. В чём разница между расширением пользователя через `OneToOneField` и через `AbstractUser`?
2. Почему Django рекомендует создавать кастомную модель пользователя с самого начала проекта — даже если лишних полей нет?
3. Что такое контекстный процессор и чем он отличается от передачи данных через `render()`?
4. Что произойдёт, если обратиться к `request.user.profile` у пользователя, для которого не был создан объект `UserProfile`?
5. Почему важно держать запросы в контекстных процессорах максимально лёгкими?

---

## Практическая задача

**Тип: расширь проект**

**Часть 1.** Создай и примени миграцию для модели `UserProfile`. Обнови `RegisterView`, чтобы при регистрации автоматически создавался профиль.

**Часть 2.** Создай страницу профиля с двумя формами — `UserUpdateForm` и `ProfileUpdateForm`. Защити страницу через `LoginRequiredMixin`. Добавь маршрут `profile/` и ссылку в навигации.

**Часть 3.** Зарегистрируй контекстный процессор `catalog_stats` в `settings.py` и добавь в футер `base.html` строку с количеством фильмов и жанров. Убедись, что данные отображаются на всех страницах без явной передачи в контексте.

---

[Предыдущий урок](lesson25.md) | [Следующий урок](lesson27.md)