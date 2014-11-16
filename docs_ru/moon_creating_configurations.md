title: Синтаксис конфигурации на MoonScript
--
<div class="override_lang"></div>

# Синтаксис конфигурации на MoonScript

## Пример конфигурации

Конфигурация на MoonScript использует вызовы функций для
определения переменных. Преимущество такого подхода над
таблицей Lua в возможности подключения логики.
Такую конфигурацию можно совмещать с обычными таблицами Lua.

Пример предметно-ориентированного языка конфигурации (DSL) и
таблица, которую он генерирует:

```moon
some_function = -> steak "medium_well"

config "development", ->
  hello "world"

  if 20 > 4
    color "blue"
  else
    color "green"

  custom_settings ->
    age 10
    enabled true

  -- tables are merged
  extra ->
    name "leaf"
    mood "happy"

  extra ->
    name "beef"
    shoe_size 12

    include some_function


  include some_function

  -- вместо функции можно передать обычную таблицу
  some_list {
    1,2,3,4
  }

  -- используйте set для присвоения недоступных имён
  set "include", "hello"
```

```moon
{
  hello: "world"
  color: "blue"

  custom_settings: {
    age: 10
    enabled: true
  }

  extra: {
    name: "beef"
    mood: "happy"
    shoe_size: 12
    steak: "medium_well"
  }

  steak: "medium_well"

  some_list: { 1,2,3,4 }

  include: "hello"
}
```

