---

layout: ../../layouts/PostLayout.astro
title: "久しぶりに WordPress を Debian に入れて、WP-CLI・wp server・nghttpx で HTTPS ローカル環境を作った"
description: "Debian 上で WordPress を最小構成で検証するために、WP-CLI、MariaDB、wp server、nghttpx による TLS 終端を試した学習ログです。"
pubDate: 2026-05-25
tags:
  - WordPress
  - Debian
  - WP-CLI
  - MariaDB
  - nghttpx
  - HTTPS
---

## はじめに

久しぶりに、Debian 上で WordPress をインストールしてみました。

昔の WordPress は、Apache、PHP、MySQL、`style.css`、PHP テンプレート、クラシックテーマという印象が強かったのですが、現在の WordPress はかなり変わっていました。

今回は本番用の構築ではなく、ローカル学習用として次の構成を試しました。

```text
Browser
  ↓ HTTPS / HTTP/2
nghttpx :8443
  ↓ HTTP
wp server :8080
  ↓
WordPress
  ↓
MariaDB
```

Apache や PHP-FPM は使わず、WP-CLI と `wp server` を中心にして、最後に nghttpx で TLS 終端を試しています。

この記事は、WordPress の本番運用手順ではなく、現在の WordPress を Debian 上で軽く触るための学習ログです。

## 今回作った構成

今回の構成は次のとおりです。

```text
Debian
PHP CLI
MariaDB
WP-CLI
wp server
nghttpx
自己署名証明書
```

ポイントは、Web サーバーとして Apache を立てなかったことです。

通常の WordPress 構築では、Apache + mod_php、または Apache / Nginx + PHP-FPM のような構成を使うことが多いと思います。

しかし、ローカル検証であれば、WP-CLI の `wp server` で WordPress を起動できます。`wp server` は PHP のビルトインサーバーを使うため、本番用ではありませんが、WordPress の動作確認やテーマ・プラグインの軽い検証には便利です。

今回はさらに、前段に nghttpx を置き、HTTPS / HTTP/2 でアクセスできるようにしました。

## 必要なパッケージを入れる

Debian に必要なパッケージを入れます。

```bash
sudo apt update
sudo apt install -y \
  mariadb-server \
  php-cli \
  php-mysql \
  php-curl \
  php-gd \
  php-mbstring \
  php-xml \
  php-zip \
  unzip \
  curl \
  openssl \
  nghttp2-proxy
```

Debian では、`nghttpx` コマンドは `nghttp2-proxy` パッケージに含まれます。

確認します。

```bash
nghttpx --version
```

## WP-CLI をインストールする

WP-CLI は PHAR 版を直接入れました。

```bash
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
php wp-cli.phar --info
chmod +x wp-cli.phar
sudo mv wp-cli.phar /usr/local/bin/wp
wp --info
```

これで `wp` コマンドが使えるようになります。

## MariaDB に WordPress 用データベースを作る

MariaDB を起動します。

```bash
sudo systemctl enable --now mariadb
sudo mariadb
```

WordPress 用のデータベースとユーザーを作ります。

```sql
CREATE DATABASE wordpress
  DEFAULT CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER 'wpuser'@'localhost' IDENTIFIED BY '強いパスワード';

GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';

FLUSH PRIVILEGES;
EXIT;
```

MariaDB の新しいバージョンでは、`utf8mb4_uca1400_ai_ci` などの新しい照合順序も使えるようになっています。

ただし、WordPress の互換性や、別の MySQL / MariaDB 環境へ移す可能性を考えると、今回は無難に `utf8mb4_unicode_ci` を指定しました。

ローカル検証だけなら新しい照合順序を試してもよいのですが、WordPress の標準的な運用や移植性を考えるなら、まずは `utf8mb4` + `utf8mb4_unicode_ci` を基準にするのが安全だと思います。

## wp core download で WordPress 本体を取得する

最初に少し勘違いしたのですが、WP-CLI で設定を作る前に、まず WordPress 本体をダウンロードする必要があります。

空のディレクトリでいきなり `wp config create` を実行しても、WordPress 本体がないので進められません。

今回は次のディレクトリを使いました。

```bash
mkdir -p ~/playground/wordpress
cd ~/playground/wordpress
```

WordPress 本体を取得します。

```bash
wp core download --locale=ja
```

これで、現在のディレクトリに WordPress のソースコードが展開されます。

## wp config create と wp core install で初期設定する

