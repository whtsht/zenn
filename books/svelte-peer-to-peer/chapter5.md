---
title: "ビデオ通話機能の追加"
free: true
---

## 準備

前の章で作ったチャットアプリを拡張して，ビデオ通話機能を追加しましょう！

もしこの章からコーディングしたい場合は好きなディレクトリ下で以下のコマンドを実行してください．

```
$ git clone https://github.com/whtsht/webrtc-playground.git && cd webrtc-playground && git switch chat
```

## 設計

WebRTC ではビデオ通話を行うための機能が用意されています．しかしビデオ通話を行うためにはビデオやオーディオのコーディックを相手に伝えなければなりません．2 章で説明したようにビデオのコーディックなどは SDP に含まれています．SDP にコーディックを含めるためには実際にビデオやオーディオを取得して PeerConnection に設定する必要があります．私達は 3 章でこのような処理を書いたでしょうか．はい，書いていませんでしたね．ということで **SDP の受け渡しからやり直し**です．「またそんな面倒なことを......」と思うかもしれませんが，これが全く環境が違うコンピュータ同士で P2P 通信をするということです．しっかりと噛み締めていきましょう．

同じ部屋に入っているメンバー同士のみビデオ通話ができる仕様にします．処理の流れは以下のようにします．

### ビデオ通話を開始する側（ホスト）

1. ユーザーがビデオ通話ボタンを押す
2. 他のメンバーにビデオ通話招待メッセージを送る
3. ビデオ通話画面を開く

### ビデオ通話に参加する側

1. ビデオ通話招待メッセージを受け取る
2. ビデオ通話招待メッセージの Join ボタンを押す
3. ホストのメンバーにビデオ通話に参加しているメンバーを教えてもらう
4. 教えてもらったメンバーに対してビデオ通話のコネクションを作成し，SDP，ICE を交換する
5. ビデオ通話画面を開く

## WebRTC のチャネルを使った SDP の交換

もう一度シグナリングサーバーを使って SDP の交換を行ってもよいのですが，せっかく前章で苦労して接続した RTCPeerConnection があるので，それを使って SDP の交換を行ってみましょう．

`videoCallConnection` と `negotiateChannel` を追加します．`videoCallConnection` はビデオ通話に使う RTCPeerConnection です．`negotiateChannel` は SDP の交換に使うチャネルです．

```diff typescript:web/src/webrtc.ts
...

 interface PeerConnection {
     chatConnection: RTCPeerConnection;
+    videoCallConnection: RTCPeerConnection | null;
     chatChannel: RTCDataChannel;
+    negotiateChannel: RTCDataChannel;
 }

...
```

新しく web/src/videoCall.ts を作成します．

