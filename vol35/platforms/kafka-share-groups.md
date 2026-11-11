# Kafka share groups（Kafka をキューのように使う仕組み）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Platforms（アプリケーションを載せて動かす基盤・サービス） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

Kafka share groups は、Apache Kafka（大量のデータの流れを記録し、配る基盤）に加わった、**メッセージを「キュー」のように取り分ける**ための読み手のまとまりです。KIP-932「Queues for Kafka」（KIP は Kafka の改善提案の文書）で設計され、Kafka 4.2 で本番に使える段階になりました（[KIP-932](https://cwiki.apache.org/confluence/display/KAFKA/KIP-932%3A+Queues+for+Kafka)、[4.2.0 の告知](https://kafka.apache.org/blog/2026/02/17/apache-kafka-4.2.0-release-announcement/)）。

たとえるなら、これまでの Kafka は「棚ごとに担当を1人決めて仕事を配る」方式でした（1人が複数の棚を持つこともあります）。share groups は「全員で1つの受付の山から、手が空いた人が1件ずつ取っていく」方式を加えます。

## 2. 解決する課題（なぜ生まれたか）

Kafka では、データは**トピック**（データの種類ごとの置き場）に書かれ、トピックはさらに**パーティション**（並列に処理するための区切り）に分かれます。従来の読み手のまとまりである **consumer group** では、1つのパーティションを読めるのはグループの中の1人だけです。そのため、読み手を増やしても、パーティションの数より多くは同時に働けません（[KIP-932](https://cwiki.apache.org/confluence/display/KAFKA/KIP-932%3A+Queues+for+Kafka)）。

筆者の見立てでは、これは順序を守って流れを処理するには都合がよい一方、「注文ごとにメール送信」のように1件ずつ独立した仕事をたくさんの作業者でさばきたい場面では窮屈です。処理の重さが件ごとに違うと、遅い1件が後ろの件を待たせることもあります。そのため、Kafka とは別に RabbitMQ などのメッセージキューを並べて運用することがありました。

share groups は、Kafka のトピックをそのまま使いながら、キューのような協調的な取り分けを可能にします（[KIP-932](https://cwiki.apache.org/confluence/display/KAFKA/KIP-932%3A+Queues+for+Kafka)）。

## 3. 仕組みの概要

### 3.1 consumer group との違い

KIP-932 が挙げる違いを表にまとめると、次のとおりです（[KIP-932](https://cwiki.apache.org/confluence/display/KAFKA/KIP-932%3A+Queues+for+Kafka)。表の形と consumer group の欄の書き方は筆者の整理）。

| 観点 | consumer group（従来） | share group（新しい） |
|---|---|---|
| パーティションの割り当て | 1つのパーティションは1人だけ | 1つのパーティションを複数人で読める |
| 読み手の数 | パーティションの数までしか働けない | パーティションの数を超えてよい |
| 処理済みの記録 | 「どこまで読んだか」の位置で記録 | レコード（1件のデータ）ごとに確認応答する |
| 配信の回数 | 数える仕組みはない | 数えて、上限を超えたら諦める |

### 3.2 1件ずつの取得と確認応答

読み手がレコードを取ると、そのレコードには**取得のロック**がかかり、ほかの読み手には渡りません。ロックの既定の長さは30秒です。読み手は処理の結果を、次のどれかで返します（[KIP-932](https://cwiki.apache.org/confluence/display/KAFKA/KIP-932%3A+Queues+for+Kafka)）。

- **ACCEPT**: 正しく処理できた
- **RELEASE**: 処理できなかったので、ほかの読み手にもう一度回してほしい
- **REJECT**: 処理できず、二度と配らなくてよい

これは KIP-932 で定めた3種類です。4.2 では、処理に時間がかかるときにロックを延ばす RENEW が加わりました（[4.2.0 の告知](https://kafka.apache.org/blog/2026/02/17/apache-kafka-4.2.0-release-announcement/)）。

何も返さずにロックの期限が切れると、レコードは解放され、また配られます。配った回数は数えられ、上限（既定は5回）に達したレコードは諦めて「アーカイブ済み」の状態になります（同上）。

### 3.3 状態を管理する部品

各パーティションのリーダー（そのパーティションを主に受け持つサーバー）が、レコードの取得や確認応答の状態を管理します。その状態は、share coordinator という部品が内部のトピック `__share_group_state` に保存します（[KIP-932](https://cwiki.apache.org/confluence/display/KAFKA/KIP-932%3A+Queues+for+Kafka)）。

### 3.4 使い方

Java のクライアントには、`KafkaShareConsumer` という専用の読み手が用意されています。既定では、次の取得のときに、前回取った分がまとめて自動で確認応答されます。明示のモードにすると、レコードごとに `acknowledge` で結果（ACCEPT など）を返します（[KIP-932](https://cwiki.apache.org/confluence/display/KAFKA/KIP-932%3A+Queues+for+Kafka)）。

### 3.5 版ごとの段階

| Kafka の版 | 段階 |
|---|---|
| 4.0（2025年3月） | 早期の試用版（[4.0.0 の告知](https://kafka.apache.org/blog/2025/03/18/apache-kafka-4.0.0-release-announcement/)）。本番のクラスタでは有効にしてはいけない（[試用版の注意書き](https://cwiki.apache.org/confluence/x/i4ogF)） |
| 4.1（2025年9月） | プレビュー。まだ本番向けではない（[4.1.0 の告知](https://kafka.apache.org/blog/2025/09/04/apache-kafka-4.1.0-release-announcement/)） |
| 4.2（2026年2月） | 本番で使える段階（[4.2.0 の告知](https://kafka.apache.org/blog/2026/02/17/apache-kafka-4.2.0-release-announcement/)） |

4.0 で有効にして使ったクラスタは、その後の版へ上げられないなど、初期の版には制約がありました（[4.0 の試用版の注意書き](https://cwiki.apache.org/confluence/x/i4ogF)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: メッセージの基盤を「流れを記録するもの（Kafka）」と「仕事を配るもの（キュー）」の2つに分けていた構成を、Kafka 1つにまとめる選択肢です。Thoughtworks も、外部のキューと併用していたチームが構成を簡素にできる点を挙げています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/platforms/kafka-share-groups)）。

**得られるもの**

- **読み手を自由に増やせる**: パーティションの数に縛られずに、処理の量に合わせて読み手を増やせる（[KIP-932](https://cwiki.apache.org/confluence/display/KAFKA/KIP-932%3A+Queues+for+Kafka)）
- **失敗した件だけやり直せる**: 1件ごとに確認応答するので、失敗した件だけ別の読み手に回せる（同上）
- **運用する製品が減る**: キューの製品を別に運用しなくて済む場合がある（筆者の見立て）

**代わりに失うもの・注意点**

- **順序の保証が弱い**: 1回の取得の中では位置の順に並びますが、取得をまたいだ順序は保証されません。やり直しが入ると順番は前後します（[KIP-932](https://cwiki.apache.org/confluence/display/KAFKA/KIP-932%3A+Queues+for+Kafka)）。順序が大事な処理には consumer group が向きます（筆者の見立て）
- **同じ件が2回届くことがある**: 配信の保証は「少なくとも1回」（at-least-once）です（同上）。処理を**冪等**（同じ処理を2回しても結果が変わらない）にしておく必要があります（筆者の見立て）
- **デッドレターキューがまだ無い**: 諦めたレコードを別の置き場へ移す仕組み（デッドレターキュー）は、KIP-932 では将来の拡張とされています（同上）。Thoughtworks も、この機能（KIP-1191）がまだ使えない点を注意しています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/platforms/kafka-share-groups)）
- **トランザクションとの組み合わせ**: 確認応答をトランザクションに含めて「ちょうど1回」の処理にすることは、KIP-932 の範囲外です（[KIP-932](https://cwiki.apache.org/confluence/display/KAFKA/KIP-932%3A+Queues+for+Kafka)）
- **データの保持期間はトピック全体の設定**: 古いデータの削除はトピック全体に効くため、まだ配られていないレコードが消えることもあります（同上）

## 5. 採用を判断するときの観点

- **Kafka の版**: 本番で使うなら 4.2 以降が前提です（[4.2.0 の告知](https://kafka.apache.org/blog/2026/02/17/apache-kafka-4.2.0-release-announcement/)）。クラスタ全体の版上げの計画と合わせて考える（筆者の見立て）
- **仕事の性質**: 1件ずつ独立していて、順序を問わず、重複を許せる（または冪等にできる）仕事に向きます（筆者の見立て）
- **クライアントの言語**: KIP-932 は、Java の API と、サーバーとの通信の取り決めを定めています（[KIP-932](https://cwiki.apache.org/confluence/display/KAFKA/KIP-932%3A+Queues+for+Kafka)）。Java 以外の言語のクライアントの対応状況は、それぞれのライブラリで確かめる必要があります（筆者の見立て）
- **既存の consumer group との併用**: 同じトピックを consumer group と share group の両方で読めます。ただし、グループの名前は両者で共通の名前空間なので、同じ名前は使えません（[KIP-932](https://cwiki.apache.org/confluence/display/KAFKA/KIP-932%3A+Queues+for+Kafka)）
- **既存のキューの機能との差**: 優先度の付いたキュー・遅延配信・デッドレターキューなど、専用のキューの製品が持つ機能に頼っているなら、置き換えられるかを1つずつ確かめる（筆者の見立て）

## 6. 関連技術・代替技術

- **consumer group**: Kafka の従来の読み手のまとまり。順序が大事な流れの処理には、引き続きこちらを使います（[KIP-932](https://cwiki.apache.org/confluence/display/KAFKA/KIP-932%3A+Queues+for+Kafka)。使い分けは筆者の見立て）
- **RabbitMQ・Amazon SQS などのメッセージキュー**: 仕事を配ることに特化した製品。デッドレターキューなどの機能がそろっています（筆者の見立て）
- **KIP-1191**: share groups にデッドレターキューを足す提案。Radar のページで、まだ使えない機能として触れられています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/platforms/kafka-share-groups)）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: 読み手の数がパーティションの数に縛られるという consumer group の限界を外し、キューの使い方を Kafka の中で実現できるとして、試す価値があるとしています。一方で、重複の配信・順序の保証の無さ・デッドレターキューの欠如には注意が要るとしています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/platforms/kafka-share-groups

## 8. 深掘りするときの一次情報

- [KIP-932: Queues for Kafka](https://cwiki.apache.org/confluence/display/KAFKA/KIP-932%3A+Queues+for+Kafka)
- [Apache Kafka 4.2.0 の告知](https://kafka.apache.org/blog/2026/02/17/apache-kafka-4.2.0-release-announcement/)
- [Apache Kafka 4.1.0 の告知](https://kafka.apache.org/blog/2025/09/04/apache-kafka-4.1.0-release-announcement/) と [4.0.0 の告知](https://kafka.apache.org/blog/2025/03/18/apache-kafka-4.0.0-release-announcement/)

## 9. 確認日

- 内容の確認日: 2026-11-11
- Radar の評価の確認日: 2026-11-11

## 用語集

| 用語 | 説明 |
|---|---|
| Apache Kafka | 大量のデータの流れ（イベント）を、順番どおりに記録し、多くの読み手に配るためのオープンソースの基盤 |
| トピック | Kafka で、データを種類ごとに分けて置く場所 |
| パーティション | トピックを分割した単位。分割した数だけ並列に書き込み・読み出しができ、中の順序は保たれる |
| consumer group | Kafka の従来の読み手のまとまり。各パーティションをメンバーの1人だけに割り当て、どこまで読んだかを位置で記録する |
| キュー | 仕事を1件ずつ並べ、手の空いた作業者が取り出して処理する仕組み |
| 確認応答 | 読み手が「この件は処理できた／できなかった」と基盤に伝えること |
| at-least-once | 「少なくとも1回は届ける」という配信の保証。失敗に備えて送り直すため、同じ件が2回以上届くことがある |
| 冪等 | 同じ処理を何回しても、1回したときと結果が変わらない性質 |
| デッドレターキュー | 何度やっても処理できなかったメッセージを、あとで調べるために移しておく別の置き場 |
| KIP | Kafka Improvement Proposal。Kafka に新しい機能を足すときに書く設計の提案書 |
