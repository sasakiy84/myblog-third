---
title: シェルスクリプトが 1 処理ずつ読み込まれて実行される例
description: シェルスクリプトは、最初にすべてのファイルを読み込むわけではなく、1 処理ずつ読み込まれて実行される。そのため、シェルスクリプトが実行されている最中にファイルを変更すると意図しない挙動につながることがある、という例。
date: 2024-09-14
createdAt: 2024-09-14
updatedAt: 2024-09-14
tags:
  - Security
  - Infra
---

# 概要

シェルスクリプトは、最初にすべてのファイルを読み込むわけではなく、1 処理ずつ読み込まれて実行される。
1 処理というのは、改行で区切られた 1 行のことを指す。ただし、`\` で改行を継続させている場合は複数行になる。

スクリプトが逐次で読み込まれるため、シェルスクリプトが実行されている最中にファイルを変更すると意図しない挙動につながることがある、という例。

なお、動作確認は以下の環境で行った。

```sh
$ bash --version
GNU bash, version 3.2.57(1)-release (arm64-apple-darwin23)
Copyright (C) 2007 Free Software Foundation, Inc.
```

# 例

以下のようなスクリプトを用意する。

```sh
#!/bin/sh

TARGET_DIR=/tmp
sleep 10 # very heavy process
echo "the target dir is ${TARGET_DIR}"
```

このスクリプトを実行すると、以下のようになる。

```sh
$ ./script.sh
the target dir is /tmp
```

次に、スクリプトを実行中に、変数をリファクタリングしたとして、以下のようにスクリプトを修正したとする。

```bash
$ ./script.sh &
$ cat - > script.sh <<'EOF'
#!/bin/sh

NEW_TARGET_DIR=/tmp
sleep 10 # very heavy process
echo "the target dir is ${NEW_TARGET_DIR}"
EOF
```

すると、以下のような実行結果が表示される。

```sh
./script.sh: line 5: ess: command not found
the target dir is
```

変数宣言で `NEW` を追加したことで、3 文字分の文字列が追加されたため、読み込んでいた位置がずれてしまい、`process` の後半 3 文字分 `ess` をコマンドとして認識してしまった。 今回の環境では `ess` というコマンドは存在しないため、エラーが発生している。

そして、このシェルスクリプトにはエラー発生時に実行を止める設定が記述されていなかったため、 エラーが発生してもそのまま次の処理に進んでしまい、`echo "the target dir is ${NEW_TARGET_DIR}"` が実行された。 しかし、新しいスクリプトの 1 行目は実行されておらず `NEW_TARGET_DIR` は宣言されていない。 そのため、`the target dir is` と表示された。

# 対策

1. シェルスクリプトを使わない
2. シェルスクリプトを使う場合は、エラーが発生した場合に実行を止める設定を記述する。具体的には、`set -e` を記述する。
3. シェルスクリプトの実行中に inode を維持したままでファイルを更新するようなことをしない。
    - たとえば、`vim` コマンドは inode を変更するため、上記のようなことは起こらない
4. 変数を使うときは、シェルスクリプトの変数展開機能をフル活用する
    - たとえば、次のようにすることでデフォルト値を設定できる `echo "the target dir is ${NEW_TARGET_DIR:-DefaultDir}"`
    - あるいは、次のようにすることで変数の存在チェックができる `echo "the target dir is ${NEW_TARGET_DIR:?Variable is not set or null}"`
    - [bash の man ページ](https://man7.org/linux/man-pages/man1/bash.1.html)の `Parameter Expansion` の項目を参照

# 現実の事例
京大のスーパーコンピューターの納入会社が、シェルスクリプトを不用意に更新したことによって、データが消失したという事例がある。

- [スーパーコンピュータシステムのファイル消失のお詫び | お知らせ | 京都大学情報環境機構](https://www.iimc.kyoto-u.ac.jp/ja/whatsnew/information/detail/211228056999.html)
- [Lustre ファイルシステムのファイル消失について(PDF)(魚拓)](https://www.wareko.jp/blog/wp-content/uploads/2021/12/file_loss_insident_20211228.pdf)
- [Lustre ファイルシステムのファイル消失について(PDF)(Internet Archive)](https://web.archive.org/web/20211228130529/https://www.iimc.kyoto-u.ac.jp/services/comp/pdf/file_loss_insident_20211228.pdf)