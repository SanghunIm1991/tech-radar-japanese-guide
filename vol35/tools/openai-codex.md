# OpenAI Codex（OpenAI のコーディングエージェント）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Tools（開発や運用に使う道具） |
| リング | Adopt（Trial から移動） |
| 状態 | 下書き |
| 図解ページ | [図と動きで見る図解ページ](openai-codex-intro.html)（この解説の内容を図と動きで見せるページ。サイトで表示。GitHub のリポジトリではソースが表示されます） |

## 1. 一言でいうと

**名前の手がかり**: OpenAI〈会社名。Open〈開かれた〉＋ AI〈Artificial Intelligence、人工知能〉〉＋ Codex〈ラテン語で、紙を綴じた冊子の形の本・写本〉

OpenAI Codex は、OpenAI が提供する**コーディングエージェント**（指示を受けて、コードを読み、書き換え、コマンドを実行して作業を進める AI）です。手元の端末で動く CLI（コマンドラインの道具）のほか、デスクトップアプリ、エディタの拡張、クラウドで動く形があります（[Codex CLI](https://learn.chatgpt.com/docs/codex/cli)）。CLI の本体は Rust で書かれ（[インストールの説明](https://github.com/openai/codex/blob/main/docs/install.md)）、Apache-2.0 ライセンスのオープンソースとして公開されています（[GitHub: openai/codex](https://github.com/openai/codex)）。

たとえるなら、「この不具合を直して」と頼むと、自分でファイルを開き、直し、テストを走らせて結果を報告してくれる同僚です。ただし、どこまで勝手に動いてよいかは、こちらが決めて囲っておく必要があります。

## 2. 解決する課題（なぜ生まれたか）

チャット型の AI に質問してコード片をもらう使い方では、コードを貼り付け、結果を試し、エラーを貼り戻す往復を人が担います。コーディングエージェントは、この往復を AI 自身に任せ、リポジトリを直接読み書きし、テストなどのコマンドを実行して確かめながら進めます。

一方で、AI にコマンドを実行させると、ファイルを消す、秘密の情報を外へ送る、といった事故の危険が生まれます。そこで Codex は、AI の動ける範囲を OS の仕組みで囲う**サンドボックス**と、危ない操作の前に人の承認を求める**承認の方針**を組み合わせています（[Sandboxing](https://learn.chatgpt.com/docs/sandboxing)）。

## 3. 仕組みの概要

### 3.1 使う場所

| 形 | 内容 |
|---|---|
| CLI | 端末で `codex` と打って対話する。`codex exec` でスクリプトや CI（変更のたびに自動でビルドやテストを行う仕組み）からも使える（[非対話モード](https://learn.chatgpt.com/docs/non-interactive-mode)） |
| エディタの拡張・デスクトップアプリ | 画面上で同じエージェントを使う（[Codex CLI](https://learn.chatgpt.com/docs/codex/cli)） |
| クラウド | 作業ごとに分けられたクラウドの環境で動かし、変更を確かめてプルリクエストにできる。ChatGPT の Web やモバイルアプリからも使える（[Codex CLI](https://learn.chatgpt.com/docs/codex/cli)、[Codex Cloud](https://learn.chatgpt.com/docs/cloud)、[クラウドの環境](https://learn.chatgpt.com/docs/environments/cloud-environments)） |

### 3.2 サンドボックスと承認

AI が実行するコマンドの「囲いの広さ」は3段階のモードで決めます（[Sandboxing](https://learn.chatgpt.com/docs/sandboxing)）。

- `read-only`: 読むだけ（書き込みや、囲いの外に出る操作には承認が要る）
- `workspace-write`: 作業中のフォルダの中だけ書ける
- `danger-full-access`: 制限なし（名前のとおり危険）

これとは別に、承認の方針（`approval_policy`）で「囲いの外に出る操作の前に人に聞くか」を決めます。`on-request` は必要なときに聞き、`never` は聞かずに進めます。`never` でもサンドボックスの囲いは効いたままで、公式は CI 向けの組み合わせの例に `read-only` と `never` を挙げています。この組み合わせでは、書き込みの要らないコマンドを囲いの中で動かし、承認を求めずに進めます（[承認とセキュリティ](https://learn.chatgpt.com/docs/agent-approvals-security)）。同じページは、既定ではエージェントがネットワークに出られない状態で動くとし、`workspace-write` では設定（`sandbox_workspace_write.network_access = true`）でネットワークを許せるとしています（[承認とセキュリティ](https://learn.chatgpt.com/docs/agent-approvals-security)）。

囲いは OS ごとの仕組みで実現しています。Linux では bubblewrap（Linux の、プロセスの見える範囲を絞る道具。パッケージマネージャーで入れておく前提で、無いときは同梱の補助の仕組みに切り替わる）を既定の囲いとし、書き込みを許したフォルダの中でも `.git` と `.codex` は読み取り専用に戻します（[linux-sandbox の README](https://github.com/openai/codex/tree/main/codex-rs/linux-sandbox)）。macOS は OS 組み込みの Seatbelt、Windows は専用のサンドボックスを使います（[Sandboxing](https://learn.chatgpt.com/docs/sandboxing)、[Windows のサンドボックス](https://learn.chatgpt.com/docs/windows/windows-sandbox)）。

### 3.3 設定と指示書

- **設定ファイル**: 個人の既定は `~/.codex/config.toml`、プロジェクトごとの設定は `.codex/config.toml` に書きます。プロジェクトの設定は、そのプロジェクトを信頼したときだけ読み込まれます（[設定の基本](https://learn.chatgpt.com/docs/config-file/config-basic)）
- **AGENTS.md**: 作業の前に読む、AI 向けの指示書です。Codex のホームのフォルダ（`~/.codex`）のものと、リポジトリの根元から今のフォルダまでのものを順につなげて読み、今のフォルダに近いものが後ろに来ます（[AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md)）
- **MCP**（AI に外部の道具やデータをつなぐための共通の取り決め）: `codex mcp add` で外部のサーバーを登録し、AI から使えます（[MCP](https://learn.chatgpt.com/docs/extend/mcp)）

### 3.4 その他の主な機能

- **コードレビュー**: GitHub のプルリクエストの自動レビューや、CLI の `/review` で手元の変更を見て、優先度を付けた指摘を返します（[コードレビュー](https://learn.chatgpt.com/docs/code-review)）
- **GitHub Actions**: 公式のアクション `openai/codex-action` で、CI の中から `codex exec` を動かせます（[GitHub Action](https://learn.chatgpt.com/docs/github-action)）

### 3.5 第2節の課題への答え

- **コードを貼り付け、試し、エラーを貼り戻す往復を人が担う**: Codex は、リポジトリのファイルを自分で読んで書き換え、テストなどのコマンドを実行して結果を確かめながら作業を進めます。手元の CLI・エディタ・デスクトップアプリのほか、クラウドでは作業ごとに分けた環境で複数の作業を並べて進め、変更をプルリクエストにできます（[Codex CLI](https://learn.chatgpt.com/docs/codex/cli)、[Codex Cloud](https://learn.chatgpt.com/docs/cloud)）。`codex exec` を使えば、人が端末の前にいなくても、スクリプトや CI から作業を任せられます（[非対話モード](https://learn.chatgpt.com/docs/non-interactive-mode)）
- **AI にコマンドを実行させると、ファイルを消す・秘密を外へ送るといった事故の危険が生まれる**: サンドボックスのモードで書き込める範囲を決め（`workspace-write` なら作業中のフォルダの中だけ）、承認の方針で囲いの外に出る操作の前に人に聞くかを決めます（[Sandboxing](https://learn.chatgpt.com/docs/sandboxing)、[承認とセキュリティ](https://learn.chatgpt.com/docs/agent-approvals-security)）。既定ではネットワークに出られない状態で動くので、秘密を外へ送る経路は狭まります。ただし、承認したコマンドや、許可したネットワーク・Web 検索・MCP のサーバーは囲いの外に出られるので、閉じきれるわけではありません（[承認とセキュリティ](https://learn.chatgpt.com/docs/agent-approvals-security)、[非対話モード](https://learn.chatgpt.com/docs/non-interactive-mode)）。Linux では、書き込みを許したフォルダの中でも `.git` と `.codex` は読み取り専用に戻します（[linux-sandbox の README](https://github.com/openai/codex/tree/main/codex-rs/linux-sandbox)）。ただし、囲いを外す指定もあるので、どの設定で使うかは利用者が決めます（第4節）

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: アプリの部品ではなく、**開発の進め方を変える道具**です。人が書いていたコードの一部を AI が書き、人はレビューと判断に回る、という分担の変化を伴います（筆者の見立て）。

**得られるもの**

- **作業の自動化**: 調査・修正・テストの往復を任せられる。クラウドでは複数の作業を並べて進められる（[Codex CLI](https://learn.chatgpt.com/docs/codex/cli)）
- **企業での管理のしやすさ**: OS の仕組みによる囲いと、段階的な承認の方針がある（[Sandboxing](https://learn.chatgpt.com/docs/sandboxing)、[承認とセキュリティ](https://learn.chatgpt.com/docs/agent-approvals-security)）。設定は個人・プロジェクトのほか、組織のワークスペースにクラウドから配られる既定値や、マシン全体の設定からも読み込まれます。管理された PC では、組織が `requirements.toml` で設定を縛ることもできます（[設定の基本](https://learn.chatgpt.com/docs/config-file/config-basic)）
- **組み込みやすさ**: `codex exec`、GitHub のアクション、TypeScript と Python の SDK で、ほかの仕組みから呼べる（[GitHub: sdk](https://github.com/openai/codex/tree/main/sdk)）

**代わりに失うもの・注意点**

- **出力の質は保証されない**: 動くが古い書き方のコードを出すことがあり、自動テストと人のレビューが欠かせません。Thoughtworks は前の版（2026年4月）でこの点を注意しています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/openai-codex)）
- **権限の設計が要る**: 承認とサンドボックスを両方外す指定（`--dangerously-bypass-approvals-and-sandbox`）は、公式が「推奨しない」としており、`danger-full-access` も慎重に使うよう求めています（[承認とセキュリティ](https://learn.chatgpt.com/docs/agent-approvals-security)、[Sandboxing](https://learn.chatgpt.com/docs/sandboxing)）
- **コードが外へ出る**: モデルに渡す指示には手元のコードが含まれ、提供元のサービスへ送られます（筆者の見立て）。公式も、監視のログに指示を残すとコードや機微な情報が記録されうるため、`log_user_prompt = false` のままにするよう勧めています（[承認とセキュリティ](https://learn.chatgpt.com/docs/agent-approvals-security)）
- **提供元への依存**: 本体はオープンソースですが、開発は OpenAI が主導しており、コミュニティで運営されているわけではありません（Thoughtworks の指摘。[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/openai-codex)）

## 5. 採用を判断するときの観点

- **費用と契約**: ChatGPT のプランでサインインすると、プランに含まれる利用枠の中で使えます。API キーでサインインすると、API の通常の料金がかかります（[料金](https://learn.chatgpt.com/docs/pricing)、[認証](https://learn.chatgpt.com/docs/auth)）。クラウドの機能には ChatGPT でのサインインが必要です（[認証](https://learn.chatgpt.com/docs/auth)）
- **データの扱い**: 社外に出してよいコードかを先に決める。学習への利用や保存期間は、契約の種類ごとの規約で確かめる（筆者の見立て）
- **権限の既定値**: チームで使うなら、モードと承認の方針、ネットワークの可否を設定で決めておく。CI では、`codex exec` は既定で `read-only` で動きます（[非対話モード](https://learn.chatgpt.com/docs/non-interactive-mode)）
- **OS**: macOS・Linux に加え、Windows でも PowerShell のまま専用のサンドボックスで動かせます。Windows 11 が推奨で、Windows 10 は最善努力の扱いです（[Windows のサンドボックス](https://learn.chatgpt.com/docs/windows/windows-sandbox)）
- **レビューの体制**: AI の変更を誰がどう確かめるか。テストが薄いリポジトリでは効果より危険が勝ちやすい（筆者の見立て）

## 6. 関連技術・代替技術

- **Claude Code・OpenCode**: 同じく端末を中心に使うコーディングエージェント。Radar のこの項目のページで、Codex と並べて挙げられています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/openai-codex)）
- **Agent sandboxing**: AI エージェントの動ける範囲を囲う手法。Codex のサンドボックスはその実装の一例です（筆者の見立て。[解説](../techniques/agent-sandboxing.md)）
- **Progressive context disclosure**: AI に渡す情報を必要に応じて段階的に見せる手法。AGENTS.md やスキルはその道具になります（筆者の見立て。[解説](../techniques/progressive-context-disclosure.md)）
- **Coding harness for general-purpose agents・Open-weight models**: Radar のこの項目のページで関連の項目として挙がり、同じ版の Techniques の Trial に載っています。Thoughtworks は、Codex の中核のエージェントの仕組みが Apache-2.0 で公開されていることを前者の手法に使える点として、OpenAI 以外のモデルも使えることを後者（公開された重みのモデル）を動かす選択肢になる点として挙げています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/openai-codex)、このリポジトリの解説: [Coding harness for general-purpose agents](../techniques/coding-harness-for-general-purpose-agents.md)、[Open-weight models](../techniques/open-weight-models.md)）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Adopt**（Trial から移動）
- 過去の版: 2026年4月の版で Trial として載った（Radar のページの推移による）
- Thoughtworks の評価の要旨: チームで日常的に使えるコーディングエージェントの1つとして勧めています。主な理由は、企業で使うための管理の機能と、ほかのエージェントや OpenAI 以外のモデルとの組み合わせやすさです
- Radar のページ: https://www.thoughtworks.com/en-us/radar/tools/openai-codex

## 8. 深掘りするときの一次情報

- [GitHub: openai/codex](https://github.com/openai/codex)
- [Codex CLI](https://learn.chatgpt.com/docs/codex/cli)
- [Sandboxing](https://learn.chatgpt.com/docs/sandboxing) と [承認とセキュリティ](https://learn.chatgpt.com/docs/agent-approvals-security)
- [設定の基本](https://learn.chatgpt.com/docs/config-file/config-basic) と [AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md)
- [料金](https://learn.chatgpt.com/docs/pricing)

公式ドキュメントの URL（developers.openai.com/codex）は、確認日の時点で learn.chatgpt.com へ転送されていたため、転送先の URL を載せています。

## 9. 確認日

- 内容の確認日: 2026-11-17（2026-11-11 に書き、2026-11-17 に名前の手がかり・課題ごとの答え・元の語を足して、全体を点検し直した）
- Radar の評価の確認日: 2026-11-17

## 用語集

用語の説明の冒頭の「元の語」は、覚えるための手がかりとして、元の語（英単語の意味・略語の元の綴り・名前の元になった語）を短く示したものです。

| 用語 | 説明 |
|---|---|
| コーディングエージェント | 元の語: coding〈プログラムを書くこと〉＋ agent〈代わりに行動する者・代理人〉。指示を受けて、コードを読み、書き換え、コマンドを実行するといった複数の手順を自分で選んで進める AI の道具 |
| サンドボックス | 元の語: sandbox〈子どもが遊ぶ砂場〉。プログラムが触れられるファイルやネットワークを、OS の仕組みで限られた範囲に閉じ込める仕組み |
| 承認の方針 | 元の語: approval〈承認〉＋ policy〈方針〉（設定の名前は `approval_policy`）。AI が危ないかもしれない操作をする前に、人に確認を求めるかどうかの決まり |
| AGENTS.md | 元の語: agents〈エージェントたち〉＋ .md（Markdown のファイルの拡張子）。リポジトリに置く（全体用は `~/.codex` に置く）、AI のエージェント向けの指示書。作業の決まりや、テストの走らせ方などを書く |
| MCP | 元の語: Model Context Protocol の略（モデルにコンテキストを渡すための約束事）。AI のアプリに外部の道具やデータをつなぐための共通の取り決め |
| CLI | 元の語: Command Line Interface の略（コマンドの行で操作する接点）。文字のコマンドを打って操作する道具 |
| CI | 元の語: Continuous Integration の略（継続的な統合）。変更のたびに、自動でビルドやテストを行う仕組み |
| SDK | 元の語: Software Development Kit の略（ソフトウェアの開発の道具一式）。ほかのプログラムからその機能を呼び出すための部品一式 |
| プルリクエスト | 元の語: pull〈引く〉＋ request〈依頼〉（自分の変更を引き込んでもらう依頼）。変更を本流に取り込む前に、レビューを依頼する GitHub などの仕組み |
| API キー | 元の語: API は Application Programming Interface の略（アプリをプログラムから使うための接点）＋ key〈鍵〉。プログラムからサービスを使うときに、利用者を識別するための秘密の文字列。使った分だけ課金されることが多い |
