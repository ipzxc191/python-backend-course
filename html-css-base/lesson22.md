# Урок 22. Асинхронность: `callback` → `Promise` → `async`/`await`, `fetch`

## Последний недостающий кусочек моста

Мы почти у цели. В Уроке 15 мы спроектировали форму создания поста с полями `title`, `content`, `author_id` — точно совпадающими с моделью `Post` нашего Blog API — и честно объяснили, что нативная отправка формы не сработает с JSON-based FastAPI-эндпоинтом. В Уроке 21 мы перехватили отправку этой формы через `event.preventDefault()`, остановив перезагрузку страницы — но дальше данные никуда не уходили, только выводились в консоль.

Сегодня — последний шаг: мы научимся **на самом деле** отправлять и получать данные с сервера, без перезагрузки страницы, в формате JSON, которого ждёт наш Blog API. Инструмент называется `fetch`, но, чтобы понять его правильно, нужно сначала разобраться с тем, что вообще значит "асинхронный" код в JavaScript — и именно здесь чаще всего возникает главный вопрос новичков: **где именно нужно обрабатывать данные, которые пришли с сервера**. Разберём это подробно.

---

## 1. Почему обычный код "синхронный", и в чём проблема

Весь код, который мы писали до сих пор, выполнялся **синхронно** — строка за строкой, по порядку, каждая следующая строка ждёт завершения предыдущей:

```javascript
console.log('1');
console.log('2');
console.log('3');
// Выведется строго по порядку: 1, 2, 3
```

JavaScript — однопоточный язык: в любой момент времени выполняется только одна операция. Это прекрасно работает для быстрых вычислений, но что происходит, когда операция занимает **непредсказуемо долгое время** — например, ожидание ответа от сервера, который может прийти через 50мс, а может — через 3 секунды? Если бы JavaScript просто "остановился" и ждал ответа синхронно, вся страница на это время замерла бы — ни клика, ни прокрутки, ничего не реагировало бы, пока не придёт ответ.

Именно для таких операций — сетевые запросы, таймеры, чтение файлов — существует **асинхронный** код: способ запустить операцию, не останавливая при этом выполнение остального кода, и обработать результат **позже**, когда он будет готов.

Мы уже видели асинхронность на практике — в Уроке 20, через `setTimeout`:

```javascript
console.log('1');

setTimeout(() => {
  console.log('2 (отложено)');
}, 1000);

console.log('3');

// Порядок вывода: 1, 3, 2 (отложено) — а не 1, 2, 3!
```

`setTimeout` запускает таймер и **немедленно** возвращает управление остальному коду — строка `console.log('3')` выполняется сразу же, не дожидаясь истечения секунды. Функция внутри `setTimeout` выполнится позже, когда для неё придёт время — а до этого момента остальная программа продолжает работать как обычно.

---

## 2. Callback — самый ранний способ работы с асинхронностью

Способ, который мы уже использовали (сами того не называя явно) — передать функцию, которая будет вызвана **когда-нибудь потом**, когда асинхронная операция завершится. Такая функция называется **callback**.

```javascript
function loadUserData(callback) {
  setTimeout(() => {
    const userData = { name: 'Мария', age: 25 };
    callback(userData); // вызываем переданную функцию, когда данные "готовы"
  }, 1000);
}

loadUserData((user) => {
  console.log('Данные загружены:', user);
});

console.log('Эта строка выполнится РАНЬШЕ, чем данные загрузятся');
```

Создадим функции `loadUser`, `loadPosts` и `loadComments`, чтобы **имитировать** асинхронные операции. В реальном приложении вместо `setTimeout` здесь могла бы выполняться другая асинхронная операция — например, запрос к серверу.

```javascript
function loadUser(id, callback) {
  setTimeout(() => {
    const user = {
      id: id,
      name: 'Мария'
    };

    callback(user);
  }, 1000);
}

function loadPosts(userId, callback) {
  setTimeout(() => {
    const posts = [
      { id: 101, title: 'Первый пост' },
      { id: 102, title: 'Второй пост' }
    ];

    callback(posts);
  }, 1000);
}

function loadComments(postId, callback) {
  setTimeout(() => {
    const comments = [
      { id: 1, text: 'Отличный пост!' },
      { id: 2, text: 'Спасибо за публикацию' }
    ];

    callback(comments);
  }, 1000);
}
```

### Проблема: "ад коллбэков" (callback hell)

Если одна асинхронная операция должна начаться **после** завершения другой (например, сначала загрузить пользователя, затем по его `id` загрузить его посты, затем по `id` поста — комментарии), callback-и приходится вкладывать один в другой.

```javascript
loadUser(1, (user) => {
  loadPosts(user.id, (posts) => {
    loadComments(posts[0].id, (comments) => {
      console.log('Комментарии:', comments);
      // а если нужен ещё один шаг — вложенность станет ещё глубже...
    });
  });
});
```

Такая лестница вложенных функций получила собственное название — "callback hell" ("ад коллбэков"): код физически "уезжает" вправо с каждым новым шагом, читать и поддерживать его становится тяжело, а обработка ошибок на каждом уровне превращается в отдельную головную боль. Именно эта проблема привела к появлению `Promise`.

---

## 3. `Promise` — обещание будущего результата

