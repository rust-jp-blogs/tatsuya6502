---
title: "Rust meets Lucet：ネイティブアプリでwasm形式の動的リンクプラグインを実行する"
# summary: ""
date: 2019-12-31T12:05:00+08:00
draft: no
isCJKLanguage: true
categories:
- Rust Tips
- WebAssembly
tags:
- WebAssembly
- WASI
- Lucet
---

## はじめに

これは[WebAssembly Advent Calendar 2019][advcal]の8日目のエントリです。
（空いていたので飛び入り参加です）　12月28日にコードを書き始めて、大晦日に記事公開となりました。

[advcal]: https://qiita.com/advent-calendar/2019/wasm

この記事ではRust製アプリにwasm形式のプラグインを動的にリンクして実行する方法を紹介します。
WebAssemblyのランタイム環境にはFastlyが開発したLucetを用います。
これによりWebブラウザを通さず、Rustアプリ上で直接WebAssemblyコードが実行できます。
WebAssemblyコードはLucetのコンパイラで事前にネイティブコードにコンパイルしますので、Rustで書かれたコードに近い速度で実行できます[^1]。

[^1]: ただし、現状ではWebAssemblyからSIMD命令などを直接使えないので、コンパイラの性能にもよりますが、Rustコンパイラで直接ネイティブコードを作成した場合よりは遅くなりそうです。また現状ではWebAssemblyコードはシングルスレッドで実行されます。

プラグインにwasm形式を採用すると、特定の言語に縛られずにプラグインを開発できるというメリットがあります。
またファイルアクセスなどの制限のかかったサンドボックス内でプラグインを実行しますので、ある程度の安全性を確保できます。
なおWebAssemblyのシステムインターフェースとしてWASIを使いますので、許可されたプラグインならファイルアクセスなども可能になります。

サンプルコードのプラグイン側は一方をRustで、もう一方をCで実装しました。アプリ本体はRustで実装しましたが、Lucetのランタイムはホスト側のAPIとしてRust APIだけでなくC APIも備えていますので、C++やPythonなどC FFIが使用できる言語なら同じようなしくみが構築できると思います。



## 元記事について

この記事は、dalanceさんとubnt_intrepidさんが以下の記事で提案されていたRustのプラグインシステムを参考にして書きました。

- [Rustで動的ロードによるプラグインシステムを作る][qiita-dalance]（dalanceさん、2019-12-24）
- [Rustの（FFIまわりの安全性を考慮した）プラグインシステム][qiita-ubnt_intrepid]（ubnt_intrepidさん、2019-12-25）

[qiita-dalance]: https://qiita.com/dalance/items/1593b56ad3744c469643
[qiita-ubnt_intrepid]: https://qiita.com/ubnt_intrepid/items/6c223eecb97d6f0285f0

元記事ではRustで書かれたアプリから、Rustで書かれた共有ライブラリをプラグインとしてロードして実行しています。
この記事はそれらのプラグイン部分をwasmに置き換えたものになります。

筆者はLucetが公開されたときから、WebAssembly自体とLucetを試してみたいと思っていました。
しかしプラグインシステムのようなものが必要になる機会もなく、気がついたら公開から1年余りが過ぎていました。
この記事を書くきっかけを提供してくださったお二人に感謝します。

なお筆者にとってはこれが初めてのWebAssemblyプログラミングになりますので、間違いなどあるかもしれません。
（特にゲストメモリあたり）　もしなにかお気づきの点があれば、コメント欄などでお知らせください。

## サンプルプロジェクト

リポジトリはこちらになります。

- https://gitlab.com/tatsuya6502-rust-mokumoku/rust-wasm-plugin-example

元記事と同様、電卓プログラムに演算子をプラグインで追加するといった感じのサンプルプログラムです。
アプリ本体とプラグインを2つ含んだCargoワークスペースになっています。

- **calc**：電卓プログラムの本体
    - Rust + Lucetで実装したLinux x86_64ネイティブアプリ
- **calc-plugin-add**：足し算を行うプラグイン
    - Rustで実装。Rustコンパイラでwasm形式へコンパイル後、Lucetコンパイラでx86_64ネイティブコードを含んだ共有ライブラリ（so）へ変換する
- **calc-plugin-mul**：掛け算を行うプラグイン
    - Cで実装。clangでwasm形式へコンパイル後、Lucetコンパイラでx86_64ネイティブコードを含んだ共有ライブラリ（so）へ変換する

実際にプラグインを使っている様子は`calc/src/main.rs`にあります。

