---
title: "内部モジュールを作成する"
---

## Create a new package

まず、リポジトリのルートに `internal` ディレクトリを作成し、その中に新しいパッケージ `reverse` を作成します。特にディレクトリ名やモジュール名に規約があるわけではなく、通常のGoプロジェクトと同様に命名できます。

```bash
mkdir -p internal/reverse
```

このパッケージには与えられた文字列を逆順にする関数を実装します（特に意味はありません）。

```go:internal/reverse/string.go
package reverse

// String returns the reversed string.
func String(given string) string {
	runes := []rune(given)
	for i, j := 0, len(runes)-1; i < len(runes)/2; i, j = i+1, j-1 {
		runes[i], runes[j] = runes[j], runes[i]
	}
	return string(runes)
}
```

Ref: <https://stackoverflow.com/questions/1752414/how-to-reverse-a-string-in-go>

## Use the new package

hello_worldアプリケーションで `reverse` パッケージを使用してみます。

```go:apps/hello_world/main.go
package main

import (
	"fmt"

	"github.com/google/uuid"
	"github.com/pddg/go-bazel-playground/internal/reverse"
)

func main() {
	uuidStr := uuid.NewString()
	fmt.Printf("Hello, World!(%s)\n", uuidStr)
	fmt.Printf("Reversed: %s\n", reverse.String("Hello, World!"))
}
```

このままではビルドエラーが発生します。これは `reverse` パッケージが `BUILD.bazel` に記述されていないためです。
gazelleを使って `BUILD.bazel` を更新すると、この依存関係が追記されます。

```bash
bazel run //:gazelle
```

これにより `internal/reverse/BUILD.bazel` が作成され、以下の内容が記述されます。

```python:internal/reverse/BUILD.bazel
load("@rules_go//go:def.bzl", "go_library")

go_library(
    name = "reverse",
    srcs = ["string.go"],
    importpath = "github.com/pddg/go-bazel-playground/internal/reverse",
    visibility = ["//:__subpackages__"],
)
```

また、`apps/hello_world/BUILD.bazel` が以下の様に更新されます。

```diff
diff --git a/apps/hello_world/BUILD.bazel b/apps/hello_world/BUILD.bazel
index 5572835..0d70d85 100644
--- a/apps/hello_world/BUILD.bazel
+++ b/apps/hello_world/BUILD.bazel
@@ -11,5 +11,8 @@ go_library(
     srcs = ["main.go"],
     importpath = "github.com/pddg/go-bazel-playground/apps/hello_world",
     visibility = ["//visibility:private"],
-    deps = ["@com_github_google_uuid//:uuid"],
+    deps = [
+        "//internal/reverse",
+        "@com_github_google_uuid//:uuid",
+    ],
 )
```

## Build and run

ビルドして実行します。

```bash
bazel run //apps/hello_world:hello_world
```

```text
❯ bazel run //apps/hello_world:hello_world
INFO: Analyzed target //apps/hello_world:hello_world (2 packages loaded, 5 targets configured).
INFO: Found 1 target...
Target //apps/hello_world:hello_world up-to-date:
  bazel-bin/apps/hello_world/hello_world_/hello_world
INFO: Elapsed time: 0.328s, Critical Path: 0.14s
INFO: 5 processes: 2 internal, 3 linux-sandbox.
INFO: Build completed successfully, 5 total actions
INFO: Running command line: bazel-bin/apps/hello_world/hello_world_/hello_world
Hello, World!(dd051d1d-aa99-4318-9dbd-9b262b8202b1)
Reversed: !dlroW ,olleH
```

正しく実行できました。

## Tips: visibility

`reverse` モジュールの `go_library` ターゲットには `visibility = ["//:__subpackages__"]` という記述がありました。これはBazelに対してこのターゲットがどの範囲に公開されているかを示すものです[^internal]。 `//:__subpackages__` は同一リポジトリ内の全てのパッケージから参照できることを意味します。

[^internal]: そもそも `internal` 以下のパッケージはGoでは外部のモジュールから参照できません（[ref](https://go.dev/doc/modules/layout#package-or-command-with-supporting-packages)）が、bazelはこれについてより柔軟な仕組みを持っています。

同一リポジトリ内の特定のパッケージにだけ公開するといった設定も可能で、`apps/hello_world` パッケージにだけ公開するには以下のように記述します。

```python:internal/reverse/BUILD.bazel
go_library(
    name = "reverse",
    srcs = ["string.go"],
    importpath = "github.com/pddg/go-bazel-playground/internal/reverse",
    visibility = ["//apps/hello_world:__pkg__"],
)
```

これにより、`apps/hello_world` パッケージからのみ `internal/reverse` パッケージを参照できるようになり、他のパッケージから参照しようとするとエラーになります。

```bash
mkdir -p apps/sample
cat <<EOF > apps/sample/main.go
package main

import (
	"fmt"

	"github.com/pddg/go-bazel-playground/internal/reverse"
)

func main() {
	fmt.Println(reverse.String("Hello, World!"))
}
EOF
bazel run //:gazelle
```

```text
❯ bazel build //apps/sample:sample
ERROR: /home/pudding/ghq/github.com/pddg/go-bazel-playground/apps/sample/BUILD.bazel:3:11: in go_library rule //apps/sample:sample_lib: target '//internal/reverse:reverse' is not visible from target '//apps/sample:sample_lib'. Check the visibility declaration of the former target if you think the dependency is legitimate
ERROR: /home/pudding/ghq/github.com/pddg/go-bazel-playground/apps/sample/BUILD.bazel:3:11: Analysis of target '//apps/sample:sample_lib' failed
ERROR: Analysis of target '//apps/sample:sample' failed; build aborted: Analysis failed
INFO: Elapsed time: 0.153s, Critical Path: 0.00s
INFO: 1 process: 1 internal.
ERROR: Build did NOT complete successfully
```

より詳細な情報は以下のリンクを参照してください。

@[card](https://bazel.build/concepts/visibility)

## Conclusion

内部モジュールの作成は通常のGoアプリケーションの開発と同様にコードを記述した後、gazelleでBUILDファイルを生成するだけで、Bazelのビルドに統合でき同リポジトリ内の他のパッケージから参照できるようになります。

このように作成したモジュールを参照する際には、BUILDファイルに記述された依存関係を追加する必要がありますが、これはgazelleによって自動で行われます。もし追加せずにビルドを行うと、モジュールを参照できずエラーが発生します。

また、visibilityを使ってパッケージの公開範囲を制限できます。これにより特定のパッケージからのみ参照を許可したりできます。
