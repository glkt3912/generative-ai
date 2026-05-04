# GCS動画 → BigQuery構造化 → Vertex AI Search パイプライン

## 全体像

このリポジトリには「GCS動画をまるごと VAIS で検索可能にする」一本つながりのサンプルは存在しない。
ただし以下の3ステップを組み合わせることで実現できる。各ステップに対応するノートブックが別々に存在する。

```
GCS 上の動画ファイル
    │
    │ ① BigQuery ObjectRef / Batch Prediction
    ▼
BigQuery 構造化テーブル
（タイトル・タグ・要約・カテゴリ・緊急度スコア など）
    │
    │ ② BigQuerySource でインポート
    ▼
Vertex AI Search Datastore
    │
    │ ③ 検索クエリ
    ▼
Gemini による要約・回答
```

---

## 主要概念の整理

### ObjectRef — 図書館の請求番号

動画ファイルは大きすぎて BigQuery テーブルに直接入れられない。
ObjectRef は動画の**実物ではなく「GCS 上のどこにあるか」という住所情報（ポインタ）**を BQ テーブルに格納する仕組み。
クエリ実行時に BQ が住所を見て GCS にアクセスし、Gemini に渡して解析する。

```
GCS（倉庫）
  └── gs://my-bucket/videos/001.mp4  ← 実物はここ

BigQuery テーブル（住所帳）
  ┌──────────┬─────────────────────────────────────────────┐
  │ ticket_id │ video_ref（ObjectRef）                     │
  ├──────────┼─────────────────────────────────────────────┤
  │ 001       │ { uri: "gs://my-bucket/videos/001.mp4",    │
  │           │   authorizer: "...",  ← GCS へのアクセス鍵 │
  │           │   content_type: "video/mp4" }              │
  └──────────┴─────────────────────────────────────────────┘
  ↑ ファイルの「住所」だけが入っている。実物は GCS に残ったまま。
```

SQL で `AI.GENERATE_TABLE(... video_ref ...)` と書くだけで BQ が並列処理してくれる。
動画のコピー不要・Python のループ不要。

#### 作業者が意識すべきこと

ObjectRef は「透過的に動く」部分と「明示的に意識すべき」部分が混在している。

**① Connection（通行証）は必ず事前に作る — 最も詰まりやすい箇所**

BigQuery が GCS や Vertex AI にアクセスするための認証設定。
一度作れば使い回せるが、**サービスアカウントへの権限付与を忘れると全クエリが失敗する**。

```sql
-- 1. Connection を作成（リージョンは Datastore と揃える）
CREATE CONNECTION `us.my_connection`
  OPTIONS (type = 'CLOUD_RESOURCE');

-- 2. 作成後にサービスアカウントを確認
--    → BigQuery コンソール > 外部接続 > サービスアカウント ID をコピー
```

```bash
# 3. そのサービスアカウントに GCS 読み取り権限を付与（必須）
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:SA_FROM_STEP2" \
  --role="roles/storage.objectViewer"

# 4. Vertex AI 呼び出し権限も付与（AI.GENERATE 系を使う場合）
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:SA_FROM_STEP2" \
  --role="roles/aiplatform.user"
```

**② ObjectRef の作り方は状況で使い分ける**

| 状況 | 使う方法 | 理由 |
|-----|---------|------|
| バケット内ファイルをまとめて扱いたい | Object Table | 自動でファイル一覧 + ObjectRef を生成してくれる |
| 既存テーブルに URI カラムがある | `OBJ.MAKE_REF` | URI 文字列を ObjectRef 型に変換するだけ |

**③ ObjectRef 列は VAIS には送らない**

ObjectRef は BigQuery 内部でのみ有効なポインタ。
VAIS に送るのは `AI.GENERATE_TABLE` が生成した**テキスト・数値の構造化出力だけ**。

```
BigQuery の reports_mm テーブル
  ┌──────────┬───────────┬──────────────┬─────────────┐
  │ ticket_id │ video_ref │ summary      │ urgency_score│
  │           │(ObjectRef)│（Gemini生成）│（Gemini生成）│
  └──────────┴───────────┴──────────────┴─────────────┘
                   │              │              │
                   ✗              ✓              ✓
              VAISに送らない   VAISに送る    VAISに送る
```

**④ mimeType は正確に指定する**

