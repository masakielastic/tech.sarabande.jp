---

layout: ../../layouts/PostLayout.astro
title: "@wp-playground/cli で WordPress の調査・テーマ開発・プラグイン開発を軽く始める"
description: "WordPress Playground CLI を使い、Docker なしで WordPress を起動し、日本語化、PHP バージョン指定、intl 拡張、テーマ・プラグインのロードを試す学習記録。"
pubDate: 2026-05-26
tags:
  - WordPress
  - WordPress Playground
  - PHP
  - intl
  - Theme Development
  - Plugin Development

---

WordPress の機能調査や、テーマ・プラグイン開発のために、軽く使えるローカル環境がほしくなることがあります。

本番に近い環境を作るなら Docker Compose や `wp-env` は有力です。MySQL や Web サーバーの挙動まで含めて確認したい場合は、そのほうが向いています。

一方で、最新の WordPress の基本機能を少し確認したい、テーマを読み込んで見た目を試したい、小さなプラグインをロードして動作確認したい、という段階では、もう少し軽い環境があると便利です。

そこで今回は、`@wp-playground/cli` を試しました。

この記事では、WordPress Playground CLI を使って、次のことを確認します。

* Docker なしで WordPress を起動する
* 日本語環境で起動する
* WordPress と PHP のバージョンを指定する
* `intl` 拡張を有効にする
* プラグインをロードする
* テーマをロードする

細かい WordPress Core の内部設計や、`intl` 拡張のフォールバック実装の話には踏み込みません。この記事では、あくまで「調査や開発の入口として使える環境を作る」ことに絞ります。

## `@wp-playground/cli` とは

`@wp-playground/cli` は、WordPress Playground をローカルのコマンドラインから使うためのツールです。

ざっくり言えば、Node.js があれば WordPress を起動できる CLI です。

通常の WordPress 開発環境では、PHP、Web サーバー、データベースを用意します。Docker を使う場合でも、コンテナ構成やボリューム、ポートなどを意識することになります。

一方、WordPress Playground CLI では、PHP は WebAssembly ベースで動き、データベースには SQLite が使われます。そのため、Docker や MySQL を先に用意しなくても、WordPress を起動できます。

ここで注意したいのは、`@wp-playground/cli` は WP-CLI とは別物だということです。

WP-CLI は、起動済みの WordPress をコマンドで操作するための CLI です。たとえば、投稿を作成したり、プラグインを有効化したり、オプション値を変更したりできます。

それに対して、WordPress Playground CLI は、WordPress を動かす環境そのものを立ち上げるための CLI です。

| ツール                  | 主な用途                          |
| -------------------- | ----------------------------- |
| `@wp-playground/cli` | 軽量な WordPress 調査・開発環境を起動する    |
| WP-CLI               | 起動済み WordPress をコマンドで操作する     |
| `wp-env`             | Docker ベースで WordPress 開発環境を作る |
| Docker Compose       | 本番に近い構成を自分で組む                 |

今回の目的は、本番環境の完全再現ではありません。

最新の WordPress の管理画面を触る、プラグインをロードする、テーマを読み込む、PHP バージョンを切り替える、といった軽い調査や開発の入口として使います。

## 前提条件

`@wp-playground/cli` を使うには Node.js が必要です。

まずは、Node.js と npm のバージョンを確認します。

```bash
node -v
npm -v
```

Node.js の導入方法は環境によって異なります。Debian / Ubuntu 系なら NodeSource、`mise`、`asdf`、`nvm` などを使う方法があります。

この記事では Node.js の導入手順には踏み込みません。すでに `node` と `npm` が使える状態を前提にします。

## まず WordPress を起動する

最初は、何も考えずに起動してみます。

```bash
npx @wp-playground/cli@latest start
```

`npx` を使うことで、グローバルインストールせずに `@wp-playground/cli` を実行できます。

しばらく待つと、ローカルで WordPress が起動し、ブラウザーからアクセスできるようになります。

`start` コマンドは、初めて試すにはいちばん分かりやすいコマンドです。

