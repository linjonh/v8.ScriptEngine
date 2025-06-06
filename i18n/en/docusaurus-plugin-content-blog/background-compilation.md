---
title: "Background compilation"
author: "[Ross McIlroy](https://twitter.com/rossmcilroy), main thread defender"
avatars: 
  - "ross-mcilroy"
date: "2018-03-26 13:33:37"
tags: 
  - internals
description: "Starting with Chrome 66, V8 compiles JavaScript source code on a background thread, reducing the amount of time spent compiling on the main thread by between 5% to 20% on typical websites."
tweet: "978319362837958657"
---
TL;DR: Starting with Chrome 66, V8 compiles JavaScript source code on a background thread, reducing the amount of time spent compiling on the main thread by between 5% to 20% on typical websites.

## Background

Since version 41, Chrome has supported [parsing of JavaScript source files on a background thread](https://blog.chromium.org/2015/03/new-javascript-techniques-for-rapid.html) via V8’s [`StreamedSource`](https://cs.chromium.org/chromium/src/v8/include/v8.h?q=StreamedSource&sq=package:chromium&l=1389) API. This enables V8 to start parsing JavaScript source code as soon as Chrome has downloaded the first chunk of the file from the network, and to continue parsing in parallel while Chrome streams the file over the network. This can provide considerable loading time improvements since V8 can be almost finished parsing the JavaScript by the time the file has finished downloading.

<!--truncate-->
However, due to limitations in V8’s original baseline compiler, V8 still needed to go back to the main thread to finalize parsing and compile the script into JIT machine code that would execute the script’s code. With the switch to our new [Ignition + TurboFan pipeline](/blog/launching-ignition-and-turbofan), we are now able to move bytecode compilation to the background thread as well, thereby freeing up Chrome’s main-thread to deliver a smoother, more responsive web browsing experience.

## Building a background thread bytecode compiler

V8’s Ignition bytecode compiler takes the [abstract syntax tree (AST)](https://en.wikipedia.org/wiki/Abstract_syntax_tree) produced by the parser as input and produces a stream of bytecode (`BytecodeArray`) along with associated meta-data which enables the Ignition interpreter to execute the JavaScript source.

![](/_img/background-compilation/bytecode.svg)

Ignition’s bytecode compiler was built with multi-threading in mind, however a number of changes were required throughout the compilation pipeline to enable background compilation. One of the main changes was to prevent the compilation pipeline from accessing objects in V8’s JavaScript heap while running on the background thread. Objects in V8’s heap are not thread-safe, since Javascript is single-threaded, and might be modified by the main-thread or V8’s garbage collector during background compilation.

There were two main stages of the compilation pipeline which accessed objects on V8’s heap: AST internalization, and bytecode finalization. AST internalization is a process by which literal objects (strings, numbers, object-literal boilerplate, etc.) identified in the AST are allocated on the V8 heap, such that they can be used directly by the generated bytecode when the script is executed. This process traditionally happened immediately after the parser built the AST. As such, there were a number of steps later in the compilation pipeline that relied on the literal objects having been allocated. To enable background compilation we moved AST internalization later in the compilation pipeline, after the bytecode had been compiled. This required modifications to the later stages of the pipeline to access the _raw_ literal values embedded in the AST instead of internalized on-heap values.

Bytecode finalization involves building the final `BytecodeArray` object, used to execute the function, alongside associated metadata — for example, a `ConstantPoolArray` which stores constants referred to by the bytecode, and a `SourcePositionTable` which maps the JavaScript source line and column numbers to bytecode offset. Since JavaScript is a dynamic language, these objects all need to live in the JavaScript heap to enable them to be garbage-collected if the JavaScript function associated with the bytecode is collected. Previously some of these metadata objects would be allocated and modified during bytecode compilation, which involved accessing the JavaScript heap. In order to enable background compilation, Ignition’s bytecode generator was refactored to keep track of the details of this metadata and defer allocating them on the JavaScript heap until the very final stages of compilation.

With these changes, almost all of the script’s compilation can be moved to a background thread, with only the short AST internalization and bytecode finalization steps happening on the main thread just before script execution.

![](/_img/background-compilation/threads.svg)

Currently, only top-level script code and immediately invoked function expressions (IIFEs) are compiled on a background thread — inner functions are still compiled lazily (when first executed) on the main thread. We are hoping to extend background compilation to more situations in the future. However, even with these restrictions, background compilation leaves the main thread free for longer, enabling it to do other work such as reacting to user-interaction, rendering animations or otherwise producing a smoother more responsive experience.

## Results

We evaluated the performance of background compilation using our [real-world benchmarking framework](/blog/real-world-performance) across a set of popular webpages.

![](/_img/background-compilation/desktop.svg)

![](/_img/background-compilation/mobile.svg)

The proportion of compilation that can happen on a background thread varies depending on the proportion of bytecode compiled during top-level streaming-script compilation verses being lazy compiled as inner functions are invoked (which must still occur on the main thread). As such, the proportion of time saved on the main thread varies, with most pages seeing between 5% to 20% reduction in main-thread compilation time.

## Next steps

What’s better than compiling a script on a background thread? Not having to compile the script at all! Alongside background compilation we have also been working on improving V8’s [code-caching system](/blog/code-caching) to expand the amount of code cached by V8, thereby speeding up page loading for sites you visit often. We hope to bring you updates on this front soon. Stay tuned!
