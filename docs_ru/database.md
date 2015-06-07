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

## Модели

В Lapis есть базовый класс `Model`, от которого наследуются
классы-модели. Объекты классов-моделей (обычные таблицы Lua)
синхронизируются с записями БД.
Класс соответствует таблице БД, а объект класса -
записи БД.

Простейшая модель - пустая.


```lua
local Model = require("lapis.db.model").Model

local Users = Model:extend("users")
```

```moon
import Model from require "lapis.db.model"

class Users extends Model
```

<p class="for_lua">
Первый аргумент функции <code>extend</code> -
имя таблицы, которой соответствует данная модель.
</p>

<p class="for_moon">
Имя таблицы выводится из имени класса.
В данном случае из имени класса <code>Users</code>
вывели имя таблицы <code>users</code>.
Класс <code>HelloWorlds</code> соответствовал бы
таблице <code>hello_worlds</code>.
Обычно слово в имени класса берут во множественном числе.
</p>

<p class="for_moon">
Чтобы использовать другое имя таблицы,
в классе модели
надо переопределить статический метод <code>@table_name</code>.
</p>

```moon
class Users extends Model
  @table_name: => "active_users"
```

### Первичные ключи

По умолчанию, все модели имеют первичный ключ "id".
Чтобы поменять первичный ключ, надо изменить
значение статического поля
<span class="for_moon">`@primary_key`</span>
<span class="for_lua">`self.primary_key`</span>
класса модели.


```lua
local Users = Model:extend("users", {
  primary_key = "login"
})
```

```moon
class Users extends Model
  @primary_key: "login"
```

Чтобы получить первичный ключ из нескольких полей,
надо указать список полей:

```lua
local Followings = Model:extend("followings", {
  primary_key = { "user_id", "followed_user_id" }
})
```

```moon
class Followings extends Model
  @primary_key: { "user_id", "followed_user_id" }
```

### Загрузка одной записи

Для примеров мы будем использовать
следующие модели:

```lua
local Model = require("lapis.db.model").Model

local Users = Model:extend("users")

local Tags = Model:extend("tags", {
  primary_key = {"user_id", "tag"}
})

```


```moon
import Model from require "lapis.db.model"

class Users extends Model

class Tags extends Model
  @primary_key: {"user_id", "tag"}
```

Чтобы запросить из БД одну запись, используется статический
метод `find` класса модели.
Первый вариант этого метода принимает
переменное число аргументов, по одному на каждое
поле первичного ключа, в порядке их объявления:

```lua
local user = Users:find(23232)
local tag = Tags:find(1234, "programmer")
```

```moon
user = Users\find 23232
tag = Tags\find 1234, "programmer"
```

```sql
SELECT * from "users" where "id" = 23232 limit 1
SELECT * from "tags" where "user_id" = 1234 and "tag" = 'programmer' limit 1
```

Метод `find` возвращает объект модели.
В случае пользователей, если бы было поле `name`,
то можно было получить имя пользователя при помощи
`user.name`.

В метод `find` можно передать таблицу.
Таблица превращается в условия запроса:


```lua
local user = Users:find({ email = "person@example.com" })
```

```moon
user = Users\find email: "person@example.com"
```

```sql
SELECT * from "users" where "email" = 'person@example.com' limit 1
```

### Загрузка нескольких записей таблицы

Чтобы получить из БД сразу несколько записей таблицы,
используется метод `select`.
Метод `select` аналогичен низкоуровневой функции `select`
(см. выше), но указывается только часть запроса
с условием.


```lua
local tags = Tags:select("where tag = ?", "merchant")
```

```moon
tags = Tags\select "where tag = ?", "merchant"
```

```sql
SELECT * from "tags" where tag = 'merchant'
```

Возвращает не один объект, а список объектов.
Чтобы загрузить только определённые поля,
укажите их в поле `fields` таблицы, передаваемой
последним аргументом:


```lua
local tags = Tags:select("where tag = ?", "merchant", { fields = "created_at as c" })
```

