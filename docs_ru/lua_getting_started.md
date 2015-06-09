title: Введение (Lua)
--
<div class="override_lang" data-lang="lua"></div>

# Введение (Lua)

## Создаем новый проект

Сначала прочитайте [общее введение][2], чтобы узнать, как создать
костяк нового проекта, и получить информацию об OpenResty,
конфигурациях Nginx и команде `lapis`.

Чтобы создать в текущей папке новый проект Lapis на Lua,
наберите следующую команду:

```bash
$ lapis new --lua
```

Конфигурация `nginx.conf` читает файл с приложением `app.lua`.
Командна `lapis new` создаёт костяк этого файла.

Файл `app.lua` - обычный модуль Lua, который содержит приложение.
Можно подключать модули, как и в обычном Lua.
Выглядит это так:

```lua
-- app.lua
local lapis = require("lapis")
local app = lapis.Application()

app:get("/", function()
  return "Welcome to Lapis " .. require("lapis.version")
end)

return app
```

Давайте запустим сервер:

```bash
lapis server
```

Откройте <http://localhost:8080>, чтобы увидеть страницу.

Чтобы изменить порт, создадим файл конфигурации `config.lua`.

В следующем примере мы выставляем порт 9090 для конфигурации
разработчика:

```lua
-- config.lua
local config = require("lapis.config")

config("development", {
  port = 9090
})
```

> Подробнее о конфигурации см. раздел
> [конфигурации и окружения][3].

Окружение для разработчика (`development`) используется по
умолчанию, если команду `lapis server` запустить без
дополнительных аргументов.
(И при этом отсутствует файл `lapis_environment.lua`.)

Lapis использует несколько полей конфигурации (например, `port`),
остальные поля могут использоваться для хранения чего угодно.
Пример:

```lua
-- config.lua
local config = require("lapis.config")

config("development", {
  greeting = "Hello world"
})
```

Текущую конфигурацию можно получить при помощи функции `get`,
которая возвращает обычную таблицу Lua.

```lua
-- app.lua
local lapis = require("lapis")
local config = require("lapis.config").get()

local app = lapis.Application()

app:get("/", function(self)
  return config.greeting .. " from port " .. config.port
end)

return app
```

## Создаём представление

Научились создавать простые страницы. Что дальше?
В Lapis есть встроенная поддержка языка шаблонов [etlua][1],
в котором можно вставлять Lua в HTML.

Представление - это файл, который отвечает за генерацию HTML.
Как правило, обработчик подготавливает все данные для
представления, после чего выводит представление.

По умолчанию Lapis ищет представления в папке `views/`.
Сделаем новое представление в этой папке: `index.etlua`.
Для начала это будет простой HTML, без элементов
разметки `etlua`.

```html
<!-- views/index.etlua -->
<h1>Hello world</h1>
<p>Welcome to my page</p>
```

Теги `<html>`, `<head>`, и `<body>` отсутствуют.
Представление обычно отображается внутри страницы,
а макет отвечает за код, окружающий представление.
Макеты мы рассмотрим ниже.

Теперь создадим приложение, которое отображает
наше представление:

```lua
-- app.lua
local lapis = require("lapis")

local app = lapis.Application()
app:enable("etlua")

app:get("/", function(self)
  return { render = "index" }
end)

return app
```

Шаблоны `etlua` не подключены по умолчанию.
Чтобы их включить, надо вызвать метод `enable`
на экземпляре приложения.

Чтобы указать, какое представление мы хотим отображать,
используется параметр `render` таблицы, которую
возвращает обработчик.
В данном случае `"index"` ссылается на модуль с именем
`views.index`. `etlua` внедряет себя в глобальную функцию
Lua `require`, так что при загрузке модуля `views.index`
происходит попытка прочитать и распарсить
файл `views/index.etlua`.

Запустите сервер и откройте страницу в браузере,
чтобы посмотреть отображенный шаблон.

### Работа с `etlua`

В `etlua` есть следующие теги для встраивания Lua
в шаблоны:

* `<% lua_code %>` запускает Lua-код как есть
* `<%= lua_expression %>` вставляет результат выражения,
    HTML экранируется
* `<%- lua_expression %>` то же, но HTML не экранируется

> Подробнее об интеграции etlua см. раздел [etlua][4].

В примере ниже обработчик присваиваем какие-то данные,
после чего они печатаются в отображении.

```lua
-- app.lua
local lapis = require("lapis")

local app = lapis.Application()
app:enable("etlua")

app:get("/", function(self)
  self.my_favorite_things = {
    "Cats",
    "Horses",
    "Skateboards"
  }

  return { render = "list" }
end)

return app
```

```html
<!-- views/list.etlua -->
<h1>Here are my favorite things</h1>
<ol>
  <% for i, thing in pairs(my_favorite_things) do %>
    <li><%= thing %></li>
  <% end %>
</ol>
```

### Создаём макет

Макет - это специальный шаблон, который оборачивает содержимое
каждой страницы.
В Lapis есть встроенный макет, с которого можно начать,
но вы, скорее всего, захотите заменить его.

Как и представления, макет пишется на `etlua`.
Создайте `views/layout.etlua`:

```html
<!-- views/layout.etlua -->
<!DOCTYPE HTML>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title><%= page_title or "My Page" %></title>
</head>
<body>
  <h1>Greetings</h1>
  <% content_for("inner") %>
</body>
</html>
```

Специальная функция `content_for` встроена в шаблоны.
Она служит для передачи информации из представления в макет.
Результат отображения представления помещается в переменную
содержимого с именем `inner`.
Как видно из примера, не нужно использовать теги, которые
вставляют свой результат в страницу. Дело в том, что
`content_for` эффективно пишет свой результат в
выходной буфер.

Все обычные переменные и вспомогательные функции представлений
также доступны из макета.

Применим макет к приложению:

```lua
local app = lapis.Application()
app:enable("etlua")
app.layout = require "views.layout"

-- остальной код приложения...
```

Синтаксис немного отличается от отображения представления.
На этот раз мы не пишем имя шаблона в поле `layout`,
а присваиваем сам объект шаблона.
Этого можно достичь, вызвав `require` от имени шаблона:
`"views.layout"`.
Как отмечалось выше, etlua берет на себя работу по
превращению файла `.etlua` в объект Lua.

## Далее

Прочитайте раздел [Запросы и обработчики][5], чтобы узнать,
как Lapis обрабатывает HTTP-запросы и как писать для них ответы.

[1]: https://github.com/leafo/etlua
[2]: getting_started.html
[3]: configuration.html
[4]: etlua_templates.html
[5]: $root/reference/actions.html

