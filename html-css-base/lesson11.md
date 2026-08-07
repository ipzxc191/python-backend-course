# Урок 11. CSS Grid

## Что такое Grid и зачем он нужен

**CSS Grid** — система для построения двухмерных макетов: она управляет рядами и колонками одновременно. Grid даёт удобные и декларативные инструменты для создания страниц, панелей, карточных сеток и сложных шаблонов, где нужно явно управлять и колонками, и строками сразу.

### Grid vs Flexbox

**Flexbox** (Урок 10) идеален для одномерной компоновки — одна строка *или* одна колонка; отлично подходит для выравнивания компонентов внутри строки или колонки (навигация, группа кнопок, внутренняя компоновка карточки). **Grid** предназначен для двухмерной раскладки — вы задаёте колонки *и* ряды одновременно и располагаете элементы по сетке; удобен для глобальной структуры страницы (макет страницы, дашборд, галерея карточек).

**Практическое правило:** используйте Grid для общей структуры (layout), а Flexbox — внутри отдельных компонентов (выравнивание элементов внутри карточки, шапки, кнопки).

---

## Базовая терминология

**Grid container** — элемент с `display: grid` (контейнер-сетка). 

**Grid item** — прямые дочерние элементы grid-контейнера. 

**Track** — одна колонка или одна строка сетки (column track / row track). 

**Grid line** — линии сетки, между которыми располагаются элементы (нумеруются, начиная с 1). 

**Grid cell** — пересечение одной колонки и одной строки (ячейка). 

**Grid area** — объединённые соседние ячейки, которым удобно давать собственное имя. 

**Explicit grid** — строки/колонки, явно заданные через `grid-template-*`. 

**Implicit grid** — строки/колонки, создаваемые браузером автоматически, если элементов больше, чем было явно задано. 

**Fraction (`fr`)** — специальная единица Grid для распределения долей свободного пространства.

---

## Простейший рабочий пример

```html
<div class="grid">
  <div class="card">Элемент 1</div>
  <div class="card">Элемент 2</div>
  <div class="card">Элемент 3</div>
  <div class="card">Элемент 4</div>
  <div class="card">Элемент 5</div>
  <div class="card">Элемент 6</div>
</div>
```

```css
.grid {
  display: grid; /* делаем контейнер grid */
  grid-template-columns: 1fr 1fr 1fr; /* 3 колонки одинаковой ширины */
  gap: 16px; /* расстояние между ячейками */
}

.card {
  background: white;
  padding: 16px;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
}
```

Шесть элементов автоматически распределятся по три в ряд — сколько бы карточек ни было, Grid сам создаёт нужное число строк (implicit grid), пока явно заданы только колонки.

---

## `grid-template-columns` / `grid-template-rows` — задаём явную сетку

```html
<div class="grid">
  <div class="card">1</div>
  <div class="card">2</div>
  <div class="card">3</div>
  <div class="card">4</div>
  <div class="card">5</div>
  <div class="card">6</div>
</div>
```

### **Фиксированные размеры:**

```css
.grid {
  display: grid;
  grid-template-columns: 200px 200px 200px;
  gap: 16px;
}

.card {
  background: steelblue;
  color: white;
  padding: 30px;
  text-align: center;
}
```

Используется, когда нужна точная ширина (например, боковая панель с фиксированным контентом).

### **Проценты и `auto`:**

```css
.grid {
  display: grid;
  grid-template-columns: 20% auto 30%;
  gap: 16px;
}

.card {
  background: steelblue;
  color: white;
  padding: 30px;
  text-align: center;
}
```

Проценты задают долю от контейнера, `auto` подстраивается под содержимое. Первый столбец всегда занимает 20% ширины контейнера, последний — 30%, а средний автоматически получает всё оставшееся пространство.

### **`fr` — доли свободного пространства:**

```css
.grid {
  display: grid;
  grid-template-columns: 1fr 2fr 1fr;
  gap: 16px;
}

.card {
  background: steelblue;
  color: white;
  padding: 30px;
  text-align: center;
}
```

Вторая колонка занимает в два раза больше места, чем первая и третья.

Гораздо удобнее и предсказуемее процентов при построении гибких сеток.

