# OpenSpec（AI のコーディングエージェントのための仕様駆動開発の道具）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Tools（開発や運用で使う道具） |
| リング | Trial（Assess から移動） |
| 状態 | 下書き |

## 1. 一言でいうと

**名前の手がかり**: Open〈開かれた〉＋ Spec〈specification（仕様）の略〉

OpenSpec は、AI のコーディング**エージェント**に作業させる前に、「何を変えるか」を短い**仕様**の文書にまとめ、人とエージェントで合意してから実装させるための道具です。README は自らを「AI のコーディングアシスタントのための仕様駆動開発（**SDD**）」と説明しています。コマンドで動く道具（**CLI**）と、エージェントに読ませる**スキル**の組み合わせで、ライセンスは MIT です（[README](https://github.com/Fission-AI/OpenSpec)）。

たとえるなら、家の改築を職人に頼むときに、家全体の図面（今の正しい姿）を手元に置き、今回の工事では「どこを足し、どこを変え、どこを取り払うか」だけを書いた差分の図面を作って、施主と職人で確かめてから工事に入るようなものです。工事が終わったら、差分を家全体の図面に書き写しておきます。

## 2. 解決する課題（なぜ生まれたか）

エージェントに会話だけで作業を頼むと、意図が会話の中にしか残らず、結果もぶれがちです。Thoughtworks は、OpenSpec を使ったチームが、プロンプトだけで頼む場合より、エージェントの出力が予測しやすくなり、意図しない変更が減ったと報告している点を挙げています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/openspec)）。背景には、次の課題があります。

- **意図がぶれる**: 何を作るかを会話で伝えるだけだと、エージェントが意図と違うものを作ったり、頼んでいない場所まで変えたりします
- **既存のコードに仕様を一から書くのは重い**: 長く動いているシステムに、すべての振る舞いの仕様を最初から書き起こすのは現実的ではありません
- **仕様が古くなる**: 変更のたびの仕様が散らばると、今の正しい振る舞いがどれなのか分からなくなります
- **重い枠組みや特定の道具に縛られたくない**: 段階ごとの関門が厳しい枠組みや、特定の開発環境でしか使えない道具は、導入の負担が大きくなります（README の比較。後述）

## 3. 仕組みの概要

### 課題ごとの答え

