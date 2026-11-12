# Agent plugin marketplace（エージェントのプラグインのマーケットプレイス）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Techniques（開発や運用の進め方・手法） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

Agent plugin marketplace は、**コーディングエージェント**（コードを読み書きし、コマンドを実行しながら開発の作業を進める AI）に持たせる「部品」を**プラグイン**という単位にまとめ、組織の中のカタログ（**マーケットプレイス**）から配る手法です。部品とは、作業の手順書（スキル）、守らせたい指示、外部のツールとつなぐ設定（MCP サーバー）、特定の役割を持つ補助のエージェント（サブエージェント）の定義などです。

たとえるなら、社内で使うライブラリを各チームがコピーして持つのをやめ、社内のパッケージの置き場から版を指定して取ってくるようにするのと同じ発想を、AI エージェントの設定に当てはめたものです（筆者の見立て）。

## 2. 解決する課題（なぜ生まれたか）

コーディングエージェントは、指示ファイルやスキルを足すと、その組織のやり方に沿って動くようになります。ところが、こうした設定は最初はリポジトリごとに置かれがちです。よくできた設定は別のリポジトリにコピーされ、やがて次のような問題が起きます（筆者の見立て）。

- **版がばらばらになる**: あるリポジトリで直した手順が、コピー先には届かない
- **どれが正しいか分からない**: 似た名前のスキルがあちこちにあり、どれを使えばよいか判断できない
- **見つけられない**: 他のチームの便利な部品の存在を知る手段がない
- **安全の確認が追いつかない**: 部品には、コマンドを自動で実行する仕組み（フック）なども含まれうる。コピーが増えるほど、何が動いているかを把握しにくくなる

ソフトウェアの世界では、同じ問題をパッケージ管理（npm や Maven など）で解いてきました。部品を1か所に置き、名前と版で指定して取り込む方法です。エージェントの設定にも同じ仕組みが要る、というのがこの手法の出発点です（筆者の見立て）。

## 3. 仕組みの概要

ここでは、仕様が公開されている Claude Code を例に説明します。

**プラグイン（配る単位）**

