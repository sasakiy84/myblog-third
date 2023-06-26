---
title: Quirks Mode とは
description: セキュリティキャンプで LT をしたときの発表資料です。HTML Spec を読んでいると出てくる Quirks Mode について、具体的な例を示しながら解説します。
date: 2023-03-08
createdAt: 2023-03-08
updatedAt: 2022-03-08
tags:
  - Frontend
  - Security
---

# Quirks Mode

※ 画像はスライドのスクリーンショットです

![](/img/quirks-mode_20230308232721.png)

事前課題で HTML のパースについての仕様を読んでいると、次のような文言が出てきました。

> set the Document to quirks mode.
> https://html.spec.whatwg.org/multipage/parsing.html#the-initial-insertion-mode

ここででてきた`quirks mode`がなにか気になったので、調べた結果を発表します。

![](/img/quirks-mode_20230308233318.png)

Quirks mode とは、「現在の標準仕様から外れた書き方も解釈してあげる」モードのことです。

![](/img/quirks-mode_20230308233415.png)

Quirks mode であるかどうかによって、同じ HTML でも画像のような違いがでます。左の画像が Quirks mode のとき、右の画像が Quriks mode ではないときです。
このサイトは、Quirks mode についてまとめられたページを参考に作成しました。
https://quirks.spec.whatwg.org/

![](/img/quirks-mode_20230308233902.png)

たとえば、色の指定方法です。16 進数で色を指定するときに、現在だと`#`を付けると思いますが、付けるかどうかが統一されていない時代もあったようです。
Quirks mode の場合は、`#`をつけなくても 16 進数のカラーコードだと解釈してくれます。

![](/img/quirks-mode_20230308234544.png)

次はクラス指定です。Quirks mode では、大文字小文字を区別しません。

![](/img/quirks-mode_20230308234703.png)

最後は単位指定です。Quirks mode では、単位をつけなくても解釈してくれます。

![](/img/quirks-mode_20230308234757.png)

とはいえ、現代に残っているサイトで、そんな書き方をしているサイトなんてあるの？という疑問が出てきます。
そこで、古い HTML が残ってそうなサイトを眺めてみました。すると、`#`を使わないで色を指定している現役のサイトが存在していました。
