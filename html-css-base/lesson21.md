# Урок 21. DOM + BOM: `querySelector`, создание/изменение элементов, события, `window`/`location`/`localStorage`

## Момент, ради которого мы всё это изучали

Все 20 предыдущих уроков модуля работали изолированно от настоящей страницы — переменные, функции, классы, замыкания существовали только в консоли или во всплывающих окнах `prompt`/`alert`. Сегодня это меняется: мы подключим JavaScript к той самой странице фильмов, которую строили в HTML/CSS-блоке курса (Уроки 11–15) — кнопка "избранное" наконец-то заработает, форма поиска перестанет перезагружать страницу, а карточки фильмов будут создаваться прямо из массива объектов, а не вручную прописываться в HTML.

---

<details>
<summary><strong>Отредактированный пример разметки и стилизации из Уроков 11-15</strong></summary>

### html-разметка:

```html
<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>MovieGrid — DOM + BOM</title>
  <link rel="stylesheet" href="style.css" />
</head>
<body class="page">

  <header class="header">

    <a href="#" class="header__logo">
      🎬 MovieGrid
    </a>

    <form class="search" role="search" action="/films" method="GET">
      <label class="visually-hidden" for="site-search">
        Поиск по сайту
      </label>

      <input
        type="search"
        id="site-search"
        name="q"
        class="search__input"
        placeholder="Найти фильм..."
        autocomplete="off"
      />

      <button
        type="submit"
        class="btn btn--primary"
      >
        Найти
      </button>
    </form>

    <nav class="header__nav">
      <a href="#">Главная</a>
      <a href="#">Фильмы</a>
      <a href="#">Контакты</a>
    </nav>

  </header>


  <main class="main">

    <section class="movies-section">

      <div class="section-header">
        <div>
          <p class="section-header__eyebrow">MovieGrid</p>
          <h1>Фильмы</h1>
        </div>

        <p class="section-header__description">
          Подборка популярных фильмов
        </p>
      </div>


      <div class="cards">

        <article class="card" data-movie-id="1" data-genre="Фантастика">

          <img
            class="card__cover"
            src="https://picsum.photos/300/180?1"
            alt="Постер фильма Матрица"
          />

          <div class="card__body">

            <h3 class="card__title">
              Матрица
            </h3>

            <p class="card__meta">
              1999 · 8.7/10
            </p>

            <p class="card__description">
              Программист узнаёт, что привычный мир является иллюзией.
            </p>

            <div class="card__actions">

              <a href="#" class="btn btn--secondary">
                Смотреть
              </a>

              <button
                type="button"
                class="card__favorite"
                aria-label="Добавить фильм в избранное"
              >
                ♥
              </button>

            </div>

          </div>

        </article>


        <article class="card" data-movie-id="2" data-genre="Фантастика">

          <img
            class="card__cover"
            src="https://picsum.photos/300/180?2"
            alt="Постер фильма Начало"
          />

          <div class="card__body">

            <h3 class="card__title">
              Начало
            </h3>

            <p class="card__meta">
              2010 · 8.8/10
            </p>

            <p class="card__description">
              Команда специалистов проникает в сны людей.
            </p>

            <div class="card__actions">

              <a href="#" class="btn btn--secondary">
                Смотреть
              </a>

              <button
                type="button"
                class="card__favorite"
                aria-label="Добавить фильм в избранное"
              >
                ♥
              </button>

            </div>

          </div>

        </article>


        <article class="card" data-movie-id="3" data-genre="Драма">

          <img
            class="card__cover"
            src="https://picsum.photos/300/180?3"
            alt="Постер фильма Крестный отец"
          />

          <div class="card__body">

            <h3 class="card__title">
              Крестный отец
            </h3>

            <p class="card__meta">
              1972 · 9.2/10
            </p>

            <p class="card__description">
              История семьи Корлеоне и её криминального бизнеса.
            </p>

            <div class="card__actions">

              <a href="#" class="btn btn--secondary">
                Смотреть
              </a>

              <button
                type="button"
                class="card__favorite"
                aria-label="Добавить фильм в избранное"
              >
                ♥
              </button>

            </div>

          </div>

        </article>


        <article class="card" data-movie-id="4" data-genre="Драма">

          <img
            class="card__cover"
            src="https://picsum.photos/300/180?4"
            alt="Постер фильма Интерстеллар"
          />

          <div class="card__body">

            <h3 class="card__title">
              Интерстеллар
            </h3>

            <p class="card__meta">
              2014 · 8.7/10
            </p>

            <p class="card__description">
              Группа исследователей отправляется искать новый дом для человечества.
            </p>

            <div class="card__actions">

              <a href="#" class="btn btn--secondary">
                Смотреть
              </a>

              <button
                type="button"
                class="card__favorite"
                aria-label="Добавить фильм в избранное"
              >
                ♥
              </button>

            </div>

          </div>

        </article>

      </div>

    </section>


    <aside class="sidebar">

      <form class="filter-form">

        <fieldset class="filter-group">

          <legend>Жанр</legend>

          <label class="filter-option">
            <input
              type="checkbox"
              name="category"
              value="Фантастика"
              checked
            />
            Фантастика
          </label>

          <label class="filter-option">
            <input
              type="checkbox"
              name="category"
              value="Драма"
              checked
            />
            Драма
          </label>

        </fieldset>


        <button
          type="button"
          class="btn btn--primary btn--full"
        >
          Применить
        </button>

      </form>

    </aside>

  </main>


  <section class="form-section">

    <div class="form-section__content">

      <p class="section-header__eyebrow">
        Дополнительный пример
      </p>

      <h2>Добавить запись</h2>

      <p class="form-section__description">
        Эта форма используется в примерах работы с полями,
        событиями и отправкой формы.
      </p>


      <form
        class="post-form"
        action="/posts/create"
        method="POST"
      >

        <div class="field-group">

          <label for="post-title">
            Заголовок
          </label>

          <input
            type="text"
            id="post-title"
            name="title"
            class="field"
            placeholder="Название поста"
            required
            minlength="3"
          />

        </div>


        <div class="field-group">

          <label for="post-content">
            Текст поста
          </label>

          <textarea
            id="post-content"
            name="content"
            class="field field--textarea"
            placeholder="О чём расскажете?"
            rows="6"
            required
          ></textarea>

        </div>


        <div class="field-group">

          <label for="post-author">
            ID автора
          </label>

          <input
            type="number"
            id="post-author"
            name="author_id"
            class="field"
            placeholder="Например, 1"
            min="1"
            required
          />

        </div>


        <button
          type="submit"
          class="btn btn--primary"
        >
          Опубликовать
        </button>

      </form>

    </div>

  </section>


  <footer class="footer">

    <div>
      🎬 MovieGrid
    </div>

    <div>
      © 2026 MovieGrid
    </div>

  </footer>


  <script src="scripts.js"></script>

</body>
</html>
```

