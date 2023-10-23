---
title: "Svelteの基礎"
free: true
---

## Svelte とは

Svelte とはウェブフレームワークの一つです．リアクティブプログラミングや状態管理などがシンプルに記述できます．また仮想 DOM を使用していないのも特徴的です．実行時に仮想 DOM を動かすコードを埋め込む必要がないのでバンドルサイズの縮小，パフォーマンスの向上が期待できます．
Svelte はコンパイラだと言われています．当然 React や Vue などのフレームワークもコンパイルという行為は行っています．しかし，Svelte はコンパイル時に ライブラリのランタイムコードがほとんど存在しない JavaScript に変換されます．これが Svelte がコンパイラだと言われる所以だと筆者は理解しています．
だからといって Svelte 最強！というわけではなく，ランタイムコードが小さい代わりにコンポーネント 1 つあたりのバンドルサイズが若干大きくなるというデメリットがあります．Vue との興味深い比較がこちらにあります．

https://github.com/yyx990803/vue-svelte-size-analysis

この章では Svelte の基礎を紹介します．
全部はとても網羅できないので今回作るアプリで使う機能だけピックアップして紹介します．

## ファイル構成

Svelte はコンポーネントという単位でコードを管理します．1 ファイルが 1 コンポーネントに対応し，以下のような構成になっています．

```html
<script>
    /* ここにJavascriptを記述 */
</script>

/* ここにHTMLを記述 */

<style>
    /* CSSをここに記述 */
</style>
```

### 例

Hello world! を表示させるコンポーネントです．`{}`で囲うと変数の値が参照できます．

```javascript:App.svelte
<script>
    let name = 'world';
</script>

<h1>Hello {name}!</h1>

<style>
    h1 {
        color: #f0f
    }
</style>
```

https://svelte.dev/repl/bf1734116e634a47a11cd5282fced69c?version=4.2.1

## リアクティブプログラミング

リアクティブという単語は React や Vue などのフレームワークを触っていれば聞くことがあるかもしれません．データを受け取るたびに関連したプログラムが反応して処理を行うようにするプログラミングの考え方です．ウェブフレームワークは主に変数の変化によって DOM の値が変更されるようなときに使われます．

リアクティブプログラムの例を考えてみます．
変数 a に変更があると変数 b，変数 c にも変更が伝わります．参照とは違う概念であり，変数の変更があったときに処理が連鎖的に実行されることに注意してください．

```:リアクティブプログラミングの例
a = ?? <- aの値が変更された
b = a  <- aの変更が反映される
c = b  <- bの変更が反映される
```

React や Vue でリアクティブプログラミングをするときは`setState`や`ref`を使いますが，Svelte では従来の JavaScript の書き方でリアクティブプログラミングができます．

### 例

script 内で`count`変数と`increment`関数が定義されています．ボタンを押すことで`increment`関数が呼ばれ`count`変数に 1 足されます．`on:click`は HTML で button タグの属性である`onclick`と等価です．

`countDouble`変数は`count`変数の 2 倍の値を持ちます．`$:` の後に処理を書くと，変数の値が変更されたときに処理が実行されます．

`$:`の後に続くのは処理でなければなりません．例えば`increment`関数を実行してほしければ`$: increment`ではなく`$: increment()`と書かなければなりません．

```javascript:App.svelte
<script>
    let count = 0;
    let countDouble = 0;
    const increment = () => {
        count += 1
    }

    $: countDouble = count * 2;
</script>

<button on:click={increment}>
    count is {count}
</button>

<p>countDouble {countDouble}</p>
```

シンプルかつ直感的に書けるのが Svelte の魅力です．

https://svelte.dev/repl/d8d733107c2f4c54989122c0d6027578?version=4.2.1

## コンポーネントのパラメータ

script 内で`export`を使うと外部から変数を受け取ることができます．

### 例

Counter.svelte は`count`という変数を受け取っています．App.svelte では Counter.svelte をインポートして`count`に 100 を代入しています．

```javascript:Counter.svelte
<script>
    export let count;
    const increment = () => {
        count += 1
    }
</script>

<button on:click={increment}>
    count is {count}
</button>
```

```javascript:App.svelte
<script>
    import Counter from "./Counter.svelte"
</script>

<Counter count={100}/>
```

https://svelte.dev/repl/3f9747fd0d1747d5909a3db38e8d224b?version=4.2.1

## ロジックブロック

ロジックブロックは`{#if}`や`{#each}`などのように`{#}`で始まるものです．ロジックブロックを使うと条件分岐や繰り返しを表現できます．

```javascript:App.svelte
<script>
    const age = 16;
    const nameList = ["太郎", "次郎", "三郎"];
</script>

{#if age >= 18}
    <h1>成人しています</h1>
{:else}
    <h1>成人していません</h1>
{/if}

{#each nameList as name}
    <h2>{name}</h2>
{/each}
```

https://svelte.dev/repl/4c2a425554184b6aab1421e8597c51cb?version=4.2.1

## ストア

Svelte の Store(ストア)は状態を管理するための仕組みです．Store を使うと，複数のコンポーネントでリアクティブな変数を共有できます．

本アプリではストアの`writeable`をよく使います．`writeable`は変数の値を変更することができます．

`writeable`は以下の関数があります．

-   `set`：値を変更する
-   `update`：値を変更するが，前の値を引数に取ることができる
-   `subscribe`：値が変更されたときに呼ばれる関数を登録する

### 例

TypeScript で書いたストアを定義してエキスポートします．

```typescript:store.ts
import { writable } from "svelte/store";

export const count = writable(0);
```

`count`ストアは`writeable`で作られているので，`set`や`update`を使って値を変更できます．

```javascript:App.svelte
<script>
    import { count } from "./store.ts";
</script>

<button on:click={() => count.update(n => n + 1)}>
    count is {count.get()}
</button>
```

また，以下のように`$`プレフィックスを使い，ストアの値を直接参照することができます．

```javascript:App.svelte
<script>
    import { count } from "./store.ts";
</script>

<button on:click={() => $count += $count + 1}>
    count is {$count}
</button>
```

`$`プレフィックスによる参照はコンポーネント内でのみ有効です．

https://svelte.dev/repl/ffbaab64ae8a42529b35c0c9ba7f07a1?version=4.2.1

もしコンポーネント内のみで変数を使いたい場合は`writable`は使わず，`let`を使うと良いでしょう．

## アニメーション

Svelte はアニメーションに関するライブラリも提供しています．

以下は`fly`というアニメーションを使って要素を動かしています．Svelte のアニメーションは主にロジックブロックとセットで使います．簡単にパワーポイントのようなアニメーションを作ることができます．

```javascript:App.svelte
<script>
    import { fly } from "svelte/transition";

    let visible = false;
</script>

<button on:click={() => (visible = !visible)}>click</button>
{#if visible}
    <div transition:fly={{ x: 200, duration: 500 }}>
        <p>hello</p>
    </div>
{/if}
```

https://svelte.dev/repl/601ccf3b799444818eb4227e81fccb69?version=4.2.1

## まとめ

Svelte の基本的な機能を一部紹介しました．ここで紹介しきれないものは公式ドキュメントに書いてあるので適時確認してください．

https://svelte.dev/docs
