# Стилизация проекта перед деплоем

Пошаговый референс: дизайн-система, главная страница, видео-поле, доработка рецензий и оставшиеся страницы.

---

## 1. Дизайн-токены: единая палитра через CSS-переменные в base.css

```css
:root {
    /* Тёмные поверхности (шапка, полоса фильтров, футер) */
    --color-header-bg: #242424;
    --color-strip-bg: #333333;
    --color-text-on-dark: #F2F2F2;
    --color-text-on-dark-secondary: #B5B5B5;

    /* Светлое тело страницы */
    --color-page-bg: #F3F3F3;
    --color-surface: #FFFFFF;
    --color-text: #1A1A1A;
    --color-text-secondary: #6E6E6E;
    --color-border: #E2E2E2;

    /* Акценты */
    --color-primary: #E0A800;
    --color-primary-hover: #C99400;
    --color-accent: #E0A800;
    --color-peach: #F3E4C8;
    --color-black: #1A1A1A;

    /* Форма и тени */
    --radius: 6px;
    --radius-sm: 4px;
    --shadow-card: 0 2px 8px rgba(0, 0, 0, 0.08);
    --shadow-card-hover: 0 6px 18px rgba(0, 0, 0, 0.12);
}
```

## 2. base.html — тёмная шапка с поиском, слот под полосу фильтров

```html
{% load static %}
{% load film_tags %}
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Сайт фильмов{% endblock %}</title>
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@400;600;700&display=swap" rel="stylesheet">
    <link rel="stylesheet" href="{% static 'css/base.css' %}">
    {% block extra_css %}{% endblock %}
</head>
<body>
    <header class="site-header">
        <div class="site-header__inner">
            <a href="{% url 'films:index' %}" class="site-logo-box">🎬</a>
            <a href="{% url 'films:index' %}" class="site-logo">Film Collection</a>

            <nav class="site-nav">
                <a href="{% url 'films:index' %}">Главная</a>
                <a href="{% url 'films:film_list' %}">Каталог</a>
                <a href="{% url 'films:about' %}">О сайте</a>
            </nav>

            <form method="GET" action="{% url 'films:search_film' %}" class="site-search">
                <input type="text" name="q" placeholder="Поиск..." value="{{ request.GET.q }}">
                <button type="submit" aria-label="Искать">🔍</button>
            </form>

            <div class="site-auth">
                {% if user.is_authenticated %}
                    {% if perms.films.add_film %}
                        <a href="{% url 'films:add_film' %}">+ Фильм</a>
                    {% endif %}
                    <a href="{% url 'films:profile' %}">{{ user.username }}</a>
                    <form method="POST" action="{% url 'logout' %}" class="site-auth__logout">
                        {% csrf_token %}
                        <button type="submit" class="link-button">Выйти</button>
                    </form>
                {% else %}
                    <a href="{% url 'login' %}">Войти</a>
                {% endif %}
            </div>
        </div>
    </header>

    {% block filter_strip %}{% endblock %}

    <main class="site-main">
        {% block content %}{% endblock %}
    </main>

    <footer class="site-footer">
        <div class="site-footer__inner">
            <div class="site-footer__brand">
                <span class="site-logo-box">🎬</span>
                <span class="site-footer__brand-name">Film Collection</span>
            </div>
            <div class="site-footer__contacts">
                <p>Контакты для связи:</p>
                <p>Email: info@filmsite.local</p>
                <p>© {% current_year %} — учебный проект</p>
            </div>
        </div>
    </footer>

    {% block extra_js %}{% endblock %}
</body>
</html>
```

## 3. base.css — ядро дизайн-системы

Общие для всего сайта стили: шапка, кнопки, формы, сетка карточек, пагинация, футер, адаптив. Стили для конкретных страниц (детальные страницы, видео, рецензии) добавляются позже, отдельными блоками, по мере того как эти страницы встречаются ниже.

