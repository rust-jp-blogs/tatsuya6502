---
title: "デバッグビルドを高速化するrustc_codegen_craneliftを試してみました"
# summary: ""
date: 2020-12-17T18:30:00+08:00
draft: no
isCJKLanguage: true
categories:
- Rustツールチェーン
tags:
- コンパイラ技術
---

## はじめに

これは [Rust Advent Calendar 2020][advcal] 8日目のエントリーです。
飛び入り参加です。

この記事では、Rustコンパイラの実験的なバックエンドである「[rustc_codegen_cranelift][github-cg-clif]」について、その特徴と使いかたを紹介します。
以下の英語記事の内容をベースに、筆者なりの説明をいろいろと追加しています。

- [Inside Rust Blog &mdash; Using rustc_codegen_cranelift for debug builds][blog-cg-clif]

[advcal]: https://qiita.com/advent-calendar/2020/rust
[github-cg-clif]: https://github.com/bjorn3/rustc_codegen_cranelift/
[blog-cg-clif]: https://blog.rust-lang.org/inside-rust/2020/11/15/Using-rustc_codegen_cranelift.html


## rustc_codegen_craneliftとは？

rustc_codegen_cranelift（cg_clif）はRustコンパイラのバックエンドのひとつです。
執筆時点（2020年12月）では実験的な実装という位置づけですが、将来はデバッグビルド時のデフォルトのバックエンドとして使われることを目標にしているそうです。

コンパイラのバックエンドとは、コンパイラのフロントエンドが生成した「プログラムの中間表現」を入力にとり、ターゲット環境で実行できる機械語などのコードを出力するソフトウェアのことです。

バックエンドが入力にとる中間表現（Internal Representation）はIRとも呼ばれ、多くの場合、CPUのアーキテクチャに依存しないデータ構造になっています。
バックエンドが出力する「実行できるコード」は、CPUの機械語だけでなくWebAssembly（wasm）のようなバイトコードなども対象になります。

バックエンドは単にコードを出力するだけでなく、コードの効率や実行速度を高めるための最適化を行うのが一般的です。

現在のRustコンパイラではデフォルトのバックエンドとしてcodegen llvm（cg_llvm）が使われています。
cg_llvmはLLVMという成熟したコンパイラ基盤を採用しており、多彩なターゲット環境向けに、高度に最適化されたコードを生成します。

一方、今回紹介するcg_clifは [Cranelift][github-cranelift] というRustで書かれたコード生成器を採用しています。
CraneliftはLLVMと比べると設計がシンプルで、入力にとるCranelift IRはLLVMのIRと比べると抽象度がやや低く、機械語に少し近いものとなっています。
そのためコードの全体を見渡した最適化は得意ではなく、LLVMと比べると実行速度に劣るコードを生成します。
しかしCraneliftはシンプルであるがゆえにコード生成にかかる時間が短くなるという長所を持ちます。

デバッグビルドの際にcg_clifを用いることで、Rustプログラムの修正から動作確認までのサイクルを短時間で回せるようになり、開発効率が上がるかもしれません。

余談になりますが、Craneliftは現在はBytecode Alliance配下のプロジェクトとなっており、ソースコードはWasmtimeというスタンドアローンのwasm実行環境のリポジトリーに取り込まれています。
WasmtimeではCraneliftをバックエンドとして使うことで、wasmバイトコードを機械語へとコンパイルしています。

[Bytecode Alliance][ba] はwasmをWebブラウザーだけでなく、PCやサーバー、IoTデバイスなどあらゆる環境でセキュアに実行することを目指した団体です。
2019年後半にインテル、Mozilla、Red Hat、Fastlyの4社によって設立されました。

[github-cranelift]: https://github.com/bytecodealliance/wasmtime/tree/main/cranelift
[ba]: https://bytecodealliance.org/


## 試用する環境について

cg_clifはまだ実験的な実装ですので、未実装の機能やバグがあります。
現時点では開発者が使っているのと同じx86_64系のLinuxで試すのが無難です。

筆者は以下の環境で試してみましたが、問題が起きなかったのは最初のx86_64系Linuxだけでした。

- Ubuntu 20.10 Desktop x86_64
    - 試した範囲では問題なく動作した
- macOS Catalina 10.15 x86_64（インテルCPU）
    - cg_clifはビルドできたので使ってみたところ、curlクレートのビルドで`rustc`がクラッシュした
