---

layout: ../../layouts/PostLayout.astro
title: "GitHub Actions に全部載せない：Cloudflare Workers 制作会社のための CI/CD 設計"
description: "Cloudflare Workers を使う制作会社向けに、Workers Builds、GitHub Actions、Blacksmith、Wrangler GitHub Action の役割分担を整理する。"
pubDate: 2026-05-26
tags:
  - Cloudflare Workers
  - GitHub Actions
  - Blacksmith
  - CI/CD
  -  Astro

---

Cloudflare Workers を使って Astro サイトや小規模な Web アプリを作る場合、最初に思いつきやすいのは GitHub Actions で `npm ci`、`npm run build`、`wrangler deploy` を実行する構成です。

個人開発や小さなブログであれば、それで十分です。

しかし、制作会社として Cloudflare Workers を継続的に使うなら、少し考え方を変えたほうがよいと感じました。

すべてを GitHub Actions に載せるのではなく、Cloudflare に出すものは Cloudflare 側へ、GitHub 上の開発活動は GitHub 側へ分けたほうが、費用も責任範囲も整理しやすいからです。

この記事では、Cloudflare Workers を使う制作会社を想定して、Cloudflare Workers Builds、GitHub Actions、Blacksmith、Wrangler GitHub Action の使い分けを整理します。

## この記事の結論

最初に結論を書くと、私は次の分担がよいと考えています。

```text
Cloudflare Workers Builds
→ Cloudflare にデプロイする成果物のビルド・デプロイ

GitHub Actions
→ PR チェック、Lint、型チェック、テスト、リリース作業、GitHub 上の自動化

Blacksmith
→ GitHub Actions の runner が遅い・高いと感じたときの高速化手段

Wrangler GitHub Action
→ 承認つきデプロイ、D1 migration、複数 Workers の一括デプロイなどの例外用途
```

要するに、Cloudflare Workers 関連のビルド・デプロイ費用は Cloudflare Workers Builds に寄せる。

GitHub Actions は、GitHub 上の開発活動や品質チェックに使う。

Blacksmith は、GitHub Actions をやめるためのサービスではなく、GitHub Actions の runner を高速化する選択肢として見る。

この分担が一番わかりやすいと思います。

## GitHub Actions に全部載せると何が困るのか

GitHub Actions は便利です。

たとえば Astro サイトを Cloudflare Workers にデプロイするなら、次のような workflow を書けば、自動デプロイできます。

```yaml
name: Deploy

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v6

      - uses: actions/setup-node@v6
        with:
          node-version: 22
          cache: npm

      - run: npm ci
      - run: npm run build

      - uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          command: deploy
```

これはシンプルです。

ただし、制作会社として案件が増えてくると、GitHub Actions に載せたい処理はデプロイだけではありません。

たとえば、次のような処理も GitHub Actions に載せたくなります。

```text
PR ごとの lint
TypeScript の typecheck
unit test
Playwright などの E2E テスト
Storybook のビルド
Lighthouse のチェック
npm package の公開
GitHub Release の作成
Issue や Pull Request の自動整理
社内テンプレートの更新
ドキュメント生成
OSS 活動
```

ここに Cloudflare Workers へのビルド・デプロイまで全部載せると、GitHub Actions の利用時間や費用が混ざります。

GitHub 上の開発活動に使っているのか、Cloudflare にデプロイするために使っているのかが見えにくくなります。

個人開発なら大きな問題ではありません。

しかし制作会社では、顧客案件ごとの費用、保守契約、権限管理、移管のしやすさも考える必要があります。

その意味で、Cloudflare に出す成果物のビルド・デプロイは Cloudflare 側に寄せたほうが整理しやすくなります。

## Cloudflare Workers Builds は何を担当するのか

Cloudflare Workers Builds は、Cloudflare 側で Workers 向けのビルドとデプロイを実行する仕組みです。

GitHub などのリポジトリと連携し、push や merge をきっかけに Cloudflare 側でビルドして、Workers にデプロイできます。

