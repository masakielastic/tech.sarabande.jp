---

layout: ../../layouts/PostLayout.astro
title: "画像生成 AI API を Cloudflare Workers で扱うときの Secrets 管理"
description: "画像生成 AI API を題材に、Cloudflare Workers の環境変数、Workers Secrets、Cloudflare Secrets Store、AI Gateway、Access の使い分けを整理します。"
pubDate: 2026-05-26
tags:
  - Cloudflare
  - Cloudflare Workers
  - AI Gateway
  - Secrets
  - 画像生成AI

---

## はじめに

画像生成 AI API を使った小さなアプリを作るとき、最初に悩むのは「API キーをどこに置くべきか」です。

OpenAI、Replicate、fal.ai、その他の画像生成 API をブラウザーから直接呼び出すと、API キーがフロントエンドに露出してしまいます。これは避けるべきです。API キーが漏れると、第三者に勝手に利用され、意図しない課金や悪用につながる可能性があります。

そこで Cloudflare Workers を API の中継地点にします。

```text
ブラウザー / PWA / 管理画面
        ↓
Cloudflare Workers
        ↓
画像生成 AI API
```

この構成にすると、ブラウザー側には API キーを置かず、Worker 側で安全に画像生成 API を呼び出せます。

ただし、実際に運用を考えると、単に「Worker に API キーを置けばよい」という話では終わりません。

個人用の単一 Worker で使うのか、複数の Workers で同じ API キーを使うのか、あるいはチームで画像生成ツールを運用するのかによって、適した管理方法が変わります。

この記事では、画像生成 AI API を題材に、Cloudflare Workers の環境変数、Workers Secrets、Cloudflare Secrets Store、AI Gateway、Cloudflare Access の使い分けを整理します。

## 画像生成 AI API で管理したいもの

画像生成 AI API を扱うとき、管理したいものはいくつかあります。

たとえば、次のようなものです。

* API キー
* 利用するモデル名
* 画像サイズ
* 生成枚数の上限
* 利用者ごとの制限
* 生成履歴
* コスト
* チームメンバーの権限

このうち、すべてを同じ方法で管理する必要はありません。

たとえば、モデル名や画像サイズは機密情報ではありません。変更可能な設定値です。一方で、OpenAI や Replicate などの API キーは機密情報です。漏洩すると課金や悪用につながるため、通常の環境変数として扱うべきではありません。

まずは、設定値と機密情報を分けて考えることが重要です。

```text
通常の設定値
  - DEFAULT_IMAGE_MODEL
  - IMAGE_SIZE
  - MAX_IMAGES_PER_REQUEST
  - APP_ENV

機密情報
  - OPENAI_API_KEY
  - REPLICATE_API_TOKEN
  - FAL_API_KEY
  - AI_GATEWAY_TOKEN
```

この区別を出発点にすると、Cloudflare Workers の環境変数、Secrets、Secrets Store の使い分けが理解しやすくなります。

## ケース1: 自分専用の単一 Worker で画像生成 API を使う

最初は、個人開発や学習用の小さな画像生成ツールを想定します。

たとえば、次のような用途です。

* 自分専用の画像生成 PWA
* プロンプト実験用ツール
* 記事のアイキャッチ画像生成ツール
* デザイナー向けの簡易プロトタイプ

この場合、構成はかなりシンプルです。

```text
image-worker
  ↓
OpenAI / Replicate / fal.ai など
```

単一の Worker だけが API キーを使うなら、Workers Secrets で十分です。

通常の設定値は `wrangler.toml` または `wrangler.jsonc` の `vars` に置きます。

```toml
[vars]
APP_ENV = "production"
DEFAULT_IMAGE_MODEL = "gpt-image-2"
IMAGE_SIZE = "1024x1024"
MAX_IMAGES_PER_REQUEST = "1"
```

一方で、API キーは Secrets として登録します。

```bash
npx wrangler secret put OPENAI_API_KEY
```

Worker 側では、通常の環境変数も Secrets も `env` から参照できます。

```js
export default {
  async fetch(request, env) {
    const model = env.DEFAULT_IMAGE_MODEL;
    const apiKey = env.OPENAI_API_KEY;

    // ここで画像生成 API を呼び出す

    return new Response(`model: ${model}`);
  },
};
```

この段階で覚えておきたいことはシンプルです。

モデル名、画像サイズ、最大生成枚数のような値は通常の環境変数で管理します。API キーやトークンは Workers Secrets で管理します。

