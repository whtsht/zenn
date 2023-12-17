---
title: "VimGolfでVimテクニックを磨く"
emoji: "🏌️‍♂️"
type: "tech"
topics: ["vim", "neovim"]
published: false
publish_at: 2023-12-18
---

:::message
この記事は[Vim駅伝](https://vim-jp.org/ekiden/)2023年12月18日の記事です。

前回の記事はkawarimidollさんの[Vimで現在のファイルのgit diffを確認する](https://zenn.dev/vim_jp/articles/b15bbd5b682cd8)でした。

次回の記事はぺりーさんの「VimをDebugする」です。
:::

## はじめに

Vim駅伝初参加ということで、少しだけVim愛を語らせて下さい。

私がVimに出会ったのは、大学に1年生のとき、授業でUnix系OSを触る機会があったことがきっかけでした。その後自分のパソコンにLinux環境やVimを入れて、自分用にカスタマイズことにすっかり魅了されました。なのでかれこれ3年くらいはVim/Neovimを使っていると思います。

今はNeovim & AstroNvim + α という構成で落ち着き、プログラミングをするとき、レポートを書くとき、Zennの記事を書くときなどに使っています。

そんな中Zennの記事内でVim駅伝という言葉をちらほら見かけるようになり、その後[Vim駅伝のアーカイブ](https://vim-jp.org/ekiden/archives/)を見るようになりました。これが面白いんですね。個人的には[Project V 「思考のスピードで編集せよ。32年の伝統をもつテキストエディタをビルドする。」](https://gist.github.com/tani/04c52c12ab4254528c1ba7ad509946ad)が好きです。そして自分も参加できたら素晴らしいと思い、Vim駅伝に出ることを決めました。プラグインもまともに作れないへなちょこVimmerですがどうぞよろしくお願いします。

では本題です。

今回は、最近見つけて面白いと思ったVimGolfを紹介します。

## VimGolfとは

VimGolfはVimを使って、ファイルの修正をなるべく少ない打鍵数（keystrokes）で行うというゲームです。

各問題にはStart fileとEnd fileファイルが与えられます。私達が行うことはStart fileをEnd fileと一致させてファイルの保存、Vimを終了させることです。

### 参加方法

まずは[VimGolf](https://www.vimgolf.com/)にアクセスしてサインインします。

その後、Rubyのパッケージマネージャーであるgemを使いvimgolfをインストールします。

```
$ gem install vimgolf
```

次に以下を実行します。

```
$ vimgolf setup
```

`Paste your VimGolf key:`と出るので、[VimGolf](https://www.vimgolf.com/)の右上に表示される`Your VimGolf key: xxxxx`の`xxxxx`の部分をコピペして下さい。

これで準備は完了です。[VimGolf](https://www.vimgolf.com/)の`Open VimGolf challenges`から好きなチャレンジを選び、右上に表示されている`vimgolf put xxxxx`を実行すればOKです。

例）[Quicksort](https://www.vimgolf.com/challenges/9v00651eb20100000000025b)にチャレンジする場合

```
$ vimgolf put 9v00651eb20100000000025b
```

Vimが開くので、[Quicksort](https://www.vimgolf.com/challenges/9v00651eb20100000000025b)に表示されるEnd fileと同じになるよう編集して`:wq<Enter>`でVimを終了させます。

```
$ vimgolf put 9v00651eb20100000000025b
Downloading Vimgolf challenge: 9v00651eb20100000000025b
Launching VimGolf session for challenge: 9v00651eb20100000000025b

Here are your keystrokes:
...

Success! Your output matches. Your score: XX
[w] Upload result and retry the challenge
[x] Upload result and quit
[r] Do not upload result and retry the challenge
[q] Do not upload result and quit
Choice>
```

x + エンターキーでVimGolfのサイトに自分の解答がアップロードされます。

詳細はGithubページを参考にして下さい。

https://github.com/igrigorik/vimgolf

それでは紳士淑女の華麗なる遊戯、VimGolfを始めましょう！

## やってみる

VimGolfのサイトの一番上にあった[Quicksort](https://www.vimgolf.com/challenges/9v00651eb20100000000025b)をやってみようと思います。

```:Start file
21 8 144 3 89 5 13 34 55 2


............................................................................
9G3W"fyEW"kyEW"qyE@f w@n8Gyfj}jj$p@rkp@r@0@o"qp$hyfg$p0"qy$fj}jj$p@rkp@r@0pG
f 0y$"_dd 6Gyw@o"kp"0p0"ky$"my$dd "lyy"lp0d$ @mby2eG2@o"0p yiw@o"0p $hyfg$p$
@n6Gtoy2l}++$p@r@0{+ }jj$p@rkp@r@0@o"qp$hyfg$p0"qy$fj}"jyEW"ayEW"byE+"syEW"x
@jV"ep "eyiw@nGG8Gyfj}jj$p@rk@0{ }-"xyiw*``{ndiw{w"eyiw{dG@mlviw"xP@mhdiw"eP
@c@d@d@aww@a@b @m@s@h @o"mp$xx0"my$dd @o"qp@r@0@o"qp$hyfg$p0"qy$dd@f gg @i@g
6Gw"ryEW"iyEW"oyEW"cyEW"nyE+"dyE+l"jyEW"ayEW"byE+"syEW"gyEW"hyE5Gy$@0 oyEWgg
```

```:End file
2 3 5 8 13 21 34 55 89 144


............................................................................
9G3W"fyEW"kyEW"qyE@f w@n8Gyfj}jj$p@rkp@r@0@o"qp$hyfg$p0"qy$fj}jj$p@rkp@r@0pG
f 0y$"_dd 6Gyw@o"kp"0p0"ky$"my$dd "lyy"lp0d$ @mby2eG2@o"0p yiw@o"0p $hyfg$p$
@n6Gtoy2l}++$p@r@0{+ }jj$p@rkp@r@0@o"qp$hyfg$p0"qy$fj}"jyEW"ayEW"byE+"syEW"x
@jV"ep "eyiw@nGG8Gyfj}jj$p@rk@0{ }-"xyiw*``{ndiw{w"eyiw{dG@mlviw"xP@mhdiw"eP
@c@d@d@aww@a@b @m@s@h @o"mp$xx0"my$dd @o"qp@r@0@o"qp$hyfg$p0"qy$dd@f gg @i@g
6Gw"ryEW"iyEW"oyEW"cyEW"nyE+"dyE+l"jyEW"ayEW"byE+"syEW"gyEW"hyE5Gy$@0 oyEWgg
```

1行目にある文字列を昇順に並び替えよ、という問題です。ファイル後半にある謎文字列が気になりすぎますが、一旦無視しましょう。きっと謎文字列を使うエスパーのような解答例があるのでしょう。たぶん。

### 1回目

取り敢えず何も考えずにやってみます。

```
vimgolf put 9v00651eb20100000000025b
Downloading Vimgolf challenge: 9v00651eb20100000000025b
Launching VimGolf session for challenge: 9v00651eb20100000000025b

Here are your keystrokes:
$x0Pa <Esc>wwwwvld0lpwwwwwvld0wlpwwwwwvlldbbbbhpwwvldbbblpwwwwwvlldbbblpulpwwwvlldbbhpwwvlldbbllpllllx:wq<CR>

Success! Your output matches. Your score: 102
```

スコアは102、`Here are your keystrokes:`にあるものが自分の押したキーですね。

移動 -> ビジュアルモードに入る -> 選択 -> 削除 -> 移動 -> ペーストを繰り返しています。いかにもVim初心者がやりましたという感じの解答になってしまいました。しかもミスって`u`で戻ったりしてます。結果は140 active golfersのうち139位でした。そりゃそうですね。

VimGolfでは自分の解答をアップロードすると、自分より少し上のスコアの人の解答が見れるようになります。そのため、Vimの新しい知見も得られ、さらに自分のスコアを確実に伸ばしていくことができます。いいですね。

私よりスコアの上の人のkeystrokesを見ると、どうやらビジュアルモードに入らずに`dw`で削除しているようです。不用意にビジュアルモードに入る自分の悪癖がわかりますね。ふむふむ。

### 2回目

……待てよ

これって最初全部消してインサートモードで打ち込めば速いんじゃないか？

```
vimgolf put 9v00651eb20100000000025b
Downloading Vimgolf challenge: 9v00651eb20100000000025b
Launching VimGolf session for challenge: 9v00651eb20100000000025b

Here are your keystrokes:
Di2 3 5 8 13 21 34 55 89 144<Esc>:wq<CR>

Success! Your output matches. Your score: 33
```

うひゃひゃひゃ🤪天才だ！

と思いましたが、他にも同じ方法をやってなおかつスコアが32の人がいました。どうやったかわかるでしょうか。

:::details 答え

`R`で置換モード(Replace mode)に入ればよいですね。これで`Di`を`R`に短縮できます。私は置換モードの存在を完全に忘れてました。あと、`r`で1文字置換ができます。
:::

さらにソートを使うとスコアを30に短縮できたりします。

ここから先は皆さんの目で確認してみてください。

## おわりに

VimGolfを紹介して、実際に問題を解いてみました。他人の解答を見ると、いかに自分が初歩的なコマンドを忘れているかを痛感しました。一度[Vimtutor](https://vim-jp.org/vimdoc-ja/usr_01.html#01.3)やり直そうかな。

VimGolfの嬉しいところは、自分の最高スコアの少し上の人の解答を見れるところです。これによって、少しずつ自分の忘れていたコマンドを思い出したり、新しいテクニックを身につけたりできます。これを公式サイトでは梯子を登る（climb the ladder）と表現しています。私はQuicksortの最高スコア6を目指して挑戦し続けようと思います。

もちろん問題はQuicksortだけでなく、様々なテクニックが必要になるチャレンジが現在（2023年12月03日）556個あります。皆さんもぜひVimのテクニック向上のためにVimGolfをしてみてはいかがでしょうか。