次に `wp-config.php` を作ります。

```bash
wp config create \
  --dbname=wordpress \
  --dbuser=wpuser \
  --dbpass='強いパスワード' \
  --dbhost=localhost \
  --dbcharset=utf8mb4 \
  --dbcollate=utf8mb4_unicode_ci
```

今回は最終的に nghttpx 経由の HTTPS URL でアクセスするため、インストール時の URL は `https://localhost:8443` にしました。

```bash
wp core install \
  --url="https://localhost:8443" \
  --title="テスト" \
  --admin_user="admin" \
  --admin_password="管理者用の強いパスワード" \
  --admin_email="you@example.com"
```

すでに別の URL でインストールしてしまった場合は、あとから `home` と `siteurl` を変更できます。

```bash
wp option update home "https://localhost:8443"
wp option update siteurl "https://localhost:8443"
```

確認します。

```bash
wp option get home
wp option get siteurl
```

期待する結果は次です。

```text
https://localhost:8443
https://localhost:8443
```

## wp server で WordPress を起動する

WordPress ルートで `wp server` を起動します。

```bash
cd ~/playground/wordpress
wp server --host=127.0.0.1 --port=8080
```

ここでは、バックエンドの WordPress は `127.0.0.1:8080` で待ち受けます。

外部に直接公開するためではなく、nghttpx から中継する前提なので、`0.0.0.0` ではなく `127.0.0.1` にしました。

`wp server` は開発用です。本番公開には使いません。

ただ、今回のようなローカル検証では、Apache や PHP-FPM を立てなくても WordPress を起動できるので便利です。

## 自己署名証明書を作る

nghttpx で HTTPS を受けるため、ローカル用の自己署名証明書を作ります。

```bash
cd ~/playground/wordpress

openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout localhost.key \
  -out localhost.crt \
  -days 365 \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"
```

生成されるファイルは次です。

```text
localhost.key
localhost.crt
```

自己署名証明書なので、Chrome では「保護されていない通信」と表示されます。

今回は公開用ではなくローカル検証なので、その前提で進めました。

## nghttpx で HTTPS 終端する

別ターミナルで nghttpx を起動します。

```bash
cd ~/playground/wordpress

nghttpx \
  --conf=/dev/null \
  --frontend=127.0.0.1,8443 \
  --backend=127.0.0.1,8080 \
  --errorlog-file=- \
  --accesslog-file=- \
  ./localhost.key \
  ./localhost.crt
```

これで、ブラウザーから次の URL にアクセスします。

```text
https://localhost:8443
```

構成としては、次のようになります。

```text
Browser
  ↓ https://localhost:8443
nghttpx
  ↓ http://127.0.0.1:8080
wp server
```

### /etc/nghttpx/nghttpx.conf を読みに行った

最初に nghttpx を直接起動しようとしたところ、次のような表示が出ました。

```text
Loading configuration from /etc/nghttpx/nghttpx.conf
```

検証用にコマンドライン指定だけで動かしたかったので、`--conf=/dev/null` を付けました。

これでシステム側の `/etc/nghttpx/nghttpx.conf` を読まず、今回のコマンドライン引数だけで起動できます。

### X-Forwarded-Proto について

nghttpx には `--add-x-forwarded-proto` というオプションがあるのかと思いましたが、手元の環境では存在しませんでした。

```text
nghttpx: unrecognized option '--add-x-forwarded-proto'

Did you mean:
        --no-add-x-forwarded-proto
```

つまり、この環境では `X-Forwarded-Proto` の追加は標準で有効で、無効化するオプションとして `--no-add-x-forwarded-proto` があるようです。

必要に応じて、`wp-config.php` に次のような HTTPS 判定を追加できます。

```php
if (
    isset( $_SERVER['HTTP_X_FORWARDED_PROTO'] )
    && 'https' === $_SERVER['HTTP_X_FORWARDED_PROTO']
) {
    $_SERVER['HTTPS'] = 'on';
}
```

追加する場合は、`wp-config.php` の `/* That's all, stop editing! */` より上に書きます。

今回の検証では、まず `home` と `siteurl` を `https://localhost:8443` にそろえることのほうが重要でした。

## curl で HTTPS / HTTP2 の動作を確認する

nghttpx 経由で応答が返っているか確認します。

```bash
curl -kI https://localhost:8443/
```

テーマ CSS も確認しました。

```bash
curl -kI https://localhost:8443/wp-content/themes/twentytwentyfive/style.min.css
```