* WordPress を起動する
* ブラウザーを開く
* プロジェクト種別を自動判定する
* サイトの状態を保存する

小さく試すだけなら、まずは `start` で十分です。

## `start` と `server` の使い分け

WordPress Playground CLI では、主に `start` と `server` を使います。

```bash
npx @wp-playground/cli@latest start
```

```bash
npx @wp-playground/cli@latest server
```

最初は `start` を使えばよいです。

`start` は、簡単に WordPress を起動するためのコマンドです。プラグインやテーマのディレクトリで実行したときも、自動判定してくれるので便利です。

一方で、PHP のバージョンを指定したり、WordPress のバージョンを指定したり、Blueprint を使ったり、明示的にディレクトリをマウントしたりする場合は、`server` のほうが分かりやすいです。

| コマンド     | 向いている用途                                        |
| -------- | ---------------------------------------------- |
| `start`  | はじめての起動、テーマ・プラグインの簡単な確認                        |
| `server` | PHP / WordPress バージョン指定、Blueprint、手動マウント、細かい検証 |

この記事でも、単純な起動には `start`、構成を指定したい場面では `server` を使います。

## 日本語環境で起動する

WordPress の基本機能を調べるなら、日本語環境で起動したくなります。

WordPress Playground CLI では、Blueprint を使って初期状態を指定できます。

まず、`blueprint-ja.json` を作ります。

```json
{
  "$schema": "https://playground.wordpress.net/blueprint-schema.json",
  "landingPage": "/wp-admin/",
  "login": true,
  "steps": [
    {
      "step": "setSiteLanguage",
      "language": "ja"
    }
  ]
}
```

この Blueprint を指定して起動します。

```bash
npx @wp-playground/cli@latest start --blueprint=blueprint-ja.json
```

または、`server` で起動します。

```bash
npx @wp-playground/cli@latest server --blueprint=blueprint-ja.json
```

ここで指定する言語は `ja` です。

WordPress のロケールとしては `ja_JP` を見かけることがありますが、Playground CLI の Blueprint で `setSiteLanguage` を使う場合、日本語化には `ja` を指定するのがよさそうです。

実際に `ja_JP` を指定すると、翻訳パッケージ取得でエラーになる場合がありました。

一方で、`ja` を指定した場合、テーマ翻訳の取得エラーが表示されることはありましたが、管理画面メニューは日本語化されました。

つまり、まずは次のように覚えておけばよいです。

```json
{
  "step": "setSiteLanguage",
  "language": "ja"
}
```

Blueprint を保存しておくと、日本語 WordPress の検証環境を毎回同じように起動できます。

## WordPress と PHP のバージョンを指定する

WordPress Playground CLI では、WordPress 本体のバージョンや PHP のバージョンを指定できます。

たとえば、PHP 8.3 で起動する場合は次のようにします。

```bash
npx @wp-playground/cli@latest server --php=8.3
```

WordPress のバージョンも同時に指定できます。

```bash
npx @wp-playground/cli@latest server --wp=6.8 --php=8.3
```

日本語 Blueprint と組み合わせるなら、次のようになります。

```bash
npx @wp-playground/cli@latest server \
  --wp=6.8 \
  --php=8.3 \
  --blueprint=blueprint-ja.json
```

この機能は、WordPress の新機能を確認したり、PHP バージョン差分によるエラーを軽く確認したりするのに便利です。

ただし、Playground CLI の PHP は WebAssembly 版 PHP です。通常の Debian やレンタルサーバー上の PHP と完全に同じではありません。

そのため、次のような確認には向いています。

* プラグインやテーマが特定の PHP バージョンで fatal error にならないか
* WordPress の管理画面や基本機能がどう変わるか
* PHP 8.3 / 8.4 などで警告が出ないか
* 学習記事用に軽い再現環境を作る

一方で、MySQL / MariaDB 固有の挙動、Apache / Nginx の設定、Redis、Imagick、メール送信など、本番環境に依存する検証には向きません。

