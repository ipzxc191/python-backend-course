# Урок 19. Введение в CBV: View и TemplateView. Когда FBV, когда CBV

## Два способа писать представления

Все представления, которые мы написали за пять модулей, — это функции. В Django их называют FBV (Function Based Views). Но Django предоставляет и альтернативный подход — CBV (Class Based Views), где представление описывается классом.

Это не новая концепция, а другой способ организации того же самого кода. Оба подхода полностью взаимозаменяемы — Django одинаково хорошо работает и с функциями, и с классами в `urls.py`.

---

## Зачем нужны классы, если функции работают

Рассмотрим конкретную проблему. У нас уже есть несколько похожих представлений:

```python
def film_list(request):
    films = Film.objects.select_related('director').prefetch_related('genres')
    return render(request, 'films/film_list.html', {'films': films})


def about(request):
    return render(request, 'films/about.html', {'title': 'О нашем сайте'})
```

Оба представления делают одно и то же по структуре: получить данные, отрендерить шаблон. Если в проекте десятки таких простых страниц — код повторяется. CBV даёт способ выразить эту структуру через наследование, вынося общую логику в базовые классы, которые Django уже написал за нас.

> Важно сразу избавиться от заблуждения: CBV — это не более «правильный» или «продвинутый» способ писать представления. Это просто другой инструмент. Профессиональные Django-проекты используют оба подхода одновременно, выбирая каждый раз по ситуации.

---

## Класс View — основа всех CBV

Самый базовый класс — `django.views.generic.View`. Все остальные CBV в Django наследуются от него, прямо или через промежуточные классы.

```python
# films/views.py
from django.views.generic import View
from django.http import HttpResponse


class HelloView(View):
    def get(self, request):
        return HttpResponse('Привет из классового представления!')
```

Ключевая идея `View`: вместо одной функции, которая проверяет `request.method` через `if`, — отдельный метод для каждого HTTP-метода. `get()` обрабатывает GET-запросы, `post()` — POST-запросы, и так далее.

```python
class AddFilmView(View):
    def get(self, request):
        form = FilmForm()
        return render(request, 'films/add_film.html', {'form': form})

    def post(self, request):
        form = FilmForm(request.POST, request.FILES)
        if form.is_valid():
            film = form.save()
            return redirect(film.get_absolute_url())
        return render(request, 'films/add_film.html', {'form': form})
```

Сравним с FBV-версией того же представления из урока 18:

```python
# FBV-версия — для сравнения
def add_film(request):
    if request.method == 'POST':
        form = FilmForm(request.POST, request.FILES)
        if form.is_valid():
            film = form.save()
            return redirect(film.get_absolute_url())
    else:
        form = FilmForm()
    return render(request, 'films/add_film.html', {'form': form})
```

Разница пока невелика — это ожидаемо для простого случая. Преимущество CBV раскроется, когда мы перейдём к готовым базовым классам — `ListView`, `DetailView`, `CreateView` в следующих уроках, где Django уже реализовал типовую логику внутри.

### Подключение в urls.py

CBV нельзя передать в `path()` как обычный класс — нужно вызвать специальный метод `as_view()`, который превращает класс в функцию, понятную Django:

```python
# films/urls.py
from .views import AddFilmView

urlpatterns = [
    # ...
    path('films/add/', AddFilmView.as_view(), name='add_film'),
]
```

`as_view()` — это фабричный метод, который на каждый запрос создаёт новый экземпляр класса и вызывает у него подходящий метод (`get`, `post` и так далее) в зависимости от метода HTTP-запроса.

---

## TemplateView — для простых страниц

`TemplateView` — это специализированное представление, построенное поверх View. Оно предназначено для отображения HTML-шаблонов и уже содержит готовую реализацию обработки **GET-запроса**. Готовый класс для страниц, которым нужно просто отрендерить шаблон без сложной логики. Он отлично подходит для статических страниц вроде «О сайте», «Контакты», «Политика конфиденциальности» и подобных.

Самый простой вариант использования выглядит так:

```python
from django.views.generic import TemplateView


class AboutView(TemplateView):
    template_name = 'films/about.html'
    extra_context = {
        'title': 'О нашем сайте',
    }
```

Здесь используются два атрибута класса:

- `template_name` — указывает, какой шаблон нужно отобразить.
- `extra_context` — позволяет передать в шаблон дополнительные постоянные данные.

Такой подход хорошо подходит для статического контекста, который не зависит от запроса или базы данных.

---

### Когда `extra_context` уже недостаточно

На практике часто возникает необходимость передать в шаблон данные, которые вычисляются во время выполнения запроса.

