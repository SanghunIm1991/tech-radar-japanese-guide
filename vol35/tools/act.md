# Act（GitHub Actions のワークフローを手元で動かす）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Tools（開発や運用に使う道具） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

Act（nektos/act）は、**GitHub Actions のワークフローを、GitHub に送らずに手元の PC で動かす**ためのオープンソースの道具です。GitHub Actions は、コードを push したときなどにビルドやテストを自動で走らせる GitHub の仕組み（CI）で、その手順は `.github/workflows/` に置いた YAML のファイル（ワークフロー）に書きます。Act はこのファイルを読み、手元の Docker（アプリを隔離した箱＝コンテナで動かす仕組み）の上で同じ手順を再現します（[GitHub: nektos/act](https://github.com/nektos/act)）。

たとえるなら、本番の試験会場に行く前に、自宅で同じ問題用紙を使って模擬試験をするようなものです（筆者の見立て）。

## 2. 解決する課題（なぜ生まれたか）

ワークフローのファイルを直すとき、従来は「直す → コミットする → push する → GitHub 上で結果が出るのを待つ」をくり返すしかありませんでした。1回の確認に数分かかり、試すたびに意味の薄いコミットが履歴に積み上がります。Act の README も、コミットや push をせずに試せる速い確認（Fast Feedback）を、使い道の1つめに挙げています（[GitHub: nektos/act](https://github.com/nektos/act)）。

最近は、この問題がコーディングエージェントで大きくなっています。エージェントは人より頻繁にコミットと push をするので、CI のビルドが混み合い、順番待ちが起きやすくなります。Act を使えば、エージェントが push する前に手元でワークフローを動かし、CI の確認を「書く→試す」の輪の中に入れられます（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/act)）。

## 3. 仕組みの概要

**動き方**（[GitHub: nektos/act](https://github.com/nektos/act) の「How Does It Work?」）

1. `.github/workflows/` のワークフローを読み、ジョブどうしの依存関係から実行の順番を決める
2. ワークフローに書かれたとおりに、必要な Docker のイメージ（コンテナのひな形）を取ってくるか作る
3. 手順を動かすためのコンテナを起動し、実行する。環境変数やファイルの配置は、GitHub の環境に近づけてある

**主な使い方**（[act の使い方](https://nektosact.com/usage/index.html)）

| 例 | 意味 |
|---|---|
| `act` | push のイベントとして、該当するワークフローを動かす |
| `act pull_request` | プルリクエストのイベントとして動かす |
| `act -l` | 動かす対象のワークフローとジョブを一覧する |
| `act -j test` | `test` という名前のジョブだけを動かす |
| `act -W .github/workflows/checks.yml` | ワークフローのファイルを指定する |

GitHub の CLI（`gh`）の拡張としても入れられ、`gh act` の形で使えます（[act: GitHub CLI 拡張](https://nektosact.com/installation/gh.html)）。ライセンスは MIT です（[GitHub: nektos/act](https://github.com/nektos/act)）。

**ランナーのイメージ**（[act: Runners](https://nektosact.com/usage/runners.html)）

GitHub 上でジョブを動かす環境（ランナー）の代わりに、Act は Docker のイメージを使います。大きさの違う3種類から選べます。

- **micro**: Node.js の軽いイメージ（node:16-buster-slim）をもとにしたもの
- **medium**: Ubuntu をもとにした act 用のイメージ
- **large**: GitHub のランナーのファイルシステムを写したイメージ

ドキュメントは、既定のイメージは**わざと不完全にしてある**と明記しており、GitHub のランナーにある道具が無いために動かない手順がありうるとしています。`-P` オプションで別のイメージを指定できます。

**秘密情報の渡し方**（[act の使い方](https://nektosact.com/usage/index.html)「Secrets」）

API キーなどの秘密情報（secrets）は、`-s` オプションや `--secret-file` で渡します。値をコマンドに直接書くとシェルの履歴に残るので、環境変数か、その場で入力を求める方法が勧められています。GitHub が自動で用意するトークンは、Act では自動では渡されません。

**そのほかの設定**（[act の使い方](https://nektosact.com/usage/index.html)）

- **成果物（アーティファクト）**: ジョブが作ったファイルを次のジョブに渡す仕組みは、既定では動かない。`--artifact-server-path` で保存先を指定すると、同じ実行の中で受け渡せる
- **いつも使うオプション**: `.actrc` というファイルに1行に1つずつ書いておくと、毎回指定しなくて済む

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: CI のパイプラインの「手元での試験台」です。本番の CI を置き換えるものではなく、本番に出す前の速い確認の層です（筆者の見立て）。

**得られるもの**

- **速い確認**: push せずにワークフローを試せる（[GitHub: nektos/act](https://github.com/nektos/act)）
- **エージェントとの相性**: エージェントが push の前に CI を試せる（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/act)）
- **手元のタスクの実行**: [README](https://github.com/nektos/act) は、Makefile の代わりにワークフローを手元の作業の実行に使う用途も挙げている

**代わりに失うもの・注意点**（[act: 対応していない機能](https://nektosact.com/not_supported.html)）

- **完全な再現ではない**: 同時実行の制御（concurrency）、ジョブの権限（permissions）、ジョブの制限時間、デプロイ先の環境（environment）ごとの秘密情報などは無視される。手順の要約（step summary）も捨てられる
- **ランナーの違い**: Docker では systemd（Linux のサービスを管理する仕組み）などが動かないため、仮想マシンのランナーと動きが違うことがある。Windows や macOS のランナーは Docker では再現できない（同じ OS の PC 上で、Docker を使わずに直接動かす方法はある）（[act: Runners](https://nektosact.com/usage/runners.html)）
- **イメージの大きさ**: GitHub のランナーに近づけるほどイメージが大きくなり、取得に時間と容量がかかる（筆者の見立て）
- **秘密情報の扱い**: 手元で本物の秘密情報を使うと、漏れる経路が増える。試験用の値を使うのが安全（筆者の見立て）

## 5. 採用を判断するときの観点

- **使っている CI**: GitHub Actions を使っているか。ほかの CI には使えない（筆者の見立て）
- **ワークフローの中身**: 使っている機能が、Act の対応していない機能に当たらないか。当たるなら、手元では一部の手順を飛ばす。ドキュメントは、`env.ACT` などの印で手元の実行のときだけ手順を飛ばす方法を説明している（[act の使い方](https://nektosact.com/usage/index.html)）
- **Docker の用意**: 開発者の PC やエージェントの実行環境で、Docker が動かせるか（筆者の見立て）
- **イメージの選び方**: 軽いイメージで足りるか、道具の多い大きなイメージが要るか。大きいほど再現性は上がるが、取得の時間と容量がかかる。チームで同じイメージをそろえて、人によって結果が違わないようにする（筆者の見立て）
- **組み合わせ**: Act で動きを試すのに加えて、ワークフローの設定の危険を静的に調べる道具（zizmor など）を組み合わせると、エージェントがワークフロー自体を書き換えたときにも壊れにくくなる（[Radar のページ](https://www.thoughtworks.com/en-us/radar/tools/act)）
- **最後の確認は本番の CI で**: 手元で通っても、本番の CI での確認は残す（筆者の見立て）

## 6. 関連技術・代替技術

- **GitHub Actions**、**zizmor**、**Agent pipeline responsibility**: [Radar の Act のページ](https://www.thoughtworks.com/en-us/radar/tools/act)の関連の項目に挙がっています
- **代替: CI の側で速くする**: ワークフローを小さく分ける、キャッシュを使うなどで、push してから結果が出るまでの時間を縮める方法もある（筆者の見立て）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています。本文によると、以前（2021年）に GitHub Actions の項目の中で触れたことがあります
- Thoughtworks の評価の要旨: 完全な再現ではないとしつつ、コーディングエージェントが push の前にワークフローを手元で試せる点を評価しています
- Radar のページ: https://www.thoughtworks.com/en-us/radar/tools/act

## 8. 深掘りするときの一次情報

- [GitHub: nektos/act](https://github.com/nektos/act)
- [act のドキュメント: 使い方](https://nektosact.com/usage/index.html)
- [act のドキュメント: Runners](https://nektosact.com/usage/runners.html)
- [act のドキュメント: 対応していない機能](https://nektosact.com/not_supported.html)
- [act のドキュメント: GitHub CLI 拡張](https://nektosact.com/installation/gh.html)

## 9. 確認日

- 内容の確認日: 2026-11-13
- Radar の評価の確認日: 2026-11-13

## 用語集

| 用語 | 説明 |
|---|---|
| GitHub Actions | GitHub に組み込まれた、ビルドやテストなどを自動で走らせる仕組み |
| CI | 変更のたびに、ビルドやテストを自動で走らせる仕組み |
| ワークフロー | GitHub Actions で、いつ・何を・どの順で実行するかを YAML で書いたファイル |
| ジョブ | ワークフローの中の、1つの環境で続けて実行する手順のまとまり |
| ランナー | ジョブを実際に動かす環境（GitHub が用意する仮想マシンなど） |
| Docker | アプリを、ほかから隔離した箱（コンテナ）の中で動かす仕組み |
| イメージ | コンテナを作るためのひな形。中に入れる OS や道具が決まっている |
| YAML | 設定を、字下げで構造を表して書く文字の形式 |
| 秘密情報（secrets） | API キーやパスワードなど、外に漏らしてはいけない値 |
| コーディングエージェント | コードの読み書き・テストの実行などを自分で進める AI |
| push | 手元のコミットを、GitHub などの共有の場所に送ること |
| systemd | Linux で、サービスの起動や停止を管理する仕組み |
