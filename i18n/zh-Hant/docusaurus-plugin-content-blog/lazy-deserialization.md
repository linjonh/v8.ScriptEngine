---
title: "延遲反序列化"
author: "Jakob Gruber ([@schuay](https://twitter.com/schuay))"
avatars: 
  - "jakob-gruber"
date: "2018-02-12 13:33:37"
tags: 
  - internals
description: "延遲反序列化，已在 V8 v6.4 中提供，可平均減少每個瀏覽器標籤超過 500 KB 的 V8 記憶體使用量。"
tweet: "962989179914383360"
---
TL;DR: 延遲反序列化最近在 [V8 v6.4](/blog/v8-release-64) 中默認啟用，平均減少每個瀏覽器標籤超過 500 KB 的 V8 記憶體使用量。繼續閱讀以了解更多！

## 介紹 V8 快照

但首先，讓我們退一步來看看 V8 如何使用堆快照來加速新 Isolate 的創建（大致對應於 Chrome 中的一個瀏覽器標籤）。我的同事 Yang Guo 在他關於[自定義啟動快照](/blog/custom-startup-snapshots)的文章中給出了很好的介紹：

<!--truncate-->
> JavaScript 規範包含了許多內建功能，從數學函數到一個功能齊全的正則表達式引擎。每個新創建的 V8 上下文從開始就可用這些功能。要實現這一點，全域物件（例如瀏覽器中的 `window` 物件）和所有內建功能必須在創建上下文時，已被設置並初始化到 V8 的堆中。從零開始執行此操作需要相當長的時間。
> 
> 幸運的是，V8 使用了一種捷徑來加速這個過程：就像為快速晚餐解凍冷凍披薩一樣，我們將先前準備好的快照直接反序列化到堆中來獲得初始化的上下文。在普通的桌上電腦上，這可以將創建上下文的時間從 40 毫秒降低到不到 2 毫秒。在普通的手機上，這可能意味著從 270 毫秒降低到 10 毫秒。

簡而言之：快照對於啟動效能至關重要，它們被反序列化以為每個 Isolate 創建 V8 堆的初始狀態。因此，快照的大小決定了 V8 堆的最小大小，較大的快照會直接導致每個 Isolate 的記憶體使用量增加。

一個快照包含了完全初始化新 Isolate 所需的一切，包括語言常量（例如 `undefined` 值）、詮釋器使用的內部位元組碼處理程序、內建物件（例如 `String`），以及安裝在內建物件上的函數（例如 `String.prototype.replace`）及其可執行的 `Code` 物件。

![從 2016-01 到 2017-09 的啟動快照大小，以位元組計。x 軸顯示 V8 修訂版號。](/_img/lazy-deserialization/startup-snapshot-size.png)

在過去的兩年裡，快照大小幾乎翻了三倍，從 2016 年初的大約 600 KB 增加到今天的超過 1500 KB。這種增長的絕大部分來自於序列化的 `Code` 物件，這些物件的數量增加了（例如，隨著 JavaScript 語言規範的演變和擴展，最近添加了新功能）；並且其大小也增加了（由新的 [CodeStubAssembler](/blog/csa) 管線生成的建構模組以原生代碼形式出貨，而不是更緊湊的位元組碼或最小化的 JS 格式）。

這是壞消息，因為我們希望將記憶體使用量保持在盡可能低的水平。

## 延遲反序列化

其中一個主要的痛點是我們過去將快照的整個內容複製到每個 Isolate 中。這對於內建函數來說尤其浪費，因為所有這些函數都是無條件加載的，但可能從未被使用過。

這就是延遲反序列化的作用所在。其概念非常簡單：如果我們僅在內建函數被調用之前才反序列化它們呢？

對一些最受歡迎的網站進行的一項快速調查顯示，這個方法相當有吸引力：平均僅有 30% 的內建函數被使用，而有些網站僅使用了 16%。這看起來非常有前景，考慮到這些網站大多是大量使用 JS 的，它們的數據可以被視為對整個網頁潛在記憶體節約的（模糊的）下界。

當我們開始朝這個方向工作時，發現延遲反序列化很好地與 V8 的架構整合，只需要一些主要是非侵入性的設計變更就可以啟動和運行：

