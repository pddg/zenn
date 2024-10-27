---
title: Lintを実行する
---

Goといえば静的解析ツールが充実しており、様々な側面からコードを自動的にチェックできる仕組みが整えられています。bazelを使う場合でもこれらの静的解析ツールによる指摘を通して、コード全体の品質を維持できます。

## Run lint with nogo

rules_goには[nogo](https://github.com/bazel-contrib/rules_go/blob/v0.50.1/go/nogo.rst)という仕組みが用意されています。これは `golang.org/x/tools/go/analysis` パッケージを使用します。

`go vet` コマンドのヘルプに利用可能な解析器の一覧があるので見てみましょう。

```text
❯ bazel run @rules_go//go -- tool vet help
Starting local Bazel server and connecting to it...
INFO: Analyzed target @@rules_go~//go:go (103 packages loaded, 12207 targets configured).
INFO: Found 1 target...
Target @@rules_go~//go/tools/go_bin_runner:go_bin_runner up-to-date:
  bazel-bin/external/rules_go~/go/tools/go_bin_runner/bin/go
INFO: Elapsed time: 4.885s, Critical Path: 0.12s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
INFO: Running command line: bazel-bin/external/rules_go~/go/tools/go_bin_runner/bin/go tool vet help
vet is a tool for static analysis of Go programs.

vet examines Go source code and reports suspicious constructs,
such as Printf calls whose arguments do not align with the format
string. It uses heuristics that do not guarantee all reports are
genuine problems, but it can find errors not caught by the compilers.

Registered analyzers:

    appends      check for missing values after append
    asmdecl      report mismatches between assembly files and Go declarations
    assign       check for useless assignments
    atomic       check for common mistakes using the sync/atomic package
    bools        check for common mistakes involving boolean operators
    buildtag     check //go:build and // +build directives
    cgocall      detect some violations of the cgo pointer passing rules
    composites   check for unkeyed composite literals
    copylocks    check for locks erroneously passed by value
    defers       report common mistakes in defer statements
    directive    check Go toolchain directives such as //go:debug
    errorsas     report passing non-pointer or non-error values to errors.As
    framepointer report assembly that clobbers the frame pointer before saving it
    httpresponse check for mistakes using HTTP responses
    ifaceassert  detect impossible interface-to-interface type assertions
    loopclosure  check references to loop variables from within nested functions
    lostcancel   check cancel func returned by context.WithCancel is called
    nilfunc      check for useless comparisons between functions and nil
    printf       check consistency of Printf format strings and arguments
    shift        check for shifts that equal or exceed the width of the integer
    sigchanyzer  check for unbuffered channel of os.Signal
    slog         check for invalid structured logging calls
    stdmethods   check signature of methods of well-known interfaces
    stdversion   report uses of too-new standard library symbols
    stringintconv check for string(int) conversions
    structtag    check that struct field tags conform to reflect.StructTag.Get
    testinggoroutine report calls to (*testing.T).Fatal from goroutines started by a test
    tests        check for common mistaken usages of tests and examples
    timeformat   check for calls of (time.Time).Format or time.Parse with 2006-02-01
    unmarshal    report passing non-pointer or non-interface values to unmarshal
    unreachable  check for unreachable code
    unsafeptr    check for invalid conversions of uintptr to unsafe.Pointer
    unusedresult check for unused results of calls to some functions

# 以下略
```

まずは `BUILD.bazel` に `nogo` のルールを追加します。 `go vet` の提供する解析器を個別に指定できますが、rules_goの提供する `TOOLS_NOGO`（[ref](https://github.com/bazelbuild/rules_go/blob/9741b368beafbe8af173de66bf1ec649ce64c5b0/go/def.bzl#L81-L118)）という変数を使うことで全てを簡単に指定できます。

```python:BUILD.bazel
load("@rules_go//go:def.bzl", "nogo", "TOOLS_NOGO")

nogo(
    name = "my_nogo",
    deps = TOOLS_NOGO,
    visibility = ["//visibility:public"],
)
```

次に `MODULE.bazel` に今定義した `my_nogo` を使うように指定します。

```python:MODULE.bazel
go_sdk.nogo(
    nogo = "//:my_nogo",
    includes = [
        "//:__subpackages__",
    ],
)
```

これにより、ビルド時に自動で解析するようになります。

```text
❯ bazel build //...
INFO: Analyzed 7 targets (196 packages loaded, 17123 targets configured).
INFO: Found 7 targets...
INFO: Elapsed time: 3.134s, Critical Path: 1.52s
INFO: 84 processes: 15 internal, 69 linux-sandbox.
INFO: Build completed successfully, 84 total actions
```

ビルドが成功するということは違反が無かった事を示します。試しに違反するコードを書いてみましょう。

```diff:apps/hello_world/main.go
❯ git diff apps/hello_world/main.go                                          
diff --git a/apps/hello_world/main.go b/apps/hello_world/main.go
index 84a2eaf..8697f5d 100644
--- a/apps/hello_world/main.go
+++ b/apps/hello_world/main.go
@@ -10,5 +10,5 @@ import (
 func main() {
        uuidStr := uuid.NewString()
        fmt.Printf("Hello, World!(%s)\n", uuidStr)
-       fmt.Printf("Reversed: %s\n", reverse.String("Hello, World!"))
+       fmt.Printf("Reversed: %s\n", reverse.String("Hello, World!"), "dummy")
 }
```

再度ビルドを実行すると、失敗することがわかります[^dups]。

[^dups]: 2回エラーが出力されるのは、libraryとしてのビルド時と実行バイナリのビルド時の両方からエラーが出るためで、おそらく2回実行しているわけではない……と思います。

```text
❯ bazel build //apps/hello_world
INFO: Analyzed target //apps/hello_world:hello_world (0 packages loaded, 0 targets configured).
ERROR: /home/pudding/ghq/github.com/pddg/go-bazel-playground/apps/hello_world/BUILD.bazel:9:11: Validating nogo output for //apps/hello_world:hello_world_lib failed: (Exit 1): builder failed: error executing ValidateNogo command (from target //apps/hello_world:hello_world_lib) bazel-out/k8-opt-exec-ST-d57f47055a04/bin/external/rules_go~~go_sdk~go-bazel-playground__download_0/builder_reset/builder nogovalidation bazel-out/k8-fastbuild/bin/apps/hello_world/hello_world_lib.nogo ... (remaining 1 argument skipped)

Use --sandbox_debug to see verbose messages from the sandbox and retain the sandbox build root for debugging

nogo: errors found by nogo during build-time code analysis:
apps/hello_world/main.go:13:2: fmt.Printf call needs 1 arg but has 2 args (printf)

ERROR: /home/pudding/ghq/github.com/pddg/go-bazel-playground/apps/hello_world/BUILD.bazel:3:10: Validating nogo output for //apps/hello_world:hello_world failed: (Exit 1): builder failed: error executing ValidateNogo command (from target //apps/hello_world:hello_world) bazel-out/k8-opt-exec-ST-d57f47055a04/bin/external/rules_go~~go_sdk~go-bazel-playground__download_0/builder_reset/builder nogovalidation bazel-out/k8-fastbuild/bin/apps/hello_world/hello_world.nogo ... (remaining 1 argument skipped)

Use --sandbox_debug to see verbose messages from the sandbox and retain the sandbox build root for debugging

nogo: errors found by nogo during build-time code analysis:
apps/hello_world/main.go:13:2: fmt.Printf call needs 1 arg but has 2 args (printf)

Target //apps/hello_world:hello_world failed to build
Use --verbose_failures to see the command lines of failed build steps.
INFO: Elapsed time: 0.174s, Critical Path: 0.01s
INFO: 4 processes: 4 internal.
ERROR: Build did NOT complete successfully
```

## Use other linters

nogoは `Analyzer` という名前で [Analyzer](https://godoc.org/golang.org/x/tools/go/analysis#Analyzer) 型の変数を提供するパッケージと互換性があります。自前の解析器を作成して利用するだけでなく、サードパーティの解析器の利用も可能です。

ここではまず、[nilerr](https://github.com/gostaticanalysis/nilerr)という解析器を利用してみます。サードパーティの解析器を利用するためには、まずそのモジュールを依存関係に追加します。

```bash
bazel run @rules_go//go -- get github.com/gostaticanalysis/nilerr
cat << EOF > tools.go
package tools

import (
	_ "github.com/gostaticanalysis/nilerr"
)
EOF
bazel run @rules_go//go -- mod tidy
```

gazelleでこれらの変更を自動で反映させます。

```bash
bazel run //:update-go-repos
bazel run //:gazelle
```

`my_nogo` にnilerrを追加する。

```diff:BUILD.bazel
❯ git diff BUILD.bazel             
diff --git a/BUILD.bazel b/BUILD.bazel
index 7edba89..e489b5a 100644
--- a/BUILD.bazel
+++ b/BUILD.bazel
@@ -20,7 +20,9 @@ gazelle(
 nogo(
     name = "my_nogo",
     visibility = ["//visibility:public"],
-    deps = TOOLS_NOGO,
+    deps = TOOLS_NOGO + [
+        "@com_github_gostaticanalysis_nilerr//:nilerr",
+    ],
 )
```

ビルドを実行すると、nilerrによる解析も実行されるようになります。ここでは特に違反しないためビルドが通るはずです。

```text
❯ bazel build //apps/hello_world
INFO: Analyzed target //apps/hello_world:hello_world (1 packages loaded, 100 targets configured).
INFO: Found 1 target...
Target //apps/hello_world:hello_world up-to-date:
  bazel-bin/apps/hello_world/hello_world_/hello_world
INFO: Elapsed time: 0.327s, Critical Path: 0.00s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
```

次に、違反するコードを追加してみます。

```diff:apps/hello_world/main.go
diff --git a/apps/hello_world/main.go b/apps/hello_world/main.go
index 84a2eaf..92762a2 100644
--- a/apps/hello_world/main.go
+++ b/apps/hello_world/main.go
@@ -7,8 +7,17 @@ import (
        "github.com/pddg/go-bazel-playground/internal/reverse"
 )
 
+func getUUID() (string, error) {
+       u, err := uuid.NewRandom()
+       if err != nil {
+               // This violates the rule of nilerr
+               return "", nil
+       }
+       return u.String(), nil
+}
+
 func main() {
-       uuidStr := uuid.NewString()
+       uuidStr, _ := getUUID()
        fmt.Printf("Hello, World!(%s)\n", uuidStr)
        fmt.Printf("Reversed: %s\n", reverse.String("Hello, World!"))
 }
```

このコードは、エラーを握りつぶしてnilを返しており、これはnilerrによって違反として検出されるはずです。

```text
❯ bazel build //apps/hello_world:hello_world
INFO: Analyzed target //apps/hello_world:hello_world (0 packages loaded, 0 targets configured).
ERROR: /home/pudding/ghq/github.com/pddg/go-bazel-playground/apps/hello_world/BUILD.bazel:3:10: Validating nogo output for //apps/hello_world:hello_world failed: (Exit 1): builder failed: error executing ValidateNogo command (from target //apps/hello_world:hello_world) bazel-out/k8-opt-exec-ST-d57f47055a04/bin/external/rules_go~~go_sdk~go-bazel-playground__download_0/builder_reset/builder nogovalidation bazel-out/k8-fastbuild/bin/apps/hello_world/hello_world.nogo ... (remaining 1 argument skipped)

Use --sandbox_debug to see verbose messages from the sandbox and retain the sandbox build root for debugging

nogo: errors found by nogo during build-time code analysis:
apps/hello_world/main.go:14:3: error is not nil (line 11) but it returns nil (nilerr)

ERROR: /home/pudding/ghq/github.com/pddg/go-bazel-playground/apps/hello_world/BUILD.bazel:9:11: Validating nogo output for //apps/hello_world:hello_world_lib failed: (Exit 1): builder failed: error executing ValidateNogo command (from target //apps/hello_world:hello_world_lib) bazel-out/k8-opt-exec-ST-d57f47055a04/bin/external/rules_go~~go_sdk~go-bazel-playground__download_0/builder_reset/builder nogovalidation bazel-out/k8-fastbuild/bin/apps/hello_world/hello_world_lib.nogo ... (remaining 1 argument skipped)

Use --sandbox_debug to see verbose messages from the sandbox and retain the sandbox build root for debugging

nogo: errors found by nogo during build-time code analysis:
apps/hello_world/main.go:14:3: error is not nil (line 11) but it returns nil (nilerr)

Target //apps/hello_world:hello_world failed to build
Use --verbose_failures to see the command lines of failed build steps.
INFO: Elapsed time: 0.140s, Critical Path: 0.01s
INFO: 4 processes: 4 internal.
ERROR: Build did NOT complete successfully
```

## Use self-implemented linters

自前の解析器を作成するパターンもやってみましょう。実装には[analysis](https://pkg.go.dev/golang.org/x/tools/go/analysis)パッケージを使うので、まずはそれを含む `golang.org/x/tools` パッケージを依存関係に追加します。

```bash
bazel run @rules_go//go -- get golang.org/x/tools
```

解析器を実装します。今回はサンプルであり、何でも良いので全ての関数を探索して `sample` という名前の関数があったら違反として報告させます。

```go:analyzers/sample/sample.go
package sample

import (
	"go/ast"

	"golang.org/x/tools/go/analysis"
	"golang.org/x/tools/go/analysis/passes/inspect"
	"golang.org/x/tools/go/ast/inspector"
)

var Analyzer = &analysis.Analyzer{
	Name: "sample",
	Doc:  "sample analyzer",
	Run:  run,
	Requires: []*analysis.Analyzer{
		inspect.Analyzer,
	},
}

func run(pass *analysis.Pass) (interface{}, error) {
	i := pass.ResultOf[inspect.Analyzer].(*inspector.Inspector)

	//  関数の定義定義のみを取得
	filter := []ast.Node{
		(*ast.FuncDecl)(nil),
	}
	i.Preorder(filter, func(n ast.Node) {
		fn := n.(*ast.FuncDecl)
		// 関数名がsampleの場合に警告を出す
		if fn.Name.Name == "sample" {
			pass.Reportf(fn.Pos(), "sample function found")
		}
	})
	return nil, nil
}
```

`go mod tidy` した後、gazelleに依存を更新・BUILD.bazelを生成させます。

```bash
bazel run @rules_go//go -- mod tidy
bazel run //:update-go-repos
bazel run //:gazelle
```

`my_nogo` に `sample` を追加してみましょう。

```diff:BUILD.bazel
diff --git a/BUILD.bazel b/BUILD.bazel
index e489b5a..5bacb0a 100644
--- a/BUILD.bazel
+++ b/BUILD.bazel
@@ -22,6 +22,7 @@ nogo(
     visibility = ["//visibility:public"],
     deps = TOOLS_NOGO + [
         "@com_github_gostaticanalysis_nilerr//:nilerr",
+        "//analyzers/sample:sample",
     ],
 )
```

ビルドを実行するとsampleによって解析されるようになりますが、現状sampleという関数はないため失敗しません。

```text
❯ bazel build //apps/hello_world:hello_world
INFO: Analyzed target //apps/hello_world:hello_world (3 packages loaded, 119 targets configured).
INFO: Found 1 target...
Target //apps/hello_world:hello_world up-to-date:
  bazel-bin/apps/hello_world/hello_world_/hello_world
INFO: Elapsed time: 1.423s, Critical Path: 1.23s
INFO: 62 processes: 2 internal, 60 linux-sandbox.
INFO: Build completed successfully, 62 total action
```

違反するコードを追加してみます。

```diff:apps/hello_world/main.go
diff --git a/apps/hello_world/main.go b/apps/hello_world/main.go
index 84a2eaf..77995ab 100644
--- a/apps/hello_world/main.go
+++ b/apps/hello_world/main.go
@@ -7,8 +7,13 @@ import (
        "github.com/pddg/go-bazel-playground/internal/reverse"
 )
 
+func sample() {
+       fmt.Println("sample")
+}
+
 func main() {
        uuidStr := uuid.NewString()
        fmt.Printf("Hello, World!(%s)\n", uuidStr)
        fmt.Printf("Reversed: %s\n", reverse.String("Hello, World!"))
+       sample()
 }
```

sample analyzerによる解析によって違反が検出され、ビルドが失敗します。

```text
❯ bazel build //apps/hello_world:hello_world
INFO: Analyzed target //apps/hello_world:hello_world (0 packages loaded, 0 targets configured).
ERROR: /home/pudding/ghq/github.com/pddg/go-bazel-playground/apps/hello_world/BUILD.bazel:9:11: Validating nogo output for //apps/hello_world:hello_world_lib failed: (Exit 1): builder failed: error executing ValidateNogo command (from target //apps/hello_world:hello_world_lib) bazel-out/k8-opt-exec-ST-d57f47055a04/bin/external/rules_go~~go_sdk~go-bazel-playground__download_0/builder_reset/builder nogovalidation bazel-out/k8-fastbuild/bin/apps/hello_world/hello_world_lib.nogo ... (remaining 1 argument skipped)

Use --sandbox_debug to see verbose messages from the sandbox and retain the sandbox build root for debugging

nogo: errors found by nogo during build-time code analysis:
apps/hello_world/main.go:10:1: sample function found (sample)

ERROR: /home/pudding/ghq/github.com/pddg/go-bazel-playground/apps/hello_world/BUILD.bazel:3:10: Validating nogo output for //apps/hello_world:hello_world failed: (Exit 1): builder failed: error executing ValidateNogo command (from target //apps/hello_world:hello_world) bazel-out/k8-opt-exec-ST-d57f47055a04/bin/external/rules_go~~go_sdk~go-bazel-playground__download_0/builder_reset/builder nogovalidation bazel-out/k8-fastbuild/bin/apps/hello_world/hello_world.nogo ... (remaining 1 argument skipped)

Use --sandbox_debug to see verbose messages from the sandbox and retain the sandbox build root for debugging

nogo: errors found by nogo during build-time code analysis:
apps/hello_world/main.go:10:1: sample function found (sample)

Target //apps/hello_world:hello_world failed to build
Use --verbose_failures to see the command lines of failed build steps.
INFO: Elapsed time: 0.170s, Critical Path: 0.03s
INFO: 6 processes: 4 internal, 2 linux-sandbox.
ERROR: Build did NOT complete successfully
```

## staticcheck and golangci-lint

Goプロジェクトでよく使われる有名な静的解析ツールに [staticcheck](https://staticcheck.io/) と [golangci-lint](https://github.com/golangci/golangci-lint) があります。これらのツールはnogoとダイレクトに統合することは現状出来ませんが、有志によって提供されているルールを使うことでBazelから実行可能です。

@[card](https://github.com/sluongng/nogo-analyzer)

::: message
nogo-analyzerは2024/10現在Bzlmodに対応していません。いくつかの課題が残っており、以下のIssueで進捗が確認できます。
@[card](https://github.com/sluongng/nogo-analyzer/issues/40)
:::

プロジェクトが初期の段階やコード量がそう多くない間は、BazelからLintを実行するのではなく、単にこれらのサードパーティのツールを使うだけで十分な場合もあります。
ただし、大規模なプロジェクトではgolangci-lintの実行は非常に遅くなるため、nogoによるBazelとのよりネイティブな連携が必要になるかもしれません。

## Conclusion

rules_goの提供するnogoを使うことで、ビルド時に静的解析を走らせ、ルールに違反するコードのビルドを失敗させることで、リポジトリ内のコード品質を維持する仕組みを作れます。
この静的解析はGoの準標準パッケージである `golang.org/x/tools/go/analysis` を使って実装します。
`go vet`の提供する解析器、サードパーティの解析器、自分で作成した解析器のいずれも利用できます。

staticcheckやgolangci-lintのような有名な静的解析ツールも使えますが、やや複雑な手順が必要です。Bazelと統合せずに直接それらを使うことも可能であるため、プロジェクトの規模や状況に応じて使い分けると良いでしょう。