**Promise** (промис, "обещание") — специальный объект, представляющий результат асинхронной операции, который пока не готов, но станет готов **когда-нибудь**: либо успешно (**fulfilled**), либо с ошибкой (**rejected**). До этого момента промис находится в состоянии **pending** ("в ожидании").

```javascript
const loadUserPromise = new Promise((resolve, reject) => {
  setTimeout(() => {
    const success = true;

    if (success) {
      resolve({ name: 'Мария', age: 25 }); // операция успешна — передаём результат
    } else {
      reject(new Error('Не удалось загрузить пользователя')); // операция провалилась
    }
  }, 1000);
});
```

Создавать собственные промисы вручную (через `new Promise(...)`) на практике приходится редко — гораздо чаще вы будете **получать** уже готовый промис от встроенных функций браузера (как `fetch`, который мы разберём чуть ниже). Но полезно понимать, что происходит "под капотом": промис — это просто объект-обёртка вокруг значения, которое станет доступно позже.

### `.then()` / `.catch()` / `.finally()`

```javascript
loadUserPromise
  .then((user) => {
    console.log('Пользователь загружен:', user);
  })
  .catch((error) => {
    console.error('Ошибка:', error.message);
  })
  .finally(() => {
    console.log('Операция завершена (успешно или с ошибкой)');
  });
```

- **`.then(callback)`** — выполнится, если промис успешно завершился (`resolve`); получает переданное в `resolve` значение как аргумент.
- **`.catch(callback)`** — выполнится, если промис завершился с ошибкой (`reject`).
- **`.finally(callback)`** — выполнится в любом случае, независимо от результата — удобно для действий вроде "скрыть индикатор загрузки", которые нужны и при успехе, и при ошибке.

### Цепочка `.then()` вместо вложенных callback-ов

Ключевое преимущество промисов — их можно **объединять цепочкой**, вместо вложенности. Но сначала — важная деталь: функции `loadUser`, `loadPosts` и `loadComments`, которые мы написали чуть выше, принимают `callback` последним аргументом и ничего не `return`-ят. У обычной функции без `return` не может быть метода `.then()` — если прямо сейчас написать `loadUser(1).then(...)`, JavaScript выдаст ошибку `Cannot read properties of undefined (reading 'then')`, потому что `loadUser(1)` вернёт `undefined`.

Чтобы пользоваться промисами, эти функции нужно переписать — не принимать `callback` параметром, а возвращать `new Promise(...)`, который сам вызовет `resolve` именно там, где раньше вызывался `callback`:

```javascript
function loadUser(id) {
  return new Promise((resolve) => {
    setTimeout(() => {
      const user = { id, name: 'Мария' };
      resolve(user); // то же место, где раньше был callback(user)
    }, 1000);
  });
}

function loadPosts(userId) {
  return new Promise((resolve) => {
    setTimeout(() => {
      const posts = [
        { id: 101, title: 'Первый пост' },
        { id: 102, title: 'Второй пост' }
      ];
      resolve(posts);
    }, 1000);
  });
}

function loadComments(postId) {
  return new Promise((resolve) => {
    setTimeout(() => {
      const comments = [
        { id: 1, text: 'Отличный пост!' },
        { id: 2, text: 'Спасибо за публикацию' }
      ];
      resolve(comments);
    }, 1000);
  });
}
```

Сигнатура поменялась — было `loadUser(id, callback)`, стало `loadUser(id)`. Тело функции почти не изменилось: тот же `setTimeout`, те же данные — просто вместо вызова переданного извне `callback(user)` мы вызываем встроенный в промис `resolve(user)`. Именно эти, promise-based версии функций используются во всех примерах дальше по уроку — включая раздел про `async`/`await`.

Теперь цепочка `.then()` работает корректно:

```javascript
loadUser(1)
  .then((user) => loadPosts(user.id))
  .then((posts) => loadComments(posts[0].id))
  .then((comments) => {
    console.log('Комментарии:', comments);
  })
  .catch((error) => {
    console.error('Что-то пошло не так на любом из шагов:', error.message);
  });
```

Код больше не "уезжает" вправо с каждым новым шагом — каждый `.then()` возвращает **новый** промис, к которому можно снова прицепить `.then()`, и так далее. Одна-единственная `.catch()` в конце цепочки перехватывает ошибку, произошедшую на **любом** из предыдущих шагов — не нужно обрабатывать ошибки на каждом уровне отдельно, как было с callback-ами.

---

## 4. `async`/`await` — самый читаемый способ работы с промисами

`async`/`await` — не отдельный, принципиально новый механизм, а более удобный **синтаксис** поверх тех же самых промисов. Перепишем цепочку выше:

```javascript
async function loadCommentsFlow() {
  try {
    const user = await loadUser(1);
    const posts = await loadPosts(user.id);
    const comments = await loadComments(posts[0].id);
    console.log('Комментарии:', comments);
  } catch (error) {
    console.error('Что-то пошло не так на любом из шагов:', error.message);
  }
}

loadCommentsFlow();
```

Ключевые правила:

- Функция, внутри которой используется `await`, обязательно должна быть объявлена с ключевым словом `async` перед `function` (или перед стрелочной функцией: `const loadCommentsFlow = async () => { ... }`).
- `await` перед вызовом функции, возвращающей промис, "приостанавливает" выполнение именно этой `async`-функции (не всей программы!) до тех пор, пока промис не завершится — и сразу возвращает **готовое значение**, а не сам промис.
- Обработка ошибок происходит через привычный `try/catch` (тот же самый синтаксис, что и в обычном синхронном коде), а не через `.catch()` — что делает асинхронный код внешне почти неотличимым от обычного, последовательного кода.

