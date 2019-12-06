---
title: "RustのLinux muslターゲット （その1：Linux向けのポータブルなバイナリを作る）"
# summary: ""
date: 2019-12-07T19:00:00+08:00
draft: no
isCJKLanguage: true
categories:
- Rust Tips
tags:
- Linux-musl
- Docker
- 実践Rust入門
---

これは [Rustその3 Advent Calendar 2019 &mdash; Qiita][qiita-rust3-advcal-2019] の8日目の記事です。

[qiita-rust3-advcal-2019]: https://qiita.com/advent-calendar/2019/rust3

Rustの`x86_64-unknown-linux-musl`ターゲットを使って、libc（標準Cライブラリ）やSQLiteなどの外部ライブラリに静的リンクしたバイナリを作成する方法を紹介します。

こうして作ったバイナリは、Alpine Linuxを含むさまざまなx86_64 Linux環境で実行できます[^1]。
またUbuntuやCentOSなどの一般的なLinuxディストリビューションを使っている場合でも、実行環境にSQLiteやOpenSSLなどを別途インストールしなくてすみますので、バイナリの配布が楽になります。

[^1]: ただしLinuxカーネルのバージョンは2.6.18かそれ以降でなければなりません。2.6.18がリリースされたのは2006年ですので、カーネルのバージョンが問題になることはまずないでしょう

さらにこれらを極小のDockerイメージに入れることで、Webアプリケーションなどではデプロイが容易になるかもしれません。

以下の内容を2回に分けて説明します。

- **今回**：Alpine Linuxでも実行できるHello Worldバイナリを作成する
- 次回：[SQLite 3と静的リンクしたバイナリを作成し、Dockerイメージに収める][next-article]

[next-article]: ../2019-12-small-docker-image/

なお、これは今年5月に出版された『[実践Rust入門][practical-rust-primer]』の補足記事になります。
2-5-4項でクロスコンパイルを応用した極小のDockerイメージを紹介しましたが、具体的な作成方法については分量の問題から書籍内で説明できませんでした。

[practical-rust-primer]: http://gihyo.jp/book/2019/978-4-297-10559-4

**実践Rust入門の該当部分**

> ```bash
> $ docker images hello-sqlite
> REPOSITORY    TAG     IMAGE ID      CREATED        SIZE
> hello-sqlite  latest  0f60b9e23a91  5 minutes ago  1.95MB
> alpine        latest  196d12cf6ab1  2 months ago   4.41MB
> ubuntu        18.04   ea4c82dcd15a  4 weeks ago    85.8MB
> ```
>
> 　hello-sqliteはSQLiteサーバを組み込んだRustサンプルプログラムを実行するためのコンテナです。
> `x86_64-unknown-linux-musl`ターゲット向けにビルドしたバイナリを、`scratch`という空のDockerイメージに入れました。
> このバイナリにはSQLiteはもちろん、全てのRustバイナリが依存しているlibcなども埋め込まれています。
> シェルなどのLinuxコマンドがなくても実行できますので、そのサイズは1.95MBとなっており、Dockerイメージとしては極端に小さい部類に入るAlpine Linuxのイメージ（4.41MB）よりも小さくなっています。
>
> 　誌面の都合から具体的な作成手順は省略します。
> 筆者らが管理するWebサイトにて他のターゲットと共に紹介していますので、興味があればそちらをご覧になってください。

実は原稿の基本的な部分は1年前に書いてあったのですが...。
公開が遅れてすみません。

## Linux muslターゲットとは

Rustには`*-linux-musl`というターゲットがあります。
現時点（Rust 1.39）ではTier 2だけでも以下のものが用意されています。

**[Rust Platform Support][rust-platform-support]（Tier 2）より**