```css
* {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
}

body {
    font-family: 'Poppins', -apple-system, 'Segoe UI', sans-serif;
    background: var(--color-page-bg);
    color: var(--color-text);
    line-height: 1.5;
}

a {
    color: var(--color-primary);
    text-decoration: none;
}

a:hover {
    color: var(--color-primary-hover);
}

/* ===== Шапка ===== */

.site-header {
    background: var(--color-header-bg);
    position: sticky;
    top: 0;
    z-index: 10;
}

.site-header__inner {
    max-width: 1200px;
    margin: 0 auto;
    padding: 14px 24px;
    display: flex;
    align-items: center;
    gap: 20px;
    flex-wrap: wrap;
}

.site-logo-box {
    width: 34px;
    height: 34px;
    background: #1A1A1A;
    border: 1px solid #444;
    border-radius: var(--radius-sm);
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 15px;
}

.site-logo {
    color: var(--color-text-on-dark);
    font-weight: 700;
    letter-spacing: 0.5px;
    text-transform: uppercase;
    font-size: 1rem;
}

.site-nav {
    display: flex;
    gap: 20px;
    margin-left: 8px;
}

.site-nav a {
    color: var(--color-text-on-dark-secondary);
    font-weight: 600;
    font-size: 0.85rem;
    text-transform: uppercase;
    letter-spacing: 0.4px;
}

.site-nav a:hover {
    color: var(--color-accent);
}

.site-search {
    display: flex;
    margin-left: auto;
    flex: 1 1 200px;
    max-width: 260px;
}

.site-search input {
    flex: 1;
    padding: 7px 12px;
    border: 1px solid #444;
    border-radius: var(--radius-sm) 0 0 var(--radius-sm);
    background: #1A1A1A;
    color: #fff;
    font-family: inherit;
}

.site-search input::placeholder {
    color: #888;
}

.site-search button {
    border: 1px solid var(--color-accent);
    background: var(--color-accent);
    color: #1A1A1A;
    border-radius: 0 var(--radius-sm) var(--radius-sm) 0;
    padding: 0 14px;
    cursor: pointer;
}

.site-auth {
    display: flex;
    align-items: center;
    gap: 14px;
}

.site-auth a, .site-auth .link-button {
    color: var(--color-text-on-dark-secondary);
    font-size: 0.85rem;
    font-weight: 600;
}

.site-auth a:hover, .site-auth .link-button:hover {
    color: var(--color-accent);
}

.link-button {
    background: none;
    border: none;
    cursor: pointer;
    font: inherit;
}

/* ===== Полоса фильтрации по жанрам (только на главной) ===== */

.genre-strip {
    background: var(--color-strip-bg);
    padding: 18px 24px 22px;
}

.genre-strip__inner {
    max-width: 1200px;
    margin: 0 auto;
}

.genre-strip__title {
    color: #fff;
    font-weight: 700;
    text-transform: uppercase;
    font-size: 1rem;
    letter-spacing: 0.4px;
    margin-bottom: 10px;
}

.genre-tabs {
    display: flex;
    gap: 22px;
    flex-wrap: wrap;
}

.genre-tabs a {
    color: #c9c9c9;
    font-weight: 600;
    font-size: 0.88rem;
    text-transform: uppercase;
    letter-spacing: 0.3px;
}

.genre-tabs a:hover,
.genre-tabs a.active {
    color: var(--color-accent);
}

/* ===== Дополнительные фильтры: год / рейтинг ===== */

.quick-filters {
    max-width: 1200px;
    margin: 20px auto 0;
    padding: 0 24px;
}

.quick-filters__form {
    display: flex;
    align-items: flex-end;
    gap: 14px;
    flex-wrap: wrap;
    background: var(--color-surface);
    border: 1px solid var(--color-border);
    border-radius: var(--radius);
    padding: 14px 18px;
}

.quick-filters__field label {
    display: block;
    font-size: 0.75rem;
    text-transform: uppercase;
    letter-spacing: 0.3px;
    color: var(--color-text-secondary);
    margin-bottom: 4px;
}

.quick-filters__field input {
    padding: 7px 10px;
    border: 1px solid var(--color-border);
    border-radius: var(--radius-sm);
    width: 130px;
    font-family: inherit;
}

/* ===== Кнопки ===== */

.btn, main form button[type="submit"] {
    display: inline-block;
    padding: 9px 20px;
    border-radius: var(--radius-sm);
    border: 1px solid var(--color-primary);
    background: var(--color-primary);
    color: #1A1A1A;
    font-family: inherit;
    font-size: 0.9rem;
    font-weight: 600;
    cursor: pointer;
    transition: background 0.15s, transform 0.1s;
}

.btn:hover, main form button[type="submit"]:hover {
    background: var(--color-primary-hover);
    border-color: var(--color-primary-hover);
    color: #1A1A1A;
}

.btn:active, main form button[type="submit"]:active {
    transform: scale(0.98);
}

.btn--small {
    padding: 6px 14px;
    font-size: 0.82rem;
}

.btn--ghost {
    background: transparent;
    color: var(--color-text);
    border-color: var(--color-border);
}

.btn--ghost:hover {
    border-color: var(--color-accent);
    color: var(--color-accent);
    background: transparent;
}

.btn--danger {
    background: transparent;
    border-color: #C0392B;
    color: #C0392B;
}

.btn--danger:hover {
    background: #C0392B;
    color: #fff;
}

.btn--block {
    display: block;
    width: 100%;
    text-align: center;
    margin-top: 8px;
}

/* ===== Формы (общие для всего сайта — Django .as_p) ===== */

main form p {
    margin-bottom: 16px;
}

main form label {
    display: block;
    font-weight: 600;
    margin-bottom: 6px;
    color: var(--color-text);
}

main input[type="text"],
main input[type="email"],
main input[type="password"],
main input[type="number"],
main input[type="file"],
main textarea,
main select {
    width: 100%;
    padding: 10px 14px;
    border: 1px solid var(--color-border);
    border-radius: var(--radius-sm);
    background: var(--color-surface);
    font-family: inherit;
    font-size: 1rem;
    color: var(--color-text);
}

main input:focus, main textarea:focus, main select:focus {
    outline: none;
    border-color: var(--color-primary);
    box-shadow: 0 0 0 3px rgba(224, 168, 0, 0.18);
}

main small {
    color: var(--color-text-secondary);
    font-size: 0.85rem;
}

.error, main .form-errors, main .errorlist {
    color: #C0392B;
    font-size: 0.9rem;
    list-style: none;
}

/* ===== Общий layout ===== */

.site-main {
    max-width: 1200px;
    margin: 0 auto;
    padding: 28px 24px 60px;
}

.section-title {
    font-size: 1.3rem;
    font-weight: 700;
    margin: 24px 0 16px;
    color: var(--color-black);
    text-transform: uppercase;
    letter-spacing: 0.3px;
}

.empty-state {
    color: var(--color-text-secondary);
    padding: 24px 0;
}

/* ===== Сетка карточек фильмов ===== */

.film-grid {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 26px;
    margin-top: 20px;
}

.film-card {
    background: var(--color-surface);
    border: 1px solid var(--color-border);
    border-radius: var(--radius);
    overflow: hidden;
    box-shadow: var(--shadow-card);
    text-align: center;
    transition: box-shadow 0.15s, transform 0.15s;
}

.film-card:hover {
    box-shadow: var(--shadow-card-hover);
    transform: translateY(-2px);
}

.film-card__poster-link {
    display: block;
}

.film-card__poster,
.film-card__poster--placeholder {
    width: 100%;
    aspect-ratio: 4 / 3;
    object-fit: cover;
    object-position: center top;
    display: block;
}

.film-card__poster--placeholder {
    background: #DDD;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #888;
    font-size: 0.85rem;
}

.film-card__body {
    padding: 18px 16px 22px;
}

.film-card__title {
    font-size: 0.95rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.3px;
    margin-bottom: 8px;
    color: var(--color-text);
}

.film-card__title a {
    color: inherit;
}

.film-card__description {
    font-size: 0.85rem;
    color: var(--color-text-secondary);
}

/* ===== Последние добавления (обёртка и ряд — карточки внутри стилизуются ниже, в разделе про главную страницу) ===== */

.latest-films {
    margin-bottom: 8px;
}

.latest-films__row {
    display: flex;
    gap: 16px;
    overflow-x: auto;
    padding-bottom: 8px;
}

/* ===== Пагинация ===== */

.pagination {
    display: flex;
    justify-content: center;
    align-items: center;
    gap: 8px;
    margin-top: 36px;
    flex-wrap: wrap;
}

.pagination a,
.pagination__current,
.pagination__ellipsis {
    min-width: 36px;
    height: 36px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: var(--radius-sm);
    font-size: 0.85rem;
    font-weight: 600;
}

.pagination a {
    background: var(--color-surface);
    border: 1px solid var(--color-border);
    color: var(--color-text);
}

.pagination a:hover {
    border-color: var(--color-accent);
    color: var(--color-accent);
}

.pagination__current {
    background: var(--color-accent);
    color: #1A1A1A;
}

.pagination__ellipsis {
    color: var(--color-text-secondary);
}

/* ===== Футер ===== */

.site-footer {
    background: var(--color-header-bg);
    color: var(--color-text-on-dark-secondary);
    padding: 24px;
    margin-top: 48px;
}

.site-footer__inner {
    max-width: 1200px;
    margin: 0 auto;
    display: flex;
    align-items: center;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: 16px;
}

.site-footer__brand {
    display: flex;
    align-items: center;
    gap: 12px;
}

.site-footer__brand-name {
    color: #fff;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.4px;
}

.site-footer__contacts {
    text-align: right;
    font-size: 0.85rem;
    line-height: 1.6;
}

.site-footer__contacts a {
    color: var(--color-text-on-dark-secondary);
}

/* ===== Адаптив ===== */

@media (max-width: 900px) {
    .film-grid {
        grid-template-columns: repeat(2, 1fr);
    }
}

@media (max-width: 600px) {
    .film-grid {
        grid-template-columns: 1fr;
    }

    .site-search {
        order: 3;
        max-width: 100%;
        flex-basis: 100%;
    }

    .site-footer__inner {
        flex-direction: column;
        text-align: center;
    }

    .site-footer__contacts {
        text-align: center;
    }
}
```

## 4. Пагинация с многоточием — через встроенный метод Paginator

```python
# films/mixins.py
class PaginationRangeMixin:
    """Готовит компактный список номеров страниц с многоточиями для пагинации."""

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        page_obj = context.get('page_obj')
        if page_obj:
            context['page_range'] = page_obj.paginator.get_elided_page_range(
                page_obj.number, on_each_side=1, on_ends=1
            )
        return context
```

