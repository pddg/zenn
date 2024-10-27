---
title: "Go Modulesを使う"
---

## Third-party dependencies

GoはGo 1.11から[Go Modules](https://go.dev/doc/modules/managing-dependencies)という依存関係管理機能が標準でサポートされています。ほとんどのGoプロジェクトでは、なんらかのサードパーティの依存関係をこのGo Modulesを使って管理しているでしょう。
Bazelとrules_goをこのGo Modulesと組み合わせるためには、もちろん手動でリポジトリ内に配置してBUILD.bazelを書くこともできます。しかし、これは非常に面倒です。
go.modから自動で外部依存を解析しBazelのエコシステムと統合するツールがあるため、それを導入します。

## Setup Gazelle

[Gazelle](https://github.com/bazelbuild/bazel-gazelle)はGoの依存関係を解析して、Bazelのビルドファイルを生成するツールです。
`MODULE.bazel`に以下を追記します。

```python
bazel_dep(name = "gazelle", version = "0.39.1", repo_name = "gazelle")

go_deps = use_extension("@gazelle//:extensions.bzl", "go_deps")
go_deps.from_file(go_mod = "//:go.mod")
```

bazelを通してgazelleを実行するのですが、簡単に実行できるようにするためリポジトリのルートに`BUILD.bazel`を作成し、以下の内容を記述します。

```python:BUILD.bazel
load("@gazelle//:def.bzl", "gazelle")

# gazelle:prefix github.com/pddg/go-bazel-playground
gazelle(name = "gazelle")
```

`gazelle:prefix` にはgo.modで指定したモジュールパスを指定します。

これで`bazel run //:gazelle`を実行してみます。

```text
❯ bazel run //:gazelle
INFO: Analyzed target //:gazelle (50 packages loaded, 13768 targets configured).
INFO: Found 1 target...
Target //:gazelle up-to-date:
  bazel-bin/gazelle-runner.bash
  bazel-bin/gazelle
INFO: Elapsed time: 1.147s, Critical Path: 0.03s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
INFO: Running command line: bazel-bin/gazelle
```

すると、リポジトリのルートに`deps.bzl`が生成されます。現時点ではサードパーティの依存関係はないので、以下の内容になります。

```python:deps.bzl
def go_repos():
    pass
```

## Generate BUILD.bazel automatically

また、gazelleは自動でGoプロジェクトの依存関係を解析して`BUILD.bazel`を生成します。
先ほどのgazelleの実行により、既存の `apps/hello_world/BUILD.bazel` が以下の様に更新されているはずです。

```diff
diff --git a/apps/hello_world/BUILD.bazel b/apps/hello_world/BUILD.bazel
index 6d825be..0256c1e 100644
--- a/apps/hello_world/BUILD.bazel
+++ b/apps/hello_world/BUILD.bazel
@@ -1,7 +1,14 @@
-load("@rules_go//go:def.bzl", "go_binary")
+load("@rules_go//go:def.bzl", "go_binary", "go_library")
 
 go_binary(
     name = "hello_world",
-    srcs = ["main.go"],
+    embed = [":hello_world_lib"],
     visibility = ["//visibility:public"],
 )
+
+go_library(
+    name = "hello_world_lib",
+    srcs = ["main.go"],
+    importpath = "github.com/pddg/go-bazel-playground/apps/hello_world",
+    visibility = ["//visibility:private"],
+)
```

このようにgazelleは面倒な `BUILD.bazel` の記述を自動で行ってくれます。

## Install third-party dependencies

今回はHello Worldアプリケーションに、uuidの出力を追加します（意味は特にありません）。
uuidのライブラリは `github.com/google/uuid` を使います。`go get` コマンドでインストールします。

```bash
bazel run @rules_go//go -- get github.com/google/uuid
```

次に、hello_worldアプリケーションにuuidを出力するコードを追加します。

```go:apps/hello_world/main.go
package main

import (
	"fmt"

	"github.com/google/uuid"
)

func main() {
	uuidStr := uuid.NewString()
	fmt.Printf("Hello, World!(%s)\n", uuidStr)
}
```

新しいモジュールを追加したため、gazelleにその依存関係を認識させる必要があります。以下のコマンドを実行します。

```bash
# go.modとgo.sumを更新
bazel run @rules_go//go -- mod tidy
# ↑を元にgazelleに依存関係を認識させる
bazel run //:gazelle -- update-repos -from_file=go.mod -to_macro=deps.bzl%go_repos -prune -bzlmod
```

次に、`BUILD.bazel`を更新します。

```bash
bazel run //:gazelle
```

これで、`apps/hello_world/BUILD.bazel`が以下の様に更新されるはずです。

```diff
diff --git a/apps/hello_world/BUILD.bazel b/apps/hello_world/BUILD.bazel
index 0256c1e..5572835 100644
--- a/apps/hello_world/BUILD.bazel
+++ b/apps/hello_world/BUILD.bazel
@@ -11,4 +11,5 @@ go_library(
     srcs = ["main.go"],
     importpath = "github.com/pddg/go-bazel-playground/apps/hello_world",
     visibility = ["//visibility:private"],
+    deps = ["@com_github_google_uuid//:uuid"],
 )
```

gazelleの長い引数を覚えるのは面倒なので、`BUILD.bazel`に以下の内容を追記しておきます。

```python:BUILD.bazel
gazelle(
    name = "update-go-repos",
    args = [
        "-from_file=go.mod",
        "-to_macro=deps.bzl%go_repos",
        "-bzlmod",
        "-prune",
    ],
    command = "update-repos",
)
```

これで、`bazel run //:update-go-repos`を実行するだけでgazelleによるサードパーティの依存モジュールを解決できます。

## Build and Run

`bazel run //apps/hello_world:hello_world`を実行してみます。

```bash
❯ bazel run //apps/hello_world:hello_world
INFO: Analyzed target //apps/hello_world:hello_world (2 packages loaded, 21 targets configured).
INFO: Found 1 target...
Target //apps/hello_world:hello_world up-to-date:
  bazel-bin/apps/hello_world/hello_world_/hello_world
INFO: Elapsed time: 0.984s, Critical Path: 0.24s
INFO: 5 processes: 2 internal, 3 linux-sandbox.
INFO: Build completed successfully, 5 total actions
INFO: Running command line: bazel-bin/apps/hello_world/hello_world_/hello_world
Hello, World!(42b29365-410c-4ce4-b1c2-d6973382d36f)
```

`Hello, World!`の後にuuidが出力されていることが確認できます。

## Conclusion

Gazelleを使うことで自動的にGoの依存関係を解析し、Bazelのビルドファイルを生成できます。
サードパーティの依存を追加する際は、Goで通常サードパーティモジュールを追加する手順に加えて、Gazelleを使ってサードパーティモジュールの依存を解決しビルドファイルを生成します。

```bash
bazel run @rules_go//go -- get ${MODULE}
bazel run @rules_go//go -- mod tidy
bazel run //:update-go-repos
bazel run //:gazelle
```
