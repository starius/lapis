# Модели

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

## Первичные ключи

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

## Статические функции класса модели

Статические функции класса модели используются для загрузки
объектов из БД, создания новых объектов и для получения
информации о таблице.

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

### `find(...)`

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

### `select(query, ...)`

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
Если в таблице нет объектов, удовлетворяющих запросу,
то возвращает пустую таблицу.

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

Содержимое опции `fields` подставляется в запрос буквально,
поэтому не нужно включать в неё строки, пришедшие из
недоверенных источников, чтобы не допустить SQL-инъекцию.
Используйте [`db.escape_identifier`][1] для
экранирования имён колонок.

Метод `select` возвращает объекты, принадлежащие той модели,
на которой его вызвали. Чтобы это изменить, укажите опцию
`load`. Если её значение равно `false`, то метод вернёт
объекты в форме простых таблиц Lua.

### `find_all(primary_keys)`

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
* `clause` -- Дополнительный код SQL, дописываемый к концу
    запроса или список аргументов, передаваемых в
    `db.interpolate_query`.

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

### `count(clause, ...)`

Считает объекты, удовлетворяющие условиям.

```lua
local total = Users:count()
local count = Users:count("username like '%' || ? || '%'", "leafo")
```

```moon
total = Users\count!
count = Users\count "username like '%' || ? || '%'", "leafo"
```

```sql
SELECT COUNT(*) "users"
SELECT COUNT(*) "users" where username like '%' || 'leafo' || '%'
```

### `create(values, create_opts=nil)`

Новые записи добавляются в таблицу при помощи статического
метода `create` класса модели.
В метод `create` передаётся таблица ключ-значение,
переводящая поле таблицы в значение этого поля.
Возвращает созданный объект модели.
Значения первичных ключей (в том числе автоинкрементных)
записи добываются при помощи
ключевого слова PostgreSQL `RETURNING`.

> В MySQL вместо `RETURNING` используется *last insert id*

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

Если одно из значений при создании объекта является
[`db.raw`][raw], то их значения будут загружены из БД с
помощью `RETURNING`. Изначальные значения будут заменены
значениями, загруженными из БД.

К примеру, создадим новый объект, у которого значение поля
`position` равно значению, следующему за максимальным.

```lua
local user = Users:create({
  position = db.raw("(select coalesce(max(position) + 1, 0) from users)")
})
```

```moon
user = Users\create {
  position: db.raw "(select coalesce(max(position) + 1, 0) from users)"
}
```

```sql
INSERT INTO "users" (position)
VALUES ((select coalesce(max(position) + 1, 0) from users))
RETURNING "id", "position"
```

> Так как в MySQL нет `RETURNING`, этот работает только
> с PostgreSQL

Если модели присвоены ограничения, то они проверяются до
попытки добавить объект в БД. Если ограничения не
удовлетворены, то функция `create` возвращает nil и сообщение
об ошибке.

Дополнительные опции метода `create`, которые можно передать
в форме таблицы:

* `returning` -- строка, содержащая список полей,
    которые возвращает `RETURNING`

### `columns()`

Возвращает список полей таблицы, содержащий имена полей и
их типы.

```lua
local cols = Users:columns()
```

```moon
cols = Users\columns!
```

```sql
SELECT column_name, data_type
  FROM information_schema.columns WHERE table_name = 'users'
```

Выдача:


```lua
{
  {
    data_type = "integer",
    column_name = "id"
  },
  {
    data_type = "text",
    column_name = "name"
  }
}
```

```moon
{
  {
    data_type: "integer",
    column_name: "id"
  }
  {
    data_type: "text",
    column_name: "name"
  }
}
```

> Формат выдачи MySQL может немного отличаться, но
> включает ту же информацию.

### `table_name()`

Возвращает имя таблицы.

```lua
Model:extend("users"):table_name() --> "users"
Model:extend("user_posts"):table_name() --> "user_posts"
```

```moon
(class Users extends Model)\table_name! --> "users"
(class UserPosts extends Model)\table_name! --> "user_posts"
```

