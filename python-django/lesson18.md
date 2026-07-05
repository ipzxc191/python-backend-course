# Урок 18. Загрузка файлов (FileField, ImageField). Отображение медиафайлов

## Возвращаемся к разделению static и media

В уроке 7 мы разделили статические файлы (CSS, JS) и медиафайлы — с обещанием вернуться к медиа, когда дойдём до форм. Этот момент настал. Постеры фильмов, фотографии актёров и режиссёров — всё это медиафайлы, которые загружают пользователи или администраторы, и Django должен уметь их принимать, сохранять и отдавать обратно в браузер.

---

## Настройка медиафайлов

Прежде чем поле модели сможет хранить файл, нужно настроить, куда Django будет физически сохранять загруженные файлы и по какому URL их отдавать.

```python
# filmsite/settings.py

# URL-префикс для доступа к медиафайлам в браузере
MEDIA_URL = '/media/'

# Папка на диске, куда сохраняются загруженные файлы
MEDIA_ROOT = BASE_DIR / 'media'
```

В отличие от `STATIC_URL`/`STATICFILES_DIRS`, здесь только одна пара настроек — медиафайлы не распределены по приложениям, как статика, у них одна общая точка хранения.

### Подключение в режиме разработки

В режиме разработки Django сам не обслуживает медиафайлы — это нужно подключить явно в корневом `urls.py`:

```python
# filmsite/urls.py
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('films.urls')),
]

if settings.DEBUG:
    import debug_toolbar
    urlpatterns += [
        path('__debug__/', include(debug_toolbar.urls)),
    ]
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

Это та же конструкция `if settings.DEBUG`, что мы уже использовали для Debug Toolbar в уроке 12 — обслуживание медиафайлов через Django подходит только для разработки. В продакшне эту задачу берёт на себя Nginx, что мы настроим в модуле 9.

---

## ImageField и FileField

У нас уже есть поля `photo` в моделях `Director` и `Actor`, описанные ещё в уроке 10 — теперь разберём их подробно.

```python
# films/models.py
class Director(models.Model):
    name = models.CharField(max_length=200)
    bio = models.TextField(blank=True)
    photo = models.ImageField(
        upload_to='directors/',
        blank=True,
        verbose_name='Фото'
    )
```

`ImageField` — специализированная версия `FileField`, которая дополнительно проверяет, что загруженный файл действительно является изображением. `FileField` принимает любой тип файла без такой проверки.

### Параметр upload_to

`upload_to` определяет подпапку внутри `MEDIA_ROOT`, куда будет сохранён файл:

```python
photo = models.ImageField(upload_to='directors/')
```

Файл, загруженный для режиссёра с именем `coppola.jpg`, физически окажется по пути:

```
media/directors/coppola.jpg
```

А URL для доступа из браузера будет:

```
/media/directors/coppola.jpg
```

### upload_to как функция

Если нужна более сложная логика именования — например, организация по дате или по id объекта — `upload_to` может быть функцией:

```python
def director_photo_path(instance, filename):
    return f'directors/{instance.id}/{filename}'


class Director(models.Model):
    photo = models.ImageField(upload_to=director_photo_path, blank=True)
```

Функция принимает экземпляр модели и оригинальное имя файла, возвращает путь относительно `MEDIA_ROOT`.

### Добавляем postera для Film

Добавим постер к нашей основной модели:

```python
# films/models.py
class Film(models.Model):
    title = models.CharField(max_length=200)
    year = models.PositiveIntegerField(validators=[validate_film_year])
    description = models.TextField(blank=True)
    rating = models.DecimalField(max_digits=3, decimal_places=1, default=0.0)
    slug = models.SlugField(max_length=200, unique=True, blank=True)
    poster = models.ImageField(upload_to='posters/', blank=True, verbose_name='Постер')
    created_at = models.DateTimeField(auto_now_add=True)
    # ... остальные поля и связи
