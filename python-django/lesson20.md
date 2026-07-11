# Урок 20. ListView и DetailView

## Самая частая пара страниц в любом сайте

Список объектов и страница отдельного объекта — это, без преувеличения, самый частый паттерн в веб-разработке. Каталог фильмов, список товаров, лента статей — везде повторяется одна и та же пара: «покажи все» и «покажи один». Django выделил это в два готовых класса: `ListView` и `DetailView`.

Это первые generic views, где разница с FBV становится по-настоящему заметна — Django берёт на себя получение объекта из базы, пагинацию, обработку 404, а нам остаётся только настроить несколько атрибутов класса.

---

## ListView

Вспомним FBV-версию из урока 9:

```python
def film_list(request):
    films = Film.objects.select_related('director').prefetch_related('genres')
    return render(request, 'films/film_list.html', {'films': films})
```

Версия с `ListView`:

```python
# films/views.py
from django.views.generic import ListView
from .models import Film


class FilmListView(ListView):
    model = Film
    template_name = 'films/film_list.html'
```

Всего два атрибута класса заменяют тело функции. Разберём, что Django делает «под капотом»:

- `model = Film` — указывает, какую модель использовать. Django сам выполнит `Film.objects.all()`
- `template_name` — какой шаблон рендерить
- Имя переменной контекста для шаблона Django формирует автоматически: `object_list` или, что удобнее, `<имя_модели_в_нижнем_регистре>_list` — то есть `film_list`

То есть наш существующий шаблон `film_list.html`, который использует переменную `films`, перестанет работать без изменений — он ожидает `films`, а `ListView` передаёт `film_list`. Разберём, как это настроить, через `context_object_name`.

### context_object_name — переименование переменной контекста

```python
class FilmListView(ListView):
    model = Film
    template_name = 'films/film_list.html'
    context_object_name = 'films'
```

Теперь в шаблоне можно продолжать использовать `{% for film in films %}` без каких-либо изменений — мы просто подсказали `ListView`, как назвать переменную, чтобы не трогать уже написанный шаблон.

### get_queryset() — настройка запроса

По умолчанию `ListView` выполняет `Model.objects.all()`. Если нужна более сложная выборка — с `select_related`, `prefetch_related`, фильтрацией или сортировкой — переопределяем метод `get_queryset()`:

```python
class FilmListView(ListView):
    model = Film
    template_name = 'films/film_list.html'
    context_object_name = 'films'

    def get_queryset(self):
        return Film.objects.select_related('director').prefetch_related('genres', 'actors')
```

Это прямой эквивалент того, что мы писали руками в FBV — просто метод, который возвращает QuerySet, вместо обращения к ORM прямо в теле функции.

### Автоматическое имя шаблона

Если не указывать `template_name` явно, Django попытается найти шаблон по шаблону имени: `<app_label>/<model_name>_list.html`. Для нашего случая это было бы `films/film_list.html` — точно то же имя, что мы используем. Можно было бы вообще убрать `template_name` из класса, и всё продолжило бы работать:

```python
class FilmListView(ListView):
    model = Film
    context_object_name = 'films'

    def get_queryset(self):
        return Film.objects.select_related('director').prefetch_related('genres', 'actors')
```

Мы оставим `template_name` явно — для читаемости и потому что явное указание защищает от путаницы, если кто-то переименует модель в будущем.

---

## Пагинация «из коробки»

Это одна из самых ощутимых выгод `ListView` — встроенная пагинация без единой дополнительной строчки логики.

Достаточно указать атрибут `paginate_by`:

```python
class FilmListView(ListView):
    model = Film
    template_name = 'films/film_list.html'
    context_object_name = 'films'
    paginate_by = 10

    def get_queryset(self):
        return Film.objects.select_related('director').prefetch_related('genres', 'actors')
```

`paginate_by = 10` автоматически разбивает результат на страницы по 10 объектов.

Теперь достаточно перейти по адресу `/films/` чтобы увидеть первую страницу. Или указать GET-параметр `/films/?page=2` для второй страницы, для третьей и т.д.

