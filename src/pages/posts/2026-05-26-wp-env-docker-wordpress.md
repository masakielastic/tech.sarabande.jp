---

layout: ../../layouts/PostLayout.astro
title: "wp-env で Docker ベースの WordPress 開発環境を作る"
description: "Docker 公式リポジトリの導入から wp-env の起動、WordPress の日本語化、ローカルプラグインの読み込み、PHP バージョン切り替え、wp-playground/cli との違いまでを整理する学習記録。"
pubDate: 2026-05-26
tags:
  - WordPress
  - wp-env
  - Docker
  - PHP
  - WordPress開発
---

この記事では、WordPress のローカル開発環境ツールである `wp-env` を使い、Docker 上に WordPress 開発環境を作る手順を整理します。

扱う内容は、Docker の準備、`wp-env` の起動、管理画面へのログイン、日本語化、`.wp-env.json` の作成、ローカルプラグインの読み込み、WordPress.org プラグインの追加、PHP バージョンの切り替えです。

最後に、`wp-playground/cli` との違いや、MySQL / MariaDB のバージョン、Unicode 照合順序、`sql_mode` などを細かく検証したい場合に Docker Compose を使うべき理由も整理します。

## wp-env とは何か

`wp-env` は、WordPress 公式系の npm パッケージである `@wordpress/env` が提供するローカル開発環境ツールです。

簡単に言えば、Docker を使って WordPress 開発環境を立ち上げるためのラッパーです。

通常、WordPress のローカル環境を作るには、PHP、Web サーバー、MySQL / MariaDB、WordPress 本体、WP-CLI などを用意する必要があります。`wp-env` を使うと、そのあたりを Docker コンテナとしてまとめて起動できます。

ただし、重要なのは、`wp-env` は Docker を不要にするツールではないという点です。

`wp-env` は Docker の細かい設定を直接書かなくても WordPress 開発環境を起動できるようにしてくれますが、通常の Docker runtime を使う場合、Docker のセットアップ自体は必要です。

イメージとしては、次のような関係です。

```text
Node.js / npm
  ↓
@wordpress/env
  ↓
Docker
  ↓
WordPress + PHP + MySQL / MariaDB + WP-CLI
```

GUI でサイトを作る Local や MAMP のようなツールというより、テーマやプラグインを Git 管理しながら開発するための、CLI ベースの再現可能な開発環境と考えるとわかりやすいです。

## Docker を用意する

`wp-env` を使うには、まず Docker が動く状態になっている必要があります。

確認するコマンドは次のとおりです。

```bash
docker --version
docker compose version
docker run hello-world
```

`docker run hello-world` が成功すれば、Docker の基本的な動作確認はできています。

Debian や Chromebook の Linux 環境で試す場合、`apt install docker.io` でも動くことはあります。ただし、最新の Docker を使いたい場合は、Docker 公式リポジトリから `docker-ce`、`docker-ce-cli`、`containerd.io`、`docker-buildx-plugin`、`docker-compose-plugin` を入れるほうがよいです。

また、`wp-env` から Docker を呼び出すため、通常は `sudo` なしで `docker` コマンドを実行できる状態にしておくと扱いやすくなります。

```bash
docker run hello-world
```

このコマンドが `sudo` なしで通る状態を目標にします。

## wp-env をインストールする

今回は、プロジェクトごとに `@wordpress/env` をインストールする方法を使います。

```bash
mkdir test-wp-env
cd test-wp-env
npm init -y
npm install -D @wordpress/env
```

グローバルインストールする方法もありますが、プロジェクトごとに入れておくと、将来バージョンを固定しやすくなります。

起動は `npx` 経由で行います。

```bash
npx wp-env start
```

起動に成功すると、WordPress の開発用サイトが立ち上がります。

```text
http://localhost:8888/
```

管理画面は次です。

```text
http://localhost:8888/wp-admin/
```

初期ログイン情報は次のとおりです。

```text
ユーザー名: admin
パスワード: password
```

## 最初に出る警告の意味

設定ファイルを置かずに `npx wp-env start` を実行すると、次のような警告が出ることがあります。

```text
Warning: could not find a .wp-env.json configuration file and could not determine if
'/home/.../test-wp-env' is a WordPress installation, a plugin, or a theme.
```

これは、現在のディレクトリに `.wp-env.json` がないため、`wp-env` がこのディレクトリをどう扱えばよいか判断できない、という意味です。

