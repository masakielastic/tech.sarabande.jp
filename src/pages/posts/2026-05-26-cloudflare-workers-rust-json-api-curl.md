---

layout: ../../layouts/PostLayout.astro
title: "Cloudflare Workers と Rust で作る、cURL から学ぶ JSON API 入門"
description: "Cloudflare Workers と Rust を使って、GET、POST、クエリ文字列、JSON ボディ、リクエストヘッダー、ステータスコードを cURL で確認しながら学ぶための HTTP リクエスト練習教材。"
pubDate: 2026-05-26
tags:

* Cloudflare Workers
* Rust
* WebAssembly
* HTTP
* cURL

---

## はじめに

Cloudflare Workers と Rust を使って、HTTP リクエストを処理する練習教材を作ります。

この記事では、最初から Workers KV、D1、R2、Durable Objects、認証、CORS の詳細には進みません。まずは、HTTP リクエストを受け取り、JSON レスポンスを返すところに集中します。

HTTP クライアントには、最初に cURL を使います。

ブラウザーや JavaScript の `fetch` から呼び出す前に、cURL でリクエストとレスポンスを直接見ることで、HTTP の基本を確認しやすくするためです。

この記事で意識するポイントは次のとおりです。

* HTTP メソッドを見る
* URL のパスを見る
* クエリ文字列を読む
* リクエストヘッダーを見る
* JSON ボディを読む
* JSON レスポンスを返す
* ステータスコードを変える

Rust や WebAssembly の細かい仕組みをすべて理解することよりも、まずは「リクエストが来て、処理して、レスポンスを返す」という流れを手で確認できるようにします。

## この教材で作るもの

今回作る API は、かなり小さなものです。

```txt
GET  /                  -> 動作確認
GET  /health            -> ヘルスチェック
GET  /hello?name=Rust   -> クエリ文字列の練習
POST /echo              -> JSON ボディの練習
GET  /headers           -> リクエストヘッダー確認
その他                  -> 404 JSON
```

レスポンスはすべて JSON に統一します。

たとえば、`GET /health` では次のような JSON を返します。

```json
{
  "ok": true,
  "message": "ok",
  "data": {}
}
```

存在しないパスにアクセスした場合も、HTML ではなく JSON を返します。

```json
{
  "ok": false,
  "message": "not found",
  "data": {}
}
```

API の練習では、成功時だけでなく失敗時のレスポンス形式もそろえておくと、後から JavaScript や別のクライアントで扱いやすくなります。

## 前提環境

この記事では、次のような道具を使います。

* Rust
* Cloudflare Workers
* workers-rs
* Wrangler
* cargo-generate
* cURL

それぞれの役割を簡単に整理します。

```txt
Rust:
  Workers 用コードを書く言語

WebAssembly:
  Rust Workers が実行される形式

workers-rs:
  Rust で Cloudflare Workers を書くための SDK / クレート

Wrangler:
  Workers のローカル開発・デプロイに使う CLI

cargo-generate:
  workers-rs のテンプレートを生成する補助ツール

cURL:
  HTTP リクエストを手元から送るための CLI
```

Cloudflare Workers と Rust の組み合わせでは、Rust のコードを WebAssembly にコンパイルして Workers 上で動かします。

ここで大事なのは、`cargo-generate` は Workers の実行環境そのものではなく、テンプレート生成のための補助ツールだということです。

## セットアップ

まず、Rust の WebAssembly ターゲットを追加します。

```bash
rustup target add wasm32-unknown-unknown
```

次に、テンプレート生成に使う `cargo-generate` をインストールします。

```bash
cargo +stable install cargo-generate --locked
```

ここでは、単に `cargo install cargo-generate` ではなく、`cargo +stable install cargo-generate --locked` としています。

理由は、`cargo-generate` のインストール中に依存クレートのビルドで失敗することがあるためです。

