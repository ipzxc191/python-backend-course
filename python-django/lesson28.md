# Урок 28. Переход на PostgreSQL. Fixtures для переноса данных

## Почему SQLite не подходит для продакшна

Всё это время мы работали с SQLite — файловой базой данных, которая идеальна для разработки: не требует установки, не нужно настраивать подключение, вся база в одном файле. Но у SQLite есть принципиальные ограничения:

- **Конкурентная запись** — SQLite блокирует весь файл на запись. При нескольких одновременных запросах они выстраиваются в очередь. На продакшне с реальной нагрузкой это узкое место.
- **Нет сетевого доступа** — SQLite это файл на диске. Несколько серверов приложения не могут работать с одной базой одновременно.
- **Ограниченный SQL** — PostgreSQL поддерживает полнотекстовый поиск, JSON-поля, оконные функции и многое другое, чего нет в SQLite.

PostgreSQL — стандарт индустрии для Django-проектов на продакшне. Сегодня переводим проект на него и переносим данные.

---

## Часть 1. Fixtures — сохраняем данные из SQLite

Прежде чем менять базу данных, нужно сохранить всё, что уже есть в SQLite. Для этого Django предоставляет **fixtures** — механизм сериализации данных из БД в JSON (или XML/YAML), который затем можно загрузить в любую базу.

### dumpdata — выгрузка данных

```bash
python manage.py dumpdata --output=db_backup.json --indent=2
```

Это выгрузит **все** данные из всех таблиц. Но стандартные таблицы Django (сессии, типы контента, логи) при загрузке в новую БД могут создать конфликты. Лучше выгружать только нужные приложения:

```bash
# Выгрузить только данные приложения films
python manage.py dumpdata films --output=films_data.json --indent=2

# Выгрузить пользователей и группы
python manage.py dumpdata auth.user auth.group --output=users_data.json --indent=2
```

Посмотрим на структуру файла:

```json
[
  {
    "model": "films.genre",
    "pk": 1,
    "fields": {
      "name": "Драма",
      "slug": "drama"
    }
  },
  {
    "model": "films.film",
    "pk": 1,
    "fields": {
      "title": "Крёстный отец",
      "year": 1972,
      "slug": "krestnyj-otec",
      "rating": "9.2",
      "director": 1
    }
  }
]
```

Каждый объект описывается тремя ключами: `model` (приложение.модель), `pk` (первичный ключ), `fields` (значения полей).

### Выгрузка с исключением проблемных таблиц

```bash
python manage.py dumpdata \
    --exclude=contenttypes \
    --exclude=auth.permission \
    --exclude=sessions \
    --output=full_backup.json \
    --indent=2
```

`contenttypes` и `auth.permission` создаются автоматически при `migrate` на основе установленных приложений — загрузка их из фикстуры часто вызывает конфликты. Исключаем их и даём Django создать заново.

---

## Часть 2. Установка и настройка PostgreSQL

### Установка PostgreSQL

**macOS:**

```bash
brew install postgresql@16
brew services start postgresql@16
```

**Ubuntu/Debian:**

