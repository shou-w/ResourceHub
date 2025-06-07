# ISP（Interface Segregation Principle：インターフェース分離の原則）

ISP（Interface Segregation Principle：インターフェース分離の原則）について、具体例を用いてわかりやすく説明します。

## ISP（インターフェース分離の原則）とは

**「クライアントは使用しないメソッドに依存することを強制されるべきではない」**

これは、**大きく肥大化したインターフェースを小さく特化したインターフェースに分離する**べきという原則です。言い換えると、**クラスは必要な機能のみを実装すれば良い**ということです。

## ISPに違反した例（問題のあるコード）

### 例1: 肥大化した作業者インターフェース

```python
from abc import ABC, abstractmethod

# ISP違反：肥大化したインターフェース（問題）
class WorkerInterface(ABC):
    """全ての作業者が実装しなければならない巨大なインターフェース"""
    
    @abstractmethod
    def work(self):
        pass
    
    @abstractmethod
    def eat(self):
        pass
    
    @abstractmethod
    def sleep(self):
        pass
    
    @abstractmethod
    def take_vacation(self):
        pass
    
    @abstractmethod
    def get_salary(self):
        pass

class HumanWorker(WorkerInterface):
    """人間の作業者"""
    def work(self):
        return "人間が働いています"
    
    def eat(self):
        return "人間が食事しています"
    
    def sleep(self):
        return "人間が睡眠しています"
    
    def take_vacation(self):
        return "人間が休暇を取っています"
    
    def get_salary(self):
        return "人間が給与を受け取っています"

class RobotWorker(WorkerInterface):
    """ロボットの作業者"""
    def work(self):
        return "ロボットが働いています"
    
    def eat(self):
        # ロボットは食事しない！ISP違反
        raise NotImplementedError("ロボットは食事しません")
    
    def sleep(self):
        # ロボットは睡眠しない！ISP違反
        raise NotImplementedError("ロボットは睡眠しません")
    
    def take_vacation(self):
        # ロボットは休暇を取らない！ISP違反
        raise NotImplementedError("ロボットは休暇を取りません")
    
    def get_salary(self):
        # ロボットは給与を受け取らない！ISP違反
        raise NotImplementedError("ロボットは給与を受け取りません")

class ContractWorker(WorkerInterface):
    """契約作業者"""
    def work(self):
        return "契約作業者が働いています"
    
    def eat(self):
        return "契約作業者が食事しています"
    
    def sleep(self):
        return "契約作業者が睡眠しています"
    
    def take_vacation(self):
        # 契約作業者は有給休暇がない！ISP違反
        raise NotImplementedError("契約作業者は有給休暇がありません")
    
    def get_salary(self):
        return "契約作業者が報酬を受け取っています"

# 問題のある使用例
def manage_workers(workers):
    results = []
    for worker in workers:
        try:
            results.append(worker.work())
            results.append(worker.eat())     # RobotWorkerで例外発生！
            results.append(worker.sleep())   # RobotWorkerで例外発生！
            results.append(worker.take_vacation())  # Robot、Contractで例外発生！
        except NotImplementedError as e:
            results.append(f"エラー: {e}")
    return results

# 使用例
workers = [
    HumanWorker(),
    RobotWorker(),      # 複数の機能で例外発生
    ContractWorker()    # 一部の機能で例外発生
]

# print(manage_workers(workers))  # 複数の例外が発生！
```

### この設計の問題点：

| 問題 | 影響 |
|------|------|
| **不要な実装の強制** | ロボットが食事・睡眠メソッドを実装する必要 |
| **例外の頻発** | 使えない機能を呼び出すと例外発生 |
| **保守性の低下** | インターフェース変更時の影響範囲が広い |
| **理解しにくい** | どのクラスがどの機能を本当に持つか不明 |

## ISPに従った改善版