```rust
use calc::PluginManager;
use std::path::PathBuf;

fn main() -> anyhow::Result<()> {
    // プラグインマネージャのインスタンスを作成する
    let mut pm = PluginManager::default();
    // プラグインマネージャをとおして2つのプラグインをロードする
    pm.load(PathBuf::from("../target/wasm32-wasi/release/calc_plugin_add.so"))?;
    pm.load(PathBuf::from("../target/wasm32-wasi/release/calc_plugin_mul.so"))?;

    // それぞれのプラグインを使用する
    for plugin in pm.plugins() {
        println!("Plugin: {}", plugin.name()?);
        println!("Calc: 1 {} 2 = {}", plugin.operator()?, plugin.calc(1, 2)?);
    }

    Ok(())
}
```

プラグインマネージャのインスタンスを作成し、2つのプラグインをロードします。
続けて各プラグインの`calc`で計算しています。

実行すると以下のようになります。

```bash
Plugin: add
Calc: 1 + 2 = 3
Plugin: mul
Calc: 1 * 2 = 2
```

元記事と同じ実行結果になります。

## プラグインシステムにWebAssemblyを使うメリット

WebAssemblyはWebブラウザ上でネイティブコードに近い実行速度で高速に実行できるバイナリフォーマットとして誕生しました。
現在ではLucetのようなランタイムが現れ、Webブラウザ外の環境でも実行できるようになっています。
またそのようなブラウザ外の環境で、メモリ、スレッド、ファイル、ネットワークなどを安全に使うためのインターフェースとしてWASI（ワズィと発音）という標準規格の策定も進められています。

アプリケーションのプラグインとしてwasm形式の動的ロードライブラリを使うことで、以下のようなメリットがあります。

- 言語非依存
    - プラグインをさまざまな言語で開発できる
    - C、C++、Rust、Go、AssemblyScriptなどからwasmを生成できる
- クロスプラットフォーム
    - wasmファイルはCPU非依存の命令で構成されている。ランタイム環境で実行されるときになって初めてネイティブコードにコンパイルされる
    - プラグインを提供する側は単一のwasmファイルを用意するだけでよい。ユーザはそれを好きな環境に持っていってネイティブコードへコンパイル、実行できる
    - ただしLucetは現状ではx86_64系のLinuxと、実験的にmacOSをサポートしているのみ。将来はARMなどもサポートしていく予定
- セキュリティ
    - 使用メモリの上限やファイルアクセスの制限などのあるサンドボックス内で実行されるので、他人が開発したプラグインでも、比較的安全に実行できる

## Lucetとは

[Lucet][lucet]はWebブラウザ外で使えるWebAssembly実行環境で、コンパイラなどのツールチェインとランタイムで構成されます。
CDNプロバイダとして知られるFastlyがオープンソースで公開し、現在は[Bytecode Alliance][bytecode-alliance]傘下のプロジェクトになっています。

Lucetには以下のような特徴があります。

### 事前コンパイル方式

Lucetでは`lucetc`という最適化コンパイラで、wasmをネイティブコードにAOTコンパイル（事前コンパイル）します。
この方式ではコンパイルにかかる時間は長くなりますが、コンパイル済みのコードが最初から高速に実行できます。
同じコードを繰り返し使うプラグイン的な用途に適しているといえるでしょう。

一方、Webブラウザではダウンロードしたwasmを即実行しなければならないため、AOTとJITコンパイルを組み合わせるのが普通です。
たとえば、まず最適化をほとんどしない高速コンパイラでAOTコンパイルし、その後、最適化コンパイラによるJITコンパイルで実行中のコードを最適化されたものに差し替えていきます。
この方式はコードの1回限りの実行には適していますが、同じコードを繰り返し使う用途には無駄が多くなります。

### インスタンスの柔軟な設定

Lucetでは動的ロードしたwasmライブラリをインスタンス化する際に柔軟な設定ができます。

- 使用するリソースの上限を設定
    - 現状はメモリの上限と、WASI使用時のファイルアクセスの可否が指定できる
        - CPUの使用量などは制限できない
- コンテキストと呼ばれるしくみで機能拡張できる
    - 素のWebAssemblyか、WASIありかを選択できる
    - Lucet Hostcallsのコンテキストを作成すると、コンテキストで定義されたホスト側の関数をプラグインから呼ぶこともできる
        - 詳しくは見ていないが非同期実行もできるみたい
- メモリ領域はインスタンスごとに独立したものを与えることもできるし、ひとつの領域を複数のインスタンスで共有させることもできる
    - ホスト側からはすべてのゲスト用メモリ領域にアクセスできる

### Rust以外の言語でもランタイムを使用できる

ランタイムのホスト側のAPIはRust APIに加えてC APIが用意されています。
そのため、C FFIをサポートするさまざまな言語から利用できます。
なおランタイム自体はRustで書かれています。

