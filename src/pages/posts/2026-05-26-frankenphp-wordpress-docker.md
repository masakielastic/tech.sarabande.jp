---

layout: ../../layouts/PostLayout.astro
title: "FrankenPHP と Docker Compose で WordPress を動かし、日本語化とファイル更新まで確認する"
description: "FrankenPHP と MariaDB を Docker Compose で組み合わせ、WordPress をローカル環境で起動する手順を整理する。Caddyfile の配置先、HTTPS リダイレクト、日本語パック、FS_METHOD direct で詰まった点も記録する。"
pubDate: 2026-05-26
tags:
  - WordPress
  - FrankenPHP
  - Docker
  - Docker Compose
  - Caddy
  - MariaDB

---

# FrankenPHP と Docker Compose で WordPress を動かし、日本語化とファイル更新まで確認する

## はじめに

FrankenPHP で WordPress を動かしてみました。

FrankenPHP は Caddy をベースにした PHP アプリケーションサーバーです。WordPress では Apache や nginx + PHP-FPM の構成がよく使われますが、今回は Docker Compose を使って、FrankenPHP + MariaDB の構成で WordPress をローカル環境に立ち上げます。

この記事では、単に WordPress の初期画面を表示するところまでではなく、次のところまで確認します。

* FrankenPHP と MariaDB を Docker Compose で起動する
* WordPress の初期インストールを完了する
* Caddyfile の配置先を正しく設定する
* HTTP から HTTPS にリダイレクトされる問題を避ける
* 日本語パックをインストールできるようにする
* `FS_METHOD` を `direct` にして、WordPress からファイル更新できるようにする

実際に試してみると、FrankenPHP そのものよりも、WordPress がファイルを書き込むための権限や `wp-config.php` の設定で詰まりやすいことがわかりました。

## 今回作る構成

今回の構成は次のとおりです。

```text
ブラウザー
  ↓
FrankenPHP / Caddy
  ↓
WordPress
  ↓
MariaDB
```

ローカル環境では、ホスト側の `8081` 番ポートをコンテナ内の `80` 番ポートに転送します。

```text
http://localhost:8081
```

今回は HTTPS ではなく、ローカル検証用に HTTP で動かします。

## ディレクトリ構成

作業ディレクトリを作ります。

```bash
mkdir -p ~/playground/frankenphp-wordpress
cd ~/playground/frankenphp-wordpress
```

最終的なファイル構成は次のようにします。

```text
frankenphp-wordpress/
  compose.yml
  Dockerfile
  Caddyfile
  php.ini
```

## compose.yml を用意する

まず `compose.yml` を作成します。

```yaml
services:
  wordpress:
    build: .
    ports:
      - "8081:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
    volumes:
      - wordpress_data:/app/public
    depends_on:
      - db

  db:
    image: mariadb:11.4
    environment:
      MARIADB_DATABASE: wordpress
      MARIADB_USER: wordpress
      MARIADB_PASSWORD: wordpress
      MARIADB_ROOT_PASSWORD: root
    volumes:
      - db_data:/var/lib/mysql

volumes:
  wordpress_data:
  db_data:
```

`wordpress` サービスでは FrankenPHP のイメージを自分でビルドします。

`db` サービスでは MariaDB を使います。WordPress からは、Compose のサービス名である `db` をデータベースホスト名として指定できます。

## Dockerfile を用意する

次に `Dockerfile` を作成します。

```dockerfile
FROM wordpress:latest AS wp

FROM dunglas/frankenphp:1-php8.3

RUN apt-get update && apt-get install -y \
    curl \
    ca-certificates \
    unzip \
    && rm -rf /var/lib/apt/lists/*

RUN install-php-extensions \
    mysqli \
    pdo_mysql \
    gd \
    intl \
    zip \
    exif \
    opcache

COPY --from=wp --chown=www-data:www-data /usr/src/wordpress/ /app/public/
COPY Caddyfile /etc/frankenphp/Caddyfile
COPY php.ini /usr/local/etc/php/conf.d/wordpress.ini

RUN mkdir -p /app/public/wp-content/languages \
    && chown -R www-data:www-data /app/public/wp-content

WORKDIR /app/public
```

