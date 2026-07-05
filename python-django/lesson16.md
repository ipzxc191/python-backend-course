# Урок 16. Form и ModelForm: создание, отображение, сохранение в БД

## От админки к публичной части сайта

Все три прошлых модуля решали задачу управления контентом — модели, ORM, админ-панель. Это инструменты для администратора сайта. Но обычные посетители не имеют доступа к `/admin/`, и им нужен собственный способ взаимодействия с сайтом — например, оставить рецензию на фильм.

Формы — это механизм Django для приёма данных от пользователя: отображение полей ввода, валидация, преобразование введённых данных в Python-объекты.

> **Связь с прошлым курсом.** В FastAPI мы валидировали входящие данные через Pydantic-схемы — описывали поля, типы, ограничения, и фреймворк сам проверял запрос. Django Forms решает ту же задачу для HTML-форм: описываем поля один раз, получаем и валидацию, и автоматическую генерацию HTML.

---

## HTML-форма без Django Forms

Прежде чем подключать `Form`, вспомним, как формы работают на уровне браузера — в уроке 3 мы уже разбирали `request.POST`, но без реальной HTML-формы.

```html
<form method="POST" action="{% url 'films:add_film' %}">
    {% csrf_token %}
    <input type="text" name="title" placeholder="Название фильма">
    <input type="number" name="year" placeholder="Год выпуска">
    <button type="submit">Добавить</button>
</form>
```

Тег `{% csrf_token %}` — обязательный элемент любой POST-формы в Django. Он вставляет скрытое поле с токеном защиты от CSRF-атак (Cross-Site Request Forgery). Без него Django вернёт ошибку 403 при отправке формы.

Эта форма работает, но всю валидацию пришлось бы писать руками в представлении — проверять, что `title` не пустой, что `year` это число в разумном диапазоне. Django Forms берёт эту работу на себя.

---

## Класс Form

`Form` — это класс, который описывает набор полей формы, независимо от модели. Полезен для форм, которые не связаны напрямую с одной таблицей — например, форма поиска или форма обратной связи.

```python
# films/forms.py
from django import forms


class ReviewSearchForm(forms.Form):
    query = forms.CharField(
        max_length=200,
        required=False,
        label='Поиск рецензий',
        widget=forms.TextInput(attrs={'placeholder': 'Введите текст для поиска'})
    )
    min_rating = forms.IntegerField(
        required=False,
        min_value=1,
        max_value=10,
        label='Минимальный рейтинг'
    )
```

Каждый атрибут класса — это поле формы, аналогично тому, как атрибуты `Model` — это поля таблицы.

### Основные типы полей

| Поле формы | HTML-виджет по умолчанию | Python-тип после валидации |
|---|---|---|
| `CharField` | `<input type="text">` | `str` |
| `IntegerField` | `<input type="number">` | `int` |
| `DecimalField` | `<input type="number">` | `Decimal` |
| `BooleanField` | `<input type="checkbox">` | `bool` |
| `ChoiceField` | `<select>` | значение из списка |
| `DateField` | `<input type="date">` | `date` |
| `EmailField` | `<input type="email">` | `str` (с проверкой формата) |

### Параметры полей формы

| Параметр | Назначение |
|---|---|
| `required=False` | Поле необязательно (по умолчанию все поля обязательны) |
| `label='...'` | Текст подписи к полю |
| `help_text='...'` | Вспомогательный текст под полем |
| `widget=...` | Виджет — конкретный HTML-элемент для отображения |
| `initial=...` | Значение по умолчанию |

---

## Отображение формы в шаблоне

Представление, которое создаёт и передаёт форму в шаблон:

```python
# films/views.py
from .forms import ReviewSearchForm

def review_search(request):
    form = ReviewSearchForm()
    return render(request, 'films/review_search.html', {'form': form})
```

Django умеет рендерить всю форму целиком или поле за полем.

### Рендеринг целиком

```html
<!-- films/templates/films/review_search.html -->
{% extends 'base.html' %}

{% block content %}
    <h1>Поиск рецензий</h1>
    <form method="GET">
        {{ form.as_p }}
        <button type="submit">Найти</button>
    </form>
{% endblock %}
```

`{{ form.as_p }}` оборачивает каждое поле в тег `<p>`. Альтернативы: `{{ form.as_table }}` (для использования внутри `<table>`) и `{{ form.as_ul }}` (список). На практике для собственного оформления чаще рендерят поля вручную.

### Рендеринг по отдельным полям

