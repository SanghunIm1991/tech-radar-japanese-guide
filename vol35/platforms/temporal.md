# Temporal（長く続く業務の流れを、障害があっても最後までやり通させる耐久実行の基盤）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Platforms（アプリケーションを載せて動かす基盤・サービス） |
| リング | Trial（前の版には無く、再び載った。直前の掲載は2022年3月の版で Assess） |
| 状態 | 下書き |

## 1. 一言でいうと

Temporal は、アプリのコードの**耐久実行（durable execution）**を保証する基盤です。ネットワークの障害やサーバーの停止があっても、ワークフロー（一連の処理の流れ）は止まったところから動き続けます（[Temporal とは](https://docs.temporal.io/temporal)）。

全体は、Go で書かれた Temporal Server とデータベースからなる **Temporal Service** と、利用者が自分で動かす **Worker** に分かれます（[Temporal とは](https://docs.temporal.io/temporal)）。Uber の Cadence から枝分かれして生まれ、Cadence の作者たちが作った Temporal Technologies が開発しています。サーバーのライセンスは MIT です（[GitHub: temporalio/temporal](https://github.com/temporalio/temporal)）。

たとえるなら、工場の現場監督です。作業員（Worker）は監督の掲示板（タスクキュー）から仕事を取ってきてこなし、監督は誰が何を終えたかを台帳（イベント履歴）に残します。作業員が倒れても、台帳を見れば別の作業員が続きを引き継げます（たとえは筆者の見立て）。

## 2. 解決する課題（なぜ生まれたか）

インフラの用意、注文から出荷まで、人の承認を挟む申請など、何分から何日もかかる処理は、途中で必ず何かが失敗します。再試行・タイムアウト・途中の状態の保存・進み具合の見える化を、処理ごとにキューとデータベースで作り込むと、本来の業務のコードより、失敗に備えるコードのほうが多くなりがちです（筆者の見立て）。

Temporal では、業務の流れをふつうのコードとして書き、失敗への備え（再試行・状態の保存・再開）は基盤が引き受けます（[Temporal とは](https://docs.temporal.io/temporal)、[Activity](https://docs.temporal.io/activities)）。

## 3. 仕組みの概要

### 3.1 Workflow と Activity

| 部品 | 役割 |
|---|---|
| Workflow | 業務の流れを書いたコード。実行のたびに、耐久性のある関数の実行（Workflow Execution）が作られる |
| Activity | 外部の API の呼び出しやメールの送信など、1つのはっきりした作業をする、ふつうの関数。既定で自動的に再試行される |
| Worker | 利用者のコードを SDK で組み込んで動かすプロセス |
| Task Queue | Worker が仕事を取りに行く待ち行列 |

（[Workflow の定義](https://docs.temporal.io/workflow-definition)、[Activity](https://docs.temporal.io/activities)、[Worker](https://docs.temporal.io/workers)）

### 3.2 Worker が仕事を取りに行く

Worker は Task Queue に問い合わせて（ポーリングして）仕事を取り出し、利用者のコードを実行して結果を返します。Temporal Service そのものは、利用者のコードを一切実行しません（[Worker](https://docs.temporal.io/workers)）。サービスの側から利用者の処理を呼びに行く形ではなく、Worker の側から取りに行く形なので、Worker は外から呼ばれる入口を開けずに済みます（筆者の見立て）。

### 3.3 イベント履歴と再生

Workflow のコードが出す指示（Activity を始める、タイマーを置く、など）は、イベントとして**イベント履歴**に追記されていきます。Worker の手元に状態が残っていないときは、コードを実行し直し、出てくる指示を履歴と照らし合わせながら、元の状態を組み立て直します。これが**再生（リプレイ）**です（[Workflow の実行](https://docs.temporal.io/workflow-execution)）。

そのため、Workflow のコードは**決定的**でなければなりません。同じ入力なら、同じ順で同じ呼び出しをする必要があります。外部の API・LLM・データベースの呼び出しは Activity に置き、時刻や乱数は SDK の API を使います（[決定性の制約](https://docs.temporal.io/workflow-definition)）。

### 3.4 再試行・タイムアウト・待ち合わせ

- **再試行**: Activity は既定で、1秒から始めて間隔を2倍ずつ延ばし（間隔の上限は既定で100秒）、回数の上限なく再試行されます。Workflow 自体は既定では再試行されません（[再試行の方針](https://docs.temporal.io/encyclopedia/retry-policies)）。そのため Activity は、何度実行されても困らない冪等な作りにするよう勧められています（[Activity の定義](https://docs.temporal.io/activity-definition)）
- **タイムアウト**: Activity の1回の試行の上限（Start-To-Close）か、全体の上限（Schedule-To-Close）のどちらかを必ず決めます。止まった処理を早く見つけるための心拍（Heartbeat）もあります（[Activity の失敗の検知](https://docs.temporal.io/encyclopedia/detecting-activity-failures)）
- **外とのやりとり**: 非同期に知らせる Signal、状態を問い合わせる Query、同期的に変更を頼む Update があります（[メッセージ](https://docs.temporal.io/sending-messages)）。人の承認を待つ流れは、これで書けます（筆者の見立て）
- **定期実行**: Schedule で決まった時刻に Workflow を始められます（[Schedule](https://docs.temporal.io/schedule)）

### 3.5 サーバーの構成

Temporal Server は4つのサービスに分かれます（[Temporal Server](https://docs.temporal.io/temporal-service/temporal-server)）。

- **Frontend**: 入口。認可・検証・流量の制限を行います
- **History**: Workflow の状態とイベント履歴を保存します。シャード（分割の単位）に分けて受け持ちます
- **Matching**: Task Queue を持ち、Worker と仕事を結びつけます
- **Worker Service**: 内部の裏方の処理を担います

必須の依存先はデータベースだけで、Cassandra・PostgreSQL・MySQL を使えます（SQLite は開発と試験のみ）。検索のための可視化の保存先には、本番では Elasticsearch が勧められています（[永続化](https://docs.temporal.io/temporal-service/persistence)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: 複数のサービスや外部の API にまたがる、長く続く処理の流れを受け持つ、オーケストレーション（指揮役）の層です（筆者の見立て）。

**得られるもの**

- **失敗への備えを基盤に任せられる**: 再試行・タイムアウト・状態の保存を、設定と SDK で扱えます（[再試行の方針](https://docs.temporal.io/encyclopedia/retry-policies)、[Activity の失敗の検知](https://docs.temporal.io/encyclopedia/detecting-activity-failures)）
- **流れがコードとして読める**: 状態の表や設定の図ではなく、ふつうのプログラムの形で流れを書けます（[Workflow の定義](https://docs.temporal.io/workflow-definition)）
- **言語を選べる**: SDK は Go・Java・Python・TypeScript・.NET・PHP・Ruby・Rust にあります（[料金](https://temporal.io/pricing)）

**代わりに失うもの・注意点**

- **決定性の制約**: Activity やタイマーの呼び出しの順を、版の管理なしに入れ替え・追加・削除すると、実行中の Workflow の再生で食い違いのエラーになります（[決定性の制約](https://docs.temporal.io/workflow-definition)）
- **版の管理が要る**: 公式が勧めるのは Worker Versioning です。Worker の版ごとに配置を分け、Workflow を古い版に固定するか、新しい版へ自動で上げるかを選びます。自動で上げる場合は、コードの中に分岐（patching）を残して再生できるようにします（[Worker Versioning](https://docs.temporal.io/production-deployment/worker-deployments/worker-versioning)）
- **履歴の上限**: 1つの Workflow の実行のイベント履歴は、51,200件か50MBまでです。長く続く流れは、Continue-As-New で新しい履歴に切り替えて続けます（[上限](https://docs.temporal.io/workflow-execution/limits)、[Continue-As-New](https://docs.temporal.io/workflow-execution/continue-as-new)）
- **運用の重さ**: 自分で動かすなら、4つのサービスとデータベース、必要に応じて Elasticsearch を運用します。History のシャードの数は、データベースを設定したあとは変えられません（[Temporal Server](https://docs.temporal.io/temporal-service/temporal-server)、[永続化](https://docs.temporal.io/temporal-service/persistence)）

## 5. 採用を判断するときの観点

- **必要性**: 数秒で終わる処理や、失敗したら最初からやり直せばよい処理には大げさです。何段階にもわたり、途中の状態を失うと困る処理ほど向いています（筆者の見立て）
- **自前か Temporal Cloud か**: 自前ならサーバーは MIT で無料ですが、上の運用を負います。Temporal Cloud は Workflow・Activity・タイマーなどの「アクション」と保存量に応じた従量制で、確認日の時点では100万アクションあたり50ドルからです（[料金](https://temporal.io/pricing)）
- **可用性**: データベースが唯一の必須の依存先なので、その冗長化が全体の可用性を決めます（筆者の見立て）
- **更新**: 2026年11月の時点で "Latest" のサーバーの版は v1.32.2 です。以前の方式の Worker Versioning の API は v1.33 で取り除かれる予定で、それを含む最後の版は v1.32 です。v1.33 へ上げる前に、今の方式の API へ移る必要があります（[リリース](https://github.com/temporalio/temporal/releases)）
- **チームの学習**: 決定性と版の管理の考え方を、Workflow を書く全員が身につける必要があります（筆者の見立て）

## 6. 関連技術・代替技術

- **Restate**: 同じく耐久実行の基盤です。Temporal は Worker が仕事を取りに行く形ですが、Restate はサーバーがサービスへ呼び出しを送る形です（[Restate の解説](restate.md)、[Restate の要求の一生](https://docs.restate.dev/guides/request-lifecycle)、[Worker](https://docs.temporal.io/workers)）
- **Cadence**: Temporal の元になった、Uber 発のワークフローの基盤です（[GitHub: temporalio/temporal](https://github.com/temporalio/temporal)）
- **Saga パターン**: 複数のサービスにまたがる処理を、段階ごとの処理と取り消しの処理の組で書くやり方です。Temporal の Workflow は、その流れを書く場所になります（筆者の見立て）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（前の版には無く、再び載った。直前の掲載は2022年3月の版で Assess）
- 過去の版: 2022年3月の版で Assess（Radar のページの推移による）
- Thoughtworks の評価の要旨: 社内の開発基盤や AI のクラウドの基盤の制御で、時間のかかるインフラの用意の処理を、再試行と耐久実行で壊れにくくできた経験から、Trial としています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/platforms/temporal

## 8. 深掘りするときの一次情報

- [Temporal とは](https://docs.temporal.io/temporal) と [Workflow の実行](https://docs.temporal.io/workflow-execution)
- [Workflow の定義（決定性の制約）](https://docs.temporal.io/workflow-definition) と [Worker](https://docs.temporal.io/workers)
- [Temporal Server](https://docs.temporal.io/temporal-service/temporal-server) と [永続化](https://docs.temporal.io/temporal-service/persistence)
- [Worker Versioning](https://docs.temporal.io/production-deployment/worker-deployments/worker-versioning) と [上限](https://docs.temporal.io/workflow-execution/limits)
- [GitHub: temporalio/temporal](https://github.com/temporalio/temporal)

## 9. 確認日

- 内容の確認日: 2026-11-12
- Radar の評価の確認日: 2026-11-12

## 用語集

| 用語 | 説明 |
|---|---|
| 耐久実行（durable execution） | 処理の進み具合を記録しておき、障害で止まっても記録をもとに続きから再開できるようにする実行のやり方 |
| ワークフロー | 複数の段階からなる処理の流れ |
| ポーリング | 相手に「仕事はありますか」と繰り返し問い合わせて、取りに行くやり方 |
| イベント履歴 | 1つの Workflow の実行で起きた出来事を、追記だけで順に残した記録 |
| 再生（リプレイ） | 記録を先頭からたどり直して、止まる前の状態を組み立て直すこと |
| 決定的 | 同じ入力に対して、いつも同じ順で同じ動きをする性質 |
| 冪等（べきとう） | 同じ操作を何回行っても、1回だけ行ったときと結果が変わらない性質。再試行される Activity に求められる |
| シャード | データや仕事を分けて受け持たせるときの、1つ1つの分け前 |
| オーケストレーション | 複数の処理やサービスを、指揮役が順序と条件を決めて動かすこと |
| Saga | 複数のサービスにまたがる処理を、各段階の処理と、失敗したときの取り消しの処理の組で書く設計のやり方 |
| Elasticsearch | 大量のデータを素早く検索するための検索エンジン |
