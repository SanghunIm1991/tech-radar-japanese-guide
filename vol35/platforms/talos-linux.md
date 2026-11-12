# Talos Linux（Kubernetes を動かすためだけの、書き換えられない最小限の Linux）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Platforms（アプリケーションを載せて動かす基盤・サービス） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

Talos Linux は、**Kubernetes を動かすことだけに絞った Linux** のディストリビューション（OS としてまとめて配られる形）です。公式は、安全・イミュータブル（書き換えられない）・最小限の OS だとしています（[GitHub: siderolabs/talos](https://github.com/siderolabs/talos)）。SSH もシェルも無く、管理はすべて API を通して行います（[GitHub: siderolabs/talos](https://github.com/siderolabs/talos)）。パッケージも無く、設定できることはすべて1つの YAML の文書に書きます（[設計の考え方](https://docs.siderolabs.com/talos/v1.14/learn-more/philosophy)）。

開発元は Sidero Labs で、ライセンスは MPL-2.0 です（[GitHub: siderolabs/talos](https://github.com/siderolabs/talos)）。ほかのディストリビューションを土台にせず、OS の利用者側の部分（ユーザー空間）を主に Go で一から書いています（[設計の考え方](https://docs.siderolabs.com/talos/v1.14/learn-more/philosophy)）。

たとえるなら、ふつうの Linux が「工具箱つきの作業部屋」だとすると、Talos は「Kubernetes という機械が1台だけ置かれた、扉に鍵のかかった機械室」です。中に入って手でいじることはできず、外の操作盤（API）からだけ指示を出します（たとえは筆者の見立て）。

## 2. 解決する課題（なぜ生まれたか）

Kubernetes のノード（クラスタを構成する1台1台の機械）に汎用の Linux を使うと、SSH で入って手で設定を変える、パッケージを足す、といった操作が積み重なります。その結果、ノードごとに少しずつ設定がずれ（構成のずれ）、攻撃を受けうる入口も増えます（筆者の見立て）。

Talos は、そもそも手で入る手段を無くし、設定は1つの YAML の文書で宣言し、更新は OS のイメージを丸ごと入れ替える形にしています（[設計の考え方](https://docs.siderolabs.com/talos/v1.14/learn-more/philosophy)、[Talos の更新](https://docs.siderolabs.com/talos/v1.14/configure-your-talos-cluster/lifecycle-management/upgrading-talos)）。ノードは「直して使うもの」ではなく「同じ設定から作り直せるもの」になります（筆者の見立て）。

## 3. 仕組みの概要

### 3.1 書き換えられないルートファイルシステム

OS の本体は、読み取り専用の SquashFS（圧縮された読み取り専用のファイルシステム）で、メモリに載せて使います。書き込めるのは、起動のたびに作り直されるメモリ上の一時的な領域（`/system` など）と、ディスク上の `/var` だけです（[アーキテクチャ](https://docs.siderolabs.com/talos/v1.14/learn-more/architecture)）。公式は、イメージの大きさを SquashFS で80MB未満としています（[設計の考え方](https://docs.siderolabs.com/talos/v1.14/learn-more/philosophy)）。

ディスクは役割ごとに分かれています（[アーキテクチャ](https://docs.siderolabs.com/talos/v1.14/learn-more/architecture)）。

| パーティション | 中身 |
|---|---|
| BOOT など | 起動に使う部分 |
| META | ノードのメタデータ |
| STATE | マシンの設定とノードの識別情報 |
| EPHEMERAL | `/var` として使う書き込み領域。kubelet・etcd・コンテナの実行環境のデータが置かれる |

### 3.2 systemd の代わりの machined

カーネルが最初に起動するのは、systemd ではなく `machined` という Talos 独自のプロセスです（[設計の考え方](https://docs.siderolabs.com/talos/v1.14/learn-more/philosophy)）。etcd や kubelet はこの machined がシステムのサービスとして管理し、Kubernetes のコントロールプレーンの部品（API サーバーなど）は、Talos が中で作る定義をもとに static Pod として動きます（[既定の堅牢化と CIS](https://docs.siderolabs.com/talos/v1.14/security/talos-default-hardening-and-cis-compliance)）。内部の部品どうしは、ローカルのソケットの上の gRPC で通信します（[アーキテクチャ](https://docs.siderolabs.com/talos/v1.14/learn-more/architecture)）。

### 3.3 API と talosctl での管理

管理の操作は、相互 TLS（双方が証明書で相手を確かめる暗号化通信）で守られた API を通して行い、手元からは `talosctl` というコマンドを使います（[GitHub: siderolabs/talos](https://github.com/siderolabs/talos)、[Talos の更新](https://docs.siderolabs.com/talos/v1.14/configure-your-talos-cluster/lifecycle-management/upgrading-talos)）。パスワードは無く、証明書は短い期限で自動的に入れ替わります（[設計の考え方](https://docs.siderolabs.com/talos/v1.14/learn-more/philosophy)）。

### 3.4 OS の更新は A/B の入れ替え

更新は、新しいインストーラーのイメージを指定して API を呼ぶだけです。ノードは自分の上の Pod を退避させてから新しいイメージで起動し直し、前の版のイメージも残します。新しい版で起動できなければ、前の版に戻ります（[Talos の更新](https://docs.siderolabs.com/talos/v1.14/configure-your-talos-cluster/lifecycle-management/upgrading-talos)）。

コントロールプレーンでは、etcd の過半数を失うような更新を拒み、1台ずつ更新します。OS を更新しても Kubernetes の版は変わらず、Kubernetes の更新は別に行います（[Talos の更新](https://docs.siderolabs.com/talos/v1.14/configure-your-talos-cluster/lifecycle-management/upgrading-talos)）。

### 3.5 足りない部品は Image Factory で足す

システムの拡張（ドライバーなど）やカーネルの引数を足したいときは、**Image Factory** に「どの拡張を入れるか」を書いた設計図（schematic）を渡し、それを含むイメージを作ってもらいます。ISO・ディスクのイメージ・インストーラーのイメージなどを作れ、Image Factory を自分たちで動かすこともできます（[Image Factory](https://docs.siderolabs.com/talos/v1.14/learn-more/image-factory)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: Kubernetes のノードの OS の層です。クラウドの管理された Kubernetes では利用者がノードの OS を選ばないことが多く、Talos が効いてくるのは、自分でノードを持ってクラスタを組む場合です（筆者の見立て）。

**得られるもの**

- **攻撃の入口が少ない**: SSH・シェル・GNU の道具が無く、カーネルは KSPP（カーネルの自己防御の取り組み）の推奨に沿って設定されています（[設計の考え方](https://docs.siderolabs.com/talos/v1.14/learn-more/philosophy)、[既定の堅牢化と CIS](https://docs.siderolabs.com/talos/v1.14/security/talos-default-hardening-and-cis-compliance)）
- **構成がずれない**: 設定は宣言した文書だけで決まり、手での変更が入りません（[設計の考え方](https://docs.siderolabs.com/talos/v1.14/learn-more/philosophy)）
- **更新を戻せる**: A/B の入れ替えで、起動に失敗しても前の版に戻れます（[Talos の更新](https://docs.siderolabs.com/talos/v1.14/configure-your-talos-cluster/lifecycle-management/upgrading-talos)）

**代わりに失うもの・注意点**

- **調べものの作法が変わる**: 中に入って道具を使うことはできません。道具が要るときは、自分で用意したイメージの調査用のコンテナを特権つきで動かします（[talosctl debug](https://docs.siderolabs.com/talos/v1.14/troubleshooting/talosctl-debug)）。手に慣れた運用からの切り替えには学び直しが要ります（筆者の見立て）
- **設定の形式が動いている**: 1.14 では、これまでの設定の文書（v1alpha1）の多くの項目が非推奨になり、新しい種類の文書に移っています（[Talos の更新](https://docs.siderolabs.com/talos/v1.14/configure-your-talos-cluster/lifecycle-management/upgrading-talos)）
- **版の追従**: 各版のコミュニティの支援は、次の小版が出るまでで終わります。Talos 1.14 が対応する Kubernetes は 1.33〜1.37 です（[対応表](https://docs.siderolabs.com/talos/v1.14/getting-started/support-matrix)）
- **一斉に再起動しない**: 公式は、全ノードを同時に更新しないよう注意しています。Rook/Ceph のような保存の仕組みは、何台もの再起動に耐えられないことがあるためです（[Talos の更新](https://docs.siderolabs.com/talos/v1.14/configure-your-talos-cluster/lifecycle-management/upgrading-talos)）

## 5. 採用を判断するときの観点

- **ノードを自分で持つか**: クラウドの管理された Kubernetes を使い、ノードの OS に触れないなら、得るものは小さいです。物理サーバーや自前の仮想基盤でクラスタを組むなら候補になります（筆者の見立て）
- **動かす場所**: amd64 と arm64 に対応し、主なクラウド、VMware や Proxmox などの仮想基盤、物理サーバー、Raspberry Pi などの小型のボードで動きます。自動で試験しているのは物理サーバー・AWS・GCP です（[対応表](https://docs.siderolabs.com/talos/v1.14/getting-started/support-matrix)）
- **更新の体制**: 小版を飛ばさずに、各小版の最新の修正版を順にたどって上げるよう、公式は勧めています（[Talos の更新](https://docs.siderolabs.com/talos/v1.14/configure-your-talos-cluster/lifecycle-management/upgrading-talos)）。2026年11月の時点で "Latest" の版は v1.14.6（2026-11-11）です（[リリース](https://github.com/siderolabs/talos/releases)）
- **費用**: Talos そのものは MPL-2.0 で無料です。開発元は、多数のクラスタの用意・更新・設定・バックアップをまとめて扱う **Omni**（[Omni](https://www.siderolabs.com/omni/)。10ノード・2人までの無料のプラン、50ノードまで1ノード月25ドルのプランなど）と、FIPS 140-3 対応の版や支援をつけた有料の版（1ノード年1,000ドル、10ノードから）を売っています（[料金](https://www.siderolabs.com/pricing)）。価格は確認日の時点のものです
- **開発元への依存**: OS もその管理の道具も1社が中心に作っているので、その会社の方針の変化を見ておく必要があります（筆者の見立て）

## 6. 関連技術・代替技術

- **Bottlerocket**: コンテナを動かすことに特化した、Linux をもとにした OS です。サイトの著作権の表示は Amazon です（[Bottlerocket](https://bottlerocket.dev/)）
- **Flatcar Container Linux**: コンテナを動かすための、自動で更新される最小限の Linux です（[Flatcar](https://www.flatcar.org/)）
- **汎用の Linux と kubeadm など**: 手慣れた道具と自由がある代わりに、ノードの設定をそろえ続ける手間を自分で負います（筆者の見立て）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: 大規模な物理サーバーの群れで自前の Kubernetes を Talos で動かし、コントロールプレーンを楽に用意できた経験から、Trial としています。主に自分でノードを動かすチーム向けとし、開発元の Sidero Labs が2026年に買収されたため、今後の方向を見ておくよう注意しています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/platforms/talos-linux

## 8. 深掘りするときの一次情報

- [GitHub: siderolabs/talos](https://github.com/siderolabs/talos) と [リリース](https://github.com/siderolabs/talos/releases)
- [Talos のドキュメント](https://docs.siderolabs.com/talos/) と [設計の考え方](https://docs.siderolabs.com/talos/v1.14/learn-more/philosophy)
- [アーキテクチャ](https://docs.siderolabs.com/talos/v1.14/learn-more/architecture) と [Talos の更新](https://docs.siderolabs.com/talos/v1.14/configure-your-talos-cluster/lifecycle-management/upgrading-talos)
- [既定の堅牢化と CIS](https://docs.siderolabs.com/talos/v1.14/security/talos-default-hardening-and-cis-compliance) と [対応表](https://docs.siderolabs.com/talos/v1.14/getting-started/support-matrix)

## 9. 確認日

- 内容の確認日: 2026-11-12
- Radar の評価の確認日: 2026-11-12

## 用語集

| 用語 | 説明 |
|---|---|
| ディストリビューション | Linux のカーネルに、道具や設定をまとめて、OS として配る形にしたもの |
| イミュータブル | 動かしている間に中身を書き換えない性質。変えたいときは丸ごと新しいものに入れ替える |
| ノード | Kubernetes のクラスタを構成する1台1台の機械（物理サーバーや仮想マシン） |
| 構成のずれ | 同じはずの機械どうしで、手作業の変更などにより設定が少しずつ食い違っていくこと |
| SquashFS | 圧縮された、読み取り専用のファイルシステム |
| systemd | 多くの Linux で最初に起動し、ほかのサービスを起動・管理するプログラム |
| etcd | Kubernetes がクラスタの状態を保存する、分散型のキーと値のデータベース |
| kubelet | 各ノードで動き、割り当てられた Pod を実際に起動・監視する Kubernetes の部品 |
| static Pod | Kubernetes の API を通さず、ノードの上の定義から kubelet が直接起動する Pod |
| gRPC | Google が始めた、プログラムどうしが互いの関数を呼び出すための通信の仕組み |
| 相互 TLS | 通信の両側が証明書を出し合い、互いに相手を確かめたうえで暗号化する方式 |
| KSPP | Kernel Self-Protection Project。Linux のカーネル自身を攻撃から守る設定や機能を進める取り組み |
| FIPS 140-3 | 米国政府が定める、暗号のモジュールの安全性の基準 |
