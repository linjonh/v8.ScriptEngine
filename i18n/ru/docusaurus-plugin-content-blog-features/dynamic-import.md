---
title: "Динамический `import()`"
author: "Матиас Байненс ([@mathias](https://twitter.com/mathias))"
avatars: 
  - "mathias-bynens"
date: 2017-11-21
tags: 
  - ECMAScript
  - ES2020
description: "Динамический import() открывает новые возможности по сравнению со статическим импортом. В этой статье сравниваются оба подхода и представлен обзор новшеств."
tweet: "932914724060254208"
---
[Динамический `import()`](https://github.com/tc39/proposal-dynamic-import) представляет новую форму `import`, похожую на функцию, которая открывает новые возможности по сравнению со статическим `import`. В этой статье сравниваются оба варианта и представлен обзор новшеств.

<!--truncate-->
## Статический `import` (резюме)

Chrome 61 добавил поддержку оператора ES2015 `import` в [модулях](/features/modules).

Рассмотрим следующий модуль, расположенный в `./utils.mjs`:

```js
// Экспорт по умолчанию
export default () => {
  console.log('Привет от экспорта по умолчанию!');
};

// Именованный экспорт `doStuff`
export const doStuff = () => {
  console.log('Делаю дела…');
};
```

Вот как можно статически импортировать и использовать модуль `./utils.mjs`:

```html
<script type="module">
  import * as module from './utils.mjs';
  module.default();
  // → выводит 'Привет от экспорта по умолчанию!'
  module.doStuff();
  // → выводит 'Делаю дела…'
</script>
```

:::note
**Примечание:** В предыдущем примере используется расширение `.mjs`, чтобы обозначить, что это модуль, а не обычный скрипт. В вебе расширения файлов не имеют значения, если файлы предоставляются с правильным MIME-типом (например, `text/javascript` для JavaScript-файлов) в заголовке HTTP `Content-Type`.

Расширение `.mjs` особенно полезно на других платформах, таких как [Node.js](https://nodejs.org/api/esm.html#esm_enabling) и [`d8`](/docs/d8), где отсутствует концепция MIME-типов или других обязательных определений, таких как `type="module"`, чтобы определить, является ли файл модулем или обычным скриптом. Мы используем то же расширение здесь для согласованности между платформами и чтобы четко различать модули и обычные скрипты.
:::

Эта синтаксическая форма импорта модулей является *статической* декларацией: она принимает только строковый литерал в качестве идентификатора модуля и вводит привязки в локальную область видимости через предкомпиляционный процесс «связывания». Статический синтаксис `import` может использоваться только на верхнем уровне файла.

Статический `import` открывает важные возможности, такие как статический анализ, инструменты объединения и удаление мертвого кода.

В некоторых случаях полезно:

- импортировать модуль по запросу (или условно)
- вычислять идентификатор модуля во время выполнения
- импортировать модуль из обычного скрипта (а не из модуля)

Ни один из этих случаев невозможен со статическим `import`.

## Динамический `import()` 🔥

[Динамический `import()`](https://github.com/tc39/proposal-dynamic-import) представляет новую форму `import`, похожую на функцию, которая удовлетворяет указанным случаям. `import(moduleSpecifier)` возвращает промис для объекта пространства имен модуля запрашиваемого модуля, который создается после загрузки, инстанцирования и выполнения всех зависимостей модуля, а также самого модуля.

Вот как можно динамически импортировать и использовать модуль `./utils.mjs`:

```html
<script type="module">
  const moduleSpecifier = './utils.mjs';
  import(moduleSpecifier)
    .then((module) => {
      module.default();
      // → выводит 'Привет от экспорта по умолчанию!'
      module.doStuff();
      // → выводит 'Делаю дела…'
    });
</script>
```

Поскольку `import()` возвращает промис, можно использовать `async`/`await` вместо стиля обратных вызовов на основе `then`:

```html
<script type="module">
  (async () => {
    const moduleSpecifier = './utils.mjs';
    const module = await import(moduleSpecifier)
    module.default();
    // → выводит 'Привет от экспорта по умолчанию!'
    module.doStuff();
    // → выводит 'Делаю дела…'
  })();
</script>
```

:::note
**Примечание:** Несмотря на то, что `import()` _выглядит_ как вызов функции, он описан как *синтаксис*, который просто использует скобки (аналогично [`super()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/super)). Это означает, что `import` не наследует от `Function.prototype`, так что его нельзя вызвать через `call` или `apply`, и вещи вроде `const importAlias = import` не работают — более того, `import` даже не является объектом! На практике это не особо важно.
:::

Вот пример, как динамический `import()` позволяет загружать модули «лениво» при переходе в небольшом одностраничном приложении:

```html
<!DOCTYPE html>
<meta charset="utf-8">
<title>Моя библиотека</title>
<nav>
  <a href="books.html" data-entry-module="books">Книги</a>
  <a href="movies.html" data-entry-module="movies">Фильмы</a>
  <a href="video-games.html" data-entry-module="video-games">Видеоигры</a>
</nav>
<main>Это запасное место для контента, который будет загружен по запросу.</main>
<script>
  const main = document.querySelector('main');
  const links = document.querySelectorAll('nav > a');
  for (const link of links) {
    link.addEventListener('click', async (event) => {
      event.preventDefault();
      try {
        const module = await import(`/${link.dataset.entryModule}.mjs`);
        // Модуль экспортирует функцию под названием `loadPageInto`.
        module.loadPageInto(main);
      } catch (error) {
        main.textContent = error.message;
      }
    });
  }
</script>
```

Возможности ленивой загрузки, которые предоставляет динамический `import()`, могут быть очень мощными при правильном применении. В качестве демонстрации [Addy](https://twitter.com/addyosmani) модифицировал [пример PWA Hacker News](https://hnpwa-vanilla.firebaseapp.com/), который изначально статически импортировал все зависимости, включая комментарии, при первой загрузке. [Обновленная версия](https://dynamic-import.firebaseapp.com/) использует динамический `import()`, чтобы лениво загружать комментарии, избегая затрат на загрузку, разбор и компиляцию до момента, когда они действительно понадобятся пользователю.

:::note
**Примечание:** Если ваше приложение импортирует скрипты с другого домена (статически или динамически), необходимо, чтобы скрипты возвращались с действительными заголовками CORS (например, `Access-Control-Allow-Origin: *`). Это связано с тем, что, в отличие от обычных скриптов, модульные скрипты (и их импорты) запрашиваются с использованием CORS.
:::

## Рекомендации

Статический `import` и динамический `import()` одинаково полезны. Каждый из них имеет свои собственные, очень специфичные, случаи использования. Используйте статический `import` для зависимостей, необходимых для первоначальной отрисовки, особенно для контента выше границы прокрутки. В других случаях рассмотрите возможность подгрузки зависимостей по запросу с помощью динамического `import()`.

## Поддержка динамического `import()`

<feature-support chrome="63"
                 firefox="67"
                 safari="11.1"
                 nodejs="13.2 https://nodejs.medium.com/announcing-core-node-js-support-for-ecmascript-modules-c5d6dc29b663"
                 babel="yes https://babeljs.io/docs/en/babel-plugin-syntax-dynamic-import"></feature-support>
