# Урок 21. FormView, CreateView, UpdateView, DeleteView

## Формы через классы

В прошлом уроке мы перевели список и детальную страницу на CBV. Осталась вторая половина типичного CRUD-цикла — создание, редактирование и удаление объектов. Django предоставляет готовые классы для каждого сценария: `CreateView`, `UpdateView`, `DeleteView`. И ещё один — `FormView` — для форм, не привязанных к конкретной модели.

---

## FormView — форма без модели

`FormView` — базовый класс для форм, которые не создают и не редактируют объекты модели. Хорошие кандидаты — форма поиска, форма обратной связи, форма с несколькими шагами.

В уроке 16 мы написали представление `add_review` через `View`:

```python
class AddReviewView(View):
    def get(self, request, slug):
        film = get_object_or_404(Film, slug=slug)
        form = ReviewForm()
        return render(request, 'films/add_review.html', {'form': form, 'film': film})

    def post(self, request, slug):
        film = get_object_or_404(Film, slug=slug)
        form = ReviewForm(request.POST)
        if form.is_valid():
            review = form.save(commit=False)
            review.film = film
            review.save()
            return redirect(film.get_absolute_url())
        return render(request, 'films/add_review.html', {'form': form, 'film': film})
```

Перепишем через `FormView`:

```python
from django.views.generic.edit import FormView
from django.shortcuts import get_object_or_404
from django.urls import reverse_lazy


class AddReviewView(FormView):
    form_class = ReviewForm
    template_name = 'films/add_review.html'

    def get_film(self):
        return get_object_or_404(Film, slug=self.kwargs['slug'])

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['film'] = self.get_film()
        return context

    def form_valid(self, form):
        film = self.get_film()
        review = form.save(commit=False)
        review.film = film
        review.save()
        return redirect(film.get_absolute_url())
```

Разберём ключевые методы:

**`form_valid(self, form)`** — вызывается автоматически, когда форма прошла валидацию. Получает готовый объект формы с данными. Заменяет блок `if form.is_valid():` из FBV.

**`get_context_data()`** — уже знакомый метод, добавляем `film` в контекст, чтобы шаблон мог показать, к какому фильму относится рецензия.

**`self.kwargs['slug']`** — параметры URL доступны через `self.kwargs` (именованные) и `self.args` (позиционные). Это стандартный способ получить URL-параметры внутри любого CBV.

> `FormView` — базовый класс для работы с любой формой: он показывает форму, принимает данные, запускает валидацию и вызывает `form_valid()`, если данные корректны. Что именно делать с формой в `form_valid()` — решает разработчик.

### success_url vs form_valid() для редиректа

Если после успешной отправки всегда нужно перейти на один и тот же URL — можно задать `success_url` вместо переопределения `form_valid()`:

```python
class AddReviewView(FormView):
    form_class = ReviewForm
    template_name = 'films/add_review.html'
    success_url = reverse_lazy('films:film_list')  # всегда на список фильмов
```

`reverse_lazy()` — это ленивая версия `reverse()`. Нужна здесь потому, что атрибут класса вычисляется в момент импорта модуля, а маршруты могут быть ещё не загружены. `reverse_lazy()` откладывает вычисление URL до первого обращения — именно для этого сценария он и создан.

В нашем случае редирект зависит от конкретного фильма, поэтому `success_url` не подходит — переопределяем `form_valid()`.

---

## CreateView — создание объекта

`CreateView` специализирован для создания нового объекта через форму. Это `FormView`, который сам знает, как сохранить объект модели:

```python
from django.views.generic.edit import CreateView


class FilmCreateView(CreateView):
    model = Film
    form_class = FilmForm
    template_name = 'films/add_film.html'
```

Django автоматически:
- Создаёт пустую форму на GET-запрос
- Принимает данные формы на POST-запрос
- Вызывает `form.save()` при успешной валидации
- Перенаправляет на `success_url` или на `film.get_absolute_url()`, если он определён

Обрати внимание: `CreateView` не требует явного `success_url`, если у модели есть `get_absolute_url()` — Django сам сделает редирект на возвращаемый URL. Именно поэтому мы добавляли `get_absolute_url()` в урок 12.

> `CreateView` — специализированный подкласс `FormView`, который автоматически вызывает `form.save()` в `form_valid()` для создания нового объекта модели, и знает, как перенаправить пользователя после успешного сохранения.

### fields вместо form_class

