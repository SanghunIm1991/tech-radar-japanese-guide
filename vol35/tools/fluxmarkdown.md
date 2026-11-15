# FluxMarkdown

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Tools（開発や運用で使う道具） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

**名前の手がかり**: Flux〈流れ・絶え間ない変化〉＋ Markdown〈英語の mark down は「書き留める」の意味。名前の由来ではない〉

FluxMarkdown は、Mac の Finder でファイルを選んでスペースキーを押すだけで、**Markdown** の文書を、図や数式まで描かれた見た目でのぞける道具です。macOS に元からある **Quick Look**（ファイルを開かずに中身をすばやくプレビューする機能）に、Markdown を描く力を足す拡張として動きます（[GitHub の README](https://github.com/xykong/flux-markdown)、[Apple のサポート](https://support.apple.com/guide/mac-help/view-and-edit-files-with-quick-look-mh14119/mac)）。

たとえるなら、本棚から本を抜き出して机で開かなくても、背表紙に指を当てるだけで中のページが透けて見える眼鏡です。

## 2. 解決する課題（なぜ生まれたか）

**Markdown** は、`#` で見出し、`-` で箇条書き、のように記号で書式を表す、軽い文書の書き方です。README・設計メモ・議事録などで広く使われています。Thoughtworks は、ターミナルで動く道具（**CLI**）や AI の**コーディングエージェント**を中心に働くようになっても、Markdown を整った見た目で見るために IDE を開くことがある、と指摘しています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/fluxmarkdown)）。

すると、次のような課題が出てきます。

- **中身を見るだけなのに、重い道具を開く手間**: Markdown のファイルを整った見た目で確かめるには、エディタや統合開発環境（**IDE**）を開くのがふつうです。ターミナルで作業していると、見るためだけに別の大きなアプリを立ち上げることになります
- **図や数式は、描かれた形で確かめたい**: Markdown の中には、**Mermaid**（文字で書く図）や数式を書くことがあります。記号のままでは、図の形や数式が正しいかを判断しにくいものです

## 3. 仕組みの概要

### 課題ごとの答え

