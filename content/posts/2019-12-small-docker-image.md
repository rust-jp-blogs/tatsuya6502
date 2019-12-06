---
title: "RustのLinux muslターゲット （その2：極小Dockerイメージを実現）"
# summary: ""
date: 2019-12-07T19:15:00+08:00
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

2回シリーズの2つめの記事になります。

- 前回：[Alpine Linuxでも実行できるHello Worldバイナリを作成する][previous-article]
- **今回**：SQLite 3と静的リンクしたバイナリを作成し、Dockerイメージに収める

[previous-article]: ../2019-12-statically-linked-binary/

## SQLite 3と静的リンクさせる

今回はRustのアプリケーションをSQLite 3ライブラリと静的リンクさせます。
SQLiteはCで書かれた軽量コンパクトなリレーショナルデータベース管理システム（RDBMS）で、主に小規模システムやデスクトップアプリのデータストアとして利用されています。
SQLiteはそれ単独でデータベースサーバとして実行することもできますが、今回はライブラリとしてRustアプリ内に埋め込んで使います。

まずは適当なディレクトリに移動して、パッケージ（binクレート）を作成します。

```bash
$ cargo new hello-sqlite && cd $_
```

RustからSQLiteを使うなら [rusqliteクレート][crate-rusqlite] が便利です。
もちろんORマッパー＋クエリビルダーの [Dieselクレート][crate-diesel] を使ってもかまいませんが、今回はシンプルなrusqliteを使います。

[crate-rusqlite]: https://crates.io/crates/rusqlite
[crate-diesel]: https://crates.io/crates/diesel

`Cargo.toml`の`dependencies`セクションにrusqliteを追加します。

**hello-sqlite/Cargo.toml**

```toml:hello-sqlite/Cargo.toml
[package]
name = "hello-sqlite"
edition = "2018"

[dependencies]
rusqlite = { version = "0.20.0", features = ["bundled"] }
```

`main`関数を作成します。
ほとんど意味がありませんが、SQLiteのテーブルに1から10までの整数を格納して、SQLの`SUM`関数でその合計を得ます。

**hello-sqlite/src/main.rs**

```rust
use rusqlite::{Connection, NO_PARAMS};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // メモリ上にテーブルを作成する
    // `?`演算子を使うと処理に失敗した時にmain関数から抜けてエラーを返せる
    let conn = Connection::open_in_memory()?;
    conn.execute_batch("CREATE TABLE foo(x INTEGER)")?;

    // テーブルに1から10の数字を格納する
    let mut insert_stmt = conn.prepare("INSERT INTO foo(x) VALUES(?)")?;
    for i in 1..=10 {
        insert_stmt.execute(&[i])?;
    }

    // SQLのSUM関数でその合計を得る
    let sum = conn.query_row::<i64, _, _>(
        "SELECT SUM(x) FROM foo",
        NO_PARAMS,
        |r| r.get(0)
    )?;
    println!("{}", sum);

    Ok(())
}
```

## rust-musl-builderで簡単ビルド

hello-sqliteをmuslターゲット向けにビルドしましょう。
[前回説明した][why-not-default]ようにSQLiteのような外部ライブラリを使うときは、それなりの準備が必要です。
なぜなら、外部ライブラリはglibcと動的リンクさせる代わりに、muslと静的リンクさせる必要があるからです。

[why-not-default]: ../2019-12-statically-linked-binary/#linuxmusl

とはいえ今回は自分で準備する必要はありません。
よく使われるライブラリとRustコンパイラ/gccをセットにしたDockerイメージがいくつかありますので、それらを使うのが簡単です。

今回は [ekidd/rust-musl-builder][musl-builder]（Rustマッスルビルダー）を使います。
以下のツールとライブラリが含まれています。

- musl libcライブラリ
- musl対応のgccコンパイラ
- Rustのmuslターゲット
- OpenSSLライブラリ &mdash; 多くのRustアプリケーションで必要
- libpqライブラリ &mdash; PostgreSQLのクライアントライブラリ。Dieselなどで必要
- libzライブラリ &mdash; zip, bzip, png画像などで使われている圧縮アルゴリズムのライブラリ。libpqで必要
- SQLite 3 ライブラリ

このイメージの他にもgolddranksさん作成の [registry.gitlab.com/rust_musl_docker/image][musl-docker] もあります。
OpenSSLとPostgreSQLのバージョンがいくつかの組み合わせから選べるのが特徴でしょうか。
golddranksさんはSlackの日本語Rustコミュニティ（[登録URL][slack-rust-jp]）などにもいらっしゃいますので、日本語で質問できそうなところもいいかもしれませんね。

[musl-builder]: https://github.com/emk/rust-musl-builder
[musl-docker]: https://gitlab.com/rust_musl_docker/image
[slack-rust-jp]: https://rust-jp.herokuapp.com/

ekidd/rust-musl-builderでビルドするには、`Cargo.toml`があるディレクトリに移動して、ターミナルから以下のコマンドを実行します。

```bash
# 毎回docker runをタイプするのは面倒なので、コマンドエイリアスを登録する
$ alias rust-musl-builder='docker run --rm -it \
        -v "$(pwd)":/home/rust/src ekidd/rust-musl-builder'

# ビルドする
$ rust-musl-builder cargo build --release

# linux-musl向けのバイナリが生成された
$ ls -lh target/x86_64-unknown-linux-musl/release
-rwxr-xr-x  2 tatsuya  staff   4.1M Dec  7 16:19 hello-sqlite
-rw-r--r--  1 tatsuya  staff    97B Dec  7 16:19 hello-sqlite.d
```

