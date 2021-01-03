---
title: "Rustの環境構築（Apple Silicon） 2020年12月版"
# summary: ""
date: 2020-12-22T19:10:00+08:00
draft: no
isCJKLanguage: true
categories:
- インストール
tags:
- Apple-silicon
---

これは [Rust Advent Calendar 2020][advcal] 15日目のエントリーです。
飛び入り参加です。

本記事ではApple siliconを搭載したMacにRustの開発環境をセットアップする方法を説明します。

[advcal]: https://qiita.com/advent-calendar/2020/rust


## 重要：プレビュー版のソフトウェアについて

本記事に掲載されているのは2020年12月22日時点の情報です。
Apple siliconネイティブなソフトウェアを中心にインストール方法を説明していますが、執筆時点ではそれらのほとんどがプレビュー版となっています。
みなさんがこの記事を読まれるときには正式版がリリースされているかもしれません。
他の情報源もチェックされることをお勧めします。

また、プレビュー版のソフトウェアは品質が保証されていません。
リリース前の正式なテストが行われておらず、重大な不具合が出る可能性もあります。
大切なデータは事前にバックアップしておくなど、十分注意してお使いください。


## ソフトウェアのバージョンについて

本記事で扱うソフトウェアとバージョンは以下のとおりです。
なお、macOSのバージョンはBig Sur 11.1です。

**2020年12月22日の状況**

| ソフトウェア | バージョン | アーキテクチャー | 備考 |
|:--|:--|:--|:--|
| rustup | 1.23.1 | Apple | 正式版 |
| Rustツールチェーン | 1.49.0-beta.4 | Apple | ベータ版。日本時間の2021年元日に安定版（Tier 2）がリリースされる |
| Homebrew（AArch64） | 2.6.2 | Apple | プレビュー版 |
| Homebrew（x86_64） | 2.6.2 | Intel | Intel Mac向け正式版 |
| sccache | 0.2.14 | Apple | 正式版 |
| Visual Studio Code | 1.53.0-insider | Apple | プレビュー版 |
| Rust Analyzer | 2020-12-14 | Apple | バイナリーがまだ配布されていないので、ソースコードからビルドする |
| Docker Desktop | Preview 7 | Apple | プレビュー版。正式版のリリースは2021年第1四半期の予定 |

アーキテクチャーの欄：

- Apple: Apple siliconネイティブ（AArch64）のソフトウェア
- Intel: Intel Mac（x86_84）向けのソフトウェア。M1 Mac上ではRosetta 2トランスレーターを通して実行する

Homebrew（x86_64）は、まだApple siliconにネイティブ対応していないパッケージのインストールに使用します。
（この記事ではx86_64版のパッケージは使いません）


## rustupとRustツールチェーンのインストール

- rustupはネイティブ対応済み
- ツールチェインは1.49.0からネイティブ対応（2021年元日にリリースされる）
    - 大晦日までは1.49.0-beta（ベータ版）を使用
    - 2021年元日になったら1.49.0のstable（安定版）に変更する
- aarch64-apple-darwinはTier 2であることに注意

rustupは2020年11月27日にリリース済みの1.23.0からネイティブ対応しています。

Rustコンパイラなどを含むRustツールチェーンは1.49.0からネイティブ対応します。
1.49.0は現在はbetaで、2021年元日（日本時間）にstableになります。

ツールチェーンのターゲットトリプルは「aarch64-apple-darwin」です。
筆者が試した範囲では問題なく使えていますが、RustプロジェクトのCI環境にはまだM1 Macがなく、自動テストケースが実行されていないことに注意してください。
そのためこのターゲットは当分の間は [Tier 2][tier2] プラットフォームになります。

[tier2]: https://doc.rust-lang.org/nightly/rustc/platform-support.html#tier-2

インストール手順を説明します。
ウェブブラウザーで https://rustup.rs/ を開き、表示されたコマンドをターミナルから実行します。

```bash
% curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

インストールオプションが表示されたら、（2020年12月中は）デフォルトのツールチェーンをbetaに変更します。

```bash
## □のところでReturnキーを押す

Current installation options:

   default host triple: aarch64-apple-darwin
     default toolchain: stable (default)
               profile: default
  modify PATH variable: yes

1) Proceed with installation (default)
2) Customize installation
3) Cancel installation
> 2□

I'm going to ask you the value of each of these installation options.
You may simply press the Enter key to leave unchanged.

