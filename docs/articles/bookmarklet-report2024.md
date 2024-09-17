---
title: Bookmarklet Report 2024
description: Bookmarklet で少し複雑なことをしたので、その記録。主にブラウザのセキュリティ関連の仕様との関わりについて。
date: 2024-09-17
createdAt: 2024-09-17
updatedAt: 2024-09-17
tags:
  - Frontend
  - Work
  - Security
---

# 概要

Bookmarklet (ブックマークレット) で少し複雑なことをしたので、その記録。主にブラウザのセキュリティ関連の仕様との関わりについて、メモを残す。

# 背景 / やろうとしたこと
## Bookmarklet とは
Bookmarklet というものがある。
ブラウザのブックマークに JavaScript を登録しておくと、任意のページでそのブックマークをクリックするだけで、そのページ上で JavaScript が実行される。
検証画面のコンソールにコードを貼り付けて実行することのショートカットのようなもの。

具体的には、URL のスキーマの部分に、`javascript` という文字列を入れることで、JS を実行するリンクとなる。
Bookmarklet 以外の文脈だと、CTF などでよく見かける。

たとえば、以下のような URL を登録すると、そのページのタイトルをアラートで表示することができる。

```
javascript:alert(document.title)
```

URL であるので、当然 URL encode する必要がある。
逆に言えば、URL encode をしてしまえば、どのようなコードでも登録できる。

昔はブラウザが受け付ける URL には文字数制限があったらしいが、現在では表示上の制限があるだけで、中身の解釈される文字列に対しては制限がない。

