---
title: "`Symbol.prototype.description`"
author: "Mathias Bynens ([@mathias](https://twitter.com/mathias))"
avatars: 
  - "mathias-bynens"
date: 2019-06-25
tags: 
  - ECMAScript
  - ES2019
description: "Symbol.prototype.description proporciona una forma ergonómica de acceder a la descripción de un símbolo."
tweet: "1143432835665211394"
---
Los `Symbol`s de JavaScript pueden tener una descripción al momento de su creación:

```js
const symbol = Symbol('foo');
//                    ^^^^^
```

Anteriormente, la única forma de acceder a esta descripción programáticamente era indirectamente a través de `Symbol.prototype.toString()`:

```js
const symbol = Symbol('foo');
//                    ^^^^^
symbol.toString();
// → 'Symbol(foo)'
//           ^^^
symbol.toString().slice(7, -1); // 🤔
// → 'foo'
```

Sin embargo, el código es un tanto mágico, no muy autoexplicativo, y viola el principio de “expresar intención, no implementación”. La técnica mencionada tampoco permite distinguir entre un símbolo sin descripción (es decir, `Symbol()`) y un símbolo cuya descripción es la cadena vacía (es decir, `Symbol('')`).

<!--truncate-->
[El nuevo getter `Symbol.prototype.description`](https://tc39.es/ecma262/#sec-symbol.prototype.description) proporciona una forma más ergonómica de acceder a la descripción de un `Symbol`:

```js
const symbol = Symbol('foo');
//                    ^^^^^
symbol.description;
// → 'foo'
```

Para `Symbol`s sin una descripción, el getter devuelve `undefined`:

```js
const symbol = Symbol();
symbol.description;
// → undefined
```

## Soporte para `Symbol.prototype.description`

<feature-support chrome="70 /blog/v8-release-70#javascript-language-features"
                 firefox="63"
                 safari="12.1"
                 nodejs="12 https://twitter.com/mathias/status/1120700101637353473"
                 babel="yes https://github.com/zloirock/core-js#ecmascript-symbol"></feature-support>
