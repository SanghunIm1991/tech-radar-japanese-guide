# Figma Code Connect（デザインの部品を、本物のコードの部品に結びつける）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Tools（開発や運用に使う道具） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

Figma Code Connect は、デザインの道具の Figma で作った**画面の部品（コンポーネント）を、実際のアプリで使っているコードの部品に結びつける**道具です。結びつけておくと、デザインを見た開発者や AI のツールに、自動で作った仮のコードではなく、チームの部品集（デザインシステム）の本物のコードの例が示されます。ライセンスは MIT です（[GitHub: figma/code-connect](https://github.com/figma/code-connect)）。

たとえるなら、家具のカタログの写真ごとに、実際に倉庫にある商品の品番を付けておくようなものです。写真を見た人が似た家具を一から作るのではなく、品番の商品をそのまま使えます（筆者の見立て）。

## 2. 解決する課題（なぜ生まれたか）

デザインから画面のコードを作る道具は多くありますが、汎用の道具は、チームがすでに持っている部品を知りません。そのため、ボタン1つでも、部品集にある本物のボタンを使わずに、見た目だけをまねた生の HTML などを出してしまいます（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/figma-code-connect)）。その結果、書き直しが必要になりがちです（筆者の見立て）。

AI のコーディングエージェントにデザインから画面を作らせる場合も同じで、どの部品を、どの引数（props）で使うべきかを知らなければ、チームの決まりから外れたコードを書きます（筆者の見立て）。Code Connect は、デザインの部品とコードの部品の対応を明示することで、この「知らない」を減らします。

**例**（筆者の例）: Figma に「主ボタン・大・無効」の状態のボタンがあるとします。対応づけが無ければ、AI は「青い四角に白い文字」をまねたコードを一から書くかもしれません。対応づけがあれば、「部品集の Button 部品を、種類＝主、大きさ＝大、無効＝はい、で使う」というコードの例が示され、AI も開発者もそれをそのまま使えます。色や角の丸みは部品の中で決まっているので、画面ごとにずれることもありません。

## 3. 仕組みの概要

**対応づけ**（[GitHub: figma/code-connect](https://github.com/figma/code-connect)。この項の出典は同じ）

- コードの側の部品と、Figma の側の部品を対応づけます
- コードの部品の引数（props）を、Figma の部品の設定項目（プロパティ）に対応づけられるので、Figma で選んだ状態（色の種類やサイズなど）に合わせた正しいコードの例を出せます
- 対応づけると、Figma の開発者向けの表示（Dev Mode）に、自動生成のコードの例ではなく、部品集の本物のコードの例が表示されます

**対応する技術**（[GitHub: figma/code-connect](https://github.com/figma/code-connect)）

React（React Native を含む）、Storybook、HTML（Web Components・Angular・Vue を含む）、SwiftUI（iOS）、Jetpack Compose（Android）に対応しています。README の冒頭の注意書きによると、フレームワークごとの読み取りの仕組みは今後更新されず、更新が続くのは、フレームワークに依らない「テンプレートのファイル」で対応を書く方法だけになりました。

**利用の条件**（[GitHub: figma/code-connect](https://github.com/figma/code-connect)）

Figma の Organization と Enterprise のプランで使え、Design または Dev Mode のフルの利用枠（Full シート）が要ります。

**AI のツールとの組み合わせ**（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/figma-code-connect)）

Radar によると、Figma の MCP サーバー（Figma の情報を AI のツールに渡すための部品）やコーディングエージェントと組み合わせると、実際のコードベースの文脈をエージェントに渡せ、設計がずれていく危険や、コードレビューの往復を減らせます。

**運用の例**（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/figma-code-connect)）

Thoughtworks のチームは、モバイルのデザインシステムで、次のように使ったとしています。

1. iOS と Android の部品を、Figma の部品に対応づける
2. 本流のブランチ（main）にリリースされたあと、対応の情報を Figma に自動で公開する
3. CI（変更のたびに検査を自動で走らせる仕組み）で、デザインとコードのずれ（ドリフト）を見つける

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: デザインとコードの「対応表」です。デザインシステムを、デザインの側とコードの側の二重の管理から、1本につながったものにする部品だといえます（筆者の見立て）。プロンプトだけに頼ったコードの生成から、より制御の効いたエージェントの作業の流れへ移る動きとも相性がよい、と Radar は見ています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/figma-code-connect)）。

**得られるもの**

- **本物の部品を使ったコード**: 開発者も AI も、部品集の部品と引数を使ったコードを出せる（[README](https://github.com/figma/code-connect)、[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/figma-code-connect)）
- **レビューの手間の削減**: レビューの往復が減り、設計がずれていく危険も下がる（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/figma-code-connect)）
- **ずれの早期発見**: CI に組み込めば、デザインとコードのずれを早く見つけられる（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/figma-code-connect)のチームの例）

