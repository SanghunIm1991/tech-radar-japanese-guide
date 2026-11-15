# Linear

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Tools（開発や運用で使う道具） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

**名前の手がかり**: Linear〈直線の・一直線の〉

Linear は、ソフトウェアの開発チームが「やること」（**イシュー**）を登録し、計画を立てて進み具合を追うための、クラウドで使う**イシュートラッカー**です。公式の文書は、小さなバグの修正から会社全体の取り組みまでを、いくつかの基本の概念に沿って整理できる道具だと説明しています（[Linear の概念の説明](https://linear.app/docs/conceptual-model)）。最近は、AI のコーディング**エージェント**をワークスペースの一員として迎え、イシューを任せられることでも注目されています（[Agents in Linear](https://linear.app/docs/agents-in-linear)）。

たとえるなら、チームの作業の掲示板に、人だけでなく AI の作業者の名札も並べ、「この札の仕事はこの作業者に任せた」と書き込めるようにしたものです。任せた仕事の途中経過も、同じ掲示板に貼り出されます。

## 2. 解決する課題（なぜ生まれたか）

開発チームは、バグや機能の追加をイシューとして記録し、期間を区切って計画を立てます。このための道具として、これまでは Jira のような多機能のイシュートラッカーがよく使われてきました。Thoughtworks は、Linear が始めやすさと、ふつうは Jira に求めるような計画の機能を両立している点を挙げています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/linear)）。その背景には、次の課題があります。

- **計画の道具の準備が重い**: 作業の流れ（状態の並び）や、期間を区切った計画の仕組みを、チームごとに一から組み立てると手間がかかります
- **コードとイシューが離れている**: コードの変更（**ブランチ**や**プルリクエスト**）とイシューのつながりを人が手で書き、状態も手で「進行中」「完了」に変えていると、記録が実態に遅れます
- **AI のエージェントの仕事が見えない**: エージェントにコードを書かせても、その仕事はエージェントの画面の中で進み、チームの計画の道具からは誰が何を任されて、どこまで進んだのかが見えません
- **エージェントがイシューの文脈を知らない**: 手元で動くエージェントが、イシューに書かれた要件や議論を読めなければ、人が内容を写して渡すことになります

## 3. 仕組みの概要

### 課題ごとの答え

