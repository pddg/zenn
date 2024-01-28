---
title: "Nginxでファイルを配信する"
---

## Nginxとは

[Nginx](https://k6.io/)は、Webサーバーとして使われることが多いオープンソースソフトウェアです。HTTPなどのプロトコルを使ってファイルを配信したり、リバースプロキシとして動作してバックエンドのWebアプリケーションサーバーにリクエストを転送したりできます。

https://nginx.org/en/

HTMLやCSS・JavaScriptのファイルを配信するために使われることが多く、それらをアプリケーションサーバから配信する場合と比べて高いパフォーマンスを発揮します。

他に同様の機能を持つソフトウェアとしては以下の様なものが知られています。

- [Apache HTTP Server](https://httpd.apache.org/)
- [Envoy](https://www.envoyproxy.io/)
- [H2O](https://h2o.examp1e.net/)
- [Microsoft IIS](https://learn.microsoft.com/ja-jp/iis/get-started/introduction-to-iis/iis-web-server-overview)

## Nginxのインストール

まず、NginxをインストールするVMを作成します。ここでは VMID 400、ホスト名を `nginx` とします。

```bash
# @node1
TEMPLATE_VMID=999
VMID=400
VM_NAME=nginx
IP_ADDRESS="192.168.16.132/32"
GATEWAY="192.168.16.1"
qm clone $TEMPLATE_VMID $VMID --name $VM_NAME --full
qm set $VMID --ipconfig0 "ip=${IP_ADDRESS},gw=${GATEWAY}"
qm start $VMID
```

sshでログインします。VM作成が完了するまで、sshが失敗することがあります。その場合、少し待ってから再度試してください。

```bash
# @client
IP_ADDRESS="192.168.16.132/32"
ssh ubuntu@${IP_ADDRESS}
```

次に、Nginxをインストールします。NginxはCで書かれたソフトウェアですが、Ubuntuのaptリポジトリでホストされており、aptコマンドでインストールできます。

```bash
# @nginx
sudo apt update && sudo apt install -y nginx
```

## Nginxの設定

Nginxの設定は独自の形式で記述します。`/etc/nginx/` 以下に様々な設定ファイルが配置されています。
ここでは、いくつかのファイルの関係性を説明します。重要な点以外は全てスキップしているので、詳細は公式ドキュメントを参照してください。

http://nginx.org/en/docs/beginners_guide.html

```nginx:/etc/nginx/nginx.conf
# nginxを実行するユーザ名
user www-data;
# HTTP関係の設定
http {
    # ログの出力先
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    # 以下のファイルのconfigを読み込む
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

```nginx:/etc/nginx/sites-enabled/default
server {
    # リクエストを受け付けるポート番号。
    # default_serverをつけると、リクエストに一致する設定が無い場合にこの設定にフォールバックする。
    listen 80 default_server;
    # リクエストを受け付けるホスト名。_は全てのホスト名にマッチする。
    server_name _;
    # ファイルを配信するディレクトリ
    root /var/www/html;

    # ディレクトリにアクセスした際にどのファイルを表示するかを指定する。
    index index.html index.htm index.nginx-debian.html;

    location / {
        # まずファイルとしてアクセスし、次にディレクトリとしてアクセスし、
        # それでもファイルが見つからなければ404を返す。
        try_files $uri $uri/ =404;
    }
}
```

nginxインストール時に `/var/www/html` に `index.nginx-debian.html` が配置されています。設定において`index` ディレクティブでこのファイルが指定されているので、`/` にアクセスすればこのファイルを表示できます。`http://192.168.16.132/`にブラウザでアクセスしてみます。

![nginx-welcome](/images/books/introduction-for-high-availability/nginx-welcome.png)

コマンドラインからもアクセスしてみます。

```bash
# @nginx
curl http://localhost/
```

以下の様にHTMLが表示されます。

```text
ubuntu@nginx:~$ curl http://localhost/
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
ubuntu@nginx:~$
``````

実際に `/var/www/html` に配置されているファイルと比較してみましょう。

```bash
# @nginx
# curlで取得したHTMLを標準入力から受け取り、diffコマンドで比較する。
curl -qs http://localhost/ | diff -s - /var/www/html/index.nginx-debian.html
```

```text
ubuntu@nginx:~$ curl -qs http://localhost/ | diff -s - /var/www/html/index.nginx-debian.html
Files - and /var/www/html/index.nginx-debian.html are identical
```

実際にこのファイルを配信できていることが分かります。

## Nginxをメンテナンスしたい

例えばNginxの更新が必要になったとします。このとき、通常はNginxを止めてから新しいバージョンを起動します。これらがリッスンしているポートは同じであり、新しいバージョンのプロセスを古いバージョンが起動している間に立ち上げることは出来ません。Nginxを停止すると、リクエストは受け付けられなくなります。

:::message
UbuntuにおいてNginxのプロセスは[systemd](https://systemd.io/)で管理されています。
ここではsystemdの詳細は解説せず、実行する操作のみを記述します。より詳しく知りたい場合は下記の書籍を参照してください。

[systemdの思想と機能―Linuxを支えるシステム管理のためのソフトウェアスイート](https://gihyo.jp/book/2024/978-4-297-13893-6)

:::

```bash
# @nginx
sudo systemctl stop nginx
curl http://localhost/
```

以下の様にリクエストが拒否され、アクセス出来なくなっていることがわかります。

```text
ubuntu@nginx:~$ curl http://localhost/
curl: (7) Failed to connect to localhost port 80 after 0 ms: Connection refused
ubuntu@nginx:~$
```

また、ブラウザからアクセスしても以下の様にエラーが表示されます。

![nginx-connection-refused](/images/books/introduction-for-high-availability/nginx-connection-refused.png)

### 再起動する

完全に停止した状態でアクセスしたので、アクセス出来ない状態になるのは当然だと思うでしょう。では、ここではアクセス中にNginxを再起動してみます。

```bash
# @nginx
# 停止させたままだったら起動しておく
sudo systemctl start nginx
# アクセス出来ることを確かめる。エラーにならなければよい。
curl -sS http://localhost/ > /dev/null

# while trueで無限ループを作り、100msごとにcurlでアクセスを続ける。
while true; do
    curl -sS http://localhost/ > /dev/null
    sleep 0.01
done
```

別のSSHセッションを開き、Nginxを再起動します。

```bash
# @nginx
sudo systemctl restart nginx
```

運が良かった人は、curlで繰り返しアクセスしているセッションにおいて何も表示されないかもしれません。しかし、運が悪い人は以下の様なエラーが表示されるかもしれません。

```text
curl: (7) Failed to connect to localhost port 80 after 0 ms: Connection refused
```

このエラーは、Nginxが起動していない間にリクエストが来てしまったために発生しています。Nginxを再起動すると、古いプロセスの停止から新しいプロセスが立ち上がって準備が完了するまでの間、リクエストを受け付けられない状態になります。
つまり、Nginxを一台だけ使っている場合は、Nginxをメンテナンスするときにはアクセスが出来なくなってしまいます。

## 負荷試験ツールで再起動時の断を観測する

先ほどはcurlでアクセスする簡単な動作テストをやってみましたが、今後の章ではより詳細な試験をするため [k6](https://k6.io/) という負荷試験ツールを用います。Grafana Labsが開発しているオープンソースの負荷試験ツールで、Goで書かれています。試験の内容自体はJavaScriptで記述します。

https://k6.io/

### k6のインストール

VMの外側からリクエストを送るため、以下のドキュメントを参照して手元のクライアントにk6をインストールします。

:::message
WindowsでWSLを使って本書を読んでいる場合は、WSLにk6をインストールしてください。
その場合、WindowsではなくLinux（おそらくUbuntu）の手順を参照してください。WSLに異なるLinuxディストリビューションを使用している方はそのディストリビューションの手順を参照してください。
:::

https://grafana.com/docs/k6/latest/get-started/installation/#installation


k6コマンドが実行できる事を確かめます。

```bash
# @client
k6 --version
```

このドキュメントでは以下のバージョンを使用しています。

```text
$ k6 --version
k6 v0.48.0 (commit/47c0a26798, go1.21.5, linux/amd64)
```

### 試験内容を記述する

適当なディレクトリを作り、 その中に負荷試験の内容を記述することにします（ディレクトリ自体は何でも良いので適当に変えてもらって良いです）。

```bash
# @client
mkdir -p ~/intro-for-high-availability/k6
cd ~/intro-for-high-availability/k6
```

次に、以下の内容を `firstrun.js` という名前で保存します。アクセス先IPアドレスは適宜変更してください。

```javascript:firstrun.js
import http from 'k6/http';
import { check } from 'k6';

export default function () {
  let res = http.get('http://192.168.16.132/');
  check(res, {'success': (r) => r.status === 200});
}
```

vimなどの操作に詳しくなく、GUIの操作に慣れていない場合は以下の様に `cat` とヒアドキュメントを使って書き込むと良いでしょう。

```bash
# @client
cat << 'EOF' > firstrun.js
import http from 'k6/http';
import { check } from 'k6';

export default function () {
  let res = http.get('http://192.168.16.132/');
  check(res, {'success': (r) => r.status === 200});
}
EOF
```

これを実行してみます。nginx VMでnginxを起動しておいてください。

```bash
# @client
# 30秒間、1秒当たり100回のリクエストを送る。
k6 run --duration 30s --rps 100 firstrun.js
```

色々な情報が表示されます。

```text
❯ k6 run --duration 30s --rps 100 firstrun.js

          /\      |‾‾| /‾‾/   /‾‾/
     /\  /  \     |  |/  /   /  /
    /  \/    \    |     (   /   ‾‾\
   /          \   |  |\  \ |  (‾)  |
  / __________ \  |__| \__\ \_____/ .io

  execution: local
     script: firstrun.js
     output: -

  scenarios: (100.00%) 1 scenario, 1 max VUs, 1m0s max duration (incl. graceful stop):
           * default: 1 looping VUs for 30s (gracefulStop: 30s)


     ✓ success

     checks.........................: 100.00% ✓ 2997      ✗ 0
     data_received..................: 2.6 MB  86 kB/s
     data_sent......................: 240 kB  8.0 kB/s
     http_req_blocked...............: avg=12.88µs min=1.29µs   med=4.18µs   max=1.68ms   p(90)=5.34µs   p(95)=6.91µs
     http_req_connecting............: avg=7.82µs  min=0s       med=0s       max=1.59ms   p(90)=0s       p(95)=0s
     http_req_duration..............: avg=1.02ms  min=622.61µs med=969.39µs max=28.93ms  p(90)=1.08ms   p(95)=1.25ms
       { expected_response:true }...: avg=1.02ms  min=622.61µs med=969.39µs max=28.93ms  p(90)=1.08ms   p(95)=1.25ms
     http_req_failed................: 0.00%   ✓ 0         ✗ 2997
     http_req_receiving.............: avg=60.54µs min=34.14µs  med=53.46µs  max=277.83µs p(90)=86.01µs  p(95)=92.16µs
     http_req_sending...............: avg=36.01µs min=5.43µs   med=36.59µs  max=178.61µs p(90)=46.64µs  p(95)=50.41µs
     http_req_tls_handshaking.......: avg=0s      min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s
     http_req_waiting...............: avg=924.4µs min=542.3µs  med=873.26µs max=28.81ms  p(90)=968.55µs p(95)=1.14ms
     http_reqs......................: 2997    99.895273/s
     iteration_duration.............: avg=10ms    min=955.32µs med=10.25ms  max=38.36ms  p(90)=10.46ms  p(95)=10.57ms
     iterations.....................: 2997    99.895273/s
     vus............................: 1       min=1       max=1
     vus_max........................: 1       min=1       max=1


running (0m30.0s), 0/1 VUs, 2997 complete and 0 interrupted iterations
default ✓ [======================================] 1 VUs  30s
```
`http_reqs` を見ると `2997 99.895273/s` と書かれており、合計2997リクエストを送っていることがわかります。 `http_req_failed` の項目にリクエストが失敗した割合が表示されていますが、ここでは `0.00%` となっており、全てのリクエストが成功していることが分かります。

### 試験中にnginxを再起動してみる

まず、負荷テストを実行します。

:::message
サーバ側の操作が負荷試験中に間に合わない場合はdurationを適宜伸ばしてください。
:::

```bash
# @client
k6 run --duration 30s --rps 100 firstrun.js
```

先ほどと同様に別のSSHセッションを開き、Nginxを再起動します。

```bash
# @nginx
sudo systemctl restart nginx
```

このように実行中にエラーが表示されると思います。

```text
❯ k6 run --duration 30s --rps 100 firstrun.js

          /\      |‾‾| /‾‾/   /‾‾/
     /\  /  \     |  |/  /   /  /
    /  \/    \    |     (   /   ‾‾\
   /          \   |  |\  \ |  (‾)  |
  / __________ \  |__| \__\ \_____/ .io

  execution: local
     script: firstrun.js
     output: -

  scenarios: (100.00%) 1 scenario, 1 max VUs, 1m0s max duration (incl. graceful stop):
           * default: 1 looping VUs for 30s (gracefulStop: 30s)

WARN[0002] Request Failed                                error="Get \"http://192.168.16.132/\": dial tcp 192.168.16.132:80: connect: connection refused"
WARN[0002] Request Failed                                error="Get \"http://192.168.16.132/\": dial tcp 192.168.16.132:80: connect: connection refused"
WARN[0002] Request Failed                                error="Get \"http://192.168.16.132/\": dial tcp 192.168.16.132:80: connect: connection refused"
WARN[0002] Request Failed                                error="Get \"http://192.168.16.132/\": dial tcp 192.168.16.132:80: connect: connection refused"
WARN[0002] Request Failed                                error="Get \"http://192.168.16.132/\": dial tcp 192.168.16.132:80: connect: connection refused"

     ✗ success
      ↳  99% — ✓ 2988 / ✗ 5

     checks.........................: 99.83% ✓ 2988      ✗ 5
     data_received..................: 2.6 MB 86 kB/s
     data_sent......................: 239 kB 8.0 kB/s
     http_req_blocked...............: avg=13.06µs  min=0s       med=3.93µs   max=1.76ms   p(90)=5.22µs   p(95)=6.97µs
     http_req_connecting............: avg=8.15µs   min=0s       med=0s       max=1.73ms   p(90)=0s       p(95)=0s
     http_req_duration..............: avg=1.02ms   min=0s       med=942.28µs max=30.3ms   p(90)=1.05ms   p(95)=1.17ms
       { expected_response:true }...: avg=1.02ms   min=575.61µs med=942.5µs  max=30.3ms   p(90)=1.05ms   p(95)=1.17ms
     http_req_failed................: 0.16%  ✓ 5         ✗ 2988
     http_req_receiving.............: avg=58.4µs   min=0s       med=51.54µs  max=431.99µs p(90)=82.56µs  p(95)=90.28µs
     http_req_sending...............: avg=34.37µs  min=0s       med=34.82µs  max=211.7µs  p(90)=45.18µs  p(95)=49.25µs
     http_req_tls_handshaking.......: avg=0s       min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s
     http_req_waiting...............: avg=930.78µs min=0s       med=849.91µs max=30.19ms  p(90)=942.15µs p(95)=1.05ms
     http_reqs......................: 2993   99.747785/s
     iteration_duration.............: avg=10.01ms  min=651.99µs med=10.21ms  max=38.59ms  p(90)=10.48ms  p(95)=10.59ms
     iterations.....................: 2993   99.747785/s
     vus............................: 1      min=1       max=1
     vus_max........................: 1      min=1       max=1


running (0m30.0s), 0/1 VUs, 2993 complete and 0 interrupted iterations
default ✓ [======================================] 1 VUs  30s
```

今回は一度だけ再起動し、その間に5つのリクエストが失敗したことがわかりました。`http_req_failed` を見ると、全リクエストのうち0.16%が失敗になったようです。
非常に少ない割合ですが、これはデフォルトの設定を使っている場合であり、Nginx自体の設定などによっては起動・停止により長い時間がかかる可能性もあります。

### 試験中にnginx VMを再起動してみる

今度はNginxを起動しているVMごと再起動してみます。例えばLinux kernelの更新などが必要になった場合、大抵はVMごと再起動が必要になります。
この操作はProxmox VEのホストから行います。

```bash
# @client
# 先に負荷試験を実行しておく。長めに5分実行する。
k6 run --duration 60s --rps 100 firstrun.js
```

```bash
# @node1
# VMID 400を再起動する
VMID=400
qm reboot ${VMID}
```

筆者の環境では以下の様な結果になりました。VMの再起動によって、Nginxのプロセス再起動よりも多くのリクエストが失敗していることが分かります。

```text
     ✗ success
      ↳  95% — ✓ 4179 / ✗ 178

     checks.........................: 95.91% ✓ 4179      ✗ 178
     data_received..................: 3.6 MB 60 kB/s
     data_sent......................: 334 kB 5.6 kB/s
     http_req_blocked...............: avg=12.17µs  min=0s       med=3.97µs   max=2.76ms   p(90)=5.19µs   p(95)=6.48µs
     http_req_connecting............: avg=7.52µs   min=0s       med=0s       max=2.7ms    p(90)=0s       p(95)=0s
     http_req_duration..............: avg=969.66µs min=0s       med=936.73µs max=28.17ms  p(90)=1.04ms   p(95)=1.2ms
       { expected_response:true }...: avg=1ms      min=439.93µs med=941.8µs  max=28.17ms  p(90)=1.04ms   p(95)=1.22ms
     http_req_failed................: 4.08%  ✓ 178       ✗ 4179
     http_req_receiving.............: avg=58.01µs  min=0s       med=52.6µs   max=522.05µs p(90)=84.56µs  p(95)=91µs
     http_req_sending...............: avg=32.83µs  min=0s       med=36.15µs  max=210.84µs p(90)=46.13µs  p(95)=49.58µs
     http_req_tls_handshaking.......: avg=0s       min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s
     http_req_waiting...............: avg=878.81µs min=0s       med=844.43µs max=27.98ms  p(90)=930.58µs p(95)=1.08ms
     http_reqs......................: 4357   72.615763/s
     iteration_duration.............: avg=13.76ms  min=514.16µs med=10.21ms  max=15.41s   p(90)=10.44ms  p(95)=10.65ms
     iterations.....................: 4357   72.615763/s
     vus............................: 1      min=1       max=1
     vus_max........................: 1      min=1       max=1


running (1m00.0s), 0/1 VUs, 4357 complete and 0 interrupted iterations
default ✓ [======================================] 1 VUs  1m0s
```

## まとめ

Nginxを使ってファイルを配信している最中にNginxを更新したい場合、現在動いているNginxのプロセスを止め、新しいプロセスを立ち上げる必要があります。このとき、古いプロセスの停止から新しいプロセスの起動までの間に来たリクエストはエラーになります。1秒当たり100リクエスト程度で試験したとき、今回は5つのリクエストが失敗していました。

Nginxが起動しているVM自体を更新し再起動しなければならない場合、VMが起動してNginxの起動が完了するまでの間に来たリクエストはエラーになります。1秒当たり100リクエスト程度で試験したとき、今回は178のリクエストが失敗していました。

このように、Nginxを使ってファイルを配信している場合、Nginxをメンテナンスするときにはアクセスが出来なくなってしまいます。この問題を解決するため、次はNginxを冗長化していきます。
