---
title: "WebRTCを利用してビデオ通話や対戦ゲーム機能のあるアプリを作る"
emoji: "🌸"
type: "tech"
topics: ["web", "webrtc", "svelte"]
published: true
---

## はじめに

WebRTCを利用してビデオ通話や対戦ゲーム機能のあるアプリを作りました．実用に耐えうるものでは全くなくWebRTCの試験的なアプリとなっています．動機としては将来的に個人でリアルタイムに通信するアプリを公開したいと思ったからです．WebRTCはサーバーを介さないP2P通信です．よってサービス提供者は（通信においては）シグナリングサーバーのみ管理すれば良いのでコスト削減になると考えています．

このアプリを作る本を執筆しました．よろしくお願いします．

https://zenn.dev/whiteshirt/books/svelte-peer-to-peer

## 完成品

https://github.com/whtsht/webrtc-playground

主な機能は以下のとおりです

- リアルタイムチャット
- ビデオ通話
- Pongゲーム

こちらで動いているところを確認できます．
https://www.youtube.com/watch?v=ON8khxFI73k

## 実行方法

ここに書いてます．

https://github.com/whtsht/webrtc-playground/blob/main/README-jp.md#%E5%AE%9F%E8%A1%8C%E6%96%B9%E6%B3%95
Webサーバーとシグナリングサーバー両方起動できたら，http://localhost:5173 でアプリが起動するはずです．

複数のタブで開き，`New Room`を押して部屋名を指定して`Join`を押すとチャットが使えるようになります．ビデオ通話やPongをしたい場合は下のアイコンを押します．他のメンバーにはそれらの招待メッセージが送られるので`Join`を押して参加します．

## 使用した技術，フレームワーク

### WebRTC

今回の主役です．ブラウザ上でP2Pのリアルタイム通信を実現します．お互いに映像，音声のコーディックや通信経路をシグナリングサーバー越しに送りあい，お互いに使える通信経路を発見次第P2P通信を開始します．現在モバイルも含めほとんどのブラウザがサポートしています．

WebRTCはOSSプロジェクトでありApple、Google、Microsoft、Mozillaなどがサポートしています．

https://webrtc.org

https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API

https://caniuse.com/?search=webrtc

### WebSocket (Socket.IO)

シグナリングサーバーの通信に使いました．WebSocketは双方向通信をするためのプロトコルであり，HTTPと違いサーバーからクライアントに向けてデータを送信することができ，クライアントはポーリングせずにそのデータを受け取ることができます．

WebSocketのラッパーライブラリのSocket.IOを使いました．Socket.IOにはRoomという概念がありRoomに入っているメンバー全員に一斉送信することなどができます．素のWebSocketでも可能ですが，Socket.IOのほうがシンプルに記述できます．

https://en.wikipedia.org/wiki/WebSocket

https://socket.io/docs/v4/rooms/

### Svelte

最近お気に入りのWebフレームワークです．リアクティブプログラミングや状態管理などがシンプルに記述できるのが好印象です．また仮想DOMを使用していないのも特徴的です．実行時に仮想DOMを動かすコードを埋め込む必要がないのでバンドルサイズの縮小，パフォーマンスの向上が期待できます．

https://svelte.dev

## さいごに

バグの報告，コードの修正など大歓迎です．気軽にイシュー，プルリクください👋