Например:

- количество фильмов в базе;
- список жанров;
- текущего пользователя;
- результаты запросов к базе данных.

В таких случаях используют метод `get_context_data()`.

```python
class AboutView(TemplateView):
    template_name = 'films/about.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['title'] = 'О нашем сайте'
        context['film_count'] = Film.objects.count()
        return context
```

Именно этот способ чаще всего используется в реальных проектах, потому что он позволяет сформировать контекст любой сложности.

Разберём этот паттерн подробнее, поскольку он будет постоянно встречаться во всех следующих CBV.

### get_context_data() — как это работает

`get_context_data()` — это метод, который формирует словарь контекста для шаблона. По умолчанию Django уже создает базовый контекст. Наша задача — получить его, дополнить своими данными и вернуть обратно.

```python
def get_context_data(self, **kwargs):
    context = super().get_context_data(**kwargs)   # 1. Получаем базовый контекст
    context['title'] = 'О нашем сайте'             # 2. Добавляем свои данные
    context['film_count'] = Film.objects.count()
    return context                                 # 3. Возвращаем итоговый словарь
```

Здесь происходит три последовательных шага:

1. Получаем базовый контекст родительского класса.
2. Добавляем собственные данные.
3. Возвращаем итоговый словарь.

Очень важно не забывать вызывать `super().get_context_data(**kwargs)`. Если этого не сделать, можно потерять данные, которые Django автоматически добавляет в контекст (например, параметры, полученные из URL).

### Подключение TemplateView в urls.py

```python
# films/urls.py
from .views import AboutView

urlpatterns = [
    path('about/', AboutView.as_view(), name='about'),
]
```

---

## Атрибуты класса вместо логики функции

Одна из ключевых идей CBV — заменять небольшие логические операции декларативными атрибутами класса. Это уже видно в `TemplateView.template_name` — вместо вызова `render(request, 'films/about.html')` мы просто объявляем имя шаблона как атрибут, а сам вызов `render()` происходит внутри родительского класса.

Эта идея будет развиваться в следующих уроках: `ListView` использует атрибут `model` вместо ручного вызова `Model.objects.all()`, `DetailView` — атрибуты `model` и `slug_field` вместо `get_object_or_404()`. Сейчас достаточно увидеть сам принцип на примере `template_name`.

---

## Сравнение FBV и CBV для нашего проекта

Перепишем `index` (главная страница) через `TemplateView` — это представление идеально подходит для демонстрации, потому что оно простое и статичное:

```python
# FBV-версия (была с урока 1)
def index(request):
    context = {'title': 'Лучшие фильмы всех времён'}
    return render(request, 'films/index.html', context)
```

```python
# CBV-версия
class IndexView(TemplateView):
    template_name = 'films/index.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['title'] = 'Лучшие фильмы всех времён'
        return context
```

```python
# films/urls.py
from .views import IndexView

urlpatterns = [
    path('', IndexView.as_view(), name='index'),
]
```

Обе версии абсолютно равноценны по результату. Какую выбрать — вопрос команды и личных предпочтений. Часто проект придерживается одного подхода для однотипных страниц, чтобы код был предсказуемым.

---

## Когда выбирать FBV, когда CBV

Это не строгое правило, а ориентир, который вырабатывается с опытом:

| Ситуация | Предпочтительный подход |
|---|---|
| Простая логика, один HTTP-метод | FBV — короче и нагляднее |
| Стандартный CRUD (список, деталь, создание, редактирование, удаление) | CBV с готовыми generic views — меньше повторяющегося кода |
| Сложная нестандартная логика, ветвления, несколько форм на одной странице | FBV — проще читать линейную логику |
| Нужно расширяемое поведение через наследование (например, общая логика для нескольких похожих страниц) | CBV — наследование классов для этого создано |
| API-эндпоинт с обработкой нескольких методов (GET, POST, PUT, DELETE) | CBV — структура `View` с отдельными методами естественно подходит |

> **Важная честная оговорка.** В реальных проектах эта грань размыта, и многие опытные разработчики Django предпочитают FBV почти везде, аргументируя это явностью и простотой отладки — в функции весь поток исполнения виден сразу, без необходимости прыгать по цепочке родительских классов. Другие предпочитают CBV за компактность для типовых задач. Мы изучим оба подхода, чтобы ты мог читать код, написанный в любом стиле, и сознательно выбирать инструмент под задачу.

---

## Подводные камни

### Забытый as_view()

```python
# TypeError — Django получит сам класс, а не функцию-обработчик
path('about/', AboutView, name='about'),

# Правильно — as_view() возвращает функцию
path('about/', AboutView.as_view(), name='about'),
```

