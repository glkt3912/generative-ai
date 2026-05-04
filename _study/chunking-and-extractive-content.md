# Chunking と Extractive Content（VAIS での正式名称）

## 用語の全体マップ

VAIS における「文書分割」と「回答抽出」は、処理タイミングが異なる2つのフェーズに分かれる。

```
【インジェスト時】                     【クエリ時】
PDF / HTML / Word
    ↓ Parsing（パーシング）
テキスト化
    ↓ Chunking（チャンキング）        →  検索
チャンク単位でインデックス化               ↓
                                    Snippet（スニペット）          ← 短いプレビュー
                                    Extractive Answer（抽出回答）  ← 1〜2文の直接回答
                                    Extractive Segment（抽出区間） ← パラグラフ単位の長い抜粋
                                    LLM Summary（要約）            ← Gemini が生成
```

---

## フェーズ1: Chunking（チャンキング）— インジェスト時

**参照ノートブック**: `search/vais-building-blocks/parsing_and_chunking_with_BYO.ipynb`

### なぜ Chunking が必要か

Gemini のコンテキストウィンドウは有限で、200ページのPDF全体を1回のプロンプトに入れると精度が落ちる。
Chunking はドキュメントを意味的なまとまり（チャンク）に分割してインデックス化することで、
クエリに関係する箇所だけを取り出して Gemini に渡せるようにする。

### Datastore 作成時に Chunk Mode を指定する

```python
# Chunk Mode は Datastore 作成時に決定する。後から変更不可。
payload = {
    "displayName": "my-datastore",
    "industryVertical": "GENERIC",
    "solutionTypes": ["SOLUTION_TYPE_SEARCH"],
    "contentConfig": "CONTENT_REQUIRED",
    "documentProcessingConfig": {
        "chunkingConfig": {
            "layoutBasedChunkingConfig": {
                "chunkSize": 500,               # 1チャンクの最大トークン数（推奨: 500）
                "includeAncestorHeadings": True, # 「第2章 > 2.3節」の見出し情報をチャンクに付加
            }
        },
        "defaultParsingConfig": {"layoutParsingConfig": {}},
        # layoutParsingConfig: 表・リスト・見出し構造を理解する高精度パーサー
    },
}
```

### パーサーの選択

| パーサー | 向いている文書 | 精度 vs 速度 |
|---------|-------------|------------|
| `layoutParsingConfig` | 表・図・リストが多い複雑なPDF | 精度優先 |
| `digitalParsingConfig` | シンプルな文章中心のPDF | 速度優先 |
| `ocrParsingConfig` | スキャン文書（画像PDF） | OCR処理が入る |

### チャンクの内部構造を確認する

インポート済みドキュメントのチャンク分割結果は API で取得できる。

```python
# Parsed Document（パーシング結果）を取得
url = (
    f"https://discoveryengine.googleapis.com/v1alpha/projects/{project_id}"
    f"/locations/global/collections/default_collection"
    f"/dataStores/{data_store_id}/branches/0/documents/{document_id}"
    f":getProcessedDocument?processed_document_type=PARSED_DOCUMENT"
)
parsed = authed_session.get(url).json()

# Chunked Document（チャンキング結果）を取得
url = (
    f"https://discoveryengine.googleapis.com/v1alpha/projects/{project_id}"
    f"/locations/global/collections/default_collection"
    f"/dataStores/{data_store_id}/branches/0/documents/{document_id}"
    f":getProcessedDocument?processed_document_type=CHUNKED_DOCUMENT"
)
chunked = authed_session.get(url).json()

# チャンクの構造（jsonData.chunks の各要素）
# {
#   "id": "doc-001_chunk_0003",
#   "content": "3.2節 製品概要\n本製品は...",   ← includeAncestorHeadings で見出しが付加される
#   "documentMetadata": { "uri": "gs://...", "title": "製品マニュアル" }
# }

# チャンクを順番に繋げてドキュメントを再構築する例
for chunk in chunked["jsonData"]["chunks"]:
    print(f"--- chunk: {chunk['id']} ---")
    print(chunk["content"])
```

### BYOC（Bring Your Own Chunks）— 自分のチャンクを持ち込む

VAISが自動生成したチャンクを取得・編集して、カスタムチャンクを再投入できる。
見出し検出の修正・追加コンテキストの付加・特定テンプレートへの対応などに使う。

> **注意**: BYOC は執筆時点でプライベートプレビュー。利用にはアカウントチームへの申請が必要。

