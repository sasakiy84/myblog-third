---
title: TSG CTF 2024 で構築した CTFd の環境
description: TSG CTF 2024 にて、CTFd のサーバー担当をした。CTFd を Google Cloud で稼働させた
date: 2024-12-19
createdAt: 2024-12-19
updatedAt: 2024-12-19
tags:
  - Work
---

# 前書き

Google Cloud で CTFd を構築したときのメモと、メトリクスの記録。
以後、TSG に限らず CTF のインフラを担当する人に向けた参考情報として書く。


# 大まかな構成

![サーバーの構成図](../img/tsgctf-2024-ctfd-architecture.png)

完成図としては、上のような状態を目指す。なお、以下の点に注意する。

- **バックエンドの CTFd を2台にするとログインエラーの不具合が発生したため、今回実際には強いインスタンスを一つ立てて、LB の通信をすべてそこに流した。**これはおそらく CTFd の secret を明示的に指定していなかったために発生した不具合なので、次回は環境変数でそれを設定すれば、問題なく動くと思われる
- また、他のネットワークで問題サーバーや問題のヘルスチェックサーバーなどが動いている。CTFd をたてるとき、これらは直接関係ないため、今回の図からは省いている
    - CTFd は手伝ってもらいつつ私がメインで担当しましたが、他のサーバーは他の人が担当していました
    - 問題サーバーと異なり普通のウェブサービスなので、CTF 特有のセキュリティ考慮などはしてない（本当なら、問題サーバーとネットワークを別にしたほうがよさそうだけど）
- リハーサル環境において oauth2proxy をたてる場合は、CTFd の docker-compose のサイドカーとして、それぞれのインスタンスの内部に oauth2proxy を立てた
- 終了後はプラグインを使ってスコアボードを静的なファイルとしてエクスポートしてホスティングし、CTFd のインスタンスを停止した。


以下、おおまかな手順を記載しつつ、沼りそうな箇所について補足をする。
事前準備として、以下のことをしておくとよい。

- 手元のコンピューターに gcloud コマンドをインストールして、ログインをしておく
- Mailgun の認証情報を適切な人からもらっておく
- ドメインを Cloud DNS からいじれるように設定しておく
- 予算を確認して、アラートをきちんと設定しておく
    - 今回は、Google Cloud からインフラ面のクレジットを受け取っている（Google Cloud でサーバーを構築した理由）