`AI.GENERATE_TABLE` や `OBJ.MAKE_REF` は mimeType を自動判定しない。
誤ると Gemini がファイルを正しく読めずエラーになる。

| ファイル種別 | mimeType |
|------------|---------|
| MP4動画 | `video/mp4` |
| MOV動画 | `video/quicktime` |
| MP3音声 | `audio/mp3` |
| JPEG画像 | `image/jpeg` |
| PDF | `application/pdf` |
| 種別不明の動画 | `video/*`（ワイルドカード） |

---

### Batch Prediction — Vertex AI の「まとめ処理モード」

**Batch Prediction は独立したサービスではなく、Vertex AI の機能のひとつ。**

```
Google Cloud
└── Vertex AI（AIプラットフォーム）
    ├── Online Prediction（リアルタイム、1件ずつ即時応答）
    └── Batch Prediction（バッチ、大量を非同期でまとめて処理）  ← これ
```

| | Online Prediction | Batch Prediction |
|---|---|---|
| 処理方式 | 同期（結果がすぐ返る） | 非同期（後で取りに行く） |
| 向いている規模 | 〜数十件 | 数百〜数千件 |
| コスト | 通常料金 | 割引あり（50%以上の場合も） |
| 結果が返る速さ | 即時 | 数分〜数時間 |
| 入出力 | API リクエスト/レスポンス | BigQuery テーブル or GCS |

「社員食堂で事前にまとめ注文して翌朝受け取る」イメージ。
個別に毎回 API を叩く代わりに、リクエストを BQ テーブルにまとめて投げ、完了後に結果テーブルを読みに行く。

---

## ステップ1-A: BigQuery ObjectRef による動画構造化（リアルタイム）

**参照ノートブック**: `gemini/use-cases/applying-llms-to-data/multimodal-analysis-bigquery/analyze_multimodal_data_bigquery.ipynb`

GCS ファイルを BigQuery の「ObjectRef」（ポインタ）として参照し、SQL の中から Gemini に解析させる方法。
GCS にファイルを置いたままクエリできるため、コピー不要。

### 事前準備: Cloud Resource Connection の作成

BigQuery が Vertex AI と GCS を呼び出すための認証設定。

```sql
-- BigQuery Studio / bq コマンドで実行
-- us リージョンに test_connection という名前で接続を作成
CREATE CONNECTION `us.test_connection`
  OPTIONS (type = 'CLOUD_RESOURCE');
```

### 方法A: Object Table（GCSバケット全体をテーブル化）

```sql
-- GCS バケット内のファイルを BigQuery テーブルとして直接マッピング
-- WITH CONNECTION で認証設定を参照し、各ファイルに ObjectRef が自動生成される
CREATE OR REPLACE EXTERNAL TABLE `my_dataset.video_object_table`
WITH CONNECTION `us.test_connection`
OPTIONS (
  object_metadata = 'SIMPLE',
  uris = ['gs://my-bucket/videos/*.mp4']
);
```

### 方法B: OBJ.MAKE_REF（URIカラムから動的にObjectRefを生成）

```sql
-- 既存テーブルの URI カラムを ObjectRef に変換してマルチモーダルテーブルを作成
CREATE OR REPLACE TABLE `my_dataset.reports_mm` AS
SELECT
  r.*,                                                                        -- 構造化データ（既存カラム）
  OBJ.FETCH_METADATA(OBJ.MAKE_REF(m.video_uri, 'us.test_connection')) AS video_ref,
  OBJ.FETCH_METADATA(OBJ.MAKE_REF(m.image_uri, 'us.test_connection')) AS image_ref
  --   ↑ GCS の URI 文字列を ObjectRef（ポインタ型）に変換
FROM `my_dataset.incidents` r
LEFT JOIN `my_dataset.media` m ON r.ticket_id = m.ticket_id;
```

### AI.GENERATE 系 SQL 関数で構造化データを抽出

BigQuery ML の Remote Model（Gemini）に対して SQL からマルチモーダル推論を実行する。

```sql
-- まず Remote Model を作成（1回のみ）
CREATE OR REPLACE MODEL `my_dataset.gemini`
  REMOTE WITH CONNECTION `us.test_connection`
  OPTIONS (endpoint = 'gemini-2.0-flash');
```

