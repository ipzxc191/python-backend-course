# Урок 32. Тестирование с unittest (Django TestCase)

## Зачем тестировать Django-проект

К этому моменту у нас уже приличный по размеру проект: восемь модулей, десятки представлений, формы с кастомной валидацией, права доступа. Любое изменение в одном месте рискует незаметно сломать что-то в другом — например, правка `FilmForm` может случайно сделать поле `year` необязательным, а вы узнаете об этом только когда пользователь пришлёт баг-репорт.

Ручная проверка через браузер после каждой правки не масштабируется. Тесты берут эту рутину на себя: один раз описываем ожидаемое поведение — дальше просто запускаем `python manage.py test` и получаем ответ за секунды.

Важно сразу зафиксировать: тесты **не заменяют** проектирование и код-ревью. Они защищают от регрессий — то есть от случайной поломки того, что раньше работало.

---

## Assert — идея, лежащая в основе любого теста

Прежде чем говорить про Django, разберём вещь гораздо более простую и общую — она встроена прямо в язык Python и никак не связана с тестированием как отдельной технологией.

В Python есть оператор `assert`. Он проверяет условие и, если условие ложно, прерывает выполнение программы с исключением `AssertionError`:

```python
assert 2 + 2 == 4
print('Всё хорошо, до сюда дошли')
```

Программа отработает без единого замечания — выражение `2 + 2 == 4` истинно, `assert` промолчал и пропустил выполнение дальше.

А теперь заведомо ложное условие:

```python
assert 2 + 2 == 5
print('Эта строка никогда не выполнится')
```

```
Traceback (most recent call last):
  File "example.py", line 1, in <module>
    assert 2 + 2 == 5
AssertionError
```

Выполнение останавливается на строке с `assert`, вторая строка кода не выполняется вовсе.

Можно добавить и пояснение к ошибке — вторым аргументом через запятую:

```python
year = 2099
assert year <= 2026, f'Год {year} больше текущего — это фильм из будущего?'
```

```
AssertionError: Год 2099 больше текущего — это фильм из будущего?
```

`assert` можно использовать где угодно в обычном коде — например, чтобы на этапе разработки убедиться, что функция действительно получает то, что ожидается:

```python
def calculate_average_rating(ratings):
    assert len(ratings) > 0, 'Список рейтингов не может быть пустым'
    return sum(ratings) / len(ratings)
```

Именно эта идея — «утверждение, которое обязано быть истинным, иначе что-то пошло не так» — лежит в основе абсолютно любого теста, на любом языке и в любом фреймворке. Тестирование по сути и есть системная организация таких утверждений: вместо разбросанных по коду одиночных `assert` мы собираем их в отдельные, изолированные, автоматически запускаемые проверки.

---

## От ручных assert к тестовым фреймворкам

Если бы мы тестировали проект только голыми `assert`, разбросанными по коду, у нас быстро возникли бы проблемы: непонятно, когда и в каком порядке их запускать, одна упавшая проверка обрывает вообще всё выполнение (а не только показывает, что именно сломалось), и нет удобного отчёта — что из сотен проверок прошло, а что нет.

Поэтому в реальной разработке `assert` не пишут россыпью — их оформляют в **тесткейсы**: отдельные функции или методы, каждый из которых проверяет одну конкретную вещь, запускается независимо от других и попадает в общий отчёт о прогоне.

То, какие именно тесты нужны и что они проверяют, всегда зависит от логики конкретного проекта — для каталога фильмов это одни сценарии («год выпуска не может быть в будущем»), для интернет-магазина — совсем другие («нельзя оформить заказ с отрицательным количеством товара»). Сама механика запуска и организации тестов, впрочем, стандартна и переиспользуется от проекта к проекту.

В Python для этого есть встроенный модуль `unittest` — именно на нём построено тестирование в Django, и именно его мы разберём в этом уроке. В более крупных и сложных проектах разработчики нередко используют сторонние библиотеки поверх стандартного механизма — например, `pytest` и `pytest-django`, которые дают более компактный синтаксис и удобные дополнительные инструменты (fixtures, параметризацию тестов). Для наших задач возможностей встроенного в Django `unittest`-подхода более чем достаточно, поэтому в рамках курса мы останемся на нём.

---

## Тестирование в Django: TestCase и структура тестов

Django-тесты почти всегда наследуют `django.test.TestCase` — это расширенная версия стандартного `unittest.TestCase`, дополненная удобствами специально под Django (тестовая база данных, тестовый HTTP-клиент и так далее, разберём их по ходу урока).

