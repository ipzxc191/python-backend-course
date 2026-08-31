# Урок 28. Финальный проект — Финализация

<details>
<summary><b>Готовый seed файл для заполнения БД с комментариями</b></summary>


```python
# seed.py
"""
Сидер для Blog API (SQLAlchemy ORM).

Это тот же самый сидер, что мы писали в Уроке 12 — только поверх ORM
вместо raw sqlite3.cursor(). Сравните подходы:

    Урок 12 (raw sqlite3)                    Этот файл (SQLAlchemy ORM)
    ─────────────────────────────────────    ─────────────────────────────────────
    cursor.execute('DELETE FROM ...')        session.execute(delete(Model))
    cursor.executemany('INSERT ...', rows)   session.add_all([Model(...), ...])
    SELECT id заново, чтобы узнать id        session.flush() — id уже на объекте
    вручную считать внешние ключи            relationship (post.tags = [...])
                                              сам управляет таблицей post_tags

Запуск:
    python seed.py
"""

import random
from datetime import date

from faker import Faker
from sqlalchemy import delete
from sqlalchemy.orm import Session

from database import engine
from models import Base, User, Post, Comment, Tag, post_tags

# -----------------------------------------------------------------
# Константы — количество записей (как и в Уроке 12)
# -----------------------------------------------------------------
NUM_USERS = 12
NUM_POSTS = 25
NUM_COMMENTS = 60

# -----------------------------------------------------------------
# Фиксированный список тегов — по той же причине, что в Уроке 12
# продукты магазина брались из готового списка, а не fake.word():
# случайные слова не похожи на реальные теги технического блога.
# -----------------------------------------------------------------
TAG_NAMES = [
    'python', 'backend', 'sql', 'fastapi', 'django',
    'алгоритмы', 'дебаггинг', 'тестирование', 'api', 'devops',
    'базы-данных', 'асинхронность',
]

fake = Faker('ru_RU')
Faker.seed(42)  # воспроизводимый результат, как и в Уроке 12


# -----------------------------------------------------------------
# Очистка таблиц
# -----------------------------------------------------------------
def clear_tables(session: Session) -> None:
    """
    Удаляет все строки из таблиц в правильном порядке — зависимые таблицы первыми.

    В Уроке 12 порядок задавался вручную через несколько DELETE-запросов.
    Здесь то же самое, но через session.execute(delete(Model)) — это
    core-уровень SQLAlchemy, работает быстрее, чем загружать все объекты
    через ORM и удалять их по одному.

    post_tags удаляем отдельным delete(), так как это Table, а не класс-модель
    (напоминание из Урока 26 финального проекта) — обратиться к ней напрямую
    через ORM-класс, как к Comment или Post, нельзя.
    """
    session.execute(delete(Comment))
    session.execute(delete(post_tags))
    session.execute(delete(Post))
    session.execute(delete(Tag))
    session.execute(delete(User))
    session.commit()
    print('Таблицы очищены.')


# -----------------------------------------------------------------
# Генерация пользователей
# -----------------------------------------------------------------
def seed_users(session: Session, count: int = NUM_USERS) -> list[User]:
    """
    Генерирует пользователей с уникальными username и email.

    Отличие от Урока 12: там executemany() принимал список кортежей
    и итоговые id узнавались отдельным SELECT после вставки.
    Здесь session.add_all() принимает список ORM-объектов, а после
    session.flush() каждый объект уже содержит свой настоящий id —
    отдельный SELECT для этого не нужен.
    """
    users = [
        User(
            username=fake.unique.user_name(),
            email=fake.unique.email(),
            created_at=str(fake.date_between(start_date='-2y', end_date='today')),
        )
        for _ in range(count)
    ]

    session.add_all(users)
    session.flush()  # получаем id для каждого user, не коммитя транзакцию целиком

    print(f'Пользователей добавлено: {len(users)}')
    return users


# -----------------------------------------------------------------
# Генерация тегов
# -----------------------------------------------------------------
def seed_tags(session: Session) -> list[Tag]:
    """Вставляет фиксированный список тегов — аналог seed_categories из Урока 12."""
    tags = [Tag(name=name) for name in TAG_NAMES]

    session.add_all(tags)
    session.flush()

    print(f'Тегов добавлено: {len(tags)}')
    return tags


# -----------------------------------------------------------------
# Генерация постов
# -----------------------------------------------------------------
def seed_posts(session: Session, users: list[User], tags: list[Tag], count: int = NUM_POSTS) -> list[Post]:
    """
    Генерирует посты со случайным автором, случайным статусом и случайным
    набором тегов (0-3 тега на пост, без повторов).

    Ключевое отличие от Урока 12: там для связи многие-ко-многим пришлось бы
    вручную собирать пары (post_id, tag_id) и вставлять их в post_tags
    отдельным executemany(). Здесь достаточно присвоить relationship —
    post.tags = [...] — и SQLAlchemy сам вставит нужные строки в post_tags
    при коммите (это мы разбирали в Уроке 24 и в models.py финального проекта).
    """
    posts = []

    for _ in range(count):
        author = fake.random_element(users)

        # Статус: чаще published, реже draft — реалистичное распределение,
        # а не 50/50, как дал бы fake.random_element(['published', 'draft'])
        status = random.choices(['published', 'draft'], weights=[3, 1])[0]

        num_tags = fake.pyint(min_value=0, max_value=3)
        post_tags_selection = (
            fake.random_elements(tags, length=num_tags, unique=True) if num_tags > 0 else []
        )

        post = Post(
            title=fake.sentence(nb_words=6).rstrip('.'),
            content='\n\n'.join(fake.paragraphs(nb=3)),
            status=status,
            author_id=author.id,
            created_at=str(fake.date_between(start_date='-1y', end_date='today')),
        )
        post.tags = post_tags_selection  # relationship сама заполнит post_tags

        posts.append(post)

    session.add_all(posts)
    session.flush()

    print(f'Постов добавлено: {len(posts)}')
    return posts


# -----------------------------------------------------------------
# Генерация комментариев
# -----------------------------------------------------------------
def seed_comments(session: Session, users: list[User], posts: list[Post], count: int = NUM_COMMENTS) -> None:
    """
    Генерирует комментарии со случайным постом и случайным автором.

    В Уроке 12 для order_items мы получали id постов и товаров через
    отдельный SELECT после вставки заказов. Здесь у нас уже есть
    полноценные ORM-объекты posts/users из seed_posts()/seed_users() —
    их id доступны напрямую (post.id, author.id), без лишнего запроса к базе.
    """
    comments = []

    for _ in range(count):
        post = fake.random_element(posts)
        author = fake.random_element(users)

        comments.append(
            Comment(
                content=fake.sentence(nb_words=12),
                post_id=post.id,
                author_id=author.id,
                created_at=str(fake.date_between(start_date=date.fromisoformat(post.created_at), end_date='today')),
            )
        )

    session.add_all(comments)
    session.flush()

    print(f'Комментариев добавлено: {len(comments)}')


# -----------------------------------------------------------------
# Проверка результата
# -----------------------------------------------------------------
def print_stats(session: Session) -> None:
    """Выводит количество строк в каждой таблице — аналог print_stats из Урока 12."""
    print('\n--- Статистика базы данных ---')
    print(f'{"users":15}: {session.query(User).count()} строк')
    print(f'{"tags":15}: {session.query(Tag).count()} строк')
    print(f'{"posts":15}: {session.query(Post).count()} строк')
    print(f'{"comments":15}: {session.query(Comment).count()} строк')


# -----------------------------------------------------------------
# Точка входа
# -----------------------------------------------------------------
def main() -> None:
    # Base.metadata.create_all здесь НЕ вызываем — таблицы уже созданы
    # через Alembic-миграции (Урок 26). Сидер только наполняет данными.

    with Session(engine) as session:
        clear_tables(session)

        users = seed_users(session)
        tags = seed_tags(session)
        posts = seed_posts(session, users, tags)
        seed_comments(session, users, posts)

        session.commit()  # фиксируем всё одной транзакцией

        print_stats(session)
        print('\nБаза данных успешно заполнена.')


if __name__ == '__main__':
    main()
```

