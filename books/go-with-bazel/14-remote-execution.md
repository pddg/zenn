---
title: "リモート実行する"
---

Bazelはビルドという操作を入力に対して出力を生成する純粋関数として捉えています。つまり、入力が同じであれば異なるマシン上でビルドしても同じ出力を得られるということです。この性質を利用して、リモートにあるマシン上でビルドできます。
これによりローカルのマシンのスペックに依存せず、リモートのマシンのスペックを利用し、さらに複数のマシンでビルドを並列化できます。

これをBazelではRemote Build Execution（RBE）と呼んでいます。
@[card](https://bazel.build/remote/rbe)

## Buildbarn

リモート実行を実装したOSSとして、[Buildbarn](https://github.com/buildbarn)が知られています。そのexampleが以下に公開されています。

@[card](https://github.com/buildbarn/bb-deployments)

アーキテクチャや概念が難しくて筆者は今心が折れています。

## BuildBuddy

[BuildBuddy](https://www.buildbuddy.io/)はBazelのリモート実行をサポートするSaaSです。オンプレミス版も提供されています。

@[card](https://github.com/buildbuddy-io/buildbuddy)

オンプレミス版も無料で利用できるバージョンもあるのですが、RBEの機能はエンタープライズ版限定になっています。

@[card](https://www.buildbuddy.io/docs/config-rbe)

> Remote Build Execution is only configurable in the Enterprise version of BuildBuddy.

国内だとメルカリがiOSアプリのビルドにBuildBuddyを利用しているという話が公開されています。

@[card](https://engineering.mercari.com/blog/entry/20221215-16cdd59909/)

## Other services

他にもBazelのリモート実行をサポートするサービスはいくつかあります。Bazelの公式ドキュメントにもリストされています。

@[card](https://bazel.build/community/remote-execution-services)

Bazelを本格的に採用するなら、これらのSaaSの活用は検討に値するでしょう。

## Conclusion

Bazelのリモート実行は、ビルドの高速化やスケーラビリティを向上させるための重要な機能です。リモート実行をサポートするサービスはいくつかあり、それらを活用することでビルドを高速化できます。
オンプレミスでの運用を考える場合は、Buildbarnが第一の選択肢としてあげられる他、BuildBuddyのエンタープライズサポートを利用することも検討に値します。
筆者は家のKubernetesクラスタでRBEしたかったのですが、今のところ心が折れています。