Lucetの特徴は以上になります。

余談ですが、Lucetが生まれた背景についてはPublickeyの独占インタビュー記事が面白かったです。

- [Fastly CTOに聞く、同社がWebAssembly実行環境の「Lucet」をエッジコンピューティング環境として開発している理由とは？][publickey-article]（Publickey、2019-05-14）

同記事によると、Lucetという名前は、バイキングがロープや組ひもなどを作るために使った道具に由来しているそうです。

[Lucet]: https://github.com/bytecodealliance/lucet/
[bytecode-alliance]: https://bytecodealliance.org/
[publickey-article]: https://www.publickey1.jp/blog/19/fastly_ctowebassemblylucet.html

## 環境のセットアップ

コードの内容を説明する前に、サンプルコードのビルドと実行に必要な環境のセットアップ方法を説明します。

### Linux x86_64とDocker

Lucetを使用するには現状ではx86_64上のLinuxが必要です。
（macOSも実験的にサポートしているようです）　筆者はFedora 31を使用しました。

またLucetのツールチェインを簡単にセットアップするためにDockerも必要です。

### Rust

Rustはstable版を使います。
筆者は最新のRust 1.40.0を使用しました。

ターゲットを追加するので、[`rustup`][rustup]でインストールするのがお勧めです。
stable版がインストールできたら、`wasm32-wasi`ターゲットを追加します。

```bash
$ rustup target add wasm32-wasi
```

またx86_64環境向けのリンカが必要です。`sudo apt install gcc binutils`（Ubuntuの場合）などでインストールしてください。

[rustup]: https://rustup.rs

### Lucetツールチェイン

Lucetのツールチェイン（AOTコンパイラなど）はDockerイメージ内に構築されます。
以下のようにしてセットアップします。

```bash
# Lucetリポジトリをクローンする
$ git clone https://github.com/bytecodealliance/lucet.git
$ cd lucet

# このあとcalcアプリ本体で使用するLucetランタイムのバージョンと
# 同じ名前のタグをチェックアウトする
$ git checkout 0.4.1

# 開発環境のセットアップスクリプトを読み込む
# 初回の読み込みでは、ツールチェインを含んだDockerイメージがビルドされる
$ source ./devenv_setenv.sh

# 以下のDockerイメージが作られる
$ docker images | grep lucet
lucet          latest      9576e750e3de     1 hours ago    3.7GB
lucet-dev      latest      dd5c8b1b4661     1 hours ago    3.51GB
```

## コードの解説

ここからはコードの内容を解説していきます。

さきほどクローンした`lucet`ディレクトリの下にサンプルコードを置きます。
というのは、LucetのDockerベースのツールチェインがアクセスできるのは`lucet`ディレクトリ配下のファイルに限定されているからです。

```bash
$ pwd
lucet

# サンプルリポジトリをクローンする
$ git clone https://gitlab.com/tatsuya6502-rust-mokumoku/rust-wasm-plugin-example.git
```

なおサンプルコードをCargoワークスペースにはしたものの、コンポーネントごとにコンパイルのターゲットが異なったりするため、`cargo build`で全体を一括してビルドすることができません。
今回は`calc`、`calc-plugin-*`のそれぞれに対してコマンドを叩いてコンパイルします。

### プラグインのAPI

元記事では各プラグインが提供する機能（API）を`Plugin`トレイトとして定義していました。

**元記事より引用**

```rust
pub trait Plugin {
    fn name(&self) -> String;
    fn operator(&self) -> String;
    fn calc(&self, a: u32, b: u32) -> u32;
}
```

電卓として必要なのは`calc`だけですが、表示用に`String`を返すメソッドも用意されています。

WebAssemblyで実現する場合は好きな言語で書ける反面、APIも複数の言語で通用する形式で定義しなければなりません。
このようなケースでは、APIをインターフェース記述言語（IDL）という専用の言語で記述し、それを元に実装言語向けのスタブコードを生成するのが一般的です。
LucetにもIDLがあるのですが、今回は使用しませんでした。
というのは現状で生成できるのはCのコードだけなので、Rustで実装するときにには、あまり役に立ちそうもなかったからです。IDLはまた別の機会に試してみようと思います。

というわけで今回はスタブなどを自動生成せず、手でコードを書いていきます。
このやり方ですと、もし関数のシグネチャ（名前と引数）を間違えたり、関数自体を定義し忘れたりしても、コンパイルエラーにはなってくれません。
実行時のエラーになります。
ちょっと不便ですが今回のように小規模なAPIなら大きな問題にはならないでしょう。

### calc関数の実装（Rust版）