ここでは、公式の `wordpress:latest` イメージから WordPress 本体を取り出し、FrankenPHP のイメージへコピーしています。

重要なのはこの行です。

```dockerfile
COPY --from=wp --chown=www-data:www-data /usr/src/wordpress/ /app/public/
```

WordPress のファイルを `www-data` 所有でコピーします。これをしないと、あとで日本語パックやプラグインをインストールするときに、WordPress がファイルを書き込めずに失敗することがあります。

また、WordPress 用に最低限必要な PHP 拡張も追加します。

```dockerfile
RUN install-php-extensions \
    mysqli \
    pdo_mysql \
    gd \
    intl \
    zip \
    exif \
    opcache
```

`mysqli` は WordPress で必要です。`gd`, `intl`, `zip`, `exif`, `opcache` も、ローカル開発で入れておくと扱いやすい拡張です。

## Caddyfile を用意する

次に `Caddyfile` を作成します。

```caddyfile
{
    auto_https off
}

http://:80 {
    root * /app/public
    encode zstd gzip
    php_server
}
```

ここで重要なのは、FrankenPHP 公式イメージでは Caddyfile の配置先が次であることです。

```text
/etc/frankenphp/Caddyfile
```

最初は `/etc/caddy/Caddyfile` にコピーしていたのですが、それでは設定が反映されませんでした。

ログを見ると、FrankenPHP は次のファイルを読んでいました。

```text
/etc/frankenphp/Caddyfile
```

そのため、Dockerfile では次のようにコピーします。

```dockerfile
COPY Caddyfile /etc/frankenphp/Caddyfile
```

また、ローカル検証では HTTPS 自動化を避けるために、グローバルオプションで `auto_https off` を指定し、サイトアドレスも `http://:80` と明示しています。

## php.ini を用意する

次に `php.ini` を作成します。

```ini
memory_limit = 256M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 120

opcache.enable=1
opcache.memory_consumption=128
opcache.max_accelerated_files=10000
opcache.validate_timestamps=1
```

ローカル検証用なので、アップロードサイズやメモリ上限を少し余裕のある値にしています。

## 起動する

ここまで用意できたら、ビルドして起動します。

```bash
sudo docker compose up -d --build
```

起動状態を確認します。

```bash
sudo docker ps
```

WordPress 側のコンテナに、次のようなポート表示が出ていればよいです。

```text
0.0.0.0:8081->80/tcp
```

MariaDB だけが表示され、WordPress 側のコンテナが表示されない場合は、FrankenPHP 側のコンテナが起動直後に終了しています。

その場合はログを確認します。

```bash
sudo docker compose logs wordpress
```

または次のように確認します。

```bash
sudo docker logs frankenphp-wordpress-wordpress-1
```

## ブラウザーでアクセスする

ブラウザーで次にアクセスします。

```text
http://localhost:8081
```

ここで注意するのは、`https://localhost:8081` ではなく、`http://localhost:8081` でアクセスすることです。

もし HTTPS でアクセスすると、HTTP 用のポートに TLS 接続しようとして、次のようなエラーになることがあります。

```text
SSL routines::wrong version number
```

これは WordPress や MariaDB の問題ではなく、HTTP のポートに HTTPS でアクセスしていることが原因です。

## HTTPS にリダイレクトされる場合

`http://localhost:8081` にアクセスしているのに、次のようなレスポンスが返ることがあります。

```bash
curl -I http://localhost:8081
```

```text
HTTP/1.1 308 Permanent Redirect
Location: https://localhost/
Server: FrankenPHP Caddy
```

この場合、Caddy が HTTP から HTTPS へ自動リダイレクトしています。

原因としては、Caddyfile が正しく反映されていない可能性が高いです。

コンテナ内の Caddyfile を確認します。

```bash
sudo docker exec -it frankenphp-wordpress-wordpress-1 cat /etc/frankenphp/Caddyfile
```

次の内容になっているか確認します。

```caddyfile
{
    auto_https off
}

http://:80 {
    root * /app/public
    encode zstd gzip
    php_server
}
```

また、ログも確認します。

```bash
sudo docker logs frankenphp-wordpress-wordpress-1
```

もしログに次のような表示があれば、Caddy の自動 HTTPS が有効になっています。

