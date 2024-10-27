---
title: Protocol BuffersとgRPCを扱う
---

[gRPC](https://grpc.io/) とは、Googleが開発したRPCフレームワークで、そのシリアライゼーションフォーマットとして[Protocol Buffers](https://protobuf.dev/)がよく用いられます。

Protocol Buffersはデータの構造を定義するためのIDL（Interface Definition Language）です。このIDLをコンパイルすることで各言語向けのデータ構造を生成でき、gRPCのスキーマの定義にも用いられます。

## Setup toolchains_protoc

Protocol Buffersのリポジトリを直接依存に加えると、少し変更がある度にprotobufのソースからの再ビルドが発生し、これが非常に遅いという問題があります。
Bazel 7.0から実装された `--incompatible_enable_proto_toolchain_resolution` と [toolchains_protoc](https://github.com/aspect-build/toolchains_protoc)により、コンパイル済みのバイナリを使ってビルド出来るようになります。

`MODULE.bazel` に以下の内容を追記します。

```python:MODULE.bazel
bazel_dep(name = "toolchains_protoc", version = "0.3.3")
protoc = use_extension("@toolchains_protoc//protoc:extensions.bzl", "protoc")
protoc.toolchain(
    google_protobuf = "com_google_protobuf",
    version = "v28.0",
)
use_repo(protoc, "com_google_protobuf", "toolchains_protoc_hub")
register_toolchains("@toolchains_protoc_hub//:all")
```

`.bazelrc` に以下の内容を追記します。

```sh:.bazelrc
common --incompatible_enable_proto_toolchain_resolution
```

## Add proto files

`proto` ディレクトリ以下にprotoファイルを置くこととし、以下の様なprotoファイルを作成します。

```sh
mkdir -p proto/hello/v1
```

```proto:proto/hello/v1/hello_request.proto
syntax = "proto3";

option go_package = "github.com/pddg/go-bazel-playground/proto/hello/v1";

package pddg.hello.v1;

message HelloRequest {
  string name = 1;
}
```

```proto:proto/hello/v1/hello_response.proto
syntax = "proto3";

option go_package = "github.com/pddg/go-bazel-playground/proto/hello/v1";

package pddg.hello.v1;

message HelloResponse {
  string message = 1;
}
```

```proto:proto/hello/v1/greeter.proto
syntax = "proto3";

option go_package = "github.com/pddg/go-bazel-playground/proto/hello/v1";

package pddg.hello.v1;

import "proto/hello/v1/hello_request.proto";
import "proto/hello/v1/hello_response.proto";

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloResponse) {}
}
```

`go mod tidy` が実行できるよう、空のファイルを1つ置いておきます。

::: message
これを置かない場合、 `go mod tidy` はimportしようとしたパッケージが見つからずエラーを返してしまいます。
:::

```sh
cat << 'EOF' > proto/hello/v1/_empty.go
package v1

# DO NOT EDIT. required for `go mod tidy`
EOF
```

gazelleにいくつかのオプションを渡すため、 `BUILD.bazel` の `gazelle` ターゲットを以下のように修正します。

```diff:BUILD.bazel
diff --git a/BUILD.bazel b/BUILD.bazel
index 7750e78..c0ddc0b 100644
--- a/BUILD.bazel
+++ b/BUILD.bazel
@@ -4,6 +4,10 @@ load("@rules_go//go:def.bzl", "TOOLS_NOGO", "go_library", "nogo")
 # このprefixには、このリポジトリ自体のパスを指定する
 
 # gazelle:prefix github.com/pddg/go-bazel-playground
+# gazelle:exclude _empty.go
+# gazelle:proto file
+# gazelle:go_grpc_compilers @rules_go//proto:go_grpc
+# gazelle:go_proto_compilers @rules_go//proto:go_proto
 gazelle(name = "gazelle")
 
 gazelle(
```

- `gazelle:exclude _empty.go`: `_empty.go` を対象とする `go_library` が生成されないよう無視させる
- `gazelle:proto file`: 1つのprotoファイルに対して1つの `proto_library` が生成されるようにする
- `gazelle:go_grpc_compilers @rules_go//proto:go_grpc`: 指定しない場合、後方互換性のため `@io_bazel_rules_go//` というプレフィックスで指定されてしまいエラーになる。
- `gazelle:go_proto_compilers @rules_go//proto:go_proto`: 指定しない場合、後方互換性のため `@io_bazel_rules_go//` というプレフィックスで指定されてしまいエラーになる。

gazelleでビルドファイルを生成します。

```sh
bazel run //:gazelle
```

以下の様なBUILDファイルが生成されるはずです。

```python:pddg/hello/v1/BUILD.bazel
load("@rules_go//proto:def.bzl", "go_proto_library")
load("@rules_proto//proto:defs.bzl", "proto_library")

proto_library(
    name = "greeter_proto",
    srcs = ["greeter.proto"],
    visibility = ["//visibility:public"],
    deps = [
        ":hello_request_proto",
        ":hello_response_proto",
    ],
)

proto_library(
    name = "hello_request_proto",
    srcs = ["hello_request.proto"],
    visibility = ["//visibility:public"],
)

proto_library(
    name = "hello_response_proto",
    srcs = ["hello_response.proto"],
    visibility = ["//visibility:public"],
)

go_proto_library(
    name = "v1_go_proto",
    compilers = ["@rules_go//proto:go_grpc"],
    importpath = "github.com/pddg/go-bazel-playground/proto/hello/v1",
    protos = [
        ":greeter_proto",
        ":hello_request_proto",
        ":hello_response_proto",
    ],
    visibility = ["//visibility:public"],
)
```

## Implement gRPC server

先ほど定義したAPIを実装します。

```sh
mkdir -p apps/greeter/server
```

```go:apps/greeter/server/greeter.go
package server

import (
	"context"

	hellov1 "github.com/pddg/go-bazel-playground/proto/hello/v1"
)

// GreeterServer is the server API for Greeter service.
type GreeterServer struct {
	hellov1.UnimplementedGreeterServer
}

// NewGreeterServer creates a new GreeterServer.
func NewGreeterServer() *GreeterServer {
	return &GreeterServer{}
}

// SayHello implements GreeterServer
func (s *GreeterServer) SayHello(ctx context.Context, in *hellov1.HelloRequest) (*hellov1.HelloResponse, error) {
	return &hellov1.HelloResponse{
		Message: "Hello, " + in.Name,
	}, nil
}
```

このサーバを実際にサーブする実装と、そのサーバを叩くクライアントを実装します。まずはサーバから。

```sh
mkdir -p apps/greeter/cmd/server
```

```go:apps/greeter/cmd/server/main.go
package main

import (
	"context"
	"fmt"
	"net"
	"os"
	"os/signal"

	"google.golang.org/grpc"

	greeterserver "github.com/pddg/go-bazel-playground/apps/greeter/server"
	hellov1 "github.com/pddg/go-bazel-playground/proto/hello/v1"
)

func main() {
	server := grpc.NewServer()
	hellov1.RegisterGreeterServer(server, greeterserver.NewGreeterServer())

	listener, err := net.Listen("tcp", ":8080")
	if err != nil {
		fmt.Fprintf(os.Stderr, "failed to listen: %v\n", err)
	}

	ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt)
	defer cancel()
	go func() {
    if err := server.Serve(listener); err != nil {
      fmt.Fprintf(os.Stderr, "failed to serve: %v\n", err)
    }
	}()
  <-ctx.Done()
  server.GracefulStop()
}
```

次にクライアントを実装します。

```sh
mkdir -p apps/greeter/cmd/client
```

```go:apps/greeter/cmd/client/main.go
package main

import (
	"context"
	"flag"
	"fmt"
	"os"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"

	hellov1 "github.com/pddg/go-bazel-playground/proto/hello/v1"
)

func main() {
	serverAddr := flag.String("server", "localhost:8080", "server address")
	flag.Parse()

	name := flag.Arg(0)

	client, err := grpc.NewClient(
		*serverAddr,
		grpc.WithTransportCredentials(insecure.NewCredentials()),
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "failed to create client: %v\n", err)
	}
	res, err := hellov1.NewGreeterClient(client).SayHello(context.Background(), &hellov1.HelloRequest{
		Name: name,
	})
	if err != nil {
		fmt.Fprintf(os.Stderr, "failed to call SayHello: %v\n", err)
	}
	fmt.Println(res.Message)
}
```

`google.golang.org/grpc` はまだgo.modに追加されていないので、`go mod tidy` を実行して依存関係を解決し、gazelleでビルドファイルを生成します。

```sh
bazel run @rules_go//go -- mod tidy
bazel run //:update-go-repos
bazel run //:gazelle
```

これにより、greeterサーバとクライアントのビルドファイルが生成され、ビルドできるようになるはずです。

```sh
bazel build //apps/greeter/...
```

実際にserverとclientを起動し、通信してみましょう。

```sh
bazel run //apps/greeter/cmd/server

# in another terminal
bazel run //apps/greeter/cmd/client -- world
```

```text
❯ bazel run //apps/greeter/cmd/greeter-client -- world
INFO: Analyzed target //apps/greeter/cmd/greeter-client:greeter-client (1 packages loaded, 3 targets configured).
INFO: Found 1 target...
Target //apps/greeter/cmd/greeter-client:greeter-client up-to-date:
  bazel-bin/apps/greeter/cmd/greeter-client/greeter-client_/greeter-client
INFO: Elapsed time: 0.435s, Critical Path: 0.27s
INFO: 10 processes: 4 internal, 6 darwin-sandbox.
INFO: Build completed successfully, 10 total actions
INFO: Running command line: bazel-bin/apps/greeter/cmd/greeter-client/greeter-client_/greeter-client user
Hello, world
```

## Build protobuf from source

toolchains_protocを使わずに、Protocol Buffersのリポジトリを直接依存に加える方法のほうが実はスタンダードです。

@[card](https://registry.bazel.build/modules/protobuf)

`MODULE.bazel` を以下の様に修正します。

```diff:MODULE.bazel
diff --git a/MODULE.bazel b/MODULE.bazel
index ac9477a..d4ef002 100644
--- a/MODULE.bazel
+++ b/MODULE.bazel
@@ -52,11 +52,4 @@ use_repo(apt, "bookworm")
 
 bazel_dep(name = "toolchains_protoc", version = "0.3.3")
 
-protoc = use_extension("@toolchains_protoc//protoc:extensions.bzl", "protoc")
-protoc.toolchain(
-    google_protobuf = "com_google_protobuf",
-    version = "v28.0",
-)
-use_repo(protoc, "com_google_protobuf", "toolchains_protoc_hub")
-
-register_toolchains("@toolchains_protoc_hub//:all")
+bazel_dep(name = "protobuf", version = "28.2")
```

`.bazelrc`から不要になったオプションを削除します。

```diff:.bazelrc
diff --git a/.bazelrc b/.bazelrc
index 3c72c74..a034735 100644
--- a/.bazelrc
+++ b/.bazelrc
@@ -2,4 +2,3 @@ test --test_output=errors --test_verbose_timeout_warnings
 # Use credential helper to push image
 common --credential_helper=ghcr.io=%workspace%/build_tools/auth/docker-credential-gh
 
-common --incompatible_enable_proto_toolchain_resolution
```

:::message
Bazel 8.0からは `proto_library` ルールについて、Bazel組み込みおよび `rules_proto` からの提供がdeprecateされ、protobufのリポジトリに移管されることが決まっています。  
<https://bazel.build/about/roadmap#migration-android,>

先んじてrules_protoのリポジトリではdeprecateが宣言されています。
<https://github.com/bazelbuild/rules_proto>

今後のBazelのリリースでは、protobufを扱う方法が変わっていく可能性が高いのですが、具体的にどうなるのかの議論を筆者は追えていません。
おそらく先にGazelleの生成するビルドファイルが問題となるため、今後のリリースに注目しています。
:::

## Conclusion

Bazelを用いてprotoファイルをコンパイルし、gRPCサーバとそのクライアントを実装できます。
Protocを取り扱うためには、以下の2つの方法がありました。

- `--incompatible_enable_proto_toolchain_resolution` を有効にし、 `toolchains_protoc` を使ってコンパイル済みのバイナリを使う
- Protocol Buffersのリポジトリを直接依存に加え、ソースからビルドする

今後Bazel本体やrules_protoからのprotobuf関連のルール提供はdeprecateされ、protobufのリポジトリに移管されることが決まっています。今後の動向次第では、これらの方法は変わる可能性があります。
