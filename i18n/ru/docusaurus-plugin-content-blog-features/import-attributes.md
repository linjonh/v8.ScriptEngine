---
title: "Импорт атрибутов"
author: "Шу-ю Гуо ([@_shu](https://twitter.com/_shu))"
avatars: 
  - "shu-yu-guo"
date: 2024-01-31
tags: 
  - ECMAScript
description: "Импорт атрибутов: эволюция утверждений импорта"
tweet: ""
---

## Ранее

V8 реализовал функцию [утверждений импорта](https://chromestatus.com/feature/5765269513306112) в версии 9.1. Эта функция позволяет инструкциям импорта модулей включать дополнительную информацию, используя ключевое слово `assert`. В настоящее время эта дополнительная информация используется для импорта JSON- и CSS-модулей внутри JavaScript-модулей.

<!--truncate-->
## Атрибуты импорта

С тех пор утверждения импорта эволюционировали в [атрибуты импорта](https://github.com/tc39/proposal-import-attributes). Суть функции остается та же: позволить инструкциям импорта модулей включать дополнительную информацию.

Самое важное отличие заключается в том, что утверждения импорта имели семантику только утверждений, тогда как атрибуты импорта имеют более расслабленную семантику. Семантика только утверждений означает, что дополнительная информация не влияет на _то, как_ загружается модуль, а только на _то, будет ли_ он загружен. Например, JSON-модуль всегда загружается как JSON-модуль благодаря своему MIME-типу, а конструкция `assert { type: 'json' }` может лишь вызвать сбой загрузки, если MIME-тип запрашиваемого модуля не является `application/json`.

Однако у семантики только утверждений был фатальный недостаток. В вебе форма HTTP-запросов различается в зависимости от типа запрашиваемого ресурса. Например, заголовок [`Accept`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept) влияет на MIME-тип ответа, а метаданный заголовок [`Sec-Fetch-Dest`](https://web.dev/articles/fetch-metadata) влияет на то, примет ли веб-сервер запрос или отклонит его. Поскольку утверждение импорта не могло повлиять на _то, как_ загружать модуль, оно не могло изменить форму HTTP-запроса. Тип запрашиваемого ресурса также влияет на то, какие [Политики безопасности содержимого](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) используются: утверждения импорта не могли корректно работать с моделью безопасности веба.

Атрибуты импорта смягчают семантику только утверждений, позволяя атрибутам влиять на то, как загружается модуль. Другими словами, атрибуты импорта могут генерировать HTTP-запросы, содержащие соответствующие заголовки `Accept` и `Sec-Fetch-Dest`. Чтобы синтаксис соответствовал новой семантике, старое ключевое слово `assert` обновляется на `with`:

```javascript
// main.mjs
//
// Новый синтаксис 'with'.
import json from './foo.json' with { type: 'json' };
console.log(json.answer); // 42
```

## Динамический `import()`

Аналогично, [динамический `import()`](https://v8.dev/features/dynamic-import#dynamic) также обновляется, чтобы принимать опцию `with`.

```javascript
// main.mjs
//
// Новая опция 'with'.
const jsonModule = await import('./foo.json', {
  with: { type: 'json' }
});
console.log(jsonModule.default.answer); // 42
```

## Доступность `with`

Атрибуты импорта включены по умолчанию в V8 версии 12.3.

## Устаревание и последующее удаление `assert`

Ключевое слово `assert` устарело начиная с версии V8 12.3 и планируется быть удаленным к версии 12.6. Пожалуйста, используйте `with` вместо `assert`! Использование конструкции `assert` будет выводить предупреждение в консоль с просьбой использовать `with` вместо этого.

## Поддержка атрибутов импорта

<feature-support chrome="123 https://chromestatus.com/feature/5205869105250304"
                 firefox="no"
                 safari="17.2 https://developer.apple.com/documentation/safari-release-notes/safari-17_2-release-notes"
                 nodejs="20.10 https://nodejs.org/docs/latest-v20.x/api/esm.html#import-attributes"
                 babel="yes https://babeljs.io/blog/2023/05/26/7.22.0#import-attributes-15536-15620"></feature-support>
