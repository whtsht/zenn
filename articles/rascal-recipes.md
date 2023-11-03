---
title: "Rascalでメタプログラミングを楽しむ"
emoji: "🐾"
type: "tech"
topics: ["rascal", "メタプログラミング"]
published: true
---

## はじめに

皆さんはコンパイラコンパイラ（コンパイラジェネレータ）と聞いたら何を想像するでしょうか。そんなものは聞いたことがないという人もいるでしょうが、yaccなどのパーサージェネレータを想像する人もいると思います。

パーサージェネレータは文法を記述すると、その構文を解析するパーサーのプログラムを自動生成してくれるものです。パーサージェネレータは様々なものが存在し、今でも活躍していると思います。

しかしパーサージェネレータがやってくれるのは構文解析だけです。構文解析、意味解析、コード生成をすべてしてくれるコンパイラコンパイラは全然メジャーじゃないです。40年くらい前にMETAⅡなどのコンパイラを作成する専用のDSLがあったらしいですが知らない人がほとんどだと思います。

構文解析、意味解析、コード生成をすべてサポートしたツールがあれば確実に便利なはずですよね？なぜ普及しないのでしょうか。

そんなモヤモヤをChatGPTにぶつけていたらRascalという面白いものを見つけました。

```
私
    コンパイラを記述するDSL

ChatGPT
    ... (パーサージェネレータをいくつか提案)

私
    ASTなどの木構造から他の木構造に変換できるDSL

ChatGPT
    AST（抽象構文木）などの木構造から他の木構造に変換するためのDSL（Domain Specific Language）としては、以下のようなものがあります：

    Tree Transformation Languages (TTLs): ASTの変換や操作を行うために設計されたDSLです。例えば、RascalやStrategoなどがあります。

    ...

    これらのDSLは、特定の文法やルールに基づいて木構造を変換するための便利な手段となります。適切なDSLを選んで使うことで、特定の目的に合った木構造の変換を効果的に行うことができます。
```

https://www.rascal-mpl.org/

https://github.com/usethesource/rascal

Rascalはメタプログラミングをするための言語やその処理系です。処理系はJVMを使っているみたいです。

アライグマではないようです[^1]。

## メタプログラミングとは

そもそもメタプログラミングとは何でしょうか。

Rascal公式サイトから抜粋

> Meta-programs are programs that analyze, transform or generate other programs. Ordinary programs work on data; meta-programs work on programs.

wikipediaからの抜粋

> Metaprogramming is a programming technique in which computer programs have the ability to treat other programs as their data. It means that a program can be designed to read, generate, analyze or transform other programs, and even modify itself while running.

つまり

- プログラムを解析、変換したり、新たなプログラムを生成したりできる。
- プログラムはデータに作用する、メタプログラムはプログラムに作用する（＝プログラムをデータとして扱える）。

と言ったところでしょう。