```sql
-- AI.GENERATE_TABLE: 複数フィールドの構造化出力を一括生成
-- テキスト・画像・動画・音声を同時に Gemini に渡せる
SELECT
  ticket_id,
  issue,
  urgency_score,
  city_response_department
FROM AI.GENERATE_TABLE(
  MODEL `my_dataset.gemini`,
  (
    SELECT
      (
        'Rate urgency 1-10 (10=critical). Assign a city department (Roads/Sanitation/Parks).',
        description,   -- テキストのメタデータも一緒に渡せる
        video_ref,     -- 動画の ObjectRef
        image_ref      -- 画像の ObjectRef
      ) AS prompt,
      ticket_id,
      description
    FROM `my_dataset.reports_mm`
  ),
  STRUCT(
    -- Gemini の出力を以下のスキーマで受け取る
    "issue STRING, urgency_score INT64, city_response_department STRING" AS output_schema
  )
);
```

```sql
-- AI.GENERATE_BOOL: 条件フィルタリング（WHERE 句で使用可能）
-- 「買う意向がある」通話だけに絞り込む
SELECT company_name, customer_name
FROM `my_dataset.calls_combined`
WHERE company_revenue > 15000000
  AND AI.GENERATE_BOOL(
    prompt => ('Wants to buy something', ref),
    connection_id => 'us.test_connection'
  ).result;
```

```sql
-- AI.GENERATE + ARRAY_AGG: グループ単位でまとめて解析
-- 地区ごとに複数動画を束ねて Gemini に投げる
SELECT district, summary
FROM AI.GENERATE_TABLE(
  MODEL `my_dataset.gemini`,
  (
    SELECT
      ('Summarize the top issues needing attention in this district',
        ARRAY_AGG(description),
        ARRAY_AGG(video_ref),
        ARRAY_AGG(image_ref)
      ) AS prompt,
      district
    FROM `my_dataset.reports_mm`
    GROUP BY district
  ),
  STRUCT("summary ARRAY<STRING>" AS output_schema)
);
```

---

## ステップ1-B: Batch Prediction による大量動画の一括構造化

**参照ノートブック**: `gemini/use-cases/video-analysis/video_analysis_with_youtube_data_api_and_batch_prediction.ipynb`

動画が大量にある場合（数百〜数千本）は Vertex AI の Batch Prediction 機能を使う。
処理の流れは「注文書作成 → まとめ発注 → 結果受け取り」の3段階。

```
① 注文書作成（Python）
   動画1本ぶんの「何を・どう解析してほしいか」を JSON で定義
   → DataFrame の各行に格納

② まとめ発注（BQ → Gemini → BQ）
   注文書テーブルを BQ に書き込み、BatchPredictionJob.submit() を呼ぶだけ
   → 非同期で処理開始。コードはすぐ次の行へ進む

③ 結果受け取り（BQ テーブルを読む）
   job.has_ended が True になったら BQ の結果テーブルを参照
   → 構造化データ（要約・カテゴリ・スコア等）が全件入っている
```

入力・出力ともに BigQuery テーブルを指定できる。

### Gemini へのリクエスト形式（動画1本ぶん）を定義

```python
video_extraction_response_schema = {
    "type": "array",
    "items": {
        "type": "object",
        "properties": {
            "summary":    {"type": "string"},   # 動画の要約テキスト
            "category":   {"type": "string"},   # カテゴリ
            "tags": {
                "type": "array",
                "items": {"type": "string"},    # タグリスト
            },
            "urgency_score": {"type": "integer"},  # 緊急度スコア
        },
    },
}

def build_batch_request(video_gcs_uri: str) -> str:
    """1本の動画に対する Gemini Batch Prediction リクエストを JSON 文字列で返す"""
    return json.dumps({
        "system_instruction": {"parts": [{"text": "動画を詳しく分析してください。"}]},
        "contents": [{
            "role": "user",
            "parts": [
                {"text": "動画の要約・カテゴリ・タグ・緊急度スコアを抽出してください"},
                {"file_data": {"mimeType": "video/*", "fileUri": video_gcs_uri}},
                #                                        ↑ GCS の gs:// URI を直接指定
            ],
        }],
        "generation_config": {
            "response_mime_type": "application/json",
            "response_schema": video_extraction_response_schema,
        },
    })
```

### リクエストを BigQuery に書き込み → Batch Prediction 実行

