# LiteLLM（多くの LLM を同じ形で呼ぶための SDK と AI ゲートウェイ）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Platforms（アプリケーションを載せて動かす基盤・サービス） |
| リング | Trial（前の版も同じリング。2026年4月の版でも Trial。象限は Languages & Frameworks から移った） |
| 状態 | 下書き |

## 1. 一言でいうと

LiteLLM は、OpenAI・Anthropic・Google・AWS Bedrock など100以上の LLM の提供元を、**同じ形（OpenAI の API の形）で呼び出せるようにする**オープンソースの道具です。アプリに組み込む Python の SDK と、組織の LLM の利用を1か所に集める**プロキシサーバー**（AI ゲートウェイ）の2つの使い方があります（[LiteLLM のドキュメント](https://docs.litellm.ai/docs/)、[GitHub: BerriAI/litellm](https://github.com/BerriAI/litellm)）。開発元は BerriAI です。`enterprise/` のフォルダを除いて MIT ライセンスで、`enterprise/` は商用のライセンスです（[LICENSE](https://github.com/BerriAI/litellm/blob/main/LICENSE)）。

たとえるなら、国ごとに形の違うコンセントに、1つの変換プラグで差し込めるようにする道具です。さらにプロキシの形では、電気の使用量を部署ごとに計って上限をかける「分電盤」の役も担います（たとえは筆者の見立て）。

## 2. 解決する課題（なぜ生まれたか）

LLM の提供元ごとに、API の呼び方・応答の形・エラーの形が少しずつ違います。提供元を替えたり、複数を使い分けたりするたびにコードを書き直すのは手間です（筆者の見立て）。LiteLLM は、どの提供元でも応答を OpenAI の形にそろえて返します（[LiteLLM のドキュメント](https://docs.litellm.ai/docs/)）。

さらに、組織の中で LLM の利用が広がると、「誰がどのモデルをどれだけ使ったか」「API キーを各チームに配ってよいか」「障害の時に別の提供元へ逃がせるか」といった管理の問題が出てきます。プロキシの形の LiteLLM は、提供元の本物の鍵を1か所に預かり、利用者には代わりの鍵を配って、利用量や予算を管理します（[仮想キー](https://docs.litellm.ai/docs/proxy/virtual_keys)）。

## 3. 仕組みの概要

### 3.1 2つの使い方

| 使い方 | 内容 | 向いている場面 |
|---|---|---|
| Python SDK | `from litellm import completion` のように読み込み、関数の呼び出しで各社の LLM を使う | 1つのアプリの中で提供元を切り替えたいとき |
| プロキシサーバー | 自分たちで動かす、OpenAI 互換の API を持つゲートウェイ。アプリは OpenAI の SDK の接続先をこのサーバーに向けるだけで使える | 組織の LLM の利用をまとめて管理したいとき |

（[LiteLLM のドキュメント](https://docs.litellm.ai/docs/)）

### 3.2 プロキシの主な機能

- **仮想キー**: 利用者やチームに配る代わりの鍵。使えるモデル・有効期限・予算・1分あたりの上限などを鍵ごとに決められ、使った金額を鍵・利用者・チームごとに追えます（[仮想キー](https://docs.litellm.ai/docs/proxy/virtual_keys)）
- **負荷分散と予備への切り替え**: 同じモデルを複数の配置先に置き、空いている所や応答の速い所へ振り分けます。失敗したら再試行し、別のモデルへ切り替える（フォールバック）設定もできます（[負荷分散](https://docs.litellm.ai/docs/proxy/load_balancing)）
- **ガードレール**: 呼び出しの前・後・同時に、入力や出力を検査する仕組み。外部の検査サービスとつなぐほか、組み込みの検査もあります（鍵ごとの切り替えなど一部は有料版。[ガードレール](https://docs.litellm.ai/docs/proxy/guardrails/quick_start)）
- **ログの連携**: Langfuse などの可観測性（動きを外から観察する）の道具に記録を送れます（[LiteLLM のドキュメント](https://docs.litellm.ai/docs/)）
- **MCP とエージェントの中継**: MCP（AI に外部の道具をつなぐ取り決め）のサーバーをまとめて1つの入口から提供し、鍵やチームごとに使える道具を決められます。エージェントどうしの通信の取り決め（A2A）の中継にも対応します（[MCP ゲートウェイ](https://docs.litellm.ai/docs/mcp)、[A2A](https://docs.litellm.ai/docs/a2a)）

### 3.3 配置と必要な部品

Docker のイメージや Kubernetes 向けの Helm のチャートで動かします。鍵・チーム・利用量の記録には PostgreSQL が要り、プロキシを2台以上で動かすときは、状態を共有するために Redis が要ります（[配置](https://docs.litellm.ai/docs/proxy/deploy)）。

### 3.4 無料の範囲と有料版

同じイメージに有料のライセンスキーを入れると、有料版の機能が使えます。有料の機能には、管理画面の SSO（5人までは無料）、監査ログ、組織やチームの管理者のロール、秘密の情報の管理サービスとの連携、鍵の自動の入れ替えなどがあります。価格は公開されていません（[Enterprise](https://docs.litellm.ai/docs/enterprise)）。

### 3.5 Rust への書き換え

プロキシの処理を段階的に Rust へ移す作業が進んでいます。対応済みの要求は Rust で処理し、それ以外は Python の側に回す形です（[Rust への移行](https://docs.litellm.ai/rust-migration)）。Thoughtworks は、Python の SDK と設定は保たれるとしています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/platforms/litellm)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: アプリと LLM の提供元の間に入る**中継の層**です。Thoughtworks は、複数のチームが基盤として使っており、ライブラリというより基盤に近いものとして扱われているため、今回の版で象限を Platforms に移したとしています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/platforms/litellm)）。

**得られるもの**

- **提供元を替えやすい**: 呼び出し方がそろうので、モデルの差し替えや比較がしやすい（[LiteLLM のドキュメント](https://docs.litellm.ai/docs/)）
- **管理を1か所に集められる**: 鍵・予算・利用量・ログ・ガードレールをゲートウェイでまとめて扱える（[仮想キー](https://docs.litellm.ai/docs/proxy/virtual_keys)、[ガードレール](https://docs.litellm.ai/docs/proxy/guardrails/quick_start)）
- **障害に強くできる**: 再試行と予備への切り替えで、1つの提供元の障害の影響を減らせる（[負荷分散](https://docs.litellm.ai/docs/proxy/load_balancing)）

**代わりに失うもの・注意点**

- **セキュリティの更新が多い**: 公式の告知の一覧には、2026年だけでも、認証を迂回されるものや、有料版のプロキシで管理者の鍵が漏れるものなど、重大度が Critical の脆弱性が複数載っています（[セキュリティの告知](https://github.com/BerriAI/litellm/security/advisories)）。Thoughtworks も、今年25件を超える CVE（公開された脆弱性の識別番号）が出たことを、Adopt にしない主な理由に挙げています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/platforms/litellm)）。ゲートウェイは全社の鍵を預かるので、速やかに更新できる体制が欠かせません（筆者の見立て）
- **共通化の限界**: 提供元に特有の機能を使うと、ゲートウェイへの依存と、提供元に結びついたコードの両方を抱えることになります。同じ指示でもモデルによって応答が違うため、呼び方をそろえるだけでは結果はそろいません（[Radar のページ](https://www.thoughtworks.com/en-us/radar/platforms/litellm)の2026年4月の版と2024年の版の説明）
- **黙って失われる設定**: Thoughtworks は、対応しない引数を黙って捨てる設定（`drop_params`）では、機能が失われても気づきにくいと注意しています（同ページの2026年4月の版の説明）
- **中継の遅れ**: ゲートウェイを通る分だけ遅れが加わります。開発元は、模擬の接続先を使った計測で、4台（各4 vCPU）で1秒あたり約1,170件の要求を処理したとき、加わる遅れは95%の要求で8ミリ秒以内（2台・約1,036件では29ミリ秒）と示しています（開発元の主張。[ベンチマーク](https://docs.litellm.ai/docs/benchmarks)）

## 5. 採用を判断するときの観点

- **SDK かプロキシか**: 1つのアプリで提供元を切り替えるだけなら SDK で足ります。複数のチームの利用を管理したいならプロキシを選ぶ（筆者の見立て）
- **可用性**: プロキシを止めると全社の LLM の利用が止まるので、複数台で動かし、PostgreSQL と Redis も含めて冗長にする（[配置](https://docs.litellm.ai/docs/proxy/deploy)。冗長化の要否は筆者の見立て）
- **更新の体制**: 版を固定しつつ、セキュリティの告知を購読して速やかに上げる。公式は告知のメーリングリストを用意しています（[SECURITY.md](https://github.com/BerriAI/litellm/blob/main/SECURITY.md)）
- **費用**: 無料の範囲で鍵・予算・管理画面まで使えます。SSO を多くの人で使う、監査ログが要る、といった場合は有料版の見積もりが要ります（[Enterprise](https://docs.litellm.ai/docs/enterprise)）
- **秘密の情報の扱い**: データベースに保存する提供元の鍵は、`LITELLM_SALT_KEY` で暗号化します。この値は、モデルを登録したあとは変えられません（変えると、保存した鍵を復号できなくなります。[配置](https://docs.litellm.ai/docs/proxy/deploy)）

## 6. 関連技術・代替技術

- **agentgateway**: Thoughtworks が、MCP やエージェント間の通信まで管理したいときに LiteLLM と組み合わせる候補として挙げている、別のゲートウェイ（[Radar のページ](https://www.thoughtworks.com/en-us/radar/platforms/litellm)）
- **クラウドの AI ゲートウェイ**: Cloudflare AI Gateway など、提供元が運用するゲートウェイ。Radar のページの2025年11月の版の説明で、「AI ゲートウェイ」の語のリンク先として示されています（同上）
- **LibreChat**: 社員向けの AI チャットの画面。LiteLLM のプロキシを、その裏の接続先にする構成が考えられます（筆者の見立て。[LibreChat の解説](librechat.md)）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（前の版も同じリング。2026年4月の版でも Trial）。象限は Languages & Frameworks から Platforms に移った
- 過去の版: 2024年4月の版で Assess として初めて載り、2024年10月・2025年11月・2026年4月の版では Trial（Radar のページの推移による。今回より前は Languages & Frameworks の項目）
- Thoughtworks の評価の要旨: 多くの提供元を1つの入口で扱え、経路の制御・鍵・予算・ガードレールをまとめられる AI ゲートウェイとして、試す価値があるとしています。Adopt にしない主な理由は、脆弱性の多さです
- Radar のページ: https://www.thoughtworks.com/en-us/radar/platforms/litellm

## 8. 深掘りするときの一次情報

- [GitHub: BerriAI/litellm](https://github.com/BerriAI/litellm)
- [LiteLLM のドキュメント](https://docs.litellm.ai/docs/)
- [仮想キー](https://docs.litellm.ai/docs/proxy/virtual_keys) と [負荷分散](https://docs.litellm.ai/docs/proxy/load_balancing)
- [配置](https://docs.litellm.ai/docs/proxy/deploy) と [Enterprise](https://docs.litellm.ai/docs/enterprise)
- [SECURITY.md](https://github.com/BerriAI/litellm/blob/main/SECURITY.md) と [セキュリティの告知](https://github.com/BerriAI/litellm/security/advisories)

## 9. 確認日

- 内容の確認日: 2026-11-11
- Radar の評価の確認日: 2026-11-11

## 用語集

| 用語 | 説明 |
|---|---|
| LLM | 大規模言語モデル。大量の文章で学習し、文章を生成する AI モデル |
| SDK | ソフトウェア開発キット。ほかのプログラムからその機能を呼び出すための部品一式 |
| プロキシサーバー | 利用者と相手のサーバーの間に入り、要求を中継するサーバー。中継のついでに記録や制限をかけられる |
| AI ゲートウェイ | アプリと LLM の提供元の間に置く中継の基盤。鍵の管理・利用量の記録・振り分け・検査をまとめて行う |
| 仮想キー | 本物の API キーの代わりに利用者へ配る鍵。ゲートウェイが本物の鍵に置き換えて提供元を呼ぶので、本物の鍵を配らずに済む |
| フォールバック | 使おうとしたものが失敗したとき、あらかじめ決めた別のものに切り替えること |
| ガードレール | AI への入力や AI の出力を検査し、まずい内容を止める仕組み |
| A2A | Agent-to-Agent。AI のエージェントどうしが通信するための取り決め |
| CVE | Common Vulnerabilities and Exposures。公開された脆弱性に付ける共通の識別番号 |
| SSO | シングルサインオン。社内の認証基盤に1回ログインすれば、ほかのサービスにもログインできる仕組み |
| 冗長化 | 1台が止まっても全体が止まらないよう、同じ役割のものを複数用意すること |
