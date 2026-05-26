---

layout: ../../layouts/PostLayout.astro
title: "Cloudflare Workers で HTTP リクエスト処理を練習する：cURL から fetch へ進む JSON API 入門"
description: "Cloudflare Workers で JSON を返す小さな API を作り、cURL で GET・POST・ヘッダー・ボディ・ステータスコードを確認しながら、最後に JavaScript の fetch に置き換える練習記録。"
pubDate: 2026-05-26
tags:

* Cloudflare Workers
* Wrangler
* HTTP
* cURL
* JavaScript
* JSON API

---

## はじめに

Cloudflare Workers を使うと、サーバーを自分で用意しなくても HTTP リクエストを受け取り、レスポンスを返す小さな API を作ることができます。

今回は、Cloudflare Workers を題材にして、HTTP リクエスト処理の基本を練習します。

レスポンスは HTML ではなく JSON に限定します。最初は cURL でリクエストを送り、慣れてきたら JavaScript の `fetch()` に置き換える流れにします。

この記事の目的は、立派なアプリケーションを完成させることではありません。

HTTP の基本要素である、URL、メソッド、ヘッダー、ボディ、ステータスコードを、Cloudflare Workers の小さなコードを通して確認することです。

## 対象読者

この記事は、次のような人を想定しています。

* Cloudflare Workers を触り始めた人
* HTTP リクエストの仕組みを実際に手を動かして学びたい人
* cURL の使い方に慣れたい人
* JavaScript の `fetch()` をなんとなく使っているが、HTTP との対応関係を整理したい人
* JSON API の最小構成を作ってみたい人

この記事では、認証、Cookie、CSRF、CORS、データベース永続化などは扱いません。

まずは、HTTP リクエストを受け取り、JSON を返すところに集中します。

## 今回作るもの

今回は、Cloudflare Workers 上に次のような JSON API を作ります。

```text
GET  /
GET  /hello
GET  /greet?name=Masaki
GET  /method
POST /echo
GET  /todos
GET  /todos/:id
POST /todos
```

最初は小さな API から始め、最後に簡単な Todo API 風の形にまとめます。

ただし、今回の Todo API は学習用です。配列にデータを入れるだけなので、本格的な保存処理ではありません。

Cloudflare Workers でデータを永続化したい場合は、次の学習テーマとして D1、KV、Durable Objects などを扱うことになります。

## プロジェクトを作成する

まずは Worker プロジェクトを作成します。

```bash
npm create cloudflare@latest -- workers-http-json-practice
cd workers-http-json-practice
npm run dev
```

開発サーバーが起動すると、通常は次のようなローカル URL でアクセスできます。

```text
http://localhost:8787
```

別のターミナルを開いて、cURL で確認します。

```bash
curl -i http://localhost:8787/
```

`-i` オプションを付けると、レスポンスボディだけでなく、ステータス行やレスポンスヘッダーも表示されます。

HTTP の練習では、この `-i` を付けておくと理解しやすいです。

## 最初の JSON レスポンス

まずは、どのリクエストに対しても JSON を返す Worker にします。

`src/index.js` または `src/index.ts` を次のようにします。

```js
export default {
  async fetch(request) {
    return Response.json({
      message: "Hello from Cloudflare Workers",
      method: request.method,
      url: request.url,
    });
  },
};
```

確認します。

```bash
curl -i http://localhost:8787/
```

レスポンスの例です。

```http
HTTP/1.1 200 OK
content-type: application/json

{"message":"Hello from Cloudflare Workers","method":"GET","url":"http://localhost:8787/"}
```

ここで確認したいポイントは、次の3つです。

* `Response.json()` を使うと JSON レスポンスを返せる
* `request.method` で HTTP メソッドを取得できる
* `request.url` でリクエスト URL を取得できる

まずはこれだけで十分です。

## URL のパスで処理を分ける

次に、URL のパスによって処理を分けます。

