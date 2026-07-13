# Урок 22. Mixins. Пагинация с ListView

## Последний урок модуля — собираем всё вместе

Мы прошли четыре урока по CBV: `TemplateView`, `ListView`, `DetailView`, `CreateView`, `UpdateView`, `DeleteView`. Все они хорошо работают по отдельности, но в реальном проекте один класс часто должен делать несколько вещей одновременно: отображать список и обрабатывать форму фильтрации, требовать авторизации и логировать действия. Для этого Django предлагает Mixins.

Вторая тема урока — пагинация. В прошлом уроке мы включили её в `ListView` через `paginate_by`, а сейчас разберём механику изнутри и реализуем вручную для FBV.

---

## Что такое Mixin

Mixin — это класс, предназначенный не для самостоятельного использования, а для добавления одной конкретной способности другому классу через множественное наследование. В Python множественное наследование выглядит так:

```python
class FilmListView(LoginRequiredMixin, ListView):
    ...
```

`LoginRequiredMixin` не является полноценным CBV — у него нет метода `get()` для обработки запроса. Он только добавляет одну способность: проверять авторизацию перед любым запросом. Полноценный `ListView` берёт на себя всю логику работы со списком. Вместе они дают класс, который и требует авторизации, и показывает список.

> Mixins — это не изобретение Django. Это распространённый паттерн Python-разработки. В Django он используется особенно активно, потому что CBV построены на нём с самого начала: `CreateView`, `UpdateView` — это сами по себе комбинации Mixin-классов.

---

## Встроенные Mixins Django

### LoginRequiredMixin — требование авторизации

Пожалуй, самый используемый. Перенаправляет неавторизованного пользователя на страницу входа:

```python
from django.contrib.auth.mixins import LoginRequiredMixin


class FilmCreateView(LoginRequiredMixin, CreateView):
    model = Film
    form_class = FilmForm
    template_name = 'films/add_film.html'
    login_url = '/login/'         # куда перенаправить (можно настроить глобально в settings.py)
```

Если `login_url` не указан — Django использует значение `settings.LOGIN_URL` (по умолчанию `/accounts/login/`). Подключим `LoginRequiredMixin` полноценно в модуле 7, когда настроим авторизацию. Сейчас важно понять механику.

**Порядок Mixin'ов важен**: `LoginRequiredMixin` должен стоять **перед** основным CBV-классом. Это связано с порядком разрешения методов в Python (MRO — Method Resolution Order): Django ищет методы слева направо по цепочке наследования, и проверка авторизации должна произойти до того, как основной класс начнёт обрабатывать запрос.

```python
# Правильно — LoginRequiredMixin проверяет авторизацию первым
class FilmCreateView(LoginRequiredMixin, CreateView):
    ...

# Неправильно — CreateView обработает запрос до проверки авторизации
class FilmCreateView(CreateView, LoginRequiredMixin):
    ...
```

### PermissionRequiredMixin — проверка разрешений

```python
from django.contrib.auth.mixins import PermissionRequiredMixin


class FilmDeleteView(LoginRequiredMixin, PermissionRequiredMixin, DeleteView):
    model = Film
    permission_required = 'films.delete_film'   # <app_label>.<action>_<model>
    template_name = 'films/film_confirm_delete.html'
    success_url = reverse_lazy('films:film_list')
```

`permission_required` принимает строку `'<app_label>.<permission>'`. Django автоматически создаёт четыре стандартных разрешения для каждой модели при выполнении миграций: `add_film`, `change_film`, `delete_film`, `view_film`. Разрешениями и группами мы займёмся подробно в модуле 7.

---

## Пользовательский Mixin — выносим повторяющуюся логику

Вспомним: у нас в нескольких CBV есть одинаковый `get_queryset()`:

```python
# В FilmListView
def get_queryset(self):
    return Film.objects.select_related('director').prefetch_related('genres', 'actors')

# В FilmDetailView — то же самое
def get_queryset(self):
    return Film.objects.select_related('director').prefetch_related('genres', 'actors')
```