```html
<form method="GET">
    <div>
        {{ form.query.label_tag }}
        {{ form.query }}
        {% if form.query.errors %}
            <span class="error">{{ form.query.errors }}</span>
        {% endif %}
    </div>
    <div>
        {{ form.min_rating.label_tag }}
        {{ form.min_rating }}
    </div>
    <button type="submit">Найти</button>
</form>
```

Такой подход даёт полный контроль над HTML-структурой — каждое поле, подпись и блок ошибок размещаются именно там, где нужно.

---

## ModelForm — форма из модели

Если форма должна создавать или редактировать объект модели — писать поля вручную в `Form` избыточно: они дублируют то, что уже описано в `models.py`. `ModelForm` решает эту проблему — генерирует поля формы автоматически на основе модели.

Сделаем форму для добавления рецензии. Сначала добавим модель `Review`, которую мы анонсировали в самой первой схеме курса:

```python
# films/models.py
class Review(models.Model):
    film = models.ForeignKey(
        Film, on_delete=models.CASCADE, related_name='reviews', verbose_name='Фильм'
    )
    author_name = models.CharField(max_length=100, verbose_name='Имя автора')
    text = models.TextField(verbose_name='Текст рецензии')
    rating = models.PositiveSmallIntegerField(verbose_name='Оценка')
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ['-created_at']
        verbose_name = 'Рецензия'
        verbose_name_plural = 'Рецензии'

    def __str__(self):
        return f'Рецензия на «{self.film.title}» от {self.author_name}'
```

Создаём и применяем миграцию:

```bash
python manage.py makemigrations
python manage.py migrate
```

Теперь создаём `ModelForm`:

```python
# films/forms.py
from django import forms
from .models import Review


class ReviewForm(forms.ModelForm):
    class Meta:
        model = Review
        fields = ['author_name', 'text', 'rating']
```

Вложенный класс `Meta` — обязательная часть `ModelForm`. `model` указывает, на основе какой модели строится форма, `fields` — какие поля модели включить.

> Обрати внимание: поле `film` не указано в `fields`. Это намеренно — фильм, к которому относится рецензия, мы определим в представлении из URL, а не дадим пользователю выбирать его произвольно из выпадающего списка.

### Альтернатива: exclude вместо fields

```python
class ReviewForm(forms.ModelForm):
    class Meta:
        model = Review
        exclude = ['film', 'created_at']
```

`exclude` указывает поля, которые нужно **исключить** — все остальные поля модели попадут в форму автоматически. Использовать `fields` явно безопаснее: если в модель добавится новое поле, `exclude` рискует случайно открыть его пользователю, тогда как `fields` нужно расширять осознанно.

### Настройка виджетов в ModelForm

```python
class ReviewForm(forms.ModelForm):
    class Meta:
        model = Review
        fields = ['author_name', 'text', 'rating']
        widgets = {
            'text': forms.Textarea(attrs={'rows': 4, 'placeholder': 'Поделитесь впечатлениями...'}),
            'rating': forms.NumberInput(attrs={'min': 1, 'max': 10}),
        }
        labels = {
            'author_name': 'Ваше имя',
            'text': 'Рецензия',
            'rating': 'Оценка (1–10)',
        }
```

---

## Обработка формы в представлении

Теперь главное — представление, которое показывает форму и обрабатывает её отправку. Это паттерн, который повторяется почти на каждой форме в Django:

```python
# films/views.py
from django.shortcuts import render, get_object_or_404, redirect
from .forms import ReviewForm
from .models import Film


def add_review(request, slug):
    film = get_object_or_404(Film, slug=slug)

    if request.method == 'POST':
        form = ReviewForm(request.POST)
        if form.is_valid():
            review = form.save(commit=False)  # создаём объект, но пока не сохраняем
            review.film = film                  # привязываем фильм из URL
            review.save()                       # теперь сохраняем
            return redirect(film.get_absolute_url())
    else:
        form = ReviewForm()

    return render(request, 'films/add_review.html', {'form': form, 'film': film})
```

Разберём логику пошагово:

1. **GET-запрос** — пользователь впервые открывает страницу. Создаём пустую форму (`ReviewForm()`) и показываем её.
2. **POST-запрос** — пользователь отправил данные. Создаём форму, заполненную данными из `request.POST`.
3. **`form.is_valid()`** — запускает валидацию всех полей. Возвращает `True`, если всё корректно.
4. **`form.save(commit=False)`** — создаёт объект модели в памяти, но не сохраняет в БД. Это нужно, когда требуется что-то изменить перед сохранением — в нашем случае привязать `film`, которого нет в самой форме.
5. **`review.save()`** — теперь сохраняем объект в базу, со всеми полями заполненными.
6. **`redirect(...)`** — паттерн PRG из урока 3: после успешной отправки — перенаправление, а не повторный рендер той же страницы.

