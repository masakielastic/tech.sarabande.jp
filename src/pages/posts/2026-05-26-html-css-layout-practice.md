---

layout: ../../layouts/PostLayout.astro
title: "HTML と CSS のレイアウトをコピペで練習する学び方"
description: "HTML と CSS の初心者向けに、レイアウトを小さな部品ごとにコピペして、値を変えながら理解する練習方法を紹介します。"
pubDate: 2026-05-26
tags:
  - HTML
  - CSS
  - Web制作
  - レイアウト
  - 初心者向け

---

HTML と CSS のレイアウトは、最初からきれいな Web サイトを作ろうとすると難しく感じます。

理由は単純で、実際の Web ページには多くの要素が同時に出てくるからです。

ヘッダー、ナビゲーション、余白、カード、2カラム、スマホ対応、ボタン、画像、フッター。これらを一度に理解しようとすると、どこで何が起きているのか追いかけにくくなります。

そこで初心者のうちは、完成したページを眺めるよりも、**小さなレイアウト部品をコピペして、CSS の値を1つずつ変えて観察する**練習がおすすめです。

この記事では、HTML と CSS のレイアウトをコピペで練習するための学び方を紹介します。

## この記事の対象者

この記事は、次のような人を対象にしています。

* HTML と CSS を学び始めた人
* Flexbox や Grid がまだよくわからない人
* Web ページのレイアウトをどう練習すればよいかわからない人
* コピペした CSS がなぜ動いているのか理解したい人
* Astro ブログや静的サイト制作の前に、基本レイアウトを練習したい人

この記事では、難しいデザイン理論よりも、まずは手を動かして理解することを重視します。

## 最初は index.html だけで練習する

最初の練習では、HTML ファイルと CSS ファイルを分けなくても大丈夫です。

まずは `index.html` だけを作り、その中に `<style>` タグで CSS を書きます。

```html
<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8">
  <title>CSS レイアウト練習</title>
  <style>
    * {
      box-sizing: border-box;
    }

    body {
      margin: 0;
      font-family: system-ui, sans-serif;
      line-height: 1.7;
      background: #f5f5f5;
      color: #222;
    }

    .box {
      background: white;
      border: 1px solid #ddd;
      padding: 24px;
      margin: 24px;
    }
  </style>
</head>
<body>
  <div class="box">
    <h1>レイアウト練習</h1>
    <p>まずは箱の余白を観察します。</p>
  </div>
</body>
</html>
```

このファイルをブラウザで開くと、白い箱が表示されます。

最初に見るべき CSS は次の3つです。

```css
padding: 24px;
margin: 24px;
border: 1px solid #ddd;
```

`padding` は箱の内側の余白です。

`margin` は箱の外側の余白です。

`border` は箱の境界線です。

CSS レイアウトの基本は、要素を「箱」として見ることです。まずは、この箱の感覚をつかむことが大切です。

## コピペ練習では値を1つだけ変える

コピペ練習で重要なのは、貼り付けて終わりにしないことです。

コードを貼り付けたら、CSS の値を1つだけ変えます。

たとえば、次の値を変更してみます。

```css
padding: 24px;
```

これを次のように変えます。

```css
padding: 8px;
```

次に、さらに大きくしてみます。

```css
padding: 48px;
```

このようにすると、箱の内側の余白がどう変わるかを確認できます。

次は `margin` です。

```css
margin: 8px;
```

```css
margin: 48px;
```

`padding` と `margin` の違いは、文章で説明されるよりも、実際に値を変えたほうが理解しやすいです。

初心者のうちは、CSS を暗記しようとするよりも、**変えたらどう見た目が変わるのか**を観察するほうが効果的です。

## 練習の基本サイクル

コピペ練習は、次の流れで進めます。

1. 動くコードをそのまま貼る
2. ブラウザで表示を確認する
3. CSS の値を1つだけ変える
4. 表示がどう変わったか確認する
5. 何が変わったのか自分の言葉でメモする

たとえば、次のようにメモします。

> `padding` を大きくすると、箱の内側の余白が広がった。

> `margin` を大きくすると、箱と画面端の距離が広がった。

> `gap` を小さくすると、子要素同士の間隔が狭くなった。

このような短いメモで十分です。

CSS の理解は、コードを読んだ量だけでなく、観察した回数で深まります。

## 横並びは Flexbox で練習する

次に、横並びの練習をします。