</details>

## Остановитесь и посмотрите назад

Прежде чем что-то улучшать — осознайте что сделано.

За этот курс вы прошли путь от `SELECT * FROM products` в SQLiteStudio до работающего REST API с ORM, миграциями, Pydantic-схемами и Dependency Injection. Это не "hello world" — это архитектурно правильный бэкенд-проект.

Этот урок устроен иначе чем предыдущие. Здесь нет длинного списка нового кода. Здесь три части:
1. Рефакторинг — небольшие улучшения которые делают проект чище
2. Честный разбор — что сделано "учебно" и почему
3. Карта дальнейшего пути — что вы уже умеете и куда идти дальше

---

## Часть 1. Рефакторинг

### Убираем дублирование `_serialize_*` функций

В Уроке 27 в каждом роутере есть функции `_serialize_post`, `_serialize_comment`. Это признак что Pydantic-схемы не полностью берут на себя сериализацию. Правильное решение — использовать `model_validate` чтобы Pydantic сам собирал объект из ORM-модели.

Для этого добавим в схемы настройку `model_config`:

```python
# schemas/posts.py — улучшенная версия
from pydantic import BaseModel, Field
from typing import List
from pydantic import ConfigDict


class PostResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    # from_attributes=True — говорит Pydantic читать данные
    # из атрибутов объекта, а не только из словаря.
    # Это позволяет передавать ORM-объект напрямую в схему.

    id:              int
    title:           str
    content:         str
    status:          str
    created_at:      str
    author_id:       int
```

