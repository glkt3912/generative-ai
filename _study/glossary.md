# 用語集（Gemini / Vertex AI の概念・型一覧）

## SDK・API 基本型

| 型 / クラス | パッケージ | 説明 |
|------------|----------|------|
| `GenerativeModel` | `vertexai.generative_models` | Gemini モデルのメインクラス。`generate_content()` / `start_chat()` を提供 |
| `ChatSession` | `vertexai.generative_models` | マルチターン会話を管理。`chat.history` で履歴参照 |
| `GenerationResponse` | `vertexai.generative_models` | `generate_content()` の戻り値。`response.text` で本文取得 |
| `Candidate` | `vertexai.generative_models` | 1つの回答候補。`finish_reason` / `content` / `grounding_metadata` を持つ |
| `Content` | `vertexai.generative_models` | ロール（user/model）と Parts のセット |
| `Part` | `vertexai.generative_models` | テキスト・画像・FunctionCall・FunctionResponse などのコンテンツ単位 |
| `GenerationConfig` | `vertexai.generative_models` | `temperature` / `top_p` / `max_output_tokens` / `thinking_config` などを設定 |

---

## Function Calling 関連

| 型 / 概念 | 説明 |
|----------|------|
| `FunctionDeclaration` | ツールの JSON Schema 定義。`name` / `description` / `parameters` を持つ |
| `Tool` | 複数の `FunctionDeclaration` をまとめたオブジェクト |
| `ToolConfig` | Function Calling モード（AUTO / ANY / NONE）を設定 |
| `FunctionCall` | Gemini が返す「このツールを呼んでほしい」という指示。`name` / `args` を持つ |
| `FunctionResponse` | アプリがツール実行結果を Gemini に返すための型。`Part.from_function_response()` で生成 |
| Forced Function Calling | `ToolConfig.Mode.ANY` でツール呼び出しを強制するパターン |
| Parallel Function Calling | 1ターンで複数の `FunctionCall` を同時に返す機能 |

---

## Grounding・RAG 関連

| 型 / 概念 | 説明 |
|----------|------|
| `GoogleSearchRetrieval` | Google Search を使ったグラウンディングツール |
| `GroundingMetadata` | 回答の根拠情報。`grounding_chunks`（引用元）と `grounding_supports`（文章の対応）を含む |
| `GroundingChunk` | 引用元の情報。`web.uri` / `web.title` を持つ |
| `GroundingSupport` | テキストの特定範囲と引用元の対応関係。`segment` / `grounding_chunk_indices` / `confidence_scores` |
| RAG Corpus | Vertex AI RAG Engine のベクトルDB。文書をインポートして構築する |
| Embedding | テキストを多次元ベクトルに変換した表現。意味的類似度の計算に使用 |
| Chunk / Chunking | 長文書を小さな単位に分割する処理。`chunk_size` / `chunk_overlap` で制御 |
| Top-K | 類似度検索で上位 K 件を取得するパラメータ |

---

## Thinking 関連

| 型 / 概念 | 説明 |
|----------|------|
| `ThinkingConfig` | `thinking_budget`（0〜最大値、または -1で自動）を設定 |
| `thought` | `Part.thought == True` のとき、その Part は最終回答ではなく内部思考過程 |
| Thought Signature | マルチターンで思考を引き継ぐためのシリアライズされた思考トークン（Base64） |
| `thoughts_token_count` | `usage_metadata` の中の思考トークン数。課金対象 |
| thinking_budget | 思考に使えるトークンの上限。0=無効, -1=自動, 正整数=上限設定 |

---

## Context Caching 関連

| 型 / 概念 | 説明 |
|----------|------|
| `CachedContent` | キャッシュされたコンテンツを表すオブジェクト。`create()` / `update()` / `delete()` / `list()` を持つ |
| TTL (Time To Live) | キャッシュの有効期間。最大 1 時間。`datetime.timedelta` で指定 |
| キャッシュヒット | クエリ時にキャッシュが使用されること。通常入力の 25% のコストで処理される |
| キャッシュミス | キャッシュが存在しないか期限切れの場合。通常コストで処理される |
| 最小キャッシュサイズ | 32,768 tokens（それ未満はキャッシュ作成不可） |

---

## Agents 関連

| 型 / 概念 | 説明 |
|----------|------|
| `reasoning_engines.ReasoningEngine` | Vertex AI のマネージドエージェント実行環境 |
| `reasoning_engines.Queryable` | ReasoningEngine にデプロイするエージェントクラスの基底。`set_up()` / `query()` を実装 |
| ReAct | Reasoning and Acting の略。「思考→行動→観察→再思考」のループパターン |
| Orchestrator Agent | マルチエージェント構成でタスクを分解・割り振る中心エージェント |
| Memory (Short-term) | `ChatSession.history` で管理される会話内の文脈 |
| Memory (Long-term) | Firestore / Bigtable / Vector Search で永続化された記憶 |
| Always-on Memory | 会話をまたいで記憶を保持し続けるエージェントパターン |

---

## モデル・料金関連

| 概念 | 説明 |
|-----|------|
| Token | LLM が処理する最小単位。英語 ≈ 4文字、日本語 ≈ 1-2文字 |
| Context Window | 一度のリクエストで処理できる最大トークン数（Flash: 1M, Pro: 2M） |
| Input Token | プロンプト・画像・動画など、入力として送った情報のトークン数 |
| Output Token | モデルが生成したテキストのトークン数 |
| `finish_reason` | 生成が止まった理由。`STOP`（正常完了）/ `MAX_TOKENS`（上限到達）/ `SAFETY`（安全フィルタ） |
| `temperature` | 出力のランダム性（0.0=決定的, 1.0=創造的, 2.0=最大ランダム） |
| `top_p` | 累積確率でサンプリング候補を絞る（0.95 が一般的） |
| `top_k` | 上位 K 件のトークンからサンプリング |

---

## Vertex AI インフラ関連

| 概念 | 説明 |
|-----|------|
| Project | GCP プロジェクト。課金・IAM の単位 |
| Region / Location | モデルのエンドポイントが存在するリージョン（例: `us-central1`）。近いリージョンを選ぶとレイテンシが下がる |
| Staging Bucket | Reasoning Engine デプロイ時に使う GCS バケット |
| Service Account | GCP リソースへのアクセス権限を持つサービスアカウント |
| ADC (Application Default Credentials) | ローカル開発での認証方式（`gcloud auth application-default login`） |
| Vertex AI API | 有効化が必要な API（`gcloud services enable aiplatform.googleapis.com`） |
