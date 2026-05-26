---

layout: ../../layouts/PostLayout.astro
title: "Cloudflare Workers で学ぶ CSRF トークン入門：フォーム埋め込み方式からヘッダー方式まで"
description: "Cloudflare Workers のシンプルなフォームを題材に、CSRF トークンの発行、hidden input 方式、X-CSRF-Token ヘッダー方式、Cookie 属性の基本を初心者向けに整理します。"
pubDate: 2026-05-26
tags:
  - Cloudflare Workers
  - Security
  - CSRF
  - Web Crypto API
  - JavaScript
---

## はじめに

Web アプリケーションでログイン機能や管理画面を作るとき、避けて通れない話題のひとつに CSRF 対策があります。

CSRF は Cross-Site Request Forgery の略で、日本語では「クロスサイトリクエストフォージェリ」と呼ばれます。

名前だけを見ると難しそうですが、中心にある考え方はそれほど複雑ではありません。

ざっくり言うと、CSRF は「ログイン済みユーザーのブラウザーに、本人が意図していないリクエストを送らせる攻撃」です。

この記事では、Cloudflare Workers を題材にして、CSRF 対策の基本である CSRF トークンについて整理します。

最初から SPA や複雑な認証基盤を扱うと、Cookie、HTTP ヘッダー、リクエストボディ、JavaScript、CORS などの話が一気に出てきます。初心者には少し追いかけにくくなります。

そこで、この記事ではまず通常の HTML フォームを使います。

```text
フォームを表示する
↓
CSRF トークンを hidden input に埋め込む
↓
POST 時にフォームデータとして送る
↓
Cloudflare Workers 側で検証する
```

この流れを理解したうえで、後半では `fetch()` と `X-CSRF-Token` ヘッダーを使う方式も紹介します。

## CSRF とは何か

たとえば、あるサイトにプロフィール更新フォームがあるとします。

```html
<form action="/profile" method="post">
  <label>
    表示名:
    <input name="display_name">
  </label>

  <button>更新する</button>
</form>
```

ログイン済みユーザーがこのフォームを送信すると、サーバーは Cookie を見て「このユーザーはログインしている」と判断し、プロフィールを更新します。

ここで問題になるのは、Cookie はブラウザーが自動的に送るという点です。

悪意あるサイトは、次のようなフォームを用意できます。

```html
<form action="https://example.com/profile" method="post">
  <input type="hidden" name="display_name" value="attacker">
</form>

<script>
  document.forms[0].submit();
</script>
```

ユーザーが `example.com` にログインしたまま、この悪意あるページを開いたとします。

するとブラウザーは、`https://example.com/profile` に対するリクエストを送るとき、`example.com` 用の Cookie を自動的に付けます。

サーバー側が Cookie だけを見て「ログイン済みだから本人の操作だ」と判断していると、本人が意図していないプロフィール変更が実行されてしまう可能性があります。

重要なのは、次の点です。

```text
Cookie が送られたこと
=
本人がその操作を意図したこと

ではない
```

CSRF 対策では、「そのリクエストが、自分のサイトで表示した画面から送られたものか」を確認する必要があります。

## CSRF トークンの考え方

CSRF トークンは、フォームを表示したサーバーが発行するランダムな値です。

イメージとしては、フォームに埋め込まれた「合言葉」です。

流れは次のようになります。

```text
サーバーがランダムなトークンを作る
↓
フォームに埋め込む
↓
ユーザーがフォームを送信する
↓
サーバーがトークンを検証する
↓
正しければ処理する
↓
間違っていれば拒否する
```

攻撃者は、外部サイトから勝手にフォームを送信させることはできます。

しかし、正しい CSRF トークンを知らなければ、サーバー側の検証を通過できません。

この記事では、まず学習用として `crypto.randomUUID()` を使って CSRF トークンを作ります。

```js
const csrfToken = crypto.randomUUID();
```

`crypto.randomUUID()` は、暗号学的に安全な乱数に基づく UUID v4 を生成します。

より実務寄りには `crypto.getRandomValues()` で 32 バイト程度のランダム値を作る方法もありますが、最初の教材としては `crypto.randomUUID()` の方がコードが短く、流れを追いやすくなります。

## まずは hidden input 方式で理解する

初心者向けには、最初に hidden input 方式で CSRF トークンを説明するのがわかりやすいです。

hidden input 方式では、CSRF トークンをフォームの中に埋め込みます。

```html
<form method="post" action="/profile">
  <input type="hidden" name="csrf_token" value="abc123">

  <label>
    表示名:
    <input name="display_name">
  </label>

  <button>更新する</button>
</form>
```