```moon
tags = Tags\select "where tag = ?", "merchant", fields: "created_at as c"
```

```sql
SELECT created_at as c from "tags" where tag = 'merchant'
```

Чтобы загрузить несколько записей по их первичным ключам,
воспользуйтесь методом `find_all`.
В него подаётся список первичных ключей.
Этот метод работает только с таблицей,
у которой один первичный ключ.

```lua
local users = Users:find_all({ 1,2,3,4,5 })
```

```moon
users = Users\find_all { 1,2,3,4,5 }
```

```sql
SELECT * from "users" where "id" in (1, 2, 3, 4, 5)
```

Чтобы фильтровать не по первичному ключу,
а по другому полю, это поле передаётся
вторым аргументом:


```lua
local users = UserProfile:find_all({ 1,2,3,4,5 }, "user_id")
```

```moon
users = UserProfile\find_all { 1,2,3,4,5 }, "user_id"
```

```sql
SELECT * from "UserProfile" where "user_id" in (1, 2, 3, 4, 5)
```

Кроме того, вторым аргументом можно передать
таблицу параметров запроса. Существуют следующие параметры:

* `key` -- Имя поля, по которому идёт поиск (то же,
    что просто указать это имя вторым аргументом)
* `fields` -- Список загружаемых полей через запятую
    (по умолчанию все: `*`)
* `where` -- Дополнительное условие

Пример:

```lua
local users = UserProfile:find_all({1,2,3,4}, {
  key = "user_id",
  fields = "user_id, twitter_account",
  where = {
    public = true
  }
})
```

```moon
users = UserProfile\find_all {1,2,3,4}, {
  key: "user_id"
  fields: "user_id, twitter_account"
  where: {
    public: true
  }
}
```

```sql
SELECT user_id, twitter_account from "things" where "user_id" in (1, 2, 3, 4) and "public" = TRUE
```

### Добавление записей

Новые записи добавляются в таблицу при помощи статического
метода `create` класса модели.
В метод `create` передаётся таблица ключ-значение,
переводящая поле таблицы в значение этого поля.
Возвращает созданный объект модели.
Значения первичных ключей (в том числе автоинкрементных)
записи добываются при помощи
ключевого слова PostgreSQL `RETURN`.

```lua
local user = Users:create({
  login = "superuser",
  password = "1234"
})
```

```moon
user = Users\create {
  login: "superuser"
  password: "1234"
}
```

```sql
INSERT INTO "users" ("password", "login") VALUES ('1234', 'superuser') RETURNING "id"
```

### Обновление записи

У объектов модели есть метод `update` для обновления
записи.
При обновлении запись однозначно идентифицируется по
первичным ключам.

Первый вариант `update` принимает переменное число аргументов:
список названий полей, значения которых надо перенести
из объекта модели в БД.

```lua
local user = Users:find(1)
user.login = "uberuser"
user.email = "admin@example.com"
user:update("login", "email")
```

```moon
user = Users\find 1
user.login = "uberuser"
user.email = "admin@example.com"

user\update "login", "email"
```

```sql
UPDATE "users" SET "login" = 'uberuser', "email" = 'admin@example.com' WHERE "id" = 1
```

Второй вариант `update` принимает таблицу,
переводящую названия полей в их новые значения.
Объект модели при этом тоже обновляется.
Перепишем предыдущий пример:

```lua
local user = Users:find(1)
user:update({
  login = "uberuser",
  email = "admin@example.com",
})
```

```moon
user = Users\find 1
user\update {
  login: "uberuser"
  email: "admin@example.com"
}
```

```sql
UPDATE "users" SET "login" = 'uberuser', "email" = 'admin@example.com' WHERE "id" = 1
```

> Список обновляемых полей можно передавать в форме
> таблицы-списка
> (ещё один способ вызвать первый вариант `update`)

### Удаление записи

Вызовите на объекте модели метод `delete`:

```lua
local user = Users:find(1)
user:delete()
```

```moon
user = Users\find 1
user\delete!
```

```sql
DELETE FROM "users" WHERE "id" = 1
```