```typescript:web/src/videoCall.ts
import { get, writable, type Writable } from "svelte/store";
import { userName } from "./chat";
import { open } from "./lib/Snackbar.svelte";
import { connects, sendAnswer, sendOffer, setAnswer, setOffer } from "./webrtc";

// ビデオ通話の参加者
export const videoCallMembers: Writable<string[]> = writable([]);

// 自身のストリーム
export const localStream: Writable<null | MediaStream> = writable(null);

// リモートのストリーム
export const remoteStreams: Writable<Map<string, MediaStream>> = writable(
    new Map()
);

// ビデオ通話画面を表示するかどうか
export const display = writable(false);

// ピア間でやり取りするメッセージの種類
type Negotiate =
    | { type: "startNegotiate"; from: string }
    | { type: "offer"; from: string; sdp: string }
    | { type: "answer"; from: string; sdp: string }
    | { type: "candidate"; from: string; candidate: string }
    | { type: "getMembers"; from: string }
    | { type: "members"; from: string; members: string[] };

export function createNegotiationChannel(peer: RTCPeerConnection) {
    const negotiateChannel = peer.createDataChannel("negotiate");
    negotiateChannel.onmessage = async (ev) => {
        const negotiate: Negotiate = JSON.parse(ev.data);
        if (negotiate.type === "startNegotiate") {
            console.log(`[video startNegotiate] from: ${negotiate.from}`);
            const connect = connects.get(negotiate.from)!;
            const peer = newPeerConnection(negotiate.from);
            connect.videoCallConnection = peer;
            return;
        }
        // チャットメンバーの取得
        if (negotiate.type === "getMembers") {
            console.log(`[video getMembers] from: ${negotiate.from}`);
            sendNegotiate(negotiate.from, {
                type: "members",
                from: get(userName),
                members: get(videoCallMembers),
            });
            return;
        }
        // メンバーの更新
        if (negotiate.type === "members") {
            console.log(
                `[video members] from: ${negotiate.from}, members: ${negotiate.members}`
            );
            // メンバーがいなかったらメッセージを表示して終了
            if (negotiate.members.length === 0) {
                open("video call is finished");
                return;
            }
            videoCallMembers.set(negotiate.members);
            display.set(true);
            return;
        }
        if (negotiate.type === "offer") {
            console.log(
                `[video offer] from: ${negotiate.from}, sdp: ${negotiate.sdp}`
            );
            const peer = getPeer(negotiate.from);
            await setOffer(peer, JSON.parse(negotiate.sdp));
            await sendAnswer(peer, negotiate.from, sendSdp);
            return;
        }
        if (negotiate.type === "answer") {
            console.log(
                `[video answer] from: ${negotiate.from}, sdp: ${negotiate.sdp}`
            );

            const peer = getPeer(negotiate.from);
            await setAnswer(peer, JSON.parse(negotiate.sdp));
            return;
        }
        if (negotiate.type === "candidate") {
            console.log(
                `[video candidate] from: ${negotiate.from}, sdp: ${negotiate.candidate}`
            );

            getPeer(negotiate.from).addIceCandidate(
                JSON.parse(negotiate.candidate)
            );
        }
    };

    return negotiateChannel;
}

export function getMembers(to: string) {
    sendNegotiate(to, { type: "getMembers", from: get(userName) });
}

export async function startNegotiate(to: string, from: string) {
    sendNegotiate(to, {
        type: "startNegotiate",
        from,
    });
    await sendOffer(newPeerConnection, to, sendSdp);
}

function getPeer(to: string) {
    return connects.get(to)!.videoCallConnection!;
}

function newPeerConnection(to: string): RTCPeerConnection {
    const peer = new RTCPeerConnection();
    peer.onconnectionstatechange = (_) => {
        console.log(`connection state: ${peer.connectionState}`);
        if (peer.connectionState === "connected") {
            videoCallMembers.update((members) => [...members, to]);
        } else if (
            peer.connectionState === "disconnected" ||
            peer.connectionState === "failed" ||
            peer.connectionState === "closed"
        ) {
            videoCallMembers.update((members) =>
                members.filter((member) => member !== to)
            );
            remoteStreams.update((streams) => {
                streams.delete(to);
                return streams;
            });
        }
    };

    peer.onicecandidate = (ev) => {
        if (ev.candidate) {
            sendNegotiate(to, {
                type: "candidate",
                from: get(userName),
                candidate: JSON.stringify(ev.candidate),
            });
        }
    };

    // ストリームのトラックを追加
    const stream = get(localStream)!;
    stream.getTracks().forEach((track) => {
        peer.addTrack(track, stream);
    });

    // リモートのトラックを受け取ったときの処理
    peer.ontrack = (ev) => {
        console.log(peer.connectionState);
        // リモートのストリームを更新
        remoteStreams.update((streams) => streams.set(to, ev.streams[0]));
    };

    connects.get(to)!.videoCallConnection = peer;
    return peer;
}

function sendNegotiate(to: string, value: Negotiate) {
    const connect = connects.get(to)!;
    connect.negotiateChannel.send(JSON.stringify(value));
}

function sendSdp(
    type: "offer" | "answer",
    sdp: RTCSessionDescription,
    to: string,
    from: string
) {
    sendNegotiate(to, { type, from, sdp: JSON.stringify(sdp) });
}
```

