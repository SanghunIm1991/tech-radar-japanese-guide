# 着手時の確認（2026-11-08）

`docs/idea_origin.md` の「着手時に確認すること」と、GitHub Pages の条件を確かめた結果。Web 上の公式ページを読んで確かめた（2026-11-08 時点）。今後変わりうるので、使う前に改めて確かめる。

## Technology Radar の最新の版と構成

- **最新の版**: Vol.35（2026年11月）。前の版は Vol.34（2026年4月）
- **項目数**: 117（Techniques 45、Platforms 22、Tools 36、Languages & Frameworks 14）
- **リングの名前**: Adopt／Trial／Assess／**Caution**。元アイデアに書いた「Hold」は、Vol.35 の公式ページでは「Caution」になっている。ただし、Build Your Own Radar のページはいまも Hold と書いている。Vol.35 で Caution に置かれた項目は Techniques の10項目だけだった
- **テーマ**: 4つ（Harness engineering at scale／Beyond frontier models／Putting more agents to work／Constructing the AI factory）
- **新規・移動の印**: 読み取ったテキストには、項目ごとの「新規／移動」の印が含まれていなかった。解説を書くときに、項目ごとのページで確かめる
- 出典: https://www.thoughtworks.com/en-us/radar と、各象限のページ

## 公式の日本語版の有無

- `https://www.thoughtworks.com/ja-jp/radar` は 404 だった
- Radar のページに言語の切り替えは見当たらなかった。PDF は英語とポルトガル語だけだった
- 公式の日本語版は確認できなかった（ページの見出し部分に切り替えがある可能性までは否定できない）

## Radar のサイトの利用条件

- 出典: https://www.thoughtworks.com/radar/tos （最終更新は 2016-11-03。対象は radar.thoughtworks.com とそのサブドメインと書かれている）。thoughtworks.com 全体の利用条件のページ（`/terms-of-use`）は 404 で、見つからなかった
- 要点（自分の言葉での要約）:
  - 内容をダウンロード・印刷してよいのは、個人で使う場合に限られる。それ以外の複製・改変・再配布には、書面による許可が要る。商用の利用にはライセンスが要る
  - サイトの内容を自動で収集すること（スクレイピング等）や、他人にそれを勧めることは禁じられている
  - リンクを張るのはよい。ただし、公正かつ合法であることと、Thoughtworks との関係や推奨を示唆しないことが条件
  - 引用・要約・翻訳について、特別な定めはない
- 方針への反映（CLAUDE.md に追記した）:
  - Radar の文章を翻訳・転載しない（翻訳は改変・複製にあたる）
  - Radar での評価は、自分の言葉で書いたごく短い要旨と、元のページへのリンクにとどめる
  - 非公式である旨を README に明記する
  - Radar のページを自動で一括収集しない（解説を書くときに項目ごとのページを読むのにとどめる）
- 法律の専門家の判断ではない。疑わしい場合は安全側に倒す

## GitHub Pages の条件

- GitHub Free では、Pages を使えるのは public のリポジトリだけ。private のリポジトリで使うには Pro 等の有料プランが要る
- private のリポジトリから公開しても、サイト自体はインターネット上で誰でも見られる（アクセスを制限できるのは Enterprise Cloud の組織だけ）
- 上限: リポジトリは 1GB 以下を推奨、公開するサイトは 1GB まで、帯域はおおむね月 100GB（ソフトな上限）、ビルドはおおむね 1時間に10回（独自の Actions のワークフローで公開する場合は対象外）、デプロイは10分でタイムアウト
- 商用サイトの無料ホスティングとしての利用は禁じられている（本プロジェクトは非商用なので該当しない）
- 出典: https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits 、https://docs.github.com/en/pages/getting-started-with-github-pages/what-is-github-pages
