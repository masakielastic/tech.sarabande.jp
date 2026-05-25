---
title: "Astro と Cloudflare Workers でテックブログを作り、GitHub Actions で自動デプロイするまで"
description: "Astro の最小構成から Cloudflare Workers にデプロイし、custom_domain: true で独自ドメインを割り当て、GitHub Actions で自動デプロイするまでの学習記録。"
pubDate: 2026-05-25
tags:
  - Astro
  - Cloudflare Workers
  - Wrangler
  - GitHub Actions
---

## はじめに

Astro と Cloudflare Workers を使って、自分用のテックブログを作ることにした。

これまで Cloudflare Tunnel、Access、WARP、Workers などを少しずつ触ってきたが、Cloudflare の Web ダッシュボードは機能が多く、初心者にはかなり複雑に見える。

そこで今回は、できるだけ Wrangler と GitHub Actions を使い、コマンドライン中心でテックブログを公開するところまで試した。

この記事では、次の流れを学習記録として整理する。

* Astro の最小構成でテックブログを作る
* Cloudflare Workers にデプロイする
* `tech.sarabande.jp` に独自ドメインを割り当てる
* `routes` と `custom_domain: true` の違いでつまずく
* GitHub Actions で自動デプロイする
* Environment secrets を使って Cloudflare の API Token を管理する

完成したあとから見ると単純だが、実際にはいくつか迷いやすいポイントがあった。

## 今回作ったもの

今回作ったのは、Astro の最小構成を Cloudflare Workers にデプロイするテックブログである。

公開先は次のサブドメインにした。

```text
tech.sarabande.jp
```

リポジトリ名も、公開先に合わせて次のようにした。

```text
tech.sarabande.jp
```

必ずしもリポジトリ名をドメイン名と一致させる必要はない。たとえば `tech-blog` のような名前でもよい。

ただ、今回は「このリポジトリは `tech.sarabande.jp` の中身である」と一目でわかるように、サブドメイン名に合わせた。

## 最初はブログテンプレートではなく最小構成にした

最初は Wrangler で初期化するときにブログテンプレートを選んだ。

しかし、テンプレートには最初からいろいろなファイルや設定が含まれている。

* サンプル記事
* コンポーネント
* レイアウト
* タグや著者情報
* スタイルシート
* RSS や SEO 用の設定

完成品としては便利だが、学習目的では「どのファイルが何をしているのか」を把握するのが大変になる。

今回は、Cloudflare Workers、Astro、Wrangler、GitHub Actions の関係を理解することが目的だったので、ブログテンプレートを捨てて、Astro の最小構成からやり直した。

最小構成にすると、最初は次のような構成になる。

```text
astro.config.mjs
package.json
package-lock.json
public/
src/
  pages/
    index.astro
tsconfig.json
wrangler.jsonc
```

この段階では、まだ本格的なブログ機能はない。

しかし、まずは「トップページが表示される」「Cloudflare Workers にデプロイできる」「独自ドメインで表示できる」というところまで確認するには十分である。

## Wrangler の設定ファイル

最終的な `wrangler.jsonc` は次のようになった。

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "tech-sarabande-jp",
  "compatibility_date": "2026-05-25",
  "assets": {
    "directory": "./dist",
    "not_found_handling": "404-page"
  },
  "routes": [
    {
      "pattern": "tech.sarabande.jp",
      "custom_domain": true
    }
  ]
}
```

ここで重要なのは、次の部分である。

```jsonc
"assets": {
  "directory": "./dist",
  "not_found_handling": "404-page"
}
```

これは、Astro がビルドした `dist` ディレクトリを Workers の静的アセットとして配信するという意味である。

もうひとつ重要なのが、独自ドメインの指定である。

```jsonc
"routes": [
  {
    "pattern": "tech.sarabande.jp",
    "custom_domain": true
  }
]
```

今回の用途では、`tech.sarabande.jp` 全体を Workers のサイトとして公開したかった。

そのため、`custom_domain: true` を使うのが適切だった。

## `dist` が存在しないエラー

最小構成にしてから、いきなり `npx wrangler deploy` を実行するとエラーになった。

```text
✘ [ERROR] The directory specified by the "assets.directory" field in your configuration file does not exist:

  /home/masakielastic/playground/tech.sarabande.jp/dist
