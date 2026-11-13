# Chaos Mesh（Kubernetes の上で、わざと障害を起こして試す）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Tools（開発や運用に使う道具） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

Chaos Mesh は、Kubernetes（コンテナを多数のサーバーで動かすための管理の仕組み）の上で動くシステムに、**わざと障害を起こして、システムが耐えられるかを確かめる**ためのオープンソースの道具です。こうした試し方を**カオスエンジニアリング**と呼びます。CNCF（クラウド向けのオープンソースを育てる団体）の incubating（育成中）の段階のプロジェクトで、ライセンスは Apache 2.0 です（[GitHub: chaos-mesh/chaos-mesh](https://github.com/chaos-mesh/chaos-mesh)）。

たとえるなら、本物の火事を待たずに、決まった手順で避難訓練をくり返すようなものです（筆者の見立て）。

## 2. 解決する課題（なぜ生まれたか）

多数の部品が通信し合うシステムでは、サーバーが1台止まる、ネットワークが遅くなる、ディスクが遅くなるといった障害が、いつか必ず起きます。設計の上では「1台止まっても大丈夫」のはずでも、実際に止めてみないと、本当に耐えられるかは分かりません（筆者の見立て）。

かといって、手作業で障害を起こすのは危険で、同じ条件をくり返すのも難しくなります。Chaos Mesh は、起こす障害を Kubernetes の設定のファイルとして書けるようにし、同じ実験をくり返し、影響の範囲を絞って実行できるようにします。Chaos Mesh はもともと、分散データベースの TiDB をテストする仕組みから生まれました（[Chaos Mesh: Overview](https://chaos-mesh.org/docs/)）。

Thoughtworks は、GPU（AI の計算に使う演算装置）を多く積んだ AI の基盤のような環境でも使えることを挙げています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/chaos-mesh)）。

## 3. 仕組みの概要

**構成要素**（[Chaos Mesh: Overview](https://chaos-mesh.org/docs/)「Architecture overview」）

- **Chaos Dashboard**: 実験を作ったり、様子を見たりする Web の画面と API。権限の管理を持つ。Kubernetes を直接操作できるなら使わなくてもよい
- **Chaos Controller Manager**: 中心の部品。実験とその予定を管理し、障害の種類ごとの処理を受け持つ
- **Chaos Daemon**: 各サーバー（ノード）で動き、対象のコンテナの中に入り込んで、ネットワークやファイルシステム、カーネルなどに手を加える。既定では強い権限（特権）で動く

**起こせる障害の例**（[GitHub: chaos-mesh/chaos-mesh](https://github.com/chaos-mesh/chaos-mesh) の一覧）

| 種類 | 内容の例 |
|---|---|
| PodChaos | コンテナのまとまり（Pod）を止める、使えなくする |
| NetworkChaos | 通信を遅らせる、切る |
| StressChaos | CPU やメモリに負荷をかける |
| IOChaos | ファイルの読み書きを遅らせる、失敗させる |
| TimeChaos | 時計をずらす |
| DNSChaos、HTTPChaos、JVMChaos、KernelChaos | 名前の解決、HTTP、Java のアプリ、Linux のカーネルへの障害 |
| BlockChaos | ブロックデバイス（ディスク）への障害 |
| AWSChaos、AzureChaos、GCPChaos | クラウドのサービスへの障害 |

物理のサーバー向けには、実験的な道具の Chaosd があり、Chaos Mesh からは PhysicalMachineChaos で呼び出せます（[Chaos Mesh の公式サイト](https://chaos-mesh.org/)）。

**実験の書き方の例**（[Chaos Mesh: Simulate Pod Faults](https://chaos-mesh.org/docs/simulate-pod-chaos-on-kubernetes/)）

実験は Kubernetes の独自のリソース（CRD）として YAML で書きます。たとえば PodChaos では、次の項目を指定します。

- `action`: 何をするか（`pod-kill` で Pod を止める、`pod-failure` で一定時間使えなくする、`container-kill` でコンテナを止める）
- `selector`: 対象をどう選ぶか（名前空間やラベルで絞る）
- `mode`: 選んだ中からいくつに起こすか（`one` で1つ、`all` ですべて、`fixed` で決めた数、`fixed-percent` で決めた割合など）
- `duration`: どれだけ続けるか（`pod-failure` の場合）

同じページは、次の注意も挙げています。`pod-kill` は、止めた Pod を自動で作り直す仕組み（ReplicaSet など）があることが前提です。`pod-failure` は、対象のコンテナのイメージを何もしないイメージに差し替えて使えなくするので、コンテナの生存や準備の確認（livenessProbe・readinessProbe）を設定しておくことが勧められています。

**ワークフロー**（[Chaos Mesh: Create Chaos Mesh Workflow](https://chaos-mesh.org/docs/create-chaos-mesh-workflow/)）

複数の実験を、順番に、または同時に実行する流れを組めます。決まった間隔でくり返す予定（Schedule）や、システムの健全さを確かめる点検（StatusCheck）を入れて、設定すれば、異常があったときに途中で止めることもできます（既定では止まりません）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: システムの「障害への強さ（レジリエンス）を確かめる試験の層」です。テストの自動化と同じように、障害の実験をコードとして管理し、CI や定期の実行に組み込む使い方ができます（筆者の見立て）。

**得られるもの**

- **宣言的に書ける**: 障害を Kubernetes の設定として書くので、既存の道具や版管理でそのまま扱える（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/chaos-mesh)）
- **幅の広さ**: Pod・ネットワーク・負荷・ファイル・時刻・クラウドなど、多くの種類の障害を1つの道具で起こせる（[GitHub: chaos-mesh/chaos-mesh](https://github.com/chaos-mesh/chaos-mesh) の一覧）
- **再現性と範囲の制御**: 同じ実験をくり返せて、影響の範囲（blast radius）を絞れるので、大規模で自動のくり返しの試験に向く（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/chaos-mesh)）

**代わりに失うもの・注意点**

- **強い権限**: Chaos Daemon は既定で特権で動き、対象のコンテナの中に入り込む。道具そのものが乗っ取られたときの影響が大きい（[Chaos Mesh: Overview](https://chaos-mesh.org/docs/) の記述に基づく筆者の見立て）
- **本番での実施の重さ**: 本物の利用者がいる環境で障害を起こすには、範囲の制限・すぐ止める手順・関係者への周知が要る（筆者の見立て）
- **Kubernetes が前提**: Kubernetes の外のシステムは、Chaosd などの別の手段が要り、Chaosd は実験的な道具とされている（[Chaos Mesh の公式サイト](https://chaos-mesh.org/)）
- **観測が前提**: 障害を起こしても、システムの様子を測る仕組み（監視）が無ければ、何が起きたか分からない（筆者の見立て）

## 5. 採用を判断するときの観点

- **前提**: Kubernetes の上でシステムを動かしているか。監視と警報が整っているか（筆者の見立て）
- **始める場所**: まず検証用の環境で、影響の小さい実験（Pod を1つ止めるなど）から始める（筆者の見立て）
- **範囲の制限**: 実験を許す名前空間を絞る。公式サイトは、名前空間の許可リストで影響の範囲を制御できるとしている（[Chaos Mesh の公式サイト](https://chaos-mesh.org/)）
- **権限の管理**: 誰が実験を作り、実行できるかを、権限の設定で絞る。公式サイトは、権限の管理（RBAC）が既定で有効だとしている（[Chaos Mesh の公式サイト](https://chaos-mesh.org/)）
- **仮説を先に書く**: 「この Pod を止めても、利用者の応答は○秒以内に戻る」のように、確かめたいことを先に決めてから実験する（筆者の見立て）

## 6. 関連技術・代替技術

- **Chaos Engineering**: [Radar の Chaos Mesh のページ](https://www.thoughtworks.com/en-us/radar/tools/chaos-mesh)の関連の項目に挙がっています（手法としてのカオスエンジニアリング）
- **代替: クラウドの障害注入のサービス**: 大手のクラウドは、自社のサービス向けに障害を起こすサービスを用意している。クラウドの資源が中心ならそちらも候補になる（筆者の見立て）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: GPU を多く使う AI の基盤のような環境も含め、大規模にくり返す試験の道具として評価しています。理由には、障害を Kubernetes の設定として書けることと、実験を再現でき影響の範囲を絞れることを挙げています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/tools/chaos-mesh

## 8. 深掘りするときの一次情報

- [Chaos Mesh の公式サイト](https://chaos-mesh.org/)
- [Chaos Mesh: Overview](https://chaos-mesh.org/docs/)
- [Chaos Mesh: Simulate Pod Faults](https://chaos-mesh.org/docs/simulate-pod-chaos-on-kubernetes/)
- [Chaos Mesh: Create Chaos Mesh Workflow](https://chaos-mesh.org/docs/create-chaos-mesh-workflow/)
- [GitHub: chaos-mesh/chaos-mesh](https://github.com/chaos-mesh/chaos-mesh)

## 9. 確認日

- 内容の確認日: 2026-11-13
- Radar の評価の確認日: 2026-11-13

## 用語集

| 用語 | 説明 |
|---|---|
| カオスエンジニアリング | わざと障害を起こして、システムが耐えられるかを確かめ、弱点を見つける手法 |
| Kubernetes | コンテナを多数のサーバーで動かし、配置や再起動を自動で管理する仕組み |
| Pod | Kubernetes で、いっしょに動くコンテナのまとまり。動かす単位 |
| CRD（カスタムリソース） | Kubernetes に、独自の種類の設定を足す仕組み。Chaos Mesh は実験をこれで表す |
| 名前空間 | Kubernetes の中で、資源を区切って管理するための区画 |
| ラベル | Kubernetes の資源に付ける目印。対象を選ぶのに使う |
| CNCF | クラウド向けのオープンソースを育てる団体。段階（sandbox・incubating・graduated）がある |
| 特権 | コンテナが、サーバーの深い部分まで操作できる強い権限 |
| RBAC | 役割ごとに、できる操作を決めて権限を管理する仕組み |
| レジリエンス | 障害が起きても、機能を保つか、すぐに回復できる強さ |
| blast radius（影響の範囲） | 障害や変更が影響を及ぼす範囲 |
| GPU | 大量の計算を並べて行う演算装置。AI のモデルの学習や実行に使う |