具体的には、次のどれなのかがわからないということです。

```text
WordPress 本体のディレクトリなのか
プラグイン開発用ディレクトリなのか
テーマ開発用ディレクトリなのか
ただの空ディレクトリなのか
```

また、次のような警告が出ることもあります。

```text
wp-env starts both development and tests environments by default.
This behavior is deprecated and will be removed in a future version.
To avoid this warning, add "testsEnvironment": false to your .wp-env.json.
```

これは、現在の `wp-env` が、開発用環境とテスト用環境を両方起動していることを知らせる警告です。

実際に、次のように表示されることがあります。

```text
WordPress development site started at http://localhost:8888
MySQL is listening on port 32770
WordPress test site started at http://localhost:8889
MySQL for automated testing is listening on port 32771
```

通常の練習やプラグイン開発の最初の確認では、テスト用環境までは不要です。

そこで、`.wp-env.json` を作って、`wp-env` の挙動を明示します。

## .wp-env.json で環境を明示する

プロジェクトルートに `.wp-env.json` を作ります。

```bash
nano .wp-env.json
```

最初は次のような内容で十分です。

```json
{
  "core": null,
  "testsEnvironment": false,
  "config": {
    "WP_DEBUG": true,
    "SCRIPT_DEBUG": true,
    "FS_METHOD": "direct"
  }
}
```

それぞれの意味は次のとおりです。

`core: null` は、通常の WordPress コアを使う指定です。

`testsEnvironment: false` は、テスト用環境を起動しない指定です。警告文にもあるように、この項目自体も将来的には非推奨方向ですが、現時点で通常の練習環境をわかりやすくするには便利です。

`WP_DEBUG: true` は、WordPress のデバッグを有効にします。

`SCRIPT_DEBUG: true` は、圧縮前の JavaScript や CSS を使いやすくする設定です。

`FS_METHOD: direct` は、WordPress が翻訳ファイルやプラグイン関連ファイルを直接扱えるようにするための設定です。ローカル開発環境では入れておくと便利です。

設定を変更したら、いったん停止して、更新付きで起動します。

```bash
npx wp-env stop
npx wp-env start --update
```

これで、`.wp-env.json` に従って環境が起動します。

## WordPress を日本語化する

`wp-env` では、コンテナ内の WP-CLI を使って WordPress の設定を変更できます。

日本語化するには、次のコマンドを実行します。

```bash
npx wp-env run cli wp language core install ja --activate
```

タイムゾーンも日本向けに変更しておきます。

```bash
npx wp-env run cli wp option update timezone_string 'Asia/Tokyo'
```

現在の言語設定を確認するには、次のコマンドを使います。

```bash
npx wp-env run cli wp option get WPLANG
```

`ja` と表示されれば、日本語設定になっています。

注意点として、`npx wp-env stop` でサーバーを停止している状態では、WP-CLI も実行できません。

その状態で次のようなコマンドを実行すると、

```bash
npx wp-env run cli wp option get WPLANG
```

次のようなエラーになります。

```text
service "cli" is not running
```

これは、`wp-env run cli ...` の実行先である `cli` サービスが動いていないという意味です。

対処は単純で、先に `wp-env` を起動します。

```bash
npx wp-env start
```

その後に、WP-CLI コマンドを実行します。

## ローカルプラグインを読み込む

`wp-env` でローカルプラグインを読み込むには、`.wp-env.json` の `plugins` にローカルパスを指定します。

現在のディレクトリ全体をプラグインとして読み込むなら、次のようにします。

```json
{
  "plugins": [
    "."
  ],
  "testsEnvironment": false,
  "config": {
    "WP_DEBUG": true,
    "SCRIPT_DEBUG": true,
    "FS_METHOD": "direct"
  }
}
```

この場合、プロジェクト構成は次のようになります。

```text
test-wp-env/
├─ .wp-env.json
├─ package.json
├─ my-sample-plugin.php
└─ src/
```

`my-sample-plugin.php` には、最低限プラグインヘッダーが必要です。

```php
<?php
/**
 * Plugin Name: My Sample Plugin
 * Description: wp-env で読み込むローカル開発用プラグイン。
 * Version: 0.1.0
 */

add_action('admin_notices', function () {
    echo '<div class="notice notice-success"><p>My Sample Plugin is loaded.</p></div>';
});
```

設定を反映します。

```bash
npx wp-env stop
npx wp-env start --update
```

