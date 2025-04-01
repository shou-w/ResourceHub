# pgvectorにおける「複雑な権限テーブルとの結合クエリ」の詳細説明

pgvectorの最大の強みは、PostgreSQLのリレーショナルデータベース機能とベクトル検索を組み合わせられる点にあります。これが「複雑な権限テーブルとの結合クエリが可能」という意味です。具体的に説明します：

## 1. リレーショナルモデルを活用した権限管理

PostgreSQLでは、以下のような権限テーブル構造を実装できます：

```sql
-- ユーザーテーブル
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(100) NOT NULL
);

-- ファイルメタデータテーブル
CREATE TABLE documents (
    doc_id SERIAL PRIMARY KEY,
    file_path VARCHAR(255) NOT NULL,
    project_id VARCHAR(100),
    folder_path VARCHAR(255),
    content_embedding vector(1536),  -- pgvectorの拡張型
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 権限テーブル（ファイルレベル）
CREATE TABLE file_permissions (
    permission_id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(user_id),
    doc_id INTEGER REFERENCES documents(doc_id),
    access_type VARCHAR(20) CHECK (access_type IN ('allow', 'deny')),
    UNIQUE(user_id, doc_id)
);

-- 権限テーブル（フォルダレベル）
CREATE TABLE folder_permissions (
    permission_id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(user_id),
    folder_path VARCHAR(255),
    access_type VARCHAR(20) CHECK (access_type IN ('allow', 'deny')),
    UNIQUE(user_id, folder_path)
);

-- 権限テーブル（プロジェクトレベル）
CREATE TABLE project_permissions (
    permission_id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(user_id),
    project_id VARCHAR(100),
    access_type VARCHAR(20) CHECK (access_type IN ('allow', 'deny')),
    UNIQUE(user_id, project_id)
);
```

## 2. 結合クエリによる権限フィルタリングとベクトル検索の統合

特定のユーザー（user_id=123）がアクセスできるドキュメントの中から、クエリに関連するドキュメントを検索する例：

```sql
-- ユーザークエリのベクトル
WITH query_embedding AS (
    SELECT '[0.1, 0.2, ...]'::vector AS embedding
),

-- アクセス可能なドキュメントIDを取得
accessible_docs AS (
    SELECT d.doc_id FROM documents d
    -- 直接ファイルレベルで許可されているもの
    WHERE EXISTS (
        SELECT 1 FROM file_permissions fp 
        WHERE fp.user_id = 123 
        AND fp.doc_id = d.doc_id 
        AND fp.access_type = 'allow'
    )
    -- または、フォルダレベルで許可されていて、ファイルレベルで拒否されていないもの
    OR (
        EXISTS (
            SELECT 1 FROM folder_permissions fp 
            WHERE fp.user_id = 123 
            AND d.folder_path LIKE fp.folder_path || '%' 
            AND fp.access_type = 'allow'
        )
        AND NOT EXISTS (
            SELECT 1 FROM file_permissions fp 
            WHERE fp.user_id = 123 
            AND fp.doc_id = d.doc_id 
            AND fp.access_type = 'deny'
        )
    )
    -- または、プロジェクトレベルで許可されていて、フォルダ/ファイルレベルで拒否されていないもの
    OR (
        EXISTS (
            SELECT 1 FROM project_permissions pp 
            WHERE pp.user_id = 123 
            AND d.project_id = pp.project_id 
            AND pp.access_type = 'allow'
        )
        AND NOT EXISTS (
            SELECT 1 FROM folder_permissions fp 
            WHERE fp.user_id = 123 
            AND d.folder_path LIKE fp.folder_path || '%' 
            AND fp.access_type = 'deny'
        )
        AND NOT EXISTS (
            SELECT 1 FROM file_permissions fp 
            WHERE fp.user_id = 123 
            AND fp.doc_id = d.doc_id 
            AND fp.access_type = 'deny'
        )
    )
)

-- アクセス可能なドキュメントの中からベクトル検索を実行
SELECT d.doc_id, d.file_path, 
       d.content_embedding <=> (SELECT embedding FROM query_embedding) AS distance
FROM documents d
JOIN accessible_docs ad ON d.doc_id = ad.doc_id
ORDER BY distance
LIMIT 10;
```

## 3. この方法の主なメリット

1. **単一環境での完結**：権限管理とベクトル検索を別々のシステムで管理する必要がない

2. **トランザクションの保証**：権限変更とコンテンツインデックス更新を単一トランザクションで処理可能

3. **SQL言語の表現力**：複雑な階層的権限解決ロジックをSQLの宣言的な方法で表現可能

4. **既存のSQLスキルの活用**：多くの開発者が慣れているSQLを使用できる

5. **インデックス最適化**：PostgreSQLの豊富なインデックスオプションを活用して権限チェックのパフォーマンスを向上

## 4. 実際のユースケース例

例えば、以下のようなシナリオで威力を発揮します：

```
ユーザーAは：
- project1全体にアクセス可能
- ただしproject1/folder2には特別な理由でアクセス不可
- しかしproject1/folder2/sample4.pdfだけは例外的にアクセス可能
```

このような細かいファイルレベルの制御を、専用の権限テーブルとSQLの表現力で効率的に実現できます。従来のベクトルDBだけでは実装が複雑になる階層的なオーバーライドルールも、SQL JOINと条件式の組み合わせで自然に表現できます。

これが、pgvectorが「複雑な権限テーブルとの結合クエリが可能」と説明した理由です。