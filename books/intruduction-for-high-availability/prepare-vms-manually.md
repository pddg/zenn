---
title: "VMを作ってみる"
---

## Proxmox VEでVMを作る

このセクションでは、実際に構築済みのProxmox VEを使ってVMを作っていきます。ProxmoxはKVM/QEMUを利用してVMを作成します。これらの技術に関する詳細は特に触れません。気になった方は以下の参考書籍・サイトを参照するのが良いのではないでしょうか。

- [［試して理解］Linuxのしくみ―実験と図解で学ぶOS、仮想マシン、コンテナの基礎知識【増補改訂版】](https://gihyo.jp/book/2022/978-4-297-13148-7)
- [Documents - linux-kvm.org](https://www.linux-kvm.org/page/Documents)

## 通常のサーバ用イメージを使ってインストールする

今回はサーバOSとしてUbuntu Server 22.04 LTSを使用します。物理マシンにUbuntu Serverをインストールする手段として用いられる、ISOファイルのインストーラーを使う方法でUbuntuがインストールされたVMを作ってみます。

:::message
本セクションではISOイメージをダウンロードするためそれなりの量の通信が発生します。
インターネット接続にモバイル回線を利用している場合は注意してください。
:::

### ISOファイルをアップロードする

今回は日本国内のミラーからISOを取得します。以下のページは理研のミラーです。
https://ftp.riken.jp/Linux/ubuntu-releases/jammy/

Server install imageというセクションのリンクから最新のlive serverのISOファイルをダウンロードします。
`ubuntu-22.04.3-live-server-amd64.iso` のようなISOファイルがダウンロードできるはずです（細かいバージョンの差異があるかもしれません）。

ProxmoxのWeb UIを開いて `Datacenter` > `node1` > `local (node1)` を選択し、 `ISO Images` を開きます。

![proxmox-webui-iso.png](/images/books/introduction-for-high-availability/proxmox-webui-iso.png)

ダウンロードしたISOファイルを指定してアップロードします。

![proxmox-webui-upload-iso.png](/images/books/introduction-for-high-availability/proxmox-webui-upload-iso.png)

最終的に `Task viewer: Copy data` という画面が出ます。 `TASK OK` になっていれば成功です。

![proxmox-webui-complete-upload-iso.png](/images/books/introduction-for-high-availability/proxmox-webui-complete-upload-iso.png)

右上の×マークを押してメッセージを閉じ、以下の様にISOがアップロードされていることを確認します。

![proxmox-webui-iso-ubuntu.png](/images/books/introduction-for-high-availability/proxmox-webui-iso-ubuntu.png)

### アップロードしたISOからブートするVMを作る

ProxmoxのWeb UIの `Create VM` という上部のボタンから、VMの作成シーケンスを起動します。いくつも入力項目がありますが、簡単に今回作るVMを説明すると以下の様になっています。

- VM ID：`100`
- ホスト名：`sample`
- CPU：1コア
- メモリ：1 GiB
- ディスク：32 GiB
- ネットワーク：ブリッジネットワーク（ホストと同じネットワーク直下に紐付く）

![promox-webui-sample-vm-general.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-general.png)
![promox-webui-sample-vm-os.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-os.png)
![promox-webui-sample-vm-system.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-system.png)
![promox-webui-sample-vm-disks.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-disks.png)
![promox-webui-sample-vm-cpu.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-cpu.png)
![promox-webui-sample-vm-memory.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-memory.png)
![promox-webui-sample-vm-network.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-network.png)

最終的にconfirmして進めると、以下の様に `node1` の直下に `100` というVM IDを持つVMが作成されます。

![promox-webui-sample-vm-pre-start.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-pre-start.png)

### 実際にブートしてOSをインストールする

`▶ Start` を押すとブートすることができます。押した後、`Console` というメニューを選択することでブート中の画面をブラウザから開けます。
まずGRUBというブートローダ[^grub]の起動画面が表示されます。

[^grub]: https://wiki.archlinux.jp/index.php/GRUB

![promox-webui-sample-vm-iso-grub.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-iso-grub.png)

そのまま放っておいてもインストール画面に進みますが、早く進めたい場合はキーボードの `Enter` を押すことで先に進めます。

:::message
以降では見やすさのため、インストーラの画面のみのキャプチャを掲載します。
:::

しばらく待つと、以下の様にlive installerの画面が始まります。ここからは通常通りUbuntu Serverのインストールをするだけで特に大きな注意点等もないため、既にやったことの有る方はそのまま進めて適当にインストールしてもらえればOKです。

![proxmox-webui-sample-vm-installer-language.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-installer-language.png)
