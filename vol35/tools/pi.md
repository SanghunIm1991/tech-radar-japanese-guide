# Pi（最小限で、自分の流れに合わせて組み立てるエージェントの土台）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Tools（開発や運用で使う道具） |
| リング | Trial（Assess から移動） |
| 状態 | 下書き |

## 1. 一言でいうと

**名前の手がかり**: Pi〈英語でギリシャ文字 π の名前。名前の由来ではない〉

Pi は、AI の**エージェント**を動かすための、小さく作られた土台（**ハーネス**）です。公式サイトは、Pi を最小限のエージェントのハーネスと呼び、利用者が Pi に合わせるのではなく、Pi を利用者の流れに合わせることを目指すと説明しています（[公式サイト](https://pi.dev/)）。複数の部品からなり、多くのモデルの提供元を同じ書き方で呼ぶ API（pi-ai）、エージェントの実行の仕組み（pi-agent-core）、それらの上に作った端末で動くコーディングエージェント（pi-coding-agent）などがあります。TypeScript で書かれ、ライセンスは MIT です（[GitHub のリポジトリ](https://github.com/earendil-works/pi)）。

たとえるなら、完成品の家具ではなく、丈夫な骨組みと工具箱です。棚や引き出しは最初から付いていませんが、自分の部屋に合わせて足していけます。骨組みだけを買って、自分の製品の中に組み込むこともできます。

## 2. 解決する課題（なぜ生まれたか）

Thoughtworks は、Pi がほかの道具の土台として広く使われるようになった点を、Trial に上げた理由に挙げています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/pi)）。背景には、次の課題があります。

- **完成品のエージェントは自分の流れに合わせにくい**: 機能が最初から決まっていて、指示（システムプロンプト）や流れを細かく変えにくいことがあります（筆者の見立て）
- **エージェントをほかの仕組みに組み込みたい**: 自分の製品や自動化の中で、エージェントの部分だけを使いたい場面があります（筆者の見立て）
- **特定のモデルの提供元に縛られる**: 提供元ごとに API の書き方が違い、乗り換えや使い分けに手間がかかります（筆者の見立て）
- **長いやり取りの扱い**: 会話が長くなると、モデルが一度に扱える量（**コンテキスト**）を超え、途中からやり直したくなったときにも元の流れが失われがちです（筆者の見立て）

## 3. 仕組みの概要

### 課題ごとの答え