### Отметки времени
Время создания и последнего изменения
часто хранят в моделях. Модели Lapis могут автоматизировать
управление такими полями.

Создавая таблицу, добавьте в неё следующие поля:


```sql
CREATE TABLE ... (
  ...
  "created_at" timestamp without time zone NOT NULL,
  "updated_at" timestamp without time zone NOT NULL
)
```

Установите значение
<span class="for_moon">статического поля `@timestamp`</span>
<span class="for_lua">свойства `@timestamp`</span>
в значение `true`:

```lua
local Users = Model:extend("users", {
  timestamp = true
})
```

```moon
class Users extends Model
  @timestamp: true
```

При вызове методов `create` и `update` соответствующее
поле обновляется автоматически.

Чтобы отключить обновление поля, передайте в `update`
последним аргументом таблицу
<span class="for_moon">`timestamp: false`</span>
<span class="for_lua">`timestamp = false`</span>:

```lua
local Users = Model:extend("users", {
  timestamp = true
})

local user = Users:find(1)

-- первый вариант
user:update({ name = "hello world" }, { timestamp = false })


-- второй вариант
user.name = "hello world"
user.age = 123
user:update("name", "age", { timestamp = false})
```

```moon
class Users extends Model
  @timestamp: true

user = Users\find 1

-- первый вариант
user\update { name: "hello world" }, { timestamp: false }

-- второй вариант
user.name = "hello world"
user.age = 123
user\update "name", "age", timestamp: false
```

### Предзагрузка связанных объектов

При использовании систем типа ActiveRecord часто
сталкиваются с проблемой множественных запросов
для загрузки связанных объектов
при обходе списка объектов.
Чтобы справиться с этой проблемой, надо загрузить данные
для наиболее широкого множества объектов
перед проходом по ним.

Нам понадобится парочка моделей, чтобы проиллюстрировать
эту ситуацию: (поля прописаны в комментариях над моделью.)

```lua
local Model = require("lapis.db.model").Model

-- таблица с полями: id, name
local Users = Model:extend("users")

-- таблица с полями: id, user_id, text_content
local Posts = Model:extend("posts")
```

```moon
import Model from require "lapis.db.model"

-- таблица с полями: id, name
class Users extends Model

-- таблица с полями: id, user_id, text_content
class Posts extends Model
```

Мы хотим для каждой статьи (post) получить
соответствующего ей пользователя.
Чтобы включить данные о пользователе в объект статьи,
используется статический метод `Users:include_in`:

```lua
local posts = Posts:select() -- загрузить все статьи
Users:include_in(posts, "user_id")

print(posts[1].user.name) -- имя пользователя первой статьи
```

```moon
posts = Posts\select! -- загрузить все статьи

Users\include_in posts, "user_id"

print posts[1].user.name -- имя пользователя первой статьи
```

```sql
SELECT * from "posts"
SELECT * from "users" where "id" in (1,2,3,4,5,6)
```

В объект статьи добавляется поле `user`, в котором
хранится объект модели `Users`.
Первым аргументом в метод `include_in` передаётся
список объектов модели.
Второй аргумент - имя внешнего ключа
(имя поля в объектах, переданных в первом аргументе).

Имя поля, в которое помещается загружаемый объект,
выводится из имени внешнего ключа.
В данном примере, имя поля `user` выводится
из `user_id` (имя внешнего ключа).
А можно задать этому полю своё имя:


```lua
Users:include_in(posts, "user_id", { as: "author" })
```

```moon
Users\include_in posts, "user_id", as: "author"
```

Теперь каждая статья содержит поле `author`,
в котором хранится объект модели `Users`.

В предыдущем примере мы подгружали объекты,
на которые ссылаются объекты из списка.
Бывают обратные случаи, когда нужно загрузить
объекты, ссылающиеся на
объекты из списка. Такое часто случается
в случае отношений "один-к-одному".

Рассмотрим модели, иллюстрирующие эту ситуацию:

```lua
local Model = require("lapis.db.model").Model

-- поля: id, name
local Users = Model:extend("users")

-- поля: user_id, twitter_account, facebook_username
local UserData = Model:extend("user_data")
```

```moon
import Model from require "lapis.db.model"

-- поля: id, name
class Users extends Model

-- поля: user_id, twitter_account, facebook_username
class UserData extends Model
```

Подгрузим `UserData` для каждого пользователя из списка:

```lua
local users = Users:select()
UserData:include_in(users, "user_id", { flip: true })

print(users[1].user_data.twitter_account)
```

```moon
users = Users\select!
UserData\include_in users, "user_id", flip: true

print users[1].user_data.twitter_account
```

```sql
SELECT * from "user_data" where "user_id" in (1,2,3,4,5,6)
```

Мы передали параметр `flip` (перевернуть)
со значением `true` в метод `include_in`.
Произошёл поиск объектов, несущих внешние ключи,
ссылающиеся на объекты из списка.

Кроме того, имя поля, в которое помещается подгруженный
объект, выводится из имени таблицы, объекты которой
подгружаются. В примере подгруженный объект помещается в
поле `user_data`.
(Если бы имя таблицы было во множественном числе,
то имя поля было в единственном.)

### Ограничения

Часто перед добавлением или обновлением записей
надо проверить выполнение определённых условий.
Допустим, объекты модели `Users` не должны
иметь имя "admin".

Этого можно добиться следующим образом:

```lua
local Model = require("lapis.db.model").Model

local Users = Model:extend("users", {
  constraints = {
    name = function(self, value)
      if value:lower() == "admin"
        return "User can not be named admin"
      end
    end
  }
})

assert(Users:create({
  name = "Admin"
}))

```

```moon
import Model from require "lapis.db.models"

class Users extends Model
  @constraints: {
    name: (value) =>
      if value\lower! == "admin"
        "User can not be named admin"
  }


assert Users\create {
  name: "Admin"
}
```

<span class="for_moon">Статическое поле `@constraints`</span>
<span class="for_lua">Свойство `constraints`</span> -
таблица, которая переводит поле в функцию,
которая проверяет, нарушено ли ограничение.
Если функция возвращает истинное значение,
то вставка или обновление записи срывается, а это
значение используется в качестве сообщения об ошибке.

В этом примере вызов `assert` сорвётся со следующим
сообщением об ошибке: `"User can not be named admin"`.

Проверяющая функция получает 4 аргумента: класс модели,
значение поля, имя поля и проверяемый объект.
В случае вставки новой записи в качестве этого объекта
выступает таблица, переданная в метод `create`.
В случае обновления передаётся объект модели.

### Пагинация

Запрос, возвращающий слишком много результатов,
можно разбить на страницы при помощи метода `paginated`.
Аргументы те же, что у `select`, но вместо самого результата
возвращает специальный объект `Paginator`.

