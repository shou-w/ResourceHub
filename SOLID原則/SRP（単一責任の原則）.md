SRP（Single Responsibility Principle：単一責任の原則）について、具体例を用いてわかりやすく説明します。

## SRP（単一責任の原則）とは

**「クラスが変更される理由は1つだけであるべき」**

これは、**1つのクラスは1つの責任（役割）のみを持つべき**という原則です。言い換えると、クラスを変更する理由が複数ある場合、そのクラスは複数の責任を持っているということになります。

## SRPに違反した例（問題のあるコード）

```python
# SRP違反：複数の責任を持つクラス
class Employee:
    def __init__(self, name, position, salary):
        self.name = name
        self.position = position
        self.salary = salary
    
    # 責任1: 従業員情報の管理
    def get_name(self):
        return self.name
    
    def set_salary(self, salary):
        self.salary = salary
    
    # 責任2: 給与計算（SRP違反）
    def calculate_monthly_salary(self):
        if self.position == "正社員":
            return self.salary / 12
        elif self.position == "契約社員":
            return self.salary * 0.8 / 12
        else:
            return self.salary * 0.6 / 12
    
    # 責任3: レポート生成（SRP違反）
    def generate_salary_report(self):
        monthly_salary = self.calculate_monthly_salary()
        return f"従業員名: {self.name}\n月給: {monthly_salary:,.0f}円"
    
    # 責任4: データベース操作（SRP違反）
    def save_to_database(self):
        print(f"データベースに{self.name}の情報を保存しました")
    
    # 責任5: メール送信（SRP違反）
    def send_salary_notification(self):
        print(f"{self.name}に給与明細をメール送信しました")

# 使用例
employee = Employee("田中太郎", "正社員", 4800000)
print(employee.generate_salary_report())
employee.save_to_database()
employee.send_salary_notification()
```

### この設計の問題点：

| 変更理由 | 影響するクラス |
|----------|----------------|
| **給与計算ルールの変更** | `Employee`クラス |
| **レポート形式の変更** | `Employee`クラス |
| **データベース構造の変更** | `Employee`クラス |
| **メール送信方法の変更** | `Employee`クラス |
| **従業員情報の項目追加** | `Employee`クラス |

1つのクラスが**5つの異なる理由で変更される**可能性があります！

## SRPに従った改善版

```python
# 責任1: 従業員情報の管理のみ
class Employee:
    def __init__(self, name, position, salary):
        self.name = name
        self.position = position
        self.salary = salary
    
    def get_name(self):
        return self.name
    
    def get_position(self):
        return self.position
    
    def get_salary(self):
        return self.salary
    
    def set_salary(self, salary):
        self.salary = salary

# 責任2: 給与計算のみ
class SalaryCalculator:
    def calculate_monthly_salary(self, employee):
        annual_salary = employee.get_salary()
        position = employee.get_position()
        
        if position == "正社員":
            return annual_salary / 12
        elif position == "契約社員":
            return annual_salary * 0.8 / 12
        else:
            return annual_salary * 0.6 / 12
    
    def calculate_bonus(self, employee, performance_rate):
        base_salary = employee.get_salary()
        return base_salary * 0.2 * performance_rate

# 責任3: レポート生成のみ
class SalaryReportGenerator:
    def __init__(self, salary_calculator):
        self.salary_calculator = salary_calculator
    
    def generate_basic_report(self, employee):
        monthly_salary = self.salary_calculator.calculate_monthly_salary(employee)
        return f"従業員名: {employee.get_name()}\n月給: {monthly_salary:,.0f}円"
    
    def generate_detailed_report(self, employee, performance_rate):
        monthly_salary = self.salary_calculator.calculate_monthly_salary(employee)
        bonus = self.salary_calculator.calculate_bonus(employee, performance_rate)
        
        return f"""
=== 給与明細 ===
従業員名: {employee.get_name()}
役職: {employee.get_position()}
基本月給: {monthly_salary:,.0f}円
ボーナス: {bonus:,.0f}円
合計: {monthly_salary + bonus:,.0f}円
        """.strip()

# 責任4: データベース操作のみ
class EmployeeRepository:
    def save(self, employee):
        print(f"データベースに{employee.get_name()}の情報を保存しました")
    
    def find_by_name(self, name):
        print(f"データベースから{name}を検索しました")
        return None
    
    def update_salary(self, employee_name, new_salary):
        print(f"{employee_name}の給与を{new_salary}に更新しました")

# 責任5: 通知送信のみ
class NotificationService:
    def send_salary_notification(self, employee, report):
        print(f"{employee.get_name()}に給与明細をメール送信しました")
        print("送信内容:")
        print(report)
    
    def send_bonus_notification(self, employee, bonus_amount):
        print(f"{employee.get_name()}にボーナス通知を送信: {bonus_amount:,.0f}円")

# 使用例
# 各責任が分離された使い方
employee = Employee("田中太郎", "正社員", 4800000)

salary_calculator = SalaryCalculator()
report_generator = SalaryReportGenerator(salary_calculator)
repository = EmployeeRepository()
notification_service = NotificationService()

# 基本レポート生成
basic_report = report_generator.generate_basic_report(employee)
print(basic_report)

# 詳細レポート生成
detailed_report = report_generator.generate_detailed_report(employee, 1.2)
print(detailed_report)

# データ保存
repository.save(employee)

# 通知送信
notification_service.send_salary_notification(employee, detailed_report)
```