```python
from abc import ABC, abstractmethod

# 小さく分離されたインターフェース群
class WorkerInterface(ABC):
    """基本的な作業機能"""
    @abstractmethod
    def work(self):
        pass

class EaterInterface(ABC):
    """食事機能"""
    @abstractmethod
    def eat(self):
        pass

class SleeperInterface(ABC):
    """睡眠機能"""
    @abstractmethod
    def sleep(self):
        pass

class VacationTakerInterface(ABC):
    """休暇取得機能"""
    @abstractmethod
    def take_vacation(self):
        pass

class SalaryReceiverInterface(ABC):
    """給与受取機能"""
    @abstractmethod
    def get_salary(self):
        pass

# 必要なインターフェースのみを実装
class HumanWorker(WorkerInterface, EaterInterface, SleeperInterface, 
                  VacationTakerInterface, SalaryReceiverInterface):
    """人間は全ての機能を持つ"""
    def work(self):
        return "人間が働いています"
    
    def eat(self):
        return "人間が食事しています"
    
    def sleep(self):
        return "人間が睡眠しています"
    
    def take_vacation(self):
        return "人間が休暇を取っています"
    
    def get_salary(self):
        return "人間が給与を受け取っています"

class RobotWorker(WorkerInterface):
    """ロボットは作業のみ"""
    def work(self):
        return "ロボットが働いています"

class ContractWorker(WorkerInterface, EaterInterface, SleeperInterface, SalaryReceiverInterface):
    """契約作業者は休暇以外の機能を持つ"""
    def work(self):
        return "契約作業者が働いています"
    
    def eat(self):
        return "契約作業者が食事しています"
    
    def sleep(self):
        return "契約作業者が睡眠しています"
    
    def get_salary(self):
        return "契約作業者が報酬を受け取っています"

# ISPに準拠した管理システム
class WorkerManager:
    def manage_work(self, workers):
        """全ての作業者に作業を依頼"""
        results = []
        for worker in workers:
            if isinstance(worker, WorkerInterface):
                results.append(worker.work())
        return results
    
    def manage_meals(self, workers):
        """食事可能な作業者のみに食事を提供"""
        results = []
        for worker in workers:
            if isinstance(worker, EaterInterface):
                results.append(worker.eat())
        return results
    
    def manage_sleep(self, workers):
        """睡眠可能な作業者のみに休憩を提供"""
        results = []
        for worker in workers:
            if isinstance(worker, SleeperInterface):
                results.append(worker.sleep())
        return results
    
    def manage_vacations(self, workers):
        """休暇取得可能な作業者のみに休暇を提供"""
        results = []
        for worker in workers:
            if isinstance(worker, VacationTakerInterface):
                results.append(worker.take_vacation())
        return results
    
    def pay_salaries(self, workers):
        """給与受取可能な作業者のみに給与を支払い"""
        results = []
        for worker in workers:
            if isinstance(worker, SalaryReceiverInterface):
                results.append(worker.get_salary())
        return results

# 使用例
workers = [
    HumanWorker(),
    RobotWorker(),
    ContractWorker()
]

manager = WorkerManager()

print("=== 作業管理 ===")
for result in manager.manage_work(workers):
    print(result)

print("\n=== 食事管理 ===")
for result in manager.manage_meals(workers):
    print(result)

print("\n=== 睡眠管理 ===")
for result in manager.manage_sleep(workers):
    print(result)

print("\n=== 休暇管理 ===")
for result in manager.manage_vacations(workers):
    print(result)

print("\n=== 給与管理 ===")
for result in manager.pay_salaries(workers):
    print(result)
```

## より実践的な例：マルチメディアデバイス

### ISP違反の例

