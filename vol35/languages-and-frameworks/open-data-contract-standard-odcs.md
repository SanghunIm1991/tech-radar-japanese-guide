# Open Data Contract Standard（ODCS、データ契約の標準の書式）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Languages & Frameworks（プログラミング言語・フレームワーク・標準の書式など） |
| リング | Adopt（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

Open Data Contract Standard（ODCS）は、**データ契約**（データを渡す側と使う側が「どんな形・品質・頻度のデータを渡すか」を取り決めたもの）を YAML（人が読み書きしやすい設定ファイルの書式）で書くための、オープンな標準です（[bitol.io](https://bitol.io/)、[GitHub: open-data-contract-standard](https://github.com/bitol-io/open-data-contract-standard)）。確認日の時点の最新版は v3.2.0 で、標準そのものは Apache-2.0 ライセンスで公開されています（[CHANGELOG](https://github.com/bitol-io/open-data-contract-standard/blob/dev/CHANGELOG.md)、[GitHub](https://github.com/bitol-io/open-data-contract-standard)）。

たとえるなら、Web の API の仕様を書く OpenAPI の「データ版」です。表の列の名前や型だけでなく、「空の値は何件まで」「毎日何時までに更新」といった約束まで1つのファイルに書きます（たとえは筆者の見立て）。

## 2. 解決する課題（なぜ生まれたか）

会社の中では、あるシステムが出したデータを、別のチームが分析や機械学習、帳票に使います。ここでよく起きるのが、出す側が列の名前や意味を何気なく変え、使う側の集計が黙って壊れる事故です。取り決めが口頭や表計算のファイルにしかないと、どこが壊れるのか事前に分からず、機械で確かめることもできません（筆者の見立て）。

さらに、データを「製品」として各部門が責任を持って提供する**データメッシュ**という考え方が広がると、提供する側と使う側の取り決めがいっそう重要になります。ところが、取り決めの書き方はばらばらでした。Thoughtworks も、競合する書式や自社独自の書式が並んでいたと述べています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/languages-and-frameworks/open-data-contract-standard-odcs)）。

