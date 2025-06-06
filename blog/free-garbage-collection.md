---
title: "免费获得垃圾回收"
author: "Hannes Payer和Ross McIlroy，Idle垃圾收集器"
avatars: 
  - "hannes-payer"
  - "ross-mcilroy"
date: "2015-08-07 13:33:37"
tags: 
  - 内部结构
  - 内存
description: "Chrome 41在小的闲置时间片段中隐藏了昂贵的内存管理操作，从而减少卡顿现象。"
---
JavaScript性能仍然是Chrome核心价值的关键部分，尤其是在提供流畅体验方面。自Chrome 41开始，V8采用了一种新技术，通过在小的、否则会被闲置的时间片段中隐藏昂贵的内存管理操作，提高了网页应用程序的响应性。因此，网页开发者可以期望更流畅的滚动效果和柔顺动画，同时由于垃圾回收减少了大量卡顿现象。

<!--truncate-->
许多现代语言引擎（如Chrome的V8 JavaScript引擎）动态管理运行应用程序的内存，使开发者无需自己担心内存管理问题。引擎会定期扫描分配给应用程序的内存，确定哪些数据不再需要，并将其清除以释放空间。这个过程称为[垃圾回收](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science))。

在Chrome中，我们努力提供流畅的每秒60帧（FPS）视觉体验。虽然V8已经尝试在小片段中执行垃圾回收，但较大的垃圾回收操作可能会在不可预测的时间发生——有时是在动画过程中——暂停执行并阻止Chrome达到60 FPS目标。

