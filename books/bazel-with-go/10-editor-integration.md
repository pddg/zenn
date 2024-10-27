---
title: "エディタと連携する"
---

bazelでは自動で生成されるファイルはサンドボックス内に配置されるため、リポジトリにコミットされません。そのため、gRPCの自動生成されたパッケージをインポートする場合、VSCodeのGo Extensionなどのようなツールで補完が効かずエディタ上ではエラー扱いになります。

これは開発体験が非常に悪いので、補完などが動作するようにするための設定しましょう。

過去の記事ではbazelが自動生成したファイルを実際にリポジトリ内に持ち込んでコミットし、それを参照する方法が紹介されていることが多かったと思います。選択肢は他にもあるため、今回はそちらを紹介します。

## Integrate with gopls

[gopls](https://github.com/golang/tools/tree/master/gopls)はGoのエディタサポートを提供するツールで、Language Server Protocol互換のLanguage Server実装です。これは通常リポジトリのルートにある `go.mod` ファイルとGoのソース、ホスト上の `~/go/mod` などを読み取ることで、補完などを提供しています。

bazelによって生成されるファイルなどの情報をgoplsに提供するためには、 `GOPACKAGESDRIVER` という環境変数を利用します。これは通常とは異なるビルドシステムを利用する場合でもgoパッケージを読み込み、その依存関係を認識するために必要なメタデータを提供する別のプログラムを指定できるようにする機能です。
rules_go にはこの機能が実装されている。

まずはこれを起動するためのスクリプトを作成します。

```sh
mkdir -p build_tools/integrations
cat << 'EOF' > build_tools/integrations/gopackagesdriver.sh
#!/usr/bin/env bash
# See https://github.com/bazelbuild/rules_go/wiki/Editor-setup#3-editor-setup
exec bazel run -- @rules_go//go/tools/gopackagesdriver "${@}"
EOF
chmod +x build_tools/integrations/gopackagesdriver.sh
```

ここでは例としてVSCodeを用いますが、vimなど他のエディタでもgoplsを用いる場合は概ね同様の設定になります。詳しくは以下を参照してください。

@[card](https://github.com/bazelbuild/rules_go/wiki/Editor-and-tool-integration)

また、以下の拡張機能を予めインストールしておきましょう。

@[card](https://marketplace.visualstudio.com/items?itemName=golang.Go)

VSCodeは開いているプロジェクト直下に `.vscode` ディレクトリがある場合、そのディレクトリ内の設定ファイルを読み込みます。これを利用してこのリポジトリにおける設定を一部上書きします。

```sh
mkdir -p .vscode
```

```json:.vscode/settings.json
{
    "go.goroot": "${workspaceFolder}/bazel-${workspaceFolderBasename}/external/rules_go~~go_sdk~${workspaceFolderBasename}__download_0/",
    "go.toolsEnvVars": {
        "GOPACKAGESDRIVER": "${workspaceFolder}/build_tools/integrations/gopackagesdriver.sh"
    },
    "gopls": {
        "build.directoryFilters": [
            "-bazel-bin",
            "-bazel-out",
            "-bazel-testlogs",
            "-bazel-go-bazel-playground",
        ],
    },
    "go.useLanguageServer": true,
}
```

最小限の設定としてはこのようになります。Wikiでは更に余計な機能の無効化や、フォーマット設定の変更などが記載されているので、必要に応じてそれらも設定するとよいでしょう。

念のため利用前にはGo Language Serverを再起動します。 `Ctrl + Shift + P` でコマンドパレットを開き、 `Go: Restart Language Server` を選択することで再起動できます。

### Tips: import時に `module not found`

通常のGoの開発プロジェクトの場合、`go mod tidy` したり一度ビルドして依存関係をダウンロードするとgoplsがその依存関係を認識できるようになり、補完が動作するようになります。

bazelはBUILDファイル内で許可された依存関係のみを `GOPACKAGESDRIVER` を用いてgoplsに伝えます。よって、 `go.mod`に記載されているからといって、あるモジュール内でそのモジュールの依存解決ができるわけではありません。

例えば`go.mod` には `google.golang.org/grpc` がありますが、ビルド対象のdependencyとして指定されていなければ、goplsは存在しない依存として報告します。この場合、gazelleを用いてビルドファイルを生成し、正しい依存関係をgoplsに伝えましょう。

```sh
# 必要なら
# bazel run @rules_go//go -- mod tidy
# bazel run //:update-go-repos
bazel run //:gazelle
```

ビルドファイルに必要な依存関係が追加され、ビルド・エディタでの補完が動作するようになるはずです。

### Pros/Cons

- Pros
  - bazelのビルドファイルを利用するため、bazelのビルド設定に従った補完が可能
  - bazelがルールに従って自動生成したファイルも補完対象になる
- Cons
  - やや遅い
    - 内部的にはbazel queryなどを活用しているため、goplsのように単にファイルシステムをスキャンするだけのものよりも遅い
  - ビルドファイルの生成が必要
    - 依存関係が変わるたびにビルドファイルを更新する必要がある
    - 依存関係の変更は成熟したプロダクトにおいてそれほど多い作業ではないものの、新規開発中などでは頻繁に行うことがある

bazelを使ってビルド・テストを実行するため、VSCodeのGo Extensionにあるテストケースごとの実行などはサポートされていません。go.modは維持しているため、以下の条件を満たせばVSCodeのUIからの実行も可能です。

- bazelによる自動生成ファイルが依存上にない
- サードパーティモジュールをbazelでパッチしていない

ただし、それは今後も同様である補償はなく、bazelからテストを起動した方が確実でしょう。

## Conclusion

goplsを使っている場合、`GOPACKAGESDRIVER` 環境変数を利用することでBazelのビルドの設定をエディタに反映させられます。
動作はやや遅いものの、bazelが自動生成したファイルや、パッチを当てたサードパーティモジュールなども補完対象になります。
ただし、bazelのビルドにおける依存関係をそのまま利用するため、正しい依存関係の記述されたビルドファイルの生成が必要であることに注意してください。
