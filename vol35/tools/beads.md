# Beads（コーディングエージェントのための課題管理）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Tools（開発や運用に使う道具） |
| リング | Trial（Assess から移動） |
| 状態 | 下書き |

## 1. 一言でいうと

Beads は、**AI のコーディングエージェントが使うための課題管理（イシュートラッカー）**です。コマンド名は `bd` で、課題とその依存関係（どの課題が終わらないと、どの課題に着手できないか）を、エージェントが読み書きできる形で保存します。README は「AI エージェントのための、分散型のグラフ型イシュートラッカー」と説明しています。ライセンスは MIT です（[GitHub: gastownhall/beads](https://github.com/gastownhall/beads)）。

たとえるなら、エージェントが毎回の作業でメモ用紙に書いては捨てていた「やることリスト」を、チーム共有の作業ボードに置き換えるようなものです（筆者の見立て）。

## 2. 解決する課題（なぜ生まれたか）

コーディングエージェントは、長い作業の計画を Markdown の TODO リストなどに書きながら進めることがよくあります。しかし、次のような問題があります（筆者の見立て）。

- **記憶が切れる**: 会話が長くなって要約されたり、新しい会話に切り替わったりすると、計画や途中で見つけた課題が失われやすい
- **依存関係が書けない**: ただの箇条書きでは、「これが終わるまで、あれには着手できない」という関係を表しにくい
- **複数のエージェントがぶつかる**: 同時に動くエージェントが、同じ課題に手を付けてしまう

Beads は、課題と依存関係を構造のあるデータとしてコードの近くに保存し、会話をまたいでも計画が残るようにします。各エージェントは、ほかとぶつからずに次の空いている課題を取れます（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/beads)）。

## 3. 仕組みの概要

**データの保存先**（[GitHub: gastownhall/beads](https://github.com/gastownhall/beads)「Storage Modes」）

データは Dolt に保存します。Dolt は、Git と同じように分岐（branch）・統合（merge）・送受信（push・pull）ができる SQL のデータベースで、「Git はファイルを版管理し、Dolt は表を版管理する」と説明されています（[GitHub: dolthub/dolt](https://github.com/dolthub/dolt)）。

- 既定では、リポジトリの中の `.beads/` にデータベースを埋め込みます。書き込みは1つのプロセスからに限られます
- 複数から書き込むときは、外部の Dolt のサーバーにつなぐモードを使います
- `bd dolt push` と `bd dolt pull` で、Git のリモートを通じてデータを共有します

**課題の ID と依存関係**（[GitHub: gastownhall/beads](https://github.com/gastownhall/beads)、[Beads: Dependencies](https://github.com/gastownhall/beads/blob/main/docs/core-concepts/dependencies.md)）

課題の ID は `bd-a1b2` のようなハッシュの形で、`bd-a3f8.1` のように階層も表せます。ハッシュの ID にすることで、複数のエージェントやブランチで同時に課題を作っても番号がぶつかりにくくしています。依存関係には、着手を止める種類（`blocks` など）と、関連を記録するだけの種類（`related`、`discovered-from` など）があります。プルリクエストの取り込みや CI の成功、人の承認などを「待つ条件（ゲート）」として組み込むこともできます。

**主なコマンド**（[GitHub: gastownhall/beads](https://github.com/gastownhall/beads)）

| コマンド | 役割 |
|---|---|
| `bd init` | リポジトリで Beads を使い始める |
| `bd create "題名" -p 0` | 課題を作る（`-p` は優先度） |
| `bd ready` | 着手を止めている依存先がすべて閉じた、着手できる課題を一覧する |
| `bd update <id> --claim` | 課題を自分の担当にして「作業中」にする。この2つを一度に行うので、ほかのエージェントと取り合いにならない |
| `bd close <id>` | 課題を閉じる |
| `bd remember` / `bd prime` | `bd remember` で覚えておく事実を保存し、`bd prime` で作業の流れの説明と保存した事実をエージェント向けに出力する |

**エージェントとの連携**（[GitHub: gastownhall/beads](https://github.com/gastownhall/beads)「Quick Start」）

`bd init` は、既定で AGENTS.md（エージェントへの指示をまとめたファイル）を作るか更新し、Claude Code・Codex・Cursor 向けの連携も入れます。書き込む指示の中身は「`bd prime` を実行する」「Markdown の TODO リストで課題を管理しない」などです。書き込みを止めるオプション（`--skip-agents`）もあります。

**外部の課題管理との同期**: Radar は、Linear・Jira・GitHub Issues と双方向に同期できると説明しています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/beads)）。人が見る課題と、エージェントが扱う課題をつなげられるということです。

**版の上げ方の注意**（[GitHub: gastownhall/beads](https://github.com/gastownhall/beads)「Upgrading?」「Schema Version Guard」）

新しい版がデータベースの構造（スキーマ）を移行したあとに古い版で開くと、`bd` はエラーで止まります。止めずに開く上書きの方法もありますが、移行が安全だと分かっているときに限るとされています。共有のデータベースでは、移行を1台だけで行い、ほかの端末は新しい版を入れてから取り込む手順が示されています。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: エージェントの「作業の記憶」と「調整の場」です。人のチームの計画は Linear などの課題管理で行い、Beads はエージェントが作る細かい作業を扱う、という分担で補い合うと Radar は説明しています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/beads)）。

**得られるもの**

- **会話をまたぐ計画**: 課題がデータとして残るので、新しい会話でも続きから始められる（README）
- **依存関係に基づく順番**: `bd ready` で、今着手できる課題だけを出せる（README）
- **取り合いの防止**: `--claim` で、複数のエージェントが同じ課題に手を付けにくい（README）

**代わりに失うもの・注意点**

- **重くなった**: Radar によると、1.0 に達し、保存先が Dolt だけになり、HTTP の API や作業の貸し出し（work leases）などが加わって、初期の軽い道具より重くなった（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/beads)）
- **版をそろえる必要**: 版の違う `bd` は同じデータベースを共有できないので、共有するデータベースごとに版を固定する（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/beads)。README のスキーマの注意とも合う）
- **AGENTS.md への書き込み**: Beads が AGENTS.md に書き込む指示を確かめる（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/beads)）。ほかの指示とぶつからないかも見る（筆者の見立て）
- **もう1つの保存先**: 課題がコードとは別の Dolt のデータとして増えるので、バックアップや共有の方法を決める必要がある（筆者の見立て）

## 5. 採用を判断するときの観点

- **作業の長さ**: エージェントに、会話をまたぐ長い作業や、複数の段階に分かれる作業を任せているか。短い作業だけなら要らない（筆者の見立て）
- **複数のエージェント**: 同時に複数のエージェントを動かすなら、取り合いの防止が効く（筆者の見立て）
- **人の課題管理との分担**: 人が見る計画は既存の課題管理に残し、Beads はエージェントの細かい作業に限る（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/beads)の分担に基づく筆者の見立て）
- **版の管理**: チームで使う `bd` の版をそろえ、上げるときは手順に従う（README）
- **リポジトリを汚したくないとき**: ファイルをコミットせずに使うモード（`--stealth`）もある（README）

