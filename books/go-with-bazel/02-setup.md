---
title: "BazelとGoをセットアップする"
---

## Install bazel using bazelisk

直接Bazelをインストールするのではなく、[Bazelisk](https://github.com/bazelbuild/bazelisk)をインストールします。
Bazeliskは、bazelのバージョンを指定して実行するためのラッパースクリプトです。リポジトリのルートにある `.bazeliskrc` ファイルの設定に従って、実行されるbazelのバージョンを決め、実行時に自動でダウンロードします。
各自の環境にはBazeliskのみを入れておき、`.bazeliskrc` を通してプロジェクト内で使うBazelのバージョンを制御することで、各自の環境のBazelのバージョンを気にかける必要がなくなります。

```sh
# Linux
ARTIFACT="bazelisk-linux-$(uname -m | sed 's/x86_64/amd64/')"
wget -O bazelisk \
  "https://github.com/bazelbuild/bazelisk/releases/latest/download/${ARTIFACT}"
chmod +x bazelisk
sudo mv bazelisk /usr/local/bin
# Install as `bazel`
sudo ln -s /usr/local/bin/bazelisk /usr/local/bin/bazel

# macOS
# This will install `bazelisk` and alias for it named`bazel`
brew install bazelisk
```

上記の手順で `bazel` コマンドは `bazelisk` コマンドへのシンボリックリンクになっています。以下の様に `.bazelrc` に記述することで、実行されるbazelのバージョンを決められます。ここでは、現在の最新安定版である 7.x を指定しています。`7.4.0` のようにパッチバージョンまで決め打つこともできます（本番環境ではそうした方が良いでしょう）。

```sh
cat  << EOF > .bazeliskrc
USE_BAZEL_VERSION=7.x
EOF
```

実行されるbazelのバージョンを確認してみます。

```text
❯ bazel --version     
bazel 7.4.0
```

## Setup Go project

Goプロジェクトとして初期化します。`module` の名前は各自の環境で調整して下さい。

```sh
cat << EOF > go.mod
module github.com/pddg/go-bazel-playground

go 1.23.2
EOF
```

## Setup rules_go

Bazelでは言語ごとにビルドツールの呼び出しを抽象化したルールを用意するのが慣習となっています。これらは `rules_` というプレフィックスがついていることが多く、Go言語には[rules_go](https://github.com/bazelbuild/rules_go)が用意されています。

これらの外部ruleを使うために、Bazelではモジュールの仕組みが用意されています。これは[Bzlmod](https://bazel.build/external/module)と呼ばれる比較的新しい機能です。これは例えばGo言語におけるGo Modulesのようなものです。モジュールを配布するための[Registry](https://bazel.build/external/registry)からモジュールを取得でき、代表的なものは[Bazel Central Registry](https://registry.bazel.build/)で配布されています。特に指定しなければBazel Central Registryから取得されます。

リポジトリのルートに `MODULE.bazel` ファイルを作成します。このファイルは[Starlark](https://github.com/bazelbuild/starlark)というPythonのサブセットのような言語で記述します。

```python:MODULE.bazel
"""
This is a playground for developing application with Bazel.
"""

# このリポジトリの名前とバージョンを宣言する
module(
    name = "go-bazel-playground",
    version = "0.0.1",
)
```

rules_goを導入するため`MODULE.bazel`に追記します。最新のバージョンはGitHub Releasesの他、Bazel Central Registryからも確認できます。
<https://registry.bazel.build/modules/rules_go>

```python
bazel_dep(name = "rules_go", version = "0.50.1", repo_name = "rules_go")
```

::: message
Bzlmodは各ルールが依存する他のルールも自動的に解決します。異なるルール間で共通して依存するルールがある場合、そのバージョンはGoと同じく[Minimal Version Selection（MVS）](https://research.swtch.com/vgo-mvs)によって解決されます。
:::

`bazel mod tidy` コマンドでいくつかの必要な記述の追記やフォーマットを行えます。

```bash
bazel mod tidy
```

これまでのBazelでは `WORKSPACE.bazel` ファイルにてバージョンやそのチェックサムを指定していました[^workspace]が、Bazel Central Registryが整備されたことで、名前とバージョンのみで簡潔に指定できるようになりました。

[^workspace]: <https://bazel.build/versions/7.4.0/external/overview#workspace-system>

一方で、まだ `WORKSPACE.bazel` ファイルは利用可能であり、その存在に依存するルールも多いため、ここでは空ファイルだけを作成しておきます。

```bash
touch WORKSPACE.bazel
```

::: message
WORKSPACEの機能はBazel 8.0でデフォルト無効になり、Bazel 9.0で削除されることが予告されています。
<https://bazel.build/about/roadmap>
Bazel 7.1以降では `--noenable_workspace` オプションで先に機能が削除された状態を試し、問題が起きないか確認できます。
:::

## Setup Go SDK

Bazel moduleは[extension](https://bazel.build/external/extension)という機能を使って、外部の依存をBazelのエコシステムに統合します。rules_goのextensionを使って、このプロジェクトで利用するGo SDKをダウンロードします。

```python
go_sdk = use_extension("@rules_go//go:extensions.bzl", "go_sdk")
go_sdk.download(version = "1.23.2")
```

ダウンロードされるSDKは、公開されているチェックサムとともに `MODULE.bazel.lock` に記録されます（[ref](https://github.com/bazel-contrib/rules_go/blob/1e855e21ad8273e470d0b417235304cd0fbd55b0/go/private/sdk.bzl#L75-L110)）。公開されているチェックサムを信用せず、自分でチェックサムを固定したい場合は以下の様に記述します。

```python
go_sdk.download(
    version = "1.23.2",
    sdks = {
        "linux_amd64": ("go1.23.2.linux-amd64.tar.gz", "542d3c1705f1c6a1c5a80d5dc62e2e45171af291e755d591c5e6531ef63b454e"),
        "linux_arm64": ("go1.23.2.linux-arm64.tar.gz", "f626cdd92fc21a88b31c1251f419c17782933a42903db87a174ce74eeecc66a9"),
        "darwin_amd64": ("go1.23.2.darwin-amd64.tar.gz", "445c0ef19d8692283f4c3a92052cc0568f5a048f4e546105f58e991d4aea54f5"),
        "darwin_arm64": ("go1.23.2.darwin-arm64.tar.gz", "d87031194fe3e01abdcaf3c7302148ade97a7add6eac3fec26765bcb3207b80f"),
    }
)
```

チェックサム及びファイル名は <https://go.dev/dl/> から取得できます。また、アーカイブはデフォルトでは <https://dl.google.com/go/> からダウンロードされます。

## Run go command

導入したGo SDKを使って、`go` コマンドを実行してみます。

```bash
bazel run @rules_go//go -- version
```

以下の様にGoのバージョンが表示されれば成功です。

```text
❯ bazel run @rules_go//go -- version
INFO: Analyzed target @@rules_go~//go:go (2 packages loaded, 12 targets configured).
INFO: Found 1 target...
Target @@rules_go~//go/tools/go_bin_runner:go_bin_runner up-to-date:
  bazel-bin/external/rules_go~/go/tools/go_bin_runner/bin/go
INFO: Elapsed time: 0.428s, Critical Path: 0.03s
INFO: 2 processes: 2 internal.
INFO: Build completed successfully, 2 total actions
INFO: Running command line: bazel-bin/external/rules_go~/go/tools/go_bin_runner/bin/go version
go version go1.23.2 linux/amd64
```

## Build Go application

`apps` ディレクトリを作成し、`hello_world` というGoアプリケーションを作成します。このアプリケーションは単に `Hello, World!` と出力するだけのものです。

```bash
mkdir -p apps/hello_world
cat <<EOF > apps/hello_world/main.go
package main

import "fmt"

func main() {
	fmt.Println("Hello, World!")
}
EOF
```

bazelでは、Goアプリケーションをビルドするために開発者が `go` コマンドの使い方を覚える必要はありません。ビルドするアプリケーションのディレクトリに `BUILD.bazel` ファイルを作成し、ビルドルールを記述します。

```python:apps/hello_world/BUILD.bazel
load("@rules_go//go:def.bzl", "go_binary")

go_binary(
    name = "hello_world",
    srcs = ["main.go"],
    visibility = ["//visibility:public"],
)
```

`BUILD.bazel` ファイルを作成したら、`bazel build` コマンドでビルドします。Bazelはアーティファクトベースのビルドツールであるため、何をするかではなく何をビルドするかをコマンドで指定します。

```bash
bazel build //apps/hello_world:hello_world
```

これは `apps/hello_world` ディレクトリにある `hello_world` という名前のビルド対象をビルドすることを指定しています。

以下の様なログが表示されれば成功です。

```text
❯ bazel build //apps/hello_world:hello_world
INFO: Analyzed target //apps/hello_world:hello_world (72 packages loaded, 11985 targets configured).
INFO: Found 1 target...
Target //apps/hello_world:hello_world up-to-date:
  bazel-bin/apps/hello_world/hello_world_/hello_world
INFO: Elapsed time: 0.727s, Critical Path: 0.09s
INFO: 6 processes: 4 internal, 2 linux-sandbox.
INFO: Build completed successfully, 6 total actions
```

実際にこのアプリケーションを実行してみます。 `bazel run` コマンドを使うことで実行可能なターゲットはビルド・実行できます。

```bash
bazel run //apps/hello_world:hello_world
```

```text
❯ bazel run //apps/hello_world:hello_world
INFO: Analyzed target //apps/hello_world:hello_world (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
Target //apps/hello_world:hello_world up-to-date:
  bazel-bin/apps/hello_world/hello_world_/hello_world
INFO: Elapsed time: 0.154s, Critical Path: 0.00s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
INFO: Running command line: bazel-bin/apps/hello_world/hello_world_/hello_world
Hello, World!
```

## Conclusion

Bazelを使ってGoアプリケーションをビルドするための環境を構築しました。
Bazelでは、ビルドツールの呼び出しを抽象化したルールを用意することが慣習となっており、Goではrules_goが用意されています。
rules_goはBzlmodを使って簡単に導入でき、指定したバージョンのGo SDKをダウンロード・実行できます。
rules_goを使ってGoアプリケーションをビルドするためには、`BUILD.bazel` ファイルを作成し、ビルドルールを記述します。
