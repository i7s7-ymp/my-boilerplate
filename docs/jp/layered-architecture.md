# Layered Architecture（レイヤードアーキテクチャ）

## 概要
レイヤードアーキテクチャ（n層アーキテクチャとも呼ばれる）は、ソフトウェアを論理的な層に分離する構造パターンです。各層は特定の責任を持ち、隣接する層とのみ通信することで、関心の分離と保守性を実現します。

## 基本構造

### 典型的な4層構造
```
┌─────────────────────────────────┐
│     Presentation Layer          │ ← UI/Controllers
├─────────────────────────────────┤
│     Business Logic Layer        │ ← Domain/Service Logic
├─────────────────────────────────┤
│     Data Access Layer           │ ← Repository/DAO
├─────────────────────────────────┤
│     Database Layer              │ ← Data Storage
└─────────────────────────────────┘
```

## 各層の責任

### 1. プレゼンテーション層（Presentation Layer）
- **責任**: ユーザーインターフェースとユーザー操作の処理
- **含まれるもの**:
  - Webコントローラー
  - APIエンドポイント
  - ビューテンプレート
  - 入力検証
  - レスポンス形成

```python
# 例: Webコントローラー
class UserController:
    def __init__(self, user_service: UserService):
        self.user_service = user_service
    
    def create_user(self, request: CreateUserRequest) -> Response:
        # 入力検証
        if not request.is_valid():
            return Response.bad_request("Invalid input")
        
        # ビジネス層への委譲
        user = self.user_service.create_user(request.name, request.email)
        
        # レスポンス形成
        return Response.ok(UserResponse.from_user(user))
```

### 2. ビジネス層（Business Logic Layer）
- **責任**: ビジネスルールとアプリケーションロジックの実装
- **含まれるもの**:
  - ドメインサービス
  - ビジネスルール
  - ワークフロー管理
  - トランザクション境界
  - セキュリティルール

```python
# 例: ビジネスサービス
class UserService:
    def __init__(self, user_repository: UserRepository, email_service: EmailService):
        self.user_repository = user_repository
        self.email_service = email_service
    
    def create_user(self, name: str, email: str) -> User:
        # ビジネスルール: 重複チェック
        if self.user_repository.exists_by_email(email):
            raise UserAlreadyExistsError(f"User with email {email} already exists")
        
        # ビジネスオブジェクト生成
        user = User(name=name, email=email)
        
        # 永続化
        saved_user = self.user_repository.save(user)
        
        # ビジネスプロセス: ウェルカムメール送信
        self.email_service.send_welcome_email(saved_user)
        
        return saved_user
```

### 3. データアクセス層（Data Access Layer）
- **責任**: データの永続化とデータソースへのアクセス
- **含まれるもの**:
  - リポジトリパターン
  - DAO（Data Access Object）
  - ORM設定
  - クエリ実装
  - データマッピング

```python
# 例: リポジトリ実装
class UserRepository:
    def __init__(self, db_session: Session):
        self.db_session = db_session
    
    def save(self, user: User) -> User:
        db_user = UserEntity.from_domain(user)
        self.db_session.add(db_user)
        self.db_session.commit()
        return db_user.to_domain()
    
    def find_by_id(self, user_id: str) -> Optional[User]:
        db_user = self.db_session.query(UserEntity).filter_by(id=user_id).first()
        return db_user.to_domain() if db_user else None
    
    def exists_by_email(self, email: str) -> bool:
        count = self.db_session.query(UserEntity).filter_by(email=email).count()
        return count > 0
```

### 4. データベース層（Database Layer）
- **責任**: データの物理的な保存と管理
- **含まれるもの**:
  - データベースサーバー
  - ファイルシステム
  - 外部データソース
  - インデックス
  - ストアドプロシージャ

## アーキテクチャルール

### 依存関係の方向
```
Presentation ──▶ Business ──▶ Data Access ──▶ Database
     ↑                                              │
     └──────────────── 依存の逆転 ───────────────────┘
```

### 基本原則
1. **上位層は下位層に依存する**: プレゼンテーション層はビジネス層に依存
2. **下位層は上位層に依存しない**: データアクセス層はビジネス層を知らない
3. **層をスキップしない**: 隣接する層とのみ通信
4. **関心の分離**: 各層は特定の責任のみを持つ

## メリット

### 保守性
- **関心の分離**: 各層が独立した責任を持つ
- **変更の局所化**: 変更の影響範囲が限定される
- **理解しやすさ**: 明確な構造で理解が容易

