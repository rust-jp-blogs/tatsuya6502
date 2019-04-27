---
title: "Comming Soon：Rustバイナリ入りの極小Dockerイメージを作る"
# summary: ""
date: 2019-04-27T12:17:54+08:00
draft: false
isCJKLanguage: true
categories:
- Rust Tips
tags:
- Docker
- Linux-musl
---

書籍『[実践Rust入門][practical-rust-primer]』の2-5-4項では、クロスコンパイルを応用した極小のDockerイメージを紹介しています。
書籍内で予告したとおり、その具体的な作成手順を当ブログで紹介する予定です。**数日中に公開できると思いますので、もう少しお待ちください**。

[practical-rust-primer]: http://gihyo.jp/book/2019/978-4-297-10559-4


**実践Rust入門の該当部分**

> ```console
> $ docker images hello-sqlite
> REPOSITORY    TAG     IMAGE ID      CREATED        SIZE
> hello-sqlite  latest  0f60b9e23a91  5 minutes ago  1.95MB
> alpine        latest  196d12cf6ab1  2 months ago   4.41MB
> ubuntu        18.04   ea4c82dcd15a  4 weeks ago    85.8MB
> ```
> </br>
> 　hello-sqliteはSQLiteサーバを組み込んだRustサンプルプログラムを実行するためのコンテナです。
> `x86_64-unknown-linux-musl`ターゲット向けにビルドしたバイナリを、`scratch`という空のDockerイメージに入れました。
> このバイナリにはSQLiteはもちろん、全てのRustバイナリが依存しているlibcなども埋め込まれています。
> シェルなどのLinuxコマンドがなくても実行できますので、そのサイズは1.95MBとなっており、Dockerイメージとしては極端に小さい部類に入るAlpine Linuxのイメージ（4.41MB）よりも小さくなっています。
>
> 　誌面の都合から具体的な作成手順は省略します。
> 筆者らが管理するWebサイトにて他のターゲットと共に紹介していますので、興味があればそちらをご覧になってください。