`csrf_token` は、ユーザーに入力してもらう値ではありません。

サーバーがフォームを表示するときに埋め込む値です。

フォームを送信すると、`display_name` と `csrf_token` が一緒に送られます。

```text
display_name=masaki
csrf_token=abc123
```

この方式のメリットは、説明対象がフォームの中にまとまることです。

初心者は、まず次のように理解できます。

```text
フォームには、表示名だけでなく CSRF トークンも入っている
送信すると、それらがまとめてサーバーに送られる
サーバーは CSRF トークンを確認してから更新する
```

HTTP ヘッダーや JSON API の話を同時に追わなくてよいので、CSRF の本質に集中しやすくなります。

## Cloudflare Workers でフォームを表示する

ここからは、Cloudflare Workers で最小限のコードを書いてみます。

まず、リクエストのパスとメソッドで処理を分けます。

```js
export default {
  async fetch(request) {
    const url = new URL(request.url);

    if (url.pathname === "/profile" && request.method === "GET") {
      return showForm();
    }

    if (url.pathname === "/profile" && request.method === "POST") {
      return updateProfile(request);
    }

    return new Response("Not Found", { status: 404 });
  },
};
```

`GET /profile` ではフォームを表示します。

`POST /profile` ではフォーム送信を受け取ります。

次に、フォームを表示する `showForm()` を書きます。

```js
function showForm() {
  const csrfToken = crypto.randomUUID();

  const html = `<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8">
  <title>プロフィール更新</title>
</head>
<body>
  <h1>プロフィール更新</h1>

  <form method="post" action="/profile">
    <input type="hidden" name="csrf_token" value="${escapeHtml(csrfToken)}">

    <label>
      表示名:
      <input name="display_name">
    </label>

    <button>更新する</button>
  </form>
</body>
</html>`;

  return new Response(html, {
    headers: {
      "Content-Type": "text/html; charset=utf-8",
      "Set-Cookie": `csrf_token=${csrfToken}; Path=/; Secure; SameSite=Lax`,
    },
  });
}
```

ここでは、同じ CSRF トークンを 2 か所に入れています。

```text
hidden input
Cookie
```

フォーム送信時には、hidden input の値がフォームデータとして送られます。

一方、Cookie はブラウザーが自動的に送ります。

Worker 側では、この 2 つの値を比較します。

この構成は、学習用としてわかりやすい Double Submit Cookie 方式に近い形です。

## POST を受け取ってトークンを検証する

次に、POST を受け取る `updateProfile()` を書きます。

```js
async function updateProfile(request) {
  const formData = await request.formData();

  const tokenFromForm = formData.get("csrf_token");
  const displayName = formData.get("display_name");

  const cookieHeader = request.headers.get("Cookie") ?? "";
  const tokenFromCookie = getCookie(cookieHeader, "csrf_token");

  if (!tokenFromForm || !tokenFromCookie) {
    return new Response("CSRF token is missing", { status: 403 });
  }

  if (tokenFromForm !== tokenFromCookie) {
    return new Response("Invalid CSRF token", { status: 403 });
  }

  return new Response(`更新しました: ${displayName}`, {
    headers: {
      "Content-Type": "text/plain; charset=utf-8",
    },
  });
}
```

フォームデータは `request.formData()` で取得できます。

```js
const formData = await request.formData();
```

hidden input の値は、通常のフォーム項目と同じように取得できます。

```js
const tokenFromForm = formData.get("csrf_token");
```

Cookie は、`Cookie` ヘッダーから取得します。

```js
const cookieHeader = request.headers.get("Cookie") ?? "";
const tokenFromCookie = getCookie(cookieHeader, "csrf_token");
```

補助関数 `getCookie()` は次のように書けます。

```js
function getCookie(cookieHeader, name) {
  const cookies = cookieHeader.split(";");

  for (const cookie of cookies) {
    const [key, ...valueParts] = cookie.trim().split("=");

    if (key === name) {
      return valueParts.join("=");
    }
  }

  return null;
}
```

最後に、HTML に値を埋め込むためのエスケープ関数も用意しておきます。

```js
function escapeHtml(value) {
  return value
    .replaceAll("&", "&amp;")
    .replaceAll('"', "&quot;")
    .replaceAll("<", "&lt;")
    .replaceAll(">", "&gt;");
}
```

ここまでをまとめると、Worker 側では次のことをしています。

```text
フォームから csrf_token を受け取る
Cookie から csrf_token を受け取る
2 つを比較する
一致しなければ 403 を返す
一致すれば更新処理に進む
```

この記事のサンプルでは実際の DB 更新はしていません。