## intl 拡張を有効にする

Playground CLI には `--intl` オプションがあります。

```bash
npx @wp-playground/cli@latest server \
  --php=8.3 \
  --intl \
  --blueprint=blueprint-ja.json
```

`intl` は PHP の国際化関連の拡張です。

WordPress 本体の最低要件ではありませんが、現代的なサイト開発では使いたくなる場面が多い拡張です。

たとえば、次のような用途があります。

* 日本語や多言語の文字列処理
* 絵文字や結合文字を含むテキストの扱い
* 日付や数値の地域化
* 多言語プラグインの検証
* `grapheme_strlen()` や `grapheme_extract()` などの確認

この記事では、`intl` の細かい設計論には踏み込みません。

重要なのは、Playground CLI では `--intl` を付けることで、`intl` ありの WordPress 検証環境を作れるという点です。

確認用に、簡単な PHP ファイルやプラグイン内で次のように確認できます。

```php
<?php
var_dump(extension_loaded('intl'));
var_dump(function_exists('grapheme_strlen'));
```

WordPress 本体の基本機能を確認するだけなら `--intl` なしでもよいです。

しかし、日本語、Unicode、絵文字、多言語対応、文字列処理を含む開発を試すなら、`--intl` ありで起動できることはかなり重要です。

## プラグインをロードして開発する

次に、簡単なプラグインをロードしてみます。

まず、プラグイン用のディレクトリを作ります。

```bash
mkdir my-playground-plugin
cd my-playground-plugin
```

最小限のプラグインファイルを作ります。

```php
<?php
/**
 * Plugin Name: My Playground Plugin
 * Description: Playground CLI test plugin.
 */

add_action('admin_notices', function () {
    echo '<div class="notice notice-success"><p>Hello from Playground CLI.</p></div>';
});
```

ファイル名は、たとえば `my-playground-plugin.php` にします。

```bash
npx @wp-playground/cli@latest start
```

プラグインディレクトリで `start` を実行すると、Playground CLI がプラグインとして認識してくれます。

管理画面でプラグインを有効化すると、管理画面に通知が表示されるはずです。

このように、小さなプラグインを作って WordPress 上で確認するだけなら、かなり軽く始められます。

手動でマウントしたい場合は、次のように指定できます。

```bash
npx @wp-playground/cli@latest server \
  --mount=.:/wordpress/wp-content/plugins/my-playground-plugin
```

自動判定で十分な場合は `start`、マウント先を明示したい場合は `server --mount` と考えると分かりやすいです。

## テーマをロードして開発する

テーマも同じようにロードできます。

テーマディレクトリで起動します。

```bash
cd my-theme
npx @wp-playground/cli@latest start
```

ブロックテーマやクラシックテーマのファイルを置いた状態で起動すれば、WordPress からテーマを確認できます。

手動でマウントする場合は、次のようにします。

```bash
npx @wp-playground/cli@latest server \
  --mount=.:/wordpress/wp-content/themes/my-theme
```

この場合、ファイルはテーマディレクトリに配置されますが、テーマの有効化は別途必要です。

管理画面からテーマを有効化してもよいですし、Blueprint で有効化してもよいです。

たとえば、Blueprint でテーマを有効化する場合は次のようにします。

```json
{
  "$schema": "https://playground.wordpress.net/blueprint-schema.json",
  "landingPage": "/wp-admin/themes.php",
  "login": true,
  "steps": [
    {
      "step": "setSiteLanguage",
      "language": "ja"
    },
    {
      "step": "activateTheme",
      "themeFolderName": "my-theme"
    }
  ]
}
```

テーマ開発では、次のような確認に使えます。

* `theme.json` の設定確認
* テンプレートの確認
* スタイルバリエーションの確認
* ブロックテーマの基本動作確認
* 管理画面からの見え方の確認

本格的なテーマ制作のすべてを Playground CLI だけで完結させる必要はありません。

ただ、最初にテーマを読み込んで、WordPress 上でどう見えるか確認する環境としてはかなり便利です。

