---
title: "V8にBigIntsを追加する"
author: "Jakob Kummerow、精度の仲裁人"
date: "2018-05-02 13:33:37"
tags: 
  - ECMAScript
description: "V8がBigIntsをサポートしました。これは、任意精度の整数を可能にするJavaScript言語機能です。"
tweet: "991705626391732224"
---
過去数か月間、将来のECMAScriptバージョンに含まれる予定の[この提案](https://github.com/tc39/proposal-bigint)に基づき、V8に[BigInts](/features/bigint)のサポートを実装しました。以下の記事では、この冒険の物語をお届けします。

<!--truncate-->
## 要約

JavaScriptプログラマーとして、現在[^1]任意[^2]精度の整数をツールボックスに追加できます：

```js
const a = 2172141653n;
const b = 15346349309n;
a * b;
// → 33334444555566667777n     // やった！
Number(a) * Number(b);
// → 33334444555566670000      // 残念！
const such_many = 2n ** 222n;
// → 6739986666787659948666753771754907668409286105635143120275902562304n
```

新しい機能の詳細とその使い方については、[BigIntに関する詳細な記事](/features/bigint)をご覧ください。皆さんがこれを使って素晴らしいものを作るのを楽しみにしています！

[^1]: Chrome Beta、Dev、またはCanary、または[プレビュー版Node.js](https://github.com/v8/node/tree/vee-eight-lkgr)を使用している場合は_現在_、それ以外の場合は_近日中_（おそらくChrome 67やNode.jsの開発版も同時期）。

[^2]: 実装が定義する制限までの任意精度。申し訳ありませんが、無限量のデータをコンピューターの有限メモリに収める方法はまだ解明されていません。

## BigIntsのメモリ内での表現

通常、コンピューターは整数をCPUのレジスタ（現在では通常32ビットまたは64ビット幅）またはレジスタサイズのメモリチャンクに格納します。この結果、馴染みのある最小値および最大値が得られます。たとえば、32ビット符号付き整数は-2,147,483,648から2,147,483,647までの値を保持できます。一方で、BigIntsのアイデアはそうした制限によらないという点です。

では、100ビット、1,000ビット、さらには1,000,000ビットのBigIntをどのように保存するのでしょうか？レジスタには収まらないため、メモリにオブジェクトを割り当てます。このオブジェクトは、BigIntの全ビットを保持できるだけ十分な大きさで、いくつかのチャンクに分割されています。これらのチャンクを「桁」と呼びます。この考えは、“9”より大きい数を「10」などのような複数の桁によって表現する十進法に非常によく似ています。ただし、十進法が0から9までの桁を使用するのに対して、私たちのBigIntsは0から4294967295（つまり`2**32-1`）までの桁を使用します。この桁は32ビットCPUレジスタ[^3]の値範囲で、符号ビットが付属していない場合の値です。符号ビットは別途格納します。擬似コードでは、`3*32 = 96`ビットを持つ`BigInt`オブジェクトは次のようになります：

```js
{
  type: 'BigInt',
  sign: 0,
  num_digits: 3,
  digits: [0x12…, 0x34…, 0x56…],
}
```

[^3]: 64ビットマシンでは、64ビットの桁（0から18446744073709551615、つまり`2n**64n-1n`まで）が使用されます。

## 学校に戻る、そしてKnuthに戻る

CPUレジスタに格納された整数を操作するのは非常に簡単です。たとえば、2つのレジスタを掛け合わせる場合、ソフトウェアは「これら2つのレジスタの内容を掛け合わせて！」とCPUに指示するマシン命令を使用できます。そしてCPUがそれを実行します。一方で、BigIntの算術演算では自力で解決法を見つける必要があります。この特定の作業については、文字通り全ての子供が何らかの時点で学ぶ方法がここに役立ちます。学校で電卓を使わない条件で345×678の掛け算をする練習をしたことを覚えていますか？

```
345 * 678
---------
     30    //   5 * 6
+   24     //  4  * 6
+  18      // 3   * 6
+     35   //   5 *  7
+    28    //  4  *  7
+   21     // 3   *  7
+      40  //   5 *   8
+     32   //  4  *   8
+    24    // 3   *   8
=========
   233910
```

まさにこれがV8がBigIntsを掛け算する方法です：1桁ずつ、途中結果を加算していきます。このアルゴリズムは、`0`から`9`までの数値桁でも、BigIntのはるかに大きな桁でも同様に機能します。

Donald Knuthは、彼の古典的な著書『コンピュータプログラムの芸術 (The Art of Computer Programming)』第2巻（1969年刊）で、小さなチャンクから構成される大きな数値を掛け算および割り算する具体的な実装を提供しています。V8の実装はこの本に基づいており、これが非常に永続的なコンピュータサイエンスの知識であることを証明しています。

## “デシュガリングを減らす” == より多くの甘味？

意外に思われるかもしれませんが、`-x`のような一見単純な単項演算を機能させるために、かなりの努力を費やしました。これまでは、`-x`はちょうど`x * (-1)`と同じで、簡素化のためにV8はJavaScriptを処理する際の最初の段階、つまりパーサーでこの置換を適用していました。このアプローチは「デシュガリング」と呼ばれるもので、`-x`のような式を`x * (-1)`という“構文シュガー”として扱います。他のコンポーネント（インタプリタ、コンパイラ、ランタイムシステム全体）は単項演算が何であるかを知る必要すらありませんでした。このアプローチでは常に掛け算しか見ないからです。当然、掛け算のサポートは必須です。

しかし、ビッグイントの場合、この実装は突然無効になります。なぜなら、ビッグイントを数値（例えば `-1`）と掛けることは `TypeError`[^4] をスローする必要があるためです。パーサは、`x` がビッグイントである場合に `-x` を `x * (-1n)` に変換する必要がありますが、パーサは `x` がどのような値になるかを知る方法がありません。そのため、この初期の変換に頼るのをやめ、数値とビッグイント両方で一元操作を適切にサポートする必要が出てきました。

[^4]: `BigInt` と `Number` のオペランド型を混在させることは一般的に許可されていません。それはJavaScriptにとってやや異例ですが、この決定には[説明](/features/bigint#operators)があります。

## ビット演算でちょっとした楽しみ

現在使用されているほとんどのコンピューターシステムは、「2の補数」と呼ばれる巧妙な方法を使用して符号付き整数を格納しています。この方法は、最初のビットが符号を示し、ビットパターンに1を追加すると常に数値が1増加するという優れた特性を持ち、符号ビットも自動的に処理されます。例えば、8ビット整数の場合:

- `10000000` は -128、表現可能な最小値、
- `10000001` は -127、
- `11111111` は -1、
- `00000000` は 0、
- `00000001` は 1、
- `01111111` は 127、表現可能な最大値。

このエンコーディングは非常に一般的で、多くのプログラマーがこれを期待し、それに依存しています。そして、BigInt仕様もこの事実を反映し、BigIntは2の補数表現を使用しているかのように振る舞わなければならないとしています。前述のように、V8のBigIntはそうではありません！

したがって、仕様に基づいてビット演算を行うために、V8のBigIntは内部的に2の補数を使用しているふりをする必要があります。正の値の場合、それは違いを生じませんが、負の数値ではこれを達成するために余分な作業が必要です。その結果、`a & b` の場合、もし `a` と `b` が両方とも負のBigIntであるなら、実際には _4つのステップ_ （両方が正なら1つのステップで済む）を実行します。両方の入力が偽の2の補数形式に変換され、次に実際の操作が行われ、その結果が元の表現に戻されます。なぜ往復するのかと思うかもしれません。それは、すべての非ビット演算がその方がはるかに簡単になるからです。

## TypedArrayの新しい2種類

BigInt提案には、新しいTypedArrayの種類 `BigInt64Array` と `BigUint64Array` が含まれています。BigIntが提供する自然な方法で、これらの要素内のビット全てを読み書きできるようになったため、64ビット幅の整数要素を持つTypedArrayの使用が可能になります。一方、数値を使用しようとすると一部のビットが失われる可能性があります。このため、新しい配列は既存の8/16/32ビット整数TypedArrayとは少し異なります。これらの要素へのアクセスは常にBigIntで行われ、数値を使用しようとすると例外がスローされます。

```js
> const big_array = new BigInt64Array(1);
> big_array[0] = 123n;  // OK
> big_array[0]
123n
> big_array[0] = 456;
TypeError: Cannot convert 456 to a BigInt
> big_array[0] = BigInt(456);  // OK
```

これらの種類の配列を操作するJavaScriptコードは、従来のTypedArrayコードとは少し異なり、動作も異なります。同様に、TypedArrayの実装を一般化して新しい種類に対応させる必要がありました。

## 最適化に関する考慮事項

現在、BigIntの基本的な実装を出荷しています。これは機能的に完全で、既存のユーザーランドライブラリより少し高速の性能を提供するはずですが、特に最適化されているわけではありません。その理由は、人工的なベンチマークよりも実世界のアプリケーションを優先するという目標に沿って、まずあなたがBigIntをどのように使用するかを観察したいからです。そしてその後、あなたが重要と考えるケースを正確に最適化していきたいと考えています。

例えば、比較的小さいBigInt（最大64ビット）が重要な使用例であることが分かれば、それらを特殊な表現を使用して、よりメモリー効率を高めることができます:

```js
{
  type: 'BigInt-Int64',
  value: 0x12…,
}
```

これを「int64」値範囲、「uint64」範囲、またはその両方で行うべきか、まだ見極める必要があります。迅速なパスをサポートする数が少ないほど早く出荷できる可能性があり、一方で追加の迅速なパスが増えるたびに、影響を受ける操作は常に適用可能かどうか確認する必要があるため、他の全てが皮肉にも少し遅くなるという事実も考慮する必要があります。

もう一つの話題は、最適化コンパイラにおけるBigIntのサポートです。64ビット値を操作し64ビットハードウェア上で実行される計算集約型アプリケーションの場合、それらの値をヒープ上にオブジェクトとして割り当てる現在の方法よりも、レジスタ内で保持する方がはるかに効率的です。このようなサポートを実装する計画はありますが、それが本当にユーザーが最も気にしていることなのか、または別のことに取り組むべきなのかを先に見極めたいと考えています。

BigIntを何に使用しているか、また遭遇する問題についてフィードバックを送ってください！[crbug.com/v8/new](https://crbug.com/v8/new)、[v8-users@googlegroups.com](mailto:v8-users@googlegroups.com)、またはTwitterで [@v8js](https://twitter.com/v8js) を通じて連絡できます。
