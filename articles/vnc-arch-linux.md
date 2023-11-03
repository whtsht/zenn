---
title: "[Arch Linux] 外出先から家のPCを使いたい"
emoji: "👜"
type: "tech"
topics: ["linux", "vnc"]
published: true
---

学校でUnityを使うことになりました。しかしノートPCのメモリが足りないのかUnityとVSCodeを起動するとカックカクになって作業にならないです。なので家のPCに接続してそこで作業すれば良いのでは？と考えました。通信速度によっては遠隔地にPCを画面を送るだけでカクついて見えそうですが取り敢えずやってみます。

## やること

- VNC設定
- SSHポートフォワーディング
- Wake-on-LAN

## 環境

### サーバー側(家PC)

```
OS: Arch Linux
Kernel: x86_64 Linux 6.5.5-arch1-1
Uptime: 13m
Packages: 2316
Shell: zsh 5.9
Resolution: 1920x1080
DE: KDE 5.110.0 / Plasma 5.27.8
CPU: AMD Ryzen 7 5700G with Radeon Graphics @ 16x 4.673GHz
GPU: AMD Radeon Graphics (renoir, LLVM 16.0.6, DRM 3.54, 6.5.5-arch1-1)
RAM: 6736MiB / 15356MiB
```

### クライアント側(ノートPC)

```
OS: Arch Linux
Kernel: x86_64 Linux 6.5.5-arch1-1
Uptime: 1h 2m
Packages: 1085
Shell: zsh 5.9
Resolution: 1920x1080
DE: KDE 5.110.0 / Plasma 5.27.8
CPU: AMD Ryzen 5 5500U with Radeon Graphics @ 12x 4.056GHz
GPU: AMD Radeon Graphics (renoir, LLVM 16.0.6, DRM 3.54, 6.5.5-arch1-1)
RAM: 2799MiB / 6753MiB
```

## VNC設定

サーバーの設定をしていきます。今回はTigerVNCを使います。[tigervnc](https://archlinux.org/packages/?name=tigervnc)パッケージをインストールします。

```
$ sudo pacman -S tigervnc
```

その後パスワードを設定します。

```
$ vncpasswd
Password:
Verify:
Would you like to enter a view-only password (y/n)? n
```

VNCの設定を書きます。

```:~/.vnc/config
session=plasma
geometry=1920x1080
localhost
alwaysshared
```

パスワード付きで起動します。

```
$ x0vncserver -rfbauth ~/.vnc/passwd
```

xprofileに以下のように記述をすれば自動的に起動できます。

```:~/.xprofile
...
$ x0vncserver -rfbauth ~/.vnc/passwd &
```

## SSHポートフォワーディング

VNCはデータを圧縮して送りますが暗号化はしません。なのでSSH(TSL)の通信経路を通して暗号化するSSHポートフォワーディングを行うと良いらしいです。あえて図示化するならこんな感じです。

![](https://storage.googleapis.com/zenn-user-upload/a8c4fd81db5c-20231002.jpg)

まずはサーバーとクライアントでSSH接続できるようにします。このへんの説明は割愛します。

クライアント側からSSHポートフォワーディングします。

```
$ ssh <target_IP_address> -L 9900:localhost:5900
```

後はVNCViewを使って遠隔操作ができるようになります。GUI画面から電源を切ることも可能です。

```
$ vncviewer localhost:9900
```

## Wake-on-LAN

Wake-on-LANとは遠隔でPCの電源を入れたり、スリープ状態から復帰させたりする仕組みです。
これがあれば家のPCを使ってないときは電源を切ったりして電気代を節約できます。

まずサーバー側で[ethtool](https://archlinux.org/packages/?name=ethtool)をインストールします。

Wake-on-LANが有効かどうか確認します。

```
$ sudo ethtool <interface> | grep Wake-on
Supports Wake-on: pumbag
Wake-on: d
```

d(disable) になっていたらWake-on-LANを有効にします。

```
$ sudo ethtool <interface> wol g
```

私の環境ではこれでうまく行きました。うまく行かない場合はBIOS/UEFIで"PCI Power up"、"Allow PCI wake up event"、"Boot from PCI/PCI-E"などの項目を探して有効化します。これらの項目がない場合はWake-on-LANをサポートしていない可能性があります。

次にクライアント側で[wol](https://archlinux.org/packages/?name=wol)をインストールします。

```
$ sudo pacman -S wol
```

クライアントからサーバーの電源を入れることを試みます。同じネットワーク内に属している場合はMACアドレスのみを指定すればOKです。

```
$ wol <target_MAC_address>
```

違うネットワークに属している場合はルータのIPアドレス(グローバルIPアドレス)を指定します。

```
$ wol -p <forwarded_port> -i <router_IP> <target_MAC_address>
```

MACアドレスは以下のコマンドで確認できます。

```
$ ip link | grep <interface> -A 1 | awk '/link\/ether/ {print $2}'
```

グローバルIPアドレスは以下のコマンドで確認できます。

```
$ curl inet-ip.info
```

## 終わりに

あとは家のPCのIPアドレスを固定、ポート転送の設定をして5900を外部に公開すればどこからでも使えるようになります。不正アクセスされる可能性があるので不要なポートは公開しないように注意して設定しましょう。

## 参考文献

https://wiki.archlinux.org/title/TigerVNC

https://wiki.archlinux.org/title/Wake-on-LAN

大抵のことはArchWikiに書いてあります。
