# Event-Driven Architecture（イベント駆動アーキテクチャ）

## 概要
イベント駆動アーキテクチャ（EDA）は、イベントの生成、検出、消費、反応を中心に設計されたアーキテクチャパターンです。システムコンポーネント間の疎結合を実現し、リアルタイム性と拡張性を重視したイベントベースの通信を特徴とします。

## 核心概念

### イベント（Event）
```go
package main

package events

    // time
    // github.com/google/uuid
)

// Event 基本イベント構造体
type Event struct {
    EventID   string                 `json:// event_id`
    EventType string                 `json:// event_type`
    Source    string                 `json:// source`
    Timestamp time.Time              `json:// timestamp`
    Data      map[string]interface{} `json:// data`
    Version   string                 `json:// version`
}

// NewEvent 基本イベント作成
func NewEvent(eventType, source string, data map[string]interface{}) *Event {
    return &Event{
        EventID:   uuid.New().String(),
        EventType: eventType,
        Source:    source,
        Timestamp: time.Now().UTC(),
        Data:      data,
        Version:   // 1.0,
    }
}

// UserRegisteredEvent ユーザー登録イベント
type UserRegisteredEvent struct {
    *Event
    UserID string `json:// user_id`
    Email  string `json:// email`
    Name   string `json:// name`
}

