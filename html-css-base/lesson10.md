# Урок 10. Введение во Flexbox

## Зачем нужен Flexbox

**Flexbox** — инструмент для расположения элементов внутри одного контейнера по одной оси (горизонтальной или вертикальной). Его сильные стороны: простая и надёжная центровка по горизонтали и вертикали; распределение свободного пространства между элементами (равномерное заполнение, «растяжение»); удобная организация строк/колонок с переносом (`wrap`) — карточки, кнопки, элементы меню; управление порядком отображения без изменения DOM (через `order`). Flexbox заменяет множество старых `float`/`inline-block`-приёмов, которые мы разбирали в предыдущих уроках, — код становится заметно чище.

Flexbox чаще всего применяют для: горизонтальных/вертикальных меню, шапок с логотипом и кнопками, групп элементов управления, карточек товаров (картинка + текст + кнопка внутри одной карточки), центрирования контента (например, модальных окон), и вёрстки отдельных компонентов страницы — для вёрстки же **всей страницы целиком** чаще применяют Grid, с которым мы познакомимся в следующем модуле.

---

## Основные концепции и терминология

### Flex container и flex items

**Flex container** — элемент, к которому применили `display: flex` (или `inline-flex`). **Flex items** — непосредственные дочерние элементы этого контейнера, именно они выстраиваются по правилам Flexbox.

```html
<div class="container">
  <div class="item">1</div>
  <div class="item">2</div>
  <div class="item">3</div>
</div>
```

```css
.container {
  display: flex;
  border: 2px solid #333;
  padding: 10px;
  gap: 10px;
}

.item {
  width: 80px;
  height: 80px;
  background: steelblue;
  color: white;
  text-align: center;
  line-height: 80px;
}
```

### Main axis и cross axis

**Main axis** (основная ось) — направление, вдоль которого располагаются flex-элементы; задаётся свойством `flex-direction`: `row` — main axis горизонтальна (слева направо, значение по умолчанию), `column` — main axis вертикальна (сверху вниз). **Cross axis** (поперечная ось) — всегда перпендикулярна main axis.

```
flex-direction: row  (main → → →)
[ item1 ][ item2 ][ item3 ]
   ↑
 cross

flex-direction: column (main ↓ ↓ ↓)
[ item1 ]
[ item2 ]
[ item3 ]
   ←
 cross
```

Ключевое правило, о котором легко забыть: `justify-content` управляет выравниванием по **main axis**, а `align-items` — по **cross axis**. Если `flex-direction` меняется с `row` на `column`, смысл этих двух свойств визуально «меняется местами» — `justify-content` начинает управлять вертикалью, а `align-items` — горизонталью.

```html
<div class="container">
  <div class="item">1</div>
  <div class="item">2</div>
  <div class="item">3</div>
</div>
```

`flex-direction: row`:
```css
.container {
  display: flex;

  flex-direction: row;

  gap: 10px;
  border: 2px solid #333;
  padding: 10px;
}

.item {
  width: 80px;
  height: 80px;
  background: steelblue;
  color: white;
  text-align: center;
  line-height: 80px;
}
```

`flex-direction: column` :
```css
.container {
  display: flex;

  flex-direction: column;

  gap: 10px;
  border: 2px solid #333;
  padding: 10px;
}
```

### Grow, shrink, basis — как элементы делят пространство

`flex-grow` — насколько элемент растёт, занимая свободное место (0 — не растёт, 1 и больше — растёт пропорционально своему значению относительно других элементов). 

`flex-shrink` — насколько элемент сжимается, если места не хватает (по умолчанию 1 — может сжиматься, 0 — сохраняет свой размер даже в ущерб соседям). 

`flex-basis` — начальный (базовый) размер элемента до применения роста/сжатия — может быть в `px`, `%` или `auto`.

**Сокращённая запись**: `flex: <grow> <shrink> <basis>;` — например, `flex: 1;` эквивалентно `flex: 1 1 0%` (частый способ равномерного заполнения пространства), а `flex: 0 0 200px;` задаёт фиксированную ширину 200px без роста и сжатия.