```html
<!-- films/templates/films/includes/pagination.html -->
{% if is_paginated %}
<nav class="pagination">
    {% if page_obj.has_previous %}
        <a href="?{% if querystring %}{{ querystring }}&{% endif %}page={{ page_obj.previous_page_number }}">‹</a>
    {% endif %}

    {% for page_num in page_range %}
        {% if page_num == page_obj.number %}
            <span class="pagination__current">{{ page_num }}</span>
        {% elif page_num == '…' %}
            <span class="pagination__ellipsis">…</span>
        {% else %}
            <a href="?{% if querystring %}{{ querystring }}&{% endif %}page={{ page_num }}">{{ page_num }}</a>
        {% endif %}
    {% endfor %}

    {% if page_obj.has_next %}
        <a href="?{% if querystring %}{{ querystring }}&{% endif %}page={{ page_obj.next_page_number }}">›</a>
    {% endif %}
</nav>
{% endif %}
```

```python
# films/views.py
class FilmListView(FilmQuerySetMixin, PaginationRangeMixin, ListView):
    model = Film
    template_name = 'films/film_list.html'
    context_object_name = 'films'
    paginate_by = 10
```

## 5. Главная страница целиком: фильтры, поиск, сетка, последние добавления

### FilmFilterForm

```python
# films/forms.py
from .models import Genre


class FilmFilterForm(forms.Form):
    genre = forms.ModelChoiceField(
        queryset=Genre.objects.all(),
        required=False,
        empty_label='Все жанры',
        label='Жанр',
    )
    year = forms.IntegerField(
        required=False,
        label='Год выпуска',
        widget=forms.NumberInput(attrs={'placeholder': 'Например, 2010'}),
    )
    min_rating = forms.DecimalField(
        required=False,
        label='Рейтинг от',
        max_digits=3,
        decimal_places=1,
        widget=forms.NumberInput(attrs={'step': '0.1', 'placeholder': '0.0'}),
    )
```

### FilmFilterMixin

Жанры сразу выводятся вкладками с ограничением «первые 5 + разворот» — сразу пишем финальную версию:

```python
# films/mixins.py
from .forms import FilmFilterForm
from .models import Genre


class FilmFilterMixin:
    """Применяет фильтрацию по жанру/году/рейтингу через GET-параметры."""

    def get_filter_form(self):
        return FilmFilterForm(self.request.GET or None)

    def get_queryset(self):
        queryset = super().get_queryset()
        form = self.get_filter_form()

        if form.is_valid():
            genre = form.cleaned_data.get('genre')
            year = form.cleaned_data.get('year')
            min_rating = form.cleaned_data.get('min_rating')

            if genre:
                queryset = queryset.filter(genres=genre)
            if year:
                queryset = queryset.filter(year=year)
            if min_rating:
                queryset = queryset.filter(rating__gte=min_rating)

        return queryset

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['filter_form'] = self.get_filter_form()

        genres = list(Genre.objects.all())
        context['genres'] = genres

        selected_genre_id = self.request.GET.get('genre', '')
        context['selected_genre_id'] = selected_genre_id

        # если выбранный жанр находится за пределами первых пяти — сразу разворачиваем список,
        # иначе активная вкладка окажется скрыта без каких-либо объяснений
        visible_ids = {str(g.pk) for g in genres[:5]}
        context['genre_extra_expanded'] = bool(selected_genre_id) and selected_genre_id not in visible_ids

        # для пагинации — сохраняем ВСЕ активные фильтры
        get_params = self.request.GET.copy()
        get_params.pop('page', None)
        context['querystring'] = get_params.urlencode()

        # для вкладок жанра — сохраняем year/min_rating, но не сам genre
        # (иначе при клике на другой жанр в ссылке остался бы старый genre вторым параметром)
        genre_link_params = self.request.GET.copy()
        genre_link_params.pop('page', None)
        genre_link_params.pop('genre', None)
        context['genre_link_querystring'] = genre_link_params.urlencode()

        return context
```

### IndexView

```python
# films/views.py
class IndexView(FilmFilterMixin, PaginationRangeMixin, FilmQuerySetMixin, ListView):
    model = Film
    template_name = 'films/index.html'
    context_object_name = 'films'
    paginate_by = 6

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['title'] = 'Лучшие фильмы всех времён'
        return context
```

### index.html — итоговая версия целиком

Табы жанров сворачиваются после первых пяти через CSS-приём на скрытом чекбоксе — без единой строчки JavaScript. Блок «Последние добавления» — под пагинацией, не над контентом:

```html
{% extends 'base.html' %}
{% load film_tags %}

{% block title %}Главная — Film Collection{% endblock %}

{% block filter_strip %}
<div class="genre-strip">
    <div class="genre-strip__inner">
        <p class="genre-strip__title">Фильтрация по жанрам</p>
        <nav class="genre-tabs">
            <a href="{% url 'films:index' %}{% if genre_link_querystring %}?{{ genre_link_querystring }}{% endif %}"
               class="{% if not selected_genre_id %}active{% endif %}">Все</a>

            {% for genre in genres|slice:":5" %}
                <a href="?{% if genre_link_querystring %}{{ genre_link_querystring }}&{% endif %}genre={{ genre.pk }}"
                   class="{% if selected_genre_id == genre.pk|stringformat:'s' %}active{% endif %}">{{ genre.name }}</a>
            {% endfor %}

            {% if genres|length > 5 %}
                <input type="checkbox" id="genre-toggle" class="genre-toggle-checkbox" {% if genre_extra_expanded %}checked{% endif %}>

                <span class="genre-tabs__extra">
                    {% for genre in genres|slice:"5:" %}
                        <a href="?{% if genre_link_querystring %}{{ genre_link_querystring }}&{% endif %}genre={{ genre.pk }}"
                           class="{% if selected_genre_id == genre.pk|stringformat:'s' %}active{% endif %}">{{ genre.name }}</a>
                    {% endfor %}
                </span>

                <label for="genre-toggle" class="genre-tabs__more genre-tabs__more--show">Больше ▾</label>
                <label for="genre-toggle" class="genre-tabs__more genre-tabs__more--hide">Свернуть ▴</label>
            {% endif %}
        </nav>
    </div>
</div>
{% endblock %}

{% block content %}
<div class="quick-filters">
    <form method="GET" class="quick-filters__form">
        {% if selected_genre_id %}
            <input type="hidden" name="genre" value="{{ selected_genre_id }}">
        {% endif %}
        <div class="quick-filters__field">
            {{ filter_form.year.label_tag }}
            {{ filter_form.year }}
        </div>
        <div class="quick-filters__field">
            {{ filter_form.min_rating.label_tag }}
            {{ filter_form.min_rating }}
        </div>
        <button type="submit" class="btn btn--primary btn--small">Показать</button>
        {% if request.GET.year or request.GET.min_rating %}
            <a href="?{% if selected_genre_id %}genre={{ selected_genre_id }}{% endif %}" class="btn btn--ghost btn--small">Сбросить</a>
        {% endif %}
    </form>
</div>

<h1 class="section-title">Каталог фильмов</h1>

<div class="film-grid">
    {% for film in films %}
        {% include 'films/includes/film_card.html' %}
    {% empty %}
        <p class="empty-state">По заданным фильтрам фильмов не найдено.</p>
    {% endfor %}
</div>

{% include 'films/includes/pagination.html' %}

{% latest_films 5 %}
{% endblock %}
```

### film_card.html