```python
from google.cloud import bigquery
from vertexai.batch_prediction import BatchPredictionJob

BQ_CLIENT = bigquery.Client(project=PROJECT_ID)

# ① リクエストテーブルを BigQuery に書き込む
#    DataFrame の "request" カラムに各動画のリクエスト JSON を格納
videos_df["request"] = videos_df["gcs_uri"].apply(build_batch_request)
BQ_CLIENT.load_table_from_dataframe(
    videos_df,
    f"{DATASET}.batch_requests",
    job_config=bigquery.LoadJobConfig(write_disposition="WRITE_TRUNCATE"),
).result()

# ② Batch Prediction を起動（BQ → BQ）
job = BatchPredictionJob.submit(
    source_model="gemini-2.0-flash",
    input_dataset=f"bq://{PROJECT_ID}.{DATASET}.batch_requests",   # 入力: BQ テーブル
    output_uri_prefix=f"bq://{PROJECT_ID}.{DATASET}.batch_results", # 出力: BQ テーブル
)

# ③ 完了を待つ
while not job.has_ended:
    job.refresh()
print("完了" if job.has_succeeded else f"失敗: {job.error}")
```

**ObjectRef 方式 vs Batch Prediction 方式の選択**

| 観点 | ObjectRef + AI.GENERATE | Batch Prediction |
|-----|------------------------|-----------------|
| 向いている規模 | 数本〜数十本（インタラクティブ） | 数百〜数千本（非同期） |
| 実行方法 | SQL（BigQuery Studio から直接） | Python SDK |
| コスト効率 | クエリごと課金 | バッチ割引あり |
| 結果の即時性 | 即時 | 完了に数分〜数時間 |

---

## ステップ2: BigQuery → Vertex AI Search へのインポート

**参照ファイル**: `gemini/sample-apps/genwealth/function-scripts/update-search-index/main.py`

BigQuery テーブルを Vertex AI Search の Datastore に取り込む。
`data_schema="custom"` を指定することで任意のスキーマを受け付ける。

```python
from google.api_core.client_options import ClientOptions
from google.cloud import discoveryengine

def import_from_bigquery(
    project_id: str,
    location: str,
    data_store_id: str,
    bq_dataset: str,
    bq_table: str,
) -> str:
    client = discoveryengine.DocumentServiceClient(
        client_options=(
            ClientOptions(api_endpoint=f"{location}-discoveryengine.googleapis.com")
            if location != "global" else None
        )
    )

    operation = client.import_documents(
        request=discoveryengine.ImportDocumentsRequest(
            parent=client.branch_path(project_id, location, data_store_id, "default_branch"),
            bigquery_source=discoveryengine.BigQuerySource(
                project_id=project_id,
                dataset_id=bq_dataset,   # "my_dataset"
                table_id=bq_table,       # "batch_results"（構造化済みテーブル）
                data_schema="custom",    # 任意スキーマを許容
            ),
            # INCREMENTAL: 差分追加（新しいレコードだけ追加・更新）
            # FULL: 全件洗い替え
            reconciliation_mode=discoveryengine.ImportDocumentsRequest.ReconciliationMode.INCREMENTAL,
        )
    )
    operation.result()  # 完了を待つ
    return operation.operation.name
```

### BigQuery テーブルのスキーマ設計

VAIS に取り込むテーブルには `id` フィールドが必要。
検索・フィルタリングに使うフィールドを明示的に用意しておくと後でブースティング・ファセットに使える。

```
BigQuery テーブルスキーマ例（動画コンテンツ向け）

id            STRING    -- ドキュメント一意ID（必須）
title         STRING    -- 動画タイトル
gcs_uri       STRING    -- 元動画のGCSパス
summary       STRING    -- Geminiが生成した要約（全文検索の対象）
category      STRING    -- カテゴリ（フィルタリング・ファセットに使用）
tags          ARRAY<STRING> -- タグ（フィルタリングに使用）
urgency_score INT64     -- 緊急度スコア（ブースティングに使用）
created_at    TIMESTAMP -- 取り込み日時
```

### Cloud Functions でイベント駆動の自動更新

`gemini/sample-apps/genwealth` の実装パターン: GCS にファイルが追加されたら
Cloud Functions が起動し、自動的に VAIS インデックスを更新する。

```python
import functions_framework

@functions_framework.cloud_event
def update_search_index(cloud_event):
    """GCS への書き込みイベントをトリガーに VAIS を自動更新"""
    data = cloud_event.data
    # 新しいファイルが GCS に追加されたら BQ→VAIS インポートを実行
    import_from_bigquery(
        project_id=os.environ["PROJECT_ID"],
        location="global",
        data_store_id=os.environ["DATASTORE_ID"],
        bq_dataset=os.environ["BQ_DATASET"],
        bq_table=os.environ["BQ_TABLE"],
    )
```

