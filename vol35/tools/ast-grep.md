# ast-grep（コードの構造で検索・検査・書き換えをする道具）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Tools（開発や運用に使う道具） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

ast-grep は、ソースコードを**文字の並びとしてではなく、構文の木（AST）として**検索し、検査（lint）し、書き換えるためのコマンドラインの道具です。Rust で書かれたオープンソースで、ライセンスは MIT です（[GitHub: ast-grep/ast-grep](https://github.com/ast-grep/ast-grep)）。

ふつうの grep が「この文字列を含む行」を探すのに対し、ast-grep は「関数の呼び出しで、引数が1つのもの」のように、コードの形で探します。たとえるなら、文章を単語で検索するのではなく、「主語＋述語」の文の形で探すようなものです（筆者の見立て）。

## 2. 解決する課題（なぜ生まれたか）

コードを検索したり、まとめて書き換えたりするとき、文字列や正規表現（文字のパターンを表す記法）で探すと、次のような問題が起きます（筆者の見立て）。

- 空白や改行の入れ方が違うだけで、同じ意味のコードを見逃す
- コメントや文字列の中にある、たまたま同じ文字を拾ってしまう
- 「引数が何個でもよい」「同じ変数が2回出る」のような、構造の条件を書きにくい

IDE（統合開発環境）の多くは構文を理解した検索を持っていますが、自分で決めた検査のルールや、まとめての書き換えまでは手軽にできません。ast-grep は、この点を補う道具です（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/ast-grep)も、独自の lint のルールと書き換えを ast-grep の違いに挙げています）。

さらに最近は、コーディングエージェントが書いたコードを、決まったルールで機械的に確かめる必要が出てきました。Radar によると、あるチームは、エージェントの環境（ハーネス）の一部として ast-grep の独自ルールでエージェントのコードを検査し、結果をエージェントに返していました（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/ast-grep)）。

## 3. 仕組みの概要