```html
<!-- films/templates/films/includes/film_card.html -->
<article class="film-card">
    <a href="{{ film.get_absolute_url }}" class="film-card__poster-link">
        {% if film.poster %}
            <img src="{{ film.poster.url }}" alt="{{ film.title }}" class="film-card__poster">
        {% else %}
            <div class="film-card__poster--placeholder">Нет постера</div>
        {% endif %}
    </a>
    <div class="film-card__body">
        <h3 class="film-card__title">
            <a href="{{ film.get_absolute_url }}">{{ film.title }}</a>
        </h3>
        <p class="film-card__description">{{ film.description|default:"Описание отсутствует"|truncatechars:70 }}</p>
    </div>
</article>
```

### film_list.html

```html
<!-- films/templates/films/film_list.html -->
{% extends 'base.html' %}

{% block content %}
    <h1 class="section-title">Каталог фильмов</h1>

    <div class="film-grid">
        {% for film in films %}
            {% include 'films/includes/film_card.html' %}
        {% empty %}
            <p class="empty-state">В каталоге пока нет фильмов.</p>
        {% endfor %}
    </div>

    {% include 'films/includes/pagination.html' %}
{% endblock %}
```

### latest_films.html — только постеры, без подписей

```html
<!-- films/templates/films/includes/latest_films.html -->
{% load film_tags %}

<div class="latest-films">
    <h2 class="section-title">Последние добавления</h2>
    <div class="latest-films__row">
        {% for film in films %}
            <a href="{{ film.get_absolute_url }}" class="latest-films__item" title="{{ film.title }}">
                {% if film.poster %}
                    <img src="{{ film.poster.url }}" alt="{{ film.title }}">
                {% else %}
                    <div class="latest-films__placeholder">Нет постера</div>
                {% endif %}
            </a>
        {% empty %}
            <p class="empty-state">Фильмов пока нет.</p>
        {% endfor %}
    </div>
</div>
```

`title="{{ film.title }}"` — название доступно всплывающей подсказкой при наведении и через `alt` для скринридеров, просто не выводится текстом на странице.

### CSS для главной страницы — добавить в base.css

```css
/* ===== Разворачивание жанров: «Больше»/«Свернуть» без JS ===== */

.genre-toggle-checkbox {
    display: none;
}

.genre-tabs__extra {
    display: none;
    gap: 22px;
}

.genre-toggle-checkbox:checked ~ .genre-tabs__extra {
    display: contents;
}

.genre-tabs__more {
    cursor: pointer;
    color: var(--color-accent);
    font-weight: 600;
    font-size: 0.88rem;
    text-transform: uppercase;
}

.genre-tabs__more--hide {
    display: none;
}

.genre-toggle-checkbox:checked ~ .genre-tabs__more--show {
    display: none;
}

.genre-toggle-checkbox:checked ~ .genre-tabs__more--hide {
    display: inline;
}

/* ===== Последние добавления — карточки ===== */

.latest-films__item {
    flex: 0 0 auto;
    width: 130px;
}

.latest-films__item img,
.latest-films__placeholder {
    width: 130px;
    height: 185px;
    object-fit: cover;
    border-radius: var(--radius-sm);
    box-shadow: var(--shadow-card);
    display: block;
    transition: transform 0.15s;
}

.latest-films__item:hover img {
    transform: scale(1.03);
}

.latest-films__placeholder {
    background: var(--color-peach);
    display: flex;
    align-items: center;
    justify-content: center;
    color: var(--color-text-secondary);
    font-size: 0.8rem;
    text-align: center;
    padding: 8px;
}
```

Приём разворачивания жанров работает так: `.genre-tabs__extra` по умолчанию `display: none`. Когда чекбокс `#genre-toggle` отмечен — срабатывает `:checked ~ .genre-tabs__extra`, и вместо `none` применяется `display: contents`. Это значение убирает у обёртки собственный блок из потока вёрстки, но оставляет дочерние `<a>` полноправными flex-элементами родительского `.genre-tabs` — скрытые жанры визуально «вливаются» в общий ряд, а не сидят отдельным блоком. Клик по `<label for="genre-toggle">` переключает чекбокс нативно, HTML делает это сам.

---

## 6. Видео-поле на Film: загруженный файл или внешняя ссылка

### Валидатор размера файла

```python
# films/validators.py
def validate_video_size(value):
    max_size_mb = 50
    if value.size > max_size_mb * 1024 * 1024:
        raise ValidationError(f'Размер видео не должен превышать {max_size_mb} МБ.')
```

### Поля Film.video_file и Film.video_url

Приоритет — за файлом: если он загружен, ссылка игнорируется, даже если тоже заполнена.

```python
# films/models.py
from django.core.validators import FileExtensionValidator
from .validators import validate_file_size, validate_video_size


class Film(models.Model):
    # ... существующие поля без изменений ...

    video_file = models.FileField(
        upload_to='videos/',
        blank=True,
        validators=[validate_video_size, FileExtensionValidator(allowed_extensions=['mp4', 'webm', 'ogg'])],
        verbose_name='Видеофайл',
        help_text='MP4/WebM/OGG, до 50 МБ. Используется вместо ссылки, если загружен.',
    )
    video_url = models.URLField(
        blank=True,
        verbose_name='Ссылка на видео',
        help_text='Например, ссылка на YouTube — используется, если файл не загружен.',
    )

    def clean(self):
        super().clean()
        if self.video_file and self.video_url:
            raise ValidationError('Укажите либо файл, либо ссылку на видео — не оба варианта сразу.')

    def get_video_embed_url(self):
        """Преобразует обычную ссылку YouTube в embed-формат для <iframe>."""
        if not self.video_url:
            return ''
        url = self.video_url
        if 'youtu.be/' in url:
            video_id = url.split('youtu.be/')[-1].split('?')[0]
            return f'https://www.youtube.com/embed/{video_id}'
        if 'watch?v=' in url:
            video_id = url.split('watch?v=')[-1].split('&')[0]
            return f'https://www.youtube.com/embed/{video_id}'
        return url  # уже embed-ссылка или хостинг, не требующий преобразования
```

### Миграция

```bash
python manage.py makemigrations
python manage.py migrate
```

### FilmForm — с учётом новых полей

```python
# films/forms.py
class FilmForm(forms.ModelForm):
    class Meta:
        model = Film
        fields = ['title', 'year', 'description', 'director', 'genres', 'poster', 'video_file', 'video_url']
        widgets = {
            'description': forms.Textarea(attrs={'rows': 5}),
        }

    def clean_title(self):
        title = self.cleaned_data['title']
        if title.strip().isdigit():
            raise forms.ValidationError('Название фильма не может состоять только из цифр.')
        return title
```

Шаблон `add_film.html` менять не нужно — новые поля появляются в форме автоматически через `.as_p`.

---

## 7. Review: связь с User, публикация, доработка формы

### Модель

```python
# films/models.py
class Review(models.Model):
    film = models.ForeignKey(
        Film, on_delete=models.CASCADE, related_name='reviews', verbose_name='Фильм'
    )
    author = models.ForeignKey(
        User,
        on_delete=models.SET_NULL,
        null=True,
        blank=True,
        related_name='reviews',
        verbose_name='Автор',
    )
    author_name = models.CharField(
        max_length=100,
        blank=True,
        verbose_name='Имя автора (снимок на момент публикации)',
    )
    text = models.TextField(verbose_name='Текст рецензии')
    rating = models.PositiveSmallIntegerField(
        validators=[MinValueValidator(1), MaxValueValidator(10)],
        help_text='Оценка от 1 до 10',
    )
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

    def __str__(self):
        return f'Рецензия на «{self.film.title}» от {self.author_name}'

    def get_display_author(self):
        """Актуальное имя автора, если аккаунт ещё существует, иначе — сохранённый снимок."""
        if self.author:
            return self.author.get_full_name() or self.author.username
        return self.author_name or 'Пользователь удалён'
```