```python
# ISP違反：肥大化したマルチメディアインターフェース
class MultimediaDeviceInterface(ABC):
    """全てのマルチメディア機能を含む巨大なインターフェース（問題）"""
    
    @abstractmethod
    def play_audio(self, file_path):
        pass
    
    @abstractmethod
    def play_video(self, file_path):
        pass
    
    @abstractmethod
    def record_audio(self):
        pass
    
    @abstractmethod
    def record_video(self):
        pass
    
    @abstractmethod
    def take_photo(self):
        pass
    
    @abstractmethod
    def stream_content(self, url):
        pass
    
    @abstractmethod
    def edit_content(self, content):
        pass

class SimpleAudioPlayer(MultimediaDeviceInterface):
    """シンプルな音楽プレーヤー"""
    def play_audio(self, file_path):
        return f"音楽再生: {file_path}"
    
    # 以下は不要な機能だが実装を強制される（ISP違反）
    def play_video(self, file_path):
        raise NotImplementedError("動画再生機能はありません")
    
    def record_audio(self):
        raise NotImplementedError("録音機能はありません")
    
    def record_video(self):
        raise NotImplementedError("動画録画機能はありません")
    
    def take_photo(self):
        raise NotImplementedError("写真撮影機能はありません")
    
    def stream_content(self, url):
        raise NotImplementedError("ストリーミング機能はありません")
    
    def edit_content(self, content):
        raise NotImplementedError("編集機能はありません")

class SecurityCamera(MultimediaDeviceInterface):
    """防犯カメラ"""
    def record_video(self):
        return "動画録画中"
    
    def take_photo(self):
        return "写真撮影完了"
    
    # 以下は不要な機能だが実装を強制される（ISP違反）
    def play_audio(self, file_path):
        raise NotImplementedError("音楽再生機能はありません")
    
    def play_video(self, file_path):
        raise NotImplementedError("動画再生機能はありません")
    
    def record_audio(self):
        raise NotImplementedError("録音機能はありません")
    
    def stream_content(self, url):
        raise NotImplementedError("ストリーミング機能はありません")
    
    def edit_content(self, content):
        raise NotImplementedError("編集機能はありません")
```

### ISP適用の改善版

