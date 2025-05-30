---
title: "V8における制御フローの整合性"
description: "本ブログ記事では、V8に制御フローの整合性（CFI）を実装する計画について説明します。"
author: "Stephen Röttger"
date: 2023-10-09
tags: 
 - セキュリティ
---
制御フローの整合性（CFI）は、制御フローの乗っ取りによる攻撃を防止することを目的としたセキュリティ機能です。攻撃者がプロセスのメモリを改ざんすることに成功しても、追加の整合性チェックにより任意のコード実行を防ぐことができます。本ブログ記事では、V8にCFIを有効にするための作業について説明します。

<!--truncate-->
# 背景

Chromeの人気により、それは0デイ攻撃の価値あるターゲットとなっており、これまで見られた野生の攻撃のほとんどは、初期のコード実行を得るためにV8をターゲットとしています。V8のエクスプロイトは一般的に次のようなパターンに従います：最初のバグがメモリの破損を引き起こしますが、多くの場合、最初の破損は限定的であり、攻撃者はアドレス空間全体で任意の読み取り/書き込みを行う方法を見つける必要があります。これにより、制御フローを乗っ取り、Chromeサンドボックスからの脱出を試みるエクスプロイトチェーンの次のステップを実行するシェルコードを実行することが可能になります。


攻撃者がメモリの破損をシェルコード実行に変えるのを防ぐため、V8で制御フローの整合性を実装しています。これはJITコンパイラが存在する場合に特に困難です。データをランタイムで機械コードに変換する場合、改ざんされたデータが悪意のあるコードになるのを防ぐ必要があります。幸いなことに、現代のハードウェア機能は、改ざんされたメモリを処理しながらも堅牢なJITコンパイラを設計する際の基盤を提供してくれます。


以下では、問題を3つの部分に分けて説明します。

- **前進方向のCFI** は、関数ポインタやvtbl呼び出しなどの間接制御フロー転送の整合性を検証します。
- **後退方向のCFI** は、スタックから読み取られる戻りアドレスが有効であることを確認する必要があります。
- **JITメモリの整合性** は、ランタイムで実行可能なメモリに書き込まれるすべてのデータを検証します。

# 前進方向のCFI

間接呼び出しやジャンプを保護するために使用したい2つのハードウェア機能があります：着地点パッドとポインタ認証です。


## 着地点パッド