### テスタビリティ
- **単体テスト**: 各層を独立してテスト可能
- **モックの利用**: 依存関係を容易にモック化
- **統合テスト**: 層ごとの統合テスト

### 再利用性
- **ビジネスロジック**: 複数のUIから同じロジックを利用
- **データアクセス**: 複数のサービスから同じデータアクセス
- **コンポーネント**: 層内のコンポーネントを再利用

## デメリット

### パフォーマンス
- **オーバーヘッド**: 層間の呼び出しコスト
- **データ変換**: 層間でのオブジェクト変換
- **メモリ使用量**: 複数層でのオブジェクト保持

### 複雑性
- **過度の分離**: 小さな変更でも複数層の修正
- **ボイラープレート**: 層間の接続コード
- **設計の硬直性**: 厳格な層構造による制約

## 実装バリエーション

### 1. 厳密な層分離
```python
# 各層が厳密に分離され、隣接層のみと通信
class PresentationLayer:
    def __init__(self, business_layer: BusinessLayer):
        self.business_layer = business_layer  # OK
        # self.data_layer = data_layer  # NG: 層をスキップ
```

### 2. 緩い層分離
```python
# 上位層が複数の下位層にアクセス可能
class PresentationLayer:
    def __init__(self, business_layer: BusinessLayer, data_layer: DataLayer):
        self.business_layer = business_layer  # OK
        self.data_layer = data_layer         # OK（緩い分離では許可）
```

### 3. オニオンアーキテクチャ
```python
# ビジネス層を中心に、外側に向かって依存
# 依存性の逆転を使用
class BusinessLayer:
    def __init__(self, data_port: UserRepositoryPort):
        self.data_port = data_port  # インターフェースに依存

class DataLayer(UserRepositoryPort):  # インターフェースを実装
    def save_user(self, user: User) -> User:
        # 実装
        pass
```

## 適用場面

### 適している場合
- **中規模以上のアプリケーション**: 複雑性管理が重要
- **チーム開発**: 層ごとの担当分担
- **企業システム**: 安定性と保守性重視
- **段階的な開発**: 層ごとの独立開発

### 適していない場合
- **小規模アプリケーション**: オーバーエンジニアリング
- **高パフォーマンス要求**: 層間オーバーヘッドが問題
- **頻繁な変更**: 硬直した構造による開発効率低下

## 実装時の考慮事項

### 依存性の管理
```python
# 依存性注入を使用した層間の疎結合
class DIContainer:
    def __init__(self):
        self.db_session = create_db_session()
        self.user_repository = UserRepository(self.db_session)
        self.email_service = EmailService()
        self.user_service = UserService(self.user_repository, self.email_service)
        self.user_controller = UserController(self.user_service)
```

### エラーハンドリング
```python
# 層間でのエラー変換
class UserService:
    def create_user(self, name: str, email: str) -> User:
        try:
            return self.user_repository.save(User(name, email))
        except DatabaseError as e:
            # データベースエラーをビジネスエラーに変換
            raise UserCreationError(f"Failed to create user: {str(e)}")
```

### データ変換
```python
# DTOを使用した層間データ転送
@dataclass
class CreateUserRequest:  # プレゼンテーション層
    name: str
    email: str

@dataclass 
class User:  # ビジネス層
    id: Optional[str] = None
    name: str = ""
    email: str = ""

class UserEntity:  # データアクセス層
    def __init__(self, id: str, name: str, email: str):
        self.id = id
        self.name = name
        self.email = email
```

## 関連パターン
- **MVC (Model-View-Controller)**: プレゼンテーション層の構造化
- **Repository Pattern**: データアクセス層の抽象化
- **Service Layer**: ビジネス層の実装パターン
- **Dependency Injection**: 層間の依存関係管理
- **Clean Architecture**: 依存性の逆転を適用した層構造

## 実装例リンク
- [Go実装例](https://github.com/i7s7-ymp/go-layered)
- [Python実装例](https://github.com/i7s7-ymp/python-layered)

## 参考資料
- [Martin Fowler - PresentationDomainDataLayering](https://martinfowler.com/bliki/PresentationDomainDataLayering.html)
- [Microsoft - N-tier architecture style](https://docs.microsoft.com/en-us/azure/architecture/guide/architecture-styles/n-tier)
- [Patterns of Enterprise Application Architecture](https://martinfowler.com/books/eaa.html)