1. **快照內的已知位置。** 在延遲反序列化之前，序列化快照內的物件順序是無關緊要的，因為我們只會一次性反序列化整個堆。延遲反序列化必須能夠獨立反序列化任何給定的內建函數，因此必須知道它在快照中的位置。
2. **單個物件的反序列化。** V8 的快照最初是為整個堆的反序列化而設計的，要新增支援單個物件反序列化必須處理幾個特性，例如非連續快照佈局（某一物件的序列化數據可能被其他物件的數據穿插），以及所謂的反向參照（可以直接引用當前執行過程中之前反序列化的物件）。
3. **懶惰反序列化機制本身。** 在運行時，懶惰反序列化處理器必須能夠 a) 判斷需要反序列化的代碼物件是什麼，b) 執行實際的反序列化，及 c) 將序列化的代碼物件附加到所有相關的函數之上。

我們對前兩點的解決方案是新增一個[專用的內建區域](https://cs.chromium.org/chromium/src/v8/src/snapshot/snapshot.h?l=55&rcl=f5b1d1d4f29b238ca2f0a13bf3a7b7067854592d)到快照中，該區域僅可以包含序列化的代碼物件。序列化以明確定義的順序進行，每個 `Code` 物件的起始偏移量都保存在內建快照區域內的一個專用區段中。不允許反向參照以及穿插的物件數據。

[懶惰內建反序列化](https://goo.gl/dxkYDZ)由名副其實的 [`DeserializeLazy` 內建函數](https://cs.chromium.org/chromium/src/v8/src/builtins/x64/builtins-x64.cc?l=1355&rcl=f5b1d1d4f29b238ca2f0a13bf3a7b7067854592d)處理，該函數在反序列化時安裝到所有懶惰內建函數之上。當在運行時調用時，它反序列化相關的 `Code` 物件，並最終安裝到 `JSFunction`（表示函數物件）和 `SharedFunctionInfo`（在由同一函數文字創建的函數之間共享）上。每個內建函數最多只反序列化一次。

除了內建函數，我們還實現了[字節碼處理器的懶惰反序列化](https://goo.gl/QxZBL2)。字節碼處理器是包含執行 V8 的 [Ignition](/blog/ignition-interpreter) 解析器中每個字節碼邏輯的代碼物件。與內建函數不同，它們既沒有附加的 `JSFunction`，也沒有 `SharedFunctionInfo`。相反，它們的代碼物件直接存儲在[分派表](https://cs.chromium.org/chromium/src/v8/src/interpreter/interpreter.h?l=94&rcl=f5b1d1d4f29b238ca2f0a13bf3a7b7067854592d)中，解析器在分派到下一個字節碼處理器時將其索引。懶惰反序列化類似於內建函數：[`DeserializeLazy`](https://cs.chromium.org/chromium/src/v8/src/interpreter/interpreter-generator.cc?l=3247&rcl=f5b1d1d4f29b238ca2f0a13bf3a7b7067854592d)處理器通過檢查字節碼數組來確定需要反序列化的處理器，反序列化代碼物件，並最終將反序列化的處理器存儲在分派表中。同樣，每個處理器最多只反序列化一次。

## 結果

我們通過使用 Android 設備上的 Chrome 65 來加載最受歡迎的 1000 個網站，並對啟用和未啟用懶惰反序列化進行比較來評估內存節省。

![](/_img/lazy-deserialization/memory-savings.png)

平均而言，V8 的堆大小減少了 540 KB，測試的網站中有 25% 節省超過 620 KB，50% 節省超過 540 KB，75% 節省超過 420 KB。

運行時性能（在 Speedometer 等標準 JS 基準測試以及多個受歡迎網站上測量）未因懶惰反序列化而受到影響。

## 下一步

懶惰反序列化確保每個隔離執行環境只加載實際使用的內建代碼物件。這已經是一個巨大提升，但我們認為可以再進一步，將每個隔離執行環境（與內建有關的）成本減少到幾乎為零。

我們希望在今年晚些時候為您帶來這方面的更新，敬請期待！
