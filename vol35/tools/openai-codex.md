# OpenAI Codex（OpenAI のコーディングエージェント）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Tools（開発や運用に使う道具） |
| リング | Adopt（Trial から移動） |
| 状態 | 下書き |

## 1. 一言でいうと

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
| クラウド | 作業ごとに分けられたクラウドの環境で動かし、変更を確かめてプルリクエストにできる（[Codex Cloud](https://learn.chatgpt.com/docs/cloud)、[クラウドの環境](https://learn.chatgpt.com/docs/environments/cloud-environments)） |

### 3.2 サンドボックスと承認

AI が実行するコマンドの「囲いの広さ」は3段階のモードで決めます（[Sandboxing](https://learn.chatgpt.com/docs/sandboxing)）。

- `read-only`: 読むだけ（編集やコマンドの実行には承認が要る）
- `workspace-write`: 作業中のフォルダの中だけ書ける
- `danger-full-access`: 制限なし（名前のとおり危険）

これとは別に、承認の方針（`approval_policy`）で「囲いの外に出る操作の前に人に聞くか」を決めます。`on-request` は必要なときに聞き、`never` は聞かずに進めます。`never` でもサンドボックスの囲いは効いたままで、公式は CI 向けの組み合わせの例に `read-only` と `never` を挙げています（[承認とセキュリティ](https://learn.chatgpt.com/docs/agent-approvals-security)）。同じページは、既定ではエージェントがネットワークに出られない状態で動くとしています。`workspace-write` では、設定（`network_access = true`）でネットワークを許せます（[Sandboxing](https://learn.chatgpt.com/docs/sandboxing)）。

囲いは OS ごとの仕組みで実現しています。Linux では bubblewrap（Linux の、プロセスの見える範囲を絞る道具）を既定の囲いとし、書き込みを許したフォルダの中でも `.git` と `.codex` は読み取り専用に戻します（[linux-sandbox の README](https://github.com/openai/codex/tree/main/codex-rs/linux-sandbox)）。macOS は OS 組み込みの Seatbelt、Windows は専用のサンドボックスを使います（[Sandboxing](https://learn.chatgpt.com/docs/sandboxing)、[Windows のサンドボックス](https://learn.chatgpt.com/docs/windows/windows-sandbox)）。

### 3.3 設定と指示書

- **設定ファイル**: 個人の既定は `~/.codex/config.toml`、プロジェクトごとの設定は `.codex/config.toml` に書きます。プロジェクトの設定は、そのプロジェクトを信頼したときだけ読み込まれます（[設定の基本](https://learn.chatgpt.com/docs/config-file/config-basic)）
- **AGENTS.md**: 作業の前に読む、AI 向けの指示書です。Codex のホームのフォルダ（`~/.codex`）のものと、リポジトリの根元から今のフォルダまでのものを順につなげて読み、今のフォルダに近いものが後ろに来ます（[AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md)）
- **MCP**（AI に外部の道具やデータをつなぐための共通の取り決め）: `codex mcp add` で外部のサーバーを登録し、AI から使えます（[MCP](https://learn.chatgpt.com/docs/extend/mcp)）

### 3.4 その他の主な機能

- **コードレビュー**: GitHub のプルリクエストの自動レビューや、CLI の `/review` で手元の変更を見て、優先度を付けた指摘を返します（[コードレビュー](https://learn.chatgpt.com/docs/code-review)）
- **GitHub Actions**: 公式のアクション `openai/codex-action` で、CI の中から `codex exec` を動かせます（[GitHub Action](https://learn.chatgpt.com/docs/github-action)）

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: アプリの部品ではなく、**開発の進め方を変える道具**です。人が書いていたコードの一部を AI が書き、人はレビューと判断に回る、という分担の変化を伴います（筆者の見立て）。

**得られるもの**

- **作業の自動化**: 調査・修正・テストの往復を任せられる。クラウドでは複数の作業を並べて進められる（[Codex CLI](https://learn.chatgpt.com/docs/codex/cli)）
- **企業での管理のしやすさ**: OS の仕組みによる囲いと、段階的な承認の方針がある（[Sandboxing](https://learn.chatgpt.com/docs/sandboxing)、[承認とセキュリティ](https://learn.chatgpt.com/docs/agent-approvals-security)）。設定は個人・プロジェクトのほか、管理者が配る層からも読み込まれます（[設定の基本](https://learn.chatgpt.com/docs/config-file/config-basic)）
- **組み込みやすさ**: `codex exec`、GitHub のアクション、TypeScript と Python の SDK で、ほかの仕組みから呼べる（[GitHub: sdk](https://github.com/openai/codex/tree/main/sdk)）

**代わりに失うもの・注意点**

- **出力の質は保証されない**: 動くが古い書き方のコードを出すことがあり、自動テストと人のレビューが欠かせません。Thoughtworks は前の版でこの点を注意しています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/openai-codex)）
- **権限の設計が要る**: 承認とサンドボックスを両方外す指定（`--dangerously-bypass-approvals-and-sandbox`）は、公式が「推奨しない」としており、`danger-full-access` も慎重に使うよう求めています（[承認とセキュリティ](https://learn.chatgpt.com/docs/agent-approvals-security)、[Sandboxing](https://learn.chatgpt.com/docs/sandboxing)）
- **コードが外へ出る**: モデルに渡す指示には手元のコードが含まれ、提供元のサービスへ送られます（筆者の見立て）。公式も、監視のログに指示を残すとコードや機微な情報が記録されうるため、既定の `log_user_prompt = false` を保つよう勧めています（[承認とセキュリティ](https://learn.chatgpt.com/docs/agent-approvals-security)）
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

- 内容の確認日: 2026-11-11
- Radar の評価の確認日: 2026-11-11

## 用語集

| 用語 | 説明 |
|---|---|
| コーディングエージェント | 指示を受けて、コードを読み、書き換え、コマンドを実行するといった複数の手順を自分で選んで進める AI の道具 |
| サンドボックス | プログラムが触れられるファイルやネットワークを、OS の仕組みで限られた範囲に閉じ込める仕組み |
| 承認の方針 | AI が危ないかもしれない操作をする前に、人に確認を求めるかどうかの決まり |
| AGENTS.md | リポジトリに置く（全体用は `~/.codex` に置く）、AI のエージェント向けの指示書。作業の決まりや、テストの走らせ方などを書く |
| MCP | Model Context Protocol。AI のアプリに外部の道具やデータをつなぐための共通の取り決め |
| CLI | コマンドラインインターフェース。文字のコマンドを打って操作する道具 |
| SDK | ソフトウェア開発キット。ほかのプログラムからその機能を呼び出すための部品一式 |
| プルリクエスト | 変更を本流に取り込む前に、レビューを依頼する GitHub などの仕組み |
| API キー | プログラムからサービスを使うときに、利用者を識別するための秘密の文字列。使った分だけ課金されることが多い |