本来は、CSRF トークンの検証に通った後で、ログイン中ユーザーのプロフィールを更新します。

## 完成コード

学習用の完成コードをまとめると、次のようになります。

```js
export default {
  async fetch(request) {
    const url = new URL(request.url);

    if (url.pathname === "/profile" && request.method === "GET") {
      return showForm();
    }

    if (url.pathname === "/profile" && request.method === "POST") {
      return updateProfile(request);
    }

    return new Response("Not Found", { status: 404 });
  },
};

function showForm() {
  const csrfToken = crypto.randomUUID();

  const html = `<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8">
  <title>プロフィール更新</title>
</head>
<body>
  <h1>プロフィール更新</h1>

  <form method="post" action="/profile">
    <input type="hidden" name="csrf_token" value="${escapeHtml(csrfToken)}">

    <label>
      表示名:
      <input name="display_name">
    </label>

    <button>更新する</button>
  </form>
</body>
</html>`;

  return new Response(html, {
    headers: {
      "Content-Type": "text/html; charset=utf-8",
      "Set-Cookie": `csrf_token=${csrfToken}; Path=/; Secure; SameSite=Lax`,
    },
  });
}

async function updateProfile(request) {
  const formData = await request.formData();

  const tokenFromForm = formData.get("csrf_token");
  const displayName = formData.get("display_name");

  const cookieHeader = request.headers.get("Cookie") ?? "";
  const tokenFromCookie = getCookie(cookieHeader, "csrf_token");

  if (!tokenFromForm || !tokenFromCookie) {
    return new Response("CSRF token is missing", { status: 403 });
  }

  if (tokenFromForm !== tokenFromCookie) {
    return new Response("Invalid CSRF token", { status: 403 });
  }

  return new Response(`更新しました: ${displayName}`, {
    headers: {
      "Content-Type": "text/plain; charset=utf-8",
    },
  });
}

function getCookie(cookieHeader, name) {
  const cookies = cookieHeader.split(";");

  for (const cookie of cookies) {
    const [key, ...valueParts] = cookie.trim().split("=");

    if (key === name) {
      return valueParts.join("=");
    }
  }

  return null;
}

function escapeHtml(value) {
  return value
    .replaceAll("&", "&amp;")
    .replaceAll('"', "&quot;")
    .replaceAll("<", "&lt;")
    .replaceAll(">", "&gt;");
}
```

## `crypto.randomUUID()` を使う理由

このサンプルでは、CSRF トークンの生成に `crypto.randomUUID()` を使いました。

```js
const csrfToken = crypto.randomUUID();
```

理由は、コードが短く、Cloudflare Workers でそのまま使いやすいからです。

CSRF トークンには、推測されにくいランダムな値が必要です。

悪い例は次のようなものです。

```js
const csrfToken = Date.now().toString();
```

これは時刻ベースなので予測されやすいです。

次のようなコードも避けるべきです。

```js
const csrfToken = Math.random().toString();
```

`Math.random()` は暗号用途の乱数ではありません。

Cloudflare Workers では Web Crypto API が使えるため、学習用には次のように書くのが簡単です。

```js
const csrfToken = crypto.randomUUID();
```

ただし、実務寄りに「CSRF トークン生成関数」として明示したい場合は、`crypto.getRandomValues()` でランダムバイト列を作る方法もあります。

```js
function generateCsrfToken() {
  const bytes = new Uint8Array(32);
  crypto.getRandomValues(bytes);

  return [...bytes]
    .map((byte) => byte.toString(16).padStart(2, "0"))
    .join("");
}
```

この場合は、32 バイト、つまり 256 bit のランダム値になります。

整理すると、次のように使い分けるとよいでしょう。

```text
crypto.randomUUID()
  コードが短い
  説明しやすい
  チュートリアル向き

crypto.getRandomValues()
  トークン生成としてより直接的
  長さを自由に決められる
  実務寄り
```

## hidden input 方式のメリット

hidden input 方式は、古いだけの方式ではありません。

HTML フォーム中心の画面では、今でも自然な方式です。

初心者向けの教材としても優れています。

理由は、CSRF トークンがフォーム要素の中に見えるからです。

```html
<form method="post" action="/profile">
  <input type="hidden" name="csrf_token" value="abc123">
  <input name="display_name">
  <button>更新する</button>
</form>
```

このコードを見れば、フォーム送信時に `display_name` と `csrf_token` が一緒に送られることを理解しやすくなります。

一方、HTTP ヘッダー方式では、HTML、JavaScript、HTTP ヘッダー、JSON ボディ、Cookie をまたいで追いかける必要があります。