```html
<div class="container">
  <div class="item item-1">1</div>
  <div class="item item-2">2</div>
  <div class="item item-3">3</div>
</div>
```

```css
.container {
  display: flex;
  border: 2px solid #333;
}

.item {
  padding: 20px;
  color: white;
  text-align: center;
}

.item-1 {
  background: steelblue;
  flex: 1;
}

.item-2 {
  background: tomato;
  flex: 2;
}

.item-3 {
  background: seagreen;
  flex: 1;
}
```

---

## Свойства контейнера

### `display: flex` / `display: inline-flex`

```css
.flex-container {
  display: flex;
}
```

Все прямые потомки этого контейнера становятся flex-элементами и по умолчанию выстраиваются в одну строку. `display: inline-flex` даёт то же поведение внутри, но сам контейнер при этом ведёт себя как строчный элемент относительно окружающей разметки.

```html
<div class="flex-container">
  <div class="item">1</div>
  <div class="item">2</div>
  <div class="item">3</div>
</div>
```

```css
.flex-container {
  display: flex;
  border: 2px solid #333;
  padding: 10px;
  gap: 10px;
}

.item {
  width: 80px;
  height: 80px;
  background: steelblue;
  color: white;
  text-align: center;
  line-height: 80px;
}
```

### `flex-direction`

Задаёт направление main axis: `row` (по умолчанию, слева направо), `row-reverse` (справа налево, порядок элементов визуально обратный), `column` (сверху вниз), `column-reverse` (снизу вверх).

```html
<div class="flex-container">
  <div class="item">1</div>
  <div class="item">2</div>
  <div class="item">3</div>
</div>
```

```css
.flex-container {
  display: flex;

  flex-direction: row;
  flex-direction: row-reverse;
  flex-direction: column;
  flex-direction: column-reverse;

  gap: 10px;
  border: 2px solid #333;
  padding: 10px;
}
```

### `flex-wrap`

По умолчанию все flex-элементы стараются поместиться в одну строку, даже если места не хватает. `flex-wrap: wrap` разрешает перенос на следующую строку; `nowrap` (по умолчанию) запрещает перенос; `wrap-reverse` переносит в обратном порядке (вверх).

```html
<div class="flex-container">
  <div class="item">1</div>
  <div class="item">2</div>
  <div class="item">3</div>
  <div class="item">4</div>
  <div class="item">5</div>
  <div class="item">6</div>
</div>
```

```css
.flex-container {
  display: flex;

  flex-wrap: nowrap;
  flex-wrap: wrap;

  gap: 10px;
  width: 320px;
  border: 2px solid #333;
  padding: 10px;
}

.item {
  width: 90px;
  height: 70px;
  background: steelblue;
  color: white;
  text-align: center;
  line-height: 70px;
}
```

Когда у вас карточки товаров или набор кнопок, `flex-wrap: wrap` позволяет им красиво переноситься на новую строку, а не вылезать за пределы контейнера или сжиматься до нечитаемого состояния.

### `justify-content` — выравнивание по main axis

```css
justify-content: flex-start | flex-end | center | space-between | space-around | space-evenly;
```

`flex-start`/`flex-end` — прижимает к началу/концу оси; 

`center` — центрирует; 

`space-between` — равные промежутки между элементами, крайние элементы прижаты к краям; 

`space-around` — равные отступы вокруг каждого элемента; 

`space-evenly` — равные промежутки везде, включая края.

```html
<div class="flex-container">
  <div class="item">1</div>
  <div class="item">2</div>
  <div class="item">3</div>
</div>
```

```css
.flex-container {
  display: flex;

  justify-content: flex-start;
  justify-content: flex-end;
  justify-content: center;
  justify-content: space-between;
  justify-content: space-around;
  justify-content: space-evenly;

  border: 2px solid #333;
  padding: 20px;
  height: 120px;
}

.item {
  width: 70px;
  height: 70px;
  background: steelblue;
  color: white;
  text-align: center;
  line-height: 70px;
}
```

### `align-items` — выравнивание по cross axis

```css
align-items: stretch | flex-start | flex-end | center | baseline;
```

