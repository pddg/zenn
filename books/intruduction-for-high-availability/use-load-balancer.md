---
title: "ロードバランサでリクエストを分散する"
---

## ロードバランサの役割

ロードバランサはそのままロード（負荷）をバランシング（調整・分散）するものです。Webサーバの文脈では、リクエストを受け取り、複数のサーバに分散して処理を行う役割を持ちます。
リクエストを分散する対象はWebサーバだけでなく、データベースやキャッシュサーバなどであることもあります。

単に指定された対象にリクエストを順に割り振るだけが役割ではなく、対象のサーバがダウンしたとき、そのサーバにリクエストを送らないようにすることもロードバランサの役割です。この機能を利用することで、前章では発生したサーバのダウンによるサービスの停止を防ぐことができます。

## ロードバランサの種類

ロードバランサは主に以下の2つの種類に分けられます。

- L4ロードバランサ
  - ネットワーク層（レイヤ4）で動作するロードバランサ
- L7ロードバランサ
  - アプリケーション層（レイヤ7）で動作するロードバランサ

### L4ロードバランサ

L4ロードバランサは、TCP/IPレイヤの情報を利用してリクエストを分散します。HTTPを解釈したり、リクエストの内容を見ることはできません。そのため、HTTP以外のプロトコルにも対応できます。
例えばデータベースのリードレプリカへの接続を分散したり、Postfixなどのメールサーバのリクエストを分散できます。

### L7ロードバランサ

L7ロードバランサは、HTTPなどのアプリケーション層の情報を利用してリクエストを分散します。そのため、HTTPのヘッダやクエリパラメータなどを利用してリクエストを分散できます。
例えば、特定のホスト名へのリクエストが来たときにはサービスAに送る、特定のパスにリクエストが来た場合にはサービスBにリクエストを送る、といったことが可能です。これにより1つのL7ロードバランサで複数のサービスを提供できます。

L7ロードバランサはHTTPを解釈する以外に、TLSの終端を行うこともできます。これによりバックエンドのサーバはTLSを利用せずに平文で通信できます[^tlsinbackend]。

[^tlsinbackend]: なお、昨今ではこういったバックエンドのサービス間でもTLSを利用することが推奨されています。Istio、Linkerdなどのサービスメッシュを利用することの目的の1つに、こうしたバックエンドのサービス間の通信を暗号化することがあります。

一般的にL7ロードバランサはHTTPを解釈する必要がある都合上、L4ロードバランサよりも高負荷になりやすいです。つまり同じ数のリクエストを処理する場合、L7ロードバランサはL4ロードバランサよりも多くのリソースを必要とします。L4ロードバランサとL7ロードバランサを併用して、小数のL4ロードバランサでリクエストを分散し、その後のL7ロードバランサで更にアプリケーションサーバに振り分けるといった処理を行うこともあります。

## 利用できるロードバランサ

ロードバランサには、ハードウェア製品とソフトウェア製品があります。ハードウェア製品は専用のハードウェアが存在し、一般的にとても高価です。

