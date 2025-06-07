# LSP（Liskov Substitution Principle：リスコフの置換原則）

LSP（Liskov Substitution Principle：リスコフの置換原則）について、具体例を用いてわかりやすく説明します。

## LSP（リスコフの置換原則）とは

**「派生クラスは、基底クラスと置換可能でなければならない」**

これは、**基底クラスの参照を使っているコードで、派生クラスのオブジェクトを使っても正常に動作する**べきという原則です。言い換えると、**基底クラスで期待される振る舞いを、派生クラスも保持する**必要があります。

## LSPに違反した例（問題のあるコード）

### 例1: 鳥の継承問題

```python
# LSP違反の典型例
class Bird:
    def __init__(self, name):
        self.name = name
    
    def fly(self):
        return f"{self.name}が飛んでいます"
    
    def make_sound(self):
        return f"{self.name}が鳴いています"

class Sparrow(Bird):
    def fly(self):
        return f"スズメ{self.name}が空を飛んでいます"
    
    def make_sound(self):
        return f"スズメ{self.name}がチュンチュン鳴いています"

class Penguin(Bird):
    def fly(self):
        # ペンギンは飛べない！LSP違反
        raise Exception("ペンギンは飛べません")
    
    def make_sound(self):
        return f"ペンギン{self.name}がガーガー鳴いています"

# 問題のある使用例
def make_birds_fly(birds):
    results = []
    for bird in birds:
        # 基底クラス（Bird）の参照で呼び出し
        results.append(bird.fly())  # Penguinで例外発生！
    return results

# 使用例
birds = [
    Sparrow("ピーちゃん"),
    Penguin("ペンペン")  # これが問題を引き起こす
]

# print(make_birds_fly(birds))  # 例外発生！LSP違反
```

### 例2: 矩形と正方形の問題

```python
# LSP違反：正方形は数学的には矩形だが、プログラムでは問題がある
class Rectangle:
    def __init__(self, width, height):
        self._width = width
        self._height = height
    
    def set_width(self, width):
        self._width = width
    
    def set_height(self, height):
        self._height = height
    
    def get_width(self):
        return self._width
    
    def get_height(self):
        return self._height
    
    def get_area(self):
        return self._width * self._height

class Square(Rectangle):
    def __init__(self, side):
        super().__init__(side, side)
    
    def set_width(self, width):
        # 正方形なので幅と高さを同じにする
        self._width = width
        self._height = width  # LSP違反：期待される動作と異なる
    
    def set_height(self, height):
        # 正方形なので幅と高さを同じにする
        self._width = height   # LSP違反：期待される動作と異なる
        self._height = height

# 問題のある使用例
def test_rectangle_area(rectangle):
    # 矩形の幅を5、高さを4に設定して面積20を期待
    rectangle.set_width(5)
    rectangle.set_height(4)
    expected_area = 5 * 4  # 20を期待
    actual_area = rectangle.get_area()
    
    print(f"期待値: {expected_area}, 実際: {actual_area}")
    return expected_area == actual_area

# テスト
normal_rectangle = Rectangle(0, 0)
square = Square(0)

print("通常の矩形:")
print(test_rectangle_area(normal_rectangle))  # True

print("正方形:")
print(test_rectangle_area(square))  # False！LSP違反
```

### この設計の問題点：

| 問題 | 影響 |
|------|------|
| **例外の発生** | 基底クラスでは例外が発生しないが、派生クラスで例外発生 |
| **期待と異なる動作** | 基底クラスでの期待される振る舞いが派生クラスで変わる |
| **置換不可能** | 基底クラスの参照で安全に派生クラスを使えない |

## LSPに従った改善版

### 改善例1: 鳥の階層構造を再設計

