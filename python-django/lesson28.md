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

*Если вы работаете на Windows и при загрузке фикстуры возникают ошибки кодировки, используйте следующую команду:*

```bash
python -Xutf8 manage.py dumpdata --output=db_backup.json --indent=2
```

Флаг `-Xutf8` запускает Python в режиме UTF-8 независимо от региональных настроек Windows.

---

```bash
# Выгрузить только данные приложения films
python manage.py dumpdata films --output=films_data.json --indent=2
# для Windows
python -Xutf8 manage.py dumpdata films --output=films_data.json --indent=2

# Выгрузить пользователей и группы
python manage.py dumpdata auth.user auth.group --output=users_data.json --indent=2
# для Windows
python -Xutf8 manage.py dumpdata auth.user auth.group --output=users_data.json --indent=2
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

**для Windows PowerShell:**

```bash
python -Xutf8 manage.py dumpdata `
    --exclude=contenttypes `
    --exclude=auth.permission `
    --exclude=sessions `
    --output=full_backup.json `
    --indent=2
```

`contenttypes` и `auth.permission` создаются автоматически при `migrate` на основе установленных приложений — загрузка их из фикстуры часто вызывает конфликты. Исключаем их и даём Django создать заново.

---

## Часть 2. Установка и настройка PostgreSQL

### Установка PostgreSQL

Сначала необходимо установить сам PostgreSQL. Способ установки зависит от вашей операционной системы.

**macOS:**

Если вы используете Homebrew, установите PostgreSQL следующей командой:

```bash
brew install postgresql@16
```

После установки запустите PostgreSQL как фоновый сервис:

```bash
brew services start postgresql@16
```

После этого PostgreSQL будет запущен, и с ним можно будет работать через терминал.

---

**Ubuntu/Debian:**

Установите PostgreSQL:

```bash
sudo apt install postgresql postgresql-contrib
```

Запустите сервер PostgreSQL:

```bash
sudo systemctl start postgresql
```

Чтобы PostgreSQL автоматически запускался вместе с операционной системой:

```bash
sudo systemctl enable postgresql
```

---

**Windows:**