### css-стилизация:

```css
* {
  box-sizing: border-box;
}

body {
  margin: 0;
  font-family: Arial, sans-serif;
  background: #f3f4f6;
  color: #1f2937;
}

a {
  color: inherit;
}

button,
input,
textarea {
  font: inherit;
}


/* =========================
   Layout
   ========================= */

.page {
  min-height: 100vh;
}

.header {
  display: flex;
  align-items: center;
  gap: 24px;
  padding: 16px 32px;
  background: #1f2937;
  color: white;
}

.header__logo {
  font-size: 20px;
  font-weight: 700;
  text-decoration: none;
  white-space: nowrap;
}

.header__nav {
  display: flex;
  gap: 16px;
  margin-left: auto;
}

.header__nav a {
  text-decoration: none;
  color: #d1d5db;
}

.header__nav a:hover {
  color: white;
}


/* =========================
   Search
   ========================= */

.search {
  display: flex;
  flex: 1;
  max-width: 480px;
  margin: 0 auto;
}

.search__input {
  min-width: 0;
  flex: 1;
  padding: 10px 14px;
  border: 1px solid #d1d5db;
  border-radius: 8px 0 0 8px;
  outline: none;
}

.search__input:focus-visible {
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.15);
}


/* =========================
   Main content
   ========================= */

.main {
  display: grid;
  grid-template-columns: 1fr 240px;
  gap: 24px;
  max-width: 1200px;
  margin: 0 auto;
  padding: 32px 24px;
}

.movies-section {
  min-width: 0;
}

.section-header {
  display: flex;
  align-items: end;
  justify-content: space-between;
  gap: 20px;
  margin-bottom: 20px;
}

.section-header h1,
.section-header h2 {
  margin: 4px 0 0;
}

.section-header__eyebrow {
  margin: 0;
  font-size: 13px;
  color: #6b7280;
  text-transform: uppercase;
  letter-spacing: 0.08em;
}

.section-header__description {
  margin: 0;
  color: #6b7280;
}


/* =========================
   Cards
   ========================= */

.cards {
  display: grid;
  grid-template-columns: repeat(
    auto-fill,
    minmax(220px, 1fr)
  );
  gap: 20px;
}

.card {
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background: white;
  border-radius: 12px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.08);
}

.card__cover {
  display: block;
  width: 100%;
  height: 150px;
  object-fit: cover;
}

.card__body {
  display: flex;
  flex-direction: column;
  flex: 1;
  padding: 16px;
}

.card__title {
  margin: 0 0 6px;
  font-size: 19px;
}

.card__meta {
  margin: 0 0 10px;
  color: #6b7280;
  font-size: 14px;
}

.card__description {
  margin: 0 0 16px;
  color: #4b5563;
  font-size: 14px;
  line-height: 1.5;
}

.card__actions {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-top: auto;
}

.card__favorite {
  padding: 9px 12px;
  border: none;
  border-radius: 6px;
  background: #f3f4f6;
  color: #6b7280;
  cursor: pointer;
}

.card__favorite:hover {
  background: #e5e7eb;
}

.card__favorite.is-active {
  color: #dc2626;
  background: #fee2e2;
}


/* =========================
   Buttons
   ========================= */

.btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 10px 16px;
  border: none;
  border-radius: 7px;
  font-weight: 600;
  text-decoration: none;
  cursor: pointer;
}

.btn--primary {
  background: #2563eb;
  color: white;
}

.btn--primary:hover {
  background: #1d4ed8;
}

.btn--secondary {
  background: #e5e7eb;
  color: #374151;
}

.btn--secondary:hover {
  background: #d1d5db;
}

.btn--full {
  width: 100%;
}


/* =========================
   Filters
   ========================= */

.sidebar {
  padding-top: 52px;
}

.filter-form {
  padding: 18px;
  background: white;
  border-radius: 10px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.06);
}

.filter-group {
  margin: 0 0 18px;
  padding: 0;
  border: none;
}

.filter-group legend {
  margin-bottom: 12px;
  font-weight: 700;
}

.filter-option {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 10px;
  cursor: pointer;
}


/* =========================
   Post form
   ========================= */

.form-section {
  padding: 48px 24px;
  background: white;
  border-top: 1px solid #e5e7eb;
}

.form-section__content {
  max-width: 700px;
  margin: 0 auto;
}

.form-section h2 {
  margin: 6px 0;
}

.form-section__description {
  margin: 0 0 24px;
  color: #6b7280;
}

.post-form {
  display: flex;
  flex-direction: column;
  gap: 18px;
  max-width: 480px;
  padding: 24px;
  background: #f9fafb;
  border-radius: 12px;
}

.field-group {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.field-group label {
  font-size: 14px;
  font-weight: 600;
  color: #374151;
}

.field {
  width: 100%;
  padding: 10px 14px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  outline: none;
  background: white;
}

.field::placeholder {
  color: #9ca3af;
}

.field:focus-visible {
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.15);
}

.field:invalid:not(:placeholder-shown) {
  border-color: #dc2626;
}

.field--textarea {
  min-height: 120px;
  resize: vertical;
}


/* =========================
   Footer
   ========================= */

.footer {
  display: flex;
  justify-content: space-between;
  gap: 20px;
  padding: 20px 32px;
  background: #1f2937;
  color: #d1d5db;
}


/* =========================
   Accessibility
   ========================= */

.visually-hidden {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}


/* =========================
   Responsive
   ========================= */

@media (max-width: 800px) {

  .header {
    flex-wrap: wrap;
  }

  .search {
    order: 3;
    flex-basis: 100%;
    max-width: none;
  }

  .main {
    grid-template-columns: 1fr;
  }

  .sidebar {
    padding-top: 0;
  }

}

@media (max-width: 560px) {

  .header {
    padding: 16px;
  }

  .header__nav {
    display: none;
  }

  .main {
    padding: 24px 16px;
  }

  .section-header {
    display: block;
  }

  .section-header__description {
    margin-top: 8px;
  }

  .footer {
    flex-direction: column;
    padding: 20px 16px;
  }

}
```