Default host triple?
□

Default toolchain? (stable/beta/nightly/none)
beta□

Profile (which tools and data to install)? (minimal/default/complete)
□

Modify PATH variable? (y/n)
□

Current installation options:

   default host triple: aarch64-apple-darwin
     default toolchain: beta
               profile: default
  modify PATH variable: yes

1) Proceed with installation (default)
2) Customize installation
3) Cancel installation
```

betaに変更できたらReturnキーを押してツールチェーンをインストールします。

```bash
info: profile set to 'default'
info: setting default host triple to aarch64-apple-darwin
info: syncing channel updates for 'beta-aarch64-apple-darwin'
...

  beta-aarch64-apple-darwin installed - rustc 1.49.0-beta.4 (877c7cbe1 2020-12-10)

Rust is installed now. Great!
```

現在のシェルセッションでツールチェーンを使えるよう、以下のコマンドを実行します。

```bash
$ source $HOME/.cargo/env
```

バージョン情報などを表示してみましょう。

```bash
## ツールチェーンの情報を表示する
$ rustup show
Default host: aarch64-apple-darwin
rustup home:  /Users/tatsuya/.rustup

beta-aarch64-apple-darwin (default)
rustc 1.49.0-beta.4 (877c7cbe1 2020-12-10)

## rustupのバージョンを表示する
$ rustup -V 
rustup 1.23.1 (3df2264a9 2020-11-30)
info: This is the version for the rustup toolchain manager, not the rustc compiler.
info: The currently active `rustc` version is `rustc 1.49.0-beta.4 (877c7cbe1 2020-12-10)`

## Rustコンパイラのバージョンを表示する
$ rustc -Vv
rustc 1.49.0-beta.4 (877c7cbe1 2020-12-10)
binary: rustc
commit-hash: 877c7cbe142a373f93d38a23741dcc3a0a17a2af
commit-date: 2020-12-10
host: aarch64-apple-darwin
release: 1.49.0-beta.4
```


### Appleコマンドラインツールのインストール

Rustコンパイラが使用するリンカーをインストールしましょう。
リンカーはAppleのコマンドラインツールに含まれています。
ターミナルから以下のコマンドを実行し、画面の表示に従ってインストールしてください。

```bash
$ xcode-select --install
```


### 2021年の元日になったら

2021年の元日に1.49.0のstable版（安定版）がリリースされます。
以下のようにしてデフォルトツールチェーンをstableに変更してください。
これにより、1.49.0 stableが自動的にインストールされます。

```bash
$ rustup default stable
... （1.49.0 stableがインストールされる） ...

$ rustup show
```


## Homebrewのインストール

- Homebrewのネイティブ版は現在プレビュー版
- ネイティブ版ではビルドできないパッケージが多い
- ネイティブ版が正式リリースされるまでは、x86_64版のHomebrewもインストールし、ネイティブ対応していないパッケージはこちらでインストールする

HomebrewはmacOS（とLinux）向けのパッケージマネージャーです。
ネイティブ版は現在のところはプレビュー版の扱いで、ビルドできないパッケージも多いです。
そのためx86_64版のHomebrewもインストールしておきます。

ネイティブ版のHomebrewでビルドできないパッケージには、たとえば、Rustで書かれた`exa`、`procs`、`ripgrep`、`starship`などのコマンドラインツールがあります。
これらはRust 1.49.0のstableがリリースされるまではHomebrewでビルドできません。

筆者は現在のところRustで書かれたツールはRust 1.49.0-betaの`cargo install`を使ってビルドとインストールをしています。
また、それ以外でビルドできないパッケージについてはx86_64版のHomebrewでインストールしています。


### Homebrewのインストールディレクトリー

Homebrewの [ドキュメント][homebrew-doc] ではHomebrewを以下のディレクトリーにインストールすることを推奨しています。

| 種類 | アーキテクチャー | インストールディレクトリー |
|:--|:--|:--|
| ネイティブ版 | Apple | `/opt/homebrew` |
| x86_64版 | Intel | `/usr/local` |

[homebrew-doc]: https://docs.brew.sh/Installation#alternative-installs

これに従ってインストールしましょう。


### Homebrew（Apple siliconネイティブ版）

ネイティブ版は以下のコマンドでインストールします。

```bash
$ sudo mkdir /opt/homebrew
$ sudo chown ${USER}:staff /opt/homebrew
$ curl -L https://github.com/Homebrew/brew/tarball/master | tar xz --strip 1 -C /opt/homebrew
```

`PATH`環境変数を設定して、`brew`コマンドとHomebrewでインストールしたコマンド（例：`pkg-config`）が実行できるようにします。
zshなら以下のようにします。

```bash
$ echo 'export PATH=/opt/homebrew/bin:$PATH' >> ~/.zshrc
$ source ~/.zshrc
```

こうすることで、ネイティブ版によってインストールされたコマンドの方が、x86_64版によって`/usr/local/bin`へインストールされたコマンドよりも優先されるようになります。

<!--
zsh order: https://stackoverflow.com/a/27742679

```
/etc/zshenv    # Read for every shell
~/.zshenv      # Read for every shell except ones started with -f
/etc/zprofile  # Global config for login shells, read before zshrc
~/.zprofile    # User config for login shells
/etc/zshrc     # Global config for interactive shells
~/.zshrc       # User config for interactive shells
/etc/zlogin    # Global config for login shells, read after zshrc
~/.zlogin      # User config for login shells
~/.zlogout     # User config for login shells, read upon logout
/etc/zlogout   # Global config for login shells, read after user logout file
```
-->

最後に`brew update`を実行します。

```bash
$ brew update
```


### Homebrew（x86_64版）

x86_64版は以下のコマンドでインストールします。

```bash
## 念のため、PATHからネイティブ版のHomebrewを外す
$ export PATH=/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin

## x86_64版のbashでインストールスクリプトを実行する
$ arch -x86_64 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

なお、MacにRosetta 2がインストールされていないときは、以下のようなエラーがでます。

```bash
arch: posix_spawnp: /bin/bash: Bad CPU type in executable
```

その場合はRosetta 2をインストールしてから、再度、Homebrewのインストールスクリプトを実行してください。

```bash
$ softwareupdate --install-rosetta
I have read and agree to the terms of the software license agreement. A list of Apple SLAs may be found here: http://www.apple.com/legal/sla/
Type A and press return to agree:
A□
...

Install of Rosetta 2 finished successfully
```


### brewコマンドの実行方法

**Apple siliconネイティブ版**

ネイティブ版は単に「`brew サブコマンド`」とすれば実行できます。

```bash
$ brew config 
HOMEBREW_VERSION: 2.6.2
HOMEBREW_PREFIX: /opt/homebrew
CPU: octa-core 64-bit arm_firestorm_icestorm
macOS: 11.1-arm64
Rosetta 2: false
...
```

**x86_64版**

x86_64版はRosetta 2を通して実行するため、以下のように`arch`コマンドを併用します。
またネイティブ版が優先されるよう`PATH`を設定したので、x86_64版を実行するときはフルパスで指定します。

```bash
$ arch -x86_64 /usr/local/bin/brew config
HOMEBREW_VERSION: 2.6.2
HOMEBREW_PREFIX: /usr/local
CPU: octa-core 64-bit westmere
macOS: 11.1-x86_64
Rosetta 2: true
...
```

なお、フルパス指定を忘れると、x86_64版でありながら`HOMEBREW_PREFIX`はネイティブ版のものになってしまいます。
忘れないように気をつけてください。

```bash
## フルパス指定を忘れると、HOMEBREW_PREFIXが間違ったものになってしまう
$ arch -x86_64 brew config 
HOMEBREW_VERSION: 2.7.0
HOMEBREW_PREFIX: /opt/homebrew   ## ← /usr/localが正しい
CPU: octa-core 64-bit westmere
macOS: 11.1-x86_64
Rosetta 2: true
```


### パッケージがネイティブ対応しているか調べる（2021年1月3日 追記）

以下のコマンドで、ネイティブのバイナリー（bottle）が用意されているパッケージ 一覧を表示できます。
2021年1月3日の時点では約3,600パッケージについてbottleが用意されていました。

```bash
$ brew update
$ cd $(brew --repo homebrew/core)/Formula/
$ rg -l ':arm64_big_sur' | xargs -I{} basename {} .rb | sort
a2ps
a52dec
aacgain
aalib
...
zyre
zzuf

$ rg -l ':arm64_big_sur' | wc -l
    3602

## なお、`rg`コマンドは`brew install ripgrep` でインストールできる
```


## sccache

