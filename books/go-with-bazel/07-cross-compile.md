---
title: "クロスビルドする"
---

Goは `CGO_ENABLED=0` なら容易にクロスビルドして、コンパイルするホストとは異なるOS・Arch向けにバイナリを生成できます。通常通りGoコマンドを使うだけなら、これは `GOOS` や `GOARCH` などの環境変数を操作することで実現できていました。
ここでは、bazelとrules_goを使ってどのようにクロスビルドするかを見ていきます。

## Specify platform via CLI

ビルド時にCLIから利用するツールチェインのプラットフォームを指定できます。

```sh
bazel build --platforms=@rules_go//go/toolchain:linux_arm64 //apps/hello_world
```

なお、当然アーキテクチャやOSが違えば `bazel run` による実行はできません。

```text
❯ bazel run --platforms=@rules_go//go/toolchain:windows_arm64 //apps/hello_world
INFO: Analyzed target //apps/hello_world:hello_world (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
Target //apps/hello_world:hello_world up-to-date:
  bazel-bin/apps/hello_world/hello_world_/hello_world.exe
INFO: Elapsed time: 0.209s, Critical Path: 0.00s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
INFO: Running command line: bazel-bin/apps/hello_world/hello_world_/hello_world.exe
/bin/bash: /private/var/tmp/_bazel_pddg/62cf4753b1a98e6c727224d46be86540/execroot/_main/bazel-out/darwin_arm64-fastbuild/bin/apps/hello_world/hello_world_/hello_world.exe: cannot execute binary file
```

このplatformというのはbazelがネイティブにサポートするクロスビルドの抽象化の仕組みです。詳しくは以下を参照してください。