| ターゲット | 対象プロセッサ |
|:--|:--|
| x86_64-unknown-linux-musl | x86_64 64ビット |
| i686-unknown-linux-musl | x86 32ビット |
| i586-unknown-linux-musl | x86 32ビット SSEなし |
| aarch64-unknown-linux-musl | ARM64 |
| armv7-unknown-linux-musleabihf | ARMv7 |
| arm-unknown-linux-musleabi | ARMv6 |
| arm-unknown-linux-musleabihf | ARMv6 hardfloat |
| armv5te-unknown-linux-musleabi | ARMv5TE |
| mips-unknown-linux-musl | MIPS |
| mipsel-unknown-linux-musl | MIPS(LE) |

またTier 3にはPowerPC/PowerPC64、MIPS64、Hexagon向けのものがあります。

[rust-platform-support]: https://forge.rust-lang.org/release/platform-support.html

ではmuslとはなんでしょうか。
**musl**はLinux向けの標準Cライブラリ実装のひとつで、静的リンクに最適化されています。

**[ウィキペディア（Wikipedia）][wikipedia-musl]より**

> **musl** (マッスル) は MIT License でリリースされている Linux の標準Cライブラリ。開発者は Rich Felker。クリーンで、効率的で、標準に準拠した標準Cライブラリの実装を目標としている。1から設計されており、アプリケーションを単一のポータブルなバイナリファイルとして配布できるように静的リンクに最適化している。POSIX:2008 と C99 準拠であるとしている。Linux, BSD, glibc の非標準な関数も実装されている。

[wikipedia-musl]: https://ja.wikipedia.org/wiki/Musl

`rustup`を使ってLinux環境にRustをインストールすると、デフォルトのターゲットとして`*-linux-gnu`が選択されます。
`*-linux-gnu`と`*-linux-musl`には以下のような違いがあります。

| ターゲット | 標準Cライブラリ | リンクの方式 |
|:--|:--|:--|
| `*-linux-gnu` | glibc（GNUが開発） | 動的リンク |
| `*-linux-musl` | musl | 静的リンク |

UbuntuやCentOSを含むほとんどのLinuxディストリビューションではシステムの標準Cライブラリとしてglibcが使われています。
そしてSQLiteやOpenSSLのようなパッケージマネージャでインストールできるライブラリは、基本的にglibcと動的リンクしています。

Rustで`*-linux-gnu`ターゲットを使うと、Rustのバイナリもglibcと動的リンクします。
このようなバイナリを配布する際は、バイナリを実行するLinux環境にglibcがなければなりません。
さらに、もしバイナリがOpenSSLなどの外部ライブラリにも依存してるなら（動的リンクしているなら）、`apt`などのパッケージマネージャを使ってそれらのライブラリもインストールしなければなりません。
バイナリを一般のユーザーに使ってもらうなら、DEBやRPMなどのパッケージに入れて、適切な依存ライブラリを設定するのが無難でしょう。
確実な方法ではありますが、ちょっと面倒ですよね。

一方、Rustで`*-linux-musl`ターゲットを使うと、バイナリはmuslと静的リンクします。
またOpenSSLなどの外部ライブラリに依存しているなら、それらとも静的リンクします。
つまりRustの単一のバイナリファイルにこれら全てのライブラリが埋め込まれている状態になります。
このようなバイナリならCPUアーキテクチャさえ一致すればどんなLinux環境でも実行できます。
DEBやRPMなどのパッケージを使わず、単一のバイナリを渡すだけですみますので、気軽にバイナリを配布できます。

## *-linux-muslがデフォルトのターゲットではない理由

バイナリの配布に便利な`*-linux-musl`ターゲットですが、なぜLinux環境ではそれをデフォルトにせず、`*-linux-gnu`ターゲットが使われるのでしょうか？

それはmuslを使うとビルドが面倒になるからです。
`*-linux-musl`ではRustバイナリが外部ライブラリに依存しているときは、それなりの準備が必要です。
なぜなら、外部ライブラリはglibcと動的リンクする代わりに、muslと静的リンクしている必要があるからです。
そのようなものは`apt`などではインストールできないのが普通ですので、musl向けのgccを使ってソースコードからビルドすることになります。

