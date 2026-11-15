# Sourcebot（自社で動かす、人とエージェントのためのコード検索）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Tools（開発や運用で使う道具） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

**名前の手がかり**: Source〈源。ここではソースコード〉＋ bot〈robot の略。自動で動くプログラム〉

Sourcebot は、会社の中にある大量のリポジトリをまとめて検索し、コードを読み進められるようにする道具です。公式サイトは「人とエージェントが、複雑な企業のコードベースを検索し、たどり、理解するのを助ける、自社で動かす（**セルフホスト**の）道具」と説明しています。開発元は Taqla Inc. で、GitHub のリポジトリのライセンスの表示は MIT です（[公式サイト](https://www.sourcebot.dev/)、[GitHub のリポジトリ](https://github.com/sourcebot-dev/sourcebot)）。

たとえるなら、社内の全部の書庫の本に索引を付け、1つの窓口で「この言葉はどの本のどこに出てくるか」をすばやく引けるようにした図書館です。窓口には人だけでなく、AI の**エージェント**も問い合わせに来られます。

## 2. 解決する課題（なぜ生まれたか）

Thoughtworks は、Sourcebot が、多数のリポジトリにまたがる大きなコードベースの「文脈の層」になる点を挙げています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/sourcebot)）。背景には、次の課題があります。

- **目的のコードが見つからない**: コードが何百・何千ものリポジトリに分かれていると、どこに何があるのかを探すだけで時間がかかります。Radar も、開発者には多数のリポジトリにまたがる広い文脈が要ると書いています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/sourcebot)）
- **コードをたどるのに手間がかかる**: 関数の定義や使われている場所をたどるには、リポジトリを手元に取ってきて開発環境で開く必要があります（筆者の見立て）
- **エージェントが手元のリポジトリしか見えない**: Radar は、大規模な**リファクタリング**や古いシステムの調査には、エージェントにも開発者と同じ広い文脈が要ると書いています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/sourcebot)）。筆者の見立てでは、コーディングエージェントはふつう作業中のリポジトリしか読めないため、その文脈が足りません
- **開発者以外がコードについて聞きにくい**: コードの振る舞いを知りたい開発者以外の人は、疑問の答えを開発者に頼りがちです（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/sourcebot)の趣旨）

## 3. 仕組みの概要

### 課題ごとの答え