</details>

---

**Два ключевых понятия сегодняшнего урока**:

- **DOM** (Document Object Model) — представление HTML-страницы в виде объектов, с которыми может работать JavaScript.
- **BOM** (Browser Object Model) — объекты браузера, не относящиеся напрямую к содержимому страницы: окно, адресная строка, локальное хранилище.

---

## 1. Что такое DOM

Когда браузер загружает HTML, он не просто отображает текст — он строит в памяти **дерево объектов**, где каждый тег становится отдельным объектом-узлом (node) с собственными свойствами и методами. Это дерево и называется DOM. JavaScript работает не с "текстом HTML", а именно с этим деревом объектов — а браузер автоматически перерисовывает страницу, как только дерево меняется.

```html
<div id="app">
  <h1>Заголовок</h1>
  <p>Текст</p>
</div>
```

Браузер строит из этого примерно такую структуру (упрощённо):

```
div#app
 ├── h1 "Заголовок"
 └── p "Текст"
```

`document` — глобальный объект, представляющий корень этого дерева, точку входа для любой работы с DOM.

---

## 2. Выбор элементов: `querySelector` и `querySelectorAll`

### `document.querySelector(selector)` — найти первый подходящий элемент

```javascript
const heading = document.querySelector('h1');
const firstCard = document.querySelector('.card');
const searchInput = document.querySelector('#site-search');
```

Ключевая приятная новость: селектор внутри `querySelector` — это **обычный CSS-селектор**, тот же самый синтаксис, которым вы пользовались весь HTML/CSS-блок курса. Если вы умеете написать `.card`, `#site-search`, `.filter-option input[type="checkbox"]` в CSS-файле — вы уже умеете находить эти элементы через `querySelector`.

```javascript
const firstFilterCheckbox = document.querySelector('.filter-option input[type="checkbox"]');
const primaryButton = document.querySelector('.btn--primary');
```

### `document.querySelectorAll(selector)` — найти все подходящие элементы

```javascript
const allCards = document.querySelectorAll('.card');
console.log(allCards.length); // например, 6 — количество найденных элементов

allCards.forEach((card) => {
  console.log(card);
});
```

`querySelectorAll` возвращает не обычный массив, а **NodeList** — специальную коллекцию, похожую на массив (есть `.length`, работает `forEach`), но не являющуюся полноценным массивом (`map`/`filter`/`find` на ней напрямую не работают). Если нужны полноценные методы массива — преобразуйте NodeList в массив:

```javascript
const cardsArray = Array.from(allCards);
// или короче, через spread-оператор (Урок 18):
const cardsArray2 = [...allCards];

const cardTitles = cardsArray.map((card) => card.querySelector('.card__title').textContent);
```

### Устаревшие методы — знать, но не использовать