### **`repeat()` — сокращённая запись:**

```css
.grid {
  display: grid;

  grid-template-columns: repeat(3, 1fr);
  grid-template-columns: 1fr 1fr 1fr;

  gap: 16px;
}
```

### **Комбинация фиксированного и гибкого:**

```css
.grid {
  display: grid;
  grid-template-columns: 220px 1fr 1fr;
  gap: 16px;
}

.card {
  background: steelblue;
  color: white;
  padding: 30px;
  text-align: center;
}
```

### **Строки (`grid-template-rows`):**

Немного изменим HTML:

```html
<div class="layout">
  <header>Шапка</header>
  <main>Основной контент</main>
  <footer>Подвал</footer>
</div>
```

```css
.layout {
  display: grid;

  grid-template-rows: 80px auto 60px;

  height: 350px;
  gap: 10px;
}

header,
main,
footer {
  display: grid;
  place-items: center;

  color: white;
}

header {
  background: steelblue;
}

main {
  background: tomato;
}

footer {
  background: seagreen;
}
```

---

## Единицы измерения в Grid

| Единица | Что делает | Когда использовать |
|---|---|---|
| `px` | Фиксированная величина | Sidebar, колонки с точным контентом |
| `%` | Процент от контейнера | Редко нужен внутри Grid — `fr` обычно лучше читается |
| `fr` | Доля свободного пространства после вычета фиксированных треков | Гибкие сетки — основной инструмент Grid |
| `minmax(min, max)` | Диапазон: трек не меньше `min` и не больше `max` | Адаптивные сетки, где важен минимальный размер |
| `auto` | Размер по содержимому | Колонки с короткими метками, кнопками |

Практический пример комбинации нескольких единиц:

```html
<div class="grid">
  <div class="card">Первая колонка</div>
  <div class="card">Вторая колонка</div>
</div>
```

```css
.grid {
  display: grid;

  grid-template-columns:
    minmax(150px, 1fr)
    minmax(150px, 2fr);

  gap: 16px;
}

.card {
  background: steelblue;
  color: white;
  padding: 30px;
  text-align: center;
}
```

Обе колонки не станут уже 150px, но при этом вторая будет расти вдвое быстрее первой, если места достаточно.

---

## `repeat()` с `auto-fill`/`auto-fit` — адаптивные сетки без медиа-запросов

`repeat(n, value)` — сокращённая запись вместо повторения значения `n` раз. `repeat(auto-fill, minmax(...))` и `repeat(auto-fit, minmax(...))` — мощный приём для адаптивных сеток карточек: браузер сам вычисляет, сколько колонок заданной минимальной ширины поместится в текущую ширину контейнера, без единого медиа-запроса.

```html
<div class="grid">
  <div class="card">1</div>
  <div class="card">2</div>
  <div class="card">3</div>
  <div class="card">4</div>
  <div class="card">5</div>
  <div class="card">6</div>
  <div class="card">7</div>
  <div class="card">8</div>
</div>
```

```css
.grid {
  display: grid;

  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));

  gap: 16px;
}

.card {
  background: steelblue;
  color: white;
  padding: 30px;
  text-align: center;
}
```

Карточки автоматически перестраиваются в нужное число колонок при изменении ширины окна — минимум 180px на карточку, а всё оставшееся пространство распределяется поровну между уместившимися колонками.

`auto-fill` и `auto-fit` очень похожи; разница проявляется, когда в ряду остаётся свободное место при небольшом количестве элементов — `auto-fit` в этом случае «растягивает» уже имеющиеся колонки, заполняя пустоту, тогда как `auto-fill` может оставить место под невидимые пустые колонки. На практике для большинства карточных сеток разница малозаметна, и оба варианта можно пробовать взаимозаменяемо.

---

## Размещение элементов: `grid-column`, `grid-row`, `span`

`grid-column: start / end` задаёт, между какими линиями сетки должен располагаться элемент (линии нумеруются от 1). `grid-column: span N` — элемент занимает `N` колонок начиная с текущей позиции.

```html
<div class="grid">
  <div class="card card-a">A</div>
  <div class="card card-b">B</div>
  <div class="card card-c">C</div>
  <div class="card">D</div>
  <div class="card">E</div>
  <div class="card">F</div>
</div>
```