```python
from django.test import TestCase


class ExampleTests(TestCase):
    def test_something(self):
        self.assertEqual(1 + 1, 2)
```

Обратите внимание: вместо голого `assert 1 + 1 == 2` мы написали `self.assertEqual(1 + 1, 2)`. Это не случайность и не просто другой синтаксис — `TestCase` предоставляет целый набор специальных методов-проверок (их называют **assertion-методами**, или просто *ассертами*), которые делают то же самое, что обычный `assert`, но с гораздо более понятными сообщениями об ошибках и с проверками, специфичными для Django (например, «это HTTP-редирект именно на такой-то адрес» — вручную через `assert` такое было бы куда многословнее). К обзору самых частых из них перейдём чуть ниже.

### Откуда Django узнаёт, какие файлы — это тесты

По умолчанию `startapp` создаёт в каждом приложении файл `tests.py` — это то самое место, куда мы будем писать тесты для приложения `films`:

```
films/
├── models.py
├── views.py
├── forms.py
└── tests.py
```

Команда `python manage.py test` ищет во всех приложениях, перечисленных в `INSTALLED_APPS`, файлы и классы, оформленные как тесты, и запускает их автоматически — вручную указывать, что и где искать, не нужно.

Если проект вырастет и `tests.py` разрастётся на сотни строк, его можно превратить в пакет из нескольких файлов — этот вариант мы разберём отдельно в конце урока, как более «взрослый» подход к организации тестов. Для наших задач сейчас достаточно одного файла `tests.py`.

### Тестовая база данных

При запуске `python manage.py test` Django делает следующее:

1. создаёт **отдельную тестовую базу данных** (обычно с префиксом `test_`) и прогоняет по ней все миграции;
2. каждый тестовый метод выполняется внутри транзакции, которая откатывается сразу после теста — поэтому тесты не мешают друг другу и не засоряют реальную БД;
3. по завершении всех тестов тестовая база удаляется.

Это значит, что переключение проекта на PostgreSQL никак не повлияет на логику тестов — Django создаст тестовую Postgres-базу точно так же, как создавал тестовую SQLite.

---

## setUp() и setUpTestData() — подготовка данных для тестов

Почти каждому тесту нужны какие-то исходные объекты — например, фильм и режиссёр, которых ещё нет в пустой тестовой базе. Создавать их заново внутри каждого отдельного тестового метода — обречь себя на дублирование кода. Для этого у `TestCase` есть два специальных метода, и важно понимать разницу между ними.

### setUp() — выполняется перед каждым тестом

```python
class FilmModelTests(TestCase):
    def setUp(self):
        self.director = Director.objects.create(name='Фрэнсис Форд Коппола')
        self.film = Film.objects.create(
            title='Крёстный отец',
            year=1972,
            director=self.director,
            slug='krestnyy-otec',
        )
```

`setUp()` вызывается заново **перед каждым** тестовым методом класса. Если в классе пять тестов — `setUp()` выполнится пять раз, и каждый тест получит свежий, независимый набор объектов. Даже если один тест что-то изменит в `self.film` (например, поменяет `title`), следующий тест всё равно начнёт с чистого, заново созданного объекта.

### setUpTestData() — выполняется один раз на весь класс

```python
class FilmModelTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.director = Director.objects.create(name='Фрэнсис Форд Коппола')
        cls.film = Film.objects.create(
            title='Крёстный отец',
            year=1972,
            director=cls.director,
            slug='krestnyy-otec',
        )
```

`setUpTestData()` — это `classmethod`, который выполняется **один раз для всего класса**, перед первым тестом, а не перед каждым. Данные создаются один раз внутри транзакции на уровне класса, а затем каждый отдельный тест получает свою обёртку-транзакцию вокруг них, которая откатывается после теста — то есть один тест по-прежнему не может испортить данные для другого, но сами объекты не пересоздаются в базе заново перед каждым методом.

**Почему это важно:** если в классе не 2 теста, а 50, и `setUp()` каждый раз заново обращается к базе данных для создания одних и тех же объектов — это 50 лишних обращений к БД. `setUpTestData()` делает это один раз, что заметно быстрее на больших тестовых классах.

**Когда что выбирать:**

| | `setUp()` | `setUpTestData()` |
|---|---|---|
| Когда выполняется | Перед каждым тестовым методом | Один раз перед всеми тестами класса |
| Скорость на большом числе тестов | Медленнее | Быстрее |
| Когда использовать | Если тест изменяет объект и последующим тестам это не должно быть видно (перестраховка) или объекты в разных тестах должны отличаться | Если данные в `setUp` одинаковые для всех тестов класса и не изменяются деструктивно |