`createNegotiationChannel` は `negotiateChannel` を作成する関数です．
`Negotiate`という型のメッセージの送受信を行います．`Negotiate`で送っているメッセージは前章でシグナリングサーバーに送っていいたメッセージと同じような形をしているのがわかると思います．

`getMembers`はビデオ通話に参加したいメンバーがビデオ通話のホストに対して送るメッセージです．`getMembers`が送られていたホストはビデオ通話に参加したいメンバーに対して `members` を送ります．`members`を受け取ったメンバーはビデオ通話にすでに参加しているメンバーに対して `startNegotiate` を送ります．この後は SDP や ICE の交換を行います．

次に web/src/chat.ts に`negotiateChannel`を追加します．

```diff typescript:web/src/chat.ts
 import { connect } from "socket.io-client";
 import { get, writable, type Writable } from "svelte/store";
+import { createNegotiationChannel } from "./videoCall";
 import { connects, sendAnswer, sendOffer, setAnswer, setOffer } from "./webrtc";

...

 function newPeerConnection(to: string): RTCPeerConnection {
     const peer = new RTCPeerConnection();
     peer.onconnectionstatechange = (_) => {
         console.log(`connection state: ${peer.connectionState}`);
         if (peer.connectionState === "connected") {
             chatMembers.update((members) => [...members, to]);
         } else if (
             peer.connectionState === "disconnected" ||
             peer.connectionState === "failed" ||
             peer.connectionState === "closed"
         ) {
             chatMembers.update((members) =>
                 members.filter((member) => member !== to)
             );
         }
     };

     peer.ondatachannel = (ev) => {
-        const { chatConnection } = connects.get(to)!;
-        connects.set(to, {
+        const {
             chatConnection,
-            chatChannel: ev.channel,
-        });
+            videoCallConnection,
+            chatChannel,
+            negotiateChannel,
+        } = connects.get(to)!;
         // チャネルのラベルでチャネルの種類を判別
+        if (ev.channel.label == "chat") {
+            connects.set(to, {
+                chatConnection,
+                videoCallConnection,
+                chatChannel: ev.channel,
+                negotiateChannel,
+            });
+        } else {
+            connects.set(to, {
+                chatConnection,
+                videoCallConnection,
+                chatChannel,
+                negotiateChannel: ev.channel,
+            });
+        }
     };

     const chatChannel = peer.createDataChannel("chat");
     chatChannel.onmessage = (ev) => {
         const message: ChatMessage = JSON.parse(ev.data);
         console.log(message);
         chatMessages.update((messages) => {
             return [...messages, message];
         });
     };

+    const negotiateChannel = createNegotiationChannel(peer);
+
     peer.onicecandidate = (ev) => {
         if (ev.candidate) {
             console.log(ev.candidate);
             sendCandidate(ev.candidate, to, socket.id);
         }
     };

     connects.set(to, {
         chatConnection: peer,
         // ここではまだビデオ通話のコネクションは作成しない
+        videoCallConnection: null,
         chatChannel,
+        negotiateChannel,
     });

     return peer;
 }

```

## ビデオ通話の UI を作成する

新しく web/src/lib/VideoCall.svelte を作成します．