プラグインが WordPress に認識されているか確認します。

```bash
npx wp-env run cli wp plugin list
```

一覧に表示されていれば、WordPress から見えています。

ただし、読み込まれていることと有効化されていることは別です。有効化するには次のようにします。

```bash
npx wp-env run cli wp plugin activate my-sample-plugin
```

プラグインのスラッグがわからない場合は、先に `wp plugin list` で確認します。

## WordPress.org のプラグインを読み込む

WordPress.org で配布されているプラグインは、個別に ZIP をダウンロードしてローカルに展開しなくても、`.wp-env.json` にスラッグを書くだけで読み込めます。

たとえば Query Monitor と Classic Editor を入れたい場合は、次のようにします。

```json
{
  "plugins": [
    ".",
    "query-monitor",
    "classic-editor"
  ],
  "testsEnvironment": false,
  "config": {
    "WP_DEBUG": true,
    "SCRIPT_DEBUG": true,
    "FS_METHOD": "direct"
  }
}
```

反映します。

```bash
npx wp-env start --update
```

必要に応じて有効化します。

```bash
npx wp-env run cli wp plugin activate query-monitor
npx wp-env run cli wp plugin activate classic-editor
```

確認します。

```bash
npx wp-env run cli wp plugin list
```

整理すると、指定方法の違いは次のようになります。

```text
"."                  現在のローカルディレクトリをプラグインとして読み込む
"./plugins/my-plugin" ローカルの特定ディレクトリをプラグインとして読み込む
"query-monitor"     WordPress.org のプラグインをスラッグで取得する
```

有料プラグイン、WordPress.org にないプラグイン、改造版のプラグインは、ローカルに配置して読み込むことになります。

## PHP バージョンを切り替える

`wp-env` では、`.wp-env.json` に `phpVersion` を指定することで PHP バージョンを切り替えられます。

たとえば PHP 8.3 で起動するなら、次のようにします。

```json
{
  "core": null,
  "phpVersion": "8.3",
  "plugins": [
    ".",
    "query-monitor"
  ],
  "testsEnvironment": false,
  "config": {
    "WP_DEBUG": true,
    "SCRIPT_DEBUG": true,
    "FS_METHOD": "direct"
  }
}
```

変更後は、停止して更新付きで起動します。

```bash
npx wp-env stop
npx wp-env start --update
```

PHP バージョンを確認します。

```bash
npx wp-env run cli php -v
```

または、WordPress 側の情報として確認します。

```bash
npx wp-env run cli wp --info
```

ここで注意したいのは、`phpVersion` で切り替えているのは、`wp-env` の Docker コンテナ内の PHP です。

ホスト OS 側の PHP とは別物です。

つまり、次のコマンドで見える PHP と、

```bash
php -v
```

次のコマンドで見える PHP は別です。

```bash
npx wp-env run cli php -v
```

WordPress のプラグインやテーマの互換性を見る場合は、後者を確認します。

## wp-playground/cli との違い

`wp-env` と近い文脈で、`wp-playground/cli` もあります。

ざっくり分けると、`wp-env` は Docker ベースの WordPress 開発環境であり、`wp-playground/cli` は WordPress Playground 系の軽量な実行環境です。

`wp-env` の Docker runtime では、WordPress、PHP、MySQL / MariaDB、WP-CLI などを含む、実サーバーに近い環境を扱えます。

一方、`wp-playground/cli` や Playground runtime は、Docker なしで WordPress を軽く試しやすいのが強みです。ブラウザや WebAssembly、Blueprint との相性もよく、チュートリアルや軽い検証には向いています。

最近の `wp-env` には、Playground runtime も追加されています。

```bash
npx wp-env start --runtime=playground
```

これにより、`wp-env` のコマンド体系から、Docker runtime と Playground runtime を使い分ける方向が見えてきました。

ただし、両者は完全な代替ではありません。

整理すると、次のようになります。

```text
wp-env Docker runtime:
  MySQL / MariaDB を含む
  PHP バージョン切り替えに向く
  ローカルプラグイン開発に向く
  WP-CLI を使った確認に向く
  実サーバー寄りの検証に向く

wp-env Playground runtime / wp-playground/cli:
  軽量
  Docker 不要
  SQLite ベース
  ブラウザ・WASM・Blueprint との相性がよい
  DB 実装差の検証には向きにくい
```

プラグインやテーマを軽く試すだけなら Playground 系も便利です。