- 関係するリポジトリを確認しておく
    - [tsg-ut/CTFd](https://github.com/tsg-ut/CTFd)
    - [tsg-ut/ctfd-theme-tsgctf](https://github.com/tsg-ut/ctfd-theme-tsgctf)
    - [tsg-ut/ctfd-plugin-tsgctf](https://github.com/tsg-ut/ctfd-plugin-tsgctf)
    - [tsg-ut/score.ctf.tsg.ne.jp](https://github.com/tsg-ut/score.ctf.tsg.ne.jp)


インスタンスの立ち上げにそこそこ時間がかかったりするので、最短でも1h~2h程度見ておくとよい。
規模としては、899 users、496 teams。
CTFd のサーバーとしては、開始時に少し重くなったくらいで、ほぼ安定して稼働していた。

# 各コンポーネントの説明

## ネットワークを作る
論理構成や配線を考える必要がないため特に難しいことはなく、適切な名前をつけるだけ。
サブネットも必要なときに簡単に追加できる。
以下で作成するリソースでネットワークを選べる場合は、必ずこのネットワークに所属するようにしておく。（ネットワークの設定は、「高度な設定」とかに隠されている場合があるので、きちんと確認する）

## Cloud SQL をたてる
適切な名前をつけて、ネットワークを設定する。設定項目とそれに対するコメントは以下のとおり。

- MySQL 8.0.31
- 2vCPU, 8GM メモリ、100GB ストレージ
    - 終わったときにストレージ使用量は 1GB強なので、十分に余裕あり
    - メモリの使用量は、最大で 12%程度なので、十分に余裕あり
    - CPU 使用率は最大で17%程度なので、十分に余裕あり
- Enterprise エディション
- パブリック IP 接続は無効
- root ユーザーのパスワードをコンソールから設定するので、控えておく
    - アットマークを含めるとエスケープが必要になって面倒くさいため、記号はできるだけ含めないでパスワードを十分に長くするほうが楽。エスケープする場合は、確か URL のエスケープでいけたはず
- 暗号化されていない直接接続は許可した
- インスタンスが起動したら、サイドバーの「データベース」みたいな項目からデータベースを作成することができる
    - ctfd という名前のデータベースを作成する。文字セットは utf8mb4 。照合はデフォルトのものを使う


[Cloud SQL の CPU 利用率](../img/tsgctf-2024-ctfd-sql-cpu.png)


## Redis をたてる

- 特に難しいことはない。
- Memorystore には、Redis クラスタ、Redis、Memcached がある。Redis をたてる
- v7.0 を使った
- レプリカ数 0、最大メモリ数 2GB で立てた
    - メモリ使用量は常に 0.01 GB 程度。十分に余裕あり



## Cloud Storage を設定する

- 問題の dist ファイルをアップロードするためのストレージ。問題登録画面からファイルをアップロードできる
- バケットを設定して認証情報を取得するだけだが、セキュリティ上の注意点がある
- 本番環境では、東京と大阪のデュアルリージョンにした
- ストレージクラスはスタンダード
- 権限のタブから、以下を設定
    - プリンシパル：allUsers、ロール：Storage レガシー オブジェクト読み取り
    - allUsers は Google  Cloud の予約語で、パブリックアクセスを表す
    - 「Storage レガシー オブジェクト読み取り」は、リスト権限は与えず、個々のオブジェクトの URL を知っている場合のみ許可するというもの。バケットに含まれるオブジェクトのリスト権限を与えないことが重要
    - `https://storage.googleapis.com/バケット名` にアクセスして、Denied になっていればよい
    - cf. https://zenn.dev/catnose99/articles/18720e3af36d22
- CTFd に渡す認証情報を取得する
    - CTFd は S3 に対応しているので、Cloud Storage を利用する場合は S3 互換の API キーを生成する
    - API キーは HMAC キーと呼ばれる。公式ドキュメントを参照
        - cf. https://cloud.google.com/storage/docs/authentication/hmackeys?hl=ja
    - ただし、本番で実際にやろうとしたところ、バケットに対してサービスアカウントに紐づけて生成したキーを利用してもうまくいかなかった。admin 権限をもっているユーザーアカウントに対して生成した API キーを利用するとうまくいった。たぶん、サービスアカウントに対してうまく権限が割り振られていなかったのかもしれない？



## Compute Engine を立てて CTFd を起動する

- ぽちぽちと設定を行なって、インスタンスをたてる
    - e2-standard-8 (8vCPU, 32GB)
        - 一台で賄う場合、CTF 開始時を除いて安定して稼働していた。開始時はすこしサイトが重くなった（とはいえ、落ちたとは言えないレベル）
        - LB によるスケールアウトがうまく機能している場合、開始時は半分くらいのスペックを 3 台くらい立てて、その後は 1 台まで落とすくらいの運用でよさそう
    - Container-Optimized OS, 117-18613.75.66 LTS (x86-64)
        - セキュリティのことを考えて、最低限の機能のみが入っている Container-Optimized OS を使う
    - Storage 50GB
        - 20GB あれば余裕だった。けど、そこまで高いわけでもないので一応
    - ファイヤーウォールは、HTTP トラフィック、HTTPS トラフィック、ロードバランサのヘルスチェックをすべて許可した。SSH の許可も必要だったかも
        - HTTPS はいらない気もする。HTTP ももしかしたらいらないかも
        - **ただし、外部IPアドレスを付与している場合、CTFd の admin アカウントを登録する前に先にアクセスされてしまう可能性がある。気になる場合は、SSH でトンネルを使って設定したあとに、HTTP 接続を許可する**
    - 外部 IP アドレスもつけた
        - 本来ならいらないはずだけど、なぜか外部IPアドレスをつけないと外部への通信（curl）が通らなかった
        - VPC network あたりの設定が怪しい？時間がなかったため、あまり調べていない
        - 外部 IP アドレスをつけない場合、gcloud コマンドなどで SSH をするときは、アカウントに iap.tunnelInstances.accessViaIAP 権限が必要
- docker compose を使って CTFd を起動する
    - container optimized OS は、docker compose コマンドが入っていないため、compose が入っている docker コンテナを使って compose コマンドを使う
    - `docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v "$PWD:$PWD" -w="$PWD" docker:24.0.9 compose up -d`
    - compose.yaml は下の資料の通り
        - 素朴に、サーバー上で clone したものを編集している
    - 注意点としては、SECRET の環境変数を各インスタンス共通の文字列に設定すること。こうしなければ、インスタンスごとに異なる値でクレデンシャルの生成・検証を行うようになってしまい、LB の下に複数のインスタンスをたてたときに、login error に確率でなってしまう
        - シークレットキーについての昔の調査記録は以下
        - https://docs.ctfd.io/docs/deployment/configuration/
    - リハーサルのときは、aouth2proxy の設定もする。
- CTFd にブラウザからアクセスして、admin アカウントを作成して、データベースの設定をする
    - ローカルとのトンネルを開いて設定してしまうのが安全
        - `gcloud compute ssh --zone "asia-northeast1-b" "tsgctf-2024-ctfd-1" --tunnel-through-iap --project "tsg-ctf" -- -L 8000:localhost:8000`
        - 動作確認もできる
    - テーマが選べないときは、きちんと CTFd フォルダの中の CTFd フォルダに移動してからテーマやプラグインを clone しているか確かめる

### Compute Engine の CTFd 関連の資料
#### ctfd/docker-compose.yaml

```yaml
services:
    ctfd:
        build: .
        user: root
        restart: always
        ports:
            - "8000:8000"
        environment:
            - SECRET_KEY=tsgctf-foo-2024
            - UPLOAD_FOLDER=/var/uploads
            - DATABASE_URL=mysql+pymysql://root:pass@IPADDRESS/ctfd
            - REDIS_URL=redis://IPADDRESS:6379
            - WORKERS=1
            - LOG_FOLDER=/var/log/CTFd
            - ACCESS_LOG=-
            - ERROR_LOG=-
            - REVERSE_PROXY=true
            - MAILFROM_ADDR=tsgctf@tsg.ne.jp
            - MAIL_SERVER=smtp.mailgun.org
            - MAIL_PORT=465
            - MAIL_USEAUTH=True
            - MAIL_USERNAME=tsgctf@tsg.ne.jp
            - MAIL_PASSWORD=
            - UPLOAD_PROVIDER=s3
            - AWS_ACCESS_KEY_ID=
            - AWS_SECRET_ACCESS_KEY=
            - AWS_S3_BUCKET=bucketname
            - AWS_S3_ENDPOINT_URL=https://bucketname.storage.googleapis.com
        volumes:
            - .data/CTFd/logs:/var/log/CTFd
            - .data/CTFd/uploads:/var/uploads
            - .:/opt/CTFd:ro
        networks:
            default:
            internal:
 
   nginx:
        image: nginx:stable
        restart: always
        volumes:
            - ./conf/nginx/http.conf:/etc/nginx/nginx.conf
        ports:
            - 80:80
        depends_on:
            - ctfd

networks:
    default:
    internal:
```

リハーサルで認証をかけたいときは、以下の項目も設定する。
```yaml
    auth2proxy:
        image: quay.io/oauth2-proxy/oauth2-proxy:v7.7.1-alpine
        restart: always
        volumes:
            - ./conf/oauth2-proxy/config.cfg:/etc/oauth2-proxy.cfg
        command: ["oauth2-proxy", "--config", "/etc/oauth2-proxy.cfg"]
```

#### conf/oauth2-proxy/config.cfg

```cfg
http_address = "0.0.0.0:4180"

provider = "github"
client_id = "id"
client_secret = "secret"

redirect_url = "https://to-rehearsal-domain-of.tsg.ne.jp/oauth2/callback"

cookie_secret = "cookiesecret"
github_org = "tsg-ut"

email_domains = ["*"]
pass_access_token = true
pass_authorization_header = true

set_xauthrequest = true

whitelist_domains = ["*.tsg.ne.jp"]
```


#### conf/nginx/http.conf

```conf
worker_processes 4;
events {
  worker_connections 1024;
}
http {
  # Configuration containing list of application servers
  upstream app_servers {
    server ctfd:8000;
  }
  server {
    listen 80;
    gzip on;
    client_max_body_size 4G;

    location /oauth2/ {
        proxy_pass       http://auth2proxy:4180/oauth2/;
        proxy_set_header X-Real-IP               $remote_addr;
        proxy_set_header X-Scheme                $scheme;
        proxy_set_header X-Auth-Request-Redirect $request_uri;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

   location = /oauth2/auth {
        proxy_pass                        http://auth2proxy:4180/oauth2/auth;
        proxy_set_header X-Real-IP        $remote_addr;
        proxy_set_header X-Scheme         $scheme;
        proxy_set_header Content-Length   "";
        proxy_pass_request_body           off;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Handle Server Sent Events for Notifications
    location /events {

      proxy_pass http://app_servers;
      proxy_set_header Connection '';
      proxy_http_version 1.1;
      chunked_transfer_encoding off;
      proxy_buffering off;
      proxy_cache off;
      proxy_redirect off;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Host $server_name;
    }

    # Proxy connections to the application servers
    location / {

      proxy_pass http://app_servers;
      proxy_redirect off;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Host $server_name;

      auth_request /oauth2/auth;
      error_page 401 = /oauth2/sign_in;
      auth_request_set $user   $upstream_http_x_auth_request_user;
      auth_request_set $email  $upstream_http_x_auth_request_email;
      proxy_set_header X-User  $user;
      proxy_set_header X-Email $email;
    }
  }
}
```

#### 当日の鯖の情報
ただし、強化したインスタンスをたてたのは開始直前なので、それ以前の情報はない。
Google Cloud 側で勝手に取ってくれる情報。

CPU利用率
![CTFd のインスタンスの CPU メトリクス](../img/tsgctf-2024-ctfd-ce-cpu.png)

ネットワーク
![CTFd のインスタンスのネットワークメトリクス](../img/tsgctf-2024-ctfd-ce-network.png)

ディスク IO
![CTFd のインスタンスの Disk IO メトリクス](../img/tsgctf-2024-ctfd-ce-diskio.png)

パケット数
![CTFd のインスタンスのパケット数メトリクス](../img/tsgctf-2024-ctfd-ce-packet.png)




## LB をたてる

- ぽちぽちするだけ
- LB は少し複雑で、論理的なコンポーネントとしては、ロードバランサー、バックエンドサービス、フロントエンドサービス、ヘルスチェックで構成される
- バックエンドサービスには、バックエンドタイプとしてインスタンスグループを指定したうえで、複数の unmanged instance group を登録できる。これを使って、ゾーンの分散構成を行う。
- グローバル外部アプリケーション ロードバランサという種類のものをたてた。manged の SSL にするために。
    - 通信の向き先の規則は、セッションアフィニティという機能で設定できる。おそらくこれを設定しなくても動くはずだとはおもうが...

## ドメインを設定する

- Aレコードを設定するだけ
- なにかあったときのために、TTL は最初は短くしておくと良い
- 証明書が取得されるまで１日とか時間がかかる場合がありそう。早めにやっておく必要あり（とはいえ、今回は数分で取得できたが）

## LB にインスタンスを追加してみる

- CTFd を別のゾーンでもう一つ建てて、それのインスタンスグループを作成して、バックエンドサービスに追加するだけ
- NGINX で各インスタンスを示すヘッダとかをつけるとデバッグしやすいのかもしれない
- 何回かリロードしたり、いろんな IP アドレスから試したりして、問題ないことを確認する


# 値段

本番環境だけの値段をみると、１日あたりの値段は以下のようにおよそ 4000 円となった。
Compute Engine と Cloud SQL が大部分を占めている。ただし、この料金には問題サーバーなど CTFd 関連以外のサービスも含まれている。

![１日の値段のグラフ](../img/tsgctf-2024-ctfd-pricing-per-day.png)

リハーサルの環境を含めた合計の値段は、およそ 6 万円程度だった。
予算を考えるときは、CTF 当日だけではなく、リハーサルの環境や当日までの登録期間、終了後にもスコア関係なしにある程度問題を公開しておく期間などを考慮しておく必要がある。
