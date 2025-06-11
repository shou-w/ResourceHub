# AIプロジェクト実装方針決定書（簡素化構成版）

## 修正された技術要件

### 実際の構成要素
ユーザーの要件に基づく実際の構成：

```
AIプロジェクト全体構成:
├── FastAPI アプリケーション（モジュラモノリス）
│   ├── Chatbot Service
│   ├── Summary Service  
│   ├── LLM Service
│   └── DB Management Service
├── PostgreSQL + pgvector（AI専用データ）
├── Langfuse v2（単一コンテナ + PostgreSQL）
├── RabbitMQ（メッセージブローカー）
└── Redis（キャッシュ・セッション管理）
```

### 重要な発見
Langfuse v2では、PostgreSQL のみが必要で、ClickHouse、Redis、S3/MinIOは不要です。これにより構成が大幅に簡素化されます。

---

## コンテナ構成の選択肢分析

### Option 1: 統合コンテナ（AIプロジェクト統合）
```yaml
# 3コンテナ構成
services:
  ai-system:          # FastAPI + PostgreSQL + Redis統合
  langfuse:           # Langfuse v2（独立）
  rabbitmq:           # RabbitMQ（独立）
```

### Option 2: 機能別分離コンテナ
```yaml
# 5コンテナ構成
services:
  ai-app:             # FastAPI アプリケーション
  ai-db:              # PostgreSQL + pgvector
  redis:              # Redis キャッシュ
  langfuse:           # Langfuse v2
  rabbitmq:           # RabbitMQ
```

### Option 3: 完全分離コンテナ
```yaml
# 6-7コンテナ構成
services:
  ai-app:             # FastAPI アプリケーション
  ai-db:              # PostgreSQL + pgvector
  redis:              # Redis キャッシュ
  langfuse-web:       # Langfuse アプリ
  langfuse-db:        # Langfuse専用 PostgreSQL
  rabbitmq:           # RabbitMQ
  celery-worker:      # Celery ワーカー（Optional）
```

---

## 詳細技術分析

### Langfuse v2の統合可能性

#### **LangfuseをAIシステムに統合する場合**
```dockerfile
# 統合コンテナでのLangfuse統合例
FROM python:3.11

# FastAPI アプリケーション
COPY ./ai-app /app/ai-app

# Langfuse v2のソースコード追加
RUN git clone -b v2 https://github.com/langfuse/langfuse.git /app/langfuse

# 両方の依存関係をインストール
RUN pip install -r /app/ai-app/requirements.txt
RUN pip install -r /app/langfuse/requirements.txt

# PostgreSQL + Redis をインストール
RUN apt-get update && apt-get install -y postgresql redis-server

# 起動スクリプト
COPY start.sh /start.sh
```

**統合のメリット：**
- データベース共有が可能（両方ともPostgreSQL使用）
- 同一プロセス内でのLangfuse統合
- ネットワーク遅延ゼロでのトレーシング

**統合の課題：**
- Langfuse v2はNext.jsベース（Node.js）、FastAPIはPython
- 異なるランタイム環境が必要
- ビルド・デプロイの複雑化

### RabbitMQの統合可能性

#### **RabbitMQを統合コンテナに含める場合**
```bash
# 統合コンテナでのRabbitMQ起動例
#!/bin/bash
# PostgreSQL 起動
su postgres -c "pg_ctl start -D /var/lib/postgresql/data"

# Redis 起動
redis-server --daemonize yes

# RabbitMQ 起動
rabbitmq-server -detached

# FastAPI アプリケーション起動
cd /app && uvicorn main:app --host 0.0.0.0 --port 8000
```

**統合の課題：**
- プロセス管理の複雑化（PostgreSQL + Redis + RabbitMQ + FastAPI）
- 異なるユーザー権限が必要
- 障害時の影響範囲拡大

---

## 運用負荷とメンテナンス性の再評価

### 各構成の運用負荷比較

| 項目 | 統合コンテナ<br/>（3個） | 機能別分離<br/>（5個） | 完全分離<br/>（6-7個） |
|------|-------------------------|----------------------|----------------------|
| **学習コスト** | 低 | 中 | 高 |
| **デプロイ複雑性** | 低 | 中 | 高 |
| **障害切り分け** | 困難 | 中程度 | 容易 |
| **個別更新** | 困難 | 中程度 | 容易 |
| **リソース効率** | 高 | 中 | 低 |
| **スケーラビリティ** | 低 | 中 | 高 |

### 実際の運用シナリオ分析

#### **Langfuse v2アップデート時**
```
統合コンテナの場合:
1. 全システム停止（FastAPI + Langfuse + DB + Redis + RabbitMQ）
2. イメージ再ビルド（Node.js + Python環境）
3. 全体デプロイ・テスト
4. システム再起動
→ ダウンタイム: 30-60分

分離コンテナの場合:
1. Langfuseコンテナのみ停止
2. 新しいLangfuseイメージに更新
3. Langfuseコンテナ再起動
→ ダウンタイム: 2-5分（FastAPIは継続稼働）
```

#### **RabbitMQ設定変更時**
```
統合コンテナの場合:
1. 全システム停止
2. 設定ファイル変更
3. 全体再起動
→ ダウンタイム: 5-15分

分離コンテナの場合:
1. RabbitMQコンテナのみ再起動
2. 設定適用確認
→ ダウンタイム: 1-2分（他は継続稼働）
```

---

## パフォーマンス分析

### FastAPI + Langfuse統合でのパフォーマンス

