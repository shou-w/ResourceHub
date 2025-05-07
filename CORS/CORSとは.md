# CORS（クロス・オリジン・リソース・シェアリング）総まとめ

## CORSとは
CORSは「Cross-Origin Resource Sharing」の略で、異なるオリジン（ドメイン、プロトコル、ポートの組み合わせ）間でのリソース共有を安全に行うための仕組みです。ウェブブラウザに組み込まれたセキュリティ機能で、悪意あるサイトからの不正アクセスを防止します。

## なぜCORSが必要なのか
ブラウザの「同一オリジンポリシー」という安全対策により、異なるオリジンからのリソース取得は基本的に制限されています。しかし現代のウェブではサイト間の連携が必要なケースが多く、CORSによって安全に許可できるようになりました。

## CORSの仕組み

### 1. 基本的な流れ
1. **リクエスト発生**: サイトA（`https://a.com`）がサイトB（`https://b.com`）のリソースにアクセス
2. **Origin情報付加**: ブラウザはリクエストに`Origin: https://a.com`ヘッダーを自動的に追加
3. **サーバー応答**: サイトBのサーバーは許可するなら特別なヘッダーを付加して応答
4. **ブラウザ判断**: 許可されていればデータを受け取り、されていなければブロック

### 2. プリフライトリクエスト
複雑なリクエスト（特定のHTTPメソッドや特殊なヘッダーを使う場合）は、本番リクエスト前に確認が行われます：

1. **事前確認**: ブラウザが`OPTIONS`メソッドでサーバーに許可を確認
2. **応答ヘッダー**:
   - `Access-Control-Allow-Origin`: 許可するオリジン
   - `Access-Control-Allow-Methods`: 許可するHTTPメソッド
   - `Access-Control-Allow-Headers`: 許可するヘッダー
   - `Access-Control-Max-Age`: 許可の有効期間
3. **本番リクエスト**: 許可された場合のみ実行される

## CORS設定の場所

### 1. ウェブサーバーでの設定
**Nginxの例**:
```
add_header Access-Control-Allow-Origin https://trusted-site.com;
```

**適している場合**:
- 静的コンテンツの提供
- 複数アプリケーションへの共通ポリシー適用
- セキュリティの一元管理

### 2. アプリケーションでの設定
**Python (FastAPI)の例**:
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://trusted-site.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["*"],
)
```

**適している場合**:
- エンドポイントごとの細かい制御
- 動的なCORSポリシー
- 開発の柔軟性確保

## よくあるCORS設定パターン
1. **特定のオリジンのみ許可**:
   ```
   Access-Control-Allow-Origin: https://trusted-site.com
   ```

2. **すべてのオリジンを許可** (注意: 本番環境では非推奨):
   ```
   Access-Control-Allow-Origin: *
   ```

3. **認証情報の送信許可** (Cookieなどがある場合):
   ```
   Access-Control-Allow-Credentials: true
   ```
   注: この場合は特定のオリジンのみ指定可能（`*`は使えない）

## CORSのセキュリティポイント
- ブラウザが強制的に実施するため、クライアント側で回避できない
- サーバー側が明示的に許可しない限りアクセス不可
- プリフライトリクエストにより、危険な操作の事前防止が可能

## 他のセキュリティ技術との違い
- **OAuth**: ユーザー認証と認可のフレームワーク（アクセス権限管理）
- **mTLS**: 通信の両端が互いに証明書で本人確認（相互認証）
- **CORS**: ブラウザレベルでのリソースアクセス制御（同一オリジンポリシーの緩和）

CORSはウェブセキュリティの重要な一部であり、現代のウェブアプリケーションには欠かせない機能です。適切に設定することで、安全にクロスオリジン通信を実現できます。