```javascript
document.getElementById('site-search');           // работает только по id, без гибкости CSS-селекторов
document.getElementsByClassName('card');          // возвращает "живую" HTMLCollection, ведёт себя ещё менее предсказуемо, чем NodeList
```

Эти методы старше `querySelector`/`querySelectorAll` и до сих пор встречаются в старом коде — но в новом коде мы используем именно `querySelector`/`querySelectorAll`, потому что они принимают единый, уже знакомый нам CSS-синтаксис для любого селектора, а не отдельный метод под каждый тип селектора.

### `null`, если элемент не найден — частая ловушка

```javascript
const missing = document.querySelector('.does-not-exist');
console.log(missing); // null

console.log(missing.textContent); // TypeError: Cannot read properties of null
```

Если селектор не находит ни одного элемента, `querySelector` возвращает `null`, а не ошибку — но попытка сразу же обратиться к свойству результата (`.textContent`, `.style`, что угодно) вызовет ошибку, потому что у `null` нет никаких свойств. Возвращаемся к этому в разделе "Подводные камни".

---

## 3. Чтение и изменение содержимого

### `.textContent` — простой текст

```javascript
const heading = document.querySelector('h1');

console.log(heading.textContent); // текущий текст заголовка

heading.textContent = 'Новый заголовок'; // заменяет текст полностью
```

### `.innerHTML` — содержимое как HTML-разметка

```javascript
const card = document.querySelector('.card__title');
card.innerHTML = '<strong>Матрица</strong>'; // вставит именно как HTML — жирный текст отобразится
```

**Важное предупреждение по безопасности:** `.innerHTML` интерпретирует переданную строку как настоящий HTML — если в эту строку попадёт текст, введённый пользователем (например, из поля комментария), и в нём случайно (или намеренно, со злым умыслом) окажется `<script>...</script>` — этот код может выполниться на странице. Это называется XSS-атака (Cross-Site Scripting) и является серьёзной уязвимостью. **Правило этого курса:** используйте `.textContent`, если нужно вставить именно текст (в том числе — любые данные, пришедшие от пользователя или с сервера), и прибегайте к `.innerHTML` только тогда, когда вам действительно нужно вставить заведомо безопасную, контролируемую вами разметку — например, собственный шаблон карточки, а не "сырой" пользовательский ввод.

### `.value` — значение полей форм

```javascript
const searchInput = document.querySelector('#site-search');

console.log(searchInput.value); // текущее значение, введённое пользователем
searchInput.value = 'матрица';  // программно устанавливаем значение поля
```

`.textContent`/`.innerHTML` не работают для получения значения `input`/`textarea` — у полей ввода для этого есть отдельное свойство `.value`.

---

## 4. Атрибуты и классы

### `getAttribute` / `setAttribute`

```javascript
const image = document.querySelector('.card__cover');

console.log(image.getAttribute('src')); // текущий путь к изображению
image.setAttribute('src', 'img/new-poster.jpg');
image.setAttribute('alt', 'Новый постер фильма');
```

### `classList` — самый практичный инструмент для интерактивности

```javascript
const favoriteButton = document.querySelector('.card__favorite');

favoriteButton.classList.add('is-active');       // добавить класс
favoriteButton.classList.remove('is-active');     // убрать класс
favoriteButton.classList.toggle('is-active');     // добавить, если его нет; убрать, если есть
console.log(favoriteButton.classList.contains('is-active')); // true/false — проверка наличия класса
```

`classList.toggle(...)` — самый частый инструмент для переключения состояний вроде "избранное включено/выключено", "меню открыто/закрыто", "фильтр активен/неактивен": один вызов вместо ручной проверки `if (contains) remove else add`.

```css
/* В CSS заранее готовим стиль для "включённого" состояния */
.card__favorite.is-active {
  color: #dc2626;
  background: #fee2e2;
}
```

JavaScript здесь только переключает CSS-класс — вся визуальная логика (как именно должна выглядеть активная кнопка) остаётся в CSS, где ей и место. Это хорошая практика разделения ответственности: JS решает **когда** класс должен быть активен, CSS решает **как** это должно выглядеть.

---

## 5. Создание и удаление элементов

### `document.createElement` + `append`

```javascript
const card = document.createElement('article');
card.classList.add('card');
card.textContent = 'Новая карточка';

const container = document.querySelector('.cards');
container.append(card); // добавляет card последним элементом внутрь .cards
```

`container.prepend(card)` — добавит элемент первым, а не последним. Старый способ — `container.appendChild(card)` — тоже работает и часто встречается в старом коде, но `.append()` современнее и немного гибче (может принимать сразу несколько элементов или строк текста через запятую).

### Удаление элемента

```javascript
const card = document.querySelector('.card');
card.remove(); // удаляет элемент из DOM полностью
```

### Практика: рендер карточки фильма из объекта данных

Здесь мы напрямую соединяем массив объектов (Урок 18) с реальным DOM:

```javascript
const movies = [
  { id: 1, title: 'Матрица', year: 1999, rating: 8.7 },
  { id: 2, title: 'Начало', year: 2010, rating: 8.8 },
  { id: 3, title: 'Дюна', year: 2021, rating: 8.0 },
];

const cardsContainer = document.querySelector('.cards');

const renderMovieCard = (movie) => {
  const card = document.createElement('article');
  card.classList.add('card');
  card.dataset.movieId = movie.id; // dataset — удобный способ хранить произвольные данные прямо на элементе

  const title = document.createElement('h3');
  title.classList.add('card__title');
  title.textContent = movie.title; // textContent — безопасно, данные не интерпретируются как HTML

  const meta = document.createElement('p');
  meta.classList.add('card__meta');
  meta.textContent = `${movie.year} · ${movie.rating}/10`;

  card.append(title, meta); // append умеет принимать сразу несколько элементов
  return card;
};

movies.forEach((movie) => {
  const card = renderMovieCard(movie);
  cardsContainer.append(card);
});
```

`element.dataset.movieId = movie.id` записывает значение в HTML-атрибут `data-movie-id` — специальный вид атрибута, зарезервированный именно для хранения собственных, "пользовательских" данных на элементе, не относящихся к стандартным HTML-атрибутам. Это удобно, когда позже нужно узнать, какому именно объекту данных соответствует конкретный DOM-элемент — например, при клике по карточке.

---

## 6. События — реакция на действия пользователя

### `addEventListener` — базовый синтаксис

```javascript
const button = document.querySelector('.card__favorite');

button.addEventListener('click', () => {
  console.log('Кнопка нажата!');
});
```

`addEventListener(тип_события, функция)` — регистрирует функцию, которая выполнится, когда указанное событие произойдёт на этом элементе.

Частые типы событий: 
- `click`, 
- `submit` (отправка формы), 
- `input` (изменение значения поля в реальном времени), 
- `keydown`/`keyup` (нажатие клавиш), 
- `change` (изменение значения после потери фокуса или для чекбоксов/select).

### Объект события (`event`) и `preventDefault()`

```javascript
const searchForm = document.querySelector('.search');

searchForm.addEventListener('submit', (event) => {
  event.preventDefault(); // отменяет стандартное поведение браузера — в данном случае, отправку формы и перезагрузку
  console.log('Форма перехвачена, страница не перезагрузится');
});
```

Это ровно тот момент, который мы анонсировали ещё в Уроке 15: браузер по умолчанию при отправке формы перезагружает страницу и переходит по `action`. `event.preventDefault()`, вызванный внутри обработчика `submit`, **отменяет** это поведение по умолчанию — теперь мы полностью контролируем, что происходит с данными формы дальше, средствами JavaScript.

```javascript
searchForm.addEventListener('submit', (event) => {
  event.preventDefault();

  const input = searchForm.querySelector('input[name="q"]');
  console.log('Пользователь ищет:', input.value);
  // В следующем уроке именно здесь появится fetch — отправка запроса без перезагрузки
});
```

### `this` внутри обработчика события — регулярная функция vs стрелочная

```javascript
const button = document.querySelector('.card__favorite');

button.addEventListener('click', function () {
  console.log(this); // this — сам элемент button, на котором произошло событие
  this.classList.toggle('is-active');
});
```

```javascript
button.addEventListener('click', () => {
  console.log(this); // this — НЕ сам button! Стрелочная функция берёт this из окружающего кода (Урок 19)
});
```

Это важный практический нюанс, прямое продолжение темы `this` из Урока 19: обычная `function`, переданная в `addEventListener`, получает `this`, равный тому элементу, на котором произошло событие — что часто удобно (не нужно отдельно искать элемент через `querySelector`, он уже доступен как `this`). Стрелочная функция такого поведения не даёт — `this` внутри неё "смотрит" на окружающий код, а не на элемент события. 

**Практический вывод:** если внутри обработчика события нужен доступ именно к элементу, на котором произошло событие, через `this` — используйте обычную `function`. Если такой доступ не нужен (или элемент уже получен другим способом, например, через переменную снаружи) — стрелочная функция ничем не хуже и часто компактнее.

### `input` — событие для живой реакции на ввод

```javascript
const searchInput = document.querySelector('#site-search');

searchInput.addEventListener('input', () => {
  console.log('Текущее значение поля:', searchInput.value);
});
```

`input` срабатывает при **каждом** изменении значения поля — полезно для мгновенной фильтрации списка по мере набора текста, без ожидания отправки формы.

---

## 7. BOM: `window`, `location`, `localStorage`

### `window` — глобальный объект браузера

```javascript
console.log(window.innerWidth);  // текущая ширина окна браузера в пикселях
console.log(window.innerHeight); // текущая высота окна браузера в пикселях
```

`window` — это тот самый глобальный объект, в котором технически "живут" все глобальные переменные и функции, которые мы писали на протяжении всего модуля (`console`, `document`, `setTimeout`, даже сам `window` — всё это его свойства). На практике напрямую с `window` работают не так часто, но полезно знать, что это корень всего окружения браузера.

### `location` — адрес текущей страницы

```javascript
console.log(location.href);   // полный текущий адрес страницы
console.log(location.search); // строка запроса, например "?q=матрица&category_films=1"
console.log(location.pathname); // "/films" — путь без домена и параметров
```

Здесь мы возвращаемся к теме Урока 15 — форма с `method="GET"` кладёт данные в `location.search` в виде query string. Теперь мы можем **прочитать** эти данные при загрузке страницы:

```javascript
const params = new URLSearchParams(location.search);

console.log(params.get('q'));               // "матрица" — если в URL было ?q=матрица
console.log(params.get('category_films'));  // "1" или null, если параметра не было в URL
```

`URLSearchParams` — встроенный инструмент для разбора query string на отдельные параметры, без необходимости вручную парсить строку по `&` и `=`. Практический сценарий: пользователь перешёл по ссылке `/films?q=матрица` (например, из закладки браузера, куда сохранил результат поиска ещё в Уроке 15) — при загрузке страницы мы можем прочитать `q` из `location.search` и автоматически подставить это значение обратно в поле поиска:

```javascript
const params = new URLSearchParams(location.search);
const searchQuery = params.get('q');

if (searchQuery) {
  document.querySelector('#site-search').value = searchQuery;
}
```

### `localStorage` — постоянное хранилище в браузере

```javascript
localStorage.setItem('username', 'Мария'); // сохранить значение
console.log(localStorage.getItem('username')); // 'Мария'
localStorage.removeItem('username'); // удалить значение
```

`localStorage` сохраняет данные **между перезагрузками страницы и даже между сессиями браузера** — в отличие от обычных переменных JavaScript, которые исчезают при каждой перезагрузке. Ключевое ограничение: `localStorage` умеет хранить **только строки**. Если нужно сохранить массив или объект, их нужно предварительно превратить в строку через `JSON.stringify`, а при чтении — превратить обратно через `JSON.parse`:

```javascript
const favorites = [1, 5, 12];

localStorage.setItem('favorites', JSON.stringify(favorites));
// в localStorage реально сохранится строка: "[1,5,12]"

const storedFavorites = JSON.parse(localStorage.getItem('favorites'));
console.log(storedFavorites); // [1, 5, 12] — снова настоящий массив, а не строка
```

**Важная деталь:** если ключа ещё не существует в `localStorage`, `getItem` вернёт `null`, а не пустую строку или пустой массив — `JSON.parse(null)` вернёт `null`, поэтому перед первым использованием стоит проверять этот случай явно (пример — в практике ниже).

---

## 8. Итоговая практика: оживляем страницу фильмов

Соберём три ключевых сценария воедино на реальной разметке из Уроков 11–15 — рендер карточек, переключение избранного с сохранением в `localStorage`, и живая фильтрация по чекбоксам.

### HTML (фрагмент, напоминание структуры из прошлых уроков)

```html
<div class="cards"></div>

<form class="filter-form">
  <fieldset class="filter-group">
    <legend>Категория</legend>
    <label class="filter-option">
      <input type="checkbox" name="category" value="Фантастика" checked />
      Фантастика
    </label>
    <label class="filter-option">
      <input type="checkbox" name="category" value="Драма" checked />
      Драма
    </label>
  </fieldset>
</form>
```

### JavaScript

```javascript
const movies = [
  { id: 1, title: 'Матрица', year: 1999, rating: 8.7, genre: 'Фантастика' },
  { id: 2, title: 'Начало', year: 2010, rating: 8.8, genre: 'Фантастика' },
  { id: 3, title: 'Крестный отец', year: 1972, rating: 9.2, genre: 'Драма' },
];

const cardsContainer = document.querySelector('.cards');
const filterForm = document.querySelector('.filter-form');

// Загружаем избранное из localStorage при старте страницы
const getFavorites = () => {
  const stored = localStorage.getItem('favorites');
  return stored ? JSON.parse(stored) : [];
};

const saveFavorites = (favorites) => {
  localStorage.setItem('favorites', JSON.stringify(favorites));
};

const toggleFavorite = (movieId) => {
  const favorites = getFavorites();
  const isFavorite = favorites.includes(movieId);

  const updated = isFavorite
    ? favorites.filter((id) => id !== movieId)
    : [...favorites, movieId];

  saveFavorites(updated);
  return !isFavorite;
};

const renderMovieCard = (movie) => {
  const favorites = getFavorites();
  const isFavorite = favorites.includes(movie.id);

  const card = document.createElement('article');
  card.classList.add('card');
  card.dataset.movieId = movie.id;
  card.dataset.genre = movie.genre;

  const title = document.createElement('h3');
  title.classList.add('card__title');
  title.textContent = movie.title;

  const meta = document.createElement('p');
  meta.textContent = `${movie.year} · ${movie.rating}/10`;

  const favoriteButton = document.createElement('button');
  favoriteButton.type = 'button';
  favoriteButton.classList.add('card__favorite');
  favoriteButton.textContent = '♥';
  favoriteButton.classList.toggle('is-active', isFavorite);

  favoriteButton.addEventListener('click', () => {
    const nowFavorite = toggleFavorite(movie.id);
    favoriteButton.classList.toggle('is-active', nowFavorite);
  });

  card.append(title, meta, favoriteButton);
  return card;
};

const renderAllCards = () => {
  cardsContainer.innerHTML = ''; // очищаем контейнер перед перерисовкой
  movies.forEach((movie) => {
    cardsContainer.append(renderMovieCard(movie));
  });
};

renderAllCards();

// Живая фильтрация по чекбоксам
filterForm.addEventListener('change', () => {
  const checkedGenres = [...filterForm.querySelectorAll('input[name="category"]:checked')].map(
    (checkbox) => checkbox.value
  );

  document.querySelectorAll('.card').forEach((card) => {
    const shouldShow = checkedGenres.includes(card.dataset.genre);
    card.style.display = shouldShow ? '' : 'none';
  });
});
```

