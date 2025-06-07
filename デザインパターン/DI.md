DI（依存性の注入）について、具体例を用いてわかりやすく説明します。

## DI（Dependency Injection）とは

**依存性の注入**は、オブジェクトが必要とする依存関係（他のオブジェクト）を、外部から「注入」する設計パターンです。オブジェクト自身が依存関係を作成するのではなく、外部から受け取ることで、柔軟性と保守性を向上させます。

## DIがない場合の問題

まず、DIを使わない例を見てみましょう：

```python
# DIなしの例（問題のあるコード）
class EmailService:
    def send_email(self, message):
        print(f"メールを送信: {message}")

class OrderService:
    def __init__(self):
        # 問題：OrderServiceがEmailServiceに強く依存している
        self.email_service = EmailService()
    
    def place_order(self, order_details):
        print(f"注文を処理: {order_details}")
        self.email_service.send_email("注文確認メールを送信しました")

# 使用例
order_service = OrderService()
order_service.place_order("商品A x 2")
```

### この方法の問題点：
1. **強い結合**: `OrderService`が`EmailService`に直接依存
2. **テストしにくい**: `EmailService`をモックに置き換えるのが困難
3. **変更に弱い**: メール送信方法を変更したい場合、`OrderService`も修正が必要

## DIを使った改善版

```python
from abc import ABC, abstractmethod

# 1. インターフェース（抽象クラス）を定義
class NotificationService(ABC):
    @abstractmethod
    def send_notification(self, message):
        pass

# 2. 具体的な実装クラス
class EmailService(NotificationService):
    def send_notification(self, message):
        print(f"メールを送信: {message}")

class SMSService(NotificationService):
    def send_notification(self, message):
        print(f"SMS送信: {message}")

# 3. 依存性を注入されるクラス
class OrderService:
    def __init__(self, notification_service: NotificationService):
        # 依存関係を外部から注入
        self.notification_service = notification_service
    
    def place_order(self, order_details):
        print(f"注文を処理: {order_details}")
        self.notification_service.send_notification("注文確認通知を送信しました")

# 使用例
# メール通知を使用
email_service = EmailService()
order_service_email = OrderService(email_service)
order_service_email.place_order("商品A x 2")

# SMS通知を使用（簡単に切り替え可能）
sms_service = SMSService()
order_service_sms = OrderService(sms_service)
order_service_sms.place_order("商品B x 1")
```

## テスト用のモック例

```python
# テスト用のモック実装
class MockNotificationService(NotificationService):
    def __init__(self):
        self.sent_messages = []
    
    def send_notification(self, message):
        self.sent_messages.append(message)
        print(f"モック通知: {message}")

# テストコード例
def test_order_service():
    # モックを注入してテスト
    mock_service = MockNotificationService()
    order_service = OrderService(mock_service)
    
    order_service.place_order("テスト商品")
    
    # 通知が送信されたかを確認
    assert len(mock_service.sent_messages) == 1
    assert "注文確認通知" in mock_service.sent_messages[0]
    print("テスト成功！")

test_order_service()
```

## DIの3つの主要な手法

### 1. コンストラクタ注入（推奨）
```python
class OrderService:
    def __init__(self, notification_service: NotificationService):
        self.notification_service = notification_service
```

### 2. セッター注入
```python
class OrderService:
    def set_notification_service(self, notification_service: NotificationService):
        self.notification_service = notification_service
```

### 3. インターフェース注入
```python
class NotificationServiceInjector(ABC):
    @abstractmethod
    def inject_notification_service(self, notification_service: NotificationService):
        pass
```

## DIの利点

| 項目 | DIなし | DIあり |
|------|--------|--------|
| **結合度** | 強い結合 | 疎な結合 |
| **テスト性** | 困難 | 簡単（モック使用可能） |
| **柔軟性** | 低い | 高い（実装を簡単に変更可能） |
| **保守性** | 低い | 高い |
| **拡張性** | 困難 | 簡単 |

## 実際の使用場面

- **データベースアクセス**: 本番環境では実際のDB、テスト環境ではメモリDB
- **外部API呼び出し**: 本番環境では実際のAPI、テスト環境ではモック
- **ログ出力**: コンソール出力、ファイル出力、クラウドログを切り替え
- **設定管理**: 環境に応じて異なる設定サービスを注入

DIは**疎結合**で**テストしやすい**コードを書くための重要な設計パターンです。最初は複雑に感じるかもしれませんが、プロジェクトが大きくなるほどその価値が実感できるでしょう。