### Забытый super().get_context_data(**kwargs)

```python
# Опасно — может потерять служебные данные, которые добавляет родительский класс
def get_context_data(self, **kwargs):
    return {'title': 'О сайте'}

# Правильно — сначала берём базовый контекст, потом дополняем
def get_context_data(self, **kwargs):
    context = super().get_context_data(**kwargs)
    context['title'] = 'О сайте'
    return context
```

### Смешивание методов в одном классе View без явного назначения

Если в `View` определить метод с произвольным именем (не `get`, `post`, `put`, `delete` и так далее) — Django его просто не вызовет, и запрос с соответствующим HTTP-методом получит `405 Method Not Allowed`:

```python
class AddFilmView(View):
    def get(self, request):
        ...

    def save_film(self, request):  # Django не знает про этот метод
        ...
```

Имя метода должно строго соответствовать HTTP-методу в нижнем регистре: `get`, `post`, `put`, `patch`, `delete`, `head`, `options`.

---

## Итоговый код урока

```python
# films/views.py
from django.views.generic import View, TemplateView
from django.shortcuts import render, redirect
from django.http import HttpResponse
from .forms import FilmForm
from .models import Film


class IndexView(TemplateView):
    template_name = 'films/index.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['title'] = 'Лучшие фильмы всех времён'
        return context


class AboutView(TemplateView):
    template_name = 'films/about.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['title'] = 'О нашем сайте'
        context['film_count'] = Film.objects.count()
        return context


class AddFilmView(View):
    def get(self, request):
        form = FilmForm()
        return render(request, 'films/add_film.html', {'form': form})

    def post(self, request):
        form = FilmForm(request.POST, request.FILES)
        if form.is_valid():
            film = form.save()
            return redirect(film.get_absolute_url())
        return render(request, 'films/add_film.html', {'form': form})
```

```python
# films/urls.py
from django.urls import path
from .views import IndexView, AboutView, AddFilmView
from . import views

app_name = 'films'

urlpatterns = [
    path('', IndexView.as_view(), name='index'),
    path('about/', AboutView.as_view(), name='about'),
    path('films/add/', AddFilmView.as_view(), name='add_film'),
    # остальные маршруты пока остаются как FBV — переведём в следующих уроках
    path('films/', views.film_list, name='film_list'),
    path('films/search/', views.search_film, name='search_film'),
    path('films/<slug:slug>/', views.film_detail, name='film_detail'),
    path('films/<slug:slug>/review/', views.add_review, name='add_review'),
    path('directors/<slug:slug>/', views.director_detail, name='director_detail'),
    path('stats/', views.catalog_stats, name='catalog_stats'),
]
```

---

## Вопросы для проверки

1. Зачем нужен метод `as_view()` при подключении CBV в `urls.py`?
2. В чём принципиальное отличие класса `View` от FBV в плане обработки разных HTTP-методов?
3. Зачем переопределять `get_context_data()`, а не просто передавать словарь напрямую, как в `render()`?
4. Что произойдёт, если в классе `View` определить метод с именем, не соответствующим HTTP-методу — например, `def handle_post(self, request)`?
5. Почему не существует универсального правила «всегда используй CBV» или «всегда используй FBV»?

---

## Практическая задача

**Тип: переведи**

Переведи представление `catalog_stats` (страница со статистикой каталога, написанная в уроке 11) с FBV на CBV, используя `TemplateView`.

**Требования:**

1. Создай класс `CatalogStatsView`, наследующий `TemplateView`
2. Укажи `template_name = 'films/catalog_stats.html'`
3. Перенеси всю логику из тела функции `catalog_stats` внутрь `get_context_data()`
4. Обнови маршрут в `films/urls.py`, заменив `views.catalog_stats` на `CatalogStatsView.as_view()`

Исходная FBV-версия для справки:

```python
def catalog_stats(request):
    overall_stats = Film.objects.aggregate(
        total=Count('id'),
        avg_rating=Avg('rating'),
        max_rating=Max('rating'),
        min_rating=Min('rating'),
    )
    top_directors = Director.objects.annotate(
        film_count=Count('films')
    ).filter(film_count__gt=0).order_by('-film_count')[:5]
    films_by_year = Film.objects.values('year').annotate(count=Count('id')).order_by('-year')

    context = {
        'overall_stats': overall_stats,
        'top_directors': top_directors,
        'films_by_year': films_by_year,
    }
    return render(request, 'films/catalog_stats.html', context)
```

---

[Предыдущий урок](lesson18.md) | [Следующий урок](lesson20.md)