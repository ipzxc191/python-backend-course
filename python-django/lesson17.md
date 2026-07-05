# Урок 17. Валидация полей. Пользовательские валидаторы

## Где живёт валидация в Django

В прошлом уроке мы вызывали `form.is_valid()` и доверяли Django самостоятельно проверить данные. Сегодня разберём, что происходит внутри этого вызова и как добавлять собственные правила проверки.

Валидация в Django происходит на нескольких уровнях, и важно понимать разницу между ними:

1. **Валидация на уровне поля формы** — встроенные проверки типа и базовых правил (`required`, `max_length`, диапазон чисел)
2. **Валидаторы (validators)** — переиспользуемые функции проверки, которые можно прикрепить к полю модели или формы
3. **clean_<field>()** — собственная логика проверки конкретного поля формы
4. **clean()** — проверка, затрагивающая несколько полей одновременно

Разберём каждый уровень.

---

## Встроенная валидация

Когда вызывается `form.is_valid()`, Django проверяет каждое поле согласно его типу и параметрам:

```python
class ReviewForm(forms.ModelForm):
    class Meta:
        model = Review
        fields = ['author_name', 'text', 'rating']
```

Поле `rating` в модели — `PositiveSmallIntegerField`. Django автоматически проверит, что введённое значение — целое положительное число. Если пользователь введёт `'abc'` или `-5` — форма будет невалидна без единой строчки нашего кода.

### Доступ к ошибкам валидации

```python
form = ReviewForm(request.POST)
if not form.is_valid():
    print(form.errors)
    # {'rating': ['Введите целое число.']}
```

В шаблоне ошибки доступны через атрибут поля:

```html
{{ form.rating }}
{% if form.rating.errors %}
    <span class="error">{{ form.rating.errors }}</span>
{% endif %}
```

---

## Встроенные валидаторы Django

Django предоставляет набор готовых функций-валидаторов в `django.core.validators`. Их можно прикреплять к полям модели:

```python
# films/models.py
from django.core.validators import MinValueValidator, MaxValueValidator


class Review(models.Model):
    film = models.ForeignKey(Film, on_delete=models.CASCADE, related_name='reviews')
    author_name = models.CharField(max_length=100)
    text = models.TextField()
    rating = models.PositiveSmallIntegerField(
        validators=[MinValueValidator(1), MaxValueValidator(10)],
        help_text='Оценка от 1 до 10'
    )
    created_at = models.DateTimeField(auto_now_add=True)
```

Важная деталь: `PositiveSmallIntegerField` сам по себе не ограничивает максимум — он просто запрещает отрицательные числа. Без `MaxValueValidator(10)` пользователь мог бы поставить оценку `1000`. Валидаторы добавляют именно те ограничения, которые не покрываются типом поля.

### Часто используемые встроенные валидаторы

| Валидатор | Назначение |
|---|---|
| `MinValueValidator(N)` | Минимальное числовое значение |
| `MaxValueValidator(N)` | Максимальное числовое значение |
| `MinLengthValidator(N)` | Минимальная длина строки |
| `MaxLengthValidator(N)` | Максимальная длина строки (CharField уже учитывает `max_length`) |
| `EmailValidator()` | Проверка формата email (применяется автоматически в `EmailField`) |
| `URLValidator()` | Проверка формата URL |
| `RegexValidator(regex, message)` | Проверка по регулярному выражению |

### Пример с RegexValidator

```python
from django.core.validators import RegexValidator

phone_validator = RegexValidator(
    regex=r'^\+?\d{10,15}$',
    message='Номер телефона должен содержать от 10 до 15 цифр, можно с символом + в начале.'
)


class UserProfile(models.Model):
    phone = models.CharField(
        max_length=16,
        validators=[phone_validator],
        blank=True
    )
```

Эта модель появится полноценно в модуле 7 — пока используем как пример синтаксиса.

---

## Пользовательские валидаторы

Если встроенных валидаторов недостаточно — пишем собственную функцию. Валидатор — это обычная функция, которая принимает значение и выбрасывает `ValidationError`, если значение некорректно.

### Валидатор для модели

Создадим валидатор, который проверяет, что год выпуска фильма не превышает текущий и не слишком стар (например, до 1888 года — раньше не было кинематографа):

```python
# films/validators.py
from datetime import date
from django.core.exceptions import ValidationError


def validate_film_year(value):
    current_year = date.today().year
    if value > current_year:
        raise ValidationError(
            f'Год выпуска не может быть больше текущего ({current_year}).'
        )
    if value < 1888:
        raise ValidationError(
            'Год выпуска не может быть раньше 1888 года — до этого кинематографа не существовало.'
        )
```

Подключаем к полю модели:

```python
# films/models.py
from .validators import validate_film_year


class Film(models.Model):
    title = models.CharField(max_length=200)
    year = models.PositiveIntegerField(validators=[validate_film_year])
    # ... остальные поля
```

Создаём отдельный файл `validators.py` — это хорошая практика: валидаторы переиспользуются между моделями и формами, логично держать их в одном месте, а не разбрасывать по `models.py`.