`stretch` (по умолчанию) — элементы растягиваются на всю высоту контейнера по cross axis; 

`flex-start`/`flex-end` — прижимает к началу/концу поперечной оси; `center` — центрирует; 

`baseline` — выравнивает элементы по линии текста внутри них.

```html
<div class="flex-container">
  <div class="item">1</div>
  <div class="item">2</div>
  <div class="item">3</div>
</div>
```

```css
.flex-container {
  display: flex;

  align-items: stretch;
  align-items: flex-start;
  align-items: flex-end;
  align-items: center;
  align-items: baseline;

  height: 250px;
  border: 2px solid #333;
  gap: 10px;
}

.item {
  width: 70px;
  height: 70px;
  background: steelblue;
  color: white;
  text-align: center;
  line-height: 70px;
}
```

### `align-content` — выравнивание нескольких строк

Работает только при наличии переноса (`flex-wrap: wrap`) и нескольких строк flex-элементов — управляет распределением этих строк вдоль cross axis, точно так же, как `justify-content` распределяет отдельные элементы вдоль main axis. Если все элементы помещаются в одну строку, `align-content` не оказывает видимого эффекта.

```html
<div class="flex-container">
  <div class="item">1</div>
  <div class="item">2</div>
  <div class="item">3</div>
  <div class="item">4</div>
  <div class="item">5</div>
  <div class="item">6</div>
</div>
```

```css
.flex-container {
  display: flex;
  flex-wrap: wrap;

  align-content: stretch;
  align-content: flex-start;
  align-content: flex-end;
  align-content: center;
  align-content: baseline;
  align-content: space-between;

  width: 280px;
  height: 350px;

  border: 2px solid #333;
  gap: 10px;
}

.item {
  width: 70px;
  height: 70px;
  background: steelblue;
  color: white;
  text-align: center;
  line-height: 70px;
}
```

### `gap` — межэлементные отступы

Современная и более удобная альтернатива расстановке `margin` вручную у каждого элемента — `gap` задаёт равномерный промежуток между элементами, не требуя компенсировать лишний отступ у крайних элементов ряда, как это часто приходится делать с `margin`.

```html
<div class="flex-container">
  <div class="item">1</div>
  <div class="item">2</div>
  <div class="item">3</div>
</div>
```

```css
.flex-container {
  display: flex;
  gap: 20px;
  border: 2px solid #333;
  padding: 10px;
}

.item {
  width: 80px;
  height: 80px;
  background: steelblue;
  color: white;
  text-align: center;
  line-height: 80px;
}
```

### Центрирование блока по обеим осям — классический пример

Одна из самых частых задач вёрстки — центрирование блока — решается Flexbox буквально в две строки, без единого «хака» вроде `margin: 0 auto` вместе с абсолютным позиционированием, которые нам приходилось применять раньше.

```html
<div class="wrapper">
  <div class="box">Центр</div>
</div>
```

```css
.wrapper {
  display: flex;
  justify-content: center; /* по горизонтали */
  align-items: center; /* по вертикали */
  height: 300px;
}
```

---

## Свойства flex-элементов

### `flex-grow`, `flex-shrink`, `flex-basis` по отдельности

`flex-grow`:

```html
<div class="container">
  <div class="item">1</div>
  <div class="item item-big">2</div>
  <div class="item">3</div>
</div>
```

```css
.container {
  display: flex;
  border: 2px solid #333;
}

.item {
  flex-grow: 1;

  padding: 20px;
  background: steelblue;
  color: white;
  text-align: center;
}

.item-big {
  flex-grow: 2;
  background: tomato;
}
```

Получится примерно так:

```
|------1------|-----------2-----------|------3------|
```

Сразу видно, что второй элемент получил в два раза больше свободного пространства.

---

`flex-shrink`:

Теперь можно уменьшать ширину контейнера и наблюдать, что второй элемент практически не изменяет свой размер.

```html
<div class="container">
  <div class="item">1</div>
  <div class="item item-fixed">2</div>
  <div class="item">3</div>
</div>
```