```

Создаём и применяем миграцию:

```bash
python manage.py makemigrations
python manage.py migrate
```

---

## Установка Pillow

`ImageField` требует библиотеку Pillow для обработки изображений — без неё Django при попытке использовать `ImageField` выбросит ошибку при старте сервера:

```bash
pip install Pillow
```

Это единственная внешняя зависимость, которая нужна для работы с изображениями — она упомянута в нашей таблице технологий ещё в начале курса.

---

## Форма с загрузкой файла

Загрузка файла через HTML-форму требует особого атрибута `enctype` — без него браузер не отправит содержимое файла, только его имя.

```html
<!-- Без enctype файл не отправится корректно -->
<form method="POST">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Сохранить</button>
</form>

<!-- Правильно -->
<form method="POST" enctype="multipart/form-data">
    {% csrf_token %}
    {{ form.as_p }}
    <button type="submit">Сохранить</button>
</form>
```

`enctype="multipart/form-data"` — обязательный атрибут для любой формы, содержащей поле для загрузки файла. Это легко забыть, и ошибка не всегда очевидна: форма отправляется без видимых проблем, но файл просто не попадает на сервер.

### Обновляем FilmForm

```python
# films/forms.py
class FilmForm(forms.ModelForm):
    class Meta:
        model = Film
        fields = ['title', 'year', 'description', 'director', 'genres', 'poster']
        widgets = {
            'description': forms.Textarea(attrs={'rows': 5}),
        }

    def clean_title(self):
        title = self.cleaned_data['title']
        if title.strip().isdigit():
            raise forms.ValidationError('Название фильма не может состоять только из цифр.')
        return title
```

### Обновляем представление — request.FILES

Файлы передаются в запросе отдельно от обычных текстовых данных — не в `request.POST`, а в `request.FILES`. Форма должна получить оба объекта:

```python
# films/views.py
def add_film(request):
    if request.method == 'POST':
        form = FilmForm(request.POST, request.FILES)  # обязательно передать оба
        if form.is_valid():
            film = form.save()
            return redirect(film.get_absolute_url())
    else:
        form = FilmForm()

    return render(request, 'films/add_film.html', {'form': form})
```

Если передать только `request.POST`, без `request.FILES` — форма будет считать, что файл не был загружен, даже если пользователь его выбрал в браузере.

```html
<!-- films/templates/films/add_film.html -->
{% extends 'base.html' %}

{% block title %}Добавить фильм{% endblock %}

{% block content %}
    <h1>Добавить новый фильм</h1>

    <form method="POST" enctype="multipart/form-data">
        {% csrf_token %}
        {{ form.as_p }}
        <button type="submit">Сохранить</button>
    </form>
{% endblock %}
```

---

## Отображение изображений в шаблоне

Когда `ImageField` заполнен, доступ к файлу из шаблона происходит через два главных атрибута:

```html
{{ film.poster.url }}    <!-- /media/posters/krestnyj-otec.jpg -->
{{ film.poster.name }}   <!-- posters/krestnyj-otec.jpg -->
```

`.url` — это полный URL для использования в `<img src="">`, `.name` — относительный путь от `MEDIA_ROOT`, без префикса `/media/`. В шаблонах почти всегда нужен именно `.url`.

```html
<!-- films/templates/films/film_detail.html -->
{% extends 'base.html' %}