```css
.grid {
  display: grid;

  grid-template-columns: repeat(4, 1fr);

  gap: 16px;
}

.card {
  background: steelblue;
  color: white;
  padding: 30px;
  text-align: center;
}

.card-a {
  grid-column: 1 / 3;
}

.card-b {
  grid-column: 3 / span 2;
}

.card-c {
  grid-column: span 2;

  background: tomato;
}
```

То же самое работает для строк через `grid-row`. Этот механизм особенно полезен, когда один элемент (например, крупная рекламная карточка или выделенная новость) должен занимать больше места, чем остальные, в общей равномерной сетке.

---

## Отступы и выравнивания в Grid

### `gap`, `row-gap`, `column-gap`

`gap` задаёт расстояние между строками и колонками одновременно; `row-gap`/`column-gap` — по отдельности для каждого направления. `gap` предпочтительнее `margin` между элементами — не требует компенсации отступов у крайних элементов ряда и корректно работает прямо на уровне системы размещения.

```html
<div class="grid">
  <div class="card">1</div>
  <div class="card">2</div>
  <div class="card">3</div>
  <div class="card">4</div>
  <div class="card">5</div>
  <div class="card">6</div>
</div>
```

```css
.grid {
  display: grid;

  grid-template-columns: repeat(3, 1fr);

  gap: 20px;

  row-gap: 30px;
  column-gap: 10px;
}

.card {
  background: steelblue;
  color: white;
  padding: 30px;
  text-align: center;
}
```

### `align-items`/`justify-items` vs `align-content`/`justify-content`

Это разделение — источник частой путаницы, поэтому стоит зафиксировать чётко: 
- `align-items`/`justify-items` управляют тем, как содержимое располагается **внутри каждой отдельной ячейки** (по вертикали и горизонтали соответственно). 
- `align-content`/`justify-content` управляют тем, как **вся сетка целиком** располагается внутри контейнера, если сетка меньше контейнера и остаётся свободное пространство.

#### `align-items` / `justify-items`:

```html
<div class="grid">
  <div class="card">A</div>
  <div class="card">B</div>
  <div class="card">C</div>
</div>
```

```css
.grid {
  display: grid;

  grid-template-columns: repeat(3, 1fr);
  grid-auto-rows: 140px;

  align-items: center;
  justify-items: center;

  gap: 16px;

  border: 2px solid #333;
}

.card {
  width: 70px;
  height: 70px;

  display: grid;
  place-items: center;

  background: steelblue;
  color: white;
}
```

`place-items` — сокращённая запись для `align-items` + `justify-items` одновременно;

---

#### `align-content` / `justify-content`:

```html
<div class="wrapper">
  <div class="grid">
    <div class="card">1</div>
    <div class="card">2</div>
    <div class="card">3</div>
    <div class="card">4</div>
  </div>
</div>
```

```css
.wrapper {
  height: 450px;

  display: grid;

  border: 2px solid #333;
}

.grid {
  display: grid;

  grid-template-columns: repeat(2, 100px);
  grid-auto-rows: 100px;

  justify-content: center;
  align-content: center;

  gap: 16px;
}

.card {
  display: grid;
  place-items: center;

  background: steelblue;
  color: white;
}
```

`place-content` — аналогично для `align-content` + `justify-content`.

---

## `grid-template-areas` — именованные области сетки

`grid-template-areas` позволяет задать «карту» сетки текстовым шаблоном, где каждый участок имеет собственное имя — это делает раскладку страницы наглядной и легко читаемой прямо в CSS.

```html
<div class="layout">
  <header>HEADER</header>
  <aside>SIDEBAR</aside>
  <main>MAIN CONTENT</main>
  <footer>FOOTER</footer>
</div>
```

```css
.layout {
  display: grid;

  grid-template-columns: 220px 1fr;
  grid-template-rows: 80px 1fr 60px;

  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";

  gap: 10px;
  height: 400px;
}

header,
aside,
main,
footer {
  display: grid;
  place-items: center;

  color: white;
}

header {
  grid-area: header;
  background: steelblue;
}

aside {
  grid-area: sidebar;
  background: tomato;
}

main {
  grid-area: main;
  background: seagreen;
}

footer {
  grid-area: footer;
  background: #444;
}
```