しかし、MySQL / MariaDB に依存する挙動を見たい場合は、Docker runtime のほうが向いています。

## DB バージョンや照合順序を調べるなら Docker Compose を使う

今回、`wp-env` と `wp-playground/cli` の違いを考えるうえで重要だと思ったのが、データベースの扱いです。

`wp-env` の Docker runtime は MySQL / MariaDB を含む環境を起動できます。そのため、Playground 系よりも実サーバーに近い検証ができます。

ただし、PHP バージョンのように、`.wp-env.json` だけで MySQL / MariaDB のバージョンを細かく切り替える用途には向いていません。

たとえば、次のようなことを検証したい場合です。

```text
MySQL 8.0 と MySQL 8.4 の違い
MariaDB 10.11 と MariaDB 11.x の違い
utf8mb4_unicode_ci の挙動
utf8mb4_unicode_520_ci の挙動
utf8mb4_0900_ai_ci の挙動
utf8mb4_bin の挙動
sql_mode の違い
my.cnf の設定差
```

このようなテーマでは、`wp-env` よりも Docker Compose を直接書くほうが向いています。

たとえば、Docker Compose なら次のように DB イメージを明示できます。

```yaml
services:
  db:
    image: mysql:8.0
```

MariaDB にしたいなら、次のようにできます。

```yaml
services:
  db:
    image: mariadb:10.11
```

さらに、DB 設定ファイルをマウントできます。

```yaml
services:
  db:
    image: mysql:8.0
    volumes:
      - ./mysql/conf.d:/etc/mysql/conf.d
```

たとえば `./mysql/conf.d/custom.cnf` に次のような設定を書けます。

```ini
[mysqld]
character-set-server=utf8mb4
collation-server=utf8mb4_0900_ai_ci
sql_mode=STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
```

Unicode 照合順序や MySQL / MariaDB の差分は、PHP のコードだけでは再現できません。DB サーバー側の実装や設定が効いてきます。

そのため、次のように使い分けるのがよさそうです。

```text
通常の WordPress プラグイン・テーマ開発:
  wp-env

PHP バージョンを切り替えた互換性確認:
  wp-env

WordPress.org プラグインを含む再現可能な検証環境:
  wp-env

MySQL / MariaDB のバージョン差を検証:
  Docker Compose

Unicode 照合順序や sql_mode の検証:
  Docker Compose

Docker を使いたくない常用 DB 環境:
  OS に MySQL / MariaDB を直接インストール
```

## Podman について

Docker の代替として Podman を使いたい場合もあります。

ただし、`wp-env` は基本的に Docker / Docker Compose 前提のツールです。Podman は Docker 互換 CLI を持っていますが、`wp-env` の内部で使われる Compose、ボリューム、ネットワーク、CLI コンテナ、テスト環境などがすべて同じように動くとは限りません。

そのため、`wp-env` で Podman を公式に安定サポートされる前提として使うのは避けたほうがよさそうです。

Podman を使いたい場合は、`wp-env` に寄せるより、Podman / podman-compose 用に WordPress 構成を自分で書くほうがわかりやすいと思います。

## まとめ

`wp-env` は、WordPress のテーマ・プラグイン開発を始めるための Docker ベースの開発環境として便利です。

今回確認したことを整理すると、次のようになります。

```text
wp-env は Docker ベースの WordPress 開発環境ツール
通常利用では Docker のセットアップが必要
管理画面の初期ログインは admin / password
.wp-env.json を書くと環境の用途を明示できる
WP-CLI で日本語化やタイムゾーン変更ができる
サーバー停止中は wp-env run cli は実行できない
ローカルプラグインは plugins に "." を指定して読み込める
WordPress.org プラグインはスラッグ指定で取得できる
PHP バージョンは phpVersion で切り替えられる
wp-playground/cli は軽量な Playground 系環境として使い分ける
DB バージョンや照合順序の厳密な検証は Docker Compose が向いている
```

自分の使い分けとしては、通常の WordPress プラグイン・テーマ開発は `wp-env`、軽い WordPress の機能確認や Playground 連携は `wp-playground/cli`、MySQL / MariaDB の実装差や Unicode 照合順序の検証は Docker Compose、という整理がよさそうです。

まずは `wp-env` で WordPress 開発環境をすぐ立ち上げられるようにしておくと、テーマやプラグインの検証、WordPress 本体の基本機能調査、PHP バージョン違いの確認がかなり進めやすくなります。