着地点パッドは、有効な分岐ターゲットをマークするために使用できる特殊な命令です。これが有効になると、間接分岐は着地点パッド命令にのみジャンプすることができ、それ以外の場合は例外が発生します。
例えばARM64では、着地点パッドがArmv8.5-Aで導入されたBranch Target Identification（BTI）機能で利用可能です。BTIのサポートはV8で[すでに有効化されています](https://bugs.chromium.org/p/chromium/issues/detail?id=1145581)。
x64では、着地点パッドはControl Flow Enforcement Technology（CET）機能の一部であるIndirect Branch Tracking（IBT）により導入されました。


ただし、間接分岐のすべての潜在的なターゲットに着地点パッドを追加するだけでは粗粒度の制御フローの整合性しか提供されず、攻撃者にはまだ多くの自由が残されています。制限をさらに強めるために、関数シグネチャのチェック（呼び出し場所の引数と戻り値の型が呼び出される関数と一致する必要がある）を追加したり、不要な着地点パッド命令をランタイムで動的に削除することができます。
これらの機能は最近の[FineIBT提案](https://arxiv.org/abs/2303.16353)の一部であり、OS採用が進むことを期待しています。

## ポインタ認証

Armv8.3-Aはポインタ認証（PAC）を導入しており、ポインタの未使用ビット上部に署名を埋め込むことができます。署名はポインタが使用される前に検証されるため、攻撃者は間接分岐に任意の偽造ポインタを提供することはできません。

# 後退方向のCFI

戻りアドレスを保護するために、影のスタックとPACという2つのハードウェア機能を使用したいと考えています。

## 影のスタック

Intel CETの影のスタックや、[Armv9.4-A](https://community.arm.com/arm-community-blogs/b/architectures-and-processors-blog/posts/arm-a-profile-architecture-2022)におけるガード制御スタック（GCS）を使用すれば、戻りアドレス専用の別のスタックを作成でき、悪意のある書き込みに対するハードウェア保護を提供します。これらの機能は戻りアドレスの上書きに対して強力な保護を提供しますが、最適化/最適化の解除や例外処理中に戻りスタックを正当に変更するケースに対処する必要があります。

## ポインタ認証（PAC-RET）

間接分岐と同様に、ポインタ認証は戻りアドレスがスタックにプッシュされる前に署名するために使用できます。これもARM64 CPU上のV8で[すでに有効化されています](https://bugs.chromium.org/p/chromium/issues/detail?id=919548)。


前進方向および後退方向のCFIのためにハードウェアサポートを使用する副作用として、パフォーマンスの影響を最小限に抑えることが可能になります。

# JITメモリの整合性

JITコンパイラでのCFIにおけるユニークな課題は、実行時に実行可能なメモリに機械コードを書き込む必要があることです。JITコンパイラがそのメモリに書き込むことを許可しつつ、攻撃者がメモリを書き換えることができないようにメモリを保護する必要があります。単純なアプローチとして、ページの権限を一時的に変更して書き込みアクセスを追加/削除する方法がありますが、これには競争条件が生じます。攻撃者が別のスレッドから任意の書き込みを同時にトリガーできると仮定しなければならないからです。


## スレッドごとのメモリ権限

現代のCPUでは、現在のスレッドにのみ適用され、ユーザランドで迅速に変更可能なメモリ権限の異なるビューを持つことができます。
x64 CPUの場合、メモリ保護キー(pkeys)を使用して実現できます。ARMは、Armv8.9-Aで[permission overlay extensions](https://community.arm.com/arm-community-blogs/b/architectures-and-processors-blog/posts/arm-a-profile-architecture-2022)を発表しました。
これにより、書き込みアクセスを詳細に制御することが可能になります。たとえば、別のpkeyをタグ付けすることで実行可能メモリへの書き込みアクセスを切り替えることができます。


JITページはもう攻撃者が書き込めるものではなくなりましたが、JITコンパイラは依然として生成されたコードをそこに書き込む必要があります。V8では、生成されたコードはヒープ上の[AssemblerBuffers](https://source.chromium.org/chromium/chromium/src/+/main:v8/src/codegen/assembler.h;l=255;drc=064b9a7903b793734b6c03a86ee53a2dc85f0f80)に存在し、攻撃者によって破壊される可能性があります。同じ方法でAssemblerBuffersを保護することもできますが、それでは問題が移るだけです。たとえば、AssemblerBufferのポインターが置かれているメモリも保護する必要が出てきます。
実際、保護されたメモリへの書き込みアクセスを有効にするすべてのコードはCFI攻撃の対象となり、非常に慎重にコード化する必要があります。たとえば、保護されていないメモリから来たポインターへの書き込みは即座にゲームオーバーです。攻撃者がこれを利用して実行可能メモリを破壊する可能性があるためです。そのため、設計目標として、このような重要なセクションをできるだけ少なくし、そのコードを短く自己完結型に保つことを目指しています。

## 制御フローの検証

すべてのコンパイラデータを保護しない場合、それをCFIの観点から信頼できないものとみなすことができます。実行可能メモリに何かを書き込む前に、それが任意の制御フローを引き起こさないことを検証する必要があります。たとえば、書き込まれるコードがシステムコール命令を実行しないことや、任意のコードにジャンプしないことを確認する必要があります。当然、現在のスレッドのpkey権限を変更しないことも確認する必要があります。コードが任意のメモリを破損することを防ぐことは試みていません。コードが破損している場合、攻撃者が既にその能力を持っていると仮定できるからです。
そのような検証を安全に行うために、必要なメタデータを保護されたメモリに保持し、スタックのローカル変数を保護する必要があります。
そのような検証が性能に与える影響を評価するため、いくつかの予備テストを実施しました。幸いなことに、検証は性能が重要なコードパスでは発生しておらず、jetstreamやspeedometerベンチマークでの退化は観察されませんでした。

# 評価

攻撃的なセキュリティ研究は、あらゆる緩和設計の重要な部分であり、私たちは常に保護を回避する新しい方法を見つけようとしています。以下は可能だと考える攻撃例と、それらへの対応アイデアの例です。

## 破損したシステムコール引数

前述のように、攻撃者が他の実行中スレッドと同時にメモリ書き込みのプリミティブをトリガーできると仮定しています。他のスレッドがシステムコールを実行する場合、引数がメモリから読み取られると攻撃者が制御できる可能性があります。Chromeは制約されたシステムコールフィルターで動作していますが、CFI保護を回避するのに使用され得るいくつかのシステムコールがまだ存在します。


例えばSigactionは、シグナルハンドラを登録するためのシステムコールです。私たちの研究では、CFI準拠の方法でChrome内でsigaction呼び出しが到達可能であることが判明しました。引数はメモリ内に格納されるため、攻撃者はこのコードパスをトリガーし、任意のコードを指すシグナルハンドラ関数を設定することができます。幸い、この問題は簡単に対処できます。sigaction呼び出しへのパスをブロックするか、初期化後にシステムコールフィルターでブロックすれば良いのです。


他の興味深い例としては、メモリ管理のシステムコールがあります。例えば、スレッドが破損したポインターを使ってmunmapを呼び出すと、攻撃者は読み取り専用ページをマッピング解除し、続くmmap呼び出しでこのアドレスを再利用してページに書き込み権限を追加する可能性があります。
いくつかのOSは既にこの攻撃に対する保護をメモリシーリングで提供しています。Appleのプラットフォームは[VM\_FLAGS\_PERMANENT](https://github.com/apple-oss-distributions/xnu/blob/1031c584a5e37aff177559b9f69dbd3c8c3fd30a/osfmk/mach/vm_statistics.h#L274)フラグを提供し、OpenBSDには[mimmutable](https://man.openbsd.org/mimmutable.2)システムコールがあります。

## シグナルフレームの破損

カーネルがシグナルハンドラを実行するとき、現在のCPU状態をユーザランドのスタックに保存します。別のスレッドが保存された状態を破損させると、カーネルがそれを復元します。
シグナルフレームデータが信頼できない場合、ユーザ空間でこれを防ぐのは難しいようです。その時点では、常に終了するか、既知の保存状態でシグナルフレームを上書きして戻る必要があります。
より有望なアプローチは、スレッドごとのメモリ許可を使用してシグナルスタックを保護することです。例えば、pkeyタグ付きのsigaltstackは悪意のある上書きから保護できますが、CPU状態を保存する際にカーネルが一時的に書き込み権限を許可する必要があります。

# v8CTF

これらは我々が対処に取り組んでいる潜在的な攻撃のほんの一例であり、セキュリティコミュニティからさらに学びたいと考えています。興味がある場合は、最近開始された[v8CTF](https://security.googleblog.com/2023/10/expanding-our-exploit-reward-program-to.html)に挑戦してみてください！ V8を侵害して報酬を獲得しましょう。n-day脆弱性を標的とするエクスプロイトも対象として明確に含まれています！