sccacheはRust製のコンパイルキャッシュです。
複数のRustプロジェクトから同じバージョンのクレートを使っているときなどに、2回目以降のビルド時間を短縮できます。
Rustで開発するうえで必須ではありませんが、筆者は1年半ほど前から愛用しています。
sccacheについて詳しくは [こちらの記事][qiita-sccache] をご覧ください。

[qiita-sccache]: https://qiita.com/tatsuya6502/items/76b28a6786a1ddc9d479

sccacheは2020年12月21日にリリース済みの0.2.14からネイティブ対応しています。
Homebrewにパッケージがありますが、Rust 1.49.0がまだstableでないためHomebrewのネイティブ版ではインストールできません。

**2021年1月3日 追記**：sccacheは現在はHomebrewでバイナリーパッケージからインストールできるようになっています。

```bash
## 2021年1月現在はbrewでバイナリーパッケージからインストールできる
$ brew install sccache
```

（**追記おわり**）

Homebrewのネイティブ版でインストールできるようになるまでの間は、`cargo install`コマンドでビルドしてインストールします。

```bash
## 2020年12月中はcargoでビルド、インストールする必要があった
$ cargo install sccache
```

Rustプロジェクトのビルド時に環境変数`RUSTC_WRAPPER`を`sccache`にセットするとsccacheが使われるようになります。

```bash
$ export RUSTC_WRAPPER=$(which sccache)
```


## 参考：Cargo Editをビルドしてみる

RustツールチェーンとHomebrewがインストールできたので、試しに何かビルドしてみましょう。
筆者は [cargo edit][crate-cargo-edit] を選びました。

[crate-cargo-edit]: https://crates.io/crates/cargo-edit

cargo editはcrates.ioのAPIにアクセスするためにHTTPS通信を行います。
ネイティブライブラリーのOpenSSLが必要になりますので、Homebrewのネイティブ版で`openssl@1.1`と`pkg-config`パッケージをインストールします。

```bash
$ brew install openssl@1.1 pkg-config
```

`cargo install`コマンドでcargo editをビルド、インストールします。

```bash
$ cargo install cargo-edit
...
  Installing /Users/tatsuya/.cargo/bin/cargo-add
  Installing /Users/tatsuya/.cargo/bin/cargo-rm
  Installing /Users/tatsuya/.cargo/bin/cargo-upgrade
   Installed package `cargo-edit v0.7.0` (executables `cargo-add`, `cargo-rm`, `cargo-upgrade`)
```


## Visual Studio Code

- Visual Studio Codeは現在はプレビュー版

Visual Studio Code（VS Code）はオープンソースかつクロスプラットフォームのコードエディターです。
ネイティブ版は現時点ではプレビュー版です。

ネイティブ版は「Insider Builds」のページからダウンロードできます。

- https://code.visualstudio.com/insiders/

「.zip」の横にある「ARM64」ボタンをクリックし、ダウンロードしてください。
（大きなDownloadのボタンをクリックすると、x86_64版がダウンロードされてしまいます）

ダウンロードできたらzipファイルを解凍します。
すると「Visual Studio Code - Insiders」というアプリになりますので、Finderで選択して`Command` + `I`で情報を表示します。
ファイルの種類が「アプリケーション（Appleシリコン）」になっていることを確認してください。OKならアプリをアプリケーションフォルダー（Applicationsフォルダー）へ移動します。


## Rust Analyzerのビルドとインストール

**2021年1月3日 追記**：2020年12月28日からRust AnalyzerのApple siliconネイティブバイナリーの配布が始まりました。
そのため、ここで解説しているソースコードからのビルドは**不要**です。
VS Codeならmatklad.rust-analyzer拡張機能をインストールすると、自動的にネイティブ版がダウンロードされます。（**追記おわり**）

- Rust Analyzerは ~~まだバイナリー版のダウンロードはできない。ソースコードからビルドする~~ → 12月28日からバイナリー版が提供されるようになった
- Rust AnalyzerのVS CodeプラグインをビルドするにはNode.jsとnpmが必要。Homebrew（AArch64）でインストールできる

Rust AnalyzerはRustのLSPサーバー実装のひとつです。
LSP（Language Server Protocol）はマイクロソフトが提唱するオープンなプロトコルで、これを使うとエディターやIDEがコンパイラなどの言語処理系と連携できるようになります。
Rust AnalyzerにはVS Codeの拡張機能が用意されており、VS Codeからすぐに使えます。
またviやEmacsのような他のエディターからもLSPプラグインを通して使えます。