Допустим, у нас есть следующие таблица и модель:
(как создавать таблицы. см
[следующий раздел](#database-schemas-creating-and-dropping-tables)):

```lua
create_table("users", {
  { "id", types.serial },
  { "name", types.varchar },
  { "group_id", types.foreign_key },

  "PRIMARY KEY(id)"
})

local Users = Model:extend("users")
```


```moon
create_table "users", {
  { "id", types.serial }
  { "name", types.varchar }
  { "group_id", types.foreign_key }

  "PRIMARY KEY(id)"
}

class Users extends Model

```

Разобьем результаты на страницы:

```lua
local paginated = Users:paginated("where group_id = ? order by name asc", 123)
```

```moon
paginated = Users\paginated [[where group_id = ? order by name asc]], 123
```

Чтобы настроить пагинацию,
надо передать таблицу опций последним аргументом.
Есть следующие опции:

`per_page`: устанавливает число объектов на странице

```moon
local paginated_alt = Users:paginated("where group_id = ?", 4, { per_page = 100 })
```

```moon
paginated_alt = Users\paginated [[where group_id = ?]], 4, per_page: 100
```

`prepare_results`: функция, через которую "пропускаются"
результаты метода `get_page` и `get_all` (см. ниже).
Полезно для включения дополнительной информации в пагинатор.
Функция `prepare_results` принимает список объектов
и возвращает список переработанных объектов:

```lua
local preloaded = Posts:paginated("where category = ?", "cats", {
  per_page = 10,
  prepare_results = function(posts)
    Users:include_in(posts, "user_id")
    return posts
  end
})
```

```moon
preloaded = Posts\paginated [[where category = ?]], "cats", {
  per_page: 10
  prepare_results: (posts) ->
    Users\include_in posts, "user_id"
    posts
}
```

Все дополнительные аргументы отправляются в `select`.
Например, можно передать опцию `fields`, чтобы указать
набор загружаемых полей.

У объекта-пагинатора есть следующие методы:

#### `get_all()`

Возвращает полный список всех возможных объектов.
То же, что вызов `select`.

```lua
local users = paginated:get_all()
```

```moon
users = paginated\get_all!
```

```sql
SELECT * from "users" where group_id = 123 order by name asc
```

#### `get_page(page_num)`

Возвращает список объектов со страницы с номером `page_num`,
номера начинаются с 1.
Число объектов на странице регулируется опцией `per_page`,
значение по умолчанию 10.


```lua
local page1 = paginated:get_page(1)
local page6 = paginated:get_page(6)
```

```moon
page1 = paginated\get_page 1
page6 = paginated\get_page 6
```

```sql
SELECT * from "users" where group_id = 123 order by name asc limit 10 offset 0
SELECT * from "users" where group_id = 123 order by name asc limit 10 offset 50
```

#### `num_pages()`

Возвращает число страниц.

#### `total_items()`

Возвращает общее число объектов.
Для этого из запроса удаляются все части, кроме `WHERE`,
добавляется `COUNT`.

```lua
local users = paginated:total_items()
```

```moon
users = paginated\total_items!
```

```sql
SELECT COUNT(*) as c from "users" where group_id = 123
```

#### `each_page(starting_page=1)`

Возвращает функцию-итератор, перебирающую объекты одной
страницы. С помощью этой функции можно обработать
много объектов, не загружая их в память одновременно.

```lua
for page_results, page_num in paginated:each_page() do
  print(page_results, page_num)
end
```

```moon
for page_results, page_num in paginated\each_page!
  print(page_results, page_num)
```

### Описание отношений моделей

Для описания отношений между моделями в классе модели
существует поле `relations`.

```lua
local Model = require("lapis.db.model").Model
local Posts = Model:extend("posts", {
  relations = {
    {"users", has_one = "Users"}
  }
})
```

```moon
import Model from require "lapis.db.models"
class Posts extends Model
  @relations: {
    {"user", has_one: "Users"}
  }
```

### Получение списка полей таблицы

Список полей таблицы можно получить с помощью
метода `columns`:

```lua
local Posts = Model:extend("posts")
for _, col in ipairs(Posts:columns()) do
  print(col.column_name, col.data_type)
end
```

```moon
class Posts extends Model

for {column_name, data_type} in Posts\columns!
  print column_name, data_type
```

```sql
SELECT column_name, data_type
  FROM information_schema.columns WHERE table_name = 'posts'
```

### Перезагрузка данных в объекте модели

Данные в объекте модели могут устаревать из-за
внешних изменений записи в БД.
Перечитать данные из БД можно при помощи
метода модели `refresh`.

```moon
class Posts extends Model
post = Posts\find 1
post\refresh!
```

```lua
local Posts = Model:extend("posts")
local post = Posts:find(1)
post:refresh()
```

```sql
SELECT * from "posts" where id = 1
```

По умолчанию перезагружаются все поля модели.
В аргументах можно указать набор полей:


```moon
class Posts extends Model
post = Posts\find 1
post\refresh "color", "height"
```

```lua
local Posts = Model:extend("posts")
local post = Posts:find(1)
post:refresh("color", "height")
```

```sql
SELECT "color", "height" from "posts" where id = 1
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