Cloudflare Workers を主戦場にするなら、デプロイ処理を Cloudflare 側に寄せるのは自然です。

Cloudflare 上で動くものは、Cloudflare 側のビルド基盤で作る。

このほうが、Cloudflare 関連の費用やリソースを Cloudflare 側に集約できます。

たとえば、制作会社が Cloudflare Workers を使う場合、Cloudflare 側には次のようなリソースが集まります。

```text
Workers
Workers Builds
Pages
KV
R2
D1
Durable Objects
Hyperdrive
Cloudflare Access
Zero Trust 関連の設定
```

ここにビルド・デプロイも寄せると、顧客に説明するときもわかりやすくなります。

「このサイトは Cloudflare 上で動いているので、ビルド、実行、ストレージ、認証まわりの費用は Cloudflare 側にまとまります」と説明できます。

## GitHub Actions は何に残すべきか

Cloudflare Workers Builds を使うからといって、GitHub Actions が不要になるわけではありません。

むしろ GitHub Actions は、開発プロセスの品質管理に残したほうがよいです。

たとえば、Pull Request ごとに次のようなチェックを走らせます。

```yaml
name: Check

on:
  pull_request:
    branches:
      - main

jobs:
  check:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v6

      - uses: actions/setup-node@v6
        with:
          node-version: 22
          cache: npm

      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm test
```

この workflow では、`wrangler deploy` は実行しません。

GitHub Actions は、Pull Request の品質チェックに専念します。

そして `main` に merge されたあとの本番デプロイは Cloudflare Workers Builds に任せます。

流れとしては、次のようになります。

```text
開発者が Pull Request を作る
        ↓
GitHub Actions が lint / typecheck / test を実行する
        ↓
問題なければ main に merge する
        ↓
Cloudflare Workers Builds が build / deploy を実行する
```

この形にすると、GitHub Actions は GitHub 上の開発活動を支える基盤として残ります。

Cloudflare へのデプロイ基盤とは役割が分かれます。

## Blacksmith はどこに入るのか

ここで Blacksmith です。

Blacksmith は、GitHub Actions を完全に置き換える CI サービスというより、GitHub Actions の runner を高速・低コストな外部 runner に置き換えるサービスです。

通常の GitHub Actions では、次のように書きます。

```yaml
runs-on: ubuntu-latest
```

Blacksmith を使う場合は、runner の指定を Blacksmith の runner に変えます。

```yaml
runs-on: blacksmith-4vcpu-ubuntu-2404
```

workflow 全体を CircleCI や GitLab CI に移すのではありません。

GitHub Actions の書き方、Pull Request 連携、GitHub Secrets、GitHub Environments などは維持しつつ、実行環境だけを Blacksmith に変えるイメージです。

そのため、Blacksmith は Cloudflare Workers Builds の競合というより、GitHub Actions を残す領域を高速化するための選択肢です。

整理すると、次のようになります。

```text
Cloudflare Workers Builds
→ Cloudflare にデプロイするための標準基盤

GitHub Actions
→ GitHub 上の開発活動を自動化する基盤

Blacksmith
→ GitHub Actions の runner を高速化する選択肢
```

## Blacksmith を使うべき場面

Blacksmith が向いているのは、GitHub Actions の実行時間や待ち時間が明確に問題になっている場合です。

たとえば、次のようなケースです。

```text
Docker build が重い
E2E テストが長い
依存関係のインストールやキャッシュ復元に時間がかかる
Pull Request が多く、CI 待ちが開発速度を落としている
GitHub Actions の無料枠や課金が気になっている
既存の GitHub Actions workflow はできるだけ維持したい
```

逆に、次のようなケースでは優先度は低いです。

```text
Astro ブログをたまにビルドするだけ
CI が月に数十回しか走らない
1回のチェックが1〜2分で終わる
Cloudflare Workers Builds で十分足りている
GitHub Actions の利用時間も問題になっていない
```

