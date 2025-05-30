---
title: "その `.wasm` に何が入っているのか？新機能: `wasm-decompile` を紹介"
author: "Wouter van Oortmerssen ([@wvo](https://twitter.com/wvo))"
avatars: 
  - "wouter-van-oortmerssen"
date: 2020-04-27
tags: 
  - WebAssembly
  - ツール
description: "WABT に新しい逆コンパイルツールが加わり、Wasm モジュールの内容を読みやすくします。"
tweet: "1254829913561014272"
---
現在、`.wasm` ファイルを生成または操作するためのコンパイラやその他のツールが増えてきています。時には内部を調べたいと思うこともあるでしょう。ツールの開発者である場合や、Wasm を直接対象とするプログラマーであり、生成されたコードが性能やその他の理由でどのように見えるか気になる場合です。

<!--truncate-->
しかし問題は、Wasm が非常に低レベルであり、実際のアセンブリコードのようであることです。特に JVM のように、すべてのデータ構造が便利に名前付けされたクラスやフィールドではなく、load/store 操作にコンパイルされている点が挙げられます。LLVM のようなコンパイラは、生成されたコードが元のコードとまったく異なるように見えるほどの変換を実行することができます。

## アセンブル解除または逆コンパイル？

`wasm2wat` などのツールを使用すると、`.wasm` を Wasm の標準テキスト形式である `.wat` に変換できます（[WABT](https://github.com/WebAssembly/wabt) ツールキットの一部）。これは非常に忠実な表現ですが、特に読みやすくはありません。

例えば、以下のような単純な C 関数、ドット積の場合:

```c
typedef struct { float x, y, z; } vec3;

float dot(const vec3 *a, const vec3 *b) {
    return a->x * b->x +
           a->y * b->y +
           a->z * b->z;
}
```

`clang dot.c -c -target wasm32 -O2` を使用してコンパイルした後、`wasm2wat -f dot.o` を使ってこれを `.wat` に変換すると、次のようになります:

```wasm
(func $dot (type 0) (param i32 i32) (result f32)
  (f32.add
    (f32.add
      (f32.mul
        (f32.load
          (local.get 0))
        (f32.load
          (local.get 1)))
      (f32.mul
        (f32.load offset=4
          (local.get 0))
        (f32.load offset=4
          (local.get 1))))
    (f32.mul
      (f32.load offset=8
        (local.get 0))
      (f32.load offset=8
        (local.get 1))))))
```

これは非常に少ないコードですが、すでに多くの理由で読みづらいものです。表現ベースの構文が欠けていることや、一般的な冗長さのほか、データ構造をメモリロードとして理解するのが難しい点が挙げられます。大規模なプログラムの出力を見ていることを想像してみてください。すぐに理解できなくなります。

`wasm2wat` の代わりに `wasm-decompile dot.o` を実行すると、次のようになります:

```c
function dot(a:{ a:float, b:float, c:float },
             b:{ a:float, b:float, c:float }):float {
  return a.a * b.a + a.b * b.b + a.c * b.c
}
```

これのほうがかなり馴染み深い見た目です。表現ベースの構文に加え、デコンパイラーは関数内のすべてのロードとストア操作を検査し、それらの構造を推測しようとします。そして各変数を「インライン」構造体宣言で注釈します。3 つの float が同じ概念を表しているかどうかを必ずしも知ることができないため、名前付きの構造体宣言は作成されません。

## 何に逆コンパイルする？

`wasm-decompile` は、できるだけ平均的なプログラミング言語に似せた出力を生成しつつ、それが表す Wasm に近いものを維持します。

その第 1 の目標は読みやすさです: `.wasm` に含まれる内容を可能な限り簡単に理解できるコードで示唆することです。その第 2 の目標は、逆アセンブラーとしての有用性を失わないように、Was また 1:1 に表現することです。もちろんこれら 2 つの目標は必ずしも統一できるものではありません。

この出力は実際のプログラミング言語を意味しているわけではなく、現在のところ、それを Wasm に再コンパイルする方法はありません。

### ロードとストア

前述のように、`wasm-decompile` は特定のポインターに対して行われるすべてのロードとストア操作を検査します。それらが連続するアクセスセットを形成する場合、これら「インライン」構造体宣言の 1 つを出力します。

すべての「フィールド」がアクセスされていない場合、それが構造体としての意図されたものなのか、無関係なメモリアクセスなのかを確定することはできません。その場合、`float_ptr` のようなシンプルな型にデフォルト化します（型が同じ場合）。または、最悪の場合、`o[2]:int` のように配列アクセスを出力します。これは `o` が `int` 値を指していて、3 番目をアクセスしていることを意味します。

この最後のケースは思ったよりも頻繁に発生します。Wasm のローカルは変数というよりもよりレジスタのように機能するため、最適化されたコードは異なるオブジェクトに対して同じポインタを共有する可能性があるからです。

デコンパイラーはインデックス処理について賢いため、`(base + (index << 2))[0]:int` のようなパターンを検出します。これは通常の C 配列インデックス操作である `base[index]` から生じるものであり、`base` が 4 バイト型を指している場合です。これらはコード中で非常に一般的であり、Wasm のロードやストアは定数オフセットのみを持つためです。`wasm-decompile` の出力はこれらを再び `base[index]:int` に変換します。

さらに絶対アドレスがデータセクションを指していることを認識します。

### 制御フロー

最も馴染み深いのはWasmのif-then構造で、これはおなじみの`if (cond) { A } else { B }`の構文に翻訳されます。Wasmでは値を返すことが可能なため、一部の言語で利用できる三項演算の`cond ? A : B`構文を表すこともできます。

Wasmの制御フローの残りは、`block`と`loop`ブロック、および`br`、`br_if`、`br_table`ジャンプに基づいています。デコンパイラは、これらの構造にかなり近い形を維持しており、これらが由来しているかもしれないwhile/for/switch構造を推測しようとはしません。この方法の方が最適化された出力との相性が良いためです。例えば、`wasm-decompile`の出力では典型的なループは以下のようになります。

```c
loop A {
  // ここにループの本体。
  if (cond) continue A;
}
```

ここで、`A`はこれらを複数ネストできるラベルです。ループを制御するために`if`と`continue`を使用することは、whileループと比較して少し馴染みが薄いかもしれませんが、これは直接的にWasmの`br_if`に対応しています。

ブロックは似ていますが、後方ではなく前方に分岐します:

```c
block {
  if (cond) break;
  // 本体はこちら。
}
```

これは実際にはif-thenを実装しています。将来のデコンパイラのバージョンでは、可能な場合にこれを実際のif-thenに翻訳するかもしれません。

Wasmの最も驚くべき制御構文は`br_table`で、これはネストされた`block`を使用して`switch`のようなものを実装しますが、これを読むのは通常困難です。デコンパイラはこれを平坦化して少し読みやすくします。例えば以下のように:


```c
br_table[A, B, C, ..D](a);
label A:
return 0;
label B:
return 1;
label C:
return 2;
label D:
```

これは`a`の`switch`に類似しており、`D`がデフォルトケースになります。

### その他の便利な機能

デコンパイラは以下をサポートします:

- デバッグ情報やリンク情報から名前を取得したり、自身で名前を生成したりできます。既存の名前を使用する場合、C++の名前マングリングされたシンボルを簡略化する特別なコードがあります。
- マルチ値プロポーザルをすでにサポートしており、これにより式やステートメントに変換する作業が少し困難になります。複数の値が返される場合は追加の変数が使用されます。
- データセクションの_内容_から名前を生成することも可能です。
- コードだけでなく、すべてのWasmセクションタイプに対してきれいな宣言を出力します。例えば、可能な場合はテキストとして出力することでデータセクションを読みやすくしようとします。
- 共通のCスタイル言語に一般的な演算子の優先順位をサポートしており、一般的な式の`()`を減少させます。

### 制約

Wasmのデコンパイルは、例えばJVMバイトコードよりも基本的に困難です。

後者は非最適化されており、元のコードの構造に比較的忠実で、名前が欠けている場合でもメモリ位置だけでなくユニークなクラスを参照しています。

対照的に、ほとんどの`.wasm`出力はLLVMによって徹底的に最適化されており、元の構造のほとんどを失っています。出力コードはプログラマーが書くようなものとは非常に異なります。そのため、Wasmのデコンパイラを有用にする大きな課題となりますが、それでも試みる価値はあるでしょう。

## さらに

もっと知りたい場合は、自分のWasmプロジェクトをデコンパイルするのが一番です！

さらに、`wasm-decompile`に関するより詳細なガイドは[こちら](https://github.com/WebAssembly/wabt/blob/master/docs/decompiler.md)にあります。その実装は[こちら](https://github.com/WebAssembly/wabt/tree/master/src)の`decompiler`で始まるソースファイルにあります（改善するためのPRを自由に投稿してください！）。`.wat`とデコンパイラの間の違いを示すさらに多くの例を含むテストケースは[こちら](https://github.com/WebAssembly/wabt/tree/master/test/decompile)にあります。
