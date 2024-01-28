---
title: "仮想サーバ基盤を構築する"
---

## 物理マシンでサービスを提供する

物理マシンにOSをインストールし、Webサーバを入れ、サービスを提供することももちろん可能です。余計なオーバーヘッドもなく、パフォーマンスも良いかも知れません。
一方で、この構成はメンテナンスが難しくなることもあります。

- 1つのOS内に様々なものがインストールされていると、アップデート時や再起動時に多くの考慮が必要となる
  - あるライブラリやランタイムの更新時にそれによって影響を受ける範囲をしっかり考慮しなければならない
  - 同じPythonを使うソフトウェアでも、それらが要求するバージョンなどは異なるかもしれない
- 物理マシンは入れ替えにかかるコストが重い
  - 物理マシンのOS更新などを行うと必ず再起動が発生するため、その上で動いているプロセスの処理をまるごと他の物理マシンに移さなければならない。
    - 日頃から構築手順をちゃんと整えておく必要がある[^immutable-infra]
  - 何らかの事情で起動しなかった場合などのリカバリの操作も容易ではない

[^immutable-infra]: 実はこれが確実にできており、かつ物理マシンが潤沢に存在すれば、いわゆるイミュータブル・インフラストラクチャというプラクティスを実践できます。デプロイされたリソースはイミュータブル、つまり不変であり、稼働中のマシンにパッチを当てたりはせず必ず初期状態から構築して古いものと入れ替えるといった操作をします。これはいわゆる[Configuration drift](https://www.aquasec.com/cloud-native-academy/vulnerability-management/configuration-drift/)などの問題を抑えることができ、本番環境の状態をコントローラブルにするための重要な概念の1つです。物理マシンに対してもこういったプラクティスを適用できることがベストですが、出来ているところはあまり多くないでしょう。

とはいえ物理マシンは大抵高いスペックを持つため一台あたりが高価であり、依存関係のためだけにわざわざ複数台の物理マシンを用意するのは現実的とは言えません。この高スペックを活かし、かつコンポーネント間を疎結合にして管理しやすくする方法は無いものでしょうか。
そのために必要な概念としてコンピューティングリソースの抽象化について考えて見ます。

### コンピューティングリソースの割り振りとコンポーネントの分離

まずコンピューティングリソースとは、CPUやメモリ・ディスクの容量など物理マシンがサービス提供に供与可能なリソース量を指します。ここに4コア、32GBメモリ、256GBのディスク容量を持つサーバが3台あるとき、利用可能なコンピューティングリソースとしては12コア、96GBメモリ、768GBのディスクがある事になります。ただしこの合計リソースが単一のプロセスで直接利用可能なわけでは無いことには注意が必要です。

物理マシンが単なるコンピューティングリソースを供与するものであると考えると、反対にサービスを提供するプロセスはコンピューティングリソースを消費するものに当たります。プロセスがどれくらいリソースを消費するかを考慮した上でそれが実現できる物理マシンに適切に配置することが重要です。ここでは複数あるサーバ1つ1つは単なるコンピューティングリソースの塊であって、どのプロセスがどこに配置されるかは単にコンピューティングリソースの都合によって決まります。

ただし単にプロセスを配置すると言っても、やり方を考えなければ前述したように1つの物理マシンの中で複雑に絡まり合い、メンテナンスを困難にしてしまいます。そこで、仮想的な「箱」を用意し、その「箱」に対してリソースを割り当てることを考えます。この「箱」は他の「箱」や「箱」を収容するホストに直接干渉できず、それらが利用可能なリソース以上にリソースを利用できないようになっているとします。

この状態では、各「箱」ごとにメンテナンスが可能になります。またコンピューティングリソースの割り振りを変えることで、各「箱」の使えるリソース量の変更もできます。
この1つ1つの「箱」を **仮想マシン（VM）** と呼びます。Linuxには仮想マシンを作るための機能があり、適切に利用することでホストのコンピューティングリソースを仮想マシンごとにそれぞれ割り振ることができるようになっています[^why-no-container]。

[^why-no-container]: 現代にはコンテナと呼ばれる仮想化技術も存在します。これらはLinuxのcgroup（リソースの割り振り・制限ができる）やnamespace（プロセスごとに見える空間の分離）などを使って、この「箱」のようなものを作ることが出来るようになっています。ただし、Linuxカーネル自体はホストのものを共有するような作りになっているものが多く使われており、完全に分離されているわけではありません。VMの場合はホストとは異なるバージョンのカーネルを使うこともできます。今回の範囲ではコンテナは一旦取り扱いませんが、同様の構成をコンテナで行う場合何に気をつけなければならないか、逆に何かメリットはあるかなど考えてみるのも面白いでしょう。

## 仮想マシン（VM）とは

VM（Virtual Machineの略）とは「実際のコンピューターのように動作するコンピューター ファイル」[^ms-vm-def]です。これらは物理マシンのようにCPU・メモリ・ディスクなどを持ち、ネットワーク経由で通信できますが、その実体は物理マシンのリソースを抽象化したものを操っており物理的に目に見えるものではありません。

[^ms-vm-def]: <https://azure.microsoft.com/ja-jp/resources/cloud-computing-dictionary/what-is-a-virtual-machine>

VMはCPUやメモリ・ディスク・NICのようなハードウェアをエミュレートし、同様のふるまいをするソフトウェアやハードウェアの支援機能を活用しています。これによって実際のハードウェアと密結合した状態から一部解放され、異なるハードウェア間での移動がある程度容易になっています。

このような操作を実現するために有償の物からOSSまで様々な仮想化ソフトウェアが存在しており、ライブラリも整備されているのでユーザはそれらを活用することでボタンを押したら仮想マシンが立ち上がる、みたいな機能も実装可能です。[AWS EC2](https://aws.amazon.com/jp/ec2/)や[Google Compute Engine](https://cloud.google.com/compute)などでインスタンスを立ち上げるのと似たような機能を個人でも実装できます。

物理マシンと同じように振る舞うので、通常の物理マシンと同じようにしてOSをインストールして使うことが出来ます。仮想的なディスクドライブとしてISOファイルをマウントし、与えられた仮想的なディスクのパーティションを分割してOSをインストールし、そのディスクから起動して……といった感じです。当然グラフィックデバイスも仮想化でき、VMの画面をリモートから見ることができるような技術も存在します。

### なぜVMを用いるのか

ケースに応じて多様な事情が存在します。使わないのが悪いと言うことでは無くケースバイケースであり、以下の様な例に該当する場合は採用してみると良いでしょう。今回の演習では、こういうやり方もできますよという一例を示すため、またユーザごとの環境をできる限り均質化するため、複数台の物理マシンの代わりにVMを利用します。

- アプリケーションやミドルウェアの実行環境を分離するため
  - それぞれを個別に更新でき、互いの変更がOSやライブラリのレベルでは影響しない
- 管理するチームが異なっておりそれぞれに適切な権限を付与したい
  - 複数のアプリケーションをそれぞれ異なるチームが運用する際、全員が全てのアプリケーションへの管理者権限を持つというのは過剰である
  - 異なるVMに分けてそれぞれへの権限を適切なチームが持つことで、事故を防止しセキュリティを向上できる
- 物理マシンは柔軟な管理が難しく管理のコストが大きいため
  - VMの起動コストよりも物理マシンの起動コストの方が遥かに大きい
  - 物理マシンを状態そっくりそのまま入れ替えることは難しいが、VMならば状態を保存したまま物理マシン間を移動することも可能[^vm-migration]

[^vm-migration]: これはマイグレーションと呼ばれる操作ですが、今回は取り扱いません。VMをシャットダウンしてから移動させる方法や、VMを起動したまま移動させるライブマイグレーションと呼ばれるものまであります。メンテナンスしたいサーバからVMを適切に移動させることで、サービス断や再セットアップの手間を抑えて効率的にメンテナンスできます。

### 様々な仮想化製品

これらが厳密にどのような機能を提供する仮想化ソフトウェアなのかは詳しく解説しません。おそらく名前だけは聞いたことあるみたいなこれらのソフトウェアは、VMを作成するといった仮想化の機能を提供しているソフトウェアであると覚えてもらえれば良いかなと考えています。

- [Windows Hyper-V](https://learn.microsoft.com/ja-jp/virtualization/hyper-v-on-windows/about/)
- [Linux KVM](https://linux-kvm.org/page/Main_Page)
- [VirtualBox](https://www.virtualbox.org/)
- [VMware ESXi](https://www.vmware.com/jp/products/esxi-and-esx.html)
- [Nutanix AHV](https://www.nutanix.com/jp/products/ahv)

今回は主にKVM（とQEMU）を用いたVMを提供する、[Proxmox VE](https://pve.proxmox.com/pve-docs/)というOSSを用いてVM基盤を構築していきます。まずは中身については触れず、実際にVMを作って遊べるようにします。

## Proxmox VEのインストール

:::message
Proxmox VEのバージョン8.1.3を用いて動作を検証しています。他のバージョンではUIや機能などが異なる可能性があります。
異なるバージョンで挑戦する場合、また詳細なインストール方法については下記のページを参照してください。
<https://pve.proxmox.com/pve-docs/chapter-pve-installation.html>
:::

### インストール用USBメモリを作成する

Proxmox VEのISOをダウンロードし、起動可能な形式でUSBメモリに書き込みます。このUSBメモリをサーバとして利用するマシンに挿して起動することでインストールを開始できます。
以下のページから最新のProxmox VEのISOファイルをダウンロードしてください。
<https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso>

ダウンロードしたISOファイルを各プラットフォームごとに異なる方法を使ってUSBメモリに書き込みます。

:::message alert
ここで用いるUSBメモリに保存されているデータは全て削除されます。重要なデータを失わないよう、中身を確認した上で利用してください。
誤って重要なデータが削除されてしまった場合でも、筆者は一切の責任を負いかねます。
<!-- textlint-disable ja-technical-writing/ja-no-mixed-period -->
:::

:::details Windowsの場合
<!-- textlint-enable ja-technical-writing/ja-no-mixed-period -->
[Rufas](https://github.com/pbatard/rufus)を使います。[最新のリリース](https://github.com/pbatard/rufus/releases)からexeファイルをダウンロードして実行してください。

![rufas-window.png](/images/books/introduction-for-high-availability/rufas-window.png)

各項目をそれぞれ設定し、スタートを押してください。

- デバイス：あなたが書き込みたいUSBメモリ
- ブートの種類：「選択」からダウンロードしたISOファイルを指定する

ISOファイルを指定する際に下記の様な警告が表示されるかもしれません。その場合はOKを押して進んでください。

![rufas-warn-window.png](/images/books/introduction-for-high-availability/rufas-warn-window.png)

スタートをクリックした後、準備完了になれば正しく書き込めています。

![rufas-complete-window.png](/images/books/introduction-for-high-availability/rufas-complete-window.png)
<!-- textlint-disable ja-technical-writing/ja-no-mixed-period -->
:::

:::details macOSの場合
<!-- textlint-enable ja-technical-writing/ja-no-mixed-period -->
以下のページを参照してください。
<https://pve.proxmox.com/pve-docs/chapter-pve-installation.html#_instructions_for_macos>
<!-- textlint-disable ja-technical-writing/ja-no-mixed-period -->
:::

:::details Linuxの場合
<!-- textlint-enable ja-technical-writing/ja-no-mixed-period -->
以下のページを参照してください。
<https://pve.proxmox.com/pve-docs/chapter-pve-installation.html#_instructions_for_gnu_linux>
:::

### サーバマシンの設定をする

ここはハードウェアによって異なるため、お使いのハードウェアに合わせて読んでください。  

サーバとして使うPCにディスプレイ、キーボード、電源、LANケーブルを接続し、電源を投入して `F2`（機種によっては `Del`）を連打しているとUEFI（もしくはBIOS）の設定画面に入ることが出来ます。
設定すべきポイントは2つです。

- ブート順でUSBデバイスが優先されるようにする
  - UEFI（またはBIOS）には、どのデバイスから起動するかを選択する機能がある。
  - 既にLinuxなど他のOSがディスクにインストールされている場合、USBデバイスから起動されるように設定しないと、正しくインストーラが起動できない。
- CPUの仮想化支援機能を有効にする
  - CPUに搭載される仮想化の支援機能を有効化するとパフォーマンスが向上する
  - Intel製のCPUの場合：Intel VTやVirtualization Technologyという項目を探して有効化する
  - AMD製のCPUの場合：SVM ModeやAMD-Vという機能を探して有効化する

完了したら作成済みのUSBメモリを挿し、再起動します。

### （可能であれば）ネットワークの設定を確認しておく

<!-- textlint-disable ja-technical-writing/no-doubled-joshi -->
利用しているルータの設定を確認し、DHCPで配られるIPアドレスの範囲を確認しておきます。DHCPとは、Dynamic Host Configuration Protocolの略で、「IPv4ネットワークにおいて通信用の基本的な設定を自動的に行うためのプロトコル」[^dhcp]です。これによりLANケーブルを接続するだけでIPアドレスやゲートウェイ、用いるDNSサーバなどが自動設定されてインターネットに接続出来るようになります。
<!-- textlint-enable ja-technical-writing/no-doubled-joshi -->

[^dhcp]: <https://www.nic.ad.jp/ja/basics/terms/dhcp.html>

例えば `192.168.16.1`～`192.168.16.100` までが配られるアドレスの範囲になっているとします。これらのアドレスは永久に特定の端末にのみ与えるわけではなく、一定期間に決まっています。そのアドレスを使用していない間にこの期間を満了すると、そのアドレスは開放されます。本書での実験をする端末をシャットダウンしている間などにこの期間を迎えてしまうと、接続先のサーバのアドレスが変わってしまう可能性があります。

多くの場合、DHCPで配られるIPアドレスは利用可能なアドレスの全てではありません。例えば `192.168.16.0/24` のネットワークでは、256個のIPアドレスが利用可能です。つまり `192.168.16.1` ～ `192.168.16.254` が使えます。ここでDHCP範囲内のIPアドレスのみが通信できるというわけではなく、単に自動で割り振られないだけです。手動で割り振って使うことは可能なので、それを設定することで家庭内でも固定のIPアドレスが利用可能です。ただし、同じIPアドレスを複数の端末で同時に設定するとうまく通信できません。

`ping`コマンドなどで軽くそのIPアドレスが使われていないか確認しておきましょう。応答が返ってくるIPアドレスは何らかのデバイスが利用中です。

```sh
# 192.168.16.150が使われているか知りたい場合
ping -c 3 192.168.16.150
```

### インストールする

正しく起動できれば以下の様な画面が表示されます。

![proxmox-installer-1.png](/images/books/introduction-for-high-availability/proxmox-installer-1.png)

`Install Proxmox VE (Graphical)`を選択して進めていきます。
一瞬コンソールの黒い画面に白文字が流れますが、しばらく待つとEULAの同意画面に入るので、同意して進めていきます。マウスがあればマウスも利用可能ですが、キーボードのTabでフォーカスを移動し、スペースでボタンを押すことが出来ます。

:::message
EULAの前に下記の様な画面が表示される場合、UEFI（またはBIOS）においてCPUの仮想化支援機能が有効化されていません。前セクションの設定をやり直してください。
![proxmox-installer-warn.png](/images/books/introduction-for-high-availability/proxmox-installer-warn.png)
:::

以下の様な画面まで進めたらインストールする対象のディスクを選択します。

:::message alert
この操作によりそのPCのディスクの内容が完全に消去されます。消去されて問題ないPCを選択してインストールしてください。
誤って重要なデータが削除されてしまった場合でも、筆者は一切の責任を負いかねます。
<!-- textlint-disable ja-technical-writing/ja-no-mixed-period -->
:::
<!-- textlint-enable ja-technical-writing/ja-no-mixed-period -->

![proxmox-installer-2.png](/images/books/introduction-for-high-availability/proxmox-installer-2.png)

次の画面ではタイムゾーンやキーボードのレイアウトを指定します。デフォルトはおそらくJapaneseとかが選ばれていますが、人によってはUSレイアウトを設定した方がよいかもしれません。

![proxmox-installer-3.png](/images/books/introduction-for-high-availability/proxmox-installer-3.png)

適当なパスワードを設定します。最終的にこのパスワードを使ってWeb UIからログインするので、パスワードは最低でも8文字以上のランダムな文字列を推奨します。

![proxmox-installer-4.png](/images/books/introduction-for-high-availability/proxmox-installer-4.png)

最後にネットワークの設定をします。ここが一番重要です。

<!-- textlint-disable ja-technical-writing/no-mix-dearu-desumasu -->
- `Management Interface`
  - もし複数のNICを搭載している場合は、Proxmoxの管理用に使いたいものを選択してください。
- `Hostname (FQDN)`
  - 適当なホスト名を設定します。サーバのホスト名は人によって流儀がありますが、迷ったら星の名前にしておけば良いです。
- `IP Address (CIDR)`
  - 前のセクションでIPアドレスを把握している場合、DHCPの範囲外の適当なIPアドレスを設定します。分からなかった場合はそのままで大丈夫です。
<!-- textlint-enable ja-technical-writing/no-mix-dearu-desumasu -->

他はデフォルトのままで問題ありません。

![proxmox-installer-5.png](/images/books/introduction-for-high-availability/proxmox-installer-5.png)

最後に確認画面が出るので問題があれば `Previous` ボタンから戻って再設定してください。`Install` を押すと開始します。  
完了するまで待ち、rebootされたらUSBメモリは抜いて大丈夫です。もしうっかりUSBからまた起動してしまった場合は、電源を落とし、USBメモリを抜いてから起動すれば問題ありません。

```text
Welcome to the Proxmox Virtual Environment. Please use your browser to
Configure this server - connect to:

  https://YOUR_IP_ADDRESS:8006/
```

上記のような表示が出たら成功です。クライアント端末のブラウザで上記アドレスにアクセスしてみてください。
アクセス出来ない場合はネットワークの設定が間違っていた可能性があります。インストールをやり直してみてください。

:::message
接続するプロトコルがHTTPではなく HTTPS であることに注意してください。これは[自己署名証明書](https://www.gmo.jp/security/ciphersecurity/http-https/blog/self-signed-certificate/)を使って提供されているため、ブラウザからアクセスするときに下記の様なエラーが表示されます。

![proxmox-webui-warning.png](/images/books/introduction-for-high-availability/proxmox-webui-warning.png)

Firefoxの場合、`詳細へ進む`→`危険姓を承知で続行` でアクセスできます。今回はローカルでの実験的なサービスであることから、許容してアクセスしてください。
なお、通常このような警告が出るサイトにアクセスするべきではありません。
:::

## ProxmoxのWeb UI

本書では、Proxmox VEをインストールした物理マシンのIPアドレスを `192.168.16.130`、ホスト名を `node1` として説明します。以降、このIPアドレスが使われた場合、自身のProxmox VEをインストールした物理マシンと読み替えてください。

![proxmox-webui-login.png](/images/books/introduction-for-high-availability/proxmox-webui-login.png)

Web UIへのログインには、インストール時に設定したパスワードを利用します。

- User name：`root`
- Password：インストール時に設定したパスワード
- Realm：`Linux PATM standard authentication`
- Language：`English - English`

なお、説明上ではLanguageを `English` で進めますが、ご自身の判断で日本語にしていただいても構いません。

## Proxmoxの初期設定

ログイン完了時に下記の様な警告が表示されると思います。これはProxmoxが提供する有償サブスクリプションを契約していないために表示されています。この警告は初期設定後も出続けますが、特に本書での利用範囲において大きな影響はありません。

![proxmox-webui-subscription-warning.png](/images/books/introduction-for-high-availability/proxmox-webui-subscription-warning.png)

:::message
Proxmox VEを使って商用サービスを提供したい場合、迅速なセキュリティアップデートや技術サポートを受けるため、有償サブスクリプションを契約することをおすすめします。
:::

### 無償版リポジトリへ切り替える

Proxmox VEの商用サポートが提供される有償版パッケージリポジトリ（Enterpriseリポジトリ）がデフォルトで指定されています。この状態ではパッケージの更新などに失敗するので、まずはEnterpriseリポジトリを見ないように設定を変えます。
画面左のカラムから`Datacenter` > `node1`を選び、画面右の設定一覧から `Updates` > `Repositories` を選択します。

![proxmox-webui-repos-enterprise-enabled.png](/images/books/introduction-for-high-availability/proxmox-webui-repos-enterprise-enabled.png)

`/etc/apt/sources.list.d/ceph.list` および `/etc/apt/sources.list.d/pve-enterprise.list` に表示されている行を選択し、 `Disable` します。

![proxmox-webui-repos-no-pve.png](/images/books/introduction-for-high-availability/proxmox-webui-repos-no-pve.png)

次に `Add` をクリックし、（警告画面をOKでスキップして）以下の様に `No-Subscription` リポジトリを選択して `Add` します。

![proxmox-webui-repos-add-no-sub.png](/images/books/introduction-for-high-availability/proxmox-webui-repos-add-no-sub.png)

以下の様に追加されれば完了です。

![proxmox-webui-repos-no-sub-enabled.png](/images/books/introduction-for-high-availability/proxmox-webui-repos-no-sub-enabled.png)

### （Optional）公開鍵認証でログインできるようにする

まずはssh時に公開鍵認証が使えるように公開鍵を設定します。GitHubに登録している鍵を流用する場合、このセクションはスキップして次のセクションで公開鍵を設定しても構いません。

もしまだ手元の環境でSSHの鍵を一度も作っていない場合は `ssh-keygen` コマンドで鍵を生成します。不要な場合はスキップして構いません。

```sh
ssh-keygen -t ed25519
```

お使いのターミナルエミュレータを開いて、以下の様なコマンドを打ちます。

```sh
# サーバのアドレスは各自の環境に応じて変える
PROXMOX_SERVER_ADDR="192.168.16.130"
ssh-copy-id -n root@${PROXMOX_SERVER_ADDR}
```

これはドライランになっていて、実際には公開鍵は設定されていません。表示された鍵の情報が自分の想定通りである場合、下記の様に `-n` を除いて実行し直します。

```sh
ssh-copy-id root@${PROXMOX_SERVER_ADDR}
```

パスワードを入力すると公開鍵が登録されます。

```text
$ ssh-copy-id root@${PROXMOX_SERVER_ADDR}
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/pudding/.ssh/id_ed25519.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@192.168.16.130's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@192.168.16.130'"
and check to make sure that only the key(s) you wanted were added
```

パスワードを入力することなくsshできることを確かめます。

```sh
ssh root@${PROXMOX_SERVER_ADDR}
```

エラーになる場合はなんらかの設定ミス、指定した鍵が異なるなどの影響が考えられます。対応する公開鍵を登録し直してみてください。

### （Optional）GitHubから公開鍵をインポートする

GitHubにSSHの鍵を登録している場合、その登録された公開鍵をそのまま取り込んで利用できます。
`Datacenter` > `node1` を選択し、 `Shell` を開くと以下の様にrootユーザでログインしたシェルが開かれます。簡単な作業はここから行うこともできます。

![proxmox-webui-shell.png](/images/books/introduction-for-high-availability/proxmox-webui-shell.png)

先に公開鍵のインポートに使われるコマンドをインストールします。

```sh
apt update
apt install -y ssh-import-id
```

次に自身のGitHubのユーザ名（筆者の場合 [pddg](https://github.com/pddg)）を指定してimportします。

```sh
GH_USER=pddg
ssh-import-id gh:${GH_USER}
```

手元の端末からはGitHubへSSHするときに使っている秘密鍵を指定するだけで `node1` へ接続出来ます。
