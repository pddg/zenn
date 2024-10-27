---
title: "ユニットテストを実行する"
---

## Create a test file

前回作成した `reverse` パッケージに対して、ユニットテストを実装してみましょう。

```go:internal/reverse/string_test.go
package reverse_test

import (
    "testing"

    "github.com/pddg/go-bazel-playground/internal/reverse"
)

func TestString(t *testing.T) {
    t.Parallel()
    tests := []struct {
        name     string
        given    string
        expected string
    }{
        {
            name:     "empty",
            given:    "",
            expected: "",
        },
        {
            name:     "single",
            given:    "a",
            expected: "a",
        },
        {
            name:     "multiple",
            given:    "abc",
            expected: "cba",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            actual := reverse.String(tt.given)
            if actual != tt.expected {
                t.Errorf("expected: %s, actual: %s", tt.expected, actual)
            }
        })
    }
}
```

## Run the test

Bazelではユニットテストもビルドと同様に抽象化されたルールを通して実行します。これによりBazelはテストの依存関係を解決し、必要なテストのみを実行できます。

rules_goでは `go_test` というルールが用意されています。Gazelleを使うことでテストファイルの依存関係を解決し、 `BUILD.bazel` に自動で追記できます。

```bash
bazel run //:gazelle
```

これにより、 `internal/reverse/BUILD.bazel` が以下の様に更新されるはずです。

```diff:internal/reverse/BUILD.bazel
diff --git a/internal/reverse/BUILD.bazel b/internal/reverse/BUILD.bazel
index dd7f598..43227d0 100644
--- a/internal/reverse/BUILD.bazel
+++ b/internal/reverse/BUILD.bazel
@@ -1,4 +1,4 @@
-load("@rules_go//go:def.bzl", "go_library")
+load("@rules_go//go:def.bzl", "go_library", "go_test")
 
 go_library(
     name = "reverse",
@@ -6,3 +6,9 @@ go_library(
     importpath = "github.com/pddg/go-bazel-playground/internal/reverse",
     visibility = ["//:__subpackages__"],
 )
+
+go_test(
+    name = "reverse_test",
+    srcs = ["string_test.go"],
+    deps = [":reverse"],
+)
```

`bazel test` コマンドでテストを実行できます。

```bash
bazel test //internal/reverse:reverse_test
```

```text
❯ bazel test //internal/reverse:reverse_test
INFO: Analyzed target //internal/reverse:reverse_test (4 packages loaded, 27 targets configured).
INFO: Found 1 test target...
Target //internal/reverse:reverse_test up-to-date:
  bazel-bin/internal/reverse/reverse_test_/reverse_test
INFO: Elapsed time: 0.582s, Critical Path: 0.32s
INFO: 12 processes: 4 internal, 8 linux-sandbox.
INFO: Build completed successfully, 12 total actions
//internal/reverse:reverse_test                                          PASSED in 0.0s

Executed 1 out of 1 test: 1 test passes.
There were tests whose specified size is too big. Use the --test_verbose_timeout_warnings command line option to see which ones these are.
```

テストが成功すると、二度目以降の実行はキャッシュされており実行されません。`go test` にもこのようなキャッシュの機能がありますが、Bazelは全ての言語に対してこのような機能を提供しています。

```text
❯ bazel test //internal/reverse:reverse_test
INFO: Analyzed target //internal/reverse:reverse_test (0 packages loaded, 0 targets configured).
INFO: Found 1 test target...
Target //internal/reverse:reverse_test up-to-date:
  bazel-bin/internal/reverse/reverse_test_/reverse_test
INFO: Elapsed time: 0.139s, Critical Path: 0.00s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
//internal/reverse:reverse_test                                 (cached) PASSED in 0.0s

Executed 0 out of 1 test: 1 test passes.
There were tests whose specified size is too big. Use the --test_verbose_timeout_warnings command line option to see which ones these are.
```

## Tips: テストサイズ

上記の実行結果にはWarningが表示されていました。表示されている通り、`--test_verbose_timeout_warnings` オプションを使うことで詳細な情報を表示できます。
このオプションはコマンドラインから指定しても良いのですが、`.bazelrc` ファイルに設定するとデフォルトで有効にできます。

```bash:.bazelrc
test --test_output=errors --test_verbose_timeout_warnings
```

この状態でテストを再実行してみます。

```text
❯ bazel test //internal/reverse:reverse_test
INFO: Analyzed target //internal/reverse:reverse_test (1 packages loaded, 4 targets configured).
INFO: Found 1 test target...
Target //internal/reverse:reverse_test up-to-date:
  bazel-bin/internal/reverse/reverse_test_/reverse_test
INFO: Elapsed time: 0.182s, Critical Path: 0.02s
INFO: 2 processes: 1 internal, 1 linux-sandbox.
INFO: Build completed successfully, 2 total actions
//internal/reverse:reverse_test                                          PASSED in 0.0s
  WARNING: //internal/reverse:reverse_test: Test execution time (0.0s excluding execution overhead) outside of range for MODERATE tests. Consider setting timeout="short" or size="small".

Executed 1 out of 1 test: 1 test passes.
```