現在の CSS では、横並びには `display: flex` を使うことが多いです。

```html
<div class="nav">
  <a href="#">ホーム</a>
  <a href="#">記事</a>
  <a href="#">お問い合わせ</a>
</div>
```

```css
.nav {
  display: flex;
  gap: 16px;
  padding: 24px;
  background: white;
}
```

このコードを貼り付けると、リンクが横に並びます。

ここで注目するのは、次の2つです。

```css
display: flex;
gap: 16px;
```

`display: flex;` は、子要素を横方向に並べるための指定です。

`gap` は、子要素同士の間隔です。

次に、`gap` の値を変えてみます。

```css
gap: 4px;
```

```css
gap: 32px;
```

リンク同士の間隔が変わるはずです。

この練習で、`gap` は「中の要素同士の距離を決めるもの」と理解できます。

## ヘッダーの定番レイアウトを練習する

次は、実際の Web サイトでよく見るヘッダーを作ります。

```html
<header class="header">
  <h1 class="logo">Practice</h1>

  <nav class="nav">
    <a href="#">Home</a>
    <a href="#">About</a>
    <a href="#">Contact</a>
  </nav>
</header>
```

```css
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 24px;
  background: white;
  border-bottom: 1px solid #ddd;
}

.logo {
  margin: 0;
  font-size: 24px;
}

.nav {
  display: flex;
  gap: 16px;
}
```

このヘッダーでは、左にロゴ、右にナビゲーションが配置されます。

重要なのは、次の2行です。

```css
justify-content: space-between;
align-items: center;
```

`justify-content: space-between;` は、要素を左右に分ける指定です。

`align-items: center;` は、縦方向を中央に揃える指定です。

この組み合わせは、ヘッダーで非常によく使います。

次のように値を変えてみると、Flexbox の動きがわかりやすくなります。

```css
justify-content: center;
```

```css
justify-content: space-around;
```

```css
align-items: flex-start;
```

最初からすべての値を覚える必要はありません。

まずは `space-between` と `center` の組み合わせを覚えるだけでも十分です。

## カード一覧は Grid で練習する

記事一覧や商品一覧のようなカードレイアウトには、`display: grid` が便利です。

```html
<section class="cards">
  <article class="card">
    <h2>HTML</h2>
    <p>文書の構造を作ります。</p>
  </article>

  <article class="card">
    <h2>CSS</h2>
    <p>見た目や配置を整えます。</p>
  </article>

  <article class="card">
    <h2>JavaScript</h2>
    <p>動きを追加します。</p>
  </article>
</section>
```

```css
.cards {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 24px;
  padding: 24px;
}

.card {
  background: white;
  padding: 24px;
  border: 1px solid #ddd;
  border-radius: 12px;
}
```

このコードでは、3つのカードが横に並びます。

注目するのは、次の指定です。

```css
grid-template-columns: repeat(3, 1fr);
```

これは、同じ幅の列を3つ作るという意味です。

次のように変えると、2列になります。

```css
grid-template-columns: repeat(2, 1fr);
```

次のようにすると、1列になります。

```css
grid-template-columns: 1fr;
```

Grid は、縦横の並びを作るときに便利です。

初心者のうちは、まずカード一覧で `grid-template-columns` を変える練習をすると理解しやすいです。

## スマホ対応はモバイルファーストで考える

Web ページは、PC だけでなくスマホでも見られます。

そのため、画面幅に応じてレイアウトを変える必要があります。

初心者には、まずスマホ向けの1列レイアウトを書き、画面が広いときだけ列を増やす方法がおすすめです。

```css
.cards {
  display: grid;
  grid-template-columns: 1fr;
  gap: 24px;
  padding: 24px;
}

@media (min-width: 768px) {
  .cards {
    grid-template-columns: repeat(3, 1fr);
  }
}
```

このコードでは、基本は1列です。

画面幅が 768px 以上になったときだけ、3列になります。

このように、スマホ向けを先に書いて、広い画面用の指定を後から追加する考え方を **モバイルファースト** と呼びます。

初心者は、まずこの形だけ覚えるとよいです。

## 中央寄せコンテナを練習する

Web サイトでは、本文が画面いっぱいに広がりすぎないように、中央に幅を制限したコンテナを置くことがよくあります。

```html
<main class="container">
  <h1>中央寄せコンテナ</h1>
  <p>本文が画面いっぱいに広がりすぎないようにします。</p>
</main>
```