Вынесем это в Mixin:

```python
# films/mixins.py
from .models import Film


class FilmQuerySetMixin:
    """Добавляет оптимизированный QuerySet для запросов к Film."""
    def get_queryset(self):
        return Film.objects.select_related('director').prefetch_related('genres', 'actors')
```

Применяем:

```python
from .mixins import FilmQuerySetMixin


class FilmListView(FilmQuerySetMixin, ListView):
    model = Film
    template_name = 'films/film_list.html'
    context_object_name = 'films'
    paginate_by = 10


class FilmDetailView(FilmQuerySetMixin, DetailView):
    model = Film
    template_name = 'films/film_detail.html'
    context_object_name = 'film'

    def get(self, request, *args, **kwargs):
        response = super().get(request, *args, **kwargs)
        FilmStats.objects.filter(film=self.object).update(views_count=F('views_count') + 1)
        return response
```

Теперь логика `select_related`/`prefetch_related` живёт в одном месте. Если добавится ещё одна связь — поменяем в одном файле, а не в двух классах.

### Mixin с дополнительным контекстом

Другой частый случай — добавить в контекст данные, нужные на многих страницах. Например, список последних фильмов для боковой панели:

```python
# films/mixins.py
class RecentFilmsMixin:
    """Добавляет список последних фильмов в контекст любого CBV."""
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['recent_films'] = Film.objects.recent(3)
        return context
```

```python
class FilmDetailView(FilmQuerySetMixin, RecentFilmsMixin, DetailView):
    model = Film
    template_name = 'films/film_detail.html'
    context_object_name = 'film'
```

`super().get_context_data(**kwargs)` в Mixin — это ключевой элемент корректной цепочки. Когда несколько классов переопределяют один и тот же метод, `super()` идёт по MRO и вызывает следующий метод в цепочке, гарантируя, что все Mixin'ы получат шанс добавить своё содержимое в контекст.

---

## Пагинация с ListView — разбираем механику

В уроке 20 мы добавили `paginate_by = 10` и получили пагинацию «из коробки». Посмотрим, что Django добавляет в контекст:

| Переменная | Что содержит |
|---|---|
| `page_obj` | Объект текущей страницы (`Page`) |
| `paginator` | Объект пагинатора (`Paginator`) |
| `is_paginated` | `True`, если записей больше, чем `paginate_by` |
| `object_list` | Объекты на текущей странице |

Полноценный блок пагинации для шаблона:

```html
<!-- films/templates/films/includes/pagination.html -->
{% if is_paginated %}
<nav class="pagination">
    {% if page_obj.has_previous %}
        <a href="?page=1">« Первая</a>
        <a href="?page={{ page_obj.previous_page_number }}">← Назад</a>
    {% endif %}

    {% for page_num in paginator.page_range %}
        {% if page_num == page_obj.number %}
            <span class="current">{{ page_num }}</span>
        {% elif page_num >= page_obj.number|add:"-2" and page_num <= page_obj.number|add:"2" %}
            <a href="?page={{ page_num }}">{{ page_num }}</a>
        {% endif %}
    {% endfor %}

    {% if page_obj.has_next %}
        <a href="?page={{ page_obj.next_page_number }}">Вперёд →</a>
        <a href="?page={{ paginator.num_pages }}">Последняя »</a>
    {% endif %}

    <span class="info">
        Страница {{ page_obj.number }} из {{ paginator.num_pages }}
        (всего записей: {{ paginator.count }})
    </span>
</nav>
{% endif %}
```

Подключаем в `film_list.html` через `{% include %}`:

```html
{% extends 'base.html' %}

{% block content %}
    <h1>Каталог фильмов</h1>

    {% for film in films %}
        {% include 'films/includes/film_card.html' %}
    {% empty %}
        <p>В каталоге пока нет фильмов.</p>
    {% endfor %}

    {% include 'films/includes/pagination.html' %}
{% endblock %}
```