Blacksmith は便利な選択肢ですが、最初から入れる必要はありません。

まずは Cloudflare Workers Builds と GitHub Actions の役割を分ける。

そのうえで、GitHub Actions 側が重くなったら Blacksmith を検討する。

この順番で十分です。

## Wrangler GitHub Action はいつ使うか

Wrangler GitHub Action は、GitHub Actions から Cloudflare Wrangler を実行するための公式 Action です。

これは便利ですが、制作会社の標準運用では、すべてのデプロイを Wrangler GitHub Action に寄せるよりも、例外用途として考えたほうがよいと思います。

たとえば、次のような場面です。

```text
GitHub Environments の承認後に production deploy したい
D1 migration と Workers deploy を同じ workflow で管理したい
複数 Workers を順番にデプロイしたい
monorepo で Cloudflare Workers Builds より GitHub Actions のほうが制御しやすい
Cloudflare 以外のサービスにも同時にデプロイしたい
```

つまり、標準は Cloudflare Workers Builds。

特殊な制御が必要な案件では Wrangler GitHub Action。

この位置づけがよいと思います。

## 使い分けの判断表

制作会社として標準化するなら、次のように整理できます。

| やりたいこと                                | おすすめ                                  |
| ------------------------------------- | ------------------------------------- |
| Astro サイトを Cloudflare に自動デプロイしたい      | Cloudflare Workers Builds             |
| Workers / Pages の通常デプロイを管理したい         | Cloudflare Workers Builds             |
| PR ごとに lint / typecheck / test を走らせたい | GitHub Actions                        |
| GitHub Actions が遅い、または高い              | Blacksmith                            |
| Docker build や E2E テストが重い             | GitHub Actions + Blacksmith           |
| D1 migration と Workers deploy をまとめたい  | Wrangler GitHub Action                |
| production deploy に承認フローを入れたい         | GitHub Actions / GitHub Environments  |
| 顧客別に費用を分けたい                           | 顧客の Cloudflare アカウント + Workers Builds |

この表で重要なのは、Blacksmith をデプロイ基盤として見ないことです。

Blacksmith は GitHub Actions の runner を置き換えるものです。

Cloudflare に出すデプロイは、原則として Cloudflare Workers Builds に寄せます。

## 顧客案件では Cloudflare アカウントの分け方も重要

制作会社の場合、CI/CD の話はアカウント設計とも関係します。

理想は、顧客ごとに Cloudflare アカウントを分けることです。

```text
顧客 A のサイト
→ 顧客 A の Cloudflare アカウント
→ 顧客 A の Workers / Builds / R2 / D1

顧客 B のサイト
→ 顧客 B の Cloudflare アカウント
→ 顧客 B の Workers / Builds / R2 / D1
```

この形にすると、Cloudflare 側の利用量や費用が顧客ごとに分離されます。

請求や保守契約の説明もしやすくなります。

逆に、制作会社の Cloudflare アカウントにすべての顧客サイトを集約すると、最初は楽です。

しかし、将来的には次のような問題が出やすくなります。

```text
請求の分離が難しい
権限管理が複雑になる
顧客解約時の移管が面倒になる
利用量の説明が難しい
制作会社側のアカウントにリスクが集中する
```

そのため、制作会社の標準ルールとしては、原則として顧客の Cloudflare アカウントに構築するのがよいと思います。

制作会社のアカウントに集約する場合は、保守契約、請求方法、解約時の移管ルールを明文化したほうが安全です。

## 小規模案件ではやりすぎない

CI/CD は、凝り始めるとどこまでも複雑になります。

しかし、小規模な Astro サイトやブログでは、最初から複雑な構成にする必要はありません。

導入順としては、次でよいと思います。

```text
最初
→ Cloudflare Workers Builds だけでデプロイする

品質チェックが必要になったら
→ GitHub Actions で lint / typecheck / test を追加する

GitHub Actions が重くなったら
→ Blacksmith を検討する

特殊なデプロイ制御が必要になったら
→ Wrangler GitHub Action を使う
```

