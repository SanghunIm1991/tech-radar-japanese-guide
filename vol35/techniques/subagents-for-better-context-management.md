# Subagents for better context management（サブエージェントで親の文脈を守る）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Techniques（開発や運用の進め方・手法） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

Subagents for better context management は、AI のコーディングエージェントが、調べもののような作業を**サブエージェント**（親のエージェントから仕事を任される別のエージェント）に任せ、その途中経過を親のエージェントの**コンテキスト**（モデルが一度に見ている文章の範囲。作業の記憶にあたる）に持ち込まないようにする手法です。ファイルの探索、ツールの出力、やり直し、途中の考えはサブエージェントの中にとどめ、親には短い要約だけを返します（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/subagents-for-better-context-management)の定義による）。

サブエージェントは、作業を並列に動かしたり、役割を分けたチームにしたりするために使われることが多いですが（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/subagents-for-better-context-management)）、この手法の主な目的は**親の頭の中を散らかさないこと**です（第2節の Garg の記事は、サブエージェントの本当の価値は速さではなくこの点にあるとしています）。たとえるなら、上司が部下に調査を頼み、調べた資料の山ではなく、1枚のまとめだけを受け取るようなものです（筆者の見立て）。

## 2. 解決する課題（なぜ生まれたか）

エージェントが長い作業をすると、読んだファイルやコマンドの出力がコンテキストに積み上がります。コンテキストに入った情報は、そのあとのすべての判断で、モデルの注意を取り合います。Thoughtworks の Rahul Garg は、トークン（LLM が文章を数える単位）は一度使えば終わるが、コンテキストに残った不要な情報はそのあとのすべてのターンに影響し続けると述べ、親のエージェントの「作業の記憶」こそが足りない資源（一度汚れると会話の終わりまで負担がかかり続ける資源）だとしています。コンテキストの上限が大きくなっても、注意の取り合いは解決しないとも書いています（[martinfowler.com: The Orchestrator's Tax](https://martinfowler.com/articles/orchestrator-tax.html)）。

記事の題名の「オーケストレーター（全体を取り仕切る親）の税金」は、こうした親の記憶が払う代償を指しています（記事は、税金はサブエージェントの側ではなく親の側にあったと結んでいます）。サブエージェントを使っても、親がサブエージェントの記録をまるごと読み込んでしまえば、この代償は払ったままになります（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/subagents-for-better-context-management)も、記録の全文を親に戻すと利点が消えるとしています）。

## 3. 仕組みの概要

**基本の形**

1. 親のエージェントが、作業の一部（コードの調査、ログの確認など）を、目的と返してほしい形を書いてサブエージェントに渡す
2. サブエージェントは**自分専用のコンテキスト**で作業する。親の会話の履歴は見えない
3. 終わったら、結論の要約だけを親に返す
4. 親は要約をもとに次の判断をする

**Claude Code の例**

Claude Code では、サブエージェントはそれぞれ独自のコンテキストと、専用の指示文（システムプロンプト）、使ってよいツールの範囲を持ち、親の会話の履歴は見えない状態で始まります（親の会話を引き継ぐ fork という例外もあります）。定義は Markdown のファイルで、冒頭に名前・いつ使うかの説明・使えるツール・使うモデルなどを書き、プロジェクトの `.claude/agents/` などに置きます。読み取り専用でコードの検索に向けた Explore などの、組み込みのサブエージェントもあります（[Claude Code Docs: Create custom subagents](https://code.claude.com/docs/en/sub-agents)）。

**Anthropic の調査システムの例**

Anthropic は、調べもの用の機能を、親（リード）のエージェントが計画を立て、複数のサブエージェントを並列に動かし、結果をまとめる形で作ったと報告しています。サブエージェントは自分のコンテキストで探索し、最も大事な部分だけを親に返すので、情報の圧縮の役割を果たすとしています（[Anthropic: How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)。この項の出典は同じ）。

同じ記事は、社内の評価で、親に上位のモデル・サブエージェントに中位のモデルを使った構成が、上位のモデル1つで行うより90.2%良い結果だったとしています。一方で、トークンの使用量は、通常のチャットに比べて、エージェントで約4倍、複数のエージェントで約15倍になるとしています。

また、初めのころは簡単な質問に50個ものサブエージェントを作ってしまう失敗があったため、仕事の大きさに合わせた目安を親への指示に書いたとしています。単純な事実の確認ならエージェント1つでツールの呼び出しは3〜10回、比較なら2〜4個のサブエージェント、複雑な調査なら10個を超えるサブエージェント、という目安です。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: エージェントに渡す情報を設計する「コンテキストエンジニアリング」の一つの形です（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/subagents-for-better-context-management)も同様に位置づけています）。ソフトウェアの設計でいえば、細かい処理を関数やモジュールに閉じ込め、呼び出し側には結果だけを返す「カプセル化」に近い考え方です（筆者の見立て）。

**得られるもの**

- **親の判断の質を保つ**: 親のコンテキストが途中経過で埋まらない（第2節の記事）
- **安いモデルの使い分け**: 狭い調査なら、サブエージェントに安いモデルを使える（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/subagents-for-better-context-management)。Claude Code でも定義ごとにモデルを選べる）
- **並列による時短**: 独立した調査を同時に進められる（第3節の Anthropic の記事）