```js
export default {
  async fetch(request) {
    const url = new URL(request.url);

    if (url.pathname === "/") {
      return Response.json({
        message: "HTTP JSON practice API",
      });
    }

    if (url.pathname === "/hello") {
      return Response.json({
        message: "Hello",
      });
    }

    return Response.json(
      {
        error: "Not Found",
        path: url.pathname,
      },
      {
        status: 404,
      },
    );
  },
};
```

cURL で確認します。

```bash
curl -i http://localhost:8787/
curl -i http://localhost:8787/hello
curl -i http://localhost:8787/not-found
```

`/` と `/hello` は `200 OK` になります。

一方で、存在しない `/not-found` にアクセスすると、`404 Not Found` を返します。

ここでは、次のことを練習しています。

```js
const url = new URL(request.url);
```

`request.url` は文字列です。そのため、パスやクエリパラメータを扱いやすくするために `new URL()` で URL オブジェクトに変換しています。

パスは次のように取り出します。

```js
url.pathname
```

`http://localhost:8787/hello` であれば、`url.pathname` は `/hello` になります。

## クエリパラメータを読む

次に、クエリパラメータを読みます。

`/greet?name=Masaki` のような URL にアクセスしたとき、`name` の値を読み取って JSON に含めます。

```js
export default {
  async fetch(request) {
    const url = new URL(request.url);

    if (url.pathname === "/greet") {
      const name = url.searchParams.get("name") ?? "guest";

      return Response.json({
        message: `Hello, ${name}`,
      });
    }

    return Response.json(
      {
        error: "Not Found",
      },
      {
        status: 404,
      },
    );
  },
};
```

確認します。

```bash
curl -i "http://localhost:8787/greet"
curl -i "http://localhost:8787/greet?name=Masaki"
```

`name` を指定しない場合は、`guest` を使います。

```json
{
  "message": "Hello, guest"
}
```

`name=Masaki` を指定した場合は、その値を使います。

```json
{
  "message": "Hello, Masaki"
}
```

クエリパラメータは次のように取得できます。

```js
url.searchParams.get("name")
```

この段階で、URL は単なる文字列ではなく、パスやクエリパラメータに分解して扱えるものだと理解できます。

## HTTP メソッドを確認する

次に、HTTP メソッドを確認します。

```js
export default {
  async fetch(request) {
    const url = new URL(request.url);

    if (url.pathname === "/method") {
      return Response.json({
        method: request.method,
      });
    }

    return Response.json(
      {
        error: "Not Found",
      },
      {
        status: 404,
      },
    );
  },
};
```

cURL でメソッドを変えながら試します。

```bash
curl -i http://localhost:8787/method
curl -i -X POST http://localhost:8787/method
curl -i -X PUT http://localhost:8787/method
curl -i -X DELETE http://localhost:8787/method
```

`curl` は、何も指定しなければ基本的に `GET` リクエストを送ります。

`-X POST` のように指定すると、HTTP メソッドを変更できます。

ここで重要なのは、HTTP メソッドは単なる説明ではなく、リクエストの一部として実際に送られているということです。

Cloudflare Workers では、次のように取得できます。

```js
request.method
```

## POST された JSON を読む

次に、リクエストボディを読みます。

`POST /echo` に JSON を送信し、Worker 側で受け取った内容をそのまま返します。

```js
export default {
  async fetch(request) {
    const url = new URL(request.url);

    if (url.pathname === "/echo" && request.method === "POST") {
      const body = await request.json();

      return Response.json({
        received: body,
      });
    }

    if (url.pathname === "/echo") {
      return Response.json(
        {
          error: "Method Not Allowed",
          allowed: ["POST"],
        },
        {
          status: 405,
        },
      );
    }

    return Response.json(
      {
        error: "Not Found",
      },
      {
        status: 404,
      },
    );
  },
};
```

cURL で JSON を送信します。