```python
from abc import ABC, abstractmethod

# 基底クラス：全ての鳥に共通する行動のみ定義
class Bird(ABC):
    def __init__(self, name):
        self.name = name
    
    @abstractmethod
    def make_sound(self):
        pass
    
    @abstractmethod
    def move(self):
        """移動方法（飛ぶ、歩く、泳ぐなど）"""
        pass
    
    def get_name(self):
        return self.name

# 飛べる鳥の抽象クラス
class FlyingBird(Bird):
    @abstractmethod
    def fly(self):
        pass
    
    def move(self):
        return self.fly()

# 泳げる鳥の抽象クラス
class SwimmingBird(Bird):
    @abstractmethod
    def swim(self):
        pass
    
    def move(self):
        return self.swim()

# 飛べる鳥の実装
class Sparrow(FlyingBird):
    def fly(self):
        return f"スズメ{self.name}が空を飛んでいます"
    
    def make_sound(self):
        return f"スズメ{self.name}がチュンチュン鳴いています"

class Eagle(FlyingBird):
    def fly(self):
        return f"鷲{self.name}が大空を舞っています"
    
    def make_sound(self):
        return f"鷲{self.name}がキーキー鳴いています"

# 泳げる鳥の実装
class Penguin(SwimmingBird):
    def swim(self):
        return f"ペンギン{self.name}が水中を泳いでいます"
    
    def make_sound(self):
        return f"ペンギン{self.name}がガーガー鳴いています"

class Duck(FlyingBird, SwimmingBird):
    """アヒルは飛ぶことも泳ぐこともできる"""
    def fly(self):
        return f"アヒル{self.name}が飛んでいます"
    
    def swim(self):
        return f"アヒル{self.name}が泳いでいます"
    
    def make_sound(self):
        return f"アヒル{self.name}がガーガー鳴いています"
    
    def move(self):
        # アヒルは状況に応じて飛ぶか泳ぐかを選択
        return f"アヒル{self.name}が移動しています（飛行または水泳）"

# LSPに準拠した使用例
def make_birds_move(birds):
    """全ての鳥が安全に移動できる"""
    results = []
    for bird in birds:
        results.append(bird.move())  # どの鳥でも安全に呼び出せる
    return results

def make_birds_sound(birds):
    """全ての鳥が安全に鳴ける"""
    results = []
    for bird in birds:
        results.append(bird.make_sound())  # どの鳥でも安全に呼び出せる
    return results

def make_flying_birds_fly(flying_birds):
    """飛べる鳥だけを対象とした処理"""
    results = []
    for bird in flying_birds:
        results.append(bird.fly())  # FlyingBirdなら安全に呼び出せる
    return results

# 使用例
all_birds = [
    Sparrow("ピーちゃん"),
    Eagle("イーグル"),
    Penguin("ペンペン"),
    Duck("ダッキー")
]

flying_birds = [
    Sparrow("ピーちゃん"),
    Eagle("イーグル"),
    Duck("ダッキー")
]

# 全ての鳥で安全に実行可能
print("=== 全ての鳥の移動 ===")
for result in make_birds_move(all_birds):
    print(result)

print("\n=== 全ての鳥の鳴き声 ===")
for result in make_birds_sound(all_birds):
    print(result)

print("\n=== 飛べる鳥の飛行 ===")
for result in make_flying_birds_fly(flying_birds):
    print(result)
```

### 改善例2: 図形の階層構造

```python
from abc import ABC, abstractmethod
import math

# 基底クラス：全ての図形に共通する行動のみ
class Shape(ABC):
    @abstractmethod
    def get_area(self):
        pass
    
    @abstractmethod
    def get_perimeter(self):
        pass
    
    @abstractmethod
    def get_shape_type(self):
        pass

# サイズ変更可能な図形
class ResizableShape(Shape):
    @abstractmethod
    def scale(self, factor):
        """図形を指定倍率でスケールする"""
        pass

# 固定比率でサイズ変更可能な図形
class UniformlyResizableShape(Shape):
    @abstractmethod
    def scale_uniformly(self, factor):
        """図形を縦横同じ比率でスケールする"""
        pass

# 矩形（自由にサイズ変更可能）
class Rectangle(ResizableShape):
    def __init__(self, width, height):
        self.width = width
        self.height = height
    
    def get_area(self):
        return self.width * self.height
    
    def get_perimeter(self):
        return 2 * (self.width + self.height)
    
    def get_shape_type(self):
        return "Rectangle"
    
    def scale(self, width_factor, height_factor=None):
        """幅と高さを独立してスケール"""
        if height_factor is None:
            height_factor = width_factor
        
        self.width *= width_factor
        self.height *= height_factor
    
    def set_dimensions(self, width, height):
        self.width = width
        self.height = height

# 正方形（比率固定でサイズ変更）
class Square(UniformlyResizableShape):
    def __init__(self, side):
        self.side = side
    
    def get_area(self):
        return self.side * self.side
    
    def get_perimeter(self):
        return 4 * self.side
    
    def get_shape_type(self):
        return "Square"
    
    def scale_uniformly(self, factor):
        """正方形は常に同じ比率でスケール"""
        self.side *= factor
    
    def set_side(self, side):
        self.side = side

# 円（比率固定でサイズ変更）
class Circle(UniformlyResizableShape):
    def __init__(self, radius):
        self.radius = radius
    
    def get_area(self):
        return math.pi * self.radius * self.radius
    
    def get_perimeter(self):
        return 2 * math.pi * self.radius
    
    def get_shape_type(self):
        return "Circle"
    
    def scale_uniformly(self, factor):
        self.radius *= factor

# LSPに準拠した使用例
def calculate_total_area(shapes):
    """どの図形でも安全に面積計算可能"""
    total = 0
    for shape in shapes:
        total += shape.get_area()  # 全ての図形で安全
    return total

def scale_uniform_shapes(shapes, factor):
    """比率固定でスケール可能な図形のみを対象"""
    results = []
    for shape in shapes:
        if isinstance(shape, UniformlyResizableShape):
            original_area = shape.get_area()
            shape.scale_uniformly(factor)
            new_area = shape.get_area()
            results.append(f"{shape.get_shape_type()}: {original_area:.2f} → {new_area:.2f}")
    return results

def print_shape_info(shapes):
    """全ての図形の情報を表示"""
    for shape in shapes:
        print(f"{shape.get_shape_type()}: 面積={shape.get_area():.2f}, 周囲={shape.get_perimeter():.2f}")

# 使用例
shapes = [
    Rectangle(5, 4),
    Square(3),
    Circle(2)
]

print("=== 初期状態 ===")
print_shape_info(shapes)

print(f"\n総面積: {calculate_total_area(shapes):.2f}")

print("\n=== 2倍にスケール後 ===")
uniform_shapes = [shape for shape in shapes if isinstance(shape, UniformlyResizableShape)]
scale_results = scale_uniform_shapes(uniform_shapes, 2)
for result in scale_results:
    print(result)

print_shape_info(shapes)
```