まずは実際に計算を行う`calc`関数を実装します。Rust版のcalc-plugin-addでは以下のようになります。

**calc-plugin-add/lib.rs**

```rust
#[no_mangle]
pub extern "C" fn calc(a: u32, b: u32) -> u32 {
    a + b
}
```

FFI（外部関数インターフェース）を通じて呼び出せるように`#[no_mangle]`と`extern "C"`が必要ですが、それ以外は特に難しい事はありません。

Rust版プラグインの`Cargo.toml`も作成します。

**calc-plugin-add/Cargo.toml**

```toml
[package]
name = "calc-plugin-add"
...
edition = "2018"

[lib]
path = "lib.rs"
crate-type = ["cdylib"]
```

クレートタイプは`cdylib`にします。

またコンパイルターゲットはwasm32-wasiになります。
`.cargo/config`を作成してデフォルトのターゲットを変更します。

**calc-plugin-add/.cargo/config**

```toml
[build]
target = "wasm32-wasi"
```

### calc関数の実装（C版）

C版のcalc-plugin-mulは以下のようになります。

**calc-plugin-add/c-src/calc_plugin_mul.c**

```c
#include <stdint.h>

uint32_t calc(uint32_t a, uint32_t b)
{
    return a * b;
}
```

こちらはごく普通のCプログラムですね。

### ホスト側のCargoパッケージの設定

プラグイン側に並行して本体側の実装も見ていきます。
calcパッケージの`Cargo.toml`の依存クレートは以下のようになっています。

**calc/Cargo.toml**

```toml
[dependencies]
anyhow = "1"
lucet-runtime = "0.4"
wasi-common-lucet = "0.3"
```

元記事にならってエラー処理用のanyhowクレートを使用します。

lucet-runtimeがWebAssemblyランタイムです。

このランタイムのバージョンと、Lucetツールチェインのバージョンが合っていないと、せっかく作成した共有ライブラリがバージョン不一致のエラーでロードできなくなるので注意してください。
筆者はそれで理由でツールチェインのDockerイメージを再構築することになってしまいました。
この記事の最初の方の手順に従って、lucetリポジトリのタグをチェックアウトしていれば大丈夫です。
（`git checkout 0.4.1`）

wasi-common-lucetはWASIのサポートをランタイムに追加するためのものです。

実は実装し始めたときはWASIなしの素のWebAsseblyで書いていたのですが、それだとデバッグが難しく、諦めてWASIを追加しました。
ランタイムのメモリ制限の設定値を勘違いしていてRustプラグインがヒープアロケーションでpanicを起こしていたのですが、WASIなしだと標準出力と標準エラー出力がないのでpanic時のメッセージ（out of memoryエラー）が出力されず、辛かったです。

ホスト側（calcパッケージ）のRustコンパイラには追加のパラメータが必要です。
`.cargo/config`ファイルを追加して以下の内容を書きました。

**calc/.cargo/config**

```toml
[build]
rustflags = ["-C", "link-args=-rdynamic"]
```

実はこれと同じ内容が親ディレクトリである`lucet`リポジトリの`.cargo/config`にも書かれていて、Cargoはそれを探し出して読んでいるようです。
ですから、いまのディレクトリ構成ですと、ここで指定しなくても大丈夫かもしれません。
しかし将来、calcを`lucet`ディレクトリの外でビルドすることもあるかもしれないので、念のためここにも書いておきます。

### PluginManager

ロードした個々のプラグインを表す`Plugin`と、それらを管理する`PluginManager`を実装します。
まずは構造体の定義です。

**calc/src/lib.rs**

```rust
use lucet_runtime::{DlModule, InstanceHandle, Limits, MmapRegion, Region};
use std::{ffi::CStr, path::Path};
use wasi_common_lucet::WasiCtxBuilder;

// Pluginは個々のプラグインに対応する
pub struct Plugin {
    // Lucetランタイムにロードされたプラグインのインスタンスへのハンドル
    instance: InstanceHandle,
}

// PluginManagerは全てのPluginを管理する
pub struct PluginManager {
    plugins: Vec<Plugin>,
}
```

`Plugin`はLucetランタイムにロードされたプラグインのインスタンスへのハンドルを持ちます。
`PluginManager`はロードした`Plugin`を保持する`Vec`を持ちます。

`PluginManager`の実装は以下のとおりです。

