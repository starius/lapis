title: Проверка ввода
--
# Проверка ввода

## Пример проверки

В Lapis есть набор инструментов для проверки пользовательского ввода.
Пример простой валидации:

```lua
local lapis = require("lapis")
local app_helpers = require("lapis.application")
local validate = require("lapis.validate")

local capture_errors = app_helpers.capture_errors

local app = lapis.Aplication()

app:match("/create-user", capture_errors(function(self)
  validate.assert_valid(self.params, {
    { "username", exists = true, min_length = 2, max_length = 25 },
    { "password", exists = true, min_length = 2 },
    { "password_repeat", equals = self.params.password },
    { "email", exists = true, min_length = 3 },
    { "accept_terms", equals = "yes", "Вы должны согласиться с Условиями" }
  })

  create_the_user(self.params)
  return { render = true }
end))


return app

```

```moon
import capture_errors from require "lapis.application"
import assert_valid from require "lapis.validate"

class App extends lapis.Application
  "/create-user": capture_errors =>

    assert_valid @params, {
      { "username", exists: true, min_length: 2, max_length: 25 }
      { "password", exists: true, min_length: 2 }
      { "password_repeat", equals: @params.password }
      { "email", exists: true, min_length: 3 }
      { "accept_terms", equals: "yes", "Вы должны согласиться с Условиями" }
    }

    create_the_user @params
    render: true
```

`assert_valid` принимает два аргумента: проверяемую таблицу
и таблицу с применяемыми проверками.
Проверка - это таблица следующего вида:

    { Проверяемый_ключ, [Сообщение_об_ошибке], Проверяющая_функция: Аргумент, ... }

`Проверяемый_ключ` - ключ в проверямой таблице.

Можно передать любое количество проверяющих функций.
Если проверяющая функция принимает несколько аргументов,
их передают в таблице (списке).

`Сообщение_об_ошибке` - необязательный позиционный элемент, содержащий
сообщение об ошибке, которое будет использоваться вместо стандартного
сообщения об ошибке.
В связи с устройством таблиц Lua, его можно указать
после проверяющих функций (см. пример выше).

## Проверяющие функции

* `exists: true` -- проверяет, что значение определено и не равно пустой строке
* `min_length: Min_Length` -- значение включает не менее `Min_Length` символов
* `max_length: Max_Length` -- значение включает не более `Max_Length` символов
* `is_integer: true` -- значение можно преобразовать в число
* `is_color: true` -- значение - цвет в hex-коде (пример: `#1234AA`)
* `is_file: true` -- значение - загруженный файл, см. [Загрузки файлов][0]
* `equals: Строка` -- значение равно Строке
* `type: Строка` -- тип значения равен Строке
* `one_of: {A, B, C, ...}` -- значение равно одному из элементов таблицы
* `file_exists: true` -- значение - загруженный файл, см. [Загрузки файлов][0]
    (устаревшее, используйте `is_file`)


## Определение произвольной проверки

Пример определения произвольной проверки

```lua
local validate = require("lapis.validate")

validate.validate_functions.integer_greater_than = function(input, min)
  local num = tonumber(input)
  return num and num > min, "%s должно быть больше, чем " .. min
end

local app_helpers = require("lapis.application")
local capture_errors = app_helpers.capture_errors

local app = lapis.Aplication()

app:match("/", capture_errors(function(self)
  validate.assert_valid(self.params, {
    { "number", integer_greater_than = 100 }
  })
end))
```

```moon
import validate_functions, assert_valid from require "lapis.validate"

validate_functions.integer_greater_than = (input, min) ->
  num = tonumber input
  num and num > min, "%s должно быть больше, чем #{min}"

import capture_errors from require "lapis.application"

class App extends lapis.Application
  "/": capture_errors =>
    assert_valid @params, {
      { "number", integer_greater_than: 100 }
    }
```

## Ручная проверка

Помимо `assert_valid` есть ещё одна полезная функция валидации:

```lua
local validate = require("lapis.validate").validate
```

```moon
import validate from require "lapis.validate"
```

* `validate(object, validation)` -- принимает такие же же аргументы,
  как `assert_valid`, но не выкидывает ошибку, а возвращает `nil`
  или сообщения об ошибках.