```bash
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

**Windows:** Скачать установщик с [postgresql.org/download/windows](https://www.postgresql.org/download/windows/) и запустить.

### Создание базы данных и пользователя

```bash
# Входим в PostgreSQL
sudo -u postgres psql          # Linux
psql postgres                  # macOS с Homebrew
```

```sql
-- Внутри psql
CREATE USER filmsite_user WITH PASSWORD 'ваш_надёжный_пароль';
CREATE DATABASE filmsite_db OWNER filmsite_user;
GRANT ALL PRIVILEGES ON DATABASE filmsite_db TO filmsite_user;
\q
```

### Установка psycopg2

`psycopg2` — это специальная библиотека, которая позволяет Python работать с базой данных PostgreSQL. Сам Django не умеет напрямую отправлять SQL-запросы в PostgreSQL.

Когда вы вызываете, например `Film.objects.all()`, Django строит SQL-запрос, а затем передаёт его библиотеке `psycopg2`. Именно она устанавливает соединение с PostgreSQL, отправляет запрос серверу базы данных, получает результат и возвращает его обратно Django.

Именно поэтому `psycopg2` часто называют **адаптером** — он выступает посредником между Python и PostgreSQL.

Существует две основные версии этой библиотеки.

- `psycopg2-binary`: библиотека устанавливается сразу и не требует дополнительной настройки системы.
- `psycopg2`: этот пакет содержит исходный код библиотеки. Во время установки он компилируется непосредственно на вашем компьютере.

Предварительно собранные бинарные файлы (`psycopg2-binary`) отлично подходят для разработки: они устанавливаются буквально одной командой и позволяют сразу начать работу.

Однако на боевых серверах чаще используют обычный `psycopg2`, который собирается непосредственно под конкретную операционную систему и установленную версию PostgreSQL. Такой вариант считается более надёжным и рекомендуется разработчиками библиотеки для production-окружения.

**Во время обучения мы будем использовать:**

```bash
pip install psycopg2-binary
```

---

## Часть 3. Настройка Django

### Обновляем settings.py

```python
# filmsite/settings.py

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'filmsite_db',
        'USER': 'filmsite_user',
        'PASSWORD': 'ваш_надёжный_пароль',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}
```

| Параметр | Значение | Назначение |
|---|---|---|
| `ENGINE` | `postgresql` | Использовать PostgreSQL-бэкенд |
| `NAME` | имя БД | База данных, созданная выше |
| `USER` | пользователь БД | Пользователь PostgreSQL |
| `PASSWORD` | пароль | Пароль пользователя PostgreSQL |
| `HOST` | `localhost` | Сервер БД (на продакшне — IP или hostname) |
| `PORT` | `5432` | Стандартный порт PostgreSQL |

### Учётные данные в переменных окружения

Как и с SMTP в уроке 25 — не храним учётные данные прямо в `settings.py`:

```python
# filmsite/settings.py
import os

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME', 'filmsite_db'),
        'USER': os.environ.get('DB_USER', 'filmsite_user'),
        'PASSWORD': os.environ.get('DB_PASSWORD', ''),
        'HOST': os.environ.get('DB_HOST', 'localhost'),
        'PORT': os.environ.get('DB_PORT', '5432'),
    }
}
```

### Применяем миграции к новой базе

PostgreSQL подключён, но таблиц в ней ещё нет:

```bash
python manage.py migrate
```

Django создаст все таблицы заново по существующим файлам миграций. База чистая — данных нет.

---

## Часть 4. Загружаем данные через loaddata

```bash
# Загружаем пользователей
python manage.py loaddata users_data.json

# Загружаем данные приложения
python manage.py loaddata films_data.json
```

`loaddata` читает JSON-файл и создаёт объекты в базе данных. Порядок загрузки имеет значение: если в `films_data.json` есть записи `Film` со ссылками на `Director` по `pk`, то `Director` должен быть загружен раньше. Если выгрузить все данные приложения `films` одной командой — Django сам сохранит правильный порядок.

### Проверка загрузки

```bash
python manage.py shell
```

```python
from films.models import Film, Genre, Director

print(f'Фильмов: {Film.objects.count()}')
print(f'Жанров: {Genre.objects.count()}')
print(f'Режиссёров: {Director.objects.count()}')
```

---

## Fixtures как тестовые данные

Fixtures полезны не только для миграции — это стандартный способ хранить начальные данные для тестов и для быстрого наполнения базы при разворачивании на новом сервере.

```bash
# Создать фикстуру с тестовыми жанрами
python manage.py dumpdata films.genre \
    --output=films/fixtures/genres.json \
    --indent=2