実際に、`rustc 1.94.1 (e408947bf 2026-03-25)` の環境では、`cargo install cargo-generate` の途中で `rhai` の大量のコンパイルエラーが発生することがありました。

そのときは、次のコマンドで解決できました。

```bash
cargo +stable install cargo-generate --locked
```

`+stable` は stable toolchain で `cargo` を実行する指定です。

`--locked` は、パッケージ側の `Cargo.lock` に基づいて依存関係を固定してビルドする指定です。

Rust 製 CLI ツールを `cargo install` するとき、依存関係の更新状況によってビルドに失敗することがあります。その場合、`--locked` を付けると解決することがあります。

## cargo-generate のインストールで詰まる場合

もし次のようなエラーが出た場合は、Cloudflare Workers のコードではなく、`cargo-generate` の依存関係のビルドで失敗している可能性があります。

```txt
error[E0433]: cannot find `rhai` in the crate root
  --> /home/masakielastic/.cargo/registry/src/index.crates.io-.../rhai-1.24.0/src/packages/time_basic.rs:27:1
   |
27 | #[export_module]
   | ^^^^^^^^^^^^^^^^ could not find `rhai` in the list of imported crates
```

この時点では、まだ自分の Workers コードを書いていません。

失敗しているのは、テンプレート生成ツールである `cargo-generate` のインストール中です。

そのため、まずは次のようにインストールし直します。

```bash
cargo +stable install cargo-generate --locked
```

これで解決しない場合は、少し前の Rust toolchain を明示して試す方法もあります。

```bash
rustup toolchain install 1.93.0
cargo +1.93.0 install cargo-generate --locked
```

今回の記事では、`cargo +stable install cargo-generate --locked` で解決した前提で進めます。

## プロジェクトを作成する

`cargo-generate` が入ったら、workers-rs のテンプレートからプロジェクトを作成します。

```bash
cargo generate cloudflare/workers-rs
```

テンプレートを選ぶ画面が出たら、まずは最小構成のテンプレートを選びます。

ここでは、仮にプロジェクト名を `workers-rust-json-api` とします。

```bash
cd workers-rust-json-api
```

生成されたプロジェクトには、だいたい次のようなファイルがあります。

```txt
Cargo.toml
wrangler.toml
src/lib.rs
```

この記事では、主に `src/lib.rs` を編集します。

## ローカルで起動する

Cloudflare Workers をローカルで動かすには、Wrangler を使います。

```bash
npx wrangler dev
```

起動できたら、別のターミナルから cURL でアクセスします。

```bash
curl -i http://localhost:8787/
```

`-i` を付けると、レスポンスボディだけでなく、ステータス行やレスポンスヘッダーも表示されます。

この記事では、基本的に `curl -i` を使います。

確認したいのは次の3つです。

* HTTP ステータスコード
* レスポンスヘッダー
* JSON ボディ

HTTP の練習では、JSON の中身だけでなく、`200`、`404`、`content-type` なども見るようにします。

## 依存関係を確認する

`Cargo.toml` に、`worker`、`serde`、`serde_json` が入っていることを確認します。

テンプレートによって最初から入っている場合もありますが、JSON API の練習では次の依存関係を使います。

```toml
[dependencies]
worker = "0.6"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

`worker` は、Cloudflare Workers 用の `Request` や `Response` などを扱うために使います。

`serde` は、Rust の構造体と JSON を対応させるために使います。

`serde_json` は、JSON のシリアライズやデシリアライズに使います。

テンプレート作成時点で `worker` のバージョンが異なる場合は、生成された設定を優先してください。

## 最初の JSON レスポンスを返す

まずは、`GET /` にアクセスしたときに JSON を返すところから始めます。

`src/lib.rs` を次のようにします。

```rust
use serde::Serialize;
use worker::*;

#[derive(Serialize)]
struct SimpleResponse {
    ok: bool,
    message: String,
}