```python
# 分離された小さなインターフェース群
class AudioPlayerInterface(ABC):
    @abstractmethod
    def play_audio(self, file_path):
        pass

class VideoPlayerInterface(ABC):
    @abstractmethod
    def play_video(self, file_path):
        pass

class AudioRecorderInterface(ABC):
    @abstractmethod
    def record_audio(self):
        pass

class VideoRecorderInterface(ABC):
    @abstractmethod
    def record_video(self):
        pass

class CameraInterface(ABC):
    @abstractmethod
    def take_photo(self):
        pass

class StreamingInterface(ABC):
    @abstractmethod
    def stream_content(self, url):
        pass

class EditingInterface(ABC):
    @abstractmethod
    def edit_content(self, content):
        pass

# 必要な機能のみを実装する具体クラス
class SimpleAudioPlayer(AudioPlayerInterface):
    """音楽再生のみ"""
    def play_audio(self, file_path):
        return f"音楽再生: {file_path}"

class SecurityCamera(VideoRecorderInterface, CameraInterface):
    """動画録画と写真撮影のみ"""
    def record_video(self):
        return "セキュリティカメラで動画録画中"
    
    def take_photo(self):
        return "セキュリティカメラで写真撮影完了"

class Smartphone(AudioPlayerInterface, VideoPlayerInterface, AudioRecorderInterface, 
                VideoRecorderInterface, CameraInterface, StreamingInterface):
    """多機能スマートフォン"""
    def play_audio(self, file_path):
        return f"スマートフォンで音楽再生: {file_path}"
    
    def play_video(self, file_path):
        return f"スマートフォンで動画再生: {file_path}"
    
    def record_audio(self):
        return "スマートフォンで録音中"
    
    def record_video(self):
        return "スマートフォンで動画録画中"
    
    def take_photo(self):
        return "スマートフォンで写真撮影"
    
    def stream_content(self, url):
        return f"スマートフォンでストリーミング: {url}"

class ProfessionalCamera(CameraInterface, VideoRecorderInterface, EditingInterface):
    """プロ用カメラ"""
    def take_photo(self):
        return "プロ用カメラで高品質写真撮影"
    
    def record_video(self):
        return "プロ用カメラで4K動画録画"
    
    def edit_content(self, content):
        return f"プロ用カメラで編集: {content}"

class StreamingDevice(AudioPlayerInterface, VideoPlayerInterface, StreamingInterface):
    """ストリーミング専用デバイス"""
    def play_audio(self, file_path):
        return f"ストリーミングデバイスで音楽再生: {file_path}"
    
    def play_video(self, file_path):
        return f"ストリーミングデバイスで動画再生: {file_path}"
    
    def stream_content(self, url):
        return f"ストリーミングデバイスでコンテンツ配信: {url}"

# デバイス管理システム
class MediaDeviceManager:
    def __init__(self):
        self.devices = []
    
    def add_device(self, device):
        self.devices.append(device)
    
    def play_music_on_capable_devices(self, file_path):
        """音楽再生可能なデバイスでのみ音楽を再生"""
        results = []
        for device in self.devices:
            if isinstance(device, AudioPlayerInterface):
                results.append(device.play_audio(file_path))
        return results
    
    def take_photos_with_cameras(self):
        """写真撮影可能なデバイスでのみ写真撮影"""
        results = []
        for device in self.devices:
            if isinstance(device, CameraInterface):
                results.append(device.take_photo())
        return results
    
    def start_recording_on_capable_devices(self):
        """録画可能なデバイスでのみ録画開始"""
        results = []
        for device in self.devices:
            if isinstance(device, VideoRecorderInterface):
                results.append(device.record_video())
        return results
    
    def stream_content_on_streaming_devices(self, url):
        """ストリーミング可能なデバイスでのみ配信"""
        results = []
        for device in self.devices:
            if isinstance(device, StreamingInterface):
                results.append(device.stream_content(url))
        return results
    
    def get_device_capabilities(self):
        """各デバイスの機能一覧を取得"""
        capabilities = []
        for device in self.devices:
            device_name = device.__class__.__name__
            device_capabilities = []
            
            if isinstance(device, AudioPlayerInterface):
                device_capabilities.append("音楽再生")
            if isinstance(device, VideoPlayerInterface):
                device_capabilities.append("動画再生")
            if isinstance(device, AudioRecorderInterface):
                device_capabilities.append("録音")
            if isinstance(device, VideoRecorderInterface):
                device_capabilities.append("動画録画")
            if isinstance(device, CameraInterface):
                device_capabilities.append("写真撮影")
            if isinstance(device, StreamingInterface):
                device_capabilities.append("ストリーミング")
            if isinstance(device, EditingInterface):
                device_capabilities.append("編集")
            
            capabilities.append({
                'device': device_name,
                'capabilities': device_capabilities
            })
        
        return capabilities

# 使用例
manager = MediaDeviceManager()

# 様々なデバイスを追加
manager.add_device(SimpleAudioPlayer())
manager.add_device(SecurityCamera())
manager.add_device(Smartphone())
manager.add_device(ProfessionalCamera())
manager.add_device(StreamingDevice())

print("=== デバイス機能一覧 ===")
for device_info in manager.get_device_capabilities():
    print(f"{device_info['device']}: {', '.join(device_info['capabilities'])}")

print("\n=== 音楽再生テスト ===")
for result in manager.play_music_on_capable_devices("favorite_song.mp3"):
    print(result)

print("\n=== 写真撮影テスト ===")
for result in manager.take_photos_with_cameras():
    print(result)

print("\n=== 動画録画テスト ===")
for result in manager.start_recording_on_capable_devices():
    print(result)

print("\n=== ストリーミングテスト ===")
for result in manager.stream_content_on_streaming_devices("https://example.com/stream"):
    print(result)
```

## ISPの適用パターン

### 1. ロール分離パターン

