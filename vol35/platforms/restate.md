# Restate（処理の途中で落ちても続きから再開できる、耐久実行の基盤）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Platforms（アプリケーションを載せて動かす基盤・サービス） |
| リング | Trial（Assess から移動） |
| 状態 | 下書き |

## 1. 一言でいうと

Restate は、サービスの処理の各段階を記録しておき、障害で止まっても**続きから再開させる**ための基盤です。この考え方を**耐久実行（durable execution）**と呼びます。開発者はふだんどおりに関数を書き、呼び出しの記録・再試行・状態の保存は Restate のサーバーが引き受けます（[Restate のドキュメント](https://docs.restate.dev/)）。

サーバーは1つの実行ファイルとして配られ（[サーバーの概要](https://docs.restate.dev/server/overview)）、そのリポジトリは Rust のプロジェクトです（[GitHub: restatedev/restate](https://github.com/restatedev/restate)）。SDK（サービスを書くための部品）は、TypeScript・Java・Kotlin・Python・Go・Rust などの言語向けにあります（[Restate のドキュメント](https://docs.restate.dev/)）。サーバーのライセンスは Business Source License 1.1（BSL）で、SDK は MIT ライセンスです（[サーバーの LICENSE](https://github.com/restatedev/restate/blob/main/LICENSE)、[TypeScript の SDK の LICENSE](https://github.com/restatedev/sdk-typescript/blob/main/LICENSE)）。

たとえるなら、長い手続きの書類の欄が1つ終わるたびに判子を押していく係です。担当者が途中で倒れても、次の担当者は判子の無い欄から続ければよく、済んだ欄をやり直さずに済みます（たとえは筆者の見立て）。

## 2. 解決する課題（なぜ生まれたか）

たとえば注文の処理が「支払い → 在庫の引き当て → 確認のメール」と進むとします。途中でサーバーが落ちると、支払いだけ済んで在庫が引かれていない、という中途半端な状態が残ります。これを防ぐために、再試行の仕組み、進み具合を覚える表、二重の実行を防ぐ仕組みを、サービスごとに自前で作り込むことが多くありました（筆者の見立て）。

Restate は、こうした「落ちても正しく続ける」ための仕組みを基盤の側に寄せます。各段階とその結果を記録し、障害のあとは記録を読み直して、済んだ段階を飛ばして再開します（[主な考え方](https://docs.restate.dev/concepts/durable_execution)）。

## 3. 仕組みの概要

### 3.1 ジャーナルと再生

1. Restate のサーバーは、届いた要求を保存し、その呼び出し専用の記録（**ジャーナル**）を作ります
2. サーバーはサービスとの間に HTTP/2 の双方向の通信路を開き、処理を始めさせます
3. 接続が切れる・エラーが出る・一定時間応答が無い、といった失敗があると、サーバーは最新のジャーナルを付けて再試行します
4. SDK はジャーナルを**再生**し、記録済みの段階は実行し直さずに、記録された結果を使います

（[要求の一生](https://docs.restate.dev/guides/request-lifecycle)）

外部の API を呼ぶなどの副作用は、`ctx.run` で包むと結果がジャーナルに残り、再試行の時に繰り返されません（[主な考え方](https://docs.restate.dev/concepts/durable_execution)、[アクション](https://docs.restate.dev/foundations/actions.md)）。公式は、呼び出しは失敗があっても1回だけ実行される、とうたっています。サービスどうしの呼び出しも呼び出し元のジャーナルに記録されるので、端から端まで冪等（何回やっても結果が同じ）になるとしています（[主な考え方](https://docs.restate.dev/concepts/durable_execution)、[要求の一生](https://docs.restate.dev/guides/request-lifecycle)）。ただし、外部の API を呼んだ直後で記録の前に落ちると、その呼び出しが繰り返される余地が残ると考えられます。相手の側でも冪等キーなどで重複を防ぐのが安全です（筆者の見立て）。

### 3.2 3種類のサービス

| 種類 | 性質 | 向いている用途 |
|---|---|---|
| Basic Service | 状態を持たない処理。いくつでも並行して動く | API、Saga、裏方のジョブ |
| Virtual Object | キーごとの状態（キーと値の組）を持つ。書き込みのできる処理は、同じキーでは同時に1つだけ動く | 利用者のアカウント、買い物かご、状態機械 |
| Workflow | 本体の処理は、同じ ID につき1回だけ実行される。別の処理から合図を送ったり、状態を問い合わせたりできる | 承認を挟む手続き |

（[サービスの種類](https://docs.restate.dev/concepts/services)）

Virtual Object では、同じキーへの書き込みが順番待ちになるので、ロックを自分で書かずに競合を避けられます（[サービスの種類](https://docs.restate.dev/concepts/services)）。

### 3.3 待つための部品

- **耐久タイマー**: `ctx.sleep` で決まった時間だけ止まります。止まっている間は資源を使わず、再起動をまたいでも時刻どおりに再開します（[アクション](https://docs.restate.dev/foundations/actions.md)）
- **awakeable**: 一意の ID を持つ1回きりの合図です。別のハンドラーが SDK から、または外部のシステムが HTTP の API から合図を返すまで、処理は止まって待ちます（[外部のイベント](https://docs.restate.dev/develop/ts/external-events)）
- **呼び出し方**: HTTP、Kafka、プログラムから呼び出せます。冪等キーを付けた要求は、重複しても最初と同じ結果を返します（[主な考え方](https://docs.restate.dev/concepts/durable_execution)）

### 3.4 配置

Restate のサーバーは、リバースプロキシやメッセージブローカーのように、サービスの前に立ちます。サービスはコンテナ・サーバーレスの関数・仮想マシン・Kubernetes の Pod のどれで動かしてもかまいません（[主な考え方](https://docs.restate.dev/concepts/durable_execution)）。AWS Lambda のように1回の要求と応答で終わる実行環境では、処理をいったん止め、サーバーからの再試行で続きを行います（[要求の一生](https://docs.restate.dev/guides/request-lifecycle)）。

データは、複製されたディスク上のログに保存します。これが耐久性の土台で、RocksDB（組み込みのデータベース）はログから作り直せるキャッシュです。スナップショットは S3 などのオブジェクトストレージに置き、1台で動かすときも永続的なボリュームが要ります（[サーバーの概要](https://docs.restate.dev/server/overview)）。クラスタは過半数の合意で動き、n 台の故障に耐えるには 2n+1 台以上が要ります（3台なら1台の故障に耐えます。[クラスタ](https://docs.restate.dev/server/clusters)）。

開発元は、管理されたサービスの Restate Cloud も提供しています。料金は「耐久アクション」の数に応じた従量制で、月10万アクションまでの無料プランがあります。自社のクラウドのアカウントの中で動かす形（BYOC）もあります（[料金](https://restate.dev/pricing)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: サービスの前に立ち、呼び出しの中継・記録・再試行・状態の保存をまとめて担う層です。メッセージキュー、ワークフローエンジン、状態の保存先を1つにまとめたものとみなせます（筆者の見立て）。

**得られるもの**

- **回復の処理を自前で書かずに済む**: ジャーナルの再生で、落ちたところから続けられます（[主な考え方](https://docs.restate.dev/concepts/durable_execution)）
- **競合を避けやすい**: Virtual Object のキーごとの順番待ちで、同じ相手への同時の書き込みを防げます（[サービスの種類](https://docs.restate.dev/concepts/services)）
- **長く待つ処理を書きやすい**: 人の承認や外部の合図を待つ間、処理をいったん止め、合図が届いたら再開できます（[外部のイベント](https://docs.restate.dev/develop/ts/external-events)）

**代わりに失うもの・注意点**

- **決定的なコードが要る**: 再生の時に同じ順で同じ操作が出てこないと、ジャーナルの食い違いとしてエラーになります。`ctx.run` の外での HTTP の呼び出し、乱数、現在時刻などは、`ctx.run` か SDK の補助の関数を通す必要があります（[版の管理](https://docs.restate.dev/services/versioning)、[アクション](https://docs.restate.dev/foundations/actions.md)）
- **コードの更新に作法がある**: 版ごとに別の配置先を登録し、実行中の呼び出しは元の版で最後まで動かします。古い版は、その呼び出しが終わるまで残す必要があります。公式は、何日も何か月も動き続ける処理を避け、遅延した呼び出しに分けるよう勧めています（[版の管理](https://docs.restate.dev/services/versioning)）
- **新しい依存先が増える**: すべての呼び出しが Restate のサーバーを通るので、ここが止まると全体が止まります。クラスタでは過半数のノードが要ります（[クラスタ](https://docs.restate.dev/server/clusters)。影響の大きさは筆者の見立て）
- **ライセンスの制約**: サーバーの BSL は、本番での利用を認めています。ただし、第三者が自分のサービスを登録して呼び出せる、公開の Restate の基盤のサービスとして提供することは除かれます。各版は公開の4年後に Apache License 2.0 に切り替わります（[サーバーの LICENSE](https://github.com/restatedev/restate/blob/main/LICENSE)）

## 5. 採用を判断するときの観点

- **規模と必要性**: 失敗しても再実行すればよい単純な非同期のジョブなら、既存のキューで足ります。複数のサービスにまたがり、途中で落ちると不整合が残る処理ほど向いています（筆者の見立て）
- **可用性**: 公式は、3台で1台の故障に耐える例を示しています。止められない用途なら、3台以上で組むことになります（[クラスタ](https://docs.restate.dev/server/clusters)）
- **運用**: 永続的なボリュームと、クラスタならオブジェクトストレージが要ります。公式は、管理された Kubernetes の上で専用の Operator を使う形を勧めています（[サーバーの概要](https://docs.restate.dev/server/overview)）
- **更新**: 版の付け方は Semantic Versioning で、1.x から 1.(x+1) への更新では手作業のデータの移行が要らないとしています（[GitHub: restatedev/restate](https://github.com/restatedev/restate)）。2026年11月の時点で "Latest" の版は v1.8.1（2026-10-21）です（[リリース](https://github.com/restatedev/restate/releases)）
- **費用**: 自分で動かす分にはライセンス料はかかりません。Restate Cloud は従量制で、有料プランの単価はページに載っていません（[料金](https://restate.dev/pricing)）
- **依存の深さ**: 業務のコードが SDK の `ctx` の作法に沿って書かれるので、あとで別の基盤に移すには書き直しが要ります（筆者の見立て）

## 6. 関連技術・代替技術

- **Temporal**: 同じく耐久実行の基盤で、ワークフローの考え方や向く用途が近い技術です（筆者の見立て）。Vol.35 では Temporal も Platforms の Trial に入っています
- **Saga パターン**: 複数のサービスにまたがる処理を、段階ごとの処理と、失敗した時の取り消しの処理の組で書く設計のやり方です。Restate は、これを Basic Service の用途の1つに挙げています（[サービスの種類](https://docs.restate.dev/concepts/services)）
- **メッセージキューと自前の状態の表**: 従来のやり方です。部品を選べる自由がある代わりに、回復と重複の処理を自分で書きます（筆者の見立て）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（Assess から移動）
- 過去の版: 2025年4月の版で Assess として初めて載り、2025年11月の版も Assess（Radar のページの推移による）
- Thoughtworks の評価の要旨: 大規模な保険のシステムで、業務の流れの自動化の中核に使われ、障害からの回復と同時実行の制御を自前で書かずに済んだ実績から、Trial に上げています。注意点として、サーバーのライセンスが BSL であることを挙げています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/platforms/restate

## 8. 深掘りするときの一次情報

- [Restate のドキュメント](https://docs.restate.dev/) と [主な考え方](https://docs.restate.dev/concepts/durable_execution)
- [サービスの種類](https://docs.restate.dev/concepts/services) と [要求の一生](https://docs.restate.dev/guides/request-lifecycle)
- [版の管理](https://docs.restate.dev/services/versioning) と [サーバーの概要](https://docs.restate.dev/server/overview)
- [GitHub: restatedev/restate](https://github.com/restatedev/restate) と [LICENSE](https://github.com/restatedev/restate/blob/main/LICENSE)

## 9. 確認日

- 内容の確認日: 2026-11-12
- Radar の評価の確認日: 2026-11-12

## 用語集

| 用語 | 説明 |
|---|---|
| 耐久実行（durable execution） | 処理の各段階とその結果を記録しておき、障害で止まっても記録をもとに続きから再開できるようにする実行のやり方 |
| ジャーナル | 1回の呼び出しの中で行った操作とその結果を、順に書き残した記録 |
| 再生（リプレイ） | ジャーナルを先頭から読み直し、済んだ段階は記録の結果を使って飛ばしながら、止まったところまで戻ること |
| 冪等（べきとう） | 同じ操作を何回行っても、1回だけ行ったときと結果が変わらない性質 |
| 冪等キー | 要求に付ける一意の印。同じ印の要求が重ねて届いても、1回分として扱うために使う |
| 副作用 | 外部の API の呼び出しやメールの送信など、関数の外の世界に影響を与える操作 |
| Saga | 複数のサービスにまたがる処理を、各段階の処理と、失敗したときの取り消しの処理の組で書く設計のやり方 |
| BSL（Business Source License） | ソースコードを公開しつつ、決めた種類の商用の利用を制限するライセンス。決めた期間が過ぎると、指定したオープンソースのライセンスに切り替わる |
| RocksDB | アプリに組み込んで使う、キーと値を保存するデータベース |
| オブジェクトストレージ | ファイルを「オブジェクト」として大量に保存するサービス。AWS の S3 が代表例 |
| BYOC | Bring Your Own Cloud。提供元が運用するソフトウェアを、利用者自身のクラウドのアカウントの中で動かす形 |