```

原因は単純だった。

Wrangler は Astro のビルドを自動では実行してくれない。

`wrangler.jsonc` では `./dist` をアップロードする設定にしているが、その `dist` は `npm run build` を実行しないと作られない。

そのため、正しい順番は次のようになる。

```bash
npm run build
npx wrangler deploy
```

毎回忘れないようにするなら、`package.json` に次のようなスクリプトを追加しておくとよい。

```json
{
  "scripts": {
    "dev": "astro dev",
    "build": "astro build",
    "preview": "astro preview",
    "deploy": "astro build && wrangler deploy"
  }
}
```

これで、ローカルでは次のコマンドだけでビルドとデプロイを実行できる。

```bash
npm run deploy
```

## `routes` ではなく `custom_domain: true` が必要だった

独自ドメインの設定でもつまずいた。

最初は、次のように `routes` を設定していた。

```jsonc
"routes": [
  {
    "pattern": "tech.sarabande.jp/*",
    "zone_name": "sarabande.jp"
  }
]
```

一見すると正しそうに見える。

しかし、この設定は「既存のホスト名に来たリクエストに Worker を差し込む」ような用途に近い。

今回の目的は、`tech.sarabande.jp` というサブドメイン全体を Workers のサイト本体として公開することだった。

そこで、最終的には次の設定にした。

```jsonc
"routes": [
  {
    "pattern": "tech.sarabande.jp",
    "custom_domain": true
  }
]
```

これで `tech.sarabande.jp` が正常に表示されるようになった。

今回の理解としては、次のように整理できる。

```text
custom_domain: true
→ Worker 自体を tech.sarabande.jp のサイト本体として公開する

routes の pattern: "tech.sarabande.jp/*"
→ 既存サイトや origin の前段で Worker を差し込む用途に近い
```

Astro の静的サイトを Workers で配信し、サブドメイン全体をそのサイトにしたい場合は、`custom_domain: true` のほうが素直だった。

## GitHub Actions で自動デプロイする

ローカルでデプロイできるようになったので、次に GitHub Actions で自動デプロイすることにした。

やりたいことは単純である。

```text
main ブランチに push
↓
GitHub Actions が起動
↓
npm ci
↓
npm run build
↓
npx wrangler deploy
↓
tech.sarabande.jp に反映
```

最終的な workflow は次のような形にした。

```yaml
name: Deploy to Cloudflare Workers

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 10

    environment: Cloudflare

    permissions:
      contents: read

    env:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}

    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: Setup Node.js
        uses: actions/setup-node@v6
        with:
          node-version: 22
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Build Astro
        run: npm run build

      - name: Check build output
        run: |
          pwd
          ls -la
          ls -la dist
          npx wrangler --version

      - name: Check Cloudflare env
        run: |
          test -n "$CLOUDFLARE_API_TOKEN" && echo "CLOUDFLARE_API_TOKEN is set"
          test -n "$CLOUDFLARE_ACCOUNT_ID" && echo "CLOUDFLARE_ACCOUNT_ID is set"

      - name: Deploy to Cloudflare Workers
        run: npx wrangler deploy
```

ここでは、`cloudflare/wrangler-action` ではなく、直接 `npx wrangler deploy` を実行している。

理由は、ローカルで成功した手順と近い形にしたほうが理解しやすかったからである。

また、GitHub Actions で `wrangler-action@v3` を使ったときに Node.js 20 の非推奨警告が出たため、今回は通常の shell コマンドとして Wrangler を実行する構成にした。

## API Token と Account ID

GitHub Actions から Cloudflare Workers にデプロイするには、Cloudflare の API Token が必要である。

GitHub には、次の2つを Secret として登録した。

```text
CLOUDFLARE_API_TOKEN
CLOUDFLARE_ACCOUNT_ID
```

ここで注意したいのは、API Token の verify API で返ってくる `id` は Account ID ではないという点である。

たとえば、次のようなリクエストでトークンを検証できる。

```bash
curl "https://api.cloudflare.com/client/v4/user/tokens/verify" \
  -H "Authorization: Bearer xxxxxxxx"
