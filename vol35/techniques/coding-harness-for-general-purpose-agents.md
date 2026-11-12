# Coding harness for general-purpose agents（コーディングエージェントの実行基盤を汎用のエージェントに使う）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Techniques（開発や運用の進め方・手法） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

Coding harness for general-purpose agents は、コード以外の仕事（調べもの、データの整理、業務の手続きなど）をする **AI エージェント**を作るときに、ゼロから組み立てるのではなく、Claude Code や Codex のような**コーディングエージェントの実行基盤（ハーネス）**を部品として使う手法です。

ハーネスとは、LLM（大規模言語モデル）を「考える→ツールを使う→結果を見る」の繰り返しで動かすための周りの仕組み一式です。会話の状態の管理、ツールの実行、権限の確認、長い会話の扱いなどを含みます。たとえるなら、車を作るのにエンジンから設計するのではなく、実績のある車台（シャシー）を買ってきて、上に載せる荷台だけを自分の用途に合わせて作るようなものです（筆者の見立て）。

## 2. 解決する課題（なぜ生まれたか）

AI エージェントを自分で作る場合、LLM を呼ぶ部分は簡単でも、その周りに必要なものが多くあります。ツールを呼んで結果を返す繰り返しの処理、会話が長くなったときの扱い、途中で止まった作業の再開、危ない操作の前の確認、ファイルやコマンドを扱うツールなどです。これらを毎回作り、試し、直すのは手間がかかります（筆者の見立て）。