プラグインは、1つのフォルダに部品をまとめたものです。フォルダの中の `.claude-plugin/plugin.json` に名前などを書き、スキル（`skills/` の下の `SKILL.md`）、エージェントの定義（`agents/`）、フック（`hooks/hooks.json`）、MCP サーバーの設定（`.mcp.json`）などを入れます（[Claude Code: Plugins](https://code.claude.com/docs/en/plugins)）。

**マーケットプレイス（カタログ）**

マーケットプレイスは、プラグインの一覧を書いたファイル `.claude-plugin/marketplace.json` を置いた場所です。必須の項目は、マーケットプレイスの名前（`name`）、持ち主（`owner`）、プラグインの一覧（`plugins`）です。一覧の各行には、プラグインの名前と取得元（`source`）を書きます。取得元には、同じリポジトリの中の相対パス、GitHub のリポジトリ、ほかの Git のリポジトリ、npm のパッケージなどを指定でき、Git の場合はブランチやコミットを固定できます（[Claude Code: Create a marketplace](https://code.claude.com/docs/en/plugin-marketplaces)）。つまり、社内の Git のリポジトリ1つを、そのままカタログとして使えます。

**使う側の流れ**

1. カタログを登録する: `/plugin marketplace add <組織名>/<リポジトリ名>`
2. プラグインを入れる: `/plugin install <プラグイン名>@<マーケットプレイス名>`
3. 入れる範囲（自分だけ・プロジェクト全体など）を選ぶ

出典: [Claude Code: Install and manage plugins](https://code.claude.com/docs/en/discover-plugins)

**チームと組織での配り方**

- リポジトリに置く共有の設定ファイル（`.claude/settings.json`）に、カタログ（`extraKnownMarketplaces`）と、有効にするプラグイン（`enabledPlugins`）を書けます。ただし、`enabledPlugins` を書いただけではダウンロードされず、各開発者がインストールの操作をする必要があります（[Install and manage plugins](https://code.claude.com/docs/en/discover-plugins)）。`extraKnownMarketplaces` も、各開発者がそのフォルダを信頼してから効きます（[Claude Code: Settings](https://code.claude.com/docs/en/settings)）
- 組織の管理者は、管理用の設定で、使ってよいカタログを許可リストで絞ったり、特定のプラグインを必ず入れさせたりできます（[Claude Code: Plugins](https://code.claude.com/docs/en/plugins)）

**ほかのツール**

- **VS Code**: Copilot のエージェント向けのプラグインに対応しています（マーケットプレイスを足す設定は、確認日の時点で実験的な扱い）。マーケットプレイスは Git のリポジトリで、Copilot・Claude などの既存のプラグインの形式も読めるとしています（[VS Code: Agent plugins](https://code.visualstudio.com/docs/copilot/customization/agent-plugins)）
- **GitHub Copilot CLI**: カタログのファイルを `.github/plugin/marketplace.json` に置きます（[GitHub Docs: Plugin marketplaces](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-marketplace)）
- **Agent Package Manager（APM）**: 特定のエージェントに縛られない依存関係の管理の道具です。`apm.yml` に使う部品を書き、`apm.lock.yaml` で版を固定し、Copilot・Claude・Cursor・Codex など複数のエージェント向けに入れられます（[microsoft/apm](https://github.com/microsoft/apm)）

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: エージェントを動かす仕組み一式（指示・スキル・ツールなど。**ハーネス**と呼ばれる）を、組織の共通の基盤として配る「配布の層」です。社内のパッケージの置き場や、社内向けのアプリストアと同じ役割にあたります（筆者の見立て）。

**得られるもの**

- **1か所で直せば配れる**: カタログの版を上げれば、利用者は更新の操作で新しい版を取り込める（自動の更新を有効にすれば、操作なしで届く。[Install and manage plugins](https://code.claude.com/docs/en/discover-plugins)）
- **見つけやすさ**: カタログが「この組織で使ってよい部品の一覧」になる（筆者の見立て）
- **統制**: 管理者の設定で、使ってよいカタログを絞れる（第3節）

**代わりに失うもの・注意点**

- **サプライチェーンの危険**: Claude Code のドキュメントは、インストールしたプラグインは利用者の権限でコードを実行できると注意しています（[Claude Code: Plugins](https://code.claude.com/docs/en/plugins)）。VS Code のドキュメントも、フックや MCP サーバーが手元のマシンでコードを動かすため、中身と提供元を確かめてから入れるよう勧めています（[VS Code: Agent plugins](https://code.visualstudio.com/docs/copilot/customization/agent-plugins)）。社内のカタログでも、登録の前の審査が要ります（筆者の見立て）
- **自動の更新の扱い**: Claude Code では、社内の Git のリポジトリなど第三者のカタログは、既定では自動で更新されません（Anthropic の公式のカタログの多くは既定で自動更新。[Install and manage plugins](https://code.claude.com/docs/en/discover-plugins)）。新しい版を全員に行き渡らせるのか、版を固定して確かめてから上げるのかを決める必要があります
- **形式がまだ揃っていない**: ツールごとにファイルの置き場所や設定の名前が違います（第3節）。複数のエージェントを使う組織は、形式の違いを吸収する手間がかかります（筆者の見立て）
- **入れすぎ**: スキルの名前と説明文はコンテキストを毎回使います（[Claude Code: Skills](https://code.claude.com/docs/en/skills)）。配りやすくなったぶん、全員に何でも入れると、エージェントの判断がかえって鈍りえます（筆者の見立て。[Progressive context disclosure の解説](progressive-context-disclosure.md)を参照）

## 5. 採用を判断するときの観点

- **規模**: リポジトリやチームが少なく、設定のコピーで困っていないなら、まだ要らない。同じ部品を複数のチームが使い始めたら検討する（筆者の見立て）
- **持ち主**: カタログと各プラグインを誰が保守するかを決める。持ち主のいない部品が残ると、コピーの問題が形を変えて戻ってくる（筆者の見立て）
- **審査と版の固定**: フックや MCP サーバーを含むプラグインは、登録の前に中身を確かめる。Git の取得元はコミットを固定できる（[Create a marketplace](https://code.claude.com/docs/en/plugin-marketplaces)）
- **使っているエージェントの種類**: 1つのツールに揃っているなら、そのツールのマーケットプレイスの仕組みで足りる。複数なら、APM のような横断の道具も候補になる（筆者の見立て）
- **入れる範囲**: 全員に必須の部品と、使いたい人だけが入れる部品を分ける（筆者の見立て）

## 6. 関連技術・代替技術

- **Agent Skills**: 手順書をフォルダにまとめる形式。プラグインに入れる主な部品の1つ（[Agent Skills の仕様](https://agentskills.io/specification)）
- **MCP**: AI のアプリと外部のツールやデータをつなぐ共通の約束事（[Model Context Protocol](https://modelcontextprotocol.io/)）
- **Enterprise harness**、**Agent Plugins**、**Agent Package Manager**: Radar のこの項目のページから、それぞれの項目へリンクが張られています
- **Tessl のレジストリ**: スキルを検索して入れられる公開の登録簿（[Tessl: Skills Registry](https://tessl.io/registry)）
- **代替: リポジトリごとのコピー**: 規模が小さいうちは、これで足りることも多い（筆者の見立て）
- **代替: 社内のパッケージの置き場**: npm などの既存の仕組みでスキルを配るやり方。Claude Code のカタログも npm のパッケージを取得元に指定できます（[Create a marketplace](https://code.claude.com/docs/en/plugin-marketplaces)）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: スキルや指示などの共有の部品を、リポジトリ間でコピーする代わりに組織全体へ配る方法として挙げています。文書化した Git のリポジトリで実際に作った経験から、組織向けのハーネス（Enterprise harness）を行き渡らせる実用的な方法と見ています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/techniques/agent-plugin-marketplace

## 8. 深掘りするときの一次情報

- [Claude Code: Plugins](https://code.claude.com/docs/en/plugins)
- [Claude Code: Create a marketplace](https://code.claude.com/docs/en/plugin-marketplaces)
- [Claude Code: Install and manage plugins](https://code.claude.com/docs/en/discover-plugins)
- [VS Code: Agent plugins](https://code.visualstudio.com/docs/copilot/customization/agent-plugins)
- [GitHub Docs: Plugin marketplaces（Copilot CLI）](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-marketplace)
- [microsoft/apm](https://github.com/microsoft/apm)

## 9. 確認日

- 内容の確認日: 2026-11-12
- Radar の評価の確認日: 2026-11-12

## 用語集

| 用語 | 説明 |
|---|---|
| コーディングエージェント | コードの読み書きやコマンドの実行を自分で行いながら、開発の作業を何段階も進める AI の仕組み |
| ハーネス | エージェントを動かすための仕組み一式。指示ファイル、スキル、使えるツール、安全のための制限などを含む |
| プラグイン | スキル・エージェントの定義・フック・MCP サーバーの設定などを1つにまとめた、配布の単位 |
| マーケットプレイス | プラグインの一覧（カタログ）。利用者はここに登録されたプラグインを名前で選んで入れる |
| スキル | 作業の手順・スクリプト・資料を1つのフォルダにまとめ、エージェントが必要なときに読み込めるようにしたもの |
| サブエージェント | 親のエージェントから仕事を任される、役割を絞った補助のエージェント |
| フック | エージェントの動作の特定の時点（ツールを使う前など）で、決めておいたコマンドを自動で実行する仕組み |
| MCP | AI のアプリと外部のツールやデータをつなぐための共通の約束事（プロトコル） |
| コンテキスト | LLM（大規模言語モデル）に一度に渡す、指示・会話・資料などの情報の全体 |
| サプライチェーンの危険 | 外から取り込んだ部品に問題（悪意のあるコードなど）が混ざり、それを使う側まで被害が及ぶ危険 |
| パッケージ管理 | ライブラリなどの部品を名前と版で指定して、取得・更新・削除する仕組み。npm や Maven など |