**Разбор ключевых решений:**

- `favoriteButton.classList.toggle('is-active', isFavorite)` — у `toggle` есть необязательный второй аргумент: если передать явное `true`/`false`, метод не переключает класс, а принудительно устанавливает его в конкретное состояние — удобно, когда состояние уже известно (например, только что вычислено), а не должно именно "переключиться" от текущего.
- `cardsContainer.innerHTML = ''` — здесь мы сознательно используем `innerHTML` не для вставки пользовательских данных, а для быстрой полной очистки контейнера перед перерисовкой — это безопасный и частый случай использования `innerHTML`, отличный от вставки непроверенного текста.
- `card.style.display = shouldShow ? '' : 'none'` — прямое управление инлайн-стилем элемента через свойство `.style`. Пустая строка `''` сбрасывает инлайн-стиль `display` к тому, что задано в CSS-файле (обычно `flex` или `block` для карточки), а не жёстко прописывает конкретное значение — так карточка "возвращается" к своему обычному виду из стилей, а не получает жёстко зашитое в JS значение.

---

## Подводные камни

### Обращение к результату `querySelector`, когда элемент не найден

```javascript
const button = document.querySelector('.button-that-does-not-exist');
button.addEventListener('click', () => {}); // TypeError: Cannot read properties of null
```

Если селектор написан с опечаткой, либо код выполняется до того, как соответствующий элемент появился в HTML (например, скрипт подключён в `<head>` без `defer`, разобрано в Уроке 16) — `querySelector` молча вернёт `null`, а ошибка проявится только на следующей строке, где к этому `null` пытаются обратиться. Хорошая привычка при отладке — сначала вывести результат `querySelector` в консоль и убедиться, что это действительно нужный элемент, а не `null`.

### `.innerHTML` с непроверенными данными — риск XSS

```javascript
// ❌ Опасно, если comment.text пришёл от другого пользователя и не был проверен
card.innerHTML = comment.text;
```

Разобрано в разделе 3 — используйте `.textContent` для вставки любых данных, которые потенциально могли быть введены пользователем (в том числе — другим пользователем, чей комментарий вы показываете сейчас).

### Забытый `event.preventDefault()` в обработчике `submit`

```javascript
form.addEventListener('submit', () => {
  console.log('Отправка перехвачена'); // выведется в консоль...
  // ...но страница всё равно перезагрузится, потому что preventDefault() не вызван!
});
```

Без `event.preventDefault()` браузер выполнит и вашу функцию-обработчик, и своё стандартное поведение (перезагрузку страницы) — оба одновременно. Если цель — полностью взять управление отправкой формы на себя, `preventDefault()` обязателен первой строкой внутри обработчика.

### `NodeList` — не полноценный массив

```javascript
const checkboxes = document.querySelectorAll('input[type="checkbox"]');
const values = checkboxes.map((checkbox) => checkbox.value); // TypeError: checkboxes.map is not a function
```

Разобрано в разделе 2 — `querySelectorAll` возвращает `NodeList`, а не `Array`. `forEach` у него работает, но `map`/`filter`/`find` — нет, пока коллекция не преобразована в настоящий массив через `Array.from(...)` или spread (`[...checkboxes]`).

### Данные в `localStorage` без `JSON.stringify`/`JSON.parse`

```javascript
const favorites = [1, 2, 3];

localStorage.setItem('favorites', favorites); // ❌ сохранится строка "1,2,3" — потеряна структура массива!

const stored = localStorage.getItem('favorites');
console.log(stored); // "1,2,3" — это просто строка, не массив
console.log(stored[0]); // "1" — обращение по индексу к СТРОКЕ, а не элементу массива
```

`localStorage` хранит только строки — если передать массив или объект напрямую, JavaScript молча преобразует его в строку неявным образом (для массива — через запятую), теряя исходную структуру данных. Всегда используйте `JSON.stringify` при сохранении и `JSON.parse` при чтении, если сохраняете что-либо сложнее одиночной строки или числа.

### Динамически созданные элементы "не реагируют" на события, добавленные раньше

```javascript
document.querySelectorAll('.card__favorite').forEach((button) => {
  button.addEventListener('click', () => console.log('Клик!'));
});

// Позже добавляем новую карточку с новой кнопкой .card__favorite
cardsContainer.append(renderMovieCard(newMovie));
// Клик по кнопке НОВОЙ карточки не сработает — обработчик был назначен ДО её создания!
```

`addEventListener`, вызванный для конкретных, уже существующих на момент вызова элементов, не распространяется автоматически на элементы, которые появятся в DOM **позже**. В примере из раздела 8 мы обошли эту проблему, назначая обработчик клика прямо внутри функции `renderMovieCard`, в момент создания каждой отдельной карточки — так каждая новая карточка сразу получает собственный обработчик. Существует и более продвинутая техника — "делегирование событий" (навешивание одного обработчика на родительский контейнер вместо каждого дочернего элемента по отдельности), которая элегантно решает эту проблему для очень динамичных интерфейсов, но пока для наших задач достаточно варианта "назначать обработчик в момент создания элемента".

---

## Итоги урока