**代わりに失うもの・注意点**

- **記録をまるごと読むと台無し**: サブエージェントの作業の記録全体を親に読み込むと、利点が消える。Garg の記事は、親が記録をまるごと取り込んだことが大きな、避けられたはずのコストだったと振り返っている（最大だったかは、呼び出しごとのトークンを測っていないので仮説だと断っている）（[The Orchestrator's Tax](https://martinfowler.com/articles/orchestrator-tax.html)）
- **同じ理解の作り直し**: 同じ部分のコードを扱う作業を複数のサブエージェントに分けると、それぞれが同じ設計の理解を一から作り直す。同じ理解が要る作業は1つにまとめる（同じ記事）
- **作業どうしの衝突**: 並列に動くサブエージェントが、リポジトリ全体に効く操作（Git の一時退避など）をすると、ほかの作業を壊しうる（同じ記事）
- **トークンの総量は増える**: 親の文脈は軽くなっても、全体の使用量は増える（第3節の Anthropic の記事）
- **向かない仕事**: 全員が同じ文脈を共有する必要がある仕事や、作業どうしの依存が多い仕事には向かない。Anthropic は、多くのコーディングの作業は調べものより並列にしにくいとしている（[Anthropic の記事](https://www.anthropic.com/engineering/multi-agent-research-system)）

## 5. 採用を判断するときの観点

- **作業の形**: 結果を短くまとめて返せる、自己完結した作業か。親とこまめにやりとりする作業なら、親が自分でやるほうがよい（筆者の見立て）
- **同時に動かす数**: 同時に2〜4個を目安にし、それ以上は作業をまとめられないか見直す。Garg の記事は、この目安が特定のモデル向けに調整したもので、どこでも通じる値ではないと断っている（[The Orchestrator's Tax](https://martinfowler.com/articles/orchestrator-tax.html)。2〜4個の目安は [Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/subagents-for-better-context-management)も同様）
- **頼み方**: 目的・返す形・使ってよい道具・作業の範囲を明確に書く。Anthropic の記事は、これらが欠けると作業の重複や抜けが起きたとしている
- **前提の渡し漏れ**: サブエージェントは親の会話を見ていないので、守るべき決まりや必要な知識は、頼むときに明示して渡す。Garg の記事は、サブエージェントが親で有効なスキル（手順書）を自動では引き継がないため、必要なものを名指しで渡すようにしたと述べている（[The Orchestrator's Tax](https://martinfowler.com/articles/orchestrator-tax.html)）
- **効果の測り方**: 親のコンテキストが本当に小さくなっているかを測る。Radar は、この手法の効果はまだ体系的に測られておらず、実践者の経験に基づくとしている（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/subagents-for-better-context-management)）
- **費用**: サブエージェントの分だけトークンが増える。安いモデルの割り当てで相殺できるかを確かめる（筆者の見立て）

## 6. 関連技術・代替技術

- **Context engineering**、**Context profiling**、**Blanket frontier model usage**、**Team of coding agents**: [Radar の Subagents for better context management のページ](https://www.thoughtworks.com/en-us/radar/techniques/subagents-for-better-context-management)の関連の項目に挙がっています
- **Progressive context disclosure**: 必要になったときに必要な情報だけをエージェントに渡す手法。親の文脈を小さく保つという目的が共通する（[解説](progressive-context-disclosure.md)）
- **代替: コンテキストの圧縮や作業の区切り**: 会話が長くなったら要約して詰める、作業ごとに新しい会話で始めるなど、サブエージェントを使わずに文脈を軽くする方法もある（筆者の見立て）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: サブエージェントは並列化や役割分担に使われることが多いが、親の文脈を守ることも使う理由になると見ています。効果がまだ体系的に測られていないため Trial としています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/techniques/subagents-for-better-context-management

## 8. 深掘りするときの一次情報

- [martinfowler.com: The Orchestrator's Tax](https://martinfowler.com/articles/orchestrator-tax.html)
- [Claude Code Docs: Create custom subagents](https://code.claude.com/docs/en/sub-agents)
- [Anthropic: How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)

## 9. 確認日

- 内容の確認日: 2026-11-13
- Radar の評価の確認日: 2026-11-13

## 用語集

| 用語 | 説明 |
|---|---|
| AI エージェント | 指示を受けて、ツールを自分で使いながら作業を何段階も進める AI の仕組み |
| サブエージェント | 親のエージェントから作業の一部を任され、自分専用のコンテキストで動く別のエージェント |
| オーケストレーター | 全体の作業を取り仕切り、サブエージェントに仕事を割り振って結果をまとめる親のエージェント |
| コンテキスト | モデルが一度に見ている文章の範囲。会話の履歴や読んだファイルが入る。上限がある |
| トークン | LLM が文章を数える単位。料金や、コンテキストに入る量の基準になる |
| ターン | 対話の中の1往復のやりとり |
| システムプロンプト | エージェントの役割や守ることを書いた、最初に渡される指示文 |
| 並列 | 複数の作業を同時に進めること |
| カプセル化 | 細かい処理を内側に閉じ込め、外には決まった結果だけを見せる設計の考え方 |
| コンテキストエンジニアリング | エージェントに何を、いつ、どれだけ渡すかを設計すること |