```python
# 1. VAIS が生成したチャンクを取得して編集
chunked_doc = get_chunked_document(project_id, data_store_id, document_id)

# 2. チャンクを編集（例: 最初のチャンクに追加情報を付加）
chunked_doc["jsonData"]["chunks"][0]["content"] = (
    "[補足] この文書は2024年版です。\n" + chunked_doc["jsonData"]["chunks"][0]["content"]
)

# 3. 編集済みチャンクを GCS にアップロード
upload_json_to_gcs(bucket_name=GCS_BUCKET, file_name="my_chunks.json", json_data=chunked_doc)

# 4. VAIS にインポート（INCREMENTAL で既存文書を上書き）
payload = {
    "reconciliationMode": "INCREMENTAL",
    "gcsSource": {"inputUris": [f"gs://{GCS_BUCKET}/my_chunks.json"], "dataSchema": "content"},
}
authed_session.post(
    f"https://discoveryengine.googleapis.com/v1alpha/projects/{project_id}"
    f"/locations/global/collections/default_collection"
    f"/dataStores/{data_store_id}/branches/default_branch/documents:import",
    json=payload,
)
```

---

## フェーズ2: Extractive Content（抽出コンテンツ）— クエリ時

**参照ファイル**: `search/web-app/vais_utils.py`, `search/create_datastore_and_search.ipynb`

クエリ時の `ContentSearchSpec` で指定する。Snippet・Extractive Answer・Extractive Segment・Summary の4種類が返せる。

### 4種類の出力の違い

| 種類 | 長さ | 生成方法 | 用途 |
|-----|-----|---------|------|
| **Snippet** | 数十文字 | インデックスから抽出（クエリ語句をハイライト） | 検索結果一覧のプレビュー |
| **Extractive Answer** | 1〜2文 | ドキュメントから直接抜粋 | チャットUIでの即答 |
| **Extractive Segment** | パラグラフ単位 | ドキュメントから直接抜粋（より長い） | 詳細な参照文脈の提示 |
| **LLM Summary** | 自由長 | Gemini が複数結果を統合して生成 | 最終回答（Enterprise Tier が必要） |

### ContentSearchSpec の実装

```python
from google.cloud import discoveryengine

content_search_spec = discoveryengine.SearchRequest.ContentSearchSpec(

    # Snippet: 検索結果一覧のプレビューテキスト（HTMLハイライト付き）
    snippet_spec=discoveryengine.SearchRequest.ContentSearchSpec.SnippetSpec(
        return_snippet=True,
    ),

    # Extractive Answer: 1〜2文の直接回答（Enterprise Tier 必須）
    # Extractive Segment: パラグラフ単位の長い抜粋（Enterprise Tier 必須）
    extractive_content_spec=discoveryengine.SearchRequest.ContentSearchSpec.ExtractiveContentSpec(
        max_extractive_answer_count=1,   # 0〜5 件（0=無効）
        max_extractive_segment_count=1,  # 0〜5 件（0=無効）
    ),

    # LLM Summary: Gemini による統合要約（Enterprise Tier + LLM Add-on 必須）
    summary_spec=discoveryengine.SearchRequest.ContentSearchSpec.SummarySpec(
        summary_result_count=5,           # 要約に使う検索結果の件数
        include_citations=True,           # 引用元（[1][2]）を含める
        ignore_adversarial_query=True,    # 敵対的なクエリを無視
        ignore_non_summary_seeking_query=True,  # 要約に向かないクエリを無視
        model_prompt_spec=discoveryengine.SearchRequest.ContentSearchSpec.SummarySpec.ModelPromptSpec(
            preamble="あなたは社内文書の専門家です。以下の情報に基づいて回答してください。",
        ),
        model_spec=discoveryengine.SearchRequest.ContentSearchSpec.SummarySpec.ModelSpec(
            version="stable",  # "stable" | "preview"
        ),
    ),
)
```

### レスポンスから各種コンテンツを取り出す

```python
response = search_client.search(
    discoveryengine.SearchRequest(
        serving_config=serving_config,
        query="CEOは誰ですか？",
        content_search_spec=content_search_spec,
    )
)

# LLM Summary（Gemini による統合要約）
print(response.summary.summary_text)

# 各検索結果ごとの抽出コンテンツ
for result in response.results:
    data = result.document.derived_struct_data

    # Snippet（プレビュー、HTMLタグ除去が必要なことがある）
    snippets = [s.get("htmlSnippet", s.get("snippet", ""))
                for s in data.get("snippets", [])]

    # Extractive Answer（短い直接回答）
    extractive_answers = [e["content"]
                          for e in data.get("extractive_answers", [])]

    # Extractive Segment（パラグラフ単位の長い抜粋）
    extractive_segments = [e["content"]
                           for e in data.get("extractive_segments", [])]
```