とはいえ、Rustでよく使われる外部ライブラリなら、自分で準備する必要はありません。
よく使われるライブラリとRustコンパイラ/gccをセットにしたDockerイメージがいくつかありますので、それらを使えば簡単です。
[次回の記事][next-article] で紹介します。

## Hello Worldで実験

前置きが長くなりました。
今回はHello Worldをビルドしてみましょう。
このバイナリは標準Cライブラリだけに依存しますので、muslターゲットでも簡単にビルドできます。

ビルドする環境にはLinux向けのリンカが必要ですので、macOSやWindowsでビルドするならDockerを使うのが楽です。
今回もDockerを使い、Ubuntu 18.04で作成したバイナリが、CentOS 8やAlpine Linux 3.10でも実行できることを確認します。
なおDockerのインストール手順については [こちら][installing-docker] を参照してください。

[installing-docker]: https://github.com/ghmagazine/rustbook/blob/master/install/docker.md

バイナリをビルドするためのUbuntu 18.04環境（Dockerコンテナ）を起動して、Rustなどの必要なソフトウェアをインストールします。

```bash
# ターミナルからUbuntu 18.04のDockerコンテナを実行する
# コンテナ名は`ubuntu`
# 環境変数USERが設定されていないとcargo newが失敗するのでここで設定
$ docker run -it --name ubuntu -h ubuntu -e USER=root ubuntu:18.04 bash

# コマンドプロンプトを変更する
root@ubuntu:/# PS1='\h$ '
ubuntu$  　# ← プロンプトが変わった

# 必要なソフトウェアをインストールする
ubuntu$ apt update && apt install -y curl gcc vim

# rustupとRust stableをインストールする
ubuntu$ curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
ubuntu$ source $HOME/.cargo/env
```

バイナリを実行するための環境も起動しておきます。
CentOS 8とAlpine Linux 3.10にしました。

```bash
# 別のターミナルからCentOSのDockerコンテナを実行する
# コンテナ名は`centos`
$ docker run -it --rm --name centos -h centos centos:centos8 bash

# コマンドプロンプトを変更する
[root@centos /]# PS1='\h$ '
centos$
```

```bash
# さらに別のターミナルからAlpine LinuxのDockerコンテナを実行する
# コンテナ名は`alpine`
$ docker run -it --rm --name alpine -h alpine alpine:3.10

# コマンドプロンプトを変更する
/ # PS1='\h$ '
alpine$
```

Alpile Linuxはセキュリティを重視したシンプルかつ軽量なLinuxディストリビューションです。
組み込み系でよく使われているBusyBoxをベースにしているのでインストールサイズが小さいのが特徴です。
Dockerのイメージサイズを比べてみましょう。

```bash
# イメージサイズを調べる
$ docker images
REPOSITORY   TAG      IMAGE ID       CREATED        SIZE
ubuntu       18.04    775349758637   5 weeks ago    64.2MB
alpine       3.10     965ea09ff2eb   6 weeks ago    5.55MB
centos       centos8  0f3e07c0138f   2 months ago   220MB
```

Ubuntu 18.04の64.2MBに対して、Alpine Linux 3.10は5.55MBです[^2]。
Dockerではイメージが小さいほうがユーザから喜ばれる傾向がありますので、Dockerの公式イメージもUbuntuベースからAlpineベースに移行していく動きがあります。

[^2]: DockerイメージにはLinuxカーネルが含まれていないので、カーネル抜きのサイズになります。

Ubuntuに戻りHello Worldプログラムを作成します。
まずは普通の方法でビルドします。

```bash
# Hello Worldプログラムを作成して普通にビルドする
ubuntu$ cd
ubuntu$ cargo new --bin hello && cd $_
ubuntu$ cargo build --release
```

Linux環境ではデフォルトのターゲットが`x86_64-unknown-linux-gnu`に設定されています。
`ldd`コマンドでどのライブラリと動的リンクしているか調べましょう。

