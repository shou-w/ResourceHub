# ファイルレベルでのアクセス制御の最適なアプローチ

ファイルレベルでアクセス制御する場合、単一ベクトルDBを使用した「拡張ハイブリッドアプローチ」が最適です。これは先ほどのハイブリッドアプローチをさらに洗練させたものです。

## 実装方針

### 1. 単一ベクトルDBと詳細メタデータの組み合わせ

```json
// チャンクごとのメタデータ例
{
  "file_id": "sample4.pdf",
  "file_path": "project1/folder2/sample4.pdf",
  "project_id": "project1",
  "folder_path": "project1/folder2",
  "chunk_id": "sample4_chunk3",
  "last_updated": "2025-04-01"
}
```

各ドキュメントチャンクには詳細なパス情報を含めることで、あらゆるレベルでのフィルタリングが可能になります。

### 2. 階層的な権限モデルの実装

```json
// 権限DB内のユーザー権限例
{
  "user_id": "user_A",
  "permissions": [
    {"resource_type": "project", "resource_id": "project1", "access": "allow"},
    {"resource_type": "folder", "resource_id": "project1/folder2", "access": "deny"},
    {"resource_type": "file", "resource_id": "project1/folder1/sample1.pdf", "access": "allow"},
    {"resource_type": "file", "resource_id": "project1/folder2/sample4.pdf", "access": "allow"}
  ]
}
```

この階層的権限モデルでは、より詳細なレベルの設定が上位レベルの設定を上書きします（例：フォルダが拒否でも、特定ファイルは許可可能）。

### 3. 効率的なアクセス解決アルゴリズム

```python
# 検索時のフィルタリング（概念コード）
def build_access_filter(user_id):
    # ユーザーの権限を取得
    permissions = permission_db.get_permissions(user_id)
    
    # 明示的に許可/拒否されたリソースを分類
    allowed = {
        "files": [p.resource_id for p in permissions if p.resource_type == "file" and p.access == "allow"],
        "folders": [p.resource_id for p in permissions if p.resource_type == "folder" and p.access == "allow"],
        "projects": [p.resource_id for p in permissions if p.resource_type == "project" and p.access == "allow"]
    }
    
    denied = {
        "files": [p.resource_id for p in permissions if p.resource_type == "file" and p.access == "deny"],
        "folders": [p.resource_id for p in permissions if p.resource_type == "folder" and p.access == "deny"],
        "projects": [p.resource_id for p in permissions if p.resource_type == "project" and p.access == "deny"]
    }
    
    # 複合フィルタを構築
    filter_query = {
        "$or": [
            # 直接許可されたファイル
            {"file_id": {"$in": allowed["files"]}},
            
            # 許可されたフォルダ内のファイル（ただし明示的に拒否されたものを除く）
            {"$and": [
                {"folder_path": {"$in": allowed["folders"]}},
                {"file_id": {"$nin": denied["files"]}}
            ]},
            
            # 許可されたプロジェクト内のファイル（ただし拒否されたフォルダ/ファイルを除く）
            {"$and": [
                {"project_id": {"$in": allowed["projects"]}},
                {"folder_path": {"$nin": denied["folders"]}},
                {"file_id": {"$nin": denied["files"]}}
            ]}
        ]
    }
    
    return filter_query
```

### 4. パフォーマンス最適化戦略

1. **権限キャッシング**: ユーザーごとのアクセス権限マップをキャッシュ
2. **事前計算されたアクセスリスト**: 定期的にユーザーごとにアクセス可能ファイルIDのリストを計算
3. **権限変更の増分更新**: 権限変更時にキャッシュを完全に再構築せず増分更新
4. **クエリプラン最適化**: ベクトル検索前にメタデータフィルタを適用できるDBを選択

## 実装上の注意点

1. **ベクトルDBの選定**: Weaviate、Qdrantなど、複雑なメタデータフィルタをサポートし効率的に処理できるものを選ぶ

2. **権限解決ロジックの分離**: 検索ロジックと権限解決ロジックを分離し、独立してテスト・最適化可能にする

3. **非同期処理の活用**: 権限チェックとベクトル検索を並行処理し、レイテンシを削減

4. **段階的なフィルタリング**:
   - 第1段階: 明らかにアクセス不可のものをクエリレベルで除外
   - 第2段階: ベクトル検索結果に対し、詳細な権限チェックを適用

この拡張ハイブリッドアプローチは実装の複雑さとパフォーマンスのバランスがとれており、ファイルレベルの細かいアクセス制御にも対応できる最も現実的な解決策だと考えます。