#[event(fetch)]
pub async fn main(_req: Request, _env: Env, _ctx: Context) -> Result<Response> {
    let body = SimpleResponse {
        ok: true,
        message: "Cloudflare Workers Rust API is running".to_string(),
    };

    Response::from_json(&body)
}
```

起動します。

```bash
npx wrangler dev
```

cURL で確認します。

```bash
curl -i http://localhost:8787/
```

期待するレスポンスは次のような形です。

```json
{
  "ok": true,
  "message": "Cloudflare Workers Rust API is running"
}
```

ここで大事なのは、`#[event(fetch)]` が Worker の入口になることです。

HTTP リクエストが来ると、この `main` 関数が呼ばれ、`Response` を返します。

## パスで処理を分ける

次に、URL のパスによって処理を分けます。

`/` と `/health` と、それ以外を分けます。

```rust
use serde::Serialize;
use worker::*;

#[derive(Serialize)]
struct ApiResponse<T> {
    ok: bool,
    message: String,
    data: T,
}

#[derive(Serialize)]
struct EmptyData {}

fn json_response<T: Serialize>(status: u16, message: &str, data: T) -> Result<Response> {
    let body = ApiResponse {
        ok: status < 400,
        message: message.to_string(),
        data,
    };

    let mut response = Response::from_json(&body)?;
    response.headers_mut().set("content-type", "application/json; charset=utf-8")?;
    response = response.with_status(status);

    Ok(response)
}

#[event(fetch)]
pub async fn main(req: Request, _env: Env, _ctx: Context) -> Result<Response> {
    let url = req.url()?;
    let path = url.path();
    let method = req.method();

    match (method, path) {
        (Method::Get, "/") => {
            json_response(200, "Cloudflare Workers Rust API is running", EmptyData {})
        }
        (Method::Get, "/health") => {
            json_response(200, "ok", EmptyData {})
        }
        _ => {
            json_response(404, "not found", EmptyData {})
        }
    }
}
```

確認します。

```bash
curl -i http://localhost:8787/
```

```bash
curl -i http://localhost:8787/health
```

```bash
curl -i http://localhost:8787/not-found
```

`/not-found` では、ステータスコードが `404` になり、JSON の `ok` が `false` になることを確認します。

```json
{
  "ok": false,
  "message": "not found",
  "data": {}
}
```

ここでは、HTTP パスとステータスコードの関係を確認できます。

## クエリ文字列を読む

次に、`GET /hello?name=Rust` を作ります。

URL の `?` 以降がクエリ文字列です。

```txt
/hello?name=Rust
```

この場合、`name` というキーに `Rust` という値が入っています。

`HelloData` を追加します。

```rust
#[derive(Serialize)]
struct HelloData {
    name: String,
}
```

`match` に `/hello` を追加します。

```rust
(Method::Get, "/hello") => {
    let name = url
        .query_pairs()
        .find(|(key, _)| key == "name")
        .map(|(_, value)| value.to_string())
        .unwrap_or_else(|| "world".to_string());

    json_response(200, "hello", HelloData { name })
}
```

確認します。

```bash
curl -i "http://localhost:8787/hello?name=Rust"
```

期待する JSON は次の形です。

```json
{
  "ok": true,
  "message": "hello",
  "data": {
    "name": "Rust"
  }
}
```

クエリ文字列を付けない場合も確認します。

```bash
curl -i http://localhost:8787/hello
```

この場合は、デフォルト値として `world` を返します。

```json
{
  "ok": true,
  "message": "hello",
  "data": {
    "name": "world"
  }
}
```

## POST で JSON ボディを送る

次に、`POST /echo` を作ります。

これは、クライアントから送られてきた JSON をそのまま返す練習です。

入力用の構造体と、レスポンス用の構造体を追加します。

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct EchoInput {
    text: String,
}