### LangChain 経由での利用（簡略版）

```python
from langchain_google_community import VertexAISearchRetriever

retriever = VertexAISearchRetriever(
    project_id=PROJECT_ID,
    location_id="global",
    data_store_id=DATASTORE_ID,
    get_extractive_answers=True,       # Extractive Answer を取得
    max_documents=10,
    max_extractive_answer_count=5,     # 最大5件
    max_extractive_segment_count=1,    # Extractive Segment も1件取得
)
docs = retriever.invoke("返品ポリシーは？")
```

---

## Chunk Mode の有無による挙動の違い

| 観点 | Chunk Mode あり | Chunk Mode なし（Document Mode） |
|-----|---------------|-------------------------------|
| インデックスの単位 | チャンク（500トークン前後） | ドキュメント全体 |
| Extractive Answer の精度 | 高い（関連チャンクから抽出） | 低め（全文検索） |
| LLM Summary の精度 | 高い | 普通 |
| BYOC | 可能 | 不可 |
| Datastore 作成後の変更 | 不可（再作成が必要） | 不可 |

---

## 使い分けの判断フロー

```
Q. 検索結果に「回答の根拠となる文章」を表示したい？
    ├── 1〜2文の短い直接回答が欲しい
    │       → Extractive Answer（max_extractive_answer_count=1〜5）
    │
    ├── 前後の文脈も含めたパラグラフ単位で欲しい
    │       → Extractive Segment（max_extractive_segment_count=1〜5）
    │
    └── Gemini に統合・言い換えさせた回答が欲しい
            → LLM Summary（Enterprise Tier + LLM Add-on）

Q. 精度を上げるためにチャンク分割を最適化したい？
    ├── VAIS のチャンクをそのまま使う
    │       → Chunk Mode Datastore（chunkSize, includeAncestorHeadings を調整）
    │
    └── 自前でチャンクを加工・補完したい
            → BYOC（プライベートプレビュー、申請が必要）
```

---

## 運用上の注意点

### Chunk Mode は作成後に変更不可

チャンク設定（`chunkSize`・`includeAncestorHeadings`・パーサー種別）は Datastore 作成時に確定する。
変更したい場合は Datastore を削除して作り直す必要がある。
本番移行前に小規模なデータで精度を確認してから設定を決める。

### Extractive Answer / Segment は Enterprise Tier 必須

Standard Tier で `max_extractive_answer_count > 0` を指定するとエラーではなく**空のリストが返る**（サイレント失敗）。
「Extractive Answer が返ってこない」場合は Tier を最初に確認する。

```python
# Tier の確認（コンソール or API）
# Datastore の詳細画面 > 「検索層」が "Enterprise" になっているか確認

# サイレント失敗の例
extractive_answers = data.get("extractive_answers", [])
# → Standard Tier だと常に [] が返る。エラーは出ない。
```

### chunkSize の選び方

小さすぎると文脈が途切れ、大きすぎると Gemini のコンテキストを無駄に占有する。

| chunkSize | 向いているケース |
|-----------|--------------|
| 200〜300 | 短い FAQ・箇条書き中心のドキュメント |
| **500（推奨）** | 一般的な社内文書・マニュアル |
| 800〜1000 | 長い段落・技術文書・法律文書 |

`includeAncestorHeadings: True` を使うと見出し情報がチャンクに付加されるため、
「どの章の内容か」の文脈が保たれ、Extractive Answer の精度が上がる。

### BYOC のトークン制限

自前チャンクを投入する場合、Datastore 作成時に指定した `chunkSize` を超えるチャンクは**インポートエラーになる**。
編集後のチャンクが元の `chunkSize` 以下に収まっているか事前に確認する。

---

## 主要リソース（このリポジトリ内）

| ファイル | 内容 |
|---------|------|
| `search/vais-building-blocks/parsing_and_chunking_with_BYO.ipynb` | Chunk Mode Datastore 作成・チャンク確認・BYOC の全フロー |
| `search/vais-building-blocks/ingesting_unstructured_documents_with_metadata.ipynb` | Chunk Mode + メタデータ付きインジェスト |
| `search/web-app/vais_utils.py` | Snippet・Extractive Answer・Extractive Segment を取得する実装 |
| `search/vertexai-search-options/vertexai_search_options.ipynb` | LangChain 経由での Extractive Answer 利用 |
| `search/create_datastore_and_search.ipynb` | ExtractiveContentSpec の基本的な使い方 |