| 課題 | FluxMarkdown の答え |
|---|---|
| 中身を見るだけなのに、重い道具を開く手間 | Finder でファイルを選んでスペースキーを押すと、Quick Look の画面に整った見た目で表示する。アプリを開く必要がない（[README](https://github.com/xykong/flux-markdown)、[Apple のサポート](https://support.apple.com/guide/mac-help/view-and-edit-files-with-quick-look-mh14119/mac)） |
| 図や数式は、描かれた形で確かめたい | プレビューの中で、Mermaid の図、**KaTeX** による数式、**Vega**・Vega-Lite のグラフ、**Graphviz** の図を描く（[README](https://github.com/xykong/flux-markdown)） |

### どう動くか

Quick Look は、アプリが追加した拡張に、特定の種類のファイルのプレビューを受け持たせられる作りです（一般的な説明）。FluxMarkdown はその拡張として登録され、Markdown のファイルが選ばれたときに描画を受け持ちます。README は、登録されているかを `pluginkit` というコマンドで確かめる方法を案内しています（[README](https://github.com/xykong/flux-markdown)）。

描画に使う部品は、アプリに同梱されたものを使うと README に書かれています（"bundled local assets"）。つまり、文書を描くたびにインターネットから部品を取ってくる作りではありません（[README](https://github.com/xykong/flux-markdown)）。

### 主な機能

README に挙がっている主な機能は次のとおりです（[README](https://github.com/xykong/flux-markdown)）。

- **GFM**（GitHub で使われる Markdown の書き方）の表・タスクリスト・注意書きの囲み
- 40以上の言語の**構文ハイライト**（コードを色分けして読みやすくする表示）
- 見出しから作る目次（今読んでいる節が分かる）
- 明るい表示・暗い表示・システムの設定に合わせる表示の切り替え
- PDF・HTML への書き出し
- タブ区切りの表のファイル（.tsv）の読み取り専用の表示

### 入れ方

**Homebrew**（Mac で道具を入れるための仕組み）で、作者が用意した入手先（tap）から入れる方法を、README は推奨しています。ほかに、配布されているディスクイメージ（DMG）から入れる方法と、ソースから組み立てる方法もあります（[README](https://github.com/xykong/flux-markdown)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: 開発者の手元の端末で使う「見るための道具」です。チームの仕組み（サーバーや CI）に組み込むものではなく、1人ずつの作業の流れを軽くします。

**代わりに失うもの・注意点**

- **Mac だけ**: macOS の Quick Look の拡張なので、Windows や Linux では使えません。チームの端末がまちまちなら、全員の共通の道具にはなりません
- **見るだけで、書けない**: プレビューの道具で、編集はできません。表のファイルの表示も「読み取り専用」と README に明記されています。大きな表は、500 KiB・1,000行・100列・20,000セルの上限で切り詰められます（[README](https://github.com/xykong/flux-markdown)）
- **外部への通信が既定で有効**: 描画そのものは同梱の部品で行いますが、アプリは更新の確認をし、既定で仮名の利用統計を送ります。設定（Share Usage Statistics）で止められます。統計には文書の名前・パス・中身を含まないと README は書いています（[README](https://github.com/xykong/flux-markdown)）。筆者の見立てでは、会社の端末で使うなら、社内の規程に照らして、この設定を先に確かめておくのが安全です
- **入れ方の注意**: README の困ったときの案内には、「壊れている」「開発元を確認できない」と表示されたときに、`xattr -cr` でファイルに付いた検疫の印を外す方法が書かれています（[README](https://github.com/xykong/flux-markdown)）。筆者の見立てでは、これは macOS の確認の仕組み（**Gatekeeper**）を回避する操作なので、会社の端末では管理者の判断に従うべきです。なお README には、Apple の**公証**（notarization）を受けているかの記載は見当たりませんでした。比べると、同じ種類の道具の QLMarkdown は、配布するアプリが公証と署名を受けていると明記しています（[QLMarkdown](https://github.com/sbarex/QLMarkdown)）
- **ライセンス**: オープンソース版は GPL-3.0 で、閉じたソースの製品に使う場合のための商用ライセンスも用意された**デュアルライセンス**です（[README](https://github.com/xykong/flux-markdown)）。筆者の見立てでは、手元で道具として使うだけなら影響は小さく、組み込んで配る場合に条件の確認が要ります
- **個人の開発に依存**: リポジトリは個人のアカウントのもので、組織の記載はありません。更新は頻繁ですが、Finder のプレビューが FluxMarkdown ではなく標準の表示に戻ってしまうという報告が、未解決のまま残っています（v1.45.774 のリリースノート。[Releases](https://github.com/xykong/flux-markdown/releases)）

## 5. 採用を判断するときの観点

- **端末**: チームの大半が Mac で、Markdown のファイルを Finder から開く場面が多いか
- **書くものの中身**: Mermaid の図や数式を多く書くか。書かないなら、より単純な道具や IDE のプレビューで足りることもあります
- **会社の規程**: 外部の入手先（tap）からの導入、検疫の印を外す操作、利用統計の送信が、社内の規程で許されるか
- **代わりの道具との比較**: 同じ Quick Look の拡張の QLMarkdown と、機能（Vega・Graphviz の対応など）や配布の形（公証・署名）を比べる
- **試し方**: 1人の端末で入れ、普段の README や設計メモで表示と動作の軽さを確かめてから広げる

## 6. 関連技術・代替技術

- **QLMarkdown**: Markdown を Quick Look で表示する、同じ種類の拡張。数式（MathJax）と Mermaid に対応し、ライセンスは GPL-3.0（[QLMarkdown](https://github.com/sbarex/QLMarkdown)）。Thoughtworks は、FluxMarkdown のほうが対応する範囲が広いと見ています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/fluxmarkdown)）
- **Mermaid**: 文字で図を書く仕組み。Radar では 2023年9月の版で Adopt（[Radar の Mermaid のページ](https://www.thoughtworks.com/en-us/radar/tools/mermaid)）
- **markdown-preview-enhanced**: FluxMarkdown の README が、着想を得て一部をもとにしたと書いている Markdown のプレビューの道具（[README](https://github.com/xykong/flux-markdown)）
- **IDE やエディタの Markdown のプレビュー**: 多くのエディタが備える、編集しながら見た目を確かめる機能（一般的な説明）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: CLI やエージェントを中心に働く開発者が、図や数式を含む Markdown を、エディタを開かずに確かめられる軽い手段として、試す価値があると見ています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/tools/fluxmarkdown

## 8. 深掘りするときの一次情報

- [FluxMarkdown の GitHub リポジトリ（README）](https://github.com/xykong/flux-markdown)
- [FluxMarkdown のリリースの一覧](https://github.com/xykong/flux-markdown/releases)
- [Quick Look の使い方（Apple のサポート）](https://support.apple.com/guide/mac-help/view-and-edit-files-with-quick-look-mh14119/mac)
- [QLMarkdown の GitHub リポジトリ](https://github.com/sbarex/QLMarkdown)

## 9. 確認日

- 内容の確認日: 2026-11-15（確認時点の最新のリリースは v1.50.828）
- Radar の評価の確認日: 2026-11-15（リングの推移は、Radar の項目のページで確認）

## 用語集

用語の説明の冒頭の「元の語」は、覚えるための手がかりとして、元の語（英単語の意味・略語の元の綴り・名前の元になった語）を短く示したものです。

| 用語 | 説明 |
|---|---|
| Markdown | 元の語: 英語の mark down は「書き留める」の意味（名前の由来ではなく、覚えるための手がかり）。`#` で見出し、`-` で箇条書き、のように記号で書式を表す、軽い文書の書き方 |
| Quick Look | 元の語: quick〈すばやい〉＋ look〈見ること〉。macOS で、ファイルを選んでスペースキーを押すと、開かずに中身をプレビューできる機能 |
| Finder | 元の語: finder〈見つけるもの〉。macOS で、ファイルやフォルダを探して操作するための標準のアプリ |
| プレビュー | 元の語: preview〈前もって見ること〉。ファイルを開いて編集する前に、中身の見た目を確かめる表示 |
| CLI | 元の語: Command Line Interface の略（command〈命令〉＋ line〈行〉＋ interface〈接点〉）。文字のコマンドを打って操作する使い方や道具 |
| IDE | 元の語: Integrated Development Environment の略（統合された開発の環境）。コードの編集・実行・調べものを1つにまとめたアプリ |
| コーディングエージェント | 元の語: coding〈プログラムを書くこと〉＋ agent〈代理人〉。人の指示を受けて、コードを読んだり書いたり、コマンドを実行したりする AI |
| Mermaid | 元の語: mermaid〈人魚〉。文字で書いた記述から、流れ図などの図を描く仕組み |
| KaTeX | 数式を Web の画面にすばやく描くための仕組み。数式の組版で広く使われる TeX の書き方で数式を書く |
| Vega / Vega-Lite | 元の語: Vega〈こと座の1等星の名前〉。グラフの形を文字（JSON）で書いて描く仕組み。Vega-Lite はその簡単な書き方 |
| Graphviz | 元の語: graph〈グラフ。点と線のつながり〉＋ viz（visualization〈目に見える形にすること〉を縮めた語）。点と線のつながりを文字で書いて、図に描く道具 |
| GFM | 元の語: GitHub Flavored Markdown の略（flavored は「風味の付いた」）。GitHub で使われる、表やタスクリストなどを足した Markdown の書き方 |
| 構文ハイライト | 元の語: syntax highlighting（syntax〈文の組み立て〉＋ highlight〈目立たせる〉）。コードの命令・文字列・コメントなどを色分けして読みやすくする表示 |
| Homebrew / tap | 元の語: homebrew〈自家醸造〉、tap〈樽の栓〉。Homebrew は Mac で道具を入れるための仕組み。tap は、公式の一覧のほかに足せる、道具の入手先 |
| Gatekeeper | 元の語: gatekeeper〈門番〉。macOS で、インターネットから入れたアプリが信頼できるかを確かめる仕組み |
| 公証 | 元の語: notarization〈公証人が証明すること〉。Apple が配布前のアプリを自動で検査し、悪意のあるソフトが見つからなかったことを記録する手続き。品質や安全を全面的に保証するものではない |
| GPL-3.0 / デュアルライセンス | 元の語: GNU General Public License の略（general public は「一般の人々」）、dual〈二重の〉。GPL は、改変したものを配るときにソースの公開を求めるオープンソースのライセンス。デュアルライセンスは、同じソフトを2種類の条件から選べるようにした形 |
| 利用統計（仮名） | 元の語: usage statistics〈使われ方の統計〉、pseudonymous〈仮の名前の〉。アプリの使われ方を開発元へ送る情報。仮名は、本名ではない識別子で扱うこと |
