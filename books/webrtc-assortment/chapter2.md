---
title: "WebRTCの基礎"
free: true
---

## WebRTC の構成要素

### Session Description Protocol (SDP)

RFC 8866

SDP は以下のようなテキスト形式で表されます．

```
v=0
o=alice 2890844526 2890844526 IN IP4 host.anywhere.com
s=
c=IN IP4 host.anywhere.com
t=0 0
m=audio 49170 RTP/AVP 0
a=rtpmap:0 PCMU/8000
m=video 51372 RTP/AVP 31
a=rtpmap:31 H261/90000
m=video 53000 RTP/AVP 32
a=rtpmap:32 MPV/90000
```

https://datatracker.ietf.org/doc/html/rfc8866

### Interactive Connectivity Establishment (ICE)

RFC 5245

```
candidate:193081078 1 udp 2113937151 94c69b82-ff4b-44eb-a103-5f423791f8a9.local 48968 typ host generation 0 ufrag NGK3 network-cost 999
```

https://datatracker.ietf.org/doc/html/rfc5245

## シグナリングサーバー

WebRTC ではシグナリングサーバーの仕様は定められていないです．なので，シグナリング方法は実装者に任されています．SDP をメールで送っても良いし．伝書鳩で送っても良いです．シグナリングサーバーとしてよく使われるのは WebSocket です．Firebase の Realtime Database や Firestore などのリアルタイムデータベースを使うこともできます．本アプリでは WebSocket を使います．

## WebRTC の通信の流れ

```

```

## STUN サーバー，TURN サーバーについて
