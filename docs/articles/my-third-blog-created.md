---
title: ブログを新調した二回目
description: 以前作った個人ブログを新調しました。　Material for MkDocs　を利用しています。
date: 2023-06-26
createdAt: 2023-06-26
updatedAt: 2022-06-26
tags:
  - Frontend
---

# ブログを新調した二回目
大してブログを書いてないのにまたブログを新調してしまいました。前回、前々回は一応自分でコードを書く部分がありましたが、今回は既存のフレームワークを利用しただけです。

## Material for MkDocs
[MkDocs](https://www.mkdocs.org/) は、 Python で書かれたサイト生成フレームワークです。 [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) は、 MkDocs のプラグインです。
これのいいところは、日本語検索に簡単に対応できることです。yaml ファイルに適当に行を追加するだけで結構な精度で日本語検索してくれます。
しかも、表示の仕方が結構いい感じで、スニペットを表示してくれます。
たとえば、honkit だと、どのページにあるかは表示してくれるのですが、前後の文脈は表示してくれません。rust-book は日本語検索に対応していないため、分かち書きがうまくいかず、うまく検索にヒットしません。
その点、MkDocs はパッと見る感じきちんと検索できました。いいですね。

テーマカラーの変更や、タグ機能なども設定ファイルをいじるだけで簡単に利用できました。フレームワークのちからってすごい。

## blogging ライブラリ
Material for MkDocs には有償版があります。そちらでは、blog 機能があります。blog 機能を利用すると、特定ディレクトリにファイルを配置するだけでいい感じにナビゲーションに登録してくれます。
そこで代替になるのが、[Mkdocs Blogging Plugin](https://liang2kl.github.io/mkdocs-blogging-plugin/)　です。こちらは、OSS で開発されていて、無料で利用できます。

## Python with Rye
Python で書かれているのがちょっと…と思っていました。環境構築でつらい思いをした経験しかないからです。特に Windows では...
しかし、最近 Rye というツールが話題になっていたので使ってみました。Windows10, 11 で試したのですが、どちらも問題なく環境構築できました。
公式のインストールガイドの通りセットアップして、
```bash
$ rye pin 3.11
$ rye install mkdocs
$ rye add mkdocs-material mkdocs-blogging-plugin
$ rye sync -f
$ .venv\Scripts\activate
$ mkdocs serve
```

みたいな感じで起動できました。

## 欲しい機能
zenn とか qiita, はてぶみたいにリンク先の記事のタイトルとか OGP を取得していい感じに表示する機能が欲しいです。前のブログでその機能は実装していたのですが。
最新の情報に追従する必要はないので、ビルド時にデータを埋め込むので十分ですし、同じ発想でプラグインを作っている人とかいるかな、と思って探してみましたが、見当たりませんでした。
なので、次拡張するとしたら、そういうプラグインを作ることから始めようと思います。