```css
.container {
  display: flex;
  width: 250px;
  border: 2px solid #333;
}

.item {
  width: 120px;
  padding: 20px;
  background: steelblue;
  color: white;
  text-align: center;
}

.item-fixed {
  flex-shrink: 0;
  background: tomato;
}
```

---

`flex-basis`:

```html
<div class="container">
  <div class="item">1</div>
  <div class="item item-big">2</div>
  <div class="item">3</div>
</div>
```

```css
.container {
  display: flex;
  border: 2px solid #333;
  gap: 10px;
}

.item {
  flex-basis: 80px;

  background: steelblue;
  color: white;
  padding: 20px;
  text-align: center;
}

.item-big {
  flex-basis: 180px;
  background: tomato;
}
```

Сразу видно, что второй элемент получает больший стартовый размер.

### Сокращённая запись `flex`

```html
<div class="layout">
  <div class="left">Меню</div>
  <div class="center">Основной контент</div>
  <div class="right">Профиль</div>
</div>
```

```css
.layout {
  display: flex;
  border: 2px solid #333;
}

.left,
.right {
  flex: 0 0 120px;

  background: steelblue;
  color: white;
  padding: 20px;
  text-align: center;
}

.center {
  flex: 1;

  background: tomato;
  color: white;
  padding: 20px;
  text-align: center;
}
```

Это классический паттерн «фиксированные боковые колонки + гибкий центр» — очень частая раскладка в реальных интерфейсах.

### `order` — визуальный порядок без изменения HTML

```html
<div class="container">
  <div class="item item-1">Первый</div>
  <div class="item item-2">Второй</div>
  <div class="item item-3">Третий</div>
</div>
```

```css
.container {
  display: flex;
  gap: 10px;
}

.item {
  width: 90px;
  padding: 20px;
  background: steelblue;
  color: white;
  text-align: center;
}

.item-1 {
  order: 2;
}

.item-2 {
  order: 1;
}

.item-3 {
  order: 3;
}
```

Элемент с меньшим значением `order` отображается раньше; значение по умолчанию — `0`. Важная оговорка: `order` меняет только **визуальный** порядок — порядок фокуса при навигации с клавиатуры (`Tab`) и порядок чтения для экранных читалок при этом определяется исходным порядком в HTML, а не значением `order`. Не полагайтесь на `order` там, где важна логика или доступность, — только для чисто визуальной перестановки.

### `align-self` — переопределение выравнивания для одного элемента

```html
<div class="container">
  <div class="item">1</div>
  <div class="item item-special">2</div>
  <div class="item">3</div>
</div>
```

```css
.container {
  display: flex;
  align-items: center;

  height: 220px;
  gap: 10px;

  border: 2px solid #333;
}

.item {
  width: 80px;
  height: 80px;

  background: steelblue;
  color: white;

  text-align: center;
  line-height: 80px;
}

.item-special {
  align-self: flex-start;
  background: tomato;
}
```

---

## Практический пример: сайдбар + основной контент

```html
<div class="layout">
  <aside class="sidebar">
    Меню<br>
    Главная<br>
    Каталог<br>
    Контакты
  </aside>

  <main class="content">
    <h2>Основной контент</h2>

    <p>
      Здесь располагается содержимое страницы.
      Этот блок занимает всё свободное пространство.
    </p>
  </main>
</div>
```

```css
.layout {
  display: flex;
  gap: 16px;

  border: 2px solid #333;
  padding: 15px;
}

.sidebar {
  flex: 0 0 220px;

  background: #dceeff;
  padding: 15px;
}

.content {
  flex: 1;

  background: #f7f7f7;
  padding: 15px;
}
```

`.sidebar` всегда остаётся 220px, а `.content` эластично заполняет всё, что осталось от ширины родителя — классический макет «фиксированная колонка + гибкий контент».

---

## Практические паттерны Flexbox

### **Горизонтальная навигация с равномерным распределением**

`justify-content: space-between` растягивает первый пункт к началу, последний — к концу, а промежутки между остальными делает равными.

```html
<nav class="menu">
  <a href="#">Главная</a>
  <a href="#">Каталог</a>
  <a href="#">Блог</a>
  <a href="#">Контакты</a>
</nav>
```

