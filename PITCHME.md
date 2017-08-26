# fork爆弾爆発中のロードアベレージを見る

@第30回シェル芸勉強会 大阪サテライト (2017/08/26)

---
# 目次
* 前回の復習、今回の目標
* fork爆弾爆発中のロードアベレージの取り方を考える
* デモ
* まとめ

---
# 自己紹介

* ハンドルネーム: MSR
 * [Twitter ID: @msr386](https://twitter.com/msr386)
* Webブラウザ [Tungsten](https://app.tungsten-start.net/) の作者

---
# 前回の復習

* fork爆弾実行時のロードアベレージ取得に成功した  
※SysRqキーでカーネルパニックを起こし、クラッシュダンプを解析
* 高ロードアベレージを記録するには1時間回せば十分
* 24時間回しただけ無駄だった

---
# 今回の目標

fork爆弾爆発中のロードアベレージをリアルタイムで見たい

---
## ロードアベレージを見る方法
* uptime / w
* top
* カーネルダンプの解析

---
## uptime

実行時点の稼働時間、ロードアベレージを表示  

## w

uptimeの結果+ログインユーザー一覧を表示

---
## top

ロードアベレージだけでなくプロセス一覧、CPU使用率、メモリ使用率を一定間隔で更新して表示  
一言で言えば「多機能だが重い」

## カーネルダンプの解析

ダンプした時点のuptime情報も同時に記録されている
何度もダンプできないので、リアルタイム計測は不可能

---
### fork爆弾実行中にリアルタイムでロードアベレージを見る大きな壁

* CPU / メモリ リソースはfork爆弾によるプロセス生成ですぐに枯渇するから
 * プロセスの起動もままならないのでuptimeやwは使用不能
 * topでも割り当てられるCPUリソースがなくなるので応答停止

---
## 大きな壁を越える手段：cgroup

---
# cgroup

* Linuxカーネルに搭載されている強力なリソース制御機能  
※CPUリソースは、優先度設定で使われるnice値とは比べものにならないほど強力
* Linuxカーネル 2.6.24から標準搭載
* リソース制限の対象: CPU、メモリ、ディスクI/O、ネットワーク・・・
* コンテナ仮想化で重要な機能

---
## できることの一例

* 特定のユーザーのリソースの制限
* 特定ユーザーの特定プロセスに対するリソースの制限  
→fork爆弾が発生しても制限が可能になる

---
# cgroup設定方針

* fork爆弾でCPUリソースを埋め尽くさないよう、1コアだけ使用しないようにする  
※CPU使用率で制御するより単純！
* fork爆弾でメモリを埋め尽くさないよう、メモリ上限を設定する

---
# デモ環境

* CPU: 4コア (VM)
* メモリ: 16GB
* OS: CentOS 7

---
# インストール

``` ShellSession
# yum install libcgroup
# yum install libcgroup-tools
```

※自動起動を有効にする場合は以下を実行

``` ShellSession
# systemctl enable cgconfig
# systemctl enable cgred
```

---
## 設定例

/etc/cgconfig.conf

```
group forkbomb {
  cpuset {
    # 使用するCPUコアの指定
    cpuset.cpus	= "0-2";
    # 使用するメモリノードの指定(NUMAでなければ0でよい。必須)
    cpuset.mems = "0";
  }
  memory {
    # 上限1GB
    memory.limit_in_bytes = 1073741824;
  }
}
```

/etc/cgrules.conf

```
root:bash cpuset,memory /forkbomb
```

---
## 設定の反映

``` ShellSession
# systemctl restart cgconfig
# systemctl restart cgred
```

## 対象プロセスの確認
``` ShellSession
$ cat /sys/fs/cgroup/cpuset/forkbomb/tasks
```

---
## <デモンストレーション>

---
# 設定上の注意
* cgconfig.confによる設定はCentOS7では非推奨とされている
* fork爆弾実行時のシェルと、ログインで使用するシェルは別にしなければならない  
メモリ制限に引っかかりcgroupがシェルを強制終了させるから  
(デモではcgroup制御対象はbash、ログインシェルはzsh)

---
# まとめ

* CPUやメモリを食い尽くすfork爆弾の制御はcgroupで!

---
# 参考

* 「Red Hat Enterprise Linux 7 リソース管理ガイド」  
https://access.redhat.com/documentation/ja-JP/Red_Hat_Enterprise_Linux/7/pdf/Resource_Management_Guide/Red_Hat_Enterprise_Linux-7-Resource_Management_Guide-ja-JP.pdf
* 「cgroupsを利用したリソースコントロール入門 — | サイオスOSS | サイオステクノロジー」  
https://oss.sios.com/yorozu-blog/cgroups-20150708