- [BIG-IP (F5 Networks)](https://www.f5.com/ja_jp/products/big-ip-services/iseries-appliance)
- [NetScaler (Citryx Systems)](https://docs.netscaler.com/ja-jp/citrix-adc.html)
- [Thunder ADC (A10 Networks)](https://www.a10networks.co.jp/products/thunder-adc/)

などの製品が知られています。筆者はこれらのハードウェア製品を利用したことがないため、詳細については触れません。

一方、ソフトウェア製品としては、リバースプロキシとして動作するものを含めると以下のようなものがあります。

- [LVS（Linux Virtual Server）](http://www.linuxvirtualserver.org/about.html)
  - IPVS（IP Virtual Server）というカーネルモジュールを利用することで、ロードバランサを実現する。
  - Linuxがインストールされていれば利用できる
- [HAProxy](https://www.haproxy.org/)
  - TCP/HTTPのロードバランサとして利用できるOSS
- [Nginx](https://nginx.org/)
  - リバースプロキシとして動作するWebサーバであり、ロードバランサとしても利用できる

また、各パブリッククラウドプロバイダもロードバランサのサービスを提供しています。

- [AWS Elastic Load Balancing](https://aws.amazon.com/jp/elasticloadbalancing/)
- [Azure Load Balancer](https://azure.microsoft.com/ja-jp/products/load-balancer/)
- [Google Cloud Load Balancing](https://cloud.google.com/load-balancing/)
- [Oracle Cloud Infrastructure Load Balancing](https://www.oracle.com/jp/cloud/networking/load-balancing/)

今回は、複数のnginxに対してリクエストを分散するL4ロードバランサとしてHAProxyを採用します。これにより複数あるNginxのうちの1つを落としても、他のNginxにリクエストを送ることでサービスを継続できます。
HAProxyが1つであれば、HAProxyがダウンすることでサービスが停止するのではないか、という疑問が生じると思います。L4ロードバランサの冗長化については次章で触れます。

## HAProxyを使ったL4ロードバランサの構築

ここで最終的に構築する構成は以下の様になります。1台のHAProxyが複数のnginxにリクエストを分散し、nginxがレスポンスを返します。全てのnginxは同じコンテンツを提供し、全てのnginxがダウンしない限りはサービスが継続します。

![l4lb-single.drawio.png](/images/books/introduction-for-high-availability/l4lb-single.drawio.png)

### VMの作成

ここでは3つのVMを作成します。

```bash
# @node1
# 共通して使う変数
TEMPLATE_VMID=999
GATEWAY="192.168.16.1"

# L4ロードバランサ
L4LB_VMID=500
L4LB_NAME=l4lb-1
L4LB_IP="192.168.16.133"

# Webサーバ
NGINX1_VMID=510
NGINX1_NAME=nginx-1
NGINX1_IP="192.168.16.135"

NGINX2_VMID=511
NGINX2_NAME=nginx-2
NGINX2_IP="192.168.16.136"

# 一括で作成する
cat << EOF > /tmp/vms.txt
${L4LB_VMID} ${L4LB_NAME} ${L4LB_IP}
${NGINX1_VMID} ${NGINX1_NAME} ${NGINX1_IP}
${NGINX2_VMID} ${NGINX2_NAME} ${NGINX2_IP}
EOF

while read vmid name ip; do
  qm clone ${TEMPLATE_VMID} ${vmid} --name ${name} --full
  qm set ${vmid} --ipconfig0 "ip=${ip}/32,gw=${GATEWAY}"
  qm start ${vmid}
done < /tmp/vms.txt
```

### HAProxyのインストール

l4lb-1にHAProxyをインストールします。Ubuntuではaptリポジトリからインストールできます。

```bash
# @l4lb-1
sudo apt update
sudo apt install -y --no-install-recommends haproxy
```

22.04 LTSではv2.4.24がインストールされました。

```text
ubuntu@l4lb-1:~$ haproxy -v
HAProxy version 2.4.24-0ubuntu0.22.04.1 2023/10/31 - https://haproxy.org/
Status: long-term supported branch - will stop receiving fixes around Q2 2026.
Known bugs: http://www.haproxy.org/bugs/bugs-2.4.24.html
Running on: Linux 5.15.0-92-generic #102-Ubuntu SMP Wed Jan 10 09:33:48 UTC 2024 x86_64
ubuntu@l4lb-1:~$ 
```

以降ではこのバージョンを前提として進めます。異なるバージョンの場合は、設定ファイルの記述が異なるかもしれません。

### HAProxyの設定

設定ファイルは `/etc/haproxy/haproxy.cfg` です。デフォルトではHTTPのロードバランサとしての設定が記述されていますが、ここではL4ロードバランサとしての設定を記述します。

```bash
# @l4lb-1
cat << EOF | sudo tee /etc/haproxy/haproxy.cfg
# グローバルの設定
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    # HAProxyの状態を見るためのソケットを作成
    stats socket ipv4@localhost:8080 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    timeout connect 5000

# どのポートでlistenしてどこにリクエストを流すか
frontend main
    # 80番ポートでlistenする
    bind *:80
    # nginxにリクエストを分散する
    default_backend nginx

# バックエンドのサービスの設定
backend nginx
    # ランドロビン方式でリクエストを分散する
    balance roundrobin
    # 分散する対象のサーバを列挙
    # checkを付けるとTCPでリクエストを受け付ける常態かどうかを定期的に確認する
    server ${NGINX1_NAME} ${NGINX1_IP}:80 check
    server ${NGINX2_NAME} ${NGINX2_IP}:80 check
EOF
```

リクエストの分散にはいくつかの方式があります。HAProxyがサポートするいくつかの方法を以下に示します。

- `roundrobin`
  - 各サーバに設定された重みに従って交互にリクエストを分散する
  - 重みが同じ場合、同じ数のリクエストが各サーバに分散される
- `leastconn`
  - コネクション数が最も少ないサーバにリクエストを分散する
  - 1つのコネクションのセッションが長い場合に有効であり、HTTPなどではセッションが短いため適していないらしい
- `source`
  - クライアントのIPアドレスのハッシュを計算し、全サーバの重みの合計値で割って余りを求め、その余りに対応するサーバにリクエストを分散する
  - バックエンドのサーバの数が変わらない限り、あるIPアドレスからアクセスしたリクエストは常に同じサーバに分散される
- `first`
  - 各サーバに振られたidの若い順に、最大接続コネクション数に達するまで順にリクエストを分散する
  - つまり、使用されるサーバの数が最小になるようにリクエストを分散する

他の分散アルゴリズムや、詳細な説明は以下のリンクを参照してください。

@[card](https://www.haproxy.com/documentation/haproxy-configuration-manual/latest/#4.2-balance)

HAProxyを再起動します。

```bash
# @l4lb-1
sudo systemctl restart haproxy
```

### HAProxyの状態を見る

l4lb-1の8080番ポートにソケットが作成されているため、HAProxyの状態を見ることができます。

```bash
# @l4lb-1
echo 'show info' | nc localhost 8080 | head -n 5
```

```text
ubuntu@l4lb-1:~$ echo 'show info' | nc localhost 8080 | head -n 5
Name: HAProxy
Version: 2.4.24-0ubuntu0.22.04.1
Release_date: 2023/10/31
Nbthread: 2
Nbproc: 1
ubuntu@l4lb-1:~$ 
```

`show stat` コマンドの結果を整形することで、今のバックエンドの状態を表示できます。

```bash
# @l4lb-1
echo 'show stat nginx 4 -1' | nc localhost 8080 | cut -d "," -f 1-2,18 | column -s, -t
```

現状ではnginx-1、nginx-2共にnginxを起動していないため、いずれも `DOWN` となっています。

```text
ubuntu@l4lb-1:~$ echo 'show stat nginx 4 -1' | nc localhost 8080 | cut -d "," -f 1-2,18 | column -s, -t
# pxname  svname   status
nginx     nginx-1  DOWN
nginx     nginx-2  DOWN
ubuntu@l4lb-1:~$ 
```

@[card](https://www.haproxy.com/documentation/haproxy-runtime-api/reference/show-stat/)

### nginxのインストール

nginx-1およびnginx-2にnginxをインストールします。

```bash
# @nginx-1, nginx-2
sudo apt update
sudo apt install -y --no-install-recommends nginx
```

ここで、各サーバにリクエストが分散されていることをわかりやすく確かめるため、各nginxからはレスポンスボディとして自身のホスト名を返すようにします。

```bash
# @nginx-1, nginx-2
cat << 'EOF' | sudo tee /etc/nginx/sites-enabled/default
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    root /var/www/html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /hostname {
        default_type text/plain;
        return 200 "$hostname\n";
    }
}
EOF
sudo systemctl reload nginx
```

それぞれアクセス可能になっていることを確かめます。

```bash
# @l4lb-1
# nginx-1が返るはず
curl -s http://${NGINX1_IP}/hostname
# nginx-2が返るはず
curl -s http://${NGINX2_IP}/hostname
```

### ロードバランサの動作確認

l4lb-1に対してリクエストを送り、nginx-1とnginx-2にリクエストが分散されることを確認します。

```bash
# @client
L4LB_IP="192.168.16.133"
curl -s http://${L4LB_IP}/hostname
curl -s http://${L4LB_IP}/hostname
```

`nginx-1` と `nginx-2` が交互に返ってくれば成功です。ここで、HAProxyの統計情報を見てみます。

```bash
# @l4lb-1
echo 'show stat nginx 4 -1' | nc localhost 8080 | cut -d "," -f 1-2,18 | column -s, -t
```

```text
ubuntu@l4lb-1:~$ echo 'show stat nginx 4 -1' | nc localhost 8080 | cut -d "," -f 1-2,18 | column -s, -t
# pxname  svname   status
nginx     nginx-1  UP
nginx     nginx-2  UP
ubuntu@l4lb-1:~$ 
```

ここで、nginx-1を落としてみます。

```bash
# @nginx-1
sudo systemctl stop nginx
```

以下の様にnginx-1だけが落ちている事が分かります。

```text
ubuntu@l4lb-1:~$ echo 'show stat nginx 4 -1' | nc localhost 8080 | cut -d "," -f 1-2,18 | column -s, -t
# pxname  svname   status
nginx     nginx-1  DOWN
nginx     nginx-2  UP
ubuntu@l4lb-1:~$ 
```

この状態でアクセスしても、アクセスは成功します。ただし、必ずnginx-2からのレスポンスしか返ってきません。

```bash
curl -s http://${L4LB_IP}/hostname
curl -s http://${L4LB_IP}/hostname
```

## 負荷試験中にnginx-1を落としてみる

再びk6を使ってリクエストを送り、nginx-1を落とすとどうなるかを見てみます。

nginx-1を落としたままの場合は復旧しておきます。

```bash
# @nginx-1
sudo systemctl start nginx
```

```bash
# @client
cd ~/intro-for-high-availability/k6
cat << 'EOF' > haproxytest.js
import http from 'k6/http';
import { check } from 'k6';

export default function () {
  let res = http.get('http://192.168.16.133/hostname');
  check(res, {'success': (r) => r.status === 200});
}
EOF
k6 run --vus 10 --rps 1000 --duration 30s haproxytest.js
```

実行中にnginx-1を落とします。

```bash
# @nginx-1
sudo systemctl stop nginx
```

いくつかリクエストが失敗してしまいました。一方で、片方を完全にダウンさせてしまったにもかかわらず、リクエストの99.92％が成功しています。

```bash
     ✗ success
      ↳  99% — ✓ 26137 / ✗ 20

     checks.........................: 99.92% ✓ 26137      ✗ 20   
     data_received..................: 4.3 MB 143 kB/s
     data_sent......................: 2.3 MB 77 kB/s
     http_req_blocked...............: avg=8.02µs   min=390ns    med=1.78µs   max=2.86ms   p(90)=2.76µs   p(95)=3.29µs 
     http_req_connecting............: avg=5.36µs   min=0s       med=0s       max=2.82ms   p(90)=0s       p(95)=0s     
     http_req_duration..............: avg=3.08ms   min=407.23µs med=711.38µs max=3.01s    p(90)=1.02ms   p(95)=1.19ms 
       { expected_response:true }...: avg=782.75µs min=407.23µs med=711.23µs max=37.1ms   p(90)=1.02ms   p(95)=1.18ms 
     http_req_failed................: 0.07%  ✓ 20         ✗ 26137
     http_req_receiving.............: avg=39.98µs  min=0s       med=35.31µs  max=649.74µs p(90)=66.23µs  p(95)=72.37µs
     http_req_sending...............: avg=11.19µs  min=2.06µs   med=6.29µs   max=387.3µs  p(90)=26.31µs  p(95)=29.62µs
     http_req_tls_handshaking.......: avg=0s       min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s     
     http_req_waiting...............: avg=3.03ms   min=365.63µs med=660.17µs max=3.01s    p(90)=969.61µs p(95)=1.14ms 
     http_reqs......................: 26157  871.615104/s
     iteration_duration.............: avg=11.46ms  min=473.35µs med=9.87ms   max=3.01s    p(90)=10.75ms  p(95)=10.98ms
     iterations.....................: 26157  871.615104/s
     vus............................: 10     min=10       max=10 
     vus_max........................: 10     min=10       max=10 
```

### 更にダウンタイムを減らす

現状のHAProxyでは、バックエンドが `DOWN` と判定されるまでの間に来たリクエストはそのバックエンドに送られてしまう可能性があります。これを防ぐため、特定の問題が発生した場合にはリトライの後に他のバックエンドにリクエストを送れるようにします。

`/etc/haproxy/haproxy.cfg` に以下の様に追記します。

```bash
# @l4lb-1
cat << 'EOF' | sudo tee -a /etc/haproxy/haproxy.cfg
    # リクエスト失敗時に他のバックエンドにリクエストの送信を許可する
    option redispatch
    # リクエストにおいて3回までリトライする
    retries 3
    # どのようなエラーにおいてリトライするか
    # ここではコネクションエラーなど、リトライ可能なエラー全てでリトライする
    retry-on all-retryable-errors
EOF
```

これにより、HAProxyのbackendの設定は以下の様になります。

```text
# バックエンドのサービスの設定
backend nginx
    # ランドロビン方式でリクエストを分散する
    balance roundrobin
    # 分散する対象のサーバを列挙
    # checkを付けるとTCPでリクエストを受け付ける常態かどうかを定期的に確認する
    server ${NGINX1_NAME} ${NGINX1_IP}:80 check
    server ${NGINX2_NAME} ${NGINX2_IP}:80 check
    # リクエスト失敗時に他のバックエンドにリクエストの送信を許可する
    option redispatch
    # リクエストにおいて3回までリトライする
    retries 3
    # どのようなエラーにおいてリトライするか
    # ここではコネクションエラーなど、リトライ可能なエラー全てでリトライする
    retry-on all-retryable-errors
```

この設定を適用したHAProxyを再起動します。

```bash
# @l4lb-1
sudo systemctl restart haproxy
```

適用後、再度負荷試験を行います。実行中にnginx-1のnginxを立ち上げたり、落としたりしてみましょう。

```bash
# @client
k6 run --vus 10 --rps 1000 --duration 30s haproxytest.js
```

このようにリクエストの失敗は発生しませんでした。

```text
  scenarios: (100.00%) 1 scenario, 10 max VUs, 1m0s max duration (incl. graceful stop):
           * default: 10 looping VUs for 30s (gracefulStop: 30s)


     ✓ success

     checks.........................: 100.00% ✓ 29980      ✗ 0    
     data_received..................: 4.9 MB  164 kB/s
     data_sent......................: 2.6 MB  88 kB/s
     http_req_blocked...............: avg=112.67µs min=521ns    med=2.3µs    max=1.06s    p(90)=3.4µs   p(95)=4.18µs 
     http_req_connecting............: avg=109.48µs min=0s       med=0s       max=1.06s    p(90)=0s      p(95)=0s     
     http_req_duration..............: avg=868.7µs  min=400.5µs  med=751.07µs max=38.36ms  p(90)=1.15ms  p(95)=1.39ms 
       { expected_response:true }...: avg=868.7µs  min=400.5µs  med=751.07µs max=38.36ms  p(90)=1.15ms  p(95)=1.39ms 
     http_req_failed................: 0.00%   ✓ 0          ✗ 29980
     http_req_receiving.............: avg=42.98µs  min=8.26µs   med=37.7µs   max=532.45µs p(90)=70.31µs p(95)=80.98µs
     http_req_sending...............: avg=14.19µs  min=3µs      med=7.7µs    max=1.27ms   p(90)=29.57µs p(95)=33.62µs
     http_req_tls_handshaking.......: avg=0s       min=0s       med=0s       max=0s       p(90)=0s      p(95)=0s     
     http_req_waiting...............: avg=811.51µs min=355.73µs med=695.71µs max=38.28ms  p(90)=1.08ms  p(95)=1.33ms 
     http_reqs......................: 29980   998.992042/s
     iteration_duration.............: avg=10ms     min=669.41µs med=9.9ms    max=1.07s    p(90)=10.85ms p(95)=11.1ms 
     iterations.....................: 29980   998.992042/s
     vus............................: 10      min=10       max=10 
     vus_max........................: 10      min=10       max=10 
```

このように設定しておくことでバックエンドのnginxのシャットダウンをしてもリクエストは失敗しなくなりました。
たまたまシャットダウンするサーバへのリクエストを引き当てた場合、HAProxyによるリトライによって多少レイテンシが増えることはありますが、リクエスト自体は成功します。

### コラム: nginxを止めてもリクエストが失敗しない理由

今回、nginxは `systemctl stop` を使って停止しました。これがもし機材の障害のように本当に突然の断によって停止するなら、新規のリクエストはともかく、実行中のリクエストはレスポンスが正しく返って来ないため失敗するはずです。しかし今回の負荷試験中にそのような挙動は見られませんでした。

その理由を知るため、まずは `systemctl stop nginx` が実際には何をやっているかを見てみます。実はnginxのsystemd unitに詳細が書かれています。

```text
ubuntu@nginx-1:~$ cat /lib/systemd/system/nginx.service
# Stop dance for nginx
# =======================
#
# ExecStop sends SIGSTOP (graceful stop) to the nginx process.
# If, after 5s (--retry QUIT/5) nginx is still running, systemd takes control
# and sends SIGTERM (fast shutdown) to the main process.
# After another 5s (TimeoutStopSec=5), and if nginx is alive, systemd sends
# SIGKILL to all the remaining processes in the process group (KillMode=mixed).
#
# nginx signals reference doc:
# http://nginx.org/en/docs/control.html
#
[Unit]
Description=A high performance web server and a reverse proxy server
Documentation=man:nginx(8)
After=network.target nss-lookup.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t -q -g 'daemon on; master_process on;'
ExecStart=/usr/sbin/nginx -g 'daemon on; master_process on;'
ExecReload=/usr/sbin/nginx -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx.pid
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target
ubuntu@nginx-1:~$ 
```

一部書いてあることが間違っており、 `SIGSTOP` ではなく使うのは `SIGQUIT` ですが、概ね以下のような流れでnginxを停止します。

1. `SIGQUIT` を送り、nginxにGraceful Shutdownを要求する
2. 5秒後にnginxがまだ生きている場合、 `SIGTERM` を送ってFast Shutdownを要求する
3. さらに5秒後にnginxがまだ生きている場合、 `SIGKILL` を送ってプロセスを強制終了する

Graceful Shutdownとは、新規のリクエストを受け付けないが実行中のリクエストを完了するまで待ってプロセスを終了することを指します。

nginxは `SIGQUIT` を受け取ると新規のリクエストを受け付けなくなりますが、実行中のリクエストは完了するまで待ちます。その後 `SIGTERM` を受け取るまでの5秒間は正常なレスポンスを返すことができます。

@[card](http://nginx.org/en/docs/control.html)

Graceful Shutdownのタイムアウト時間はシステムによって様々であるため、適切な時間を設定することが重要です。例えば1リクエストに10秒かかることが分かっているサーバで5秒のタイムアウトは短すぎます。一方で、あまりに長く設定するとサービスの停止が遅れる事になります。どの程度の時間のリクエストは仕方ないと割り切るかなどは、普段のレスポンスタイムなどを計測して把握しておくと良いでしょう。

### 負荷試験中にVMを再起動してみる

こちらも念のためやってみます。

```bash
# @client
k6 run --vus 10 --rps 1000 --duration 30s haproxytest.js
```

負荷試験中にVMを再起動します。

```bash
# @node1
NGINX1_VMID=510
qm reboot ${NGINX1_VMID}
```

このrebootではVMは正しい再起動の手順をとるため、nginxはGraceful Shutdownを行ってから再起動します。そのため、リクエストの失敗は発生しないはずです。

### HAProxyを再起動してみる

これはもはやわかりきったことですが、HAProxy自体の再起動が必要になるシチュエーションを考えてみます。現状ではL4ロードバランサは一台であり冗長化できていないので、HAProxyを再起動すると一時的にサービスが停止します。

```bash
# @client
k6 run --vus 10 --rps 1000 --duration 30s haproxytest.js
```

```bash
# @l4lb-1
sudo systemctl restart haproxy
```

筆者の環境では以下の様に、47個のリクエストが失敗しました。

```text
     ✗ success
      ↳  99% — ✓ 29954 / ✗ 47

     checks.........................: 99.84% ✓ 29954      ✗ 47   
     data_received..................: 4.9 MB 164 kB/s
     data_sent......................: 2.6 MB 88 kB/s
     http_req_blocked...............: avg=111.27µs min=0s       med=2.17µs   max=1.04s    p(90)=3.24µs  p(95)=3.95µs 
     http_req_connecting............: avg=108.33µs min=0s       med=0s       max=1.04s    p(90)=0s      p(95)=0s     
     http_req_duration..............: avg=863.08µs min=0s       med=766.64µs max=36.67ms  p(90)=1.13ms  p(95)=1.33ms 
       { expected_response:true }...: avg=864.16µs min=410.53µs med=766.9µs  max=36.67ms  p(90)=1.13ms  p(95)=1.33ms 
     http_req_failed................: 0.15%  ✓ 47         ✗ 29954
     http_req_receiving.............: avg=42.81µs  min=0s       med=37.1µs   max=629.39µs p(90)=69.92µs p(95)=80.94µs
     http_req_sending...............: avg=13.08µs  min=0s       med=7.27µs   max=416.16µs p(90)=28.56µs p(95)=32.67µs
     http_req_tls_handshaking.......: avg=0s       min=0s       med=0s       max=0s       p(90)=0s      p(95)=0s     
     http_req_waiting...............: avg=807.18µs min=0s       med=712.05µs max=36.61ms  p(90)=1.07ms  p(95)=1.27ms 
     http_reqs......................: 30001  999.680939/s
     iteration_duration.............: avg=9.99ms   min=1.36ms   med=9.91ms   max=1.05s    p(90)=10.83ms p(95)=11.06ms
     iterations.....................: 30001  999.680939/s
     vus............................: 10     min=10       max=10 
     vus_max........................: 10     min=10       max=10 


running (0m30.0s), 00/10 VUs, 30001 complete and 0 interrupted iterations
default ✓ [======================================] 10 VUs  30s
```

### 後片付け

今回使ったVMは次章でも使うので削除せずそのままにしてください。

## まとめ

HAProxyを使ったL4ロードバランサを構築することで、複数あるnginxにリクエストを分散し、冗長化できました。また、HAProxyの設定を変更することで、バックエンドのサーバがダウンしてもリクエストの失敗を防ぐことができました。
Webサーバであるnginxが冗長化されたことで、nginxの更新や再起動などのメンテナンスを行う際にもサービス停止せずに行えることがわかりました。

一方で、HAProxy自体の冗長化は行っていないため、HAProxyの再起動時には一時的にサービスが停止することもわかりました。次章ではHAProxyの冗長化を行い、更にサービスの可用性を向上させる方法を見ていきます。
