---
title: "Public and private class fields"
author: "Mathias Bynens ([@mathias](https://twitter.com/mathias))"
avatars: 
  - "mathias-bynens"
date: 2018-12-13
tags: 
  - ECMAScript
  - ES2022
  - io19
  - Node.js 14
description: "Several proposals expand the existing JavaScript class syntax with new functionality. This article explains the new public class fields syntax in V8 v7.2 and Chrome 72, as well as the upcoming private class fields syntax."
tweet: "1121395767170740225"
---
Several proposals expand the existing JavaScript class syntax with new functionality. This article explains the new public class fields syntax in V8 v7.2 and Chrome 72, as well as the upcoming private class fields syntax.

Here’s a code example that creates an instance of a class named `IncreasingCounter`:

```js
const counter = new IncreasingCounter();
counter.value;
// logs 'Getting the current value!'
// → 0
counter.increment();
counter.value;
// logs 'Getting the current value!'
// → 1
```

Note that accessing the `value` executes some code (i.e., it logs a message) before returning the result. Now ask yourself, how would you implement this class in JavaScript? 🤔

## ES2015 class syntax

Here’s how `IncreasingCounter` could be implemented using ES2015 class syntax:

```js
class IncreasingCounter {
  constructor() {
    this._count = 0;
  }
  get value() {
    console.log('Getting the current value!');
    return this._count;
  }
  increment() {
    this._count++;
  }
}
```

The class installs the `value` getter and an `increment` method on the prototype. More interestingly, the class has a constructor that creates an instance property `_count` and sets its default value to `0`. We currently tend to use the underscore prefix to denote that `_count` should not be used directly by consumers of the class, but that’s just a convention; it’s not _really_ a “private” property with special semantics enforced by the language.

<!--truncate-->
```js
const counter = new IncreasingCounter();
counter.value;
// logs 'Getting the current value!'
// → 0

// Nothing stops people from reading or messing with the
// `_count` instance property. 😢
counter._count;
// → 0
counter._count = 42;
counter.value;
// logs 'Getting the current value!'
// → 42
```

## Public class fields

The new public class fields syntax allows us to simplify the class definition:

```js
class IncreasingCounter {
  _count = 0;
  get value() {
    console.log('Getting the current value!');
    return this._count;
  }
  increment() {
    this._count++;
  }
}
```

The `_count` property is now nicely declared at the top of the class. We no longer need a constructor just to define some fields. Neat!

However, the `_count` field is still a public property. In this particular example, we want to prevent people from accessing the property directly.

## Private class fields

That’s where private class fields come in. The new private fields syntax is similar to public fields, except [you mark the field as being private by using `#`](https://github.com/tc39/proposal-class-fields/blob/master/PRIVATE_SYNTAX_FAQ.md). You can think of the `#` as being part of the field name:

```js
class IncreasingCounter {
  #count = 0;
  get value() {
    console.log('Getting the current value!');
    return this.#count;
  }
  increment() {
    this.#count++;
  }
}
```

Private fields are not accessible outside of the class body:

```js
const counter = new IncreasingCounter();
counter.#count;
// → SyntaxError
counter.#count = 42;
// → SyntaxError
```

## Public and private static properties

Class fields syntax can be used to create public and private static properties and methods as well:

```js
class FakeMath {
  // `PI` is a static public property.
  static PI = 22 / 7; // Close enough.

  // `#totallyRandomNumber` is a static private property.
  static #totallyRandomNumber = 4;

  // `#computeRandomNumber` is a static private method.
  static #computeRandomNumber() {
    return FakeMath.#totallyRandomNumber;
  }

  // `random` is a static public method (ES2015 syntax)
  // that consumes `#computeRandomNumber`.
  static random() {
    console.log('I heard you like random numbers…');
    return FakeMath.#computeRandomNumber();
  }
}

FakeMath.PI;
// → 3.142857142857143
FakeMath.random();
// logs 'I heard you like random numbers…'
// → 4
FakeMath.#totallyRandomNumber;
// → SyntaxError
FakeMath.#computeRandomNumber();
// → SyntaxError
```

## Simpler subclassing

The benefits of the class fields syntax become even clearer when dealing with subclasses that introduce additional fields. Imagine the following base class `Animal`:

```js
class Animal {
  constructor(name) {
    this.name = name;
  }
}
```

To create a `Cat` subclass that introduces an additional instance property, you’d previously have to call `super()` to run the constructor of the `Animal` base class before creating the property:

```js
class Cat extends Animal {
  constructor(name) {
    super(name);
    this.likesBaths = false;
  }
  meow() {
    console.log('Meow!');
  }
}
```

That’s a lot of boilerplate just to indicate that cats don’t enjoy taking baths. Luckily, the class fields syntax removes the need for the whole constructor, including the awkward `super()` call:

```js
class Cat extends Animal {
  likesBaths = false;
  meow() {
    console.log('Meow!');
  }
}
```

## Feature support

### Support for public class fields

<feature-support chrome="72 /blog/v8-release-72#public-class-fields"
                 firefox="yes https://developer.mozilla.org/en-US/docs/Mozilla/Firefox/Releases/69#JavaScript"
                 safari="yes https://bugs.webkit.org/show_bug.cgi?id=174212"
                 nodejs="12 https://twitter.com/mathias/status/1120700101637353473"
                 babel="yes https://babeljs.io/docs/en/babel-plugin-proposal-class-properties"></feature-support>

### Support for private class fields

<feature-support chrome="74 /blog/v8-release-74#private-class-fields"
                 firefox="90 https://spidermonkey.dev/blog/2021/05/03/private-fields-ship.html"
                 safari="yes"
                 nodejs="12 https://twitter.com/mathias/status/1120700101637353473"
                 babel="yes https://babeljs.io/docs/en/babel-plugin-proposal-class-properties"></feature-support>

### Support for private methods and accessors

<feature-support chrome="84 /blog/v8-release-84#private-methods-and-accessors"
                 firefox="90 https://spidermonkey.dev/blog/2021/05/03/private-fields-ship.html"
                 safari="yes https://webkit.org/blog/11989/new-webkit-features-in-safari-15/"
                 nodejs="14.6.0"
                 babel="yes https://babeljs.io/docs/en/babel-plugin-proposal-private-methods"></feature-support>
