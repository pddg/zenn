---
title: "ロードバランサを冗長化する"
---

## ロードバランサの冗長化方法

ロードバランサを冗長化する方法はいくつかありますが、ここでは、[Keepalived](https://www.keepalived.org/)を使ってVirtual IPアドレスによるactive/standby構成を構築します。
active/standby構成とは、複数のロードバランサを用意し、そのうちの1つだけがトラフィックを受け持っており（active）他のロードバランサは待機している状態（standby）を指します。activeなロードバランサがダウンした場合、standbyなロードバランサがactiveに昇格してトラフィックを受け持つようになります。

### Keepalivedとは

KeepalivedはCで書かれたOSSであり、ロードバランシングと高可用性を提供するためのソフトウェアです。VRRP（Virtual Router Redundancy Protocol）を使って、複数のマシン間でVirtual IPアドレスを共有できます。また、前章で名前だけ出したLVS（Linux Virtual Server）はKeepalivedを用いたロードバランサで頻繁に使われます。

ここではKeepalivedは単にVRRPを使ってVirtual IPアドレスを共有するためのソフトウェアとして使います。Virtual IPアドレスとは、文字通り仮想的なIPアドレスで、特定の物理マシンが所有するものではなく、複数のマシンが共有するIPアドレスです。とはいえ、実際には共有している全てのマシンへのアクセスにそのIPアドレスを使用できるわけではなく、特定のマシンのみがこのIPアドレスを持ち、他のマシンは待機している状態になります。

### VRRPとは

これは[RFC5798](https://datatracker.ietf.org/doc/html/rfc5798)で定義されているプロトコルで、主にルータの冗長化で使われます。
複数のルータが互いにVRRPで通信し、その中で1つのルータがactiveになり、他のルータはstandbyになります。activeなルータがダウンした場合、standbyなルータの中で1つがactiveに昇格します。

いわゆる分散合意アルゴリズムと異なり、VRRPはsplit-brain[^splitbrain]耐性を持ちません。VRRPで互いに通信できなくなったサーバは、それぞれがactiveになろうとします。通信が回復した後には正常な状態に戻りますが、それまではどちらもactiveであると認識してIPアドレスを取得しようとします。そのため、例えばアプリケーションのロジックとしてリーダー選出が必要な場合などにVRRPを使うべきではありません。

![vppr-plit-brain.drawio.png](/images/books/introduction-for-high-availability/vrrp-split-brain.drawio.png)

この可能性を下げるため、VRRPで通信するマシン間のリンクを冗長化することが推奨されます。ただし、今回は省きます。実際にプロダクション環境で利用する場合、異なる電源に接続された異なるスイッチを介してマシン間を接続するなどの対策をした方が良いでしょう。

![vrrp-ha-switch.drawio.png](/images/books/introduction-for-high-availability/vrrp-ha-switch.drawio.png)

[^splitbrain]: クラスタなど複数のマシンが1つのシステムを構成しており、その各マシン間の通信が途絶えたときに、各マシンがそれぞれ独立してシステムを運用しようとする現象。ステートを持つクラスタではデータの不整合などの致命的な問題を引き起こすことがある。

## Keepalivedを使ってロードバランサを冗長化する

この手順では最終的に以下のような構成を目指します。

![l4lb-ha.drawio.png](/images/books/introduction-for-high-availability/l4lb-ha.drawio.png)

### l4lb-2 VMを作成する

まずはl4lb-2 VMを作成します。l4lb-1 VMと同じように作成しますが、IPアドレスは異なるものにします。

```bash
# @node1
TEMPLATE_VMID=999
GATEWAY="192.168.16.1"
L4LB2_VMID=501
L4LB2_NAME="l4lb-2"
L4LB2_IP="192.168.16.134"

qm clone ${TEMPLATE_VMID} ${L4LB2_VMID} --name ${L4LB2_NAME} --full
qm set ${L4LB2_VMID} --ipconfig0 "ip=${L4LB2_IP}/32,gw=${GATEWAY}"
qm start ${L4LB2_VMID}
```

l4lb-2 VMにSSHでログインし、HAProxyをインストールします。

```bash
# @l4lb-2
sudo apt update
sudo apt install -y --no-install-recommends haproxy
```

l4lb-1に設定したHAProxyの設定ファイルをl4lb-2にコピーします。

```bash
# @client
scp ubuntu@${L4LB1_IP}:/etc/haproxy/haproxy.cfg ubuntu@${L4LB2_IP}:/tmp
```

```bash
# @l4lb-2
sudo mv /tmp/haproxy.cfg /etc/haproxy
sudo systemctl restart haproxy
```

この状態でl4lb-2にアクセスすると、l4lb-1と同じようにHAProxyが動作していることが確認できます。

```bash
# @client
curl -s http://${L4LB2_IP}/hostname
curl -s http://${L4LB2_IP}/hostname
```

### Keepalivedをインストールする

l4lb-1およびl4lb-2にKeepalivedをインストールします。Ubuntuではaptを使ってインストールできます。

```bash
# @l4lb-1, l4lb-2
sudo apt update
sudo apt install -y --no-install-recommends keepalived
```

Ubuntu 22.04 LTSではv2.2.4がインストールされました。

```text
ubuntu@l4lb-1:~$ keepalived -v
Keepalived v2.2.4 (08/21,2021)

Copyright(C) 2001-2021 Alexandre Cassen, <acassen@gmail.com>

Built with kernel headers for Linux 5.15.27
Running on Linux 5.15.0-92-generic #102-Ubuntu SMP Wed Jan 10 09:33:48 UTC 2024
Distro: Ubuntu 22.04.3 LTS

configure options: --build=x86_64-linux-gnu --prefix=/usr --includedir=${prefix}/include --mandir=${prefix}/share/man --infodir=${prefix}/share/info --sysconfdir=/etc --localstatedir=/var --disable-option-checking --disable-silent-rules --libdir=${prefix}/lib/x86_64-linux-gnu --runstatedir=/run --disable-maintainer-mode --disable-dependency-tracking --enable-snmp --enable-sha1 --enable-snmp-rfcv2 --enable-snmp-rfcv3 --enable-dbus --enable-json --enable-bfd --enable-regex --with-init=systemd build_alias=x86_64-linux-gnu CFLAGS=-g -O2 -ffile-prefix-map=/build/keepalived-NeItXh/keepalived-2.2.4=. -flto=auto -ffat-lto-objects -flto=auto -ffat-lto-objects -fstack-protector-strong -Wformat -Werror=format-security LDFLAGS=-Wl,-Bsymbolic-functions -flto=auto -ffat-lto-objects -flto=auto -Wl,-z,relro CPPFLAGS=-Wdate-time -D_FORTIFY_SOURCE=2

Config options:  NFTABLES LVS REGEX VRRP VRRP_AUTH VRRP_VMAC JSON BFD OLD_CHKSUM_COMPAT SNMP_V3_FOR_V2 SNMP_VRRP SNMP_CHECKER SNMP_RFCV2 SNMP_RFCV3 DBUS INIT=systemd SYSTEMD_NOTIFY

System options:  VSYSLOG MEMFD_CREATE IPV4_DEVCONF LIBNL3 RTA_ENCAP RTA_EXPIRES RTA_NEWDST RTA_PREF FRA_SUPPRESS_PREFIXLEN FRA_SUPPRESS_IFGROUP FRA_TUN_ID RTAX_CC_ALGO RTAX_QUICKACK RTEXT_FILTER_SKIP_STATS FRA_L3MDEV FRA_UID_RANGE RTAX_FASTOPEN_NO_COOKIE RTA_VIA FRA_PROTOCOL FRA_IP_PROTO FRA_SPORT_RANGE FRA_DPORT_RANGE RTA_TTL_PROPAGATE IFA_FLAGS LWTUNNEL_ENCAP_MPLS LWTUNNEL_ENCAP_ILA NET_LINUX_IF_H_COLLISION LIBIPVS_NETLINK IPVS_DEST_ATTR_ADDR_FAMILY IPVS_SYNCD_ATTRIBUTES IPVS_64BIT_STATS IPVS_TUN_TYPE IPVS_TUN_CSUM IPVS_TUN_GRE VRRP_IPVLAN IFLA_LINK_NETNSID GLOB_BRACE GLOB_ALTDIRFUNC INET6_ADDR_GEN_MODE VRF SO_MARK
ubuntu@l4lb-1:~$
```

### Keepalivedの設定

Keepalivedの設定ファイルは`/etc/keepalived/keepalived.conf`です。まずl4lb-1に設定ファイルを配置します。また、l4lb-1が基本的にactiveとなるようにします。
VIPの選び方ですが、VMに静的にアドレスを割り当てるときと同じく、使われていないアドレスを適当に指定します。

@[card])(<https://www.keepalived.org/manpage.html>)

```bash
VIP="192.168.16.140"
# @l4lb-1
cat << EOF | sudo tee /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
    # Master昇格時にGARPを送信する間隔
    # 古いARPエントリを持つ機器があるとき、その機器は通信できなくなってしまう
    # gratuitous ARP（GARP）を送信することで、ARPテーブルを更新させることができる
    vrrp_garp_master_refresh 60
}

# HAProxyが動作していない場合、このマシンはactiveにならない
vrrp_script chk_haproxy {
    script "/usr/bin/systemctl is-active haproxy"
}

# maintenance_modeファイルが存在する場合、このマシンはpriorityが低くなる
vrrp_script chk_maintenance_mode {
    script "/bin/sh -c 'if [ -f /etc/keepalived/maintenance_mode ]; then exit 1; else exit 0; fi'"
    weight 10
}

vrrp_instance VI_1 {
    # 初期ステート
    state MASTER
    # VM上のインターフェース名
    interface eth0
    # 複数のKeepalivedが同じネットワークにいるとき、同じvirtual_router_idを持つもの同士がVRRPで通信する
    virtual_router_id 51
    # 高いpriorityを持つマシンがactiveになる
    priority 100
    # MASTERになったときに設定される仮想IPアドレス
    virtual_ipaddress {
        ${VIP}
    }
    track_script {
        chk_haproxy
        chk_maintenance_mode
    }
}
EOF
sudo systemctl restart keepalived

# 仮想IPアドレスが設定されていることを確認
ip a | grep -w inet
```

l4lb-1に仮想IPアドレス `192.168.16.140` でアクセスできることを確かめます。

```bash
# @client
VIP="192.168.16.140"
curl -s http://${VIP}/hostname
```

次にl4lb-2にも設定ファイルを配置します。こちらが基本的にstandbyとなるようにします。

```bash
# @l4lb-2
VIP="192.168.16.140"
cat << EOF | sudo tee /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
}

vrrp_script chk_haproxy {
    script "/usr/bin/systemctl is-active haproxy"
}

vrrp_script chk_maintenance_mode {
    script "/bin/sh -c 'if [ -f /etc/keepalived/maintenance_mode ]; then exit 1; else exit 0; fi'"
    weight 10
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    # l4lb-1よりも低いpriorityを持つ
    priority 99
    virtual_ipaddress {
        ${VIP}
    }
    track_script {
        chk_haproxy
        chk_maintenance_mode
    }
}
EOF
sudo systemctl restart keepalived

# 仮想IPアドレスが設定されていないことを確認
ip a | grep -w inet
```

### フェイルオーバーの確認

l4lb-1のKeepalivedを停止すると、l4lb-2がactiveになるはずです。試しに止めてみましょう。

```bash
# @l4lb-1
sudo systemctl stop keepalived
```

l4lb-2に仮想IPアドレス `192.168.16.140` が付与されていることを確かめます。

```bash
# @l4lb-2
ip a | grep -w inet
```

l4lb-1が復帰したときにはl4lb-1のpriorityの方が高いため、l4lb-1がactiveになります。これをフェイルバックと言います。

```bash
# @l4lb-1
sudo systemctl start keepalived
ip a | grep -w inet
```

```bash
# @l4lb-2
ip a | grep -w inet
```

### HAProxyを停止してみる

Keepalivedの設定が正しく行われていると、HAProxyを停止しても仮想IPアドレスが移行するはずです。試しに止めてみましょう。

```bash
# @l4lb-1
# メンテナンスモードから抜ける
sudo rm -f /etc/keepalived/maintenance_mode
# しばらく待って、仮想IPアドレスが設定されていることを確認
ip a | grep -w inet
# HAProxyを停止
sudo systemctl stop haproxy
# 仮想IPアドレスが設定されていないことを確認
ip a | grep -w inet
```

```bash
# @l4lb-2
ip a | grep -w inet
```

確認できたら、HAProxyを起動しておきます。

```bash
# @l4lb-1
sudo systemctl start haproxy
```

### メンテナンスモードの確認

今回、KeepalivedやHAProxyを落とすことなくトラフィックを他のロードバランサに移行できるよう、特定のパスにファイルを置くとそのサーバのpriorityが下がる仕組みを用意しています。また、メンテナンス中にうっかりkeepalivedなどが起動しても、このファイルがあればactiveにならないためメンテナンス中の事故を防げます。
これを使ってl4lb-1をメンテナンスモードにしてみましょう。

```bash
# @l4lb-1
sudo touch /etc/keepalived/maintenance_mode
```

l4lb-2に仮想IPアドレスが付与されていることを確かめます。

```bash
# @l4lb-2
ip a | grep -w inet
```

このファイルを削除すると、フェイルバックします。

```bash
# @l4lb-1
sudo rm /etc/keepalived/maintenance_mode
```

## 負荷試験中にL4ロードバランサの冗長度を落とす

今回はVIPに対してリクエストを送り、L4ロードバランサの冗長性を確認していきます。

```bash
# @client
cd ~/intro-for-high-availability/k6
VIP="192.168.16.140"
cat << EOF > l4lb-ha-test.js
import http from 'k6/http';
import { check } from 'k6';

export default function () {
  let res = http.get('http://${VIP}/hostname');
  check(res, {'success': (r) => r.status === 200});
}
EOF

k6 run -d 30s --vus 10 --rps 1000 l4lb-ha-test.js
```

負荷試験中にl4lb-1のKeepalivedを停止してみます。l4lb-1からの応答が途絶えるため、l4lb-2がactiveになり操作が継続できるはずです。

```bash
# @l4lb-1
# 仮想IPアドレスが設定されていることを確認
ip a | grep -w inet
# keepalivedを停止
sudo systemctl stop keepalived
# 仮想IPアドレスが設定されていないことを確認
ip a | grep -w inet
```

筆者の環境では以下の様に全てのリクエストが成功しました。

```text
  scenarios: (100.00%) 1 scenario, 10 max VUs, 1m0s max duration (incl. graceful stop):
           * default: 10 looping VUs for 30s (gracefulStop: 30s)


     ✓ success

     checks.........................: 100.00% ✓ 29984      ✗ 0    
     data_received..................: 4.9 MB  164 kB/s
     data_sent......................: 2.6 MB  88 kB/s
     http_req_blocked...............: avg=79.7µs   min=461ns    med=2.44µs   max=1.02s    p(90)=3.94µs  p(95)=4.84µs 
     http_req_connecting............: avg=76.19µs  min=0s       med=0s       max=1.02s    p(90)=0s      p(95)=0s     
     http_req_duration..............: avg=877.02µs min=412.18µs med=767.16µs max=219.3ms  p(90)=1.11ms  p(95)=1.36ms 
       { expected_response:true }...: avg=877.02µs min=412.18µs med=767.16µs max=219.3ms  p(90)=1.11ms  p(95)=1.36ms 
     http_req_failed................: 0.00%   ✓ 0          ✗ 29984
     http_req_receiving.............: avg=44.91µs  min=7.89µs   med=38.52µs  max=599.03µs p(90)=72.55µs p(95)=86.32µs
     http_req_sending...............: avg=22.34µs  min=3.04µs   med=8.17µs   max=218.53ms p(90)=31.06µs p(95)=36.2µs 
     http_req_tls_handshaking.......: avg=0s       min=0s       med=0s       max=0s       p(90)=0s      p(95)=0s     
     http_req_waiting...............: avg=809.76µs min=368.46µs med=710.18µs max=36.87ms  p(90)=1.04ms  p(95)=1.3ms  
     http_reqs......................: 29984   998.884981/s
     iteration_duration.............: avg=10ms     min=958.84µs med=9.93ms   max=1.03s    p(90)=10.85ms p(95)=11.08ms
     iterations.....................: 29984   998.884981/s
     vus............................: 10      min=10       max=10 
     vus_max........................: 10      min=10       max=10 


running (0m30.0s), 00/10 VUs, 29984 complete and 0 interrupted iterations
default ✓ [======================================] 10 VUs  30s
```

### 負荷試験中にHAProxyを停止してみる

Keepalivedを止める方法では、仮想IPへの通信が途絶えることなく成功することを確かめました。しかし、HAProxyを止めたわけではなく、l4lb-1は動作している状況でした。ここで確実にl4lb-1では処理が出来なくなる状況を作るため、HAProxyを止めて検証します。

```bash
# @client
k6 run -d 30s --vus 10 --rps 1000 l4lb-ha-test.js
```

```bash
# @l4lb-1
sudo systemctl start keepalived
# 仮想IPアドレスが設定されていることを確認
ip a | grep -w inet
sudo systemctl stop haproxy
# 仮想IPアドレスが設定されていないことを確認
ip a | grep -w inet
```

このとき、筆者の環境では以下の様に2％ほどのリクエストが失敗しました。

```text
     ✗ success
      ↳  98% — ✓ 28863 / ✗ 483

     checks.........................: 98.35% ✓ 28863      ✗ 483  
     data_received..................: 4.7 MB 158 kB/s
     data_sent......................: 2.5 MB 85 kB/s
     http_req_blocked...............: avg=429.26µs min=0s       med=2.22µs   max=1.07s    p(90)=3.38µs  p(95)=4.15µs 
     http_req_connecting............: avg=426.19µs min=0s       med=0s       max=1.07s    p(90)=0s      p(95)=0s     
     http_req_duration..............: avg=826.35µs min=0s       med=761.66µs max=9.55ms   p(90)=1.11ms  p(95)=1.32ms 
       { expected_response:true }...: avg=839.84µs min=394.54µs med=764.9µs  max=9.55ms   p(90)=1.11ms  p(95)=1.33ms 
     http_req_failed................: 1.64%  ✓ 483        ✗ 28863
     http_req_receiving.............: avg=42.58µs  min=0s       med=37.36µs  max=735.79µs p(90)=70.36µs p(95)=81.39µs
     http_req_sending...............: avg=13.51µs  min=0s       med=7.53µs   max=452.64µs p(90)=29.11µs p(95)=33.83µs
     http_req_tls_handshaking.......: avg=0s       min=0s       med=0s       max=0s       p(90)=0s      p(95)=0s     
     http_req_waiting...............: avg=770.25µs min=0s       med=705.75µs max=9.53ms   p(90)=1.04ms  p(95)=1.26ms 
     http_reqs......................: 29346  977.684975/s
     iteration_duration.............: avg=10.21ms  min=499.56µs med=9.91ms   max=1.08s    p(90)=10.83ms p(95)=11.05ms
     iterations.....................: 29346  977.684975/s
     vus............................: 10     min=10       max=10 
     vus_max........................: 10     min=10       max=10 


running (0m30.0s), 00/10 VUs, 29346 complete and 0 interrupted iterations
default ✓ [======================================] 10 VUs  30s
```

これはKeepalivedの仕組み上、避けることが難しいものです。l4lb-1においてHAProxyの状態を確認しているスクリプト（`systemctl is-active haproxy`）が失敗してから、l4lb-2がactiveになるまでの間に来たリクエストは失敗します。
そのため、冗長化したL4LBのメンテナンスを行う場合、Keepalivedを止めるかメンテナンスファイルを配置するなどして、l4lb-1のHAProxyが動作している間にl4lb-2へ仮想IPアドレスが移行するようにしましょう。

### 負荷試験中にメンテナンスモードに入れてl4lb-1を再起動する

メンテナンスモードに入れれば、l4lb-1を再起動しても可用性が下がることはありません。

```bash
# @client
k6 run -d 30s --vus 10 --rps 1000 l4lb-ha-test.js
```

```bash
# @l4lb-1
sudo systemctl start haproxy
# 仮想IPアドレスが設定されていることを確認
ip a | grep -w inet
# メンテナンスモードに入れてVIPをl4lb-2に移行
sudo touch /etc/keepalived/maintenance_mode
# 仮想IPアドレスが設定されていないことを確認
ip a | grep -w inet
sudo reboot
```

結果は省略します。l4lb-1が起動してきたら、メンテナンスモードから抜けておきましょう。

```bash
# @l4lb-1
sudo rm /etc/keepalived/maintenance_mode
```

## コラム：active/active構成

実は仮想IPを2つ用意することで、active/active構成にできます。この場合、この仮想IPのいずれかにアクセスすればサービスに接続出来ます。
今回はl4lb-1が優先的に仮想IPアドレス `192.168.16.140` を持っていました。ここで仮想IPアドレス `192.168.16.141` を用意し、l4lb-2にl4lb-1よりも高いpriorityを設定するconfigを用意することで、l4lb-2もactiveにできます。
下記の様なconfigをl4lb-1とl4lb-2に設定します。それぞれのVMに対してVIPが付与され、どちらかを止めればこれらのVIPが移動することを確認してみましょう。

```text
vrrp_instance VI_2 {
    state MASTER
    interface eth0
    virtual_router_id 52
    # l4lb-2の方が高くなるようにする
    priority 100
    virtual_ipaddress {
        192.168.16.141
    }
    track_script {
        chk_haproxy
        chk_maintenance_mode
    }
}
```

## 後片付け

これらのVMは次章以降でも使うため、削除せずそのままにしておきます。

## まとめ

この章では、Keepalivedを使ってロードバランサを冗長化する方法を学びました。KeepalivedはVRRPを使って複数のマシン間で仮想IPアドレスを共有できます。この仮想IPアドレスへアクセスすることで、L4ロードバランサが冗長化された状態で運用できるようになります。

Keepalivedで冗長化されたL4ロードバランサをメンテナンスする場合、ロードバランサとしての機能を損なわない状態で先にVIPを移動させることが重要です。先にKeepalivedを停止する、メンテナンスモードに入れる方法を用意しておくなどの方法により、可用性を損なわずにメンテナンスを行えます。

ここまでは基本的に各々がステートを持っていないシステムを取り扱ってきました。各VMは状態を持たなかったため、どのVMにアクセスしても同じ結果が得られました。次章ではステートを持つ場合のことを考えていきましょう。
