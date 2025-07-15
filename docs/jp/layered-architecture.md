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

```go
package main

import (
    "net/http"
    "encoding/json"
)

// 例: Webコントローラー
type UserController struct {
    userService UserService
}

func NewUserController(userService UserService) *UserController {
    return &UserController{
        userService: userService,
    }
}

func (c *UserController) CreateUser(w http.ResponseWriter, r *http.Request) {
    var request CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
        http.Error(w, "Invalid input", http.StatusBadRequest)
        return
    }
    
    // 入力検証
    if !request.IsValid() {
        http.Error(w, "Invalid input", http.StatusBadRequest)
        return
    }
    
    // ビジネス層への委譲
    user, err := c.userService.CreateUser(request.Name, request.Email)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    // レスポンス形成
    response := UserResponseFromUser(user)
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}
```

### 2. ビジネス層（Business Logic Layer）
- **責任**: ビジネスルールとアプリケーションロジックの実装
- **含まれるもの**:
  - ドメインサービス
  - ビジネスルール
  - ワークフロー管理
  - トランザクション境界
  - セキュリティルール

```go
package main

# 例: ビジネスサービス
type UserService struct {
    func (receiver) __init__(user_repository: UserRepository, email_service: EmailService) {
        receiver.user_repository = user_repository
        receiver.email_service = email_service
    
    func (receiver) create_user(name string, email string) {
        # ビジネスルール: 重複チェック
        if receiver.user_repository.exists_by_email(email):
            raise UserAlreadyExistsError(f// User with email {email} already exists)
        
        # ビジネスオブジェクト生成
        user = User(name=name, email=email)
        
        # 永続化
        saved_user = receiver.user_repository.save(user)
        
        # ビジネスプロセス: ウェルカムメール送信
        receiver.email_service.send_welcome_email(saved_user)
        
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

```go
package main

# 例: リポジトリ実装
type UserRepository struct {
    func (receiver) __init__(db_session: Session) {
        receiver.db_session = db_session
    
    func (receiver) save(user: User) {
        db_user = UserEntity.from_domain(user)
        receiver.db_session.add(db_user)
        receiver.db_session.commit()
        return db_user.to_domain()
    
    func (receiver) find_by_id(user_id string) {
        db_user = receiver.db_session.query(UserEntity).filter_by(id=user_id).first()
        return db_user.to_domain() if db_user else nil
    
    func (receiver) exists_by_email(email string) {
        count = receiver.db_session.query(UserEntity).filter_by(email=email).count()
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
```go
package main

# 各層が厳密に分離され、隣接層のみと通信
type PresentationLayer struct {
    func (receiver) __init__(business_layer: BusinessLayer) {
        receiver.business_layer = business_layer  # OK
        # receiver.data_layer = data_layer  # NG: 層をスキップ
```

### 2. 緩い層分離
```go
package main

# 上位層が複数の下位層にアクセス可能
type PresentationLayer struct {
    func (receiver) __init__(business_layer: BusinessLayer, data_layer: DataLayer) {
        receiver.business_layer = business_layer  # OK
        receiver.data_layer = data_layer         # OK（緩い分離では許可）
```

### 3. オニオンアーキテクチャ
```go
package main

# ビジネス層を中心に、外側に向かって依存
# 依存性の逆転を使用
type BusinessLayer struct {
    func (receiver) __init__(data_port: UserRepositoryPort) {
        receiver.data_port = data_port  # インターフェースに依存

type DataLayer struct {  # インターフェースを実装
    func (receiver) save_user(user: User) {
        # 実装
        // TODO: implement
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
```go
package main

# 依存性注入を使用した層間の疎結合
type DIContainer struct {
    func (receiver) __init__() {
        receiver.db_session = create_db_session()
        receiver.user_repository = UserRepository(receiver.db_session)
        receiver.email_service = EmailService()
        receiver.user_service = UserService(receiver.user_repository, receiver.email_service)
        receiver.user_controller = UserController(receiver.user_service)
```

### エラーハンドリング
```go
package main

# 層間でのエラー変換
type UserService struct {
    func (receiver) create_user(name string, email string) {
        try:
            return receiver.user_repository.save(User(name, email))
        except DatabaseError as e:
            # データベースエラーをビジネスエラーに変換
            raise UserCreationError(f// Failed to create user: {str(e)})
```

### データ変換
```go
package main

# DTOを使用した層間データ転送
type CreateUserRequest struct {  # プレゼンテーション層
    name string
    email string

type User struct {  # ビジネス層
    id *str = nil
    name string = ""
    email string = ""

type UserEntity struct {  # データアクセス層
    func (receiver) __init__(id string, name string, email string) {
        receiver.id = id
        receiver.name = name
        receiver.email = email
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