```rust
impl Default for PluginManager {
    // PluginManagerを作成する
    fn default() -> Self {
        Self::init();
        Self {
            plugins: Vec::default(),
        }
    }
}

impl PluginManager {
    fn init() {
        // ランタイムとWASI関連の機能を動作させるためのおまじない
        // これらがないと--releaseモードでビルドした時にリンクエラーになるみたい
        lucet_runtime::lucet_internal_ensure_linked();
        wasi_common_lucet::export_wasi_funcs();
    }

    // プラグインをロードして、ゲストメモリ領域（Region）と共にインスタンス化する
    pub fn load<P: AsRef<Path>>(&mut self, path: P) -> anyhow::Result<()> {
        // プラグインのインスタンスごとのメモリ制限（Limits）を定義する
        // Rustで書かれたプラグインは初期メモリの割り当て量がheap_memory_sizeの
        // デフォルト制限（1MB）を超えてしまうので、その値を増やしておく
        //
        // なおheapと呼ばれているが、Rustでいうヒープ領域とは意味が異なる
        // ここではゲストが読み書き可能かつ動的に拡張可能なメモリ領域という意味になる
        // その中をどう使うかはプラグインに任されており、RustやCだとヒープ領域に
        // 加えてスタック領域としても使っているらしい
        //
        // Rustで書かれたプラグインはスタック領域のために1MB超必要らしい
        // WebAssemblyのメモリは1ページあたり64KBなので17ページ必要（1.06MB）
        // これに加えてヒープ領域として最低1ページ必要。余裕をみて2ページまで
        // 使えるようにする
        let mem_pages = 17 + 2;
        let limits = Limits {
            heap_memory_size: mem_pages * 64 * 1024,
            ..Limits::default()
        };

        // LucetのDlModule（動的リンクモジュール）でwasmの共有ライブラリ（so）を
        // ロードする
        let module = DlModule::load(path).map_err(anyhow::Error::msg)?;
        // LucetのMmapRegionでプラグインが使用するゲストメモリを作成する
        // 作成時の引数として、上で作成したLimits（メモリ制限設定）を与える
        let region = MmapRegion::create(1, &limits).map_err(anyhow::Error::msg)?;
        // ゲストメモリに紐づくプラグインのインスタンスを作成する
        let mut instance = region.new_instance(module).map_err(anyhow::Error::msg)?;

        // インスタンスにWASIコンテキストを追加する
        // これによりプラグインがpanicしたときにpanicのメッセージが表示される
        // ようになる（デバッグに便利）
        let wasi_ctx = WasiCtxBuilder::new()
            // ホストの標準入出力を継承させる
            .and_then(|x| x.inherit_stdio())
            .and_then(|x| x.build())
            .map_err(anyhow::Error::msg)?;
        instance.insert_embed_ctx(wasi_ctx);

        // インスタンスへのハンドルをPluginの値にセットして、plugins Vecに格納する
        self.plugins.push(Plugin { instance });
        Ok(())
    }

    // 管理しているすべてのPluginを返す
    pub fn plugins(&mut self) -> &mut [Plugin] {
        &mut self.plugins
    }
}
```

`init`関数で呼んでいるLucetの2つの関数ですが、これらについてのドキュメントがないので、なにをするためのものかはわかりません。
恐らく動的リンクモジュールのリンクエラーを防ぐためのものだと想像しています。
動的リンクモジュールから使われる予定の関数が、ビルド時の最適化で削除されないようにする効果があるようです。

特に1つ目の`lucet_runtime::lucet_internal_ensure_linked()`関数はコード例にも載っていません。
しかしこれがないと本体側を`--release`モードでビルドした際、実行時にプラグインの動的リンクに失敗します。
この関数がクローズ済みのpull request（Lucetランタイムのバージョン0.5向け）の修正後のコードで呼ばれていることを見つけて、ここに追加したところその問題が解決しました。
Lucetランタイムのバージョン0.5以降では1つ目の関数はランタイム内部で呼ばれるので不要になるようです。

プラグインをロードする`load`関数では最初に`lucet_runtime::Limits`というプラグインのインスタンスに課す制限を定義しています。
現状では設定できるのはメモリ関連の上限値だけのようです。

このなかの`heap_memory_size`の設定値を変更します。
デフォルトの1MBでは、Rustで書いたプラグインをインスタンス化するときに制限値オーバーでエラーになってしまうためです。

なお、ここではヒープと呼ばれていますが、このWebAssemblyのヒープはRustでいうヒープ領域を表すわけではありません。
ゲストが読み書き可能かつ動的に拡張可能なメモリ領域という意味になります。
そのゲストメモリの中をどう使うかはプラグインに任されており、RustやCだとヒープ領域に加えてスタック領域としても使っているようです。

WebAssemblyのゲストメモリのページサイズは64KBです。
Rustのプラグインはスタック領域として1MB超割り当てるようなので、17ページ（1.06MB）の使用を許可したら動きました。
これに加えてヒープ領域も必要です。
今回はヒープ領域は文字列を格納するくらいにしか使わないので1ページで十分ですが、余裕をみて2ページまで許可しましょう。
`heap_memory_size`は合計で19ページ分の1.19MBに設定しました。

