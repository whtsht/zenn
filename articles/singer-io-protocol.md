---
title: "OSSのETLプロトコル Singerを深掘る"
emoji: "🎤"
type: "tech"
topics: ["データ基盤", "etl"]
published: false
---

ETLの仕組み、具体的にどんなこと考慮しているのかなってことが気になってました
そこでSingerというプロトコルがとてもわかりやすいと思ったので整理したものです
使いかたではなくSingerの仕組みを深掘るだけの内容です

## Singerとは

データを分析する上で必要に応じてETL(Extract/Transform/Load)という作業が発生します

あまりイメージ湧かない場合は
Google spreadsheetsのデータを読み込んで（Extract）
分析に不要な列を削除して（Tranform）
分析用のに保存（Load）
みたいなワークフローを想像してください

https://www.singer.io/

それの仕組みを共通化して良い感じに再利用できるのがSingerという仕組みです

イメージはLanguage server protocolのように
TapとTargetが交換可能なことを示せる図がいいです

Salesforce
Mysql
S3

![](/images/singer_tap_target.png)

公式がPythonのテンプレートを用意しているため、公開されているSingerに準拠したツールのほとんどはPythonで実装されています。
しかし、標準入出力でJSONのやりとりができればどんな言語でも実装できます。
それくらいとてもシンプルなプロトコルです。

では見ていきましょう

## Singerの基本

Tap

Target

## スキーマの検出

## 増分インポートの仕組み

## おさらい：カスタムTapとTargetを作ってみよう

公式ではPythonをサポートしていますが
Python以外の言語でも作成できます

RustでTapとTargetを作成してみましょう
ローカルファイルのCSVを読み込むTap
Duckdbに書き込みを行うTarget

次にTapにディレクトリを指定できるようにして
1度出力したTapは出力されないようにする増分インポートを実装してみましょう

## 参考

https://github.com/singer-io/getting-started