Получается, `ListView` самостоятельно:

- читает параметр `page` из URL;
- определяет, какую страницу нужно показать;
- разбивает queryset на части;
- передает нужные объекты в шаблон.

### Объект `page_obj`

Когда используется `paginate_by`, Django автоматически добавляет в контекст шаблона объект `page_obj`.

Это экземпляр класса `Page`, который содержит информацию о текущей странице и позволяет строить навигацию.

Наиболее часто используются следующие атрибуты и методы:

| Выражение | Что делает |
|-----------|------------|
| `page_obj.number` | номер текущей страницы |
| `page_obj.has_previous` | есть ли предыдущая страница |
| `page_obj.previous_page_number` | номер предыдущей страницы |
| `page_obj.has_next` | есть ли следующая страница |
| `page_obj.next_page_number` | номер следующей страницы |
| `page_obj.paginator.num_pages` | общее количество страниц |

Именно их обычно используют при построении пагинации.

```html
<!-- films/templates/films/film_list.html -->
{% extends 'base.html' %}

{% block content %}
    <h1>Каталог фильмов</h1>

    {% for film in films %}
        {% include 'films/includes/film_card.html' %}
    {% empty %}
        <p>В каталоге пока нет фильмов.</p>
    {% endfor %}

    {% if page_obj.has_other_pages %}
        <div class="pagination">
            {% if page_obj.has_previous %}
                <a href="?page={{ page_obj.previous_page_number }}">← Назад</a>
            {% endif %}

            <span>Страница {{ page_obj.number }} из {{ page_obj.paginator.num_pages }}</span>

            {% if page_obj.has_next %}
                <a href="?page={{ page_obj.next_page_number }}">Вперёд →</a>
            {% endif %}
        </div>
    {% endif %}
{% endblock %}
```

Полноценную пагинацию — включая ручную реализацию через класс `Paginator` для FBV — мы разберём подробно в уроке 22. Сейчас достаточно знать, что `ListView` даёт её бесплатно через один атрибут.

---

## DetailView

Теперь страница отдельного фильма. Вспомним FBV-версию из урока 12:

```python
def film_detail(request, slug):
    film = get_object_or_404(
        Film.objects.select_related('director').prefetch_related('genres', 'actors'),
        slug=slug
    )
    FilmStats.objects.filter(film=film).update(views_count=F('views_count') + 1)
    return render(request, 'films/film_detail.html', {'film': film})
```

Версия с `DetailView`:

```python
from django.views.generic import DetailView


class FilmDetailView(DetailView):
    model = Film
    template_name = 'films/film_detail.html'
    context_object_name = 'film'
```

`DetailView` автоматически делает то, что мы раньше писали через `get_object_or_404()`: находит объект по параметру из URL и возвращает 404, если объект не найден.

### Как DetailView находит объект по умолчанию

По умолчанию `DetailView` умеет искать объект двумя способами:

1. по первичному ключу (`pk`);
2. по `slug`.

Алгоритм работы выглядит следующим образом:

- если в URL есть параметр `pk`, поиск выполняется по первичному ключу;
- если параметра `pk` нет, но есть параметр `slug`, поиск выполняется по `slug`;
- если не найден ни `pk`, ни `slug`, Django не сможет определить, какой объект нужно открыть.

Поэтому самый простой вариант выглядит так:

```python
# films/urls.py
path('films/<int:pk>/', FilmDetailView.as_view(), name='film_detail')
```

### Почему `pk` и `id` — это не совсем одно и то же

В Django `pk` (Primary Key) — это не название поля, а специальное сокращение для **первичного ключа** модели. Например, если модель объявлена так:

```python
class Film(models.Model):
    title = models.CharField(max_length=200)
```

то Django автоматически создаст поле:

```python
id = models.AutoField(primary_key=True)
```

В этом случае `pk` и `id` указывают на одно и то же поле, поэтому запросы

```python
Film.objects.get(id=15)
```

и

```python
Film.objects.get(pk=15)
```

полностью эквивалентны.