Если специального класса формы нет, `CreateView` может сгенерировать её сам прямо из модели:

```python
class FilmCreateView(CreateView):
    model = Film
    fields = ['title', 'year', 'description', 'director', 'genres', 'poster']
    template_name = 'films/add_film.html'
```

Это удобно для простых случаев, но не позволяет настроить виджеты, добавить кастомную валидацию или метки — для этого нужен явный `form_class`. В нашем проекте у нас уже есть `FilmForm` с кастомной валидацией из урока 17, поэтому используем `form_class`.

### Шаблон для CreateView

Если не указывать `template_name`, `CreateView` ищет шаблон по имени `<app>/<model>_form.html` — `films/film_form.html`. Мы явно указали `films/add_film.html`, чтобы переиспользовать существующий шаблон, а не создавать новый.

---

## UpdateView — редактирование объекта

`UpdateView` идентичен `CreateView` по настройке, но ищет существующий объект и заполняет форму его текущими данными:

```python
from django.views.generic.edit import UpdateView


class FilmUpdateView(UpdateView):
    model = Film
    form_class = FilmForm
    template_name = 'films/film_edit.html'
    context_object_name = 'film'
```

Объект ищется так же, как в `DetailView` — по `pk` или `slug` из URL:

```python
# films/urls.py
path('films/<slug:slug>/edit/', FilmUpdateView.as_view(), name='film_edit'),
```

Шаблон отличается от формы создания только тем, что поля уже заполнены — можно переиспользовать один и тот же шаблон через `template_name`, но обычно делают отдельный, чтобы легко изменить заголовок и кнопку:

```html
<!-- films/templates/films/film_edit.html -->
{% extends 'base.html' %}

{% block title %}Редактировать: {{ film.title }}{% endblock %}

{% block content %}
    <h1>Редактировать фильм «{{ film.title }}»</h1>

    <form method="POST" enctype="multipart/form-data">
        {% csrf_token %}
        {{ form.as_p }}
        <button type="submit">Сохранить изменения</button>
    </form>

    <a href="{{ film.get_absolute_url }}">← Отменить</a>
{% endblock %}
```

### get_success_url() — динамический редирект

После редактирования логично перенаправить пользователя на страницу изменённого объекта. Поскольку slug может измениться при редактировании названия, используем `get_success_url()`:

```python
class FilmUpdateView(UpdateView):
    model = Film
    form_class = FilmForm
    template_name = 'films/film_edit.html'
    context_object_name = 'film'

    def get_success_url(self):
        return self.object.get_absolute_url()
```

`self.object` — это уже сохранённый объект с обновлёнными данными, включая, возможно, новый slug.

---

## DeleteView — удаление объекта

`DeleteView` находит объект и предлагает пользователю подтвердить удаление:

```python
from django.views.generic.edit import DeleteView


class FilmDeleteView(DeleteView):
    model = Film
    template_name = 'films/film_confirm_delete.html'
    context_object_name = 'film'
    success_url = reverse_lazy('films:film_list')
```

Для `DeleteView` нет форм данных — поэтому `success_url` вместо `get_success_url()` вполне уместен: после удаления объекта `self.object` уже не существует, поэтому `self.object.get_absolute_url()` невозможен.

### Шаблон подтверждения

```html
<!-- films/templates/films/film_confirm_delete.html -->
{% extends 'base.html' %}

{% block title %}Удалить {{ film.title }}{% endblock %}

{% block content %}
    <h1>Удалить фильм?</h1>
    <p>Вы уверены, что хотите удалить «{{ film.title }}»? Это действие необратимо.</p>

    <form method="POST">
        {% csrf_token %}
        <button type="submit">Да, удалить</button>
        <a href="{{ film.get_absolute_url }}">Отменить</a>
    </form>
{% endblock %}
```

Обрати внимание: форма подтверждения не содержит никаких полей — только `{% csrf_token %}` и кнопку. GET-запрос показывает страницу подтверждения, POST-запрос выполняет удаление.

### Маршрут для DeleteView

```python
path('films/<slug:slug>/delete/', FilmDeleteView.as_view(), name='film_delete'),
```

---

## Связывание операций в шаблоне

Добавим на страницу фильма ссылки для редактирования и удаления:

```html
<!-- films/templates/films/film_detail.html — добавляем в конец block content -->
<div class="film-actions">
    <a href="{% url 'films:film_edit' slug=film.slug %}">Редактировать</a>
    <a href="{% url 'films:film_delete' slug=film.slug %}">Удалить</a>
</div>
```