```css
.menu {
  display: flex;
  justify-content: space-between;

  padding: 15px;
  background: #333;
}

.menu a {
  color: white;
  text-decoration: none;
}
```

---

### **Центрирование блока внутри контейнера**

`justify-content: center` + `align-items: center`, разобрано выше.


Используйте пример из разделов `justify-content` и `align-items`, рассмотренный выше.

---

### **Сетка карточек с переносом**

`flex-wrap: wrap` вместе с `flex: 1 1 200px` на каждой карточке: карточки растут и сжимаются, но не уже 200px, и автоматически переносятся на новую строку, если не помещаются.

```html
<div class="cards">
  <div class="card">Карточка 1</div>
  <div class="card">Карточка 2</div>
  <div class="card">Карточка 3</div>
  <div class="card">Карточка 4</div>
</div>
```

```css
.cards {
  display: flex;
  flex-wrap: wrap;
  gap: 15px;
}

.card {
  flex: 1 1 200px;

  padding: 30px;
  background: steelblue;
  color: white;
  text-align: center;
}
```

---

### **Sticky footer (прижатый футер)**

Классическая задача, которую Flexbox решает элегантно:

```html
<div class="page">
  <header>Шапка сайта</header>

  <main>
    Основной контент страницы
  </main>

  <footer>
    Подвал сайта
  </footer>
</div>
```

```css
.page {
  display: flex;
  flex-direction: column;

  min-height: 100vh;
}

header,
main,
footer {
  padding: 20px;
}

header {
  background: steelblue;
  color: white;
}

main {
  flex: 1;
  background: #f2f2f2;
}

footer {
  margin-top: auto;

  background: #333;
  color: white;
}
```

Контейнер `.page` со всей высотой окна (`min-height: 100vh`) и `flex-direction: column` — `main` растягивается через `flex: 1`, забирая всё оставшееся пространство, а футер с `margin-top: auto` в результате всегда оказывается прижат к низу, даже если контента на странице очень мало.

### **Карточка товара (картинка + текст)**

`flex: 0 0 120px` у изображения (фиксированная ширина) и `flex: 1` у текстового блока (растягивается на оставшееся место) — та же логика «фиксированная часть + гибкая часть», что и в примере с сайдбаром.

```html
<div class="product">
  <img
    src="https://placehold.co/120x120"
    alt="Товар"
  >

  <div class="info">
    <h3>Название товара</h3>

    <p>
      Краткое описание товара.
      Этот блок занимает всё оставшееся пространство.
    </p>
  </div>
</div>
```

```css
.product {
  display: flex;
  gap: 20px;

  max-width: 600px;

  border: 2px solid #333;
  padding: 15px;
}

.product img {
  flex: 0 0 120px;

  width: 120px;
  height: 120px;
}

.info {
  flex: 1;
}
```

---

### Резюме паттернов

| Задача | Решение |
|---|---|
| Выравнивание по центру | `justify-content` + `align-items` |
| Перенос карточек | `flex-wrap: wrap` |
| Прижатый футер | `flex-direction: column` + `margin-top: auto` на футере |
| Растягивание блока | `flex: 1` |
| Фиксированная колонка + гибкий контент | `flex: 0 0 Xpx` + `flex: 1` |

---

## Вопросы для проверки

1. В чём разница между flex-контейнером и flex-элементом?
2. Что определяет `flex-direction`, и как это влияет на смысл `justify-content` и `align-items`?
3. Чем `flex-grow: 0` отличается от `flex-grow: 1` у элемента?
4. Что делает `flex-wrap: wrap`, и что произойдёт с элементами без этого свойства, если места в контейнере не хватает?
5. Приведи пример сокращённой записи `flex`, которая задаёт элементу фиксированную ширину 150px без возможности расти или сжиматься.
6. Что делает свойство `order`, и какое важное ограничение стоит о нём помнить?
7. Как с помощью Flexbox реализовать классический sticky footer — футер, прижатый к низу экрана, даже если контента на странице мало?
8. В чём разница между `align-items` и `align-content`, и при каком условии `align-content` вообще оказывает видимый эффект?

