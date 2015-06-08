title: Работа с базой данных
--
# Работа с базой данных

Lapis использует
[БД PostgreSQL](http://www.postgresql.org/).
Планируется поддержка других БД.
Вместо API, который предоставляет Lapis,
можно напрямую использовать API OpenResty.

Каждый запрос выполняется асинхронно при помощи
[OpenResty cosocket
API](http://wiki.nginx.org/HttpLuaModule#ngx.socket.tcp).
Выполнение кода, породившего запрос к БД,
приостанавливается и продолжается автоматически,
поэтому нет необходимости в callback'ах.
Запросы можно писать последовательно,
как будто они выполняются синхронно.
Соединения с сервером объединяются в пул для
большей эффективности.

[*pgmoon*](https://github.com/leafo/pgmoon) -
драйвер PostgreSQL,
который использует Lapis. Достоинством этого драйвера является
возможность использовать его как асинхронно
(OpenResty cosocket),
так и синхронно (LuaSocket) с командной строки.

## Установление соединения

В секции `postgres` файла конфигурации
<span class="for_moon">`config.moon`</span>
<span class="for_lua">`config.lua`</span>.
задаются параметры подключения к БД:

```lua
-- config.lua
config("development", {
  postgres = {
    backend = "pgmoon",
    host = "127.0.0.1",
    user = "pg_user",
    password = "the_password",
    database = "my_database"
  }
})
```

```moon
-- config.moon
config "development", ->
  postgres ->
    backend "pgmoon"
    host "127.0.0.1"
    user "pg_user"
    password "the_password"
    database "my_database"
```

Значения по умолчанию: host `127.0.0.1`, user  `postgres`,
port `5432`.
Если параметры БД совпадают со значениями по умолчанию,
в файле конфигурации их можно пропустить.
Если используется нестандартный порт,
его надо приписать к адресу: `my_host:1234`.

Перейдём к запросам.

## Выполнение запросов

Есть два способа выполнять запросы:

1. Низкоуровневые функции, работающие с SQL.
1. Класс [`Model`](#models): объекты Lua
    синхронизуются с записями БД.

Лучше использовать класс `Model`.
Низкоуровневые функции предназначены для задач, с которыми
плохо справляются модели.

Пример использования низкоуровневых функций:

```lua
local lapis = require("lapis")
local db = require("lapis.db")

local app = lapis.Application()

app:match("/", function()
  local res = db.query("select * from my_table where id = ?", 10)
  return "ok!"
end)
```

```moon
lapis = require "lapis"
db = require "lapis.db"

class extends lapis.Application
  "/": =>
    res = db.query "select * from my_table where id = ?", 10
    "ok!"
```

Аналогичный код на моделях:

```lua
local lapis = require("lapis")
local Model = require("lapis.db.model").Model

local app = lapis.Application()

local MyTable = Model:extend("my_table")

app:match("/", function()
  local row = MyTable:find(10)
  return "ok!"
end)
```

```moon
lapis = require "lapis"
import Model from require "lapis.db.model"

class MyTable extends Model

class extends lapis.Application
  "/": =>
    row = MyTable\find 10
    "ok!"
```

По умолчанию, все запросы печатаются в лог Nginx уровня notice.

## Query Interface

```lua
local db = require("lapis.db")
```

```moon
db = require "lapis.db"
```

В модуле `db` есть следующие функции:

### `query(query, params...)`

Выполняет запрос и в случае успеха
возвращает список с результатами запроса,
иначе возвращает nil.

Первым аргументом передаётся текст запроса.
Если запрос содержит вопросительные знаки (`?`),
они заменяются последующими аргументами в порядке
их появления в запросе.
Значения аргументов экранируются при помощи функции
`escape_literal`, что исключает SQL-инъекции.

```lua
local res

res = db.query("SELECT * FROM hello")
res = db.query("UPDATE things SET color = ?", "blue")
res = db.query("INSERT INTO cats (age, name, alive) VALUES (?, ?, ?)", 25, "dogman", true)
```

```moon
res = db.query "SELECT * FROM hello"
res = db.query "UPDATE things SET color = ?", "blue"
res = db.query "INSERT INTO cats (age, name, alive) VALUES (?, ?, ?)", 25, "dogman", true
```

```sql
SELECT * FROM hello
UPDATE things SET color = 'blue'
INSERT INTO cats (age, name, alive) VALUES (25, 'dogman', TRUE)
```

Если запрос завершается с ошибой,
то срабатывает ошибка Lua,
в которую включается сообщение об ошибке от PostgreSQL
и сам запрос.

### `select(query, params...)`

То же, что `query`, но добавляет в начало запроса `"SELECT"`.

```lua
local res = db.select("* from hello where active = ?", db.FALSE)
```

```moon
res = db.select "* from hello where active = ?", db.FALSE
```

```sql
SELECT * from hello where active = FALSE
```

### `insert(table, values, returning...)`

Добавляет запись в таблицу `table`. Аргумент `values` -
таблица Lua с именами полей и соответствующими значениями.

```lua
db.insert("my_table", {
  age = 10,
  name = "Hello World"
})
```


```moon
db.insert "my_table", {
  age: 10
  name: "Hello World"
}
```

```sql
INSERT INTO "my_table" ("age", "name") VALUES (10, 'Hello World')
```

Имена возвращаемых колонок можно указать после
аргумента `values`.

```lua
local res = db.insert("some_other_table", {
  name = "Hello World"
}, "id")
```

```moon
res = db.insert "some_other_table", {
  name: "Hello World"
}, "id"
```

```sql
INSERT INTO "some_other_table" ("name") VALUES ('Hello World') RETURNING "id"
```

### `update(table, values, conditions, params...)`

Обновляет в таблице `table` все записи, удовлетворяющие
критериям `conditions`, согласно таблице значений `values`.

```lua
db.update("the_table", {
  name = "Dogbert 2.0",
  active = true
}, {
  id = 100
})

```

```moon
db.update "the_table", {
  name: "Dogbert 2.0"
  active: true
}, {
  id: 100
}
```

```sql
UPDATE "the_table" SET "name" = 'Dogbert 2.0', "active" = TRUE WHERE "id" = 100
```

Аргумент `conditions` может быть строкой.
В таком случае в неё подставляются аргументы `params`:

```lua
db.update("the_table", {
  count = db.raw("count + 1")
}, "count < ?", 10)
```

```moon
db.update "the_table", {
  count: db.raw"count + 1"
}, "count < ?", 10
```

```sql
UPDATE "the_table" SET "count" = count + 1 WHERE count < 10
```

Если условия заданы в форме таблицы Lua, то все дополнительные
аргументы передаются в `RETURNING`:

```lua
db.update("cats", {
  count = db.raw("count + 1")
}, {
  id = 1200
}, "count")
```

```moon
db.update "cats", {
  count: db.raw "count + 1"
}, {
  id: 1200
}, "count"
```

```sql
UPDATE "cats" SET "count" = count + 1, WHERE "id" = 1200 RETURNING count
```

> `RETURNING` работает в PostgreSQL, но не в MySQL

### `delete(table, conditions, params...)`

Удаляет записи таблицы, удовлетворяющие условиям `conditions`.

```lua
db.delete("cats", { name: "Roo"})
```

```moon
db.delete "cats", name: "Roo"
```

```sql
DELETE FROM "cats" WHERE "name" = 'Roo'
```

Аргумент `conditions` также может быть строкой.

```moon
db.delete("cats", "name = ?", "Gato")
```

```moon
db.delete "cats", "name = ?", "Gato"
```

```sql
DELETE FROM "cats" WHERE name = 'Gato'
```

### `raw(str)`

Возвращает специальный объект, который подставляется в
запрос дословно, без экранирования:

```lua
db.update("the_table", {
  count = db.raw("count + 1")
})

db.select("* from another_table where x = ?", db.raw("now()"))
```

```moon
db.update "the_table", {
  count: db.raw"count + 1"
}

db.select "* from another_table where x = ?", db.raw"now()"
```

```sql
UPDATE "the_table" SET "count" = count + 1
SELECT * from another_table where x = now()
```

### `escape_literal(value)`

Экранирует значение для запроса.
Значение может быть любым типом, который хранится
в поле таблицы. Экранирует числа, строки и
логические переменные.

```lua
local escaped = db.escape_literal(value)
local res = db.query("select * from hello where id = " .. escaped")
```

```moon
escaped = db.escape_literal value
res = db.query "select * from hello where id = #{escaped}"
```

Не надо использовать `escape_literal` для экранирования
имён полей или таблиц.
Для этого есть `escape_identifier`.

### `escape_identifier(str)`

Экранирует идентификатор (имя поля или таблицы).

```lua
local table_name = db.escape_identifier("table")
local res = db.query("select * from " .. table_name)
```

```moon
table_name = db.escape_identifier "table"
res = db.query "select * from #{table_name}"
```

Не надо использовать `escape_identifier` для экранирования
значений. Экранируйте значения при помощи `escape_literal`.

### Константы

Доступны следующие константы:

 * `NULL` -- представляет `NULL` в SQL
 * `TRUE` -- представляет `TRUE` в SQL
 * `FALSE` -- представляет `FALSE` в SQL


```lua
db.update("the_table", {
  name = db.NULL
})
```

```moon
db.update "the_table", {
  name: db.NULL
}
```

## Структура БД

В модуле `lapis.db.schema` есть инструменты для
манипуляций со структурой БД.

### Создание и удаление таблиц

#### `create_table(table_name, { table_declarations... })`

Первый аргумент - имя таблицы.
Второй аргумент - список, описывающий таблицу.

```lua
local schema = require("lapis.db.schema")

local types = schema.types

schema.create_table("users", {
  {"id", types.serial},
  {"username", types.varchar},

  "PRIMARY KEY (id)"
})
```

```moon
schema = require "lapis.db.schema"

import create_table, types from schema

create_table "users", {
  {"id", types.serial}
  {"username", types.varchar}

  "PRIMARY KEY (id)"
}
```

Получился следующий SQL:

```sql
CREATE TABLE IF NOT EXISTS "users" (
  "id" serial NOT NULL,
  "username" character varying(255) NOT NULL,
  PRIMARY KEY (id)
);
```

Элементы списка, который передаётся вторым аргументом
в `create_table`, могут быть таблицами или строками.
Если элемент - таблица, то он воспринимается как
кортеж (имя поля, тип поля):

    { column_name, column_type }

Оба элемента кортежа - строки.
Имя поля экранируется автоматически.
Тип поля пропускается через `tostring` и
подставляется дословно.
Частые типы собраны в `schema.types`.
Например, `schema.types.varchar` превращается в
`character varying(255) NOT NULL`.
Подробную информацию о типах см. ниже.

Если на месте кортежа (поле, тип) стоит строка,
то она подставляется в `CREATE TABLE` дословно.
Мы пользовались этим выше, когда создавали
первичный ключ.

#### `drop_table(table_name)`

Удаляет таблицу.

```lua
schema.drop_table("users")
```

```moon
import drop_table from schema

drop_table "users"
```

```sql
DROP TABLE IF EXISTS "users";
```

### Индексы

#### `create_index(table_name, col1, col2..., [options])`

Функция `create_index` создаёт новый индекс таблицы.
Первый аргумент - имя таблицы.
Остальные аргументы - имена полей, на которых создаётся
индекс.
Последним аргументом можно передать таблицу с опциями.

Этот метод принимает две опции:
`unique: BOOL`, `where: условие`.

Если такой индекс уже есть, то ничего не происходит.

Примеры индексов:

```lua
local create_index = schema.create_index

create_index("users", "created_at")
create_index("users", "username", { unique = true })

create_index("posts", "category", "title")
create_index("uploads", "name", { where = "not deleted" })
```

```moon
import create_index from schema

create_index "users", "created_at"
create_index "users", "username", unique: true

create_index "posts", "category", "title"
create_index "uploads", "name", where: "not deleted"
```

Соответствующий SQL:

```sql
CREATE INDEX ON "users" (created_at);
CREATE UNIQUE INDEX ON "users" (username);
CREATE INDEX ON "posts" (category, title);
CREATE INDEX ON "uploads" (name) WHERE not deleted;
```

#### `drop_index(table_name, col1, col2...)`

Удаляет индекс. Имя индекса вычисляется из имени таблицы
и имён полей индекса.
Имена генерируются так же, как их по умолчанию
генерирует сам PostgreSQL.

```lua
local drop_index = schema.drop_index

drop_index("users", "created_at")
drop_index("posts", "title", "published")
```

```moon
import drop_index from schema

drop_index "users", "created_at"
drop_index "posts", "title", "published"
```

Соответствующий SQL:

```sql
DROP INDEX IF EXISTS "users_created_at_idx"
DROP INDEX IF EXISTS "posts_title_published_idx"
```

### Изменение структуры таблицы

#### `add_column(table_name, column_name, column_type)`

Добавляет новое поле в таблицу.

```lua
schema.add_column("users", "age", types.integer)
```

```moon
import add_column, types from schema

add_column "users", "age", types.integer
```

Соответствующий SQL:

```sql
ALTER TABLE "users" ADD COLUMN "age" integer NOT NULL DEFAULT 0
```

#### `drop_column(table_name, column_name)`

Удаляет поле из таблицы.

```lua
schema.drop_column("users", "age")
```

```moon
import drop_column from schema

drop_column "users", "age"
```

Соответствующий SQL:

```sql
ALTER TABLE "users" DROP COLUMN "age"
```

#### `rename_column(table_name, old_name, new_name)`

Переименование поля таблицы.

```lua
schema.rename_column("users", "age", "lifespan")
```

```moon
import rename_column from schema

rename_column "users", "age", "lifespan"
```

Соответствующий SQL:

```sql
ALTER TABLE "users" RENAME COLUMN "age" TO "lifespan"
```

#### `rename_table(old_name, new_name)`

Переименование таблицы.

```lua
schema.rename_table("users", "members")
```

```moon
import rename_table from schema

rename_table "users", "members"
```

Соответствующий SQL:

```sql
ALTER TABLE "users" RENAME TO "members"
```

### Типы колонок

Генераторы названий частых типов собраны в `schema.types`.
Генератор названия можно конвертировать в строку
при помощи `tostring` или вызвать как функцию
(последний вариант используется для модифицирования типа).

Значения по умолчанию:


```lua
local types = require("lapis.db.schema").types

types.boolean       --> boolean NOT NULL DEFAULT FALSE
types.date          --> date NOT NULL
types.double        --> double precision NOT NULL DEFAULT 0
types.foreign_key   --> integer NOT NULL
types.integer       --> integer NOT NULL DEFAULT 0
types.numeric       --> numeric NOT NULL DEFAULT 0
types.real          --> real NOT NULL DEFAULT 0
types.serial        --> serial NOT NULL
types.text          --> text NOT NULL
types.time          --> timestamp without time zone NOT NULL
types.varchar       --> character varying(255) NOT NULL
```

```moon
import types from require "lapis.db.schema"

types.boolean       --> boolean NOT NULL DEFAULT FALSE
types.date          --> date NOT NULL
types.double        --> double precision NOT NULL DEFAULT 0
types.foreign_key   --> integer NOT NULL
types.integer       --> integer NOT NULL DEFAULT 0
types.numeric       --> numeric NOT NULL DEFAULT 0
types.real          --> real NOT NULL DEFAULT 0
types.serial        --> serial NOT NULL
types.text          --> text NOT NULL
types.time          --> timestamp without time zone NOT NULL
types.varchar       --> character varying(255) NOT NULL
```

По умолчанию все типы помечены как `NOT NULL`,
значения числовых типов установлены в `0`,
а логического типа - в `false`.

При вызове типа как функции в него передаётся единственный
аргумент, таблица опций:

* `default: value` -- значение по умолчанию
* `null: boolean` -- является ли тип `NOT NULL`
* `unique: boolean` -- является ли поле уникальным ключом
* `primary_key: boolean` -- является ли поле первичным ключом

Примеры:

```lua
types.integer({ default = 1, null = true })  --> integer DEFAULT 1
types.integer({ primary_key = true })        --> integer NOT NULL DEFAULT 0 PRIMARY KEY
types.text({ null = true })                  --> text
types.varchar({ primary_key = true })        --> character varying(255) NOT NULL PRIMARY KEY
```

```moon
types.integer default: 1, null: true  --> integer DEFAULT 1
types.integer primary_key: true       --> integer NOT NULL DEFAULT 0 PRIMARY KEY
types.text null: true                 --> text
types.varchar primary_key: true       --> character varying(255) NOT NULL PRIMARY KEY
```

## Миграции БД

В течение жизни веб-приложения требования к нему
могут меняться, поэтому полезно иметь инструмент
для расширения структуры БД
(например, добавление полей, индексов).

Миграции - это таблица, переводящая имена миграций
в "мигрирующие" функции. Имена миграций могут быть
произвольными, однако принято использовать
метки времени Unix в качестве имён:

```lua
local schema = require("lapis.db.schema")

return {
  [1368686109] = function()
    schema.add_column("my_table", "hello", schema.types.integer)
  end,

  [1368686843] = function()
    schema.create_index("my_table", "hello")
  end
}
```

```moon
import add_column, create_index, types from require "lapis.db.schema"

{
  [1368686109]: =>
    add_column "my_table", "hello", types.integer

  [1368686843]: =>
    create_index "my_table", "hello"
}
```

Мигрирующая функция - это обычная функция.
В общем случае она может вызывать все функции,
описанные выше, но обычно это не нужно.

Мигрирующие функции запускаются только один раз.
Список имён применённых мигрирующих функций
хранится в таблице миграций.
Миграции запускаются в порядке возрастания их имён.


### Запуск миграций

У программы `lapis` есть специальный режим
для запуска миграций: `lapis migrate`.

Эта команда получает таблицу миграций
в вышеописанном формате из модуля `migrations`.

Пример модуля `migrations` с единственной миграцией:

```lua
-- migrations.lua

local schema = require("lapis.db.schema")
local types = schema.types

return {
  [1] = function()
    schema.create_table("articles", {
      { "id", types.serial },
      { "title", types.text },
      { "content", types.text },

      "PRIMARY KEY (id)"
    })
  end
}
```

```moon
-- migrations.moon

import create_table, types from require "lapis.db.schema"

{
  [1]: =>
    create_table "articles", {
      { "id", types.serial }
      { "title", types.text }
      { "content", types.text }

      "PRIMARY KEY (id)"
    }
}
```

Если пишете на MoonScript, не забудьте скомпилировать этот
файл в Lua.
Затем запустите `lapis migrate`.
Эта команда проверит наличие таблицы миграций
и создаст её, если её нет.
Затем она применит все миграции,
которые не применялись прежде.

См. подробнее про
[команду миграции](#command-line-interface-lapis-migrate).

### Запуск миграций вручную

Таблицу миграций можно создать вручную:

```lua
local migrations = require("lapis.db.migrations")
migrations.create_migrations_table()
```

```moon
migrations = require "lapis.db.migrations"
migrations.create_migrations_table!
```

Соответствующий SQL:

```sql
CREATE TABLE IF NOT EXISTS "lapis_migrations" (
  "name" character varying(255) NOT NULL,
  PRIMARY KEY(name)
);
```

Запускаем миграции:

```lua
local migrations = require("lapis.db.migrations")
migrations.run_migrations(require("migrations"))
```

```moon
import run_migrations from require "lapis.db.migrations"
run_migrations require "migrations"
```