| 課題 | OpenSpec の答え |
|---|---|
| 意図がぶれる | 変更ごとにフォルダを作り、提案（`proposal.md`: なぜ・何を）、設計（`design.md`: どうやって）、作業の一覧（`tasks.md`）、仕様の差分を書く。これらを確かめてから、エージェントに作業の一覧に沿って実装させる（[概念の説明](https://github.com/Fission-AI/OpenSpec/blob/main/docs/concepts.md)） |
| 既存のコードに仕様を一から書くのは重い | 仕様の全体ではなく、今回の変更で「足す・変える・取り除く」要件だけを、差分（**デルタ**）として書く。文書は既存のコードベースに手を入れる使い方（**ブラウンフィールド**）を第一に考えるとし、README も新規の開発だけでなく既存のコードベース向けだとしている（[概念の説明](https://github.com/Fission-AI/OpenSpec/blob/main/docs/concepts.md)、[README](https://github.com/Fission-AI/OpenSpec)） |
| 仕様が古くなる | 変更を終えるとき（**アーカイブ**）に、差分を `openspec/specs/` の仕様に取り込み、変更のフォルダを日付つきで保管庫に移す。`openspec/specs/` が常に「正しい姿の拠り所」になり、エージェントもそれを参照できる（[概念の説明](https://github.com/Fission-AI/OpenSpec/blob/main/docs/concepts.md)、[CLI の説明](https://github.com/Fission-AI/OpenSpec/blob/main/docs/cli.md)） |
| 重い枠組みや特定の道具に縛られたくない | 文書の作成の順には依存関係があるが、それは関門ではなく手助けだと位置づけている（[概念の説明](https://github.com/Fission-AI/OpenSpec/blob/main/docs/concepts.md)）。30を超える AI の道具に対応し（[README](https://github.com/Fission-AI/OpenSpec)）、Claude Code・Cursor・Codex・GitHub Copilot などで同じスキルを使える（[対応する道具の一覧](https://github.com/Fission-AI/OpenSpec/blob/main/docs/supported-tools.md)） |

### 仕様の書き方

仕様は領域ごとのフォルダ（例: `auth/spec.md`）に分けた Markdown です。要件は「`### Requirement: 名前`」の見出しの下に、SHALL（〜するものとする）や MUST などの決まった語（RFC 2119 の語）を使って書きます。続けて「`#### Scenario: 名前`」の見出しで、前提（GIVEN）・条件（WHEN）・結果（THEN）などの決まった形の具体例（**シナリオ**）を添えます。変更のフォルダの差分の仕様では、要件を「ADDED（足す）」「MODIFIED（置き換える）」「REMOVED（取り除く）」の見出しに分けます（[概念の説明](https://github.com/Fission-AI/OpenSpec/blob/main/docs/concepts.md)）。

### 作業の流れ

1. （任意）探る: エージェントと相談しながら考えをまとめる
2. 提案する: 変更のフォルダと、提案・設計・作業の一覧・仕様の差分を作る
3. 実装する: 作業の一覧に沿ってエージェントがコードを書く
4. （任意）確かめる
5. アーカイブする: 差分を仕様に取り込み、変更を保管庫に移す

エージェントの中では `/openspec-propose` のようなスキルで、手元では `openspec init`（準備）・`openspec list`（一覧）・`openspec validate`（書式の点検）・`openspec archive`（アーカイブ）などのコマンドで操作します（[README](https://github.com/Fission-AI/OpenSpec)、[CLI の説明](https://github.com/Fission-AI/OpenSpec/blob/main/docs/cli.md)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: エージェントと人のあいだに置く「合意の文書」の層です。実装そのものはエージェントに任せ、OpenSpec は何を作るかの記録と、その記録を最新に保つ流れを受け持ちます。

**代わりに失うもの・注意点**

- **文書が増える**: 仕様は Markdown で、エージェントが大量に書き出します。差分をアーカイブすることで和らぎますが、拠り所の仕様を保つには規律が要ると Thoughtworks は注意しています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/openspec)）
- **エージェント自身の進化**: Thoughtworks は前の版で、エージェントに備わる機能が、仕様駆動開発の道具を不要にしないかを見続けるよう勧めていました（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/openspec)）
- **変化が速い**: 最新の v2.0.0 では、スキルの置き場所が `.agents/skills/` にまとめられ、以前の `/opsx:` で始まるコマンドが無くなりました。v2.0.0 のリリースの説明は、`.agents/skills/` のスキルを直接書き換えていると `openspec update` で上書きされると注意しています。文書の一部も古くなっていると文書の目次自身が書いており、たとえば概念の説明は前の版のコマンド名（`/opsx:`）のままです（[リリースの一覧](https://github.com/Fission-AI/OpenSpec/releases)、[文書の目次](https://github.com/Fission-AI/OpenSpec/tree/main/docs)）
- **外部との通信**: 匿名の利用統計（コマンド名と版だけで、引数・パス・中身・個人情報は含まないと README は説明）を集め、設定や環境変数で止められます。CI の中では送りません（[README](https://github.com/Fission-AI/OpenSpec)）。`openspec update` は、新しい版が無いかを npm の登録所に問い合わせます（[CLI の説明](https://github.com/Fission-AI/OpenSpec/blob/main/docs/cli.md)）
- **モデルの力に頼る**: README は、推論の強いモデルで最もよく動くと書いています（[README](https://github.com/Fission-AI/OpenSpec)）

## 5. 採用を判断するときの観点

- **対象のコードベース**: Thoughtworks は、既存のコードベースに仕様を後から足していく使い方でうまくいっている点を評価しています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/openspec)）
- **文書を保つ余力**: 拠り所の仕様をレビューし、最新に保つ人と手順をチームで決められるか
- **使っているエージェント**: 対応の一覧に入っているか
- **入れ方**: README の手順には、Web からスクリプトを取ってそのまま実行する形と、npm（Node.js 22.12.0 以上が必要）や Homebrew で入れる形があります（[README](https://github.com/Fission-AI/OpenSpec)）。筆者の見立てでは、会社の端末では、パッケージの管理の道具で入れ、版を固定するほうが管理しやすいでしょう
- **始め方**: 筆者の見立てでは、1つのリポジトリで、小さな機能の追加を1件、提案からアーカイブまで通してみて、仕様の量とレビューの手間が見合うかを確かめるのが安全です

## 6. 関連技術・代替技術

- **仕様駆動開発（spec-driven development）**: 仕様を先に書き、それを AI の実装の拠り所にする手法。2025年11月の版の Techniques で Assess（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/spec-driven-development)）
- **GitHub Spec Kit**: GitHub による仕様駆動開発の道具。2026年4月の版の Languages & Frameworks で Assess（[Radar のページ](https://www.thoughtworks.com/en-us/radar/languages-and-frameworks/github-spec-kit)）。OpenSpec の README は、Spec Kit を「丁寧だが重く、段階の関門が厳しい」と比べている（作者側の見方。[README](https://github.com/Fission-AI/OpenSpec)）
- **Kiro**: 開発環境（IDE）や CLI などで使える、仕様を中心にした AI の開発の道具（[Kiro](https://kiro.dev/)）。OpenSpec の README は、Kiro が自前の開発環境に結びついている点を挙げて比べている（作者側の見方。[README](https://github.com/Fission-AI/OpenSpec)）
- **AI-driven documentation sprawl**: AI が文書を大量に生み出して手に負えなくなる問題。同じ Vol.35 の Techniques で Caution。Radar の OpenSpec のページが関連の項目として挙げている（[OpenSpec の Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/openspec)、[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/ai-driven-documentation-sprawl)）
- **mattpocock/skills**（[このリポジトリの解説](mattpocock-skills.md)）: 同じ Vol.35 の Tools の Trial。会話から仕様を書いてイシュートラッカーに載せるスキルなどを含む、小さなスキルの集まり

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（Assess から移動）
- 過去の版: 2026年4月の版で Assess
- Thoughtworks の評価の要旨: 既存のコードベースに仕様を後から足していく使い方で実績が積み上がり、エージェントの出力が安定する点を評価しています。一方で、仕様の文書が増えるため、拠り所の仕様を保つ規律が要ると注意しています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/tools/openspec

## 8. 深掘りするときの一次情報

- [OpenSpec の GitHub リポジトリ（README）](https://github.com/Fission-AI/OpenSpec)
- [OpenSpec の概念の説明](https://github.com/Fission-AI/OpenSpec/blob/main/docs/concepts.md)
- [OpenSpec の CLI の説明](https://github.com/Fission-AI/OpenSpec/blob/main/docs/cli.md)
- [OpenSpec のリリースの一覧](https://github.com/Fission-AI/OpenSpec/releases)

## 9. 確認日

- 内容の確認日: 2026-11-15（確認時点の最新のリリースは v2.0.0）
- Radar の評価の確認日: 2026-11-15（リングの推移は、Radar の項目のページで確認）

## 用語集

用語の説明の冒頭の「元の語」は、覚えるための手がかりとして、元の語（英単語の意味・略語の元の綴り・名前の元になった語）を短く示したものです。

| 用語 | 説明 |
|---|---|
| エージェント | 元の語: agent〈代理人〉。人の指示を受けて、コードを読み書きしたりコマンドを実行したりする AI |
| 仕様 | 元の語: spec（specification の略。明細）。何を作るか・どう振る舞うかを書いた文書 |
| SDD | 元の語: Spec-Driven Development の略（仕様駆動開発）。仕様を先に書き、それを拠り所に実装する進め方 |
| CLI | 元の語: Command Line Interface の略。文字のコマンドを打って操作する使い方や道具 |
| スキル | 元の語: skill〈技能〉。エージェントに特定の仕事の進め方を教える手順書 |
| デルタ | 元の語: delta〈ギリシャ文字の Δ。数学で差・変化量を表す〉。今の仕様に対する差分 |
| ブラウンフィールド | 元の語: brownfield〈使われたあとの土地〉。すでに動いているシステムに手を入れる開発。反対は greenfield〈未開発の緑地〉で、一から作る開発 |
| アーカイブ | 元の語: archive〈記録の保管所〉。終わった変更を、仕様に取り込んだうえで保管庫に移すこと |
| シナリオ | 元の語: scenario〈筋書き〉。「この条件のとき、こうなる」という具体例で要件を示したもの |
| SHALL / MUST | 元の語: shall〈〜するものとする〉、must〈〜しなければならない〉。仕様で、要件の強さを示す決まった助動詞（RFC 2119 という文書で定められた使い方） |
| Markdown | 見出しや箇条書きを記号で書く、軽い文書の書き方 |
| npm | Node.js（JavaScript を動かす環境）のパッケージを配布・導入する仕組み |
| CI | 元の語: Continuous Integration の略（継続的な統合）。変更のたびに自動でビルドやテストを走らせる仕組み |
| MIT ライセンス | 元の語: MIT（Massachusetts Institute of Technology の略。マサチューセッツ工科大学）。著作権の表示を残せば自由に使える、短いオープンソースのライセンス |
