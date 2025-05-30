---
title: "`Symbol.prototype.description`"
author: "Mathias Bynens ([@mathias](https://twitter.com/mathias))"
avatars: 
  - "mathias-bynens"
date: 2019-06-25
tags: 
  - ECMAScript
  - ES2019
description: "Symbol.prototype.description は、Symbol の説明を取得するための使いやすい方法を提供します。"
tweet: "1143432835665211394"
---
JavaScript の `Symbol` は作成時に説明を付けることができます:

```js
const symbol = Symbol('foo');
//                    ^^^^^
```

以前は、この説明にプログラム的にアクセスする唯一の方法は `Symbol.prototype.toString()` を間接的に使用することでした:

```js
const symbol = Symbol('foo');
//                    ^^^^^
symbol.toString();
// → 'Symbol(foo)'
//           ^^^
symbol.toString().slice(7, -1); // 🤔
// → 'foo'
```

しかし、このコードは少し魔法のようで自明ではなく、“意図を表現し、実装を示さない”という原則に反しています。また、このテクニックでは説明がないシンボル (例: `Symbol()`) と、空の文字列を説明として持つシンボル (例: `Symbol('')`) を区別することができません。

<!--truncate-->
[新しい `Symbol.prototype.description` のゲッター](https://tc39.es/ecma262/#sec-symbol.prototype.description) は、`Symbol` の説明にアクセスするためのより使いやすい方法を提供します:

```js
const symbol = Symbol('foo');
//                    ^^^^^
symbol.description;
// → 'foo'
```

説明のない `Symbol` に対しては、ゲッターは `undefined` を返します:

```js
const symbol = Symbol();
symbol.description;
// → undefined
```

## `Symbol.prototype.description` の対応状況

<feature-support chrome="70 /blog/v8-release-70#javascript-language-features"
                 firefox="63"
                 safari="12.1"
                 nodejs="12 https://twitter.com/mathias/status/1120700101637353473"
                 babel="yes https://github.com/zloirock/core-js#ecmascript-symbol"></feature-support>
