# OpenCode（オープンソースのコーディングエージェント）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Tools（開発や運用に使う道具） |
| リング | Adopt（Assess から移動） |
| 状態 | 下書き |

## 1. 一言でいうと

OpenCode は、オープンソースの**コーディングエージェント**（指示を受けて、コードを読み、書き換え、コマンドを実行して作業を進める AI の道具）です。端末の中で動く画面（TUI）のほか、デスクトップアプリや Web の画面からも使えます（[OpenCode のドキュメント](https://opencode.ai/docs)）。ライセンスは MIT で、ソースコードは GitHub で公開されています（[GitHub: anomalyco/opencode](https://github.com/anomalyco/opencode)）。

いちばんの特徴は、**特定の AI モデルの提供元に縛られない**ことです。クラウドのモデルにも、社内や手元の PC で動かすモデルにもつなげます（[Providers](https://opencode.ai/docs/providers)）。たとえるなら、エンジン（モデル）を積み替えられる車です。

## 2. 解決する課題（なぜ生まれたか）

コーディングエージェントは、AI モデルを作る会社が自社のモデル向けに出しているものが多く、使えるモデルや費用、データの送り先がその会社に縛られがちです（筆者の見立て）。

一方、企業には「費用を抑えたい」「コードを社外に出せないので社内のモデルを使いたい」「インターネットから切り離された環境（エアギャップ）で使いたい」といった要望があります。また、**オープンウェイトモデル**（学習済みの重みが公開されていて、自分の環境で動かせるモデル）の性能が上がり、自前で動かす選択肢が現実的になってきました。Thoughtworks も、費用の管理やエアギャップを含む制限のある環境に向く点を挙げています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/opencode)）。

OpenCode は、エージェントをオープンソースにし、モデルを差し替えられるようにして、こうした要望に応えます。Thoughtworks は、オープンウェイトモデルの広がりとともに採用が伸びたと見ています（同上）。

## 3. 仕組みの概要

公式のドキュメントは、確認日の時点で「V2」の内容が中心で、V1 から設定の名前がいくつか変わっています（例: `bash` → `shell`。[Permissions](https://opencode.ai/docs/permissions)）。以下は V2 のドキュメントによります。

### 3.1 モデルのつなぎ方

- models.dev（モデルと提供元の一覧をまとめた公開のカタログ）の情報を取り込んでおり、多くの提供元を選べます。画面で `/connect` を打って提供元につなぎ、`/models` でモデルを選びます（[Providers](https://opencode.ai/docs/providers)）
- OpenAI 互換の API（OpenAI の API と同じ形の呼び出し方）を持つ独自の提供元も、設定で足せます（同上）
- 手元のモデルを動かす Ollama・LM Studio・vLLM は、決まったアドレスで動いていれば自動で見つけます（同上）

### 3.2 エージェントの種類

組み込みのエージェントが用意されています（[Agents](https://opencode.ai/docs/agents)）。

| 名前 | 役割 |
|---|---|
| build | 既定のエージェント。ファイルの編集もコマンドの実行も行う |
| plan | 調べて計画を立てる。通常のプロジェクトのファイルは編集しない |
| general | 複数の手順にわたる調査などを任せるサブエージェント（親のエージェントから呼ばれる子のエージェント） |
| explore | コードや Web を探して読むだけのサブエージェント |

自分のエージェントも、`.opencode/agents/<名前>.md` のような Markdown のファイルで定義できます（同上）。

### 3.3 権限の決め方

AI が行う操作（ファイルを読む・編集する・シェルのコマンドを実行する・Web を取得するなど）ごとに、`allow`（許可）・`ask`（毎回確認）・`deny`（拒否）を決めます。規則は並べて書き、最後に当てはまった規則が使われます（[Permissions](https://opencode.ai/docs/permissions)）。

大事なのは**既定値がゆるい**ことです。基本は「すべて許可」で、確認を求めるのは `.env`（秘密の値を書くことが多いファイル）の読み取りなどに限られます（同上）。また、シェルのコマンドは、OpenCode を動かしている利用者と同じ権限でファイル・プロセス・ネットワークに触れると、公式が注意しています（[Tools](https://opencode.ai/docs/tools)）。OS の仕組みで囲う**サンドボックス**についての記述は、確認した範囲のドキュメントには見当たりませんでした。組織で使うときは、管理者が `policies` で強制の拒否を設定できます（[Config](https://opencode.ai/docs/config)）。

### 3.4 設定と指示書

- **設定ファイル**: `opencode.json`（または `opencode.jsonc`）。全体用は `~/.config/opencode/`、プロジェクト用はプロジェクトの根元か `.opencode/` に置きます。管理者用の「システム」の設定がいちばん強く、その次がプロジェクト、その次が全体用です（[Config](https://opencode.ai/docs/config)）
- **AGENTS.md**: AI 向けの指示書です。全体用（`~/.config/opencode/AGENTS.md`）と、作業中のフォルダからプロジェクトの根元（またはホームのフォルダ）まで上へたどった各フォルダのものを読み込みます。V2 は AGENTS.md だけを認識し、`CLAUDE.md` を代わりに読むことはしません（[Instructions](https://opencode.ai/docs/instructions)）

### 3.5 広げる仕組みと、ほかの仕組みからの使い方

- **MCP**（AI に外部の道具やデータをつなぐための共通の取り決め）: 手元で起動するサーバーとネットワーク越しのサーバーの両方に対応します（[MCP servers](https://opencode.ai/docs/mcp-servers)）
- **カスタムコマンド**: よく使う指示を Markdown のファイルにしておき、コマンドとして呼べます（[Commands](https://opencode.ai/docs/commands)）
- **プラグイン**: npm のパッケージやローカルのフォルダから機能を足せます（[Plugins](https://opencode.ai/docs/plugins)）
- **画面なしで動かす**: `opencode run` で、対話の画面を開かずに指示を渡せます（[CLI](https://opencode.ai/docs/cli)）。`opencode serve` でサーバーとして起動でき、プログラムから呼ぶ SDK もあります（[Web と serve](https://opencode.ai/docs/cli/web/)、[SDK](https://opencode.ai/docs/build/sdk/)）

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: アプリの部品ではなく、**開発の進め方を変える道具**です。加えて、画面なしで動かせるため、社内向けの AI の開発基盤の「土台」にもなります。Thoughtworks も、カスタマイズのしやすさと、画面なし・サーバーとしての動かし方を、企業向けの土台としての魅力に挙げています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/opencode)）。

**得られるもの**

- **モデルを選べる**: 費用・性能・データの置き場所に合わせてモデルを替えられる。社内のモデルや手元のモデルを使えば、コードを社外に出さずに済む構成も組める（[Providers](https://opencode.ai/docs/providers)。構成の判断は筆者の見立て）
- **中身を読めて、変えられる**: MIT ライセンスで、エージェント・コマンド・プラグインを自分たちで足せる（[GitHub](https://github.com/anomalyco/opencode)、[Plugins](https://opencode.ai/docs/plugins)）
- **組み込みやすさ**: `opencode run`・サーバー・SDK で、ほかの仕組みから呼べる（[CLI](https://opencode.ai/docs/cli)、[Web と serve](https://opencode.ai/docs/cli/web/)、[SDK](https://opencode.ai/docs/build/sdk/)）

**代わりに失うもの・注意点**

- **安全の設計は自分たちの仕事**: 既定は「ほぼ許可」で、OS による囲いの記述も見当たりません（[Permissions](https://opencode.ai/docs/permissions)、[Tools](https://opencode.ai/docs/tools)）。権限の規則を締めるか、コンテナなどの外側の囲いと組み合わせる必要があります（筆者の見立て。[Agent sandboxing の解説](../techniques/agent-sandboxing.md)）
- **品質はモデル次第**: 積むモデルによって結果が大きく変わり、小さな手元のモデルでは期待した質に届かないことがあります（筆者の見立て）
- **共有や共同作業の機能**: Thoughtworks は、作業の記録の共有や共同作業の機能が、ほかのエージェントより遅れていると指摘しています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/opencode)）
- **変化が速い**: V1 から V2 で設定の名前が変わりました（[Permissions](https://opencode.ai/docs/permissions)）。チームでは版を固定し、誰が移行を確かめるかを決めておく必要があります（筆者の見立て）

## 5. 採用を判断するときの観点

- **費用と契約**: OpenCode 自体はオープンソースですが、モデルの利用料は提供元ごとにかかります。開発元は、オープンウェイトのモデルを中心に、月額の固定料金で使える「OpenCode Go」も提供しています（[OpenCode Go](https://opencode.ai/console/go)）。Thoughtworks は、提供元のサブスクリプションや API を使うときに、ライセンスと利用規約を確かめるよう注意しています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/opencode)）
- **データの置き場所**: 社外に出せないコードなら、社内や手元のモデルを選ぶ（筆者の見立て）
- **権限の既定値**: チームで使うなら、`shell` や `edit`、作業フォルダの外への操作（`external_directory`）を `ask` にするといった規則を決めて配る。強制したい規則は `policies` に書く（[Permissions](https://opencode.ai/docs/permissions)、[Config](https://opencode.ai/docs/config)）
- **OS と導入方法**: macOS・Linux ではスクリプトや npm・Homebrew などで入れられます。Windows では、端末で使う版は配布の zip を使い、Windows のパッケージ管理の道具には対応していません。デスクトップアプリには Windows 版のインストーラーがあります（[OpenCode のドキュメント](https://opencode.ai/docs)）

## 6. 関連技術・代替技術

- **OpenAI Codex・Claude Code**: 同じく端末を中心に使うコーディングエージェント。どちらも開発元が自社のモデルを中心に提供しています（筆者の見立て。[OpenAI Codex の解説](openai-codex.md)）
- **Pi**: Thoughtworks が、OpenCode の代わりとして評価してみる価値があると挙げている、より新しいコーディングエージェント（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/opencode)）
- **Oh My OpenCode**: OpenCode の上に、設定済みのエージェントのチームなどを載せた追加の仕組み。Radar のページで、よく使われている例として触れられています（同上）
- **mise**: 開発の道具の版を管理する道具。OpenCode も mise で入れられます。V2 を入れるには mise 2026.10.9 以降が必要です（[OpenCode のドキュメント](https://opencode.ai/docs)、[mise の解説](mise.md)）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Adopt**（Assess から移動）
- 過去の版: 2026年4月の版で Assess として載った（Radar のページの推移による）
- Thoughtworks の評価の要旨: オープンウェイトモデルや手元のモデルと組み合わせる用途で広く使われるようになったとして、Adopt で採用を勧めています。主な理由は、カスタマイズのしやすさと、画面なしで動かせるため企業向けの土台にしやすいことです
- Radar のページ: https://www.thoughtworks.com/en-us/radar/tools/opencode

## 8. 深掘りするときの一次情報

- [GitHub: anomalyco/opencode](https://github.com/anomalyco/opencode)
- [OpenCode のドキュメント](https://opencode.ai/docs)
- [Providers](https://opencode.ai/docs/providers) と [Agents](https://opencode.ai/docs/agents)
- [Permissions](https://opencode.ai/docs/permissions) と [Config](https://opencode.ai/docs/config)
- [Instructions（AGENTS.md）](https://opencode.ai/docs/instructions)

## 9. 確認日

- 内容の確認日: 2026-11-11
- Radar の評価の確認日: 2026-11-11

## 用語集

| 用語 | 説明 |
|---|---|
| コーディングエージェント | 指示を受けて、コードを読み、書き換え、コマンドを実行するといった複数の手順を自分で選んで進める AI の道具 |
| TUI | テキストユーザーインターフェース。端末の中に、文字で組んだ画面を出して操作する形 |
| オープンウェイトモデル | 学習済みの重み（モデルの中身の数値）が公開されていて、自分のサーバーや PC で動かせる AI モデル |
| エアギャップ | 機密を守るため、インターネットなどの外のネットワークから物理的に切り離した環境 |
| サブエージェント | 親のエージェントから仕事を任される子のエージェント。調査などを分けて行い、結果だけを親に返す |
| サンドボックス | プログラムが触れられるファイルやネットワークを、OS の仕組みで限られた範囲に閉じ込める仕組み |
| AGENTS.md | リポジトリなどに置く、AI のエージェント向けの指示書。作業の決まりや、テストの走らせ方などを書く |
| MCP | Model Context Protocol。AI のアプリに外部の道具やデータをつなぐための共通の取り決め |
| SDK | ソフトウェア開発キット。ほかのプログラムからその機能を呼び出すための部品一式 |