```typescript:web/src/lib/VideoCall.svelte
<script lang="ts">
    import Modal from "./Modal.svelte";
    import { userName } from "../chat";
    import {
        display,
        localStream,
        videoCallMembers,
        remoteStreams,
        startNegotiate,
    } from "../videoCall";
    import { connects } from "../webrtc";

    let video: HTMLVideoElement | null = null;

    function closeCall() {
        $videoCallMembers.forEach((user) => {
            if (user === $userName) return;
            const peer = connects.get(user)!;
            if (peer.videoCallConnection) {
                peer.videoCallConnection!.close();
                peer.videoCallConnection = null;
            }
        });
        video!.srcObject = null;
        $videoCallMembers = [];
        $display = false;
    }

    $: $display, onSwitch();
    async function onSwitch() {
        if ($display) {
            $localStream = await navigator.mediaDevices.getUserMedia({
                audio: true,
                video: true,
            });
            if (video) {
                video.srcObject = $localStream;
            }
            for (let i = 0; i < $videoCallMembers.length; i++) {
                if ($videoCallMembers[i] === $userName) continue;
                await startNegotiate($videoCallMembers[i], $userName);
            }
        } else {
            if (video) {
                video.pause();
                video.srcObject = null;
            }
            if ($localStream) {
                $localStream.getTracks().forEach((track) => track.stop());
            }
            $localStream = null;
        }
    }

    function srcObject(node: HTMLVideoElement, stream: MediaStream) {
        node.srcObject = stream;
        return {
            update() {
                node.srcObject = stream;
            },
            destroy() {
                node.srcObject = null;
            },
        };
    }
</script>

<Modal display={$display}>
    <div class="panel">
        <div>
            <video
                width="400px"
                height="300px"
                muted
                autoplay
                bind:this={video}
            />
            {#each $remoteStreams as [_, stream]}
                <!-- svelte-ignore a11y-media-has-caption -->
                <video
                    width="400px"
                    height="300px"
                    autoplay
                    use:srcObject={stream}
                />
            {/each}
        </div>

        <button on:click={closeCall}>Close</button>
    </div>
</Modal>

<style>
    .panel {
        background-color: #fff;
        padding: 20px;
    }

    video {
        margin: 10px;
        border: solid;
    }
</style>
```

`video`タグを使ってビデオ通話の画面を表示します．`video`タグの`srcObject`に`MediaStream`を設定することでビデオを表示することができます．`onSwitch`では`display`が`true`になったときに`getUserMedia`を使ってビデオとオーディオのストリームを取得します．`$videoCallMembers`のメンバーに対して`startNegotiate`を行い SDP を交換します．`display`が`false`になったときには`video`タグの`srcObject`を`null`に設定してストリームを停止します．

`closeCall`ではビデオ通話を終了するときに行う処理を書いています．`$videoCallMembers`のメンバーに対して`videoCallConnection`を閉じています．

画面は`video`タグを並べたものと`Close`ボタンを表示するだけのものです．

`<!-- svelte-ignore a11y-media-has-caption -->`は`A11y: Media elements must have a <track kind="captions"> `という警告を非表示にします．字幕をつけろと言われているようですが，今回は無視してください．

https://developer.mozilla.org/en-US/docs/Web/Guide/Audio_and_video_delivery/Adding_captions_and_subtitles_to_HTML5_video

https://github.com/sveltejs/svelte/issues/5967

## ビデオ通話招待メッセージ

`ChatMessage`にビデオ通話用のメッセージを追加します．

```diff typescript:web/src/chat.ts
...

-export type ChatMessage = { user: string; type: "chat"; message: string };
+export type ChatMessage =
+    | { user: string; type: "chat"; message: string }
+    | { user: string; type: "videoCall" };

 export const chatMessages: Writable<ChatMessage[]> = writable([]);

 ...
```

新しく web/src/lib/Chat/VideoCallMessage.svelte を作成します．

```typescript:web/src/lib/Chat/VideoCallMessage.svelte
<script lang="ts">
    import { userName } from "../../chat";
    import { getMembers } from "../../videoCall";
    import SpeechBubble from "../SpeechBubble.svelte";
    export let user: string;
</script>

<SpeechBubble name={user}>
    <p>Let's start video call!</p>
    {#if user !== $userName}
        <button style="margin-bottom: 20px;" on:click={() => getMembers(user)}
            >Join</button
        >
    {/if}
</SpeechBubble>
```

自分以外からのビデオ通話の招待メッセージには`Join`ボタンを表示します．

ビデオ通話招待メッセージを表示するように変更します．

