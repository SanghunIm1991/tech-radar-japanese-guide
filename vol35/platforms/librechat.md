# LibreChat（自分たちで運用する AI チャットの基盤）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Platforms（アプリケーションを載せて動かす基盤・サービス） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

LibreChat は、ChatGPT のような AI チャットの画面を、**自分たちのサーバーで動かせる**オープンソースのソフトウェアです。OpenAI・Anthropic・Google などの複数の AI の提供元のモデルを、1つの画面から使えます（[GitHub: LibreChat](https://github.com/danny-avila/LibreChat)、[LibreChat のドキュメント](https://www.librechat.ai/docs)）。ライセンスは MIT です（[GitHub](https://github.com/danny-avila/LibreChat)）。

たとえるなら、各社の AI の「窓口」を社内に1つ用意し、誰がどの AI と道具を使えるかを会社が管理できるようにする仕組みです（たとえは筆者の見立て）。

## 2. 解決する課題（なぜ生まれたか）

AI のチャットが仕事で広く使われるようになると、社員が思い思いに外部のサービスを契約し、社内のデータを貼り付ける、ということが起きます。どのデータがどこへ送られたか分からず、費用もばらばらに発生します。Thoughtworks は今回の版で、AI によって加速する管理の外の IT（シャドー IT）を Caution の項目として取り上げており、LibreChat のページはその対策としてこの基盤を位置づけています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/platforms/librechat)）。

一方、AI を禁止すれば済むわけでもありません。そこで、「会社が認めたモデルと道具を、会社の認証と権限の下で、使いやすい画面から使わせる」という考え方が出てきます。LibreChat は、その画面と管理の仕組みをオープンソースで提供します（筆者の見立て）。

## 3. 仕組みの概要

### 3.1 つなげるモデル

