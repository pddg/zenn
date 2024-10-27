---
title: リモートキャッシュを使う
---

現在はProtocol BufferのソースコードをCI上では毎回ビルドするなど、余計な時間がかかっています。Bazelにはremote cacheという機能が存在し、ビルドの状態をキャッシュしておくことで次回以降のビルド時間を短縮できます。
@[card](https://bazel.build/remote/caching)

このremote cacheは開発者全員が参照可能で、一度ビルドされたものを全員が共有できます。

## GitHub Actionsのキャッシュの問題点

GitHub Actionsにもキャッシュ機能があるため、これを使ってCI上でのビルド時間を短縮することは最近のベストプラクティスの1つです。一方、GitHub ActionsのキャッシュをBazelと組み合わせる場合、以下のような問題点があります。

- キャッシュの容量制限がある
  - リポジトリごとに10GBまでという制限があり、これを超えたキャッシュはevictされる
  - monorepoなどで多くのパッケージを持つ場合、容易に超えてしまう可能性がある
- キャッシュを毎回全てダウンロード・アップロードするため、時間がかかる
  - GitHub Actionsではワークフローの実行ごとにダウンロード・展開・アップロードを行う
  - キャッシュ容量の肥大化とともに、これらの時間は無視できないものになる
- GitHub Actionsのキャッシュへ開発者の環境かたアクセスすることは難しい
  - Bazelはその強力なキャッシュ機能から、全てを再ビルドすることなくビルド結果を流用できることが強みである
  - しかし、GitHub ActionsのキャッシュはGitHub Actions上でのみ利用できるため、開発者のローカル環境から利用できない

これらの問題点を解決するために、Bazelを採用する場合はリモートキャッシュを使う方がよいでしょう。

## Use Google Cloud Storage

最も簡単な方法は、Google Cloud Storage（GCS）を使う方法です。BazelはGCSをネイティブにサポートしており、単純にバケットを作ってクレデンシャルを渡すだけでリモートキャッシュとして利用できます。
<https://bazel.build/versions/7.4.0/remote/caching#cloud-storage>

GCSのバケットを作成し、`.bazelrc`に以下の内容を追記します。

```sh:.bazelrc
# リモートキャッシュにGCSのバケットを指定
build --remote_cache=https://storage.googleapis.com/<bucket-name>

# リモートキャッシュにアップロードする際の認証情報を指定
# ローカルではApplication Default Credentialsを使う
build --google_default_credentials

# CIでのビルド時にはサービスアカウントのキーを使う
# これはWorkload Identity Federationなどを使って連携し、短命なクレデンシャルを生成することを推奨する
build:ci --google_credentials=/path/to/service-account-key.json

# 通常はリモートキャッシュへのアップロードを無効化。読み取りのみ。
build --remote_upload_local_results=false

# CIからはリモートキャッシュに書き込める
build:ci --remote_upload_local_results=true
```

これでCI上でのビルドで作成されたキャッシュがGCSにアップロードされ、開発者全員もそれを参照できビルドが高速化します。またCI上からもこのキャッシュが参照されるため、CI上でのビルドも高速化されます。

:::message
GCSを使う場合、リモートキャッシュの肥大化を防ぐためライフサイクルポリシーを設定した方がよいでしょう。これにより古いキャッシュが自動的に削除されるため、ストレージコストを抑えられます。
@[card](https://cloud.google.com/storage/docs/managing-lifecycles)
:::

:::message
GitHub Actionsと組み合わせる場合、バケットのリージョンを考慮する必要があります。[GitHub Hosted runner自体はUSのリージョンにある](https://github.com/orgs/community/discussions/24969)ため、バケットもUSリージョンに作成することで通信速度が向上します。しかし、日本の開発者がUSリージョンのバケットにアクセスする際には通信速度が遅くなってしまうでしょう。
:::

## Use AWS S3 / Azure Blob Storage / local storage

AWS S3（およびその互換APIを提供するストレージサービス）やAzure Blob Storageはbazelではネイティブにはサポートされていません。しかし、 [bazel-remote](https://github.com/buchgr/bazel-remote)を使うことで、それらもリモートキャッシュとして利用できるようになります。また、サーバーのローカルファイルシステムもリモートキャッシュとして提供できます。

:::message
2024/10現在、bazel-remoteはキャッシュの一貫性を保つため、1インスタンスのみ存在することを想定しています。そのため高可用性を確保する構成にすることが難しくなっています。
@[card](https://github.com/buchgr/bazel-remote/issues/483)
また、OSごとに異なるインスタンスを用意して接続先を切り替えるなど、負荷分散の方法にも工夫が必要です。最大のパフォーマンスのためには高速なSSDなどのローカルストレージを利用することが望ましいとメンテナは述べています。
@[card](https://github.com/buchgr/bazel-remote/issues/786#issuecomment-2418009617)
:::

:::message
BazelのリモートキャッシュはProtocol Bufferで定義されたgRPCのAPIから構成され、bazel-remoteはそれらのAPIの一部を実装しているOSSです。
@[card](https://github.com/bazelbuild/remote-apis/blob/main/build/bazel/remote/execution/v2/remote_execution.proto)
自前でこのようなAPIを実装することで、独自のリモートキャッシュを構築できます。
:::

bazel-remoteのコンテナイメージが提供されているため、これを使うことで簡単にリモートキャッシュを構築できます。今回はどの家庭にもあるKubernetesクラスタにbazel-remoteをデプロイしてみます。

```bash
# bazelremoteという名前空間を使う
NS=bazelremote
kubectl create ns ${NS}
```

S3バックエンドを使うためのクレデンシャルを用意します。

```bash
AWS_BUCKET=bazel-remote
# アクセス先のエンドポイントを指定する。以下はダミー。
AWS_ENDPOINT=s3.ap-northeast-1.amazonaws.com
# 認証方法として今回はAccess Keyを使う。他にもIAMロールやクレデンシャルファイルを使う方法がある。
AWS_ACCESS_KEY_ID=xxx
AWS_SECRET_ACCESS_KEY=xxx
kubectl create secret generic bazel-remote-s3 \
  --from-literal=endpoint=$AWS_ENDPOINT \
  --from-literal=bucket=$AWS_BUCKET \
  --from-literal=auth_method=access_key \
  --from-literal=access_key_id=$AWS_ACCESS_KEY_ID \
  --from-literal=secret_access_key=$AWS_SECRET_ACCESS_KEY \
  --dry-run=client \
  -o yaml > manifests/bazel-remote/01-s3.yaml
```

bazel-remoteをStatefulSetとしてデプロイするマニフェストを作成します。

```yaml:manifests/bazel-remote/02-sts.yaml
---
apiVersion: apps/v1
# bazel-remoteのexamleではDeploymentを使っているが、その性質を考えればStatefulSetの方が適している
kind: StatefulSet
metadata:
  # 注意：bazel-remoteと名付けてはならない。
  # Kubernetesが `BAZEL_REMOTE_` から始まる環境変数を注入する。
  # bazel-remoteが使う環境変数と競合する可能性がある。
  name: cache
  labels:
    app.kubernetes.io/name: bazelremote
spec:
  serviceName: bazelremote
  # replicasを1にすることで、リモートキャッシュの一貫性を保つ
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: bazelremote
  template:
    metadata:
      labels:
        app.kubernetes.io/name: bazelremote
    spec:
      containers:
        - name: cache
          image: buchgr/bazel-remote-cache:latest
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 9092
              name: grpc
          volumeMounts:
            - name: cache
              mountPath: /cache
          # Kubernetesは1.30現在、gRPC ProbeにおけるTLSをサポートしていない
          # https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-grpc-liveness-probe
          # もしBASIC認証を有効化しているとREST APIの /status にも認証を求められるようになる
          # そのため、TLSを有効化し、かつBasic認証を使う場合、liveness/readinessProbeが使えない
          # https://github.com/buchgr/bazel-remote/issues/652#issuecomment-2439217105
          livenessProbe:
            grpc:
              service: /grpc.health.v1.Health/Check
              port: 9092
          env:
            # キャッシュデータを保存するディレクトリ
            # S3などのバックエンドを使う場合にも、最初にここに保存される
            - name: BAZEL_REMOTE_DIR
              value: /cache
            # キャッシュとして利用する容量の上限（GiB）
            # これを超えたキャッシュは古いものから削除される
            - name: BAZEL_REMOTE_MAX_SIZE
              value: "10"
            # 保存時の形式。zstd or uncompressed
            # zstdを使えば実行時のCPU負荷が増えるが、圧縮率が高い
            # uncompressedを使えばCPU負荷は低いが、ストレージ容量が増える
            - name: BAZEL_REMOTE_STORAGE_MODE
              value: zstd
            - name: BAZEL_REMOTE_HTTP_ADDRESS
              value: "0.0.0.0:8080"
            - name: BAZEL_REMOTE_GRPC_ADDRESS
              value: "0.0.0.0:9092"
            # .htpasswdファイルを指定することでBasic認証を有効にできる
            # 注意：HTTPSを使わない場合、パスワードが平文で送信されるため、セキュリティ上のリスクがある
            # - name: BAZEL_REMOTE_HTPASSWD_FILE
            #   value: /etc/bazel-remote/.htpasswd
            # TLS証明書を指定することでHTTPSを有効にできる。
            # - name: BAZEL_REMOTE_TLS_CERT_FILE
            #   value: /etc/ssl/certs/tls.crt
            # - name: BAZEL_REMOTE_TLS_KEY_FILE
            #   value: /etc/ssl/certs/tls.key
            # S3バックエンドを使う場合
            - name: BAZEL_REMOTE_S3_ENDPOINT
              valueFrom:
                secretKeyRef:
                  name: bazel-remote-s3
                  key: endpoint
            - name: BAZEL_REMOTE_S3_BUCKET
              valueFrom:
                secretKeyRef:
                  name: bazel-remote-s3
                  key: bucket
            # 認証方法は3つあるがここでは単純なAccess Keyを使う
            # 他にもIAMロールやクレデンシャルファイルを指定する方法がある
            - name: BAZEL_REMOTE_S3_AUTH_METHOD
              valueFrom:
                secretKeyRef:
                  name: bazel-remote-s3
                  key: auth_method
            - name: BAZEL_REMOTE_S3_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: bazel-remote-s3
                  key: access_key_id
            - name: BAZEL_REMOTE_S3_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: bazel-remote-s3
                  key: secret_access_key
      volumes:
        - name: bazel-remote-s3
          secret:
            secretName: bazel-remote-s3
  volumeClaimTemplates:
    - metadata:
        name: cache
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: cache
spec:
  selector:
    app.kubernetes.io/name: bazelremote
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
    - protocol: TCP
      port: 9092
      targetPort: 9092
```

これらを順番に適用します。

```bash
kubectl apply -n ${NS} -f manifests/bazel-remote/01-s3.yaml
kubectl apply -n ${NS} -f manifests/bazel-remote/02-sts.yaml
```

```text
❯ kubectl get po -n ${NS}
NAME      READY   STATUS    RESTARTS   AGE
cache-0   1/1     Running   0          15s
```

PodがRunningになったら、kubectlでport-forwardを使ってローカルからアクセスできるようにします。

```bash
kubectl port-forward -n ${NS} svc/cache 9092:9092
```

:::message
実際に運用する場合にはport-forwardは使わず、以下のような方法でアクセスできるようにします。

- Ingressリソースや `type: LoadBalancer` なServiceを使って外部公開する
  - BASIC認証やmTLSを使ってセキュリティを強化しましょう
- [Actions Runner Controller](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/about-actions-runner-controller)などを使ってクラスタ内にGitHub Actions Runnerを建てる

:::

まずはキャッシュを削除してビルドの流れを2回ほど実施してみます。

:::message
あくまでこの手順はデモ用であり、実際の性能向上を保証するものではありません。この手順は筆者の自宅Kubernetesクラスタにある[Minio](https://min.io/)を使っているため、AWS S3を使う場合とは異なる傾向があるかもしれません。
:::

```bash
bazel clean
bazel build //...
bazel clean
bazel build //...
```

```text
❯ bazel clean      
INFO: Starting clean (this may take a while). Consider using --async if the clean takes more than several minutes.

❯ bazel build //...
INFO: Analyzed 50 targets (538 packages loaded, 31070 targets configured).
INFO: From Linking external/protobuf~/src/google/protobuf/io/libio_win32.a [for tool]:
warning: /Library/Developer/CommandLineTools/usr/bin/libtool: archive library: bazel-out/darwin_arm64-opt-exec-ST-d57f47055a04/bin/external/protobuf~/src/google/protobuf/io/libio_win32.a the table of contents is empty (no object file members in the library define global symbols)
INFO: From Linking external/protobuf~/protoc [for tool]:
ld: warning: ignoring duplicate libraries: '-lm', '-lpthread'
INFO: Found 50 targets...
INFO: Elapsed time: 100.071s, Critical Path: 44.27s
INFO: 1571 processes: 614 internal, 952 darwin-sandbox, 5 local.
INFO: Build completed successfully, 1571 total actions

❯ bazel clean                         
INFO: Starting clean (this may take a while). Consider using --async if the clean takes more than several minutes.

❯ bazel build //...
INFO: Analyzed 50 targets (538 packages loaded, 31070 targets configured).
INFO: From Linking external/protobuf~/src/google/protobuf/io/libio_win32.a [for tool]:
warning: /Library/Developer/CommandLineTools/usr/bin/libtool: archive library: bazel-out/darwin_arm64-opt-exec-ST-d57f47055a04/bin/external/protobuf~/src/google/protobuf/io/libio_win32.a the table of contents is empty (no object file members in the library define global symbols)
INFO: From Linking external/protobuf~/protoc [for tool]:
ld: warning: ignoring duplicate libraries: '-lm', '-lpthread'
INFO: Found 50 targets...
INFO: Elapsed time: 94.892s, Critical Path: 33.16s
INFO: 1571 processes: 614 internal, 952 darwin-sandbox, 5 local.
INFO: Build completed successfully, 1571 total actions
```

いずれも約100秒程度かかっていました。では、キャッシュへの書き込みおよび参照を有効にしてみましょう。

```bash
cat << EOF >> .bazelrc
# リモートキャッシュのエンドポイント・BASIC認証のユーザおよびパスワードを指定
build --remote_cache=grpc://localhost:9092
# リモートキャッシュに書き込みを許可
build --remote_upload_local_results=true
EOF
bazel clean
bazel build //...
bazel clean
bazel build //...
```

```text
❯ bazel clean      
INFO: Invocation ID: 5be61dda-2cab-4fba-9bee-5eaa3d04f54a
INFO: Starting clean (this may take a while). Consider using --async if the clean takes more than several minutes.

❯ bazel build //...
INFO: Invocation ID: 2a45beaa-7c0e-46f7-b3a8-fc8a5791341c
INFO: Analyzed 50 targets (538 packages loaded, 31070 targets configured).
INFO: From Linking external/protobuf~/src/google/protobuf/io/libio_win32.a [for tool]:
warning: /Library/Developer/CommandLineTools/usr/bin/libtool: archive library: bazel-out/darwin_arm64-opt-exec-ST-d57f47055a04/bin/external/protobuf~/src/google/protobuf/io/libio_win32.a the table of contents is empty (no object file members in the library define global symbols)
INFO: From Linking external/protobuf~/protoc [for tool]:
ld: warning: ignoring duplicate libraries: '-lm', '-lpthread'
INFO: Found 50 targets...
INFO: Elapsed time: 103.990s, Critical Path: 58.60s
INFO: 1571 processes: 614 internal, 952 darwin-sandbox, 5 local.
INFO: Build completed successfully, 1571 total actions
```

1回目のビルドには100秒程度で同様でした。

```text
❯ bazel clean      
INFO: Invocation ID: 056e83d6-e347-451d-8de4-c6c3d3fc25bc
INFO: Starting clean (this may take a while). Consider using --async if the clean takes more than several minutes.

❯ bazel build //...
INFO: Invocation ID: ff2c8f31-8fca-40d2-b166-9bb19061c237
INFO: Analyzed 50 targets (538 packages loaded, 31070 targets configured).
INFO: From Linking external/protobuf~/src/google/protobuf/io/libio_win32.a [for tool]:
warning: /Library/Developer/CommandLineTools/usr/bin/libtool: archive library: bazel-out/darwin_arm64-opt-exec-ST-d57f47055a04/bin/external/protobuf~/src/google/protobuf/io/libio_win32.a the table of contents is empty (no object file members in the library define global symbols)
INFO: From Linking external/protobuf~/protoc [for tool]:
ld: warning: ignoring duplicate libraries: '-lm', '-lpthread'
INFO: Found 50 targets...
INFO: Elapsed time: 7.068s, Critical Path: 3.19s
INFO: 1571 processes: 957 remote cache hit, 614 internal.
INFO: Build completed successfully, 1571 total actions
```

一方、2回目のビルドは以下の様に7秒で完了しました。これがリモートキャッシュによる効果と言えるでしょう。
このリモートキャッシュは開発者全員が参照するようにすれば、開発者全員がビルド結果を共有できます。全員のビルド時間の短縮に繋がるため、Bazelを採用する際にはリモートキャッシュの導入を検討することをお勧めします。

## Conclusion

Bazelにはビルドした結果をキャッシュして外部に保存するリモートキャッシュ機能があります。
これは複数の開発者から参照可能で、CIのみならずローカルの開発におけるビルド時間の短縮にも繋がります。
リモートキャッシュの実装としてはGoogle Cloud Storageを使う方法が最も簡単です。
bazel-remoteを使うことでAWS S3やその互換実装、Azure Blob Storage、ローカルストレージを使うこともできます。