```css
.container {
  max-width: 960px;
  margin: 0 auto;
  padding: 40px 24px;
  background: white;
}
```

重要なのは、次の3つです。

```css
max-width: 960px;
margin: 0 auto;
padding: 40px 24px;
```

`max-width` は最大幅です。

`margin: 0 auto;` は左右中央寄せです。

`padding` は内側の余白です。

この形は、ブログ、企業サイト、LP など、多くのページで使えます。

## 2カラムレイアウトを練習する

ブログ記事とサイドバーのようなレイアウトでは、2カラムを使います。

```html
<div class="layout">
  <main class="main">
    <h1>記事タイトル</h1>
    <p>ここに本文が入ります。</p>
  </main>

  <aside class="sidebar">
    <h2>サイドバー</h2>
    <p>プロフィールや関連リンクを置きます。</p>
  </aside>
</div>
```

```css
.layout {
  display: grid;
  grid-template-columns: 1fr;
  gap: 24px;
  padding: 24px;
}

.main,
.sidebar {
  background: white;
  padding: 24px;
  border: 1px solid #ddd;
}

@media (min-width: 900px) {
  .layout {
    grid-template-columns: 2fr 1fr;
  }
}
```

スマホでは1列です。

画面が広くなると、本文とサイドバーの2カラムになります。

```css
grid-template-columns: 2fr 1fr;
```

これは、本文を広く、サイドバーを狭くする指定です。

この形は、ブログや管理画面などでよく使います。

## ヒーローエリアを練習する

トップページや LP では、大きな見出しを配置するヒーローエリアを作ることがあります。

```html
<section class="hero">
  <div class="hero-text">
    <h1>はじめての CSS レイアウト</h1>
    <p>小さな部品を組み合わせて、ページ全体を作ります。</p>
    <a class="button" href="#">詳しく見る</a>
  </div>
</section>
```

```css
.hero {
  min-height: 60vh;
  display: flex;
  align-items: center;
  padding: 64px 24px;
  background: white;
}

.hero-text {
  max-width: 720px;
}

.hero h1 {
  font-size: 40px;
  line-height: 1.2;
  margin: 0 0 16px;
}

.hero p {
  margin: 0 0 24px;
}

.button {
  display: inline-block;
  padding: 12px 20px;
  background: #222;
  color: white;
  border-radius: 999px;
}
```

ここで見るべきなのは、次の指定です。

```css
min-height: 60vh;
display: flex;
align-items: center;
```

`min-height: 60vh;` は、画面の高さを基準にした最低限の高さです。

`align-items: center;` によって、中のテキストが縦方向に中央寄せされます。

ヒーローエリアでは、余白を大きめに取ると、それだけでトップページらしい見た目になります。

## 部品を組み合わせて1ページにする

ここまで練習した部品を組み合わせると、簡単な1ページを作れます。

```html
<header class="header">
  <h1 class="logo">Practice</h1>
  <nav class="nav">
    <a href="#">Home</a>
    <a href="#">Articles</a>
    <a href="#">Contact</a>
  </nav>
</header>

<section class="hero">
  <div class="hero-text">
    <h1>HTML と CSS のレイアウト練習</h1>
    <p>まずはコピペして、少しずつ値を変えて覚えます。</p>
    <a class="button" href="#">Start</a>
  </div>
</section>

<main class="container">
  <section class="cards">
    <article class="card">
      <h2>Box</h2>
      <p>余白と境界線を理解します。</p>
    </article>

    <article class="card">
      <h2>Flex</h2>
      <p>横並びと中央寄せを練習します。</p>
    </article>

    <article class="card">
      <h2>Grid</h2>
      <p>カード一覧や2カラムを作ります。</p>
    </article>
  </section>
</main>

<footer class="footer">
  <p>&copy; 2026 Practice Site</p>
</footer>
```

CSS は次のようにまとめます。

