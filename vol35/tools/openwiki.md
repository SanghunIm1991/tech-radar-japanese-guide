# OpenWiki（コードベースの wiki を AI で書き、最新に保つ CLI）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Tools（開発や運用で使う道具） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

**名前の手がかり**: Open〈開かれた〉＋ Wiki〈ハワイ語で「速い」。みんなで書き足す Web の文書の呼び名〉

OpenWiki は、コードベースや手元の知識の置き場所を読み、ページどうしがリンクでつながった Markdown の **wiki** を AI に書かせ、ソースが変わるたびに更新する **CLI** です。README は「コードベースのための、エージェント向けの文書を書いて保守する CLI」と説明しています。GitHub の langchain-ai の組織で公開され、ライセンスは MIT です（[README](https://github.com/langchain-ai/openwiki)）。

たとえるなら、図書館の司書が蔵書（ソースコード）を読んで案内の冊子を作り、蔵書が入れ替わるたびに、どのページがどの本のどこに基づくかを確かめて書き直してくれるようなものです。冊子は人が読むためだけでなく、ほかの AI の**エージェント**が下調べに使うためのものでもあります。

## 2. 解決する課題（なぜ生まれたか）

Thoughtworks は、文書は欠かせないものの、最新に保つのが難しい点を出発点に挙げています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/openwiki)）。背景には、次の課題があります。

- **文書がすぐ古くなる**: コードが変わっても文書は手で直さない限りそのままで、いつの間にか実態とずれます
- **文書の中身が何に基づくか分からない**: 書かれた説明が、どのコードのどの版を見て書かれたのかが分からず、まだ正しいのかを判断できません
- **エージェントが毎回コードベースを調べ直す**: エージェントに仕事を頼むたびに、構成や約束事を一から読み直させることになり、時間も手間もかかります（筆者の見立て）
- **人が全体像をつかみにくい**: 文書のページが増えるほど、どこに何が書いてあり、どうつながっているのかが見えにくくなります（筆者の見立て）

## 3. 仕組みの概要

### 課題ごとの答え