最初から全部入りにしない。

必要になった段階で、役割を分けて追加する。

このほうが、初心者エンジニアにも説明しやすく、制作会社の標準テンプレートとしても扱いやすいです。

## Astro ブログでの標準構成例

Astro ブログを Cloudflare Workers に出す場合、まずは次の構成を標準にするとよいと思います。

```text
Repository
├── src/
├── astro.config.mjs
├── package.json
├── wrangler.jsonc または wrangler.toml
└── .github/
    └── workflows/
        └── check.yml
```

GitHub Actions には、PR チェックだけを置きます。

```yaml
name: Check

on:
  pull_request:
    branches:
      - main

jobs:
  check:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v6

      - uses: actions/setup-node@v6
        with:
          node-version: 22
          cache: npm

      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm run build
```

ここで `npm run build` を入れるかどうかは好みです。

Pull Request の段階でビルドエラーを検出したいなら入れます。

ただし、本番デプロイそのものは Cloudflare Workers Builds に任せます。

この構成なら、GitHub Actions は「開発時のチェック」、Cloudflare Workers Builds は「デプロイ」として役割が分かれます。

## Blacksmith に切り替えるときの変更点

もし PR チェックが重くなって、Blacksmith を導入する場合、最初の変更は小さく済みます。

たとえば、次の行を変更します。

```yaml
runs-on: ubuntu-latest
```

Blacksmith の runner に変更します。

```yaml
runs-on: blacksmith-4vcpu-ubuntu-2404
```

もちろん、実際には Blacksmith 側のセットアップやリポジトリ連携が必要です。

ただ、workflow の考え方を大きく変えなくてよい点がメリットです。

GitHub Actions の資産を維持したまま、実行環境だけを改善できます。

## 制作会社の標準ルール案

最後に、制作会社としてルール化するなら、次のような方針がよいと思います。

```text
原則
Cloudflare に出す案件は Cloudflare Workers Builds でデプロイする。

GitHub Actions の役割
PR チェック、Lint、型チェック、テスト、E2E、リリース作業、GitHub 上の自動化に限定する。

Blacksmith の役割
GitHub Actions の実行時間、待ち時間、料金が問題になった案件だけ導入する。

Wrangler GitHub Action の役割
Cloudflare Workers Builds で扱いづらい特殊なデプロイ制御に使う。

顧客案件
可能な限り顧客の Cloudflare アカウントにデプロイする。

費用管理
Cloudflare 利用料は Cloudflare 側、GitHub 開発自動化費用は GitHub 側として分けて見る。
```

このルールなら、GitHub Actions に全部載せるよりも、費用と責任範囲が見えやすくなります。

## まとめ

Cloudflare Workers を使う制作会社では、CI/CD を「全部 GitHub Actions に載せる」発想から少し離れたほうがよいと思います。

Cloudflare にデプロイする成果物は Cloudflare Workers Builds に寄せる。

GitHub Actions は、Pull Request の品質チェックや GitHub 上の自動化に使う。

Blacksmith は、GitHub Actions が重くなったときに runner を高速化する選択肢として見る。

Wrangler GitHub Action は、承認フローや D1 migration など、特殊なデプロイ制御が必要なときに使う。

整理すると、次のようになります。

```text
Cloudflare Workers Builds
= デプロイの標準基盤

GitHub Actions
= 開発品質と GitHub 活動の自動化基盤

Blacksmith
= GitHub Actions runner の高速化手段

Wrangler GitHub Action
= 複雑なデプロイ制御のための補助線
```

小規模案件では、まず Cloudflare Workers Builds だけで十分です。

必要になったら GitHub Actions を追加する。

さらに重くなったら Blacksmith を検討する。

特殊な要件が出たら Wrangler GitHub Action を使う。

この順番で考えると、Cloudflare Workers を使った制作案件でも、CI/CD を過剰に複雑にせずに運用できます。
