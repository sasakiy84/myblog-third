---
title: JSX をテンプレートエンジンとして使い、Vite でビルドする
description: 静的なサイトを作成するときに、JSX をテンプレートエンジンとして使う。このとき、Vite を使ってホットリロードを実現する。
date: 2024-12-13
createdAt: 2024-12-13
updatedAt: 2024-12-13
tags:
  - Work
---

# 前書き
この記事は、とあるサークルのOB/OGで行っている[N代アドベントカレンダー](https://adventar.org/calendars/10235)の 10 日目です。

# 動機

**「欲しかったものはテンプレートエンジンとしての JSX」**

Next.js をはじめとする SSR のサポートを謳った大規模なフレームワークは、本気でフロントエンドを作り込み、きちんと保守点検にコストをかけることができるサイトならばとても便利です。
一方で、古き良き静的サイトを作るには大仰すぎる感が否めません。

プレーンな HTML と CSS でできることが増えている現状、極論をいえばライブラリやフレームワークを使わずにサイトを作ることも可能です。
しかし、繰り返し処理や条件分岐、コンポーネント化などを考えると、やはりなんらかのテンプレートエンジンが欲しくなります。

そこで候補に上がるのは、Handlebars や Mustache などの昔ながらのテンプレートエンジンですが、欲を言えば慣れ親しんでいて、エディタのサポートも充実している JSX を使いたいところです。

同じモチベーションで作成されている、まさにどんぴしゃりなライブラリとして、[minista](https://minista.qranoko.jp/) があります。
シンプルなツールですが、単なるテンプレートエンジンにとどまらず、開発時のホットリロードや綺麗な HTML を納品するための機能、マークダウンへの対応など、便利な機能を搭載しています。
すこし使ってみたところ、とても使いやすかったです。

ただ、もう少し自分の好みに合わせたカスタマイズがしたかったので、minista と同じように Vite を用いながら JSX をテンプレートエンジンとして使う方法を自分で試してみました。

# 大まかな構成
中心となるのは、Vite の [Server-Side Rendering 機能](https://vite.dev/guide/ssr)です。
`createViteServer` を使って Vite のサーバーを作成し、`ssrLoadModule` でコンテンツのハンドラーを呼び出します。
開発環境のときは、おおまかにこの方法でホットリロードを用いた開発が実現できます。

静的なサイトとしてファイルを出力するには、`build` API を用いて前段でビルドをしたのちに、同じように `ssrLoadModule` を利用することで、HTML を出力することができます。

# 実装
実装を説明します。

## ssr-entrypoint.ts
まずは、`ssrLoadModule` で呼び出すコンテンツのハンドラーを作成します。
パスの部分や、細かい関数の実装などはそれぞれの環境に合わせて実装する必要がありますが、おおまかには以下のようなイメージです。


```ts
// ssr-entrypoint.ts
import { resolve } from "node:path";
import {
  renderEntryList,
  renderEntryMarkdown,
  renderTsxFromFileTree,
} from "./renderers";

export const render = async (url: string): Promise<string> => {
  const requestedPath = decodeURIComponent(url);
  console.debug(`Rendering: ${requestedPath}`);
  return renderPage(requestedPath);
};

const renderPage = async (requestedPath: string): Promise<string> => {
  switch (true) {
    case /^\/$/.test(requestedPath): {
      return await renderEntryList();
    }
    case /^\/entry\/.+/.test(requestedPath): {
      const path = resolve(import.meta.dirname, `..${requestedPath}.md`);
      return await renderEntryMarkdown(path);
    }
    default: {
      return renderTsxFromFileTree(requestedPath);
    }
  }
};
```

## develop.ts
次に開発用のスクリプトを作成します。
TS の型を合わせるためには少し修正する必要がありますが、おおまかには以下のようなイメージです。
`ssrLoadModule("./src/ssr-entrypoint.ts")` で先ほど作成したコンテンツのハンドラーを呼び出しています。

```ts
// develop.ts
import { readFile } from "node:fs/promises";
import { type Connect, createServer as createViteServer } from "vite";

const isDocumentRequest = (req: Connect.IncomingMessage): boolean => {
  const secFetchDest = req.headers["sec-fetch-dest"];
  return secFetchDest === "document";
}

const main = async () => {
  const vite = await createViteServer({
    plugins: [],
    server: {
      port: 5173,
    },
    appType: "custom",
  });
  const { transformIndexHtml, ssrLoadModule } = vite;

  vite.middlewares.use(async (req, res, next) => {
    const url = req.originalUrl;

    if (!isDocumentRequest(req)) {
      return next();
    }

    try {
      const template = await readFile("./index.html", "utf-8");
      const transformedTemplate = await transformIndexHtml(url, template);
      const { render } = await ssrLoadModule("./src/ssr-entrypoint.ts");
      const appHtml = await render(url);
      const html = transformedTemplate.replace("<!--app-html-->", appHtml);
      res.statusCode = 200;
      res.setHeader("Content-Type", "text/html");
      res.end(html);
    } catch (error) {
      vite.ssrFixStacktrace(error);
      next(error);
    }
  });

  await vite.listen();
  vite.printUrls();
};

main();
```

ちなみに、`index.html` は普通に以下のような感じです。
```html
<!doctype html>
<html>
<head>
    <meta charset="utf-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <title>Page Title</title>
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <link rel="stylesheet" href="/pages/index.css" />
</head>

<body>
    <!--app-html-->
</body>

</html>
```

これを、tsc でコンパイルして `node` で実行したり、`ts-node` で実行したりすれば、開発環境の出来上がりです。

## build.ts
静的なファイルを出力するためには、もう一工夫必要です。
以下のように、事前に出力するべき URL パスの一覧を作成したうえで、それを `ssrLoadModule` を経由した `render` 関数に渡して、HTML を出力します。

```ts
// build.ts
import { mkdir, readdir, writeFile } from "node:fs/promises";
import { dirname, join, resolve } from "node:path";
import { createServer as createViteServer, build as viteBuild } from "vite";

const main = async () => {
  const ssgResult = await viteBuild({
    configFile: false,
    build: {
      outDir: "./dist/static",
      emptyOutDir: false,
      minify: false,
    },
  });
  if (!("output" in ssgResult)) {
    throw new Error("ssgResult.output is not found");
  }

  const templateOutput = ssgResult.output.find(
    (output) => output.fileName === "index.html",
  );
  if (!templateOutput || !("source" in templateOutput)) {
    throw new Error("template is not found");
  }
  const template =
    typeof templateOutput.source === "string"
      ? templateOutput.source
      : await templateOutput.source.toString();

  const vite = await createViteServer({
    server: {
      hmr: false,
    },
  });
  const { ssrLoadModule } = vite;
  const { render } = await ssrLoadModule("./src/ssr-entrypoint.ts");

  const renderTable = [["/", "index.html"]];

  const entryMarkdownFilenames = await readdir("./entry/");
  for (const filename of entryMarkdownFilenames) {
    renderTable.push([
      `/entry/${filename}`,
      `./entry/${filename.replace(/\.md$/, "/index.html")}`,
    ]);
  }

  for (const [url, filename] of renderTable) {
    const html: string = await render(url.replace(/\.md$/, ""));
    const rendered = template.replace("<!--app-html-->", html);
    const renderedFilename = resolve("./dist/static", filename);
    const renderedFiledirFullPath = join("./dist/static", dirname(filename));
    await mkdir(renderedFiledirFullPath, { recursive: true });

    await writeFile(renderedFilename, rendered, {
      encoding: "utf-8",
    });
  }

  vite.close();
};
main();
```

これを、`develop.ts` と同じように実行すれば、静的なサイトが出力されます。

# 細かいこと
## css ライブラリとして panda を使う
公式ドキュメントの通りにセットアップすれば動きます。
`index.css` を読み込むことを忘れずに。

## マークダウンのホットリロード
vite では、[プラグインによってホットリロードを制御すること](https://vite.dev/guide/api-plugin.html#handlehotupdate)ができます。
`createViteServer` のオプションに以下のようにプラグインを設定することで、任意のファイルをホットリロードできます。

```ts
const vite = await createViteServer({
  plugins: [
    {
      name: "vite-plugin-md",
      async handleHotUpdate({ file, server }) {
        if (file.endsWith(".md")) {
          server.ws.send({
            type: "full-reload",
          });
          return [];
        }
      },
    },
  ],
  server: {
    port: 5173,
  },
  appType: "custom",
});
```

## 綺麗な HTML を出力する
prettier などを使って、出力される HTML を整形できます。

```ts
import { format } from "prettier";

const main = async () => {
    // 省略

    await mkdir(renderedFiledirFullPath, { recursive: true });
    const formatted = await format(rendered, { parser: "html" });

    await writeFile(renderedFilename, formatted, {
      encoding: "utf-8",
    });

    // 省略
}
```

# まとめ
自由度が高すぎるため、中規模・大規模なサイトを作成するには適さないですが、簡単なサイトを自由度高く構築するには使いやすい方法だと思います。
完成系の HTML が書き出されているような古き良きサイトを作るときに参考にしてください。

# 参考リンク
- [minista](https://minista.qranoko.jp/)
- [viteがプラグインなしでできることを探る | stin's Blog](https://blog.stin.ink/articles/vite-without-plugins)
- [Vite + React の SSR/SSG の基本的な動きを理解する - kasya blog](https://kasyalog.site/blog/vite-react-ssr-ssg-basic/)
- [テンプレートエンジンに React を使いつつ、きれいな HTML を生成したいんじゃ！！](https://zenn.dev/otsukayuhi/articles/e52651b4e2c5ae7c4a17)
- [vite で ssg 構築](https://ousttrue.github.io/posts/2024/0901-ssg-by-vite/index.html)