На практике для простых учебных примеров разница в скорости незаметна, и `setUp()` — вполне рабочий выбор по умолчанию. Но в реальных проектах с сотнями тестов `setUpTestData()` — стандартная практика, и полезно с самого начала понимать, что оба инструмента существуют и решают немного разные задачи.

---

## Обзор основных assert-методов

В `django.test.TestCase` доступны десятки готовых assertion-методов (он наследует их частично от `unittest.TestCase`, частично добавляет свои, специфичные для Django). Знать все наизусть не нужно — на практике в 90% тестов хватает 5–6 самых частых. Вот они, на абстрактных примерах, без привязки к нашему проекту — просто как справочный список, который дальше будем применять на реальном коде:

```python
# Проверка на равенство
self.assertEqual(2 + 2, 4)

# Проверка булева значения
self.assertTrue(5 > 3)
self.assertFalse(5 < 3)

# Проверка вхождения элемента в контейнер
self.assertIn('a', ['a', 'b', 'c'])

# Проверка, что код выбрасывает исключение
with self.assertRaises(ZeroDivisionError):
    1 / 0

# Специфичные для Django — работают с HTTP-ответами (response)
self.assertContains(response, 'Ожидаемый текст')       # текст есть в теле ответа
self.assertRedirects(response, '/some/url/')            # это редирект именно на такой адрес
self.assertTemplateUsed(response, 'app/template.html')  # использован такой шаблон
```

Полный список методов можно посмотреть в документации Django (`django.test.TestCase`) и Python (`unittest.TestCase`) — но для повседневной работы с проектом хватает именно этого набора плюс `assertFormError` для форм, который встретится чуть ниже. По ходу урока мы будем использовать эти методы уже на реальных тестах, а в конце соберём их в итоговую таблицу-рекап.

---

## Тесты моделей

### Film — базовая проверка, get_absolute_url() и валидатор года

```python
# films/tests.py
from datetime import date

from django.test import TestCase
from django.core.exceptions import ValidationError
from django.contrib.auth.models import User

from films.models import Film, Director, Genre, UserProfile, FilmStats, Review


class FilmModelTests(TestCase):
    def setUp(self):
        self.director = Director.objects.create(name='Christopher Nolan')
        self.film = Film.objects.create(
            title='Interstellar',
            year=2014,
            director=self.director,
            slug='interstellar',
        )

    def test_str_returns_title_and_year(self):
        self.assertEqual(str(self.film), 'Interstellar (2014)')

    def test_get_absolute_url(self):
        self.assertEqual(self.film.get_absolute_url(), '/films/interstellar/')
```

### Валидатор года выпуска — важная тонкость про save() и full_clean()

В `films/validators.py` у поля `Film.year` есть валидатор `validate_film_year`, который не пропускает года из будущего и года раньше 1888-го. Казалось бы, логично ожидать, что `Film.objects.create(year=3000)` тут же упадёт с ошибкой. **Это не так** — и это важно понимать, потому что рано или поздно это удивит любого, кто впервые сталкивается с валидаторами Django:

```python
class FilmYearValidatorTests(TestCase):
    def setUp(self):
        self.director = Director.objects.create(name='Test Director')

    def test_save_does_not_trigger_field_validators(self):
        # Валидаторы поля НЕ вызываются автоматически при .save() —
        # объект сохранится в базу, несмотря на некорректный год
        film = Film.objects.create(title='Film From Future', year=3000, director=self.director)
        self.assertEqual(film.year, 3000)

    def test_full_clean_raises_for_future_year(self):
        film = Film(title='Film From Future', year=date.today().year + 5, director=self.director)
        with self.assertRaises(ValidationError):
            film.full_clean()

    def test_full_clean_raises_for_year_before_cinema_existed(self):
        film = Film(title='Ancient Film', year=1800, director=self.director)
        with self.assertRaises(ValidationError):
            film.full_clean()
```

`save()` в Django **не вызывает** валидаторы полей самостоятельно — это осознанное архитектурное решение фреймворка (иначе `save()` в некоторых сценариях мог бы неожиданно упасть там, где раньше работал). Валидаторы срабатывают только при явном вызове `full_clean()` — либо вручную, как в тестах выше, либо автоматически внутри `ModelForm.is_valid()`, который вызывает `full_clean()` для вас за кулисами. Именно поэтому валидатор реально «защищает» проект только там, где данные проходят через форму (`FilmForm` — увидим это в разделе про тесты форм), а прямое создание объекта через `Film.objects.create(...)` в коде (например, в management-командах или сидинге) этот валидатор проигнорирует.