### Важная деталь: full_clean() и save()

Валидаторы, прикреплённые к полю модели через `validators=[...]`, **не вызываются автоматически** при обычном `save()`. Они срабатывают при вызове `full_clean()` — а `full_clean()` Django вызывает сам только при работе через `ModelForm`. Если сохранять объект напрямую через `Model.objects.create()`, валидаторы поля не сработают:

```python
# Валидатор validate_film_year НЕ сработает
Film.objects.create(title='Фильм из будущего', year=2099)

# Валидатор сработает — full_clean() вызывается ModelForm автоматически
form = FilmForm(data={'title': 'Фильм из будущего', 'year': 2099, ...})
form.is_valid()  # вернёт False, ошибка в form.errors

# Можно вызвать вручную, если сохраняешь не через форму
film = Film(title='Фильм из будущего', year=2099)
film.full_clean()  # выбросит ValidationError
film.save()
```

Это частый источник путаницы: валидация модели надёжно работает только через формы или явный `full_clean()`. Прямое создание объектов (как мы делали в shell в уроке 9) валидаторы поля не запускает.

---

## clean_<field>() — валидация конкретного поля формы

Если проверка специфична именно для формы (а не для модели в целом), удобнее писать её прямо в классе формы — метод `clean_<имя_поля>()`:

```python
# films/forms.py
from django import forms
from .models import Review


class ReviewForm(forms.ModelForm):
    class Meta:
        model = Review
        fields = ['author_name', 'text', 'rating']
        widgets = {
            'text': forms.Textarea(attrs={'rows': 4}),
        }

    def clean_text(self):
        text = self.cleaned_data['text']
        if len(text.split()) < 5:
            raise forms.ValidationError(
                'Рецензия слишком короткая — напишите хотя бы 5 слов.'
            )
        return text
```

Правила работы с `clean_<field>()`:

- Метод вызывается автоматически внутри `is_valid()`, для каждого поля, у которого есть такой метод
- Значение поля доступно через `self.cleaned_data['имя_поля']` — это уже проверенное и преобразованное в нужный тип значение
- Метод обязан **вернуть** значение — даже если оно не изменилось. Если забыть `return` — поле потеряет своё значение
- Чтобы пометить поле как невалидное — выбрасываем `forms.ValidationError(сообщение)`

### Ещё пример — проверка имени автора

```python
def clean_author_name(self):
    name = self.cleaned_data['author_name']
    if name.strip().lower() == 'анонимный критик':
        raise forms.ValidationError('Это имя зарезервировано системой, выберите другое.')
    return name.strip()
```

Обрати внимание: метод возвращает `name.strip()`, а не исходное `name` — это пример, как `clean_<field>()` может не только проверять, но и преобразовывать значение перед сохранением.

---

## clean() — валидация нескольких полей одновременно

Иногда правило затрагивает сразу несколько полей — например, проверка, что одно значение не превышает другое. Для этого переопределяется метод `clean()` без суффикса:

```python
class ReviewForm(forms.ModelForm):
    class Meta:
        model = Review
        fields = ['author_name', 'text', 'rating']
        widgets = {
            'text': forms.Textarea(attrs={'rows': 4}),
        }

    def clean_text(self):
        text = self.cleaned_data['text']
        if len(text.split()) < 5:
            raise forms.ValidationError('Рецензия слишком короткая.')
        return text

    def clean(self):
        cleaned_data = super().clean()
        rating = cleaned_data.get('rating')
        text = cleaned_data.get('text')

        if rating and rating <= 3 and text and len(text) < 50:
            raise forms.ValidationError(
                'Для низкой оценки (3 и ниже) распишите причину подробнее — минимум 50 символов.'
            )

        return cleaned_data
```

Важные детали:

- `super().clean()` вызывается первым — он запускает накопленную логику родительского класса и возвращает словарь `cleaned_data`
- Используем `.get()`, а не `['rating']` напрямую — если поле `rating` не прошло валидацию на предыдущем этапе, его не будет в `cleaned_data`, и обращение по ключу выбросит `KeyError`
- `clean()` без аргументов привязывает ошибку не к конкретному полю, а к форме в целом — такая ошибка отображается отдельно от полей, обычно наверху формы

### Привязка ошибки clean() к конкретному полю

Если ошибка из `clean()` логически относится к одному полю, можно явно прикрепить её туда через `add_error()`:

```python
def clean(self):
    cleaned_data = super().clean()
    rating = cleaned_data.get('rating')
    text = cleaned_data.get('text')

    if rating and rating <= 3 and text and len(text) < 50:
        self.add_error('text', 'Для низкой оценки распишите причину подробнее.')

    return cleaned_data
```

---

## Отображение ошибок формы в шаблоне

Обновим шаблон добавления рецензии, чтобы корректно показывать и ошибки полей, и общие ошибки формы:

```html
<!-- films/templates/films/add_review.html -->
{% extends 'base.html' %}

{% block content %}
    <h1>Оставить рецензию на «{{ film.title }}»</h1>

    <form method="POST">
        {% csrf_token %}

        {% if form.non_field_errors %}
            <div class="form-errors">
                {{ form.non_field_errors }}
            </div>
        {% endif %}

        {% for field in form %}
            <div class="form-field">
                {{ field.label_tag }}
                {{ field }}
                {% if field.errors %}
                    <span class="error">{{ field.errors }}</span>
                {% endif %}
                {% if field.help_text %}
                    <small>{{ field.help_text }}</small>
                {% endif %}
            </div>
        {% endfor %}

        <button type="submit">Опубликовать</button>
    </form>
{% endblock %}
```

`form.non_field_errors` — это ошибки, добавленные через `clean()` без привязки к конкретному полю (то есть без `add_error()`). Цикл `{% for field in form %}` перебирает все поля формы одним блоком — удобно, когда не нужно настраивать каждое поле отдельно, но хочется единообразного оформления.

---

## Подводные камни

### Забытый return в clean_<field>()

```python
# Забыли return — значение поля станет None
def clean_author_name(self):
    name = self.cleaned_data['author_name']
    if not name.strip():
        raise forms.ValidationError('Имя не может быть пустым.')
    # забыли вернуть name!

# Правильно
def clean_author_name(self):
    name = self.cleaned_data['author_name']
    if not name.strip():
        raise forms.ValidationError('Имя не может быть пустым.')
    return name
```

### cleaned_data.get() вместо прямого обращения в clean()

```python
# Если rating не прошёл валидацию на этапе поля — здесь упадёт KeyError
def clean(self):
    cleaned_data = super().clean()
    rating = cleaned_data['rating']

# .get() безопасно возвращает None, если поля нет
def clean(self):
    cleaned_data = super().clean()
    rating = cleaned_data.get('rating')
```

### Валидаторы поля модели не срабатывают вне формы

Уже разобрали выше, но это настолько частая ошибка, что стоит повторить отдельно: `Model.objects.create()` и `model_instance.save()` не вызывают `full_clean()` автоматически. Если данные приходят не через форму (например, из внешнего API или скрипта импорта) — валидацию нужно вызывать явно через `full_clean()`.

---

## Итоговый ReviewForm

```python
# films/forms.py
from django import forms
from .models import Review


class ReviewForm(forms.ModelForm):
    class Meta:
        model = Review
        fields = ['author_name', 'text', 'rating']
        widgets = {
            'text': forms.Textarea(attrs={'rows': 4, 'placeholder': 'Поделитесь впечатлениями...'}),
        }
        labels = {
            'author_name': 'Ваше имя',
            'text': 'Рецензия',
            'rating': 'Оценка (1–10)',
        }

    def clean_text(self):
        text = self.cleaned_data['text']
        if len(text.split()) < 5:
            raise forms.ValidationError('Рецензия слишком короткая — напишите хотя бы 5 слов.')
        return text

    def clean_author_name(self):
        name = self.cleaned_data['author_name']
        if name.strip().lower() == 'анонимный критик':
            raise forms.ValidationError('Это имя зарезервировано системой, выберите другое.')
        return name.strip()

    def clean(self):
        cleaned_data = super().clean()
        rating = cleaned_data.get('rating')
        text = cleaned_data.get('text')

        if rating and rating <= 3 and text and len(text) < 50:
            self.add_error('text', 'Для низкой оценки распишите причину подробнее — минимум 50 символов.')

        return cleaned_data
```

---

## Вопросы

1. В чём разница между валидатором модели (`validators=[...]`) и методом `clean_<field>()` в форме?
2. Что произойдёт, если в `clean_<field>()` забыть написать `return`?
3. Почему `full_clean()` не вызывается автоматически при `Model.objects.create()`?
4. Когда нужно переопределять `clean()`, а когда достаточно `clean_<field>()`?
5. Что возвращает `form.non_field_errors` и чем это отличается от `form.errors`?

---

## Практическая задача

**Тип: допиши код**

Тебе нужно добавить валидацию в форму `FilmForm` из прошлого урока. Допиши пропущенные части.

**Требования к валидации:**

1. Поле `title` не должно состоять только из цифр (например, нельзя назвать фильм `'12345'`)
2. Если жанр «Комедия» выбран одновременно с жанром «Трагедия» — форма должна показать общую ошибку (это смысловое противоречие для нашего каталога)

```python
class FilmForm(forms.ModelForm):
    class Meta:
        model = Film
        fields = ['title', 'year', 'description', 'director', 'genres']
        widgets = {
            'description': forms.Textarea(attrs={'rows': 5}),
        }

    def clean_title(self):
        title = ___________________________
        if ___________________________:
            raise forms.ValidationError('Название фильма не может состоять только из цифр.')
        return title

    def clean(self):
        cleaned_data = ___________________________
        genres = cleaned_data.get('genres')

        if genres:
            genre_names = [___________________________]
            if 'Комедия' in genre_names and 'Трагедия' in genre_names:
                ___________________________

        return cleaned_data
```

---

[Предыдущий урок](lesson16.md) | [Следующий урок](lesson18.md)