- [URLの最大文字数って結局いくつなのさ～IE亡き後の新常識を探る #RFC3986 - Qiita](https://qiita.com/_matuzaki/items/70fb639f7ed7463f9943)

実際、12000 文字程度のコードを登録しても Chrome, Edge, Vivaldi の PC 版、スマホ版で問題なく動作した（すべて Chromium ベースではあるが）。

ただし、基本的にページ上で JS を実行したい場合、ブラウザの拡張機能を使う方が便利でお行儀もよい。今回は、以下に述べる理由から、Bookmarklet を用いた。

## やろうとしたこと
サイト上の要素を取得して [mozilla/readability](https://github.com/mozilla/readability) と [mixmark-io/turndown](https://github.com/mixmark-io/turndown) によってマークダウンに変換し、かつ画像を Base64 に変換して、マークダウンと画像データを JSON 形式で自前サービスの API に投げ、結果をページ上に表示するブックマークレットを作った。

基本的にやろうとしていることはブラウザの拡張機能で実装できる。
拡張機能を使ったほうがお行儀がよく（セキュリティ上の考慮がなされているため）、リッチな機能もついてきて、配布もしやすい。

しかし、今回は以下の理由から Bookmarklet を使った。

- 自前サービスのためのツールであり、他人に配布する必要がない
- API リクエストを投げて簡単な要素をページに挿入するだけの機能なので、拡張機能に付随するリッチな機能は不要
- PC だけでなくスマホでも使いたい
    - 基本的にスマホの主要ブラウザで拡張機能に対応しているブラウザはない
    - Edge と Firefox は先行リリース版で拡張機能対応をはじめているため、今後はスマホでも拡張機能を使えるようになるかもしれない


# 事例集
上記の Bookmarklet を作成するときに苦労した点をまとめる。

## Fetch API で外部にリクエストを送信できない
Content Security Policy (CSP) により、外部にリクエストを送信できない場合がある。
この場合は、ペイロードを `textarea` に貼り付けて送信できる自前サービスのページを作成したうえで、ペイロードを JSON にしてクリップボードにコピーしたうえで、新しい専用のページを JS で開き、そこに手動でペイロードを貼り付けて送信するという方法をとった。
これがそこそこ（感覚的には 10 ~ 20 回に 1 回くらい）あるため、早めに拡張機能へ移行しようと考えている。それだけ CSP が普及してきているので、いいことだろうが。

## 要素が挿入できない
この原因はいくつか可能性がある

- Content Security Policy (CSP) によるもの
- [Trusted HTML](https://developer.mozilla.org/en-US/docs/Web/API/TrustedHTML) によるもの

たとえば、Script タグを挿入してサードパーティライブラリを読み込みたいときがある。
しかし、Contents Security Policy (CSP) を設定しているサイトでは、Script タグを挿入できずにエラーが発生する。

Trusted HTML は、一度だけ個人ブログでみかけたことがあるが、今見返したら解除されていた。

## Service Worker でリクエストが見られてしまう可能性がある
Service Worker を用いると、リクエストをインターセプトして内容をもとに何かしらの処理を行える。
Service Worker のヘッダー処理は、以下のように定義されている。

> For the purposes of fetching, there is an API layer (HTML’s img, CSS’s background-image), early fetch layer, service worker layer, and network & cache layer. `Accept` and `Accept-Language` are set in the early fetch layer (typically by the user agent). Most other headers controlled by the user agent, such as `Accept-Encoding`, `Host`, and `Referer`, are set in the network & cache layer. Developers can set headers either at the API layer or in the service worker layer (typically through a Request object). Developers have almost no control over forbidden request-headers, but can control `Accept` and have the means to constrain and omit `Referer` for instance.
> 
> [Fetch Standard](https://fetch.spec.whatwg.org/#http-header-layer-division)

Fetch API を通したヘッダーの付与は、`API layer` で行われ、Service Worker での処理はその後の `service worker layer` で行われるため、API レイヤーでセットしたヘッダーは Service Worker でキャッチできる。

たとえば、以下のように Authorization ヘッダーを付与してリクエストを送信すると Service Worker によってその情報がキャッチされて、外部に送信されてしまう可能性がある。


```html
<!DOCTYPE html>
<html lang="en">

<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Service Worker Demo</title>
  <script defer>
    console.log("Hello, Service Worker!");
    if ('serviceWorker' in navigator) {
      window.addEventListener('load', () => {
        navigator.serviceWorker.register('/service-worker-test.js')
          .then(async (registration) => {
            console.log('Service Worker registered with scope:', registration.scope);
            await fetch(
              "https://example.com/",
              {
                method: "GET",
                headers: {
                    "Authorization": "Bearer 1234567890",
                },
              },
            );
          })
          .catch((error) => {
            console.error('Service Worker registration failed:', error);
          });
      });
    } else {
      console.log("There is not serviceWorker")
    }

  </script>
</head>

<body>
  <h1>Hello, Service Worker!</h1>
</body>

</html>
```

```js
// service-worker.js

self.addEventListener("install", (event) => {
	console.log("Service Worker installing.");
});

self.addEventListener("activate", (event) => {
	console.log("Service Worker activating.");
});

self.addEventListener("fetch", (event) => {
	console.log("- Fetch -");
	for (const pair of event.request.headers.entries()) {
		console.log(pair[0] + ": " + pair[1]);
	}
	fetch("https://www.example.com").catch((error) => {
		console.error("Failed to fetch", error);
	});

	return fetch(event.request);
});
```

これを実行すると、

```
- Fetch -
service-worker-test.js:14 accept: */*
service-worker-test.js:14 authorization: Bearer 1234567890
service-worker-test.js:14 sec-ch-ua: "Not/A)Brand";v="8", "Chromium";v="126", "Google Chrome";v="126"
service-worker-test.js:14 sec-ch-ua-mobile: ?0
service-worker-test.js:14 sec-ch-ua-platform: "macOS"
service-worker-test.js:14 user-agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36
```

が表示されるし、普通に example.com にリクエストを送れている。

そのため、認証情報は Cookie を用いたほうが安全である。
一方で、Cookie を使う場合は　Same Site 属性まわりに気を付ける必要がある（おそらく `SameSite=None` を指定する必要がある？）。Same Site 属性まわりは現在さまざまな議論が定期的に行われているため、最新の情報を確認すること。

## Same Origin Policy により Fetch API で取得した情報の中身を見れない
自身が管理しているサイトへの通信ならば、CORS ヘッダーを適切に指定することで対応できる。
困るのは外部サイトへの通信である。
たとえば、URL 短縮サービスのリンクはすべて解決してからサーバー側に送信したいと考えても、Same Origin Policy により中身を見ることができない。
また、おなじことは IFrame の中身についてもいえる。
Cross Origin で IFrame を埋め込んでいた場合、その中身を取得することはできない。

# TIPs
実装するときの細かな TIPs をまとめる。

## Alert as Escape Hatch
数々のブラウザのセキュリティ関連の制約にひっかかったときに、最後に頼れるのが `alert` である。
とくに、エラー処理の最終段階でユーザーにどうなったのかを伝えるときに有用。
スマホでは検証画面をひらいて状況を確認することもできないので、JS のコード全てを `try { ... } catch (error) { alert(error) }` で囲むとよい。
また、Bookmarklet 特有の TIPs ではないが、あらかじめ予測可能なエラーは、Custom Error Class を作成しておいて、それをもとにエラーメッセージを表示するとコードの見通しが良くなる。

## CDN からのスクリプトをあらかじめ読み込んでおく
Script タグを用いた外部スクリプトの読み込みは、CSP により制限されることがある。
対策としては、CDN で取得できるコードをローカルに落としておいて、単純にそれを自分が書いたコードと結合して登録すればよい。

## Copy to Clipboard via Textarea
これは、Bookmarklet に限った話ではない。
クリップボードにアイテムを挿入するために、`navigator.clipboard.writeText` という API がある。
しかし、この API はユーザーのインタラクションなどの制約を満たさなければ動作しない。
そのため、`textarea` にテキストを挿入して、それを選択してコピーするという方法をとることがある。
この方法は、ユーザーのインタラクションを必要としないため、Bookmarklet で使うのに適している。

```js
const textarea = document.createElement("textarea");
textarea.value = "Hello, World!";
document.body.appendChild(textarea);
textarea.select();
document.execCommand("copy");
document.body.removeChild(textarea);
```

まあ、一方で、歴史的な理由があるのだろうとはいえ、`writeText` でできないことを `execCommand` でバイパスできてしまっていいの？という気持ちもある。

## 挿入する要素には IFrame を使う
IFrame で要素を挿入する利点は二つある。
一つ目は、普通のウェブサイトを作成するときのツールチェインがそのまま使えること。Bookmarklet のコードだけで要素を実現しようとすると、コードが複雑になるし、エディタ補完も効かない。
二つ目は、Content Security Policy におけるスクリプト関連の制約を回避できる場合があること。Script 関連の制約が弱まる場合がある。
たとえば、以下のような HTML の場合は、IFrame を Bookmarklet で挿入することで、`script-src 'self'` によるスクリプトの制約を IFrame の中では回避できる。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Content Security Policy サンプル</title>

  <meta http-equiv="Content-Security-Policy" content="script-src 'self';">

</head>
<body>
    <h1>CSPでスクリプトの制限がかけられたウェブページ</h1>
</body>
</html>
```


# さいごに
Bookmarklet で複雑なことをしようとすると、ブラウザのセキュリティ関連の機能に引っかかることが多い。
これは不便な一方で、ブラウザの仕様を調べるきっかけになったり、各サイトのセキュリティ関連の実装状況を手軽に知ることができるので面白い。
たとえば、ただのブログサイトなのに非常に堅牢な実装をしているサイトをみかけたり、逆に結構重要な機能をもっているのにあまりセキュリティ関連の実装がされていないサイトをみかけることもある。

また、Bookmarklet を一度自作したことで、この手段が自然に選択肢に入ってくるようになった。
面倒くさい作業をパパッと Bookmarklet にすることで、日々のネット徘徊がすこしストレスフリーになる。


