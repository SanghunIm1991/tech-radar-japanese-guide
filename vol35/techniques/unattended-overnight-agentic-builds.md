# Unattended overnight agentic builds（夜のあいだにエージェントに任せて作らせる）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Techniques（開発や運用の進め方・手法） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

Unattended overnight agentic builds は、チームが仕事を終えて帰るときに、数時間かかる大きめの作業（機能1つ分の開発や、インフラの作業など）を AI のコーディングエージェントに渡し、**人が見ていない夜のあいだに作らせて、翌朝に成果を確かめる**進め方です。動かすエージェントは1つで、多数のエージェントを協調させる「群れ」の方式は含みません（作業の一部をサブエージェントに任せるのはかまいません）（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/unattended-overnight-agentic-builds)の定義による）。

この手法の特徴は、夜に任せられるかどうかが、**エージェントに本当に自律して作業させられているかを示す、ごまかしにくい目印**になる点です。夜に人なしで動かせないチームは、昼間もおそらく完全には任せきれていない、という見方です（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/unattended-overnight-agentic-builds)の見方。第7節）。

## 2. 解決する課題（なぜ生まれたか）

コーディングエージェントは、数分で終わる小さな作業なら、人がそばで見ながら進められます。しかし、機能1つ分のように何時間もかかる作業では、人が張り付いていると、エージェントで速くなるはずの開発が人の時間に縛られます（筆者の見立て）。

一方で、長い作業をエージェントだけに任せると、別の問題が出ます。Anthropic は、長時間動くエージェントの課題として、次のようなものを挙げています（[Anthropic: Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)。この項の出典は同じ）。

- **記憶が切れる**: 一度に見られる文章の量（コンテキスト）に上限があり、作業が長いと、途中で新しい会話に切り替わる。新しい会話は前の経緯を覚えていない
- **中途半端なまま引き継ぐ**: 一度に全部作ろうとして、途中でコンテキストを使い切り、記録の無い半端な状態のまま次の会話に引き継がれる
- **早すぎる完了宣言**: いくつかの機能ができたところで「完成した」と思い込む

夜の無人の作業は、こうした課題を仕組みで抑えられているかを試す場になります。

## 3. 仕組みの概要

**Radar が挙げる前提条件**（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/unattended-overnight-agentic-builds)）

- **隔離した実行**: エージェントを、外に害が及ばない隔離した環境（サンドボックス）で動かす
- **前もって仕様にした作業**: 何を作るかを、事前に仕様として書いておく
- **自分で誤りに気づく仕組み**: テストや静的な検査の結果をエージェントが受け取り、自分で直せるようにする（フィードバックのセンサー）
- **検証の仕組み**: 一晩分の変更は1行ずつ人が読むには大きすぎることがあるので、確かめられた検査の仕組み（ハーネス）で変更を検証する

Radar は、マイクロサービスを始める前に満たすべき前提条件を挙げた Martin Fowler の文章を引き合いに出しています。Fowler は、サーバーをすぐ用意できること・基本的な監視・素早いデプロイの3つを挙げ、これらが無いうちはマイクロサービスを考えるべきではないとしています（[martinfowler.com: MicroservicePrerequisites](https://martinfowler.com/bliki/MicroservicePrerequisites.html)）。夜の無人の作業も同じで、前提をそろえることそのものに価値がある、というのが Radar の見方です。

**長時間の作業を支える工夫（Anthropic の例）**（[Anthropic の記事](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)）

- **最初に段取りをする役**: 最初の会話で、開発用のサーバーを起動するスクリプト、進み具合を書くファイル、作るべき機能の一覧を作る
- **機能の一覧**: 例では200を超える機能を JSON の形で並べ、すべてを「不合格（failing）」の状態から始める。作業役のエージェントは、合格かどうかの印だけを書き換えてよい
- **1つずつ進めて記録する**: 機能を1つずつ作り、説明の付いたコミットと進み具合のファイルを残して、悪い変更は Git で戻せるようにする
- **人のように確かめる**: 単体テストだけでは機能の不備を見逃すことがあり、ブラウザを自動で操作して画面から確かめさせると、大きく改善したとしている

**対話なしで動かす道具の例**

Claude Code は `-p` オプションで、対話なしに指示を実行できます。使ってよいツールを `--allowedTools` で指定し、結果を JSON などの形で受け取れるので、夜に決まった時刻に起動する仕組みや CI に組み込めます（[Claude Code Docs: Run Claude Code programmatically](https://code.claude.com/docs/en/headless)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: 開発の流れの中で、人の作業時間とエージェントの作業時間を切り離す設計です。夜間に一括処理（バッチ）を回し、朝に結果を確かめる運用と似ています（筆者の見立て）。

**得られるもの**

- **時間の有効利用**: 人がいない時間に、大きめの作業を進められる（筆者の見立て）
- **ごまかしにくい目印**: 夜に任せられるかどうかで、本当に人なしで数時間動かせているかが分かる（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/unattended-overnight-agentic-builds)）
- **仕組みの整備を促す**: 前提をそろえさせること自体に価値がある（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/unattended-overnight-agentic-builds)）。そろえた検査や仕様の仕組みは、昼間の作業にも効く（筆者の見立て）

