---
title: "Rascalでコンパイラを作る"
emoji: "🐾"
type: "tech"
topics: ["rascal", "メタプログラミング"]
published: true
---

## はじめに

[前回](https://zenn.dev/whiteshirt/articles/rascal-recipes)、Rascalというメタコンパイラの紹介して、簡単な計算機を作りました。

今回はRascal公式サイトを参考に、オリジナル言語Funcのコンパイラを作っていこうと思います。

Funcは変数の束縛、式、関数などがある関数型言語です。使えるデータ型は自然数のみです。

例えば、Funcで階乗を求めるコードは以下のように書けます。

```haskell:fact.func
fact(n) =
    if n == 0
        then 1
        else n * fact(n - 1)
    end
```

この記事で、Rascalでコンパイラを作る雰囲気を味わってほしいです。

## 文法

`Syntax.rsc`に文法の定義を書きます。

Func言語は以下の構成になっています。

- `Prog`: プログラム全体、`Func`の集まり
- `Func`: 関数の定義、関数名、引数、`Exp`を持つ
- `Exp` : 式の定義、四則演算や`let in`式や`if then else`式をサポート

また`%%` で1行コメント、`%`で囲うと複数行コメント扱いになります。

```rascal:Syntax.rsc
module Syntax

lexical Ident = [a-z][a-z0-9]* !>> [a-z0-9];

lexical Natural = [0-9]+ ;

lexical String = "\"" ![\"]*  "\"";

layout Layout = WhitespaceAndComment* !>> [\ \t\n\r];

lexical WhitespaceAndComment
   = [\ \t\n\r]
   | @category="Comment" ws2: "%" ![%]+ "%"
   | @category="Comment" ws3: "%%" ![\n]* $
   ;

syntax Binding = binding: Ident name "=" Exp exp;

syntax Exp
    = natCon: Natural
    | bracket "(" Exp ")"
    > left (mul: Exp e1 "*" Exp e2 | div: Exp e1 "/" Exp e2)
    > left (sub: Exp e1 "-" Exp e2 | add: Exp e1 "+" Exp e2)
    > left ( gt: Exp e1 "\>" Exp e2
           | lt: Exp e1 "\<" Exp e2
           | ge: Exp e1 "\>=" Exp e2
           | le: Exp e1 "\<=" Exp e2
           )
    > left (eq: Exp e1 "==" Exp e2 | neq: Exp e1 "/=" Exp e2)
    | id: Ident name
    | let: "let" {Binding ","}* binds "in" Exp exp "end"
    | cond: "if" Exp condExp "then" Exp thenExp "else" Exp elseExp "end"
    | call: Ident name "(" {Exp ","}* args ")"
    ;

syntax Func = func: Ident name "(" {Ident ","}* args ")" "=" Exp;

start syntax Prog = prog: Func*;
```

## SyntaxからRascalのデータ構造に変換

実は[前回](https://zenn.dev/whiteshirt/articles/rascal-recipes)の計算機はこの工程を省いて、直接syntaxの定義から評価しています。より複雑な処理をする場合はRascalのデータ構造に変換してからのほうが良いです。

`Abstract.rsc`にFunc言語で書かれたプログラムを表すデータ構造を定義します。

```rascal:Abstract.rsc
module Abstract

data BINDING
    = binding(str name, EXP exp);

data EXP
    = natCon(int iVal)
    | mul(EXP e1, EXP e2)
    | div(EXP e1, EXP e2)
    | sub(EXP e1, EXP e2)
    | add(EXP e1, EXP e2)
    | gt(EXP e1, EXP e2)
    | lt(EXP e1, EXP e2)
    | ge(EXP e1, EXP e2)
    | le(EXP e1, EXP e2)
    | eq(EXP e1, EXP e2)
    | neq(EXP e1, EXP e2)
    | id(str name)
    | let(list[BINDING] binds, EXP exp)
    | cond(EXP condExp, EXP thenExp, EXP elseExp)
    | call(str name, list[EXP] args)
    ;

data FUNC
    = func(str name, list[str] args, EXP exp);

data PROG
    = prog(list[FUNC] funcs);
```

よーく見ると`Syntax.rsc`の`Prog`、`Exp`、`Binding`と`Abstract.rsc`の`PROG`、`EXP`、`BINDING`は対応関係があることがわかります。

例えば`Syntax.rsc`の`Exp`の定義の1つである

```
cond: "if" Exp condExp "then" Exp thenExp "else" Exp elseExp "end"
```

は`Abstract.rsc`の`EXP`の定義の1つである

```
cond(EXP condExp, EXP thenExp, EXP elseExp)
```

と対応しています。

この対応を漏れなく書かないとエラーになるので注意して下さい。

`Load.rsc`にFuncソースコードから`PROG`データ構造を生成するコードを書きます。

```rascal:Load.rsc
module Load

import Prelude;
import Syntax;
import Abstract;

PROG load(str s) = implode(#PROG, parse(#start[Prog], s).top);
PROG load(loc l) = implode(#PROG, parse(#start[Prog], l).top);
```

ここで仕事をしてるのは`implode`という関数です。`parse`関数でソースコードを`Prog`に変換後、`implode`関数で`PROG`データ構造に変換しています。

`load`関数はソースコードを直接渡すこともできますが、`Location`を指定することでファイルを入力にできます。

`Location`はRascalに出てくる概念で、ファイルパスを一般化したようなものです。詳しくは以下を参考にして下さい。

https://www.rascal-mpl.org/docs/Rascal/Expressions/Values/Location/

## VM（Virtual Machine）コード

Func言語からスタックベースのVMマシンで動くことを想定したコードを生成します。

`Assembly.rsc`にそのVMマシンで動く命令を定義します。

```rascal:Assembly.rsc
module Assembly

data Instr
    // 自然数をスタックに積む
    = pushNat(int intCon)
    // 識別子をスタックに積む
    | pushId(str id)
    // 識別子をスタックから取り出す。その識別子の変数をスタックに積む
    | getVal()
    // Assign value on top, to variable at identifier top-1
    | setVal()
    // ラベルの定義
    | label(str label)
    // 無条件でラベルに飛ぶ
    | go(str label)
    // スタックから値を取り出す、取り出した値を0以外ならばラベルに飛ぶ
    | goIf(str label)
    // 四則演算
    | mul() | div() | add() | sub()
    // 比較演算子
    | gt() | lt() | ge() | le() | eq() | neq()
    // スタックから識別子を取り出してその識別子の関数を呼び出す
    | call()
    ;

alias Instrs = list[Instr];

alias Code = map[str, Instrs];
```

コンパイラを作ることが目的なのでVMコードはテキトーに設計しました。関数の引数や計算結果を保存するスタックと変数を保存するヒープが一応あります。「でもこれだと変数が上書きされね？」「 なんのために関数の引数スタックに積んでるんだよ。」などのツッコミどころ満載なのはご容赦下さい。

## コード生成

コンパイラは普通、変数が宣言されているかなどをチェックしますが、今回は省きます。面倒なんで。

`Compile.rsc`に`PROG`から`Code`を生成する`compileProg`を定義します。

```rascal:Compile.rsc
module Compile

import Assembly;
import Abstract;
import IO;

Instrs compileBinding(binding(str name, EXP exp))
    = [*compileExp(exp), pushId(name), setVal()];

Instrs compileExp(natCon (int N)) = [pushNat(N)];

Instrs compileExp(id (str name)) = [pushId(name), getVal()];

// 変数を束縛した後、expのコード生成
Instrs compileExp(let (list[BINDING] binds, EXP exp))
    = [*([*compileBinding(bind) | bind <- binds]), *compileExp(exp)];

// ラベル生成用
private int nLabel = 0;

// ラベル生成
private str nextLabel() {
  nLabel += 1;
  return "L<nLabel>";
}

// 分岐するためのラベルを生成する。goIfはスタックの先頭が0以外なら分岐。
Instrs compileExp(cond(EXP condExp, EXP thenExp, EXP elseExp)) {
    thenLabel = nextLabel();
    endLabel = nextLabel();
    return [
        *compileExp(condExp),
        goIf(thenLabel),
        *compileExp(elseExp),
        go(endLabel),
        label(thenLabel),
        *compileExp(thenExp),
        label(endLabel)
    ];
}

// e1 e2を計算するとそれぞれの計算結果がスタックに残るはず。最後にmul()やdiv()命令を呼び、
// スタックから値を取り出し計算を行い、計算結果を再びスタックにプッシュする。
Instrs compileExp(mul(EXP e1, EXP e2)) = [*compileExp(e1), *compileExp(e2), mul()];
Instrs compileExp(div(EXP e1, EXP e2)) = [*compileExp(e1), *compileExp(e2), div()];
Instrs compileExp(sub(EXP e1, EXP e2)) = [*compileExp(e1), *compileExp(e2), sub()];
Instrs compileExp(add(EXP e1, EXP e2)) = [*compileExp(e1), *compileExp(e2), add()];
Instrs compileExp(gt(EXP e1, EXP e2)) = [*compileExp(e1), *compileExp(e2), gt()];
Instrs compileExp(lt(EXP e1, EXP e2)) = [*compileExp(e1), *compileExp(e2), lt()];
Instrs compileExp(ge(EXP e1, EXP e2)) = [*compileExp(e1), *compileExp(e2), ge()];
Instrs compileExp(le(EXP e1, EXP e2)) = [*compileExp(e1), *compileExp(e2), le()];
Instrs compileExp(eq(EXP e1, EXP e2)) = [*compileExp(e1), *compileExp(e2), eq()];
Instrs compileExp(neq(EXP e1, EXP e2)) = [*compileExp(e1), *compileExp(e2), neq()];

// 引数のコード生成をしてスタックにプッシュする。その後関数名をスタックにプッシュしcall()を呼ぶ。
Instrs compileExp(call(str name, list[EXP] args))
    = [*[*compileExp(arg) | arg <- args], pushId(name), call()];

// スタックの値を変数に束縛後、expのコード生成。
Instrs compileFunc(list[str] args, EXP exp)
    = [*[*[pushId(arg), setVal()] | arg <- args], *compileExp(exp)];

Code compileProg(prog (list[FUNC] funcs))
    = (name : compileFunc(args, exp)
        | func(str name, list[str] args, EXP exp) <- funcs);
```

所々に`[*..., *...]`みたいに`*`が付くのが気になると思います。これは配列を平坦化する文法です。例えば`[[1, 2], [3, 4]]`は`list[list[int]]`型になるのですが、`[*[1, 2], *[3, 4]]`と書くと、これは`[1, 2, 3, 4]`に展開されて型は`list[int]`になります。この文法を駆使すると、ほとんどがリストの定義と内包表現で書けます。素晴らしい！

最後に`Main.rsc`にこれらの関数を呼び出すための関数を定義します。

```rascal:Main.rsc
module Main

import Load;
import Compile;
import Assembly;

Code compile(str s) = compileProg(load(s));
Code compile(loc l) = compileProg(load(l));
```

## 今回実装したソースコード

https://github.com/whtsht/rascal-func

`examples`ディレクトリ下に`fib.func`と`fact.func`があります。どのようなコードが生成されるか見たい場合はrascalをREPLを起動して確認できます。

```bash
$ rascal
rascal>import Main;
rascal>compile(|cwd:///examples/fib.func|);
map[str, list[Instr]]: ("fib":[
    pushId("n"),
    setVal(),
    pushId("n"),
    getVal(),
    pushNat(2),
    lt(),
    goIf("L1"),
    pushId("n"),
    getVal(),
    pushNat(1),
    sub(),
    pushId("fib"),
    call(),
    pushId("n"),
    getVal(),
    pushNat(2),
    sub(),
    pushId("fib"),
    call(),
    add(),
    go("L2"),
    label("L1"),
    pushId("n"),
    getVal(),
    label("L2")
  ])
```

## おわりに

Rascalでコンパイラを作りました。

配列の平坦化や柔軟なパターンマッチなどが書きやすいだけで、コンパイラ書くときの快適さが他言語とかなり違うと私は思いました。実際に比較してみたいです。

今回は変数が宣言されているかどうかなどのソースコードの検証を全くしていません。Rascal公式サイトの[型チェックの例](https://www.rascal-mpl.org/docs/Recipes/Languages/Pico/Typecheck/)などを参考にすればできると思うので、興味のある方はぜひ。

## 参考文献

https://www.rascal-mpl.org/docs/Recipes/Languages/Func/

https://www.rascal-mpl.org/docs/Recipes/Languages/Pico/