小さく始めるなら、この構成で十分です。

## ローカル開発では `.dev.vars` を使う

Cloudflare Workers をローカルで開発するときは、`.dev.vars` を使うことがあります。

```env
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxx
DEFAULT_IMAGE_MODEL=gpt-image-2
IMAGE_SIZE=1024x1024
```

`.dev.vars` はローカル開発用の値を置くファイルです。Git にコミットしないように、必ず `.gitignore` に追加します。

```gitignore
.dev.vars
```

本番環境では、API キーを `wrangler secret put` で登録します。

```bash
npx wrangler secret put OPENAI_API_KEY
```

ローカルと本番を分けることで、開発用 API キーと本番用 API キーを分離できます。

## ケース2: 複数の Workers で同じ画像生成 API を使う

次に、少し実践的な構成を考えます。

画像生成 AI をいろいろな用途で使い始めると、Worker が増えていきます。

たとえば、次のような構成です。

```text
thumbnail-worker
prompt-lab-worker
admin-image-worker
batch-generate-worker
        ↓
同じ OpenAI API / Replicate API を使う
```

この場合、各 Worker に同じ API キーを個別登録すると、管理が面倒になります。

たとえば、OpenAI API キーを更新したいとします。Worker ごとに Secret を登録していると、すべての Worker で更新作業が必要です。更新漏れが起きると、一部の Worker だけ古いキーを使い続けることになります。

```text
thumbnail-worker      OPENAI_API_KEY を登録
prompt-lab-worker     OPENAI_API_KEY を登録
admin-image-worker    OPENAI_API_KEY を登録
batch-generate-worker OPENAI_API_KEY を登録
```

Worker が少ないうちは問題になりません。しかし、数が増えるほど重複管理がつらくなります。

この段階で Cloudflare Secrets Store の価値が出てきます。

Secrets Store は、Cloudflare アカウント単位で Secrets を管理し、複数の Workers から利用するための仕組みです。

```text
Cloudflare Secrets Store
  - OPENAI_API_KEY
  - REPLICATE_API_TOKEN
  - FAL_API_KEY
        ↓
複数の Workers から参照
```

API キーを各 Worker に重複登録するのではなく、Secrets Store に集約して、必要な Worker に binding します。

考え方としては、次のようになります。

```text
thumbnail-worker
  ↓
Secrets Store の OPENAI_API_KEY を参照

prompt-lab-worker
  ↓
Secrets Store の OPENAI_API_KEY を参照

admin-image-worker
  ↓
Secrets Store の OPENAI_API_KEY を参照
```

複数 Workers で同じ API キーを使うなら、Secrets Store を使うことで更新作業や重複管理を減らせます。

## ケース3: チームで画像生成 AI ツールを運用する

チームで画像生成 AI ツールを運用する場合は、さらに考えることが増えます。

個人用ツールなら、自分だけが使えれば十分です。しかし、チームで使う場合は、誰が使えるのか、どれだけ使えるのか、どのモデルを使うのか、コストがどれだけ発生しているのかを管理する必要があります。

この場合、Cloudflare Workers だけでなく、Cloudflare Access、AI Gateway、Secrets Store、R2、D1 などを組み合わせる構成が考えられます。

```text
Cloudflare Access
  ↓
Cloudflare Workers
  ↓
Cloudflare AI Gateway
  ↓
OpenAI / Replicate / fal.ai / Workers AI
  ↓
R2 / D1 に保存
```

それぞれの役割は次のように分けられます。

```text
Cloudflare Access
  - 画像生成ツールにアクセスできる人を制限する

Cloudflare Workers
  - リクエストを受け取る
  - 入力を検証する
  - ユーザーごとの制限を確認する
  - 画像生成 API を呼び出す
  - 生成履歴を保存する

Cloudflare AI Gateway
  - AI API 呼び出しのログを取る
  - 利用量を分析する
  - レート制限をかける
  - キャッシュを使う
  - 必要に応じてモデルやプロバイダーを切り替える

Cloudflare Secrets Store
  - OpenAI / Replicate / fal.ai などの API キーを集中管理する

R2 / D1
  - 生成画像や生成履歴を保存する
```

画像生成 AI は、テキスト生成よりもコストが見えやすく膨らみやすい領域です。

ボタンを連打したり、フロントエンドの不具合で同じリクエストが繰り返し送られたりすると、意図しないコストが発生します。そのため、チーム運用では AI Gateway によるログ、分析、レート制限が重要になります。