```text
enabling automatic HTTP->HTTPS redirects
```

この場合は、Dockerfile のコピー先が `/etc/frankenphp/Caddyfile` になっているか、Caddyfile の内容が `http://:80` になっているかを確認します。

## WordPress をインストールする

ブラウザーで `http://localhost:8081` にアクセスすると、WordPress の初期インストール画面が表示されます。

データベース接続情報を求められた場合は、次のように入力します。

```text
データベース名: wordpress
ユーザー名: wordpress
パスワード: wordpress
データベースのホスト名: db
テーブル接頭辞: wp_
```

`db` は `compose.yml` で定義した MariaDB サービス名です。Docker Compose のネットワーク内では、このサービス名をホスト名として使えます。

その後、サイトタイトル、管理者ユーザー名、パスワード、メールアドレスを入力して、WordPress のインストールを完了します。

## 日本語化する

WordPress の管理画面にログインします。

```text
http://localhost:8081/wp-admin/
```

英語表示の場合は、次に進みます。

```text
Settings → General → Site Language
```

`日本語` を選んで保存します。

ただし、ここで日本語パックのインストールに失敗することがあります。

私の環境でも、最初は日本語パックがうまく入りませんでした。

手動で日本語パックをダウンロードして配置すれば回避できますが、世界中の WordPress ユーザーが毎回その作業をしているとは考えにくいです。

本質的には、WordPress が `wp-content` 配下に直接ファイルを書き込めるようにする必要があります。

## FS_METHOD direct を追加する

WordPress がテーマ、プラグイン、翻訳ファイルなどを書き込むとき、内部では `WP_Filesystem` という仕組みを使います。

Docker の自作構成では、WordPress が「直接ファイルを書き込める」と判断できず、翻訳ファイルのインストールに失敗することがあります。

そこで、`wp-config.php` に次を追加します。

```php
define( 'FS_METHOD', 'direct' );
```

追加位置は、次の行より前です。

```php
require_once ABSPATH . 'wp-settings.php';
```

手作業で編集してもよいですが、コマンドで追加する場合は次のようにします。

```bash
sudo docker exec -it frankenphp-wordpress-wordpress-1 sh -lc '
cp /app/public/wp-config.php /app/public/wp-config.php.bak &&
grep -q "FS_METHOD" /app/public/wp-config.php || \
sed -i "/\/\* That'\''s all, stop editing! Happy publishing\. \*\//i define( '\''FS_METHOD'\'', '\''direct'\'' );" /app/public/wp-config.php
'
```

確認します。

```bash
sudo docker exec -it frankenphp-wordpress-wordpress-1 grep FS_METHOD /app/public/wp-config.php
```

次のように表示されれば成功です。

```php
define( 'FS_METHOD', 'direct' );
```

## wp-content の権限も確認する

`FS_METHOD` を `direct` にしても、実際のファイル権限で書き込めなければ失敗します。

念のため、`wp-content` の所有者を確認します。

```bash
sudo docker exec -it frankenphp-wordpress-wordpress-1 ls -ld /app/public/wp-content /app/public/wp-content/languages
```

必要であれば、次のように所有者を変更します。

```bash
sudo docker exec -it frankenphp-wordpress-wordpress-1 chown -R www-data:www-data /app/public/wp-content
```

今回の Dockerfile では、あらかじめ次の処理を入れています。

```dockerfile
RUN mkdir -p /app/public/wp-content/languages \
    && chown -R www-data:www-data /app/public/wp-content
```

さらに WordPress 本体をコピーするときも `--chown=www-data:www-data` を指定しています。

```dockerfile
COPY --from=wp --chown=www-data:www-data /usr/src/wordpress/ /app/public/
```

これにより、管理画面から日本語パック、テーマ、プラグインをインストールしやすくなります。

## 日本語パックのインストールを再試行する

`FS_METHOD` を追加したあと、管理画面で再度言語を変更します。

```text
Settings → General → Site Language → 日本語
```

保存すると、日本語パックが自動でダウンロードされ、管理画面が日本語になります。

日本語化後は、次のような設定も確認しておくとよいです。

```text
設定 → 一般
```

* サイトの言語: 日本語
* タイムゾーン: 東京
* 日付形式: Y年n月j日
* 時刻形式: H:i
* 週の始まり: 月曜日