`on_delete=SET_NULL` + `author_name` как «снимок»: если пользователь удалит аккаунт, рецензия не исчезнет, `author` станет `NULL`, но имя, сохранённое на момент публикации, останется — `get_display_author()` покажет его как fallback.

`is_published` по умолчанию `False` — рецензия ждёт модерации, если только у автора нет разрешения `films.publish_review`, которое публикует её сразу.

### Миграция

```bash
python manage.py makemigrations
python manage.py migrate
```

### ReviewForm — имя больше не спрашивается вручную

```python
# films/forms.py
class ReviewForm(forms.ModelForm):
    class Meta:
        model = Review
        fields = ['text', 'rating']
        widgets = {
            'text': forms.Textarea(attrs={'rows': 4, 'placeholder': 'Поделитесь впечатлениями...'}),
            'rating': forms.NumberInput(attrs={'min': 1, 'max': 10}),
        }
        labels = {
            'text': 'Рецензия',
            'rating': 'Оценка (1–10)',
        }

    def clean_text(self):
        text = self.cleaned_data['text']
        if len(text.split()) < 5:
            raise forms.ValidationError('Рецензия слишком короткая — напишите хотя бы 5 слов.')
        return text

    def clean(self):
        cleaned_data = super().clean()
        rating = cleaned_data.get('rating')
        text = cleaned_data.get('text')
        if rating and rating <= 3 and text and len(text) < 50:
            raise forms.ValidationError('Для низкой оценки распишите причину подробнее — минимум 50 символов.')
        return cleaned_data
```

### AddReviewView — автор подставляется на сервере

```python
# films/views.py
class AddReviewView(LoginRequiredMixin, FormView):
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
        review.author = self.request.user
        review.author_name = self.request.user.get_full_name() or self.request.user.username
        if self.request.user.has_perm('films.publish_review'):
            review.is_published = True
        review.save()
        return redirect(film.get_absolute_url())
```

Шаблон `add_review.html` рендерит поля циклом `{% for field in form %}` — поле `author_name` пропадёт из формы автоматически, шаблон трогать не нужно.

---

## 8. Остальные компоненты в base.css

Используются страницами из следующих разделов: детальные страницы, подтверждение удаления, формы, авторизация, профиль, статистика, видео, рецензии.

```css
/* ===== Детальные страницы (фильм / режиссёр) ===== */

.detail-layout {
    display: grid;
    grid-template-columns: 280px 1fr;
    gap: 32px;
    margin-bottom: 40px;
}

.detail-poster img,
.detail-poster__placeholder {
    width: 100%;
    border-radius: var(--radius);
    box-shadow: var(--shadow-card);
    display: block;
}

.detail-poster__placeholder {
    aspect-ratio: 2 / 3;
    background: var(--color-peach);
    display: flex;
    align-items: center;
    justify-content: center;
    color: var(--color-text-secondary);
}

.detail-info h1 {
    font-size: 1.8rem;
    margin-bottom: 10px;
    color: var(--color-black);
}

.detail-meta {
    display: flex;
    align-items: center;
    gap: 10px;
    flex-wrap: wrap;
    color: var(--color-text-secondary);
    margin-bottom: 14px;
}

.detail-description {
    margin: 16px 0;
    color: var(--color-text);
}

.detail-views {
    color: var(--color-text-secondary);
    font-size: 0.9rem;
    margin-bottom: 16px;
}

.badge {
    display: inline-block;
    padding: 3px 10px;
    border-radius: 20px;
    font-size: 0.8rem;
    font-weight: 600;
}

.badge--rating {
    background: var(--color-black);
    color: var(--color-accent);
}

.chip-row {
    display: flex;
    gap: 8px;
    flex-wrap: wrap;
    margin-bottom: 16px;
}

.chip {
    background: var(--color-peach);
    color: var(--color-black);
    padding: 4px 12px;
    border-radius: 20px;
    font-size: 0.82rem;
}

.cast-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(90px, 1fr));
    gap: 14px;
    margin: 16px 0 24px;
}

.cast-item {
    text-align: center;
    font-size: 0.8rem;
}

.cast-item img,
.cast-item__placeholder {
    width: 100%;
    aspect-ratio: 1 / 1;
    object-fit: cover;
    border-radius: 50%;
    box-shadow: var(--shadow-card);
    margin-bottom: 6px;
    display: block;
}

.cast-item__placeholder {
    background: var(--color-primary);
    color: #1A1A1A;
    display: flex;
    align-items: center;
    justify-content: center;
    font-weight: 700;
    font-size: 1.2rem;
}

.actions-row {
    display: flex;
    gap: 10px;
    flex-wrap: wrap;
    margin: 20px 0;
}

.back-link {
    display: inline-block;
    margin-top: 10px;
    color: var(--color-text-secondary);
}

.back-link:hover {
    color: var(--color-primary);
}

/* ===== Подтверждение удаления ===== */

.confirm-card {
    max-width: 480px;
    margin: 40px auto;
    background: var(--color-surface);
    border: 1px solid var(--color-border);
    border-left: 4px solid #C0392B;
    border-radius: var(--radius);
    padding: 28px;
    text-align: center;
}

.confirm-card h1 {
    margin-bottom: 12px;
    color: var(--color-black);
}

.confirm-card p {
    color: var(--color-text-secondary);
    margin-bottom: 20px;
}

.confirm-card__actions {
    display: flex;
    gap: 12px;
    justify-content: center;
}

/* ===== Страницы с формами (добавление/редактирование) ===== */

.form-page {
    max-width: 640px;
    margin: 0 auto;
    background: var(--color-surface);
    border: 1px solid var(--color-border);
    border-radius: var(--radius);
    padding: 32px;
    box-shadow: var(--shadow-card);
}

.form-page h1 {
    margin-bottom: 20px;
    color: var(--color-black);
    font-size: 1.5rem;
}

.form-field {
    margin-bottom: 16px;
}

/* ===== Авторизационные страницы ===== */

.auth-card {
    max-width: 420px;
    margin: 40px auto;
    background: var(--color-surface);
    border: 1px solid var(--color-border);
    border-radius: var(--radius);
    padding: 32px;
    box-shadow: var(--shadow-card);
}

.auth-card h1 {
    margin-bottom: 20px;
    font-size: 1.4rem;
    text-align: center;
    color: var(--color-black);
}

.auth-card p {
    margin-top: 14px;
    text-align: center;
    font-size: 0.9rem;
    color: var(--color-text-secondary);
}

.auth-card__divider {
    text-align: center;
    color: var(--color-text-secondary);
    margin: 18px 0;
    font-size: 0.85rem;
}

/* ===== Профиль ===== */

.profile-avatar {
    display: block;
    width: 120px;
    height: 120px;
    border-radius: 50%;
    object-fit: cover;
    margin: 0 auto 20px;
    box-shadow: var(--shadow-card);
}

/* ===== Статистика ===== */

.stats-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(160px, 1fr));
    gap: 16px;
    margin-bottom: 32px;
}

.stat-card {
    background: var(--color-surface);
    border: 1px solid var(--color-border);
    border-radius: var(--radius);
    padding: 20px;
    text-align: center;
}

.stat-card__value {
    display: block;
    font-size: 2rem;
    font-weight: 700;
    color: var(--color-primary);
}

.stat-card__label {
    color: var(--color-text-secondary);
    font-size: 0.85rem;
}

.simple-list {
    list-style: none;
    margin-bottom: 24px;
}

.simple-list li {
    padding: 10px 14px;
    border-bottom: 1px solid var(--color-border);
}

.simple-list li:last-child {
    border-bottom: none;
}

/* ===== Видео на странице фильма ===== */

.video-block {
    margin: 20px 0 28px;
}

.video-block__title {
    font-size: 1.1rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.3px;
    margin-bottom: 10px;
    color: var(--color-black);
}

.video-frame {
    position: relative;
    width: 100%;
    aspect-ratio: 16 / 9;
    background: #000;
    border-radius: var(--radius);
    overflow: hidden;
    box-shadow: var(--shadow-card);
}

.video-frame video,
.video-frame iframe {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    border: 0;
    display: block;
}

/* ===== Рецензии ===== */

.reviews-block {
    margin: 8px 0 24px;
}

.review-card {
    background: var(--color-surface);
    border: 1px solid var(--color-border);
    border-radius: var(--radius);
    box-shadow: var(--shadow-card);
    padding: 16px 18px;
    margin-bottom: 14px;
}

.review-card__header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 8px;
    font-weight: 600;
    color: var(--color-black);
}

.review-card__rating {
    background: var(--color-black);
    color: var(--color-accent);
    padding: 2px 10px;
    border-radius: 20px;
    font-size: 0.8rem;
    font-weight: 700;
}

.review-card__text {
    color: var(--color-text-secondary);
    font-size: 0.9rem;
    line-height: 1.5;
}

.review-card__date {
    font-size: 0.75rem;
    color: var(--color-text-secondary);
    margin-top: 8px;
}

/* ===== Адаптив для детальных страниц ===== */

@media (max-width: 700px) {
    .detail-layout {
        grid-template-columns: 1fr;
    }
}
```

