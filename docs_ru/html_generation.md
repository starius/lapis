title: Генерация HTML
--
<div class="override_lang"></div>

# Генерация HTML

Рассмотрим генерацию HTML из кода на Lua/MoonScript.
Генерация HTML из шаблонов рассматривается в
разделе [etlua]($root/reference/etlua_templates.html).

## Генерация HTML в обработчиках

Чтобы сгенерировать HTML непосредственно в обработчике,
используйте метод `@html`:

```moon
"/": =>
  @html ->
    h1 class: "header", "Hello"
    div class: "body", ->
      text "Привет!"
```

HTML записывается как код на MoonScript (или Lua).
Это очень крутая возможность, вдохновлённая
пакетом [Erector](http://erector.rubyforge.org/).
Пользоваться удобнее, чем обычными шаблонами.
(Не лапша-код!)

Метод `@html` переопределяет окружение для функции,
которую в него подают.
Функции, создающие HTML-теги, создаются на лету,
когда их вызывают.
Выдача этих функций склеивается вместе и возвращается
как результат обработчика.

Примеры генерации HTML:

```moon
div!                -- <div></div>
b "Hello World"     -- <b>Hello World</b>
div "hi<br/>"       -- <div>hi&lt;br/&gt;</div>
text "Hi!"          -- Hi!
raw "<br/>"         -- <br/>

element "table", width: "100%", ->  -- <table width="100%"></table>

div class: "footer", "The Foot"     -- <div class="footer">The Foot</div>

div ->                              -- <div>Hey</div>
  text "Hey"

div class: "header", ->             -- <div class="header"><h2>My Site</h2>
  h2 "My Site"                      --    <p>Welcome!</p></div>
  p "Welcome!"
```

С помощью функции `element` можно создать любой тег.
Тег передаётся первым аргументом.
В последующих аргументах передаются атрибуты и содержимое.

Методы генератора HTML имеют меньший приоритет,
чем существующие переменные, поэтому, если у вас есть
переменная `div` и вам нужно создать тег `<div>`,
воспользуйтесь `element "div"`.

> Элементы `<table>` и `<select>` можно создать только при
> помощи `element`, так как `table` и `select` являются
> встроенными именами Lua.

## HTML-виджеты

Рекомендуется генерировать HTML через виджеты.
Виджет - это класс, основное предназначение которого
состоит в генерации HTML.
В виджетах используется такой же синтаксис,
как во вспомогательной функции `@html` (см. выше).

Автоматическая загрузка виджета происходит через имя модуля.
К примеру, виджет `"index"` загружается как модуль
`views.index`. Модуль должен возвращать виджет.

Вот как выглядит виджет:

```moon
-- views/index.moon
import Widget from require "lapis.html"

class Index extends Widget
  content: =>
    h1 class: "header", "Hello"
    div class: "body", ->
      text "Welcome to my site!"
```


> Имя виджета значения не имеет, но всё-таки стоит дать
> ему осмысленное имя, так как некоторые системы
> используют его при генерации HTML.

### Отображение виджета из обработчика

Если обработчик передаёт имя виджета в ключе `render`
в таблице, которую он возвращает, то Lapis отображает
этот виджет.
Пример: отображение виджета `"index"`:

```moon
"/": =>
  render: "index"
```

Если у обработчика есть имя, то он может установить `render`
в значение `true`, тогда отобразится виджет с тем же именем:

```moon
[index: "/"]: =>
  render: true
```

По умолчанию к имени виджета приписывается префикс `views.`,
полученная строка передается в `require`.
Этот префикс регулируется через статическое поле `views_prefix`
класса приложения:

```moon
class Application extends lapis.Application
  views_prefix: "app_views"

  -- will use "app_views.home" as the view
  [home: "/home"]: => render: true
```

### Передача данных в виджет

Виджет имеет доступ ко всем переменным,
установленным при помощи `@`.
Вспомогательные функции тоже доступны
(например, `@url_for`).

```moon
-- web.moon
[index: "/"]: =>
  @page_title = "Welcome To My Page"
  render: true
```

```moon
-- views/index.moon
import Widget from require "lapis.html"

class Index extends Widget
  content: =>
    h1 class: "header", @page_title
    div class: "body", ->
      text "Welcome to my site!"
```

### Отображение виджетов вручную

Чтобы отобразить виджет вручную, надо создать
экземпляр класса этого виджета и вызвать на нём метод
`render_to_string`:

```moon
Index = require "views.index"

widget = Index page_title: "Hello World"
print widget\render_to_string!
```

Если виджет использует вспомогательные методы
(например, `@url_for`), их надо добавить в экземпляр виджета.
Можно подключить любой объект, и его методы
станут доступны из виджета.

```moon
html = require "lapis.html"
class SomeWidget extends html.Widget
  content: =>
    a href: @url_for("test"), "Test Page"

class extends lapis.Application
  [test: "/test_render"]: =>
    widget = SomeWidget!
    widget\include_helper @
    widget\render_to_string!
```

Лучше избегать по возможности ручного отображения виджетов.
В обработчиках используйте
[опцию запроса](#request-object-request-options) `render`.
При отображении из других виджетов используйте
вспомогательную функцию `widget`.
В обоих случаях используется общий выходной буфер,
чтобы не было лишних объединений строк.

## Макеты

Результат обработчика обычно вставляется в макет.
Макет - это обычный виджет, но он используется
многими страницами.
В макете прописываются теги вроде `<html>` и `<head>`.

Макет по умолчанию устроен следующим образом:

```moon
html = require "lapis.html"

class DefaultLayout extends html.Widget
  content: =>
    html_5 ->
      head -> title @title or "Lapis Page"
      body -> @content_for "inner"
```

Используйте этот макет как отправную точку
для написания собственного макета.
Содержимое страницы подставляется в месте вызова
`@content_for "inner"`.

Макет можно задать всему приложению или конкретному
обработчику.
Предположим, что наш макет хранится в файле
`views/my_layout.moon`.

```moon
class extends lapis.Application
  layout: require "views.my_layout"
```

Чтобы задать этот макет этому обработчику,
его надо передать из этого обработчика
в опции `layout`.

```moon
class extends lapis.Application
  -- the following two have the same effect
  "/home1": =>
    layout: "my_layout"

  "/home2": =>
    layout: require "views.my_layout"

  -- this doesn't use a layout at all
  "/no_layout": =>
    layout: false, "No layout rendered!"

```

Если значение макета равно `false`,
то шаблон не отображается.

## Методы виджета

### `@@include(other_class)`

Копирует все методы другого класса в класс, к которому его
применили.
Очень полезен для включения общей функциональности
в несколько виджетов.

```moon
class MyHelpers
  item_list: (items) =>
    ul ->
      for item in *items
        li item


class SomeWidget extends html.Widget
  @include MyHelpers

  content: =>
    @item_list {"hello", "world"}
```


### `@content_for(name, [content])`

Включает HTML или строку в макет.
Вызов `@content_for "inner"` упоминался выше
при обсуждении макетов.
По умолчанию содержимое представления
размещается в блоке `"inner"`.

Можно создавать произвольные блоки при помощи `@content_for`,
в который передаётся имя блока и его содержимое:

```moon
class MyView extends Widget
  content: =>
    @content_for "title", "This is the title of my page!"

    @content_for "footer", ->
      div class: "custom_footer", "The Footer"

```

В качестве содержимого можно использовать
функции, генерирующие HTML, или строки.

Чтобы получить доступ к содержимому блоков из макета,
надо вызвать `@content_for` с одним аргументом:

```moon
class MyLayout extends Widget
  content: =>
    html ->
      body ->
        div class: "title", ->
          @content_for "title"

        @content_for "inner"
        @content_for "footer"
```

Если содержимое блока - строка, то она экранируется
перед добавлением в буфер.
Чтобы отключить экранирование строки,
её надо передать в функцию `raw`:

```moon
@content_for "footer", ->
  raw "<pre>this wont' be escaped</pre>"
```

### `@has_content_for(name)`

Возвращает, есть ли блок с именем `name`.

```moon
class MyView extends Widget
  content: =>
    if @has_content_for "things"
      @content_for "things"
    else
      div ->
        text "default content"
```

## Модуль HTML

```moon
html = require "lapis.html"
```

### `render_html(fn)`

Вызывает функцию `fn` в контексте HTML
(см. выше) и возвращает полученный HTML.
В контексте HTML обращение к несуществующей
глобальной переменной заменяется вызовом функции,
отображающей соответствующей тег.

```moon
import render_html from require "lapis.html"

print render_html ->
  div class: "item", ->
    strong "Hello!"
```

### `escape(str)`

Экранирует специальные символы HTML в строке:

 * `&` -- `&amp;`
 * `<` -- `&lt;`
 * `>` -- `&gt;`
 * `"` -- `&quot;`
 * `'` -- `&#039;`