**代わりに失うもの・注意点**

- **朝のレビューの重さ**: 一晩分の変更は大きく、人が全部を読むのは難しい。検査の仕組みを信頼できないと、朝の確認が形だけになる（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/unattended-overnight-agentic-builds)の懸念に基づく筆者の見立て）
- **方向違いの作業**: 仕様があいまいだと、一晩かけて違うものを作ってしまう。途中で人が止められないぶん、無駄が大きい（筆者の見立て）
- **費用**: 長時間動かすぶん、トークン（LLM が文章を数える単位で、料金の基準）がかかる。上限の設定が要る（筆者の見立て）
- **安全**: 人が見ていないあいだの誤った操作に備え、隔離した環境と、触れてよい範囲の制限が欠かせない（筆者の見立て。隔離は Radar の前提条件の1つ）

## 5. 採用を判断するときの観点

- **前提がそろっているか**: 隔離した実行・仕様・自動の検査・検証の仕組みの4つがあるか。無ければ、まずそれらを作ることを目標にする（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/unattended-overnight-agentic-builds)）
- **作業の選び方**: 仕様がはっきりしていて、自動の検査で正しさを確かめやすい作業から始める（筆者の見立て）
- **権限の絞り込み**: 対話なしで動かすときは、使ってよいツールやコマンドを明示して絞る（第3節の Claude Code の例）
- **進み具合の記録**: 進み具合のファイルやコミットで、朝に何が起きたかをたどれるようにする（Anthropic の記事の工夫）
- **費用と時間の上限**: 一晩あたりのトークンや実行時間の上限を決める（筆者の見立て）
- **失敗したときの扱い**: 朝に成果が使えなかったときに、捨てて昼に作り直すのか、続きを任せるのかを決めておく（筆者の見立て）

## 6. 関連技術・代替技術

- **Managed agents runtime**: エージェントを動かす実行の環境を、事業者に任せる手法。夜の作業を動かす場所として補い合う（[解説](managed-agents-runtime.md)。[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/unattended-overnight-agentic-builds)も補い合う技術に挙げています）
- **Agent sandboxing**: エージェントの動ける範囲を閉じ込める手法（[解説](agent-sandboxing.md)）
- **Spec-driven development**、**Feedback sensors for coding agents**、**Harness quality evaluation**、**Coding agent swarms**、**Agent pipeline responsibility**、**Sandboxed execution for coding agents**: [Radar の Unattended overnight agentic builds のページ](https://www.thoughtworks.com/en-us/radar/techniques/unattended-overnight-agentic-builds)の関連の項目に挙がっています
- **代替: 昼間に人が見ながら進める**: 前提がそろわないうちは、作業を小さく分けて、人がそばで見ながら進めるほうが安全（筆者の見立て）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: 夜に人なしで任せられるかは、エージェントに本当に自律して作業させられているかを示すごまかしにくい目印だと見ています。前提の仕組みを持つチームには試す価値があり、持たないチームには、それを整える目標になるとしています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/techniques/unattended-overnight-agentic-builds

## 8. 深掘りするときの一次情報

- [Anthropic: Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [martinfowler.com: MicroservicePrerequisites](https://martinfowler.com/bliki/MicroservicePrerequisites.html)
- [Claude Code Docs: Run Claude Code programmatically](https://code.claude.com/docs/en/headless)

## 9. 確認日

- 内容の確認日: 2026-11-13
- Radar の評価の確認日: 2026-11-13

## 用語集

| 用語 | 説明 |
|---|---|
| AI エージェント | 指示を受けて、ツールを自分で使いながら作業を何段階も進める AI の仕組み |
| コーディングエージェント | コードの読み書き・テストの実行などを自分で進める AI エージェント |
| サブエージェント | 親のエージェントから作業の一部を任され、自分専用のコンテキストで動く別のエージェント |
| コンテキスト | モデルが一度に見ている文章の範囲。会話の履歴や読んだファイルが入る。上限がある |
| サンドボックス | 外に影響が出ないよう、隔離した環境でプログラムを動かす仕組み |
| ハーネス | エージェントの周りの仕組み。ここでは特に、エージェントの変更を確かめる検査の仕組みを指す |
| マイクロサービス | 1つのシステムを、独立して動かせる小さなサービスの集まりとして作る設計 |
| コミット | Git で、変更をひとまとまりの記録として保存すること |
| JSON | データを、名前と値の組で書く、機械が読みやすい文字の形式 |
| CI | 変更のたびに、ビルドやテストを自動で走らせる仕組み |
| バッチ | まとめた処理を一括で実行すること。夜間など決まった時刻に回すことが多い |
| トークン | LLM が文章を数える単位。料金の基準になる |
