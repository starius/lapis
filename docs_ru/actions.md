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
  "/post/:post_id/:post_name" =>
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
  "/browse/*": => print @params.splat
  "/user/:name/file/*" =>
```

В настоящий момент ошибочно дописывать что-то после звёздочки,
так как звёздочка захватывает все символы.

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

### `url_for(name_or_obj, params)`

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

Если `name_or_obj` - таблица, тогда вызывается её метод `url_params`,
в который подаётся объект-запрос и остальные аргументы, поданные в
`url_for`. Результат работы `url_params` вновь подаётся в `url_for`.

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
    return { redirect_to = self:build_url(stripped, {query: self.req.parsed_url.query, status: 301}) }
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