Скачайте установщик PostgreSQL с официального сайта: [https://www.postgresql.org/download/windows/](https://www.postgresql.org/download/windows/). Запустите установщик и пройдите стандартную процедуру установки.

Во время установки вам будет предложено задать **пароль пользователя `postgres`**. Обязательно запомните этот пароль — он понадобится для подключения к PostgreSQL с правами администратора.

После установки PostgreSQL в Windows обычно уже запущен как системная служба, поэтому вручную запускать сервер, как на Linux или macOS, чаще всего не требуется.

После установки нам необходимо настроить переменную среды `PATH`. Это позволит запускать команду `psql` из PowerShell или командной строки Windows, находясь в любой директории.

---

### Добавление PostgreSQL в PATH на Windows

Во время установки PostgreSQL исполняемые файлы PostgreSQL помещаются в папку `bin`. Обычно она находится примерно здесь:

```text
C:\Program Files\PostgreSQL\16\bin
```

Номер версии может отличаться (16, 17, 18).

Далее нужно добавить эту папку в `PATH`.

Выполните следующие действия:

**Шаг 1.**

Откройте меню «Пуск» и введите в поиске `Переменные среды`. 

Выберите пункт: **«Изменение системных переменных среды»**. 

Откроется окно **«Свойства системы»**.

---

**Шаг 2.**

Нажмите кнопку: **«Переменные среды...»**

---

**Шаг 3.**

В верхней части окна найдите раздел **«Переменные среды пользователя»**.

Выберите переменную `Path` и нажмите: **«Изменить...»**

---

**Шаг 4.**

Нажмите: **«Создать»**

и добавьте путь к папке `bin` (прим. `C:\Program Files\PostgreSQL\16\bin`) вашей установки PostgreSQL.

Подтвердите изменения кнопками **«ОК»**.

---

Если PowerShell был открыт во время изменения `PATH`, закройте его и откройте заново. Теперь команда `psql` должна быть доступна из любой директории. Проверить это можно следующей командой:

```powershell
psql --version
```

Если всё настроено правильно, вы увидите установленную версию PostgreSQL, например: `psql (PostgreSQL) 16.x`.

Теперь вы можете запускать `psql` непосредственно из PowerShell, находясь в любой директории.

---

### Создание базы данных и пользователя

После установки PostgreSQL нам необходимо создать отдельного пользователя и базу данных для нашего Django-проекта. В PostgreSQL существует стандартный административный пользователь `postgres`. Сначала необходимо подключиться к PostgreSQL от его имени. Способ подключения зависит от операционной системы.

---

**macOS с Homebrew:**

Откройте терминал и выполните:

```bash
psql postgres
```

Если команда выполняется успешно, вы попадёте в интерактивную консоль PostgreSQL:

```text
postgres=#
```

---

**Ubuntu/Debian:**

Откройте терминал и выполните:

```bash
sudo -u postgres psql
```

После успешного подключения вы также увидите:

```text
postgres=#
```

---

**Windows через PowerShell:**

Благодаря тому, что мы добавили PostgreSQL в `PATH`, теперь можно открыть PowerShell **в любой директории** и выполнить:

```powershell
psql -U postgres -d postgres
```

После этого PostgreSQL попросит ввести пароль:

```text
Password for user postgres:
```

Введите пароль пользователя `postgres`, который был задан во время установки PostgreSQL. Если подключение прошло успешно, появится приглашение:

```text
postgres=#
```

> **Важно:** при вводе пароля символы не отображаются на экране. Это нормальное поведение. Просто введите пароль и нажмите `Enter`.

---

### Создание пользователя и базы данных для Django

Теперь мы находимся внутри консоли PostgreSQL. Это можно определить по приглашению:

```text
postgres=#
```

Все следующие команды необходимо выполнять **внутри `psql`**, а не в PowerShell, CMD или обычном терминале.

Создадим отдельного пользователя для нашего Django-проекта:

```sql
CREATE USER filmsite_user WITH PASSWORD 'ваш_надёжный_пароль';
```

Создадим базу данных и сразу назначим `filmsite_user` её владельцем:

```sql
CREATE DATABASE filmsite_db OWNER filmsite_user;
```

Теперь подключимся к созданной базе данных:

```sql
\c filmsite_db
```

После выполнения этой команды приглашение должно измениться примерно на:

```text
filmsite_db=#
```

Теперь необходимо предоставить пользователю права на схему `public`.

```sql
GRANT ALL ON SCHEMA public TO filmsite_user;
```

Эта команда важна для Django. Во время выполнения миграций Django должен создавать и изменять таблицы, индексы и другие объекты базы данных. Для этого пользователь, под которым Django подключается к PostgreSQL, должен иметь необходимые права на схему, в которой будут находиться эти объекты.

Таким образом, Django сможет выполнять стандартные операции с базой данных, необходимые для работы проекта.

После этого выйдите из консоли PostgreSQL:

```sql
\q
```

**Теперь у нас есть**:

* пользователь PostgreSQL: `filmsite_user`;
* база данных: `filmsite_db`;
* пароль: тот, который был указан при создании пользователя;
* пользователь `filmsite_user` является владельцем базы данных;
* пользователь `filmsite_user` имеет необходимые права на схему `public`.

После настройки подключения Django сможет выполнить миграции и создать необходимые таблицы проекта.

---

## Часть 3. Настройка Django

### Установка psycopg2

`psycopg2` — это специальная библиотека, которая позволяет Python работать с базой данных PostgreSQL. Сам Django не умеет напрямую отправлять SQL-запросы в PostgreSQL.

Когда вы вызываете, например `Film.objects.all()`, Django строит SQL-запрос, а затем передаёт его библиотеке `psycopg2`. Она устанавливает соединение с PostgreSQL, отправляет запрос серверу базы данных, получает результат и возвращает его обратно Django.

`psycopg2` часто называют **адаптером** — он выступает посредником между Python и PostgreSQL.

Существует две основные версии этой библиотеки:

* `psycopg2-binary` — библиотека устанавливается сразу и не требует дополнительной настройки системы;
* `psycopg2` — этот пакет содержит исходный код библиотеки. Во время установки он компилируется непосредственно на вашем компьютере.

Предварительно собранные бинарные файлы (`psycopg2-binary`) отлично подходят для разработки: они устанавливаются буквально одной командой и позволяют сразу начать работу.

Однако на боевых серверах чаще используют обычный `psycopg2`, который собирается непосредственно под конкретную операционную систему и установленную версию PostgreSQL. Такой вариант считается более надёжным и рекомендуется разработчиками библиотеки для production-окружения.

**Во время обучения мы будем использовать:**

```bash
pip install psycopg2-binary
```

---

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

**для Windows PowerShell:**

```bash
# Создать фикстуру с тестовыми жанрами
python -Xutf8 manage.py dumpdata films.genre `
    --output=films/fixtures/genres.json `
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

## Полностью очистить PostgreSQL

Если необходимо полностью начать работу заново, можно удалить все созданные базы данных и пользователей, оставив только стандартного пользователя `postgres`.

> ⚠️ **Внимание!** Все данные будут безвозвратно удалены.

**Подключиться под пользователем `postgres`**

```bash
psql -U postgres
```

**Посмотреть существующие базы данных**:

```sql
\l
```

Например:

```text
postgres
filmsite_db
shop_db
test_db
```

*`postgres` — это системная база данных. Её удалять не нужно.*

**Удалить созданные базы данных**:

Если к базе подключены пользователи, сначала необходимо разорвать все активные соединения:

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'filmsite_db'
  AND pid <> pg_backend_pid();
```

После этого можно удалить базу:

```sql
DROP DATABASE filmsite_db;
```

Аналогично удаляются остальные базы данных:

```sql
DROP DATABASE shop_db;
DROP DATABASE test_db;
```

**Посмотреть существующих пользователей**:

```sql
\du
```

Например:

```text
postgres
filmsite_user
shop_user
test_user
```

**Удалить созданных пользователей**:

```sql
DROP ROLE filmsite_user;
DROP ROLE shop_user;
DROP ROLE test_user;
```

или

```sql
DROP USER filmsite_user;
DROP USER shop_user;
DROP USER test_user;
```

> `DROP ROLE` и `DROP USER` в PostgreSQL являются эквивалентными командами.

*Теперь PostgreSQL находится практически в исходном состоянии и можно заново создавать пользователей, базы данных и выполнять миграции Django.*

</details>

---

[Предыдущий урок](lesson27.md) | [Следующий урок](lesson29.md)