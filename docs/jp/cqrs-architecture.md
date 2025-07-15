# CQRS Architecture（Command Query Responsibility Segregation）

## 概要
CQRS（Command Query Responsibility Segregation：コマンドクエリ責任分離）は、データストアの読み取り操作と書き込み操作を分離するアーキテクチャパターンです。このパターンにより、読み取りワークロードと書き込みワークロードを独立してスケールし、それぞれを最適化することが可能になります。

## 特徴

### コマンド（Command）
- **責任**: データの変更（作成、更新、削除）
- **特徴**: 
  - 副作用を持つ
  - 戻り値を持たない（またはステータスのみ）
  - ビジネスロジックを含む

### クエリ（Query）
- **責任**: データの読み取り
- **特徴**:
  - 副作用を持たない
  - データを返す
  - シンプルな読み取りロジック

## アーキテクチャ図

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│   Client    │───▶│   Command    │───▶│  Write DB   │
│             │    │   Handler    │    │             │
└─────────────┘    └──────────────┘    └─────────────┘
       │                                        │
       │                                        ▼
       │           ┌──────────────┐    ┌─────────────┐
       └──────────▶│    Query     │───▶│   Read DB   │
                   │   Handler    │    │             │
                   └──────────────┘    └─────────────┘
```

## メリット

### スケーラビリティ
- 読み取りと書き込みを独立してスケール可能
- 読み取り専用レプリカの活用
- 異なるデータベース技術の使用

### パフォーマンス
- 読み取り用にデータを非正規化可能
- クエリ専用のインデックス最適化
- キャッシュ戦略の独立実装

### 複雑性の管理
- 責任の明確な分離
- 読み取りモデルと書き込みモデルの独立進化
- ビジネスロジックの集約

## デメリット

### 複雑性の増加
- 2つの異なるモデルの維持
- データ同期の複雑性
- 結果整合性の管理

### 開発コスト
- 初期実装コストの増加
- 追加のインフラストラクチャ
- チーム学習コスト

## 実装パターン

### 1. シンプルなCQRS
```go
package cqrs

// Command側
type CreateUserCommand struct {
    Name  string
    Email string
}

func NewCreateUserCommand(name, email string) *CreateUserCommand {
    return &CreateUserCommand{
        Name:  name,
        Email: email,
    }
}

type UserCommandHandler struct {
    repository UserRepository
}

func (h *UserCommandHandler) Handle(command *CreateUserCommand) error {
    user := NewUser(command.Name, command.Email)
    return h.repository.Save(user)
}

// Query側
type UserQueryHandler struct {
    readRepository UserReadRepository
}

func (h *UserQueryHandler) GetUserByID(userID string) (*UserViewModel, error) {
    return h.readRepository.FindByID(userID)
}
```

### 2. Event Sourcing との組み合わせ
```go
type UserCommandHandler struct {
    eventStore EventStore
}

func (h *UserCommandHandler) Handle(command *CreateUserCommand) error {
    event := NewUserCreatedEvent(command.Name, command.Email)
    events := []Event{event}
    return h.eventStore.SaveEvents(events)
}

type UserProjection struct {
    readRepository UserReadRepository
}

func (p *UserProjection) Handle(event *UserCreatedEvent) error {
    viewModel := NewUserViewModel(event.Name, event.Email)
    return p.readRepository.Save(viewModel)
}
```

## 適用場面

### 適している場合
- **高い読み取り負荷**: 読み取りが書き込みより遥かに多い
- **複雑なビジネスロジック**: 書き込み時の複雑な検証・処理
- **異なるアクセスパターン**: 読み取りと書き込みで異なる要件
- **スケーラビリティ要件**: 独立したスケーリングが必要

### 適していない場合
- **シンプルなCRUD**: 基本的な作成・読み取り・更新・削除のみ
- **小規模システム**: 複雑性がメリットを上回る
- **強い整合性要求**: リアルタイムデータ整合性が必須

## 実装時の考慮事項

### データ整合性
- **結果整合性の受容**: 読み取りデータの遅延を許容
- **補償アクション**: 失敗時のロールバック戦略
- **冪等性**: 重複実行に対する安全性

### 同期メカニズム
- **イベント駆動**: ドメインイベントによる同期
- **メッセージング**: 非同期メッセージキュー
- **バッチ処理**: 定期的なデータ同期

### モニタリング
- **レプリケーションラグ**: 読み取り・書き込み間の遅延監視
- **イベント処理**: イベント処理の成功・失敗監視
- **パフォーマンス**: 各側面の独立監視

## 関連パターン
- **Event Sourcing**: イベントストアを使った状態管理
- **Domain-Driven Design**: ドメインモデルの明確化
- **Microservices**: サービス境界での責任分離
- **Saga Pattern**: 分散トランザクション管理

## 参考資料
- [Martin Fowler - CQRS](https://martinfowler.com/bliki/CQRS.html)
- [Microsoft - CQRS Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [Event Store - CQRS Documents](https://eventstore.com/docs/event-sourcing-basics/cqrs)