| 課題 | OpenWiki の答え |
|---|---|
| 文書がすぐ古くなる | `openwiki --update` で、前回うまく終えたとき以降のリポジトリの変更と、既存の記述の根拠を確かめ、必要なページだけを書き直す。変更が無ければモデルを使わずに終わる。GitHub・GitLab・Bitbucket の **CI** で定期的に動かす例があり、wiki が変わったときに文書の**プルリクエスト**を開く（GitHub Actions には、文書だけのプルリクエストを自動でマージする例もある）（[README](https://github.com/langchain-ai/openwiki)） |
| 文書の中身が何に基づくか分からない | 記述（**Claim**）ごとに、根拠になったファイルと行の範囲（例: `repo://src/server.ts#L40-L82`）と、そのときの根拠の版を記録する。ページごとに、どの時点のコードをもとに作ったかの基準も残す（[README](https://github.com/langchain-ai/openwiki)）。Thoughtworks も、各文書が最後にいつ作られたかを記録する仕組み（チェックポイント）を挙げている（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/openwiki)） |
| エージェントが毎回コードベースを調べ直す | wiki をエージェントの下調べの材料にする。リポジトリの最上位の `AGENTS.md`（エージェント向けの説明書）に、自分の管理する区画を書き込んで保つ。**MCP** のサーバーとして、wiki を検索・読み取りする道具も出す。Codex・Claude Code・Cursor・Copilot などのエージェントに組み込むコマンドもある（[README](https://github.com/langchain-ai/openwiki)） |
| 人が全体像をつかみにくい | `openwiki visualize` で、ページのつながりを点と線の図（ノードのグラフ）にし、横に Markdown の本文を並べて見せる。静的なファイルに書き出して、GitHub Pages などに置くこともできる（[README](https://github.com/langchain-ai/openwiki)）。Thoughtworks も、人が探索できる図を利点に挙げている（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/openwiki)） |

### wiki の作り方

`openwiki --init` を実行すると、AI がリポジトリを調べ、wiki の構成を計画し、ページを1つずつ書いていきます。最後に、記述の根拠の記録や目次、`AGENTS.md` の区画などをまとめて書き出します。成果物は、リポジトリの中の `openwiki/` フォルダに置かれる Markdown です。手元のリポジトリで途中で止まった場合は、同じコマンドをもう一度実行すれば続きから再開できます。使い捨ての CI の環境では、作業の場所を残さない限り最初からやり直しになります（[README](https://github.com/langchain-ai/openwiki)）。

AI のモデルは、OpenAI・Anthropic・Gemini・AWS Bedrock など15の提供元から選べます。OpenAI 互換の窓口を指定すれば、Ollama や LM Studio のように手元で動くモデルも使えます。Claude Code などのエージェントの中から使う場合は、そのエージェントのモデルをそのまま使います（[README](https://github.com/langchain-ai/openwiki)）。

### 持ち運べる形式と個人用のモード

wiki は、Google の **Open Knowledge Format（OKF）** v0.2 の形でも書き出され、OKF に対応したほかの道具でも使えると README は説明しています。また、コードではなく、Gmail・Slack・Notion などの情報源をつないで、自分用の wiki（`~/.openwiki/wiki`）を作るモードもあります（[README](https://github.com/langchain-ai/openwiki)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: コードと人・エージェントのあいだに置く「生きた文書」の層です。手で書く設計文書を置き換えるというより、コードから読み取れることを自動でまとめ、最新に保つ役を受け持ちます。

**代わりに失うもの・注意点**

- **コードをモデルに渡す**: README は明記していませんが、筆者の見立てでは、wiki を書くために、AI はリポジトリのファイルを読み、選んだモデルの提供元に送ることになります。エージェントの中から使う場合も、そのエージェントのモデルの提供元に送られることになります。社外に出せないコードでは、手元のモデルを使うか、提供元との契約を確かめる必要があります。Thoughtworks も、手元のモデルで外に出さずに生成できる点を挙げています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/openwiki)）
- **除外の設定は完全ではない**: `.openwikiignore` に書いた場所は読まれませんが、テストや README、コミットのメッセージなどから推し量って触れられることはありうると README 自身が書いています（[README](https://github.com/langchain-ai/openwiki)）
- **利用統計**: 利用統計は最初から有効です。ファイルの中身・プロンプト・モデルの出力・パスなどは送らないと README は説明しており、環境変数で止められます（[README](https://github.com/langchain-ai/openwiki)）
- **表示の道具はネットにつながる**: 図の表示に使う部品を公開の CDN から読み込むので、手元で開くときもインターネットへの接続が要ります（[README](https://github.com/langchain-ai/openwiki)）
- **若い道具**: 版は 0.x 台で、確認時点の最新は v0.9.0 です。リリースの一覧の最初のページでは、数日〜1週ほどの間隔で新しい版が出ています（[リリースの一覧](https://github.com/langchain-ai/openwiki/releases)）。`--init` をやり直すと、生成した wiki と記述の記録が置き換わります。利用者が書いた `openwiki/INSTRUCTIONS.md` は残ります（[README](https://github.com/langchain-ai/openwiki)）
- **AI が書いた文書の正しさ**: 筆者の見立てでは、根拠の記録があっても、説明の解釈が正しいかは人が確かめる必要があります。文書が増えすぎる問題（第6節の documentation sprawl）にも気を配るべきです

## 5. 採用を判断するときの観点

- **評価はまだ途中**: Thoughtworks は、進行中のプロジェクトへの効果を見極めている段階として Trial に置いています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/openwiki)）
- **コードを渡してよいモデル**: 社外のモデルに送ってよいか。だめなら手元のモデルで十分な質が出るか
- **費用**: 筆者の見立てでは、ページ数とモデルの料金で費用が決まるので、最初の作成と、日々の更新の両方で見積もるのがよいでしょう
- **更新の運用**: CI で定期的に更新し、文書だけのプルリクエストを誰がレビューするかを決められるか
- **入れ方**: `npm install -g openwiki` で入れ、Node.js 22.22.0 以上が要ります（[README](https://github.com/langchain-ai/openwiki)）
- **始め方**: 筆者の見立てでは、外に出してよい小さめのリポジトリで `--init` を試し、できた wiki の正しさと、エージェントの仕事の質が上がるかを確かめてから広げるのが安全です

## 6. 関連技術・代替技術

- **AGENTS.md や CLAUDE.md を手で書く**: エージェント向けの説明書を人が書いて保つやり方（一般的な説明）。OpenWiki は、この説明書に自分の区画を書き足して保つ（[README](https://github.com/langchain-ai/openwiki)）
- **ドキュメントサイトの生成の道具**: コードのコメントや Markdown から文書のサイトを作る道具（一般的な説明）。OpenWiki は、AI が内容そのものを書き、根拠を記録する点が違う
- **AI-driven documentation sprawl**: AI が文書を大量に生み出して手に負えなくなる問題。同じ Vol.35 の Techniques で Caution（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/ai-driven-documentation-sprawl)）。Radar の OpenWiki のページは触れていないが、筆者の見立てでは、文書を自動で作る道具を使うときに合わせて読む価値がある
- **OpenSpec**（[このリポジトリの解説](openspec.md)）: 同じ Vol.35 の Tools の Trial。変更の前に仕様を書いて合意する道具で、OpenWiki がコードから「今の姿」をまとめるのとは向きが逆

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: ソースの変化に合わせて文書を自動で更新し、各文書を最後にいつ作ったかを記録し、人が探索できる図も持つ点を評価しています。進行中のプロジェクトへの効果は、まだ見極めている段階だとしています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/tools/openwiki

## 8. 深掘りするときの一次情報

- [OpenWiki の GitHub リポジトリ（README）](https://github.com/langchain-ai/openwiki)
- [OpenWiki のリリースの一覧](https://github.com/langchain-ai/openwiki/releases)

## 9. 確認日

- 内容の確認日: 2026-11-15（確認時点の最新のリリースは v0.9.0）
- Radar の評価の確認日: 2026-11-15（リングの推移は、Radar の項目のページで確認）

## 用語集

用語の説明の冒頭の「元の語」は、覚えるための手がかりとして、元の語（英単語の意味・略語の元の綴り・名前の元になった語）を短く示したものです。

| 用語 | 説明 |
|---|---|
| wiki | 元の語: ハワイ語の wiki〈速い〉。ページどうしをリンクでつなぎ、書き足していく文書の集まり |
| CLI | 元の語: Command Line Interface の略。文字のコマンドを打って操作する使い方や道具 |
| エージェント | 元の語: agent〈代理人〉。人の指示を受けて、コードを読み書きしたりコマンドを実行したりする AI |
| Markdown | 見出しや箇条書きを記号で書く、軽い文書の書き方 |
| Claim | 元の語: claim〈主張〉。OpenWiki で、wiki の中の1つ1つの記述と、その根拠の記録 |
| チェックポイント | 元の語: checkpoint〈確認の地点〉。どこまで作ったか、いつ作ったかを区切って残した記録 |
| CI | 元の語: Continuous Integration の略（継続的な統合）。変更のたびや決まった時刻に、自動で処理を走らせる仕組み |
| プルリクエスト | 元の語: pull request〈取り込みの依頼〉。変更を共有のリポジトリに取り込んでもらうよう依頼し、レビューを受ける仕組み |
| MCP | 元の語: Model Context Protocol の略。AI のアプリが外部の道具やデータにつなぐための共通の約束事 |
| AGENTS.md | エージェント向けに、リポジトリの構成や約束事を書いておく説明書のファイル |
| OKF | 元の語: Open Knowledge Format の略。知識の文書を、道具を問わず持ち運べるようにするための形式 |
| CDN | 元の語: Content Delivery Network の略。Web の部品やファイルを、世界中の配信の拠点から届ける仕組み |
| MIT ライセンス | 元の語: MIT（Massachusetts Institute of Technology の略。マサチューセッツ工科大学）。著作権の表示を残せば自由に使える、短いオープンソースのライセンス |
