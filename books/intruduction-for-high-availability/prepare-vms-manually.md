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

`Datacenter` > `node1` > `100 (node1)` を選択し、右上あたりに表示される `▶ Start` を押すとブートできます。押した後、`Console` というメニューを選択することでブート中の画面をブラウザから開けます。
まずGRUBというブートローダ[^grub]の起動画面が表示されます。

[^grub]: https://wiki.archlinux.jp/index.php/GRUB

![promox-webui-sample-vm-iso-grub.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-iso-grub.png)

そのまま放っておいてもインストール画面に進みますが、早く進めたい場合はキーボードの `Enter` を押すことで先に進めます。

:::message
以降では見やすさのため、インストーラの画面のみのキャプチャを掲載します。
:::

しばらく待つと、以下の様にlive installerの画面が始まります。
ここからは通常通りUbuntu Serverのインストールをするだけで特に大きな注意点等もないため、既にやったことの有る方はそのまま進めて適当にインストールしてもらえればOKです。

![proxmox-webui-sample-vm-installer-language.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-installer-language.png)

基本的にはTabキー、矢印キーで移動し、Enterで決定、Spaceでチェックボックスをトグルして進めていきます。ユーザ名やパスワードなどはキーボードで通常通り入力できます。
いくつか省略しているページがありますが、それらは基本的にデフォルトのまま進めればOKです。

基本的には自動で最新のものを使うようにして進めます。
![proxmox-webui-sample-vm-installer-newver.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-installer-newver.png)

キーボード配列を選択します。自分はUSなのでそのままです。
![proxmox-webui-sample-vm-installer-keyboard.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-installer-keyboard.png)

インストールの方針を決めます。minimizedにすると本当に最小限になってしまうので、デフォルトで進めます。
![proxmox-webui-sample-vm-installer-type.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-installer-type.png)

ネットワークの設定をします。ここでは、基本的にはDHCPで設定されていれば良いです。また、このとき割り振られたIPアドレスのレンジがProxmoxをインストールした物理マシンのアドレスと同じレンジであることを確認してください。これはLinuxのブリッジインタフェースを使って実現されており、ホストと同じネットワークにVMが作成されることを意味します。
![proxmox-webui-sample-vm-installer-network.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-installer-network.png)

ここから先しばらくはデフォルトの設定のまま進めます。proxyやmirrorの設定は全てデフォルトのままで大丈夫です。
ストレージやパーティションの設定もデフォルトのままで問題ありません。

![proxmox-webui-sample-vm-installer-storage.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-installer-storage.png)
![proxmox-webui-sample-vm-installer-partition.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-installer-partition.png)

こういう警告が出ますが、特に消えて困る物は何もないのでそのままContinueします。
![proxmox-webui-sample-vm-installer-storage-confirm.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-installer-storage-confirm.png)

ユーザ名やパスワードの設定をします。適当に設定してください。
![proxmox-webui-sample-vm-installer-user.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-installer-user.png)

Ubuntuを提供するCanonical社の有料サブスクリプションであるUbuntu Proを有効化するかどうか聞かれます。これは個人ユーザであれば5台まで無料で提供され、広範囲のセキュリティフィックスなどを受けることができるものですが今回は不要なのでSkipします。
![proxmox-webui-sample-vm-installer-ubuntupro.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-installer-ubuntupro.png)

`Install OpenSSH server` にチェックを入れて進めます。今回はProxmoxのUIからコンソールに触れるため必須ではありませんが、物理マシンに直接インストールする際はこれを忘れてディスプレイから外し、後で接続出来なくて頭を抱えることになるのでチェックを入れましょう。
最初から `ssh-import-key` コマンドによる公開鍵のインポートが出来るようになっています。下記の画像の様にGitHubのユーザを指定して公開鍵をインポートできます。これを利用しない場合、利用できる認証方式がパスワードのみのため `Allow password authentication over SSH` を必ず有効化しましょう。
![proxmox-webui-sample-vm-installer-sshd.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-installer-sshd.png)