```python
# ユーザー管理システムでの役割分離
class ReadableUserInterface(ABC):
    @abstractmethod
    def get_user_info(self, user_id):
        pass

class WritableUserInterface(ABC):
    @abstractmethod
    def create_user(self, user_data):
        pass
    
    @abstractmethod
    def update_user(self, user_id, user_data):
        pass

class DeletableUserInterface(ABC):
    @abstractmethod
    def delete_user(self, user_id):
        pass

# 読み取り専用サービス
class UserViewService(ReadableUserInterface):
    def get_user_info(self, user_id):
        return f"ユーザー{user_id}の情報を取得"

# 管理者サービス
class UserAdminService(ReadableUserInterface, WritableUserInterface, DeletableUserInterface):
    def get_user_info(self, user_id):
        return f"管理者がユーザー{user_id}の情報を取得"
    
    def create_user(self, user_data):
        return f"管理者がユーザーを作成: {user_data}"
    
    def update_user(self, user_id, user_data):
        return f"管理者がユーザー{user_id}を更新: {user_data}"
    
    def delete_user(self, user_id):
        return f"管理者がユーザー{user_id}を削除"

# 編集者サービス
class UserEditorService(ReadableUserInterface, WritableUserInterface):
    def get_user_info(self, user_id):
        return f"編集者がユーザー{user_id}の情報を取得"
    
    def create_user(self, user_data):
        return f"編集者がユーザーを作成: {user_data}"
    
    def update_user(self, user_id, user_data):
        return f"編集者がユーザー{user_id}を更新: {user_data}"
```

### 2. 機能レベル分離パターン

```python
# ファイル操作システムでの機能分離
class FileReaderInterface(ABC):
    @abstractmethod
    def read_file(self, file_path):
        pass

class FileWriterInterface(ABC):
    @abstractmethod
    def write_file(self, file_path, content):
        pass

class FileCompressorInterface(ABC):
    @abstractmethod
    def compress_file(self, file_path):
        pass

class FileEncryptorInterface(ABC):
    @abstractmethod
    def encrypt_file(self, file_path, key):
        pass

# 基本ファイルハンドラー
class BasicFileHandler(FileReaderInterface, FileWriterInterface):
    def read_file(self, file_path):
        return f"ファイル読み取り: {file_path}"
    
    def write_file(self, file_path, content):
        return f"ファイル書き込み: {file_path}"

# 高度なファイルハンドラー
class AdvancedFileHandler(FileReaderInterface, FileWriterInterface, 
                         FileCompressorInterface, FileEncryptorInterface):
    def read_file(self, file_path):
        return f"高度な読み取り: {file_path}"
    
    def write_file(self, file_path, content):
        return f"高度な書き込み: {file_path}"
    
    def compress_file(self, file_path):
        return f"ファイル圧縮: {file_path}"
    
    def encrypt_file(self, file_path, key):
        return f"ファイル暗号化: {file_path}"

# 読み取り専用ハンドラー
class ReadOnlyFileHandler(FileReaderInterface):
    def read_file(self, file_path):
        return f"読み取り専用: {file_path}"
```

## ISPの利点と効果

| 項目 | ISP違反 | ISP適用 |
|------|---------|---------|
| **実装の負担** | 不要な機能も実装が必要 | 必要な機能のみ実装 |
| **例外の発生** | 未対応機能で例外多発 | 対応機能のみ呼び出し |
| **保守性** | インターフェース変更の影響大 | 関連機能のみに影響限定 |
| **テスト性** | 全機能をテストする必要 | 実装機能のみテスト |
| **理解しやすさ** | 何ができるか不明確 | 機能が明確で分かりやすい |
| **柔軟性** | 機能追加時の影響大 | 独立して機能追加可能 |

## ISP適用のガイドライン

### 適用すべき場面
- **多様な実装**が想定される場合
- **機能の組み合わせ**が多岐にわたる場合
- **権限レベル**が異なるクライアントがある場合

### 適用時の注意点
- **細かすぎる分離は避ける**：単一メソッドのインターフェースが乱立
- **関連性を考慮**：密接に関連する機能は同じインターフェースに
- **将来の拡張性**：予想される機能追加を考慮した設計

## まとめ

ISPは**「クライアントは不要な機能に依存しない」**という原則です。

### 実践のポイント：
- **機能ごとに小さなインターフェース**を作成
- **必要な機能のみ実装**すれば良い設計
- **役割や権限レベル**に応じたインターフェース分離
- **関連性の高い機能**は適切にグループ化

ISPを適用することで、**柔軟で保守しやすく、理解しやすいシステム**を構築できます。各クラスが本当に必要な機能のみに集中でき、システム全体の複雑性を大幅に削減できるでしょう。