OpenAI・Azure OpenAI・Anthropic・Google・AWS Bedrock などの提供元に加え、Ollama（手元で LLM を動かす道具）や OpenRouter などにもつなげます（[GitHub](https://github.com/danny-avila/LibreChat)）。OpenAI 互換の API（OpenAI の API と同じ形の呼び出し方）を持つものは、**カスタムエンドポイント**として設定で足せます（[カスタムエンドポイント](https://www.librechat.ai/docs/quick_start/custom_endpoints)）。

### 3.2 主な機能

公式の機能の一覧には、次のようなものが挙がっています（[機能の一覧](https://www.librechat.ai/docs/features)）。

| 機能 | 内容 |
|---|---|
| エージェント | 指示・ファイル・道具を持たせた、使い回せるアシスタントを作り、共有できる |
| MCP | AI に外部の道具やデータをつなぐ共通の取り決め。エージェントやチャットから使える |
| ファイルとの対話 | 上げた文書を検索できる形にして、内容に基づいて答えさせる（次の 3.3） |
| Artifacts | AI が作った HTML や図などを、画面の横に表示する |
| Web 検索 | 検索して、ページを読み、出典を示して答える |
| 会話の分岐 | 途中のメッセージから会話を枝分かれさせる |

### 3.3 ファイルを使った回答（RAG）

**RAG**（検索で取り出した文書を AI に渡して答えさせる方法）のために、別の部品の RAG API を使います。RAG API は Python で書かれ、文書の検索には PostgreSQL と pgvector（ベクトル検索の拡張）を使います。文書を数値のベクトルに変える**埋め込み**には、既定で OpenAI を使い、Hugging Face や Ollama も選べます（[RAG API](https://www.librechat.ai/docs/configuration/rag_api)）。

### 3.4 認証と利用量の管理

- **ログインの方法**: メールとパスワードのほか、OAuth/OIDC（Entra ID や Keycloak などの社内の認証基盤とつなぐ方式）、SAML、LDAP に対応します。メールとパスワード、または LDAP でログインする利用者は、2段階の認証も使えます（SSO の場合は認証基盤の側で行います。[認証](https://www.librechat.ai/docs/features/authentication)）
- **利用量**: 利用者ごとに「トークンの残高」を持たせ、使った分だけ減らす仕組みを設定できます（[トークンの使用量](https://www.librechat.ai/docs/configuration/token_usage)）
- **管理画面**: 本体とは別に動かす管理画面（v0.8.5 以降）で、利用者・グループ・ロール（役割ごとの権限）・設定を管理できます（[管理画面](https://www.librechat.ai/docs/features/admin_panel)）

### 3.5 設定と配置

設定は2つのファイルに分かれます。`.env` には API キー・データベースの接続先・認証の秘密の値などを置き、`librechat.yaml` にはつなぐモデルや MCP のサーバー・画面の設定などを書きます（[設定](https://www.librechat.ai/docs/configuration)）。

動かし方は、Docker Compose（複数のコンテナをまとめて起動する道具）が推奨です。本体のほか、MongoDB（会話などを保存するデータベース）・Meilisearch（検索の部品）・RAG API・ベクトルのデータベースを一緒に起動します。Kubernetes 向けの Helm のチャートもあります（[ローカルへの導入](https://www.librechat.ai/docs/local)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: 社内の利用者と、外部・社内の AI モデルや道具の間に立つ**利用者向けの入口**です。モデルそのものを動かす基盤（[KServe](kserve.md) など）とは役割が違い、画面・認証・権限・道具の管理を担います（筆者の見立て）。

**得られるもの**

- **提供元を選べる**: 複数の提供元や社内のモデルを1つの画面で切り替えられ、特定の提供元に縛られにくい（[GitHub](https://github.com/danny-avila/LibreChat)。縛られにくさは筆者の見立て）
- **管理された形で配れる**: 会社の認証基盤でログインさせ、ロールで権限を分け、利用量を残高で抑えられる（[認証](https://www.librechat.ai/docs/features/authentication)、[トークンの使用量](https://www.librechat.ai/docs/configuration/token_usage)）
- **エンジニア以外も道具を作れる**: Thoughtworks は、管理者が使える MCP のサーバーを決め、権限のある人がコードを書かずにエージェントを作って共有できる点を評価しています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/platforms/librechat)）

**代わりに失うもの・注意点**

- **運用の責任を負う**: データベースや検索などの部品を含めて、版上げ・バックアップ・監視を自分たちで行います（[ローカルへの導入](https://www.librechat.ai/docs/local)。運用の負担は筆者の見立て）
- **安全の設定に気を配る**: コードを実行させる機能（Code Interpreter）には、管理者が別に立てるサンドボックスの部品が要ります。仮想マシンや手元の PC を実行環境としてつなぐ場合、公式は、それを未知のコードを動かす機械と同じように隔離するよう求めています（[Code Interpreter](https://www.librechat.ai/docs/features/code_interpreter)）。画面を守る HTTP のヘッダーの一部（CSP）は既定で無効です（[セキュリティのヘッダー](https://www.librechat.ai/docs/configuration/security_headers)）
- **最初の管理者**: 最初に作られたアカウントが管理者になるため、導入の直後に自分で作るよう公式が勧めています（[認証](https://www.librechat.ai/docs/features/authentication)）
- **版上げで互換性が変わることがある**: 更新の前に、互換性を壊す変更を変更履歴で確かめるよう公式が勧めています（[GitHub](https://github.com/danny-avila/LibreChat)）

## 5. 採用を判断するときの観点

- **目的**: 社員に AI を安全に配ることが目的なら候補になる。開発者がコードの中から AI を呼ぶだけなら、画面の基盤は要らない（筆者の見立て）
- **認証基盤との接続**: 社内の認証（Entra ID など）とつなげるかを先に確かめる（[認証](https://www.librechat.ai/docs/features/authentication)）
- **道具の管理**: どの MCP のサーバーを誰に使わせるか、エージェントの共有をどこまで許すかの決まりを作る（筆者の見立て）
- **費用**: LibreChat 自体は無料だが、モデルの利用料と、サーバー・運用の人手がかかる（筆者の見立て）
- **運営体制**: 公式のドキュメントは、LibreChat が ClickHouse（データベースの会社）に加わったと告知しています（[LibreChat のドキュメント](https://www.librechat.ai/docs)）。Thoughtworks は、2025年11月の買収のあとも MIT ライセンスが保たれているとしています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/platforms/librechat)）。今後の方針は見守る必要がある（筆者の見立て）

## 6. 関連技術・代替技術

- **Open WebUI**: 同じく自分たちで動かす AI チャットの画面。Radar のページで、似た機能を持つものとして触れられています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/platforms/librechat)）
- **MCP**: LibreChat のエージェントが外部の道具を使うための共通の取り決め（[機能の一覧](https://www.librechat.ai/docs/features)）
- **KServe**: 社内でモデルを動かす基盤。LibreChat から OpenAI 互換の API でつなぐ構成が考えられます（筆者の見立て。[KServe の解説](kserve.md)）
- **商用の AI チャットの企業向けプラン**: 運用を提供元に任せる選択肢（筆者の見立て）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: 顧客の社内向け AI 基盤を作った経験をもとに、エンジニア以外の人にも、管理された形で確かめ済みの AI の道具を使わせる手段として、試す価値があるとしています。主な理由は、認証・権限・RAG に対応し、管理者が選んだ MCP のサーバーでエージェントを作れることです
- Radar のページ: https://www.thoughtworks.com/en-us/radar/platforms/librechat

## 8. 深掘りするときの一次情報

- [GitHub: LibreChat](https://github.com/danny-avila/LibreChat)
- [LibreChat のドキュメント](https://www.librechat.ai/docs) と [機能の一覧](https://www.librechat.ai/docs/features)
- [設定](https://www.librechat.ai/docs/configuration) と [ローカルへの導入](https://www.librechat.ai/docs/local)
- [認証](https://www.librechat.ai/docs/features/authentication)
- [RAG API](https://www.librechat.ai/docs/configuration/rag_api)

## 9. 確認日

- 内容の確認日: 2026-11-11
- Radar の評価の確認日: 2026-11-11

## 用語集

| 用語 | 説明 |
|---|---|
| シャドー IT | 会社の IT 部門が把握・管理していないところで、社員や部署が独自に使い始めたサービスや道具 |
| エージェント | 指示と道具を持たせ、複数の手順を自分で選んで仕事を進める AI のアシスタント |
| MCP | Model Context Protocol。AI のアプリに外部の道具やデータをつなぐための共通の取り決め |
| RAG | Retrieval-Augmented Generation。質問に関係する文書を検索で取り出し、それを AI に渡して答えさせる方法 |
| 埋め込み | 文章の意味を数値の並び（ベクトル）に変えること。意味の近い文章を探すのに使う |
| ベクトル検索 | 埋め込みのベクトルどうしの近さで、意味の近いデータを探す検索 |
| OAuth/OIDC | ほかのサービスのアカウントや社内の認証基盤を使ってログインさせるための、標準の仕組み |
| SAML・LDAP | 企業でよく使われる、ログインや利用者の情報を社内の認証基盤とつなぐための仕組み |
| ロール | 「管理者」「一般利用者」のような役割。役割ごとに使える機能を決める |
| Docker Compose | 複数のコンテナの組み合わせを1つの設定ファイルに書き、まとめて起動する道具 |
| CSP | Content Security Policy。Web の画面で読み込んでよいスクリプトなどを制限し、攻撃を防ぐための HTTP のヘッダー |