Однако `pk` считается более универсальным способом. Если позже в модели изменить первичный ключ, например:

```python
class Film(models.Model):
    uuid = models.UUIDField(primary_key=True)
```

то код с `pk` продолжит работать без изменений, а запрос по `id` уже вызовет ошибку, потому что поля `id` больше не существует.

### Поиск по `slug`

У нас в проекте маршрут построен на `slug`, не на `pk`.

При необходимости это поведение можно настроить двумя атрибутами:

```python
class FilmDetailView(DetailView):
    model = Film
    template_name = 'films/film_detail.html'
    context_object_name = 'film'


    slug_field = 'slug'        # имя поля в модели (по умолчанию тоже 'slug', можно не указывать)
    slug_url_kwarg = 'slug'    # имя параметра в urls.py (по умолчанию тоже 'slug')
```

Они отвечают за разные вещи:

- `slug_field` — имя поля модели, в котором выполняется поиск.
- `slug_url_kwarg` — имя параметра, который берётся из URL.

Поскольку у нас и поле модели, и параметр URL называются `slug` — оба атрибута можно опустить, они совпадают со значениями по умолчанию. 

Но явное указание не будет ошибкой, и часто облегчает чтение кода тем, кто впервые видит класс.

```python
# films/urls.py
path('films/<slug:slug>/', FilmDetailView.as_view(), name='film_detail'),
```

### get_queryset() в DetailView

Точно так же, как в `ListView`, переопределяем `get_queryset()`, чтобы добавить `select_related`/`prefetch_related`:

```python
class FilmDetailView(DetailView):
    model = Film
    template_name = 'films/film_detail.html'
    context_object_name = 'film'

    def get_queryset(self):
        return Film.objects.select_related('director').prefetch_related('genres', 'actors')
```

### Логика инкремента просмотров — переопределяем get()

В FBV-версии увеличение счётчика просмотров было частью линейного кода. В CBV для добавления побочного эффекта (то, что происходит «вокруг» стандартной логики) переопределяем метод `get()`:

```python
from django.db.models import F
from .models import FilmStats


class FilmDetailView(DetailView):
    model = Film
    template_name = 'films/film_detail.html'
    context_object_name = 'film'

    def get_queryset(self):
        return Film.objects.select_related('director').prefetch_related('genres', 'actors')

    def get(self, request, *args, **kwargs):
        response = super().get(request, *args, **kwargs)
        FilmStats.objects.filter(film=self.object).update(views_count=F('views_count') + 1)
        return response
```

Это важный паттерн: `super().get(...)` выполняет всю стандартную логику `DetailView` (поиск объекта, рендеринг шаблона) и сохраняет найденный объект в `self.object`. После этого мы можем использовать `self.object` для дополнительных действий — в нашем случае обновить статистику — и только потом вернуть готовый `response`.

---

## Подключение в urls.py

```python
# films/urls.py
from django.urls import path
from .views import IndexView, AboutView, AddFilmView, CatalogStatsView, FilmListView, FilmDetailView
from . import views

app_name = 'films'

urlpatterns = [
    path('', IndexView.as_view(), name='index'),
    path('about/', AboutView.as_view(), name='about'),
    path('films/', FilmListView.as_view(), name='film_list'),
    path('films/search/', views.search_film, name='search_film'),
    path('films/add/', AddFilmView.as_view(), name='add_film'),
    path('films/<slug:slug>/', FilmDetailView.as_view(), name='film_detail'),
    path('films/<slug:slug>/review/', views.add_review, name='add_review'),
    path('directors/<slug:slug>/', views.director_detail, name='director_detail'),
    path('stats/', CatalogStatsView.as_view(), name='catalog_stats'),
]
```

---

## Сравнительная таблица: что заменяет каждый атрибут

| Атрибут / метод CBV | Эквивалент в FBV |
|---|---|
| `model = Film` | `Film.objects.all()` (для ListView) |
| `get_queryset()` | Ручной вызов `.select_related()`, `.prefetch_related()`, `.filter()` |
| `context_object_name` | Имя ключа в словаре контекста для `render()` |
| `paginate_by` | Ручное создание `Paginator` и расчёт страницы |
| `slug_field`, `slug_url_kwarg` | Аргумент в `get_object_or_404(Film, slug=slug)` |
| Автоматический 404 | `get_object_or_404()` |

