# DTOs（Data Transfer Objects）完全ガイド

## DTOsとは

**DTOs（Data Transfer Objects）** は、異なるシステムやレイヤー間でデータを安全かつ効率的に転送するための専用オブジェクトです。

データベースの内部構造をそのまま外部に公開せず、必要な情報だけを適切な形式で送受信するために使用されます。

## なぜDTOsが必要なのか

### 1. セキュリティの向上
- パスワードハッシュや内部管理情報などの機密データを隠蔽
- 外部に公開すべきでない情報の流出を防止

### 2. システム間の疎結合化
- データベース構造の変更がAPIに直接影響しない
- 内部実装と外部インターフェースを分離

### 3. データ形式の最適化
- 用途に応じて必要最小限のデータだけを転送
- ネットワーク負荷の軽減とパフォーマンス向上

## DTOsの双方向利用

DTOsは以下の両方向で使用されます：

| 方向 | 名称 | 説明 | 用途例 |
|------|------|------|--------|
| **内部→外部** | アウトバウンド | 内部データを外部向けに変換 | - APIレスポンス<br>- 外部システムへのデータ送信 |
| **外部→内部** | インバウンド | 外部データを内部形式に変換 | - 外部APIからのデータ取得<br>- フロントエンドからのリクエスト |

## 基本的な実装例

### データベースモデルとDTO

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Optional

# データベースモデル（内部的な詳細情報を含む）
@dataclass
class UserModel:
    id: int
    username: str
    email: str
    password_hash: str      # セキュリティ上、外部に公開すべきでない
    created_at: datetime
    last_login: datetime
    is_admin: bool
    internal_notes: str     # 内部管理用の情報

# DTO：APIレスポンス用（公開しても安全な情報のみ）
@dataclass
class UserResponseDTO:
    id: int
    username: str
    email: str
    member_since: str       # 日付を文字列形式に変換

# DTO：ユーザー作成リクエスト用
@dataclass
class CreateUserRequestDTO:
    username: str
    email: str
    password: str

# DTO：ユーザー一覧表示用（最小限の情報）
@dataclass
class UserListDTO:
    id: int
    username: str
    member_since: str
```

### 変換処理の実装

```python
class UserService:
    @staticmethod
    def convert_to_response_dto(user_model: UserModel) -> UserResponseDTO:
        """データベースモデルをAPIレスポンス用DTOに変換（アウトバウンド）"""
        return UserResponseDTO(
            id=user_model.id,
            username=user_model.username,
            email=user_model.email,
            member_since=user_model.created_at.strftime("%Y-%m-%d")
        )
    
    @staticmethod
    def convert_from_request_dto(request_dto: CreateUserRequestDTO) -> UserModel:
        """リクエストDTOを内部モデルに変換（インバウンド）"""
        return UserModel(
            id=0,  # 新規作成時は0、後でDBが割り当て
            username=request_dto.username,
            email=request_dto.email,
            password_hash=hash_password(request_dto.password),
            created_at=datetime.now(),
            last_login=None,
            is_admin=False,
            internal_notes=""
        )

# 使用例
def get_user_api(user_id: int) -> UserResponseDTO:
    # データベースからユーザー情報を取得
    user_model = database.get_user(user_id)
    
    # DTOに変換して安全な情報のみを返す
    return UserService.convert_to_response_dto(user_model)
```

## 実践的な例：外部API連携

### ECサイトでの商品情報管理

```python
# 内部商品モデル
@dataclass
class ProductModel:
    id: int
    name: str
    price: int
    cost: int               # 原価（内部情報）
    inventory_count: int
    supplier_id: int        # 仕入先情報（内部情報）
    description: str

# 顧客向け表示用DTO
@dataclass
class ProductDisplayDTO:
    id: int
    name: str
    price: int
    description: str
    in_stock: bool          # 在庫の有無のみ公開

# 管理者向け詳細DTO
@dataclass
class ProductManagementDTO:
    id: int
    name: str
    price: int
    cost: int
    profit_margin: float    # 利益率を計算して追加
    inventory_count: int
    supplier_name: str      # 仕入先名に変換

def convert_for_customer(product: ProductModel) -> ProductDisplayDTO:
    """顧客向けに変換"""
    return ProductDisplayDTO(
        id=product.id,
        name=product.name,
        price=product.price,
        description=product.description,
        in_stock=product.inventory_count > 0
    )