一方、コーディングエージェントは、まさにこうした仕組みを、多くの利用者が日々の開発で使う中で鍛えてきました（筆者の見立て）。そして各社が、その実行基盤をプログラムから呼べる形で公開し始めています（第3節）。Anthropic は、Claude Code を動かしているのと同じツール、エージェントの繰り返しの処理、コンテキストの管理を使ってエージェントを作れる、と説明しています（[Claude Agent SDK: Overview](https://code.claude.com/docs/en/agent-sdk/overview)）。

## 3. 仕組みの概要

コーディングエージェントの実行基盤を、自分のプログラムから呼び出します。主な提供の形は次のとおりです。

- **Claude Agent SDK**（Anthropic）: Python と TypeScript で使える、Claude Code の本体（バイナリ）を自分のアプリから動かすライブラリです。ファイルの読み書き、コマンドの実行、Web の検索などのツール、フック、サブエージェント、MCP、会話の続きからの再開や分岐（セッション）が使えます。プロジェクトの `.claude/` フォルダからスキルや指示（メモリ）を読み込むのも Claude Code と同じです（[Overview](https://code.claude.com/docs/en/agent-sdk/overview)、[Sessions](https://code.claude.com/docs/en/agent-sdk/sessions)）
- **Codex SDK と app-server**（OpenAI）: Codex のハーネスは、会話の状態の管理、実行の逐次の通知、ツールの使用、設定した**サンドボックス**（動ける範囲を閉じ込める仕組み）と承認の方針の強制を担います。app-server は、アプリから会話（スレッド）を作り、処理を始め、イベントを受け取り、承認の求めに答えるための、文書化された通信の約束事を公開しています（[OpenAI: Codex as a platform](https://developers.openai.com/blog/codex-as-a-platform)）
- **OpenCode の SDK**: OpenCode のサーバーと通信する、型の付いた JavaScript/TypeScript のクライアントです。セッションの作成、指示の送信、イベントの購読などができます（[OpenCode: SDK](https://opencode.ai/docs/sdk/)）
- **Pi の SDK**: Pi を Node.js や Bun のプロセスに組み込み、エージェント・セッション・ツール・モデルに TypeScript から直接触れられるようにします。セッションは既定でファイルに保存され、会話を分岐させることもできます（[Pi: SDK](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/sdk.md)）

**使い方の流れ（例）**

1. 自分のアプリから SDK でセッションを作り、仕事の指示を送る
2. エージェントは、与えたツール（ファイル、コマンド、MCP でつないだ社内のシステムなど）を使いながら作業を進める
3. 危ない操作の前には、アプリ側で承認の求めを受け、人に確かめるか、規則で許可・拒否する
4. 結果やイベントを受け取り、画面に出したり次の処理に回したりする

OpenAI は、配送の運用のアプリの見本（架空のデータ）で、ダッシュボードの横に Codex を組み込み、アプリが持つ MCP のツールを使わせ、配送の予約を取り直す前には人の承認を求める例を示しています（[Codex as a platform](https://developers.openai.com/blog/codex-as-a-platform)）。

**権限の決め方の例**: Claude Agent SDK では、フック、拒否の規則、確認の規則、権限のモード、許可の規則、最後にアプリが用意する確認の関数（`canUseTool`）の順に、ツールを使ってよいかを判断します（[Claude Agent SDK: Permissions](https://code.claude.com/docs/en/agent-sdk/permissions)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: エージェントの「実行の層」を、自分で作る（エージェントの枠組みのライブラリで繰り返しの処理を組む）か、既存のコーディングエージェントから借りるか、という選択です。後者を選ぶと、アプリのコードは「何をさせるか」（指示・ツール・権限）に集中できます（筆者の見立て）。

**得られるもの**

- **鍛えられた実行基盤**: 会話の状態、ツールの実行、サンドボックスと承認などを、自分で作らずに済む（[Codex as a platform](https://developers.openai.com/blog/codex-as-a-platform)）
- **開発の資産をそのまま使える**: Claude Agent SDK は、Claude Code と同じくプロジェクトのスキルや指示を読み込める（[Overview](https://code.claude.com/docs/en/agent-sdk/overview)）。開発で育てた設定を、そのまま業務のエージェントに持ち込める（筆者の見立て）

**代わりに失うもの・注意点**

- **細かい制御**: 繰り返しの処理の中身は基盤の側が決めるので、自作するより自由が利かない（筆者の見立て）
- **コード向けの前提**: 中身はコーディング用に作られているので、組み込みの指示やツールが業務に合うか確かめ、要らないツールは外す（筆者の見立て）
- **安全の仕組みの差**: サンドボックスや承認の仕組みの有無・作りは、基盤ごとに違う。Pi の SDK の説明には、サンドボックスや承認の記述が見当たらない（[Pi: SDK](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/sdk.md)）
- **費用**: 繰り返しの処理と長い組み込みの指示で、トークン（LLM が文章を数える単位で、料金の基準）が増えやすい（筆者の見立て）
- **利用条件**: Claude Agent SDK で作ったエージェントを他者に提供する場合、事前の承認が無い限り claude.ai のログインや利用枠は使えず、API キーで認証します。製品名に「Claude Code」を使うことも認められていません（[Overview](https://code.claude.com/docs/en/agent-sdk/overview)）

同様の注意点（制御との引き換え、組み込みの指示やツールの見直し、トークンの費用）は、Radar も挙げています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/coding-harness-for-general-purpose-agents)。第7節）。

## 5. 採用を判断するときの観点

- **仕事の性質**: ファイルを読み書きし、コマンドや外部のツールを何度も使う、手順の長い仕事ほど向く。1回の質問と答えで済む仕事なら、LLM の API を直接呼ぶほうが簡単（筆者の見立て）
- **制御の必要度**: 処理の流れを細かく決めたい、決まった手順を必ず踏ませたい場合は、エージェントの枠組みで自作するほうが合うことがある（筆者の見立て）
- **安全**: サンドボックス、承認の方針、ツールの許可・拒否の規則を、使う基盤で確かめる（第3節・第4節）
- **依存先**: 特定の提供元の SDK と LLM に結びつく。OpenCode のように、複数の提供元の LLM を選べる基盤もある（[OpenCode の解説](../tools/opencode.md)）
- **費用の測り方**: 実際の仕事でトークンの量を測り、組み込みの指示の大きさも含めて見積もる（筆者の見立て）

## 6. 関連技術・代替技術

- **エージェントの枠組みで自作する**: 繰り返しの処理を自分で組む、従来の作り方。この手法の代替（筆者の見立て）
- **Agent Skills**: 手順書をフォルダにまとめる形式。Radar のこの項目のページから、Agent Skills の項目へリンクが張られています
- **MCP**: 社内のシステムをエージェントのツールとしてつなぐ共通の約束事（[Model Context Protocol](https://modelcontextprotocol.io/)）
- **OpenAI Codex**、**OpenCode**: この手法で使える基盤の例（[OpenAI Codex の解説](../tools/openai-codex.md)、[OpenCode の解説](../tools/opencode.md)）
- **Agent sandboxing**: エージェントの動ける範囲を閉じ込める手法（[解説](agent-sandboxing.md)）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: 細かい制御を手放す代わりに、試され済みの実行基盤を得られる方法と見ており、注意点として、基盤ごとに安全の仕組みが違うこと（Pi は拡張に任せている）とトークンの費用を挙げています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/techniques/coding-harness-for-general-purpose-agents

## 8. 深掘りするときの一次情報

- [Claude Agent SDK: Overview](https://code.claude.com/docs/en/agent-sdk/overview)
- [Claude Agent SDK: Permissions](https://code.claude.com/docs/en/agent-sdk/permissions)
- [OpenAI: Codex as a platform](https://developers.openai.com/blog/codex-as-a-platform)
- [OpenCode: SDK](https://opencode.ai/docs/sdk/)
- [Pi: SDK](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/sdk.md)

## 9. 確認日

- 内容の確認日: 2026-11-12
- Radar の評価の確認日: 2026-11-12

## 用語集

| 用語 | 説明 |
|---|---|
| AI エージェント | 指示を受けて、ツールを自分で使いながら作業を何段階も進める AI の仕組み |
| コーディングエージェント | コードの読み書きやコマンドの実行を自分で行いながら、開発の作業を進める AI エージェント |
| ハーネス | LLM をエージェントとして動かすための周りの仕組み一式。繰り返しの処理、ツールの実行、会話の状態、権限の確認などを含む |
| LLM（大規模言語モデル） | 大量の文章で学習し、文章を読んで続きを作り出す AI のモデル |
| SDK | あるサービスや仕組みを、自分のプログラムから使うためのライブラリや道具のまとまり |
| セッション | エージェントとの一続きの会話。保存しておけば、あとで続きから再開できる |
| サンドボックス | プログラムが触れられるファイルや通信の範囲を閉じ込め、外に害が及ばないようにする仕組み |
| 承認の方針 | どの操作を自動で許し、どの操作の前に人の確認を求めるかの決まり |
| フック | エージェントの動作の特定の時点（ツールを使う前など）で、決めておいた処理を自動で実行する仕組み |
| サブエージェント | 親のエージェントから仕事を任される、役割を絞った補助のエージェント |
| MCP | AI のアプリと外部のツールやデータをつなぐための共通の約束事（プロトコル） |
| トークン | LLM が文章を数える単位。多くのサービスの料金はトークンの数で決まる |
