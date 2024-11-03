---
title: "バージョン情報を埋め込む"
---

例えばコンテナイメージはタグを用いてバージョンを指定することが多いでしょう。ここまでの例では常に `latest` というタグを用いていましたが、本番環境で `latest` を利用することはアンチパターンの1つとされています。

また、実行コマンドにバージョン情報などを動的に埋め込むこともよく行われています。 `--version` オプションや `version` サブコマンドを持つコマンドラインツールなどがその例です。

bazelを用いて生成したコンテナイメージやバイナリに対して、このようなバージョン情報を埋め込む方法を見ていきます。

## How to inject version information

### Go Binary

方法としてはいくつかありますが、まずは伝統的な `-ldflags` オプションを利用する方法を紹介します。

```go
package main

import (
	"fmt"
)

var Version string

func main() {
	fmt.Println("Version: " + Version)
}
```

このようなコードをビルドする際に `-ldflags` オプションを指定することで、バイナリにバージョン情報を埋め込めます。

```sh
$ go build -ldflags "-X main.Version=1.0.0" -o main main.go
$ ./main
Version: 1.0.0
```

同様の設定がbazelでも可能になっています。
<https://github.com/bazel-contrib/rules_go/blob/master/docs/go/core/defines_and_stamping.md#defines-and-stamping>

hello_worldコマンドにバージョンを埋め込んでみましょう。

```diff:apps/hello_world/main.go
diff --git a/apps/hello_world/main.go b/apps/hello_world/main.go
index a18d207..0ab0dc1 100644
--- a/apps/hello_world/main.go
+++ b/apps/hello_world/main.go
@@ -4,10 +4,14 @@ import (
        "fmt"
 
        "github.com/google/uuid"
+
        "github.com/pddg/go-bazel-playground/internal/reverse"
 )
 
+var Version = "dev"
+
 func main() {
+       fmt.Printf("Version: %s\n", Version)
        uuidStr := uuid.NewString()
        fmt.Printf("Hello, World!(%s)\n", uuidStr)
        fmt.Printf("Reversed: %s\n", reverse.String("Hello, World!"))
```

そして以下の様に `go_binary` ルールを設定します。

```python:apps/hello_world/BUILD.bazel
go_binary(
    name = "hello_world",
    embed = [":hello_world_lib"],
    visibility = ["//visibility:public"],
    x_defs = {
      "Version": "1.0.0",
    },
)
```

これにより、bazelでのビルド時に `-X main.Version=1.0.0` が指定された状態でビルドされます。

```text
❯ bazel run //apps/hello_world     
INFO: Analyzed target //apps/hello_world:hello_world (0 packages loaded, 0 targets configured).
INFO: Found 1 target...
Target //apps/hello_world:hello_world up-to-date:
  bazel-bin/apps/hello_world/hello_world_/hello_world
INFO: Elapsed time: 0.130s, Critical Path: 0.00s
INFO: 1 process: 1 internal.
INFO: Build completed successfully, 1 total action
INFO: Running command line: bazel-bin/apps/hello_world/hello_world_/hello_world
Version: 1.0.0
Hello, World!(c18327c7-9f6a-4e57-b4f1-8c780765f1a2)
Reversed: !dlroW ,olleH
OsName: darwin
```

### Container Image

rules_ociでは `oci_push` ルールの `remote_tags` にリモートリポジトリへpushする際のタグを指定できます。

```python:apps/fortune_cowsay/BUILD.bazel
oci_push(
    name = "image_push",
    image = ":image_index",
    # here
    remote_tags = ["1.0.0"],
    repository = REPOSITORY,
)
```

## Versioning strategies

### Manual

最も一般的なバージョニング手法はSemantic Versioningと言って良いと思います。これはバージョン番号を `Major.Minor.Patch` の形式で表現するものです。例えば `1.0.0` などのバージョニングを採用するものがそうです。

BUILDファイル内で `VERSION` のような変数を宣言し、Goのバイナリとそのコンテナにそれぞれ指定すれば同じバージョン情報を付与できます。

- Pros
  - 一般的なバージョニング手法であるため、他の開発者が理解しやすい
- Cons
  - 新しい変更をリリースするためには人が手動でバージョン番号を更新しなければならない
  - 手動での指定は人為的なミスが発生しやすい
    - 変更を忘れる
    - ブランチ間で重複するバージョン番号を指定する
    - バージョン番号のフォーマットを間違える
    - ...etc

