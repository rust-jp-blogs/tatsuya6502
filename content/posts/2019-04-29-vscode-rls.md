---
title: "VS Code Rust (rls)拡張機能でcargo runタスクが見つからない場合の対処法"
# summary: ""
date: 2019-04-29T11:15:00+08:00
draft: no
isCJKLanguage: true
categories:
- Rust Tips
tags:
- 実践Rust入門
- VS Code
---

## まずは近況など

共著で出版した書籍『実践Rust入門』（以下 本書）ですが、2019年4月26日に電子版が発売されました。
また物理本（紙の本）の発売日は10連休明けの5月8日ですが、東京都内の一部の大手書店では先行販売しているようです。

電子版の販売サイトへは [本書の書籍情報ページ][book-info]（gihyo.jp）からリンクされています。
Gihyo Digital Publising（EPUB/PDFフォーマット）、Amazon Kindle、楽天koboから販売されています。

物理本の先行販売については、共著者である [κeenさんのツィート][presales] を参照してください。

[book-info]: http://gihyo.jp/book/2019/978-4-297-10559-4
[presales]: https://twitter.com/blackenedgold/status/1120501628858249217

Twitterなどでは、早くも本書を入手し、連休を活用してRustに入門・再入門されている方が観測できます。
内容については概ね好評のようで、私もホッと胸をなでおろすと同時に、Rustの普及に少しでも役立っているのかなと、共著者の一人として喜びを感じています。

## 本題：Rust (rls) プラグイン最新版の不具合について

前置きが長くなりました。
本題に入りましょう。

本書の2章では開発環境のセットアップと称して、Visual Studio Code（VS Code）のインストール手順や、ごく基本的な使いかたを紹介しています。

> 本節では執筆時点の最新版を使ってインストール方法を説明します。
>
> - VS Code 1.32.0
> - Rust (rls)拡張機能 0.5.3
>
> ...
>
> #### 2-3-5　基本的な使い方
> ...
>
> また`Ctrl`+`Shift`+`p`でコマンドパレットを開き、`run`と入力してから`Tasks: Run Tasks`を選びます。
> `cargo run`や`cargo test`などが表示されますので`cargo run`を選んでみてください。

_&mdash;実践Rust入門 ［言語仕様から開発手法まで］ より_

この章は私が執筆したのですが、試してくださった方からこんなツィートが、

> **puffin** 2019-04-28 9:18PM JST
>
> 実践Rust入門でVSCodeに拡張入れてcargo runしろって書いてあるのにコマンドが表示されないのでGitHubのissuesを調べてみたら一時的に削除したみたいなことが書かれてあってオイ

_&mdash;https://twitter.com/puffin555/status/1122475268533149697 より_

えっ、そんなことが？

Rust (rls)拡張機能のGitHub issueを調べてみたところ、たしかにそういうissueがありました。

- rust-lang/rls-vscode: [#557 VS tasks not present on install version 0.6.0][vscode-rls-557]

経緯をまとめると、以下のようになります。

[vscode-rls-557]: https://github.com/rust-lang/rls-vscode/issues/557

### 経緯

1. Rust (rls) 拡張機能 0.6.0で行ったリファクタリングの結果、タスク機能が動作しなくなってしまった
2. リリース後 issue #557 で報告があり、タスク機能が動作するように修正
  - しかし`cargo run`タスクだけは意図的に復活させなかった
  - この状態で0.6.1をリリース
  - `cargo run`タスクを復活させなかった理由は、同じことが「デバッグ」メニューの「デバッグの開始」や「デバッグなしで開始」でできるため
  - ただしこれらのメニューを使うには、ユーザーが`tasks.json`ファイルを作成する必要がある
3. Issue #557のコメントで複数のユーザーが`cargo run`タスクの復活を希望
4. 4月5日の [このコミット][vscode-rls-commit] で`cargo run`タスクを復活
  - 次回のポイントリリース（0.6.2）からは元通り使えるようになる予定

次のバージョン（0.6.2）では`cargo run`タスクが復活するはずですが、リリースの時期はまだわかりません。

[vscode-rls-commit]: https://github.com/rust-lang/rls-vscode/commit/ff119775bdd8760c94502036ec6af431e7f6fede

## 修正版がリリースされるまでのワークアラウンド

Rust (rls)拡張機能0.6.2がリリースされるまでの間は、古いバージョン（0.5.4）を使うのが良いでしょう。
以下の手順でダウングレードできます。

### 古いバージョン（0.5.4）へのダウングレード

以下の手順はVS Codeの最新版 1.33.1で動作確認済みです。

1. 拡張機能の自動更新をオフにする
   - `Ctrl/Command`+`Shift`+`p`でコマンドパレットを開き、`disable auto`と入力する
   - 表示された「Disable Auto Updating Extentions」を選択する
   - この操作で、インストールされている全ての拡張機能の自動更新が停止します
   - 現時点（VS Code 1.33.1）では個別の拡張機能について自動更新をオフにすることはできないようです
   - この設定でも更新のチェックは働きますので、機能拡張ビューで個別に手動更新をかけることはできます
2. Rust (rls)拡張機能を0.5.4にダウングレードする
   - 機能拡張ビューに切り替え（`Ctrl/Command`+`Shift`+`X`）、Rust (rls)を右クリックする
   - 「別のバージョンをインストール」を選び、`0.5.4`を選択する

**参考**：

- [VS Code - Extension auto-update][vscode-auto-ext-update]
- [VS Code 1.30 Release Notes - Extensions: Install previous versions][vscode-install-prev-ext]

[vscode-auto-ext-update]: https://code.visualstudio.com/docs/editor/extension-gallery#_extension-autoupdate
[vscode-install-prev-ext]: https://code.visualstudio.com/updates/v1_30#_install-previous-versions

### 修正版がリリースされたら

0.6.2がリリースされたら、拡張機能の自動更新をオンに戻すといいでしょう。

1. `Ctrl/Command`+`Shift`+`p`でコマンドパレットを開き、`enable auto`と入力する
2. 表示された「Enable Auto Updating Extentions」を選択する

これでRust (rls)拡張機能が最新版に更新されます。

## VS Code内でターミナルを使う方法もある

本書に書き忘れてしまったのですが、「ターミナル」メニューから「新しいターミナル」を選ぶと、VS Code内でターミナルを開けます。
VS Codeで開いているフォルダがカレントディレクトリとして設定されますので、その場で、`carge run`などを実行できます。

覚えておくと便利です。