---

## Практические задания

### Задание 1. Шапка сайта с логотипом, названием и навигацией

Создай `<header>` с тремя элементами: логотип слева, название сайта по центру, навигация (3–4 ссылки) справа. Все элементы шапки должны быть выровнены по вертикали по центру. Ссылки внутри навигации должны идти в один ряд с небольшими промежутками между ними.

<details>
<summary><b>Готовая html разметка</b></summary>

```html
<header class="site-header">
  <div class="logo">MyLogo</div>
  <div class="site-title">Мой сайт</div>
  <nav class="site-nav">
    <a href="#">Главная</a>
    <a href="#">О нас</a>
    <a href="#">Услуги</a>
    <a href="#">Контакты</a>
  </nav>
</header>
```

</details>

---

### Задание 2. Карточки с переносом и вертикальной компоновкой

Создай блок `<main>` с четырьмя карточками (`div.card`). В каждой карточке — картинка (с одинаковыми пропорциями, обрезанная по краям без искажений), заголовок `<h3>` и абзац с описанием. Карточки должны переноситься на новую строку при нехватке места, иметь белый фон, скруглённые углы и лёгкую тень; содержимое карточки располагается вертикально — сверху картинка, затем заголовок и текст.

<details>
<summary><b>Готовая html разметка</b></summary>

```html
<main class="cards">
  <div class="card">
    <div class="card__cover">
      <img src="https://picsum.photos/300/200?1" alt="Иллюстрация 1" />
    </div>
    <h3>Карточка 1</h3>
    <p>Краткое описание первой карточки для примера.</p>
  </div>
  <div class="card">
    <div class="card__cover">
      <img src="https://picsum.photos/300/200?2" alt="Иллюстрация 2" />
    </div>
    <h3>Карточка 2</h3>
    <p>Краткое описание второй карточки для примера.</p>
  </div>
  <div class="card">
    <div class="card__cover">
      <img src="https://picsum.photos/300/200?3" alt="Иллюстрация 3" />
    </div>
    <h3>Карточка 3</h3>
    <p>Краткое описание третьей карточки для примера.</p>
  </div>
  <div class="card">
    <div class="card__cover">
      <img src="https://picsum.photos/300/200?4" alt="Иллюстрация 4" />
    </div>
    <h3>Карточка 4</h3>
    <p>Краткое описание четвёртой карточки для примера.</p>
  </div>
</main>
```

</details>

---

### Задание 3. Цельная страница с прижатым футером — объединяющая практика

Собери страницу целиком: шапка из Задания 1, карточки из Задания 2 внутри `main`, и футер с контактной информацией. Сделай так, чтобы футер всегда оставался внизу окна браузера, даже если карточек мало и контента на странице недостаточно для заполнения экрана. Добавь фоновое изображение для всей страницы.

<details>
<summary><b>Готовая html разметка</b></summary>

```html
<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8" />
  <title>Задание 3</title>
  <link rel="stylesheet" href="style.css" />
</head>
<body class="page">
  <header class="site-header">
    <div class="logo">MyLogo</div>
    <div class="site-title">Мир животных</div>
    <nav class="site-nav">
      <a href="#">Главная</a>
      <a href="#">Каталог</a>
      <a href="#">Контакты</a>
    </nav>
  </header>

  <main class="cards">
    <div class="card">
      <div class="card__cover">
        <img src="https://picsum.photos/300/200?11" alt="Кот" />
      </div>
      <h3>Кошки</h3>
      <p>Ласковые и независимые питомцы.</p>
    </div>
    <div class="card">
      <div class="card__cover">
        <img src="https://picsum.photos/300/200?12" alt="Собака" />
      </div>
      <h3>Собаки</h3>
      <p>Верные друзья и компаньоны.</p>
    </div>
  </main>

  <footer class="site-footer">
    <p>Телефон: +7 900 000-00-00 · Email: info@example.com</p>
  </footer>
</body>
</html>
```

</details>

---

[Предыдущий урок](lesson09.md) | [Следующий урок](lesson11.md)