- macOS Bug Sur 11.1 arm64（Apple silicon）
    - cg_clifがコンパイルエラーになり、ビルドできなかった

arm64（aarch64）はmacOSでは全然ダメでしたが、Cranelift自体はaarch64にある程度対応しているようです。aarch64系のLinuxだったらもう少し動いたのかもしれません。


## cg_clifをビルドする

cg_clifはRustのリポジトリーに取り込まれていますが、現時点ではnightly版Rustであっても特別な設定をしないとビルドされません。
Rustコンパイラ（`rustc`）をソースコードからビルドすると時間がかかるため、もっと楽にcg_clifを試せる方法が用意されています。
今回はその方法を紹介します。

筆者自身も何をしているかよくわかってないのですが、多分、以下のようなことをしているのだと思います。

1. rustupを使い、指定した日にビルドされたnightlyツールチェーンと、いくつかの追加コンポーネントを取得する
2. そのnightlyの`rustc`向けに、cg_clifを共有ライブラリとしてビルドする
3. そのnightlyの`rustc`をcg_clifの共有ライブラリと共に動かし、stdクレート（標準ライブラリ）などのsysrootをビルドする

作業を始めましょう。
適当なディレクトリーに移動して、cg_clifのリポジトリーをクローンします。

```bash
$ git clone https://github.com/bjorn3/rustc_codegen_cranelift.git
$ cd rustc_codegen_cranelift
```

ちなみに筆者が試したときのgitリビジョンは以下のとおりです。

```bash
$ git branch -v
* master 44b3310 Also emit vcode when emitting clif ir while using new style backends
```

次に`prepare.sh`というシェルスクリプトを実行します。

```bash
$ ./prepare.sh
```

これによりrustupが実行され、指定した日付のnightlyツールチェーンがダウンロードされます。またbuild_sysrootというディレクトリーの中にソースコードが用意されたり、`cargo install`コマンドでHyperfineといった性能測定用のツールがインストールされたりします。

`prepare.sh`が最後まで実行できたら、`build.sh`スクリプトを実行します。

```bash
$ ./build.sh
```

これにより、cg_clifとツールチェーンのsysrootがビルドされます。


## cg_clifを使ってみる

cg_clifを使ってRustパッケージ（Rustプロジェクト）をビルドしてみましょう。
ここではInside Rust Blogの記事を真似てCargoをソースコードからビルドしますが、もちろんどんなパッケージをビルドしても構いません。
もし問題を見つけたら、[cg_clif][github-cg-clif]のGitHubリポジトリーで報告すると喜ばれるでしょう。

[github-cg-clif]: https://github.com/bjorn3/rustc_codegen_cranelift

cg_clifのディレクトリーから一段上にあがって、Cargoのソースリポジトリーをクローンします。

```bash
$ cd ..
$ git clone https://github.com/rust-lang/cargo.git
$ cd cargo
```

ビルドにかかる時間を測りたいのですが、依存するクレートのダウンロードにかかる時間は含めたくありません。
`cargo fetch`コマンドで事前にダウンロードしておきましょう。

```bash
$ cargo fetch
```

また、CargoをビルドするためにはOpenSSLライブラリが必要です。
Ubuntuなら以下のようにしてインストールします。

```bash
$ sudo apt install libssl-dev
```

cg_clifでビルドしましょう。
以下のように`cargo.sh`スクリプトを実行します。

```bash
$ ../rustc_codegen_cranelift/build/cargo.sh build
```

このスクリプトでは環境変数を設定したり特別なコマンド引数を与えてCargoを実行することで、上で準備したnightly `rustc`、cg_clif、sysrootがCargoから使われるしくみになっているようです。

```bash
    Finished dev [unoptimized + debuginfo] target(s) in 2m 09s
```

私の環境ではビルドに2分9秒かかりました。

比較のために通常のcg_llvm（LLVMベースのバックエンド）でビルドしましょう。
cg_clifが使っているのと同じnightlyツールチェーンを使います。

```bash
$ rustup toolchain list
stable-x86_64-unknown-linux-gnu (default)
nightly-2020-12-03-x86_64-unknown-linux-gnu

$ cargo clean
$ cargo +nightly-2020-12-03 build
...
    Finished dev [unoptimized + debuginfo] target(s) in 2m 44s
```

2分44秒でした。
cg_clifを使うことでビルドにかかる時間が35秒短縮されました。
約20%の短縮です。


## 差分ビルドしてみる

