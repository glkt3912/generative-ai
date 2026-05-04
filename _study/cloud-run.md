# Cloud Run と Cloud Run Jobs

## 2つの違い

Cloud Run と Cloud Run Jobs は同じサービス群だが、動作モデルが根本的に異なる。

| | **Cloud Run** | **Cloud Run Jobs** |
|---|---|---|
| 動作モデル | HTTP サーバーとして常駐し、リクエストを待つ | 処理を完了したら終了（exit 0） |
| 起動トリガー | HTTP リクエスト | `gcloud run jobs execute` / Cloud Scheduler |
| スケーリング | リクエスト数に応じて自動スケール | タスク並列数（`--tasks`）を指定 |
| タイムアウト上限 | 60分 | 168時間 |
| 課金 | リクエスト処理時間 | ジョブ実行時間 |
| 向いている処理 | API サーバー・チャット UI | バッチ処理・定期ジョブ・長時間処理 |

---

## Cloud Run — HTTP サーバー型

### このリポジトリでの用途

Gemini を使ったアプリの**デプロイ先**として使われている。フロントエンドは問わず、Streamlit・Quart・Mesop など複数のフレームワーク例がある。

| ディレクトリ | フレームワーク | 内容 |
|---|---|---|
| `gemini/sample-apps/gemini-streamlit-cloudrun/` | Streamlit | Gemini マルチモーダル UI |
| `gemini/sample-apps/gemini-quart-cloudrun/` | Quart（非同期 Flask） | ストリーミング対応チャット |
| `gemini/sample-apps/gemini-mesop-cloudrun/` | Mesop | Google 製 UI フレームワーク |
| `gemini/sample-apps/llamadeploy-on-cloud-run/` | LlamaIndex | エージェント API サーバー |
| `agents/cloud_run/agents_with_memory/` | ADK | Memory Bank 付きエージェント |
| `open-models/serving/cloud_run_ollama_gemma3_inference.ipynb` | Ollama | Gemma3 推論エンドポイント（GPU） |
| `open-models/serving/cloud_run_vllm_gemma3_inference.ipynb` | vLLM | Gemma3 高速推論（GPU） |

### 共通のデプロイパターン

**Dockerfile（Streamlit の例）**

```dockerfile
FROM python:3.13-slim

EXPOSE 8080          # Cloud Run はポート 8080 を要求
WORKDIR /app
COPY . ./
RUN pip install --no-cache-dir -r requirements.txt

# Cloud Run は 0.0.0.0:8080 でリッスンするサーバーを期待する
ENTRYPOINT ["streamlit", "run", "app.py", "--server.port=8080", "--server.address=0.0.0.0"]
```

**ビルド & デプロイ（gcloud コマンド）**

```bash
PROJECT_ID=$(gcloud config get-value project)
REGION=us-central1
IMAGE=gcr.io/$PROJECT_ID/my-gemini-app

# 1. コンテナイメージをビルドして Container Registry に push
gcloud builds submit --tag $IMAGE

# 2. Cloud Run にデプロイ
gcloud run deploy my-gemini-app \
  --image $IMAGE \
  --platform managed \
  --region $REGION \
  --allow-unauthenticated \
  --set-env-vars=PROJECT_ID=$PROJECT_ID \
  --set-env-vars=LOCATION=$REGION
```

**GPU 付き推論エンドポイントの場合（Ollama/vLLM）**

```bash
gcloud run deploy ollama-gemma3 \
  --image $IMAGE \
  --region $REGION \
  --gpu 1 \
  --gpu-type nvidia-l4 \
  --max-instances 1 \          # L4 GPU はクォータ制限が厳しい
  --memory 16Gi \
  --cpu 4 \
  --no-allow-unauthenticated   # GPU サービスは認証を必須にする
```

### 認証トークンの取得（GPU サービスへのリクエスト）

```bash
# Identity Token を取得してリクエストに付与する
SERVICE_URL=$(gcloud run services describe my-service --region $REGION --format 'value(status.url)')
AUTH_TOKEN=$(gcloud auth print-identity-token)

curl -X POST "$SERVICE_URL/api/generate" \
  -H "Authorization: Bearer $AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"model": "gemma3", "prompt": "Hello", "stream": false}'
```

---

## Cloud Run Jobs — バッチ型

### このリポジトリでの用途

**`search/gemini-enterprise/group-licensing/`** が唯一の本格実装。  
Go 言語で書かれた Gemini Enterprise ライセンス管理ユーティリティ（"Gemini Box Office"）。

1つのコンテナイメージから環境変数 `JOB_TYPE` で処理を切り替え、**2種類のジョブ**を Cloud Scheduler で定期実行する。

```
Cloud Scheduler（毎日）
    ↓
gemini-box-office-joiner（JOB_TYPE=joiner）
    → グループメンバーを確認 → ライセンスを付与

Cloud Scheduler（6時間ごと）
    ↓
gemini-box-office-gc（JOB_TYPE=garbage_collection）
    → 非アクティブユーザーを確認 → ライセンスを剥奪
```

### Cloud Run Jobs 固有の環境変数

Cloud Run Jobs ランタイムが自動的に注入する変数。並列タスク設計で使う。

| 変数 | 説明 | デフォルト |
|-----|------|---------|
| `CLOUD_RUN_TASK_INDEX` | このタスクの 0 始まりインデックス | `0` |
| `CLOUD_RUN_TASK_COUNT` | 並列タスクの総数 | `1` |

### 並列シャーディングの実装パターン（Go）

