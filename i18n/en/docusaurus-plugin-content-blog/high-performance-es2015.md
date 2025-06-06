---
title: "High-performance ES2015 and beyond"
author: "Benedikt Meurer [@bmeurer](https://twitter.com/bmeurer), ECMAScript Performance Engineer"
avatars: 
  - "benedikt-meurer"
date: "2017-02-17 13:33:37"
tags: 
  - ECMAScript
description: "V8’s performance of ES2015+ language features is now on par with their transpiled ES5 counterparts."
---
Over the last couple of months the V8 team focused on bringing the performance of newly added [ES2015](https://www.ecma-international.org/ecma-262/6.0/) and other even more recent JavaScript features on par with their transpiled [ES5](https://www.ecma-international.org/ecma-262/5.1/) counterparts.

<!--truncate-->
## Motivation

Before we go into the details of the various improvements, we should first consider why performance of ES2015+ features matter despite the widespread usage of [Babel](http://babeljs.io/) in modern web development:

1. First of all there are new ES2015 features that are only polyfilled on demand, for example the [`Object.assign`](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Object/assign) builtin. When Babel transpiles [object spread properties](https://github.com/sebmarkbage/ecmascript-rest-spread) (which are heavily used by many [React](https://facebook.github.io/react) and [Redux](http://redux.js.org/) applications), it relies on [`Object.assign`](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Object/assign) instead of an ES5 equivalent if the VM supports it.
1. Polyfilling ES2015 features typically increases code size, which contributes significantly to the current [web performance crisis](https://channel9.msdn.com/Blogs/msedgedev/nolanlaw-web-perf-crisis), especially on mobile devices common in emerging markets. So the cost of just delivering, parsing and compiling the code can be fairly high, even before you get to the actual execution cost.
1. And last but not least, the client-side JavaScript is only one of the environments that relies on the V8 engine. There’s also [Node.js](https://nodejs.org/) for server side applications and tools, where developers don’t need to transpile to ES5 code, but can directly use the features supported by the [relevant V8 version](https://nodejs.org/en/download/releases/) in the target Node.js release.

Let’s consider the following code snippet from the [Redux documentation](http://redux.js.org/docs/recipes/UsingObjectSpreadOperator.html):

```js
function todoApp(state = initialState, action) {
  switch (action.type) {
    case SET_VISIBILITY_FILTER:
      return { ...state, visibilityFilter: action.filter };
    default:
      return state;
  }
}
```

There are two things in that code that demand transpilation: the default parameter for state and the spreading of state into the object literal. Babel generates the following ES5 code:

```js
'use strict';

var _extends = Object.assign || function(target) {
  for (var i = 1; i < arguments.length; i++) {
    var source = arguments[i];
    for (var key in source) {
      if (Object.prototype.hasOwnProperty.call(source, key)) {
        target[key] = source[key];
      }
    }
  }
  return target;
};

function todoApp() {
  var state = arguments.length > 0 && arguments[0] !== undefined ? arguments[0] : initialState;
  var action = arguments[1];

  switch (action.type) {
    case SET_VISIBILITY_FILTER:
      return _extends({}, state, { visibilityFilter: action.filter });
    default:
      return state;
  }
}
```

Now imagine that [`Object.assign`](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Object/assign) is orders of magnitude slower than the polyfilled `_extends` generated by Babel. In that case upgrading from a browser that doesn’t support [`Object.assign`](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Object/assign) to an ES2015 capable version of the browser would be a serious performance regression and probably hinder adoption of ES2015 in the wild.

This example also highlights another important drawback of transpilation: The generated code that is shipped to the user is usually considerably bigger than the ES2015+ code that the developer initially wrote. In the example above, the original code is 203 characters (176 bytes gzipped) whereas the generated code is 588 characters (367 bytes gzipped). That’s already a factor of two increase in size. Let’s look at another example from the [async iterators](https://github.com/tc39/proposal-async-iteration) proposal:

```js
async function* readLines(path) {
  let file = await fileOpen(path);
  try {
    while (!file.EOF) {
      yield await file.readLine();
    }
  } finally {
    await file.close();
  }
}
```

Babel translates these 187 characters (150 bytes gzipped) into a whopping 2987 characters (971 bytes gzipped) of ES5 code, not even counting the [regenerator runtime](https://babeljs.io/docs/plugins/transform-regenerator/) that is required as an additional dependency:

```js
'use strict';

var _asyncGenerator = function() {
  function AwaitValue(value) {
    this.value = value;
  }

  function AsyncGenerator(gen) {
    var front, back;

    function send(key, arg) {
      return new Promise(function(resolve, reject) {
        var request = {
          key: key,
          arg: arg,
          resolve: resolve,
          reject: reject,
          next: null
        };
        if (back) {
          back = back.next = request;
        } else {
          front = back = request;
          resume(key, arg);
        }
      });
    }

    function resume(key, arg) {
      try {
        var result = gen[key](arg);
        var value = result.value;
        if (value instanceof AwaitValue) {
          Promise.resolve(value.value).then(function(arg) {
            resume('next', arg);
          }, function(arg) {
            resume('throw', arg);
          });
        } else {
          settle(result.done ? 'return' : 'normal', result.value);
        }
      } catch (err) {
        settle('throw', err);
      }
    }

    function settle(type, value) {
      switch (type) {
        case 'return':
          front.resolve({
            value: value,
            done: true
          });
          break;
        case 'throw':
          front.reject(value);
          break;
        default:
          front.resolve({
            value: value,
            done: false
          });
          break;
      }
      front = front.next;
      if (front) {
        resume(front.key, front.arg);
      } else {
        back = null;
      }
    }
    this._invoke = send;
    if (typeof gen.return !== 'function') {
      this.return = undefined;
    }
  }
  if (typeof Symbol === 'function' && Symbol.asyncIterator) {
    AsyncGenerator.prototype[Symbol.asyncIterator] = function() {
      return this;
    };
  }
  AsyncGenerator.prototype.next = function(arg) {
    return this._invoke('next', arg);
  };
  AsyncGenerator.prototype.throw = function(arg) {
    return this._invoke('throw', arg);
  };
  AsyncGenerator.prototype.return = function(arg) {
    return this._invoke('return', arg);
  };
  return {
    wrap: function wrap(fn) {
      return function() {
        return new AsyncGenerator(fn.apply(this, arguments));
      };
    },
    await: function await (value) {
      return new AwaitValue(value);
    }
  };
}();

var readLines = function () {
  var _ref = _asyncGenerator.wrap(regeneratorRuntime.mark(function _callee(path) {
    var file;
    return regeneratorRuntime.wrap(function _callee$(_context) {
      while (1) {
        switch (_context.prev = _context.next) {
          case 0:
            _context.next = 2;
            return _asyncGenerator.await(fileOpen(path));

          case 2:
            file = _context.sent;
            _context.prev = 3;

          case 4:
            if (file.EOF) {
              _context.next = 11;
              break;
            }

            _context.next = 7;
            return _asyncGenerator.await(file.readLine());

          case 7:
            _context.next = 9;
            return _context.sent;

          case 9:
            _context.next = 4;
            break;

          case 11:
            _context.prev = 11;
            _context.next = 14;
            return _asyncGenerator.await(file.close());

          case 14:
            return _context.finish(11);

          case 15:
          case 'end':
            return _context.stop();
        }
      }
    }, _callee, this, [[3,, 11, 15]]);
  }));

  return function readLines(_x) {
    return _ref.apply(this, arguments);
  };
}();
```

This is a **650%** increase in size (the generic `_asyncGenerator` function might be shareable depending on how you bundle your code, so you can amortize some of that cost across multiple uses of async iterators). We don’t think it’s viable to ship only code transpiled to ES5 long-term, as the increase in size will not only affect download time/cost, but will also add additional overhead to parsing and compilation. If we really want to drastically improve page load and snappiness of modern web applications, especially on mobile devices, we have to encourage developers to not only use ES2015+ when writing code, but also to ship that instead of transpiling to ES5. Only deliver fully transpiled bundles to legacy browsers that don’t support ES2015. For VM implementors, this vision means we need to support ES2015+ features natively **and** provide reasonable performance.

## Measurement methodology

As described above, absolute performance of ES2015+ features is not really an issue at this point. Instead the highest priority currently is to ensure that performance of ES2015+ features is on par with their naive ES5 and even more importantly, with the version generated by Babel. Conveniently there was already a project called [SixSpeed](https://github.com/kpdecker/six-speed) by [Kevin Decker](http://www.incaseofstairs.com/), that accomplishes more or less exactly what we needed: a performance comparison of ES2015 features vs. naive ES5 vs. code generated by transpilers.

![The SixSpeed benchmark](/_img/high-performance-es2015/sixspeed.png)

So we decided to take that as the basis for our initial ES2015+ performance work. We [forked SixSpeed](https://fhinkel.github.io/six-speed/) and added a couple of benchmarks. We focused on the most serious regressions first, i.e. line items where slowdown from naive ES5 to recommended ES2015+ version was above 2x, because our fundamental assumption is that the naive ES5 version will be at least as fast as the somewhat spec-compliant version that Babel generates.

## A modern architecture for a modern language

In the past V8’s had difficulties optimizing the kind of language features that are found in ES2015+. For example, it never became feasible to add exception handling (i.e. try/catch/finally) support to Crankshaft, V8’s classic optimizing compiler. This meant V8’s ability to optimize an ES6 feature like for...of, which essentially has an implicit finally clause, was limited. Crankshaft’s limitations and the overall complexity of adding new language features to full-codegen, V8’s baseline compiler, made it inherently difficult to ensure new ES features were added and optimized in V8 as quickly as they were standardized.

Fortunately, Ignition and TurboFan ([V8’s new interpreter and compiler pipeline](/blog/test-the-future)), were designed to support the entire JavaScript language from the beginning, including advanced control flow, exception handling, and most recently `for`-`of` and destructuring from ES2015. The tight integration of the architecture of Ignition and TurboFan make it possible to quickly add new features and to optimize them fast and incrementally.

Many of the improvements we achieved for modern language features were only feasible with the new Ignition/TurboFan pipeline. Ignition and TurboFan proved especially critical to optimizing generators and async functions. Generators had long been supported by V8, but were not optimizable due to control flow limitations in Crankshaft. Async functions are essentially sugar on top of generators, so they fall into the same category. The new compiler pipeline leverages Ignition to make sense of the AST and generate bytecodes which de-sugar complex generator control flow into simpler local-control flow bytecodes. TurboFan can more easily optimize the resulting bytecodes since it doesn’t need to know anything specific about generator control flow, just how to save and restore a function’s state on yields.

![How JavaScript generators are represented in Ignition and TurboFan](/_img/high-performance-es2015/generators.svg)

## State of the union

Our short-term goal was to reach less than 2× slowdown on average as soon as possible. We started by looking at the worst test first, and from Chrome 54 to Chrome 58 (Canary) we managed to reduce the number of tests with slowdown above 2× from 16 to 8, and at the same time reduce the worst slowdown from 19× in Chrome 54 to just 6× in Chrome 58 (Canary). We also significantly reduced the average and median slowdown during that period:

![Slowdown of ES2015+ compared to native ES5 equivalent](/_img/high-performance-es2015/slowdown.svg)

You can see a clear trend towards parity of ES2015+ and ES5. On average we improved performance relative to ES5 by over 47%. Here are some highlights that we addressed since Chrome 54.

![ES2015+ performance compared to naive ES5 equivalent](/_img/high-performance-es2015/comparison.svg)

Most notably we improved performance of new language constructs that are based on iteration, like the spread operator, destructuring and `for`-`of` loops. For example, using array destructuring:

```js
function fn() {
  var [c] = data;
  return c;
}
```

…is now as fast as the naive ES5 version:

```js
function fn() {
  var c = data[0];
  return c;
}
```

…and a lot faster (and shorter) than the Babel-generated code:

```js
'use strict';

var _slicedToArray = function() {
  function sliceIterator(arr, i) {
    var _arr = [];
    var _n = true;
    var _d = false;
    var _e = undefined;
    try {
      for (var _i = arr[Symbol.iterator](), _s; !(_n = (_s = _i.next()).done); _n = true) {
        _arr.push(_s.value);
        if (i && _arr.length === i) break;
      }
    } catch (err) {
      _d = true;
      _e = err;
    } finally {
      try {
        if (!_n && _i['return']) _i['return']();
      } finally {
        if (_d) throw _e;
      }
    }
    return _arr;
  }
  return function(arr, i) {
    if (Array.isArray(arr)) {
      return arr;
    } else if (Symbol.iterator in Object(arr)) {
      return sliceIterator(arr, i);
    } else {
      throw new TypeError('Invalid attempt to destructure non-iterable instance');
    }
  };
}();

function fn() {
  var _data = data,
      _data2 = _slicedToArray(_data, 1),
      c = _data2[0];

  return c;
}
```

You can check out the [High-Speed ES2015](https://docs.google.com/presentation/d/1wiiZeRQp8-sXDB9xXBUAGbaQaWJC84M5RNxRyQuTmhk) talk we gave at the last [Munich NodeJS User Group](http://www.mnug.de/) meetup for additional details:

We are committed to continue improving the performance of ES2015+ features. In case you are interested in the nitty-gritty details please have a look at V8’s [ES2015 and beyond performance plan](https://docs.google.com/document/d/1EA9EbfnydAmmU_lM8R_uEMQ-U_v4l9zulePSBkeYWmY).
