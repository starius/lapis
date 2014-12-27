title: Конфигурации и окружения
--
# Конфигурации и окружения

Сервер Lapis можно запускать в разных окружениях
(конфигурациях).
К примеру, есть отладочная конфигурация, в которой используется
локальная БД, отключено кеширование кода и запущен
только один рабочий процесс.
А ещё есть "боевая" конфигурация, в которой используется
удалённая БД, включено кеширование кода и запущено
8 рабочих процессов.

При запуске сервера можно передать второй аргумент, название
конфигурации:

```bash
$ lapis server [environment]
```

Окружение по умолчанию: `development`.
Название окружения влияет только на то, какая конфигурация
будет загружена.
Если вы не определили ни одной конфигурации, то
этот аргумент ни на что не влияет.

## Создание конфигурации

При выполнении кода, зависящего от конфигурации Lapis
загружает модуль `"config"`, в котором хранятся
опции конфигурации.
Это обычный модуль Lua/MoonScript.
Давайте его создадим.

> Если модуль `config` отсутствует, то загружается
> конфигурация по умолчанию (ошибок не происходит).

```lua
local config = require("lapis.config")

config("development", {
  port = 8080
})

config("production", {
  port = 80,
  num_workers = 4,
  lua_code_cache = "off"
})

```

```moon
-- config.moon
config = require "lapis.config"

config "development", ->
  port 8080


config "production", ->
  port 80
  num_workers 4
  lua_code_cache "off"

```

В примере, приведённом выше, определяются две конфигурации,
в которых сервер запускается на различных портах.

Конфигурация хранится в обычной таблице.
Для создания конфигураций используются вспомогательные
функции из модуля `"lapis.config"`.

Опции можно присваивать
сразу нескольким конфигурациям, для этого нужно передать
список этих конфигураций первым аргументом:

```lua
config({"development", "production"}, {
  session_name = "my_app_session"
})
```

```moon
config {"development", "production"}, ->
  session_name "my_app_session"
```

В файле конфигурации можно использовать вложенные таблицы.
У Lua и MoonScript есть свои особенности,
см. следующие разделы.

* [MoonScript configuration syntax][0]
* [Lua configuration syntax][0]

## Конфигурация Nginx

Опции конфигурации используются при сборке
`nginx.conf`.
Они регистронезависимые.
Обычно имена опций записываются большими буквами,
потому что переменные окружения shell имеют приоритет
над опциями конфигурации.

Фрагмент конфигурации Nginx:

```nginx
events {
  worker_connections ${{WORKER_CONNECTIONS}};
}
```

При компиляции этой конфигурации Nginx сначала
проверяется переменная окружения shell
`LAPIS_WORKER_CONNECTIONS`.
Если этой переменной не присвоено значение,
используется опция `worker_connections` текущей конфигурации.

## Чтение опций конфигурации из приложения

Пример кода, получающего доступ к таблице
опций конфигурации:

```lua
local config = require("lapis.config").get()
print(config.port) -- выводит текущий порт
```


```moon
config = require("lapis.config").get!
print config.port -- shows the current port
```

Имя конфигурации хранится в опции `_name`.

```lua
print(config._name) -- development, production, и т.п.
```

```moon
print config._name -- development, production, и т.п.
```

## Опции конфигурации по умолчанию

Таблица значений опций конфигурации по умолчанию:

```lua
default_config = {
  port = "8080",
  secret = "please-change-me",
  session_name = "lapis_session",
  num_workers = "1",
  logging = {
    queries = true,
    requests = true
  }
}
```

```moon
default_config = {
  port: "8080"
  secret: "please-change-me"
  session_name: "lapis_session"
  num_workers: "1"
  logging: {
    queries: true
    requests: true
  }
}
```


## Доступные опции конфигурации

Некоторые опции конфигурации зарезервированы для Lapis
и вспомогательных библиотек:

* `port` (`number`) -- порт Nginx (файл `nginx.conf`)
* `num_workers` (`number`) -- число рабочих процессов
    Nginx (файл `nginx.conf`)
* `session_name` (`string`) -- имя Cookie, в котором хранится
    [сессия
    ]($root/reference/actions.html#request-object-session)
* `secret` (`string`) -- секретный ключ, используемый
    функцией `encode_with_secret`; этим же ключом подписывается
    Cookie, в котором хранится сессия
* `measure_performance` (`bool`) -- включает отслеживание
    быстродействия (времени работы обработчиков и
    запросов к БД)
* `logging` (`table`) -- определяет, какие события
    выводить на консоль или в логи

## Настройка логгирования

Опция конфигурации `logging` определяет,
какие события попадают в логи.
По умолчанию эта опция имеет следующее значение:

```lua
{
  queries = true
  requests = true
}
```

```moon
{
  queries: true
  requests: true
}
```

Всё логгирование идёт в лог notice Nginx'а
при помощи функции `print` из OpenResty.
По умолчанию Nginx пишет сообщения notice в `stderr`.
Это место контролируется директивой Nginx
[`error_log`
](http://nginx.org/en/docs/ngx_core_module.html#error_log).

## Измерение быстродействия

Если установить значение `measure_performance` в true, то Lapis
будет собирать данные по количеству событий
разных типов и затраченному времени в течение текущего запроса.

Данные хранятся в таблице `ngx.ctx.performance`.
У этой таблицы есть следующие поля:

* `view_time` -- время, затраченное на отображение представления (с)
* `layout_time` -- время, затраченное на отображение макета (с)
* `db_time` -- время, затраченное на запросы к БД (с)
* `db_count` -- число запросов к БД
* `http_time` -- время, потраченное на выполнение запросов HTTP,
    инициированных сайтом (с)
* `http_count` -- число запросов HTTP, инициированных сайтом

Если поле в таблице отсутствует (равно `nil`), значит действия
соответствующего типа не совершались.
Таблица заполняется в течение всего запроса, поэтому
лучше всего брать из неё данные в самом конце,
чтобы учесть все действия.
При помощи функции `after_dispatch` можно задать функцию,
выполняемую в самом конце обработки запроса.

Пример: информация о быстродействии печатается в журнал
после каждого запроса:

```lua
local lapis = require("lapis")
local after_dispatch = require("lapis.nginx.context").after_dispatch
local to_json = require("lapis.util").to_json

local config = require("lapis.config")

config("development", {
  measure_performance = true
})


local app = lapis.Application()

app:before_filter(function(self)
  after_dispatch(function()
    print(to_json(ngx.ctx.performance))
  end)
end)

-- ...

return app

```


```moon
lapis = require "lapis"
import after_dispatch from require "lapis.nginx.context"
import to_json from require "lapis.util"

config = require "lapis.config"
config "development", ->
  measure_performance true

class App extends lapis.Application
  @before_filter: =>
    after_dispatch ->
      print to_json(ngx.ctx.performance)

  -- ...
```