最初の学習では、これは少し大変です。

hidden input 方式には、学習者の注意を散漫にしないメリットがあります。

```text
HTML フォームを見る
↓
hidden input を見る
↓
POST される値を見る
↓
request.formData() で受け取る
```

流れが素直です。

そのため、CSRF トークンの最初の説明では、hidden input 方式から入るのが堅実です。

## 次の段階：HTTP ヘッダー方式

hidden input 方式を理解したら、次に HTTP ヘッダー方式を学ぶとよいです。

特に、`fetch()` で JSON API にリクエストを送る構成では、CSRF トークンを HTTP ヘッダーで送る方法がよく使われます。

たとえば、HTML に CSRF トークンを埋め込んでおきます。

```html
<meta name="csrf-token" content="abc123">
```

JavaScript 側でそれを読み取ります。

```js
const csrfToken = document
  .querySelector('meta[name="csrf-token"]')
  .getAttribute("content");
```

そして、`fetch()` の `headers` に入れて送信します。

```js
await fetch("/profile", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "X-CSRF-Token": csrfToken,
  },
  body: JSON.stringify({
    display_name: "masaki",
  }),
});
```

この場合、CSRF トークンはリクエストボディではなく、HTTP ヘッダーとして送られます。

```http
POST /profile
Content-Type: application/json
X-CSRF-Token: abc123

{"display_name":"masaki"}
```

Cloudflare Workers 側では、次のように取得できます。

```js
const tokenFromHeader = request.headers.get("X-CSRF-Token");
```

JSON ボディは `request.json()` で読みます。

```js
const body = await request.json();
const displayName = body.display_name;
```

## hidden input 方式とヘッダー方式の比較

hidden input 方式とヘッダー方式は、どちらか一方だけが正解というものではありません。

用途が違います。

```text
hidden input 方式
  HTML フォーム向き
  JavaScript なしでも使える
  フォームデータとして送る
  初心者が流れを追いやすい

HTTP ヘッダー方式
  fetch() や JSON API 向き
  SPA と相性がよい
  リクエストのメタ情報として送る
  フォームデータとトークンを分けやすい
```

初心者に説明するときは、まず hidden input 方式の方がわかりやすいです。

理由は、フォームの中に必要な要素がまとまっているからです。

一方、HTTP ヘッダー方式では、次の要素を同時に追う必要があります。

```text
HTML の meta タグ
JavaScript の DOM 読み取り
fetch() の headers
JSON の body
Cookie
Worker の request.headers.get()
Worker の request.json()
```

CSRF の本質に入る前に、HTTP のヘッダーとボディ、JavaScript の `fetch()`、Cookie の扱いで混乱しやすくなります。

そのため、記事や教材では次の順番が自然です。

```text
1. hidden input 方式で CSRF トークンの基本を理解する
2. fetch() を使う API ではヘッダー方式も使えると理解する
3. Cookie 属性や Origin チェックなどの補強策に進む
```

## HTTP ヘッダー方式のメリット

ヘッダー方式にも、もちろんメリットがあります。

特に API 設計では、フォームの値とリクエストのメタ情報を分けやすくなります。

```text
リクエストボディ
  ユーザーが送るデータ

HTTP ヘッダー
  リクエストに関するメタ情報
```

CSRF トークンを `X-CSRF-Token` ヘッダーに入れると、JSON ボディには本来の更新内容だけを入れられます。

```json
{
  "display_name": "masaki"
}
```

また、外部サイトの通常の HTML フォーム送信では、`X-CSRF-Token` のような任意のカスタムヘッダーを付けることはできません。

悪意あるサイトは、次のようなフォームを作ることはできます。

```html
<form action="https://example.com/profile" method="post">
  <input type="hidden" name="display_name" value="attacker">
</form>
```

しかし、この通常のフォーム送信では、次のようなヘッダーを自由に付けることはできません。

```http
X-CSRF-Token: 正しいトークン
```

そのため、JSON API や SPA では、CSRF トークンをヘッダーで送る方式がよく使われます。

ただし、注意点もあります。

ヘッダー方式が安全なのは、単に「ヘッダーだから」ではありません。

重要なのは、推測困難なランダムトークンを発行し、それをサーバー側で正しく検証することです。

## Cookie 属性も確認する

CSRF の話では、Cookie の属性も重要です。

今回のサンプルでは、CSRF トークン用 Cookie を次のように発行しました。

```http
Set-Cookie: csrf_token=...; Path=/; Secure; SameSite=Lax
```

`Secure` は、HTTPS のときだけ Cookie を送る指定です。