[snap](https://snapcraft.io/docs) というCanonicalが推しているパッケージマネージャを使ってアプリケーションをインストールしますか、という確認が出ますが全て不要なのでデフォルトのまま次へ進めます。
![proxmox-webui-sample-vm-installer-snap.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-installer-snap.png)

しばらく待っていると以下の様に `Install complete!` という表示になるので `Reboot Now` します。リブート中にcdromのunmountに失敗した、みたいなエラーが出ますが気にする必要は無いです。
![proxmox-webui-sample-vm-installer-snap.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-installer-completed.png)

再起動後、コンソールにはcloud-initの実行ログが流れます。これはちょっと画面が見づらくなりますが、特にユーザの操作を阻害するものではありません。設定したユーザ名とパスワードを入力してログインしましょう。

### qemu-guest-agentのインストール

`Datacenter` > `node1` > `100 (sample)` を選択し、 `Summary` を開くとCPUやメモリの使用状態が見られる様になっています。一方、 `IPs` という項目は `Guest Agent not running` となっておりどのようなIPアドレスを設定されているのか見ることができません。これを見るためにQEMUのGuest Agentをインストールします。

![proxmox-webui-sample-vm-summary-without-agent.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-summary-without-agent.png)

ProxmoxのWeb UIのコンソール経由だとおそらくコピー&ペーストできないので、手で入力してください。

```sh
# 今回はrootユーザではないので sudo を使って一時的に権限を得る
sudo apt update
sudo apt install -y qemu-guest-agent
# 再起動する
sudo reboot
```

再起動後、以下の様にIPアドレスが見えるようになっているはずです[^maskv6]。
![proxmox-webui-sample-vm-summary-with-agent.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-summary-with-agent.png)

[^maskv6]: 筆者の環境ではグローバルなIPv6アドレスが見えていたのでマスクしています。IPv6非対応の環境では、IPv4アドレスのみが表示されると思います。

## 作成したVMにアクセスする

Guest Agentをインストールしたことによって、割り振られているIPv4アドレスがわかりました。このアドレスに対して、クライアントマシンからSSHでアクセスしてみます。

```sh
# 対象のIPアドレスを入れる。今回の例であれば以下の様になる。
ADDR="192.168.16.38"
# 設定したユーザ名を入れる。クライアントマシンと同じユーザ名であれば以下の様にする。
USER=$(whoami)
ssh ${USER}@${ADDR}
```

パスワード、もしくは公開鍵認証でログイン出来れば成功です。例えばfreeコマンドなどでメモリを見てみると、1GBしかメモリが割り振られていないことが分かります。

```
pudding@sample:~$ free -h
               total        used        free      shared  buff/cache   available
Mem:           957Mi       196Mi       206Mi       1.0Mi       554Mi       620Mi
Swap:          1.9Gi          0B       1.9Gi
pudding@sample:~$
```

`/proc/cpuinfo` を見るとQEMUの仮想CPUが見えており、実際のハードウェアとは異なっていることが分かります。

```
pudding@sample:~$ cat /proc/cpuinfo
processor       : 0
vendor_id       : GenuineIntel
cpu family      : 15
model           : 107
model name      : QEMU Virtual CPU version 2.5+
stepping        : 1
microcode       : 0x1
cpu MHz         : 2711.998
cache size      : 16384 KB
physical id     : 0
siblings        : 1
core id         : 0
cpu cores       : 1
apicid          : 0
initial apicid  : 0
fpu             : yes
fpu_exception   : yes
cpuid level     : 13
wp              : yes
flags           : fpu de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 syscall nx lm constant_tsc nopl xtopology cpuid tsc_known_freq pni ssse3 cx16 sse4_1 sse4_2 x2apic popcnt aes hypervisor lahf_lm cpuid_fault pti
bugs            : cpu_meltdown spectre_v1 spectre_v2 spec_store_bypass l1tf mds swapgs itlb_multihit mmio_unknown
bogomips        : 5423.99
clflush size    : 64
cache_alignment : 128
address sizes   : 40 bits physical, 48 bits virtual
power management:

pudding@sample:~$
```

Linuxカーネルのバージョンを見ると、Proxmoxのホストとは異なるバージョンになっていることが分かるはずです。

```
pudding@sample:~$ uname -r
5.15.0-91-generic
pudding@sample:~$
```

今回はUbuntuを用いたのでProxmox本体がベースにしているDebianと比較的近いOSを入れましたが、KVMではより様々なOSをサポートしており、好みのOSを入れることが可能です。
このように、ホストから独立した環境にリソースを割り当て、ホストとLinuxカーネルやOSの異なる仮想マシンを実行できることが分かりました。

## VMを削除する

毎回このような苦労する方法は使いたくありません。次節以降でもっと楽してVMを作れるようにしていきます。今回作ったVMはもう不要ですので削除します。
削除する前にシャットダウンする必要があります。 `Datacenter` > `node1` > `100 (node1)` を選択し、右上あたりに表示される `Shutdown` ボタンを押してVMを止めます。
停止が完了すると、以下の様に `Status` が `stopped` になります。

![proxmox-webui-sample-vm-summary-stopped.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-summary-stopped.png)

:::message
ProxmoxのUI経由での操作は基本的に非同期で実行されます。そのため、何らかの操作をしたとき、即座に結果が反映されるとは限りません。そのため、 `Shutdown` を押しても即座には `stopped` にならない場合があります。しばらく待ってみてください。
:::

停止が完了したら右上の `More` を選び `Remove` をクリックします。下記の様な確認画面が出るので、VM IDの入力および2つのチェックをマークします。

![proxmox-webui-sample-vm-delete.png](/images/books/introduction-for-high-availability/proxmox-webui-sample-vm-delete.png)

しばらく待ってUIから `100 (node1)` が消えたら成功です。
