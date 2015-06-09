title: Введение (MoonScript)
--
<div class="override_lang"></div>

# Введение (MoonScript)

## Создаем базовое приложение

Чтобы создать в текущей папке новый проект Lapis на MoonScript,
наберите следующую команду:

```bash
$ lapis new
```

Теперь у нас есть стандартный `nginx.conf` и костяк приложения `app.moon`:

```moon
-- app.moon
lapis = require "lapis"

class extends lapis.Application
  "/": =>
    "Welcome to Lapis #{require "lapis.version"}!"
```

Здесь определеляется обычный модуль Lua, который возвращает
класс приложения. (В MoonScript работает неявный возврат последней
инструкции.)

> Не забывайте компилировать файлы `.moon`, когда создаёте
> или изменяете их. Чтобы отслеживать изменения файлов в текущей папке
> и компилировать автоматически, запустите `moonc -w`.

Запустим наш сервер:

```bash
lapis server
```

Если скомпилировали файл `.moon`, тогда наша страница будет
доступна на <http://localhost:8080>.

Члены приложения соответствуют паттернам, с которыми сопоставляются
входящие запросы. Здесь мы имеем дело с так называемыми `путями` и
`обработчиками`, где путь - это паттерн, а обработчик - это функция,
которая обрабатывает соответствующий путь.
В этом примере путь `"/"` соответствует обработчику, который
выводит `"Hello World!"`

Значение, возвращаемое обработчиком, определяет, что попадёт в ответ.
В простейшем случае возвращается строка, которая и пишется в ответ.

> Подробнее о путях и обработчиках написано в разделе [Пути и
> Обработчики][2] guide.

## Приложения Lapis

Когда мы употребляем термин `приложение Lapis`, мы имеем в виду
класс, унаследованный от `lapis.Application`.
Свойства приложения составляют пути, которые обслуживает
приложение, и обработчики, которые оно запускает.

Рассмотрим более сложное приложение.

```moon
lapis = require "lapis"

favorite_foods = {
  "pizza": "Wow pizza is the best! Definitely my favorite"
  "egg": "A classic breakfast, never leave home without"
  "ice cream": "Can't have a food list without a dessert"
}

class App extends lapis.Application
  [index: "/"]: =>
    -- Render HTML inline for simplicity
    @html ->
      h1 "My homepage"
      a href: @url_for("foods"), "Check out my favorite foods"

  [list_foods: "/foods"]: =>
    @html ->
      ul ->
        for food in pairs favorite_foods
          li ->
            a href: @url_for("food", name: food), food

  [food: "/food/:name"]: =>
    food_description = favorite_foods[@params.name]
    unless food_description
      return "Not found", status: 404

    @html ->
      h1 @params.name
      h2 "My thoughts on this food"
      p food_description
```

Компоненты, использующиеся в этом приложении, описаны в
разделе [Запросы и обработчики][2].

## Про MoonScript

Lapis - библиотека для двух языков программирования.
Он работает как с MoonScript, так и с Lua. Нижеизложенное
относится только к MoonScript.
Может оказаться полезным начать с чтения раздела [Запросы и
обработчики][2] перед освоением этого материала.

### Обработчики

Члены класса приложения задают набор правил, которые
сопоставляются с входящими запросами. Мы вводим понятия *путь*
и *обработчик*. Путь - это то самое правило, а обработчик - это
функция, которая обрабатывает совпавший путь.

```moon
lapis = require "lapis"
import respond_to from require "lapis.application"

class App extends lapis.Application
  "/hello": => "Hello World!"
```

В этом примере путь `"/helo"` сопоставили функции-обработчику,
которая возвращает `"Hello World!"`.

Обработчик - это функция, которая вызывается, когда
соответствующий ей путь приложения совпадает с путём запроса.
Обработчики пишутся с жирными стрелками (`=>`), так что
можно подумать, что в `self` передаётся экземпляр приложения,
но на самом деле это экземпляр класса `Request`,
представляющего запрос.

Доступ к объекту-приложению можно получить через переменную
`@app`. Не стоит изменять этот объект, так как он один на
много запросов.

Если имя свойства является строкой и начинается с `"/"` или
свойство является таблицей, то оно определяет путь.
Остальные свойства являются методами приложения.

### Подприложения

По мере усложнения приложения, полезно разбить его на несколько
подприложений. Lapis не предписывает правил деления на
подприложения, но облегчает соединение приложений.

#### `@include(other_application, [opts])`

Допустим, у нас есть отдельное приложение для
обработки пользователей.

```moon
-- applications/users.moon
lapis = require "lapis"

class UsersApplication extends lapis.Application
  [login: "/login"]: do_login!
  [logout: "/logout"]: do_logout!
```

Подключим это приложение к основному приложению:

```moon
-- app.moon
lapis = require "lapis"

class extends lapis.Application
  @include "applications.users"

  [index: "/"]: =>
    @html ->
      a href: @url_for("login"), "Log in"
```

В этом примере `applications/user.moon` - это модуль, который
возвращает подприложение. Метод `include`
(метод класса `Application`) загружает это приложение в
наше корневое приложение. Метод `include` копирует все пути
в другое приложение, не изменяя их в исходном приложении.

Подприложения могут иметь предобработчики, которые выполняются
только для обработчиков, пришедших из подприложения.

Класс подприложения может содержать специальные поля `path`
и `name`:

* `path` -- префикс, добавляемый к паттернам копируемых путей
* `name` -- префикс, добавляемый к именам копируемых путей

```moon
class UsersApplication extends lapis.Application
  @path: "/users"
  @name: "user_"

  -- etc...
```

У `include` есть дополнительный второй аргумент, таблица опций.
Эти опции определяют или переопределяют значения полей
`path` и `name`:

Давайте добавим следующие префиксы приложению `UsersApplication`:

```moon
class extends lapis.Application
  @include "applications.users", path: "/users", name: "user_"

  "/": =>
    @url_for("user_login") -- returns "/users/login"
```

### Методы класса

#### `@find_action(action_name)`

Возвращает функцию-обработчик по имени.

[1]: http://moonscript.org/reference/#moonc
[2]: $root/reference/actions.html