Вынос пагинации в отдельный `include` — хорошая практика: она понадобится и на странице результатов поиска, и на странице списка рецензий.

---

## Ручная пагинация через Paginator для FBV

`ListView` использует встроенный класс `Paginator`. Разберём его, чтобы применять в FBV — например, для представления `search_film`, которое пока осталось функцией.

```python
# films/views.py
from django.core.paginator import Paginator, InvalidPage
from django.http import Http404


def search_film(request):
    query = request.GET.get('q', '').strip()
    films = Film.objects.none()

    if query:
        films = Film.objects.search(query)

    paginator = Paginator(films, 10)          # по 10 фильмов на странице
    page_number = request.GET.get('page', 1)  # номер страницы из GET-параметра

    try:
        page_obj = paginator.page(page_number)
    except InvalidPage:
        raise Http404('Страница не найдена')

    context = {
        'films': page_obj.object_list,   # объекты текущей страницы
        'page_obj': page_obj,
        'paginator': paginator,
        'is_paginated': paginator.num_pages > 1,
        'query': query,
    }
    return render(request, 'films/search_results.html', context)
```

Разберём Paginator:

```python
paginator = Paginator(queryset, per_page)   # принимает QuerySet и размер страницы
page = paginator.page(page_number)          # возвращает объект Page для конкретной страницы
page.object_list                            # объекты на этой странице
page.has_previous() / page.has_next()       # булевы флаги
page.previous_page_number()                 # номер предыдущей страницы
page.next_page_number()                     # номер следующей страницы
paginator.num_pages                         # всего страниц
paginator.count                             # всего объектов
```

Именно с этими переменными и работает шаблон пагинации, который мы вынесли в `includes/pagination.html` — поэтому он одинаково работает как для `ListView` (где `page_obj` добавляется автоматически), так и для FBV (где мы передаём его вручную).

---

## Пагинация и GET-параметры поиска

Есть тонкий момент: если пользователь на странице `/films/search/?q=коппола` кликает «Следующая страница», ссылка должна быть `/films/search/?q=коппола&page=2`, а не просто `/films/search/?page=2`. Иначе параметр `q` потеряется и вместо второй страницы результатов поиска появится вторая страница всех фильмов.

Решение — в шаблоне включать параметр `q` в ссылки пагинации:

```html
<!-- Для страницы поиска параметр q нужно сохранить в ссылках пагинации -->
<a href="?q={{ query }}&page={{ page_obj.next_page_number }}">Вперёд →</a>
```

Для общего `includes/pagination.html` удобнее использовать django-querystring-based подход или передавать дополнительные GET-параметры через контекст. Простейший вариант — сделать отдельный шаблон пагинации для поиска с учётом `query`:

```html
<!-- films/templates/films/includes/pagination_search.html -->
{% if is_paginated %}
<nav class="pagination">
    {% if page_obj.has_previous %}
        <a href="?q={{ query }}&page={{ page_obj.previous_page_number }}">← Назад</a>
    {% endif %}

    <span>{{ page_obj.number }} / {{ paginator.num_pages }}</span>

    {% if page_obj.has_next %}
        <a href="?q={{ query }}&page={{ page_obj.next_page_number }}">Вперёд →</a>
    {% endif %}
</nav>
{% endif %}
```

---

## Итоговые файлы

```python
# films/mixins.py
from .models import Film


class FilmQuerySetMixin:
    def get_queryset(self):
        return Film.objects.select_related('director').prefetch_related('genres', 'actors')


class RecentFilmsMixin:
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['recent_films'] = Film.objects.recent(3)
        return context
```