このWarningはテストの実行時間が指定されたタイムアウト時間 `MODERATE` に対して小さすぎることを意味しています。

bazelではテストに対しその大きさを指定でき、以下の4つのカテゴリに分類されます。また、各テストサイズには暗黙的にタイムアウトが設定されています。

| size     | timeout label | timeout (sec) |
|----------|---------------|---------------|
| small    | short         | 60            |
| medium   | moderate      | 300           |
| large    | long          | 900           |
| enormous | eternal       | 3600          |

Ref: <https://bazel.build/reference/test-encyclopedia#role-test-runner>

デフォルトではタイムアウトに `MODERATE` が設定されているため、 `MODERATE` に対して小さすぎるテストはWarningが表示されます。

テストの大きさは `size` パラメータで指定できるため、gazelleが生成した `BUILD.bazel` に `size` パラメータを追記することでWarningを解消できます。

```diff:internal/reverse/BUILD.bazel
diff --git a/internal/reverse/BUILD.bazel b/internal/reverse/BUILD.bazel
index 43227d0..aab058e 100644
--- a/internal/reverse/BUILD.bazel
+++ b/internal/reverse/BUILD.bazel
@@ -9,6 +9,7 @@ go_library(
 
 go_test(
     name = "reverse_test",
+    size = "small",
     srcs = ["string_test.go"],
     deps = [":reverse"],
 )
```

これでWarningが表示されなくなります。

```text
❯ bazel test //internal/reverse:reverse_test
INFO: Analyzed target //internal/reverse:reverse_test (1 packages loaded, 4 targets configured).
INFO: Found 1 test target...
Target //internal/reverse:reverse_test up-to-date:
  bazel-bin/internal/reverse/reverse_test_/reverse_test
INFO: Elapsed time: 0.187s, Critical Path: 0.02s
INFO: 2 processes: 1 internal, 1 linux-sandbox.
INFO: Build completed successfully, 2 total actions
//internal/reverse:reverse_test                                          PASSED in 0.0s

Executed 1 out of 1 test: 1 test passes.
```

## テスト時にファイルを読み込む

Bazelではテストはサンドボックス内で行われるため、依存を明示しない限りテスト時にファイルを読み込むことはできません。例えば今回のテストで `testdata` ディレクトリにファイルを配置し、テスト時に読み込むようなテストを記述してみましょう[^unneed]。

[^unneed]: この例ではファイルを読み込む必要がないため、このような実装は本来不要です。

まずテストのinputとoutputを定義したファイルを作成します。

```bash
mkdir -p internal/reverse/testdata
echo -n "" > internal/reverse/testdata/empty_string_input.txt
echo -n "" > internal/reverse/testdata/empty_string_output.txt
echo -n "a" > internal/reverse/testdata/single_string_input.txt
echo -n "a" > internal/reverse/testdata/single_string_output.txt
echo -n "abc" > internal/reverse/testdata/multiple_string_input.txt
echo -n "cba" > internal/reverse/testdata/multiple_string_output.txt
```

これらを使ってテストを書き直してみましょう。

```go:internal/reverse/string_test.go
package reverse_test

import (
	"os"
	"testing"

	"github.com/pddg/go-bazel-playground/internal/reverse"
)

func TestString(t *testing.T) {
	t.Parallel()
	tests := []struct {
		name string
	}{
		{
			name: "empty",
		},
		{
			name: "single",
		},
		{
			name: "multiple",
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			// Setup
			givenBytes, err := os.ReadFile("testdata/" + tt.name + "_string_input.txt")
			if err != nil {
				t.Fatal(err)
			}
			expectedBytes, err := os.ReadFile("testdata/" + tt.name + "_string_output.txt")
			if err != nil {
				t.Fatal(err)
			}
			given := string(givenBytes)
			expected := string(expectedBytes)

			// Exercise
			actual := reverse.String(string(given))

			// Verify
			if actual != expected {
				t.Errorf("expected: %s, actual: %s", expected, actual)
			}
		})
	}
}
```

このテストを実行すると、ファイルを読み込めずにエラーが発生します。

```text
❯ bazel test //internal/reverse:reverse_test
INFO: Analyzed target //internal/reverse:reverse_test (0 packages loaded, 0 targets configured).
FAIL: //internal/reverse:reverse_test (see /home/pudding/.cache/bazel/_bazel_pudding/99730a75027f37c6494047af2b092bf0/execroot/_main/bazel-out/k8-fastbuild/testlogs/internal/reverse/reverse_test/test.log)
INFO: From Testing //internal/reverse:reverse_test:
==================== Test output for //internal/reverse:reverse_test:
--- FAIL: TestString (0.00s)
    --- FAIL: TestString/empty (0.00s)
        string_test.go:31: open testdata/empty_string_input.txt: no such file or directory
    --- FAIL: TestString/single (0.00s)
        string_test.go:31: open testdata/single_string_input.txt: no such file or directory
    --- FAIL: TestString/multiple (0.00s)
        string_test.go:31: open testdata/multiple_string_input.txt: no such file or directory
FAIL
================================================================================
INFO: Found 1 test target...
Target //internal/reverse:reverse_test up-to-date:
  bazel-bin/internal/reverse/reverse_test_/reverse_test
INFO: Elapsed time: 0.467s, Critical Path: 0.31s
INFO: 9 processes: 1 internal, 8 linux-sandbox.
INFO: Build completed, 1 test FAILED, 9 total actions
//internal/reverse:reverse_test                                          FAILED in 0.0s
  /home/pudding/.cache/bazel/_bazel_pudding/99730a75027f37c6494047af2b092bf0/execroot/_main/bazel-out/k8-fastbuild/testlogs/internal/reverse/reverse_test/test.log

Executed 1 out of 1 test: 1 fails locally.
```