### Director — автогенерация slug

```python
class DirectorModelTests(TestCase):
    def test_slug_is_auto_generated_when_not_provided(self):
        director = Director.objects.create(name='Denis Villeneuve')
        self.assertEqual(director.slug, 'denis-villeneuve')

    def test_get_absolute_url(self):
        director = Director.objects.create(name='Bong Joon-ho', slug='bong-joon-ho')
        self.assertEqual(director.get_absolute_url(), '/directors/bong-joon-ho/')
```

> Имена режиссёров в тестах намеренно латиницей — `slugify()` без `allow_unicode=True` транслитерирует кириллицу, но точный результат транслитерации зависит от версии библиотеки `python-slugify`. Для латинских имён результат детерминирован и не «поплывёт» при обновлении зависимости.

### UserProfile — OneToOne и related_name

```python
class UserProfileModelTests(TestCase):
    def test_profile_accessible_via_related_name(self):
        user = User.objects.create_user(username='ivan', password='testpass123')
        profile = UserProfile.objects.create(user=user, bio='Люблю нуар')

        self.assertEqual(user.profile, profile)
        self.assertEqual(user.profile.bio, 'Люблю нуар')

    def test_missing_profile_raises_related_object_does_not_exist(self):
        user = User.objects.create_user(username='no_profile_user', password='testpass123')

        with self.assertRaises(UserProfile.DoesNotExist):
            user.profile
```

### FilmManager — кастомные методы менеджера

```python
class FilmManagerTests(TestCase):
    def setUp(self):
        self.director = Director.objects.create(name='Tim Burton')
        self.film_a = Film.objects.create(title='Film A', year=2020, director=self.director, slug='film-a', rating=9.0)
        self.film_b = Film.objects.create(title='Film B', year=2021, director=self.director, slug='film-b', rating=5.0)

    def test_high_rated_returns_only_films_with_rating_at_least_8(self):
        titles = list(Film.objects.high_rated().values_list('title', flat=True))
        self.assertEqual(titles, ['Film A'])

    def test_by_year_filters_correctly(self):
        results = Film.objects.by_year(2020)
        self.assertEqual(results.count(), 1)
        self.assertEqual(results.first(), self.film_a)

    def test_search_finds_film_by_partial_title_case_insensitive(self):
        results = Film.objects.search('film a')
        self.assertEqual(results.count(), 1)
        self.assertEqual(results.first(), self.film_a)

    def test_recent_orders_by_creation_date_and_limits_count(self):
        recent_films = list(Film.objects.recent(1))
        self.assertEqual(recent_films, [self.film_b])  # film_b создан последним в setUp
```

`test_recent_orders_by_creation_date_and_limits_count` опирается на порядок `created_at` (`auto_now_add=True`), а не на `year` — `film_b` создан вторым в `setUp()`, значит именно он окажется единственным элементом при `recent(1)`.

---

## Тесты форм

### FilmForm — реальные валидаторы: цифры в названии, конфликт жанров, год

```python
# films/tests.py
from films.forms import FilmForm, ReviewForm


class FilmFormTests(TestCase):
    def setUp(self):
        self.director = Director.objects.create(name='Denis Villeneuve')
        self.comedy = Genre.objects.create(name='Комедия', slug='komediya')
        self.tragedy = Genre.objects.create(name='Трагедия', slug='tragediya')

    def test_valid_data_passes_validation(self):
        form = FilmForm(data={
            'title': 'Dune',
            'year': 2021,
            'director': self.director.id,
        })
        self.assertTrue(form.is_valid())

    def test_title_consisting_only_of_digits_is_invalid(self):
        form = FilmForm(data={
            'title': '12345',
            'year': 2021,
            'director': self.director.id,
        })
        self.assertFalse(form.is_valid())
        self.assertIn('title', form.errors)

    def test_conflicting_genres_are_invalid(self):
        form = FilmForm(data={
            'title': 'Странная история',
            'year': 2021,
            'director': self.director.id,
            'genres': [self.comedy.id, self.tragedy.id],
        })
        self.assertFalse(form.is_valid())
        self.assertIn('genres', form.errors)

    def test_future_year_is_invalid(self):
        # ModelForm вызывает full_clean() под капотом — валидатор из модели сработает и здесь
        future_year = date.today().year + 5
        form = FilmForm(data={
            'title': 'Film From Future',
            'year': future_year,
            'director': self.director.id,
        })
        self.assertFalse(form.is_valid())
        self.assertIn('year', form.errors)
```