---

## 9. Детальная страница фильма и режиссёра

### FilmDetailView — передаёт только опубликованные рецензии

Фильтрация сделана во вьюшке, а не прямо в шаблоне (`{% if review.is_published %}` внутри цикла) — иначе `{% empty %}` сработал бы только если рецензий вообще нет, а не если все существующие ещё не опубликованы.

```python
# films/views.py
class FilmDetailView(FilmQuerySetMixin, RecentFilmsMixin, DetailView):
    model = Film
    template_name = 'films/film_detail.html'
    context_object_name = 'film'

    def get(self, request, *args, **kwargs):
        response = super().get(request, *args, **kwargs)
        FilmStats.objects.filter(film=self.object).update(views_count=F('views_count') + 1)
        return response

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['published_reviews'] = self.object.reviews.filter(is_published=True)
        return context
```

### film_detail.html

```html
<!-- films/templates/films/film_detail.html -->
{% extends 'base.html' %}

{% block title %}{{ film.title }} — Сайт фильмов{% endblock %}

{% block content %}
<div class="detail-layout">
    <div class="detail-poster">
        {% if film.poster %}
            <img src="{{ film.poster.url }}" alt="Постер фильма «{{ film.title }}»">
        {% else %}
            <div class="detail-poster__placeholder">Нет постера</div>
        {% endif %}
    </div>

    <div class="detail-info">
        <h1>{{ film.title }}</h1>

        <p class="detail-meta">
            <span class="badge badge--rating">★ {{ film.rating|floatformat:1 }}</span>
            {{ film.year }}
            {% if film.director %}
                · Режиссёр: <a href="{{ film.director.get_absolute_url }}">{{ film.director.name }}</a>
            {% endif %}
        </p>

        {% if film.genres.all %}
            <div class="chip-row">
                {% for genre in film.genres.all %}
                    <span class="chip">{{ genre.name }}</span>
                {% endfor %}
            </div>
        {% endif %}

        <p class="detail-description">{{ film.description|default:"Описание отсутствует." }}</p>

        {% if film.video_file %}
            <div class="video-block">
                <h2 class="video-block__title">Смотреть онлайн</h2>
                <div class="video-frame">
                    <video controls preload="metadata" {% if film.poster %}poster="{{ film.poster.url }}"{% endif %}>
                        <source src="{{ film.video_file.url }}">
                        Ваш браузер не поддерживает воспроизведение этого видео.
                    </video>
                </div>
            </div>
        {% elif film.video_url %}
            <div class="video-block">
                <h2 class="video-block__title">Смотреть онлайн</h2>
                <div class="video-frame">
                    <iframe src="{{ film.get_video_embed_url }}" title="{{ film.title }}" allowfullscreen></iframe>
                </div>
            </div>
        {% endif %}

        {% if film.stats %}
            <p class="detail-views">👁 {{ film.stats.views_count }} просмотров</p>
        {% endif %}

        {% if film.actors.all %}
            <h2 class="section-title">Актёры</h2>
            <div class="cast-grid">
                {% for actor in film.actors.all %}
                    <div class="cast-item">
                        {% if actor.photo %}
                            <img src="{{ actor.photo.url }}" alt="{{ actor.name }}">
                        {% else %}
                            <div class="cast-item__placeholder">{{ actor.name|first }}</div>
                        {% endif %}
                        <span>{{ actor.name }}</span>
                    </div>
                {% endfor %}
            </div>
        {% endif %}

        <h2 class="section-title">Рецензии</h2>
        <div class="reviews-block">
            {% for review in published_reviews %}
                <div class="review-card">
                    <div class="review-card__header">
                        <span>{{ review.get_display_author }}</span>
                        <span class="review-card__rating">★ {{ review.rating }}/10</span>
                    </div>
                    <p class="review-card__text">{{ review.text }}</p>
                    <p class="review-card__date">{{ review.created_at|date:"d.m.Y" }}</p>
                </div>
            {% empty %}
                <p class="empty-state">Пока никто не оставил рецензию — станьте первым!</p>
            {% endfor %}
        </div>

        <div class="actions-row">
            {% if user.is_authenticated %}
                <a href="{% url 'films:add_review' slug=film.slug %}" class="btn btn--primary">Оставить рецензию</a>
            {% endif %}
            {% if perms.films.change_film %}
                <a href="{% url 'films:film_edit' slug=film.slug %}" class="btn btn--ghost">Редактировать</a>
            {% endif %}
            {% if perms.films.delete_film %}
                <a href="{% url 'films:film_delete' slug=film.slug %}" class="btn btn--danger">Удалить</a>
            {% endif %}
        </div>

        <a href="{% url 'films:film_list' %}" class="back-link">← Назад к каталогу</a>
    </div>
</div>
{% endblock %}
```

### director_detail.html

```html
<!-- films/templates/films/director_detail.html -->
{% extends 'base.html' %}

{% block title %}{{ director.name }} — Сайт фильмов{% endblock %}

{% block content %}
<div class="detail-layout">
    <div class="detail-poster">
        {% if director.photo %}
            <img src="{{ director.photo.url }}" alt="{{ director.name }}">
        {% else %}
            <div class="detail-poster__placeholder">Нет фото</div>
        {% endif %}
    </div>

    <div class="detail-info">
        <h1>{{ director.name }}</h1>

        {% if director.bio %}
            <p class="detail-description">{{ director.bio }}</p>
        {% endif %}

        <div class="actions-row">
            {% if perms.films.change_director %}
                <a href="{% url 'films:director_edit' slug=director.slug %}" class="btn btn--ghost">Редактировать</a>
            {% endif %}
            {% if perms.films.delete_director %}
                <a href="{% url 'films:director_delete' slug=director.slug %}" class="btn btn--danger">Удалить</a>
            {% endif %}
        </div>
    </div>
</div>

<h2 class="section-title">Фильмы режиссёра</h2>
<div class="film-grid">
    {% for film in films %}
        {% include 'films/includes/film_card.html' %}
    {% empty %}
        <p class="empty-state">Фильмов этого режиссёра пока нет в каталоге.</p>
    {% endfor %}
</div>

<a href="{% url 'films:film_list' %}" class="back-link">← К каталогу</a>
{% endblock %}
```