一度ビルドしたあと、ソースコードの一部を修正したときのビルド時間も測定してみましょう。
まずはcg_clifです。

```bash
## 一度ビルドする
$ cargo clean
$ ../rustc_codegen_cranelift/build/cargo.sh build

## Cargoのソースコードの一部を修正する
$ cat cargo-lib.patch
diff --git a/src/cargo/lib.rs b/src/cargo/lib.rs
index bccb41121..703afa754 100644
--- a/src/cargo/lib.rs
+++ b/src/cargo/lib.rs
@@ -36,8 +36,8 @@ use anyhow::Error;
 use log::debug;
 use std::fmt;
 
-pub use crate::util::errors::{InternalError, VerboseError};
 pub use crate::util::{CargoResult, CliError, CliResult, Config};
+pub use crate::util::errors::{InternalError, VerboseError};
 
 pub const CARGO_ENV: &str = "CARGO";
 
$ git apply cargo-lib.patch

## 差分ビルドする
$ ../rustc_codegen_cranelift/build/cargo.sh build
...
    Finished dev [unoptimized + debuginfo] target(s) in 18.61s
```

cg_llvmでもやってみます。

```bash
$ git restore src/cargo/lib.rs
$ cargo clean
$ cargo +nightly-2020-12-03 build
$ git apply cargo-lib.patch
$ cargo +nightly-2020-12-03 build
...
    Finished dev [unoptimized + debuginfo] target(s) in 20.67s
```

18.61秒と20.67秒でした。
ほとんど同じですね。
実は [Inside Rust Blogの記事][blog-cg-clif-how-do-i-use] ではこのケースでcg_clifの方が少し遅くなってしまっています。
その理由はcg_clifが生成したproc macro（serde_deriveクレート）の実行速度が、cg_llvmが生成したものよりも遅いためだそうです。

proc macroはビルド時に他のクレートに先立ってコンパイルされ、残りのクレートのコンパイルの際に`rustc`が読み込んで実行します。
cg_clifによってコンパイルされたproc macroは最適化がほとんど行われていないため、実行速度の面で不利になってしまうわけです。

[blog-cg-clif-how-do-i-use]: https://blog.rust-lang.org/inside-rust/2020/11/15/Using-rustc_codegen_cranelift.html#how-do-i-use-rustc_codegen_cranelift


## Rust Analyzerでもcg_clifを使う

VS CodeのRust Analyzer拡張機能はソースファイルを保存するたびに`cargo check`コマンドを実行します。
デフォルトではstableツールチェーンが使われますので、ターミナルからnightlyツールチェーンのcg_clifを実行するのは具合がよくありません。
それぞれが用いる`rustc`のバージョンが異なるため、差分ビルドの代わりにフルビルドが走ってしまいます。
cg_clifの実行時にnightlyによるフルビルドが走り、ファイルの保存時に￼￼Rust Analyzerにより、stableによるフルビルドが走ってしまうわけです。

これを回避するにはRust Analyzerによる`cargo check`でもcg_clifと同じバージョンの`rustc`を使う必要があります。
設定方法については以下のissueを参考にしてください。

- [#1112 &mdash; How to use with Rust Analyzer?](https://github.com/bjorn3/rustc_codegen_cranelift/issues/1112)


## まとめ

本記事ではデバッグビルドを高速化するrustc_codegen_craneliftを試す方法を紹介しました。
またコンパイラのバックエンドの役割や、LLVMとCraneliftの違いについて簡単に説明しました。

Cargoをソースコードからフルビルドしたところ、ビルド時間が約20%（35秒）短縮されました。
cg_clifを開発している人がいろいろ試したところ、ビルドする対象によってはビルド時間が20%から80%くらい短縮されるようです。

一方でCargoの差分ビルドのように、ビルド時間がほとんど変わらないか、かえって長くなるケースもあることもわかりました。
proc macroのコードがあまり最適化されないことが原因のようです。
このあたりが将来どのように解決されるのかはまだよくわかりません。

筆者はcf_clifを試す前は、実験的な実装なのでほとんど動かないのではと思っていました。
しかしLinux x86_64上ならすんなりと動いたので、いい意味で驚きました。
ただ [README][readme] によると、現状では一部の種類のFFIは動作しないそうですので注意してください。

[readme]: https://github.com/bjorn3/rustc_codegen_cranelift/#wip-cranelift-codegen-backend-for-rust