**代わりに失うもの・注意点**（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/figma-code-connect)）

- **部品集の質しだい**: 結果は、デザインシステムそのものの質に左右される。色の値をコードに直接書くこと（ハードコード）を禁じる検査（lint）や、プルリクエストごとの画面の写し（スナップショット）との比較のような歯止めが要る
- **対応の保守**: 部品の種類（バリアント）ごとの対応を保つのに、継続的な手間がかかる
- **部品の単位まで**: 今は部品ごとの対応で、画面全体が正しいかは確かめられない。配置・状態・画面の移り変わりは、人が確かめる必要がある
- **プランの制約**: 使えるのは上位のプランに限られる（[README](https://github.com/figma/code-connect)）
- **特定の道具への依存**: 対応の情報は Figma の仕組みの上に載るので、デザインの道具を乗り換えると作り直しになる（筆者の見立て）

## 5. 採用を判断するときの観点

- **デザインシステムの成熟度**: Figma とコードの両方に、部品集がそろっているか。部品集が無いうちは効果が小さい（筆者の見立て）
- **使っているフレームワーク**: 対応の一覧に入っているか。新しく始めるなら、更新が続くテンプレートのファイルで書く（[README](https://github.com/figma/code-connect)）
- **Figma のプラン**: Organization 以上のプランと、必要な利用枠があるか（[README](https://github.com/figma/code-connect)）
- **自動化**: 対応の情報の公開と、ずれの検出を CI に組み込めるか（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/figma-code-connect)のチームの例に基づく筆者の見立て）
- **人の確認の残し方**: 画面全体の配置や状態の確認を、誰がどの段階で行うかを決めておく（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/figma-code-connect)の注意に基づく筆者の見立て）
- **小さく始める**: まずよく使う部品（ボタン・入力欄・カードなど）だけを対応づけ、AI の出すコードの直しがどれだけ減ったかを測りながら広げる（筆者の見立て）
- **AI との組み合わせ**: AI のツールに Figma の情報を渡す仕組み（MCP サーバーなど）を使うなら、その権限と、社外にデザインのデータが出ないかを確かめる（筆者の見立て）

## 6. 関連技術・代替技術

- [Radar の Figma Code Connect のページ](https://www.thoughtworks.com/en-us/radar/tools/figma-code-connect)には、関連の項目の欄はありませんでした
- **Anchoring coding agents to a reference application**: 手本のアプリにエージェントを結びつけて、決まりに沿ったコードを書かせる手法。決まりをエージェントに伝えるという目的が近い（[解説](../techniques/anchoring-coding-agents-to-a-reference-application.md)。筆者の見立て）
- **代替: 部品の使い方の文書**: Storybook などの部品のカタログと、使い方の文書をエージェントに読ませる方法もある（筆者の見立て）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: AI のツールや開発者が本物の部品を使ったコードを出せる点を評価し、モバイルのデザインシステムでの自社の経験を紹介しています。一方で、結果がデザインシステムの質に左右され、画面全体の確認は人に残ることに注意を促しています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/tools/figma-code-connect

## 8. 深掘りするときの一次情報

- [GitHub: figma/code-connect](https://github.com/figma/code-connect)

## 9. 確認日

- 内容の確認日: 2026-11-13
- Radar の評価の確認日: 2026-11-13

## 用語集

| 用語 | 説明 |
|---|---|
| Figma | 画面のデザインを作り、チームで共有する道具 |
| コンポーネント | ボタンや入力欄のように、画面をくり返し組み立てるための部品 |
| デザインシステム | 部品・色・文字の大きさなど、画面作りの決まりと部品をまとめたもの |
| props | コードの部品に渡す設定の値（大きさ、色の種類など） |
| バリアント | 1つの部品の、種類や状態の違い（大・小、押せる・押せないなど） |
| Dev Mode | Figma の、開発者向けの表示。寸法やコードの例を見られる |
| MCP | AI のツールに外部の情報や道具をつなぐための共通の仕組み（Model Context Protocol） |
| CI | 変更のたびに、ビルドや検査を自動で走らせる仕組み |
| ドリフト | 決めた状態から、少しずつずれていくこと。ここではデザインとコードのずれ |
| lint | コードを機械的に調べ、決まりへの違反を見つけること |
| スナップショット | ある時点の画面などの写し。変更の前後を比べるのに使う |
| SwiftUI / Jetpack Compose | それぞれ iOS と Android で画面を作るための仕組み |
| シート（利用枠） | Figma で、利用者ごとに割り当てる有料の利用の権利。種類によって使える機能が違う |
| プルリクエスト | コードの変更を取り込む前に、ほかの人に確かめてもらうための依頼の仕組み |
| React | Web の画面を、部品を組み合わせて作るための JavaScript のライブラリ |
| Storybook | 画面の部品を1つずつ表示して、見た目や状態を確かめるためのカタログの道具 |