![](https://storage.googleapis.com/zenn-user-upload/ade3a17c51de-20231103.png)

図にはプログラムとコードがありますが、ここではソースコードをコード、それをコンパイルしたものをプログラムとしてます。

https://www.rascal-mpl.org/docs/WhyRascal/Motivation/

https://en.wikipedia.org/wiki/Metaprogramming

## コンパイラコンパイラとの関係

メタプログラミングとコンパイラコンパイラの関係性を考えてみましょう。

コンパイラはソースコードをオブジェクトコードに変換するプログラムです。コンパイラになるコードを生成するプログラムがコンパイラコンパイラです。これはプログラムから新たなプログラムを生成しているので、メタプログラムに含まれることがわかります。

![](https://storage.googleapis.com/zenn-user-upload/26ef64b4c53b-20231103.png)

## Rascalをインストール

前置きが長くなってしまいました。Rascalを実際に触ってみましょう。javaの環境が必要なのでインストールしておいて下さい。

公式からjarファイルをダウンロードします。以下のコマンドでRascalのREPLが起動するはずです。

```
$ wget https://update.rascal-mpl.org/console/rascal-shell-stable.jar
$ java -jar rascal-shell-stable.jar
```

Arch LinuxにはAURがあります。

```
$ paru -S rascal
```

## 計算機を作る

まずは簡単な計算機を作ってみます。目標は括弧付きの四則演算を解析して評価することです。

適当にディレクトリを作ります。

```
$ mkdir Exp
$ cd Exp
$ touch Syntax.rsc Eval.rsc
```

Syntax.rscに構文を定義していきます。

```haskell:Syntax.rsc
module Syntax

layout Whitespace = [\t-\n\r\ ]*;

lexical IntegerLiteral = [0-9]+;

start syntax Exp
    = IntegerLiteral
    | bracket "(" Exp ")"
    > left Exp "*" Exp
    | left Exp "/" Exp
    > left Exp "-" Exp
    | left Exp "+" Exp
    ;
```

`layout Whitespace = [\t-\n\r\ ]*`で解析する上で無視していいパターンを定義しています。
`lexical IntegerLiteral = [0-9]+`は数字の定義です。

`start syntax Exp`以下で式を定義しています。`|`はひとつ上の定義と同じ優先度を持つことを意味します。例えば`"(" Exp ")"`は`IntegerLiteral`と同じ優先度を持ちます。`>`はひとつ上の定義より優先度が低いことを表します。例えば`Exp "-" Exp`は`Exp "/" Exp`より優先度が低いです。`left`は左結合演算子であることを表しています。

次に今定義した式を評価する処理を書いていきます。

```haskell:Eval.rsc
module Eval

import String;
import ParseTree;
import Syntax;

int eval(str txt) = eval(parse(#start[Exp], txt).top);

int eval((Exp)`<IntegerLiteral l>`) = toInt("<l>");
int eval((Exp)`<Exp e1> + <Exp e2>`) = eval(e1) + eval(e2);
int eval((Exp)`<Exp e1> - <Exp e2>`) = eval(e1) - eval(e2);
int eval((Exp)`<Exp e1> * <Exp e2>`) = eval(e1) * eval(e2);
int eval((Exp)`<Exp e1> / <Exp e2>`) = eval(e1) / eval(e2);
int eval((Exp)`( <Exp e> )`) = eval(e);

test bool tstEval1() = eval(" 7") == 7;
test bool tstEval2() = eval("7 * 3") == 21;
test bool tstEval2() = eval("3 - 7") == -4;
test bool tstEval2() = eval("7 / 3") == 2;
test bool tstEval3() = eval("7 + 3") == 10;
test bool tstEval4() = eval(" 3 + 4*5 ") == 23;
```

```
rascal>import Eval;
Loading module |cwd:///Eval.rsc|
Loading module |cwd:///Syntax.rsc|
Loading module |std:///ParseTree.rsc|
...
```

`eval`という名前の変数を定義しています。
引数のパターンマッチを行っています。`eval`に`str`（文字列）が渡されたときは`parse`関数で`Exp`に変換後再び`eval`を呼び出します。`eval`に`Exp`が渡されたときは愚直に計算してるだけです。

ちなみに`toInt`関数は文字列を`Int`型に変換する関数で`String`モジュールで定義されています。`parse`関数も`ParseTree`モジュールで既に定義されています。これらのモジュールは標準ライブラリです。詳しくはこちらをご覧ください。

https://www.rascal-mpl.org/docs/Library/

Rascalで定義されている型は以下を参照して下さい。

https://www.rascal-mpl.org/docs/Recipes/BasicProgramming/Datatypes/

下の方にある`test bool ...`はユニットテストです。

以上です。30行足らずで計算機を実装できました！ 早速試してみましょう。
REPLを起動して`Eval`モジュールを読み込みます。

```
rascal>import Eval;
Loading module |cwd:///Eval.rsc|
...
```

Eval.rscで定義した`eval`関数を呼び出してみます。

```
rascal>eval("(8 + 2) * 3");
int: 30
```

すべてのユニットテストを実行するには`:test`と入力します。

```
rascal>:test
Running tests for Eval
| testing 0/6 | testing 0/6 success                                                    Test report for Evals
        all 6/6 tests succeeded
bool: true
```

## おわりに

Rascalにどんな印象を抱いたでしょうか。私は「だだのパターンマッチが強力な関数型言語やないかい」と思いました。

でもこれがコンパイラの本質とも言える気がします。プログラムはデータに作用するものですが、そのデータをソースコードにしてはいけないなんてルールはどこにも存在しません。

最初の構文解析からコード生成までをすべて行うコンパイラコンパイラは何故メジャーでないのかという疑問に戻ります。これの答えは、プログラミング言語そのものが自身のコンパイラを生成できるほど発達したからだと思います。メタプログラミングはもはや普通のプログラミング言語でもできるということです。

とは言え、Rascalの有用性もちゃんとあると思います。非常に簡潔に構文の定義ができました。オレオレプログラミング言語を作りたいときにRascalなどのメタプログラミング用に作られたツールを使うのも十分に選択肢になると思います。

最後に

メタプログラミングはいいぞ......。

[^1]: https://www.araiguma-rascal.com/about/works/