結果は次のようになりました。

```text
HTTP/2 200
date: Mon, 25 May 2026 09:46:00 GMT
content-type: text/css; charset=UTF-8
content-length: 611
server: nghttpx
via: 1.1 nghttpx
```

`HTTP/2 200` で返っており、`server: nghttpx` と `via: 1.1 nghttpx` も確認できました。

## CSS が効いていないと思ったら、Twenty Twenty-Five がミニマルだった

途中で、CSS が反映されていないのではないかと勘違いしました。

画面がとてもシンプルだったため、昔の WordPress テーマの感覚で見ると、スタイルが当たっていないように見えたのです。

そこで、まず WordPress の URL 設定を確認しました。

```bash
wp option get home
wp option get siteurl
```

結果は次のとおりです。

```text
https://localhost:8443
https://localhost:8443
```

次に、HTML 内の CSS URL を確認しました。

```bash
curl -kL https://localhost:8443/ -o /tmp/wp.html
grep -Eo 'https?://[^"]+\.css[^"]*|/[^"]+\.css[^"]*' /tmp/wp.html | head -30
```

すると、次のように `https://localhost:8443` の CSS が出ていました。

```text
/*# sourceURL=https://localhost:8443/wp-includes/blocks/site-title/style.min.css */
/*# sourceURL=https://localhost:8443/wp-includes/blocks/page-list/style.min.css */
/*# sourceURL=https://localhost:8443/wp-includes/blocks/navigation/style.min.css */
/*# sourceURL=https://localhost:8443/wp-includes/blocks/group/style.min.css */
/*# sourceURL=https://localhost:8443/wp-content/themes/twentytwentyfive/style.min.css */
```

テーマ CSS も `HTTP/2 200` で返っていました。

つまり、CSS が読めていないわけではありませんでした。

Twenty Twenty-Five の初期表示がかなりミニマルで、余白やフォント、ブロック単位の CSS によって構成されているだけでした。

最近の WordPress ブロックテーマでは、昔のように巨大な `style.css` だけで見た目を作るわけではありません。

見た目は、次のような複数の仕組みで構成されます。

```text
ブロックテーマ
theme.json
グローバルスタイル
ブロック単位の CSS
インライン CSS
```

この変化を知らずに見ると、シンプルすぎて CSS が効いていないように見えてしまいます。

## 久しぶりの WordPress との違い

最後に WordPress を本格的に触ったのはかなり前だったので、いろいろな部分が変わっていました。

昔の感覚では、WordPress のテーマといえば次のようなものでした。

```text
クラシックテーマ
style.css
PHP テンプレート
ウィジェット
メニュー
TinyMCE エディター
```

現在の WordPress では、次の要素が中心になっています。

```text
ブロックエディター
ブロックテーマ
theme.json
サイトエディター
グローバルスタイル
ブロック単位の CSS
```

もちろん WordPress は今でも WordPress です。

しかし、テーマ開発や表示の仕組みはかなり変わっています。

特に Twenty Twenty-Five のような標準ブロックテーマを見ると、昔の `style.css` 中心の感覚だけでは理解しにくい部分があります。

## 今回の到達点

今回確認できたことは次のとおりです。

```text
Debian に WP-CLI を導入できた
wp core download で WordPress 本体を取得できた
MariaDB に WordPress 用 DB を作成できた
wp config create / wp core install で初期化できた
wp server で WordPress を起動できた
nghttpx で HTTPS / HTTP2 化できた
ログイン画面と管理画面に入れた
CSS 未反映に見えた問題は勘違いだった
```

ローカル検証用としては、かなり軽い構成で WordPress を触れることがわかりました。

Apache や PHP-FPM を使った本番に近い構成は、必要になってから確認すればよさそうです。

## 次に試したいこと

次は、10年前の WordPress との違いをもう少し整理したいです。

具体的には、次のあたりを確認したいと思います。

```text
クラシックテーマとブロックテーマの違い
theme.json の基本
Twenty Twenty-Five の構造
最小ブロックテーマの作成
プラグイン開発の入口
Apache + PHP-FPM 構成での動作確認
Cloudflare Tunnel / Access との組み合わせ
```

今回は、WordPress のインストールそのものよりも、現在の WordPress の入口を確認する作業になりました。

久しぶりの感覚で見ると戸惑う部分もありましたが、WP-CLI と `wp server` を使えば、ローカルでの再入門はかなり楽に始められそうです。