{% block content %}
    <h1>{{ film.title }}</h1>

    {% if film.poster %}
        <img src="{{ film.poster.url }}" alt="Постер фильма «{{ film.title }}»" width="300">
    {% else %}
        <p>Постер пока не загружен.</p>
    {% endif %}

    <p>Год выпуска: {{ film.year }}</p>
    <!-- ... остальная разметка ... -->
{% endblock %}
```

Проверка `{% if film.poster %}` обязательна — поле `blank=True` означает, что постер может отсутствовать. Без этой проверки `<img src="">` сгенерирует пустой `src`, что вызовет лишний неудачный запрос в браузере.

### Карточка фильма с миниатюрой

Обновим `film_card.html`, чтобы каждая карточка в списке показывала постер:

```html
<!-- films/templates/films/includes/film_card.html -->
<div class="film-card">
    {% if film.poster %}
        <img src="{{ film.poster.url }}" alt="{{ film.title }}" width="150">
    {% endif %}
    <h3><a href="{{ film.get_absolute_url }}">{{ film.title }}</a></h3>
    <p>Год: {{ film.year }}</p>
    <p>{{ film.description|default:"Нет описания"|truncatechars:80 }}</p>
</div>
```

---

## Загрузка через админку

Здесь стоит вспомнить, что мы уже подготовили почву для этого в модуле 4: поля `ImageField` в админ-панели Django автоматически отображаются как стандартный виджет загрузки файла, без какой-либо дополнительной настройки. Достаточно открыть форму редактирования фильма в админке — поле `poster` уже будет показывать кнопку выбора файла и (если файл уже загружен) превью текущего изображения с возможностью его заменить или удалить.

---

## Валидация загружаемых файлов

### Ограничение размера файла

Django не ограничивает размер файла на уровне поля модели по умолчанию — для этого нужен собственный валидатор:

```python
# films/validators.py
from django.core.exceptions import ValidationError


def validate_file_size(value):
    max_size_mb = 5
    if value.size > max_size_mb * 1024 * 1024:
        raise ValidationError(f'Размер файла не должен превышать {max_size_mb} МБ.')
```

```python
# films/models.py
from .validators import validate_file_size


class Film(models.Model):
    poster = models.ImageField(
        upload_to='posters/',
        blank=True,
        validators=[validate_file_size]
    )
```

Помним из урока 17: этот валидатор сработает только при сохранении через форму (`is_valid()` вызывает `full_clean()`), не при прямом `Model.objects.create()`.

### Ограничение расширения файла

```python
from django.core.validators import FileExtensionValidator


class Film(models.Model):
    poster = models.ImageField(
        upload_to='posters/',
        blank=True,
        validators=[
            validate_file_size,
            FileExtensionValidator(allowed_extensions=['jpg', 'jpeg', 'png', 'webp'])
        ]
    )
```

`FileExtensionValidator` — встроенный валидатор Django, проверяет расширение файла по списку разрешённых.

---

## Подводные камни

### Забытый enctype="multipart/form-data"

Самая частая и самая незаметная ошибка с формами файлов. Форма отправляется без видимых ошибок, поле файла просто игнорируется браузером:

```html
<!-- Файл не передастся на сервер -->
<form method="POST">
```

```html
<!-- Обязательно для форм с файлами -->
<form method="POST" enctype="multipart/form-data">
```

### Забытый request.FILES в представлении

```python
# Файл потеряется — форма не получит данные из request.FILES
form = FilmForm(request.POST)

# Оба аргумента обязательны при работе с файлами
form = FilmForm(request.POST, request.FILES)
```

### Удаление объекта не удаляет файл с диска

Это важная и не всегда очевидная деталь: когда объект с `ImageField` удаляется через `obj.delete()` или массовое удаление, сам файл на диске **не удаляется** автоматически. Django хранит только путь к файлу в базе данных — удаление записи не означает удаление физического файла. Для автоматической очистки файлов при удалении объектов используют сторонние библиотеки (например, `django-cleanup`) — для учебного проекта это не критично, но важно знать об этом ограничении при проектировании реальных систем.

### Отсутствие проверки на пустое поле перед .url

```html
<!-- Если поле пустое, .url выбросит ValueError при рендеринге -->
<img src="{{ film.poster.url }}">