| 課題 | Pi の答え |
|---|---|
| 完成品のエージェントは自分の流れに合わせにくい | 中心を小さく保ち、短いシステムプロンプトで動かす。足りない機能は、TypeScript で書く**拡張**、必要なときに読み込む**スキル**、`/名前` で呼ぶプロンプトのひな形、見た目のテーマで足し、それらをまとめたパッケージを npm や git から入れられる（[公式サイト](https://pi.dev/)） |
| エージェントをほかの仕組みに組み込みたい | 4つの使い方を持つ: 端末の画面で使う対話のモード、1回だけ実行して結果を出すモード（JSON の出力も可）、標準入出力で JSON をやり取りする **RPC** のモード、自分のアプリに埋め込む **SDK**（[公式サイト](https://pi.dev/)）。Thoughtworks も、SDK と RPC のモードでほかのエージェントの仕組みに組み込める点を挙げている（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/pi)） |
| 特定のモデルの提供元に縛られる | pi-ai が、OpenAI・Anthropic・Google・Amazon Bedrock など多くの提供元を同じ書き方で呼べるようにする。OpenAI 互換の窓口を使えば、Ollama などの手元で動くモデルも使える。トークンの数と費用を記録し、会話の途中で別のモデルに引き継げる（[pi-ai の README](https://github.com/earendil-works/pi/blob/main/packages/ai/README.md)） |
| 長いやり取りの扱い | 会話（**セッション**）を木の形で保存し、前の地点に戻っても、離れた枝は消さずに残す。コンテキストの上限が近づくと、要約を足して最近のやり取りを残す**コンパクション**を自動で行うが、元の記録は消さない（[セッションの説明](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/sessions.md)） |

### あえて入れていない機能

公式サイトの「作らなかったもの」の欄は、ほかのコーディングエージェントにある機能をあえて入れず、代わりのやり方を示しています（[公式サイト](https://pi.dev/)）。

- **サブエージェント**（エージェントが別のエージェントを呼ぶ機能）: 端末を分ける道具（tmux）で Pi をいくつも起動するか、拡張で作る
- **計画のモード**: 計画をファイルに書くか、拡張やパッケージで足す
- **操作ごとの許可の確認**: コンテナの中で動かすか、拡張で確認の流れを作る
- **やることの一覧**: `TODO.md` のファイルを使うか、拡張で作る

Thoughtworks は、Pi が MCP・サブエージェント・計画のモード・許可の確認を意図して省いていると書いています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/pi)）。ただし MCP については、確認時点の公式サイトが「MCP を組み込んだ」と書いており、Radar の記述より新しい状態になっています（[公式サイト](https://pi.dev/)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: そのまま使う完成品ではなく、エージェントを組み立てるための部品と土台です。Thoughtworks は、セキュリティの検査の道具や、複数のエージェントを取りまとめる仕組みなどが Pi を土台にしており、利用者が気づかない依存先になっていることも多いと書いています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/pi)）。

**代わりに失うもの・注意点**

- **許可の仕組みが無い**: リポジトリの README は、ファイル・プロセス・ネットワーク・認証情報への操作を制限する許可の仕組みを持たず、起動した利用者の権限のまま動くと書いています。対策として、コンテナや小さな仮想マシンの中で動かす方法を文書で紹介しています（[リポジトリの README](https://github.com/earendil-works/pi)、[コンテナで動かす説明](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/containerization.md)）
- **そのまま使うエージェントとしては弱い**: Thoughtworks は、直接使うコーディングエージェントとしては良い選択ではないとし、企業のコーディングには OpenCode のほうが向くとしています。2026年4月の版（Assess）では、技術者向けの部品で、守りや支援の整った企業向けの基盤ではないとしていました（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/pi)）
- **自分で足す手間**: 省いた機能が要るなら、自分で拡張を書くか、パッケージを探して入れることになります。筆者の見立てでは、他人の拡張やパッケージを入れるときは、エージェントと同じ権限で動くことを踏まえて中身を確かめるべきです
- **持ち主と今後**: 作者の Mario Zechner は Earendil という会社に加わり、Pi はその会社の持ち物になりました。作者の記事は、中心の部分は MIT のままにすると約束し、付加価値の機能や企業向けの部分は別の条件で出す計画を示しています（記事の時点では未着手）（[作者の記事](https://mariozechner.at/posts/2026-04-08-ive-sold-out/)）
- **派生の版**: Thoughtworks は、人気のある派生の版（Oh My Pi）が、言語サーバーとのつなぎ込みやサブエージェント・MCP を足すなど大きく分かれてきたため、別に評価すべきだとしています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/pi)）
- **セッションの記録の扱い**: セッションのファイルには、プロンプトや道具の引数、ファイルの中身が入りうるので、共有する前に確かめるよう文書は注意しています（[セッションの説明](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/sessions.md)）

## 5. 採用を判断するときの観点

- **用途**: 自分たちの製品や仕組みにエージェントを組み込む土台が欲しいのか、すぐ使えるコーディングエージェントが欲しいのか。後者なら、Thoughtworks は OpenCode を勧めています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/pi)）
- **安全の確保**: 許可の仕組みが無いので、コンテナなどの閉じた環境で動かす運用を用意できるか
- **作り込む人手**: 拡張を書いて保守できる技術者がいるか。Thoughtworks も、2026年4月の版で、Pi を技術者向けの部品としていました（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/pi)）
- **入れ方**: 公式サイトの手順には、Web からスクリプトを取って実行する形と、npm で入れる形があり、Node.js 22.19 以上が要ります（[coding-agent の README](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/README.md)）
- **始め方**: 筆者の見立てでは、コンテナの中で pi-coding-agent を動かして小さな作業を任せ、必要な拡張を1つずつ足していくのが安全です。組み込みが目的なら、SDK で小さな試作を作ってから判断します

## 6. 関連技術・代替技術

- **OpenCode**（[このリポジトリの解説](opencode.md)）: 同じ Vol.35 の Tools の Adopt。Thoughtworks が企業のコーディングで Pi より勧めているコーディングエージェント（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/pi)）
- **ADK・LangGraph・Mastra**: エージェントを組み立てるための枠組み。Radar の Pi のページは、2026年4月の版の説明で、これらの枠組みで一から組み立てるより Pi を手直しするほうが楽だとしていた（[ADK](https://www.thoughtworks.com/en-us/radar/languages-and-frameworks/agent-development-kit-adk)、[LangGraph](https://www.thoughtworks.com/en-us/radar/languages-and-frameworks/langgraph)、[Mastra](https://www.thoughtworks.com/en-us/radar/languages-and-frameworks/mastra)）
- **Oh My Pi**: Pi から派生した版。Thoughtworks は別に評価すべきだとしている（[Oh My Pi](https://github.com/can1357/oh-my-pi)、[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/pi)）
- **OpenClaw**: Pi を土台にした道具の例として、公式サイトと作者の記事が挙げている（[公式サイト](https://pi.dev/)、[作者の記事](https://mariozechner.at/posts/2026-04-08-ive-sold-out/)）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（Assess から移動）
- 過去の版: 2026年4月の版で Assess（完成品のエージェントより作り替えやすい、技術者向けの部品とした）
- Thoughtworks の評価の要旨: ほかの道具の気づかれにくい土台として採用が広がり、SDK と RPC で組み込める点を評価しています。一方で、そのまま使うコーディングエージェントとしては最良ではなく、企業のコーディングには OpenCode のほうが向くとしています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/tools/pi

## 8. 深掘りするときの一次情報

- [Pi の公式サイト](https://pi.dev/)
- [Pi の GitHub リポジトリ](https://github.com/earendil-works/pi)
- [pi-coding-agent の README](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/README.md)
- [pi-ai の README](https://github.com/earendil-works/pi/blob/main/packages/ai/README.md)
- [セッションの説明](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/sessions.md)
- [作者の記事（Earendil に加わったこと）](https://mariozechner.at/posts/2026-04-08-ive-sold-out/)

## 9. 確認日

- 内容の確認日: 2026-11-15（確認時点の最新のリリースは v1.5.2）
- Radar の評価の確認日: 2026-11-15（リングの推移は、Radar の項目のページで確認）

## 用語集

用語の説明の冒頭の「元の語」は、覚えるための手がかりとして、元の語（英単語の意味・略語の元の綴り・名前の元になった語）を短く示したものです。

| 用語 | 説明 |
|---|---|
| エージェント | 元の語: agent〈代理人〉。人の指示を受けて、コードを読み書きしたりコマンドを実行したりする AI |
| ハーネス | 元の語: harness〈馬具。力を引き出して制御する道具〉。AI のモデルを、道具の呼び出しや会話の管理と組み合わせて、エージェントとして動かす仕組み |
| API | 元の語: Application Programming Interface の略。プログラムからほかのソフトウェアの機能を呼び出すための決まりごと |
| 拡張 | 元の語: extension〈広げること〉。あとから機能を足すための追加の部品 |
| スキル | 元の語: skill〈技能〉。エージェントに特定の仕事の進め方を教える手順書。必要なときに読み込む |
| RPC | 元の語: Remote Procedure Call の略（離れた手続きの呼び出し）。別のプログラムに処理を頼み、結果を受け取るやり取りの方式 |
| SDK | 元の語: Software Development Kit の略。ソフトウェアを作るための部品や道具一式 |
| セッション | 元の語: session〈ひと続きの会合〉。エージェントとの、ひと続きのやり取り |
| コンテキスト | 元の語: context〈文脈〉。モデルが一度に読んで考えに使える、会話や資料の範囲 |
| コンパクション | 元の語: compaction〈圧縮〉。長くなった会話を要約して、コンテキストに収まるようにすること |
| サブエージェント | 元の語: sub〈下の・副の〉＋ agent。エージェントが、別の仕事を任せるために呼び出す、もう1つのエージェント |
| MCP | 元の語: Model Context Protocol の略。AI のアプリが外部の道具やデータにつなぐための共通の約束事 |
| トークン | 元の語: token〈しるし・引換券〉。モデルが文章を区切って処理する単位で、単語や単語の一部ほどの大きさ。利用料金の基準になる |
| TypeScript | JavaScript に型の仕組みを足したプログラミング言語 |
| MIT ライセンス | 元の語: MIT（Massachusetts Institute of Technology の略。マサチューセッツ工科大学）。著作権の表示を残せば自由に使える、短いオープンソースのライセンス |