`SameSite=Lax` は、クロスサイトリクエストで Cookie が送られる場面を制限する指定です。

ログインセッション用の Cookie では、通常 `HttpOnly` も付けます。

```http
Set-Cookie: session_id=...; Path=/; HttpOnly; Secure; SameSite=Lax
```

`HttpOnly` を付けると、JavaScript から Cookie を読めなくなります。

セッション Cookie は、JavaScript から読ませる必要がないため、`HttpOnly` を付けるのが基本です。

一方、CSRF トークン用 Cookie は、設計によって扱いが変わります。

```text
session_id
  認証用
  JavaScript から読ませない
  HttpOnly を付ける

csrf_token
  CSRF 検証用
  hidden input 方式なら HTML に埋め込む
  ヘッダー方式で JavaScript から読むなら HttpOnly を付けない場合がある
```

この記事のサンプルでは、CSRF トークンを Cookie と hidden input の両方に入れて比較しました。

ただし、実務ではログインセッションと紐づけて CSRF トークンを管理する設計や、署名付きトークンを使う設計もあります。

## 実務ではどう強くするか

ここまでのコードは、学習用にかなりシンプルにしています。

実務では、次のような補強を検討します。

```text
ログインセッションと CSRF トークンを紐づける
トークンを一定時間で更新する
重要操作では再認証を求める
Origin / Referer チェックを追加する
SameSite Cookie を適切に設定する
XSS 対策も行う
```

Cloudflare Workers はステートレスに動かしやすい環境です。

そのため、HMAC を使って署名付き CSRF トークンを作る設計も考えられます。

たとえば、次のような形式です。

```text
token.signature
```

`token` はランダム値です。

`signature` は、サーバー側の秘密鍵で `token` に署名した値です。

検証時には、Worker 側でもう一度署名を計算し、送られてきた署名と一致するか確認します。

ただし、署名付きトークンは少し発展的な内容です。

最初は、この記事で扱ったように、フォームに埋め込んだトークンを受け取り、Cookie 側の値と比較するところから理解するとよいでしょう。

## CSRF と XSS の関係

CSRF 対策を学ぶときは、XSS との関係にも注意が必要です。

CSRF トークンがあっても、XSS があるとトークンを盗まれる可能性があります。

たとえば、攻撃者が同じサイト上で JavaScript を実行できる状態になっていると、フォーム内の hidden input を読めてしまうかもしれません。

```js
document.querySelector('input[name="csrf_token"]').value;
```

つまり、CSRF トークンは XSS 対策の代わりにはなりません。

CSRF 対策と XSS 対策は別の対策ですが、実務では組み合わせて考える必要があります。

Cloudflare Workers で HTML を直接組み立てる場合は、HTML への値の埋め込みにも注意します。

この記事のサンプルでは、CSRF トークンを HTML に埋め込むときに `escapeHtml()` を使いました。

```js
function escapeHtml(value) {
  return value
    .replaceAll("&", "&amp;")
    .replaceAll('"', "&quot;")
    .replaceAll("<", "&lt;")
    .replaceAll(">", "&gt;");
}
```

今回の `csrfToken` は `crypto.randomUUID()` で生成しているため、危険な文字は通常入りません。

それでも、HTML に値を埋め込むときはエスケープする、という習慣をつけておくと安全です。

## まとめ

この記事では、Cloudflare Workers を題材に、CSRF トークンの基本を整理しました。

重要なポイントは次の通りです。

```text
CSRF は、ログイン済みユーザーのブラウザーに意図しないリクエストを送らせる攻撃である
Cookie は自動で送られるため、Cookie があるだけでは本人の操作とは判断できない
CSRF トークンは、自分の画面から送られたリクエストかを確認するための値である
初心者向けには hidden input 方式が理解しやすい
Cloudflare Workers では request.formData() でフォームデータを受け取れる
fetch() や JSON API では X-CSRF-Token ヘッダー方式も使いやすい
crypto.randomUUID() は学習用のトークン生成として扱いやすい
実務では Cookie 属性、Origin チェック、セッション管理、XSS 対策も組み合わせる
```

CSRF 対策を学ぶとき、最初からすべての実務設計を理解しようとすると大変です。

まずは、HTML フォームに hidden input として CSRF トークンを埋め込み、POST 時に Worker 側で検証するところから始めると、仕組みを追いやすくなります。

その後で、`fetch()` を使う JSON API 向けに `X-CSRF-Token` ヘッダー方式を学ぶと、HTTP ヘッダーとボディの役割も整理しやすくなります。

Cloudflare Workers は、こうした HTTP の基本を学ぶ題材としても扱いやすい環境です。