Rust Analyzerは毎週バイナリー版のリリースが作られますが、現時点ではApple silicon向けのバイナリーはありません。
そのため、ソースコードを取得して自分でビルドする必要があります。
と、言っても特に難しいところはないと思います。

なお、Rust Analyzer本体はRustツールチェーンだけでビルドできます。
VS Code拡張機能もビルドする場合はNode.jsとnpmも必要です。


### Rust Analyzerのソースコード取得

適当なディレクトリーに移動してから、Rust Analyzerのgitリポジトリーをクローンします。

```bash
$ cd 適当なディレクトリー
$ git clone https://github.com/rust-analyzer/rust-analyzer.git
$ cd rust-analyzer
```


### Node.jsとnpmのインストール

Rust AnalyzerのVS Code拡張機能をビルドする場合は、ビルドに必要なNode.jsとnpmをインストールします。
VS Code拡張機能が不要な場合はこのステップは飛ばして構いません。

```bash
## ネイティブ版のHomebrewでnodeパッケージをインストールする
## Node.jsとnpmがインストールされる
$ brew install node

$ node -v
v15.4.0

$ npm -v 
7.0.15
```


### 最新リリースのタグでチェックアウト

Rust Analyzerは開発が活発に続けられており、毎週リリースされています。
`git tag`コマンドで最新のリリースのタグを調べ、チェックアウトします。

```bash
$ git tag | sort | tail -6
2020-11-30
2020-12-07
2020-12-14
2020-12-21
guide-2019-01
nightly

$ git checkout 2020-12-21
...
Note: switching to '2020-12-21'.

You are in 'detached HEAD' state. ...
...

HEAD is now at ...
```


### シェルからVS CodeバイナリーへのPATHを設定

VS Code拡張機能をビルドする場合はビルドスクリプトからVS Codeのバイナリー（`code`コマンド）が実行できるように設定します。
VS Code拡張機能が不要な場合はこのステップは飛ばして構いません。

```bash
## 筆者はビルドのたびにPATHを設定するのが面倒だったので以下の内容のファイルを作った
$ cat vs-code-env
export PATH="$PATH:/Applications/Visual Studio Code - Insiders.app/Contents/Resources/app/bin"

$ source ./vs-code-env 

## codeコマンドが使えることを確認する

$ which code
/Applications/Visual Studio Code - Insiders.app/Contents/Resources/app/bin/code

$ code -v 
1.53.0-insider
c927a8015b9e26bd454d6e293bb0384aa1975d06
arm64
```


### Rust Analyzerのビルドとインストール

Rust Analyzerをビルドしましょう。

```bash
# Rust AnalyzerとVS Code拡張機能をインストールする
$ cargo xtask install

# または
# Rust AnalyzerのLSPサーバーだけをインストールする
$ cargo xtask install --server
```

**実行例**

```bash
$ cargo xtask install
...

   Replacing /Users/tatsuya/.cargo/bin/rust-analyzer
...

Installing extensions...
Extension 'rust-analyzer.vsix' was successfully installed.
```


### 新しいバージョンがリリースされたら

Rust Analyzerの新しいバージョンがリリースされたら、以下のようにして最新のコードを取得します。

```bash
## ローカルのgitリポジトリーを初期状態に戻す
$ git restore .
$ git checkout master
Previous HEAD position was ...
Switched to branch 'master'
Your branch is up to date with 'origin/master'.

## リモートリポジトリー（GitHub）から最新のコードを取得する
$ git fetch origin
$ git pull origin master
```