```

Django ищет fixtures в папке `fixtures/` каждого приложения — если там лежат файлы, их можно загрузить без указания пути:

```bash
python manage.py loaddata genres.json
```

Это удобно для автоматического заполнения базы при разворачивании проекта: добавить вызов `loaddata` в скрипт деплоя или `Makefile`.

---

## PostgreSQL vs SQLite — что изменится в коде

Хорошая новость: практически ничего. Django ORM абстрагирует различия между базами данных. QuerySet'ы, фильтры, аннотации — всё работает одинаково. Несколько отличий, которые стоит знать:

**Чувствительность к регистру.** SQLite нечувствителен к регистру в строках. PostgreSQL — чувствителен. Если где-то использовался `filter(title='крёстный отец')` вместо `filter(title__iexact='крёстный отец')` — в PostgreSQL это даст пустой результат. Везде в нашем коде мы использовали `icontains` и `iexact` — это корректная практика, которая работает одинаково в обеих базах.

**Строгие типы.** PostgreSQL строго проверяет типы: нельзя вставить строку в числовое поле. SQLite был более снисходителен. Если в данных были «грязные» значения — PostgreSQL может отказать при загрузке.

**Полнотекстовый поиск.** PostgreSQL поддерживает `SearchVector` и `SearchQuery` из `django.contrib.postgres.search` — это настоящий полнотекстовый поиск с рангированием, а не просто `LIKE '%...%'`. Для нашего каталога это было бы заметное улучшение, но выходит за рамки базового курса.

---

## Подводные камни

### Конфликт первичных ключей при loaddata

Если в PostgreSQL уже есть данные (например, суперпользователь из `createsuperuser`) и загружается фикстура с теми же `pk` — возникнет конфликт:

```
django.db.utils.IntegrityError: duplicate key value violates unique constraint
```

Решение: загружать данные в абсолютно чистую базу (после `migrate`, до создания суперпользователя), либо исключать конфликтующие модели из фикстуры.

### Последовательности (sequences) в PostgreSQL

PostgreSQL использует sequences для автоинкремента `id`. После загрузки данных через `loaddata` sequence может не знать о максимальном `pk` из фикстуры и начать новые записи с `id=1`, что вызовет конфликт при следующем сохранении. Исправляется командой:

```bash
python manage.py sqlsequencereset films | python manage.py dbshell
```

`sqlsequencereset` генерирует SQL для сброса последовательностей по текущим максимальным значениям `id` в таблицах.

### Файлы миграций и PostgreSQL

Иногда старые миграции содержат конструкции, специфичные для SQLite. PostgreSQL строже к типам и может отказать при их применении. Это редкий случай для проектов, которые изначально проектировались корректно (как наш), но важно иметь это в виду при работе с чужим кодом.

---

## Вопросы для проверки

1. Что такое fixture в Django и для каких задач он используется?
2. Почему при выгрузке данных рекомендуется исключать `contenttypes` и `auth.permission`?
3. Зачем нужен `psycopg2` и чем отличается `psycopg2-binary` от `psycopg2`?
4. Почему после `loaddata` в PostgreSQL иногда нужно запускать `sqlsequencereset`?
5. Что изменится в поведении `filter()` при переходе с SQLite на PostgreSQL?

---

## Практическая задача

**Тип: расширь проект**

Выполни полный цикл переноса данных с SQLite на PostgreSQL.

**Шаги:**

1. Выгрузи данные из SQLite: `films_data.json` (приложение `films`) и `users_data.json` (`auth.user`, `auth.group`). При выгрузке исключи `contenttypes` и `auth.permission`
2. Установи PostgreSQL и `psycopg2-binary`, создай базу данных `filmsite_db` и пользователя `filmsite_user`
3. Обнови `DATABASES` в `settings.py` — вынеси учётные данные в переменные окружения
4. Примени миграции командой `migrate`
5. Загрузи данные через `loaddata` — сначала `films_data.json`, потом `users_data.json`
6. Запусти сервер и проверь, что каталог фильмов отображается корректно
7. Если появились ошибки последовательностей — запусти `sqlsequencereset`

---

<details>
<summary><b>Шпаргалка по работе с PostgreSQL через терминал</b></summary>

## Подключение к PostgreSQL

Подключаемся под пользователем `postgres`:

```bash
psql -U postgres
```

После успешного подключения приглашение терминала изменится:

```text
postgres=#
```

---

## Создание пользователя

```sql
CREATE USER filmsite_user WITH PASSWORD 'MyStrongPassword123!';
```

---

## Создание базы данных

```sql
CREATE DATABASE filmsite_db OWNER filmsite_user;
```

---

## Выдача прав пользователю

```sql
GRANT ALL PRIVILEGES ON DATABASE filmsite_db TO filmsite_user;
```

---

## Посмотреть список пользователей

```sql
\du
```

Пример:

```text
 Role name      | Attributes