## 状態をリセットする

`start` コマンドはサイトの状態を保存します。

これは便利ですが、何度も試していると、最初の状態に戻したくなることがあります。

その場合は `--reset` を使います。

```bash
npx @wp-playground/cli@latest start --reset
```

学習や検証では、「壊しても戻せる」ことが重要です。

新しいプラグインを試す、テーマを切り替える、設定を変更する、といった作業をしていると、環境がだんだん汚れていきます。

そのため、検証用のコマンドには `--reset` を組み合わせられることを覚えておくと安心です。

## Playground CLI が向いている用途

実際に触ってみると、Playground CLI は次のような用途に向いていると感じました。

* 最新 WordPress の基本機能を確認する
* 管理画面の変更点を確認する
* ブロックエディターを軽く試す
* 小さなプラグインをロードして動作確認する
* テーマや `theme.json` を確認する
* PHP バージョン差分を軽く見る
* 日本語化した WordPress をすぐ用意する
* `intl` ありの文字列処理を試す
* チュートリアル記事用の再現環境を作る

特に、WordPress の調査記事を書くときには便利です。

Docker 構成の説明から始めなくても、まず WordPress を起動し、管理画面を開き、機能を確認できます。

テーマやプラグイン開発でも、初期段階の確認環境として使いやすいです。

## Playground CLI が向いていない用途

一方で、万能ではありません。

次のような用途では、Docker Compose や `wp-env`、あるいは本番に近い検証環境を使ったほうがよいです。

* MySQL / MariaDB 固有の挙動確認
* Apache / Nginx の設定確認
* `.htaccess` やリライトルールの詳細確認
* メール送信の確認
* Redis やオブジェクトキャッシュの確認
* Imagick などサーバー拡張に依存する処理
* 大規模サイトの性能検証
* 本番環境に近い構成の再現

Playground CLI は、軽く調べるための環境として見るのがよいです。

本番環境の完全な代替ではなく、調査や試作の入口です。

## 今回使ったコマンド

最後に、この記事で使ったコマンドをまとめます。

```bash
# WordPress を起動
npx @wp-playground/cli@latest start

# 詳細指定モードで起動
npx @wp-playground/cli@latest server

# WordPress / PHP バージョンを指定
npx @wp-playground/cli@latest server --wp=6.8 --php=8.3

# 日本語 Blueprint を指定
npx @wp-playground/cli@latest server --blueprint=blueprint-ja.json

# intl 拡張を有効化
npx @wp-playground/cli@latest server --php=8.3 --intl

# 日本語 + PHP 指定 + intl
npx @wp-playground/cli@latest server \
  --wp=6.8 \
  --php=8.3 \
  --intl \
  --blueprint=blueprint-ja.json

# プラグインを手動マウント
npx @wp-playground/cli@latest server \
  --mount=.:/wordpress/wp-content/plugins/my-playground-plugin

# テーマを手動マウント
npx @wp-playground/cli@latest server \
  --mount=.:/wordpress/wp-content/themes/my-theme

# 状態をリセットして起動
npx @wp-playground/cli@latest start --reset
```

## まとめ

`@wp-playground/cli` を使うと、WordPress の調査環境をかなり軽く作れます。

Node.js と `npx` が使えれば、Docker や MySQL を用意せずに WordPress を起動できます。

今回確認した範囲では、日本語化、PHP バージョン指定、`intl` 拡張の有効化、プラグインのロード、テーマのロードができました。

もちろん、本番サーバーと完全に同じ環境ではありません。PHP は WebAssembly 版であり、データベースは SQLite です。そのため、本番に近い検証が必要な場面では、Docker Compose や `wp-env` などを使うべきです。

しかし、最新 WordPress の基本機能を調べる、テーマやプラグインを軽く試す、記事用の再現環境を作る、といった用途ではかなり便利です。

WordPress を「まず動かして調べる」ための道具として、`@wp-playground/cli` は今後も使っていきたいと思います。
