# RAGシステムの更新管理戦略

## 1. 権限変更発生時の更新フロー

### グローバルマテリアライズドビューの更新
```sql
-- 権限変更時の対応（PostgreSQL例）
CREATE OR REPLACE FUNCTION refresh_permissions_on_change() RETURNS TRIGGER AS $$
BEGIN
  -- 部分的な更新（変更されたユーザーのみ）
  DELETE FROM user_permissions_materialized WHERE user_id = NEW.user_id;
  
  INSERT INTO user_permissions_materialized
  SELECT * FROM generate_user_permissions(NEW.user_id);
  
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 権限テーブル変更時のトリガー
CREATE TRIGGER permissions_changed 
AFTER INSERT OR UPDATE OR DELETE ON user_permissions
FOR EACH ROW EXECUTE FUNCTION refresh_permissions_on_change();
```

### キャッシュの更新
```python
# 権限変更イベントハンドラー
def handle_permission_change(event):
    user_id = event.user_id
    affected_paths = event.affected_paths
    
    # 1. ユーザー権限キャッシュを無効化
    cache.delete(f"user_permissions:{user_id}")
    
    # 2. 関連する検索結果キャッシュも無効化
    pattern = f"search_results:{user_id}:*"
    cache.delete_pattern(pattern)
    
    # 3. 関連するファイルへのアクセス情報も無効化
    for path in affected_paths:
        cache.delete(f"access:{user_id}:{path}")
```

## 2. ファイル変更発生時の更新フロー

### ベクトルDB更新プロセス

```python
# ファイル更新イベントハンドラー
def handle_file_change(event):
    file_path = event.file_path
    event_type = event.type  # 'create', 'update', 'delete'
    
    # 1. ベクトルDBから既存エントリを削除
    vector_db.delete(filter={"file_path": file_path})
    
    # 2. 'delete'以外のイベントでは再インデックス処理
    if event_type != 'delete':
        # ファイル内容を取得
        content = file_system.read(file_path)
        
        # チャンク化
        chunks = chunker.process(content)
        
        # エンベディング生成と保存
        embeddings = []
        for i, chunk in enumerate(chunks):
            embedding = embedding_model.embed(chunk)
            metadata = {
                "file_path": file_path,
                "chunk_id": f"{i}",
                "last_updated": datetime.now().isoformat()
            }
            embeddings.append((chunk, embedding, metadata))
        
        # ベクトルDBにアップサート
        vector_db.upsert(embeddings)
    
    # 3. 関連キャッシュの無効化
    affected_users = permission_service.get_users_with_access(file_path)
    for user_id in affected_users:
        cache.delete_pattern(f"search_results:{user_id}:*")
```

## 3. 実践的な統合戦略

ファイル変更と権限変更の両方に効率的に対応するには、イベント駆動型のアーキテクチャが最適です：

```
1. 変更イベント発生
   ↓
2. イベントキューに送信
   ↓
3. 処理サービスが非同期で対応
   ↓
4. インデックス/キャッシュ更新完了通知
```

### 最適化ポイント

1. **バッチ処理**：短時間に複数の変更が発生した場合はバッチで処理

```python
# 複数ファイル変更の効率的処理
def batch_process_file_changes(file_events):
    # 削除処理をまとめて実行
    paths_to_delete = [e.file_path for e in file_events if e.requires_deletion]
    if paths_to_delete:
        vector_db.batch_delete(filter={"file_path": {"$in": paths_to_delete}})
    
    # 更新処理をまとめて実行
    docs_to_update = []
    for event in [e for e in file_events if e.type != 'delete']:
        chunks = process_file(event.file_path)
        docs_to_update.extend(chunks)
    
    if docs_to_update:
        vector_db.batch_upsert(docs_to_update)
```

2. **部分更新の活用**：ファイルの一部分だけが変更された場合

```python
# 差分更新の例（ファイル全体ではなく変更部分だけ更新）
def diff_update(file_path, old_content, new_content):
    # 差分を検出
    changed_sections = diff_detector.find_changes(old_content, new_content)
    
    # 変更されたセクションだけ再インデックス
    for section in changed_sections:
        # 元のチャンク識別子を取得
        chunk_ids = chunk_mapping.get_chunk_ids(file_path, section.start, section.end)
        
        # 該当チャンクだけ削除
        vector_db.delete(filter={"chunk_id": {"$in": chunk_ids}})
        
        # 新しいチャンクを作成
        new_chunks = chunker.process(section.new_content)
        
        # 新しいチャンクのエンベディングを生成
        # ...
```

3. **整合性維持メカニズム**：障害発生時のリカバリ

```python
# 整合性確認ジョブ（定期実行）
def verify_index_consistency():
    # ファイルシステムと検索インデックスを比較
    indexed_files = set(vector_db.get_all_file_paths())
    actual_files = set(file_system.list_all_files())
    
    # 不足しているファイルをインデックス
    missing = actual_files - indexed_files
    for file_path in missing:
        queue_indexing_job(file_path)
    
    # 余分なファイルをインデックスから削除
    extra = indexed_files - actual_files
    if extra:
        vector_db.delete(filter={"file_path": {"$in": list(extra)}})
```

この統合アプローチにより、RAGシステムのデータと権限の整合性を効率的に維持しながら、低レイテンシのユーザー体験を提供できます。システム規模に応じて、これらの戦略を適宜調整することが重要です。