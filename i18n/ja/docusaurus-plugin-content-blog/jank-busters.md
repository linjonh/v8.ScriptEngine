---
title: "ジャンクバスターズ パート1"
author: "ジャンクバスターズ: ジョッヘン・アイジンガー、ミハエル・リッパウツ、ハンネス・パイヤー"
avatars: 
  - "michael-lippautz"
  - "hannes-payer"
date: "2015-10-30 13:33:37"
tags: 
  - メモリ
description: "この記事では、Chrome 41からChrome 46の間に実装された最適化について説明します。これにより、ガベージコレクションの一時停止が大幅に短縮され、ユーザー体験が向上します。"
---
ジャンク、言い換えれば目に見えるスタッター（処理の遅延）は、Chromeが16.66ms以内にフレームをレンダリングできない場合（60フレーム毎秒の動きが邪魔される）に発生します。現時点では、V8のガベージコレクションのほとんどがメインレンダリングスレッドで実行されています（図1参照）。これにより、多くのオブジェクトを管理する必要がある場合にジャンクが発生することが多々あります。ジャンクの排除はV8チームにとって常に最優先事項でした（[1](https://blog.chromium.org/2011/11/game-changer-for-interactive.html), [2](https://www.youtube.com/watch?v=3vPOlGRH6zk), [3](/blog/free-garbage-collection)）。この記事では、Chrome 41からChrome 46までに実装された最適化について説明し、ガベージコレクションの一時停止を大幅に削減し、ユーザー体験を向上させる結果となったものを紹介します。

<!--truncate-->
![図1: メインスレッドで実行されるガベージコレクション](/_img/jank-busters/gc-main-thread.png)

ガベージコレクション中のジャンクの主な原因の1つは、さまざまな管理用データ構造を処理することです。これらのデータ構造の多くは、ガベージコレクションとは無関係な最適化を可能にします。例として、すべてのArrayBufferのリストや、各ArrayBufferのビューのリストがあります。これらのリストは、ArrayBufferビューのアクセスに影響を与えることなくDetachArrayBuffer操作を効率的に実行できるようにします。しかし、Webページが何百万ものArrayBufferを作成する場合（例：WebGLベースのゲーム）には、ガベージコレクション中にこれらのリストを更新することで重大なジャンクが引き起こされます。Chrome 46では、これらのリストを削除し、代わりにArrayBuffersへの読み込みや書き込みの前にチェックを挿入することで切り離されたバッファを検出しました。これにより、GC中に大きな管理リストを歩くコストをプログラム実行全体に分散することができ、ジャンクが減少します。理論的には、アクセスごとのチェックによりArrayBufferを多用するプログラムのスループットが低下する可能性がありますが、実際にはV8の最適化コンパイラが冗長なチェックを削除し、残りのチェックをループ外に持ち上げることが頻繁に可能であるため、全体的な実行プロファイルが大幅に滑らかになり、ほとんどまたは全く性能のペナルティが発生しません。

もう1つのジャンクの原因は、ChromeとV8の間で共有されるオブジェクトのライフタイムを追跡するための管理作業です。ChromeとV8のメモリヒープは別々ですが、DOMノードのようにChromeのC++コードで実装されながらJavaScriptからアクセス可能な特定のオブジェクトについては同期する必要があります。V8はハンドルと呼ばれる不透明なデータ型を作成し、これを使用してChromeがV8ヒープオブジェクトを実装の詳細を知らずに操作できるようにします。オブジェクトのライフタイムはハンドルに結び付けられており、Chromeがハンドルを保持している限り、V8のガベージコレクターはオブジェクトを破棄しません。V8は、ChromeにV8 APIを通じて外部に戻された各ハンドルのためにグローバル参照と呼ばれる内部データ構造を作成します。これらのグローバル参照が、オブジェクトがまだ生存していることをV8のガベージコレクターに知らせます。WebGLゲームの場合、Chromeは何百万ものハンドルを作成し、V8はそれに対応するグローバル参照を作成してそのライフサイクルを管理する必要があります。これらの大量のグローバル参照をメインのガベージコレクション中に処理することは、ジャンクとして観察されます。幸いなことに、WebGLに渡されたオブジェクトは単に渡されるだけで実際に変更されることがほとんどないため、単純な静的[エスケープ解析](https://en.wikipedia.org/wiki/Escape_analysis)が可能です。例えば、通常小さな配列をパラメータとして渡すことが知られているWebGL関数については、基礎的なデータをスタック上にコピーしてグローバル参照を不要にします。このような混合アプローチの結果、レンダリングに負荷が高いWebGLゲームにおける一時停止時間が最大50%減少します。

V8のガベージコレクションのほとんどはメインレンダリングスレッドで実行されます。ガベージコレクション操作を並行スレッドに移動することで、ガベージコレクターの待機時間を短縮し、ジャンクをさらに減少させます。これは本質的に複雑な作業で、メインのJavaScriptアプリケーションとガベージコレクターが同時に同じオブジェクトを観測および変更する可能性があるためです。これまで、並行処理は通常のオブジェクトJSヒープの古い世代のスイープに限定されていました。最近では、V8ヒープのコードとマップスペースの並行スイープも実装しました。また、メインスレッドで実行する必要がある作業を減らすため、未使用ページの並行アンマッピングも実装しました（図2参照）。

![図2: 同時ガベージコレクションスレッドで実行されたいくつかのガベージコレクション操作。](/_img/jank-busters/gc-concurrent-threads.png)

前述の最適化の影響は、たとえば[TurbolenzのOort Onlineデモ](http://oortonline.gl/)など、WebGLベースのゲームで明確に見られます。以下のビデオでは、Chrome 41とChrome 46を比較しています:

<figure>
  <div class="video video-16:9">
    <iframe src="https://www.youtube.com/embed/PgrCJpbTs9I" width="640" height="360" loading="lazy"></iframe>
  </div>
</figure>

現在、メインスレッドでのガベージコレクション停止時間をさらに短縮するために、より多くのガベージコレクションコンポーネントを段階的、並行的、並列的にする作業を進めています。興味深いパッチが進行中なので、ご期待ください。