<p class="for_moon">
Этот статический метод можно переопределить, чтобы изменить
имя таблицы.
</p>

```moon
class Users extends Model
  @table_name: => "active_users"
```

### `singular_name()`

Возвращает имя таблицы в единственном числе.

```lua
Model:extend("users"):singular_name() --> "user"
Model:extend("user_posts"):singular_name() --> "user_post"
```

```moon
(class Users extends Model)\singular_name! --> "user"
(class UserPosts extends Model)\singular_name! --> "user_post"
```

Этот метод используется другими функциями Lapis:
`include_in` и отношениями `has_one` и `has_many`.

### `include_in(model_instances, column_name, opts={})`

Запрашивает объекты данной модели загружает их в массив
объектов другой модели.
См. [Предзагрузка связанных объектов][preloading].

### `paginated(query, ...)`

Делает то же, что `select`, но возвращает `Paginator`.
См. [Пагинация][pagination].

## Методы объектов модели

### `update(...)`

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

Если любое из обновляемых значений сгенерировано из SQL при
помощи [`db.raw`][raw], то эти значения будут заменены на
значения, которые вернула БД при помощи `RETURNING`.
См. [`create`][create].

### `delete()`

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

Метод `delete` возвращает `true`, если объект действительно
был удалён. Важно проверять это значение, чтобы избежать
гонок при выполнении кода в ответ на удаление.

Рассмотрим следующий код:

```lua
local user = Users:find()
if user then
  user:delete()
  decrement_total_user_count()
end
```

```moon
user = Users\find 1
if user
  user\delete!
  decrement_total_user_count!
```

Так как OpenResty асинхронен, существует вероятность того, что
если эти два запроса попадут в этот блок кода примерно в одно
время, то `delete` может быть вызван дважды. Само по себе это
не является проблемой, но ведь функция
`decrement_total_user_count` тоже будет вызвана дважды и
испортит те данные, за которые она отвечает.

### `refresh(...)`

Загружает объект из базы данных заново.

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

## Отметки времени

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

## Предзагрузка связанных объектов

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

И последний сценарий предзагрузки отношений "один-к-одному".
Опция `many` указывает, что `include_in` должен загрузить
много связанных объектов для данной модели. К примеру, мы
загружаем все статьи каждого пользователя:

```lua
local users = Users:select()
Posts:include_in(users, "user_id", { flip = true, many = true })
```

```moon
users = Users\select!
Posts\include_in users, "user_id", flip: true, many: true
```

```sql
SELECT * from "posts" where "user_id" in (1,2,3,4,5,6)
```

Каждый пользователь получает дополнительное поле `posts`,
в котором хранится список статей.

Опции, которые поддерживает `include_in`:

* `as` -- имя поля, в которое записывается связанный объект
* `flip` -- если `true`, то внешний ключ находится во
    включённой модели (а не в данной)
* `where` -- список дополнительных условий, ограничивающих
    запрос
* `fields` -- список загружаемых полей включённой модели
* `many` -- если `true`, то загружает много связанных
    объектов, а не один

## Ограничения

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

## Пагинация

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

### `get_all()`

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

### `get_page(page_num)`

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

### `num_pages()`

Возвращает число страниц.

### `total_items()`

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

### `each_page(starting_page=1)`

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

> Будьте внимательны при одновременном обходе и изменении
> объектов, чтобы не пропустить объект и не обработать
> дважды

### `has_items()`

Возвращает, не пуст ли пагинатор, т.е. есть ли в нём хотя бы
один объект. Более эффективен, чем подсчёт объектов, так как
не вызывает запросов, производящих подсчёт.

```lua
if pager:has_items() then
  -- ...
end
```

```moon
if pager\has_items!
  -- ...
```

```sql
SELECT 1 FROM "users" where group_id = 123 limit 1
```

## Описание отношений моделей

Часто модели ссылаются друг на друга с помощью *внешних
ключей*. Для описания отношений между моделями в классе модели
существует поле `relations`.

```lua
local Model = require("lapis.db.model").Model
local Posts = Model:extend("posts", {
  relations = {
    {"users", belongs_to = "Users"},
    {"posts", has_many = "Tags"}
  }
})
```