## LSPの実践的な適用例

### 例3: データ処理システム

```python
from abc import ABC, abstractmethod

class DataProcessor(ABC):
    """データ処理の基底クラス"""
    
    @abstractmethod
    def process(self, data):
        """データを処理して結果を返す"""
        pass
    
    @abstractmethod
    def validate_input(self, data):
        """入力データが有効かチェック"""
        pass
    
    def get_processor_name(self):
        return self.__class__.__name__

# 具体的な処理クラス（LSPに準拠）
class NumberProcessor(DataProcessor):
    def validate_input(self, data):
        return isinstance(data, (int, float)) and data >= 0
    
    def process(self, data):
        if not self.validate_input(data):
            raise ValueError("数値データが必要です（0以上）")
        return data * 2

class TextProcessor(DataProcessor):
    def validate_input(self, data):
        return isinstance(data, str) and len(data) > 0
    
    def process(self, data):
        if not self.validate_input(data):
            raise ValueError("空でない文字列が必要です")
        return data.upper()

class ListProcessor(DataProcessor):
    def validate_input(self, data):
        return isinstance(data, list) and len(data) > 0
    
    def process(self, data):
        if not self.validate_input(data):
            raise ValueError("空でないリストが必要です")
        return sorted(data)

# データ処理マネージャー（LSPに依存）
class DataProcessingManager:
    def __init__(self):
        self.processors = []
    
    def add_processor(self, processor: DataProcessor):
        self.processors.append(processor)
    
    def process_data_with_all_processors(self, data_items):
        """全てのプロセッサーで安全にデータ処理"""
        results = []
        for data in data_items:
            for processor in self.processors:
                try:
                    if processor.validate_input(data):
                        result = processor.process(data)
                        results.append({
                            'processor': processor.get_processor_name(),
                            'input': data,
                            'output': result,
                            'status': 'success'
                        })
                    else:
                        results.append({
                            'processor': processor.get_processor_name(),
                            'input': data,
                            'output': None,
                            'status': 'skipped (invalid input)'
                        })
                except Exception as e:
                    results.append({
                        'processor': processor.get_processor_name(),
                        'input': data,
                        'output': None,
                        'status': f'error: {str(e)}'
                    })
        return results

# 使用例
manager = DataProcessingManager()
manager.add_processor(NumberProcessor())
manager.add_processor(TextProcessor())
manager.add_processor(ListProcessor())

# 様々なデータタイプをテスト
test_data = [
    10,           # 数値
    "hello",      # 文字列
    [3, 1, 4, 2], # リスト
    -5,           # 負の数値（NumberProcessorで無効）
    "",           # 空文字列（TextProcessorで無効）
    []            # 空リスト（ListProcessorで無効）
]

results = manager.process_data_with_all_processors(test_data)

print("=== データ処理結果 ===")
for result in results:
    print(f"{result['processor']}: {result['input']} → {result['output']} ({result['status']})")
```

## LSPの重要なポイント

### 契約による設計（Design by Contract）

