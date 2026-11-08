# Progressive context disclosure（段階的なコンテキストの開示）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Techniques（開発や運用の進め方・手法） |
| リング | Adopt（前の版の Trial から移動） |
| 状態 | 下書き |

## 1. 一言でいうと

Progressive context disclosure は、**AI エージェント**（指示を受けて、自分でツールを使いながら作業を進める AI）に、必要な情報を最初から全部渡すのではなく、まず「何があるか」の短い一覧だけを見せ、詳しい中身は**必要になったときに読み込ませる**手法です。

たとえるなら、新人に分厚いマニュアルを全部読ませてから仕事を始めさせるのではなく、目次だけを渡しておき、「経費精算をするときは第5章を読んで」と、必要な章をその都度開いてもらうようなものです。

## 2. 解決する課題（なぜ生まれたか）

大規模言語モデル（**LLM**）は、一度に読める量に上限があります。この「一度に読める範囲」を**コンテキストウィンドウ**と呼び、その中に入れる指示や資料を**コンテキスト**と呼びます。量は**トークン**（文字や単語の断片を数える単位）で数えます。

上限に収まれば何を入れてもよい、というわけではありません。Anthropic は、コンテキストを「入れるほど1つあたりの効果が下がる、限りある資源」として扱うべきだと説明しています。LLM には注意を向けられる量の「予算」のようなものがあり、トークンが増えるほど、その予算が薄まるからです（[Anthropic: Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)）。

入力が長くなると性能が落ちる現象は **context rot**（コンテキストの劣化）と呼ばれます。Chroma の研究は、18 の LLM を調べ、課題が単純でも、入力が長くなるにつれて性能が落ちることを報告しています。答えに似ているが答えではない文（妨害文）が混ざると、さらに落ちました（[Chroma: Context Rot](https://www.trychroma.com/research/context-rot)）。

ところが実際には、エージェントへの指示は膨らみがちです。プロジェクトのルール、使えるツールの説明、作業の手順書を全部最初に渡すと、今の作業に関係のない情報が大半を占めます。Thoughtworks は、2026年4月の版で、「〜せよ」「〜するな」を大量に並べた指示がかえって性能を下げると指摘し、必要な情報だけをその場で読み込む方法を勧めていました（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/progressive-context-disclosure)）。

## 3. 仕組みの概要