@[card](https://bazel.build/extending/platforms)

## `go_cross_binary` rule

`go_binary` ルールはplatformの指定が無ければ実行したマシンのネイティブなプラットフォームでビルドするようになっています。
`BUILD.bazel` 上で宣言的に異なるプラットフォームへビルドするターゲットを生成するルールは `go_cross_binary` として実装されています。

```python:apps/hello_world/BUILD.bazel
load("@rules_go//go:def.bzl", "go_binary", "go_library", "go_cross_binary")

go_binary(
    name = "hello_world",
    embed = [":hello_world_lib"],
    visibility = ["//visibility:public"],
)

go_cross_binary(
    name = "hello_world_linux_amd64",
    platform = "@rules_go//go/toolchain:linux_amd64",
    target = ":hello_world",
)

go_cross_binary(
    name = "hello_world_linux_arm64",
    platform = "@rules_go//go/toolchain:linux_arm64",
    target = ":hello_world",
)

go_cross_binary(
    name = "hello_world_darwin_amd64",
    platform = "@rules_go//go/toolchain:darwin_amd64",
    target = ":hello_world",
)

go_cross_binary(
    name = "hello_world_darwin_arm64",
    platform = "@rules_go//go/toolchain:darwin_arm64",
    target = ":hello_world",
)

go_cross_binary(
    name = "hello_world_windows_amd64",
    platform = "@rules_go//go/toolchain:windows_amd64",
    target = ":hello_world",
)

go_cross_binary(
    name = "hello_world_windows_arm64",
    platform = "@rules_go//go/toolchain:windows_arm64",
    target = ":hello_world",
)
```

これでlinux・macOS・Windowsに対してそれぞれamd64およびarm64をターゲットとしたビルドルールが書けます。
なお、リスト内包表記を使うことで以下の様に記述できます。

```python:apps/hello_world/BUILD.bazel
[
    go_cross_binary(
        name = "hello_world_" + os + "_" + arch,
        platform = "@rules_go//go/toolchain:" + os + "_" + arch,
        target = ":hello_world",
    )
    for os in [
        "linux",
        "windows",
        "darwin",
    ]
    for arch in [
        "amd64",
        "arm64",
    ]
]
```

bazelではビルド対象を指定する際に `...` を使うことで、指定したディレクトリ以下の全てのビルドルールを評価できます[^recursive]。

[^recursive]: hello_world以下のビルドルールを全て評価するため、`go_cross_binary` だけでなく他のビルドルールも全て可能な限り並列にビルドされます。

```sh
bazel build //apps/hello_world/...
```

## Build constraints

Goに存在するBuild constraintsとは、Goのビルドタグやファイルの命名によって、OSやアーキテクチャごとに異なるファイルのビルドを指定できるという機能です[^gotags]。
<https://pkg.go.dev/cmd/go#hdr-Build_constraints>

[^gotags]: ビルドタグ自体は様々な使い方があるため、OSやアーキテクチャ依存の処理を記述するためだけに使うとは限りません。

以下の様にOSごとに異なる値を持つファイルを生成してみます。

```sh
for OSNAME in "darwin" "linux" "windows"; do
    cat << EOF > "apps/hello_world/os_${OSNAME}.go"
package main

const OsName = "${OSNAME}"
EOF
done
```

`main.go` からこの変数を呼び出してみましょう。

```diff:apps/hello_world/main.go
diff --git a/apps/hello_world/main.go b/apps/hello_world/main.go
index 84a2eaf..a18d207 100644
--- a/apps/hello_world/main.go
+++ b/apps/hello_world/main.go
@@ -11,4 +11,5 @@ func main() {
        uuidStr := uuid.NewString()
        fmt.Printf("Hello, World!(%s)\n", uuidStr)
        fmt.Printf("Reversed: %s\n", reverse.String("Hello, World!"))
+       fmt.Printf("OsName: %s\n", OsName)
 }
```

これによりどのターゲットに対してビルドされたかで表示される内容が変化します。Linux向けにビルドすれば `linux` が、Windowsなら `windows` が表示されるはずです。
これらの依存を認識させてgazelleにBUILDファイルを生成させてみましょう。

```sh
bazel run //:gazelle
```

生成されたファイルを見ると、`srcs` には全てのファイルが単純に列挙されています。これでは特定のOS向けのファイルを変更すると、それ以外のOSでも無駄にビルドが走ってしまいそうに見えます。

```diff:apps/hello_world/BUILD.bazel
 go_library(
     name = "hello_world_lib",
-    srcs = ["main.go"],
+    srcs = [
+        "main.go",
+        "os_darwin.go",
+        "os_linux.go",
+        "os_windows.go",
+    ],
     importpath = "github.com/pddg/go-bazel-playground/apps/hello_world",
     visibility = ["//visibility:private"],
     deps = [
@@ -24,3 +40,22 @@ go_library(
         "@com_github_google_uuid//:uuid",
     ],
 )
```

bazelの [select](https://bazel.build/docs/configurable-attributes) を使うことで異なるプラットフォーム向けには異なるファイルを指定できます。しかし、これはrules_goが内部でフィルタする処理[^filter]と重複しており、通常BUILDファイルレベルでこれを先に排除しておくことは冗長である、という理由でgazelleからは機能が削除されているようです。

[^filter]: rules_goの処理を読んでも自分は見つけられなかったので、Goコンパイラの挙動を指しているのかもしれません。

@[card](https://github.com/bazelbuild/bazel-gazelle/issues/205#issuecomment-391067841)

`bazel cquery` を使って依存を解析すると、本来無関係なはずの `hello_world_linux_amd64` のビルドの依存にも `os_windows.go` などが存在することがわかります。

```text
❯ bazel cquery 'filter("^//apps", kind("source file", deps(//apps/hello_world:hello_world_linux_amd64)))'
INFO: Analyzed target //apps/hello_world:hello_world_linux_amd64 (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
//apps/hello_world:os_darwin.go (null)
//apps/hello_world:os_linux.go (null)
//apps/hello_world:main.go (null)
//apps/hello_world:os_windows.go (null)
INFO: Elapsed time: 0.206s, Critical Path: 0.00s
INFO: 0 processes.
INFO: Build completed successfully, 0 total actions
```

一方、パッケージレベルの依存関係では異なるようです。

@[card](https://github.com/bazelbuild/rules_go/blob/v0.50.1/docs/go/core/platform-specific_dependencies.md)

試しに `internal/windows` パッケージを作り、そこに定義した値を `os_windows.go` でimportしてGazelleにBUILDファイルを生成させてみましょう。

```sh
mkdir -p internal/windows
cat << 'EOF' > internal/windows/windows.go
package windows

const OsName = "defined by windows package"
EOF
cat << 'EOF' > apps/hello_world/os_windows.go
package main

import (
	"github.com/pddg/go-bazel-playground/internal/windows"
)

const OsName = windows.OsName
EOF
bazel run //:gazelle
```

以下の様に `select` を使って依存が分けられます。

```diff:apps/hello_world/BUILD.bazel
 go_library(
     name = "hello_world_lib",
@@ -27,5 +38,30 @@ go_library(
     deps = [
         "//internal/reverse",
         "@com_github_google_uuid//:uuid",
-    ],
+    ] + select({
+        "@rules_go//go/platform:windows": [
+            "//internal/windows",
+        ],
+        "//conditions:default": [],
+    }),
 )
```

実際にビルド時の依存からも取り除かれます。cqueryで依存関係を解析すると、linux向けにはこの依存は含まれないことがわかります。

```text
❯ bazel cquery 'filter("^//internal", kind("go_library", deps(//apps/hello_world:hello_world_windows_amd64)))'             
INFO: Analyzed target //apps/hello_world:hello_world_windows_amd64 (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
//internal/reverse:reverse (96f18ec)
//internal/windows:windows (96f18ec)
INFO: Elapsed time: 0.173s, Critical Path: 0.00s
INFO: 0 processes.
INFO: Build completed successfully, 0 total actions

❯ bazel cquery 'filter("^//internal", kind("go_library", deps(//apps/hello_world:hello_world_linux_amd64)))'  
INFO: Analyzed target //apps/hello_world:hello_world_linux_amd64 (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
//internal/reverse:reverse (6014adc)
INFO: Elapsed time: 0.190s, Critical Path: 0.00s
INFO: 0 processes.
INFO: Build completed successfully, 0 total actions
```

:::message
bazelには `cquery` の他に `query` もあります。 `bazel query` は `select` を解決しないため、これによって解析される依存はもっとも広い範囲のものを示します。今回の場合、queryで解析すると異なる結果を示します。

```text
❯ bazel query 'filter("^//internal", kind("go_library", deps(//apps/hello_world:hello_world_linux_amd64)))' 
//internal/reverse:reverse
//internal/windows:windows
```

`cquery` はbazelによる解析が完了した後の状態に対するクエリを発行するため、platformによる `select` を解決した後の結果を返します。
@[card](https://bazel.build/query/cquery)
:::

従って `internal/windows` を編集しても `apps/hello_world:hello_world_linux_*` の出力には影響しません。Bazelの都合だけで言うと、このようにOS依存のものは別パッケージに隔離した方が効率的と言えるでしょう。

## CGO

CGOとは、Go言語からC言語の関数を呼び出すための仕組みです。

@[card](https://pkg.go.dev/cmd/cgo)

これまでのルールの記述ではCGOは暗黙的に無効化されていました。オプション無しで `go_binary` および `go_cross_bynary` を使った場合のデフォルトの挙動がそうなっています。

クロスコンパイルとCGOは一般的に困難であり、これはBazelでも例外ではありません。一応 `go_binary` ルールはCGOをちゃんとサポートしており様々なオプションを指定できます。

```python:apps/hello_world/BUILD.bazel
go_binary(
    name = "hello_world",
    embed = [":hello_world_lib"],
    visibility = ["//visibility:public"],
    # Enable CGO for thiss binary
    cgo = True,
    cdeps = [
        # Add your C libraries to link
    ],
    copts = select({
        "@rules_go//go/platform:darwin": [
            # Add C compiler option to compile the C target on macOS
        ],
        "@rules_go//go/platform:linux": [
            # ...
        ],
    }),
    cppopts = [
        # Add flags to the C/C++ preprocessor
    ],
    cxxopts = [
        # Add flags to the C++ compiler
    ],
    clinkopts = select({
        "@rules_go//go/platform:darwin": [
            # Add option for linker on macOS
        ],
    })
)

[
    go_cross_binary(
        name = "hello_world_" + os + "_" + arch,
        # Use *_cgo platform to enable CGO
        platform = "@rules_go//go/toolchain:" + os + "_" + arch + "_cgo",
        target = ":hello_world",
    )
    for os in [
        "linux",
        "windows",
        "darwin",
    ]
    for arch in [
        "amd64",
        "arm64",
    ]
]
```

さて、ここで問題なのはCコンパイラをどう用意するかです。ホスト上のCコンパイラを利用することはBazelの大きな利点の1つであるサンドボックス内での密閉されたビルドを破壊してしまうことがあります。
最近 [Zig](https://ziglang.org/) が登場したことにより、この扱いが少し容易になってきた……かもしれません。実際uberは[hermetic_cc_toolchain](https://github.com/uber/hermetic_cc_toolchain)というルールを作っているようです。

CGOを有効化したビルドのexampleも用意されているので見てみましょう。
<https://github.com/uber/hermetic_cc_toolchain/tree/v3.1.1/examples/bzlmod>

このexampleに以下の様に`go_cross_binary`ルールを追加してビルドすると（大量のwarningは出るものの）ビルドできました。

```python:BUILD.bazel
[
    go_cross_binary(
        name = "hello_world_" + os + "_" + arch,
        platform = "@io_bazel_rules_go//go/toolchain:" + os + "_" + arch + "_cgo",
        target = ":cgo",
    )
    for os in [
        "linux",
        "windows",
    ]
    for arch in [
        "amd64",
        "arm64",
    ]
]
```

darwin arm64なマシンからlinux amd64をターゲットとしたバイナリを生成し、どの自宅にもあるlinux amd64なサーバへ転送して実行してみたところ、正常に動作することを確認しました。

```text
❯ bazel build //...
❯ scp $(bazel cquery --output=files ':hello_world_linux_amd64') your_server:
❯ ssh your_server
(your_server) ~$ ./hello_world_linux_amd64
hello, world
```

ただし、darwinをターゲットに追加したところビルドできませんでした。このような簡単なexampleであっても、CGOとクロスコンパイルは困難であることがわかります。
筆者は幸いにしてCGOが必要なシチュエーションにはまだ遭遇しておらず、詳しくもないためこれ以上詳細な解説は控えます。

## Conclusion

Bazelにはplatformsと呼ばれる仕組みがあり、クロスビルドのための抽象化が存在します。
rules_goはこのplatformsに対応しており、ビルド時にこれを指定したり、宣言的に対象のplatformを指定したビルドルールを記述できます。
Gazelleが生成するビルドルールにおいて、プラットフォームごとの依存はソースファイルレベルでは分離されませんが、パッケージレベルでは分離されます。
CGOを用いたクロスビルドは非常に困難であり、また事例も多くありません。Zigなどの新たなツールの登場で今後の発展の余地があると思います。