また、チームメンバー全員に API キーを配る必要はありません。Cloudflare Access で利用者を制限し、Worker が裏側で API キーを使って画像生成 API を呼び出す構成にすれば、利用者は API キーを知らなくてもツールを使えます。

## AI Gateway は画像生成 API 管理と相性がよい

Cloudflare AI Gateway は、AI API 呼び出しを Cloudflare 経由で管理するための仕組みです。

画像生成 API と組み合わせると、次のような用途で役立ちます。

```text
ログ
  - いつ、どのモデルに、どのリクエストを送ったか確認する

分析
  - どのモデルをどれだけ使ったか確認する

レート制限
  - 連打や誤操作による使いすぎを防ぐ

キャッシュ
  - 同じ入力に対して同じ出力でよい場合に再利用する

プロバイダー切り替え
  - OpenAI、Workers AI、Replicate などを用途に応じて使い分ける
```

ただし、AI Gateway は画像を保存するためのストレージではありません。

生成画像を保存したい場合は、R2 や外部ストレージを使います。生成履歴を管理したい場合は、D1 や外部データベースを使います。

AI Gateway は、あくまで AI API 呼び出しの観測、制御、管理のための層です。

## キャッシュは便利だが画像生成では注意する

AI Gateway にはキャッシュ機能があります。

同じリクエストに対して同じレスポンスを返したい場合、キャッシュはコスト削減に役立ちます。

しかし、画像生成では注意が必要です。

画像生成では、同じプロンプトでも毎回違う画像がほしい場合があります。その場合、キャッシュが効くと期待と違う結果になります。

一方で、教材用のサンプル画像、デモ用画像、同じサムネイルの再取得など、同じ入力なら同じ出力でよい用途ではキャッシュが有効です。

画像生成 API でキャッシュを考える場合は、少なくとも次のような値をキャッシュキーに含める必要があります。

```text
- model
- prompt
- size
- quality
- style
- seed
- user_id または project_id
```

特に、ユーザーごとに生成結果を分けたい場合は、`user_id` や `project_id` を含めることが重要です。

## 使い分けのまとめ

Cloudflare Workers で画像生成 AI API を扱う場合、使い分けは次のように考えると整理しやすいです。

```text
通常の環境変数
  - 秘密ではない設定値を置く
  - モデル名、画像サイズ、最大生成枚数など

Workers Secrets
  - 単一 Worker で使う機密情報を置く
  - 個人用ツールや小規模な検証に向いている

Cloudflare Secrets Store
  - 複数 Workers で共有する機密情報を置く
  - API キーの集中管理に向いている

Cloudflare AI Gateway
  - AI API 呼び出しを観測、制御、管理する
  - ログ、分析、レート制限、キャッシュに使う

Cloudflare Access
  - 画像生成ツールを使える人を制限する
  - チーム内ツールや自分専用アプリに向いている

R2 / D1
  - 生成画像や生成履歴を保存する
```

小さく始めるなら、Cloudflare Workers と Workers Secrets だけで十分です。

```text
ブラウザー
  ↓
Cloudflare Workers
  ↓
画像生成 AI API
```

複数の Workers で同じ API キーを使うようになったら、Secrets Store を検討します。

```text
複数の Workers
  ↓
Cloudflare Secrets Store
  ↓
画像生成 AI API
```

チームで運用するなら、Access と AI Gateway も組み合わせます。

```text
Cloudflare Access
  ↓
Cloudflare Workers
  ↓
Cloudflare AI Gateway
  ↓
画像生成 AI API
```

## おわりに

画像生成 AI API を扱うと、Cloudflare Workers の環境変数、Secrets、Secrets Store の違いが理解しやすくなります。

単一 Worker であれば Workers Secrets で十分です。複数 Workers で同じ API キーを使うなら、Secrets Store による集中管理が便利です。チームで運用するなら、Cloudflare Access で利用者を制限し、AI Gateway で利用量やレート制限を管理する構成が現実的です。

最初から大げさな構成にする必要はありません。

まずは Workers Secrets で小さく始め、Workers が増えたら Secrets Store に移行し、チーム運用になったら Access や AI Gateway を組み合わせる。この段階的な考え方が、画像生成 AI API を安全に扱ううえで重要です。

Cloudflare Workers は、単なるサーバーレス実行環境としてだけでなく、AI API を安全に扱うための中継地点としても使えます。画像生成 AI のように API キー、コスト、利用者管理が絡む題材では、その価値が特にわかりやすいと思います。
