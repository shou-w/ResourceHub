# Summary Service: SSE + RabbitMQ 非同期要約アーキテクチャ

## 概要

**Summary Service**は、大きなファイルの要約処理において発生するタイムアウト問題を解決するため、**RabbitMQ**による非同期処理と**SSE（Server-Sent Events）**によるリアルタイム進捗通知を組み合わせたアーキテクチャです。

---

## 全体アーキテクチャ

### サービス構成
```mermaid
graph TB
    Client[クライアント] --> Gateway[API Gateway]
    
    subgraph "Summary Service"
        Gateway --> TaskAPI[タスク登録API]
        Gateway --> StreamAPI[進捗確認API]
        
        TaskAPI --> Queue[RabbitMQ]
        StreamAPI --> Redis[Redis]
        Queue --> Worker[Summary Worker]
        Worker --> Redis
    end
    
    Worker --> LLMService[LLM Service]
    
    classDef service fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef external fill:#f3e5f5
    
    class TaskAPI,StreamAPI,Queue,Redis,Worker service
    class LLMService external
```

---

## 基本的な仕組み

### 1. 2つのAPIによる責任分離

| API | エンドポイント | 役割 |
|-----|---------------|------|
| **タスク登録API** | `POST /api/v1/summarize` | ファイル受信、タスクキューイング |
| **進捗確認API** | `GET /api/v1/summarize/{job_id}/stream` | SSE接続、リアルタイム進捗通知 |

### 2. 非同期処理の流れ
```mermaid
graph LR
    Upload[ファイルアップロード] --> Queue[キューに登録]
    Queue --> Background[バックグラウンド処理]
    Background --> Progress[進捗通知]
    Progress --> Complete[完了通知]
```

---

## 詳細処理フロー

### 全体シーケンス
```mermaid
sequenceDiagram
    participant C as クライアント
    participant TA as タスク登録API
    participant Q as RabbitMQ
    participant SA as 進捗確認API
    participant R as Redis
    participant W as Summary Worker
    participant LLM as LLM Service
    
    Note over C,LLM: Phase 1: タスク登録
    C->>TA: POST /summarize (ファイル)
    TA->>Q: タスクをキューに投入
    TA->>R: 初期状態保存 (queued)
    TA->>C: job_id 返却
    
    Note over C,LLM: Phase 2: 進捗監視
    C->>SA: GET /stream/{job_id}
    SA-->>C: SSE接続開始
    
    Note over C,LLM: Phase 3: バックグラウンド処理
    W->>Q: タスク取得
    W->>R: 状態更新 (processing)
    R->>SA: 状態変更通知
    SA-->>C: SSE進捗送信
    
    loop Map-Reduce処理
        W->>LLM: チャンク要約依頼
        LLM->>W: 要約結果
        W->>R: 進捗更新 (20%, 40%, 60%...)
        R->>SA: 進捗通知
        SA-->>C: SSE進捗送信
    end
    
    W->>LLM: 最終統合要約
    LLM->>W: 最終要約結果
    W->>R: 完了状態保存 (completed)
    R->>SA: 完了通知
    SA-->>C: SSE完了送信
```

---

## コンポーネント詳細

### Summary Service 内部構成
```mermaid
graph TD
    subgraph "Summary Service"
        subgraph "API Layer"
            TaskAPI[タスク登録API<br/>POST /summarize]
            StreamAPI[進捗確認API<br/>GET /stream]
        end
        
        subgraph "Processing Layer"
            Worker[Summary Worker<br/>Map-Reduce処理]
        end
        
        subgraph "Infrastructure Layer"
            Queue[RabbitMQ<br/>タスクキュー]
            Redis[Redis<br/>状態管理]
        end
        
        TaskAPI --> Queue
        StreamAPI --> Redis
        Queue --> Worker
        Worker --> Redis
    end
```

### 各コンポーネントの役割

| コンポーネント | 役割 |
|----------------|------|
| **タスク登録API** | ファイル受信、バリデーション、キューへの投入 |
| **進捗確認API** | SSE接続管理、Redisからの状態取得、進捗配信 |
| **RabbitMQ** | タスクの非同期キューイング、ワーカーへの配信 |
| **Summary Worker** | 実際の要約処理、LLM連携、進捗更新 |
| **Redis** | 処理状態の保存、進捗情報の管理 |

---

## 状態管理