---

## Подводные камни

### Имя переменной контекста по умолчанию легко забыть

Если не указать `context_object_name`, `ListView` передаёт переменную как `object_list` (всегда доступна) и дополнительно как `<model>_list` (`film_list` для модели `Film`). `DetailView` передаёт `object` и `<model>` (`film`). Если шаблон написан под одно имя, а класс настроен без `context_object_name` — переменная в шаблоне окажется пустой без какой-либо ошибки, просто молча не отрендерится.

```html
<!-- Если забыли context_object_name, а в шаблоне написано films — будет пусто -->
{% for film in films %}  {# films не определена — ничего не выведется #}
```

### Забытый super().get() при переопределении get()

```python
# Полностью переопределили get() — потеряли всю логику DetailView
def get(self, request, *args, **kwargs):
    FilmStats.objects.filter(film=self.object).update(views_count=F('views_count') + 1)
    # self.object ещё не существует на этом этапе! AttributeError

# Сначала вызываем родительский get(), который установит self.object
def get(self, request, *args, **kwargs):
    response = super().get(request, *args, **kwargs)
    FilmStats.objects.filter(film=self.object).update(views_count=F('views_count') + 1)
    return response
```

### slug_field против slug_url_kwarg — путаница в именах

`slug_field` — это имя поля **в модели**. `slug_url_kwarg` — это имя параметра **в URL-маршруте**. Если они называются по-разному — оба нужно указать явно:

```python
# Если в модели поле называется code, а в URL параметр film_slug
class FilmDetailView(DetailView):
    model = Film
    slug_field = 'code'
    slug_url_kwarg = 'film_slug'
```

---

## Итоговый код урока

```python
# films/views.py
from django.views.generic import ListView, DetailView
from django.db.models import F
from .models import Film, FilmStats


class FilmListView(ListView):
    model = Film
    template_name = 'films/film_list.html'
    context_object_name = 'films'
    paginate_by = 10

    def get_queryset(self):
        return Film.objects.select_related('director').prefetch_related('genres', 'actors')


class FilmDetailView(DetailView):
    model = Film
    template_name = 'films/film_detail.html'
    context_object_name = 'film'

    def get_queryset(self):
        return Film.objects.select_related('director').prefetch_related('genres', 'actors')

    def get(self, request, *args, **kwargs):
        response = super().get(request, *args, **kwargs)
        FilmStats.objects.filter(film=self.object).update(views_count=F('views_count') + 1)
        return response
```

---

## Вопросы для проверки

1. Какие два атрибута класса минимально нужны для работающего `ListView`?
2. Зачем нужен `context_object_name` и что произойдёт, если его не указать?
3. Почему при переопределении `get()` в `DetailView` важно вызвать `super().get(request, *args, **kwargs)` в начале, а не в конце метода?
4. В чём разница между `slug_field` и `slug_url_kwarg`?
5. Что заменяет `get_queryset()` в `ListView` и `DetailView` по сравнению с FBV?

---

## Практическая задача

**Тип: переведи**

Переведи представление `director_detail` (страница режиссёра со списком его фильмов, из урока 10) с FBV на `DetailView`.

Исходная FBV-версия:

```python
def director_detail(request, slug):
    director = get_object_or_404(Director, slug=slug)
    films = director.films.select_related('director').prefetch_related('genres')
    context = {'director': director, 'films': films}
    return render(request, 'films/director_detail.html', context)
```

**Требования:**

1. Создай класс `DirectorDetailView`, наследующий `DetailView`
2. Настрой поиск по `slug`
3. Шаблон `director_detail.html` ожидает переменные `director` и `films` — обеспечь их через `context_object_name` и `get_context_data()`
4. Обнови маршрут в `films/urls.py`

---

[Предыдущий урок](lesson19.md) | [Следующий урок](lesson21.md)