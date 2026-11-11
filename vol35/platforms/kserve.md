# KServe（Kubernetes で AI モデルを提供する基盤）

> 非公式の解説です（Thoughtworks とは関係がありません）。Claude Code（AI）が作成しました。状態が「公開承認済み」の解説は人が公開を承認していますが、内容の正確さを人が個別に確かめてはいません。「下書き」の解説は承認前の作成中のものです。詳しくは [README](../../README.md) を参照してください。

| 項目 | 内容 |
|---|---|
| 版 | Technology Radar Vol.35（2026年11月） |
| 象限 | Platforms（アプリケーションを載せて動かす基盤・サービス） |
| リング | Trial（新規か移動かは Radar のページに明記なし） |
| 状態 | 下書き |

## 1. 一言でいうと

KServe は、**Kubernetes**（コンテナを多数のサーバーで動かし、管理する基盤）の上で、学習済みの AI モデルを**推論**（入力を受けて予測や文章を返すこと）のサービスとして動かすための仕組みです。従来の機械学習のモデル（予測 AI）と、大規模言語モデル（LLM、生成 AI）の両方を扱います（[GitHub: kserve/kserve](https://github.com/kserve/kserve)）。ライセンスは Apache-2.0 で、CNCF（クラウドネイティブの技術を育てる団体）の incubating（育成段階）のプロジェクトです（[CNCF のプロジェクトのページ](https://www.cncf.io/projects/kserve/)）。

たとえるなら、モデルという「料理人」を雇ってくる派遣会社ではなく、厨房の配置・客の案内・混雑時の増員を引き受ける「店の運営の仕組み」です。料理そのもの（推論の計算）は、vLLM や Triton などの専用の実行環境が担います（たとえは筆者の見立て）。

## 2. 解決する課題（なぜ生まれたか）

モデルを作るまでと、それを本番で安定して動かすのは別の仕事です。本番では、Web の API として公開する、アクセスの量に合わせて台数を増減する、新しい版に少しずつ切り替える、GPU を無駄なく使う、といった運用が要ります。モデルごとに作り方がばらばらだと、モデルが増えるほど運用が重くなります（筆者の見立て）。

KServe は、こうした運用の部分を Kubernetes の**カスタムリソース**（Kubernetes に独自の種類の設定を足す仕組み）として標準化します。「このモデルを、この形式で動かす」と宣言すると、KServe のコントローラーがその推論サービスの作成から削除までを管理します（[データプレーンの説明](https://kserve.github.io/website/docs/concepts/architecture/data-plane)）。

もとは2019年に Kubeflow（Kubernetes 上の機械学習の道具一式）のプロジェクトとして、Google・IBM・Bloomberg・NVIDIA・Seldon が協力して始めました。2022年に KFServing から KServe へ名前を変えて Kubeflow から独立し、2025年に CNCF の incubating のプロジェクトになりました（[CNCF の告知](https://www.cncf.io/blog/2025/11/11/kserve-becomes-a-cncf-incubating-project/)）。

## 3. 仕組みの概要

### 3.1 主なリソース

| リソース | 役割 |
|---|---|
| InferenceService | 1つのモデルの推論サービス。必須の predictor（推論本体）に、任意の transformer（前後の加工）と explainer（結果の説明）を足せる（[データプレーン](https://kserve.github.io/website/docs/concepts/architecture/data-plane)） |
| ServingRuntime | モデルを動かす実行環境のひな形。実行環境を名指ししなければ、自動選択を許した実行環境の中から、モデルの形式に合うものが選ばれる。クラスタ全体用の ClusterServingRuntime もある（[ServingRuntime](https://kserve.github.io/website/docs/concepts/resources/servingruntime)） |
| InferenceGraph | 複数のモデルを順につなぐ・条件で振り分ける・結果をまとめる、といった組み合わせを定義する（[InferenceGraph](https://kserve.github.io/website/docs/model-serving/inferencegraph/overview)） |
| LLMInferenceService | LLM 専用のリソース。複数のサーバーにまたがる推論などの高度な機能を使える（次の 3.3） |

### 3.2 予測 AI のモデル

scikit-learn・XGBoost・LightGBM・TensorFlow などの形式に、それぞれ用意された実行環境があります。NVIDIA の Triton（多くの形式を動かせる推論サーバー）を通して、TorchScript や ONNX の形式も動かせます（[対応するフレームワーク](https://kserve.github.io/website/docs/model-serving/predictive-inference/frameworks/overview)）。

### 3.3 生成 AI（LLM）

- Hugging Face のモデル向けの実行環境は、vLLM（LLM を高速に動かすための推論エンジン）を使い、vLLM が対応しないモデルでは別の方式に切り替えます。OpenAI 互換の API（チャットの応答・文章の補完・埋め込み・並べ替え〈rerank〉）を出せます（[生成 AI の推論](https://kserve.github.io/website/docs/model-serving/generative-inference/overview)）
- LLMInferenceService は、llm-d（LLM の分散推論の仕組み）を土台にしています。1台の GPU に載らないモデルを複数のサーバーで動かす、入力の読み込み（prefill）と文章の生成（decode）を別のサーバーに分ける、といった構成をとれます（[LLMInferenceService](https://kserve.github.io/website/docs/model-serving/generative-inference/llmisvc/llmisvc-overview)）

### 3.4 動かし方のモード

| モード | 内容 |
|---|---|
| Knative モード（既定） | Knative（リクエストに応じてコンテナを増減する仕組み）と、ネットワークの層（Istio を推奨。Kourier などもある）を使う。アクセスが無いときに台数を0まで減らせ、新しい版への少しずつの切り替え（カナリア）もできる。現行の文書では予測 AI のモデル向け（[Standard モード](https://kserve.github.io/website/docs/admin-guide/kubernetes-deployment)、[Knative モード](https://kserve.github.io/website/docs/admin-guide/serverless)、[管理者向けの概要](https://kserve.github.io/website/docs/admin-guide/overview)） |
| Standard モード | Kubernetes の標準の部品だけで動かす。生成 AI と予測 AI の両方を扱え、GPU を使う本番向け。依存が少ない代わりに、HTTP のアクセスに応じた0からの起動とカナリアには対応しない（[Standard モード](https://kserve.github.io/website/docs/admin-guide/kubernetes-deployment)、[管理者向けの概要](https://kserve.github.io/website/docs/admin-guide/overview)、[GitHub](https://github.com/kserve/kserve)） |

### 3.5 モデルの置き場所

モデルのファイルは、S3・GCS・Azure Blob・Hugging Face などからは起動時に取り込み、Kubernetes の永続ボリュームはそのままつないで使います。モデルを OCI イメージ（コンテナのイメージの形式）に入れて配る方式もあります（[ストレージ](https://kserve.github.io/website/docs/model-serving/storage/overview)）。大きなモデルの取り込みを速くするため、サーバーのローカルのディスクにモデルを置いておくキャッシュの機能もあります（既定では無効。[LocalModelCache](https://kserve.github.io/website/docs/model-serving/generative-inference/modelcache/localmodel)）。

## 4. アーキテクチャ上の位置づけとトレードオフ

**位置づけ**: 推論の計算そのものではなく、**推論サービスの運用を担う層**です。vLLM や Triton などの実行環境と、利用者のアプリの間に入ります。Thoughtworks も、推論エンジンではなく実行環境を束ねる役であり、多くのモデルを GPU の基盤で運用するときの管理の課題に応えるものだと位置づけています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/platforms/kserve)）。

**得られるもの**

- **モデルの運用をそろえられる**: 形式の違うモデルを、同じリソースと同じ手順で公開・増減・切り替えできる（[ServingRuntime](https://kserve.github.io/website/docs/concepts/resources/servingruntime)。運用面の評価は筆者の見立て）
- **実行環境を選べる**: 実行環境をひな形として差し替えられるので、新しい推論エンジンが出ても乗り換えやすい（同上。乗り換えやすさは筆者の見立て）
- **資源の節約**: Knative モードでは、使われていないモデルの台数を0にできる（[Knative モード](https://kserve.github.io/website/docs/admin-guide/serverless)）

**代わりに失うもの・注意点**

- **Kubernetes の運用力が前提**: Knative モードでは、cert-manager・Knative・ネットワークの層（v0.21 の手順では Istio）などの部品も入れて保守する必要があります（[依存するもの](https://kserve.github.io/website/docs/install/dependencies)、[Knative モード](https://kserve.github.io/website/docs/admin-guide/serverless)）。Kubernetes を運用していない組織には重い選択です（筆者の見立て）
- **0からの起動には時間がかかる**: 台数を0にしたモデルは、最初の要求でコンテナの起動とモデルの取り込みを待つことになります。大きなモデルほど待ち時間が長くなります（筆者の見立て）
- **LLM 向けの機能は新しい**: LLMInferenceService の API は v1alpha1（初期の段階）で、Thoughtworks も、周辺の仕組みがまだ育ちつつあると注意しています（[LLMInferenceService](https://kserve.github.io/website/docs/model-serving/generative-inference/llmisvc/llmisvc-overview)、[Radar のページ](https://www.thoughtworks.com/en-us/radar/platforms/kserve)）
- **ドキュメントの古い箇所**: 公式サイトのトップの手順には古い版の例が残っており、版ごとの手順を確かめる必要があります（[公式サイト](https://kserve.github.io/website/)、[依存するもの](https://kserve.github.io/website/docs/install/dependencies)）

## 5. 採用を判断するときの観点

- **規模**: 推論に使うモデルが少数なら、クラウドの推論サービスや、推論エンジンを直接コンテナで動かすほうが軽いこともある。多くのモデルや多くのチームの推論を1つの基盤にまとめたいときに効く（筆者の見立て）
- **モードの選択**: 予測 AI のモデルで、台数を0まで減らす・カナリアを使うなら Knative モード。LLM を動かす、または依存を減らしたいなら Standard モード（[管理者向けの概要](https://kserve.github.io/website/docs/admin-guide/overview)、[Standard モード](https://kserve.github.io/website/docs/admin-guide/kubernetes-deployment)）
- **LLM の大きさ**: 1台のサーバーで動く LLM なら、InferenceService（Standard モード）でも動かせる。複数のサーバーにまたがる推論、prefill と decode の分離、賢い振り分けが要るなら LLMInferenceService を使う（[LLMInferenceService](https://kserve.github.io/website/docs/model-serving/generative-inference/llmisvc/llmisvc-overview)）
- **費用**: KServe 自体はオープンソースだが、GPU のサーバーと Kubernetes の運用の人手が主な費用になる（筆者の見立て）
- **実績**: 公式の採用企業の一覧には、本番で使うと示した組織が十数社載っている（網羅的ではない。掲載は、使っていることを公にしたか、掲載を許可した組織。[採用企業](https://kserve.github.io/website/docs/community/adopters)）

## 6. 関連技術・代替技術

- **vLLM・SGLang・NVIDIA Triton**: KServe が束ねる推論の実行環境。Radar のページでも例として挙がっています（[Radar のページ](https://www.thoughtworks.com/en-us/radar/platforms/kserve)）
- **llm-d**: LLM の分散推論の仕組み。KServe の LLMInferenceService の土台です（[CNCF の告知](https://www.cncf.io/blog/2025/11/11/kserve-becomes-a-cncf-incubating-project/)）
- **Knative**: KServe の既定のモードで使う、要求に応じた増減の仕組み（[Standard モード](https://kserve.github.io/website/docs/admin-guide/kubernetes-deployment)、[Knative モード](https://kserve.github.io/website/docs/admin-guide/serverless)）
- **クラウドの推論サービス**: Amazon SageMaker や Vertex AI など、運用を提供元に任せる選択肢（筆者の見立て）

## 7. Radar での扱い

- Vol.35（2026年11月）: **Trial**（新規か移動かは Radar のページに明記なし）
- 過去の版: Radar のページの推移には、今回の版だけが載っています
- Thoughtworks の評価の要旨: Kubernetes 上で予測 AI と生成 AI の推論を運用する仕組みとして、社内に推論の基盤を作るときにまず選ぶ候補にしています。主な理由は、推論の実行環境とサービスの運用を切り分けて、多くのモデルをまとめて管理できることです
- Radar のページ: https://www.thoughtworks.com/en-us/radar/platforms/kserve

## 8. 深掘りするときの一次情報

- [GitHub: kserve/kserve](https://github.com/kserve/kserve)
- [KServe のドキュメント](https://kserve.github.io/website/)
- [データプレーン](https://kserve.github.io/website/docs/concepts/architecture/data-plane) と [ServingRuntime](https://kserve.github.io/website/docs/concepts/resources/servingruntime)
- [LLMInferenceService](https://kserve.github.io/website/docs/model-serving/generative-inference/llmisvc/llmisvc-overview)
- [CNCF の告知（incubating になった経緯）](https://www.cncf.io/blog/2025/11/11/kserve-becomes-a-cncf-incubating-project/)

## 9. 確認日

- 内容の確認日: 2026-11-11
- Radar の評価の確認日: 2026-11-11

## 用語集

| 用語 | 説明 |
|---|---|
| Kubernetes | コンテナを多数のサーバーで動かし、台数の増減や障害時の入れ替えを自動で行う基盤 |
| 推論 | 学習済みのモデルに入力を渡し、予測や生成の結果を得ること |
| カスタムリソース | Kubernetes に、標準には無い種類の設定（ここでは「推論サービス」など）を足す仕組み |
| CNCF | Cloud Native Computing Foundation。Kubernetes などのクラウドネイティブの技術を育てる団体。プロジェクトは sandbox・incubating・graduated の順に段階が上がる |
| 予測 AI | 分類や数値の予測など、決まった形の答えを返す従来型の機械学習のモデル |
| LLM | 大規模言語モデル。大量の文章で学習し、文章を生成する AI モデル |
| vLLM | LLM の推論を高速・省メモリで行うための、オープンソースの推論エンジン |
| prefill と decode | LLM の推論の2つの段階。入力の文章をまとめて読み込む段階（prefill）と、1語ずつ生成する段階（decode） |
| カナリア | 新しい版に一部の利用者だけを先に流し、問題が無ければ少しずつ広げる切り替え方 |
| OpenAI 互換の API | OpenAI の API と同じ形で呼び出せるようにした API。既存のアプリから呼び先を変えるだけで使える |
