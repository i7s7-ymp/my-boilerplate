# Microservices Architecture（マイクロサービスアーキテクチャ）

## 概要
マイクロサービスアーキテクチャは、アプリケーションを小さく独立したサービスの集合として設計するアーキテクチャスタイルです。各サービスは特定のビジネス機能を担当し、軽量な通信メカニズム（通常はHTTP API）を通じて通信します。

## 基本原則

### 1. ビジネス機能による分離
```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│    User     │  │   Order     │  │  Payment    │
│  Service    │  │  Service    │  │  Service    │
│             │  │             │  │             │
│ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │
│ │User DB  │ │  │ │Order DB │ │  │ │Pay DB   │ │
│ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │
└─────────────┘  └─────────────┘  └─────────────┘
       │                 │                 │
       └─────────────────┼─────────────────┘
                        │
              ┌─────────────┐
              │   API       │
              │  Gateway    │
              └─────────────┘
```

### 2. 独立したデプロイメント
- 各サービスは独立してビルド・デプロイ・スケール可能
- 異なる技術スタックの使用が可能
- 障害の局所化

### 3. 分散データ管理
- 各サービスが専用のデータベースを所有
- データの整合性は結果整合性で管理
- サービス間でのデータベース共有を避ける

## アーキテクチャコンポーネント

### 1. マイクロサービス
```python
# 例: ユーザーサービス
from flask import Flask, jsonify, request
from dataclasses import dataclass
from typing import Optional

@dataclass
class User:
    id: str
    name: str
    email: str

class UserService:
    def __init__(self):
        self.users = {}  # 実際にはデータベース
        
    def create_user(self, name: str, email: str) -> User:
        user_id = self.generate_id()
        user = User(id=user_id, name=name, email=email)
        self.users[user_id] = user
        
        # イベント発行
        self.publish_event("user.created", {
            "user_id": user_id,
            "name": name,
            "email": email
        })
        
        return user
        
    def get_user(self, user_id: str) -> Optional[User]:
        return self.users.get(user_id)

app = Flask(__name__)
user_service = UserService()

@app.route('/users', methods=['POST'])
def create_user():
    data = request.json
    user = user_service.create_user(data['name'], data['email'])
    return jsonify({
        'id': user.id,
        'name': user.name, 
        'email': user.email
    })

@app.route('/users/<user_id>')
def get_user(user_id):
    user = user_service.get_user(user_id)
    if user:
        return jsonify({
            'id': user.id,
            'name': user.name,
            'email': user.email
        })
    return jsonify({'error': 'User not found'}), 404
```

### 2. API Gateway
```python
# API Gatewayパターンの実装例
class APIGateway:
    def __init__(self):
        self.services = {
            'users': 'http://user-service:8001',
            'orders': 'http://order-service:8002',
            'payments': 'http://payment-service:8003'
        }
        self.auth_service = AuthService()
        
    def route_request(self, request):
        # 認証チェック
        if not self.auth_service.validate_token(request.headers.get('Authorization')):
            return {'error': 'Unauthorized'}, 401
            
        # ルーティング
        service_name = self.extract_service_name(request.path)
        service_url = self.services.get(service_name)
        
        if not service_url:
            return {'error': 'Service not found'}, 404
            
        # リクエスト転送
        return self.forward_request(service_url, request)
        
    def forward_request(self, service_url, request):
        # サービスディスカバリとロードバランシング
        instance = self.service_discovery.get_healthy_instance(service_url)
        
        # サーキットブレーカー
        if self.circuit_breaker.is_open(service_url):
            return {'error': 'Service unavailable'}, 503
            
        # リクエスト実行
        response = requests.request(
            method=request.method,
            url=f"{instance}/{request.path}",
            headers=request.headers,
            data=request.data,
            timeout=5
        )
        
        return response.json(), response.status_code
```

### 3. サービス間通信

#### 同期通信（REST API）
```python
class OrderService:
    def __init__(self):
        self.user_service_client = UserServiceClient()
        self.payment_service_client = PaymentServiceClient()
        
    def create_order(self, user_id: str, items: list, amount: float):
        # ユーザー情報取得
        user = self.user_service_client.get_user(user_id)
        if not user:
            raise UserNotFoundError()
            
        # 注文作成
        order = Order(user_id=user_id, items=items, amount=amount)
        self.orders[order.id] = order
        
        # 決済処理
        payment_result = self.payment_service_client.process_payment(
            user_id=user_id,
            amount=amount,
            order_id=order.id
        )
        
        if payment_result.success:
            order.status = OrderStatus.CONFIRMED
        else:
            order.status = OrderStatus.FAILED
            
        return order

class UserServiceClient:
    def __init__(self, base_url: str = "http://user-service:8001"):
        self.base_url = base_url
        
    def get_user(self, user_id: str) -> Optional[dict]:
        try:
            response = requests.get(f"{self.base_url}/users/{user_id}")
            if response.status_code == 200:
                return response.json()
            return None
        except requests.RequestException:
            # フォールバック処理
            return self.get_user_from_cache(user_id)
```