基本の形は「**一覧（軽い）→ 本文（必要なとき）→ 付属の資料（さらに必要なとき）**」という段階的な読み込みです。代表例が **Agent Skills** です。スキルとは、作業の手順やスクリプトを1つのフォルダにまとめたもので、中心に `SKILL.md` というファイルを置きます（[Agent Skills の仕様](https://agentskills.io/specification)）。

1. **発見**: エージェントの起動時には、各スキルの名前と説明文（`SKILL.md` の先頭のメタデータ）だけを読み込む。仕様では、このメタデータを約100トークンとしている
2. **起動**: 依頼の内容に関係があると判断したら、そのスキルの `SKILL.md` の本文を読み込む。本文は5,000トークン未満、500行未満に抑え、詳細は別ファイルに分けることが勧められている
3. **実行**: 本文から参照されている資料やスクリプトは、本当に必要になったときだけ読む（または実行する）

出典: [Agent Skills の仕様](https://agentskills.io/specification)、[Anthropic: Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)、[Claude Code: Skills](https://code.claude.com/docs/en/skills)

```mermaid
flowchart LR
  A[起動時: 名前と説明文の一覧だけ] -->|依頼に関係がありそう| B[SKILL.md の本文を読む]
  B -->|さらに必要| C[参照資料・スクリプト]
```

**ほかの形**

- **ツール定義の後読み**: Claude の API には、ツールの定義を最初はコンテキストに載せず、エージェントが検索して見つけたものだけを読み込む仕組みがあります。Anthropic は、複数の外部サービスをつないだ構成ではツールの定義だけで約55,000トークンを使うことがあり、この仕組みで85%以上減らせると主張しています（開発元の主張。[Claude: Tool search tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)）
- **必要なときに取りに行く**: 資料の中身を先に渡す代わりに、ファイルの場所やリンクだけを持たせ、エージェントが検索や読み込みのツールでその場で取りに行くやり方です。Anthropic は、これを「just in time」の取得と呼んでいます（[Anthropic: Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)）
- **指示ファイルを短く保つ**: AGENTS.md は、コーディングエージェント向けの指示を書く Markdown のファイルです（[agents.md](https://agents.md/)）。Thoughtworks は、これを短く保ち、詳しい資料へのリンクを置くことを、この手法の例に挙げています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/progressive-context-disclosure)）

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: AI エージェントを組み込んだシステムで、「何を、いつ、どれだけ LLM に見せるか」を設計する **context engineering**（コンテキストの設計）の基本の手法の1つです。人間向けの画面設計で、最初は要点だけを見せ、詳しい設定は「詳細」を押したときに見せる「段階的な開示」と同じ発想です（筆者の見立て）。

**得られるもの**

- **費用と速さ**: 毎回送るトークンが減るので、料金と応答時間を抑えやすい（Thoughtworks もトークンと費用の削減を挙げています〈[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/progressive-context-disclosure)〉）
- **精度**: 関係のない情報が減り、context rot の影響を受けにくくなる。Claude のドキュメントも、ツールが30〜50個を超えると正しいツールを選ぶ精度が落ちるとしています（[Claude: Tool search tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)）

**代わりに失うもの・注意点**

- **見落とし**: エージェントが「この情報が必要だ」と気づかなければ、読み込まれません。Anthropic は、適切な手がかりが無いと、ツールの使い方を誤ったり、行き止まりを追ったり、大事な情報を見落としたりすると注意しています（[Anthropic: Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)）
- **説明文の書き方に左右される**: 一覧の説明文が判断材料のすべてです。Anthropic は、スキルの名前と説明文に特に気を配るよう勧めています（[Anthropic: Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)）
- **読み込んだ本文は残る**: Claude Code では、一度読み込んだスキルの本文はその後のやりとりでもコンテキストに残り、一覧の説明文も毎回トークンを使います。本文も短く保つ必要があります（[Claude Code: Skills](https://code.claude.com/docs/en/skills)）
- **遅さと手間**: その場で探すのは、先に用意したデータを使うより時間がかかります（[Anthropic: Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)）。往復が増えて作業が長くなることもあります（筆者の見立て）
- **何でも後読みにすればよいわけではない**: いつも使うツールまで後読みにすると、検索の手間が増えるだけです。Claude のドキュメントは、ツールが10個未満なら tool search を使わないほうがよく、よく使う3〜5個は最初から載せておくよう勧めています（[Claude: Tool search tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)）

## 5. 採用を判断するときの観点

- **規模**: 全部を渡しても短いなら、無理に分けなくてよい。指示が膨らんできたら分ける（筆者の見立て。ツールの数の目安は第4節）
- **常に必要なものと、場合による情報を分ける**: 安全のための決まりなど、どの作業でも守らせたいものは最初から渡し、特定の作業でしか使わない手順を後読みにする。Anthropic も、最初に読み込む情報と、その場で探す情報を組み合わせるやり方を勧めています（[Anthropic: Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)）
- **説明文の品質**: 「いつ使うか」が分かるように書き、呼ばれるべき場面で呼ばれているかを確かめる
- **費用の測り方**: 後読みで読み込まれた分も、通常の入力トークンとして数えられます。また、後読みにしたツールの定義も、リクエストには毎回すべて含めて送ります（コンテキストに載るのは見つかったものだけ）（[Claude: Tool search tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)）。減った量は実際の作業で測る
- **依存先**: Agent Skills や AGENTS.md は公開の形式です（[agentskills.io](https://agentskills.io/)、[agents.md](https://agents.md/)）。ただし、どこまで対応しているかはツールによって違いうるので、使うエージェントの対応状況を確かめる（筆者の見立て）

## 6. 関連技術・代替技術

- **Context engineering**: LLM に何をどう見せるかを設計すること全般。この手法はその一部（Radar のこの項目のページから、Context engineering の項目へリンクが張られている）
- **Agent Skills**: 段階的な読み込みを前提にした、スキルの形式（第3節）
- **AGENTS.md**: コーディングエージェント向けの指示ファイル（第3節）
- **RAG（検索拡張生成）**: 質問に関係のある文書を検索して、その部分だけを LLM に渡す手法。「必要な情報だけを渡す」点で近い仲間（一般的な説明）
- **MCP**: AI のアプリと外部のツールやデータをつなぐ共通の約束事（[Model Context Protocol](https://modelcontextprotocol.io/)）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Adopt**（Trial から移動）
- 過去の版: 2026年4月の版で Trial（Radar のページの推移による）
- Thoughtworks の評価の要旨: 最小限の情報から始めて必要に応じて足していくこの手法は、本格的な AI のアプリではほぼ当たり前になったと見ており、AGENTS.md、スキル、RAG などを例に挙げています。効果としては、費用の削減と context rot の抑制を挙げています。4月の版では、エージェント自身に必要な情報を選ばせるやり方として紹介していました
- Radar のページ: https://www.thoughtworks.com/en-us/radar/techniques/progressive-context-disclosure

## 8. 深掘りするときの一次情報

- [Agent Skills の仕様](https://agentskills.io/specification)
- [Anthropic: Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Anthropic: Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Chroma: Context Rot](https://www.trychroma.com/research/context-rot)
- [Claude: Tool search tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool)

## 9. 確認日

- 内容の確認日: 2026-11-08
- Radar の評価の確認日: 2026-11-08

## 用語集

| 用語 | 説明 |
|---|---|
| LLM（大規模言語モデル） | 大量の文章で学習し、文章を読んで続きを作り出す AI のモデル |
| AI エージェント | 指示を受けて、ファイルの読み書きやコマンドの実行などのツールを自分で使いながら、作業を何段階も進める AI の仕組み |
| コンテキスト | LLM に一度に渡す、指示・会話・資料などの情報の全体 |
| コンテキストウィンドウ | LLM が一度に読める量の上限。トークンの数で表す |
| トークン | LLM が文章を数える単位。おおよそ単語や文字の断片にあたる。多くのサービスの料金はトークンの数で決まる |
| context rot | 入力が長くなるにつれて、LLM の正確さが落ちていく現象 |
| context engineering | LLM に何を、いつ、どれだけ見せるかを設計すること |
| Agent Skills | 作業の手順・スクリプト・資料を1つのフォルダにまとめ、エージェントが必要なときに読み込めるようにする形式 |
| AGENTS.md | コーディングエージェント向けに、ビルドやテストの手順、コードの書き方の決まりなどを書いておく Markdown のファイル |
| MCP | AI のアプリと外部のツールやデータをつなぐための共通の約束事（プロトコル） |
| RAG（検索拡張生成） | 質問に関係のある文書を検索し、その部分を LLM に渡して答えさせる手法 |
| ツール定義 | エージェントが使えるツールの名前・説明・受け取る値を書いたもの。これもトークンを使う |
