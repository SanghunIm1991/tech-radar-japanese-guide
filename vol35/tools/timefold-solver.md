# Timefold Solver（制約のもとで計画を最適化するオープンソースのエンジン）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Tools（開発や運用で使う道具） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

**名前の手がかり**: Time〈時間〉＋ fold〈折りたたむ〉＋ Solver〈解くもの〉

Timefold Solver は、配送の経路・従業員のシフト・作業の割り当てのような「計画の問題」を、決めたルール（**制約**）のもとで、できるだけ良い答えに近づける Java と Kotlin のライブラリです。文書は「軽く、アプリに組み込める、制約を満たす解を探すエンジン」と説明しています。開発元は Timefold BV で、オープンソースの版（Community Edition）のライセンスは Apache-2.0 です（[GitHub の README](https://github.com/TimefoldAI/timefold-solver)、[文書の紹介](https://docs.timefold.ai/timefold-solver/latest/introduction)）。

Red Hat の **OptaPlanner** から2023年4月に分かれた（**フォーク**した）もので、OptaPlanner を作ったチームが開発を続けています（[GitHub の README](https://github.com/TimefoldAI/timefold-solver)、[製品のページ](https://timefold.ai/solver)）。

たとえるなら、時間割を作る先生のようなものです。「同じ先生が同じ時間に2つの教室にいてはいけない」という絶対の決まりを守りつつ、「先生の空き時間をなるべくまとめたい」という希望をできるだけ満たす時間割を、限られた時間の中で何度も組み替えながら探します。

## 2. 解決する課題（なぜ生まれたか）

Thoughtworks は、配送の経路・シフト・作業の割り当て・保守の予定づくりのように、すべての組み合わせを調べ尽くすのが現実的でない問題に使える点を挙げています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/timefold-solver)）。背景には、次の課題があります。

- **組み合わせが多すぎる**: 車の台数や訪問先、人や勤務枠が増えると、組み合わせは爆発的に増え、総当たりでは時間内に終わりません（筆者の見立て）
- **守るべき決まりと、望ましい目標が混ざる**: 法律や物理的な制約のように絶対に守る決まりと、移動の時間を減らしたいといった目標を、同時に扱う必要があります（筆者の見立て）
- **業務の決まりが変わるたびに作り直す**: 計画の仕組みを問題ごとに一から書くと、決まりが1つ変わるだけで大きな手直しになります（筆者の見立て）
- **既存のシステムに組み込みたい**: 計画の機能だけを、すでにある業務のアプリの中で使いたい場面があります（筆者の見立て）

## 3. 仕組みの概要

### 課題ごとの答え

| 課題 | Timefold Solver の答え |
|---|---|
| 組み合わせが多すぎる | すべてを調べ尽くさず、まず手早く一通りの案を作る**構築ヒューリスティック**を行い、続けて案を少しずつ組み替えて良くしていく**局所探索**（タブー探索・焼きなまし法など）を行う。何も設定しなければこの2段階で動く。決めた時間などで打ち切り、それまでに見つけた最良の案を返す。文書は、最適な答えを見つけるのはふつう誰にもできないので、使える時間の中で最良の答えを探すことに重きを置くと書いている（[最適化の手法の概要](https://docs.timefold.ai/timefold-solver/latest/optimization-algorithms/overview)） |
| 守るべき決まりと、望ましい目標が混ざる | 案の良さを**スコア**で表し、絶対の決まり（**ハード制約**）と望ましい目標（**ソフト制約**）を別の段にする。ハード制約の違反が無い案は、ソフトの点がどれだけ悪くても、違反のある案より良いと見なす。ソフトの段より常に優先される中間の段（medium）もある（[制約とスコアの概要](https://docs.timefold.ai/timefold-solver/latest/constraints-and-score/overview)）。Thoughtworks も、ハード制約とソフト制約の考え方を説明している（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/timefold-solver)） |
| 業務の決まりが変わるたびに作り直す | 問題を、業務のクラスに目印（アノテーション）を付けて表し、制約は **Constraint Streams** という書き方で、Java や Kotlin のコードとして1つずつ書く。探す仕組みはエンジンが受け持つので、決まりが変われば制約のコードを足したり直したりすればよい。案を組み替えたとき、影響を受けた部分だけのスコアを計算し直す（[問題の表し方](https://docs.timefold.ai/timefold-solver/latest/domain-modeling/modeling-planning-problems)、[スコアの計算](https://docs.timefold.ai/timefold-solver/latest/constraints-and-score/score-calculation)） |
| 既存のシステムに組み込みたい | 純粋な Java のライブラリとしてアプリに組み込める。Quarkus と Spring Boot に対応し、最適化のサービスとして作ることもできる（[文書の紹介](https://docs.timefold.ai/timefold-solver/latest/introduction)） |

### 問題の表し方

時間割の例でいうと、次のように表します（[問題の表し方](https://docs.timefold.ai/timefold-solver/latest/domain-modeling/modeling-planning-problems)）。

- **計画の対象（planning entity）**: エンジンが値を決めるもの。例: 授業
- **計画の変数（planning variable）**: 対象のうち、エンジンが選んで埋める項目。例: 授業の時間帯と教室
- **問題の事実（problem fact）**: 変わらない入力。例: 時間帯の一覧、教室の一覧
- **計画の解（planning solution）**: 上のすべてとスコアをまとめたもの

配送の経路のように「順番」を決める問題には、順番つきの一覧を埋めて並べ替える変数も用意されています。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: 業務のアプリの中に組み込む、計画を作るための部品です。機械学習の仕組みではありません。Thoughtworks は、AI の看板の下で語られることがあっても、学習用のデータや学習したモデルは使わず、利用者が決めた制約のもとでヒューリスティックな探索を行うものだと説明しています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/timefold-solver)）。

**代わりに失うもの・注意点**

- **最適の保証は無い**: 決めた時間の中で見つけた最良の答えを返すもので、本当に最適かどうかは保証されません（[最適化の手法の概要](https://docs.timefold.ai/timefold-solver/latest/optimization-algorithms/overview)）
- **答えの質は、制約の書き方と調整しだい**: 文書は、初期設定の値は多くの場合（特に試作では）十分だとしつつ、調整には用意された比較の道具（Benchmarker）で手法や設定を比べるよう勧めています。時間で打ち切る条件は機械によって結果が変わるとも書いています。また、できるだけ早く本番の規模のデータで試すよう勧めています（[最適化の手法の概要](https://docs.timefold.ai/timefold-solver/latest/optimization-algorithms/overview)）
- **スコアの計算の注意**: スコアの計算に小数（float・double）を使わない、大きな重みで段を作るごまかしをしないなどの注意が文書にあります（[制約とスコアの概要](https://docs.timefold.ai/timefold-solver/latest/constraints-and-score/overview)）
- **Java の世界の道具**: Java 21 以上の Java の実行環境で動き、書く言語は Java か Kotlin です（[文書の紹介](https://docs.timefold.ai/timefold-solver/latest/introduction)）
- **一部の機能は商用の版**: 複数のスレッドで解く機能、近くの候補に絞って組み替える機能（nearby selection）、問題を分けて解く機能などは、オープンソースではない Enterprise の版にあり、ライセンスのファイルを使います（[商用の版](https://docs.timefold.ai/timefold-solver/latest/commercial-editions/commercial-editions)）

## 5. 採用を判断するときの観点

- **実績**: Thoughtworks は、サプライチェーンの分野で本番に使い、複雑な業務の状況をすばやく表せたとしています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/timefold-solver)）
- **問題の形**: 割り当て・順番・時間の決定を伴う計画の問題か。予測（需要の見込みなど）が主な課題なら、別の道具の出番です（筆者の見立て）
- **規模と速さ**: 問題の大きさに対して、オープンソースの版で間に合うか、複数のスレッドなどの Enterprise の機能が要るか
- **作り込みか、出来合いか**: 開発元は、配送や訪問の経路・シフト表づくりなどの出来合いのモデルを、REST の API で使えるサービスとしても提供しています（[文書の紹介](https://docs.timefold.ai/timefold-solver/latest/introduction)、[料金](https://timefold.ai/pricing)）。筆者の見立てでは、問題がそれらに近く、データを外部に出せるなら、出来合いのほうが早く始められます
- **始め方**: 筆者の見立てでは、ハード制約だけで「とにかく守れる案」が出るかを先に確かめ、それからソフト制約を1つずつ足して、比較の道具で手法を選ぶのが安全です

## 6. 関連技術・代替技術

- **OptaPlanner**: Timefold Solver の前身で、Red Hat が支えていたオープンソースの計画のエンジン（[OptaPlanner](https://optaplanner.io/)、[GitHub の README](https://github.com/TimefoldAI/timefold-solver)）
- **数理最適化のソルバー**: 問題を数式（線形計画・整数計画など）で表して解く道具（一般的な説明）。条件によっては最適であることを確かめられる一方、業務の決まりを数式に直す手間がかかる
- **自前のヒューリスティック**: 問題ごとに貪欲法などの手順を自分で書くやり方（一般的な説明）。小さな問題では十分なこともあるが、決まりが増えると保守が難しくなる
- **Timefold の出来合いのモデル**: 同じ開発元による、経路やシフトの最適化の API のサービス（[料金](https://timefold.ai/pricing)）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: OptaPlanner の後継として、総当たりが現実的でない計画の問題を、ハード制約とソフト制約で表して解ける点を、サプライチェーンでの本番の実績とともに評価しています。AI の看板とは違い、学習ではなく探索で解く道具だと説明しています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/tools/timefold-solver

## 8. 深掘りするときの一次情報

- [Timefold Solver の GitHub リポジトリ（README）](https://github.com/TimefoldAI/timefold-solver)
- [Timefold Solver の文書（紹介）](https://docs.timefold.ai/timefold-solver/latest/introduction)
- [最適化の手法の概要](https://docs.timefold.ai/timefold-solver/latest/optimization-algorithms/overview)
- [制約とスコアの概要](https://docs.timefold.ai/timefold-solver/latest/constraints-and-score/overview)
- [商用の版](https://docs.timefold.ai/timefold-solver/latest/commercial-editions/commercial-editions)

## 9. 確認日

- 内容の確認日: 2026-11-15（確認時点の最新のリリースは v2.8.0）
- Radar の評価の確認日: 2026-11-15（リングの推移は、Radar の項目のページで確認）

## 用語集

用語の説明の冒頭の「元の語」は、覚えるための手がかりとして、元の語（英単語の意味・略語の元の綴り・名前の元になった語）を短く示したものです。

| 用語 | 説明 |
|---|---|
| 制約 | 元の語: constraint〈縛り・制限〉。計画が守るべき決まりや、満たしたい条件 |
| ハード制約・ソフト制約 | 元の語: hard〈硬い〉、soft〈柔らかい〉。絶対に破れない決まりと、できるだけ満たしたい目標 |
| スコア | 元の語: score〈得点〉。案の良さを表す点数。Timefold Solver では、ハード・ソフトなどの段ごとに数える |
| ヒューリスティック | 元の語: heuristic〈発見に役立つ〉（ギリシャ語の「見つける」から）。最適の保証はないが、実用的に良い答えを早く見つける手順 |
| 構築ヒューリスティック | 元の語: construction〈組み立て〉。最初の一通りの案を手早く組み立てる手順 |
| 局所探索 | 元の語: local search〈近くを探すこと〉。今の案を少しずつ変えた近くの案を調べ、良くなる方へ進む探し方 |
| タブー探索 | 元の語: tabu〈禁忌〉。最近試した変更をしばらく禁じて、同じところをぐるぐる回らないようにする局所探索 |
| 焼きなまし法 | 元の語: simulated annealing〈まねた焼きなまし〉。金属を熱してゆっくり冷やす工程にならい、はじめは悪くなる変更もある程度受け入れ、だんだん厳しくする探し方 |
| アノテーション | 元の語: annotation〈注釈〉。Java などで、クラスや項目に `@` で始まる目印を付けて、意味を伝える書き方 |
| フォーク | 元の語: fork〈分かれ道・食器のフォーク〉。あるソフトウェアのソースコードを元に、別の開発の流れとして分かれること |
| OptaPlanner | Red Hat が支えていた、オープンソースの計画のエンジン。Timefold Solver の前身 |
| Kotlin | Java の実行環境で動く、簡潔に書けるプログラミング言語 |
| REST | 元の語: Representational State Transfer の略。Web の仕組み（URL と HTTP）でデータをやり取りする API の作り方 |
| Apache-2.0 | 元の語: Apache License 2.0。商用でも使いやすい、代表的なオープンソースのライセンス |
