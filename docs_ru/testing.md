title: Testing
--
# Тестирование

В Lapis есть две системы тестирования.

Первая система включает имитацию запросов, которые попадают
в приложение, минуя Nginx и реальные HTTP-запросы.
Преимущество такого подхода в скорости и в том, что все ошибки
происходят внутри процесса теста.

Вторая система тестирования - это отдельный сервер Nginx для
тестирования. Такой подход использует полноценные
HTTP-запросы и проверяет одновременно конфигурацию Nginx
и код приложения.
Такой сервер очень похож на реальный сайт, запущенный
в боевых условиях.

Тесты можно писать в любом фреймворке,
мы будем использовать [busted](http://olivinelabs.com/busted/)
в примерах.

## Имитация запроса

Тестируемое приложение должно быть модулем Lua, который можно
загрузить при помощи `require` без побочных эффектов.
Имеет смысл поместить класс приложения в отдельный файл,
чтобы его можно было получить, просто загрузив этот файл.

В примерах приложение и тесты определены в одном файле
для упрощения.

Для имитации запроса используется функция `mock_request`,
определённая в модуле `lapis.spec.request`:

```lua
local mock_request = require("lapis.spec.request").mock_request

local status, body, headers = mock_request(app, url, options)
```

```moon
import mock_request from require "lapis.spec.request"

status, body, headers = mock_request(app, url, options)
```

Пример тестирования стандартного приложения при помощи
[Busted](http://olivinelabs.com/busted/):

```lua
local lapis = require("lapis.application")
local mock_request = require("lapis.spec.request").mock_request

local app = lapis.Application()

app:match("/hello", function(self)
  return "Добро пожаловать"
end)

describe("my application", function()
  it("should make a request", function()
    local status, body = mock_request(app, "/hello")

    assert.same(200, status)
    assert.truthy(body:match("Добро"))
  end)
end)

```

```moon
lapis = require "lapis"

import mock_request from require "lapis.spec.request"

class App extends lapis.Application
  "/hello": => "Добро пожаловать"

describe "my application", ->
  it "should make a request", ->
    status, body = mock_request App, "/hello"

    assert.same 200, status
    assert.truthy body\match "Добро"
```

Функция `mock_request` создаёт имитацию переменной `ngx`
из модуля Nginx Lua и подаёт её в приложение.
Аргумент `options` функции `mock_request` контролирует
тип запроса.
Возможные поля таблицы `options`:

* `get` --  таблица с параметрами GET
* `post` -- таблица с параметрами POST (если это поле
    присутствует, то `method` устанавливается
    в значение `"POST"`)
* `method` -- метод HTTP (по умолчанию `"GET"`)
* `headers` -- дополнительные заголовки HTTP-запроса
* `host` -- адрес имитируемого сервера
    (по умолчанию `"localhost"`)
* `port` -- порт имитируемого сервера
    (по умолчанию `80`)
* `scheme` -- схема URL имитируемого сервера
    (по умолчанию `"http"`)
* `prev` -- таблица заголовков ответа от
    предыдущего вызова `mock_request`

Можно симитировать серию запросов.
Если в них участвуют постояные данные (куки или сессии),
имеет смысл сохранять их от запроса к запросу.
Для этого надо передать заголовки ответа на предыдущий запрос
в поле `prev`.

```lua
local r1_status, r1_res, r1_headers = mock_request(my_app, "/first_url")
local r2_status, r2_res = mock_request(my_app, "/second_url", { prev = r1_headers })
```

```moon
r1_status, r1_res, r1_headers = mock_request MyApp!, "/first_url"
r2_status, r2_res = mock_request MyApp!, "/second_url", prev: r1_headers
```

## Тестовый сервер

Хотя имитирование запросов имеет свои преимущества,
оно не даёт доступа ко всем компонентам, которые использует
реальное приложение.
Поэтому существует вторая система тестирования:
*тестовый* сервер Nginx, через который проходят реальные
HTTP-запросы.

Тестовый сервер запускается в окружении `test`
(см. также окружения `development` и `production`).
В этом окружении можно иметь отдельную БД для тестов,
чтобы она не путалась с БД, используемыми для
разработки и запуска в боевых условиях.

Для управления тестовым сервером есть две функции:
`load_test_server` и `close_test_server`,
они находятся в модуле `"lapis.spec.server"`.

Из Busted эти две функции вызываются так:

```lua
local spec_server = require("lapis.spec.server")

describe("my site", function()
  setup(function()
    spec_server.load_test_server()
  end)

  teardown(function()
    spec_server.close_test_server()
  end)

  -- тесты, которые используют сервер
end)
```


```moon
import load_test_server, close_test_server from require "lapis.spec.server"

describe "my_site", ->
  setup ->
    load_test_server!

  teardown ->
    close_test_server!

  -- тесты, которые используют сервер
```

Тестовый сервер запускается в новом Nginx
(если ни одного Nginx не запущено)
или захватывает работающий отладочный сервер до тех пор,
пока не будет вызван `close_test_server`.
Захват отладочного сервера нужен, чтобы видеть
выдачу Nginx в консоли.

Во время работы тестового сервера можно совершать запросы
к БД и работать с моделями.
Запросы к БД пересылаются на тестовый сервер через HTTP и
выполняются внутри Nginx.

Пример юнит-теста для модели:

```lua
  it("should create a User", function()
    local Users = require("models").Users
    assert(Users:create({ name = "leafo" })
  end)
```


```moon
  it "should create a User", ->
    import Users from require "models"
    assert Users\create name: "leafo"
```

### `request(path, options={})`

Используется для совершения HTTP-запросов к тестовому серверу.
Эта функция находится в модуле `"lapis.spec.server"`.

Проверим, что `/` загружается без ошибок:

```lua
local spec_server = require("lapis.spec.server")
local request = spec_server.request

describe("my site", function()
  setup(function()
    spec_server.load_test_server()
  end)

  teardown(function()
    spec_server.close_test_server()
  end)

  it("should load /", function()
    local status, body, headers = request("/")
    assert.same(200, status)
  end)
end)
```

```moon
import load_test_server, close_test_server, request
  from require "lapis.spec.server"

describe "my_site", ->
  setup ->
    load_test_server!

  teardown ->
    close_test_server!

  it "should load /", ->
    status, body, headers = request "/"
    assert.same 200, status

```

`path` - это путь или полный URL запроса, отправляемого
на тестовый сервер.
Если передать полный URL, то доменное имя сервера
извлекается из URL и передаётся в заголовке Host.

Аргумент `options` - таблица опций. Возможные поля:

* `post` -- таблица с параметрами POST (если это поле
    присутствует, то `method` устанавливается
    в значение `"POST"`, сама таблица кодируется как тело
    запроса, а заголовок `Content-type` принимает значение
    `application/x-www-form-urlencoded`)
* `method` -- метод HTTP (по умолчанию `"GET"`)
* `headers` -- дополнительные заголовки HTTP-запроса
* `expect` -- тип ожидаемого ответа
    (на данный момент поддерживает только "json":
    автоматически парсит тело ответа в таблицу Lua
    или порождает ошибку, если ответ не является
    корректным JSON)
* `port` -- порт сервера (по умолчанию случайное число,
    которое присваивается автоматически при запуске тестов)

Функция возвращает три значения:
код состояния HTTP, тело ответа и все заголовки
ответа в таблице.