---

## ステップ3: 検索クエリ実行

BigQuery 由来の構造化フィールドを活用したフィルタリング・ブースティングが効く。

```python
from google.cloud import discoveryengine

search_client = discoveryengine.SearchServiceClient()

response = search_client.search(
    discoveryengine.SearchRequest(
        serving_config=(
            f"projects/{PROJECT_ID}/locations/global/collections/default_collection"
            f"/engines/{ENGINE_ID}/servingConfigs/default_search"
        ),
        query="道路の陥没 緊急対応",
        page_size=10,
        # BigQuery で生成した urgency_score を使ってブースティング
        boost_spec=discoveryengine.SearchRequest.BoostSpec(
            condition_boost_specs=[
                discoveryengine.SearchRequest.BoostSpec.ConditionBoostSpec(
                    condition="urgency_score>=8",  # ← BQ の AI.GENERATE 生成フィールド
                    boost=0.9,
                )
            ]
        ),
        # category フィールドでフィルタリング
        filter='category: ANY("Roads", "Sanitation")',
        content_search_spec=discoveryengine.SearchRequest.ContentSearchSpec(
            summary_spec=discoveryengine.SearchRequest.ContentSearchSpec.SummarySpec(
                summary_result_count=5,
                include_citations=True,
            ),
        ),
    )
)

print(response.summary.summary_text)
for r in response.results:
    print(r.document.derived_struct_data.get("title"), "-", r.document.derived_struct_data.get("urgency_score"))
```

---

## パイプライン選択ガイド

### ObjectRef vs Batch Prediction — 根本的な違い

| | ObjectRef + AI.GENERATE | Batch Prediction |
|---|---|---|
| 何をするか | BQ の SQL の中から Gemini を呼ぶ | Vertex AI に処理をまるごと委託する |
| 動画の指定方法 | ObjectRef（BQ テーブルの列） | GCS URI を JSON に直書き |
| コードの複雑さ | SQL だけで書ける | Python で注文書 JSON を作る必要あり |
| 処理方式 | 同期（SQL が返るまで待つ） | 非同期（ジョブを投げて後で取りに行く） |
| 向いている本数 | 〜数十本 | 数百〜数千本 |
| コスト | 通常料金 | 割引あり |

### 使い分けフロー

```
動画の本数は？
    ├── 少ない（〜数十本）または対話的に処理したい
    │       ↓
    │   BigQuery ObjectRef + AI.GENERATE_TABLE
    │   SQL だけで完結。BQ Studio から実行できる。
    │   → `analyze_multimodal_data_bigquery.ipynb`
    │
    └── 多い（数百本以上）または夜間バッチで処理したい
            ↓
        Vertex AI Batch Prediction
        Python で注文書を作って BQ に書き → ジョブを投げて待つ
        → `video_analysis_with_youtube_data_api_and_batch_prediction.ipynb`

どちらの場合も BigQuery に構造化テーブルが出来上がったら
    ↓
BigQuerySource で VAIS Datastore にインポート（ステップ2）
    ↓
urgency_score でブースト・category でフィルタ + Gemini 要約（ステップ3）
```

---

## 主要リソース（このリポジトリ内）

| ファイル | ステップ | 内容 |
|---------|---------|------|
| `gemini/use-cases/applying-llms-to-data/multimodal-analysis-bigquery/analyze_multimodal_data_bigquery.ipynb` | ① | ObjectRef + AI.GENERATE による動画・音声・画像の構造化 |
| `gemini/use-cases/video-analysis/video_analysis_with_youtube_data_api_and_batch_prediction.ipynb` | ① | Batch Prediction で動画を大量処理 → BigQuery 保存 |
| `gemini/sample-apps/genwealth/function-scripts/update-search-index/main.py` | ② | BigQuerySource で VAIS にインポート（Cloud Functions 実装） |
| `search/search_data_blending_with_gemini_summarization.ipynb` | ③ | 複数 Datastore 横断検索 + Gemini 要約 |
| `search/vais-building-blocks/query_level_boosting_filtering_and_facets.ipynb` | ③ | ブースティング・フィルタリング・ファセットの詳細 |
