---
title: Google Cloud text-to-speech で音声ファイルを生成する Deno スクリプト
description: Google Cloud の text-to-speech を使って、テキストを音声に変換するスクリプト
date: 2024-09-15
createdAt: 2024-09-15
updatedAt: 2024-09-15
tags:
  - Work
---

# 概要
Google Cloud の text-to-speech を使って、テキストを音声に変換するスクリプト。

- [sasakiy84/gcloud-text2speech-converter-deno](https://github.com/sasakiy84/gcloud-text2speech-converter-deno)

このスクリプトの紹介と、実行結果のサンプル、簡単な使い方、Google Cloud Text-to-Speech API の感想を書く。

# 使い道
このスクリプトは、学会発表のために書いた英語原稿を音声化して、暗記するために作成した。
そのため、英語でしか動作確認をしていませんが、Google Cloud Text-to-Speech API が対応している言語ならば変換できるはず。

たとえば、英語記事などのテキストデータを音声化して、作業中に聞くなどの使い方ができる。
ブラウザや OS にもテキスト読み上げ機能がついています。これらの機能は十分高機能ですが、手元に音声ファイルとして保存したいという需要に対応できない。

以下は、音声変換したときのサンプル。

> The National Diet Library (NDL) is the parliamentary library of the National Diet of Japan and was founded in 1948 by the National Diet Library Law, in accordance with Article 130 of the Diet Law, which states in part that "The National Diet Library shall be established in the Diet by a separate law, in order to assist Diet Members in their study and research."
> https://www.ndl.go.jp/en/aboutus/index.html

- [sample.mp3](../img/gcloud-text2speech-converter-sample.mp3)

# 動機
Text-to-Speech API は、[コンソール画面](https://console.cloud.google.com/speech/text-to-speech) から短い文書を変換することができる。
しかし、長い文書を変換する場合は、人間が文書を細かく分割して、それぞれの音声ファイルをダウンロードする必要がある。

この作業を自動化するために、Deno でスクリプトを書いた。

# 使い方
[README](https://github.com/sasakiy84/gcloud-text2speech-converter-deno) に従ってセットアップを行い、以下のように実行する。

```sh
$ deno task start a.txt
```

セットアップが少し面倒くさい。
具体的には、Google Cloud のアカウント作成やプロジェクトの作成、FFmpeg のインストールなどが必要。

# Google Cloud Text-to-Speech API の感想
- 基本的に聞ける音声が出力される
- いくつかのエッジケースでは、出力がおかしくなる
    - たとえば、(a) first item, (b) second item というようにリストを読み上げるとき、(a) を /ə/ として読み上げる。本来は、/eɪ/ と読み上げて欲しい
    - 1928 のような年号として捉えられる数字を読み上げるとき、nineteen twenty-eight と読み上げる。本来は、one thousand nine hundred twenty-eight と読み上げて欲しい
    - ichibanme などの日本語ローマ字表記を読み上げるとき、変な発音になる。複数言語に対応していない
    - これらのエッジケースは応用上の課題として普通に興味深い。特に年号っぽい数字の読み上げは音声の読み上げにおいても文字の連なりではなく意味を考慮する必要があることの例として使える

