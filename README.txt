
# Mastodon 環境構築用 Docker Compose (Draft)

※ ドキュメント作成中・ドラフト

## 概要

- Docker Compose で Mastodon 実行環境をセットアップ
 - オリジナルの docker-compose.yml との主な違い
  - Nginx ( Let's Encrypt 対応 SSL ) 設定
  - Let's Encrypt (certbot) 用設定
  - Exim4 ( メール送信用) )
  - 内部通信用ネットワークの定義

## 事前準備

- Mastodon を動かすためのサーバに対して、ホスト名(Aレコード)が割り当てられていること
- Docker CE 動作環境

## 設定手順

### Docker 関連セットアップ

まず、Docker Engine（Dockerコンテナを実行するためのプラットフォーム）と Docker Compose（複数のコンテナをサービスとして一括管理するプログラム）をセットアップします。以降の手順は Linux の場合です。

Docker Engine は次のコマンドを実行してセットアップすると、Docker プロジェクトの最新安定版をダウンロードします。

```
# curl -sSL https://get.docker.com/ | sh
```

それから Docker Engine の起動と、ブート後のサービス自動起動設定をします。次のコマンドは CentOS 7 の場合です。

```
# systemctl enable docker
# systemctl start docker
```

次は、Docker Compose のバイナリをセットアップします。

```
curl -L https://github.com/docker/compose/releases/download/1.12.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

作業用 docker アカウントを作成します（root権限で全ての Docker 操作が可能なため、事実上の root ユーザとなりますので取り扱いにはご注意ください。あるいは、この手順を省略し、一般ユーザから sudo docker のように実行しても構いません）。

```
# /usr/sbin/adduser -g docker docker
```

ユーザを切り替えます。

```
# su - docker
```

最後にバージョン確認です。Server: のバージョンが表示されていれば正常です。表示されない場合は Docker Engine（dockerdデーモン）が起動しているかどうか、root 権限（正確には /var/run/docker.sock のアクセス権限があるかどうか）をご確認ください。

```
$ docker version
Client:
 Version:      17.05.0-ce
 API version:  1.29
 Go version:   go1.7.5
 Git commit:   89658be
 Built:        Thu May  4 22:06:25 2017
 OS/Arch:      linux/amd64

Server:
 Version:      17.05.0-ce
 API version:  1.29 (minimum version 1.12)
 Go version:   go1.7.5
 Git commit:   89658be
 Built:        Thu May  4 22:06:25 2017
 OS/Arch:      linux/amd64
 Experimental: false
```

### Mastodon のセットアップ

GitHub のリポジトリから、ソースコードをダウンロードします。ここでは /home/docker/ ディレクトリ以下での作業を想定していますが、任意の場所でも実行できます。

```
$ cd
$ git clone  https://github.com/tootsuite/mastodon.git live
$ cd live
$ git checkout $(git tag | tail -n 1)

```

ディレクトリを移動します。

```
$ cd live
```

Docker Compose は、現在の（カレント）ディレクトリ上のプロジェクト・ファイル（デフォルトは docker-compose.yml）を読み込み、一括してイメージの構築・ダウンロード・実行など操作を行えます。

それから、設定ファイル .env.production を作成します（これは Docker Compose 実行時、コンテナ内で有効になる環境変数を書き込むファイルです）。

```
$ cp .env.production.sample .env.production
```

詳細は後ほど編集します。

### mastodon-docker リポジトリから設定ファイルのダウンロード

専用の docker-compose.yml など、関連ファイルをダウンロードします。

```
$ cd
$ git clone https://github.com/zembutsu/mastodon-docker.git
````

### SSL 証明書の作成

Let's encrypt の証明書を発行します。まずはドライ・ランが正常に走るかどうかを確認します。

```
# docker container run -it --rm -v certbot:/etc/letsencrypt \
    -v /var/log/letsencrypt:/var/log/letsencrypt \
    -p 443:443 \
    certbot/certbot \
     certonly --standalone --agree-tos --dry-run -n \
    -d <ここをホスト名> --email <通知用のメールアドレス>
```