```bash
curl -i \
  -X POST http://localhost:8787/echo \
  -H "Content-Type: application/json" \
  -d '{"name":"Masaki","topic":"Cloudflare Workers"}'
```

ここで使っている cURL のオプションを整理します。

```text
-X POST
```

HTTP メソッドを `POST` にします。

```text
-H "Content-Type: application/json"
```

送信するボディが JSON であることを示すヘッダーを付けます。

```text
-d '{"name":"Masaki","topic":"Cloudflare Workers"}'
```

リクエストボディとして JSON 文字列を送信します。

Worker 側では、次のコードで JSON ボディを読み取ります。

```js
const body = await request.json();
```

この段階で、HTTP リクエストには少なくとも次の要素があると整理できます。

```text
URL
HTTP method
headers
body
```

cURL は、この4つを目で見える形で組み立てる練習に向いています。

## ステータスコードを意識する

API を作るときは、JSON の中身だけでなく、HTTP ステータスコードも重要です。

たとえば、存在しないパスには `404` を返しました。

```js
return Response.json(
  {
    error: "Not Found",
  },
  {
    status: 404,
  },
);
```

`/echo` に `GET` でアクセスした場合は、メソッドが違うので `405` を返しました。

```js
return Response.json(
  {
    error: "Method Not Allowed",
    allowed: ["POST"],
  },
  {
    status: 405,
  },
);
```

ステータスコードは、API の利用者に対して「何が起きたのか」を伝えるための重要な情報です。

初心者のうちは、まず次のステータスコードを意識するとよいです。

```text
200 OK
201 Created
400 Bad Request
404 Not Found
405 Method Not Allowed
500 Internal Server Error
```

## 簡単な Todo API にまとめる

最後に、ここまでの内容をまとめて、簡単な Todo API を作ります。

```js
const todos = [
  { id: 1, title: "Learn cURL", done: true },
  { id: 2, title: "Learn fetch", done: false },
];

export default {
  async fetch(request) {
    const url = new URL(request.url);

    if (url.pathname === "/") {
      return Response.json({
        name: "Workers HTTP JSON Practice API",
        endpoints: [
          "GET /todos",
          "GET /todos/:id",
          "POST /todos",
        ],
      });
    }

    if (url.pathname === "/todos" && request.method === "GET") {
      return Response.json({
        todos,
      });
    }

    if (url.pathname.startsWith("/todos/") && request.method === "GET") {
      const id = Number(url.pathname.split("/")[2]);
      const todo = todos.find((item) => item.id === id);

      if (!todo) {
        return Response.json(
          {
            error: "Todo not found",
          },
          {
            status: 404,
          },
        );
      }

      return Response.json({
        todo,
      });
    }

    if (url.pathname === "/todos" && request.method === "POST") {
      const body = await request.json();

      if (!body.title) {
        return Response.json(
          {
            error: "title is required",
          },
          {
            status: 400,
          },
        );
      }

      const todo = {
        id: todos.length + 1,
        title: body.title,
        done: false,
      };

      todos.push(todo);

      return Response.json(
        {
          todo,
        },
        {
          status: 201,
        },
      );
    }

    return Response.json(
      {
        error: "Not Found",
      },
      {
        status: 404,
      },
    );
  },
};
```

## cURL で Todo API を試す

一覧取得です。

```bash
curl -i http://localhost:8787/todos
```

1件取得です。

```bash
curl -i http://localhost:8787/todos/1
```

存在しない ID を指定します。

```bash
curl -i http://localhost:8787/todos/999
```

Todo を追加します。

```bash
curl -i \
  -X POST http://localhost:8787/todos \
  -H "Content-Type: application/json" \
  -d '{"title":"Practice HTTP requests"}'
```

`title` を指定しない場合のエラーも確認します。

```bash
curl -i \
  -X POST http://localhost:8787/todos \
  -H "Content-Type: application/json" \
  -d '{}'
```

この時点で、次の処理を一通り練習できます。