## 完全にやり直す方法

検証中に状態がわからなくなった場合は、ボリュームごと削除してやり直せます。

```bash
sudo docker compose down -v
sudo docker compose up -d --build
```

`-v` を付けると、Compose が作成した volume も削除されます。

つまり、WordPress のファイルと MariaDB のデータも消えます。

投稿や設定を残したい場合は、`-v` を付けないようにします。

```bash
sudo docker compose down
```

完全に初期状態から検証したいときだけ、`down -v` を使います。

## Docker の permission denied が出る場合

最初に次のようなエラーが出ることがあります。

```text
permission denied while trying to connect to the docker API at unix:///var/run/docker.sock
```

これは MariaDB イメージの問題ではなく、現在のユーザーが Docker デーモンに接続する権限を持っていないという意味です。

一時的には `sudo` を付ければ実行できます。

```bash
sudo docker ps
sudo docker compose up -d --build
```

毎回 `sudo` を付けたくない場合は、ユーザーを `docker` グループに追加します。

```bash
sudo usermod -aG docker "$USER"
```

その後、ログアウトしてログインし直します。Chromebook の Crostini 環境では、Linux ターミナルを開き直すか、Linux 環境を再起動したほうが確実です。

## wp-env との違い

WordPress のローカル開発環境としては、`wp-env` もよく使われます。

`wp-env` は WordPress 開発用に整えられた環境です。プラグインやテーマの開発をすぐ始めたい場合は、`wp-env` のほうが簡単です。

一方で、FrankenPHP 構成は WordPress 専用の開発ツールではありません。

FrankenPHP という PHP アプリケーションサーバーの上で、WordPress を動かしてみる検証です。

そのため、次のような点を自分で意識する必要があります。

* Caddyfile の配置先
* HTTP / HTTPS の扱い
* PHP 拡張
* ファイル所有者
* `wp-config.php` の `FS_METHOD`
* WordPress が翻訳・テーマ・プラグインを書き込めるか

`wp-env` は WordPress 開発を簡単に始めるための道具であり、FrankenPHP 構成は PHP サーバー構成そのものを学ぶための検証環境だと考えるとわかりやすいです。

## 今回つまずいた点

今回の検証でつまずいた点を整理すると、次のようになります。

* Docker API の `permission denied` は Docker ソケットの権限問題だった
* `https://localhost:8080` にアクセスすると、HTTP ポートでは TLS エラーになる
* Caddyfile の配置先は `/etc/caddy/Caddyfile` ではなく `/etc/frankenphp/Caddyfile` だった
* Caddyfile が反映されていないと、Caddy が HTTP から HTTPS へ自動リダイレクトする
* WordPress の日本語パックが入らない原因は、翻訳ファイルのダウンロードそのものではなく、ファイル更新方法や権限だった
* `wp-config.php` に `define( 'FS_METHOD', 'direct' );` を追加すると、日本語パックのインストールが通った

特に重要だったのは、最後の `FS_METHOD` です。

手動で日本語パックをダウンロードして配置することもできますが、それは回避策です。

WordPress らしく管理画面から翻訳・テーマ・プラグインを入れるには、WordPress がファイルを書き込める状態を作る必要があります。

## まとめ

FrankenPHP で WordPress を動かすこと自体は、Docker Compose を使えば比較的簡単です。

ただし、WordPress 公式 Docker イメージをそのまま使う場合と違い、FrankenPHP イメージに WordPress のファイルをコピーして動かす場合は、細かい設定を自分で補う必要があります。

今回のポイントは次の3つです。

* Caddyfile は `/etc/frankenphp/Caddyfile` に置く
* ローカル検証では `http://:80` と `auto_https off` を使う
* `wp-config.php` に `define( 'FS_METHOD', 'direct' );` を追加する

ここまで設定すれば、FrankenPHP 上でも WordPress の初期インストール、日本語化、テーマ・プラグイン更新まで確認できます。

`wp-env` のような WordPress 専用ツールとは違い、少し手間はかかります。

そのかわり、Caddy、FrankenPHP、Docker Compose、WordPress のファイル更新の仕組みをまとめて理解するには、よい題材だと感じました。
