---
title: マークダウンをVueコンポーネントで描画する
description: マークダウンのエコシステムである unified を使って、マークダウンを Vue コンポーネントで描画しました。そして、Vueコンポーネントを使う恩恵を活かして、ブログサービスなどでよく見る、OGP 情報を表示するリンクカードを作成しました。
date: 2022-11-15
createdAt: 2022-11-15
updatedAt: 2022-11-15
tags:
  - Frontend
  - Work
---

# マークダウンをVueコンポーネントで描画する
## unified

https://unifiedjs.com/

unified という単語は、二つの意味があります。
一つ目は、AST(Abstract Syntax Tree) に関する何らかの機能を提供するプラグインのエコシステムのことです。AST とは、文章を木構造にしたデータのことです。プログラミング言語の実装などでよく見かけます。
二つ目は、エコシステムのプラグインを使うためのコアパッケージの名前です。各種プラグインを登録して、統一的なインターフェースで操作できるようにしたパッケージです。

unified のエコシステムには、remark や micromark などが含まれています。unified 以外で、markdown を HTML に変換するときに使われるライブラリとしては、markdown-it や、marked などがあります。unified 側からこれらを比較しているので、参考にしてください。
https://github.com/micromark/micromark#comparison

### v-html を使う方法

今回の目的は、Vue のコンポーネントでマークダウンを描画することです。
Vue でマークダウンを描画する方法として、ネットでよく紹介されているものは、マークダウンを単純な HTML に変換してシリアライズ（文字列化）し、それを`v-html`に渡して描画するというものです。
この方法は簡単に実施可能ですが、`v-html`に渡せるものは単純な HTML のみなので、Vue コンポーネントのメリットをうけることができません。メリットというのは、たとえばインタラクティブで複雑な UI が作りやすいなどです。

### 今回使う plugin

Vue のコンポーネントを使うためには、マークダウンを AST にしたあとに、AST のレンダリングを Vue に委譲する必要があります。
unified のプラグインは、それぞれ責務をもっています。たとえば、マークダウンをマークダウン形式の AST に変換するプラグインや、HTML の AST を実際の HTML 文字列に変換するプラグインなどです。
今回は、マークダウンを MDAST (MarkDown 形式の AST) に変換するプラグイン`mdast-util-from-markdown`を使います。
https://github.com/syntax-tree/mdast-util-from-markdown

`mdast-util-from-markdown`は、unified のコアパッケージの一つで、`remark-parse`などで使われています。

`mdast-util-from-markdown`を使って取得する MDAST は、以下のようなオブジェクトです。

```ts
// source is `## Hello, *World*!`
{
  type: 'root',
  children: [
    {
      type: 'heading',
      depth: 2,
      children: [
        {type: 'text', value: 'Hello, '},
        {type: 'emphasis', children: [{type: 'text', value: 'World'}]},
        {type: 'text', value: '!'}
      ]
    }
  ]
}
```

## Vue Rendering Function

では、mdast を Vue コンポーネントでレンダリングするにはどうすればいいのでしょうか。

Vue には、Render Function という機能があります。
https://vuejs.org/guide/extras/render-function.html

Vue を使うときは、template を使うことが多いと思います。しかし、template 機能ではなく、JavaScript から直接コンポーネントを操作したい場面もあります。たとえば、今回のように AST からコンポーネントツリーを生成したい場合です。このとき使えるのが、Render Function です。

### Render Function の使い方

Render Function は、`h`という関数名で提供され、以下のように使えます。

```ts
import { h } from "vue";
import YourComponent from "path/to/component";

export const vnode = () => {
  return h(
    YourComponent, // type
    { props1: "foo", class: "bar" }, // props
    [
      /* children */
    ]
  );
};
```

`vnode`はコンポーネントとして、tepmlate 内で普通に使うことができます。

## 実際の実装

実際の実装では、MDAST の`type`に応じてコンポーネントを描画する再帰関数を使います。たとえば、`type`が`link`の場合は、`a`タグに対応するコンポーネントを描画します。そして、`children`に対して、おなじように関数を適用します。

具体的には、以下のような関数になります。

```ts
import { h, VNode } from "vue";
import { Parent, Root, Content } from "mdast";