```text
GET で一覧を取得する
GET で1件を取得する
POST で JSON を送る
入力不足なら 400 を返す
存在しないデータなら 404 を返す
作成できたら 201 を返す
```

## cURL から fetch に置き換える

cURL で HTTP リクエストの形に慣れたら、同じことを JavaScript の `fetch()` に置き換えます。

まずは `GET /todos` です。

```js
const response = await fetch("http://localhost:8787/todos");
const data = await response.json();

console.log(response.status);
console.log(data);
```

次に `POST /todos` です。

```js
const response = await fetch("http://localhost:8787/todos", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    title: "Practice fetch",
  }),
});

const data = await response.json();

console.log(response.status);
console.log(data);
```

cURL と `fetch()` の対応関係は、次のように整理できます。

```text
curl -X POST
→ fetch(url, { method: "POST" })

-H "Content-Type: application/json"
→ headers: { "Content-Type": "application/json" }

-d '{"title":"Practice fetch"}'
→ body: JSON.stringify({ title: "Practice fetch" })
```

cURL で練習してから `fetch()` に進むと、`fetch()` のオプションが何を意味しているのか理解しやすくなります。

## なぜ最初に cURL を使うのか

JavaScript の `fetch()` から始めることもできます。

しかし、初心者向けの HTTP 練習では、最初に cURL を使うメリットがあります。

cURL では、URL、HTTP メソッド、ヘッダー、ボディをコマンド上で明示します。

そのため、HTTP リクエストの構造を意識しやすくなります。

一方で、最初からブラウザーやフロントエンドのコードを使うと、HTML、DOM、イベント処理、CORS、ブラウザーの開発者ツールなど、同時に気にすることが増えます。

今回の目的は HTTP リクエスト処理の練習です。

そのため、最初は cURL でリクエストそのものに集中し、慣れてから JavaScript の `fetch()` に置き換える流れが扱いやすいです。

## 今回あえて扱わないこと

今回の記事では、次のテーマは扱いません。

* Cookie
* セッション
* CSRF 対策
* CORS
* 認証
* データベース保存
* Cloudflare D1
* Cloudflare KV
* Durable Objects
* フロントエンド画面の作成

これらは重要なテーマですが、最初から入れると学習対象が広がりすぎます。

まずは、HTTP リクエストを受け取り、JSON を返す小さな API に集中します。

## 次の学習テーマ

今回の教材に慣れたら、次は次のようなテーマに進むとよいです。

```text
D1 を使って Todo を保存する
KV を使って簡単な設定値を保存する
CORS を設定してブラウザーから呼び出す
フォーム送信と JSON API の違いを比較する
Cookie とセッションを扱う
CSRF トークンの仕組みを学ぶ
認証付き API を作る
```

特に、Cloudflare Workers で Web アプリケーションを作るなら、HTTP、JSON API、Cookie、CSRF、CORS、認証の関係を順番に整理していくと理解しやすくなります。

## まとめ

今回は、Cloudflare Workers を使って HTTP リクエスト処理を練習しました。

最初に `Response.json()` で JSON を返し、URL のパス、クエリパラメータ、HTTP メソッド、リクエストボディ、ステータスコードを順番に確認しました。

cURL を使うと、HTTP リクエストの構造を目で見ながら練習できます。

その後で JavaScript の `fetch()` に置き換えると、`method`、`headers`、`body` などの設定が何を意味しているのか理解しやすくなります。

Cloudflare Workers は、こうした HTTP の基礎を練習する教材としてかなり扱いやすいです。

サーバー構築に時間をかけず、リクエストとレスポンスの処理に集中できるからです。

まずは小さな JSON API を作り、cURL で何度も試すところから始めるとよいでしょう。

## ハッシュタグ

#CloudflareWorkers #Wrangler #HTTP #cURL #JavaScript #JSONAPI #WebAPI #サーバーレス #プログラミング学習 #テックブログ
