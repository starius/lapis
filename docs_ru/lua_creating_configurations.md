title: Синтаксис конфигурации на Lua
--
<div class="override_lang" data-lang="lua"></div>

# Синтаксис конфигурации на Lua

## Пример конфигурации

Модуль конфигурации Lapis'а позволяет объединять
таблицы рекурсивно.

К примеру, мы можем объявить базовую конфигурацию,
а потом переопределять некоторые значения
для более частных конфигураций:

```lua
-- config.moon
local config = require("lapis.config")

config({"development", "production"}, {
  host = "example.com",
  email_enabled = false,
  postgres = {
    host = "localhost",
    port = "5432",
    database = "my_app"
  }
})

config("production", {
  email_enabled = false,
  postgres = {
    database = "my_app_prod"
  }
})
```
Это приводит к следующим двум конфигурациям
(значения по умолчанию пропущены):

```lua
-- "development"
{
  host = "example.com",
  email_enabled = false,
  postgres = {
    host = "localhost",
    port = "5432",
    database = "my_app",
  },
  _name = "development"
}
```

```lua
-- "production"

{
  host = "example.com",
  email_enabled = true,
  postgres = {
    host = "localhost",
    port = "5432",
    database = "my_app_prod"
  },
  _name = "production"
}
```

Функцию `config` можно вызывать несколько раз с одним именем
конфигурации, причём переданная таблица будет "вливаться"
в конфигурацию.