```moon
import Model from require "lapis.db.models"
class Posts extends Model
  @relations: {
    {"user", belongs_to: "Users"}
    {"posts", has_many: "Tags"}
  }
```

Если в модель добавить описание отношений, то появятся методы
для получения связанных объектов. В этом примере отношение
`belongs_to` создаёт метод `get_user`:

```lua
local post = Posts:find(1)
local user = post:get_user()

-- calling again returns the cached value
local user = post:get_user()
```

```moon
post = Posts\find 1
user = post\get_user!

-- calling again returns the cached value
user = post\get_user!
```

```sql
SELECT * from "posts" where "id" = 1;
SELECT * from "users" where "id" = 123;
```

Виды отношений:

### `belongs_to`

Отношение "один-к-одному". Внешний ключ расположен в
ссылающейся модели.

Имя поля, в котором хранится внешний ключ, и имя метода,
загружающего связанную модель, образуются из названия
отношения.

К примеру, если имя связанной модели `user`, то будет добавлен
метод `get_user`, который будет возвращать объект, на который
ссылается внешний ключ `user_id`.

```lua
local Model = require("lapis.db.model").Model

local Posts = Model:extend("posts", {
  relations = {
    {"user", belongs_to = "Users"}
  }
})
```

```moon
import Model from require "lapis.db.models"

class Posts extends Model
  @relations: {
    {"user", belongs_to: "Users"}
  }
```

Загрузка связанной модели:


```lua
local user = post:get_user()
```

```moon
user = post\get_user!
```

```sql
SELECT * from "users" where "user_id" = 123;
```

Отношение `belongs_to` может содержать опцию `key`,
которая меняет поле данной модели, в котором находится
вторичный ключ.

### `has_one`

Тоже "один-к-одному", но вторичный ключ находится в связанной
модели и указывает на данную модель.

```lua
local Model = require("lapis.db.model").Model

local Users = Model:extend("users", {
  relations = {
    {"user_profile", has_one = "UserProfiles"}
  }
})
```

```moon
import Model from require "lapis.db.models"
class Users extends Model
  @relations: {
    {"user_profile", has_one: "UserProfiles"}
  }
```

Также добавляется метод `get_`, загружающий связанный объект.

```lua
local profile = user:get_user_profile()
```

```moon
profile = user\get_user_profile!
```

```sql
SELECT * from "user_profiles" where "user_id" = 123;
```

Отношение `has_one` переводит имя таблицы в
единственное число и добавляет `_id`. Таблица `users`
превращается в `user_id`. Имя поля можно изменить с помощью
опции `key`.


```lua
local Model = require("lapis.db.model").Model

local Users = Model:extend("users", {
  relations = {
    {"user_profile", has_one = "UserProfiles", key = "owner_id"}
  }
})
```

```moon
import Model from require "lapis.db.models"

class Users extends Model
  @relations: {
    {"user_profile", has_one: "UserProfiles", key: "owner_id"}
  }
```

```lua
local profile = user:get_user_profile()
```

```moon
profile = user\get_user_profile!
```

```sql
SELECT * from "user_profiles" where "owner_id" = 123;
```

### `has_many`

