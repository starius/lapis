title: Обработка ошибок
--
# Обработка ошибок

## Типы ошибок

В Lapis ошибки разделяют на два типа:
*исправимые* и *неисправимые*.
Ошибки, порождаемые Lua, и вызовы функции `error`
(в том числе, срабатывания функции `assert`)
считаются неисправимыми ошибками.

Разработчик сайта не должен допускать
неисправимые ошибки. Если они происходят,
Lapis печатает сообщение об ошибке в браузер.
Если ошибка произошла в обработчике, то он прерывается
и выводится трассировка стека.
Кроме того, устанавливается код состояния HTTP `500`.

Обычно такие ошибки - результат бага в коде сайта или другой
серьёзной ошибки и должны быть исправлены.

Исправимые ошибки - это способ прервать исполнение обработчика,
контролируемый разработчиком сайта.
Они реализованы не через Lua'шную систему обработки ошибок,
а через сопрограммы.

Примеры исправимых ошибок: неверный ввод данных
пользователем, отсутствующие записи БД.

## Перехват исправимых ошибок

Вспомогательная функция `capture_errors` -
обёртка для обработчика, которая перехватывает
исправимые ошибки и вызывает обработчик ошибок.

Эта функция не перехватывает ошибки Lua,
для этого надо использовать функцию `pcall`, как и обычно.

В Lua нет механизма исключений, как в других языках.
Вместо этого в Lapis реализована система обработки ошибок,
основанная на сопрограммах.
Чтобы перехватывать исправимые ошибки, обработчик
передаётся во вспомогательную функцию `capture_errors`.
Чтобы породить ошибку, используется функция `yield_error`.

```lua
local lapis = require("lapis")
local app_helpers = require("lapis.application")

local capture_errors, yield_error = app_helpers.capture_errors, app_helpers.yield_error

local app = lapis.Application()

app:match("/do_something", capture_errors(function(self)
  yield_error("Произошла ошибка")
  return "Привет!"
end))
```

```moon
import capture_errors, yield_error from require "lapis.application"

class App extends lapis.Application
  "/do_something": capture_errors =>
    yield_error "Произошла ошибка"
    "Привет!"
```

Рассмотрим, что происходит в случае ошибки.
Выполнение обработчика прекращается при первой ошибке,
после чего запускается обработчик ошибок.
Обработчик ошибок по умолчанию записывает
список ошибок в переменную
<span class="for_moon">`@errors`</span>
<span class="for_lua">`self.errors`</span>
и возвращает
<span class="for_moon">`render: true`</span>
<span class="for_lua">`{ render = true }`</span>.
Если у вас есть представление с тем же именем,
что и текущий путь, то оно и будет отображено.
В коде представлений надо учитывать возможность
существования ошибок (переменная `errors`).

Чтобы использовать свой обработчик ошибок, надо передать
его в `capture_errors` в составе таблицы
(внимание: значение <span class="for_moon">`@errors`</span>
<span class="for_lua">`self.errors`</span>
выставляется до вызова этого обработчика)

```lua
app:match("/do_something", capture_errors({
  on_error = function(self)
    log_erorrs(self.errors) -- у вас есть функция log_errors
    return { render = "my_error_page", status = 500 }
  end,
  function(self)
    if self.params.bad_thing then
      yield_error("произошла ошибка")
    end
    return { render = true }
  end
}))
```

```moon
class App extends lapis.Application
  "/do_something": capture_errors {
    on_error: =>
      log_errors @errors -- у вас есть функция log_erorrs
      render: "my_error_page", status: 500

    =>
      if @params.bad_thing
        yield_error "произошла ошибка"
      render: true
  }
```

Если в функцию `capture_errors` передали таблицу,
то в качестве обработчика используется первый
позиционный аргумент.

Для использования в JSON API полезен другой метод:
`capture_errors_json`, который отображает ошибку
в виде объекта JSON:

```lua
local lapis = require("lapis")
local app_helpers = require("lapis.application")

local capture_errors_json, yield_error = app_helpers.capture_errors_json, app_helpers.yield_error

local app = lapis.Application()

app:match("/", capture_errors_json(function(self)
  yield_error("произошла ошибка")
end))
```

```moon
import capture_errors_json, yield_error from require "lapis.application"

class App extends lapis.Application
  "/": capture_errors_json =>
    yield_error "произошла ошибка"
```

Будет сформирован следующий ответ с правильным
Content-type:

```json
{ errors: ["something bad happened"] }
```

### `assert_error`

В Lua принято в случае ошибки возвращать из функции
`nil` и сообщение об ошибке.
Результаты таких функций удобно передавать в функцию
`assert_error`.
Если первый аргумент `assert_error` ложный (`nil` или `false`),
то порождается ошибка, а в качестве сообщения об ошибке
используется второй аргумент.
Иначе `assert_error` возвращает все свои аргументы как есть.

Функция `assert_error` очень полезна при работе с БД.
Методы работы с БД следуют вышеописанному правилу.

```lua
local lapis = require("lapis")
local app_helpers = require("lapis.application")

local capture_errors, assert_error = app_helpers.capture_errors, app_helpers.assert_error

local app = lapis.Application()

app:match("/", capture_errors(function(self)
  local user = assert_error(Users:find({id = "leafo"}))
  return "результат: " .. user.id
end))

```

```moon
import capture_errors, assert_error from require "lapis.application"

class App extends lapis.Application
  "/": capture_errors =>
    user = assert_error Users\find id: "leafo"
    "результат: #{user.id}"
```


