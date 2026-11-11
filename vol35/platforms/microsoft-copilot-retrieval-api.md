# Microsoft Copilot Retrieval API（Microsoft 365 の文書を AI 向けに検索する API）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Platforms（アプリケーションを載せて動かす基盤・サービス） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

Microsoft Copilot Retrieval API は、SharePoint・OneDrive・Copilot コネクタ（外部のデータを Microsoft 365 に取り込む仕組み）にある社内の文書から、質問に関係する**テキストの抜粋**を返す API です。Microsoft Graph（Microsoft 365 のデータを扱う共通の API）の `/copilot/retrieval` という入口から呼び出します（[Retrieval API の概要](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/overview)、[retrieval の API リファレンス](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/copilotroot-retrieval)）。

自分で作る AI のアプリやエージェントが、社内の文書をもとに答えられるようにする **RAG**（検索で取り出した文書を AI に渡して答えさせる方法）の「検索」の部分を、Microsoft 365 にすでにある索引に任せる、と考えると分かりやすいです（言い換えは筆者の見立て）。

## 2. 解決する課題（なぜ生まれたか）

社内の文書を使う RAG を自分で作ると、文書を集めて、細かく区切り（チャンク化）、検索用の索引を作り、更新のたびに作り直し、さらに「その人が見てよい文書か」の権限を検索結果に反映する、という仕組みを持つ必要があります。文書を別の置き場へ複製するため、情報の管理の面でも気を使います（筆者の見立て）。

Retrieval API は、Microsoft 365 がすでに持っている索引を使い、データを元の場所に置いたまま、呼び出した人の権限の範囲で結果を返します。公式は、データを外へ出したり複製したりせずに、既存のアクセス制御を保ったまま AI に文脈を渡せる点を利点に挙げています（[Retrieval API の概要](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/overview)）。

## 3. 仕組みの概要

### 3.1 呼び出し方

`POST https://graph.microsoft.com/v1.0/copilot/retrieval` に、次のような内容を送ります（[retrieval の API リファレンス](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/copilotroot-retrieval)）。

- **queryString**: 探したい内容の文。1,500字まで。1文の自然な文が勧められています
- **dataSource**: 検索する場所。SharePoint・OneDrive・Copilot コネクタのどれか1つ。1回の呼び出しで混ぜることはできません
- **filterExpression**: KQL（Microsoft の検索で使う、条件の書き方）で、作成者・ファイルの種類・更新日時・置き場所などに絞り込む
- **maximumNumberOfResults**: 返す件数（1〜25、既定は25）

### 3.2 返ってくるもの

文書ごとに、URL・関係する部分のテキストの抜粋の一覧（抜粋ごとに関連度のスコア 0〜1）・必要に応じて指定したメタデータ・（SharePoint などでは）秘密度のラベルが返ります。結果は関連度の順に並んではいません（[Retrieval API の概要](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/overview)、[retrieval の API リファレンス](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/copilotroot-retrieval)）。アプリは、この抜粋を LLM への指示に入れて答えを作らせます。

### 3.3 権限

- 呼び出した利用者が見られる文書だけが返ります（[retrieval の API リファレンス](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/copilotroot-retrieval)）
- 利用者の代わりにアプリが呼ぶ**委任されたアクセス許可**だけに対応し、利用者を介さずにアプリだけで呼ぶ**アプリ専用のアクセス許可**には対応しません。SharePoint と OneDrive には `Files.Read.All` と `Sites.Read.All`、Copilot コネクタには `ExternalItem.Read.All` が要ります（同上、[Retrieval API の概要](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/overview)）

### 3.4 ライセンスと料金