В реальных проектах такие ссылки защищают проверкой прав доступа — только администратор или автор видит кнопки редактирования. Это мы разберём в модуле 7 через `LoginRequiredMixin` и `PermissionRequiredMixin`.

---

## Итоговый urls.py

```python
# films/urls.py
from django.urls import path
from .views import (
    IndexView, AboutView, CatalogStatsView,
    FilmListView, FilmDetailView, FilmCreateView, FilmUpdateView, FilmDeleteView,
    AddReviewView, DirectorDetailView,
)
from . import views

app_name = 'films'

urlpatterns = [
    path('', IndexView.as_view(), name='index'),
    path('about/', AboutView.as_view(), name='about'),
    path('films/', FilmListView.as_view(), name='film_list'),
    path('films/search/', views.search_film, name='search_film'),
    path('films/add/', FilmCreateView.as_view(), name='add_film'),
    path('films/<slug:slug>/', FilmDetailView.as_view(), name='film_detail'),
    path('films/<slug:slug>/edit/', FilmUpdateView.as_view(), name='film_edit'),
    path('films/<slug:slug>/delete/', FilmDeleteView.as_view(), name='film_delete'),
    path('films/<slug:slug>/review/', AddReviewView.as_view(), name='add_review'),
    path('directors/<slug:slug>/', DirectorDetailView.as_view(), name='director_detail'),
    path('stats/', CatalogStatsView.as_view(), name='catalog_stats'),
]
```

---

## Подводные камни

### reverse() vs reverse_lazy() в атрибутах класса

```python
# TypeError при старте — маршруты ещё не загружены на этапе импорта модуля
class FilmDeleteView(DeleteView):
    success_url = reverse('films:film_list')

# reverse_lazy() откладывает вычисление до первого использования
class FilmDeleteView(DeleteView):
    success_url = reverse_lazy('films:film_list')
```

Атрибуты класса вычисляются в момент импорта Python-модуля. В этот момент URLconf может быть ещё не загружена, и обычный `reverse()` выбросит ошибку.

`reverse_lazy()` откладывает вычисление URL до первого фактического обращения к атрибуту — к тому моменту маршруты уже загружены. 

Правило простое: `reverse()` используется внутри функций и методов, `reverse_lazy()` — в атрибутах класса.

### CreateView и файлы — нужен ли enctype?

Шаблоны для `CreateView` и `UpdateView` нужно оформлять с `enctype="multipart/form-data"`, если форма содержит поле `ImageField` или `FileField` — Django не добавляет этот атрибут автоматически. Шаблон создаёт разработчик, и ответственность за `enctype` лежит на нём.

### UpdateView и slug — проблема после изменения заголовка

Если пользователь меняет название фильма, а в `Film.save()` slug генерируется из заголовка — после сохранения объект получит новый slug. Тогда `get_success_url()` вернёт URL с новым slug, и редирект пройдёт корректно. Но если где-то в коде slug захардкожен или закэширован — эта страница потеряет доступ. Напоминаем об этом как о долгосрочном архитектурном решении — не критично для учебного проекта, но важно знать.

---

## Вопросы для проверки

1. Чем `FormView` отличается от `CreateView`?
2. Зачем нужен `reverse_lazy()` в атрибутах класса, и чем он отличается от `reverse()`?
3. Почему для `DeleteView` нельзя использовать `self.object.get_absolute_url()` в `get_success_url()`?
4. Как `UpdateView` получает существующий объект? Нужно ли писать `get_object_or_404` вручную?
5. Чем отличается `form_valid()` от `get_success_url()` и когда нужен каждый?

---

## Практическая задача

**Тип: расширь проект**

Добавь полноценный CRUD для режиссёров, используя `CreateView`, `UpdateView` и `DeleteView`.

**Требования:**

1. Создай `DirectorForm` (ModelForm) с полями `name`, `bio`, `photo`
2. Создай `DirectorCreateView` — форма создания, после успешного сохранения перенаправляет на страницу режиссёра через `get_absolute_url()`
3. Создай `DirectorUpdateView` — форма редактирования, тот же редирект
4. Создай `DirectorDeleteView` — страница подтверждения удаления, после успешного удаления перенаправляет на `films:film_list`
5. Добавь три новых маршрута в `urls.py`

---

[Предыдущий урок](lesson20.md) | [Следующий урок](lesson22.md)