| 課題 | Linear の答え |
|---|---|
| 計画の道具の準備が重い | **チーム**ごとに、既定の**ワークフロー**（イシューの状態を順に並べたもの）が最初から作られる。期間を区切った計画（**サイクル**）は、長さ（1〜8週）と開始の曜日を決めれば自動で作られ、終わらなかったイシューは、原則として次のサイクルへ自動で持ち越される（[概念の説明](https://linear.app/docs/conceptual-model)、[Cycles](https://linear.app/docs/use-cycles)）。Jira などからの取り込みの道具も用意されている（[Linear の文書の目次](https://linear.app/docs)） |
| コードとイシューが離れている | GitHub と連携し、ブランチの名前にイシューの番号を入れるか、プルリクエストの題名か説明に「Fixes ENG-123」のような決まった言葉（**マジックワード**）を書くと、つながる。既定では、プルリクエストを開くとイシューが「In Progress」に、**マージ**すると「Done」に自動で変わる（[GitHub の連携](https://linear.app/docs/github)） |
| AI のエージェントの仕事が見えない | エージェントを、管理者がワークスペースに入れる「アプリの利用者」として扱う。イシューをエージェントに任せる（**委任**）か、コメントで呼びかけると、作業のまとまり（エージェントのセッション）が作られ、エージェントは考えたこと・行った操作・質問・結果を、Linear の画面（エージェントのセッションの表示）で見せる。任せても、イシューの主な担当者は人のまま残る（[Agents in Linear](https://linear.app/docs/agents-in-linear)、[開発者向けの説明](https://linear.app/developers/agents)、[エージェントとのやり取り](https://linear.app/developers/agent-interaction)） |
| エージェントがイシューの文脈を知らない | **MCP** のサーバーをインターネット上で提供している。Claude Code・Cursor・Codex などのエージェントがこれにつなぐと、イシュー・プロジェクト・コメントを探したり、作ったり、更新したりできる（[Linear の MCP サーバー](https://linear.app/docs/mcp)） |

### 作業の単位の重なり

Linear の作業は、小さい順に次のように重なります（[概念の説明](https://linear.app/docs/conceptual-model)、[Projects](https://linear.app/docs/projects)、[Initiatives](https://linear.app/docs/initiatives)）。

- **イシュー**: 作業の基本の単位。1つのチームに属し、そのチームのワークフローの上を進む
- **サイクル**: チームごとの、繰り返す計画の期間。スプリント（決まった期間で作業を区切るやり方）に近いが、リリースとは結びつけない（[Cycles](https://linear.app/docs/use-cycles)）
- **プロジェクト**: 成果や完了の予定日に向けてイシューをまとめたもの。複数のチームにまたがってよい
- **イニシアチブ**: 会社の目標に沿って、プロジェクトをまとめたもの

チームの外の人や連携の道具から入ってきた仕事は、まず**トリアージ**の受付箱で、受ける・断る・重複としてまとめるなどを決めてから、ワークフローに入ります（[Triage](https://linear.app/docs/triage)）。

### エージェントと外部の道具のつなぎ方

エージェントは、ワークスペースの管理者が入れ、使えるチームを選びます。任されたり呼びかけられたりすると、Linear は **Webhook** でエージェントに知らせます（[Agents in Linear](https://linear.app/docs/agents-in-linear)、[開発者向けの説明](https://linear.app/developers/agents)）。公式の一覧には、Cursor・OpenAI Codex・Devin・Sentry などのエージェントが並んでいます（[Linear のエージェントのページ](https://linear.app/agents)）。ほかの道具とのつなぎ口として、**GraphQL** の **API** と Webhook も公開されています（[API と Webhook](https://linear.app/docs/api-and-webhooks)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: 開発の計画と進み具合を管理する層です。コードの置き場所（GitHub など）とエージェントのあいだに立ち、「誰が・何を・どこまで」を1か所に集めます。

**代わりに失うもの・注意点**

- **クラウドのサービスに預ける**: 筆者が読んだ料金のページと文書の範囲では、自社のサーバーで動かす形の記載は見当たりませんでした（[料金](https://linear.app/pricing)）。データの置き場所の地域や、取得している認証は、この解説では確かめていません。筆者の見立てでは、扱う情報の規則が厳しい組織は、契約の前にこれらを確かめる必要があります
- **無料の版の上限**: 無料の版は、チームが2つまで、イシューが250件までです。非公開のチームや、トリアージを自動で振り分ける機能などは、上位の有料の版にあります（[料金](https://linear.app/pricing)）
- **決まった型に乗る**: 筆者の見立てでは、作業の単位（イシュー・サイクル・プロジェクト・イニシアチブ）の型が決まっているぶん始めやすい一方、Jira のように項目や流れを細かく作り込む運用をそのまま移すのは難しい場合があります
- **エージェントに渡す範囲**: エージェントは、管理者が選んだチームのイシューを読めます。外部のエージェントのデータの扱いは、それぞれの提供元の規約に従います（[Agents in Linear](https://linear.app/docs/agents-in-linear)）。また、Linear 自身の AI（Linear Agent）は、どのワークスペースでも最初から有効で、管理者が設定で止められます（[Linear Agent](https://linear.app/docs/linear-agent)）
- **GitHub の版による差**: 自社で動かす GitHub Enterprise Server では、プルリクエストとのつながりは使えるものの、コミットとのつながりや GitHub のイシューとの同期は使えません（[GitHub の連携](https://linear.app/docs/github)）

## 5. 採用を判断するときの観点

- **規模と費用**: 有料の版のうち Basic と Business は利用者1人ごとの月額（年払いの場合の表示）で、Enterprise は個別の見積もりです（[料金](https://linear.app/pricing)）。エージェントは、料金を数える利用者に入りません（[開発者向けの説明](https://linear.app/developers/agents)）。ただし、エージェントの提供元の料金は別にかかることがあります（[Agents in Linear](https://linear.app/docs/agents-in-linear)）
- **道具を自分たちで選べるか**: Thoughtworks は、道具を自分で選べるチームで、エージェントにバックログ（これからやる作業の一覧）から直接仕事を取らせたい場合に、試す価値があるとしています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/linear)）
- **使っているエージェント**: 任せたいエージェントが公式の一覧にあるか、または MCP でつなげるか
- **今の道具からの移行**: Jira などからの取り込みの道具の有無と、移せない作り込みが無いか
- **管理の要件**: 単一のログイン（SAML）や利用者の自動の登録（SCIM）、操作の記録（監査ログ）は、最上位の Enterprise の版にあります（[料金](https://linear.app/pricing)）
- **始め方**: 筆者の見立てでは、1つのチームで無料の版を使い、GitHub の連携と1つのエージェントへの委任を試してから広げるのが安全です

## 6. 関連技術・代替技術

- **Jira**: 項目や流れを細かく作り込める、広く使われているイシュートラッカー（一般的な説明）。Linear は Jira からの取り込みの道具を用意している（[Linear の文書の目次](https://linear.app/docs)）
- **GitHub Issues**: コードの置き場所に組み込まれたイシューの管理。Linear は、GitHub のイシューと片方向または双方向で同期できる（[GitHub の連携](https://linear.app/docs/github)）
- **Beads**（[このリポジトリの解説](beads.md)）: 同じ Vol.35 の Tools の Trial。Thoughtworks は、Linear より一段下で、エージェントが作る細かい作業を追う道具として位置づけている（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/linear)）
- **OpenAI Codex**（[このリポジトリの解説](openai-codex.md)）・**Cursor**・**Claude Code**: Linear から仕事を受けたり、MCP でイシューを読み書きしたりできるコーディングエージェントとして、Radar のページが名前を挙げている（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/linear)）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: 始めやすさと計画の機能を両立し、エージェントを仕事の担い手として迎えられる点を評価し、エージェントにバックログから直接仕事を取らせたい、道具を自分で選べるチームに試す価値があると見ています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/tools/linear

## 8. 深掘りするときの一次情報

- [Linear の文書（目次）](https://linear.app/docs)
- [Linear の基本の概念](https://linear.app/docs/conceptual-model)
- [Cycles](https://linear.app/docs/use-cycles)
- [GitHub の連携](https://linear.app/docs/github)
- [Agents in Linear](https://linear.app/docs/agents-in-linear)
- [エージェントを作る開発者向けの説明](https://linear.app/developers/agents)
- [Linear の MCP サーバー](https://linear.app/docs/mcp)
- [料金](https://linear.app/pricing)

## 9. 確認日

- 内容の確認日: 2026-11-15
- Radar の評価の確認日: 2026-11-15（リングの推移は、Radar の項目のページで確認）

## 用語集

用語の説明の冒頭の「元の語」は、覚えるための手がかりとして、元の語（英単語の意味・略語の元の綴り・名前の元になった語）を短く示したものです。

| 用語 | 説明 |
|---|---|
| イシュー | 元の語: issue〈問題・論点〉。バグの修正や機能の追加など、やることを1件ずつ記録したもの |
| イシュートラッカー | 元の語: tracker〈追跡するもの〉。イシューを登録し、担当や状態を追う道具 |
| ワークスペース | 元の語: workspace〈作業場〉。Linear で、1つの会社のチーム・イシューなどすべてを入れる器 |
| ワークフロー | 元の語: workflow〈仕事の流れ〉。イシューの状態（未着手・進行中・完了など）を順に並べたもの。チームごとに決める |
| サイクル | 元の語: cycle〈周期・一巡り〉。チームが決まった期間で作業を区切る、繰り返しの計画の期間 |
| バックログ | 元の語: backlog〈たまった仕事〉。これから取りかかる作業の一覧 |
| イニシアチブ | 元の語: initiative〈率先した取り組み〉。会社の目標に沿って、複数のプロジェクトをまとめたもの |
| トリアージ | 元の語: triage〈選別〉。新しく入ってきた仕事を、受けるか・断るかなど振り分けること |
| エージェント | 元の語: agent〈代理人〉。人の指示を受けて、コードを読み書きしたりコマンドを実行したりする AI |
| 委任 | 元の語: delegate〈代理を任せる〉。Linear で、主な担当者を人のまま残して、イシューの作業をエージェントに任せること |
| ブランチ | 元の語: branch〈枝〉。Git で、本流から分けて作業を進めるための履歴の枝 |
| プルリクエスト | 元の語: pull request〈取り込みの依頼〉。自分の変更を共有のリポジトリに取り込んでもらうよう依頼し、レビューを受ける仕組み（GitHub などの機能） |
| マージ | 元の語: merge〈合流させる〉。枝の変更を本流に取り込むこと |
| API | 元の語: Application Programming Interface の略。プログラムからほかのソフトウェアの機能を呼び出すための決まりごと |
| GraphQL | 元の語: Graph〈つながりの図〉＋ QL（Query Language の略。問い合わせの言語）。欲しいデータの形を指定して取り出す API の方式 |
| Webhook | 元の語: web ＋ hook〈引っかける金具〉。出来事が起きたときに、決めておいた宛先へ知らせを送る仕組み |
| MCP | 元の語: Model Context Protocol の略。AI のアプリが外部の道具やデータにつなぐための共通の約束事 |
| SAML / SCIM | 元の語: Security Assertion Markup Language の略／System for Cross-domain Identity Management の略。会社のアカウントでまとめてログインする仕組み／利用者の登録や削除を自動で行う仕組み |