----------------+-------------------------------------
 postgres       | Superuser, Create role, Create DB
 filmsite_user  |
```

---

## Посмотреть список баз данных

```sql
\l
```

или

```sql
\list
```

---

## Подключиться к базе данных

Если вы уже находитесь внутри PostgreSQL:

```sql
\c filmsite_db filmsite_user
```

или

```sql
\connect filmsite_db filmsite_user
```

После подключения приглашение изменится:

```text
filmsite_db=>
```

Теперь все запросы будут выполняться именно в этой базе данных.

---

## Посмотреть список таблиц

```sql
\dt
```

Если таблиц много:

```sql
\dt *
```

---

## Посмотреть структуру таблицы

Например:

```sql
\d users_customuser
```

или

```sql
\d auth_user
```

Будут показаны:

- поля;
- типы данных;
- ограничения;
- индексы;
- внешние ключи.

---

## Посмотреть содержимое таблицы

Например:

```sql
SELECT * FROM auth_user;
```

или

```sql
SELECT * FROM users_customuser;
```

---

## Ограничить количество строк

```sql
SELECT * FROM auth_user
LIMIT 10;
```

---

## Вывести только некоторые столбцы

```sql
SELECT id, username, email
FROM auth_user;
```

---

## Отфильтровать записи

Например:

```sql
SELECT *
FROM auth_user
WHERE id = 1;
```

или

```sql
SELECT *
FROM auth_user
WHERE username = 'admin';
```

---

## Отсортировать результат

```sql
SELECT *
FROM auth_user
ORDER BY id DESC;
```

---

## Посчитать количество записей

```sql
SELECT COUNT(*)
FROM auth_user;
```

---

## Удалить запись

Например:

```sql
DELETE FROM auth_user
WHERE id = 5;
```

> ⚠️ Будьте осторожны! После выполнения запроса запись будет удалена.

---

## Очистить всю таблицу

```sql
TRUNCATE TABLE auth_user;
```

Если таблица связана внешними ключами:

```sql
TRUNCATE TABLE auth_user RESTART IDENTITY CASCADE;
```

- `RESTART IDENTITY` — сбрасывает счётчики `id`;
- `CASCADE` — очищает связанные таблицы.

---

## Посмотреть текущего пользователя

```sql
SELECT current_user;
```

---

## Посмотреть текущую базу данных

```sql
SELECT current_database();
```

---

## Посмотреть все схемы

```sql
\dn
```

---

## Посмотреть все последовательности (Sequences)

```sql
\ds
```

---

## Выполнить SQL из файла

Например:

```sql
\i backup.sql
```

---

## Выйти из текущей базы

Вернуться в базу `postgres`:

```sql
\c postgres
```

---

## Выйти из PostgreSQL

```sql
\q
```

---

## Полезные команды psql

| Команда | Назначение |
|----------|------------|
| `\l` | Список баз данных |
| `\c имя_базы` | Подключиться к базе |
| `\dt` | Список таблиц |
| `\d таблица` | Структура таблицы |
| `\du` | Список пользователей |
| `\dn` | Список схем |
| `\ds` | Список последовательностей |
| `\q` | Выход из PostgreSQL |

---

## Подключение напрямую под пользователем базы данных

Можно подключиться сразу к нужной базе, не заходя под `postgres`:

```bash
psql -U filmsite_user -d filmsite_db
```

После ввода пароля откроется соединение:

```text
filmsite_db=>
```

Это наиболее распространённый способ работы с PostgreSQL в повседневной разработке.

</details>

---

[Предыдущий урок](lesson27.md) | [Следующий урок](lesson29.md)