**Важно понимать:** `await` не превращает асинхронный код в синхронный "по-настоящему" — остальная программа (за пределами этой конкретной `async`-функции) продолжает работать, пока мы "ждём" внутри неё. Просто *внутри* самой функции код выглядит и читается так же линейно, как синхронный, что значительно упрощает восприятие по сравнению с цепочками `.then()`.

---

## 5. `fetch` — реальный запрос к серверу

`fetch(url)` — встроенная функция браузера для выполнения HTTP-запросов. Она возвращает `Promise`, который "разрешается" (resolve), когда сервер прислал **какой-либо** ответ — не обязательно успешный (это важная тонкость, разберём в подводных камнях).

### Готовим окружение: подключаем реальный Blog API

Прежде чем писать первый `fetch`, нужно, чтобы было куда его отправлять. Для этого используем готовый проект **Blog API** из курса `python-sql-fastapi` — его полная реализация лежит в отдельном репозитории: [blog-api-backend](https://github.com/ipzxc191/blog-api-backend). Скачайте проект и следуйте инструкциям из его `README.md`.

Frontend и backend в этой практике — два независимых процесса, работающих на разных портах, и это создаёт один побочный эффект, о котором полезно знать заранее — **CORS**.

*CORS (Cross-Origin Resource Sharing) — механизм безопасности браузера, ограничивающий обращение веб-страницы к ресурсам другого источника (домена, порта или протокола). Если frontend открыт на `http://localhost:8080`, а backend работает на `http://localhost:8000` — с точки зрения браузера это два разных источника, и по умолчанию он заблокирует `fetch`-запрос между ними, защищая пользователя от потенциальной кражи данных сторонним сайтом.*

В проекте `blog-api-backend` CORS уже настроен на стороне сервера — донастраивать ничего не придётся, примеры этого урока заработают сразу. Но само явление стоит запомнить: если однажды в консоли браузера появится ошибка со словом "CORS" — это не баг в вашем JS-коде, а недостающая настройка на стороне сервера, и решать её нужно именно там (backend должен явно разрешить обращения с адреса вашего frontend), а не в `fetch`-коде.

Запустите backend согласно инструкции проекта — он поднимется на `http://localhost:8000`. Frontend запустите отдельно, на другом порту: откройте терминал в папке с `index.html` и выполните:

```bash
python3 -m http.server 8080
```

Затем откройте `http://localhost:8080` в браузере — вы увидите страницу блога, которую дальше по уроку будем шаг за шагом наполнять данными.

<details>
  <summary>
    <b>Готовая html-разметка и стилизация к ней</b>
  </summary>

**index.html**:

```html
<!doctype html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Blog API — JavaScript Async</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <header class="site-header">
    <div class="container header-inner">
      <a class="logo" href="#">Blog<span>API</span></a>
      <div class="server"><i></i> localhost:8000</div>
    </div>
  </header>

  <main class="container">
    <section class="hero">
      <div>
        <p class="eyebrow">JavaScript · Async · Fetch</p>
        <h1>Посты из настоящего API</h1>
        <p>JavaScript получает данные с FastAPI-сервера и превращает JSON в карточки на странице.</p>
      </div>
      <div class="endpoint"><strong>GET</strong><code>/posts</code></div>
    </section>

    <section class="posts-section">
      <div class="section-title">
        <div>
          <p class="eyebrow">Данные с сервера</p>
          <h2>Посты</h2>
        </div>
      </div>
      <p class="posts__status">Загрузка постов...</p>
      <p class="posts__error" hidden></p>
      <div class="posts"></div>
    </section>

    <section class="create-post">
      <div class="section-title">
        <div>
          <p class="eyebrow">POST /posts</p>
          <h2>Создать пост</h2>
        </div>
      </div>
      <p class="form-intro">Форма используется для практики отправки данных на тот же FastAPI-сервер.</p>

      <form class="post-form">
        <div class="form-grid">
          <label class="field field-wide">
            <span>Заголовок</span>
            <input name="title" type="text" placeholder="Название поста" required>
          </label>

          <label class="field">
            <span>Автор</span>
            <select name="author_id" required>
              <option value="1">alexey</option>
              <option value="2">maria</option>
            </select>
          </label>

          <label class="field field-full">
            <span>Текст</span>
            <textarea name="content" rows="5" placeholder="Текст нового поста..." required></textarea>
          </label>
        </div>

        <div class="form-actions">
          <button type="submit">Создать пост</button>
          <span class="form-status" hidden></span>
        </div>
      </form>
    </section>
  </main>

  <footer>Учебная страница для урока «Асинхронность: callback → Promise → async/await → fetch»</footer>
  <script src="script.js"></script>
</body>
</html>
```

**style.css**

```css
* { box-sizing: border-box; }
:root {
  font-family: Inter, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  color: #172033; background: #f4f6f9; line-height: 1.5;
}
body { margin: 0; min-width: 320px; }
button, input, textarea, select { font: inherit; }
.container { width: min(1120px, calc(100% - 32px)); margin: 0 auto; }

.site-header { background: #172033; color: white; }
.header-inner { min-height: 72px; display: flex; align-items: center; justify-content: space-between; gap: 20px; }
.logo { color: white; text-decoration: none; font-size: 1.45rem; font-weight: 800; letter-spacing: -.04em; }
.logo span { color: #7dd3fc; }
.server { color: #cbd5e1; font-size: .9rem; display: flex; align-items: center; gap: 8px; }
.server i { width: 9px; height: 9px; border-radius: 50%; background: #4ade80; }

main { padding: 48px 0 64px; }
.hero {
  display: grid; grid-template-columns: minmax(0, 1fr) 190px; gap: 32px; align-items: center;
  padding: 42px; margin-bottom: 48px; border-radius: 24px; background: white;
  box-shadow: 0 12px 35px rgba(23,32,51,.07);
}
.eyebrow { margin: 0 0 8px; color: #64748b; font-size: .78rem; font-weight: 800; letter-spacing: .1em; text-transform: uppercase; }
.hero h1, h2 { margin: 0; line-height: 1.1; letter-spacing: -.035em; }
.hero h1 { max-width: 720px; font-size: clamp(2rem, 5vw, 3.4rem); }
.hero p:not(.eyebrow) { max-width: 700px; margin: 18px 0 0; color: #64748b; font-size: 1.05rem; }
.endpoint { display: grid; gap: 3px; justify-items: center; padding: 28px 15px; border: 1px solid #dbeafe; border-radius: 18px; background: #eff6ff; color: #1e3a8a; }
.endpoint strong { font-size: 1.7rem; }
.endpoint code { font-size: .9rem; }

.posts-section, .create-post { margin-top: 42px; }
.section-title { display: flex; justify-content: space-between; align-items: end; margin-bottom: 22px; }
.section-title h2 { font-size: 2rem; }
.posts { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 20px; }
.posts__status, .posts__error { margin: 0 0 20px; color: #64748b; }
.posts__error { padding: 14px 16px; border-radius: 12px; background: #fef2f2; color: #b91c1c; }

.post-card {
  display: flex; flex-direction: column; min-width: 0; padding: 24px;
  border: 1px solid #e2e8f0; border-radius: 18px; background: white;
  box-shadow: 0 6px 20px rgba(23,32,51,.05);
}
.post-card__top { display: flex; justify-content: space-between; gap: 16px; margin-bottom: 14px; }
.post-card h3 { margin: 0; font-size: 1.25rem; line-height: 1.25; }
.post-card__id { color: #94a3b8; font-family: ui-monospace, monospace; font-size: .78rem; }
.post-card__content { margin: 0 0 20px; color: #526176; }
.post-card__footer { display: flex; flex-wrap: wrap; align-items: center; gap: 8px; margin-top: auto; padding-top: 16px; border-top: 1px solid #edf0f4; }
.post-card__author { color: #334155; font-weight: 700; }
.post-card__date { color: #94a3b8; font-size: .85rem; }
.post-card__status, .post-card__tag { display: inline-flex; align-items: center; padding: 3px 9px; border-radius: 999px; font-size: .76rem; font-weight: 700; }
.post-card__status { margin-left: auto; background: #ecfdf5; color: #047857; }
.post-card__status--draft { background: #fff7ed; color: #c2410c; }
.post-card__tags { display: flex; flex-wrap: wrap; gap: 6px; width: 100%; margin-top: 5px; }
.post-card__tag { background: #f1f5f9; color: #475569; }

.create-post { padding: 30px; border-radius: 20px; background: white; box-shadow: 0 8px 25px rgba(23,32,51,.06); }
.form-intro { margin: -8px 0 24px; color: #64748b; }
.form-grid { display: grid; grid-template-columns: minmax(0, 2fr) minmax(180px, 1fr); gap: 18px; }
.field { display: grid; gap: 8px; }
.field-full { grid-column: 1 / -1; }
.field span { color: #334155; font-size: .9rem; font-weight: 700; }
.field input, .field textarea, .field select { width: 100%; border: 1px solid #cbd5e1; border-radius: 11px; padding: 12px 14px; color: #172033; background: white; outline: none; }
.field input:focus, .field textarea:focus, .field select:focus { border-color: #64748b; box-shadow: 0 0 0 3px rgba(100,116,139,.12); }
.field textarea { resize: vertical; }
.form-actions { display: flex; align-items: center; gap: 16px; margin-top: 20px; }
.form-actions button { border: 0; border-radius: 11px; padding: 12px 20px; color: white; background: #172033; cursor: pointer; font-weight: 700; }
.form-status { color: #047857; font-size: .9rem; }

footer { padding: 24px 16px; border-top: 1px solid #e2e8f0; color: #94a3b8; font-size: .8rem; text-align: center; }

@media (max-width: 760px) {
  .hero { grid-template-columns: 1fr; padding: 28px; }
  .endpoint { justify-self: start; width: 150px; }
  .posts { grid-template-columns: 1fr; }
  .form-grid { grid-template-columns: 1fr; }
  .field-full { grid-column: auto; }
}
@media (max-width: 480px) {
  .container { width: min(100% - 20px, 1120px); }
  main { padding-top: 24px; }
  .server { display: none; }
  .hero { padding: 24px; border-radius: 18px; }
  .create-post { padding: 22px; }
}

```

</details>

---

Дополнительно создайте рядом с `index.html` пустой файл `script.js` — он уже подключён в разметке через `<script src="script.js"></script>`, и весь код дальше по уроку мы пишем именно в нём.

Окружение готово — переходим к реализации запросов.

### GET-запрос: получаем список постов блога

```javascript
async function loadPosts() {
  const response = await fetch('http://localhost:8000/posts');
  const posts = await response.json(); // .json() тоже асинхронный — тоже требует await!
  return posts;
}

loadPosts().then((posts) => {
  console.log('Посты загружены:', posts);
});
```

Обратите внимание на **два** `await` подряд: `fetch(...)` возвращает промис, разрешающийся объектом `Response` (это ещё не сами данные, а обёртка вокруг ответа сервера — заголовки, статус-код и так далее). Метод `.json()` у этого объекта тоже возвращает **промис** (потому что чтение и разбор тела ответа тоже требует времени), который уже даёт настоящие данные — в нашем случае, массив объектов постов, ровно в том виде, который мы разбирали в Уроке 18.

### Использование внутри DOM — рендерим полученные посты

В `index.html` уже есть пустой контейнер `<div class="posts"></div>` — именно туда мы будем добавлять карточки постов. Задача сводится к тому, чтобы получить с сервера массив постов и превратить каждый объект этого массива в реальный DOM-элемент внутри контейнера.

```javascript
const postsContainer = document.querySelector('.posts'); // тот самый <div class="posts"></div> из HTML

// Превращает ОДИН объект поста в готовый DOM-элемент — но пока ничего не добавляет на страницу
const renderPost = (post) => {
  const article = document.createElement('article');
  article.classList.add('post');

  const title = document.createElement('h3');
  title.textContent = post.title; // текст берём прямо из данных, пришедших с сервера

  const content = document.createElement('p');
  content.textContent = post.content;

  article.append(title, content); // складываем заголовок и текст внутрь <article>
  return article; // возвращаем готовый элемент — он пока существует только в памяти
};

async function loadAndRenderPosts() {
  const response = await fetch('http://localhost:8000/posts');
  const posts = await response.json(); // posts — массив объектов постов с сервера

  posts.forEach((post) => {
    postsContainer.append(renderPost(post)); // а вот ЗДЕСЬ элемент реально появляется на странице
  });
}

loadAndRenderPosts();
```

Порядок действий здесь принципиален. `renderPost` только **создаёт** элемент и возвращает его — сам по себе вызов `renderPost(post)` ничего не меняет на странице, элемент существует лишь как значение переменной, нигде не подключённое к реальному документу. Пост физически появляется на экране только в строке `postsContainer.append(renderPost(post))` внутри `forEach`: `.append()` вставляет уже готовый элемент в DOM-дерево, внутрь `postsContainer`, и только после этого браузер его отрисовывает. Если бы мы вызвали `renderPost(post)` без `.append()`, элемент был бы создан, но остался бы «невидимым» — существующим в памяти, но не на странице.

---

## 6. Где именно обрабатывать полученные данные — разбираем подробно

Это ключевой практический вопрос, ради которого мы отдельно анонсировали этот раздел ещё в названии урока. Разберём его на конкретном, специально "сломанном" примере.

### Попытка "вытащить" данные из асинхронной функции синхронно — не работает

```javascript
async function loadPosts() {
  const response = await fetch('http://localhost:8000/posts');
  const posts = await response.json();
  return posts;
}

const posts = loadPosts(); // ❌ Забыли await!
console.log(posts); // Promise { <pending> } — а не массив постов!
console.log(posts[0]); // undefined — потому что posts — это промис, а не массив
```

Вот в чём суть: `async`-функция **всегда** возвращает промис, вне зависимости от того, что стоит после `return` внутри неё. Если вызвать такую функцию **без** `await` — переменная получит сам объект-промис (пока ещё, возможно, не завершившийся), а не то значение, которое эта функция должна вернуть в итоге. Обращаться к содержимому такого промиса напрямую, как к обычному массиву или объекту, невозможно — данных там ещё физически нет в момент выполнения этой строки.

### Правильный вариант — обрабатывать данные внутри `async`-функции или после `await`/`.then()`

```javascript
// Вариант 1: обработка внутри той же async-функции, где был await
async function loadAndLogPosts() {
  const response = await fetch('http://localhost:8000/posts');
  const posts = await response.json();

  // ✅ Обрабатываем данные ЗДЕСЬ, внутри этой же функции, после await
  console.log('Всего постов:', posts.length);
  posts.forEach((post) => console.log(post.title));
}

loadAndLogPosts();
```

```javascript
// Вариант 2: вызывающий код тоже использует await (если он сам находится внутри async-функции)
async function main() {
  const posts = await loadPosts(); // теперь дожидаемся результата
  console.log(posts); // ✅ настоящий массив, а не промис
}

main();
```

```javascript
// Вариант 3: вызывающий код использует .then(), если не может/не хочет быть async
loadPosts().then((posts) => {
  console.log(posts); // ✅ тоже настоящий массив — обработка происходит ВНУТРИ .then()
});
```

**Главное правило, отвечающее на вопрос "где обрабатывать данные":** данные, полученные асинхронно, должны обрабатываться **там же**, где было использовано `await` (внутри той же `async`-функции, после соответствующей строки) — либо в `.then()`, привязанном непосредственно к промису. Их нельзя "вынести" в обычный, синхронный код снаружи и ожидать, что переменная снаружи будет содержать готовое значение сразу после вызова — потому что сама природа асинхронности означает, что результат **ещё не готов** в момент, когда синхронный код продолжает выполняться дальше.

```javascript
// ❌ Так работать не будет — классическая ошибка новичков
let globalPosts;

async function loadPosts() {
  const response = await fetch('http://localhost:8000/posts');
  globalPosts = await response.json();
}

loadPosts(); // вызвали, но не дождались
console.log(globalPosts); // undefined — эта строка выполнится РАНЬШЕ, чем завершится loadPosts()!
```

```javascript
// ✅ Так — данные обрабатываются там, где они реально уже готовы
async function loadPosts() {
  const response = await fetch('http://localhost:8000/posts');
  const posts = await response.json();
  renderPostsToPage(posts); // используем данные СРАЗУ здесь, пока они точно готовы
}

function renderPostsToPage(posts) {
  posts.forEach((post) => {
    postsContainer.append(renderPost(post));
  });
}

loadPosts();
```

Второй пример показывает практический паттерн: сама функция обработки/рендера (`renderPostsToPage`) может быть обычной, синхронной функцией — она просто **вызывается изнутри** асинхронной функции, в тот самый момент, когда данные уже гарантированно готовы (сразу после `await`). Это и есть ответ на вопрос "где обрабатывать данные": не после вызова асинхронной функции, а **внутри неё** (или в цепочке `.then()`/другой `async`-функции, которая сама её дожидается через `await`).

---

## 7. POST-запрос: отправляем данные на сервер

Теперь — момент, обещанный ещё в Уроке 15. Вернёмся к форме создания поста и заставим её действительно отправлять данные в Blog API.

```javascript
const postForm = document.querySelector('.post-form');

postForm.addEventListener('submit', async (event) => {
  event.preventDefault(); // отменяем стандартную перезагрузку страницы (Урок 21)

  const formData = {
    title: postForm.querySelector('[name="title"]').value,
    content: postForm.querySelector('[name="content"]').value,
    author_id: Number(postForm.querySelector('[name="author_id"]').value),
  };

  try {
    const response = await fetch('http://localhost:8000/posts', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json', // говорим серверу: тело запроса — это JSON
      },
      body: JSON.stringify(formData), // превращаем объект в JSON-строку перед отправкой
    });

    if (!response.ok) {
      throw new Error(`Сервер ответил с ошибкой: ${response.status}`);
    }

    const createdPost = await response.json();
    console.log('Пост создан:', createdPost);

    postForm.reset(); // очищаем форму после успешной отправки
  } catch (error) {
    console.error('Не удалось создать пост:', error.message);
  }
});
```

Разберём ключевые новые детали относительно GET-запроса:

- **Второй аргумент `fetch(url, options)`** — объект с настройками запроса: `method` (по умолчанию `GET`, здесь явно `POST` — прямая параллель с атрибутом `method` формы из Урока 15), `headers`, `body`.
- **`headers: { 'Content-Type': 'application/json' }`** — сообщает серверу, что тело запроса нужно разбирать именно как JSON, а не как `form-urlencoded` (формат, который отправила бы нативная HTML-форма без JavaScript — как раз то несоответствие, о котором мы предупреждали в Уроке 15).
- **`body: JSON.stringify(formData)`** — тело запроса должно быть строкой, а не объектом напрямую; `JSON.stringify` превращает JS-объект в JSON-строку — это симметрично `JSON.parse`, который мы использовали в Уроке 21 для `localStorage`.
- **`response.ok`** — булево свойство, `true` для успешных HTTP-статусов (200–299), `false` для ошибок (404, 500 и подобных). Подробнее — в подводных камнях: это критически важная деталь, без которой ошибки сервера остаются незамеченными.

---

Все примеры выше были намеренно упрощены, чтобы не отвлекать от самой концепции `fetch`/`async`/`await` — мы работали только с `title` и `content`. Реальный Blog API возвращает у каждого поста ещё `id`, `author_username`, `status`, `created_at` и `tags`, и полноценный интерфейс обычно показывает всё это. Ниже — более полная версия того же самого frontend'а: она использует все поля поста, обрабатывает состояния загрузки и ошибки (`.posts__status`/`.posts__error` из разметки) и обновляет статус прямо в форме при отправке. Логика та же самая, что мы разбирали весь урок (`fetch` → `await` → `.json()` → изменение DOM) — просто применена к большему количеству данных.

**Одна сознательная деталь, отличающая этот код от предыдущих примеров:** здесь `renderPost` собирает разметку карточки через `innerHTML`, а не через `createElement`/`textContent`, как мы делали раньше и как советовали в Уроке 21. Причина — чисто практическая: у карточки много вложенных элементов (статус, дата, список тегов), и описывать каждый из них через `createElement` было бы заметно многословнее. Такое отступление от правила "используйте `textContent` для данных с сервера" допустимо именно потому, что содержимое приходит из нашего собственного backend, который мы полностью контролируем — здесь нет риска, что кто-то посторонний подложит в `title` или `content` вредоносный `<script>`. Если бы посты создавали произвольные, непроверенные пользователи — правильным выбором по-прежнему оставался бы `textContent`, а не `innerHTML`.

<details>
  <summary>
    <b>Готовый js для корректной работы всего frontend</b>
  </summary>

```javascript
const postsContainer = document.querySelector('.posts');
const postsStatus = document.querySelector('.posts__status');
const postsError = document.querySelector('.posts__error');
const postForm = document.querySelector('.post-form');

function renderPost(post) {
  const article = document.createElement('article');
  article.className = 'post-card';

  const tags = Array.isArray(post.tags) ? post.tags : [];

  article.innerHTML = `
    <div class="post-card__top">
      <h3>${post.title}</h3>
      <span class="post-card__id">#${post.id}</span>
    </div>

    <p class="post-card__content">${post.content}</p>

    <div class="post-card__footer">
      <span class="post-card__author">Автор: ${post.author_username}</span>
      ${post.created_at ? `<span class="post-card__date">${post.created_at}</span>` : ''}
      <span class="post-card__status ${post.status === 'draft' ? 'post-card__status--draft' : ''}">
        ${post.status}
      </span>

      <div class="post-card__tags">
        ${tags.map((tag) => `<span class="post-card__tag">#${tag}</span>`).join('')}
      </div>
    </div>
  `;

  postsContainer.append(article);
}

async function loadPosts() {
  const response = await fetch('http://localhost:8000/posts');

  if (!response.ok) {
    throw new Error(`Ошибка сервера: ${response.status}`);
  }

  const posts = await response.json();
  return posts;
}

async function showPosts() {
  try {
    postsStatus.hidden = false;
    postsError.hidden = true;

    const posts = await loadPosts();

    postsContainer.innerHTML = '';
    posts.forEach(renderPost);
    postsStatus.hidden = true;
  } catch (error) {
    postsStatus.hidden = true;
    postsError.hidden = false;
    postsError.textContent = `Не удалось загрузить посты: ${error.message}`;
  }
}

postForm.addEventListener('submit', async (event) => {
  event.preventDefault();

  const title = postForm.querySelector('[name="title"]').value;
  const content = postForm.querySelector('[name="content"]').value;
  const authorId = Number(postForm.querySelector('[name="author_id"]').value);
  const formStatus = postForm.querySelector('.form-status');

  formStatus.hidden = false;
  formStatus.textContent = 'Отправляем...';

  try {
    const response = await fetch('http://localhost:8000/posts', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        title,
        content,
        author_id: authorId,
      }),
    });

    if (!response.ok) {
      throw new Error(`Ошибка сервера: ${response.status}`);
    }

    const newPost = await response.json();

    renderPost(newPost);
    postForm.reset();
    formStatus.textContent = 'Пост создан.';
  } catch (error) {
    formStatus.textContent = `Ошибка: ${error.message}`;
  }
});

showPosts();
```

</details>

---

## Подводные камни

### Забытый `await` — получаем `Promise`, а не значение

Разобрано подробно в разделе 6 — это, вероятно, самая частая ошибка новичков в асинхронном JS. Если после вызова `async`-функции (или прямого вызова `fetch(...).json()`) в консоли видно `Promise { <pending> }` вместо ожидаемых данных — почти наверняка забыт `await` или `.then()`.

### `fetch` не выбрасывает ошибку при HTTP-ошибках (404, 500)

```javascript
// ❌ Опасное предположение: fetch "упадёт" сам по себе при ошибке сервера
try {
  const response = await fetch('http://localhost:8000/posts/99999'); // несуществующий пост
  const post = await response.json();
  console.log(post); // может оказаться сообщением об ошибке от сервера, а не постом!
} catch (error) {
  console.log('Сюда мы не попадём, даже если сервер ответил 404!');
}
```

Это одна из самых неожиданных особенностей `fetch`: промис, который он возвращает, переходит в состояние `reject` (и, соответственно, код попадает в `catch`) только при **сетевых** проблемах — сервер недоступен, нет интернет-соединения. Если сервер **ответил**, пусть даже кодом ошибки 404 или 500, — с точки зрения `fetch` это всё ещё "успешный" промис! Проверять реальный результат нужно вручную, через `response.ok` (или `response.status`):

```javascript
// ✅ Правильно — явная проверка response.ok
const response = await fetch('http://localhost:8000/posts/99999');

if (!response.ok) {
  throw new Error(`Ошибка сервера: ${response.status}`);
}

const post = await response.json();
```

### Забытый второй `await` для `.json()`

```javascript
const response = await fetch('http://localhost:8000/posts');
const posts = response.json(); // ❌ забыли await!
console.log(posts); // Promise { <pending> }, а не массив
```

`.json()` — тоже асинхронная операция (разбор тела ответа занимает время) и тоже возвращает промис, требующий собственного `await`, отдельного от того, что уже был использован для самого `fetch(...)`.

### Забытый `Content-Type` при отправке JSON

```javascript
// ❌ Без заголовка сервер может неправильно интерпретировать тело запроса
await fetch(url, {
  method: 'POST',
  body: JSON.stringify(formData), // headers не указаны!
});
```

Без явного `headers: { 'Content-Type': 'application/json' }` некоторые серверы (включая наш FastAPI-бэкенд, ожидающий JSON через Pydantic-схемы) могут не суметь корректно распознать формат тела запроса, даже если по факту в `body` передана правильно сформированная JSON-строка.

### CORS — если видите эту ошибку в консоли

Мы уже разбирали это при подготовке окружения в начале раздела: если frontend и backend работают на разных портах (как в нашей практике — `8080` и `8000`), браузер может заблокировать `fetch`-запрос политикой **CORS**. Напоминание: ошибка со словом "CORS" в консоли — это не баг в JS-коде, а недостающая настройка на стороне сервера; в проекте `blog-api-backend`, который мы используем, она уже включена, но если вы когда-нибудь подключите фронтенд к другому, самостоятельно написанному backend — не забудьте настроить CORS и там.

---

## Итоги урока

Асинхронный код нужен там, где операция занимает непредсказуемое время (сетевые запросы, таймеры) — вместо остановки всей программы в ожидании результата, асинхронная операция запускается, а её результат обрабатывается позже, когда он готов. Callback-и — самый ранний способ реализации этой идеи, но вложенные друг в друга callback-и приводят к плохо читаемому "аду коллбэков". `Promise` решает эту проблему, представляя асинхронный результат как объект с тремя состояниями (pending/fulfilled/rejected) и позволяя строить цепочки через `.then()`/`.catch()`/`.finally()` вместо вложенности.

`async`/`await` — синтаксис поверх промисов, делающий асинхронный код внешне похожим на обычный, последовательный: `await` перед вызовом, возвращающим промис, "дожидается" готового значения внутри `async`-функции, ошибки обрабатываются привычным `try/catch`.

`fetch(url, options)` выполняет HTTP-запрос и возвращает промис с объектом `Response`; сами данные извлекаются через `await response.json()` — второй, отдельный `await`. **Данные, полученные асинхронно, должны обрабатываться там же, где был использован `await`** — внутри той же `async`-функции сразу после получения результата, либо в `.then()`, привязанном к промису — попытка получить готовое значение синхронно, вызвав `async`-функцию без `await`/`.then()`, всегда вернёт сам объект `Promise`, а не данные внутри него.

Критически важная особенность `fetch`: он не считает HTTP-ошибки (404, 500) поводом для `reject` — необходимо явно проверять `response.ok` после каждого запроса, если важно корректно обработать ошибки сервера.

**Это был последний урок модуля перед финальной, объединяющей практикой.** В следующем, последнем уроке модуля JS мы соберём всё вместе: форму из Урока 15, DOM-рендер из Урока 21 и `fetch` из этого урока — в цельный, полностью работающий фронтенд поверх реального Blog API из курса `python-sql-fastapi`.

---

## Вопросы для проверки

1. Почему JavaScript не может просто "остановиться и подождать" во время сетевого запроса, как это происходит с обычными синхронными операциями?
2. В чём заключается проблема "ада коллбэков" (callback hell), и как `Promise` её решает?
3. Что означает `pending`, `fulfilled` и `rejected` применительно к `Promise`?
4. Что произойдёт, если вызвать `async`-функцию без `await` и попытаться сразу использовать возвращённое значение как обычные данные?
5. Сформулируйте главное правило: где именно должна происходить обработка данных, полученных асинхронно?
6. Почему `fetch` не переходит в состояние `rejected`, если сервер ответил кодом ошибки 404 или 500? Как правильно проверять такие ошибки?
7. Зачем нужен заголовок `Content-Type: application/json` при отправке POST-запроса с `fetch`, если тело уже сформировано через `JSON.stringify`?
8. Чем принципиально отличается `try/catch` вокруг `async`/`await`-кода от `.catch()` в цепочке промисов — или это, по сути, одно и то же?

---

## Практические задания

### Задание 1. Загрузка и рендер списка постов (GET)

Используя реальный (или заглушечный, если сервер сейчас недоступен, — можно временно заменить адрес на публичный тестовый API вроде `https://jsonplaceholder.typicode.com/posts`) Blog API:

1. Напишите `async`-функцию `loadPosts()`, которая делает `fetch` на `/posts`, проверяет `response.ok`, и возвращает распарсенный массив постов.
2. Оберните вызов в `try/catch`, выведя понятное сообщение об ошибке в консоль при неудаче.
3. Отрендерите каждый пост в контейнер `.posts` через `createElement`/`append` (используйте `textContent`, а не `innerHTML`, для вставки `title`/`content` — данные пришли с сервера, а не были написаны вами напрямую).

---

### Задание 2. Отправка формы поста (POST)

Возьмите форму создания поста из Урока 15, перехваченную в Уроке 21:

1. Внутри обработчика `submit` соберите данные полей в объект `{ title, content, author_id }`.
2. Отправьте их через `fetch` методом `POST` с корректным `Content-Type` и `JSON.stringify`.
3. Проверьте `response.ok`; при успехе — выведите созданный пост в консоль и сбросьте форму (`form.reset()`); при ошибке — выведите сообщение через `alert()`.
4. Оберните всё в `try/catch`.

---

### Задание 3. Полный цикл: загрузка, отправка, обновление списка без перезагрузки (объединяющая практика)

Соберите законченный сценарий на странице фильмов/блога:

1. При загрузке страницы вызовите `loadPosts()` (Задание 1) и отрендерите текущий список постов.
2. Настройте отправку формы (Задание 2) так, чтобы **после успешного создания поста** новый пост сразу добавлялся в `.posts` через `renderPost()`, без повторного запроса всего списка и без перезагрузки страницы.
3. Убедитесь, что при ошибке сети (например, временно остановите локальный сервер) пользователь видит понятное сообщение, а не "молчаливый" сбой без всякой обратной связи.

---

[Предыдущий урок](lesson21.md) | [Следующий урок](lesson23.md)