### Auto

バージョンを手動で管理する場合、依存するライブラリなどが他の人・チームなどによって変更されたとき、バージョンも変えなければ新しいバージョンはリリースされません。bazelは逆依存を辿ることも出来るため、新しい変更を入れたチームがその依存関係を解析してバージョンを書き換えることは可能です。しかし、単純なSemantic Versioningでは変更が競合したり、異なるチームが異なる変更を同じバージョンでリリースしたりしてしまう可能性があります。

そこで、簡単には重複することがないバージョン番号を自動で生成する方法を考えます。以下のブログで言及されている方法を紹介します。  
@[card](https://blog.aspect.build/versioning-releases-from-a-monorepo)

まず、リポジトリに毎週タグを打ちます。年と、年初からの週の数を用います。例えば、2022年の第1週のタグは `2022.01` となります。そして、リリースするタイミングでは前回のタグから今までのコミット数を数え、それをバージョン番号の最後に付与します。例えば、2022年の第1週から2022年の第2週までに10回コミットがあった場合、そのリリースのバージョン番号は `2022.01.10` となります。そして最後にHEADのコミットハッシュを付与することで、重複しないバージョン番号を生成できます。

この毎週タグを打つためにGitHub Actionsを利用します。curlコマンドを使うことで、Git cloneせずに最新のHEADにタグを打てます。これはいわゆる[lightweight tag](https://git-scm.com/book/en/v2/Git-Basics-Tagging)であり、単にバージョン計算の目印としてのみ利用します。  
@[card](https://gist.github.com/alexeagle/ad3f1f4f90a5394a866bbb3a3b8d1de9)

```yaml:.github/workflows/weekly-tag.yml
on:
  schedule:
    # Sunday 15:00 UTC = Monday 00:00 JST 
    - cron: '0 15 * * 0'

permissions:
  # GitHub Token can create tags
  contents: write

jobs:
  tagger:
    runs-on: ubuntu-latest
    steps:
      - name: tag HEAD with date +%G.%V
        run: |
          curl --request POST \
            --url https://api.github.com/repos/${{ github.repository }}/git/refs \
            --header 'authorization: Bearer ${{ secrets.GITHUB_TOKEN }}' \
            --data @- << EOF
          {
            "ref": "refs/tags/$(date +%G.%V)",
            "sha": "${{ github.sha }}"
          }
          EOF
```

このように付加されたタグを用いて、最新のHEADのバージョンを計算します。またOCI Imageのwell-kwnonラベルに使うため、いくつかの情報を一緒に出力します。

```sh
cat << 'EOF' > build_tools/integrations/version.sh
#!/bin/bash
set -e

# Calculate version
VERSION_WITH_HASH=$(git describe \
  --tags \
  --long \
  --match="[0-9][0-9][0-9][0-9].[0-9][0-9]" \
  | sed -e 's/-/./;s/-g/+/')

# Show version with git hash
echo "VERSION_WITH_HASH ${VERSION_WITH_HASH}"
# Show version only
echo "VERSION $(echo ${VERSION_WITH_HASH} | cut -d+ -f1)"
# Show git hash only
echo "GIT_SHA $(echo ${VERSION_WITH_HASH} | cut -d+ -f2)"

BUILD_TIMESTAMP=${BUILD_TIMESTAMP:-$(date +%s)}

# Show build timestamp in ISO8601 format
if [ "$(uname)" == "Darwin" ]; then
  BUILD_ISO8601=$(date -u -r "$BUILD_TIMESTAMP" +"%Y-%m-%dT%H:%M:%SZ")
else
  BUILD_ISO8601=$(date -u -d "@$BUILD_TIMESTAMP" +"%Y-%m-%dT%H:%M:%SZ")
fi
echo "BUILD_TIMESTAMP_ISO8601 $BUILD_ISO8601"
EOF
chmod +x build_tools/integrations/status.sh
```

このスクリプトを実行すると以下の様に表示されるはずです。バージョンやコミットハッシュ、日付などは環境によって異なります。

```sh
❯ ./build_tools/integrations/status.sh
VERSION_WITH_HASH 2024.42.41+eba4d2e
VERSION 2024.42.41
GIT_SHA eba4d2e
BUILD_TIMESTAMP_ISO8601 2024-10-23T13:17:43Z
```

Bazelには `--stamp` オプションおよび `--workspace_status_command` オプションを用いることで、ビルド時に任意の情報を埋め込む機能が存在します。これらを利用して動的に生成したこれらの情報を埋め込みます。

```sh
cat << 'EOF' >> .bazelrc
build:release --workspace_status_command=build_tools/integrations/version.sh
build:release --stamp
EOF
```

:::message
`.bazelrc` において `build` と `build:release` はそれぞれ異なる設定を持てます。
`:release` を付けた設定を有効化するには、bazelコマンドを実行する際に `--config=release` を指定します。
@[card](https://bazel.build/run/bazelrc#config)
:::

go_binaryルールを以下の様に設定します。

```python:apps/hello_world/BUILD.bazel
go_binary(
    name = "hello_world",
    embed = [":hello_world_lib"],
    visibility = ["//visibility:public"],
    x_defs = {
      "Version": "{VERSION}",
    },
)
```

これにより、`--config=release` を指定した場合のみビルド時に `build_tools/integrations/status.sh` が実行され、その結果が `Version` として埋め込まれます。

```sh
❯ bazel run --config=release //apps/hello_world
WARNING: Build option --stamp has changed, discarding analysis cache (this can be expensive, see https://bazel.build/advanced/performance/iteration-speed).
INFO: Analyzed target //apps/hello_world:hello_world (0 packages loaded, 17598 targets configured).
INFO: Found 1 target...
Target //apps/hello_world:hello_world up-to-date:
  bazel-bin/apps/hello_world/hello_world_/hello_world
INFO: Elapsed time: 0.952s, Critical Path: 0.19s
INFO: 2 processes: 1 internal, 1 darwin-sandbox.
INFO: Build completed successfully, 2 total actions
INFO: Running command line: bazel-bin/apps/hello_world/hello_world_/hello_world
Version: 2024.42.41+eba4d2e
Hello, World!(a514029c-e0a0-4227-8fbd-dd06483b6e04)
Reversed: !dlroW ,olleH
OsName: darwin
```

:::message
WARNINGにあるとおり、 `--stamp` オプションが有効化されると一部のキャッシュは無効化されます。よって常にこのオプションを有効化してビルドすべきではありません。
また、 `--workspace_status_command` でGitの情報を取得するため、これも `--config=release` でのみ有効化することをお勧めします。普段から実行すると、bazelコマンドの実行のたびにGitの操作が走るため、パフォーマンスが低下します。
:::

少し方法は異なりますが、イメージのpush時にもバージョン情報を埋め込めます。ただしコンテナイメージのタグには `+` が使えないことに注意してください。
イメージのタグをファイルに書き込み、それを `oci_push` ルールで利用します。このファイルへ書き込む際にバージョン情報で値を置換し、stampされたときのみ正しいバージョンを、それ以外では `latest` を使います。この操作を簡単にするため、 [bazel-lib](https://github.com/bazel-contrib/bazel-lib) を導入します。

```python:MODULE.bazel
bazel_dep(name = "aspect_bazel_lib", version = "2.9.3")
```

```python:apps/fortune_cowsay/BUILD.bazel
load("@aspect_bazel_lib//lib:expand_template.bzl", "expand_template")

# 中略

expand_template(
    name = "image_tags",
    out = "_stamped.tags.txt",
    template = ["latest"],
    stamp_substitutions = {
        "latest": "{{VERSION}}",
    },
)
oci_push(
    name = "image_push",
    image = ":image_index",
    remote_tags = ":image_tags",
    repository = REPOSITORY,
)
```

更にイメージに付与するannotationも同様に生成します。

```python:apps/fortune_cowsay/BUILD.bazel
expand_template(
    name = "image_annotations",
    out = "_stamped.annotations.txt",
    template = [
        "org.opencontainers.image.source=https://github.com/pddg/go-bazel-playground",
        "org.opencontainers.image.version=nightly",
        "org.opencontainers.image.revision=devel",
        "org.opencontainers.image.created=1970-01-01T00:00:00Z",
    ],
    stamp_substitutions = {
        "devel": "{{GIT_SHA}}",
        "nightly": "{{VERSION}}",
        "1970-01-01T00:00:00Z": "{{BUILD_TIMESTAMP_ISO8601}}",
    },
)
oci_image(
    name = "image_push",
    base = base,
    entrypoint = entrypoint,
    tars = tars,
    annotations = ":" + name + "_annotations",
    labels = ":" + name + "_annotations",
)
```

これを使ってイメージをpushします。

```sh
bazel run --config=release //apps/fortune_cowsay:image_push
```

<https://github.com/pddg/go-bazel-playground/pkgs/container/go-bazel-playground-fortune-cowsay/294968656?tag=2024.42.41>

annotationsが正しいかを確認してみましょう。

```sh
❯ docker buildx imagetools inspect \
    ghcr.io/pddg/go-bazel-playground-fortune-cowsay:2024.42.41@sha256:9ed000703678f09ad8274effda8fcdd2bf674a49e77b02bd17d9b216b2e8e166 \
    --raw \
    | jq -r .annotations
{
  "org.opencontainers.image.source": "https://github.com/pddg/go-bazel-playground",
  "org.opencontainers.image.version": "2024.42.41",
  "org.opencontainers.image.revision": "eba4d2e",
  "org.opencontainers.image.created": "2024-10-25T04:02:32Z"
}
```

これを毎回行うのは面倒なのでmacroとしてまとめてしまうことにします。

```python:build_tools/macros/oci.bzl
load("@aspect_bazel_lib//lib:expand_template.bzl", "expand_template")

def oci_image_with_known_annotations(
    name,
    base,
    entrypoint,
    tars,
    annotations = {},
):
    """oci_image_with_known_annotations creates a container image with known annotations.

    following annotations are added by default:
    - org.opencontainers.image.source
    - org.opencontainers.image.version
    - org.opencontainers.image.revision
    - org.opencontainers.image.created

    Obtain the value of these annotations from the build environment.
    - VERSION: Version number of the build.
    - GIT_SHA: Git commit hash of the build.
    - BUILD_TIMESTAMP_ISO8601: Timestamp of the build in ISO8601 format.

    Args:
        name: The name of this target.
        base: The base image to use.
        entrypoint: The entrypoint for the container.
        tars: The tarballs to include in the image.
        annotations: The annotations to add to the image.
    """
    expand_template(
        name = name + "_annotations",
        out = "_stamped.annotations.txt",
        template = [
            "org.opencontainers.image.source=https://github.com/pddg/go-bazel-playground",
            "org.opencontainers.image.version=nightly",
            "org.opencontainers.image.revision=devel",
            "org.opencontainers.image.created=1970-01-01T00:00:00Z",
        ] + [
            "{}={}".format(key, value) for (key, value) in annotations.items()
        ],
        stamp_substitutions = {
            "devel": "{{GIT_SHA}}",
            "nightly": "{{VERSION}}",
            "1970-01-01T00:00:00Z": "{{BUILD_TIMESTAMP_ISO8601}}",
        },
    )
    oci_image(
        name = name,
        base = base,
        entrypoint = entrypoint,
        tars = tars,
        annotations = ":" + name + "_annotations",
        labels = ":" + name + "_annotations",
    )


def oci_push_with_version(
    name,
    image,
    repository,
):
    """oci_push_with_stamped_tags pushes an image with stamped tags.

    Args:
        name: The name of this target.
        image: The image to push.
        repository: The repository to push the image to.
    """
    expand_template(
        name = name + "_tags",
        out = "_stamped.tags.txt",
        template = ["latest"],
        stamp_substitutions = {
            "latest": "{{VERSION}}",
        },
    )
    oci_push(
        name = name,
        image = image,
        repository = repository,
        remote_tags = ":" + name + "_tags",
    )
```

### Pros and Cons

- Pros
  - バージョン番号が自動で生成されるため、人為的なミスが発生しにくい
  - 重複しないバージョン番号が生成される
  - 新しいバージョンのリリース時には単に `--config=release` オプションを指定するだけで良い
- Cons
  - そのアプリケーションの機能的な変更内容を反映しない
  - 仕組みが複雑になる

## Conclusion

bazelでは `--stamp` オプションおよび `--workspace_status_command` オプションを用いることで、ビルド時に任意の情報を埋め込めます。

バージョン情報を埋め込む方法として、手動での指定と自動での生成の2つを紹介しました。

手動での指定は一般的なバージョニング手法ですが、リリースを自動化できない他、人為的なミスが発生しやすいという課題があります。自動での生成はそのようなミスを防げますが、そのアプリケーションの機能的な変更内容を反映しないという問題があります。

どちらを採用するかはそのアプリケーションの性質や開発体制によりますが、自動生成を採用することでバージョン情報の管理を自動化できます。