```css
* {
  box-sizing: border-box;
}

body {
  margin: 0;
  font-family: system-ui, sans-serif;
  line-height: 1.7;
  background: #f5f5f5;
  color: #222;
}

a {
  color: inherit;
  text-decoration: none;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 24px;
  background: white;
  border-bottom: 1px solid #ddd;
}

.logo {
  margin: 0;
  font-size: 24px;
}

.nav {
  display: flex;
  gap: 16px;
}

.hero {
  min-height: 60vh;
  display: flex;
  align-items: center;
  padding: 64px 24px;
  background: white;
}

.hero-text {
  max-width: 720px;
}

.hero h1 {
  font-size: 40px;
  line-height: 1.2;
  margin: 0 0 16px;
}

.hero p {
  margin: 0 0 24px;
}

.button {
  display: inline-block;
  padding: 12px 20px;
  background: #222;
  color: white;
  border-radius: 999px;
}

.container {
  max-width: 1120px;
  margin: 0 auto;
  padding: 40px 24px;
}

.cards {
  display: grid;
  grid-template-columns: 1fr;
  gap: 24px;
}

.card {
  background: white;
  padding: 24px;
  border: 1px solid #ddd;
  border-radius: 12px;
}

.footer {
  padding: 24px;
  text-align: center;
  background: white;
  border-top: 1px solid #ddd;
}

@media (min-width: 768px) {
  .cards {
    grid-template-columns: repeat(3, 1fr);
  }
}
```

この1ページには、HTML と CSS の基本的なレイアウト要素が多く含まれています。

* 箱の余白
* ヘッダー
* ナビゲーション
* Flexbox
* Grid
* カード一覧
* 中央寄せコンテナ
* ヒーローエリア
* レスポンシブ対応

最初は、このコードをそのまま貼り付けて動かすだけで構いません。

その後、少しずつ値を変えていきます。

## 初心者が最初に見るべき CSS

最初のうちは、CSS 全体を完璧に理解しようとしなくても大丈夫です。

まずは、次の指定を重点的に見ます。

```css
box-sizing: border-box;
margin
padding
border
display: flex;
justify-content
align-items
gap
display: grid;
grid-template-columns
max-width
@media
```

これらを理解できると、基本的なレイアウトはかなり作りやすくなります。

特に重要なのは、`display: flex;` と `display: grid;` です。

昔は `float` を使って複雑なレイアウトを作ることもありましたが、現在の練習ではまず Flexbox と Grid を覚えるほうが実用的です。

## やってはいけない練習方法

初心者がつまずきやすい練習方法もあります。

たとえば、完成された大きなテンプレートを丸ごとコピペする方法です。

もちろんテンプレートを使うこと自体は悪くありません。

しかし、初心者のうちに大量の CSS を一度に貼り付けると、どの指定がどの見た目に影響しているのかわからなくなります。

また、`position: absolute;` を多用して無理やり配置するのも避けたほうがよいです。

`position` は便利ですが、最初から多用すると、通常の文書の流れ、Flexbox、Grid の理解が遅れやすくなります。

まずは、次の順番で練習するのがおすすめです。

1. 箱の余白
2. Flexbox の横並び
3. ヘッダー
4. Grid のカード一覧
5. レスポンシブ対応
6. 2カラム
7. ヒーローエリア
8. 1ページへの組み合わせ

この順番なら、部品からページ全体へ自然に進めます。

## Astro ブログで練習記事を書く場合

Astro ブログに学習記録として残すなら、単に完成コードを貼るだけでなく、次のように書くとあとから読み返しやすくなります。

````md
## 今日練習したこと

Flexbox を使ってヘッダーを作った。

## 使った CSS

```css
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
````

## 変えてみた値

`justify-content` を `center` に変えた。

## わかったこと

`space-between` は左右に分ける指定で、`center` は中央に集める指定だった。

```

このように、コードと観察結果をセットで残すと、自分用の教材になります。

AI 時代は、コードそのものはすぐに生成できます。

だからこそ、初心者の学習では、生成されたコードを貼って終わりにするのではなく、**値を変えて、観察して、自分の言葉で説明する**ことが重要になります。

## まとめ

HTML と CSS のレイアウトは、いきなり完成された Web サイトを作ろうとすると難しく感じます。

初心者のうちは、小さな部品をコピペして練習するのがおすすめです。

まずは箱を作り、`padding`、`margin`、`border` を観察します。

次に、Flexbox で横並びを作ります。

その後、Grid でカード一覧や2カラムを作ります。

最後に、ヘッダー、ヒーロー、カード、フッターを組み合わせて、簡単な1ページにします。

大事なのは、コピペを否定しないことです。

ただし、貼り付けて終わりにするのではなく、値を1つずつ変えて、表示の変化を確認します。

CSS は、暗記するものというより、箱をどう並べるかを試しながら覚えるものです。

小さく貼って、小さく変えて、小さく理解する。

この繰り返しが、HTML と CSS のレイアウトを身につける近道です。

```
