# mise（開発ツールのバージョン・環境変数・タスクの管理）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Tools（開発や運用に使う道具） |
| リング | Adopt（前の版も同じリング。2026年4月の版でも Adopt） |
| 状態 | 下書き |
| 図解ページ | [図と動きで見る図解ページ](mise-intro.html)（この解説の内容を図と動きで見せるページ。サイトで表示。GitHub のリポジトリではソースが表示されます） |

## 1. 一言でいうと

**名前の手がかり**: mise〈フランス語の mise en place（ミザンプラス）。料理の前に、材料や道具をそろえて配置しておくこと〉から（[About](https://mise.jdx.dev/about.html)）

mise（「ミーズ」と読む）は、プロジェクトで使う**開発ツールのバージョン**、**環境変数**、**よく使うコマンド（タスク）**の3つを、1つの設定ファイル `mise.toml` にまとめて管理するコマンドラインの道具です（[mise 公式サイト](https://mise.jdx.dev/)）。

たとえるなら、フォルダに「この料理にはこの包丁とこの鍋」というメモを置くと、フォルダに入っただけで道具が持ち替えられる仕組みです。

## 2. 解決する課題（なぜ生まれたか）

必要なツールの版はプロジェクトごとに違います（Node.js の 20 系と 24 系など）。1台の PC に1つの版しか入れられないと、行き来するたびに入れ替えが要ります。

そこで言語ごとに**バージョンマネージャー**（同じツールの複数の版を並べて入れ、使う版を切り替える道具）が作られてきました。Node.js の nvm、Python の pyenv、Ruby の rbenv、Rust の rustup などです。ただ、言語ごとに別の道具を覚えるのは手間です。これをまとめる道具として **asdf** が広まりました。mise は asdf の設定ファイル `.tool-versions` を読め、asdf のプラグインも動かせます（[asdf からの移行](https://mise.jdx.dev/dev-tools/comparison-to-asdf.html)）。Thoughtworks は、mise を asdf の高速な代わりとして始まった道具と紹介しています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/mise)）。

チームでは「手元では動くのに CI（変更のたびに自動でビルドやテストを行う仕組み）では動かない」といった問題も起きます。使う版を設定ファイルに書いて Git で共有すれば、全員と CI が同じ版を使えます（筆者の見立て）。

## 3. 仕組みの概要

### 3.1 3つの役割

中心となる役割は次の3つです（[mise 公式サイト](https://mise.jdx.dev/)。公式サイトは、マシン全体の準備をする Bootstrap も挙げています）。

| 役割 | 内容 | 設定の例 |
|---|---|---|
| Dev tools（ツール） | Node.js・Python・Go・Terraform など多数のツールを入れ、フォルダごとに版を切り替える | `[tools]` に `node = "24"` |
| Environments（環境変数） | フォルダに入ったときに環境変数を設定する。`.env` ファイルも読める | `[env]` に `_.file = ".env"` |
| Tasks（タスク） | ビルドやテストのコマンドに名前を付け、`mise run test` のように実行する。依存関係も書ける | `[tasks.test]` に `run = "..."` |

`"24"` は「24 で始まる版」という意味で、正確な版を固定したいときは後で述べるロックファイルを使います（[Getting Started](https://mise.jdx.dev/getting-started.html)）。

### 3.2 設定ファイルの重なり

設定ファイルは、全体用（`~/.config/mise/config.toml`）とプロジェクトの `mise.toml` などがあり、今いるフォルダに近いものほど優先されます。個人だけの設定は、Git に入れない `mise.local.toml` に書きます（[Configuration](https://mise.jdx.dev/configuration.html)）。

### 3.3 版の切り替え方: activate と shim

「`node` と打ったときに、どの版の Node.js が動くか」は、**PATH**（コマンドを探しに行くフォルダの一覧を持つ環境変数）で決まります。mise は2つの方法で PATH を切り替えます（[Shims](https://mise.jdx.dev/dev-tools/shims.html)）。

- **activate**: シェル（コマンドを打つ画面のプログラム）にフックを入れ、プロンプトを出すときやフォルダを移動したときに PATH と環境変数を書き換えます。ツールが直接 PATH に載るので速い反面、シェルの起動時の設定を読まないエディタや GUI のアプリには効きません
- **shim**: `node` などの名前の小さな代理の実行ファイルを置き、呼ばれるたびに mise が今のフォルダの設定を読んで、正しい版を起動します。エディタやスクリプトからも使えますが、毎回の解決のぶん遅くなり、`[env]` の環境変数はシェルには設定されません

asdf は shim を毎回経由する方式でしたが、mise は activate で PATH を直接書き換えられる点が違います（[asdf からの移行](https://mise.jdx.dev/dev-tools/comparison-to-asdf.html)）。

### 3.4 ツールの入手先（バックエンド）と検証

mise は、ツールの入手先を**バックエンド**と呼ぶ部品に分けています。GitHub のリリースから取るもの、aqua の登録簿（ツールごとの入手方法をまとめたもの）をもとに取るもの、npm・PyPI・cargo などの言語のパッケージの仕組みを使うもの、asdf のプラグインを使うものなどがあります（[Backends](https://mise.jdx.dev/dev-tools/backends/)）。

入手したファイルの**チェックサム**（ファイルが壊れたり書き換えられたりしていないかを確かめる値）や**署名**、**来歴**（どこでどうビルドされたかの証明）を確かめるかどうかは、バックエンドとツールごとに違い、公開元がそれらを用意している場合に限られます。`mise tool <名前>` で、そのツールにどの検証が効くかを表示できます（[Security](https://mise.jdx.dev/security.html)）。asdf のプラグインはシェルスクリプトで、入れるときに手元で実行されるため、プラグインの作者も信頼する必要があります（[asdf からの移行](https://mise.jdx.dev/dev-tools/comparison-to-asdf.html)）。

### 3.5 ロックファイルと信頼の確認

- **ロックファイル**: `mise lock` で `mise.lock` を作ると、解決した正確な版が記録され、バックエンドによってはダウンロード元の URL・チェックサム・来歴も記録されます。`mise install --locked` は、ロックファイルと合わないと失敗します（[mise.lock](https://mise.jdx.dev/dev-tools/mise-lock.html)）
- **trust（信頼）**: 環境変数・フック・設定（settings）など、コードを実行したり環境を変えたりしうる部分は、信頼したプロジェクトでだけ読み込まれます（ツールの版の指定とタスクの定義は、信頼しなくても読まれます）（[Security](https://mise.jdx.dev/security.html)）。`mise trust` のほか、`mise install`・`mise exec`・`mise run` を実行してもそのプロジェクトは信頼済みになるので、他人のリポジトリでは、これらの前に設定とタスクを読みます（[Getting Started](https://mise.jdx.dev/getting-started.html)）。paranoid モードにすると、承認が内容のハッシュに結び付き、ファイルが書き換わると再び承認が要るようになります（[Paranoid](https://mise.jdx.dev/paranoid.html)）

### 3.6 第2節の課題への答え

- **プロジェクトごとに必要な版が違い、行き来するたびに入れ替えが要る**: mise は同じツールの複数の版を並べて入れておき、今いるフォルダの `mise.toml` に書かれた版を使います。activate ならフォルダを移動したときに、shim なら呼ばれるたびに、PATH が切り替わるので、入れ替えの作業は要りません（[Shims](https://mise.jdx.dev/dev-tools/shims.html)、[Getting Started](https://mise.jdx.dev/getting-started.html)）
- **言語ごとに別のバージョンマネージャーを覚えるのは手間**: 1つの道具が、バックエンドを通して多くの言語や CLI ツールを扱います（[Backends](https://mise.jdx.dev/dev-tools/backends/)）。asdf の `.tool-versions` も読めるので、プロジェクトごとに移せます（[asdf からの移行](https://mise.jdx.dev/dev-tools/comparison-to-asdf.html)）。Thoughtworks も今回の版で、既存の言語ごとの道具と並べて使えるので、一度にすべてを移す必要はないとしています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/mise)）
- **手元では動くのに CI では動かない**: `mise.toml` と `mise.lock` を Git で共有し、`mise install --locked` で入れれば、ロックファイルと違う版は入りません（[mise.lock](https://mise.jdx.dev/dev-tools/mise-lock.html)）。CI では公式のアクション（`jdx/mise-action`）が `mise.toml` のツールを入れ、`mise.lock` があれば `--locked` で入れます（[Continuous Integration](https://mise.jdx.dev/continuous-integration.html)）。ただし、そろうのはツールの版までで、OS のライブラリの違いは残ります（第4節）

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: アプリの中身ではなく、**開発環境と CI の土台**をそろえるための道具です。アプリが使うライブラリの依存関係（`package-lock.json` など）は対象外で、mise が扱うのは言語の処理系や CLI ツールの版です（[mise.lock](https://mise.jdx.dev/dev-tools/mise-lock.html)）。

**得られるもの**

- **道具の集約**: 言語ごとのバージョンマネージャー、環境変数の切り替え、タスクの実行を1つの道具と1つのファイルにまとめられる（[mise 公式サイト](https://mise.jdx.dev/)）
- **再現性**: `mise.toml` と `mise.lock` を Git に入れれば、開発者と CI が同じ版を使える（[Getting Started](https://mise.jdx.dev/getting-started.html)、[mise.lock](https://mise.jdx.dev/dev-tools/mise-lock.html)）
- **段階的な移行**: `.tool-versions` を読めるので、asdf からはプロジェクトごとに移せる（[asdf からの移行](https://mise.jdx.dev/dev-tools/comparison-to-asdf.html)）

**代わりに失うもの・注意点**

- **隔離はしない**: PC の上で直接ツールを切り替えるだけで、コンテナのように分けはしないので、OS のライブラリの違いまではそろいません（筆者の見立て）
- **入手先の信頼**: 検証の強さはバックエンドしだいです。プラグイン型の入手先は、その作者のコードを手元で動かすことになります（[asdf からの移行](https://mise.jdx.dev/dev-tools/comparison-to-asdf.html)、[Backends](https://mise.jdx.dev/dev-tools/backends/)）
- **設定ファイルがコードを動かす**: 環境変数の読み込みやタスクはコマンドを実行しうるため、知らないリポジトリでは `mise install` などの前に設定を読む（3.5 節を参照）
- **保守の体制**: MIT ライセンスのオープンソースで、作者の Jeff Dickey さんが中心となって保守しています（[About](https://mise.jdx.dev/about.html)）。作者1人への依存度が高いので、長く使う前提なら開発が続いているかを見ておきます（筆者の見立て）

## 5. 採用を判断するときの観点

- **チームの規模と言語の数**: 複数の言語や複数のサービスを抱えるチームほど、まとめる効果が大きい。1つの言語だけなら、その言語の標準の道具で足りる場合もあります（筆者の見立て）
- **OS**: macOS・Linux・Windows に対応します（[mise 公式サイト](https://mise.jdx.dev/)）。Windows では asdf のプラグインが動かず（[Backends](https://mise.jdx.dev/dev-tools/backends/)）、GUI のアプリから使うには shim のフォルダを Path に足す必要があります（[Shims](https://mise.jdx.dev/dev-tools/shims.html)）
- **CI**: GitHub Actions 向けの公式のアクション（`jdx/mise-action`）があり、`mise.toml` のツールを入れてキャッシュします。CI ではシェルの activate は不要で、`mise run` や `mise exec` を使います（[Continuous Integration](https://mise.jdx.dev/continuous-integration.html)）
- **供給網の安全性**: 版を固定するなら `mise.lock` をコミットし、`--locked` で入れる。公式は、公開元のビルド済みのリリースを取るバックエンドを優先し、プラグインを最後にする順を勧めています。検証の有無はツールごとに `mise tool` で確かめます（[Backends](https://mise.jdx.dev/dev-tools/backends/)、[Security](https://mise.jdx.dev/security.html)）
- **既存の道具との共存**: `.tool-versions` などを残したまま始められる。mise と asdf でツールの置き場所を共有しない点には注意します（[asdf からの移行](https://mise.jdx.dev/dev-tools/comparison-to-asdf.html)）

## 6. 関連技術・代替技術

- **asdf**: 複数の言語の版を1つの道具で切り替える、mise の先輩にあたる道具。設定ファイル `.tool-versions` は mise も読めます（[asdf からの移行](https://mise.jdx.dev/dev-tools/comparison-to-asdf.html)）。Radar のこの項目のページでは、過去の版の記載から asdf の項目へリンクが張られています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/mise)）
- **direnv**: フォルダに入ると環境変数を切り替える道具。mise の `[env]` と役割が重なります（筆者の見立て）
- **Dev Container や Nix**: OS のライブラリまで含めて環境を丸ごとそろえる方法。隔離や再現性は強い反面、準備の手間は大きくなります（筆者の見立て）
- **Cosign**: ソフトウェアに署名し、署名を確かめる道具。Thoughtworks は、mise の供給網の安全の仕組みの例として、Cosign と GitHub の artifact attestations（ビルドの来歴の証明）を挙げています。Radar のこの項目のページでは、関連の項目の1つとして Cosign（Tools・Assess。2021年10月の版）が挙がっています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/mise)）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Adopt**（前の版も同じリング。2026年4月の版でも Adopt）
- 過去の版: 2024年10月の版で Assess として初めて載り、2026年4月の版で Adopt に移った（Radar のページの推移による）
- Thoughtworks の評価の要旨: チームの開発環境をそろえるときの標準の選択肢として勧めています。主な理由は、3つの役割を宣言的な1つのファイルにまとめられることと、署名などによる供給網の安全の仕組みを備えていることです。複数の言語のサービスを抱えるチームで特に役立ち、既存の道具と並べて使えるので一度に移さなくてよいとしています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/tools/mise

## 8. 深掘りするときの一次情報

- [mise 公式サイト](https://mise.jdx.dev/)と [Getting Started](https://mise.jdx.dev/getting-started.html)
- [Shims（activate と shim の違い）](https://mise.jdx.dev/dev-tools/shims.html)
- [Backends](https://mise.jdx.dev/dev-tools/backends/) と [Security](https://mise.jdx.dev/security.html)
- [GitHub のリポジトリ（jdx/mise）](https://github.com/jdx/mise)

## 9. 確認日

- 内容の確認日: 2026-11-15（2026-11-11 に書き、2026-11-15 に名前の手がかり・課題ごとの答え・元の語を足して、全体を点検し直した）
- Radar の評価の確認日: 2026-11-15

## 用語集

用語の説明の冒頭の「元の語」は、覚えるための手がかりとして、元の語（英単語の意味・略語の元の綴り・名前の元になった語）を短く示したものです。

| 用語 | 説明 |
|---|---|
| バージョンマネージャー | 元の語: version〈版〉＋ manager〈管理するもの〉。同じツールの複数の版を1台の PC に並べて入れ、プロジェクトや場面に応じて使う版を切り替える道具 |
| 環境変数 | 元の語: environment variable〈環境の変数〉。プログラムの外から設定して、プログラムに渡す名前つきの値。接続先や動き方の切り替えに使う |
| TOML | 元の語: Tom's Obvious, Minimal Language の略（Tom の、分かりやすく最小限の言語）。設定ファイルを書くための書式。`mise.toml` はこの書式で書く（[TOML](https://toml.io/en/)） |
| CI | 元の語: Continuous Integration の略（継続的な統合）。変更のたびに、自動でビルドやテストを行う仕組み |
| PATH | 元の語: path〈道・経路〉。コマンド名を打ったときに、その実行ファイルを探しに行くフォルダの一覧を持つ環境変数。先に書かれたフォルダが優先される |
| activate | 元の語: activate〈有効にする〉。mise では、シェルにフックを入れて、フォルダの移動などに合わせて PATH と環境変数を書き換える方式 |
| shim | 元の語: shim〈すき間に挟んで高さを合わせる薄い板〉。本物のコマンドの代わりに呼ばれ、条件に応じて本物の版を選んで起動する小さな代理のプログラム |
| チェックサム | 元の語: check〈確かめる〉＋ sum〈合計〉。ファイルの内容から計算した短い値。手元で計算し直して一致すれば、途中で壊れたり書き換えられたりしていないと分かる。比べる元の値が信頼できる入手先のものであることが前提 |
| 来歴（provenance） | 元の語: provenance〈出どころ・由来〉。ソフトウェアが、どのソースから、どこで、どうビルドされたかを示す証明。偽物の配布を見分けるのに使う |
| ロックファイル | 元の語: lock〈錠をかける・固定する〉＋ file〈ファイル〉。「24 系」のような幅のある指定を、実際に使った正確な版に固定して記録したファイル。全員が同じ版を入れられる |
| 供給網（サプライチェーン）の安全 | 元の語: supply chain〈供給の鎖〉。自分が使うソフトウェアや部品が、作られてから手元に届くまでに改ざんされていないことを守る考え方 |
