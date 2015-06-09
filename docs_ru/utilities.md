title: Разное
--
# Разное

## Функции

Подключим служебные функции:

```lua
local util = require("lapis.util")
```

```moon
util = require "lapis.util"
```

### `unescape(str)`

Снимает экранирование URL со строки

### `escape(str)`

Экранирует URL в строке

### `escape_pattern(str)`

Экранирует строку для использования
в качестве паттерна Lua

### `parse_query_string(str)`

Превращает строку запроса в таблицу ключ-значение

### `encode_query_string(tbl)`

Превращает таблицу ключ-значение в строку запроса

### `underscore(str)`

Превращает строки вида `CamelCase` в `camel_case`.

### `slugify(str)`

Подготавливает строку для вставки в URL.
Заменяет все пробелы и символы на `-`.

### `uniquify(tbl)`

Отбирает уникальные эелементы списка `tbl`,
добавляет их в пустую таблицу и возвращает эту таблицу.

###  `trim(str)

Удаляет пробелы с обоих концов строки.

### `trim_all(tbl)`

Перебирает все пары ключ-значение в таблице и
применяет `trim` к каждому значению.

Эта функция меняет таблицу.

### `trim_filter(tbl, [{keys ...}], [empty_val=nil])`

Удаляет пробелы с обоих концов значений (строк)
в таблице `tbl`.
Если значение стало пустой строкой, удаляет
этот элемент таблицы.

Если передан список `keys`, то все ключи не из этого
списка будут удалены из таблицы (путём присвоения
значений к `nil`, не к `empty_val`).

Если передан аргумент `empty_val`, тогда значения, состоящие
только из пробелов, заменяются на него, а не на `nil`.

Эта функция меняет таблицу.

```lua
local db = require("lapis.db")
local trim_filter = require("lapis.util").trim_filter

unknown_input = {
  username = "     hello    ",
  level = "admin",
  description = " "
}

trim_filter(unknown_input, {"username", "description"}, db.NULL)

-- unknown_input теперь выглядит так:
-- {
--   username = "hello",
--   description = db.NULL
-- }

```

```moon
db = require "lapis.db"
import trim_filter from require "lapis.util"

unknown_input = {
  username: "  hello  "
  level: "admin"
  description: " "
}

trim_filter unknown_input, {"username", "description"}, db.NULL

-- unknown_input теперь выглядит так:
-- {
--   username: "hello"
--   description: db.NULL
-- }
```

### `to_json(obj)`

Переводит `obj` в JSON.
Удаляет из выдачи циклические связи и объекты,
которые нельзя закодировать в JSON.

###  `from_json(str)`

Переводит JSON в таблицу. Это прямая обёртка над
функцией `decode` модуля CJSON.

### Кодировщики

Подключим кодировщики:

```lua
local encoding = require("lapis.util.encoding")
```

```moon
encoding = require "lapis.util.encoding"
```

### `encode_base64(str)`

Переводит строку в Base64.

### `decode_base64(str)`

Переводит строку из Base64.

### `hmac_sha1(secret, str)`

Вычисляет хеш hmac-sha1 от строки `str` с ключом `secret`.
Возвращается бинарная строка.

### `encode_with_secret(object, secret=config.secret)`

Возвращает строку, содержащую объект `object`
в закодированном виде и его подпись.

### `decode_with_secret(msg_and_sig, secret=config.secret)`

Раскодирует строку, созданную при помощи `encode_with_secret`.
Если подпись не сходится, возращает `nil` и сообщение
об ошибке.

Значение `secret` должно совпадать со значением,
поданным в `encode_with_secret`.

### `autoload(prefix, tbl={})`

Делает так, что попытка обращения к несуществующему полю
в `tbl` приводит к загрузке соответствующего
модуля при помощи `require`.
Результат загрузки сохраняется в таблице.
Помогает организовать автоподгрузку
компонентов, разбросанных по нескольим файлам.
Переопределяет метаметод `__index`.


