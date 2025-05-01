# モジュラモノリスのパブリックAPIのPython実装例

PythonでモジュラモノリスのパブリックAPIを実装する例を示します。モジュラモノリスでは、各モジュールがパブリックAPIを通じて他のモジュールとコミュニケーションを取ります。これにより、モジュール間の結合を制御し、将来的なマイクロサービスへの移行を容易にします。

以下の例では、ECサイトの「注文モジュール」と「支払いモジュール」の間のパブリックAPIを実装します。

## プロジェクト構造

```
ecommerce/
├── shared/                 # 共有インターフェースと型定義
│   ├── __init__.py
│   ├── payments.py         # 支払いモジュールのパブリックAPI定義
│   └── orders.py           # 注文モジュールのパブリックAPI定義
├── payments/               # 支払いモジュール
│   ├── __init__.py
│   ├── implementation.py   # 内部実装（非公開）
│   └── service.py          # パブリックAPIの実装
└── orders/                 # 注文モジュール
    ├── __init__.py
    ├── implementation.py   # 内部実装（非公開）
    └── service.py          # パブリックAPIの実装
```

## 1. 共有インターフェースの定義

まず、`shared`パッケージに各モジュールのパブリックインターフェースと共有データモデルを定義します。

### shared/payments.py
```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Optional, List
from enum import Enum
from datetime import datetime
from uuid import UUID


class PaymentStatus(Enum):
    PENDING = "pending"
    COMPLETED = "completed"
    FAILED = "failed"
    REFUNDED = "refunded"


@dataclass
class PaymentRequest:
    order_id: UUID
    amount: float
    currency: str
    payment_method: str
    customer_id: UUID


@dataclass
class PaymentResult:
    payment_id: UUID
    status: PaymentStatus
    transaction_id: Optional[str] = None
    error_message: Optional[str] = None
    processed_at: Optional[datetime] = None


class IPaymentService(ABC):
    """支払いモジュールのパブリックAPI定義"""
    
    @abstractmethod
    def process_payment(self, request: PaymentRequest) -> PaymentResult:
        """支払いを処理する"""
        pass
    
    @abstractmethod
    def get_payment_status(self, payment_id: UUID) -> PaymentStatus:
        """支払いのステータスを取得する"""
        pass
    
    @abstractmethod
    def refund_payment(self, payment_id: UUID, amount: Optional[float] = None) -> PaymentResult:
        """支払いを返金する"""
        pass
    
    @abstractmethod
    def get_customer_payments(self, customer_id: UUID) -> List[PaymentResult]:
        """顧客の支払い履歴を取得する"""
        pass
```

### shared/orders.py
```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Optional, List
from enum import Enum
from datetime import datetime
from uuid import UUID


class OrderStatus(Enum):
    CREATED = "created"
    PAID = "paid"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"


@dataclass
class OrderItem:
    product_id: UUID
    quantity: int
    unit_price: float


@dataclass
class OrderDTO:
    order_id: UUID
    customer_id: UUID
    status: OrderStatus
    items: List[OrderItem]
    total_amount: float
    created_at: datetime
    payment_id: Optional[UUID] = None
    shipped_at: Optional[datetime] = None
    delivered_at: Optional[datetime] = None


@dataclass
class CreateOrderRequest:
    customer_id: UUID
    items: List[OrderItem]


class IOrderService(ABC):
    """注文モジュールのパブリックAPI定義"""
    
    @abstractmethod
    def create_order(self, request: CreateOrderRequest) -> OrderDTO:
        """注文を作成する"""
        pass
    
    @abstractmethod
    def get_order(self, order_id: UUID) -> Optional[OrderDTO]:
        """注文を取得する"""
        pass
    
    @abstractmethod
    def update_order_status(self, order_id: UUID, status: OrderStatus) -> bool:
        """注文ステータスを更新する"""
        pass
    
    @abstractmethod
    def set_payment_id(self, order_id: UUID, payment_id: UUID) -> bool:
        """注文に支払いIDを設定する"""
        pass
    
    @abstractmethod
    def get_customer_orders(self, customer_id: UUID) -> List[OrderDTO]:
        """顧客の注文履歴を取得する"""
        pass
```

## 2. モジュール実装

次に、各モジュールで実装を行います。実装はモジュール内部に隠蔽し、パブリックAPIを通じてのみアクセスできるようにします。

