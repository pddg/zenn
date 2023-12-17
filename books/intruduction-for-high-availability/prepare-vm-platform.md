---
title: "仮想サーバ基盤を構築する"
---

## 仮想サーバ（VM）とは

VM（Virtual Machineの略）とは「仮想マシンは、実際のコンピューターのように動作するコンピューター ファイル」[^ms-vm-dev]です。これらは普通にサーバを使っている時のようにCPU・メモリ・ディスクなどを持ち、ネットワークで通信することが出来ますが、その実体は物理マシンのリソースを抽象化したものを操っており物理的に目に見えるものではありません。

[^ms-vm-def]: https://azure.microsoft.com/ja-jp/resources/cloud-computing-dictionary/what-is-a-virtual-machine

## なぜVMを用いるのか

- アプリケーションやミドルウェアの実行環境の分離のため
- 物理マシンは柔軟な管理が難しく管理が大きいため

## 様々なサーバ仮想化製品

- Windows Hyper-V
- Linux KVM
- VirtualBox
- VMware ESXi
- Proxmox VE

今回はフリーのProxmox VEを用いてVM基盤を構築する。まずは中身については触れず、実際にVMを作って遊ぶ。