**構文の解析**: 構文の解析には tree-sitter（多くの言語に対応した、速い構文解析の部品）を使います（[GitHub: ast-grep/ast-grep](https://github.com/ast-grep/ast-grep)）。対応する言語は、Bash、C、C++、C#、Go、Java、JavaScript、TypeScript、Kotlin、Python、Ruby、Rust、Swift、YAML など28です（[ast-grep: 対応言語の一覧](https://ast-grep.github.io/reference/languages.html)）。

**パターンの書き方**（[ast-grep: Pattern Syntax](https://ast-grep.github.io/guide/pattern-syntax.html)）

パターンは、ふつうのコードの一部を「メタ変数」（何にでも当てはまる穴）に置き換えて書きます。

| 書き方 | 意味 | 例 |
|---|---|---|
| `$A` | 構文の要素1つに当てはまる | `console.log($MSG)` は引数が1つの呼び出しに当てはまり、引数0個や2個には当てはまらない |
| `$$$` | 0個以上の要素に当てはまる | `foo($$$ARGS)` は引数の数を問わない |
| 同じ名前を2回 | 同じ中身に当てはまる | `$A == $A` は `a == a` に当てはまり、`a == b` には当てはまらない |

パターン自体が、tree-sitter で解析できる正しいコードである必要があります。

**コマンドの例**（[GitHub: ast-grep/ast-grep](https://github.com/ast-grep/ast-grep)、[ast-grep の公式サイト](https://ast-grep.github.io/)）

- 検索と置き換え: `ast-grep -p '$A && $A()' -l ts -r '$A?.()'`（TypeScript で、`a && a()` の形を `a?.()` に書き換える）
- ルールでの検査: `ast-grep scan`

**YAML のルール**（[ast-grep: Rule Essentials](https://ast-grep.github.io/guide/rule-config.html)）

検査のルールは YAML のファイルに書きます。最小のルールには、ルールの名前（`id`）、対象の言語（`language`）、条件（`rule`）の3つを書きます。条件は次の3種類を組み合わせます。

- **基本**: パターン（`pattern`）、構文の種類（`kind`）、正規表現（`regex`）
- **位置の関係**: 〜の中にある（`inside`）、〜を含む（`has`）、〜の後にある（`follows`）、〜の前にある（`precedes`）
- **組み合わせ**: すべて（`all`）、どれか（`any`）、否定（`not`）、別に定義したルールの参照（`matches`）

たとえば「`Promise.all(...)` の引数の中に `await` が含まれている」というルールを、`pattern` と `has` の組み合わせで書けます。

**そのほか**: Node.js から使うライブラリ（`@ast-grep/napi`）があり、プログラムの中から使えます。インストールは npm・pip・Homebrew・cargo などでできます（[GitHub: ast-grep/ast-grep](https://github.com/ast-grep/ast-grep)、[ast-grep の公式サイト](https://ast-grep.github.io/)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: 開発の流れの中の「コードの機械的な確認と一括変更」の道具です。ESLint のような言語ごとの lint の道具と、grep のような汎用の検索の道具の中間にあります（筆者の見立て）。

**得られるもの**

- **軽さ**: コードの索引（インデックス）や language server（エディタにコードの意味を教えるサーバー）を前もって用意しなくても使える。大きなコードベースで重い仕組みを入れる前の、手軽な第一歩になる（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/ast-grep)）
- **人とエージェントで同じ道具**: 開発者も同じ検索・検査・書き換えをそのまま実行できる。Thoughtworks は、エージェント向けのコード理解の道具（Code intelligence as agentic tooling）の多くと違う点として、これを挙げている（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/ast-grep)）
- **多言語**: 1つの道具で、多くの言語に同じ書き方で使える（[ast-grep の公式サイト](https://ast-grep.github.io/)）

**代わりに失うもの・注意点**

- **意味までは分からない**: 見ているのは構文の形で、型や、どの関数が実際に呼ばれるかまでは分からない。そこまで要るなら、言語ごとの静的解析の道具が要る（筆者の見立て）
- **パターンの学習**: メタ変数や YAML のルールの書き方を覚える必要がある。解析しにくいパターンでは、文脈を与える書き方の工夫が要る（[ast-grep: Pattern Syntax](https://ast-grep.github.io/guide/pattern-syntax.html)）
- **ルールの保守**: 独自ルールが増えると、ルール自体のテストや見直しが要る（筆者の見立て）

## 5. 採用を判断するときの観点

- **用途**: 一括の書き換え（ライブラリの更新に伴う呼び出しの変更など）や、チーム独自の決まりの検査があるか（筆者の見立て）
- **既存の lint との分担**: 言語ごとの lint の道具で書けるルールはそちらに任せ、書きにくいものや多言語にまたがるものを ast-grep に任せる（筆者の見立て）
- **エージェントとの組み合わせ**: エージェントの作業の後に `ast-grep scan` を走らせ、違反をエージェントに返して直させる流れを作れるか（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/ast-grep)のチームの例に基づく筆者の見立て）
- **CI への組み込み**: 検査のルールを CI で走らせ、決まりに反するコードの取り込みを止める（筆者の見立て）
- **言語**: 使っている言語が対応の一覧に入っているか（[ast-grep: 対応言語の一覧](https://ast-grep.github.io/reference/languages.html)）
- **小さく始める**: 最初は、よく指摘される決まりを2〜3個だけルールにし、誤検知の多さを見ながら増やす。ルールのファイルはコードと同じリポジトリに置き、レビューの対象にする（筆者の見立て）

## 6. 関連技術・代替技術

- **Code intelligence as agentic tooling**、**Comby**: [Radar の ast-grep のページ](https://www.thoughtworks.com/en-us/radar/tools/ast-grep)の関連の項目に挙がっています。Comby は、Radar が2021年に取り上げた、構造を考えた検索と書き換えの道具です
- **Graph-grounded codebase reasoning**: コードの関係をグラフにして、エージェントに推論させる手法。ast-grep より重いが、コードどうしの関係まで扱える（[解説](../techniques/graph-grounded-codebase-reasoning.md)。筆者の見立て）
- **代替: 言語ごとの lint の道具**: ESLint などは、1つの言語の中で、より深い検査ができる（筆者の見立て）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: 索引や language server が要らない手軽さと、開発者とエージェントが同じ検索・検査・書き換えを使える点を評価しています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/tools/ast-grep

## 8. 深掘りするときの一次情報

- [GitHub: ast-grep/ast-grep](https://github.com/ast-grep/ast-grep)
- [ast-grep の公式サイト](https://ast-grep.github.io/)
- [ast-grep: Pattern Syntax](https://ast-grep.github.io/guide/pattern-syntax.html)
- [ast-grep: Rule Essentials](https://ast-grep.github.io/guide/rule-config.html)
- [ast-grep: 対応言語の一覧](https://ast-grep.github.io/reference/languages.html)

## 9. 確認日

- 内容の確認日: 2026-11-13
- Radar の評価の確認日: 2026-11-13

## 用語集

| 用語 | 説明 |
|---|---|
| AST（抽象構文木） | ソースコードを、文法の構造に従って木の形に表したもの。空白やコメントの違いに左右されない |
| grep | ファイルの中から、指定した文字列や正規表現に合う行を探す道具 |
| 正規表現 | 「数字が3つ続く」のような、文字のパターンを表す記法 |
| lint | コードを機械的に調べ、誤りや決まりへの違反を見つけること。その道具 |
| tree-sitter | 多くのプログラミング言語に対応した、速い構文解析の部品 |
| メタ変数 | パターンの中で、何にでも当てはまる穴として使う名前 |
| YAML | 設定を、字下げで構造を表して書く文字の形式 |
| IDE | コードの編集・実行・デバッグなどをまとめて行える開発用のアプリ |
| language server | エディタに、補完や定義への移動などのコードの意味の情報を提供するサーバー |
| ハーネス | エージェントの周りの仕組み。ここでは、エージェントのコードを検査して結果を返す仕組みを含む |
| CI | 変更のたびに、ビルドや検査を自動で走らせる仕組み |
| 静的解析 | プログラムを実行せずに、ソースコードを調べて問題を見つけること |