```bash
# helloバイナリが動的リンクしているライブラリを表示する
ubuntu$ ldd target/release/hello
    linux-vdso.so.1 (0x00007ffcdafae000)
    libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007fb286043000)
    librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007fb285e3b000)
    libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007fb285c1c000)
    libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007fb285a04000)
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fb285613000)
    /lib64/ld-linux-x86-64.so.2 (0x00007fb28647a000)
```

生成されたバイナリはlibc.soやlibpthread.soといった共有ライブラリと動的リンクしています。
一般的なLinuxディストリビューションではglibcが使われており、libpthreadなどと共に最初からインストールされています。
そのため、このUbuntu 18.04で生成したバイナリをCentOS 8にコピーしても実行できます。

```bash
# 別のターミナルを開き、コンテナ間でバイナリをコピーする
$ docker cp ubuntu:/root/hello/target/release/hello .
$ docker cp hello centos:/root/
$ docker cp hello alpine:/root/
```

```bash
# CentOS環境
centos$ cd
centos$ ls -l
-rwxr-xr-x  1  501 games 2598720 Dec  7 02:13 hello

# 問題なく実行できる
centos$ ./hello
Hello, world!
```

一方、Alpine Linuxではmuslが使われており、glibcはインストールされていません。
そのため、このバイナリをAlpine環境にコピーしても実行できません。

```bash
# Alpine環境。ファイルはここにあるのだが...
alpine$ ls -l
-rwxr-xr-x  1 501 dialout 2598720 Dec  7 02:13 hello

# 実行できない
alpine$ ./hello
/bin/sh: ./hello: not found
```

Alpine Linuxでも実行できるよう、`x86_64-unknown-linux-musl`ターゲットを使いましょう。
Ubuntu環境に戻り、`rustup`で`x86_64-unknown-linux-musl`ターゲットを追加します。

```bash
# Ubuntu環境にmusl向けのターゲットを追加する
ubuntu$ rustup target add x86_64-unknown-linux-musl
info: downloading component 'rust-std' for 'x86_64-unknown-linux-musl'
info: installing component 'rust-std' for 'x86_64-unknown-linux-musl'

# ツールチェインやターゲットの情報を表示する
ubuntu$ rustup show

Default host: x86_64-unknown-linux-gnu
rustup home:  /root/.rustup

installed targets for active toolchain
--------------------------------------

x86_64-unknown-linux-gnu
x86_64-unknown-linux-musl

active toolchain
----------------

stable-x86_64-unknown-linux-gnu (default)
rustc 1.39.0 (4560ea788 2019-11-04)
```

デフォルトのターゲットはlinux-gnuのままです。
`cargo build`に`--target`オプションを追加してmuslターゲット向けにビルドします。

```bash
# muslターゲット向けにビルドする
ubutu$ cargo build --release --target=x86_64-unknown-linux-musl
```

このターゲットでは外部ライブラリと静的リンクします。
`ldd`コマンドで調べると動的リンクではないことが確認できます。

```bash
# 動的リンクではない（つまり静的リンクしている）
ubuntu$ ldd target/x86_64-unknown-linux-musl/release/hello
    not a dynamic executable

# このバイナリはあらゆるx86_64 Linux環境で実行できるので
# もちろん、このUbuntu環境でも実行できる
ubuntu$ ./target/x86_64-unknown-linux-musl/release/hello
Hello, world!
```

バイナリを他のコンテナにコピーしましょう

```bash
# 別のターミナルでコピーする
$ docker cp ubuntu:/root/hello/target/x86_64-unknown-linux-musl/release/hello .
$ docker cp hello centos:/root/
$ docker cp hello alpine:/root/
```

今度はAlpine Linuxでも実行できました。

```bash
# Alpine Linuxのコンテナでバイナリを実行する
alpine$ ./hello
Hello, world!
```

このように標準Cライブラリだけに依存するプログラムなら、静的リンク版のLinuxバイナリを簡単に作成できます。

[次回][next-article] はSQLite 3と静的リンクしたバイナリを作成して、Dockerイメージに収めます。