## 責任の判断基準

### 「責任」をどう見極めるか？

```python
# 質問: このクラスが変更される理由は何か？

class BookManager:  # この名前からして複数の責任がありそう
    def __init__(self):
        self.books = []
    
    # 変更理由1: 書籍データの管理方法が変わる
    def add_book(self, book):
        self.books.append(book)
    
    # 変更理由2: 検索アルゴリズムが変わる
    def search_books(self, keyword):
        return [book for book in self.books if keyword in book.title]
    
    # 変更理由3: レポート形式が変わる
    def generate_inventory_report(self):
        return f"総書籍数: {len(self.books)}"
    
    # 変更理由4: データベースが変わる
    def save_to_database(self):
        print("書籍データをデータベースに保存")
```

### 改善後：責任を分離

```python
# 書籍データモデル
class Book:
    def __init__(self, title, author, isbn):
        self.title = title
        self.author = author
        self.isbn = isbn

# 責任1: 書籍データの管理
class BookCollection:
    def __init__(self):
        self.books = []
    
    def add_book(self, book):
        self.books.append(book)
    
    def remove_book(self, isbn):
        self.books = [book for book in self.books if book.isbn != isbn]
    
    def get_all_books(self):
        return self.books.copy()

# 責任2: 書籍検索
class BookSearchService:
    def search_by_title(self, books, keyword):
        return [book for book in books if keyword.lower() in book.title.lower()]
    
    def search_by_author(self, books, author):
        return [book for book in books if author.lower() in book.author.lower()]
    
    def search_by_isbn(self, books, isbn):
        return [book for book in books if book.isbn == isbn]

# 責任3: レポート生成
class BookReportGenerator:
    def generate_inventory_report(self, books):
        return f"総書籍数: {len(books)}"
    
    def generate_author_summary(self, books):
        authors = {}
        for book in books:
            authors[book.author] = authors.get(book.author, 0) + 1
        return authors

# 責任4: データ永続化
class BookRepository:
    def save_books(self, books):
        print(f"{len(books)}冊の書籍をデータベースに保存")
    
    def load_books(self):
        print("データベースから書籍を読み込み")
        return []

# 使用例
collection = BookCollection()
search_service = BookSearchService()
report_generator = BookReportGenerator()
repository = BookRepository()

# 書籍追加
book1 = Book("Python入門", "田中太郎", "978-1234567890")
book2 = Book("Java基礎", "佐藤花子", "978-0987654321")
collection.add_book(book1)
collection.add_book(book2)

# 検索
results = search_service.search_by_title(collection.get_all_books(), "Python")
print(f"検索結果: {len(results)}冊")

# レポート
report = report_generator.generate_inventory_report(collection.get_all_books())
print(report)

# 保存
repository.save_books(collection.get_all_books())
```

## 実際の開発でのSRP適用ガイドライン

### 1. クラス名で判断
```python
# SRP違反の可能性が高い名前
class UserManager      # 何を「管理」するのか曖昧
class DataHandler      # 何を「処理」するのか曖昧
class SystemController # 何を「制御」するのか曖昧

# SRP適用の良い名前
class User            # ユーザー情報のみ
class UserValidator   # ユーザー検証のみ
class UserRepository  # ユーザーデータ永続化のみ
```

### 2. メソッド数で判断
```python
class TooManyResponsibilities:
    # 10個以上のメソッドがある場合は要注意
    def method1(self): pass
    def method2(self): pass
    # ... 15個のメソッド
    # 責任が分散している可能性が高い
```

### 3. 変更頻度で判断
```python
# 異なる理由で頻繁に変更されるクラスは分離を検討
class FrequentlyChangedClass:
    def business_logic(self):  # ビジネス要件で変更
        pass
    
    def data_access(self):     # DB変更で修正
        pass
    
    def ui_formatting(self):   # UI要件で変更
        pass
```

## SRPの利点まとめ

| 利点 | 説明 | 具体例 |
|------|------|--------|
| **保守性向上** | 変更範囲が限定される | 給与計算ルール変更時、`SalaryCalculator`のみ修正 |
| **テストしやすさ** | 単一の責任をテストすれば良い | `BookSearchService`のテストは検索機能のみ |
| **再利用性** | 特定の責任だけを他で使える | `NotificationService`を他のシステムでも利用 |
| **理解しやすさ** | クラスの役割が明確 | `BookRepository`はデータ操作専門と一目瞭然 |
| **並行開発** | 異なる責任を異なる開発者が担当可能 | UI担当とDB担当が独立して作業 |

## まとめ

SRPは**「1つのクラスには1つの変更理由のみ」**という原則です。

### 実践のポイント：
- **クラス名が明確**で単一の責任を表している
- **メソッドが関連性**を持っている
- **変更理由が1つ**に限定される
- **他の責任に影響を与えない**

SRPを適用することで、**保守しやすく、テストしやすく、理解しやすい**コードを書くことができます。最初は分離しすぎに感じるかもしれませんが、プロジェクトが成長するにつれて、その価値を実感できるでしょう。