#[derive(Serialize)]
struct EchoData {
    received: String,
}
```

`match` に `/echo` を追加します。

```rust
(Method::Post, "/echo") => {
    let input: EchoInput = req.json().await?;

    json_response(
        200,
        "echo",
        EchoData {
            received: input.text,
        },
    )
}
```

cURL で POST します。

```bash
curl -i \
  -X POST http://localhost:8787/echo \
  -H "content-type: application/json" \
  -d '{"text":"Hello from curl"}'
```

期待する JSON は次の形です。

```json
{
  "ok": true,
  "message": "echo",
  "data": {
    "received": "Hello from curl"
  }
}
```

ここで確認したいのは、cURL の各オプションです。

```txt
-X POST:
  HTTP メソッドを POST にする

-H "content-type: application/json":
  リクエストボディが JSON であることを伝える

-d '{"text":"Hello from curl"}':
  リクエストボディを送る
```

GET では主に URL から情報を受け取りました。

POST では、リクエストボディから JSON を受け取ります。

## リクエストヘッダーを確認する

次に、リクエストヘッダーを確認するための `/headers` を作ります。

レスポンス用の構造体を追加します。

```rust
#[derive(Serialize)]
struct HeaderData {
    x_practice_token: Option<String>,
}
```

`match` に `/headers` を追加します。

```rust
(Method::Get, "/headers") => {
    let token = req
        .headers()
        .get("x-practice-token")?;

    json_response(
        200,
        "headers",
        HeaderData {
            x_practice_token: token,
        },
    )
}
```

cURL で独自ヘッダーを付けてアクセスします。

```bash
curl -i \
  -H "x-practice-token: hello" \
  http://localhost:8787/headers
```

期待する JSON は次の形です。

```json
{
  "ok": true,
  "message": "headers",
  "data": {
    "x_practice_token": "hello"
  }
}
```

ヘッダーは、本文とは別に送られるメタ情報です。

認証トークン、Content-Type、User-Agent など、HTTP ではさまざまな情報がヘッダーとして送られます。

## エラーも JSON で返す

API では、成功時だけでなく、失敗時の形式も大事です。

今回の教材では、存在しないパスにアクセスした場合、次のように JSON で返します。

```bash
curl -i http://localhost:8787/unknown
```

```json
{
  "ok": false,
  "message": "not found",
  "data": {}
}
```

ステータスコードは `404` です。

`curl -i` を使うと、JSON の中身だけでなく、ステータスコードも確認できます。

```txt
HTTP/1.1 404 Not Found
```

API の練習では、レスポンスボディだけを見て満足するのではなく、ステータスコードも一緒に確認する習慣をつけるとよいです。

## 完成コード

ここまでの内容をまとめた `src/lib.rs` は次のようになります。

```rust
use serde::{Deserialize, Serialize};
use worker::*;

#[derive(Serialize)]
struct ApiResponse<T> {
    ok: bool,
    message: String,
    data: T,
}

#[derive(Serialize)]
struct EmptyData {}

#[derive(Serialize)]
struct HelloData {
    name: String,
}

#[derive(Serialize, Deserialize)]
struct EchoInput {
    text: String,
}

#[derive(Serialize)]
struct EchoData {
    received: String,
}

#[derive(Serialize)]
struct HeaderData {
    x_practice_token: Option<String>,
}

fn json_response<T: Serialize>(status: u16, message: &str, data: T) -> Result<Response> {
    let body = ApiResponse {
        ok: status < 400,
        message: message.to_string(),
        data,
    };

    let mut response = Response::from_json(&body)?;
    response
        .headers_mut()
        .set("content-type", "application/json; charset=utf-8")?;
    response = response.with_status(status);

    Ok(response)
}

