# GitHub Pages で公開する手順

この解説集を GitHub Pages で公開する手順。2026-11-09 に、この手順で公開まで済んだ。未確認の点は末尾に書く。

- 状態: 2026-11-08 に GitHub の公式ドキュメントで確かめた内容をもとに書き、2026-11-09 にこの手順で公開した（サイト: https://sanghunim1991.github.io/tech-radar-japanese-guide/ ）。mermaid の図は、サイトではコードのまま表示することにした（2026-11-09 決定。下の表を参照）
- 方式: 分岐点P1の選択肢A（GitHub 標準の Jekyll を使い、ブランチから公開する）。手元に Ruby や Jekyll を入れずに済む。2026-11-09 にユーザーが選択肢Aに決めた
- 公開元の設定は、ユーザーの判断で、画面の操作の代わりに Claude が `gh api` で行う（2026-11-09 決定）
- 印の意味: 【確認済み】公式ドキュメントで確かめた。【要確認】公式ドキュメントに記述が無かった（公開後に実物で確かめる）

## 前提

- GitHub Free では、Pages を使えるのは **public のリポジトリだけ**【確認済み】。public に切り替えるかはユーザーが判断し、ユーザー自身が操作する（2026-11-09 に切り替えた）
- Pages のサイトはインターネット上の誰でも見られる【確認済み】。リポジトリと同じく、機密情報・個人情報・第三者の著作物を載せない
- 公開される URL は `https://<ユーザー名>.github.io/<リポジトリ名>/` の形になる。GitHub のユーザー名が URL に出る【確認済み】
- 出典: [Creating a GitHub Pages site](https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-github-pages-site)、[What is GitHub Pages?](https://docs.github.com/en/pages/getting-started-with-github-pages/what-is-github-pages)、[GitHub Pages limits](https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits)

## 事前の準備（リポジトリの中）

1. **トップページ**: 公開元のフォルダの直下にある `index.html`・`index.md`・`README.md` のいずれかが入口になる【確認済み】。このリポジトリでは `README.md` が入口になる
2. **サイトに載せないファイルを除外する**: Jekyll は、`_`・`.`・`#` で始まるファイルとフォルダ、`~` で終わるファイル、設定ファイル（`_config.yml`）の `exclude` に書いたものを、サイトに含めない【確認済み】。この解説集では、作業用の文書をサイトから外すため、リポジトリの直下の `_config.yml` に `exclude`（CLAUDE.md・docs/・templates/・local/）を書いた。サイトの題名と説明にも「非公式」を入れた（2026-11-09）

   リポジトリが public なら、除外したファイルも GitHub 上では誰でも読める。除外はサイトの見た目を整えるためのもので、隠すためのものではない
3. **点検**: `public-release-review` スキル（作成者の手元の Claude Code のスキル）の手順に沿って点検する

出典: [About GitHub Pages and Jekyll](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/about-github-pages-and-jekyll)

## 公開の操作（GitHub の画面）

ユーザーが public に切り替えたあとに行う【確認済み】。

1. リポジトリのページで **Settings** を開く（見当たらなければ「…」のメニューから選ぶ）
2. 左のメニューの「Code, planning, and automation」にある **Pages** を開く
3. 「Build and deployment」の **Source** を「**Deploy from a branch**」にする
4. ブランチに `main`、フォルダに `/(root)` を選ぶ
5. **Save** を押す
6. 反映には最大10分ほどかかる。Settings → Pages の「**Visit site**」からサイトを開く

別の方法（このリポジトリで実際に使った方法）: 画面の代わりに、GitHub CLI で公開元を設定できる。`gh api -X POST repos/<ユーザー名>/<リポジトリ名>/pages -f "source[branch]=main" -f "source[path]=/"`。構築の状態は `gh api repos/<ユーザー名>/<リポジトリ名>/pages/builds/latest` で読める（`built` になれば完了）

出典: [Configuring a publishing source](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site)

## サイトの構築を見守る

- Pages のサイトは、どの方式でも GitHub Actions のワークフローで公開される。構築や公開の失敗は、リポジトリの **Actions** タブの実行履歴で確かめる【確認済み】
- Settings → Pages から、直近に公開したワークフローの実行へ移れる【確認済み】

## 公開後に確かめること

2026-11-09 に公開し、サイトの HTML を取得して確かめた結果。

| 確かめること | 結果 | 状態 |
|---|---|---|
| 日本語が文字化けしないか | 文字コードは UTF-8、言語は ja（`_config.yml` の `lang`）。本文の日本語はそのまま出ている | 【実物で確認済み】 |
| `.md` への相対リンクがサイトの中のページにつながるか | 索引から各解説へのリンクは `.html` のページへ、README へのリンクはサイトのトップへ変換されていた | 【実物で確認済み】 |
| mermaid の図が描画されるか | 描画されない。図はコードのまま表示される（GitHub のリポジトリの画面では描画される） | 【実物で確認済み】 |
| 見出しのアンカーが日本語の見出しでも働くか | 日本語の見出しにも id が付いている（例: `1-一言でいうと`） | 【実物で確認済み】 |
| 除外したファイルがサイトに出ていないか | CLAUDE.md・docs/・templates/・`_config.yml` は 404 | 【実物で確認済み】 |

mermaid の図の扱い（2026-11-09 にユーザーが決定）: サイトではコードのまま表示する。外部のスクリプトへの依存を増やさないためで、解説の本文は図がなくても分かるように書く。検討した別の案は、描画用のスクリプトを外部（CDN）から読み込む案と、分岐点P1の選択肢B（MkDocs 等）へ移る案。

## 未確認のこと

- 無料のプランで、Pages を公開しているリポジトリを private に戻したとき、サイトがどうなるか（公式ドキュメントの該当ページには記述が無かった）
