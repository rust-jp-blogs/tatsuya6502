# サイトのメンテナンス手順など

## リモートリポジトリ

- **公開用**
    - Public: GitHub [rust-jp-blogs/tatsuya6502][public-repo]
- **下書き用**
    - Draft: 非公開リポジトリ

[public-repo]: https://github.com/rust-jp-blogs/tatsuya6502

## ソフトウェアのセットアップ

[Hugo][hugo]をインストールしておく。

[hugo]: https://gohugo.io/

## ドラフト記事の作成

1. ローカルにトピックブランチを作成する
2. `git submodule update`
3. `hugo new posts/20YY-MM-my-post.md`
4. `hugo server -D -F`
5. [http://localhost:1313/tatsuya6502/](http://localhost:1313/tatsuya6502/) を開く
6. 以下の例を参考にfront matterを修正する

**Front Matterの例**

```
title: "記事のタイトル"
# summary: ""
date: 2019-04-29T11:15:00+08:00
draft: no
isCJKLanguage: true
categories:
- Rust Tips
tags:
- 実践Rust入門
```

## ドラフトの保存

トピックブランチ上でコミットし、draftレポジトリへpushする。

## 記事の公開準備（masterブランチのマージ）

1. Front matterを更新する
	- `date`を更新
	- `draft: `を`no`に変更
2. 必要ならファイル名（`20YY-MM`）などを変更する
3. `hugo server`で内容を確認する
4. トピックブランチへコミットする
5. 必要ならトピックブランチをrebaseしたり、過去のコミットをsquashしたりする
8. トピックブランチをdraftリポジトリへpushする
9. トピックブランチをpublicリポジトリへpushする
10. GitHubサイトで[リポジトリ][public-repo]を開き、プルリクエストを作成する。
    - マージ元：トピックブランチ
    - マージ先：masterブランチ
11. 問題がなければプルリクエストをマージする

## 記事の公開

1. publicリポジトリからmasterブランチをpullする
2. `hugo`コマンドを実行してHTMLを生成する（`doc/`フォルダ配下が更新される）
3. `doc/`フォルダ配下のファイルをmasterブランチへコミットする
4. masterブランチをdraftリポジトリへpushする
5. masterブランチをpublicリポジトリへpushする
6. [サイト][the-blog]に記事が公開されたことを確認する

[the-blog]: https://blog.rust-jp.rs/tatsuya6502/

## 記事公開後の作業

1. トピックブランチをpublicリポジトリから削除する
2. トピックブランチをdraftリポジトリから削除する

## Themeのカスタマイズ

リポジトリ： GitHub [rust-jp-blogs/hugo_theme_pickles][theme-repo]

[theme-repo]: https://github.com/rust-jp-blogs/hugo_theme_pickles