- Microsoft 365 Copilot のライセンスを持つ利用者は、追加の料金なしで使えます（[Retrieval API の概要](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/overview)）
- ライセンスの無い利用者向けには、プレビューの従量課金があります。対象は SharePoint と Copilot コネクタで、OneDrive は対象外です。確認日の時点の料金は1回の呼び出しあたり0.10ドルで、Azure のサブスクリプションと、テナントに1つ以上の Copilot のライセンスが要ります。プレビューのため SLA（稼働の保証）はありません（[従量課金](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/paygo-retrieval)）

### 3.5 主な制限

- 利用者1人あたり1時間に200件まで（[Retrieval API の概要](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/overview)）
- 画像やグラフなど、テキスト以外の内容は対象外。表の中のテキストは .doc・.docx・.pptx だけが対象（同上）
- SharePoint・OneDrive では、意味で探す検索（と、言葉の一致との組み合わせの検索）が使えるのは .doc・.docx・.pptx・.pdf・.aspx・.one だけで、それ以外は言葉の一致による検索だけ（同上）
- Microsoft 365 Copilot の**セマンティックインデックス**（意味で探すための索引）の制限をそのまま受けます（同上、[セマンティックインデックス](https://learn.microsoft.com/en-us/microsoftsearch/semantic-index-for-copilot)）
- 提供はグローバルの環境だけで、米国の政府機関向けや中国の環境では使えません（[retrieval の API リファレンス](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/copilotroot-retrieval)）

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: RAG の構成のうち「検索」の部分を、Microsoft 365 の管理された索引に任せる選択肢です。自前の索引（ベクトルのデータベースなど）を持つ構成と比べて、作る部品と運用する部品が減ります（筆者の見立て）。Thoughtworks も、区切り方・索引・同期・権限の制御を Microsoft 側に任せ、独自の検索の仕組みを保守せずに済む点を評価しています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/platforms/microsoft-copilot-retrieval-api)）。

**得られるもの**

- **権限の反映を任せられる**: 利用者が見られる文書だけが返るので、権限を自前で検索結果に反映する仕組みが要らない（[retrieval の API リファレンス](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/copilotroot-retrieval)）
- **データを複製しない**: 文書は元の場所に置いたまま使える（[Retrieval API の概要](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/overview)）
- **索引の運用が要らない**: 別のデータの流れを保守しなくてよい（[Retrieval API の概要](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/overview)）。ただし、SharePoint に新しく足した文書は、索引に入るまで1日ほどかかることがあります（[セマンティックインデックス](https://learn.microsoft.com/en-us/microsoftsearch/semantic-index-for-copilot)）

**代わりに失うもの・注意点**

- **Microsoft 365 の中のデータに限られる**: 対象は SharePoint・OneDrive・Copilot コネクタだけです（[Retrieval API の概要](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/overview)）。ほかの場所のデータも使うなら、別の検索と組み合わせる必要があります（筆者の見立て）
- **検索の中身を調整できない**: 区切り方・索引の作り方・順位の付け方は Microsoft が決めます。結果も関連度の順に並ばないため、公式は件数を絞らず、返った抜粋をすべて LLM に渡すことを勧めています（[Retrieval API の概要](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/overview)。調整できない点は筆者の見立て）
- **条件の書き間違いに気づきにくい**: KQL の書き方が誤っていても、エラーにならず、絞り込み無しで検索されます（同上）。権限の範囲は守られますが、意図より広い文書が返ることになります（筆者の見立て）
- **利用者として呼ぶ前提**: アプリ専用のアクセス許可に対応しないため、夜間の一括処理のような、利用者がいない処理には向きません（[retrieval の API リファレンス](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/copilotroot-retrieval)。向き不向きは筆者の見立て）
- **件数の上限**: 1人1時間200件の上限は、エージェントが何度も検索する作りでは足りなくなることがあります（筆者の見立て）

## 5. 採用を判断するときの観点

- **データの置き場所**: 社内の知識がすでに SharePoint などにまとまっているなら効果が大きい。Thoughtworks も、知識がすでに Microsoft の道具にある場合に特に役立つとしています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/platforms/microsoft-copilot-retrieval-api)）
- **ライセンスと費用**: 使う人が Copilot のライセンスを持っているか。持っていない人が多いなら、従量課金の費用（呼び出し1回ごと）を見積もる（[従量課金](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/paygo-retrieval)）
- **提供の段階**: 従量課金はプレビューで SLA が無い。本番で頼るなら、段階と条件を確かめる（同上）
- **検索の質の確かめ方**: 自分たちの文書と質問で、返る抜粋が足りているかを試す。Copilot コネクタから取り出すときは、スコアの無い抜粋が返ることもあります（[Retrieval API の概要](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/overview)。試し方は筆者の見立て）
- **情報の管理**: 検索の対象から外した SharePoint のサイトは、索引からも外れます（[セマンティックインデックス](https://learn.microsoft.com/en-us/microsoftsearch/semantic-index-for-copilot)）。管理者の設定と合わせて対象の範囲を決める

## 6. 関連技術・代替技術

- **RAG**: この API が支える、検索して答えさせる方法そのもの。Radar では関連する項目として挙がっています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/platforms/microsoft-copilot-retrieval-api)）
- **自前のベクトル検索**: 文書を自分で区切り、ベクトルのデータベースに入れて検索する構成。対象を選ばず、調整も自由だが、索引と権限の運用を自分で持つ（筆者の見立て）
- **Microsoft Foundry の SharePoint ツール**: 従量課金が有効になった、Copilot のライセンスの無い利用者は、Microsoft Foundry（Microsoft の AI 開発の基盤）の SharePoint のツールも従量課金で使えます（[従量課金](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/paygo-retrieval)）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: Microsoft 365 の文書から RAG 用の抜粋を、利用者の権限を守ったまま取り出せる手間の少ない API として Trial に置き、採用して良い経験を得たことを根拠に挙げています。主な利点は、索引・同期・権限の制御を任せられ、独自の検索の仕組みを保守せずに社内の知識を使うエージェントを作れることです
- Radar のページ: https://www.thoughtworks.com/en-us/radar/platforms/microsoft-copilot-retrieval-api