### payments/implementation.py
```python
# 内部実装 - モジュール外からは直接アクセスされない
import logging
from uuid import UUID, uuid4
from datetime import datetime
from typing import Dict, List, Optional

from ecommerce.shared.payments import PaymentRequest, PaymentResult, PaymentStatus


class PaymentProcessor:
    """支払い処理の内部実装"""
    
    def __init__(self):
        self.logger = logging.getLogger(__name__)
        # 実際のDBの代わりに単純な辞書を使用
        self._payments: Dict[UUID, PaymentResult] = {}
        self._customer_payments: Dict[UUID, List[UUID]] = {}
    
    def process(self, request: PaymentRequest) -> PaymentResult:
        """支払いを処理する内部メソッド"""
        self.logger.info(f"Processing payment for order {request.order_id}")
        
        # 実際の支払い処理のシミュレーション
        payment_id = uuid4()
        result = PaymentResult(
            payment_id=payment_id,
            status=PaymentStatus.COMPLETED,
            transaction_id=f"TX-{payment_id}",
            processed_at=datetime.now()
        )
        
        # 支払い情報を保存
        self._payments[payment_id] = result
        
        # 顧客の支払い履歴を更新
        if request.customer_id not in self._customer_payments:
            self._customer_payments[request.customer_id] = []
        self._customer_payments[request.customer_id].append(payment_id)
        
        return result
    
    def get_status(self, payment_id: UUID) -> Optional[PaymentStatus]:
        """支払いステータスを取得する内部メソッド"""
        if payment_id in self._payments:
            return self._payments[payment_id].status
        return None
    
    def refund(self, payment_id: UUID, amount: Optional[float] = None) -> Optional[PaymentResult]:
        """支払いを返金する内部メソッド"""
        if payment_id not in self._payments:
            return None
        
        payment = self._payments[payment_id]
        
        # 返金処理のシミュレーション
        refund_result = PaymentResult(
            payment_id=payment_id,
            status=PaymentStatus.REFUNDED,
            transaction_id=f"REFUND-{payment.transaction_id}",
            processed_at=datetime.now()
        )
        
        # 支払い情報を更新
        self._payments[payment_id] = refund_result
        
        return refund_result
    
    def get_customer_payments(self, customer_id: UUID) -> List[PaymentResult]:
        """顧客の支払い履歴を取得する内部メソッド"""
        if customer_id not in self._customer_payments:
            return []
        
        payment_ids = self._customer_payments[customer_id]
        return [self._payments[pid] for pid in payment_ids if pid in self._payments]
```

### payments/service.py
```python
from uuid import UUID
from typing import List, Optional

from ecommerce.shared.payments import IPaymentService, PaymentRequest, PaymentResult, PaymentStatus
from ecommerce.payments.implementation import PaymentProcessor


class PaymentService(IPaymentService):
    """支払いモジュールのパブリックAPI実装"""
    
    def __init__(self):
        self._processor = PaymentProcessor()
    
    def process_payment(self, request: PaymentRequest) -> PaymentResult:
        """支払いを処理する"""
        return self._processor.process(request)
    
    def get_payment_status(self, payment_id: UUID) -> PaymentStatus:
        """支払いのステータスを取得する"""
        status = self._processor.get_status(payment_id)
        if status is None:
            # 存在しない支払いの場合はデフォルト値を返す
            return PaymentStatus.FAILED
        return status
    
    def refund_payment(self, payment_id: UUID, amount: Optional[float] = None) -> PaymentResult:
        """支払いを返金する"""
        result = self._processor.refund(payment_id, amount)
        if result is None:
            # 存在しない支払いの場合はエラー結果を返す
            return PaymentResult(
                payment_id=payment_id,
                status=PaymentStatus.FAILED,
                error_message="Payment not found"
            )
        return result
    
    def get_customer_payments(self, customer_id: UUID) -> List[PaymentResult]:
        """顧客の支払い履歴を取得する"""
        return self._processor.get_customer_payments(customer_id)
```

