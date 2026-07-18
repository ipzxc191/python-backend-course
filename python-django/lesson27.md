# Урок 27. Разрешения и группы (Permissions & Groups)

## Зачем нужны разрешения

В прошлых уроках мы защитили операции записи через `LoginRequiredMixin` — только авторизованный пользователь может добавить фильм. Но все авторизованные пользователи равны между собой: любой зарегистрировавшийся может добавить, изменить или удалить что угодно.

В реальном проекте это не так. Есть администраторы, есть модераторы, есть обычные пользователи. Разрешения (Permissions) и группы (Groups) — механизм Django для разграничения доступа внутри авторизованной аудитории.

---

## Как устроены разрешения в Django

При выполнении `migrate` Django автоматически создаёт четыре разрешения для **каждой** зарегистрированной модели:

| Разрешение | Что означает |
|---|---|
| `<app>.add_<model>` | Право создавать объекты |
| `<app>.change_<model>` | Право редактировать объекты |
| `<app>.delete_<model>` | Право удалять объекты |
| `<app>.view_<model>` | Право просматривать объекты |

Для нашего проекта это, например:

```
films.add_film      films.change_film      films.delete_film      films.view_film
films.add_review    films.change_review    films.delete_review    films.view_review
films.add_director  films.change_director  ...
```

Разрешения хранятся в таблице `auth_permission` и назначаются пользователям либо напрямую, либо через группы.

### Проверка разрешений

```python
# В коде
request.user.has_perm('films.delete_film')     # True / False
request.user.has_perm('films.add_film')

# Несколько разрешений сразу
request.user.has_perms(['films.add_film', 'films.change_film'])

# В шаблоне — через perms (добавляется контекстным процессором auth)
{% if perms.films.delete_film %}
    <a href="{% url 'films:film_delete' slug=film.slug %}">Удалить</a>
{% endif %}
```

`perms` — это специальный объект, доступный во всех шаблонах через процессор `django.contrib.auth.context_processors.auth` (тот же, что добавляет `user`).

---

## Группы (Groups)

Назначать разрешения каждому пользователю отдельно — неудобно. Если нужно добавить право для 50 модераторов — нужно обновить 50 записей. Группы решают это: разрешения назначаются группе, пользователь включается в группу.

```
Группа «Модераторы»
    → films.change_film
    → films.delete_film
    → films.change_review
    → films.delete_review

Пользователь «ivan» → в группе «Модераторы»
                    → автоматически получает все разрешения группы
```

Пользователь может состоять в нескольких группах одновременно и получает объединённый набор разрешений всех групп.

### Создание групп через админку

Самый простой способ — через `/admin/` → раздел «Группы» (из `django.contrib.auth`):

1. Создать группу «Модераторы»
2. Перенести нужные разрешения из левого списка в правый
3. Сохранить

Для пользователя в разделе «Пользователи» → поле «Группы» — добавить нужную группу.

### Создание групп программно

```python
from django.contrib.auth.models import Group, Permission
from django.contrib.content_types.models import ContentType
from films.models import Film, Review


def create_moderator_group():
    moderators, created = Group.objects.get_or_create(name='Модераторы')

    # Получаем разрешения через ContentType
    film_ct = ContentType.objects.get_for_model(Film)
    review_ct = ContentType.objects.get_for_model(Review)

    permissions = Permission.objects.filter(
        content_type__in=[film_ct, review_ct],
        codename__in=['change_film', 'delete_film', 'change_review', 'delete_review']
    )

    moderators.permissions.set(permissions)
    return moderators
```

Удобно запускать через management command или через data migration — тогда группы создаются автоматически при деплое.

---

## Применение в CBV: PermissionRequiredMixin

Мы уже добавляли `PermissionRequiredMixin` к `FilmDeleteView` в уроке 22. Теперь подключим его полноценно:

```python
# films/views.py
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin


class FilmCreateView(LoginRequiredMixin, PermissionRequiredMixin, FilmEditMixin, CreateView):
    permission_required = 'films.add_film'


class FilmUpdateView(LoginRequiredMixin, PermissionRequiredMixin, FilmEditMixin, UpdateView):
    permission_required = 'films.change_film'
    context_object_name = 'film'

    def get_success_url(self):
        return self.object.get_absolute_url()


class FilmDeleteView(LoginRequiredMixin, PermissionRequiredMixin, DeleteView):
    model = Film
    permission_required = 'films.delete_film'
    template_name = 'films/film_confirm_delete.html'
    success_url = reverse_lazy('films:film_list')
```