`test_future_year_is_invalid` — прямое продолжение того, что мы разобрали в тестах модели: `ModelForm.is_valid()` вызывает `full_clean()` инстанса модели «под капотом», поэтому валидатор `validate_film_year`, который бесполезен при прямом `Film.objects.create()`, здесь срабатывает исправно.

### ReviewForm — три поля с валидацией плюс non-field ошибка

```python
class ReviewFormTests(TestCase):
    def test_valid_data_passes_validation(self):
        form = ReviewForm(data={
            'author_name': 'Иван Петров',
            'text': 'Очень достойная работа, всем советую посмотреть этот фильм',
            'rating': 8,
        })
        self.assertTrue(form.is_valid())

    def test_short_text_is_invalid(self):
        form = ReviewForm(data={
            'author_name': 'Иван Петров',
            'text': 'Норм фильм',
            'rating': 8,
        })
        self.assertFalse(form.is_valid())
        self.assertIn('text', form.errors)

    def test_reserved_author_name_is_invalid(self):
        form = ReviewForm(data={
            'author_name': 'Анонимный критик',
            'text': 'Очень достойная работа, всем советую посмотреть этот фильм',
            'rating': 8,
        })
        self.assertFalse(form.is_valid())
        self.assertIn('author_name', form.errors)

    def test_low_rating_without_detailed_text_is_invalid(self):
        # 6 слов (проходит clean_text), но короче 50 символов (не проходит clean())
        form = ReviewForm(data={
            'author_name': 'Иван Петров',
            'text': 'Ужасный, скучный и очень плохой фильм',
            'rating': 2,
        })
        self.assertFalse(form.is_valid())
        # clean() выбрасывает ValidationError напрямую — это non-field error,
        # поэтому поле нужно указать как None
        self.assertFormError(form, None, 'Для низкой оценки (3 и ниже) распишите причину подробнее — минимум 50 символов.')
```

Обратите внимание на разницу между `test_short_text_is_invalid`/`test_reserved_author_name_is_invalid` и `test_low_rating_without_detailed_text_is_invalid`: первые два метода (`clean_text()`, `clean_author_name()`) — это field-level валидаторы, их ошибки привязаны к конкретному полю. А общий `clean()` в этой форме выбрасывает `ValidationError` без указания поля — такая ошибка идёт в `form.non_field_errors()`, и в `assertFormError` вместо имени поля нужно передать `None`.

### CustomRegistrationForm — уникальность email

```python
from films.forms import CustomRegistrationForm


class CustomRegistrationFormTests(TestCase):
    def setUp(self):
        User.objects.create_user(username='existing', email='taken@example.com', password='pass12345')

    def test_duplicate_email_is_invalid(self):
        form = CustomRegistrationForm(data={
            'username': 'newuser',
            'email': 'taken@example.com',
            'password1': 'StrongPass123',
            'password2': 'StrongPass123',
        })
        self.assertFalse(form.is_valid())
        self.assertIn('email', form.errors)

    def test_valid_registration_hashes_password(self):
        form = CustomRegistrationForm(data={
            'username': 'newuser',
            'email': 'new@example.com',
            'password1': 'StrongPass123',
            'password2': 'StrongPass123',
        })
        self.assertTrue(form.is_valid())
        user = form.save()
        self.assertNotEqual(user.password, 'StrongPass123')
        self.assertTrue(user.check_password('StrongPass123'))
```

`first_name`/`last_name` в тестовых данных не передаются намеренно — оба поля добавлены в форму как `required=False`, так что форма остаётся валидной и без них.

---

## Тесты вьюшек через self.client

### FilmListView — статус, шаблон и пагинация (paginate_by = 10)

```python
from django.urls import reverse


class FilmListViewTests(TestCase):
    def setUp(self):
        director = Director.objects.create(name='Christopher Nolan')
        for i in range(15):
            Film.objects.create(title=f'Film {i}', year=2020, director=director, slug=f'film-{i}')

    def test_list_view_status_code(self):
        response = self.client.get(reverse('films:film_list'))
        self.assertEqual(response.status_code, 200)

    def test_list_view_uses_correct_template(self):
        response = self.client.get(reverse('films:film_list'))
        self.assertTemplateUsed(response, 'films/film_list.html')

    def test_pagination_is_ten_per_page(self):
        response = self.client.get(reverse('films:film_list'))
        self.assertTrue(response.context['is_paginated'])
        self.assertEqual(len(response.context['films']), 10)
        self.assertTrue(response.context['page_obj'].has_next())
```