Однако для полей `author_username` и `tags` (которые приходят через `relationship`) всё равно нужна ручная сборка — ORM-объект не знает что Pydantic ждёт плоский список строк вместо объектов `Tag`. Поэтому в финальном проекте мы оставляем `_serialize_*` как есть — это честное учебное решение, а не ошибка.

> В продакшне эту задачу решают по-разному: через кастомные `@validator`, через специальные поля `computed_field`, или через явное преобразование в репозитории. Единого "правильного" способа нет — есть компромиссы.

### Выносим константы

Допустимые статусы поста разбросаны по коду. Выносим в одно место:

```python
# constants.py
POST_STATUSES = ['draft', 'published']
```

```python
# schemas/posts.py — добавить валидацию статуса
from constants import POST_STATUSES
from pydantic import field_validator

class PostCreate(BaseModel):
    ...
    status: str = Field(default='published')

    @field_validator('status')
    @classmethod
    def validate_status(cls, v):
        if v not in POST_STATUSES:
            raise ValueError(f'Статус должен быть одним из: {POST_STATUSES}')
        return v
```

Теперь невалидный статус даёт `422` автоматически — без проверок в роутере.

### Проверка что всё работает

Запустите приложение и пройдите по каждому сценарию из Урока 27:

```bash
alembic upgrade head
uvicorn main:app --reload
```

Чеклист проверки:

```
□ POST /users — создать пользователя
□ POST /users повторно с тем же email — получить 400
□ POST /tags — создать несколько тегов
□ POST /posts с tag_ids — создать пост
□ GET /posts — получить список
□ GET /posts?tag=python — фильтрация работает
□ GET /posts?author_id=1 — фильтрация по автору
□ GET /posts/1 — полный пост с тегами и автором
□ GET /posts/9999 — получить 404 в правильном формате
□ POST /posts/1/comments — создать комментарий
□ GET /posts/1/comments — получить комментарии
□ DELETE /posts/1 — удалить пост (комментарии каскадно удалятся)
□ GET /health — проверка работоспособности
□ Открыть /docs — убедиться что документация актуальна
```

