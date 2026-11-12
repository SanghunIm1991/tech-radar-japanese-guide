# VictoriaLogs（索引の設計なしに大量のログを安く貯めて探せる、ログ専用のデータベース）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Platforms（アプリケーションを載せて動かす基盤・サービス） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

VictoriaLogs は、**ログを貯めて検索するためのデータベース**です。開発元は、時系列データベースの VictoriaMetrics を作っている VictoriaMetrics 社で、オープンソース版のライセンスは Apache 2.0 です（[VictoriaLogs のドキュメント](https://docs.victoriametrics.com/victorialogs/)、[GitHub: VictoriaMetrics/VictoriaLogs](https://github.com/VictoriaMetrics/VictoriaLogs)）。

設定なしで動く1つの実行ファイルとして配られ、スキーマ（項目の形の事前の定義）も要りません。届いたログのすべての項目に、自動で索引を付けます（[VictoriaLogs のドキュメント](https://docs.victoriametrics.com/victorialogs/)、[主な考え方](https://docs.victoriametrics.com/victorialogs/keyconcepts/)、[GitHub: VictoriaMetrics/VictoriaLogs](https://github.com/VictoriaMetrics/VictoriaLogs)）。

たとえるなら、届いた書類を棚の分類を決めずに放り込んでも、あとで「この言葉を含む書類」「この差出人の先週の書類」をすぐ取り出せる書庫です（たとえは筆者の見立て）。

## 2. 解決する課題（なぜ生まれたか）

Kubernetes の上で多くのサービスを動かすと、ログの量は1日に何十 GB から何 TB にもなります。これまでよく使われてきた道具には、それぞれ悩みがありました（筆者の見立て）。

- **Elasticsearch**: 何でも細かく検索できる代わりに、メモリとディスクを多く使い、クラスタの運用も重くなりがちです
- **Grafana Loki**: ログの本文には索引を付けず、ラベルにだけ付けるので安く貯められます。ただ、どの項目をラベルにするかの設計が要り、ラベルの値の種類が多すぎると苦しくなります

VictoriaLogs は、すべての項目に索引を付けつつ、少ない資源で動くことを目指しています。開発元は、Elasticsearch や Loki と比べて、メモリは最大30分の1、ディスクは最大15分の1で済むとしています（開発元の主張。[VictoriaLogs のドキュメント](https://docs.victoriametrics.com/victorialogs/)）。

## 3. 仕組みの概要

### 3.1 ログの形

1件のログは、項目名と値の組の集まりです。決まった項目は次の3つです（[主な考え方](https://docs.victoriametrics.com/victorialogs/keyconcepts/)）。

| 項目 | 意味 |
|---|---|
| `_msg` | ログの本文。必ず要る |
| `_time` | ログの時刻。無ければ、受け取った時刻を使う |
| `_stream` | どこから来たログかを表す項目の組。例: `{app="nginx",host="web-1"}` |

入れ子の JSON は、`host.name` のように点でつないだ項目名に平らにして保存します（[主な考え方](https://docs.victoriametrics.com/victorialogs/keyconcepts/)）。

### 3.2 ストリームの項目の選び方

`_stream` に入れる項目（ストリームの項目）は、アプリの1つのインスタンスの間は変わらないもの（コンテナ名・ホスト名など）にします。利用者の ID やトレースの ID のように値がどんどん変わる項目を、ストリームの項目にしてはいけません。そうした値の種類が多い項目も、ふつうの項目としてなら問題なく扱えます（[主な考え方](https://docs.victoriametrics.com/victorialogs/keyconcepts/)）。設計で気をつけるのは、この「ストリームの項目に何を入れるか」だけ、と言えます（筆者の見立て）。

### 3.3 LogsQL で探す

検索には **LogsQL** という独自の問い合わせの言葉を使います。条件を並べると AND でつながり、`|` の後ろに集計や並べ替えの処理（パイプ）を足せます（[LogsQL](https://docs.victoriametrics.com/victorialogs/logsql/)）。

```
_time:5m error | sort by (_time) desc | limit 10
```

これは「直近5分の、本文（`_msg`）に error という語を含むログを、新しい順に10件」という意味です。ほかの項目を探すときは、`項目名:語` の形で指定します（[LogsQL](https://docs.victoriametrics.com/victorialogs/logsql/)）。時刻の条件を付けないと、全期間が対象になり、結果は既定では並べ替えられません（[LogsQL](https://docs.victoriametrics.com/victorialogs/logsql/)）。

### 3.4 ログの受け取りと見る道具

ログは、Elasticsearch の bulk の API・Loki の JSON の API・OpenTelemetry・JSON の行などの形で受け取れます。syslog・Filebeat・Fluent Bit・Vector・Promtail・OpenTelemetry Collector など、すでに使われているログの収集の道具から、送り先を変えるだけで送れます（[データの取り込み](https://docs.victoriametrics.com/victorialogs/data-ingestion/)）。

見る側には、組み込みの Web の画面、Grafana のプラグイン、対話型のコマンドの `vlogscli`、HTTP の API があり、`tail -f` のように新しいログを流し見る機能もあります（[問い合わせ](https://docs.victoriametrics.com/victorialogs/querying/)）。

### 3.5 保存と保持期間

ログは日ごとの区画に分けて保存され、保持期間を過ぎた区画は丸ごと消されます。保持期間は `-retentionPeriod` で決め、既定は7日です。ディスクの使用量や使用率の上限を決めて、古い日から消すこともできます（[VictoriaLogs のドキュメント](https://docs.victoriametrics.com/victorialogs/)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: 可観測性（システムの動きを外から観察できること）の基盤のうち、ログを受け持つ保存と検索の層です。メトリクスを受け持つ VictoriaMetrics と組み合わせるのが典型です（筆者の見立て。[VictoriaMetrics の解説](victoriametrics.md)）。

**得られるもの**

- **索引の設計が要らない**: すべての項目に自動で索引が付くので、あとから思いついた項目でも検索できます（[主な考え方](https://docs.victoriametrics.com/victorialogs/keyconcepts/)）
- **運用が軽い**: 1台で済むなら、実行ファイル1つで動きます（[VictoriaLogs のドキュメント](https://docs.victoriametrics.com/victorialogs/)）
- **乗り換えやすい**: 既存の収集の道具と取り込みの形をそのまま使えるので、送り先を試しに切り替えられます（[データの取り込み](https://docs.victoriametrics.com/victorialogs/data-ingestion/)）

**代わりに失うもの・注意点**

- **クラスタは複製しない**: クラスタの版は、受け取る `vlinsert`・問い合わせる `vlselect`・保存する `vlstorage` に分かれます。ただし、ログを複数の `vlstorage` に複製する仕組みは無く、`vlstorage` が1台止まると、既定では問い合わせがエラーになります。止まらない構成にするには、`vlagent` で2つのクラスタに同じログを送り、問い合わせを生きている方へ振り分けます（[クラスタ](https://docs.victoriametrics.com/victorialogs/cluster/)）
- **独自の問い合わせの言葉**: LogsQL を覚える必要があり、Elasticsearch や Loki の問い合わせの言葉はそのまま使えません（筆者の見立て）
- **数値は開発元の主張**: 資源の少なさの数値は開発元の比較で、自分たちのログで確かめる必要があります。公式も、本番のログの1〜10%を流して測るよう勧めています（[よくある質問](https://docs.victoriametrics.com/victorialogs/faq/)）

## 5. 採用を判断するときの観点

- **1台かクラスタか**: 公式は、1台の性能を上げて足りるうちは1台を勧めています。そのほうが単純で速いためで、クラスタは1台の限界を超えるときのものです（[クラスタ](https://docs.victoriametrics.com/victorialogs/cluster/)）
- **可用性**: 1台やクラスタの中では複製されないので、ログを失えない用途では、2系統に送る構成と、バックアップを考えます（[クラスタ](https://docs.victoriametrics.com/victorialogs/cluster/)）
- **費用**: オープンソース版は無料です。有料の Enterprise 版には、長期の支援の版（LTS）、開発元の支援、構成部品の間の相互 TLS などが付きます。価格は公開されていません（[Enterprise](https://docs.victoriametrics.com/victoriametrics/enterprise/)）
- **更新**: 2026年11月の時点の最新の版は v1.53.0（2026-10-01）です。v1.52 の系統は、少なくとも12か月支援される LTS です（[変更履歴](https://docs.victoriametrics.com/victorialogs/changelog/)）
- **移行**: 今の収集の道具の送り先を2つにして、しばらく並べて動かし、問い合わせの速さと資源を比べるのが安全です（筆者の見立て）

## 6. 関連技術・代替技術

- **Elasticsearch / OpenSearch**: 全文検索の定番です。検索の機能は豊富ですが、資源と運用の負担が大きくなりがちです（筆者の見立て）
- **Grafana Loki**: ラベルにだけ索引を付ける、ログの保存の仕組みです。Grafana との結びつきが強い代わりに、ラベルの設計が要ります（筆者の見立て）
- **VictoriaMetrics**: 同じ開発元の時系列データベースです。メトリクスとログを同じ作りの道具でそろえられます（[VictoriaMetrics の解説](victoriametrics.md)）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: AI のクラウドの基盤の可観測性で、大量のログの取り込みと検索に使って良い経験を得たことから、Trial としています。Elasticsearch より運用が軽く、Loki よりラベルの設計に悩まずに済む点を挙げています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/platforms/victorialogs

## 8. 深掘りするときの一次情報

- [VictoriaLogs のドキュメント](https://docs.victoriametrics.com/victorialogs/) と [主な考え方](https://docs.victoriametrics.com/victorialogs/keyconcepts/)
- [LogsQL](https://docs.victoriametrics.com/victorialogs/logsql/) と [問い合わせ](https://docs.victoriametrics.com/victorialogs/querying/)
- [データの取り込み](https://docs.victoriametrics.com/victorialogs/data-ingestion/) と [クラスタ](https://docs.victoriametrics.com/victorialogs/cluster/)
- [よくある質問](https://docs.victoriametrics.com/victorialogs/faq/) と [変更履歴](https://docs.victoriametrics.com/victorialogs/changelog/)
- [GitHub: VictoriaMetrics/VictoriaLogs](https://github.com/VictoriaMetrics/VictoriaLogs)

## 9. 確認日

- 内容の確認日: 2026-11-12
- Radar の評価の確認日: 2026-11-12

## 用語集

| 用語 | 説明 |
|---|---|
| 索引（インデックス） | データを速く探すために、別に作っておく目次のようなもの |
| スキーマ | データの項目の名前や型を、あらかじめ決めた定義 |
| ストリーム | 同じ出どころ（同じアプリのインスタンスなど）から来るログのまとまり |
| ラベル | ログやメトリクスに付ける、名前と値の組の目印。Loki はこれにだけ索引を付ける |
| 値の種類が多い（高カーディナリティ） | ある項目がとりうる値の種類がとても多いこと。利用者の ID やトレースの ID が典型 |
| パイプ | 前の処理の結果を次の処理に渡してつなぐ書き方 |
| 可観測性 | ログ・メトリクス・トレースなどから、システムの中で何が起きているかを外から分かるようにすること |
| メトリクス | CPU の使用率や要求の数など、時間とともに変わる数値の記録 |
| トレース | 1つの要求が複数のサービスを通る道筋を、時間とともに記録したもの |
| 保持期間 | データを消さずに残しておく期間 |
| LTS | Long-Term Support。長い期間、修正版が出され続ける版の系統 |
| 相互 TLS | 通信の両側が証明書を出し合い、互いに相手を確かめたうえで暗号化する方式 |