#### 非同期通信（メッセージング）
```python
# イベント駆動通信
class EventPublisher:
    def __init__(self, message_broker):
        self.message_broker = message_broker
        
    def publish_event(self, event_type: str, data: dict):
        event = {
            'type': event_type,
            'data': data,
            'timestamp': datetime.utcnow().isoformat(),
            'source': self.get_service_name()
        }
        
        self.message_broker.publish(event_type, event)

class EventSubscriber:
    def __init__(self, message_broker):
        self.message_broker = message_broker
        self.handlers = {}
        
    def subscribe(self, event_type: str, handler):
        self.handlers[event_type] = handler
        self.message_broker.subscribe(event_type, self.handle_event)
        
    def handle_event(self, event):
        event_type = event['type']
        if event_type in self.handlers:
            self.handlers[event_type](event['data'])

# 注文サービスでのイベント処理
class OrderEventHandler:
    def __init__(self):
        self.event_subscriber = EventSubscriber(message_broker)
        self.event_subscriber.subscribe('user.created', self.handle_user_created)
        self.event_subscriber.subscribe('payment.completed', self.handle_payment_completed)
        
    def handle_user_created(self, user_data):
        # ユーザー作成時の処理（ウェルカムクーポン発行など）
        pass
        
    def handle_payment_completed(self, payment_data):
        # 決済完了時の処理（注文確定など）
        order_id = payment_data['order_id']
        self.order_service.confirm_order(order_id)
```

## 設計パターン

### 1. Database per Service
```python
# 各サービスが専用のデータベースを持つ
class UserService:
    def __init__(self):
        self.db = PostgreSQLConnection("user_db")
        
class OrderService:
    def __init__(self):
        self.db = MongoDBConnection("order_db")
        
class PaymentService:
    def __init__(self):
        self.db = MySQLConnection("payment_db")
```

### 2. Saga Pattern（分散トランザクション）
```python
class OrderSaga:
    def __init__(self):
        self.steps = [
            CreateOrderStep(),
            ReserveInventoryStep(),
            ProcessPaymentStep(),
            ConfirmOrderStep()
        ]
        
    def execute(self, order_data):
        executed_steps = []
        
        try:
            for step in self.steps:
                result = step.execute(order_data)
                executed_steps.append(step)
                
                if not result.success:
                    raise SagaExecutionError(result.error)
                    
        except SagaExecutionError:
            # 補償処理（逆順で実行）
            for step in reversed(executed_steps):
                step.compensate(order_data)
            raise

class CreateOrderStep:
    def execute(self, order_data):
        # 注文作成処理
        return StepResult(success=True)
        
    def compensate(self, order_data):
        # 注文削除処理
        pass
```

### 3. Circuit Breaker Pattern
```python
import time
from enum import Enum

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open" 
    HALF_OPEN = "half_open"

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED
        
    def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.timeout:
                self.state = CircuitState.HALF_OPEN
            else:
                raise CircuitBreakerOpenError()
                
        try:
            result = func(*args, **kwargs)
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise e
            
    def on_success(self):
        self.failure_count = 0
        self.state = CircuitState.CLOSED
        
    def on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN

# 使用例
payment_circuit_breaker = CircuitBreaker()

def process_payment_with_circuit_breaker(amount):
    return payment_circuit_breaker.call(payment_service.process_payment, amount)
```

## メリット

### 開発・デプロイの独立性
- **チーム自律性**: 各チームが独立して開発・デプロイ
- **技術選択の自由**: サービスごとに最適な技術スタック選択
- **個別スケーリング**: 需要に応じたサービス単位のスケーリング

### 障害の局所化
- **障害隔離**: 一つのサービス障害が全体に波及しない
- **部分的可用性**: 一部サービス停止でも他サービスは稼働継続
- **段階的復旧**: サービス単位での障害復旧

### スケーラビリティ
- **水平スケーリング**: 負荷の高いサービスのみをスケール
- **リソース最適化**: サービスごとの最適なリソース配分
- **パフォーマンス最適化**: サービス特性に応じた最適化

## デメリット

### 分散システムの複雑性
- **ネットワーク通信**: レイテンシと信頼性の問題
- **データ整合性**: 分散データの整合性管理
- **分散トランザクション**: 複雑なトランザクション管理

### 運用の複雑性
- **デプロイメント**: 多数のサービスのデプロイ管理
- **モニタリング**: 分散ログとメトリクスの統合
- **デバッグ**: 分散トレーシングとエラー追跡

### 開発オーバーヘッド
- **サービス間通信**: API設計と維持のコスト
- **テスト複雑性**: 統合テストとエンドツーエンドテスト
- **データの重複**: サービス間でのデータ複製

## 実装時の考慮事項

