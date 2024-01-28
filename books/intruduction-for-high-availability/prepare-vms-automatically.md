---
title: "VMを自動セットアップする"
---

## クラウドイメージを使う

前章では、空の仮想環境に対してUbuntu Serverのインストールを手作業で行いました。この方法では多数の仮想環境を用意するのに非常に手間がかかります。そこで、クラウドイメージを使って自動的に仮想環境をセットアップする方法を紹介します。

クラウドイメージとは、主にクラウドサービスで利用される、OSのイメージファイルです。これは「インストール済み」の状態のOSイメージで、いくつかのフォーマットで規定されています[^vmimgfmt]。前章のようなインストーラを使ってインタラクティブに構築する場合と異なり、設定は [cloud-init](https://cloudinit.readthedocs.io/en/latest/) や [ignition](https://coreos.github.io/ignition/) などのツールを使って行います。これらはOSの初回起動時に実行され、事前に定義した設定が注入されます。

[^vmimgfmt]: <https://docs.openstack.org/glance/latest/user/formats.html>

今回利用するUbuntuもクラウドイメージを公開しています。これはその時点での最新のアップデートを適用したイメージが、数日おきに公開されています。最新のイメージに追従することで、構築したVMの起動後に必要なアップデートの量が少なくなるため、大規模な仮想サーバ基盤を構築したい場合はその方法も検討すると良いでしょう。

<https://cloud-images.ubuntu.com/>

### クラウドイメージのダウンロード

今回はUbuntu 22.04 LTSのクラウドイメージを利用します。現時点での最新のクラウドイメージは以下のURLからダウンロードできます。

<https://cloud-images.ubuntu.com/jammy/current/>

まずはProxmox VEからこのイメージを利用できるよう `node1` 上にダウンロードします。ファイルが複数存在していますが、利用するのはOpenStackでよく使われるqcow2というフォーマットのイメージです。
以降の手順はProxmox VEをインストールしたサーバ上で行います。

:::message
以降、実行するコマンドを記述する際はその先頭に `# @ホスト名` として実行するホストを表記します。他のホストで実行した場合、エラーになる可能性があります。実行する前によく確認してください。
なお本書においてProxmox VEのホスト名は `node1` として表記しています。
:::

```bash
# @node1
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
```

<!-- textlint-disable ja-technical-writing/ja-no-mixed-period-->
:::details ダウンロードしたイメージの真正性を確認する
<!-- textlint-enable ja-technical-writing/ja-no-mixed-period-->

このようにWebからダウンロードしたイメージが正しくUbuntuの開発チームが用意したものであるかを確認できます。基本的に実行方法は以下の通りです。対象がISOファイルではなく、クラウドイメージのファイルに変わっているだけです。

@[card](https://ubuntu.com/tutorials/how-to-verify-ubuntu#1-overview)

イメージをダウンロードした後、同じディレクトリで以下のコマンドを実行します。

```bash
# @node1
wget https://cloud-images.ubuntu.com/jammy/current/SHA256SUMS.gpg
wget https://cloud-images.ubuntu.com/jammy/current/SHA256SUMS

gpg --keyid-format long --verify SHA256SUMS.gpg SHA256SUMS
```

以下の様に `No public key` というエラーが表示される場合は、Canonicalの公開鍵をインポートする必要があります。

```text
root@node1:~# gpg --keyid-format long --verify SHA256SUMS.gpg SHA256SUMS
gpg: Signature made Fri 26 Jan 2024 03:30:42 PM JST
gpg:                using RSA key D2EB44626FDDC30B513D5BB71A5D6C4C7DB87C81
gpg: Can't check signature: No public key
```

この場合、`D2EB44626FDDC30B513D5BB71A5D6C4C7DB87C81` という鍵を使って署名されているようです。この鍵をインポートします。

```bash
# @node1
SIG=D2EB44626FDDC30B513D5BB71A5D6C4C7DB87C81
gpg --keyid-format long --keyserver keyserver.ubuntu.com --recv-keys $SIG
```

この例では `UEC Image Automatic Signing Key <cdimage@ubuntu.com>` がこの鍵を使っているようです。Ubuntuのkeyserverから入手していること、署名者のメールアドレスにUbuntuのドメインが含まれていることから、この鍵は信頼して良いと考えられそうです（実際どうやってこれが正しいことを証明するんだろう……）。
再び実行し、以下の様に成功するか確かめます。

```text
root@node1:~# gpg --keyid-format long --verify SHA256SUMS.gpg SHA256SUMS
gpg: Signature made Fri 26 Jan 2024 03:30:42 PM JST
gpg:                using RSA key D2EB44626FDDC30B513D5BB71A5D6C4C7DB87C81
gpg: Good signature from "UEC Image Automatic Signing Key <cdimage@ubuntu.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: D2EB 4462 6FDD C30B 513D  5BB7 1A5D 6C4C 7DB8 7C81
```

WARNINGが発生していますが、これはこの署名が無効であるなどの意味ではありません。
これで `SHA256SUMS` がUbuntuチームの秘密鍵で署名されていることが確認できました。この `SHA256SUMS` に記載されているハッシュとダウンロードしたイメージのハッシュが一致するか確認します。

```bash
# @node1
sha256sum -c SHA256SUMS --ignore-missing
```

`jammy-server-cloudimg-amd64.img` というファイルについて `OK` と表示されれば、ダウンロードしたイメージは正しくCanonicalが用意したものであることを確認できます。

```bash
root@node1:~# sha256sum -c SHA256SUMS --ignore-missing
jammy-server-cloudimg-amd64.img: OK
```

:::

### クラウドイメージのカスタマイズ

ベースイメージのカスタマイズを行います。これは `virt-customize` というツールを使って行います。このツールを使うことで、仮想マシンのディスクイメージをマウントしてファイルを書き換えたり、パッケージをインストールしたりできます。まずは `virt-customize` をインストールします。

:::message
ここではrootユーザとしてログインしている前提で説明します。rootユーザでない場合は `sudo` を使って実行してください。
:::

```bash
# @node1
apt update && apt install -y --no-install-recommends libguestfs-tools
```

様々なカスタマイズができますが、ここではパッケージを追加することにします。前章では手動でインストールした `qemu-guest-agent` が最初からインストールされた状態で起動するよう、ベースイメージに追加します。

```bash
# @node1
virt-customize -a jammy-server-cloudimg-amd64.img \
  --install qemu-guest-agent
```

他にもファイルを作成/更新/削除したり、ユーザやそのパスワードの設定も可能です。今回はこれ以上のカスタマイズは行いません。詳細は以下のページのオプションなどを参照してください。

<https://libguestfs.org/virt-customize.1.html>

### クラウドイメージを使った仮想環境の作成

Proxmox VEでは `qm` コマンドを使って仮想環境を作成します。まずは仮想マシンのIDを決めます。ここでは `200` とします。

```bash
# @node1
VMID=200
# 仮想マシンを作成する
qm create $VMID \
    --name cloud-image-test \
    --cores 2 \
    --memory 1024 \
    --serial0 socket \
    --vga serial0 \
    --net0 virtio,bridge=vmbr0 \
    --agent enabled=1

# 仮想マシンのディスクとして、先ほどカスタマイズしたイメージを使う
qm importdisk $VMID jammy-server-cloudimg-amd64.img local-lvm

# インポートしたイメージをルートボリュームとし、そこからブートするようにする
qm set $VMID \
    --scsihw virtio-scsi-pci \
    --scsi0 local-lvm:vm-$VMID-disk-0 \
    --boot c \
    --bootdisk scsi0

# デフォルトではディスクサイズが小さいので20GBに拡張する
qm resize $VMID scsi0 20G
```

作成したVMをcloud-initで設定できる様にします。

:::message
Proxmox VEでcloud-initを完全に利用するには、ユーザが書いたcloud-initの設定ファイルを使う必要があります。これを使うと、GUIからのカスタマイズができなくなるため、ここでは `qm` コマンドがサポートする範囲のみで設定します。詳細は以下のページを参照してください。

<https://pve.proxmox.com/wiki/Cloud-Init_Support>
:::

```bash
# @node1
# cloud-init でカスタマイズできるようにする
qm set $VMID --ide2 local-lvm:cloudinit

# DHCPでIPアドレスを取得するように設定する
qm set $VMID --ipconfig0 ip=dhcp

# ユーザ名を設定する。ここでは `ubuntu` とする。
qm set $VMID --ciuser ubuntu

# SSHにつかう公開鍵を設定する
# GitHubから公開鍵を取得する場合
# GH_USER=
# PUBKEY_FILE=${GH_USER}.keys
# wget -O ${PUBKEY_FILE} https://github.com/${GH_USER}.keys

# 手元の公開鍵を使う場合
PUBKEY_FILE=~/.ssh/id_ed25519.pub
qm set $VMID --sshkeys $PUBKEY_FILE
```

これで準備が整いました。あとはVMを起動するだけです。

```bash
# @node1
# VMを起動する
qm start $VMID
```

VMが起動したら、SSHでログインしてみます。
正しくIPアドレスが取得できていれば、以下のコマンドでIPアドレスが表示されます。

:::message
`pvesh` コマンドはProxmox VEのAPIを叩くことができるコマンドです。Proxmox VEのホストにデフォルトでインストールされています。
`jq` コマンドがインストールされていない場合は `apt install -y jq` でインストールしてください。
:::

```bash
# @node1
PROXMOX_HOST=node1
pvesh get /nodes/${PROXMOX_HOST}/qemu/${VMID}/agent/network-get-interfaces -o json \
    | jq '.result[] | select(.name=="eth0") | .["ip-addresses"][] | select(.["ip-address-type"] == "ipv4") | .["ip-address"]'
```

手元の端末（`client`）からログインしてみます。

```bash
# @client
ssh ubuntu@<IPアドレス>
```

以下の様に `cloud-image-test` というホスト名が表示されれば成功です。

```text
ubuntu@cloud-image-test:~$ hostname
cloud-image-test
ubuntu@cloud-image-test:~$
```

クラウドイメージをベースとしてVMを起動できることが分かったので、以下の様に起動したVMを停止・削除します。

```bash
# @node1
qm stop $VMID && qm wait $VMID && qm destroy $VMID
```

このホストへSSHした際、クライアントマシンのknown_hostsに登録されています。VMを削除した後に同じアドレスを別のVMに割り当てると、SSHした際に `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` というエラーが表示されます。これを避けるため、VMを削除したら以下の様にknown_hostsから削除してください。

```bash
# @client
ssh-keygen -R <IPアドレス>
```

## Proxmox VEのテンプレート機能を使う

これでもかなり楽にVMをセットアップできましたが、Proxmox上にテンプレートを用意するとより簡単にできます。一度テンプレートを作成すると、以降はその設定をクローンするだけで利用できます。

### テンプレート用のVMを作成する

まずはテンプレート用のVMを作成します。ここでは `999` というIDのVMを作成します。このIDにした理由は特にありません。ただし100未満のIDはエラーになるようです。

```bash
# @node1
VMID=999
qm create $VMID \
    --name vm-master-image \
    --cores 2 \
    --memory 1024 \
    --serial0 socket \
    --vga serial0 \
    --net0 virtio,bridge=vmbr0 \
    --agent enabled=1
qm importdisk $VMID jammy-server-cloudimg-amd64.img local-lvm

# SSHにつかう公開鍵を設定する
# GitHubから公開鍵を取得する場合
# GH_USER=
# PUBKEY_FILE=${GH_USER}.keys
# wget -O ${PUBKEY_FILE} https://github.com/${GH_USER}.keys

# 手元の公開鍵を使う場合
PUBKEY_FILE=~/.ssh/id_ed25519.pub
qm set $VMID \
    --scsihw virtio-scsi-pci \
    --scsi0 local-lvm:vm-$VMID-disk-0 \
    --boot c \
    --bootdisk scsi0 \
    --ide2 local-lvm:cloudinit \
    --ipconfig0 ip=dhcp \
    --ciuser ubuntu \
    --sshkeys $PUBKEY_FILE

qm resize $VMID scsi0 20G
```

### テンプレートを作成する

作成したVMをテンプレート化します。

```bash
# @node1
qm template $VMID
```

テンプレート化したVMは起動できなくなります。

```text
root@node1:~# qm start $VMID
you can't start a vm if it's a template
root@node1:~#
```

### テンプレートをクローンしてVMを作成する

テンプレート化したVMをクローンしてVMを作成します。ここでは `300` というIDのVMを作成します。

```bash
# @node1
TEMPLATE_VMID=999
VMID=300
qm clone $TEMPLATE_VMID $VMID \
    --name vm-from-template \
    --full
```

`--full` オプションを指定することで、テンプレート化したVMのディスクイメージと同じ容量を消費して新しいVMを作成します。巨大なボリュームを持つVMを作成する場合は注意してください。

### クローンしたVMを起動する

クローンしたVMを起動します。

```bash
# @node1
qm start $VMID
```

しばらく待って、IPアドレスを確かめます。

```bash
# @node1
PROXMOX_HOST=node1
pvesh get /nodes/${PROXMOX_HOST}/qemu/${VMID}/agent/network-get-interfaces -o json \
    | jq '.result[] | select(.name=="eth0") | .["ip-addresses"][] | select(.["ip-address-type"] == "ipv4") | .["ip-address"]'
```

手元の端末（`client`）からログインしてみます。

```bash
# @client
ssh ubuntu@<IPアドレス>
```

以下の様に `vm-from-template` というホスト名が表示されれば成功です。

```text
ubuntu@vm-from-template:~$ hostname
vm-from-template
ubuntu@vm-from-template:~$
```

クローンしたVMを停止・削除します。

```bash
# @node1
qm stop $VMID && qm wait $VMID && qm destroy $VMID

# @client
ssh-keygen -R <IPアドレス>
```

### 静的なIPアドレスを設定して起動する

ここまでの手順では、簡単のためVMはDHCPでIPアドレスを取得するように設定していました。このような古典的な環境でサービスを構築する際には、静的なIPアドレスを設定することが望ましいです。
cloud-initを使って静的なアドレスを割り振るようにします。ここでは `192.168.16.131` をVMに割り振ります。

:::message alert
`192.168.16.131` というIPアドレスは、筆者の環境に合わせて設定しています。自身の環境に合わせて変更してください。
:::

```bash
# @node1
TEMPLATE_VMID=999
VMID=301
qm clone $TEMPLATE_VMID $VMID \
    --name vm-with-static-ip \
    --full
qm set $VMID --ipconfig0 "ip=192.168.16.131/32,gw=192.168.16.1"
```

VMを起動して、IPアドレスが割り振られていることを確認します。

```bash
# @node1
qm start $VMID
```

```bash
# @client
ssh ubuntu@192.168.16.131
```

確認できたらVMを停止・削除して後片付けしておきましょう。

```bash
# @node1
qm stop $VMID && qm wait $VMID && qm destroy $VMID

# @client
ssh-keygen -R "192.168.16.131"
```

同様に `qm set` コマンドを使って、CPUやメモリの量を変更したり、ネットワークの設定を変更したりできます。詳細は以下のページを参照してください。

<https://pve.proxmox.com/pve-docs/qm.1.html>

## （おまけ）GUIからVMを作成する

テンプレートをクローンしてGUIからVMを作成する方法も紹介しておきます。
まず、画面左のカラムから `Datacenter` > `node1` > `999 (vm-master-image)` を選択します。右上の `More` ドロップダウンを開くと `Clone` という項目があるので、これを選択します。

![proxmox-webui-clone-vm-template.png](/images/books/introduction-for-high-availability/proxmox-webui-clone-vm-template.png)

ここでは `100` というVMIDを持つ、 `vm-from-gui` という名前のVMを作成します。`Mode` を `Full Clone` に変更し、 `Clone` ボタンを押します。

![proxmox-webui-customize-clone-vm.png](/images/books/introduction-for-high-availability/proxmox-webui-customize-clone-vm.png)

画面左のカラムから `Datacenter` > `node1` > `100 (vm-from-gui)` を選択します。
中央のカラムから `Cloud-Init` を選択すると、ユーザ名やネットワークなどいくつかの設定を変更できます。

![proxmox-webui-customize-clone-vm-cloud-init.png](/images/books/introduction-for-high-availability/proxmox-webui-customize-clone-vm-cloud-init.png)

中央のカラムから `Hardware` を選択すると、CPUやメモリの量を変更したり、ディスクを追加できます。

![proxmox-webui-customize-clone-vm-hardware.png](/images/books/introduction-for-high-availability/proxmox-webui-customize-clone-vm-hardware.png)

画面上部の `Start` ボタンを押すとVMが起動します。中央のカラムから `Summary` を選択すると、IPアドレスなどの情報を確認できます。
停止する際は `Shutdown` ボタンを押します。

シャットダウンが完了するまでVMは削除出来ません。停止が完了したら右上の `More` ドロップダウンから `Remove` を選択します。
削除するVMIDを入力し、チェックボックスにチェックを入れて `Remove` ボタンを押します。

![proxmox-webui-remove-vm.png](/images/books/introduction-for-high-availability/proxmox-webui-remove-vm.png)

## まとめ

Ubuntu 22.04 LTSのクラウドイメージを使ったVMテンプレートを作り、新規VMを以下の様なコマンドで作成できるようになりました。

```bash
# @node1
VMID=
VM_NAME=
TEMPLATE_VMID=999
qm clone $TEMPLATE_VMID $VMID \
    --name ${VM_NAME} \
    --full
# 必要であれば設定を変更する
# qm set $VMID \
#     --ipconfig0 "ip=192.168.16.131/32,gw=192.168.16.1"

qm start $VMID
```

次の章からはこのようなVMを使ってWebサービスを作っていきます。