`header` занимает всю верхнюю строку (потому что имя `header` повторяется дважды подряд в первой строке шаблона), `sidebar` — слева от `main` во второй строке, `footer` растягивается на всю нижнюю строку. Каждый элемент получает своё имя области через `grid-area`, и именно по совпадению этих имён браузер расставляет их по сетке.

Правила именования: имя области может быть любым словом (`main`, `sidebar`); пустая ячейка обозначается точкой (`.`); все строки шаблона должны содержать одинаковое количество «ячеечных имён» — иначе браузер не поймёт, как правильно построить сетку.

```css
grid-template-areas:
  "header header"
  "sidebar ."
  "footer footer";
```

Здесь во второй строке правая ячейка намеренно оставлена пустой.

`grid-template-areas` особенно ценен тем, что делает раскладку **независимой от порядка элементов в HTML** — блоки могут идти в разметке в любом порядке, но окажутся на странице ровно там, где предписывает текстовая карта сетки.

---

## Вопросы для проверки

1. В чём принципиальная разница между Flexbox и Grid, и как выбрать, что использовать в конкретной ситуации?
2. Что такое grid-контейнер и grid-элемент?
3. Чем единица `fr` отличается от процентов при задании ширины колонок?
4. Что делает `repeat(auto-fill, minmax(180px, 1fr))`, и зачем это нужно?
5. Что означает запись `grid-column: 2 / span 3`?
6. В чём разница между `align-items` и `align-content` в Grid?
7. Зачем нужен `grid-template-areas`, и в чём его практическое преимущество перед указанием точных номеров линий (`grid-column`/`grid-row`) для каждого элемента?
8. Что означает точка (`.`) внутри строки `grid-template-areas`?
9. Почему все строки в `grid-template-areas` должны содержать одинаковое количество «ячеечных имён»?
10. Приведи практический пример, когда стоит использовать Grid для одной части страницы, а Flexbox — для другой части внутри неё же.

---

## Практические задания

### Задание 1. Базовая сетка карточек

Создай сетку из 6 карточек с одинаковой шириной колонок через `repeat(3, 1fr)` и промежутком `gap: 16px`.

<details>
<summary><b>Готовая html разметка</b></summary>

```html
<div class="grid">
  <div class="card">Элемент 1</div>
  <div class="card">Элемент 2</div>
  <div class="card">Элемент 3</div>
  <div class="card">Элемент 4</div>
  <div class="card">Элемент 5</div>
  <div class="card">Элемент 6</div>
</div>
```

</details>

---

### Задание 2. Адаптивная сетка без медиа-запросов

Пересобери сетку из Задания 1 так, чтобы количество колонок автоматически менялось в зависимости от ширины окна — минимальная ширина карточки 200px, без единого медиа-запроса.

---

### Задание 3. Выделенный элемент через `span`

В сетке из 4 колонок сделай так, чтобы первый элемент занимал сразу 2 колонки (например, как «главная новость» на фоне остальных обычных карточек).

<details>
<summary><b>Готовая html разметка</b></summary>

```html
<div class="grid">
  <div class="card featured">Главная новость</div>
  <div class="card">Новость 2</div>
  <div class="card">Новость 3</div>
  <div class="card">Новость 4</div>
  <div class="card">Новость 5</div>
</div>
```

</details>

---

### Задание 4. Выравнивание содержимого и всей сетки

Создай сетку из 3 колонок с карточками разной высоты текста. Сначала центрируй содержимое каждой ячейки (`align-items`/`justify-items: center`). Затем, отдельно, помести всю сетку внутрь контейнера фиксированной высоты `400px` и центрируй уже саму сетку целиком по вертикали (`align-content: center`).

<details>
<summary><b>Готовая html разметка</b></summary>

```html
<div class="grid grid--items-centered">
  <div class="box"><p>Короткий</p></div>
  <div class="box"><p>Средний текст в ячейке</p></div>
  <div class="box"><p>Очень длинное описание, занимающее несколько строк текста внутри ячейки</p></div>
</div>

<div class="wrap">
  <div class="grid grid--content-centered">
    <div class="box">1</div>
    <div class="box">2</div>
    <div class="box">3</div>
  </div>
</div>
```