---

## 10. Страницы подтверждения удаления

```html
<!-- films/templates/films/film_confirm_delete.html -->
{% extends 'base.html' %}

{% block title %}Удалить {{ film.title }}{% endblock %}

{% block content %}
<div class="confirm-card">
    <h1>Удалить фильм?</h1>
    <p>Вы уверены, что хотите удалить «{{ film.title }}»? Это действие необратимо.</p>

    <form method="POST" class="confirm-card__actions">
        {% csrf_token %}
        <button type="submit" class="btn btn--danger">Да, удалить</button>
        <a href="{{ film.get_absolute_url }}" class="btn btn--ghost">Отменить</a>
    </form>
</div>
{% endblock %}
```

```html
<!-- films/templates/films/director_confirm_delete.html -->
{% extends 'base.html' %}

{% block title %}Удалить режиссёра{% endblock %}

{% block content %}
<div class="confirm-card">
    <h1>Удалить режиссёра «{{ director.name }}»?</h1>
    <p>Фильмы режиссёра останутся в каталоге, но поле «Режиссёр» будет сброшено.</p>

    <form method="POST" class="confirm-card__actions">
        {% csrf_token %}
        <button type="submit" class="btn btn--danger">Да, удалить</button>
        <a href="{{ director.get_absolute_url }}" class="btn btn--ghost">Отменить</a>
    </form>
</div>
{% endblock %}
```

---

## 11. Формы: добавление/редактирование фильма, режиссёра, актёра, рецензии

```html
<!-- films/templates/films/add_film.html -->
{% extends 'base.html' %}

{% block title %}
    {% if object %}Изменить фильм{% else %}Добавить фильм{% endif %}
{% endblock %}

{% block content %}
<div class="form-page">
    <h1>{% if object %}Редактировать фильм{% else %}Добавить новый фильм{% endif %}</h1>
    <form method="POST" enctype="multipart/form-data">
        {% csrf_token %}
        {{ form.as_p }}
        <button type="submit" class="btn btn--primary btn--block">Сохранить</button>
    </form>
</div>
{% endblock %}
```

```html
<!-- films/templates/films/director_form.html -->
{% extends 'base.html' %}

{% block title %}
    {% if object %}Редактировать режиссёра{% else %}Добавить режиссёра{% endif %}
{% endblock %}

{% block content %}
<div class="form-page">
    <h1>{% if object %}Редактировать: {{ object.name }}{% else %}Добавить режиссёра{% endif %}</h1>
    <form method="POST" enctype="multipart/form-data">
        {% csrf_token %}
        {{ form.as_p }}
        <button type="submit" class="btn btn--primary btn--block">Сохранить</button>
    </form>
</div>
{% endblock %}
```

```html
<!-- films/templates/films/add_actor.html -->
{% extends 'base.html' %}

{% block title %}Добавить актёра{% endblock %}

{% block content %}
<div class="form-page">
    <h1>Добавить актёра</h1>
    <form method="POST" enctype="multipart/form-data">
        {% csrf_token %}
        {{ form.as_p }}
        <button type="submit" class="btn btn--primary btn--block">Сохранить</button>
    </form>
</div>
{% endblock %}
```

```html
<!-- films/templates/films/add_review.html -->
{% extends 'base.html' %}

{% block title %}Рецензия на «{{ film.title }}»{% endblock %}

{% block content %}
<div class="form-page">
    <h1>Оставить рецензию на «{{ film.title }}»</h1>

    <form method="POST">
        {% csrf_token %}

        {% if form.non_field_errors %}
            <div class="form-errors">{{ form.non_field_errors }}</div>
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

        <button type="submit" class="btn btn--primary btn--block">Опубликовать</button>
    </form>
</div>
{% endblock %}
```

---

## 12. Авторизация: login, register, восстановление и смена пароля

```html
<!-- templates/registration/login.html -->
{% extends 'base.html' %}

{% block title %}Вход — Сайт фильмов{% endblock %}

{% block content %}
<div class="auth-card">
    <h1>Вход на сайт</h1>

    {% if form.errors %}
        <div class="error">
            <p>Неверное имя пользователя или пароль. Попробуйте снова.</p>
        </div>
    {% endif %}

    <form method="POST">
        {% csrf_token %}
        {{ form.as_p }}
        <input type="hidden" name="next" value="{{ next }}">
        <button type="submit" class="btn btn--primary btn--block">Войти</button>
    </form>

    <p class="auth-card__divider">или</p>

    <form action="{% url 'social:begin' 'github' %}" method="post">
        {% csrf_token %}
        <button type="submit" class="btn btn--ghost btn--block">Войти через GitHub</button>
    </form>

    <p><a href="{% url 'password_reset' %}">Забыли пароль?</a></p>
    <p>Ещё нет аккаунта? <a href="{% url 'register' %}">Зарегистрироваться</a></p>
</div>
{% endblock %}
```

```html
<!-- templates/registration/register.html -->
{% extends 'base.html' %}

{% block title %}Регистрация{% endblock %}

{% block content %}
<div class="auth-card">
    <h1>Создать аккаунт</h1>

    <form method="POST">
        {% csrf_token %}
        {{ form.as_p }}
        <button type="submit" class="btn btn--primary btn--block">Зарегистрироваться</button>
    </form>

    <p>Уже есть аккаунт? <a href="{% url 'login' %}">Войти</a></p>
</div>
{% endblock %}
```

Оставшиеся пять шаблонов (`password_change_form.html`, `password_change_done.html`, `password_reset_form.html`, `password_reset_done.html`, `password_reset_confirm.html`, `password_reset_complete.html`) — просто оборачиваем содержимое `{% block content %}` в `<div class="auth-card">...</div>`, добавляем класс `btn btn--primary btn--block` кнопке. Пример:

```html
<!-- templates/registration/password_reset_confirm.html -->
{% extends 'base.html' %}

{% block title %}Новый пароль{% endblock %}

{% block content %}
<div class="auth-card">
    <h1>Введите новый пароль</h1>

    {% if validlink %}
        <form method="POST">
            {% csrf_token %}
            {% if form.non_field_errors %}
                <div class="error">{{ form.non_field_errors }}</div>
            {% endif %}
            {{ form.as_p }}
            <button type="submit" class="btn btn--primary btn--block">Сохранить пароль</button>
        </form>
    {% else %}
        <p>Ссылка для сброса пароля недействительна — возможно, она уже была использована или срок её действия истёк.</p>
        <p><a href="{% url 'password_reset' %}">Запросить новую ссылку</a></p>
    {% endif %}
</div>
{% endblock %}
```

---

## 13. Профиль пользователя