`PermissionRequiredMixin` по умолчанию возвращает `403 Forbidden` при отсутствии разрешения (если пользователь авторизован). Если пользователь не авторизован вовсе — `LoginRequiredMixin` перехватит запрос раньше.

### raise_exception — 403 или редирект

```python
class FilmDeleteView(LoginRequiredMixin, PermissionRequiredMixin, DeleteView):
    permission_required = 'films.delete_film'
    raise_exception = True   # 403 Forbidden для авторизованных без прав
                             # По умолчанию тоже True для авторизованных
```

Поведение без `raise_exception = True` (по умолчанию `False`): авторизованный пользователь без нужного разрешения будет перенаправлен на страницу входа — это неинтуитивно. Лучше всегда ставить `raise_exception = True`, чтобы пользователь видел 403, а не форму входа повторно.

---

## Применение в FBV: декоратор permission_required

```python
from django.contrib.auth.decorators import permission_required


@permission_required('films.add_film', raise_exception=True)
def add_film_fbv(request):
    ...
```

Или через `user_passes_test` для произвольных условий:

```python
from django.contrib.auth.decorators import user_passes_test


@user_passes_test(lambda u: u.is_staff, login_url='/accounts/login/')
def staff_only_view(request):
    ...
```

---

## Кастомные разрешения

Стандартных четырёх разрешений часто недостаточно. Например, нужно разрешение «может публиковать рецензии без модерации» — это не `add_review`, это отдельное бизнес-правило.

Кастомные разрешения описываются в `Meta` модели:

```python
# films/models.py
class Review(models.Model):
    film = models.ForeignKey(Film, on_delete=models.CASCADE, related_name='reviews')
    author_name = models.CharField(max_length=100)
    text = models.TextField()
    rating = models.PositiveSmallIntegerField()
    is_published = models.BooleanField(default=False, verbose_name='Опубликована')
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-created_at']
        verbose_name = 'Рецензия'
        verbose_name_plural = 'Рецензии'
        permissions = [
            ('publish_review', 'Может публиковать рецензии без модерации'),
            ('moderate_review', 'Может модерировать чужие рецензии'),
        ]
```

После добавления `permissions` нужна миграция:

```bash
python manage.py makemigrations
python manage.py migrate
```

Django создаст разрешения `films.publish_review` и `films.moderate_review` в таблице `auth_permission`. Их можно назначать пользователям и группам так же, как стандартные.

Применение в представлении:

```python
class AddReviewView(LoginRequiredMixin, FormView):
    form_class = ReviewForm
    template_name = 'films/add_review.html'

    def form_valid(self, form):
        film = get_object_or_404(Film, slug=self.kwargs['slug'])
        review = form.save(commit=False)
        review.film = film
        # Публикуем сразу, если есть разрешение, иначе — на модерацию
        if self.request.user.has_perm('films.publish_review'):
            review.is_published = True
        review.save()
        return redirect(film.get_absolute_url())
```

---

## Шаблон страницы 403

Когда `PermissionRequiredMixin` возвращает 403, Django ищет шаблон `403.html` в корневой папке шаблонов:

```html
<!-- templates/403.html -->
{% extends 'base.html' %}

{% block title %}Доступ запрещён{% endblock %}

{% block content %}
    <h1>403 — Доступ запрещён</h1>
    <p>У вас нет прав для выполнения этого действия.</p>
    {% if not user.is_authenticated %}
        <p><a href="{% url 'login' %}">Войдите в систему</a>, чтобы получить доступ.</p>
    {% else %}
        <p>Если вы считаете, что это ошибка — обратитесь к администратору.</p>
    {% endif %}
    <a href="{% url 'films:index' %}">← На главную</a>
{% endblock %}
```

Аналогично работает `404.html` для страниц 404 — создаётся в той же папке `templates/`.

---

## Проверка разрешений в шаблонах

Переменная `perms` позволяет динамически показывать или скрывать элементы интерфейса:

```html
<!-- films/templates/films/film_detail.html -->
{% extends 'base.html' %}

{% block content %}
    <h1>{{ film.title }}</h1>
    <!-- ... содержимое страницы ... -->

    <div class="film-actions">
        {% if perms.films.change_film %}
            <a href="{% url 'films:film_edit' slug=film.slug %}">Редактировать</a>
        {% endif %}

        {% if perms.films.delete_film %}
            <a href="{% url 'films:film_delete' slug=film.slug %}">Удалить</a>
        {% endif %}
    </div>
{% endblock %}
```