Chrome 41引入了一个[用于Blink渲染引擎的任务调度器](https://blog.chromium.org/2015/04/scheduling-tasks-intelligently-for_30.html)，使得延迟敏感任务可以优先处理，以确保Chrome保持响应迅速和流畅。除了可以优先处理任务，这个任务调度器还能集中掌握系统有多忙、需要执行哪些任务以及每个任务的紧急程度。因此，它可以估算Chrome什么时候可能处于空闲状态以及预计会闲置多长时间。

例如，当Chrome正在显示网页动画时，动画会以每秒60帧的速度更新屏幕，给Chrome大约16.6毫秒的时间来完成更新。因此，Chrome会在显示完上一帧后立即开始处理当前帧，执行输入、动画和帧渲染任务。如果Chrome在16.6毫秒内完成所有工作，那么在开始渲染下一帧之前的剩余时间里就无事可做。Chrome的调度器使V8能够利用这个闲置时间段，通过在Chrome处于空闲状态时安排特殊的_闲置任务_。

![图1:带有闲置任务的帧渲染](/_img/free-garbage-collection/frame-rendering.png)

闲置任务是一种特殊的低优先级任务，当调度器判断处于闲置时间段时运行。闲置任务有一个截止时间，这是调度器估算Chrome可能会闲置的时间长度。在图1中的动画例子中，这将是下一帧开始绘制的时间点。在其他情况下（例如，当没有屏幕活动时），这可能是下一个待执行任务计划运行的时间，上限为50毫秒，以确保Chrome对用户的意外输入保持响应性。截止时间用于闲置任务估算在不引起卡顿或输入响应延迟的情况下可以完成多少工作。

在闲置任务中进行的垃圾回收操作隐藏了对关键的延迟敏感任务的影响。这意味着这些垃圾回收任务是“免费的”。为了了解V8如何实现这一点，有必要回顾一下V8当前的垃圾回收策略。

## 深入了解V8的垃圾回收引擎

V8使用一种[分代垃圾回收器](http://www.memorymanagement.org/glossary/g.html#term-generational-garbage-collection)，将JavaScript堆分成一个用于新分配对象的小的年轻代和一个用于长期存活对象的大型老年代。[由于大多数对象短命](http://www.memorymanagement.org/glossary/g.html#term-generational-hypothesis)，这种分代策略使垃圾回收器能够在较小的年轻代中执行常规的短时间垃圾回收（称为清理），而无需追踪老年代中的对象。

年轻代使用[半空间](http://www.memorymanagement.org/glossary/s.html#semi.space)分配策略，其中新对象最初分配在年轻代的活动半空间中。一旦该半空间装满，一个清理操作会将存活的对象移动到另一个半空间。已经被移动过一次的对象会被提升到老年代并被认为是长期存活的。一旦存活的对象被移动，新半空间变为活动空间，旧半空间中剩余的死对象被丢弃。

因此，年轻代清理操作的持续时间取决于年轻代中存活对象的大小。当大多数对象在年轻代中变得不可达时，清理操作会非常快(&lt;1 毫秒)。但是，如果大多数对象在清理操作中幸存下来，清理操作的持续时间可能会显著增长。

当老年代中存活对象的大小超过根据经验推导出的限制时，会对整个堆执行一次主要的垃圾收集（GC）。老年代使用[标记-清除收集器](http://www.memorymanagement.org/glossary/m.html#term-mark-sweep)，并通过一些优化来改善延迟和内存消耗。标记延迟取决于需要标记的存活对象数量，对于大型 web 应用程序，整个堆的标记可能需要超过 100 毫秒。为了避免长时间暂停主线程，V8 长期以来具备[以许多小步骤增量标记存活对象的能力](https://blog.chromium.org/2011/11/game-changer-for-interactive.html)，目的是将每次标记步骤的时长控制在 5 毫秒以下。

标记完成后，通过清除整个老年代内存使应用程序可以再次使用自由内存。这项任务由专门的清理线程并发执行。最后，通过内存压缩来降低老年代的内存碎片。这项任务可能非常耗时，仅在内存碎片是一个问题时才执行。

总结起来，垃圾收集有四个主要任务：

1. 通常很快的年轻代清理
2. 由增量标记器执行的标记步骤，这可能根据步骤大小任意延长
3. 可能耗时很长的完整垃圾收集
4. 附带激进内存压缩的完整垃圾收集，虽然可能耗时很长，但可以清理碎片内存

为了在空闲期间执行这些操作，V8 会向调度器发送垃圾收集空闲任务。当这些空闲任务运行时，会被提供一个完成任务的截止时间。V8 的垃圾收集空闲时间处理器会评估应该执行哪些垃圾收集任务以减少内存消耗，同时遵守截止时间以避免将来在帧渲染或输入延迟中出现卡顿。

如果应用程序的测量分配速率表明年轻代可能会在下一个预期空闲期间之前填满，垃圾收集器将在空闲任务期间执行一次年轻代清理。此外，它还会计算最近清理任务的平均时长，以预测未来清理的持续时间并确保不违反空闲任务的截止时间。

当老年代中存活对象的大小接近堆限制时，增量标记开始。增量标记步骤可以根据需要标记的字节数按线性比例调整。基于平均测量的标记速度，垃圾收集空闲时间处理器尝试将尽可能多的标记工作适配到给定的空闲任务中。

如果老年代几乎满了，并且任务提供的截止时间估计足够长以完成收集，则在空闲任务期间会安排一次完整垃圾收集。收集暂停时间根据标记速度乘以分配对象数量来预测。附带额外内存压缩的完整垃圾收集仅在网页长时间处于空闲状态时执行。

## 性能评估

为了评估在空闲时间运行垃圾收集的影响，我们使用 Chrome 的[Telemetry 性能基准框架](https://www.chromium.org/developers/telemetry)来评估在网页加载时流行网站的滚动流畅度。我们在一台 Linux 工作站上基准测试了[前 25 名](https://code.google.com/p/chromium/codesearch#chromium/src/tools/perf/benchmarks/smoothness.py&l=15)网站以及在 Android Nexus 6 智能手机上的[典型移动网站](https://code.google.com/p/chromium/codesearch#chromium/src/tools/perf/benchmarks/smoothness.py&l=104)。两者都打开了流行的网页（包括像 Gmail、Google Docs 和 YouTube 这样的复杂 web 应用程序），并滚动其内容几秒钟。Chrome 的目标是保持滚动在 60 FPS，以提供流畅的用户体验。

图 2 显示了在空闲时间安排的垃圾收集百分比。工作站较快的硬件与 Nexus 6 相比提供了更多的整体空闲时间，从而使更多比例的垃圾收集可以在空闲时间进行（43%，而 Nexus 6 为 31%），这在我们[卡顿指标](https://www.chromium.org/developers/design-documents/rendering-benchmarks)上改善了约 7%。

![图2: 在空闲时间进行垃圾回收的比例](/_img/free-garbage-collection/idle-time-gc.png)

除了改善页面渲染的流畅性外，这些空闲时段还提供了一个机会，可以在页面完全空闲时执行更积极的垃圾回收。Chrome 45中的最新改进利用了这一点，大幅减少了空闲前台标签页所消耗的内存。图3显示了当Gmail的页面变得空闲时，其JavaScript堆内存使用量相比于Chrome 43同一页面能够减少约45%的情况。

<figure>
  <div class="video video-16:9">
    <iframe src="https://www.youtube.com/embed/ij-AFUfqFdI" width="640" height="360" loading="lazy"></iframe>
  </div>
  <figcaption>图3: Gmail在最新版本的Chrome 45（左）与Chrome 43的内存使用比较</figcaption>
</figure>

这些改进表明，通过更聪明地选择执行昂贵的垃圾回收操作的时机，可以隐藏垃圾回收暂停。Web开发人员即使是针对流畅度极高的60 FPS动画，也无需再担心垃圾回收暂停。请继续关注我们关于垃圾回收调度边界的更多改进。