Отношение "один-к-многим". Определяет два метода: один
возвращает [объект `Pager`](#pagination), а другой -
возвращает все объекты.

```lua
local Model = require("lapis.db.model").Model

local Users = Model:extend("users", {
  relations = {
    {"posts", has_many = "Posts"}
  }
})
```

```moon
import Model from require "lapis.db.models"
class Users extends Model
  @relations: {
    {"posts", has_many: "Posts"}
  }
```

Метод `get_` загружает все связанные объекты. Если связанный
объект уже загружен, то вернется сохраненная копия, которая
хранится в поле модели, соответствующем имени отношения.

```lua
local posts = user:get_posts()
```

```moon
posts = user\get_posts!
```

```sql
SELECT * from "posts" where "user_id" = 123
```

Метод `get_X_paginated` возвращает пагинатор, указывающий на
связанные объекты. Он полезен, если отношение связывает много
объектов и нет смысла загружать их все.

Все дополнительные аргументы, переданные в `get_X_paginated`,
передаются в конструктор пагинатора. К примеру, можно указать
поля `fields`, `prepare_results` и `per_page`:

```lua
local posts = user:get_posts_paginated({per_page = 20}):get_page(3)
```

```moon
posts = user\get_posts_paginated(per_page: 20)\get_page 3
```

```sql
SELECT * from "posts" where "user_id" = 123 LIMIT 20 OFFSET 40
```

Дополнительные опции отношения `has_many`:

* `key` -- внешний ключ, через который связаны объекты. По
    умолчанию образуется приписыванием `_id` к имени таблицы
    в единственном числе: `Users` → `user_id`
* `where` -- таблица с дополнительными ограничениями на
    возвращаемые объекты
* `order` -- фрагмент SQL-запроса, следующий за `order by`
* `as` -- префикс методов, по умолчанию `get_`

Пример более сложных отношений:

```lua
local Model = require("lapis.db.model").Model

local Users = Model:extend("users", {
  relations = {
    {"authored_posts",
			has_many = "Posts",
			where = {deleted = false},
			order = "id desc",
			key = "poster_id"}
  }
})
```

```moon
import Model from require "lapis.db.models"
class Users extends Model
  @relations: {
    {"authored_posts"
			has_many: "Posts"
			where: {deleted: false}
			order: "id desc"
			key: "poster_id"}
  }
```

```lua
local posts = user:get_authored_posts()
```

```moon
posts = user\get_authored_posts!
```

```sql
SELECT * from "posts" where "poster_id" = 123 and deleted = FALSE order by id desc
```

### `fetch`

Произвольное отношение, определенное функцией, возвращающей
связанные данные. Результат запоминается.


```lua
local Model = require("lapis.db.model").Model

local Users = Model:extend("users", {
  relations = {
    {"recent_posts", fetch = function()
      -- fetch some data
    end}
  }
})
```

```moon
import Model from require "lapis.db.models"
class Users extends Model
  @relations: {
    {"recent_posts", fetch: =>
      -- fetch some data
    }
  }
```

## Enum

Функция `enum` принимает таблицу Lua, сопоставляющую
именам целочисленные константы. Служит для полей-перечислений
в моделях. Состояние закодировано в форме числа.

```lua
local model = require("lapis.db.model")
local Model, enum = model.Model, model.enum

local Posts = Model:extend("posts")
Posts.statuses = enum {
  pending = 1,
  public = 2,
  private = 3,
  deleted = 4
}
```

```moon
import Model, enum from require "lapis.db.model"

class Posts extends Model
  @statuses: enum {
    pending: 1
    public: 2
    private: 3
    deleted: 4
  }
```

```lua
assert(Posts.statuses[1] == "pending")
assert(Posts.statuses[3] == "private")

assert(Posts.statuses.public == 2)
assert(Posts.statuses.deleted == 4)

assert(Posts.statuses:for_db("private") == 3)
assert(Posts.statuses:for_db(3) == 3)

assert(Posts.statuses:to_name(1) == "pending")
assert(Posts.statuses:to_name("pending") == "pending")

-- using to_name or for_db with undefined enum value throws error

Posts.statuses:to_name(232) -- erorr
Posts.statuses:for_db("hello") -- erorr
```

```moon
assert Posts.statuses[1] == "pending"
assert Posts.statuses[3] == "private"

assert Posts.statuses.public == 2
assert Posts.statuses.deleted == 4

assert Posts.statuses\for_db("private") == 3
assert Posts.statuses\for_db(3) == 3

assert Posts.statuses\to_name(1) == "pending"
assert Posts.statuses\to_name("pending") == "pending"

-- using to_name or for_db with undefined enum value throws error

Posts.statuses\to_name 232 -- erorr
Posts.statuses\for_db "hello" -- erorr
```


[1]: database.html#query-interface-escape_identifierstr
[raw]: database.html#query-interface-rawstr
[preloading]: #preloading-associationws
[create]: #create