---

## Часть 2. Что сделано "учебно"

Это самая важная часть урока.

Наш Blog API — **рабочий прототип**. Он делает всё что должен делать: принимает запросы, валидирует данные, работает с базой через ORM, возвращает правильные коды статусов. Если запустить его прямо сейчас — он будет работать.

Но между "рабочим прототипом" и "продакшн-приложением" есть несколько шагов. Ни один из них не перечёркивает то что вы сделали — они просто добавляются сверху на уже правильный фундамент.

### Что упрощено и как это делается "по-настоящему"

---

**1. Аутентификация и авторизация**

*Как у нас:* `author_id` передаётся в теле запроса — любой может написать пост от имени любого пользователя.

*Как в продакшне:* пользователь логинится, получает JWT-токен, передаёт его в заголовке `Authorization: Bearer <token>`. Сервер проверяет токен и извлекает `user_id` автоматически — клиент его не передаёт.

```python
# Как это выглядит с JWT в FastAPI:
@router.post('/')
def create_post(
    data: PostCreate,
    current_user: User = Depends(get_current_user)  # из токена
):
    # current_user.id — уже известен, клиент не передаёт
    ...
```

*Что изучить:* `python-jose` или `python-jwt` для работы с токенами, `passlib` для хэширования паролей, OAuth2 в FastAPI.

---

**2. Права доступа**

*Как у нас:* любой может удалить любой пост или комментарий.

*Как в продакшне:* удалять пост может только его автор (или администратор). Это называется авторизация — проверка прав конкретного действия.

```python
# Проверка прав перед удалением:
if post.author_id != current_user.id and not current_user.is_admin:
    raise HTTPException(status_code=403, detail='Нет прав')
```

*Что изучить:* RBAC (Role-Based Access Control), разрешения в Django REST Framework.

---

**3. Пагинация**

*Как у нас:* `GET /posts` возвращает все посты — при 10 000 постов это проблема.

*Как в продакшне:* курсорная или offset-пагинация с метаданными.

```json
{
    "items": [...],
    "total": 10000,
    "page": 1,
    "pages": 500,
    "has_next": true
}
```

*Что изучить:* `fastapi-pagination`, курсорная пагинация для больших объёмов.

---

**4. Валидация email**

*Как у нас:* `email: str` — принимаем любую строку.

*Как в продакшне:* проверка формата через `EmailStr` из `pydantic[email]`.

```python
from pydantic import EmailStr

class UserCreate(BaseModel):
    email: EmailStr   # автоматически проверяет формат
```

*Что изучить:* `pip install pydantic[email]`, дополнительные валидаторы Pydantic.

---

**5. Хранение дат**

*Как у нас:* `created_at: str` — клиент передаёт дату, мы доверяем формату.

*Как в продакшне:* дата создания устанавливается сервером автоматически, клиент не может её подделать. Для SQLAlchemy — `Column(DateTime, default=func.now())`.

```python
# В модели:
from sqlalchemy import DateTime, func
created_at = Column(DateTime, server_default=func.now(), nullable=False)
```

---

**6. Тестирование**

*Как у нас:* ручная проверка через `/docs`.

*Как в продакшне:* автоматические тесты которые запускаются при каждом изменении кода.

```python
# Пример теста FastAPI:
from fastapi.testclient import TestClient

def test_create_user():
    response = client.post('/users', json={'username': 'test', ...})
    assert response.status_code == 201
    assert response.json()['username'] == 'test'
```

*Что изучить:* `pytest`, `httpx`, `pytest-asyncio`, тестирование с базой в памяти `sqlite:///:memory:`.

---

**7. Переменные окружения**

