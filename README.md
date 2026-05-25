# tech.sarabande.jp

Astro と Cloudflare Workers で運用する個人用テックブログです。

## 構成

- `src/pages/index.astro`: 記事一覧
- `src/pages/posts/*.md`: 記事本文
- `src/layouts/BaseLayout.astro`: 共通 HTML、メタ情報、スタイル
- `src/layouts/PostLayout.astro`: 記事ページ用レイアウト
- `src/pages/404.astro`: 404 ページ
- `public/`: favicon や robots.txt などの静的ファイル
- `wrangler.jsonc`: Cloudflare Workers Assets のデプロイ設定

## 開発

```sh
npm install
npm run dev
```

ローカル開発サーバーは通常 `http://localhost:4321` で起動します。

## ビルド

```sh
npm run build
```

Astro が静的ファイルを `dist/` に生成します。Cloudflare Workers にはこの `dist/` を Workers Assets としてデプロイします。

## デプロイ

```sh
npm run deploy
```

このコマンドは `astro build` のあとに `wrangler deploy` を実行します。

GitHub Actions では `main` ブランチへの push をきっかけにビルドとデプロイを実行します。GitHub Environment `Cloudflare` に次の secrets が必要です。

- `CLOUDFLARE_API_TOKEN`
- `CLOUDFLARE_ACCOUNT_ID`

## 記事の追加

`src/pages/posts/` に Markdown ファイルを追加します。

```md
---
layout: ../../layouts/PostLayout.astro
title: "記事タイトル"
description: "記事の説明文"
pubDate: 2026-05-25
tags:
  - Astro
---

本文を書きます。
```

ファイル名が URL になります。たとえば `src/pages/posts/example.md` は `/posts/example/` として公開されます。