```

このレスポンスの `result.id` は、API Token 自体の ID である。

```json
{
  "result": {
    "id": "xxxxxxxxxxxxx",
    "status": "active"
  },
  "success": true,
  "errors": [],
  "messages": [
    {
      "code": 10000,
      "message": "This API Token is valid and active",
      "type": null
    }
  ]
}
```

これは `CLOUDFLARE_ACCOUNT_ID` に入れる値ではない。

`CLOUDFLARE_ACCOUNT_ID` には、Cloudflare アカウントの ID を入れる必要がある。

## Environment secrets を使った

今回、GitHub の Secret は Repository secrets ではなく Environment secrets に登録していた。

具体的には、`Cloudflare` という Environment に次の Secret を作成した。

```text
CLOUDFLARE_ACCOUNT_ID
CLOUDFLARE_API_TOKEN
```

そのため、workflow 側でも次の指定が必要だった。

```yaml
environment: Cloudflare
```

これを書かないと、GitHub Actions から Environment secrets を参照できない。

実際、最初は次のエラーになった。

```text
✘ [ERROR] In a non-interactive environment, it's necessary to set a CLOUDFLARE_API_TOKEN environment variable for wrangler to work.
```

Secret は登録されているのに、Wrangler から見ると `CLOUDFLARE_API_TOKEN` が存在しない。

原因は、Environment secrets を使っているのに、workflow の job に `environment: Cloudflare` を書いていなかったことだった。

`environment: Cloudflare` を追加すると、正常にデプロイできるようになった。

## Environment secrets のメリット

Environment secrets を使うと、デプロイ先ごとに Secret を分けやすくなる。

たとえば、将来的に次のような環境を作りたくなるかもしれない。

```text
Cloudflare
→ 本番用

Cloudflare Preview
→ プレビュー用

Staging
→ 検証用
```

それぞれの Environment に、同じ名前の Secret を登録できる。

```text
CLOUDFLARE_API_TOKEN
CLOUDFLARE_ACCOUNT_ID
```

同じ名前でも、Environment ごとに値を変えられる。

また、workflow に次のように書くことで、どの job がどの環境へデプロイするのかが明確になる。

```yaml
jobs:
  deploy:
    environment: Cloudflare
```

これは単に Secret を読むための設定ではなく、運用上の意味もある。

```text
この job は Cloudflare 環境へデプロイする
この job だけが Cloudflare 用 Secret を使う
```

個人ブログなら Repository secrets でも十分だが、本番デプロイ用の API Token を少し明確に分けて管理したいなら、Environment secrets は便利だった。

## `deploy.yml` はブランチとデプロイ先の対応表でもある

GitHub Actions の `deploy.yml` は、単に自動デプロイするためのファイルではない。

どのブランチに push されたら、どの環境の Secret を使って、どこへデプロイするかを決めるファイルでもある。

今回の最小構成では、次のようにした。

```text
main ブランチ
→ Cloudflare Environment
→ tech.sarabande.jp にデプロイ
```

workflow では、ここで `main` ブランチへの push を指定している。

```yaml
on:
  push:
    branches:
      - main
```

そして、ここで Cloudflare Environment を指定している。

```yaml
environment: Cloudflare
```

つまり、今回の運用は次のように整理できる。

```text
main に push したら、Cloudflare 用の Secret を使って、本番の tech.sarabande.jp にデプロイする
```

将来的には、次のような構成にもできる。

```text
main
→ 本番環境
→ tech.sarabande.jp

develop
→ 検証環境
→ preview 用 Worker