*Как у нас:* `DATABASE_URL = 'sqlite:///blog.db'` прямо в коде.

*Как в продакшне:* секреты (URL базы, секретный ключ JWT) хранятся в переменных окружения или `.env` файле — не в коде, не в Git.

```python
# settings.py через pydantic-settings:
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str = 'sqlite:///blog.db'
    secret_key: str

    class Config:
        env_file = '.env'

settings = Settings()
```

---

**8. Логирование и мониторинг**

*Как у нас:* `logger.info(...)` в нескольких местах.

*Как в продакшне:* структурированные логи (JSON-формат для агрегаторов), метрики (Prometheus), трейсинг запросов, алерты при ошибках.

---

### Итоговая таблица

```
Что готово:                     Что добавить для продакшна:
✓ Архитектура проекта           + Аутентификация (JWT)
✓ ORM-модели и миграции         + Авторизация (права доступа)
✓ Pydantic-валидация            + Пагинация
✓ Dependency Injection          + Тесты
✓ Обработка ошибок              + Переменные окружения
✓ REST API со всеми методами    + Хранение дат на сервере
✓ Связи между сущностями        + Email-валидация
✓ Документация /docs            + Логирование и мониторинг
```

Фундамент правильный. Всё перечисленное — это надстройки над уже готовой архитектурой, а не переписывание с нуля.

---

## Часть 3. Что вы теперь умеете

Посмотрите на этот список не как на пройденные уроки, а как на реальные навыки:

**SQL и базы данных:**
- Проектировать реляционные схемы с нормализацией
- Писать запросы любой сложности: JOIN, GROUP BY, подзапросы, CTE
- Понимать транзакции и целостность данных
- Работать с SQLite и знать где он ограничен

**Python + данные:**
- Управлять базой из кода через sqlite3 и SQLAlchemy
- Строить слой данных через паттерн Repository
- Генерировать реалистичные данные через Faker
- Управлять схемой через миграции Alembic

**Бэкенд-разработка:**
- Понимать HTTP, REST и JSON на уровне протокола
- Строить API на FastAPI с правильной архитектурой
- Валидировать данные через Pydantic
- Изолировать зависимости через DI
- Обрабатывать ошибки единообразно

**Архитектура:**
- Разделять слои: роутеры, схемы, репозитории, модели
- Понимать зачем каждый слой нужен и что он скрывает
- Читать и писать код который поддерживаем

---

## Вопросы для закрепления

**1.** Назовите три вещи которые нужно добавить в Blog API чтобы он был готов к реальному использованию.

<details>
<summary>Ответ</summary>

Любые три из: аутентификация (JWT), авторизация (права доступа), пагинация результатов, автоматические тесты, переменные окружения для секретов, валидация email через `EmailStr`, хранение дат на сервере, структурированное логирование.
</details>

---

**2.** Почему `author_id` передаётся в теле запроса в учебном проекте, и как это решается в продакшне?

<details>
<summary>Ответ</summary>

В учебном проекте нет аутентификации — мы не знаем кто делает запрос. В продакшне пользователь логинится и получает JWT-токен. При каждом запросе токен передаётся в заголовке `Authorization`. Сервер проверяет токен, извлекает из него `user_id` и передаёт в эндпоинт через зависимость `Depends(get_current_user)`. Клиент `author_id` не передаёт — только токен.
</details>

---

**3.** Что такое курсорная пагинация и чем она лучше offset-пагинации при больших объёмах?

<details>
<summary>Ответ</summary>

Offset-пагинация (`LIMIT 20 OFFSET 1000`) работает медленнее при большом `OFFSET` — база данных всё равно читает 1020 строк и выбрасывает первые 1000. Курсорная пагинация использует значение последней записи как "курсор" — `WHERE id > 1000 LIMIT 20`. База читает ровно 20 строк независимо от глубины страницы. На миллионах записей разница ощутима.
</details>

---