このような外部のファイルを読み込むテストを実行する場合、`go_test` ルールの `data` パラメータで依存を明示することで、テスト実行時にこれらのファイルを持ち込めるようになります。

次のように `go_test` ルールの `data` パラメータにこのターゲットを追加します。なお、 `testdata` ディレクトリは慣習的に特別扱いされており、gazelleが自動的に追加してくれるため単に `bazel run //:gazelle` すれば追加されます。

```diff:internal/reverse/BUILD.bazel
diff --git a/internal/reverse/BUILD.bazel b/internal/reverse/BUILD.bazel
index aab058e..02eed9a 100644
--- a/internal/reverse/BUILD.bazel
+++ b/internal/reverse/BUILD.bazel
@@ -11,5 +11,6 @@ go_test(
     name = "reverse_test",
     size = "small",
     srcs = ["string_test.go"],
+    data = glob(["testdata/**"]),
     deps = [":reverse"],
 )
```

これによりテストが成功するようになります。

```text
❯ bazel test //internal/reverse:reverse_test
INFO: Analyzed target //internal/reverse:reverse_test (2 packages loaded, 11 targets configured).
INFO: Found 1 test target...
Target //internal/reverse:reverse_test up-to-date:
  bazel-bin/internal/reverse/reverse_test_/reverse_test
INFO: Elapsed time: 0.204s, Critical Path: 0.03s
INFO: 5 processes: 4 internal, 1 linux-sandbox.
INFO: Build completed successfully, 5 total actions
//internal/reverse:reverse_test                                          PASSED in 0.0s

Executed 1 out of 1 test: 1 test passes.
```

Goの`embed`パッケージを使ってファイルを埋め込む場合は、 `data` ではなく `embedsrcs` パラメータにこれらのファイルを指定します。これもgazelleが自動的に行うため、単に `bazel run //:gazelle` します。

```go:internal/reverse/string_test.go
package reverse_test

import (
	"embed"
	"testing"

	"github.com/pddg/go-bazel-playground/internal/reverse"
)

//go:embed testdata/*
var testdata embed.FS

func TestString(t *testing.T) {
	t.Parallel()
	tests := []struct {
		name string
	}{
		{
			name: "empty",
		},
		{
			name: "single",
		},
		{
			name: "multiple",
		},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			// Setup
			givenBytes, err := testdata.ReadFile("testdata/" + tt.name + "_string_input.txt")
			if err != nil {
				t.Fatal(err)
			}
			expectedBytes, err := testdata.ReadFile("testdata/" + tt.name + "_string_output.txt")
			if err != nil {
				t.Fatal(err)
			}
			given := string(givenBytes)
			expected := string(expectedBytes)

			// Exercise
			actual := reverse.String(string(given))

			// Verify
			if actual != expected {
				t.Errorf("expected: %s, actual: %s", expected, actual)
			}
		})
	}
}
```

`bazel run //:gazelle` すると、 `embedsrcs` パラメータが追加されます。

```diff:internal/reverse/BUILD.bazel
diff --git a/internal/reverse/BUILD.bazel b/internal/reverse/BUILD.bazel
index d206891..61f2d00 100644
--- a/internal/reverse/BUILD.bazel
+++ b/internal/reverse/BUILD.bazel
@@ -11,6 +11,13 @@ go_test(
     name = "reverse_test",
     size = "small",
     srcs = ["string_test.go"],
     data = glob(["testdata/**"]),
+    embedsrcs = [
+        "testdata/empty_string_input.txt",
+        "testdata/empty_string_output.txt",
+        "testdata/multiple_string_input.txt",
+        "testdata/multiple_string_output.txt",
+        "testdata/single_string_input.txt",
+        "testdata/single_string_output.txt",
+    ],
     deps = [":reverse"],
 )
```

## Conclusion

rules_goの `go_test` ルールを使うことでBazelからGoのユニットテストを実行できました。
Bazelではユニットテストもビルドと同様に抽象化されたルールを通して実行でき、Bazelはその依存関係を解決して必要なテストのみを実行します。
テストにはサイズとタイムアウトという概念があり、テストの大きさによって暗黙的に設定されるタイムアウトが異なります。適切なテストサイズ・タイムアウトを指定するべきです。
テストにおいて外部ファイルなどソースコード以外のリソースを使う場合、 `data` パラメータや `embedsrcs` パラメータを使って依存を明示することでテスト実行時にこれらのリソースを参照できます。