## 8. 深掘りするときの一次情報

- [Retrieval API の概要](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/overview)
- [retrieval の API リファレンス](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/copilotroot-retrieval)
- [従量課金](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/api/ai-services/retrieval/paygo-retrieval)
- [セマンティックインデックス](https://learn.microsoft.com/en-us/microsoftsearch/semantic-index-for-copilot)

## 9. 確認日

- 内容の確認日: 2026-11-11
- Radar の評価の確認日: 2026-11-11

## 用語集

| 用語 | 説明 |
|---|---|
| RAG | Retrieval-Augmented Generation。質問に関係する文書を検索で取り出し、それを AI に渡して答えさせる方法 |
| Microsoft Graph | Microsoft 365 のメール・予定・ファイルなどのデータを、共通の形で読み書きするための API |
| SharePoint・OneDrive | Microsoft 365 の文書の置き場。SharePoint は組織やチームの共有用、OneDrive は個人用 |
| Copilot コネクタ | Microsoft 365 の外にあるデータを取り込み、Microsoft 365 の検索や Copilot から使えるようにする仕組み |
| チャンク化 | 長い文書を、検索や AI に渡しやすい大きさの断片に区切ること |
| セマンティックインデックス | 言葉の一致だけでなく、意味の近さで探せるようにした検索の索引 |
| KQL | Keyword Query Language。Microsoft の検索で、作成者や日付などの条件を組み合わせて書くための書き方 |
| 委任されたアクセス許可 | アプリが、ログインしている利用者の代わりに、その人の権限の範囲でデータを読む方式 |
| アプリ専用のアクセス許可 | 利用者を介さず、アプリ自身の権限でデータを読む方式。夜間の処理などで使う |
| SLA | Service Level Agreement。提供する側が約束する稼働率などの水準 |