**4.** Почему `DATABASE_URL` не должен быть прописан прямо в коде и как это исправить?

<details>
<summary>Ответ</summary>

URL базы данных содержит логин и пароль — это секрет. Если такой файл попадёт в Git (публичный или корпоративный) — секреты станут известны всем. Правильное решение: хранить чувствительные данные в переменных окружения или в `.env` файле, который добавлен в `.gitignore`. В коде читаем через `os.getenv('DATABASE_URL')` или через `pydantic-settings`.
</details>

---

**5.** Что означает "фундамент правильный" применительно к нашему Blog API?

<details>
<summary>Ответ</summary>

Архитектурные решения — разделение на слои, паттерн Repository, Dependency Injection, единый формат ошибок, ORM + миграции — сделаны правильно и не нуждаются в переписывании при добавлении продакшн-возможностей. Аутентификация добавляется как новая зависимость, пагинация — как дополнительные query-параметры, тесты — поверх уже правильно изолированных слоёв. Правильный фундамент означает что улучшения добавляются, а не заменяют существующее.
</details>

---

**6.** Почему понимание SQLAlchemy ORM помогает при изучении Django ORM?

<details>
<summary>Ответ</summary>

Концепции идентичны: модели-классы вместо таблиц, объекты вместо строк, `session.query()` ≈ `Model.objects`, `.filter()` = `.filter()`, `relationship` ≈ `ForeignKey + related_name`, Alembic-миграции ≈ Django-миграции. Django делает то же самое автоматически и с большим количеством встроенных инструментов — но понимание того что происходит под капотом приходит из SQLAlchemy.
</details>

---

**7.** Что такое `from_attributes=True` в Pydantic и зачем это нужно при работе с ORM?

<details>
<summary>Ответ</summary>

По умолчанию Pydantic создаёт объекты только из словарей. ORM-объекты — это экземпляры Python-классов с атрибутами, не словари. `from_attributes=True` (в Pydantic v2) говорит схеме читать данные через атрибуты объекта (`obj.name`) а не через ключи словаря (`data['name']`). Это позволяет передавать ORM-объекты напрямую в Pydantic-схемы без ручного преобразования через `dict()`.
</details>

---

**8.** Какой тип тестов написали бы вы для эндпоинта `POST /posts`?

<details>
<summary>Ответ</summary>

Минимальный набор: тест успешного создания поста (проверить `201`, проверить поля ответа), тест с несуществующим `author_id` (проверить `404`), тест с пустым `title` (проверить `422`), тест с несуществующими `tag_ids` (проверить корректное поведение), тест что созданный пост появляется в `GET /posts`. Для изоляции — использовать базу в памяти `sqlite:///:memory:` и откатывать транзакцию после каждого теста.
</details>

---

**9.** Почему каскадное удаление реализовано двумя способами одновременно — через ORM и через `ondelete='CASCADE'` в FK?

<details>
<summary>Ответ</summary>

Каскад ORM (`cascade='all, delete-orphan'`) работает только когда удаление инициируется через SQLAlchemy. `ondelete='CASCADE'` в базе данных работает всегда — при прямом SQL, при другом инструменте, при удалении через скрипт миграции. Совместное использование — двойная защита целостности данных на разных уровнях. Если убрать один из них — возможны утечки данных в определённых сценариях.
</details>

---

**10.** Что принципиально нового вы узнаете в курсе по Django, чего нет в этом курсе?

<details>
<summary>Ответ</summary>

Полноценная аутентификация и авторизация из коробки (сессии, токены, группы и разрешения). Административная панель — готовый интерфейс управления данными без написания кода. Встроенная работа с формами и шаблонизатор. Полный стек — от HTML до базы данных в одном фреймворке. Батарейки: email, кэширование, сигналы, management-команды. PostgreSQL как основная СУБД вместо SQLite.
</details>

---

[Предыдущий урок](lesson26.md) |