<!-- Сначала проверяем наличие файла -->
{% if film.poster %}
    <img src="{{ film.poster.url }}">
{% endif %}
```

---

## Итоговая структура проекта после урока

```
filmsite/
├── media/                          ← создаётся автоматически при первой загрузке
│   ├── posters/
│   ├── directors/
│   └── actors/
├── filmsite/
│   ├── settings.py                 ← MEDIA_URL, MEDIA_ROOT
│   └── urls.py                     ← static() для режима разработки
└── films/
    ├── models.py                   ← ImageField на Film, Director, Actor
    ├── validators.py               ← validate_file_size, validate_film_year
    └── forms.py                    ← FilmForm с полем poster
```

---

## Вопросы для проверки

**1. Чем `ImageField` отличается от `FileField`?**

<details>
<summary>Ответ</summary>

`ImageField` — это специализированная версия `FileField`, которая дополнительно проверяет, что загруженный файл действительно является корректным изображением (используя библиотеку Pillow). `FileField` принимает любой тип файла без такой проверки — подходит для документов, архивов и прочих файлов, где формат изображения не требуется.

</details>

---

**2. Зачем нужен атрибут `enctype="multipart/form-data"` в HTML-форме?**

<details>
<summary>Ответ</summary>

По умолчанию браузер кодирует данные формы как обычный текст, что не подходит для передачи бинарного содержимого файла. `multipart/form-data` — специальная кодировка, которая позволяет отправить содержимое файла вместе с остальными полями формы. Без этого атрибута браузер передаст только имя файла, но не сами данные, и файл на сервер не попадёт.

</details>

---

**3. Почему в представлении нужно передавать и `request.POST`, и `request.FILES` отдельно?**

<details>
<summary>Ответ</summary>

Django разделяет обычные текстовые данные формы (`request.POST`) и загруженные файлы (`request.FILES`) на уровне обработки запроса — это связано с тем, как HTTP кодирует multipart-запросы. Форма должна получить оба источника данных явно: `FilmForm(request.POST, request.FILES)`. Если передать только `request.POST`, форма не увидит загруженный файл, даже если он был отправлен браузером.

</details>

---

**4. Что произойдёт, если удалить объект модели с заполненным `ImageField`? Удалится ли сам файл с диска?**

<details>
<summary>Ответ</summary>

Нет, файл на диске не удаляется автоматически при удалении объекта модели. Django хранит в базе данных только путь к файлу — удаление записи означает удаление этой ссылки, но не самого файла на файловой системе. Для автоматической очистки неиспользуемых файлов используют сторонние решения, например библиотеку `django-cleanup`.

</details>

---

**5. Почему важно проверять `{% if film.poster %}` перед обращением к `{{ film.poster.url }}` в шаблоне?**

<details>
<summary>Ответ</summary>

Если поле `ImageField` пустое (что допустимо при `blank=True`), попытка обратиться к `.url` выбросит `ValueError`, потому что у отсутствующего файла нет пути для построения URL. Проверка `{% if film.poster %}` гарантирует, что обращение к `.url` происходит только тогда, когда файл действительно загружен.

</details>

---

## Практическая задача

**Тип: расширь проект**

Добавь загрузку фотографий для модели `Actor` и обнови соответствующие шаблоны.

**Требования:**

1. Убедись, что поле `photo` (ImageField, `upload_to='actors/'`, `blank=True`) уже есть в модели `Actor` — если нет, добавь и сделай миграцию
2. Создай `ModelForm` с именем `ActorForm` для модели `Actor` с полями `name` и `photo`
3. Создай представление `add_actor`, которое отображает форму при GET и сохраняет нового актёра при валидном POST-запросе, обязательно учитывая `request.FILES`
4. Создай шаблон с формой, не забыв про `enctype="multipart/form-data"`
5. На странице фильма (`film_detail.html`) выведи фотографии актёров рядом с их именами — если фото нет, не показывай пустой `<img>`

---

[Предыдущий урок](lesson17.md) | [Следующий урок](lesson19.md)