ODCS は、この取り決めを共通の書式で書けるようにします。もとは PayPal の社内で、データメッシュの実装に使っていたひな形で、それを公開して標準に育てたものです（[ODCS の経緯](https://github.com/bitol-io/open-data-contract-standard/blob/dev/history.md)）。

## 3. 仕組みの概要

### 3.1 運営と版

- 標準は、Linux Foundation AI & Data（オープンソースの AI・データの事業を支える団体）の下の **Bitol** というプロジェクトが管理しています（[bitol.io](https://bitol.io/)）。Bitol は2023年11月に作られました（[ODCS の経緯](https://github.com/bitol-io/open-data-contract-standard/blob/dev/history.md)）
- v3.0.0（2024年10月）は v2 からの**後方互換性の無い変更**を含み、`columns` が `properties` に、`isNullable` が `required` に変わるなど、多くの名前が改められました。v3.2.0 での追加の多くは、互換性を保つ変更と明記されています（[CHANGELOG](https://github.com/bitol-io/open-data-contract-standard/blob/dev/CHANGELOG.md)）
- 書いた YAML を確かめるための JSON Schema（JSON や YAML の形を機械で検査するための定義）も用意されており、エディタに読み込ませて間違いを見つけられます（[GitHub](https://github.com/bitol-io/open-data-contract-standard)）

### 3.2 契約に書くこと

主な節は次のとおりです（[ODCS のドキュメント](https://bitol-io.github.io/open-data-contract-standard/latest/)）。

| 節 | 書くこと |
|---|---|
| 基本（Fundamentals） | 契約の ID・版・状態・担当する領域・目的や使い方の説明（[Fundamentals](https://github.com/bitol-io/open-data-contract-standard/blob/dev/docs/fundamentals.md)） |
| スキーマ（Schema） | 表やオブジェクトと、その項目の名前・型・必須か・主キーか・例など（[Schema](https://github.com/bitol-io/open-data-contract-standard/blob/dev/docs/schema.md)） |
| データ品質（Data Quality） | 品質のルール（次の 3.3） |
| SLA | 更新の頻度・遅れの許容・保存期間などの約束（次の 3.4） |
| サーバー（Servers） | データの置き場所。PostgreSQL・S3・Kafka・Snowflake など多くの種類を指定できる（[Servers](https://github.com/bitol-io/open-data-contract-standard/blob/dev/docs/infrastructure-servers.md)） |
| そのほか | 担当チーム・役割（権限）・問い合わせ先・料金・独自の項目など |

ごく小さな例を自作すると、次のような形です（主な項目だけを抜き出した、説明用の簡略版です）。

```yaml
apiVersion: v3.2.0
kind: DataContract
id: orders-contract
version: 1.0.0
status: active
schema:
  - name: orders
    logicalType: object
    properties:
      - name: order_id
        logicalType: string
        required: true
        primaryKey: true
      - name: amount
        logicalType: number
```

### 3.3 品質のルール

品質のルールは4種類で書けます（[Data Quality](https://github.com/bitol-io/open-data-contract-standard/blob/dev/docs/data-quality.md)）。

- **text**: 人が読むための説明だけ
- **library**: 標準で決められた指標。空の値の数（`nullValues`）、重複（`duplicateValues`）、行数（`rowCount`）などを、「〜でなければならない」（`mustBe`）、「〜より小さい」（`mustBeLessThan`）といった比べ方で書く
- **sql**: 数（または真偽値）を返す SQL を書き、その結果を同じ比べ方で判定する
- **custom**: Soda や Great Expectations など、特定の品質検査の道具の書き方をそのまま埋め込む

### 3.4 SLA

**SLA**（サービスレベルの約束）は、「何を」「いくつ」「どの単位で」の組で書きます。標準は、推奨する項目名として、遅れ（`latency`）・保存期間（`retention`）・更新頻度（`frequency`）・可用性（`availability`）などを挙げています（[SLA](https://github.com/bitol-io/open-data-contract-standard/blob/dev/docs/service-level-agreement.md)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: データを渡すシステムと使うシステムの**境界に置く取り決め**です。アプリの間の API に仕様書を置くのと同じ考え方を、データの受け渡しに当てはめます。契約のファイルを Git で管理し、変更をレビューし、CI（変更のたびに自動で検査する仕組み）で検査する、という使い方が想定されます（筆者の見立て）。

**得られるもの**

- **機械で確かめられる**: 契約に書いたスキーマや品質のルールを、道具で実データに当てて検査できる（[Data Contract CLI](https://github.com/datacontract/datacontract-cli)）
- **道具をまたいで使える**: 標準の書式なので、特定の製品に縛られにくい。対応を表明している製品の一覧もある（[対応する製品の一覧](https://github.com/bitol-io/open-data-contract-standard/blob/dev/vendors.md)。網羅的ではない一覧です）
- **人にも AI にも読める**: 宣言的な書き方（「どうあるべきか」だけを書く形）なので、人と AI エージェントの両方が読み、確かめやすいと Thoughtworks は評価しています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/languages-and-frameworks/open-data-contract-standard-odcs)）

**代わりに失うもの・注意点**

- **書いて守る手間**: 契約を書くこと自体は品質を上げません。検査を流れに組み込み、破ったときに誰が直すかを決めて初めて効きます（筆者の見立て）
- **版の変化**: v2 から v3 で多くの名前が変わり、v3.1.0 でも JSON Schema の一部に互換性の無い変更がありました（[CHANGELOG](https://github.com/bitol-io/open-data-contract-standard/blob/dev/CHANGELOG.md)）。契約のファイルの版をそろえ、道具の対応状況を確かめる必要があります
- **組織の合意が要る**: 出す側が契約に責任を持つ体制が無いと、書いた契約が古いまま放置されます（筆者の見立て）

## 5. 採用を判断するときの観点

- **規模**: データを出すチームと使うチームが分かれていて、変更による事故が起きているなら効果が大きい。1つのチームが作って自分で使うだけなら、手間に見合わないこともある（筆者の見立て）
- **既存の書式からの移行**: 以前の Data Contract Specification（DCS）は、ODCS の v3.1.0 の公開に合わせて非推奨になり、ODCS への移行が勧められています。Data Contract CLI には DCS から ODCS へ変換する書き出しの機能があります（[DCS のリポジトリ](https://github.com/datacontract/datacontract-specification)）
- **道具**: オープンソース（MIT）の Data Contract CLI は ODCS に対応し、契約の書式の検査（`lint`）、実データとの照合（`test`）、変更が互換性を壊すかの判定（`breaking`）、ほかの書式への書き出し（`export`）などができます（[Data Contract CLI](https://github.com/datacontract/datacontract-cli)）
- **品質検査の道具との分担**: すでに Soda や Great Expectations を使っているなら、`custom` のルールでその書き方を契約に取り込めます（[Data Quality](https://github.com/bitol-io/open-data-contract-standard/blob/dev/docs/data-quality.md)）
- **費用と依存先**: 標準も主な道具もオープンソースで、費用は主に書いて運用する人の手間です。商用の製品を使う場合は、その製品の対応する ODCS の版を確かめる（筆者の見立て）

## 6. 関連技術・代替技術

- **Open Data Product Standard（ODPS）**: 同じ Bitol が管理する、「データ製品」そのものを記述する標準。ODCS とそろえて作られており、データ製品とデータ契約を結び付けることを狙っています（[ODPS v1.0.0 の告知](https://bitol.io/announcing-odps-v1-0-0-building-the-language-of-data-products/)）。Thoughtworks も、両者を組み合わせて移行する事例を挙げています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/languages-and-frameworks/open-data-contract-standard-odcs)）
- **Data Contract CLI**: 上で触れた、ODCS を扱う道具（[Data Contract CLI](https://github.com/datacontract/datacontract-cli)）
- **OpenAPI・AsyncAPI**: API や非同期のメッセージの仕様を書く標準。ODCS はデータの受け渡しの側で同じ役割を果たします（筆者の見立て）
- **Soda・Great Expectations**: データの品質を検査する道具。ODCS の品質のルールから呼び出せます（[Data Quality](https://github.com/bitol-io/open-data-contract-standard/blob/dev/docs/data-quality.md)）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Adopt**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: いくつかの書式が並ぶ中で、ODCS が主要な標準として定着してきたとし、データ製品を作るチームの既定の選択として勧めています。主な理由は、別の書式（DCS）を管理していたチームが ODCS への移行を勧めていることと、宣言的な書式で人と AI の両方が扱いやすいことです
- Radar のページ: https://www.thoughtworks.com/en-us/radar/languages-and-frameworks/open-data-contract-standard-odcs

## 8. 深掘りするときの一次情報

- [ODCS のドキュメント](https://bitol-io.github.io/open-data-contract-standard/latest/)
- [GitHub: open-data-contract-standard](https://github.com/bitol-io/open-data-contract-standard) と [CHANGELOG](https://github.com/bitol-io/open-data-contract-standard/blob/dev/CHANGELOG.md)
- [Data Quality](https://github.com/bitol-io/open-data-contract-standard/blob/dev/docs/data-quality.md) と [SLA](https://github.com/bitol-io/open-data-contract-standard/blob/dev/docs/service-level-agreement.md)
- [Bitol](https://bitol.io/)
- [Data Contract CLI](https://github.com/datacontract/datacontract-cli)

## 9. 確認日

- 内容の確認日: 2026-11-11
- Radar の評価の確認日: 2026-11-11

## 用語集

| 用語 | 説明 |
|---|---|
| データ契約 | データを渡す側と使う側の間で、データの形・意味・品質・更新の頻度などを取り決めたもの。機械で確かめられる形で書くことが多い |
| YAML | 字下げで階層を表す、人が読み書きしやすい設定ファイルの書式 |
| スキーマ | データの構造の定義。どんな項目があり、それぞれがどんな型か、必須かなどを決めたもの |
| データメッシュ | データを1か所に集めて中央のチームが管理するのではなく、各部門が自分のデータを「製品」として責任を持って提供する考え方 |
| SLA | Service Level Agreement。提供する側が守ると約束するサービスの水準（更新の頻度・遅れ・可用性など） |
| JSON Schema | JSON（や YAML）のデータが決められた形になっているかを、機械で検査するための定義の書き方 |
| 後方互換性 | 新しい版が、古い版の書き方のファイルをそのまま受け付けられること。無い場合は書き直しが要る |
| 宣言的 | 手順ではなく、「どうあるべきか」という結果の状態だけを書く書き方 |
| 主キー | 表の中で、1行を一意に見分けるための項目 |
| CI | 継続的インテグレーション。変更のたびに自動でビルドや検査を行う仕組み |