```python
# films/views.py (CBV-часть)
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin
from django.views.generic import ListView, DetailView
from django.views.generic.edit import CreateView, UpdateView, DeleteView
from django.db.models import F
from django.urls import reverse_lazy
from .mixins import FilmQuerySetMixin, RecentFilmsMixin
from .models import Film, FilmStats
from .forms import FilmForm


class FilmListView(FilmQuerySetMixin, ListView):
    model = Film
    template_name = 'films/film_list.html'
    context_object_name = 'films'
    paginate_by = 10


class FilmDetailView(FilmQuerySetMixin, RecentFilmsMixin, DetailView):
    model = Film
    template_name = 'films/film_detail.html'
    context_object_name = 'film'

    def get(self, request, *args, **kwargs):
        response = super().get(request, *args, **kwargs)
        FilmStats.objects.filter(film=self.object).update(views_count=F('views_count') + 1)
        return response


class FilmCreateView(LoginRequiredMixin, CreateView):
    model = Film
    form_class = FilmForm
    template_name = 'films/add_film.html'


class FilmUpdateView(LoginRequiredMixin, UpdateView):
    model = Film
    form_class = FilmForm
    template_name = 'films/film_edit.html'
    context_object_name = 'film'

    def get_success_url(self):
        return self.object.get_absolute_url()


class FilmDeleteView(LoginRequiredMixin, PermissionRequiredMixin, DeleteView):
    model = Film
    permission_required = 'films.delete_film'
    template_name = 'films/film_confirm_delete.html'
    context_object_name = 'film'
    success_url = reverse_lazy('films:film_list')
```

---

## Подводные камни

### Порядок Mixin'ов нарушает логику

```python
# LoginRequiredMixin должен идти первым
class FilmCreateView(CreateView, LoginRequiredMixin):
    ...

# Mixin'ы идут слева от основного CBV-класса
class FilmCreateView(LoginRequiredMixin, CreateView):
    ...
```

Правило: специализирующие Mixin'ы — слева, основной класс представления — последний.

### Потеря super() в цепочке Mixin'ов

Если Mixin переопределяет `get_context_data()` без вызова `super()` — он обрывает цепочку, и следующий Mixin в MRO не получит возможности добавить свои данные:

```python
# Обрывает MRO-цепочку — данные других Mixin'ов потеряются
class RecentFilmsMixin:
    def get_context_data(self, **kwargs):
        return {'recent_films': Film.objects.recent(3)}

# Сохраняет цепочку — данные всех Mixin'ов объединятся
class RecentFilmsMixin:
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['recent_films'] = Film.objects.recent(3)
        return context
```

### Paginator и InvalidPage

Пользователь может вручную ввести в адресную строку `/films/?page=99999`, если страниц 10. Без обработки `InvalidPage` — Django выбросит необработанное исключение, а не 404. Всегда оборачивайте `paginator.page()` в `try/except InvalidPage`.

---

## Вопросы для проверки

1. Что такое Mixin и почему его нельзя использовать самостоятельно как представление?
2. Почему порядок Mixin'ов в списке наследования имеет значение?
3. Почему `super().get_context_data(**kwargs)` внутри Mixin критически важен?
4. Что такое `InvalidPage` и почему нужно его обрабатывать при ручной пагинации?
5. Почему при пагинации в форме поиска важно включать параметр `q` в ссылки на другие страницы?

---

## Практическая задача

**Тип: расширь проект**

Создай собственный Mixin `FilmEditMixin`, который объединяет общие атрибуты для `FilmCreateView` и `FilmUpdateView`.

**Требования:**

1. Создай `FilmEditMixin` в `films/mixins.py` с атрибутами `model`, `form_class`, `template_name`, которые повторяются в обоих классах
2. Примени его к `FilmCreateView` и `FilmUpdateView` — каждый класс должен стать заметно короче
3. Добавь в `FilmListView` Mixin `GenreListMixin`, который добавляет в контекст список всех жанров (`genres`) — он понадобится, когда мы добавим фильтрацию по жанру на страницу каталога

---

[Предыдущий урок](lesson21.md) | [Следующий урок](lesson23.md)