#### **トレーシングオーバーヘッド**
```python
# 統合環境でのLangfuse呼び出し
async def chatbot_with_tracing(query: str):
    # Langfuse初期化（同一プロセス内）
    langfuse = Langfuse()  # 0.1ms
    
    # トレース開始
    trace = langfuse.trace(name="chatbot_query")  # 0.5ms
    
    # ベクトル検索
    with trace.span(name="vector_search") as span:
        vectors = await vector_search(query)  # 80ms
    
    # LLM呼び出し
    with trace.span(name="llm_call") as span:
        response = await llm_api_call(vectors, query)  # 3000ms
    
    # トレース終了
    trace.end()  # 1ms
    
    return response

# 総オーバーヘッド: 1.6ms（0.05%程度）
# 分離環境: 5-10ms（0.2%程度）
```

### RabbitMQ統合でのメモリ使用量

```bash
# プロセス別メモリ使用量試算
統合コンテナ内訳:
├── FastAPI: 256MB
├── PostgreSQL: 512MB  
├── Redis: 64MB
├── RabbitMQ: 128MB
├── Node.js(Langfuse): 256MB
└── 合計: 1,216MB

分離コンテナ内訳:
├── ai-app: 256MB
├── ai-db: 512MB
├── redis: 64MB
├── rabbitmq: 128MB
├── langfuse: 256MB
└── 合計: 1,216MB（同等）

# メモリ効率: ほぼ同等
# ただし統合時はプロセス間でのメモリ競合リスクあり
```

---

## 段階的実装戦略（修正版）

### Phase 1: 最小構成（2-3週間）
```yaml
# 2コンテナ構成：学習コスト最小化
services:
  ai-system:
    # FastAPI + PostgreSQL + Redis 統合
    # Langfuseは手動テストで代替
    # RabbitMQはFastAPI BackgroundTasksで代替
    
  rabbitmq:
    # 将来のCelery統合準備
    # 当初はモニタリングのみ
```

**実装方針：**
- AIコア機能の確実な実装
- 統合コンテナでの運用経験蓄積
- 最小限の外部依存

### Phase 2A: 観測性追加（4-6週間後）
```yaml
# 3コンテナ構成：Langfuse追加
services:
  ai-system:     # 既存のAI統合システム
  langfuse:      # 独立Langfuseコンテナ
  rabbitmq:      # 既存のRabbitMQ
```

### Phase 2B: 非同期処理強化（2-4週間後）
```yaml
# 4コンテナ構成：Celery追加
services:
  ai-system:     # 既存のAI統合システム
  langfuse:      # 既存のLangfuse
  rabbitmq:      # 既存のRabbitMQ
  celery-worker: # 大ファイル処理用ワーカー
```

### Phase 3: 本格分離（必要に応じて）
```yaml
# 5-6コンテナ構成：完全分離
services:
  ai-app:        # FastAPIのみ
  ai-db:         # PostgreSQL分離
  redis:         # Redis分離
  langfuse:      # Langfuse
  rabbitmq:      # RabbitMQ
  celery-worker: # Celery Worker
```

---

## 最終推奨事項

### **機能別分離コンテナ（5個構成）**を推奨

```yaml
# 推奨構成
version: '3.8'
services:
  ai-app:
    build: ./ai-project
    environment:
      - DATABASE_URL=postgresql://ai-db:5432/aidb
      - REDIS_URL=redis://redis:6379
      - LANGFUSE_HOST=http://langfuse:3000

  ai-db:
    image: postgres:15
    environment:
      - POSTGRES_DB=aidb
    volumes:
      - ai_pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7

  langfuse:
    image: langfuse/langfuse:2
    environment:
      - DATABASE_URL=postgresql://ai-db:5432/langfuse_db

  rabbitmq:
    image: rabbitmq:3-management
```

### 選択理由

#### **統合コンテナを避ける理由**
1. **技術的複雑性**: Node.js（Langfuse）+ Python（FastAPI）の混在
2. **運用リスク**: 複数プロセス管理の複雑化
3. **更新リスク**: 部分更新ができず、全体停止が必要

#### **機能別分離の優位性**
1. **適度な複雑性**: 5コンテナは管理可能な範囲
2. **独立更新**: 各コンポーネントの個別更新可能
3. **障害分離**: 問題の影響範囲を限定
4. **学習価値**: コンテナ運用の段階的習得

#### **完全分離を避ける理由**
1. **過剰な複雑性**: 初期段階には不要
2. **学習コスト**: チームの現状に不適合
3. **リソース効率**: 小規模運用では非効率

---

## リスク軽減策

### 運用負荷軽減
```yaml
# ヘルスチェック強化
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

### 監視体制
```python
# 統合監視ダッシュボード
async def system_health():
    return {
        "ai_app": await check_fastapi_health(),
        "database": await check_postgres_health(),
        "redis": await check_redis_health(),
        "langfuse": await check_langfuse_health(),
        "rabbitmq": await check_rabbitmq_health()
    }
```

### 段階的移行計画
```
移行判断基準:
├── パフォーマンス: 応答時間 > 5秒
├── 可用性: ダウンタイム > 月1時間
├── スケーラビリティ: 同時ユーザー > 100名
└── 運用負荷: 週次メンテナンス > 4時間
```

---

## 結論

**簡素化された技術要件（Langfuse v2 + RabbitMQ + Redis）を踏まえ、「機能別分離コンテナ（5個構成）」が最適解です。**

この選択により：
- ✅ **適度な複雑性**: 学習可能な範囲での技術習得
- ✅ **運用安定性**: 独立更新・障害分離の実現
- ✅ **将来性**: 必要に応じた更なる分離が可能
- ✅ **実用性**: 確実なリリースと段階的成長

統合コンテナの魅力は理解できますが、Langfuse（Node.js）との技術混在と運用リスクを考慮すると、**適度な分離による確実性**を優先すべきです。