### ジョブの状態遷移
```mermaid
stateDiagram-v2
    [*] --> Queued: タスク登録
    Queued --> Processing: Worker開始
    Processing --> Processing: チャンク処理中
    Processing --> Completed: 要約完了
    Processing --> Error: 処理失敗
    
    Completed --> [*]: 結果取得
    Error --> [*]: エラー処理
    
    note right of Processing
        Map-Reduce処理
        0% → 100%
    end note
```

### Redisでの状態データ
```mermaid
graph LR
    subgraph "Redis Key: summary:progress:{job_id}"
        Status[status: queued,processing,completed,error]
        Progress[progress: 0-100]
        Message[message: 処理状況メッセージ]
        Results[partial_results: チャンク要約結果]
        Summary[final_summary: 最終要約結果]
    end
```

---

## SSE進捗通知

### イベントの種類
```mermaid
graph TD
    Start[SSE接続開始] --> Connected[connected<br/>接続確認]
    Connected --> Queued[queued<br/>キュー待機]
    Queued --> Processing[processing<br/>処理開始]
    
    Processing --> Progress[progress<br/>進捗更新イベント]
    Progress --> Progress
    
    Progress --> Completed[completed<br/>処理完了]
    Processing --> Error[error<br/>エラー発生]
    
    Completed --> End[接続終了]
    Error --> End
    
    classDef event fill:#fff3e0
    classDef terminal fill:#ffcdd2
    
    class Connected,Queued,Processing,Progress event
    class Completed,Error,End terminal
```

### 進捗通知データ例
```json
{
  "status": "processing",
  "progress": 60,
  "message": "チャンク 3/5 要約中...",
  "partial_results": [
    {"chunk": 1, "summary": "第1章の要約..."},
    {"chunk": 2, "summary": "第2章の要約..."},
    {"chunk": 3, "summary": "第3章の要約..."}
  ]
}
```

---

## Summary Worker の処理詳細

### Map-Reduce処理フロー
```mermaid
graph TD
    Start[タスク取得] --> Parse[ファイル解析]
    Parse --> Split[チャンク分割]
    
    Split --> Map1[チャンク1要約]
    Split --> Map2[チャンク2要約]
    Split --> Map3[チャンクN要約]
    
    Map1 --> Update1[進捗更新 20%]
    Map2 --> Update2[進捗更新 40%]
    Map3 --> Update3[進捗更新 60%]
    
    Update1 --> Reduce[統合要約]
    Update2 --> Reduce
    Update3 --> Reduce
    
    Reduce --> Final[最終要約]
    Final --> Complete[完了通知]
    
    classDef map fill:#e1f5fe
    classDef update fill:#fff3e0
    classDef reduce fill:#e8f5e8
    
    class Map1,Map2,Map3 map
    class Update1,Update2,Update3 update
    class Reduce,Final reduce
```

---

## クライアント側処理パターン

### 基本的な利用フロー
```mermaid
graph TD
    FileSelect[ファイル選択] --> TaskCreate[タスク作成API呼び出し]
    TaskCreate --> JobID[job_id取得]
    JobID --> SSEConnect[SSE接続開始]
    
    SSEConnect --> Monitor[進捗監視]
    Monitor --> UpdateUI[UI更新]
    UpdateUI --> Monitor
    
    Monitor --> Complete[完了通知受信]
    Complete --> DisplayResult[要約結果表示]
    
    Monitor --> ErrorHandle[エラー処理]
    ErrorHandle --> ShowError[エラー表示]
```

---

## アーキテクチャのメリット

### 解決される課題
| 従来の問題 | 解決方法 |
|------------|----------|
| **タイムアウト** | 非同期処理により即座にレスポンス |
| **ユーザー体験** | SSEによるリアルタイム進捗表示 |
| **サーバー負荷** | バックグラウンド処理で負荷分散 |
| **エラーハンドリング** | 各段階での適切な状態管理 |

### 主要な利点
- **即座のフィードバック**: タスク登録完了を即座に通知
- **透明性**: 処理進捗をリアルタイムで確認可能
- **信頼性**: RabbitMQによる確実なタスク配信
- **拡張性**: ワーカー数の調整で処理能力拡張可能

---

## パターン名称

この設計は以下のパターンの組み合わせです：

- **Long Running Task Pattern**: 長時間処理の非同期化
- **Asynchronous Request-Reply Pattern**: リクエスト/レスポンスの分離
- **Event-Driven Architecture**: 状態変更によるイベント通知

**一般的な呼び方**: 「非同期タスク処理 + リアルタイム進捗通知アーキテクチャ」