Lucetの`DlModule::load`関数でwasmの共有ライブラリを読み込みます。
そして`MmapRegion::create`関数でゲストメモリを作成します。
`create`の引数に先ほど定義した`Limits`を与えます。

`MmapRegion`の`new_instance`メソッドで、そのゲストメモリに紐づくプラグインのインスタンスを作成します。
`new_instance`メソッドはインスタンスへのハンドル（`InstanceHandle`）を返すので、それを`Plugin`構造体に格納します。

Lucetの関数やメソッドの呼び出し後、`?`演算子の前に`map_err(anyhow::Error::msg)`を付けました。
Lucetのエラー型は`std::error::Error`トレイトを実装していないので、`?`演算子で`anyhow`のエラー型に暗黙的に変換できません。
`map_err()`で明示的に変換しています。

### Plugin側のcalcメソッドの実装

`Plugin`に`calc`メソッドを実装します。

```rust
impl Plugin {
    pub fn calc(&mut self, a: u32, b: u32) -> anyhow::Result<u32> {
        Ok(self
            .instance
            .run("calc", &[a.into(), b.into()])
            .map_err(anyhow::Error::msg)?
            .unwrap_returned()
            .into())
    }
}
```

このように、WebAssemblyの関数は`self.instance`が束縛された`InstanceHandle`の`run`メソッドで実行します。

`run`メソッドの戻り値は`lucet_runtime::RunResult`です。
これは`enum`として定義されており、バリアントとして戻り値を示す`Returned(UntypedRetVal)`とコルーチンのyieldを示す`Yielded(YieldedVal)`を持ちます。
後者はLucetのhostcallの使用時に現れることがあるようです。
今回は`Returned`バリアントしか使わないので、`unwrap_returned`メソッドでアンラップしてます。

なお`unwrap_returned`メソッドは値が`Yielded`バリアントだったときはpanicするので注意が必要です。
ホスト側をもっと堅牢にしたければ、`Result<UntypedRetVal, Error>`を返す`returned`メソッドを使用してください。

最後に`UntypedRetVal`の`into`メソッドで、値を`u32`型に変換しています。

### Stringを返す関数（Rust版）

**元記事より引用（再掲）**

```rust
pub trait Plugin {
    fn name(&self) -> String;
    fn operator(&self) -> String;
    fn calc(&self, a: u32, b: u32) -> u32;
}
```

`name`関数と`operator`関数を実装します。

WebAssemblyで`String`を返すのは少し面倒です。
なぜならWebAssemblyの組み込み型には`String`のような複合型がないからです。
そのため関数の引数として受け取ったり、戻り値として返すことができません。

Rustのwasm-bindgenは`String`などの複合型を簡単に扱えるようにJavaScriptコードを自動生成してくれます。
しかしLucetではFFIの相手がJavaScriptではないので、wasm-bindgenは使えません。
複合型の扱いは自分で実装することになります。

プラグインのインスタンスが使用するメモリ（ゲストメモリ）は、本体側（ホスト側）からでもアクセスできます。
プラグイン側の関数がゲストメモリのどこに文字列を格納したのか、そのアドレスを返せば、ホスト側でその文字列にアクセスできます。
文字列データのメモリ上のフォーマットは自由に決められますが、プラグインをRust以外の言語で実装することを考慮して、FFIで一般的に使われているC形式のヌル終端文字列を使うことにします。

Rust版のcalc-plugin-add側の実装はこうなります。

**calc-plugin-add/lib.rs**

```rust
use std::{ffi::CString, os::raw::c_char};

#[no_mangle]
pub extern "C" fn name() -> *mut c_char {
    str_to_char_ptr("add")
}

#[no_mangle]
pub extern "C" fn operator() -> *mut c_char {
    str_to_char_ptr("+")
}

fn str_to_char_ptr(s: &str) -> *mut c_char {
    // 文字列リテラルからCStringを作成する
    let s = CString::new(s).expect("Can't create a CString");
    // ヌル終端文字列への可変の生ポインタを得る
    s.into_raw()
}
```

`str_to_char_ptr`はプライベート（非公開）の関数です。
この関数はまず文字列リテラルから`CString`を作成します。
これによりゲストメモリのヒープ領域にヌル終端文字列が作られます。
次に`into_raw()`でヌル終端文字列の先頭アドレスへの可変の生ポインタ（`*mut c_char`）を取得しています。

`into_raw()`を呼んだことで`CString`の所有権が追跡されなくなり、`str_to_char_ptr`関数からリターンしたあともヌル終端文字列がゲストメモリ上に残ります。

