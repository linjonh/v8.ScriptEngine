---
title: "Launching Ignition and TurboFan"
author: "the V8 team"
date: "2017-05-15 13:33:37"
tags: 
  - internals
description: "V8 v5.9 comes with a brand-new JavaScript execution pipeline, built upon the Ignition interpreter and the TurboFan optimizing compiler."
---
Today we are excited to announce the launch of a new JavaScript execution pipeline for V8 v5.9 that will reach Chrome Stable in v59. With the new pipeline, we achieve big performance improvements and significant memory savings on real-world JavaScript applications. We’ll discuss the numbers in more detail at the end of this post, but first let’s take a look at the pipeline itself.

<!--truncate-->
The new pipeline is built upon [Ignition](/docs/ignition), V8’s interpreter, and [TurboFan](/docs/turbofan), V8’s newest optimizing compiler. These technologies [should](/blog/turbofan-jit) [be](/blog/ignition-interpreter) [familiar](/blog/test-the-future) to those of you who have followed the V8 blog over the last few years, but the switch to the new pipeline marks a big new milestone for both.

<figure>
  <img src="/_img/v8-ignition.svg" width="256" height="256" alt="" loading="lazy"/>
  <figcaption>Logo for Ignition, V8’s brand-new interpreter</figcaption>
</figure>

<figure>
  <img src="/_img/v8-turbofan.svg" width="256" height="256" alt="" loading="lazy"/>
  <figcaption>Logo for TurboFan, V8’s brand-new optimizing compiler</figcaption>
</figure>

For the first time, Ignition and TurboFan are used universally and exclusively for JavaScript execution in V8 v5.9. Furthermore, starting with v5.9, Full-codegen and Crankshaft, the technologies that [served V8 well since 2010](https://blog.chromium.org/2010/12/new-crankshaft-for-v8.html), are no longer used in V8 for JavaScript execution, since they no longer are able to keep pace with new JavaScript language features and the optimizations those features require. We plan to remove them completely very soon. That means that V8 will have an overall much simpler and more maintainable architecture going forward.

## A long journey

The combined Ignition and TurboFan pipeline has been in development for almost 3½ years. It represents the culmination of the collective insight that the V8 team has gleaned from measuring real-world JavaScript performance and carefully considering the shortcomings of Full-codegen and Crankshaft. It is a foundation with which we will be able to continue to optimize the entirety of the JavaScript language for years to come.

The TurboFan project originally started in late 2013 to address the shortcomings of Crankshaft. Crankshaft can only optimize a subset of the JavaScript language. For example, it was not designed to optimize JavaScript code using structured exception handling, i.e. code blocks demarcated by JavaScript’s try, catch, and finally keywords. It is difficult to add support for new language features in Crankshaft, since these features almost always require writing architecture-specific code for nine supported platforms. Furthermore, Crankshaft’s architecture is limited in the extent that it can generate optimal machine code. It can only squeeze so much performance out of JavaScript, despite requiring the V8 team to maintain more than ten thousand lines of code per chip architecture.

TurboFan was designed from the beginning not only to optimize all of the language features found in the JavaScript standard at the time, ES5, but also all the future features planned for ES2015 and beyond. It introduces a layered compiler design that enables a clean separation between high-level and low-level compiler optimizations, making it easy to add new language features without modifying architecture-specific code. TurboFan adds an explicit instruction selection compilation phase that makes it possible to write far less architecture-specific code for each supported platform in the first place. With this new phase, architecture-specific code is written once and it rarely needs to be changed. These and other decisions lead to a more maintainable and extensible optimizing compiler for all of the architectures that V8 supports.

The original motivation behind V8’s Ignition interpreter was to reduce memory consumption on mobile devices. Before Ignition, the code generated by V8’s Full-codegen baseline compiler typically occupied almost one third of the overall JavaScript heap in Chrome. That left less space for a web application’s actual data. When Ignition was enabled for Chrome M53 on Android devices with limited RAM, the memory footprint required for baseline, non-optimized JavaScript code shrank by a factor of nine on ARM64-based mobile devices.

Later the V8 team took advantage of the fact that Ignition’s bytecode can be used to generate optimized machine code with TurboFan directly rather than having to re-compile from source code as Crankshaft did. Ignition’s bytecode provides a cleaner and less error-prone baseline execution model in V8, simplifying the deoptimization mechanism that is a key feature of V8’s [adaptive optimization](https://en.wikipedia.org/wiki/Adaptive_optimization). Finally, since generating bytecode is faster than generating Full-codegen’s baseline compiled code, activating Ignition generally improves script startup times and in turn, web page loads.

By coupling the design of Ignition and TurboFan closely, there are even more benefits to the overall architecture. For example, rather than writing Ignition’s high-performance bytecode handlers in hand-coded assembly, the V8 team instead uses TurboFan’s [intermediate representation](https://en.wikipedia.org/wiki/Intermediate_representation) to express the handlers’ functionality and lets TurboFan do the optimization and final code generation for V8’s numerous supported platforms. This ensures Ignition performs well on all of V8’s supported chip architectures while simultaneously eliminating the burden of maintaining nine separate platform ports.

## Running the numbers

History aside, now let’s take a look at the new pipeline’s real-world performance and memory consumption.

The V8 team continually monitors the performance of real-world use cases using the [Telemetry - Catapult](https://catapult.gsrc.io/telemetry) framework. [Previously](/blog/real-world-performance) in this blog we’ve discussed why it’s so important to use the data from real-world tests to drive our performance optimization work and how we use [WebPageReplay](https://github.com/chromium/web-page-replay) together with Telemetry to do so. The switch to Ignition and TurboFan shows performance improvements in those real-world test cases. Specifically, the new pipeline results in significant speed-ups on user interaction story tests for well-known websites:

![Reduction in time spent in V8 for user interaction benchmarks](/_img/launching-ignition-and-turbofan/improvements-per-website.png)

Although Speedometer is a synthetic benchmark, we’ve previously uncovered that it does a better job of approximating the real-world workloads of modern JavaScript than other synthetic benchmarks. The switch to Ignition and TurboFan improves V8’s Speedometer score by 5%-10%, depending on platform and device.

The new pipeline also speeds up server-side JavaScript. [AcmeAir](https://github.com/acmeair/acmeair-nodejs), a benchmark for Node.js that simulates the server backend implementation of a fictitious airline, runs more than 10% faster using V8 v5.9.

![Improvements on Web and Node.js benchmarks](/_img/launching-ignition-and-turbofan/benchmark-scores.png)

Ignition and TurboFan also reduce V8’s overall memory footprint. In Chrome M59, the new pipeline slims V8’s memory footprint on desktop and high-end mobile devices by 5-10%. This reduction is a result of bringing the Ignition memory savings that have been [previously covered](/blog/ignition-interpreter) in this blog to all devices and platforms supported by V8.

These improvements are just the start. The new Ignition and TurboFan pipeline paves the way for further optimizations that will boost JavaScript performance and shrink V8’s footprint in both Chrome and in Node.js for years to come. We look forward to sharing those improvements with you as we roll them out to developers and users. Stay tuned.