### FilmDetailView — 200, 404 и побочный эффект (инкремент просмотров)

```python
class FilmDetailViewTests(TestCase):
    def setUp(self):
        director = Director.objects.create(name='Denis Villeneuve')
        self.film = Film.objects.create(title='Arrival', year=2016, director=director, slug='arrival')
        self.stats = FilmStats.objects.create(film=self.film, views_count=0)

    def test_detail_view_status_code(self):
        url = reverse('films:film_detail', kwargs={'slug': self.film.slug})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, 'Arrival')

    def test_detail_view_404_for_unknown_slug(self):
        url = reverse('films:film_detail', kwargs={'slug': 'no-such-film'})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 404)

    def test_views_count_increments_on_visit(self):
        url = reverse('films:film_detail', kwargs={'slug': self.film.slug})
        self.client.get(url)
        self.stats.refresh_from_db()
        self.assertEqual(self.stats.views_count, 1)
```

### FilmCreateView — три сценария авторизации/прав одним классом

```python
from django.contrib.auth.models import Permission


class FilmCreateViewTests(TestCase):
    def setUp(self):
        self.director = Director.objects.create(name='Tim Burton')
        self.url = reverse('films:add_film')
        self.post_data = {
            'title': 'Edward Scissorhands',
            'year': 1990,
            'director': self.director.id,
        }
        self.user = User.objects.create_user(username='ivan', password='testpass123')

    def test_anonymous_user_redirected_to_login(self):
        response = self.client.get(self.url)
        self.assertRedirects(response, f"{reverse('login')}?next={self.url}")

    def test_authenticated_without_permission_gets_403(self):
        self.client.force_login(self.user)
        response = self.client.get(self.url)
        self.assertEqual(response.status_code, 403)

    def test_authenticated_with_permission_can_create_film(self):
        permission = Permission.objects.get(codename='add_film')
        self.user.user_permissions.add(permission)
        self.client.force_login(self.user)

        response = self.client.post(self.url, data=self.post_data)

        self.assertEqual(Film.objects.count(), 1)
        created_film = Film.objects.get(title='Edward Scissorhands')
        self.assertRedirects(response, created_film.get_absolute_url())
```

`raise_exception = True` у `FilmCreateView` гарантирует именно `403`, а не повторный редирект на страницу входа для уже авторизованного, но недостаточно привилегированного пользователя — второй тест проверяет ровно это поведение.

### AddReviewView — теперь требует авторизации

Раньше эта вьюшка была доступна анонимно, но в финальной версии проекта у неё появился `LoginRequiredMixin`. Структура теста меняется — теперь она гораздо больше похожа на `FilmCreateViewTests`:

```python
from films.models import Review


class AddReviewViewTests(TestCase):
    def setUp(self):
        director = Director.objects.create(name='Bong Joon-ho')
        self.film = Film.objects.create(title='Parasite', year=2019, director=director, slug='parasite')
        self.url = reverse('films:add_review', kwargs={'slug': self.film.slug})
        self.user = User.objects.create_user(username='reviewer', password='testpass123')

    def test_anonymous_user_redirected_to_login(self):
        response = self.client.get(self.url)
        self.assertRedirects(response, f"{reverse('login')}?next={self.url}")

    def test_valid_post_creates_review_and_redirects(self):
        self.client.force_login(self.user)
        response = self.client.post(self.url, data={
            'author_name': 'Иван Петров',
            'text': 'Очень достойная работа, всем советую посмотреть этот фильм',
            'rating': 9,
        })

        self.assertEqual(Review.objects.count(), 1)
        review = Review.objects.first()
        self.assertEqual(review.film, self.film)
        self.assertRedirects(response, self.film.get_absolute_url())

    def test_invalid_post_rerenders_form_with_errors(self):
        self.client.force_login(self.user)
        response = self.client.post(self.url, data={
            'author_name': 'Иван Петров',
            'text': 'Коротко',
            'rating': 5,
        })

        self.assertEqual(response.status_code, 200)
        self.assertEqual(Review.objects.count(), 0)
        self.assertFormError(
            response.context['form'], 'text',
            'Рецензия слишком короткая — напишите хотя бы 5 слов.'
        )
```

---

## Тесты разрешений и групп

### Группа «Модераторы» — стандартные разрешения change_film/delete_film