ホスト側では`name`関数や`operator`関数が返した文字列の先頭アドレスを使ってゲストメモリ上のヌル終端文字列にアクセスし、その内容から自分のメモリ上に`String`を作成します。

（言葉ではわかりにくいので、後日、図を追加しようと思います）

ホスト側のコードは以下のようになります。
文字列の取り出しは`transfer_string`関数が行います。

**calc/src/lib.rs**

```rust
impl Plugin {
    pub fn name(&mut self) -> anyhow::Result<String> {
        let val = self
            .instance
            .run("name", &[])
            .map_err(anyhow::Error::msg)?
            .unwrap_returned();
        self.transfer_string(val)
    }

   pub fn operator(&mut self) -> anyhow::Result<String> {
        // プラグインのoperator関数を呼び出す
        let val = self
            .instance
            .run("operator", &[])
            .map_err(anyhow::Error::msg)?
            .unwrap_returned();
        // 戻り値（ヌル終端文字列のアドレス）を起点にして文字列を取り出す
        self.transfer_string(val)
    }

    pub fn calc(&mut self, a: u32, b: u32) -> anyhow::Result<u32> {
        // 既出のため内容は省略
    }

    // ヌル終端文字列のアドレスを起点にして文字列を取り出す
    fn transfer_string(&mut self, val: lucet_runtime::UntypedRetVal) -> anyhow::Result<String> {
        // ゲストメモリは&[u8]型
        let heap = self.instance.heap();
        // ヌル終端文字の先頭のアドレス（メモリ領域上のオフセット）を取得
        let start = val.as_u32() as usize;
        // ヌル文字（0u8）を見つける
        let mut end = start;
        while heap[end] != 0 {
            end += 1
        }
        // CStrを作成する（この時点ではまだデータは参照されるだけで、コピーされていない）
        let c_str = CStr::from_bytes_with_nul(&heap[start..=end])?;
        // CStrからStringを作成する（ここでデータがコピーされる）
        let s = String::from(c_str.to_string_lossy());
        // ゲストメモリ上のCStringはもう不要なので、プラグイン側のrelease_string関数を
        // 呼んで開放する
        self.instance
            .run("release_string", &[start.into()])
            .map_err(anyhow::Error::msg)?;
        Ok(s)
    }
}
```

本体側のヒープに`String`を作成したあとは、ゲスト側のヌル終端文字列は不要になります。
プラグインの`release_string`関数を呼び出す事で削除します。

calc-plugin-addの`release_string`関数の実装は以下のとおりです。

**calc-plugin-add/lib.rs**

```rust
#[no_mangle]
pub unsafe extern "C" fn release_string(ptr: *mut c_char) {
    // ヌル終端文字列への生ポインタから、CStringを再構築する
    // これにより所有権が再び追跡されるようになる
    CString::from_raw(ptr);

} // CStringがスコープを抜けるので、使用していたヒープ領域（ゲストメモリ）が解放される
```

### Stringを返す関数（C版）

C版のcalc-plugin-mulでは`String`を返す関数は以下のようになります。

```c
#include <stdbool.h>
#include <stdlib.h>
#include <string.h>

// ヒープ割り当てに成功したかチェックする関数
// Lucetのテスト用のコードから拝借した
// https://github.com/bytecodealliance/lucet/blob/master/lucet-runtime/lucet-runtime-tests/guests/entrypoint/use_allocator.c
//
// 失敗時はWebAssemblyのunreachable命令を実行する（結果として関数の実行がアボートされる）
static void assert(bool v)
{
    if (!v) {
        __builtin_unreachable();
    }
}

char* name()
{
    char str0[] = "mul";
    // ゲストメモリ上のヒープ領域に文字列用のスペースを確保する
    char *str = (char*) malloc(sizeof(char) * strlen(str0) + 1);
    assert(str);
    // スペースが確保できたら文字列リテラルからバイト列をコピーする
    strcpy(str, str0);
    return str;
}

char* operator()
{
    char str0[] = "*";
    char *str = (char*) malloc(sizeof(char) * strlen(str0) + 1);
    assert(str);
    strcpy(str, str0);
    return str;
}

void release_string(char *str)
{
    // ゲストメモリ上のヒープ領域にある文字列用のスペースを開放する
    free(str);
}
```

筆者はCの万年初心者なので、これが模範的なコードなのかはあまり自身がありません。

コードの解説は以上になります。

## サンプルコードのビルドと実行

サンプルコードをビルドしていきましょう。
まず`lucet`ディレクトリに移動してLucetツールチェインを使えるようにします。