DOM — представление HTML-страницы в виде дерева объектов, с которым напрямую работает JavaScript; изменения этого дерева браузер сразу же отражает визуально. `document.querySelector`/`querySelectorAll` находят элементы по обычному CSS-селектору — том же самом синтаксисе, которым вы пользовались весь HTML/CSS-блок курса. `.textContent` безопасно вставляет текст, `.innerHTML` интерпретирует строку как разметку (и требует осторожности с непроверенными данными), `classList` — основной инструмент переключения визуальных состояний через CSS-классы.

`document.createElement` в связке с `.append()` позволяет строить DOM-элементы программно — именно так мы теперь рендерим карточки фильмов из массива объектов данных, а не прописываем их вручную в HTML. `addEventListener` подписывает функцию на событие элемента; `event.preventDefault()` внутри обработчика `submit` отменяет стандартную перезагрузку страницы при отправке формы — это финальный кусочек моста, обещанного ещё в Уроке 15.

BOM даёт доступ к окружению браузера за пределами содержимого страницы: `location` — текущий адрес и его составляющие (включая query string, который мы генерировали GET-формами в Уроке 15), `localStorage` — постоянное хранилище строковых данных между перезагрузками, требующее `JSON.stringify`/`JSON.parse` для хранения массивов и объектов.

В следующем уроке — асинхронность: `callback` → `Promise` → `async`/`await`, и, наконец, `fetch` — реальный обмен данными с нашим Blog API на FastAPI, без единой перезагрузки страницы.

---

## Вопросы для проверки

1. Что такое DOM и как он связан с HTML-разметкой, которую пишет разработчик?
2. Почему `querySelector`/`querySelectorAll` считаются более удобными, чем `getElementById`/`getElementsByClassName`?
3. В чём разница между `.textContent` и `.innerHTML`, и почему `.textContent` считается более безопасным выбором при работе с пользовательскими данными?
4. Что делает `event.preventDefault()` внутри обработчика события `submit`, и зачем это было нужно применительно к материалу Урока 15?
5. Чем `NodeList`, возвращаемый `querySelectorAll`, отличается от обычного массива? Как превратить его в полноценный массив?
6. Почему `localStorage` требует `JSON.stringify` при сохранении массива или объекта, и что произойдёт, если этого не сделать?
7. Почему обработчик клика, назначенный через `addEventListener` до создания элемента, не сработает для элемента, добавленного в DOM позже? Как это было решено в практике урока?
8. В чём разница поведения `this` внутри обработчика события, если использовать обычную `function` вместо стрелочной функции?

---

## Практические задания

### Задание 1. Список фильмов из массива — рендер и фильтрация по названию

Дан контейнер `<ul class="movie-list"></ul>` и массив:

```javascript
const movies = [
  { id: 1, title: 'Матрица' },
  { id: 2, title: 'Начало' },
  { id: 3, title: 'Интерстеллар' },
  { id: 4, title: 'Дюна' },
];
```

1. Для каждого фильма создайте `<li>` с текстом названия и добавьте в `.movie-list` через `createElement`/`append`.
2. Добавьте `<input type="text" class="movie-search" placeholder="Поиск по названию...">` над списком.
3. По событию `input` на этом поле фильтруйте видимость пунктов списка: если название содержит введённый текст (используйте `.toLowerCase()` и `.includes()` для регистронезависимого поиска) — показывайте (`style.display = ''`), иначе скрывайте (`style.display = 'none'`).

---

### Задание 2. Переключатель темы с сохранением в `localStorage`

Создайте кнопку `<button class="theme-toggle">Тёмная тема</button>` и логику переключения:

1. По клику на кнопку переключайте класс `dark-theme` у `document.body` через `classList.toggle`.
2. Сохраняйте текущее состояние (включена тёмная тема или нет) в `localStorage` под ключом `theme` (`'dark'` или `'light'`).
3. При загрузке страницы читайте значение из `localStorage` и, если сохранена тёмная тема, сразу применяйте класс `dark-theme` к `body`, не дожидаясь клика — так настройка должна "запоминаться" между перезагрузками страницы.

---

### Задание 3. Объединяющая практика — страница фильмов: рендер, избранное, фильтры (полная версия)

Возьмите HTML/CSS страницы фильмов из Уроков 11–15 (`header`, `aside` с фильтрами, `main` с `.cards`, `footer`) и добавьте `script.js`, реализующий:

1. Массив из 5–6 объектов фильмов (`id`, `title`, `year`, `rating`, `genre`) вместо статичной разметки карточек в HTML.
2. Функцию рендера карточек из массива в `.cards`, аналогичную разделу 8 этого урока, включая кнопку "избранное" с сохранением состояния в `localStorage`.
3. Живую фильтрацию по чекбоксам категорий из `aside` — используя событие `change` на форме фильтров (как в разделе 8).
4. Перехват отправки формы поиска (`event.preventDefault()`) — выведите введённое значение в консоль вместо перезагрузки страницы (полноценную отправку на сервер сделаем в следующем уроке через `fetch`).
5. При загрузке страницы прочитайте `location.search` через `URLSearchParams` и, если параметр `q` присутствует в адресе, автоматически подставьте его значение в поле поиска.

---

[Предыдущий урок](lesson20.md) | [Следующий урок](lesson22.md)