## 6. 関連技術・代替技術

- **Linear**、**AGENTS.md**: [Radar の Beads のページ](https://www.thoughtworks.com/en-us/radar/tools/beads)の関連の項目に挙がっています
- **Agent-maintained knowledge bases**: エージェントが知識をまとめて保守する手法。会話をまたぐ記憶という点で近い（[解説](../techniques/agent-maintained-knowledge-bases.md)。筆者の見立て）
- **ticket**、**tracer**: 2026年4月の版で、Radar が同じ種類の初期の道具として名前を挙げていました（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/beads)）
- **代替: Markdown の計画ファイル**: 作業が短く、エージェントが1つなら、リポジトリに置いた計画のファイルで足りることも多い（筆者の見立て）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（Assess から移動）
- 過去の版: 2026年4月の版で Assess として載りました
- Thoughtworks の評価の要旨: Thoughtworks の多くの開発者が、コーディングエージェントと一緒に当たり前に使うようになったとして Trial に上げました。一方で、重くなったことと、版の違うクライアントが同じデータを共有できないことに注意を促しています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/tools/beads

## 8. 深掘りするときの一次情報

- [GitHub: gastownhall/beads](https://github.com/gastownhall/beads)
- [Beads: Dependencies](https://github.com/gastownhall/beads/blob/main/docs/core-concepts/dependencies.md)
- [GitHub: dolthub/dolt](https://github.com/dolthub/dolt)

## 9. 確認日

- 内容の確認日: 2026-11-13
- Radar の評価の確認日: 2026-11-13

## 用語集

| 用語 | 説明 |
|---|---|
| イシュートラッカー | 作業や不具合（課題）を登録し、担当・状態・優先度を管理する道具 |
| 依存関係 | ある課題が終わらないと、別の課題に着手できないという関係 |
| Dolt | Git のように、分岐・統合・送受信ができる SQL のデータベース |
| SQL | データベースの表を読み書きするための言語 |
| スキーマ | データベースの表の構造（どんな列があるかなど）の定義 |
| ハッシュ | データから計算する短い値。ここでは、ぶつかりにくい ID を作るのに使う |
| AGENTS.md | コーディングエージェントへの指示をまとめて書いておくファイル |
| コーディングエージェント | コードの読み書き・テストの実行などを自分で進める AI |
| ブランチ | Git で、本流から分かれて並行して変更を進める作業の線 |
| CI | 変更のたびに、ビルドやテストを自動で走らせる仕組み |
| 文脈（コンテキスト） | モデルが一度に見ている文章の範囲。会話の履歴などが入る |
| リモート | Git で、変更を送受信する共有の置き場所 |