## 極小のDockerイメージを作成

hello-sqliteバイナリの入ったDockerイメージを作成しましょう。

まず`.dockerignore`ファイルを作成して、あとで`docker build`コマンドを実行する際に不要なファイルが処理されないようにします。

```bash
$ echo 'target' > .dockerignore
```

`Dockerfile`を書きましょう。
Dockerのマルチステージビルドを使います。
最初のステージではrust-musl-builderでバイナリを生成し、次のステージはscratchという空のDockerイメージにバイナリをコピーします。

**hello-sqlite/Dockerifle**

```Dockerfile
# バイナリのビルド用にrust-musl-builderイメージを使用する
FROM ekidd/rust-musl-builder:stable AS builder

# カレントディレクトリ（このイメージでは/home/rust/src）にソースコードを追加する
ADD . ./

# ソースコードのパーミッションを調整する（macOSだとここで止まってしまうので
# コメントアウトしている）
# RUN sudo chown -R rust:rust /home/rust

# muslターゲット向けにバイナリをビルドする
# 必須ではないがstripコマンドでデバッグシンボルなどを削ぎ落とし、バイナリを小さくする
RUN cargo build --release && \
    strip /home/rust/src/target/x86_64-unknown-linux-musl/release/hello-sqlite

# ここからは実行用のイメージを作成する
# ベースイメージとしscratch（空のイメージ）を使用する
FROM scratch

# バイナリをルートディレクトリにコピーする
COPY --from=builder \
    /home/rust/src/target/x86_64-unknown-linux-musl/release/hello-sqlite /

# このイメージからコンテナを起動するとバイナリが実行されるようにする
ENTRYPOINT ["/hello-sqlite"]
```

実行用イメージのベースイメージにはscratchを使いました。
これは中身が空っぽのイメージです。

alpineをベースにしてもいいのですが、今回作成したプログラムはシェルなどのLinuxコマンドなどが不要なのでLinuxカーネルさえあれば動作します。
ですからscratchで十分です。

もちろん用途によってはalpineなど他のイメージの方が便利なこともあります。
たとえばOpenSSLに依存している場合はルート認証局（ルートCA）のファイルが必要です。
そういうときはベースイメージをalpineにして、パッケージ管理システムの`apk`でルートCAパッケージを追加するのが楽でしょう。
（一応、alpineからscratchへルートCAファイルをコピーするという技もあります）

| ベースイメージ | サイズ | 用途 |
|:--|--:|:--|
| `scratch` | 0 | Rustバイナリといくつかのファイルだけがあればいいとき |
| `busybox` | 約1.2MB | シェルや基本的なUNIXコマンドが必要なとき |
| `alpine` | 約5.6MB | パッケージマネージャ（`apk`）が必要なとき |

Dockerfileを元にDockerイメージを作成します。
イメージ名は`hello-sqlite`にしました。

```bash
$ docker bulid -t hello-sqlite .
```

イメージが作成できたら、Dockerコンテナを実行しましょう。

```bash
$ docker run --rm hello-sqlite
55    # ← 1から10までの数字の合計が表示された
```

イメージのサイズは約1.6MBと、とても小さくなりました。

```bash
$ docker images hello-sqlite
REPOSITORY    TAG     IMAGE ID      CREATED        SIZE
hello-sqlite  latest  81be816eaab0  2 minutes ago  1.62MB

$ docker images alpine:3.10
REPOSITORY    TAG     IMAGE ID      CREATED        SIZE
alpine        3.10    965ea09ff2eb  6 weeks ago    5.55MB

$ docker images ubuntu:18.04
REPOSITORY    TAG     IMAGE ID      CREATED        SIZE
ubuntu        18.04   775349758637  5 weeks ago    64.2MB
```

## rust-musl-builderにないライブラリを使うには？

rust-musl-builderにはOpenSSLやlibpqなどWebアプリケーションに必要そうなライブラリがすでに入っています。
しかしアプリケーションによっては、これ以外の外部ライブラリが必要になることも少なくないでしょう。
そういうときはrust-musl-builderをベースにして、必要なライブラリを追加したイメージを作るのがお勧めです。

具体的なやり方については、rust-musl-builderリポジトリのadding-a-libraryの例（[Dockerfile][adding-lib-example]）を見てください。

[adding-lib-example]: https://github.com/emk/rust-musl-builder/blob/master/examples/adding-a-library/Dockerfile

## 前回と今回のまとめ

- Rustの`x86_64-unknown-linux-musl`ターゲットを使うと、libcを含む外部ライブラリに静的リンクしたバイナリが作成できる
- ekidd/rust-musl-builderなどのDockerイメージを活用すれば、準備に時間をかけることなくmuslビルドが始められる
- こうして作ったバイナリは、Alpine Linuxを含むさまざまなx86_64 Linux環境で実行できる
- UbuntuやCentOSなどでも外部ライブラリをインストールしなくてすむので、気軽にバイナリを配布できる
- Dockerのscratchイメージでも動作するので、極小のDockerイメージも作れる