</details>

---

### Задание 5. Базовый layout через `grid-template-areas`

Собери структуру страницы `header`/`sidebar`/`main`/`footer` через `grid-template-areas`: шапка и футер растягиваются на всю ширину, сайдбар фиксированной ширины 200px слева, `main` занимает оставшееся пространство. Высота контейнера — не меньше высоты экрана.

<details>
<summary><b>Готовая html разметка</b></summary>

```html
<body class="page">
  <header>HEADER</header>
  <aside>SIDEBAR</aside>
  <main>MAIN CONTENT</main>
  <footer>FOOTER</footer>
</body>
```

</details>

---

### Задание 6. Страница фильмов — объединяющая практика (Grid + Flexbox)

Собери полноценную страницу каталога фильмов из четырёх секций, используя ровно ту же логику, что мы разбирали весь урок:

1. Вся страница строится через `grid-template-areas`: `header header` / `aside main` / `footer footer`.
2. `header` — логотип, название сайта «MovieGrid», навигация («Главная», «Фильмы», «Контакты») — расположены через Flexbox с `justify-content: space-between`.
3. `aside` — три пункта фильтра («Фильмы», «Сериалы», «Мультики»), выстроенные вертикально через `flex-direction: column`.
4. `main` — 6 карточек фильмов в сетке (используй `repeat(auto-fill, minmax(200px, 1fr))` или фиксированные 3 колонки); внутри каждой карточки — картинка, название, описание, ссылка «Смотреть», выстроенные вертикально через Flexbox.
5. `footer` — контакты слева, копирайт справа, через `justify-content: space-between`.

<details>
<summary><b>Готовая html разметка</b></summary>

```html
<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8" />
  <title>MovieGrid</title>
  <link rel="stylesheet" href="style.css" />
</head>
<body class="page">
  <header class="header">
    <div class="header__logo">🎬 MovieGrid</div>
    <nav class="header__nav">
      <a href="#">Главная</a>
      <a href="#">Фильмы</a>
      <a href="#">Контакты</a>
    </nav>
  </header>

  <aside class="filters">
    <div>Фильмы</div>
    <div>Сериалы</div>
    <div>Мультики</div>
  </aside>

  <main class="movies">
    <article class="movie-card">
      <img src="https://picsum.photos/300/200?1" alt="Фильм 1" />
      <h3>Название фильма 1</h3>
      <p>Краткое описание фильма для примера карточки.</p>
      <a href="#">Смотреть</a>
    </article>
    <article class="movie-card">
      <img src="https://picsum.photos/300/200?2" alt="Фильм 2" />
      <h3>Название фильма 2</h3>
      <p>Краткое описание фильма для примера карточки.</p>
      <a href="#">Смотреть</a>
    </article>
    <article class="movie-card">
      <img src="https://picsum.photos/300/200?3" alt="Фильм 3" />
      <h3>Название фильма 3</h3>
      <p>Краткое описание фильма для примера карточки.</p>
      <a href="#">Смотреть</a>
    </article>
    <article class="movie-card">
      <img src="https://picsum.photos/300/200?4" alt="Фильм 4" />
      <h3>Название фильма 4</h3>
      <p>Краткое описание фильма для примера карточки.</p>
      <a href="#">Смотреть</a>
    </article>
    <article class="movie-card">
      <img src="https://picsum.photos/300/200?5" alt="Фильм 5" />
      <h3>Название фильма 5</h3>
      <p>Краткое описание фильма для примера карточки.</p>
      <a href="#">Смотреть</a>
    </article>
    <article class="movie-card">
      <img src="https://picsum.photos/300/200?6" alt="Фильм 6" />
      <h3>Название фильма 6</h3>
      <p>Краткое описание фильма для примера карточки.</p>
      <a href="#">Смотреть</a>
    </article>
  </main>

  <footer class="footer">
    <div>Связаться с нами: info@moviegrid.ru</div>
    <div>© 2025 MovieGrid</div>
  </footer>
</body>
</html>
```

</details>

---

[Предыдущий урок](lesson10.md) | [Следующий урок](lesson12.md)