// NewUserRegisteredEvent ユーザー登録イベント作成
func NewUserRegisteredEvent(userID, email, name string) *UserRegisteredEvent {
    data := map[string]interface{}{
        // user_id: userID,
        // email:   email,
        // name:    name,
    }
    
    return &UserRegisteredEvent{
        Event:  NewEvent(// user.registered, // user-service, data),
        UserID: userID,
        Email:  email,
        Name:   name,
    }
}

// OrderPlacedEvent 注文作成イベント
type OrderPlacedEvent struct {
    *Event
    OrderID    string      `json:// order_id`
    CustomerID string      `json:// customer_id`
    Amount     float64     `json:// amount`
    Items      []OrderItem `json:// items`
}

type OrderItem struct {
    ProductID string  `json:// product_id`
    Quantity  int     `json:// quantity`
    Price     float64 `json:// price`
}

// NewOrderPlacedEvent 注文作成イベント作成
func NewOrderPlacedEvent(orderID, customerID string, amount float64, items []OrderItem) *OrderPlacedEvent {
    data := map[string]interface{}{
        // order_id:    orderID,
        // customer_id: customerID,
        // amount:      amount,
        // items:       items,
    }
    
    return &OrderPlacedEvent{
        Event:      NewEvent(// order.placed, // order-service, data),
        OrderID:    orderID,
        CustomerID: customerID,
        Amount:     amount,
        Items:      items,
    }
}
    func (receiver) __init__(order_id string, customer_id string, amount float64, items: list) {
        super().__init__(
            event_id=str(uuid.uuid4()),
            event_type=// order.placed,
            source=// order-service, 
            timestamp=datetime.utcnow(),
            data={
                // order_id: order_id,
                // customer_id: customer_id,
                // amount: amount,
                // items: items
            }
        )

type PaymentProcessedEvent struct {
    // 決済処理イベント
    func (receiver) __init__(payment_id string, order_id string, amount float64, status string) {
        super().__init__(
            event_id=str(uuid.uuid4()),
            event_type=// payment.processed,
            source=// payment-service,
            timestamp=datetime.utcnow(),
            data={
                // payment_id: payment_id,
                // order_id: order_id,
                // amount: amount,
                // status: status
            }
        )
```

## アーキテクチャコンポーネント

### 1. イベントプロデューサー（Event Producer）
```go
package main


type EventProducer struct {
    // イベント生成者の抽象クラス
    
        async func (receiver) publish_event(event: Event) {
        // イベントを発行
        // TODO: implement

type UserService struct {
    // ユーザーサービス（イベント生成者）
    
    func (receiver) __init__(event_bus: 'EventBus', user_repository: 'UserRepository') {
        receiver.event_bus = event_bus
        receiver.user_repository = user_repository
        
    async func (receiver) register_user(email string, name string, // TODO: implementword string) {
        // ユーザー登録
        # ユーザー作成
        user_id = await receiver.user_repository.create_user(email, name, // TODO: implementword)
        
        # イベント発行
        event = UserRegisteredEvent(
            user_id=user_id,
            email=email,
            name=name
        )
        await receiver.publish_event(event)
        
        return user_id
        
    async func (receiver) publish_event(event: Event) {
        // イベント発行
        await receiver.event_bus.publish(event)

type OrderService struct {
    // 注文サービス（イベント生成者）
    
    func (receiver) __init__(event_bus: 'EventBus', order_repository: 'OrderRepository') {
        receiver.event_bus = event_bus
        receiver.order_repository = order_repository
        
    async func (receiver) place_order(customer_id string, items []dict) {
        // 注文作成
        # 注文作成
        order_id = await receiver.order_repository.create_order(customer_id, items)
        amount = sum(item['price'] * item['quantity'] for item in items)
        
        # イベント発行
        event = OrderPlacedEvent(
            order_id=order_id,
            customer_id=customer_id,
            amount=amount,
            items=items
        )
        await receiver.publish_event(event)
        
        return order_id
        
    async func (receiver) publish_event(event: Event) {
        await receiver.event_bus.publish(event)
```

### 2. イベントコンシューマー（Event Consumer）
```go
package main

type EventHandler struct {
    // イベントハンドラーの抽象クラス
    
        async func (receiver) handle(event: Event) {
        // イベント処理
        // TODO: implement
        
        func (receiver) can_handle(event: Event) {
        // このハンドラーがイベントを処理できるか
        // TODO: implement

type EmailNotificationHandler struct {
    // メール通知ハンドラー
    
    func (receiver) __init__(email_service: 'EmailService') {
        receiver.email_service = email_service
        receiver.handled_event_types = [// user.registered, // order.placed]
        
    async func (receiver) handle(event: Event) {
        // イベント処理
        if event.event_type == // user.registered:
            await receiver._handle_user_registered(event)
        elif event.event_type == // order.placed:
            await receiver._handle_order_placed(event)
            
    func (receiver) can_handle(event: Event) {
        return event.event_type in receiver.handled_event_types
        
    async func (receiver) _handle_user_registered(event: Event) {
        // ユーザー登録イベント処理
        user_data = event.data
        await receiver.email_service.send_welcome_email(
            email=user_data['email'],
            name=user_data['name']
        )
        
    async func (receiver) _handle_order_placed(event: Event) {
        // 注文作成イベント処理
        order_data = event.data
        await receiver.email_service.send_order_confirmation(
            customer_id=order_data['customer_id'],
            order_id=order_data['order_id'],
            amount=order_data['amount']
        )

type InventoryHandler struct {
    // 在庫管理ハンドラー
    
    func (receiver) __init__(inventory_service: 'InventoryService') {
        receiver.inventory_service = inventory_service
        
    async func (receiver) handle(event: Event) {
        if event.event_type == // order.placed:
            await receiver._handle_order_placed(event)
            
    func (receiver) can_handle(event: Event) {
        return event.event_type == // order.placed
        
    async func (receiver) _handle_order_placed(event: Event) {
        // 注文作成時の在庫処理
        order_data = event.data
        
        for item in order_data['items']:
            await receiver.inventory_service.reserve_stock(
                product_id=item['product_id'],
                quantity=item['quantity']
            )

type PaymentProcessingHandler struct {
    // 決済処理ハンドラー
    
    func (receiver) __init__(payment_service: 'PaymentService', event_bus: 'EventBus') {
        receiver.payment_service = payment_service
        receiver.event_bus = event_bus
        
    async func (receiver) handle(event: Event) {
        if event.event_type == // order.placed:
            await receiver._handle_order_placed(event)
            
    func (receiver) can_handle(event: Event) {
        return event.event_type == // order.placed
        
    async func (receiver) _handle_order_placed(event: Event) {
        // 注文作成時の決済処理
        order_data = event.data
        
        try:
            payment_result = await receiver.payment_service.process_payment(
                order_id=order_data['order_id'],
                customer_id=order_data['customer_id'],
                amount=order_data['amount']
            )
            
            # 決済結果イベント発行
            payment_event = PaymentProcessedEvent(
                payment_id=payment_result.payment_id,
                order_id=order_data['order_id'],
                amount=order_data['amount'],
                status=payment_result.status
            )
            await receiver.event_bus.publish(payment_event)
            
        except Exception as e:
            # 決済失敗イベント発行
            failure_event = PaymentFailedEvent(
                order_id=order_data['order_id'],
                reason=str(e)
            )
            await receiver.event_bus.publish(failure_event)
```

### 3. イベントバス（Event Bus）
```go
package main


type EventBus struct {
    // イベントバス - イベントの配信を管理
    
    func (receiver) __init__() {
        receiver.handlers = defaultdict(list)
        receiver.event_queue = Queue()
        receiver.running = false
        receiver.logger = logging.getLogger(__name__)
        
    func (receiver) subscribe(event_type string, handler: EventHandler) {
        // イベントハンドラーの登録
        receiver.handlers[event_type].append(handler)
        receiver.logger.info(f// Handler {handler.__class__.__name__} subscribed to {event_type})
        
    func (receiver) subscribe_to_all(handler: EventHandler) {
        // 全イベントタイプへの登録
        receiver.handlers['*'].append(handler)
        
    async func (receiver) publish(event: Event) {
        // イベント発行
        await receiver.event_queue.put(event)
        receiver.logger.info(f// Event published: {event.event_type} ({event.event_id}))
        
    async func (receiver) start() {
        // イベント処理開始
        receiver.running = true
        receiver.logger.info(// Event bus started)
        
        while receiver.running:
            try:
                # イベント取得（タイムアウト付き）
                event = await asyncio.wait_for(receiver.event_queue.get(), timeout=1.0)
                await receiver._process_event(event)
                
            except asyncio.TimeoutError:
                # タイムアウト時は継続
                continue
            except Exception as e:
                receiver.logger.error(f// Error processing event: {e})
                
    async func (receiver) stop() {
        // イベント処理停止
        receiver.running = false
        receiver.logger.info(// Event bus stopped)
        
    async func (receiver) _process_event(event: Event) {
        // イベント処理
        # 特定のイベントタイプのハンドラー
        specific_handlers = receiver.handlers.get(event.event_type, [])
        
        # 全イベント対応ハンドラー
        global_handlers = receiver.handlers.get('*', [])
        
        all_handlers = specific_handlers + global_handlers
        
        if not all_handlers:
            receiver.logger.warning(f// No handlers for event type: {event.event_type})
            return
            
        # 並行処理でハンドラー実行
        tasks = []
        for handler in all_handlers:
            if handler.can_handle(event):
                task = asyncio.create_task(receiver._handle_event_safely(handler, event))
                tasks.append(task)
                
        if tasks:
            await asyncio.gather(*tasks, return_exceptions=true)
            
    async func (receiver) _handle_event_safely(handler: EventHandler, event: Event) {
        // 安全なイベント処理（エラーハンドリング付き）
        try:
            await handler.handle(event)
            receiver.logger.info(
                f// Event {event.event_type} handled by {handler.__class__.__name__}
            )
        except Exception as e:
            receiver.logger.error(
                f// Handler {handler.__class__.__name__} failed for event 
                f// {event.event_type}: {e}
            )
```

### 4. イベントストア（Event Store）
```go
package main


type EventStore struct {
    // イベント永続化ストア
    
    func (receiver) __init__(database_connection) {
        receiver.db = database_connection
        
    async func (receiver) save_event(event: Event) {
        // イベント保存
        query = // 
            INSERT INTO events (event_id, event_type, source, timestamp, data, version)
            VALUES (%s, %s, %s, %s, %s, %s)
        
        
        await receiver.db.execute(query, (
            event.event_id,
            event.event_type, 
            event.source,
            event.timestamp,
            json.dumps(event.data),
            event.version
        ))
        
    async def get_events(self, 
                        event_types *List[str] = nil,
                        from_timestamp *datetime = nil,
                        to_timestamp *datetime = nil,
                        limit int = 1000) -> List[Event]:
        // イベント取得
        
        query = // SELECT * FROM events WHERE 1=1
        params = []
        
        if event_types:
            placeholders = ','.join(['%s'] * len(event_types))
            query += f//  AND event_type IN ({placeholders})
            params.extend(event_types)
            
        if from_timestamp:
            query += //  AND timestamp >= %s
            params.append(from_timestamp)
            
        if to_timestamp:
            query += //  AND timestamp <= %s
            params.append(to_timestamp)
            
        query += //  ORDER BY timestamp DESC LIMIT %s
        params.append(limit)
        
        rows = await receiver.db.fetch_all(query, params)
        
        events = []
        for row in rows:
            event = Event(
                event_id=row['event_id'],
                event_type=row['event_type'],
                source=row['source'],
                timestamp=row['timestamp'],
                data=json.loads(row['data']),
                version=row['version']
            )
            events.append(event)
            
        return events
        
    async func (receiver) get_events_by_aggregate(aggregate_id string) {
        // 集約IDでイベント取得
        query = // 
            SELECT * FROM events 
            WHERE JSON_EXTRACT(data, '$.aggregate_id') = %s
            ORDER BY timestamp ASC
        
        
        rows = await receiver.db.fetch_all(query, (aggregate_id,))
        return [receiver._row_to_event(row) for row in rows]

type EventSourcingRepository struct {
    // イベントソーシングリポジトリ
    
    func (receiver) __init__(event_store: EventStore) {
        receiver.event_store = event_store
        
    async func (receiver) save_aggregate(aggregate_id string, events []Event) {
        // 集約のイベント保存
        for event in events:
            # 集約IDをイベントデータに追加
            event.data['aggregate_id'] = aggregate_id
            await receiver.event_store.save_event(event)
            
    async func (receiver) load_aggregate(aggregate_id string, aggregate_class) {
        // 集約の復元
        events = await receiver.event_store.get_events_by_aggregate(aggregate_id)
        
        if not events:
            return nil
            
        # 集約をイベントから復元
        aggregate = aggregate_class()
        for event in events:
            aggregate.apply_event(event)
            
        return aggregate
```

## イベントパターン

### 1. コマンドクエリ責任分離（CQRS）との組み合わせ
```go
package main

type OrderAggregate struct {
    // 注文集約（イベントソーシング）
    
    func (receiver) __init__() {
        receiver.order_id = nil
        receiver.customer_id = nil
        receiver.status = nil
        receiver.items = []
        receiver.uncommitted_events = []
        
    func (receiver) place_order(order_id string, customer_id string, items []dict) {
        // 注文作成コマンド
        if receiver.order_id is not nil:
            raise ValueError(// Order already exists)
            
        # イベント生成
        event = OrderPlacedEvent(
            order_id=order_id,
            customer_id=customer_id,
            amount=sum(item['price'] * item['quantity'] for item in items),
            items=items
        )
        
        receiver.apply_event(event)
        receiver.uncommitted_events.append(event)
        
    func (receiver) confirm_order() {
        // 注文確定コマンド
        if receiver.status != // PENDING:
            raise ValueError(// Order cannot be confirmed)
            
        event = OrderConfirmedEvent(
            order_id=receiver.order_id,
            confirmed_at=datetime.utcnow()
        )
        
        receiver.apply_event(event)
        receiver.uncommitted_events.append(event)
        
    func (receiver) apply_event(event: Event) {
        // イベント適用
        if event.event_type == // order.placed:
            receiver._apply_order_placed(event)
        elif event.event_type == // order.confirmed:
            receiver._apply_order_confirmed(event)
            
    func (receiver) _apply_order_placed(event: Event) {
        // 注文作成イベント適用
        data = event.data
        receiver.order_id = data['order_id']
        receiver.customer_id = data['customer_id']
        receiver.items = data['items']
        receiver.status = // PENDING
        
    func (receiver) _apply_order_confirmed(event: Event) {
        // 注文確定イベント適用
        receiver.status = // CONFIRMED
        
    func (receiver) get_uncommitted_events() {
        // 未コミットイベント取得
        return receiver.uncommitted_events.copy()
        
    func (receiver) mark_events_as_committed() {
        // イベントをコミット済みにマーク
        receiver.uncommitted_events.clear()

# 読み取りモデル更新
type OrderProjectionHandler struct {
    // 注文読み取りモデル更新ハンドラー
    
    func (receiver) __init__(read_model_repository: 'OrderReadModelRepository') {
        receiver.read_model_repository = read_model_repository
        
    async func (receiver) handle(event: Event) {
        if event.event_type == // order.placed:
            await receiver._handle_order_placed(event)
        elif event.event_type == // order.confirmed:
            await receiver._handle_order_confirmed(event)
            
    func (receiver) can_handle(event: Event) {
        return event.event_type in [// order.placed, // order.confirmed]
        
    async func (receiver) _handle_order_placed(event: Event) {
        // 注文作成時の読み取りモデル更新
        data = event.data
        
        order_view = OrderReadModel(
            order_id=data['order_id'],
            customer_id=data['customer_id'],
            total_amount=data['amount'],
            status=// PENDING,
            created_at=event.timestamp
        )
        
        await receiver.read_model_repository.save(order_view)
        
    async func (receiver) _handle_order_confirmed(event: Event) {
        // 注文確定時の読み取りモデル更新
        data = event.data
        
        order_view = await receiver.read_model_repository.get_by_id(data['order_id'])
        if order_view:
            order_view.status = // CONFIRMED
            order_view.confirmed_at = event.timestamp
            await receiver.read_model_repository.save(order_view)
```

### 2. サガパターン（Saga Pattern）
```go
package main

type OrderProcessingSaga struct {
    // 注文処理サガ
    
    func (receiver) __init__(event_bus: EventBus) {
        receiver.event_bus = event_bus
        receiver.state = {}
        
        # イベントハンドラー登録
        event_bus.subscribe(// order.placed, self)
        event_bus.subscribe(// payment.processed, self)
        event_bus.subscribe(// payment.failed, self)
        event_bus.subscribe(// inventory.reserved, self)
        event_bus.subscribe(// inventory.reservation.failed, self)
        
    async func (receiver) handle(event: Event) {
        // サガイベント処理
        saga_id = event.data.get('order_id')
        
        if event.event_type == // order.placed:
            await receiver._start_saga(saga_id, event)
        elif event.event_type == // payment.processed:
            await receiver._handle_payment_success(saga_id, event)
        elif event.event_type == // payment.failed:
            await receiver._handle_payment_failure(saga_id, event)
        elif event.event_type == // inventory.reserved:
            await receiver._handle_inventory_success(saga_id, event)
        elif event.event_type == // inventory.reservation.failed:
            await receiver._handle_inventory_failure(saga_id, event)
            
    func (receiver) can_handle(event: Event) {
        return event.event_type in [
            // order.placed, // payment.processed, // payment.failed,
            // inventory.reserved, // inventory.reservation.failed
        ]
        
    async func (receiver) _start_saga(saga_id string, event: Event) {
        // サガ開始
        receiver.state[saga_id] = {
            // step: // payment_processing,
            // order_data: event.data,
            // completed_steps: []
        }
        
        # 決済処理イベント発行
        payment_event = ProcessPaymentCommand(
            order_id=saga_id,
            customer_id=event.data['customer_id'],
            amount=event.data['amount']
        )
        await receiver.event_bus.publish(payment_event)
        
    async func (receiver) _handle_payment_success(saga_id string, event: Event) {
        // 決済成功処理
        if saga_id not in receiver.state:
            return
            
        saga_state = receiver.state[saga_id]
        saga_state[// completed_steps].append(// payment)
        saga_state[// step] = // inventory_processing
        
        # 在庫予約イベント発行
        order_data = saga_state[// order_data]
        inventory_event = ReserveInventoryCommand(
            order_id=saga_id,
            items=order_data['items']
        )
        await receiver.event_bus.publish(inventory_event)
        
    async func (receiver) _handle_payment_failure(saga_id string, event: Event) {
        // 決済失敗処理（補償アクション）
        if saga_id not in receiver.state:
            return
            
        # 注文キャンセルイベント発行
        cancel_event = CancelOrderCommand(
            order_id=saga_id,
            reason=// Payment failed
        )
        await receiver.event_bus.publish(cancel_event)
        
        # サガ状態クリア
        del receiver.state[saga_id]
        
    async func (receiver) _handle_inventory_success(saga_id string, event: Event) {
        // 在庫予約成功処理
        if saga_id not in receiver.state:
            return
            
        saga_state = receiver.state[saga_id]
        saga_state[// completed_steps].append(// inventory)
        
        # 注文確定イベント発行
        confirm_event = ConfirmOrderCommand(order_id=saga_id)
        await receiver.event_bus.publish(confirm_event)
        
        # サガ完了
        del receiver.state[saga_id]
        
    async func (receiver) _handle_inventory_failure(saga_id string, event: Event) {
        // 在庫予約失敗処理（補償アクション）
        if saga_id not in receiver.state:
            return
            
        saga_state = receiver.state[saga_id]
        
        # 決済を返金
        if // payment in saga_state[// completed_steps]:
            refund_event = RefundPaymentCommand(order_id=saga_id)
            await receiver.event_bus.publish(refund_event)
            
        # 注文キャンセル
        cancel_event = CancelOrderCommand(
            order_id=saga_id,
            reason=// Inventory unavailable
        )
        await receiver.event_bus.publish(cancel_event)
        
        # サガ状態クリア
        del receiver.state[saga_id]
```

## 実装パターン

### 1. 非同期メッセージング
```go
package main


type RedisEventBus struct {
    // Redis を使った分散イベントバス
    
    func (receiver) __init__(redis_url string) {
        super().__init__()
        receiver.redis_url = redis_url
        receiver.redis = nil
        receiver.pubsub = nil
        
    async func (receiver) connect() {
        // Redis接続
        receiver.redis = await aioredis.from_url(receiver.redis_url)
        receiver.pubsub = receiver.redis.pubsub()
        
    async func (receiver) publish(event: Event) {
        // イベント発行（Redis Pub/Sub）
        if not receiver.redis:
            await receiver.connect()
            
        event_data = {
            'event_id': event.event_id,
            'event_type': event.event_type,
            'source': event.source,
            'timestamp': event.timestamp.isoformat(),
            'data': event.data,
            'version': event.version
        }
        
        await receiver.redis.publish(
            f// events.{event.event_type},
            json.dumps(event_data)
        )
        
    async func (receiver) subscribe_to_events(event_types []str) {
        // イベント購読
        if not receiver.pubsub:
            await receiver.connect()
            
        for event_type in event_types:
            await receiver.pubsub.subscribe(f// events.{event_type})
            
        async for message in receiver.pubsub.listen():
            if message['type'] == 'message':
                await receiver._handle_redis_message(message)
                
    async func (receiver) _handle_redis_message(message) {
        // Redisメッセージ処理
        try:
            event_data = json.loads(message['data'])
            
            event = Event(
                event_id=event_data['event_id'],
                event_type=event_data['event_type'],
                source=event_data['source'],
                timestamp=datetime.fromisoformat(event_data['timestamp']),
                data=event_data['data'],
                version=event_data['version']
            )
            
            await receiver._process_event(event)
            
        except Exception as e:
            receiver.logger.error(f// Error processing Redis message: {e})

# Apache Kafka を使った実装
type KafkaEventBus struct {
    // Kafka を使った分散イベントバス
    
    func (receiver) __init__(bootstrap_servers string) {
        super().__init__()
        receiver.bootstrap_servers = bootstrap_servers
        receiver.producer = nil
        receiver.consumer = nil
        
    async func (receiver) publish(event: Event) {
        // イベント発行（Kafka）
        if not receiver.producer:
                        receiver.producer = AIOKafkaProducer(
                bootstrap_servers=receiver.bootstrap_servers
            )
            await receiver.producer.start()
            
        event_data = json.dumps({
            'event_id': event.event_id,
            'event_type': event.event_type,
            'source': event.source,
            'timestamp': event.timestamp.isoformat(),
            'data': event.data,
            'version': event.version
        })
        
        await receiver.producer.send(
            topic=f// events-{event.event_type},
            value=event_data.encode('utf-8'),
            key=event.event_id.encode('utf-8')
        )
```

### 2. イベント再試行とデッドレターキュー
```go
package main

type ReliableEventHandler struct {
    // 信頼性のあるイベントハンドラー
    
    func (receiver) __init__(handler: EventHandler, max_retries int = 3) {
        receiver.handler = handler
        receiver.max_retries = max_retries
        receiver.retry_count = {}
        
    async func (receiver) handle(event: Event) {
        // 再試行付きイベント処理
        event_key = f// {event.event_id}:{receiver.handler.__class__.__name__}
        retry_count = receiver.retry_count.get(event_key, 0)
        
        try:
            await receiver.handler.handle(event)
            # 成功時は再試行カウントをクリア
            receiver.retry_count.pop(event_key, nil)
            
        except Exception as e:
            retry_count += 1
            receiver.retry_count[event_key] = retry_count
            
            if retry_count <= receiver.max_retries:
                # 指数バックオフで再試行
                delay = 2 ** (retry_count - 1)
                await asyncio.sleep(delay)
                
                await receiver.handle(event)
            else:
                # 最大再試行数を超えた場合はデッドレターキューに送信
                await receiver._send_to_dead_letter_queue(event, e)
                receiver.retry_count.pop(event_key, nil)
                
    func (receiver) can_handle(event: Event) {
        return receiver.handler.can_handle(event)
        
    async func (receiver) _send_to_dead_letter_queue(event: Event, error: Exception) {
        // デッドレターキューへの送信
        dead_letter_event = DeadLetterEvent(
            original_event=event,
            handler_class=receiver.handler.__class__.__name__,
            error_message=str(error),
            retry_count=receiver.max_retries
        )
        
        # デッドレターキューに送信（実装は省略）
        await receiver._publish_to_dead_letter_queue(dead_letter_event)

type EventProcessingMonitor struct {
    // イベント処理監視
    
    func (receiver) __init__() {
        receiver.processing_stats = defaultdict(dict)
        
    func (receiver) record_processing_start(event: Event, handler_name string) {
        // 処理開始記録
        key = f// {event.event_id}:{handler_name}
        receiver.processing_stats[key] = {
            'start_time': datetime.utcnow(),
            'event_type': event.event_type,
            'handler_name': handler_name
        }
        
    func (receiver) record_processing_end(event: Event, handler_name string, success bool) {
        // 処理終了記録
        key = f// {event.event_id}:{handler_name}
        if key in receiver.processing_stats:
            stats = receiver.processing_stats[key]
            stats['end_time'] = datetime.utcnow()
            stats['success'] = success
            stats['duration'] = (stats['end_time'] - stats['start_time']).total_seconds()
            
    func (receiver) get_processing_metrics() {
        // 処理メトリクス取得
        total_events = len(receiver.processing_stats)
        successful_events = sum(1 for s in receiver.processing_stats.values() if s.get('success'))
        
        return {
            'total_events': total_events,
            'successful_events': successful_events,
            'failure_rate': (total_events - successful_events) / total_events if total_events > 0 else 0,
            'average_duration': sum(s.get('duration', 0) for s in receiver.processing_stats.values()) / total_events if total_events > 0 else 0
        }
```

## メリット

### 疎結合
- **コンポーネント独立性**: サービス間の直接的な依存関係を排除
- **インターフェース統一**: イベントという統一されたインターフェース
- **変更の局所化**: 一つのサービス変更が他に影響しない

### スケーラビリティ
- **非同期処理**: ブロッキングしない並行処理
- **水平スケーリング**: コンシューマーの独立スケーリング
- **負荷分散**: 複数のコンシューマーでの処理分散

### リアルタイム性
- **即座の反応**: イベント発生時の即座な処理
- **リアルタイム更新**: 状態変更の即座な反映
- **ライブデータ**: 常に最新状態の維持

## デメリット

### 複雑性
- **分散システム複雑性**: イベントフローの追跡困難
- **デバッグ困難性**: 非同期処理のトラブルシューティング
- **結果整合性**: 即座に一貫性が保証されない

### 信頼性の課題
- **メッセージ損失**: ネットワーク障害時のイベント損失
- **重複処理**: 同じイベントの複数回処理
- **順序保証**: イベント処理順序の管理

### 運用複雑性
- **監視の困難さ**: 分散イベント処理の監視
- **障害追跡**: エラーの根本原因特定
- **パフォーマンス調整**: 適切なスループット調整

## 適用場面

### 適している場合
- **リアルタイム要求**: 即座の応答が必要なシステム
- **疎結合が重要**: マイクロサービス間の独立性
- **非同期処理**: バックグラウンド処理が多い
- **イベント駆動ビジネス**: ビジネスプロセスがイベントベース

### 適していない場合
- **強い整合性要求**: ACIDトランザクションが必須
- **シンプルなシステム**: 複雑性がメリットを上回る
- **同期処理中心**: リアルタイム応答が不要
- **小規模システム**: イベント駆動の恩恵が少ない

## 実装時の考慮事項

### イベント設計
```go
package main

# イベントの後方互換性
type EventVersioning struct {
    // イベントバージョニング
    
        func migrate_event(event: Event) {
        // イベント移行
        if event.event_type == // user.registered and event.version == // 1.0:
            # v1.0 から v2.0 への移行
            event.data['full_name'] = event.data.pop('name', '')
            event.version = // 2.0
        return event

# イベントスキーマ定義
type EventSchema struct {
    // イベントスキーマ
    
    USER_REGISTERED_V1 = {
        // type: // object,
        // properties: {
            // user_id: {// type: // string},
            // email: {// type: // string, // format: // email},
            // name: {// type: // string}
        },
        // required: [// user_id, // email, // name]
    }
    
    USER_REGISTERED_V2 = {
        // type: // object, 
        // properties: {
            // user_id: {// type: // string},
            // email: {// type: // string, // format: // email},
            // full_name: {// type: // string},
            // registration_source: {// type: // string}
        },
        // required: [// user_id, // email, // full_name]
    }
```

### 冪等性の保証
```go
package main

type IdempotentEventHandler struct {
    // 冪等性を保証するイベントハンドラー
    
    func (receiver) __init__(handler: EventHandler, idempotency_store: 'IdempotencyStore') {
        receiver.handler = handler
        receiver.idempotency_store = idempotency_store
        
    async func (receiver) handle(event: Event) {
        // 冪等性チェック付きイベント処理
        handler_key = f// {event.event_id}:{receiver.handler.__class__.__name__}
        
        # 既に処理済みかチェック
        if await receiver.idempotency_store.is_processed(handler_key):
            return
            
        # イベント処理
        await receiver.handler.handle(event)
        
        # 処理済みマーク
        await receiver.idempotency_store.mark_processed(handler_key)
        
    func (receiver) can_handle(event: Event) {
        return receiver.handler.can_handle(event)
```

## 関連パターン
- **CQRS**: コマンドクエリ責任分離との組み合わせ
- **Event Sourcing**: イベントによる状態管理
- **Saga Pattern**: 分散トランザクション管理
- **Observer Pattern**: イベント通知メカニズム
- **Publisher-Subscriber**: メッセージング パターン

## 参考資料
- [Event-Driven Architecture - Martin Fowler](https://martinfowler.com/articles/201701-event-driven.html)
- [Building Event-Driven Microservices - Adam Bellemare](https://www.oreilly.com/library/view/building-event-driven-microservices/9781492057888/)
- [Designing Event-Driven Systems - Ben Stopford](https://www.confluent.io/designing-event-driven-systems/)
- [Event Sourcing - Martin Fowler](https://martinfowler.com/eaaDev/EventSourcing.html)