def convert_for_admin(product: ProductModel) -> ProductManagementDTO:
    """管理者向けに変換"""
    supplier_name = SupplierService.get_name(product.supplier_id)
    profit_margin = ((product.price - product.cost) / product.price) * 100
    
    return ProductManagementDTO(
        id=product.id,
        name=product.name,
        price=product.price,
        cost=product.cost,
        profit_margin=round(profit_margin, 2),
        inventory_count=product.inventory_count,
        supplier_name=supplier_name
    )
```

### 外部APIとの連携例

```python
# 外部気象APIからのデータ
@dataclass
class WeatherApiResponseDTO:
    location: str
    temperature_celsius: float
    humidity_percent: int
    weather_code: str
    timestamp: str

# 内部システムの気象データモデル
@dataclass
class InternalWeatherModel:
    id: int
    city_name: str
    temperature: float
    humidity: int
    condition: str          # 内部で管理する天気状態
    recorded_at: datetime
    data_source: str

# モバイルアプリ向けレスポンスDTO
@dataclass
class MobileWeatherDTO:
    location: str
    temp: int
    condition: str
    icon: str
    last_updated: str

class WeatherService:
    @staticmethod
    def import_from_external_api(api_data: WeatherApiResponseDTO) -> InternalWeatherModel:
        """外部APIデータを内部形式に変換（インバウンド）"""
        # 外部の天気コードを内部の条件に変換
        condition_map = {
            "01d": "晴れ",
            "02d": "曇り",
            "09d": "雨",
            "13d": "雪"
        }
        
        return InternalWeatherModel(
            id=0,
            city_name=api_data.location,
            temperature=api_data.temperature_celsius,
            humidity=api_data.humidity_percent,
            condition=condition_map.get(api_data.weather_code, "不明"),
            recorded_at=datetime.fromisoformat(api_data.timestamp),
            data_source="OpenWeatherMap"
        )
    
    @staticmethod
    def export_for_mobile_api(weather: InternalWeatherModel) -> MobileWeatherDTO:
        """内部データをモバイルアプリ用に変換（アウトバウンド）"""
        # 天気状態からアイコンを決定
        icon_map = {
            "晴れ": "☀️",
            "曇り": "☁️",
            "雨": "🌧️",
            "雪": "❄️"
        }
        
        return MobileWeatherDTO(
            location=weather.city_name,
            temp=int(weather.temperature),
            condition=weather.condition,
            icon=icon_map.get(weather.condition, "🌤️"),
            last_updated=weather.recorded_at.strftime("%H:%M")
        )
```

## DTOsのベストプラクティス

### 1. 明確な命名規則
- `ResponseDTO`：APIレスポンス用
- `RequestDTO`：APIリクエスト用
- `ListDTO`：一覧表示用
- `DetailDTO`：詳細表示用

### 2. バリデーションの実装
```python
@dataclass
class CreateUserRequestDTO:
    username: str
    email: str
    password: str
    
    def validate(self) -> list[str]:
        """バリデーションエラーのリストを返す"""
        errors = []
        
        if len(self.username) < 3:
            errors.append("ユーザー名は3文字以上である必要があります")
        
        if "@" not in self.email:
            errors.append("有効なメールアドレスを入力してください")
        
        if len(self.password) < 8:
            errors.append("パスワードは8文字以上である必要があります")
        
        return errors
```

### 3. 変換処理の一元化
```python
class DTOConverter:
    """DTO変換処理を一元管理"""
    
    @staticmethod
    def user_to_response(user: UserModel) -> UserResponseDTO:
        return UserResponseDTO(...)
    
    @staticmethod
    def user_to_list_item(user: UserModel) -> UserListDTO:
        return UserListDTO(...)
    
    @staticmethod
    def request_to_user(dto: CreateUserRequestDTO) -> UserModel:
        return UserModel(...)
```

## まとめ

DTOsは以下の利点を提供する重要なデザインパターンです：

- **セキュリティ**：機密情報の隠蔽
- **安定性**：内部変更からAPIを保護
- **最適化**：必要最小限のデータ転送
- **保守性**：明確な責任分離

特にWeb APIやマイクロサービス、外部API連携では必須の概念といえるでしょう。適切にDTOsを活用することで、安全で保守しやすいシステムを構築できます。