```bash
$ pwd
lucet

# ツールチェインを使えるようにする
# バックグランドでDockerコンテナが起動し、シェルのaliasが設定される
$ source ./devenv_setenv.sh

# サンプルプログラムのディレクトリへ移動する
$ cd rust-wasm-plugin-example
$ pwd
lucet/rust-wasm-plugin-example
```

### calcのビルド

まずはcalc（Rustアプリ本体）のビルドです。
ターゲットはx86_64-unknown-linux-gnuです。

```bash
$ (cd calc && cargo build --release)
$ ls -lh target/release/calc
-rwxr-xr-x. 2 tatsuya tatsuya 7.3M ... target/release/calc
```

### calc-plugin-addのビルド

次にcalc-plugin-addをビルドします。
このプラグインはRustで書かれており、Rustコンパイラでwasmファイルへとコンパイルします。
ターゲットはwasm32-wasiで、`calc-plugin-add/.cargo/config`で指定してあります。

なお上位の`lucet`ディレクトリにも`.cargo/config`ファイルがあって、それのせいでビルドエラーになってしまいます。
そのファイルをいったん削除してからビルドします。

```bash
# lucetディレクトリ配下のCargo設定ファイルをいったん削除する
$ rm ../cargo/config

# Rustコンパイラでプラグインをビルドしてwasmファイルを作成する
$ (cd calc-plugin-add && cargo build --release)
$ ls -lh target/wasm32-wasi/release/calc_plugin_add.wasm
-rwxr-xr-x. 2 tatsuya tatsuya 1.8M ... target/wasm32-wasi/release/calc_plugin_add.wasm

# Cargo設定ファイルを復元する
$ (cd .. && git restore .cargo/config)
```

### calc-plugin-mulのビルド

calc-plugin-mulをビルドします。
このプラグインはCで書かれており、clangでwasmファイルへとコンパイルします。
Lucetのツールチェイン（Dockerコンテナ上）にある`wasm32-wasi-clang`コンパイラを使います。

```bash
# wasm対応版のclangでプラグインをビルドしてwasmファイルを作成する
$ wasm32-wasi-clang -Ofast -nostartfiles -Wl,--no-entry,--export-all \
      -o target/wasm32-wasi/calc_plugin_mul.wasm \
      calc-plugin-mul/c-src/calc_plugin_mul.c

$ ls -lh target/wasm32-wasi/release/calc_plugin_mul.wasm
-rwxr-xr-x. 1 root root 16K ... target/wasm32-wasi/release/calc_plugin_mul.wasm
```

### wasmファイルをAOTコンパイル

Lucetの最適化コンパイラでWebAssemblyをネイティブコードへコンパイルします。

```bash
$ (cd target/wasm32-wasi/release && \
   lucetc-wasi -o calc_plugin_add.so calc_plugin_add.wasm)

$ (cd target/wasm32-wasi/release && \
   lucetc-wasi -o calc_plugin_mul.so calc_plugin_mul.wasm)

$ ls -lh target/wasm32-wasi/release/*.so
-rwxr-xr-x. 1 root root 246K ... target/wasm32-wasi/release/calc_plugin_add.so
-rwxr-xr-x. 1 root root  63K ... target/wasm32-wasi/release/calc_plugin_mul.so
```

共有ライブラリ（soファイル）ができました。

### Dockerコンテナの停止

必須ではありませんがツールチェインの入ったDockerコンテナを停止します。

```bash
$ cd ..
$ pwd
lucet

$ ./devenv_stop.sh
Stopping container
lucet
Removing container
lucet
```

### 実行

それでは実行しましょう。

```bash
$ pwd
lucet/rust-wasm-plugin-example

$ (cd calc && cargo run --release)
Plugin: add
Calc: 1 + 2 = 3
Plugin: mul
Calc: 1 * 2 = 2
```

うまく動作しているようです。

## まとめ

- Lucetを使うとRustなどで書かれたアプリケーション上でWebAssemblyコードを実行できる
- プラグインとしてwasm形式を採用することで以下のようなメリットがある
    - 言語非依存：プラグインをさまざまな言語で開発できる
    - クロスプラットフォーム：プラグインの作者はwasmファイルを提供するだけでよい。プラグインのユーザはそれをさまざまな環境でネイティブコードにコンパイルして実行できる
    - セキュリティ：プラグインがサンドボックス内で実行されるので、他人が開発したプラグインでも、比較的安全に実行できる
- Lucetのエコシステム
    - 現状では`String`などの複合型の扱いが面倒
        - Lucetを対象としたライブラリの充実に期待（複数言語に対応したデータのシリアライズなど）
    - IDLはいまのところCのコード生成のみ（今度、試してみたい）