feature/*
→ テストだけ実行してデプロイしない
```

ただし、最初から複雑にする必要はない。

個人のテックブログなら、まずは `main` に push したら本番に反映される、という単純な形で十分である。

## 記事ファイルの置き場所

最初の記事は、次のような場所に置くことにした。

```text
src/pages/posts/2026-05-25-tech-blog-with-astro-workers.md
```

Astro では `src/pages/` に Markdown ファイルを置くだけでページになる。

そのため、最小構成ではまず次のような形にするのがわかりやすい。

```text
src/pages/posts/*.md
```

記事の frontmatter は次のようにする。

```markdown
---
title: "Astro と Cloudflare Workers でテックブログを作り、GitHub Actions で自動デプロイするまで"
description: "Astro の最小構成から Cloudflare Workers にデプロイし、custom_domain: true で独自ドメインを割り当て、GitHub Actions で自動デプロイするまでの学習記録。"
pubDate: 2026-05-25
tags:
  - Astro
  - Cloudflare Workers
  - Wrangler
  - GitHub Actions
---
```

本格的にブログを育てるなら、将来的には `src/content/` と Content Collections を使う構成に移行してもよい。

たとえば、次のような構成である。

```text
src/content/blog/*.md
src/pages/posts/[slug].astro
```

ただし、最初はそこまでしなくてよい。

まずは `src/pages/posts/*.md` に記事を置き、ページとして表示されることを優先した。

## 今回学んだこと

今回の作業で、Astro、Cloudflare Workers、Wrangler、GitHub Actions の役割分担がかなり見えた。

Astro はサイトを作る。

Wrangler は Cloudflare Workers にデプロイする。

`wrangler.jsonc` は Workers 側の設定を管理する。

GitHub Actions は、push をきっかけにビルドとデプロイを自動化する。

Cloudflare API Token は、GitHub Actions から Cloudflare を操作するための認証情報である。

Environment secrets は、その認証情報を環境別に管理するための仕組みである。

今回の最小構成では、次の関係になった。

```text
Astro
→ src/pages/ から静的サイトを作る

npm run build
→ dist/ を作る

Wrangler
→ dist/ を Cloudflare Workers にデプロイする

custom_domain: true
→ tech.sarabande.jp を Worker のサイト本体として公開する

GitHub Actions
→ main への push をきっかけに自動デプロイする

Environment secrets
→ Cloudflare API Token と Account ID を管理する
```

## まとめ

今回の作業では、Astro と Cloudflare Workers を使って、自分用のテックブログを作るところまで進めた。

最初は Cloudflare の Web 画面の複雑さに戸惑ったが、Wrangler と GitHub Actions を使うことで、多くの作業をコマンドラインと設定ファイルに寄せることができた。

特につまずいたのは、次の3点だった。

```text
npm run build を実行しないと dist が作られない

tech.sarabande.jp 全体を Worker にするなら custom_domain: true が必要だった

Environment secrets を使うなら workflow に environment: Cloudflare が必要だった
```

どれも一度わかれば単純だが、最初に試すと迷いやすい。

だからこそ、こうした学習記録を自分のテックブログに残しておく意味がある。

Qiita や note に投稿するほど整った記事でなくても、自分の環境で実際につまずいたことを残しておけば、あとで同じ作業をするときの手がかりになる。

AI 時代には、学習速度が上がる一方で、学習記録も大量に増える。

すべてを外部プラットフォームに投稿するのではなく、自分のテックブログに一度置いておき、あとで必要に応じて Qiita、note、小冊子、ドキュメントへ再編集する。

そのための置き場として、Astro と Cloudflare Workers の組み合わせはかなり扱いやすいと感じた。

## 次にやること

次は、テックブログとして最低限の機能を少しずつ追加していく。

* 記事一覧ページを作る
* 共通レイアウトを作る
* RSS を追加する
* sitemap を追加する
* OGP を整える
* 記事の frontmatter を整理する
* 将来的に Content Collections へ移行するか検討する

最初から完成度の高いブログを目指すのではなく、まずは学習記録を置ける場所として運用していく。