```diff typescript:web/src/lib/Chat/Main.svelte
 <script lang="ts">
     import { fly } from "svelte/transition";
     import { chatMessages } from "../../chat";
+    import VideoCallMessage from "./VideoCallMessage.svelte";
     import ChatMessage from "./ChatMessage.svelte";
     import Sender from "./Sender.svelte";
 </script>

 <div class="chat">
     <div class="message-box">
         {#each $chatMessages as chat}
             <div in:fly={{ y: 100, duration: 500 }}>
-                <ChatMessage user={chat.user} message={chat.message} />
+                {#if chat.type === "chat"}
+                    <ChatMessage user={chat.user} message={chat.message} />
+                {:else}
+                    <VideoCallMessage user={chat.user} />
+                {/if}
             </div>
         {/each}
     </div>
     <Sender />
 </div>
 ...
```

## ビデオ通話招待メッセージ送信

Phone アイコンのボタンを押すとビデオ通話の招待メッセージを送るようにします．招待メッセージを送ると同時にビデオ通話画面を表示するようにします．

```diff typescript:web/src/lib/Chat/Sender.svelte
 <script lang="ts">
     import { chatMessages, roomName, sendMessage, userName } from "../../chat";
+    import {
+        videoCallMembers,
+        display as displayVideoCall,
+    } from "../../videoCall";
     import Send from "svelte-material-icons/Send.svelte";
+    import Phone from "svelte-material-icons/Phone.svelte";

     let value = "";

+    function startVideoCall() {
+        if ($roomName) {
+            chatMessages.update((messages) => [
+                ...messages,
+                { user: $userName, type: "videoCall" },
+            ]);
+            sendMessage({
+                type: "videoCall",
+                user: $userName,
+            });
+            $displayVideoCall = true;
+            $videoCallMembers = [...$videoCallMembers, $userName];
+        }
+    }
+
...

 </script>

 <div class="sender">
     <textarea bind:value on:keydown={handleKeyboard} />
     <button on:click={sendChatMessage}>
         <Send size="24px" viewBox="0 0 24 20" />
     </button>
+    <button on:click={startVideoCall}>
+        <Phone size="24px" viewBox="0 0 24 20" />
+    </button>
 </div>

...
```

部屋を退室するときにビデオ通話のコネクションを閉じるようにします．

```diff typescript:web/src/lib/LeaveRoom.svelte
 <script lang="ts">
...

     function leaveRoom() {
         $roomName = null;
         $chatMembers = [];
         $chatMessages = [];
         connects.forEach((connect) => {
             connect.chatConnection.close();
+            connect.videoCallConnection?.close();
         });
         close();
     }
...
 </script>
...
```

最後にビデオ通話画面を表示するために `App.svelte` を以下のように変更します．

```diff typescript:web/src/App.svelte
...
     import Chat from "./lib/Chat/Main.svelte";
     import SideBar from "./lib/SideBar.svelte";
     import Snackbar from "./lib/Snackbar.svelte";
+    import VideoCall from "./lib/VideoCall.svelte";
     import LeaveRoom from "./lib/LeaveRoom.svelte";
     import NewRoom from "./lib/NewRoom.svelte";
...

 <main>
     <SideBar bind:leaveRoom bind:newRoom />
     <Chat />
+    <VideoCall />
     <Snackbar />

     <LeaveRoom bind:this={leaveRoom} />

...
```

ここまで書けたらもうビデオ通話もできるはずです．片方が右下の Phone アイコンを押すとビデオ通話の招待メッセージが送られます．もう片方が Join ボタンを押すと 2 つのビデオ通話画面が表示されます．音声もちゃんと入っているはずです．

![](https://storage.googleapis.com/zenn-user-upload/3c39ef9f1f13-20231023.png)

## まとめ

この章ではビデオ通話機能を追加しました．WebRTC を使うとビデオ通話を簡単に実装することができました．前章との違いは SDP，ICE の交換にシグナリングサーバーを使わずに WebRTC のチャネル を使って行ったことと，メディアストリームの取得を行ったことです．
ブラウザからカメラ，マイクのアクセスを許可するかどうか聞かれると思います．許可しないとビデオ通話ができないので注意してください．