> Важно: скрытие кнопок в шаблоне — это только UX. Защита на уровне представления через `PermissionRequiredMixin` или декоратор обязательна. Пользователь может обойти шаблон и отправить запрос напрямую.

---

## Подводные камни

### Кэширование разрешений

Django кэширует разрешения пользователя в рамках одного запроса. Если разрешение было добавлено программно в той же сессии — `has_perm()` может вернуть старое значение. Чтобы сбросить кэш:

```python
from django.contrib.auth.models import Permission

# Добавили разрешение
user.user_permissions.add(permission)

# Сбрасываем кэш — перечитываем объект из БД
from django.contrib.auth.models import User
user = User.objects.get(pk=user.pk)
print(user.has_perm('films.delete_film'))  # теперь актуальное значение
```

### PermissionRequiredMixin и несколько разрешений

`permission_required` принимает как строку, так и список — если нужно потребовать сразу несколько:

```python
class FilmModerateView(LoginRequiredMixin, PermissionRequiredMixin, View):
    # Требует ОБА разрешения одновременно
    permission_required = ['films.change_film', 'films.moderate_review']
```

Все перечисленные разрешения должны быть у пользователя — это логика `AND`, а не `OR`.

### Суперпользователь и разрешения

`is_superuser=True` автоматически даёт все разрешения — `has_perm()` всегда возвращает `True` независимо от реально назначенных прав. Это удобно для разработки, но важно помнить при тестировании: тест под суперпользователем не проверяет реальную работу разрешений для обычных пользователей.

---

## Итоговая настройка представлений

```python
# films/views.py — финальная версия CBV с разрешениями

class FilmCreateView(LoginRequiredMixin, PermissionRequiredMixin, FilmEditMixin, CreateView):
    permission_required = 'films.add_film'
    raise_exception = True


class FilmUpdateView(LoginRequiredMixin, PermissionRequiredMixin, FilmEditMixin, UpdateView):
    permission_required = 'films.change_film'
    context_object_name = 'film'
    raise_exception = True

    def get_success_url(self):
        return self.object.get_absolute_url()


class FilmDeleteView(LoginRequiredMixin, PermissionRequiredMixin, DeleteView):
    model = Film
    permission_required = 'films.delete_film'
    template_name = 'films/film_confirm_delete.html'
    success_url = reverse_lazy('films:film_list')
    raise_exception = True


class DirectorCreateView(LoginRequiredMixin, PermissionRequiredMixin, CreateView):
    model = Director
    form_class = DirectorForm
    template_name = 'films/director_form.html'
    permission_required = 'films.add_director'
    raise_exception = True


class DirectorUpdateView(LoginRequiredMixin, PermissionRequiredMixin, UpdateView):
    model = Director
    form_class = DirectorForm
    template_name = 'films/director_form.html'
    permission_required = 'films.change_director'
    context_object_name = 'director'
    raise_exception = True

    def get_success_url(self):
        return self.object.get_absolute_url()


class DirectorDeleteView(LoginRequiredMixin, PermissionRequiredMixin, DeleteView):
    model = Director
    template_name = 'films/director_confirm_delete.html'
    permission_required = 'films.delete_director'
    context_object_name = 'director'
    success_url = reverse_lazy('films:film_list')
    raise_exception = True
```

---

## Вопросы для проверки

1. Какие четыре стандартных разрешения Django создаёт для каждой модели и когда?
2. В чём разница между назначением разрешения пользователю напрямую и через группу?
3. Почему скрытие кнопок через `{% if perms.films.delete_film %}` в шаблоне не является достаточной защитой?
4. Почему `is_superuser=True` может мешать тестированию разрешений?
5. Что такое кастомное разрешение и как его создать?

---

## Практическая задача

**Тип: расширь проект**

**Часть 1.** Создай группу «Модераторы» через административную панель. Назначь ей разрешения: `films.change_film`, `films.delete_film`, `films.change_review`, `films.delete_review`. Добавь в группу любого тестового пользователя (не суперпользователя) и убедись, что он может редактировать фильмы через сайт.

**Часть 2.** Добавь кастомное разрешение `can_feature_film` (название — «Может выделять фильм в подборку») к модели `Film` через `Meta.permissions`. Создай и примени миграцию. Убедись, что разрешение появилось в AdminPanel в списке разрешений для группы.

**Часть 3.** В шаблоне `film_detail.html` скрой кнопки «Редактировать» и «Удалить» для пользователей без соответствующих разрешений — через `{% if perms.films.change_film %}` и `{% if perms.films.delete_film %}`.

---

[Предыдущий урок](lesson26.md) | [Следующий урок](lesson28.md)