あとは「[最新リリースのタグでチェックアウト](#最新リリースのタグでチェックアウト)」のステップから再び実行すればOKです。


## Docker Desktop

- Docker Desktopは現時点ではプレビュー版
- Linux ARM 64（AArch64）版のイメージをネイティブ実行できる
- Docker DesktopにはQEMUが同梱されているので、x86_64など他のアーキテクチャのLinuxイメージも **一応** 実行可能

Docker Desktop for Macは2020年12月16日からプレビュー版が一般公開されています。
Linux ARM 64（AArch64）版のイメージをネイティブ実行できますが、それだけでなく、x86_64など他のアーキテクチャのLinuxイメージもQEMUを通して実行できます。

ウェブブラウザーで以下のURLを開きます。

- https://www.docker.com/blog/download-and-try-the-tech-preview-of-docker-desktop-for-m1/

「Download it here!」と書かれたリンクをクリックすると、プレビュー版のダウンロードが始まります。

ダウンロードできたら、ディスクイメージファイルをダブルクリックします。
仮想ディスクがマウントされますので、中にあるDockerアプリをアプリケーションフォルダーへコピーします。

Dockerアプリをダブルクリックで起動すれば準備完了です。


### Linux ARM 64版のDockerイメージを使う

Apple siliconのアーキテクチャーはAArch64なので、Docker Desktop for MacでDockerホストとして動く仮想マシンのLinuxカーネルもAArch64版になります。`docker pull`したときに「ARM 64」版のDockerイメージが用意されていれば、そのイメージがダウンロードされます。

```bash
$ docker pull rust

$ docker run -it --rm rust       

root@cb45a4cbe687:/# uname -sm
Linux aarch64

root@cb45a4cbe687:/# rustc -Vv
rustc 1.48.0 (7eac88abb 2020-11-16)
binary: rustc
commit-hash: 7eac88abb2e57e752f3302f02be5f3ce3d7adfb4
commit-date: 2020-11-16
host: aarch64-unknown-linux-gnu
release: 1.48.0
LLVM version: 11.0
```

`rust`のようなDockerの公式イメージなら基本的にARM 64版が用意されています。

ARM 64版のコンテナーでビルドしたRustバイナリーは、64ビット版のRaspberry PiやAWS EC2のARMインスタンスでも実行できます。
Linux x86_64上でこれらの環境向けにクロスコンパイルするよりも手軽でいいかもしれません。


### Linux x86_64版のDockerイメージを使う

`docker pull`などで指定したイメージにARM 64版がないときはx86_64版などDocker Desktopが対応している非ネイティブアーキテクチャーのイメージがダウンロードされます。
また、`docker pull`や`docker run`で`--platform`を指定することもできます。
 
```bash
## イメージのプラットフォームを指定する（amd64はx86_64と同じ意味）
$ docker pull --platform linux/amd64 rust
```

このようなイメージを使うときでも、Dockerホストとして動く仮想マシンのLinuxカーネルはAArch64版のままです。
当然、x86_64などのバイナリーは直接実行できませんので、仮想マシン上のLinuxカーネルのbin_fmtという機能によってQEMUのユーザースペースエミュレーターが起動され、それを通して実行されます。
macOSにおけるRosetta 2と似た技術ですが、Rosetta 2は事前にネイティブコードに変換してから実行するのに対し、QEMUではx86_64コードを ~~逐次解釈しながら実行するインタープリター形式のようなので、実行速度は遅くなります。~~

**2021年1月3日 訂正**：[こちらの記事][qemu-translaton] によると、QEMUのエミュレーターはインタープリター形式ではなく、ネイティブコードへの変換を行っているようです。（**追記おわり**）


**Linux x86_64版のDockerイメージからコンテナを実行する**

```bash
$ docker run -it --rm --platform linux/amd64 rust

## 実はカーネルはAArch64版だが、見かけ上はx86_64版になる
root@cd6b4eca1faf:/# uname -sm
Linux x86_64

root@cd6b4eca1faf:/# rustc -Vv
rustc 1.48.0 (7eac88abb 2020-11-16)
binary: rustc
commit-hash: 7eac88abb2e57e752f3302f02be5f3ce3d7adfb4
commit-date: 2020-11-16
host: x86_64-unknown-linux-gnu
release: 1.48.0
LLVM version: 11.0
```

なお、OEMUユーザースペースエミュレーターにはサポートしていない機械語命令やLinuxシステムコールなどがあるため、うまく動かないバイナリーも結構あるようです。
筆者が試した範囲では、x86_64版のRustコンパイラが内部エラー（Internal Compiler Error、ICE）で落ちてしまったり、QEMUプロセスがsegmentation faultしてしまうことが何度もありました。
実用的に使うのはまだ難しいかもしれません。

なお、ネイティブイメージだけを使用したいときは、`--platform`に`linux/arm64/v8`を指定します。

```bash
## Apple siliconネイティブのイメージに限定する
$ docker pull --platform linux/arm64/v8 rust
```

[qemu-translaton]: https://msyksphinz.hatenablog.com/entry/2020/12/29/040000#QEMU%E3%81%8C%E9%AB%98%E9%80%9F%E3%81%AA%E7%90%86%E7%94%B1TCG-Binary-Translation


## まとめ

Apple siliconを搭載したMacにRustの開発環境をセットアップする方法を説明しました。
