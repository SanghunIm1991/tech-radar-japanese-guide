# mise（開発ツールのバージョン・環境変数・タスクの管理）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Tools（開発や運用に使う道具） |
| リング | Adopt（前の版も同じリング。2026年4月の版でも Adopt） |
| 状態 | 下書き |

## 1. 一言でいうと

mise（「ミーズ」と読む）は、プロジェクトで使う**開発ツールのバージョン**、**環境変数**、**よく使うコマンド（タスク）**の3つを、1つの設定ファイル `mise.toml` にまとめて管理するコマンドラインの道具です（[mise 公式サイト](https://mise.jdx.dev/)）。名前は、料理で調理の前に材料や道具をそろえておく「ミザンプラス（mise en place）」から来ています（[About](https://mise.jdx.dev/about.html)）。

たとえるなら、フォルダに「この料理にはこの包丁とこの鍋」というメモを置くと、フォルダに入っただけで道具が持ち替えられる仕組みです。

## 2. 解決する課題（なぜ生まれたか）

必要なツールの版はプロジェクトごとに違います（Node.js の 20 系と 24 系など）。1台の PC に1つの版しか入れられないと、行き来するたびに入れ替えが要ります。

そこで言語ごとに**バージョンマネージャー**（同じツールの複数の版を並べて入れ、使う版を切り替える道具）が作られてきました。Node.js の nvm、Python の pyenv、Ruby の rbenv、Rust の rustup などです。ただ、言語ごとに別の道具を覚えるのは手間です。これをまとめる道具として **asdf** が広まり、mise は asdf の代わりとなる道具として、asdf の設定ファイル `.tool-versions` も読める形で作られました（[asdf からの移行](https://mise.jdx.dev/dev-tools/comparison-to-asdf.html)）。

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

設定ファイルは、全体用（ホームのフォルダの `~/.config/mise/config.toml`）と、プロジェクトのフォルダの `mise.toml` などがあり、今いるフォルダに近いものほど優先されます。個人だけの設定は `mise.local.toml` に書き、Git には入れない使い方が想定されています（[Configuration](https://mise.jdx.dev/configuration.html)）。

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
- **trust（信頼）**: 設定は環境変数の読み込みやタスクでコマンドを実行しうるため、信頼したプロジェクトの設定だけが使われます。`mise trust` のほか、`mise install` や `mise run` を実行してもそのプロジェクトは信頼済みになるので、他人のリポジトリでは、これらの前に設定とタスクを読みます（[Getting Started](https://mise.jdx.dev/getting-started.html)）。paranoid モードにすると、承認が内容のハッシュに結び付き、ファイルが書き換わると再び承認が要るようになります（[Paranoid](https://mise.jdx.dev/paranoid.html)）

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: アプリの中身ではなく、**開発環境と CI の土台**をそろえるための道具です。アプリが使うライブラリの依存関係（`package-lock.json` など）は対象外で、mise が扱うのは言語の処理系や CLI ツールの版です（[mise.lock](https://mise.jdx.dev/dev-tools/mise-lock.html)）。

**得られるもの**

- **道具の集約**: 言語ごとのバージョンマネージャー、環境変数の切り替え、タスクの実行を1つの道具と1つのファイルにまとめられる（[mise 公式サイト](https://mise.jdx.dev/)）
- **再現性**: `mise.toml` と `mise.lock` を Git に入れれば、開発者と CI が同じ版を使える（[Getting Started](https://mise.jdx.dev/getting-started.html)）
- **段階的な移行**: `.tool-versions` を読めるので、asdf からはプロジェクトごとに移せる（[asdf からの移行](https://mise.jdx.dev/dev-tools/comparison-to-asdf.html)）

**代わりに失うもの・注意点**

- **隔離はしない**: PC の上で直接ツールを切り替えるだけで、コンテナのように分けはしないので、OS のライブラリの違いまではそろいません（筆者の見立て）
- **入手先の信頼**: 検証の強さはバックエンドしだいです。プラグイン型の入手先は、その作者のコードを手元で動かすことになります（[asdf からの移行](https://mise.jdx.dev/dev-tools/comparison-to-asdf.html)、[Backends](https://mise.jdx.dev/dev-tools/backends/)）
- **設定ファイルがコードを動かす**: 環境変数の読み込みやタスクはコマンドを実行しうるため、知らないリポジトリでは `mise install` などの前に設定を読む
- **保守の体制**: MIT ライセンスのオープンソースで、作者の Jeff Dickey さんが中心となって保守しています（[About](https://mise.jdx.dev/about.html)）

## 5. 採用を判断するときの観点

- **チームの規模と言語の数**: 複数の言語や複数のサービスを抱えるチームほど、まとめる効果が大きい。1つの言語だけなら、その言語の標準の道具で足りる場合もあります（筆者の見立て）
- **OS**: macOS・Linux・Windows に対応します。Windows では asdf のプラグインが動かず（[Backends](https://mise.jdx.dev/dev-tools/backends/)）、GUI のアプリから使うには shim のフォルダを Path に足す必要があります（[Shims](https://mise.jdx.dev/dev-tools/shims.html)）
- **CI**: GitHub Actions 向けの公式のアクション（`jdx/mise-action`）があり、`mise.toml` のツールを入れてキャッシュします。CI ではシェルの activate は不要で、`mise run` や `mise exec` を使います（[Continuous Integration](https://mise.jdx.dev/continuous-integration.html)）
- **供給網の安全性**: 版を固定するなら `mise.lock` をコミットし、`--locked` で入れる。公式は、公開元のビルド済みのリリースを取るバックエンドを優先し、プラグインを最後にする順を勧めています。検証の有無はツールごとに `mise tool` で確かめます（[Backends](https://mise.jdx.dev/dev-tools/backends/)）
- **既存の道具との共存**: `.tool-versions` などを残したまま始められる。mise と asdf でツールの置き場所を共有しない点には注意します（[asdf からの移行](https://mise.jdx.dev/dev-tools/comparison-to-asdf.html)）

## 6. 関連技術・代替技術

- **asdf**: 複数の言語の版を1つの道具で切り替える、mise の先輩にあたる道具。設定ファイル `.tool-versions` は mise も読めます（[asdf からの移行](https://mise.jdx.dev/dev-tools/comparison-to-asdf.html)）。Radar のこの項目のページでは、過去の版の記載から asdf の項目へリンクが張られています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/mise)）
- **direnv**: フォルダに入ると環境変数を切り替える道具。mise の `[env]` と役割が重なります（筆者の見立て）
- **Dev Container や Nix**: OS のライブラリまで含めて環境を丸ごとそろえる方法。隔離や再現性は強い反面、準備の手間は大きくなります（筆者の見立て）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Adopt**（前の版も同じリング。2026年4月の版でも Adopt）
- 過去の版: 2024年10月の版で Assess として初めて載り、2026年4月の版で Adopt に移った（Radar のページの推移による）
- Thoughtworks の評価の要旨: チームの開発環境をそろえるときの標準の選択肢として勧めています。主な理由は、3つの役割を宣言的な1つのファイルにまとめられることと、署名などによる供給網の安全の仕組みを備えていることです
- Radar のページ: https://www.thoughtworks.com/en-us/radar/tools/mise

## 8. 深掘りするときの一次情報

- [mise 公式サイト](https://mise.jdx.dev/)と [Getting Started](https://mise.jdx.dev/getting-started.html)
- [Shims（activate と shim の違い）](https://mise.jdx.dev/dev-tools/shims.html)
- [Backends](https://mise.jdx.dev/dev-tools/backends/) と [Security](https://mise.jdx.dev/security.html)
- [GitHub のリポジトリ（jdx/mise）](https://github.com/jdx/mise)

## 9. 確認日

- 内容の確認日: 2026-11-11
- Radar の評価の確認日: 2026-11-11

## 用語集

| 用語 | 説明 |
|---|---|
| バージョンマネージャー | 同じツールの複数の版を1台の PC に並べて入れ、プロジェクトや場面に応じて使う版を切り替える道具 |
| PATH | コマンド名を打ったときに、その実行ファイルを探しに行くフォルダの一覧を持つ環境変数。先に書かれたフォルダが優先される |
| shim | 本物のコマンドの代わりに呼ばれ、条件に応じて本物の版を選んで起動する小さな代理のプログラム |
| チェックサム | ファイルの内容から計算した短い値。手元で計算し直して一致すれば、途中で壊れたり書き換えられたりしていないと分かる。比べる元の値が信頼できる入手先のものであることが前提 |
| 来歴（provenance） | ソフトウェアが、どのソースから、どこで、どうビルドされたかを示す証明。偽物の配布を見分けるのに使う |
| ロックファイル | 「24 系」のような幅のある指定を、実際に使った正確な版に固定して記録したファイル。全員が同じ版を入れられる |
| 供給網（サプライチェーン）の安全 | 自分が使うソフトウェアや部品が、作られてから手元に届くまでに改ざんされていないことを守る考え方 |