特にエラーがなければ、実際に証明書を作成します（--dry-run のオプションを削除し、実際に証明書を作成します）。

```
# docker container run -it --rm -v certbot:/etc/letsencrypt \
    -v /var/log/letsencrypt:/var/log/letsencrypt \
    -p 443:443 \
    certbot/certbot \
     certonly --standalone --agree-tos -n \
    -d <ここをホスト名> --email <通知用のメールアドレス>
```

次に openssl 用イメージを作成します。

```
$ cd ~/mastodon-docker/myopenssl
$ docker image build -t myopenssl - < ./Dockerfile
```

次に鍵交換用のパラメータ・ファイル hdparam.pem を作成します（数分かかります）。

```
$ docker container run -it --rm -v certbot:/certbot myopenssl openssl dhparam -out  /certbot/dhparam.pem 2048
```

### 設定ファイル向けの準備と編集

シークレット（APIトークンに相当）を３つ作成します。次のコマンドを3回実行します（PAPERCLIP_SECRET SECRET_KEY_BASE OTP_SECRET で使います。）。

```
$ docker-compose run --rm web rake secret
<文字列>
```

実行すると <文字列> が出ますので、控えておきます。

それから、 .env.productionファイルを編集します。

10行目以降を編集します。対象は以下の行です。

```
# Federation
LOCAL_DOMAIN=<ホスト名 mastodon.example.jp など>
LOCAL_HTTPS=false

# Application secrets
# Generate each with the `rake secret` task (`docker-compose run --rm web rake secret` if you use docker compose)
PAPERCLIP_SECRET=8979dee487...　←先ほどのシークレット
SECRET_KEY_BASE=74a137afd0b0...　←先ほどのシークレット
OTP_SECRET=aae3d46591fb057...　←先ほどのシークレット
（省略）
# Optionally change default language
DEFAULT_LOCALE=ja
```

ユーザを登録するために、認証のためにメールサーバ用のセットアップもします。同じく .env.production の以下の行を編集します。

```
SMTP_SERVER=mta
SMTP_PORT=25
SMTP_LOGIN=
SMTP_PASSWORD=
SMTP_FROM_ADDRESS=who@example.com ←※要編集（メールの From: アドレスです）
SMTP_DOMAIN=example.com ←※要編集（メールサーバ自身のホスト名、ここでは Mastodon を動かすホスト名を指定）
```

### 設定ファイルのコピーと編集

```
$ cp ~/mastodon-docker/docker-compose.yml ~/live/
$ cp ~/mastodon-docker/Dockerfile ~/live/
$ cp -r ~/mastodon-docker/frontend ~/live/
```

Nginx 用設定ファイルの編集

```
$  vi ~/live/frontend/nginx-default.conf
```

`example.com` を実際に使うホスト名（例：mastodon.example.comなど）に変更します。


メールサーバ用設定ファイルの編集

```
$  vi ~/live/.env.smtp
```

```
MAILNAME=example.com	←ここもホスト名を変更します
RELAY_NETWORKS=192.168.0.0/24
PORT=25
```

### 内部通信用ネットワーク作成

Mastodon のネットワークを作成します。

```
# docker network create --subnet=192.168.0.0/16 mastodon
```


### データベースの初期化とアセットのプリコンパイル


```
$ cd ~/live
$ docker-compose build
$ docker-compose run --rm web rails db:migrate
$ docker-compose run --rm web rails assets:precompile
```

特にエラーがでなければ準備完了です。

```
$ docker-compose up -d
```

docker-compose ps で動作状態を確認します。各サービス（コンテナ）の状態（State)が起動中(Up)なのを確認します。

起動まで時間がかかりますので、ログを参照します。画面のスクロールが止まるまで待ちます。

```
$ docker-compose logs -f
```

止まったら ctrl-fで中断できます。

ブラウザから動作確認です。 https://<ホスト名>/ にアクセスし、画面が表示されるかを確認します。
あとは、ご自由にお使いください。


## Feedbak 

Issue や Pull Request でどうぞ

