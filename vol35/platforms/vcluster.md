# vCluster（1つの Kubernetes の中に、利用者ごとの「仮想の Kubernetes」を作る仕組み）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Platforms（アプリケーションを載せて動かす基盤・サービス） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

vCluster は、既存の Kubernetes のクラスタの上に、利用者（テナント）ごとの**仮想のクラスタ**を作る道具です。各テナントのクラスタは、自分専用の API サーバー・CRD（独自のリソースの定義）・RBAC（権限の設定）を持ちます（[GitHub: loft-sh/vcluster](https://github.com/loft-sh/vcluster)、[vCluster とは](https://www.vcluster.com/docs/vcluster/introduction/what-is-vcluster/)）。オープンソース版は Apache 2.0 のライセンスで、リポジトリの著作権の表示は vCluster Labs です（[GitHub: loft-sh/vcluster](https://github.com/loft-sh/vcluster)）。

たとえるなら、1棟の建物（物理のクラスタ）の中に、それぞれ自分の受付と鍵の管理を持つテナントの事務所を作るようなものです。受付（API サーバー）は事務所ごとに別ですが、電気や水道（ノードやネットワーク）は、設定しだいで建物と共有します（たとえは筆者の見立て）。

## 2. 解決する課題（なぜ生まれたか）

Kubernetes を複数のチームや顧客で分け合うとき、これまでの選択肢は主に2つでした。1つは、1つのクラスタを名前空間（namespace）で分ける方法です。安上がりですが、CRD のようにクラスタ全体に効く設定は分けられず、利用者がクラスタの管理者の権限を持てません。もう1つは、利用者ごとにクラスタを丸ごと用意する方法で、隔離は強い代わりに費用と運用の手間が増えます（筆者の見立て）。

vCluster は、その中間を狙います。コントロールプレーン（クラスタの頭脳の部分）はテナントごとに分け、実際にコンテナを動かす部分は、必要に応じて共有します（[アーキテクチャ](https://www.vcluster.com/docs/vcluster/introduction/architecture/)）。

## 3. 仕組みの概要

### 3.1 コントロールプレーンは Pod の中で動く

各テナントのクラスタのコントロールプレーンは、土台のクラスタ（ホスト）の名前空間の1つに、既定では StatefulSet（状態を持つ Pod の配置の単位。設定で Deployment も可）として置かれます。API サーバー・コントローラーマネージャー・データの保存先・**syncer** が、1つの Pod の1つのコンテナの中で、1つのプロセスとして動きます。名前の解決をする CoreDNS は、同じ名前空間の別の Pod です（[アーキテクチャ](https://www.vcluster.com/docs/vcluster/introduction/architecture/)）。

データの保存先は、既定では埋め込みの SQLite で、ほかに etcd・MySQL・PostgreSQL などを選べます（埋め込みの etcd は有料版の機能）。保存先は作るときに決め、あとからの移行には制約があります（[配置の基本](https://www.vcluster.com/docs/vcluster/deploy/basics)）。ホストのクラスタは Kubernetes v1.28 以上が必要で、作るのにクラスタ全体の管理者の権限は要りません（[配置の基本](https://www.vcluster.com/docs/vcluster/deploy/basics)）。

### 3.2 syncer がホストへ写す

テナントの中で作った Pod は、そのままではどこでも動きません。syncer が、その Pod をホストの名前空間に写し、ホストのスケジューラーとノードで動かします。既定で写すのは、Pod・Secret・ConfigMap・Service・Endpoints・PersistentVolumeClaim です。Ingress・NetworkPolicy・CRD で定義した独自のリソースなどは、既定では写さず、テナントの中だけに置きます（[ホストへの同期](https://www.vcluster.com/docs/vcluster/configure/vcluster-yaml/sync/to-host/)）。

既定の（ホストの名前空間を1つだけ使う）モードでは、ホストの側で名前がぶつからないよう、写すときに名前を付け替えます（[ホストへの同期](https://www.vcluster.com/docs/vcluster/configure/vcluster-yaml/sync/to-host/)）。テナントの利用者は、自分のクラスタの管理者として CRD を入れても、ほかのテナントには見えません（[GitHub: loft-sh/vcluster](https://github.com/loft-sh/vcluster)）。

### 3.3 ノードの分け方

| 方式 | 内容 | 隔離の強さ |
|---|---|---|
| 共有ノード | ホストの既存のノードで、ほかのテナントと一緒に動く。オープンソース版はこの方式 | カーネル・ノード・ネットワークを共有する |
| 専用ノード | ホストのノードのうち、ラベルで選んだ一部だけを使う | 置き場所を絞るだけで、隔離ではない |
| プライベートノード | テナント専用のノードをテナントのクラスタに参加させる。ネットワークや保存の仕組みもテナントごと | ノードごと分かれる。ライセンスが要る（無料の Free プランでも使える） |

（[アーキテクチャ](https://www.vcluster.com/docs/vcluster/introduction/architecture/)、[vCluster とは](https://www.vcluster.com/docs/vcluster/introduction/what-is-vcluster/)）

### 3.4 入れ方

CLI の `vcluster create` で作るほか、Helm のチャートとしても入れられ、Terraform・Argo CD・Flux から扱う手順も公式にあります（[配置の基本](https://www.vcluster.com/docs/vcluster/deploy/basics)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: 名前空間による分割と、クラスタを丸ごと分ける方法の間にある、マルチテナンシー（1つの基盤を複数の利用者で分け合うこと）の層です（[アーキテクチャ](https://www.vcluster.com/docs/vcluster/introduction/architecture/)。中間という整理は筆者の見立て）。

**得られるもの**

- **利用者が管理者になれる**: テナントは自分のクラスタの中で、CRD や権限を自由に設定できます（[GitHub: loft-sh/vcluster](https://github.com/loft-sh/vcluster)）
- **クラスタを安く増やせる**: コントロールプレーンは主に Pod 1つ（と名前解決の CoreDNS の Pod）なので、開発・試験・CI のための使い捨てのクラスタを手早く作れます（[アーキテクチャ](https://www.vcluster.com/docs/vcluster/introduction/architecture/)、[料金](https://www.vcluster.com/pricing)）
- **ふだんの道具が使える**: テナントのクラスタは、kubectl や Helm からふつうのクラスタとして扱えます（[vCluster とは](https://www.vcluster.com/docs/vcluster/introduction/what-is-vcluster/)）

**代わりに失うもの・注意点**

- **共有ノードは安全の境界ではない**: 公式の堅牢化の手引きは、外部の顧客や信頼できない利用者が相手なら、共有ノードはプライベートノードの代わりにならず、堅牢化しても信頼の境界は変わらない、としています。カーネル・kubelet・ホストのネットワークなどを共有するためです（[共有ノードの堅牢化](https://www.vcluster.com/docs/vcluster/security/shared-nodes-hardening)）
- **追加の対策が要る**: 共有ノードでは、ノードのパスを読み書きできる hostPath の制限、ネットワークポリシー、クラウドのメタデータの窓口（169.254.169.254）への通信の制限などを、別に考える必要があります（[共有ノードの堅牢化](https://www.vcluster.com/docs/vcluster/security/shared-nodes-hardening)）
- **2層の運用**: 問題が起きたとき、テナントのクラスタとホストのクラスタの両方を見る必要があり、syncer がその間の翻訳をしている分、原因を追いにくくなります（筆者の見立て）
- **あとから変えにくい設定**: ノードの方式やデータの保存先は、作るときにしか決められません（[配置の基本](https://www.vcluster.com/docs/vcluster/deploy/basics)）

## 5. 採用を判断するときの観点

- **利用者を信頼できるか**: 社内の開発・試験・CI のように信頼できる利用者なら、共有ノードで足ります。外部の顧客に貸すなら、プライベートノードを前提にします（[共有ノードの堅牢化](https://www.vcluster.com/docs/vcluster/security/shared-nodes-hardening)）
- **費用**: オープンソース版は無料です。商用の vCluster Platform には、64 CPU か 32 GPU までの無料のプランと、価格が個別見積もりの Enterprise のプランがあります。プライベートノードなどの機能は、ライセンス（無料の Free プランを含む）が要り、Free プランはライセンスを確かめるために vCluster Platform へつなぐ必要があります（[料金](https://www.vcluster.com/pricing)、[OSS と無料版](https://www.vcluster.com/docs/vcluster/introduction/oss-vs-free)）。価格は確認日の時点のものです
- **可用性**: テナントのクラスタの頭脳は、ホストの上の Pod です。ホストが止まれば、すべてのテナントが止まります（筆者の見立て）
- **更新**: 2026年11月12日の時点で、GitHub が "Latest" と示す版は v0.37.6（2026-10-30）で、11月10日には v0.37.7 も出ています。v0.38 は候補版の段階です（[リリース](https://github.com/loft-sh/vcluster/releases)）。版の番号が 1.0 未満で、更新の頻度も高いので、上げる前に変更点を読む手間を見込みます（筆者の見立て）

## 6. 関連技術・代替技術

- **名前空間による分割**: 最も軽い分け方です。CRD などクラスタ全体の設定は分けられません（筆者の見立て）
- **利用者ごとのクラスタ**: クラウドの管理された Kubernetes を利用者ごとに用意する方法です。隔離は強い代わりに、クラスタの数だけ費用と運用が増えます（筆者の見立て）
- **Talos Linux**: 自前のクラスタのノードの OS です。プライベートノードのように物理のノードを分ける構成で、土台として組み合わせられます（筆者の見立て。[Talos Linux の解説](talos-linux.md)）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: 管理された Kubernetes のサービスや AI のクラウドの環境を作る中で、隔離と費用のつり合いを取る手段として本番で成果を上げたことから、Trial としています。顧客向けの GPU の環境では、専用の作業用ノードでより強く隔離できるとしています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/platforms/vcluster

## 8. 深掘りするときの一次情報

- [GitHub: loft-sh/vcluster](https://github.com/loft-sh/vcluster) と [リリース](https://github.com/loft-sh/vcluster/releases)
- [vCluster とは](https://www.vcluster.com/docs/vcluster/introduction/what-is-vcluster/) と [アーキテクチャ](https://www.vcluster.com/docs/vcluster/introduction/architecture/)
- [ホストへの同期](https://www.vcluster.com/docs/vcluster/configure/vcluster-yaml/sync/to-host/) と [配置の基本](https://www.vcluster.com/docs/vcluster/deploy/basics)
- [共有ノードの堅牢化](https://www.vcluster.com/docs/vcluster/security/shared-nodes-hardening)
- [料金](https://www.vcluster.com/pricing) と [OSS と無料版](https://www.vcluster.com/docs/vcluster/introduction/oss-vs-free)

## 9. 確認日

- 内容の確認日: 2026-11-12
- Radar の評価の確認日: 2026-11-12

## 用語集

| 用語 | 説明 |
|---|---|
| テナント | 1つの基盤を分け合う利用者の単位。チームや顧客ごとに分けることが多い |
| マルチテナンシー | 1つの基盤を、互いに干渉しないように複数のテナントで分け合うこと |
| コントロールプレーン | Kubernetes のクラスタの頭脳の部分。API サーバーや、状態をそろえるコントローラー、データの保存先からなる |
| 名前空間（namespace） | 1つのクラスタの中を、名前と権限の範囲で区切る仕組み |
| CRD | Custom Resource Definition。Kubernetes に独自の種類のリソースを足すための定義。クラスタ全体に効く |
| RBAC | Role-Based Access Control。役割ごとに、できる操作を決める権限の仕組み |
| Pod | Kubernetes でコンテナを動かす最小の単位。1つ以上のコンテナをまとめたもの |
| kubelet | 各ノードで動き、割り当てられた Pod を実際に起動・監視する Kubernetes の部品 |
| StatefulSet | 名前や保存領域を保ったまま Pod を動かすための、Kubernetes の配置の単位 |
| syncer | vCluster の中で、テナントのクラスタのリソースをホストのクラスタへ写し、状態を行き来させる部品 |
| スケジューラー | Pod をどのノードで動かすかを決める Kubernetes の部品 |
| hostPath | ノードのファイルシステムの場所を、そのまま Pod に見せる仕組み。使い方によってはノードの中身に触れられる |
| ネットワークポリシー | Pod どうしや外部との通信を、許す相手を決めて絞る Kubernetes の仕組み |