```python
from django.contrib.auth.models import Group


class ModeratorGroupPermissionTests(TestCase):
    def setUp(self):
        director = Director.objects.create(name='Martin Scorsese')
        self.film = Film.objects.create(title='Taxi Driver', year=1976, director=director, slug='taxi-driver')

        moderators = Group.objects.create(name='Модераторы')
        change_perm = Permission.objects.get(codename='change_film')
        delete_perm = Permission.objects.get(codename='delete_film')
        moderators.permissions.set([change_perm, delete_perm])

        self.moderator = User.objects.create_user(username='moder', password='testpass123')
        self.moderator.groups.add(moderators)

        self.regular_user = User.objects.create_user(username='regular', password='testpass123')

    def test_moderator_can_access_edit_page(self):
        self.client.force_login(self.moderator)
        url = reverse('films:film_edit', kwargs={'slug': self.film.slug})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 200)

    def test_regular_user_gets_403_on_edit_page(self):
        self.client.force_login(self.regular_user)
        url = reverse('films:film_edit', kwargs={'slug': self.film.slug})
        response = self.client.get(url)
        self.assertEqual(response.status_code, 403)
```

### Кастомные разрешения проекта — существование и назначаемость

В проекте сейчас три кастомных разрешения, не привязанных ни к одной реальной вьюшке: `films.can_feature_film` (на модели `Film`) и `films.publish_review`, `films.moderate_review` (на модели `Review`, последнее вообще нигде не используется в коде). Раз ни одна вьюшка их не проверяет, тестировать через `self.client` нечего — но полезно убедиться, что сами разрешения существуют и корректно назначаются, чтобы админка и будущий код могли на них полагаться:

```python
class CustomPermissionsExistTests(TestCase):
    def test_can_feature_film_permission_exists_and_is_assignable(self):
        permission = Permission.objects.get(codename='can_feature_film', content_type__app_label='films')
        user = User.objects.create_user(username='editor', password='testpass123')

        self.assertFalse(user.has_perm('films.can_feature_film'))
        user.user_permissions.add(permission)
        user = User.objects.get(pk=user.pk)  # сбрасываем кэш разрешений
        self.assertTrue(user.has_perm('films.can_feature_film'))

    def test_review_custom_permissions_exist(self):
        self.assertTrue(
            Permission.objects.filter(codename='publish_review', content_type__app_label='films').exists()
        )
        self.assertTrue(
            Permission.objects.filter(codename='moderate_review', content_type__app_label='films').exists()
        )
```

---

## Разбор используемых assert'ов

| Метод | Что проверяет | Где выше использован |
|---|---|---|
| `assertEqual(a, b)` | `a == b` | статус-коды, количество объектов, значения полей |
| `assertTrue(x)` / `assertFalse(x)` | булево значение | `form.is_valid()`, `is_paginated`, `has_perm()` |
| `assertIn(x, container)` | вхождение элемента | `'title' in form.errors` |
| `assertRaises(Exception)` | что код внутри `with` выбрасывает исключение | `UserProfile.DoesNotExist`, `ValidationError` из `full_clean()` |
| `assertRedirects(response, url)` | ответ — редирект именно на этот URL | login-редирект, редирект после создания фильма/рецензии |
| `assertContains(response, text)` | текст присутствует в теле ответа | заголовок фильма на странице |
| `assertTemplateUsed(response, name)` | шаблон был использован при рендере | `film_list.html` |
| `assertFormError(form, field, error)` | у поля (или у формы в целом, если `field=None`) есть конкретная ошибка | `text`/`author_name` в `ReviewForm`, non-field error в `ReviewForm.clean()` |

---

## Подводные камни

### client.login() против force_login()

```python
logged_in = self.client.login(username='ivan', password='testpass123')  # может тихо вернуть False
self.client.force_login(self.user)  # напрямую устанавливает сессию, без проверки пароля
```

### reverse() без учёта namespace

```python
reverse('film_list')          # NoReverseMatch — маршрут в namespace 'films'
reverse('films:film_list')    # правильно
```

### Забытый refresh_from_db() после операций через .update()

Как в `FilmDetailViewTests` — `update()` меняет данные на уровне SQL, минуя уже загруженный в Python объект. Нужно перечитать его явно.

### Валидаторы модели срабатывают не при save(), а при full_clean()

Мы разобрали это подробно в тестах модели `Film`: `Model.save()` **не** вызывает валидаторы полей самостоятельно. Они реально проверяются только при `full_clean()` — вручную или неявно внутри `ModelForm.is_valid()`. Если код где-то в проекте создаёт объекты напрямую через `Model.objects.create(...)` (например, в management-командах вроде `seed_films.py` или в скриптах миграции данных), валидаторы там не сработают — это стоит держать в голове при написании такого кода, а не только тестов.

