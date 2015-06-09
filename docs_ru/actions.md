title: Запросы и обработчики
--
# Запросы и обработчики

В приложении Lapis есть *пути* и *обработчики*.
*Путь* - это маска для URL.
Каждому пути сопоставлена специальная функция -
*обработчик*, которая вызывается, если URL запроса
соответствует данному пути.

Всем обработчикам в качестве аргумента передается
[*сам запрос*](#request-object).

Если обработчик возвращает строку,
эта строка передается в браузер напрямую.
Если обработчик возвращает таблицу,
то она используется в качестве
[*опций запроса*](#request-options).

Если ни один путь не подошел, то выполняется обработчик по умолчанию,
см. также [*функции приложения*](#application-callbacks).

## Пути -- паттерны URL

Простые пути не содержат параметров:


```lua
app:match("/", function(self) end)
app:match("/hello", function(self) end)
app:match("/users/all", function(self) end)
```

```moon
class extends lapis.Application
  "/": =>
  "/hello": =>
  "/users/all": =>
```

Эти пути соответствуют URL'ам дословно. Слеш в начале пути обязателен.
Путь должен полностью соответствовать URL запроса. Например, URL
`/hello/world` не соответствует пути `/hello`.

Специальный синтаксис путей позволяет задать в них параметры и присвоить
им имена. Именованный параметр задается при помощи двоеточия.
Параметр соответствует всем символам, кроме `/`.
Параметр должен содержать хотя бы один символ, иначе путь
не будет подходить.


```lua
app:match("/page/:page", function(self)
  print(self.params.page)
end)
app:match("/post/:post_id/:post_name", function(self) end)
```

```moon
class extends lapis.Application
  "/page/:page": => print @params.page
  "/post/:post_id/:post_name": =>
```

Захваченные значения параметров сохраняются в поле `params` запроса.
`self.params` является таблицей, в которой ключами служат имена
параметров.

Есть ещё один тип "захвата", звёздочка. Звёздочка (`*`) соответствует
всем (не меньше одного) символам до конца пути (включая `/`).
Значение, захваченное звёздочкой, сохраняется в поле `self.params.splat`.

```lua
app:match("/browse/*", function(self)
  print(self.params.splat)
end)
app:match("/user/:name/file/*", function(self) end)
```

```moon
class extends lapis.Application
  "/browse/*": =>
    print @params.splat

  "/user/:name/file/*": =>
    print @params.name, @params.splat
```

В настоящий момент ошибочно дописывать что-то после звёздочки,
так как звёздочка захватывает все символы.

## Именованные пути

Полезно давать имена путям, чтобы генерировать URL по этим
именам, а не зашивать URL в ссылках с других старниц.

<span class="for_moon">
Если путь обработчика является таблицей с единственной
парой ключ-значение, то эта пара воспринимается как
имя и путь паттерна.
MoonScript предоставляет удобный синтаксис для этого:
</span><span class="for_lua">
Каждый метод приложения, который определяет новый путь,
имеет второй вариант, в котором имя пути передаётся
первым аргументом:
</span>

```lua
local lapis = require("lapis")
local app = lapis.Application()

app:match("index", "/", function(self)
  return self:url_for("user_profile", { name = "leaf" })
end)

app:match("user_profile", "/", function(self)
  return "Hello " .. self.params.name .. ", go home: " .. self:url_for("index")
end)
```

```moon
lapis = require "lapis"

class extends lapis.Application
  [index: "/"]: =>
    @url_for "user_profile", name: "leaf"

  [user_profile: "/user/:name"]: =>
    "Hello #{@params.name}, go home: #{@url_for "index"}"
```

Пути различных обработчиков можно генерировать при помощи
<span class="for_moon">`@url_for`</span><span class="for_lua">
`self:url_for()`</span>. Первый аргумент - имя пути, а второй
дополнительный аргумент - таблица значений, которые
подставляются в именованные пареметры пути.

## Обработка методов HTTP

Часто обработчик одного пути делает разные вещи
в зависимости от метода HTTP.
В Lapis есть функции-помощники, которые упрощают
написание таких обработчиков.
Функция `respond_to` принимает таблицу, переводящую метод
HTTP в функцию, которая выполняется для запросов
с этим методом HTTP.

```lua
local lapis = require("lapis")
local app = lapis.Application()

app:match("create_account", "/create-account", respond_to({
  GET = function(self)
    return { render = true }
  end,
  POST = function(self)
    do_something(self.params)
    return { redirect_to = self:url_for("index") }
  end
}))
```

```moon
lapis = require "lapis"
import respond_to from require "lapis.application"

class App extends lapis.Application
  [create_account: "/create-account"]: respond_to {
    GET: => render: true

    POST: =>
      do_something @params
      redirect_to: @url_for "index"
  }
```

В `respond_to` можно также прописать предобработчик, который
вызывается до обработчика метода HTTP.
Для этого в таблице нужно указать функцию с ключом `before`.
Здесь применяется та же логика, что в
[предобработчиках](#lapis-applications-before-filters),
так что можно вызвать <span class="for_moon">`@write`</span>
<span class="for_lua">`self:write()`</span> и сам обработчик
отменится.

```lua
local lapis = require("lapis")
local app = lapis.Application()

app:match("edit_user", "/edit-user/:id", respond_to({
  before = function(self)
    self.user = Users:find(self.params.id)
    if not self.user then
      self:write({"Not Found", status = 404})
    end
  end,
  GET = function(self)
    return "Edit account " .. self.user.name
  end,
  POST = function(self)
    self.user:update(self.params.user)
    return { redirect_to = self:url_for("index") }
  end
}))
```

```moon
lapis = require "lapis"
import respond_to from require "lapis.application"

class App extends lapis.Application
  "/edit_user/:id": respond_to {
    before: =>
      @user = Users\find @params.id
      @write status: 404, "Not Found" unless @user

    GET: =>
      "Edit account #{@user.name}..."

    POST: =>
      @user\update @params.user
      redirect_to: @url_for "index"
  }
```

Запрос типа `POST`, если его заголовок `Content-type` равен
`application/x-www-form-urlencoded`, разбирается и помещается в
<span class="for_moon">`@params`</span><span
class="for_lua">`self.params`</span> вне зависимости от того,
используется ли `respond_to`.

<span class="for_lua">
Методы `app:get()` и `app:post()` -- обёртки для `respond_to`,
позволяющие установить обработчик для определенного кода HTTP.
Такие обёртки существуют для всех распространённых кодов HTTP:
`get`, `post`, `delete`, `put`.
Для остальных используйте `respond_to`.
</span>

```lua
app:get("/test", function(self)
  return "Я отвечаю только на GET-запросы"
end)

app:delete("/delete-account", function(self)
  -- удалить!
end)
```


## Предобработчики

Бывает нужно выполнять какие-то действия перед каждым запросом.
К примеру, открыть пользовательскую сессию.
Для этого надо задать предобработчик, то есть функцию, которая
запускается перед каждым обработчиком:

```lua
local app = lapis.Application()

app:before_filter(function(self)
  if self.session.user then
    self.current_user = load_user(self.session.user)
  end
end)

app:match("/", function(self)
  return "current user is: " .. tostring(self.current_user)
end)
```

```moon
lapis = require "lapis"

class App extends lapis.Application
  @before_filter =>
    if @session.user
      @current_user = load_user @session.user

  "/": =>
    "current user is: #{@current_user}"
```

Можно задать несколько предобработчков, для этого нужно
вызвать <span class="for_moon">`@before_filter`</span><span
class="for_lua">`app:before_filter`</span> несколько раз.
Множественные предобработчики запускаются в том порядке,
в котором их добавили.

Если предобработчик вызывает метод <span class="for_moon">
`@write`</span><span class="for_lua">`self:write()`</span>,
то обработчик отменяется.
Например, мы можем отменить обработчик и перенаправить
пользователя на другую страницу, если какое-то условие
не выполняется:

```lua
local app = lapis.Application()

app:before_filter(function(self)
  if not user_meets_requirements() then
    self:write({redirect_to = self:url_for("login")})
  end
end)

app:match("login", "/login", function(self)
  -- ...
end)
```

```moon
lapis = require "lapis"

class App extends lapis.Application
  @before_filter =>
    unless user_meets_requirements!
      @write redirect_to: @url_for "login"

  [login: "/login"]: => ...
```

> Результат обработчика также подаётся в
> <span class="for_moon">`@write`</span><span class="for_lua">
> `self:write()`</span>, так что в него можно подавать всё то,
> что можно вернуть из обработчика

## Объект-запрос

Каждому обработчику подается *объект-запрос* в качестве первого аргумента.
Этот объект принято называть `self`.

Поля объета-запроса:

* <span class="for_moon">`@params`</span><span class="for_lua">`self.params`</span> --
    таблица, содержащая все параметры пути (см. выше), а также GET и POST
* <span class="for_moon">`@req`</span><span class="for_lua">`self.req`</span> --
    "сырой" запрос, сгенерированный из `ngx`
* <span class="for_moon">`@res`</span><span class="for_lua">`self.res`</span> --
    "сырая" таблица ответа, по которой обновляется `ngx`
* <span class="for_moon">`@app`</span><span class="for_lua">`self.app`</span> --
    объект приложения
* <span class="for_moon">`@cookies`</span><span class="for_lua">`self.cookies`</span> --
    таблица кук, в поля которой можно присваивать для создания новых кук.
    В качестве значений поддерживает только строки
* <span class="for_moon">`@session`</span><span class="for_lua">`self.session`</span> --
    подписанная таблица сессии. В сессии можно хранить что угодно, что
    можно закодировать в JSON. Обслуживается куками
* <span class="for_moon">`@options`</span><span class="for_lua">`self.options`</span> --
    опции, определяющие, как запрос отобразится в Nginx
* <span class="for_moon">`@buffer`</span><span class="for_lua">`self.buffer`</span> --
    выходной буфер
* <span class="for_moon">`@route_name`</span><span class="for_lua">`self.route_name`</span> --
    имя пути, который соответствует запросу (если пути приписано имя)

### @req

Сырая таблица <span class="for_lua">`@req`</span><span class="for_lua">`self.req`</span>
является обёрткой данных, пришедших от `ngx`.
Ниже приведен список всех доступных полей.

* <span class="for_moon">`@req.headers`</span><span class="for_lua">`self.req.headers`</span> --
    таблица заголовков запроса
* <span class="for_moon">`@req.parsed_url`</span><span class="for_lua">`self.req.parsed_url`</span> --
    разобранный URL запроса. Таблица, содержащая поля
    `scheme`, `path`, `host`, `port`, и `query`.
* <span class="for_moon">`@req.params_post`</span><span class="for_lua">`self.req.params_post`</span> --
    таблица с параметрами POST, переданными в запросе
* <span class="for_moon">`@req.params_get`</span><span class="for_lua">`self.req.params_get`</span> --
    таблица с параметрами GET, переданными в запросе


### Куки

Через таблицу <span class="for_moon">`@cookies`</span><span
class="for_lua">`self.cookies`</span> можно читать и писать куки.
Если попытаться перебрать все куки при помощи этой таблицы,
то может обнаружиться, что она пустая:


```lua
app:match("/my-cookies", function(self)
  for k,v in pairs(self.cookies) do
    print(k, v)
  end
end)
```

```moon
"/my-cookies": =>
  for k,v in pairs(@cookies)
    print k,v
```

Существующие куки хранятся в поле `__index` метатаблицы.
Так сделано, чтобы была возможность различать новые куки от
старых. Новые куки, присвоенные обработчиком запроса, хранятся
в таблице <span class="for_moon">`@cookies`</span>
<span class="for_lua">`self.cookies`</span> напрямую.

Таким образом, чтобы установить значение куки, достаточно
просто присвоить поле в таблице
<span class="for_moon">`@cookies`</span>
<span class="for_lua">`self.cookies`</span>:

```lua
app:match("/sets-cookie", function(self)
  self.cookies.foo = "bar"
end)
```

```moon
"/sets-cookie": =>
  @cookies.foo = "bar"
```

По умолчанию все куки получают дополнительные атрибуты `Path=/; HttpOnly`,
которые создают
[*сессионную куку*](http://en.wikipedia.org/wiki/HTTP_cookie#Terminology).
Чтобы повлиять на атрибуты кук, нужно создать функцию `cookie_attributes`
в приложении. Пример того, как задать кукам время действия, чтобы
сделать их вечными (1 год):

```moon
date = require "date"

class extends lapis.Application
  cookie_attributes: (name, value) =>
    expires = date(true)\adddays(365)\fmt "${http}"
    "Expires=#{expires}; Path=/; HttpOnly"
```

```lua
local date = require("date")
local app = lapis.Application()

app.cookie_attributes = function(self)
  local expires = date(true):adddays(365):fmt("${http}")
  return "Expires=" .. expires .. "; Path=/; HttpOnly"
end
```

Аргументы метода `cookie_attributes`: объект-запрос (`self`),
имя и значение куки.

### Сессии

Более продвинутым способом хранить данные между запросами является
переменная
<span class="for_moon">`@session`</span>
<span class="for_lua">`self.session`</span>.
Сессия может содержать только примитивные типы.
Её сессии запаковывается в JSON и хранится в специальной
куке. Значение подписывается секретным ключом приложения,
поэтому не может быть подделано.

Чтение и запись в сессию происходит так же, как в куку:

```lua
app.match("/", function(self)
  if not self.session.current_user then
    self.session.current_user = "Adam"
  end
end)
```

```moon
"/": =>
  unless @session.current_user
    @session.current_user = "Adam"
```

По умолчанию сессия хранится в куке с именем `lapis_session`.
Это имя может быть изменено путём задания
[опции конфигурации](#configuration-and-environments).
Сессии подписываются секретным ключом приложения, который хранится в
опции `secret`. Крайне желательно изменить это значение.

```lua
-- config.lua
local config = require("lapis.config").config

config("development", {
  session_name = "my_app_session",
  secret = "this is my secret string 123456"
})
```

```moon
-- config.moon
import config from require "lapis.config"

config "development", ->
  session_name "my_app_session"
  secret "this is my secret string 123456"
```

## Методы объекта-запроса

###  `write(things...)`

Пишет все свои аргументы. Действие зависит от типов аргументов.

* `строка` -- строка дописывается в выходной буфер
* `функция` (или таблица, которую можно вызвать) --
    функция применяется к выходному буферу, результат рекурсивно
    подается в `write`
* `таблица` -- парю ключ/значение присваиваются в
    <span class="for_moon">`@options`</span>
    <span class="for_lua">`self.options`</span>,
    все остальные значения рекурсивно подаются в `write`

В большинстве случаев не нужно вызывать `write`, так как результат обработчика
автоматически подается в `write`. В предобработчике `write` имеет две
функции: запись результата и отмена дальнейшего обработчика.

### `url_for(name_or_obj, params, query_params=nil, ...)`

Создаёт URL для `name_or_obj`.

Если `name_or_obj` является строкой, тогда будет использован путь
с таким именем, в котором именованные параметры получат значения
согласно `params`.

Пример:

```lua
app:match("user_data", "/data/:user_id/:data_field", function()
  return "hi"
end)

app:match("/", function(self)
  -- returns: /data/123/height
  self:url_for("user_data", { user_id = 123, data_field = "height"})
end)
```

```moon
[user_data: "/data/:user_id/:data_field"]: =>
  "hi"

"/": =>
  -- returns: /data/123/height
  @url_for "user_data", user_id: 123, data_field: "height"
```

Если передан аргумент `query_params`, то его значение будет
переведено в параметры запроса и добавлено в конец пути.

```lua
-- returns: /data/123/height?sort=asc
self:url_for("user_data", { user_id = 123, data_field = "height"}, { sort = "asc" })
```

```moon
-- returns: /data/123/height?sort=asc
@url_for "user_data", { user_id: 123, data_field: "height"}, sort: "asc"
```

#### Передача объекта в `url_for`

Если `name_or_obj` - таблица, тогда вызывается её метод `url_params`,
в который подаётся объект-запрос и остальные аргументы, поданные в
`url_for`. Результат работы `url_params` вновь подаётся в `url_for`.

Пример: модель `Users`, для которой определен метод
`url_params`.

```lua
local Users = Model:extend("users", {
  url_params = function(self, req, ...)
    return "user_profile", { id = self.id }, ...
  end
})
```

```moon
class Users extends Model
  url_params: (req, ...) =>
    "user_profile", { id: @id }, ...
```

Теперь можно передать объекты модели `Users` в метод
`url_for`, он вернёт путь к `user_profile`.

```lua
local user = Users:find(100)
self:url_for(user)
-- возвращает: /user-profile/100
```

```moon
user = Users\find 100
@url_for user
-- возвращает: /user-profile/100
```

Мы передали `...` через функцию `url_params`, чтобы не сломать
третий аргумент функции `query_params`:

```lua
local user = Users:find(1)
self:url_for(user, { page = "likes" })
-- возвращает: /user-profile/100?page=likes
```

```moon
user = Users\find 1
@url_for user, page: "likes"
-- возвращает: /user-profile/100?page=likes
```

#### Метод `url_key`

Значения параметров строкового типа дословно подставляются в URL.
У параметров-таблиц вызывается метод `url_key`, результат которого
подставляется в URL.

Пример: модель `Users`, для которой генерируется URL:

```lua
local Model = require("lapis.db.model").Model

local Users = Model:extend("users", {
  url_key = function(self, route_name)
    return self.id
  end
})
```

```moon
class Users extends Model
  url_key: (route_name) => @id
```

Обычная генерация пути к профилю пользователя:

```lua
local user = Users:find(1)
self:url_for("user_profile", {id = user.id})
```

```moon
user = Users\find 1
@url_for "user_profile", id: user.id
```

Благодаря методу `url_key`, можно передавать объекты `User`
напрямую в значение `id`:

```lua
local user = Users:find(1)
self:url_for("user_profile", {id = user})
```

```moon
user = Users\find 1
@url_for "user_profile", id: user
```

> Метод `url_key` принимает первым аргументом имя пути,
> что позволяет принимать решение, что возвращать, в
> зависимости от обрабатываемого пути

### `build_url(path, [options])`

Превращает путь в абсолютный URL.
Для генерации URL используется URI текущего запроса.

К примеру, если запустили сервер `localhost:8080`:


```lua
self:build_url() --> http://localhost:8080
self:build_url("hello") --> http://localhost:8080/hello

self:build_url("world", { host = "leafo.net", port = 2000 }) --> http://leafo.net:2000/world
```

```moon
@build_url! --> http://localhost:8080
@build_url "hello" --> http://localhost:8080/hello

@build_url "world", host: "leafo.net", port: 2000 --> http://leafo.net:2000/world
```

## Опции запроса

Когда записывают таблицу, пары ключ-значения (для ключей-строк)
копируются в <span class="for_moon">`@options`</span>
<span class="for_lua">`self.options`</span>.
В примере ниже копируются поля `render` и `status`.
Когда обработчик завершается, по таблице `options`
создаётся соответствующий ответ.

```lua
app:match("/", function(self)
  return { render = "error", status = 404}
end)
```

```moon
"/": => render: "error", status: 404
```

Список возможных опций:

* `status` -- устанавливает код состояния HTTP (например, 200, 404, 500, ...)
* `render` -- содержит отображаемое представление. Если значение `true`,
  то в качестве имени представления используется имя пути.
  В противном случае значение - строка или класс-представление.
* `content_type` -- устанавливает заголовок `Content-type`
* `json` -- значение опции кодирется в JSON и возвращается пользователю.
  Кроме того, `Content-type` устанавливается в `application/json`.
* `layout` -- меняет макет страницы с значения по умолчанию
  для данного приложения
* `redirect_to` -- устанавливает состояние HTTP на 302 и
  значение заголовка `Location` на своё значение.
  Поддерживает и относительные, и абсолютные пути.
  (Можно сочетать с `status`, чтобы выполнить перенаправление
  с кодом `301`)

При отображении JSON надо выставить опцию `json`.
Будет установлен корректный `Content-type` и
отключён макет:

```lua
app:match("/hello", function(self)
  return { json = { hello = "world" } }
end)
```

```moon
class extends lapis.Application
  "/hello": =>
    json: { hello: "world!" }
```





## Функции приложения

Функции приложения - это специальные методы приложения,
которые вызываются, когда обрабатываются особые виды запросов.
Функции приложения можно переопределить.
Хотя эти функции находятся в приложении, они вызываются,
как будто они обычные обработчики, то есть первым аргументом они получают
объект-запрос.

### Обработчик по умолчанию

Если запрос не соответствует ни одному определенному пути,
он передаётся обработчику по умолчанию.
Обработчик по умолчанию из коробки:


```lua
app.default_route = function(self)
  -- strip trailing /
  if self.req.parsed_url.path:match("./$") then
    local stripped = self.req.parsed_url:match("^(.+)/+$")
    return {
      redirect_to = self:build_url(stripped, {
        status = 301,
        query = self.req.parsed_url.query,
      })
    }
  else
    self.app.handle_404(self)
  end
end
```

```moon
default_route: =>
  -- strip trailing /
  if @req.parsed_url.path\match "./$"
    stripped = @req.parsed_url.path\match "^(.+)/+$"
    redirect_to: @build_url(stripped, query: @req.parsed_url.query), status: 301
  else
    @app.handle_404 @
```

Если URL заканчивается `/`, обработчик по умолчанию пробует
перенаправить на вариант без `/` на конце.
В противном случае вызывает метод приложения `handle_404`.

Метод `default_route` - самый обычный метод приложения.
Его можно переопределить как угодно.
К примеру, этот код включает запись отчёта:


```lua
app.default_route = function(self)
  ngx.log(ngx.NOTICE, "User hit unknown path " .. self.req.parsed_url.path)

  -- call the original implementaiton to preserve the functionality it provides
  return lapis.Application.default_route(self)
end
```

```moon
class extends lapis.Application
  default_route: =>
    ngx.log ngx.NOTICE, "User hit unknown path #{@req.parsed_url.path}"
    @super!
```

Предопределённая версия `default_route` использует метод `handle_404`,
который предопределён так:

```lua
app.handle_404 = function(self)
  error("Failed to find route: " .. self.req.cmd_url)
end
```

```moon
handle_404: =>
  error "Failed to find route: #{@req.cmd_url}"
```

Каждый неверный запрос приводит к ошибке 500 и трассировке стека.
Если Вы хотите оформить страницу 404 как следует, начните отсюда.

Чтобы определить свою страницу 404, но сохранить удаление `/` с конца URL,
надо переопределить метод `handle_404`, а не `default_route`.

Простой обработчик 404, который просто печатает `"Not Found!"`

```lua
app.handle_404 = function(self)
  return { status = 404, layout = false, "Not Found!" }
end
```

```moon
class extends lapis.Application
  handle_404: =>
    status: 404, layout: false, "Not Found!"
```

## Обработчик ошибок

Каждый обработчик, запускаемый Lapis, заворачивается в [`xpcall`][1].
Благодаря этому, критические ошибки отлавливаются и отображается
осмысленное сообщение об ошибке вместо умолчательной страницы Nginx,
которая не знает о Lua.

Обработчик ошибок надо использовать только для отлова критических и
неожиданных ошибок. Обработка ожидаемых ошибоки обсуждается
в [Разделе Обработка ошибок]($root/reference/exception_handling.html).

Предопределённый обработчик ошибок получает информацию об ошибке
и отображает шаблон `"lapis.views.error"`.
Это страница ошибки содержит трассировку стека и сообщение об ошибке.

Чтобы реализовать свою логику обработки ошибок,
надо переопределить метод `handle_error`:

```lua
app.handle_error = function(self, err, trace)
  ngx.log(ngx.NOTICE, "There was an error! " .. err .. ": " ..trace)
  lapis.Application.handle_error(self, err, trace)
end
```

```moon
class extends lapis.Application
  handle_error: (err, trace) =>
    ngx.log ngx.NOTICE, "There was an error! #{err}: #{trace}"
    super!
```


[1]: http://www.lua.org/manual/5.1/manual.html#pdf-xpcall