### サービス境界の設計
```python
# ドメイン駆動設計による境界設定
class UserBoundedContext:
    """ユーザー管理のコンテキスト"""
    def __init__(self):
        self.user_service = UserService()
        self.profile_service = ProfileService()
        
class OrderBoundedContext:
    """注文管理のコンテキスト"""
    def __init__(self):
        self.order_service = OrderService()
        self.inventory_service = InventoryService()
        
class PaymentBoundedContext:
    """決済管理のコンテキスト"""
    def __init__(self):
        self.payment_service = PaymentService()
        self.billing_service = BillingService()
```

### データ整合性の管理
```python
# イベントソーシングによる整合性管理
class EventStore:
    def __init__(self):
        self.events = []
        
    def append_event(self, event):
        self.events.append(event)
        self.publish_event(event)
        
    def get_events(self, aggregate_id):
        return [e for e in self.events if e.aggregate_id == aggregate_id]

class OrderAggregate:
    def __init__(self, order_id):
        self.order_id = order_id
        self.events = []
        
    def create_order(self, user_id, items):
        event = OrderCreatedEvent(self.order_id, user_id, items)
        self.apply_event(event)
        
    def apply_event(self, event):
        self.events.append(event)
        # 状態更新ロジック
```

### モニタリングと観測可能性
```python
# 分散トレーシング
import opentracing

class TracedService:
    def __init__(self, service_name):
        self.tracer = opentracing.tracer
        self.service_name = service_name
        
    def create_order(self, order_data):
        with self.tracer.start_span('create_order') as span:
            span.set_tag('service', self.service_name)
            span.set_tag('user_id', order_data['user_id'])
            
            # ユーザー情報取得
            with self.tracer.start_span('get_user', child_of=span) as user_span:
                user = self.get_user(order_data['user_id'])
                
            # 決済処理
            with self.tracer.start_span('process_payment', child_of=span) as pay_span:
                payment = self.process_payment(order_data)
                
            return self.save_order(order_data)

# ヘルスチェック
class HealthCheck:
    def __init__(self):
        self.dependencies = [
            DatabaseHealthCheck(),
            ExternalServiceHealthCheck(),
            MessageQueueHealthCheck()
        ]
        
    def check_health(self):
        results = {}
        overall_status = "healthy"
        
        for check in self.dependencies:
            try:
                status = check.check()
                results[check.name] = status
                if status != "healthy":
                    overall_status = "unhealthy"
            except Exception as e:
                results[check.name] = f"error: {str(e)}"
                overall_status = "unhealthy"
                
        return {
            "status": overall_status,
            "checks": results
        }
```

## 適用場面

### 適している場合
- **大規模システム**: 複数チームでの並行開発
- **高い変更頻度**: 頻繁な機能追加・変更
- **異なるスケーリング要件**: サービスごとの負荷特性が異なる
- **技術多様性**: 異なる技術スタックの活用が必要

### 適していない場合
- **小規模システム**: オーバーエンジニアリングになる可能性
- **強い整合性要求**: ACIDトランザクションが必須
- **限定されたチーム**: 分散システム管理のスキルが不足
- **ネットワーク制約**: 低レイテンシ・高スループットが必須

## 移行戦略

### Strangler Fig Pattern
```python
# 段階的な移行パターン
class LegacySystemProxy:
    def __init__(self):
        self.legacy_system = LegacySystem()
        self.new_user_service = NewUserService()
        self.migration_config = MigrationConfig()
        
    def get_user(self, user_id):
        if self.migration_config.is_migrated("user", user_id):
            return self.new_user_service.get_user(user_id)
        else:
            return self.legacy_system.get_user(user_id)
            
    def create_user(self, user_data):
        # 新システムで作成
        user = self.new_user_service.create_user(user_data)
        
        # マイグレーション状態を記録
        self.migration_config.mark_migrated("user", user.id)
        
        return user
```

## ツールとテクノロジー

### コンテナとオーケストレーション
- **Docker**: サービスのコンテナ化
- **Kubernetes**: コンテナオーケストレーション
- **Istio**: サービスメッシュ

### メッセージング
- **Apache Kafka**: イベントストリーミング
- **RabbitMQ**: メッセージキュー
- **AWS SQS/SNS**: クラウドメッセージング

### API Gateway
- **Kong**: オープンソースAPI Gateway
- **AWS API Gateway**: クラウドAPI Gateway
- **Zuul**: Netflix OSS API Gateway

## 関連パターン
- **Domain-Driven Design**: サービス境界の設計
- **Event-Driven Architecture**: サービス間の非同期通信
- **CQRS**: 読み取り・書き込みの分離
- **Saga Pattern**: 分散トランザクション管理
- **API Gateway Pattern**: 統一されたエントリポイント

## 参考資料
- [Microservices.io](https://microservices.io/)
- [Building Microservices - Sam Newman](https://www.oreilly.com/library/view/building-microservices/9781491950340/)
- [Microservices Patterns - Chris Richardson](https://www.manning.com/books/microservices-patterns)
- [Martin Fowler - Microservices](https://martinfowler.com/articles/microservices.html)