### orders/implementation.py
```python
# 内部実装 - モジュール外からは直接アクセスされない
import logging
from uuid import UUID, uuid4
from datetime import datetime
from typing import Dict, List, Optional

from ecommerce.shared.orders import OrderDTO, OrderStatus, OrderItem, CreateOrderRequest


class OrderRepository:
    """注文データの内部実装"""
    
    def __init__(self):
        self.logger = logging.getLogger(__name__)
        # 実際のDBの代わりに単純な辞書を使用
        self._orders: Dict[UUID, OrderDTO] = {}
        self._customer_orders: Dict[UUID, List[UUID]] = {}
    
    def create(self, request: CreateOrderRequest) -> OrderDTO:
        """注文を作成する内部メソッド"""
        order_id = uuid4()
        total_amount = sum(item.quantity * item.unit_price for item in request.items)
        
        order = OrderDTO(
            order_id=order_id,
            customer_id=request.customer_id,
            status=OrderStatus.CREATED,
            items=request.items,
            total_amount=total_amount,
            created_at=datetime.now()
        )
        
        # 注文情報を保存
        self._orders[order_id] = order
        
        # 顧客の注文履歴を更新
        if request.customer_id not in self._customer_orders:
            self._customer_orders[request.customer_id] = []
        self._customer_orders[request.customer_id].append(order_id)
        
        return order
    
    def get(self, order_id: UUID) -> Optional[OrderDTO]:
        """注文を取得する内部メソッド"""
        return self._orders.get(order_id)
    
    def update_status(self, order_id: UUID, status: OrderStatus) -> bool:
        """注文ステータスを更新する内部メソッド"""
        if order_id not in self._orders:
            return False
        
        order = self._orders[order_id]
        order.status = status
        
        if status == OrderStatus.SHIPPED:
            order.shipped_at = datetime.now()
        elif status == OrderStatus.DELIVERED:
            order.delivered_at = datetime.now()
        
        self._orders[order_id] = order
        return True
    
    def set_payment(self, order_id: UUID, payment_id: UUID) -> bool:
        """注文に支払いIDを設定する内部メソッド"""
        if order_id not in self._orders:
            return False
        
        order = self._orders[order_id]
        order.payment_id = payment_id
        order.status = OrderStatus.PAID
        
        self._orders[order_id] = order
        return True
    
    def get_customer_orders(self, customer_id: UUID) -> List[OrderDTO]:
        """顧客の注文履歴を取得する内部メソッド"""
        if customer_id not in self._customer_orders:
            return []
        
        order_ids = self._customer_orders[customer_id]
        return [self._orders[oid] for oid in order_ids if oid in self._orders]
```

### orders/service.py
```python
from uuid import UUID
from typing import List, Optional

from ecommerce.shared.orders import IOrderService, OrderDTO, OrderStatus, CreateOrderRequest
from ecommerce.orders.implementation import OrderRepository


class OrderService(IOrderService):
    """注文モジュールのパブリックAPI実装"""
    
    def __init__(self):
        self._repository = OrderRepository()
    
    def create_order(self, request: CreateOrderRequest) -> OrderDTO:
        """注文を作成する"""
        return self._repository.create(request)
    
    def get_order(self, order_id: UUID) -> Optional[OrderDTO]:
        """注文を取得する"""
        return self._repository.get(order_id)
    
    def update_order_status(self, order_id: UUID, status: OrderStatus) -> bool:
        """注文ステータスを更新する"""
        return self._repository.update_status(order_id, status)
    
    def set_payment_id(self, order_id: UUID, payment_id: UUID) -> bool:
        """注文に支払いIDを設定する"""
        return self._repository.set_payment(order_id, payment_id)
    
    def get_customer_orders(self, customer_id: UUID) -> List[OrderDTO]:
        """顧客の注文履歴を取得する"""
        return self._repository.get_customer_orders(customer_id)
```

## 3. モジュール間の連携例

モジュール間のパブリックAPIを使用して、注文と支払いのワークフローを実現する例を示します。

```python
from uuid import UUID
from typing import Optional

from ecommerce.shared.orders import IOrderService, CreateOrderRequest
from ecommerce.shared.payments import IPaymentService, PaymentRequest, PaymentStatus


class CheckoutService:
    """注文と支払いのワークフローを処理するサービス"""
    
    def __init__(self, order_service: IOrderService, payment_service: IPaymentService):
        # 依存性注入によりモジュールのパブリックAPIを取得
        self.order_service = order_service
        self.payment_service = payment_service
    
    def checkout(self, create_order_request: CreateOrderRequest, payment_method: str) -> Optional[UUID]:
        """注文作成と支払い処理のワークフロー"""
        # 注文を作成
        order = self.order_service.create_order(create_order_request)
        
        # 支払いリクエストを作成
        payment_request = PaymentRequest(
            order_id=order.order_id,
            amount=order.total_amount,
            currency="USD",
            payment_method=payment_method,
            customer_id=order.customer_id
        )
        
        # 支払いを処理
        payment_result = self.payment_service.process_payment(payment_request)
        
        # 支払いが成功した場合、注文を更新
        if payment_result.status == PaymentStatus.COMPLETED:
            self.order_service.set_payment_id(order.order_id, payment_result.payment_id)
            return order.order_id
        
        return None
```

## 使用例