Если форма невалидна — `is_valid()` вернёт `False`, и код просто провалится дальше до `render()`, который покажет ту же форму, но уже с ошибками валидации внутри объекта `form`.

### Шаблон формы добавления рецензии

```html
<!-- films/templates/films/add_review.html -->
{% extends 'base.html' %}

{% block title %}Рецензия на {{ film.title }}{% endblock %}

{% block content %}
    <h1>Оставить рецензию на «{{ film.title }}»</h1>

    <form method="POST">
        {% csrf_token %}
        {{ form.as_p }}
        <button type="submit">Опубликовать</button>
    </form>

    <a href="{{ film.get_absolute_url }}">← Назад к фильму</a>
{% endblock %}
```

Не забываем добавить маршрут:

```python
# films/urls.py
path('films/<slug:slug>/review/', views.add_review, name='add_review'),
```

---

## save(commit=False) — когда это нужно

Этот приём стоит закрепить отдельно, потому что он встречается очень часто. Есть два случая для `ModelForm.save()`:

```python
# Обычный save() — сразу создаёт и сохраняет объект
form = ReviewForm(request.POST)
if form.is_valid():
    form.save()  # объект полностью готов, все поля есть в форме
```

```python
# save(commit=False) — когда нужно дополнить объект перед сохранением
form = ReviewForm(request.POST)
if form.is_valid():
    review = form.save(commit=False)
    review.film = film          # поле не было в форме, добавляем сами
    review.author = request.user  # появится в модуле 7 — привязка к авторизованному пользователю
    review.save()
```

`commit=False` не означает «отменить сохранение» — это означает «дай мне объект, прежде чем класть его в базу».

---

## Подводные камни

### Забытый {% csrf_token %}

Самая частая ошибка новичков. Без `{% csrf_token %}` Django вернёт `403 Forbidden` при попытке отправить форму методом POST:

```html
<!-- Забыли csrf_token — форма не отправится -->
<form method="POST">
    {{ form.as_p }}
</form>

<!-- Правильно -->
<form method="POST">
    {% csrf_token %}
    {{ form.as_p }}
</form>
```

### form.save(commit=False) без последующего save()

Если вызвать `commit=False`, но забыть `review.save()` после — объект просто потеряется, ничего не запишется в базу. `commit=False` всегда требует ручного `save()` в конце.

### Поле не в fields, но обязательно на уровне модели

Если поле модели обязательно (`null=False`, без `default`) и не включено в `fields` формы — `form.save()` упадёт с ошибкой целостности базы данных, потому что Django попытается сохранить объект без значения для этого поля. Решение — либо включить поле в форму, либо заполнить его в представлении до `save()`, как мы сделали с `review.film = film`.

### is_valid() обязателен перед save()

`save()` нельзя вызывать без предварительной проверки `is_valid()` — данные могут быть некорректны, и `ModelForm` не защитит от сохранения невалидных данных без этой проверки:

```python
# Опасно — данные не проверены
form = ReviewForm(request.POST)
form.save()

# Правильно — сначала проверка
form = ReviewForm(request.POST)
if form.is_valid():
    form.save()
```

---

## Вопросы

1. В чём принципиальная разница между `Form` и `ModelForm`?
2. Зачем нужен `{% csrf_token %}` в форме и что произойдёт без него?
3. Что делает `form.save(commit=False)` и когда это нужно?
4. Чем `fields` лучше `exclude` в `Meta` класса `ModelForm`?
5. Что произойдёт, если вызвать `form.save()` без предварительной проверки `form.is_valid()`?

---

## Практическая задача

**Тип: расширь проект**

Создай форму для добавления нового фильма через публичную часть сайта (не через админку).

**Требования:**

1. Создай `ModelForm` с именем `FilmForm` на основе модели `Film`, включи поля: `title`, `year`, `description`, `director`, `genres`
2. Настрой виджет для поля `description` — `Textarea` с атрибутом `rows=5`
3. Обнови представление `add_film` (уже существует с урока 1, сейчас работает с заглушкой) — оно должно показывать форму при GET-запросе и сохранять фильм при успешной валидации POST-запроса
4. После успешного сохранения — перенаправление на страницу созданного фильма через `get_absolute_url()`
5. Обнови шаблон, чтобы он показывал форму

---

[Предыдущий урок](lesson15.md) | [Следующий урок](lesson17.md)