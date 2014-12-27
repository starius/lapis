title: Шаблоны etlua
--

# Шаблоны etlua

[`etlua`][1] - язык шаблонов, в который можно встраивать
код на Lua для получения динамического содержимого.
`etlua` используется в `Lapis` отображения HTML-шаблонов.

Шаблоны `etlua` хранятся в файлах с расширением `.etlua`.
Чтобы файлы `.etlua` загружались прозрачно, как обычные
модули Lua, надо включить `etlua` в Lapis.

Этот шаблон выводит случайное число:

```html
<!-- views/hello.etlua -->
<div class="my_page">
  Случайное число: <%= math.random() %>
</div>
```

В `etlua` есть следующие теги для включения Lua:

* `<% lua_code %>` запускает Lua-код как есть
* `<%= lua_expression %>` вставляет результат выражения,
    HTML экранируется
* `<%- lua_expression %>` то же, но HTML не экранируется

## Отображение из обработчиков

*Обработчик* - это функция, которая обрабатывает запросы,
соответствующие определённому пути.
Обработчик выполняет бизнес-логику и подготавливает данные
для представления.
Обработчик возвращает таблицу опций, которая управляет
отображением результата.

Опция `render` из возвращаемой таблицы определяет,
какой шаблон будет отображаться.
Чтобы вывести файл `.etlua`, надо указать его имя
без расширения.
Файлы `.etlua` находятся в папке представлений
(по умолчанию `views/`).


```lua
local lapis = require("lapis")

local app = lapis.Application()
app:enable("etlua")

app:match("/", function()
  return { render: "hello" }
end)

reutrn app
```

```moon
lapis = require "lapis"

class App extends lapis.Application
  @enable "etlua"
  "/": =>
    render: "hello"
```


Отображение `"hello"` приводит к загрузке модуля
`"views.hello"`, то есть файла `views/hello.etlua`.

Обычно каждый обработчик выводит своё представление.
Чтобы не писать каждый раз имя представления,
можно задать опции `render` значение `true`.
В таком случае в качестве имени шаблона будет использоваться
имя пути (в примере `index`):


```lua
local lapis = require("lapis")

local app = lapis.Application()
app:enable("etlua")

app:match("index", "/", function()
  return { render: true }
end)

reutrn app
```

```moon
lapis = require "lapis"

class App extends lapis.Application
  @enable "etlua"
  [index: "/"]: =>
    render: true
```

```html
<!-- views/index.etlua -->
<div class="index">
  Добро пожаловать!
</div>
```


### Передача переменных в представления

Чтобы передать переменные в шаблон, надо записать их в поля
первого аргумента обработчика (`self`).
Пример передачи состояния в шаблон:

```lua
app:match("/", function(self)
  self.pets = { "Cat", "Dog", "Bird" }
  return { render = "my_template" }
end)
```

```moon
class App extends lapis.Application
  @enable "etlua"
  "/": =>
    @pets = {"Cat", "Dog", "Bird"}
    render: "my_template"
```

```html
<!-- views/my_template.etlua -->
<ul class="list">
<% for item in pets do %>
  <li><%= item %></li>
<% end %>
</ul>
```

Примечательно, к переменным обращаются по их имени,
без `self`. Все несуществующие переменные берутся
из таблицы `self`.

## Вызов вспомогательных функций из шаблонов

Вспомогательные функции можно вызывать, как будто они
находятся в области видимости.
Примером вспомогательной функции служит функция `url_for`,
которая возвращает имя пути по URL:

```html
<!-- views/about.etlua -->
<div class="about_page">
  <p>Замечательная страница!</p>
  <p>
    <a href="<% url_for("index") %>">Перейти на домашнюю</a>
  </p>
</div>
```

Любой метод объекта-запроса (известного обработчикам
под именем `self`) можно вызвать в шаблоне,
как глобальную функцию.

В `etlua` определено ещё несколько функций. См. ниже.


## Отображение подшаблонов

Подшаблон - это шаблон, отображаемый внутри другого
шаблона. Допустим, на многих страницах сайта
используется один и тот же блок навигации.
Чтобы не дублировать код в каждом шаблоне,
надо создать подшаблон с блоком навигации,
подключаемый в других шаблонах.

Чтобы отобразить подшаблон, используется функция `render`.

```html
<!-- views/navigation.etlua -->
<div class="nav_bar">
  <a href="<% url_for("index") %>">Домой</a>
  <a href="<% url_for("about") %>">О сайте</a>
</div>
```

```html
<!-- views/index.etlua -->
<div class="page">
  <% render("views.navigation") %>
</div>
```

Надо писать полное имя модуля, в котором находится подшаблон
(в данном случае `"views.navigation"`, который
соответствует `views/navigation.etlua`).
Если генераторы HTML на MoonScript тоже используются,
их тоже можно подключать через `render`.

Все переменные и вспомогательные функции, доступные шаблону,
доступны также и подшаблону.

Чтобы передать в подшаблон переменные, созданные
в шаблоне, их надо передавать во втором аргументе
функции `render`.

Искусственный пример использования подшаблонов для
отображения списка чисел:

```html
<!-- templates/list_item.etlua -->
<div class="list_item">
  <%= number_value %>
</div>
```

```html
<!-- templates/list.etlua -->
<div class="list">
<% for i, value in ipairs({}) do %>
  <% render("templates.list_item", { number_value = value }) %>
<% end %>
</div>
```

## Вспомогательные функции представлений

* `render(template_name, [template_params])` -- загружает
    и отображает шаблон
* `widget(widget_instance)` -- инстанцирует и
    отображает `Widget`

[1]: https://github.com/leafo/etlua