処理対象のプロジェクト一覧を `CLOUD_RUN_TASK_COUNT` で割り、
`CLOUD_RUN_TASK_INDEX` に対応する分だけを各タスクが担当する。

```go
// ソート済みのプロジェクトキー一覧を取得（全タスクで順序を統一）
sortedKeys := slices.SortedFunc(maps.Keys(cfg.Projects), cmp.Compare[string])

// 自分のタスクが担当する分だけを抽出
partitioned := make(map[string]config.ProjectConfig)
for i, key := range sortedKeys {
    if i%settings.TaskCount == settings.TaskIndex {
        // i を TaskCount で割った余りが自分の TaskIndex に一致するもの
        partitioned[key] = cfg.Projects[key]
    }
}
```

例: プロジェクトが 9件・タスク数 3 の場合
```
TaskIndex=0 → プロジェクト 0, 3, 6
TaskIndex=1 → プロジェクト 1, 4, 7
TaskIndex=2 → プロジェクト 2, 5, 8
```

### エントリポイントのパターン（Go）

Cloud Run Jobs のコンテナは `os.Exit(0)` で正常終了、`os.Exit(1)` で失敗終了する。

```go
func main() {
    // 設定を読み込む（Secret Manager マウントのファイルから）
    cfg, err := config.Load(models.ConfigFilePath)
    if err != nil {
        slog.Error("failed to load config", slog.Any("error", err))
        os.Exit(1)   // 非ゼロ終了 → Cloud Run Jobs がタスク失敗と判定
    }

    // Cloud Run Jobs が注入する環境変数を読む
    settings, err := config.LoadJobSettings()
    // ...

    // JOB_TYPE で処理を分岐
    switch settings.JobType {
    case models.WorkflowJoiner:
        if _, err := services.NewJoinerService(...).Run(ctx, cfg, req); err != nil {
            slog.Error("joiner failed", slog.Any("error", err))
            os.Exit(1)
        }
    case models.WorkflowGarbageCollection:
        // ...
    }

    os.Exit(0)   // 正常終了
}
```

### Dockerfile（マルチステージビルド）

```dockerfile
# ---- ビルダー ----
FROM golang:1.25-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build \
    -trimpath \
    -ldflags="-s -w" \    # バイナリサイズを最小化
    -o /job \
    ./cmd/job

# ---- ランタイム ----
# distroless: シェルもパッケージマネージャもない最小イメージ
# :nonroot で UID 65532 として実行（セキュリティベストプラクティス）
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /job /job
ENTRYPOINT ["/job"]
```

### Cloud Run Jobs のデプロイ & 実行コマンド

```bash
# ジョブの作成（初回のみ）
gcloud run jobs create gemini-box-office-joiner \
  --image $IMAGE \
  --region $REGION \
  --set-env-vars=JOB_TYPE=joiner \
  --set-secrets=/run/secrets/entitlements.json=entitlements:latest \  # Secret Manager マウント
  --tasks 3 \              # 並列タスク数（CLOUD_RUN_TASK_COUNT=3 が注入される）
  --max-retries 2 \
  --task-timeout 3600      # タスクあたりのタイムアウト（秒）

# 手動実行
gcloud run jobs execute gemini-box-office-joiner --region $REGION

# 実行ログの確認
gcloud run jobs executions list --job gemini-box-office-joiner --region $REGION

# Cloud Scheduler で定期実行（毎日 2時に実行）
gcloud scheduler jobs create http gemini-joiner-daily \
  --schedule="0 2 * * *" \
  --uri="https://$REGION-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/$PROJECT_ID/jobs/gemini-box-office-joiner:run" \
  --oauth-service-account-email=$SA_EMAIL
```

---

## 運用上の注意点

### Cloud Run

- **ポート 8080** を使う。他のポートは動作しない。
- コールドスタートが発生する（初回リクエストに数秒かかることがある）。常時起動が必要なら `--min-instances 1` を設定。
- `--allow-unauthenticated` は開発・デモ用。本番は外す。
- GPU インスタンスはコールドスタートが長い（30秒以上）。

### Cloud Run Jobs

- タスクが非ゼロ終了すると `--max-retries` 回まで自動リトライされる。
- 並列タスク（`--tasks N`）は全タスクが成功して初めてジョブ成功とみなされる。
- `DRY_RUN=true` 環境変数で書き込みなしの動作確認ができる（group-licensing の実装例）。
- Secret Manager のシークレットはボリュームマウントで渡すと、コード変更なしに設定を更新できる。

---

## 主要リソース（このリポジトリ内）

| ファイル | 種別 | 内容 |
|---|---|---|
| `gemini/sample-apps/gemini-streamlit-cloudrun/` | Cloud Run | Streamlit + Gemini デプロイの基本形 |
| `gemini/sample-apps/gemini-quart-cloudrun/app/deploy.sh` | Cloud Run | `gcloud run deploy` の実装例 |
| `open-models/serving/cloud_run_ollama_gemma3_inference.ipynb` | Cloud Run | GPU 付き推論エンドポイントのフルフロー |
| `agents/cloud_run/agents_with_memory/` | Cloud Run | ADK + Memory Bank エージェントのデプロイ |
| `search/gemini-enterprise/group-licensing/` | Cloud Run Jobs | 並列シャーディング・Secret Manager マウント・DryRun の本格実装（Go） |
| `search/gemini-enterprise/group-licensing/docs/TDD.md` | Cloud Run Jobs | 設計思想・環境変数仕様・設定スキーマの詳細 |