```python
class BankAccount:
    """銀行口座の基底クラス"""
    
    def __init__(self, balance=0):
        self._balance = balance
    
    def deposit(self, amount):
        """
        入金処理
        事前条件: amount > 0
        事後条件: 残高が増加する
        """
        if amount <= 0:
            raise ValueError("入金額は正の数である必要があります")
        
        old_balance = self._balance
        self._balance += amount
        
        # 事後条件をチェック
        assert self._balance > old_balance, "残高が増加していません"
        return self._balance
    
    def withdraw(self, amount):
        """
        出金処理
        事前条件: amount > 0 and amount <= balance
        事後条件: 残高が減少する
        """
        if amount <= 0:
            raise ValueError("出金額は正の数である必要があります")
        if amount > self._balance:
            raise ValueError("残高不足です")
        
        old_balance = self._balance
        self._balance -= amount
        
        # 事後条件をチェック
        assert self._balance < old_balance, "残高が減少していません"
        return self._balance
    
    def get_balance(self):
        return self._balance

# LSP準拠の派生クラス
class SavingsAccount(BankAccount):
    """貯蓄口座（基底クラスの契約を守る）"""
    
    def __init__(self, balance=0, interest_rate=0.02):
        super().__init__(balance)
        self.interest_rate = interest_rate
    
    def apply_interest(self):
        """利息を適用"""
        interest = self._balance * self.interest_rate
        self.deposit(interest)  # depositメソッドの契約を守る
        return interest

class CheckingAccount(BankAccount):
    """当座預金口座（基底クラスの契約を守る）"""
    
    def __init__(self, balance=0, overdraft_limit=1000):
        super().__init__(balance)
        self.overdraft_limit = overdraft_limit
    
    def withdraw(self, amount):
        """オーバードラフト機能付きの出金"""
        if amount <= 0:
            raise ValueError("出金額は正の数である必要があります")
        
        # オーバードラフト限度額を考慮
        available_amount = self._balance + self.overdraft_limit
        if amount > available_amount:
            raise ValueError("オーバードラフト限度額を超えています")
        
        old_balance = self._balance
        self._balance -= amount
        
        # 基底クラスより緩い条件だが、同じ契約を守る
        return self._balance

# LSPテスト
def test_account_operations(account: BankAccount):
    """どの口座タイプでも安全に操作可能"""
    print(f"\n=== {account.__class__.__name__} テスト ===")
    
    # 初期残高
    print(f"初期残高: {account.get_balance():,}円")
    
    # 入金テスト
    account.deposit(10000)
    print(f"10,000円入金後: {account.get_balance():,}円")
    
    # 出金テスト
    account.withdraw(5000)
    print(f"5,000円出金後: {account.get_balance():,}円")
    
    return account.get_balance()

# 使用例
accounts = [
    BankAccount(50000),
    SavingsAccount(50000, 0.03),
    CheckingAccount(50000, 20000)
]

# 全ての口座タイプで同じ操作が可能（LSP準拠）
for account in accounts:
    final_balance = test_account_operations(account)
    
    # 特定の機能をテスト
    if isinstance(account, SavingsAccount):
        interest = account.apply_interest()
        print(f"利息適用: {interest:,.0f}円, 最終残高: {account.get_balance():,}円")
    elif isinstance(account, CheckingAccount):
        try:
            account.withdraw(60000)  # オーバードラフトテスト
            print(f"オーバードラフト後: {account.get_balance():,}円")
        except ValueError as e:
            print(f"オーバードラフト制限: {e}")
```

## LSPのチェックリスト

| チェック項目 | 良い例 | 悪い例 |
|--------------|--------|--------|
| **例外の発生** | 基底クラスと同じか、より具体的な例外 | 基底クラスにない新しい例外 |
| **戻り値の型** | 基底クラスと同じか、より具体的な型 | 基底クラスと互換性のない型 |
| **事前条件** | 基底クラスと同じか、より緩い条件 | 基底クラスより厳しい条件 |
| **事後条件** | 基底クラスと同じか、より強い保証 | 基底クラスより弱い保証 |
| **不変条件** | 基底クラスの不変条件を維持 | 基底クラスの不変条件を破る |

## LSPの利点まとめ

| 利点 | 説明 |
|------|------|
| **安全な置換** | 基底クラスの参照で派生クラスを安全に使用可能 |
| **予測可能性** | 基底クラスの契約に基づいて動作が予測できる |
| **ポリモーフィズム** | 真の多態性を実現 |
| **保守性** | 新しい派生クラス追加時も既存コードが安全 |
| **テスト性** | 基底クラス用のテストが派生クラスでも有効 |

## まとめ

LSPは**「派生クラスは基底クラスと置換可能」**という原則です。

### 実践のポイント：
- **基底クラスの契約を守る**：事前条件、事後条件、不変条件
- **例外を追加しない**：基底クラスで想定していない例外を投げない
- **適切な階層設計**：「is-a」関係が成り立つ場合のみ継承
- **契約による設計**：明確な入力・出力・副作用の定義

LSPを適用することで、**安全で予測可能なポリモーフィズム**を実現でき、**保守性とテスト性の高いシステム**を構築できます。基底クラスの約束を守ることで、どの派生クラスでも安心して使えるコードが書けるでしょう。