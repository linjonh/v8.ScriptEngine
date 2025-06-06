---
title: "バックグラウンドコンパイル"
author: "[Ross McIlroy](https://twitter.com/rossmcilroy), メインスレッド擁護者"
avatars: 
  - "ross-mcilroy"
date: "2018-03-26 13:33:37"
tags: 
  - 内部構造
description: "Chrome 66から、V8がJavaScriptのソースコードをバックグラウンドスレッド上でコンパイルするようになり、典型的なウェブサイトでメインスレッドのコンパイル時間が5%から20%短縮されました。"
tweet: "978319362837958657"
---
要約: Chrome 66から、V8がJavaScriptのソースコードをバックグラウンドスレッド上でコンパイルするようになり、典型的なウェブサイトでメインスレッドのコンパイル時間が5%から20%短縮されました。

## 背景

バージョン41以降、ChromeはV8の[`StreamedSource`](https://cs.chromium.org/chromium/src/v8/include/v8.h?q=StreamedSource&sq=package:chromium&l=1389) APIを通じて[バックグラウンドスレッドでのJavaScriptソースファイルの解析](https://blog.chromium.org/2015/03/new-javascript-techniques-for-rapid.html)をサポートしています。これは、Chromeがネットワークからファイルの最初のチャンクをダウンロードするとすぐに、V8がJavaScriptソースコードの解析を開始できるようにし、Chromeがネットワーク経由でファイルをストリームする間に並行して解析を継続することを可能にします。これにより、V8がJavaScriptの解析をほぼ完了している状態でファイルのダウンロードが終了するため、読み込み時間が大幅に改善される場合があります。

<!--省略-->
しかし、V8の元々のベースラインコンパイラの制限により、V8はまだメインスレッドに戻り、解析を最終化し、スクリプトのコードを実行するためのJITマシンコードにコンパイルする必要がありました。[Ignition + TurboFanパイプライン](/blog/launching-ignition-and-turbofan)への移行により、バイトコードコンパイルをバックグラウンドスレッドに移動することが可能になり、これによりChromeのメインスレッドが解放され、よりスムーズで反応の良いウェブブラウジング体験が提供されるようになりました。

## バックグラウンドスレッドバイトコードコンパイラの構築

V8のIgnitionバイトコードコンパイラは、パーサーによって生成された[抽象構文木(AST)](https://en.wikipedia.org/wiki/Abstract_syntax_tree)を入力として受け取り、JavaScriptソースを実行するための関連メタデータとともにバイトコード(`BytecodeArray`)のストリームを生成します。

![](/_img/background-compilation/bytecode.svg)

Ignitionのバイトコードコンパイラは、マルチスレッドを念頭に置いて構築されていますが、バックグラウンドコンパイルを可能にするためにはコンパイルパイプライン全体でいくつかの変更が必要でした。主な変更の1つとして、バックグラウンドスレッドで実行中のコンパイルパイプラインがV8のJavaScriptヒープ内のオブジェクトにアクセスすることを防ぐ必要がありました。JavaScriptがシングルスレッドであるため、V8のヒープ内のオブジェクトはスレッドセーフではなく、バックグラウンドコンパイル中にメインスレッドやV8のガーベッジコレクターによって変更される可能性があります。

コンパイルパイプラインには、V8のヒープ上のオブジェクトにアクセスする主な段階が2つありました: AST内部化とバイトコード最終化です。AST内部化は、ASTで識別されたリテラルオブジェクト(文字列、数字、オブジェクトリテラルなど)をV8ヒープに割り当てるプロセスであり、スクリプトが実行されるときに生成されたバイトコードによって直接使用できるようにします。このプロセスは従来、パーサーがASTを生成した直後に行われていました。そのため、コンパイルパイプラインの後続のステップでは、リテラルオブジェクトが割り当てられていることを前提としていました。バックグラウンドコンパイルを可能にするために、AST内部化をコンパイルパイプラインの後の段階、バイトコードがコンパイルされた後に移動しました。これにより、パイプラインの後続の段階で、ヒープ上で内部化された値ではなく、ASTに埋め込まれた_raw_リテラル値にアクセスするよう変更が必要でした。

バイトコード最終化は、関連するメタデータ（例: バイトコードが参照する定数を格納する`ConstantPoolArray`や、JavaScriptソースの行番号と列番号をバイトコードのオフセットにマッピングする`SourcePositionTable`）とともに関数を実行するために使用される最終的な`BytecodeArray`オブジェクトを構築するプロセスです。JavaScriptが動的な言語であるため、これらのオブジェクトはすべてJavaScriptヒープ内に存在し、バイトコードが関連付けられたJavaScript関数が収集される場合にガーベッジコレクションが可能である必要があります。従来、これらのメタデータオブジェクトのいくつかはバイトコードコンパイル中に割り当てられ、変更されるプロセスが含まれており、これにはJavaScriptヒープへのアクセスが必要でした。バックグラウンドコンパイルを可能にするために、Ignitionのバイトコードジェネレーターはこれらのメタデータの詳細を追跡し、それらを完全なコンパイルプロセスの最終段階までJavaScriptヒープ上に割り当てることを遅らせるようにリファクタリングされました。

これらの変更により、ほとんどすべてのスクリプトのコンパイルをバックグラウンドスレッドに移動することが可能となり、メインスレッドで実行されるのは短いAST内部化のステップとバイトコード最終化ステップのみとなり、それはスクリプト実行の直前に行われます。

![](/_img/background-compilation/threads.svg)

現在のところ、トップレベルのスクリプトコードと即時呼び出し関数式（IIFE）のみがバックグラウンドスレッドでコンパイルされます。内部関数は依然として（最初に実行されたときに）メインスレッドで遅延コンパイルされます。将来的には、バックグラウンドコンパイルをより多くの状況に拡張することを目指しています。しかし、これらの制限がある場合でも、バックグラウンドコンパイルはメインスレッドをより長く自由にし、ユーザーインタラクションへの反応、アニメーションのレンダリング、またはそれ以外の場合でもスムーズで応答性の高い体験を提供するための作業を可能にします。

## 結果

人気のあるウェブページのセットを対象にした私たちの[実世界ベンチマークフレームワーク](/blog/real-world-performance)を使用して、バックグラウンドコンパイルのパフォーマンスを評価しました。

![](/_img/background-compilation/desktop.svg)

![](/_img/background-compilation/mobile.svg)

バックグラウンドスレッドでコンパイルできる対象は、トップレベルのストリーミングスクリプトコンパイル中にコンパイルされたバイトコードの割合と、呼び出された内部関数として遅延コンパイルされる割合（メインスレッドで実行する必要がある部分）によって異なります。そのため、メインスレッドで節約される時間の割合も異なり、多くのページではメインスレッドのコンパイル時間が5%から20%の範囲で削減されます。

## 次のステップ

バックグラウンドスレッドでスクリプトをコンパイルするより良い方法は何でしょうか？スクリプトをまったくコンパイルする必要がないことです！バックグラウンドコンパイルと並行して、V8の[コードキャッシングシステム](/blog/code-caching)を改良し、V8によってキャッシュされるコードの量を拡大することで、頻繁に訪問するサイトのページ読み込みを高速化する作業も行っています。この分野の進捗については、近々お知らせできることを期待しています。お楽しみに！