const toVnode = async (root: Root): Promise<VNode> => {
  // 再帰関数を定義
  const childNodeHandler = (
    node: Content,
    _parentNode: Parent,
    index: number
  ): VNode | string => {
    const { type } = node;

    // node の type を判定し、それにおうじてコンポーネントを描画する
    if (type === "heading") {
      return h(
        `h${node.depth}`,
        node.children.map((childNode) =>
          childNodeHandler(childNode, node, index)
        )
      );
    }
    if (type === "text") {
      return node.value;
    }

    // ほかのtypeについても記述する。
    return "";
  };

  return h(
    "div",
    { id: "root" },
    markdownNode.map((childNode, index) => {
      return childNodeHandler(childNode, root, index);
    })
  );
};
```

### AST と再帰関数の操作について（感想）

AST の操作は慣れていないとよくわからないと思いますが、セキュリティキャンプで HTML パーサーを眺めていたため、スラっと書けました。進研ゼミでやったところだ！現象ですね。
https://blog.sasakiy84.net/articles/seccamp2022-report

また、↓ の記事で、パーサー自作の素振りをしていたことも活きていたと思います。
https://www.m3tech.blog/entry/2021/08/23/124000

### まとめ

以上で、マークダウンを Vue コンポーネント経由でレンダリングすることができました。Vue コンポーネントにしたことで、以下のような処理を書きなれたかたちでかくことができます。

- heading をクリックするとリンクがコピーされる
- リンク先の title などを非同期で取得して表示する
- 特定言語のコードを実行ボタンをつける

また、Vue コンポーネント関係なく、unified のエコシステムを使えば、以下のようなこともできるでしょう。

- 目次(TOC / Table of Contents)の生成
- コードのハイライト機能
- front matter を使ったメタ情報の管理

## リンクカードの作成

Vue コンポーネントを使った機能の一例として、リンクカードを実装してみます。
リンクカードとは、リンク先のページタイトルや説明、サムネイル画像などを取得して表示することで、読み手がリンク先に飛ぶかどうか判断しやすくするための機能とします。

リンクカードが描画されるまでの流れを大まかに整理すると、以下のようになります。

1. リンク先の title, description, og:image などを取得する
2. 1 で取得した情報をもとに描画する

ここで、ブラウザからリクエストを送ると、同一オリジンポリシーにより中身に JavaScript からアクセスできないことに気を付ける必要があります。

これに対して、単純に考えられる解決策は、リンク先の URL を投げるとそのメタ情報を返してくれる API を作成するというものです。AWS Lambda 等で簡単に API を用意することも可能です。
しかし、今回は記事更新頻度が多くなく、すべて静的にホスティングされていることため、記事中に現れるすべての URL が事前にわかっていることを活かして、リンク先のメタ情報が格納された JSON ファイルを記事ページごとに生成することにしました。

大まかな処理のながれとしては、以下のようになります。

1. 記事を作成したら、MDAST 形式にして、URL 一覧を取得する
2. chrome の headless mode で URL にアクセスし、メタ情報を取得する
3. JSON 形式で記事ごとに保存し、フロントで取得できるようにする
4. フロントでは、リンクカードのコンポーネントを作り、メタ情報を描画する

いくつか解説を加えます。

### chrome の headless mode

SPA は、SSR や SSG をしていない限り、クライアントで title タグなどが書き換えられます。そのため、SPA できちんとメタ情報を取ろうと思ったら、単純に HTML をリクエストするだけでなく、JavaScript を実行したあとの DOM からメタ情報を取得する必要があります。ここに、headless browser を利用する理由が生まれます。

headless browser とは、GUI なしで起動するブラウザのことです。スクリーンショットをとったり、ボタンを押して操作するなどの処理がコード化できるので、テストなどで使われます。
headless mode を操作するために、いろいろなライブラリが存在します。今回はその中から、Google が提供する`puppeter`を使いました。
https://developer.chrome.com/docs/puppeteer/

実際にブラウザを起動して、ページを取得し、JS を読み込んで実行するには時間がかかります。そのため、リアルタイム性が求められるサービスでは SPA のメタ情報を取得することまでせず、単純に HTML をリクエストするだけだと思います。実際、slack や Google Document で試してみたところ、単純な SPA には対応していないようでした。

今回は、事前にデータを生成する方式だったため、SSR をしていない SPA ページに対応することにしました。

いまのところ、`Promise.all`で記事ごとに並列実行して、1 分程度でメタ情報を取得できています。

### Github Actions による定期実行

メタ情報を事前に取得しておくことによる弊害が一つあります。それは、メタ情報が変わった場合に、もとの情報が更新されないことです。
この問題に対しては、Github Actions により定期的にデータを取得しなおすことにしました。

Github Actions は、`schedule`というイベントをサポートしており、cron 形式で時間を指定し、定期実行ができます。メタ情報生成処理を Github Actions のワークフロー化し、定期実行すれば問題は解決できました。
