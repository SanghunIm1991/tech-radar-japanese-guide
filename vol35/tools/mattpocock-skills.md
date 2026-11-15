# mattpocock/skills（エンジニア向けのエージェントのスキル集）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Tools（開発や運用で使う道具） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

**名前の手がかり**: skills〈技能〉

mattpocock/skills は、AI のコーディング**エージェント**に読ませる**スキル**を集めた、GitHub の公開のリポジトリです。作者の Matt Pocock が日々の開発で使っているもので、README は「雰囲気で書かせる開発（**vibe coding**）ではなく、本当のエンジニアリングのためのスキル」と位置づけています。ライセンスは MIT です（[README](https://github.com/mattpocock/skills)、[LICENSE](https://github.com/mattpocock/skills/blob/main/LICENSE)）。

スキルとは、エージェントに特定の仕事の進め方を教える手順書です。1つのスキルは1つのフォルダで、中心に `SKILL.md` というファイルがあり、冒頭に名前と「いつ使うか」の説明を書きます（[tdd のスキル](https://github.com/mattpocock/skills/blob/main/skills/engineering/tdd/SKILL.md)）。

たとえるなら、腕のいい先輩が自分の机の引き出しにしまっていた「仕事の型のメモ」の束を、誰でも写せるように公開したものです。メモは短く、1枚ずつ取り出して組み合わせたり、書き換えたりできます。

## 2. 解決する課題（なぜ生まれたか）

エージェントにコードを書かせる開発が広がるにつれ、エージェントの手順をどう整えるかが課題になっています。Thoughtworks は、このスキル集が、しっかりしたソフトウェアエンジニアリングの習慣をエージェントを使った開発に持ち込むのに役立つと見ています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/mattpocock-skills)）。背景には、次の課題があります。

- **すり合わせが抜ける**: エージェントにいきなり作らせると、要件の前提や設計の判断を人と確かめないまま進み、あとで食い違いが分かります（筆者の見立て）
- **良い習慣を毎回指示し直す**: テストを先に書く、変更をレビューするといった習慣を、作業のたびにプロンプトで指示し直す手間がかかります（筆者の見立て）
- **大きな枠組みは重い**: 開発の流れを丸ごと握ろうとする大きな枠組みは、使う人の主導権を奪い、流れの中の不具合を直しにくくすると README は書いています（[README](https://github.com/mattpocock/skills)）
- **エージェントごとに入れ方が違う**: Claude Code・Codex・GitHub Copilot など、エージェントごとにスキルの入れ方が異なります（[README](https://github.com/mattpocock/skills) の入れ方の一覧）

## 3. 仕組みの概要

### 課題ごとの答え

| 課題 | mattpocock/skills の答え |
|---|---|
| すり合わせが抜ける | **grilling**（質問攻め）のスキルで、エージェントが人に問い続ける。計画を判断の木として整理し、前提が決まった判断をまとめて番号つきで尋ね、それぞれに推奨の答えを添えて、人の返答を待つ。事実（コードや環境のこと）は人に聞かず、エージェントが別の**サブエージェント**に調べさせる。人が共通の理解を確かめるまで作業に移らない（[grilling のスキル](https://github.com/mattpocock/skills/blob/main/skills/productivity/grilling/SKILL.md)） |
| 良い習慣を毎回指示し直す | 習慣をスキルにしておき、エージェントが仕事に合うと判断したときに自分で呼び出せるようにする（README の「model-invoked」の分類）。例: テストを先に書く `tdd`、仕様と規約の2つの観点で並行してレビューする `code-review`（[README](https://github.com/mattpocock/skills)）、失敗を確かめる仕組みを作る→再現→仮説→計測→修正と回帰テストの順で進める `diagnosing-bugs`（[diagnosing-bugs のスキル](https://github.com/mattpocock/skills/blob/main/skills/engineering/diagnosing-bugs/SKILL.md)） |
| 大きな枠組みは重い | スキルを小さく、書き換えやすく、組み合わせられるように作る。README は自由に手を加えて自分のものにするよう勧めている（[README](https://github.com/mattpocock/skills)）。Thoughtworks も、示された流れは提案にとどまる点を挙げている（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/mattpocock-skills)） |
| エージェントごとに入れ方が違う | Claude Code・Codex・GitHub Copilot には**プラグイン**として、Gemini CLI にはそのコマンドで入れる。ほかのエージェント（Cursor・OpenCode など）には、`npx skills@latest add mattpocock/skills -a <エージェント>` の形で入れる（[README](https://github.com/mattpocock/skills)） |

### スキルの分け方

README は、スキルを「エンジニアリング」と「生産性」の2つの分野に分け、さらに呼び出し方で2つに分けています（[README](https://github.com/mattpocock/skills)）。

- **人が呼び出すもの**（`/名前` と打つ）: 流れを取りまとめる。例: 会話から仕様を書いて**イシュートラッカー**に載せる `to-spec`、計画を小さな**チケット**に割る `to-tickets`、仕様やチケットから実装する `implement`、質問攻めの入口の `grill-me`、別のエージェントへの引き継ぎの文書を作る `handoff`
- **エージェントも自分で呼び出せるもの**: 繰り返し使う習慣。例: `tdd`・`code-review`・`grilling`、用語が決まるたびに用語集（`GLOSSARY.md`）を更新し、重い判断は **ADR** に残すよう提案する `domain-modeling`（[domain-modeling のスキル](https://github.com/mattpocock/skills/blob/main/skills/engineering/domain-modeling/SKILL.md)）

人が呼び出すスキルの中から、エージェントが習慣のスキルを呼ぶこともあります。たとえば `implement` は、合意した区切りで `tdd` を使い、コミットの前に `code-review` で締めくくると README は説明しています。

### 最初の設定

エンジニアリングのスキルを使う前に、リポジトリごとに一度 `/setup-matt-pocock-skills` を実行します。これは、使うイシュートラッカー（GitHub・GitLab・手元のファイルなど）、（`triage` のスキルを入れている場合は）仕分けのラベル、文書の置き場所を、人に確かめながら決め、`CLAUDE.md` か `AGENTS.md`（エージェント向けの説明書）と `docs/agents/` の下のファイルに書き込みます。GitHub か GitLab の場合は、足りないラベルを作ります。リモートが無ければ、`git init` などを実行することもあります（[setup のスキル](https://github.com/mattpocock/skills/blob/main/skills/engineering/setup-matt-pocock-skills/SKILL.md)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: エージェントそのものではなく、エージェントの上に載せる「仕事の進め方」の層です。どのモデルでも動くと README は書いています（[README](https://github.com/mattpocock/skills)）。

**代わりに失うもの・注意点**

- **丸ごと入れる危うさ**: Thoughtworks は、スキルの集まりを評価せずに丸ごと取り入れることに注意を促しています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/mattpocock-skills)）。筆者の見立てでは、手順をスキルに任せきると、チームが自分で考えて判断する機会が減るおそれがあります
- **外部への書き込みを含む**: いくつかのスキルは、イシュートラッカーに仕様を載せたり（`to-spec`）、ラベルを作ったり、`CLAUDE.md` などを書き換えたりします（[to-spec のスキル](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-spec/SKILL.md)、[setup のスキル](https://github.com/mattpocock/skills/blob/main/skills/engineering/setup-matt-pocock-skills/SKILL.md)）。筆者の見立てでは、入れる前に各スキルの `SKILL.md` を読み、エージェントに許す操作の範囲と照らし合わせるべきです
- **更新の仕方の違い**: プラグインとして入れると自動で更新され、skills.sh（`npx skills`）で入れるとファイルがプロジェクトに写され、手で更新します。両方を入れるとスキルが二重になるので、エージェントごとにどちらか1つにするよう README は書いています（[README](https://github.com/mattpocock/skills)）。筆者の見立てでは、自動の更新では手順の中身が知らないうちに変わりうるので、チームで使うなら、写して固定する形のほうが管理しやすい場合があります
- **作者の流儀**: 作者が日々の自分の仕事に使っているスキル集です（[README](https://github.com/mattpocock/skills)）。筆者の見立てでは、作者の流儀が色濃く出ています。版の付いたタグ（確認時点の最新は v1.3.2）で区切られています（[タグの一覧](https://github.com/mattpocock/skills/tags)）

## 5. 採用を判断するときの観点

- **どの課題に効かせたいか**: Thoughtworks は、エージェントを使う開発の流れにもう少し構造が欲しいチームに、見てみる価値があるとしています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/mattpocock-skills)）。筆者の見立てでは、要件のすり合わせが弱いなら grilling、品質が不安なら tdd と code-review のように、課題に合うものから選ぶのがよいでしょう
- **使っているエージェント**: README の入れ方の一覧にあるか
- **許す操作の範囲**: イシュートラッカーやリポジトリへの書き込みを、エージェントに許してよいか
- **更新の管理**: 自動の更新に任せるか、写して固定するか
- **始め方**: 筆者の見立てでは、まず `grill-me` のように何も書き込まないスキルを1つ試し、効き目を確かめてから広げるのが安全です

## 6. 関連技術・代替技術

- **Grilling**（[このリポジトリの解説](../techniques/grilling.md)）: 同じ Vol.35 の Techniques の Trial。エージェントに質問攻めにさせて、要件や前提を確かめる手法。Thoughtworks は、このスキル集が grilling による要件や前提の分析にも役立つとしている（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/mattpocock-skills)）
- **Agent skill cognitive surrender**: 同じ Vol.35 の Techniques で Caution の項目。Thoughtworks が、スキルの集まりを丸ごと取り入れることへの注意として名前を挙げている（[mattpocock/skills の Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/mattpocock-skills)、[この項目の Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/agent-skill-cognitive-surrender)）
- **Agent Skills**: スキルという仕組みそのもの。2026年4月の版の Techniques で Trial（[Radar のページ](https://www.thoughtworks.com/en-us/radar/techniques/agent-skills)）
- **Agent Package Manager**（[このリポジトリの解説](agent-package-manager.md)）: 同じ Vol.35 の Tools の Trial。エージェントの設定を依存関係として管理する道具
- **GSD・BMAD・Spec-Kit**: 開発の流れを握って助けようとする枠組みとして、README が比べる相手に挙げている（[README](https://github.com/mattpocock/skills)）。README は、これらが使う人の主導権を奪うと書いている（作者の見方）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: 小さく組み合わせやすいスキルで、エンジニアリングの良い習慣をエージェントを使う開発に持ち込める点を評価しています。一方で、スキルの集まりを評価せずに丸ごと取り入れないよう注意しています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/tools/mattpocock-skills

## 8. 深掘りするときの一次情報

- [mattpocock/skills の GitHub リポジトリ（README）](https://github.com/mattpocock/skills)
- [grilling のスキル](https://github.com/mattpocock/skills/blob/main/skills/productivity/grilling/SKILL.md)
- [tdd のスキル](https://github.com/mattpocock/skills/blob/main/skills/engineering/tdd/SKILL.md)
- [setup-matt-pocock-skills のスキル](https://github.com/mattpocock/skills/blob/main/skills/engineering/setup-matt-pocock-skills/SKILL.md)

## 9. 確認日

- 内容の確認日: 2026-11-15（確認時点の最新のタグは v1.3.2）
- Radar の評価の確認日: 2026-11-15（リングの推移は、Radar の項目のページで確認）

## 用語集

用語の説明の冒頭の「元の語」は、覚えるための手がかりとして、元の語（英単語の意味・略語の元の綴り・名前の元になった語）を短く示したものです。

| 用語 | 説明 |
|---|---|
| エージェント | 元の語: agent〈代理人〉。人の指示を受けて、コードを読み書きしたりコマンドを実行したりする AI |
| スキル | 元の語: skill〈技能〉。エージェントに特定の仕事の進め方を教える手順書。フォルダと `SKILL.md` のファイルからなる |
| サブエージェント | 元の語: sub〈下の・副の〉＋ agent。エージェントが、別の仕事を任せるために呼び出す、もう1つのエージェント |
| プラグイン | 元の語: plug in〈差し込む〉。道具にあとから機能を足すための追加の部品 |
| vibe coding | 元の語: vibe〈雰囲気・ノリ〉＋ coding。中身を細かく確かめず、AI に雰囲気で指示してコードを書かせるやり方 |
| grilling | 元の語: grill〈網焼きにする。転じて、厳しく問い詰める〉。エージェントに人を質問攻めにさせて、計画の穴を見つける使い方 |
| TDD | 元の語: Test-Driven Development の略（テスト駆動開発）。先に失敗するテストを書き、それを通すコードを書き、整える、を繰り返す進め方 |
| 仕様 | 元の語: spec（specification の略。明細）。何を作るか・どう振る舞うかを書いた文書 |
| チケット | 元の語: ticket〈札・切符〉。作業1件を記録した札。イシューとほぼ同じ意味で使う |
| イシュートラッカー | 元の語: issue〈問題・論点〉＋ tracker〈追跡するもの〉。やることを1件ずつ登録し、担当や状態を追う道具 |
| ADR | 元の語: Architecture Decision Record の略。設計上の判断と、その理由を1件ずつ残す記録 |
| 用語集（GLOSSARY.md） | 元の語: glossary〈用語集〉。プロジェクトで使う言葉の意味をそろえるための一覧 |
| コミット | 元の語: commit〈託す・確約する〉。変更をひとまとまりにして、履歴に記録すること |
| npx | Node.js（JavaScript を動かす環境）のパッケージを、その場で取ってきて実行するコマンド |
| MIT ライセンス | 元の語: MIT（Massachusetts Institute of Technology の略。マサチューセッツ工科大学）。著作権の表示を残せば自由に使える、短いオープンソースのライセンス |