```python
from uuid import UUID
import json
from datetime import datetime

from ecommerce.orders.service import OrderService
from ecommerce.payments.service import PaymentService
from ecommerce.shared.orders import CreateOrderRequest, OrderItem


# DTOのJSONシリアライズのためのヘルパー関数
def custom_json_serializer(obj):
    if isinstance(obj, (datetime, UUID)):
        return str(obj)
    raise TypeError(f"Type {type(obj)} not serializable")


def main():
    # モジュールのサービスをインスタンス化
    order_service = OrderService()
    payment_service = PaymentService()
    
    # CheckoutServiceを作成
    from checkout_example import CheckoutService
    checkout_service = CheckoutService(order_service, payment_service)
    
    # 注文アイテムを作成
    items = [
        OrderItem(
            product_id=UUID("f47ac10b-58cc-4372-a567-0e02b2c3d479"),
            quantity=2,
            unit_price=19.99
        ),
        OrderItem(
            product_id=UUID("38c0059a-9b23-4c42-b4ee-24b811be61a0"),
            quantity=1,
            unit_price=29.99
        )
    ]
    
    # 注文リクエストを作成
    create_order_request = CreateOrderRequest(
        customer_id=UUID("550e8400-e29b-41d4-a716-446655440000"),
        items=items
    )
    
    # チェックアウトを実行
    order_id = checkout_service.checkout(create_order_request, "credit_card")
    
    if order_id:
        print(f"チェックアウト成功！注文ID: {order_id}")
        
        # 注文の詳細を取得
        order = order_service.get_order(order_id)
        if order:
            print("注文詳細:")
            print(json.dumps(order.__dict__, default=custom_json_serializer, indent=2))
    else:
        print("チェックアウト失敗")


if __name__ == "__main__":
    main()
```

## まとめ

この実装例では、モジュラモノリスの主要な原則を示しています：

1. **明確なインターフェース定義**: 抽象基底クラス(`ABC`)を使用して、各モジュールのパブリックAPIを定義しています。

2. **モジュール間の結合制御**: モジュールは互いに実装の詳細を知らず、インターフェースを通じてのみ通信します。

3. **カプセル化**: 実装の詳細はモジュール内に隠蔽され、外部からはアクセスできません。

4. **依存性注入**: モジュール間の依存関係を明示的に定義し、テストやリファクタリングを容易にします。

この構造により、将来的にはマイクロサービスへの移行も容易になります。各モジュールは明確に定義された境界を持ち、パブリックAPIを通じて通信するため、必要に応じて独立したサービスに分離することができます。

## 参考
モジュラモノリスのパブリックAPIについて、以下のURLを参照しました：

1. Milan Jovanovic氏のブログ「Internal vs. Public APIs in Modular Monoliths」
   - URL: https://www.milanjovanovic.tech/blog/internal-vs-public-apis-in-modular-monoliths
   - 内容: パブリックAPIはモジュール間の結合を防ぐのではなく制御するためのものであり、各パブリックAPIはモジュール間の依存関係を明示的に定義する契約のようなものと説明しています。

2. Milan Jovanovic氏のブログ「What Is a Modular Monolith?」
   - URL: https://www.milanjovanovic.tech/blog/what-is-a-modular-monolith
   - 内容: モジュールはパブリックAPIを通じてコミュニケーションを行い、各モジュールは明確に定義されたインターフェースを通じてアクセスされる関連機能のグループとして定義しています。

3. Milan Jovanovic氏のブログ「Modular Monolith Communication Patterns」
   - URL: https://www.milanjovanovic.tech/blog/modular-monolith-communication-patterns
   - 内容: モジュラモノリスでの通信パターンについて解説しており、モジュールはパブリックAPIを通じてのみ互いを参照すべきで、主に同期的なメソッド呼び出しと非同期的なメッセージングの2つの通信パターンが紹介されています。

4. TechTarget「Understanding the modular monolith and its ideal use cases」
   - URL: https://www.techtarget.com/searchapparchitecture/tip/Understanding-the-modular-monolith-and-its-ideal-use-cases
   - 内容: コード構造においてモジュールはパブリック定義を表すインターフェースを持ち、外部と内部で異なる形でインターフェースを公開する方法が説明されています。

5. ABP.IO「Modular Monolith Architecture with .NET」
   - URL: https://abp.io/architecture/modular-monolith
   - 内容: モジュールのコードベースを「実装パッケージ」と他のモジュールと共有するための「契約パッケージ」に分割する設計パターンについて説明しています。

Python実装のURLは上記の検索結果に含まれていなかったため、私が示した例はこれらの原則に基づいて独自に作成したものです。もしPythonでの具体的な実装例を知りたい場合は、追加で検索することができます。

6. https://stackoverflow.com/questions/72537043/cross-module-communication-in-modular-monolith