#[event(fetch)]
pub async fn main(mut req: Request, _env: Env, _ctx: Context) -> Result<Response> {
    let url = req.url()?;
    let path = url.path();
    let method = req.method();

    match (method, path) {
        (Method::Get, "/") => {
            json_response(200, "Cloudflare Workers Rust API is running", EmptyData {})
        }

        (Method::Get, "/health") => {
            json_response(200, "ok", EmptyData {})
        }

        (Method::Get, "/hello") => {
            let name = url
                .query_pairs()
                .find(|(key, _)| key == "name")
                .map(|(_, value)| value.to_string())
                .unwrap_or_else(|| "world".to_string());

            json_response(200, "hello", HelloData { name })
        }

        (Method::Post, "/echo") => {
            let input: EchoInput = req.json().await?;

            json_response(
                200,
                "echo",
                EchoData {
                    received: input.text,
                },
            )
        }

        (Method::Get, "/headers") => {
            let token = req.headers().get("x-practice-token")?;

            json_response(
                200,
                "headers",
                HeaderData {
                    x_practice_token: token,
                },
            )
        }

        _ => {
            json_response(404, "not found", EmptyData {})
        }
    }
}
```

## cURL 練習まとめ

最後に、今回の API を cURL で順番に確認します。

動作確認です。

```bash
curl -i http://localhost:8787/
```

ヘルスチェックです。

```bash
curl -i http://localhost:8787/health
```

クエリ文字列を送ります。

```bash
curl -i "http://localhost:8787/hello?name=Rust"
```

クエリ文字列なしでも確認します。

```bash
curl -i http://localhost:8787/hello
```

JSON ボディを POST します。

```bash
curl -i \
  -X POST http://localhost:8787/echo \
  -H "content-type: application/json" \
  -d '{"text":"Hello from curl"}'
```

リクエストヘッダーを送ります。

```bash
curl -i \
  -H "x-practice-token: hello" \
  http://localhost:8787/headers
```

存在しないパスを確認します。

```bash
curl -i http://localhost:8787/not-found
```

`curl -i` では、毎回次の3つを確認します。

* ステータスコード
* レスポンスヘッダー
* JSON ボディ

## 発展課題

今回の教材に慣れたら、次のような課題に進むとよいです。

* `GET /calc/add?a=1&b=2` を作る
* `POST /users` で `name` と `age` を受け取る
* `GET /user-agent` で `User-Agent` を返す
* `POST /validate` で空文字をエラーにする
* `PUT` や `DELETE` を追加して HTTP メソッドの違いを見る
* `OPTIONS` リクエストを見て CORS の入口を学ぶ
* JavaScript の `fetch` から同じ API を呼ぶ

たとえば、`GET /calc/add?a=1&b=2` は、クエリ文字列の練習に向いています。

`POST /users` は、JSON ボディの練習に向いています。

`GET /user-agent` は、リクエストヘッダーの練習に向いています。

このように、HTTP の部品ごとに小さな API を増やしていくと、Cloudflare Workers と Rust の両方に慣れやすくなります。

## まとめ

今回は、Cloudflare Workers と Rust を使って、cURL から学ぶ JSON API の練習教材を作りました。

作った API は小さいですが、HTTP の基本を確認するには十分です。

この記事で扱った内容は次のとおりです。

* `GET /` で JSON レスポンスを返す
* `GET /health` でヘルスチェックを作る
* `GET /hello?name=Rust` でクエリ文字列を読む
* `POST /echo` で JSON ボディを読む
* `GET /headers` でリクエストヘッダーを読む
* 存在しないパスに 404 JSON を返す
* cURL の `-i`、`-X`、`-H`、`-d` を使う

Cloudflare Workers と Rust の学習では、最初から高度な設計に進むよりも、まず HTTP リクエストとレスポンスを手で観察するのがよいと思います。

Rust を書く。

Wrangler で起動する。

cURL でリクエストを送る。

ステータスコード、ヘッダー、JSON ボディを見る。

この繰り返しだけでも、HTTP API の理解はかなり進みます。

次の段階では、JavaScript の `fetch` から今回の Rust Workers API を呼び出す練習に進むとよさそうです。