### is_published — присвоение атрибута, которого нет в модели, не вызывает ошибку

`AddReviewView.form_valid()` содержит строку `review.is_published = True` при наличии разрешения `films.publish_review`. Поскольку `Review` не имеет поля `is_published`, Python просто создаёт временный атрибут объекта в памяти — это не ошибка (Python не проверяет соответствие атрибутов объявленным полям модели), но и не имеет никакого эффекта: при `review.save()` в базу пишутся только реальные поля модели, а `is_published` теряется бесследно. Если тестировать такой код в лоб («после сохранения `review.is_published` равно `True`»述), тест пройдёт — но проверит только состояние Python-объекта в памяти теста, а не то, что реально сохранилось в базе. Хороший общий урок: если тест не перечитывает объект из базы (`refresh_from_db()` или новый `.objects.get()`), он может незаметно проверять не то, что нужно.

### permission denied to create database

```
Got an error creating the test database: permission denied to create database
```

Проект работает только с PostgreSQL — для тестов Django создаёт отдельную тестовую базу, и роли из `DATABASES['default']['USER']` для этого нужно право `CREATEDB`:

```sql
ALTER ROLE ваш_пользователь_бд CREATEDB;
```

### Genre.slug нужно указывать вручную

В отличие от `Film` и `Director`, модель `Genre` не переопределяет `save()` и не генерирует slug автоматически. При создании `Genre` в тестах (и в реальном коде — например, в админке или сидинге) slug нужно передавать явно, иначе при попытке создать вторую запись без slug вы получите `IntegrityError` из-за нарушения `unique=True` на пустой строке.

### Медиафайлы в тестах (ImageField)

```python
import tempfile
from django.test import override_settings

@override_settings(MEDIA_ROOT=tempfile.mkdtemp())
class DirectorFormMediaTests(TestCase):
    ...
```

---

## Бонус: тесты «по-взрослому» — пакет tests/ вместо tests.py

```
films/
├── models.py
├── views.py
├── forms.py
└── tests/
    ├── __init__.py
    ├── test_models.py
    ├── test_forms.py
    └── test_views.py
```

Единственное условие — обязательный `__init__.py` (даже пустой). Запуск отдельного файла или класса:

```bash
python manage.py test films.tests.test_views
python manage.py test films.tests.test_views.FilmListViewTests
```

Для нашего проекта, где сейчас уже под два десятка тестовых классов, это, кстати, разумный момент, чтобы перейти на пакет — но техническое решение оставляем на ваше усмотрение.

---

## Вопросы для проверки

1. Чем ассерт-методы вроде `self.assertEqual()` отличаются от обычного оператора `assert`?
2. Почему `Film.objects.create(year=3000)` не вызывает ошибку валидации, хотя у поля `year` есть валидатор, отклоняющий будущие года?
3. В чём разница между `self.assertIn('text', form.errors)` и `self.assertFormError(form, None, 'сообщение')`?
4. Почему тест на `is_published` в `AddReviewView` не был написан, хотя соответствующая ветка кода существует?
5. Почему `AddReviewViewTests` в этом уроке использует `force_login()`, хотя в прошлой версии проекта эта вьюшка была доступна анонимно?
6. Что нужно указать явно при создании объекта `Genre` в тестах и почему?

---

## Практическая задача

**Тип: расширь проект**

По образцу тестов из этого урока напиши:

1. **`DirectorCreateViewTests`** — три сценария по аналогии с `FilmCreateViewTests`: аноним → редирект на login, авторизован без права `films.add_director` → 403, авторизован с правом → успешное создание и редирект через `get_absolute_url()`.

2. **`DirectorUpdateViewTests`** — три аналогичных сценария для `films:director_edit`, плюс отдельная проверка, что после успешного обновления `get_success_url()` возвращает актуальный `get_absolute_url()` изменённого объекта.

3. **`DirectorDeleteViewTests`** — проверь, что `POST` удаляет объект и делает редирект на `films:film_list`, а `GET` только показывает страницу подтверждения и не удаляет ничего.

4. **Бонус.** Напиши тест, который явно проверяет описанный в уроке подводный камень: создание двух объектов `Genre` без указания `slug` должно приводить к `IntegrityError` при попытке сохранить второй объект (оберни это в `django.db.transaction.atomic()`, как того требует тестирование ошибок целостности внутри `TestCase`).

---

[Предыдущий урок](lesson31.md) | [Следующий урок](lesson33.md)