| 課題 | Sourcebot の答え |
|---|---|
| 目的のコードが見つからない | GitHub・GitLab・Bitbucket・Azure DevOps・Gitea・Gerrit などからリポジトリを取り込み、**トライグラム**（3文字ずつの並び）の索引を作って、大きなコードベースでも速く検索する。正規表現や、リポジトリ（`repo:`）・言語（`lang:`）・ファイル（`file:`）などの絞り込みが使える（[コード検索](https://docs.sourcebot.dev/docs/features/search/code-search)、[検索の書き方](https://docs.sourcebot.dev/docs/features/search/syntax-reference)、[コードの取り込み](https://docs.sourcebot.dev/docs/connections/indexing-your-code)） |
| コードをたどるのに手間がかかる | ブラウザの上で、定義へ移動・使われている場所の一覧（参照）・リポジトリをまたいだ検索ができる。言語サーバーではなく検索をもとにした仕組みで、索引を作るときに **ctags** という道具で関数などの名前（シンボル）を拾っておき、その名前を検索して定義や参照を見つける（[コードの移動](https://docs.sourcebot.dev/docs/features/code-navigation)） |
| エージェントが手元のリポジトリしか見えない | **MCP** のサーバーを持ち、Claude Code・Cursor などのエージェントから、コードの検索・ファイルの読み取り・定義と参照の検索・リポジトリの一覧などを使える（[MCP サーバー](https://docs.sourcebot.dev/docs/features/mcp-server)）。Thoughtworks も、エージェントが組織のコードベース全体を検索できる点を挙げている（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/sourcebot)） |
| 開発者以外がコードについて聞きにくい | **Ask Sourcebot** で、コードベースについて普通の言葉で質問できる。推論の得意なモデルが、検索とコードの移動の機能を使ってコードを調べ、引用つきで答える。使うモデルの提供元は自分で設定する（[Ask Sourcebot](https://docs.sourcebot.dev/docs/features/ask/ask-sourcebot)） |

### 動かし方

Docker のコンテナとして自社の環境で動かします。設定ファイルに、取り込むコードの置き場所を書いて起動し、ブラウザで開きます。メモリは少なくとも 4GB が要ると文書は書いています（[Docker Compose での導入](https://docs.sourcebot.dev/docs/deployment/docker-compose)）。公式サイトは、コードが自社の環境の外に出ない点を強調しています（[公式サイト](https://www.sourcebot.dev/)）。

Ask Sourcebot のモデルには、Anthropic・OpenAI・Amazon Bedrock・Google などの提供元のほか、OpenAI 互換の窓口を通して手元で動くモデル（Ollama など）も設定できます（[モデルの提供元の設定](https://docs.sourcebot.dev/docs/configuration/language-model-providers)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: 各リポジトリの置き場所（GitHub など）の上に、組織全体を横断する検索と文脈の層を足すものです。人が使う検索の画面と、エージェントが使う MCP の窓口を同じ索引から出します。

**代わりに失うもの・注意点**

- **自分で運用する**: 自社で動かすので、サーバーの用意・更新・索引の容量の管理は自分たちの仕事になります。2MB を超えるファイルや、トライグラムが2万を超えるファイルは、設定しない限り索引に入れず、バイナリのファイルは索引にできません（[コードの取り込み](https://docs.sourcebot.dev/docs/connections/indexing-your-code)）
- **コードの移動は検索に頼る**: 定義や参照は、名前の検索で見つけます（[コードの移動](https://docs.sourcebot.dev/docs/features/code-navigation)）。筆者の見立てでは、同じ名前の関数が多い場合などは、言語サーバーほど正確に絞り込めないことがあります
- **権限の同期は有料**: GitHub などの置き場所の権限に合わせて、見られるリポジトリを利用者ごとに絞る機能は、有料の版だけにあります。対応する置き場所も限られ、Gitea・Gerrit などは対象外です（[権限の同期](https://docs.sourcebot.dev/docs/features/permission-syncing)）。筆者の見立てでは、この機能を使わない場合、取り込んだリポジトリを誰が見られるかを、取り込む範囲で管理する必要があります
- **外部への通信**: 初期設定では、24時間ごとに開発元へ利用状況の報告（Service Ping）を送ります。中身は、利用者数・リポジトリの数・ホスト名・機器の情報などで、ソースコードや AI の入出力、認証情報、利用者の情報は送らないと文書は書いています（[Service Ping](https://docs.sourcebot.dev/docs/misc/service-ping)）。止め方は、筆者が読んだ範囲の文書には書かれていませんでした。有料の版を標準の方式（Activation Code）で有効化した場合は、この報告で有効化を確かめるため、7日間届かないと機能が制限される状態になります。ネットにつながない環境向けには、問い合わせて得る License Key で有効化する方式もあります（[サブスクリプションの有効化](https://docs.sourcebot.dev/docs/activating-a-subscription)）
- **Ask Sourcebot とモデル**: 筆者の見立てでは、質問に答えるために、関係するコードの一部が設定したモデルの提供元に送られます。社外に出せないコードなら、手元のモデルを使うか、提供元との契約を確かめる必要があります

## 5. 採用を判断するときの観点

- **規模**: リポジトリの数が多く、横断して探す場面が多いほど効き目が大きくなります（筆者の見立て）
- **費用**: 無料の Basic は利用者5人まで、Pro は利用者1人あたり月20ドル（料金のページの表示）、Enterprise は個別の見積もりです（[料金](https://www.sourcebot.dev/pricing)）。料金のページでは、Ask Sourcebot と MCP は無料の版にも含まれています。なお、Radar のページは Ask Sourcebot を有料の機能と書いており、食い違っています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/sourcebot)）。モデルの利用料は別にかかります
- **アクセスの管理**: 置き場所の権限どおりに見せる必要があるか。あるなら、有料の版と、使っている置き場所が権限の同期に対応しているかを確かめます
- **エージェントとの組み合わせ**: 使っているエージェントが MCP でつなげるか
- **始め方**: 筆者の見立てでは、一部のリポジトリだけを取り込んだ小さな構成で、人の検索とエージェントの MCP の両方を試し、探す時間が減るかを確かめてから広げるのが安全です

## 6. 関連技術・代替技術

- **Sourcegraph**: 同じく組織のコードを横断して検索する道具。2022年3月の版の Tools で Trial。Radar の Sourcebot のページが関連の項目に挙げている（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/sourcegraph)）
- **Using GenAI to understand legacy codebases**: 生成 AI で古いコードベースを理解する手法。2025年11月の版の Techniques で Adopt。Radar の Sourcebot のページが関連の項目に挙げている（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/using-genai-to-understand-legacy-codebases)）
- **Software engineering agents**: ソフトウェア開発の作業を進める AI のエージェント。2025年4月の版の Tools で Trial。Radar の Sourcebot のページが関連の項目に挙げている（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/software-engineering-agents)）
- **OpenWiki**（[このリポジトリの解説](openwiki.md)）: 同じ Vol.35 の Tools の Trial。コードから wiki を作ってエージェントの下調べの材料にする道具で、Sourcebot がコードそのものを検索させるのとは手段が違う
- **コードの置き場所の検索**: GitHub などの置き場所が持つコード検索（一般的な説明）。その置き場所の中のリポジトリが対象になる

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: 多数のリポジトリにまたがるコードベースの文脈の層として、MCP を通じてエージェントに組織のコード全体を検索させられる点を評価しています。開発者以外の人がコードについて質問できる点も役立つとしています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/tools/sourcebot

## 8. 深掘りするときの一次情報

- [Sourcebot の公式サイト](https://www.sourcebot.dev/)
- [Sourcebot の文書](https://docs.sourcebot.dev/)
- [Sourcebot の GitHub リポジトリ](https://github.com/sourcebot-dev/sourcebot)
- [MCP サーバー](https://docs.sourcebot.dev/docs/features/mcp-server)
- [料金](https://www.sourcebot.dev/pricing)

## 9. 確認日

- 内容の確認日: 2026-11-15（確認時点の最新のリリースは v6.1.0）
- Radar の評価の確認日: 2026-11-15（リングの推移は、Radar の項目のページで確認）

## 用語集

用語の説明の冒頭の「元の語」は、覚えるための手がかりとして、元の語（英単語の意味・略語の元の綴り・名前の元になった語）を短く示したものです。

| 用語 | 説明 |
|---|---|
| セルフホスト | 元の語: self〈自分〉＋ host〈場を提供する〉。サービスを、提供元のクラウドではなく自分たちのサーバーで動かすこと |
| エージェント | 元の語: agent〈代理人〉。人の指示を受けて、コードを読み書きしたりコマンドを実行したりする AI |
| リファクタリング | 元の語: refactor〈作り直す。factor は「因数・要素」〉。外から見た振る舞いを変えずに、コードの中の構造を整えること |
| トライグラム | 元の語: tri〈3つの〉＋ gram〈書かれたもの〉。文字列を3文字ずつ区切った並び。これで索引を作ると、文字列の一部でも速く探せる |
| 正規表現 | 元の語: regular expression〈規則的な表現〉。文字の並びの型を記号で書き、合う部分を探す書き方 |
| ctags | 元の語: C ＋ tags〈札〉（もとは C 言語のための道具）。ソースコードから関数などの名前と、その場所を抜き出す道具 |
| シンボル | 元の語: symbol〈記号・しるし〉。コードの中の関数・クラス・変数などの名前 |
| 言語サーバー | プログラミング言語ごとに、定義へ移動・参照の検索などをコードの意味を理解して行う裏方のプログラム |
| MCP | 元の語: Model Context Protocol の略。AI のアプリが外部の道具やデータにつなぐための共通の約束事 |
| Docker | 元の語: docker〈港湾の荷役作業員〉。アプリを、動かすのに要るものごと「コンテナ」にまとめて動かす道具 |
| Service Ping | 元の語: service〈サービス〉＋ ping〈短く打つ音。通信の確認の合図〉。Sourcebot が開発元へ定期的に送る利用状況の報告 |
| MIT ライセンス | 元の語: MIT（Massachusetts Institute of Technology の略。マサチューセッツ工科大学）。著作権の表示を残せば自由に使える、短いオープンソースのライセンス |