```html
<!-- films/templates/films/profile.html -->
{% extends 'base.html' %}

{% block title %}Профиль — {{ user.username }}{% endblock %}

{% block content %}
<div class="form-page">
    <h1>Профиль: {{ user.username }}</h1>

    {% if user.profile.avatar %}
        <img src="{{ user.profile.avatar.url }}" alt="Аватар" class="profile-avatar">
    {% endif %}

    <form method="POST" enctype="multipart/form-data">
        {% csrf_token %}

        <h2 class="section-title">Личные данные</h2>
        {{ user_form.as_p }}

        <h2 class="section-title">О себе</h2>
        {{ profile_form.as_p }}

        <button type="submit" class="btn btn--primary btn--block">Сохранить</button>
    </form>

    <p style="text-align: center; margin-top: 16px;">
        <a href="{% url 'password_change' %}">Сменить пароль</a>
    </p>
</div>
{% endblock %}
```

---

## 14. Статистика каталога и топ режиссёров

```html
<!-- films/templates/films/catalog_stats.html -->
{% extends 'base.html' %}

{% block title %}Статистика каталога{% endblock %}

{% block content %}
<h1 class="section-title">Статистика каталога</h1>

<div class="stats-grid">
    <div class="stat-card">
        <span class="stat-card__value">{{ overall_stats.total }}</span>
        <span class="stat-card__label">Всего фильмов</span>
    </div>
    <div class="stat-card">
        <span class="stat-card__value">{{ overall_stats.avg_rating|floatformat:1 }}</span>
        <span class="stat-card__label">Средний рейтинг</span>
    </div>
    <div class="stat-card">
        <span class="stat-card__value">{{ overall_stats.max_rating|floatformat:1 }}</span>
        <span class="stat-card__label">Максимальный рейтинг</span>
    </div>
</div>

<h2 class="section-title">Самые продуктивные режиссёры</h2>
<ul class="simple-list">
    {% for director in top_directors %}
        <li>{{ director.name }} — {{ director.film_count }} фильмов</li>
    {% endfor %}
</ul>

<h2 class="section-title">Фильмы по годам</h2>
<ul class="simple-list">
    {% for item in films_by_year %}
        <li>{{ item.year }}: {{ item.count }} фильмов</li>
    {% endfor %}
</ul>
{% endblock %}
```

```html
<!-- films/templates/films/top_directors.html -->
{% extends 'base.html' %}

{% block title %}Самые продуктивные режиссёры{% endblock %}

{% block content %}
<h1 class="section-title">Самые продуктивные режиссёры</h1>
<ul class="simple-list">
    {% for director in directors %}
        <li>{{ director.name }} — {{ director.film_count }} фильмов со средним рейтингом {{ director.avg_rating|floatformat:1 }}</li>
    {% endfor %}
</ul>
{% endblock %}
```

---

## 15. about.html и review_search.html

```html
<!-- films/templates/films/about.html -->
{% extends 'base.html' %}

{% block title %}О сайте{% endblock %}

{% block content %}
<div class="form-page">
    <h1>{{ title }}</h1>
    <p style="margin: 16px 0;">Это учебный проект — каталог фильмов, созданный на Django.</p>
    <p style="margin-bottom: 16px;">Сейчас в каталоге {{ film_count }} фильмов.</p>
    <a href="{% url 'films:index' %}" class="btn btn--ghost">← На главную</a>
</div>
{% endblock %}
```

```html
<!-- films/templates/films/review_search.html -->
{% extends 'base.html' %}

{% block title %}Поиск рецензий{% endblock %}

{% block content %}
<div class="form-page">
    <h1>Поиск рецензий</h1>
    <form method="GET">
        {{ form.as_p }}
        <button type="submit" class="btn btn--primary btn--block">Найти</button>
    </form>
</div>
{% endblock %}
```

---

## 16. Поиск по каталогу

Переиспользует ту же пагинацию и ту же сетку карточек, что и главная/каталог — никакой новой инфраструктуры не требуется.

```python
# films/views.py
def search_film(request):
    query = request.GET.get('q', '').strip()
    films = Film.objects.none()

    if query:
        films = Film.objects.search(query)

    paginator = Paginator(films, 10)
    page_number = request.GET.get('page', 1)

    try:
        page_obj = paginator.page(page_number)
    except InvalidPage:
        raise Http404('Страница не найдена')

    get_params = request.GET.copy()
    get_params.pop('page', None)

    context = {
        'films': page_obj.object_list,
        'page_obj': page_obj,
        'paginator': paginator,
        'is_paginated': paginator.num_pages > 1,
        'page_range': paginator.get_elided_page_range(page_obj.number, on_each_side=1, on_ends=1),
        'querystring': get_params.urlencode(),
        'query': query,
    }
    return render(request, 'films/search_results.html', context)
```

```html
<!-- films/templates/films/search_results.html -->
{% extends 'base.html' %}

{% block title %}Поиск: {{ query }}{% endblock %}

{% block content %}
<h1 class="section-title">Результаты поиска: «{{ query }}»</h1>

<div class="film-grid">
    {% for film in films %}
        {% include 'films/includes/film_card.html' %}
    {% empty %}
        <p class="empty-state">По запросу «{{ query }}» ничего не найдено.</p>
    {% endfor %}
</div>

{% include 'films/includes/pagination.html' %}

<a href="{% url 'films:film_list' %}" class="back-link">← К каталогу</a>
{% endblock %}
```

---

## 17. Страницы ошибок: 403, 404, 500

```html
<!-- templates/403.html -->
{% extends 'base.html' %}

{% block title %}Доступ запрещён{% endblock %}

{% block content %}
<div class="auth-card">
    <h1>403 — Доступ запрещён</h1>
    <p>У вас нет прав для выполнения этого действия.</p>
    {% if not user.is_authenticated %}
        <p><a href="{% url 'login' %}">Войдите в систему</a>, чтобы получить доступ.</p>
    {% else %}
        <p>Если вы считаете, что это ошибка — обратитесь к администратору.</p>
    {% endif %}
    <p><a href="{% url 'films:index' %}">← На главную</a></p>
</div>
{% endblock %}
```

```html
<!-- templates/404.html -->
{% extends 'base.html' %}

{% block title %}Страница не найдена{% endblock %}

{% block content %}
<div class="auth-card">
    <h1>404 — Страница не найдена</h1>
    <p>Такой страницы не существует — возможно, ссылка устарела или в адресе опечатка.</p>
    <p><a href="{% url 'films:index' %}">← На главную</a></p>
</div>
{% endblock %}
```

`500.html` намеренно **не** наследует `base.html` — если причина ошибки как раз в базе данных, попытка отрендерить `base.html` (контекстные процессоры `total_films`/`total_genres` обращаются к БД) сама упадёт с новой ошибкой:

```html
<!-- templates/500.html -->
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Ошибка сервера</title>
    <style>
        body { font-family: sans-serif; background: #F3F3F3; color: #1A1A1A; display: flex; align-items: center; justify-content: center; height: 100vh; margin: 0; }
        .box { background: #fff; border: 1px solid #E2E2E2; border-radius: 6px; padding: 32px; max-width: 420px; text-align: center; box-shadow: 0 2px 8px rgba(0,0,0,.08); }
        a { color: #E0A800; }
    </style>
</head>
<body>
    <div class="box">
        <h1>500 — Ошибка сервера</h1>
        <p>Что-то пошло не так на нашей стороне. Мы уже разбираемся — попробуйте обновить страницу через пару минут.</p>
        <p><a href="/">На главную</a></p>
    </div>
</body>
</html>
```

Django подхватывает `404.html`/`500.html` автоматически при `DEBUG = False`, если они лежат в корневой папке `templates/` — регистрировать `handler404`/`handler500` вручную не нужно.