```lua
local models = autoload("models")

models.HelloWorld --> will require "models.hello_world"
models.foo_bar --> will require "models.foo_bar"
```

```moon
models = autoload("models")

models.HelloWorld --> will require "models.hello_world"
models.foo_bar --> will require "models.foo_bar"
```

## Защита от CSRF

Атака CSRF - это запросы из форм другого сайта,
маскирующиеся под определённые действия пользователя
на нашем сайте.
Чтобы защититься от CSRF, сайт выдаёт специальный
токен (строку из случайных символов), который передаётся
обратно скрытым параметром POST из форм сайта.
Обработчики POST-запросов проверяют наличие
и правильность этого токена.

Lapis использует криптографически подписанное сообщение
в качестве токена. Сайт может убедиться в аутентичности
токена.

Перед использованием криптографии в Lapis
надо изменить ключ приложения (строку, которую
знает только приложение).
Если исходный код проекта открыт, не стоит
публиковать ключ приложения.
Задайте значение ключа в
[конфигурации](#configuration-and-environments):

```lua
local config = require("lapis.config")

config("development", {
  secret = "this is my secret string 123456"
})

```

```moon
config = require "lapis.config"

config "development", ->
  secret "this is my secret string 123456"
```

Пример формы, защищённой от CSRF:


```lua
local lapis = require("lapis")
local csrf = require("lapis.csrf")

local capture_errors = require("lapis.application").capture_errors

local app = lapis.Application()

app:get("form", "/form", function(self)
  local csrf_token = csrf.generate_token(self)
  self:html(function()
    form({ method = "POST", action = self:url_for("form") }, function()
      input({ type = "hidden", name = "csrf_token", value = csrf_token })
      input({ type = "submit" })
    end)
  end)
end)

app:post("form", "/form", capture_errors(function(self)
  csrf.assert_token(self)
  "The form is valid!"
end))
```

```moon
csrf = require "lapis.csrf"

class extends lapis.Application
  [form: "/form"]: respond_to {
    GET: =>
      csrf_token = csrf.generate_token @
      @html =>
        form method: "POST", action: @url_for("form"), ->
          input type: "hidden", name: "csrf_token", value: csrf_token
          input type: "submit"

    POST: capture_errors =>
      csrf.assert_token @
      "The form is valid!"
  }
```

> Если надо защитить от CSRF много форм, имеет смысл создать
> [предобработчик](moon_getting_started.html),
> создающий CSRF-токен

В модуле CSRF определены следующие функции:

```lua
local csrf = require("lapis.csrf")
```

```moon
csrf = require "lapis.csrf"
```

###  `generate_token(req, key=nil, expires=os.time() + 28800)`

Создаёт CSRF-токен, используя ключ приложения.
Чтобы прикрепить CSRF-токен к определённой форме,
надо указать `key`, в таком случае токен будет
действительным только в том случае, если то же значение
`key` использовалось при проверке.
По умолчанию токен имеет срок годности 8 часов.

###  `validate_token(req, key)`

Проверяет CSRF-токен `req.params.csrf_token`.
Чтобы проверить токен в контексте определённой
формы, укажите значеник `key` (каждой форме - свой `key`).
Если CSRF-токен в порядке, возвращает `true`,
иначе возвращает `nil` и сообщение об ошибке.

###  `assert_token(...)`

Вызывает `validate_token` с теми же аргументами.
Если токен не принимается, вызывает `assert_error`.


## Внешние HTTP-запросы

В Lapis есть встроенный модуль для совершения
асинхронных HTTP-запросов.
Этот модуль работает за счёт директивы Nginx `proxy_pass`,
в которой определён внутренний location.
Чтобы внешние HTTP-запросы работали,
надо отредактировать конфигурацию Nginx.

Добавьте следующее в блок server:

```nginx
location /proxy {
    internal;
    rewrite_by_lua "
      local req = ngx.req

      for k,v in pairs(req.get_headers()) do
        if k ~= 'content-length' then
          req.clear_header(k)
        end
      end

      if ngx.ctx.headers then
        for k,v in pairs(ngx.ctx.headers) do
          req.set_header(k, v)
        end
      end
    ";

    resolver 8.8.8.8;
    proxy_http_version 1.1;
    proxy_pass $_url;
}
```

> Этот код устанавливает правильные заголовки в новом запросе.
> Целевой URL хранится в переменной `$_url`.
> Чтобы всё работало, эту переменную надо объявить
> в стандартном `location /`: `set $_url "";`

Теперь можно пользоваться модулем `lapis.nginx.http`.
В модуле есть две функции: `request` и `simple`.
Функция `request` реализует интерфейс
запросов библиотеки Lua Socket (с LTN12).

Функция `simple` реализует более простой интерфейс (без LTN12):

```lua
local http = require("lapis.nginx.http")

local app = lapis.Application()

app:get("/", function(self)
  -- простой запрос GET
  local body, status_code, headers = http.simple("http://leafo.net")

  -- запрос POST, таблица с данными кодируется
  -- заголовок content-type устанавливается в
  -- application/x-www-form-urlencoded
  http.simple("http://leafo.net/", {
    name = "leafo"
  })

  -- вызов того же запроса вручную
  http.simple({
    url = "http://leafo.net",
    method = "POST",
    headers = {
      "content-type" = "application/x-www-form-urlencoded"
    },
    body = {
      name = "leafo"
    }
  })
end)
```


```moon
http = require "lapis.nginx.http"

class extends lapis.Application
  "/": =>
    -- простой запрос GET
    body, status_code, headers = http.simple "http://leafo.net"

    -- запрос POST, таблица с данными кодируется
    -- заголовок content-type устанавливается в
    -- application/x-www-form-urlencoded
    http.simple "http://leafo.net/", {
      name: "leafo"
    }

    -- вызов того же запроса вручную
    http.simple {
      url: "http://leafo.net"
      method: "POST"
      headers: {
        "content-type": "application/x-www-form-urlencoded"
      }
      body: {
        name: "leafo"
      }
    }
```

### `simple(req, body)`

Выполняет HTTP-запрос через внутренний location `/proxy`.

Возвращает 3 результата: тело ответа, код состояния HTTP
и таблицу заголовков.

Если передали один аргумент-строку, то она используется
как URL для запроса GET.

Второй аргумент, если есть, воспринимается как
тело POST-запроса. Если второй аргумент - таблица,
то он кодируется при помощи `encode_query_string`,
а заголовок `Content-type` устанавливается в
значение `application/x-www-form-urlencoded`.

Первый аргумент может быть таблицей опций:

 * `url` -- целевой URL запроса
 * `method` -- `"GET"`, `"POST"`, `"PUT"`, и т.п.
 * `body` -- строка или таблица (таблица кодируется)
 * `headers` -- таблица заголовков запроса


### `request(url_or_table, body)`

Частичная реализация [`http.request` из
Lua Socket's](http://w3.impa.br/~diego/software/luasocket/http.html#request).

Не поддерживает `proxy`, `create`, `step`, и `redirect`.

## Кеширование

Результат обработчика можно полность закешировать,
используя параметры обработчика как ключ кеша.
Это помогает ускорить отображение страниц,
которые редко меняются.
Достать готовую страницу из кеша намного быстрее,
чем запрашивать информацию из БД и генерировать HTML.

Кеш хранится в словаре, разделяемом между
всеми рабочими процессами Nginx. См. [shared dictionary
API](http://wiki.nginx.org/HttpLuaModule#lua_shared_dict)
(HttpLuaModule).
Начнём с создания разделяемого словаря
в конфигурации Nginx.

Следующий код в блоке `http` создаёт кеш размером 15 Мб:

```nginx
lua_shared_dict page_cache 15m;
```

Теперь мы можем использовать модуль, отвечающий за
кеширование, `lapis.cache`.

### `cached(fn_or_tbl)`

Оборачивает обработчик кешом.

```lua
local lapis = require("lapis")
local cached = require("lapis.cache").cached

local app = lapis.Application()

app:match("my_page", "/hello/world", cached(function(self)
  return "hello world!"
end))
```

```moon
import cached from require "lapis.cache"

class extends lapis.Application
  [my_page: "/hello/world"]: cached =>
    "hello world!"
```

Первый запрос к `/hello/world` запустит обработчик,
результат которого сохранится в кеше.
Последующие запросы будут получать результат из кеша.

Кеш запоминает не только тело ответа, но токже
заголовок Content-type и код состояния HTTP.

Кеш учитывает параметры GET, поэтому запрос к
`/hello/world?one=two` попадёт в другой слот кеша.
Параметры GET сортируются, поэтому их порядок значения
не имеет.

Попадание кеша фиксируется в заголовке ответа:
`x-memory-cache-hit: 1`.
При отладке приложения с помощью этого заголовка
можно убедиться, что кеш работает.

Вместо функции в `cached` можно подать таблицу.
Обработчик должен быть первым позиционным членом этой таблицы.
В таблице можно переть следующие опции:

* `dict_name` -- переопределить имя разделяемого словаря
    (по умолчанию `"page_cache"`)
* `exptime` -- время хранения записи кеша в секундах,
    0 - вечно (по умолчанию 0)
* `cache_key` -- переопределить функцию, генерирующую
    ключ кеша (функция, которая используется по умолчанию,
    описана выше)
* `when` -- функция, возвращающая, должен ли запрос
    быть закеширован. Получает запрос первым аргументом.
    (Значение по умолчанию `nil`)

Реализуем непродолжительное кеширование:

```lua
local lapis = require("lapis")
local cached = require("lapis.cache").cached

local app = lapis.Application()

app:match("/microcached", cached({
  exptime = 1,
  function(self)
    return "hello world!"
  end
}))

```

```moon
import cached from require "lapis.cache"

class extends lapis.Application
  "/microcached": cached {
    exptime: 1
    => "hello world!"
  }
```

### `delete(key, [dict_name="page_cache"])`

Удаляет запись кеша.
Первым аргументом передаётся сам ключ
или кортеж `{path, params}`, который будет закодирован
в ключ.


```lua
local cache = require("lapis.cache")
cache.delete({ "/hello", { thing = "world" } })
```

```moon
cache = require "lapis.cache"
cache.delete { "/hello", { thing: "world" } }
```

### `delete_all([dict_name="page_cache"])`

Удаляет все записи кеша.

### `delete_path(path, [dict_name="page_cache"])`

Удаляет все записи кеша для определённого пути.

```lua
local cache = require("lapis.cache")
cache.delete_path("/hello")
```

```moon
cache = require "lapis.cache"
cache.delete_path "/hello"
```

## Загрузка файлов

Загрузка файла осуществляется через
составную форму (multipart form).
Файл доступен в
<span class="for_moon">`@params`</span>
<span class="for_lua">`self.params`</span>.

Код формы:

```moon
import Widget from require "lapis.html"

class MyForm extends Widget
  content: =>
    form {
      action: "/my_action"
      method: "POST"
      enctype: "multipart/form-data"
    }, ->
      input type: "file", name: "uploaded_file"
      input type: "submit"
```

При отправке формы файл попадает в таблицу со свойствами
`filename` и `content`.
А эта таблица хранится в
<span class="for_moon">`@params`</span>
<span class="for_lua">`self.params`</span>,
имя тега `<input>` из формы используется в качестве ключа.

```lua
locl app = lapis.Application()

app:post("/my_action", function(self)
  local file = self.params.uploaded_file
  if file then
    return "Загрузили: " .. file.filename .. ", " .. #file.content .. " байт"
  end
end)
```

```moon
class extends lapis.Application
  "/my_action": =>
    if file = @params.uploaded_file
      "Загрузили #{file.filename}, #{#file.content} байт"

```

Убедимся, что в параметре хранится загруженный файл:

```lua
local app = lapis.Application()

app:post("/my_action", function(self)
  assert_valid(self.params, {
    { "uploaded_file", is_file: true }
  })

  -- файл готов к использованию
end)
```

```moon
class extends lapis.Application
  "/my_action": capture_errors =>
    assert_valid @params, {
      { "uploaded_file", is_file: true }
    }

    -- файл готов к использованию
```

Файл загружается в память целиком, что необходимо учитывать
при расчёте требований приложения к памяти.
Размер загружаемых файлов ограничен в Nginx в директиве
[`client_max_body_size`](http://wiki.nginx.org/HttpCoreModule#client_max_body_size).
Значение по умолчанию: 1 мегабайт.
Если планируется загрузка файлов большего размера,
надо увеличить её значение.

## Служебные функции приложения

В модуле `lapis.application` определены
следующие функции:

```lua
local app_helpers = require("lapis.application")
```

```moon
application = require "lapis.application"
```

### `fn = respond_to(verbs_to_fn={})`

Принимает таблицу `verbs_to_fn`, сопоставляющую
методы HTTP функциям-обработикам.
Возвращает новый обработчик, который вызывает
один из обработчиков из `verbs_to_fn` в зависимости
от метода HTTP.
См. раздел [обработка методов
HTTP](#lapis-applications-handling-http-verbs)

Если не определить обработчик для метода `HEAD`,
то Lapis вставит следующую функцию, которая ничего
не отображает:

```lua
function() return { layout = false } end
```

```moon
-> { layout: false }
```

Если метод текущего запроса отсутствует в таблице,
то Lapis вызывает функцию Lua `error` и
отображает страницу 500.

Если установлен специальный ключ `before`,
то сначала вызывается его обработчик, а потом
обработчик метода HTTP.
Если внутри обработчика `before` вызвали метод
<span class="for_moon">`@write`</span>
<span class="for_lua">`self.write`</span>,
то обработчик метода HTTP отменяется.

### `safe_fn = capture_errors(fn_or_tbl)`

Оборачивает функцию для перехватки ошибок,
порождённых `yield_error` и `assert_error`.
См. раздел про [обработку ошибок][0].

Если первый аргумент - функция, то запрос передаётся в
неё, а для обработки ошибок используется
следующая функция:

```lua
function() return { render = true } end
```

```moon
-> { render: true }
```

Первый аргумент может быть таблицей. Обработчик находится
в первом элементе таблицы, а обработчик ошибок - в элементе
с ключом `on_error`.

Когда происходит ошибка, в текущем запросе
устанавливается переменная
<span class="for_moon">`@errors`</span>
<span class="for_lua">`self.errors`</span>
и вызывается обработчик ошибок.

### `safe_fn = capture_errors_json(fn)`

Обёртка для `capture_errors`, которая использует
следующую функцию для обработки ошибок:

```lua
function(self) return { json = { errors = self.errors } } end
```

```moon
=> { json: { errors: @errors } }
```

### `yield_error(error_message)`

Порождает ошибку, которую ловит `capture_errors`

### `obj, msg, ... = assert_error(obj, msg, ...)`

Аналог функции Lua `assert`, который порождает
ошибку, которую ловит `capture_errors`

### `wrapped_fn = json_params(fn)`

Возвращает новую функцию, которая читает JSON из тела
запроса и размещает результаты в `@params`,
если значение `content-type` равно `application/json`.

```lua
local json_params = require("lapis.application").json_params

app:match("/json", json_params(function(self)
  return self.params.value
end))
```

```moon
import json_params from require "lapis.application"

class JsonApp extends lapis.Application
  "/json": json_params =>
    @params.value
```

```bash
$ curl \
  -H "Content-type: application/json" \
  -d '{"value": "hello"}' \
  'https://localhost:8080/json'
```

Данные из JSON целиком доступны в
<span class="for_moon">`@json`</span>
<span class="for_lua">`self.json`</span>.
Если при разборе JSON произошла ошибка, то в
<span class="for_moon">`@json`</span>
<span class="for_lua">`self.json`</span>
будет лежать `nil`, а обработка запроса продолжится.

[0]: exception_handling.html
