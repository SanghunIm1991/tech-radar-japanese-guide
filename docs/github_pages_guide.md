# GitHub Pages で公開する手順（下書き）

この解説集を GitHub Pages で公開する手順。実際に1つずつ試しながら、この文書を更新していく。

- 状態: **下書き**。2026-11-08 に GitHub の公式ドキュメントで確かめた内容をもとに書いた。まだ実際には試していない
- 方式: 分岐点P1の選択肢A（GitHub 標準の Jekyll を使い、ブランチから公開する）。手元に Ruby や Jekyll を入れずに済む（Jekyll は Windows では公式にはサポートされていない）
- 印の意味: 【確認済み】公式ドキュメントで確かめた。【要確認】公式ドキュメントに記述が無かった（公開後に実物で確かめる）

## 前提

- GitHub Free では、Pages を使えるのは **public のリポジトリだけ**【確認済み】。public に切り替えるかはユーザーが判断し、ユーザー自身が操作する
- Pages のサイトはインターネット上の誰でも見られる【確認済み】。リポジトリと同じく、機密情報・個人情報・第三者の著作物を載せない
- 公開される URL は `https://<ユーザー名>.github.io/<リポジトリ名>/` の形になる。GitHub のユーザー名が URL に出る【確認済み】
- 出典: [Creating a GitHub Pages site](https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-github-pages-site)、[What is GitHub Pages?](https://docs.github.com/en/pages/getting-started-with-github-pages/what-is-github-pages)、[GitHub Pages limits](https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits)

## 事前の準備（リポジトリの中）

1. **トップページ**: 公開元のフォルダの直下にある `index.html`・`index.md`・`README.md` のいずれかが入口になる【確認済み】。このリポジトリでは `README.md` が入口になる
2. **サイトに載せないファイルを除外する**: Jekyll は、`_`・`.`・`#` で始まるファイルとフォルダ、`~` で終わるファイル、設定ファイル（`_config.yml`）の `exclude` に書いたものを、サイトに含めない【確認済み】。この解説集では、作業用の文書をサイトから外すため、`_config.yml` に次のように書く案（書き方は公開時に確かめる）

   ```yaml
   exclude:
     - CLAUDE.md
     - docs/
     - templates/
   ```

   リポジトリが public なら、除外したファイルも GitHub 上では誰でも読める。除外はサイトの見た目を整えるためのもので、隠すためのものではない
3. **点検**: `public-release-review` スキルの手順に沿って点検する

出典: [About GitHub Pages and Jekyll](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/about-github-pages-and-jekyll)

## 公開の操作（GitHub の画面）

ユーザーが public に切り替えたあとに行う【確認済み】。

1. リポジトリのページで **Settings** を開く（見当たらなければ「…」のメニューから選ぶ）
2. 左のメニューの「Code, planning, and automation」にある **Pages** を開く
3. 「Build and deployment」の **Source** を「**Deploy from a branch**」にする
4. ブランチに `main`、フォルダに `/(root)` を選ぶ
5. **Save** を押す
6. 反映には最大10分ほどかかる。Settings → Pages の「**Visit site**」からサイトを開く

出典: [Configuring a publishing source](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site)

## サイトの構築を見守る

- Pages のサイトは、どの方式でも GitHub Actions のワークフローで公開される。構築や公開の失敗は、リポジトリの **Actions** タブの実行履歴で確かめる【確認済み】
- Settings → Pages から、直近に公開したワークフローの実行へ移れる【確認済み】

## 公開後に確かめること

| 確かめること | 見込み | 状態 |
|---|---|---|
| 日本語が文字化けしないか | 文字化けしない見込み | 【要確認】 |
| `.md` への相対リンク（例: `vol35/index.md`）がサイトの中のページにつながるか | 既定で有効な `jekyll-relative-links` プラグインが変換すると見込む。プラグインの働きは公式ドキュメントでは確かめられなかった | 【要確認】 |
| mermaid の図が描画されるか | 公式ドキュメントに記述が無い。Jekyll の標準のままでは、コードのまま表示される可能性が高い | 【要確認】 |
| 見出しのアンカー（目次のリンク）が日本語の見出しでも働くか | — | 【要確認】 |
| 除外したファイルがサイトに出ていないか | — | 【要確認】 |

mermaid が描画されない場合の対応案: (1) Jekyll のレイアウトに mermaid の描画用スクリプトを読み込ませる、(2) 分岐点P1の選択肢B（MkDocs 等）へ移る。どちらにするかは、そのときに決める。

## 未確認のこと

- 無料のプランで、Pages を公開しているリポジトリを private に戻したとき、サイトがどうなるか（公式ドキュメントの該当ページには記述が無かった）
