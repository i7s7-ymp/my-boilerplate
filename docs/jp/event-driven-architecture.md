# Event-Driven Architecture（イベント駆動アーキテクチャ）

## 概要
イベント駆動アーキテクチャ（EDA）は、イベントの生成、検出、消費、反応を中心に設計されたアーキテクチャパターンです。システムコンポーネント間の疎結合を実現し、リアルタイム性と拡張性を重視したイベントベースの通信を特徴とします。

## 核心概念

### イベント（Event）
```python
from dataclasses import dataclass
from datetime import datetime
from typing import Dict, Any
import uuid

@dataclass
class Event:
    """基本イベントクラス"""
    event_id: str
    event_type: str
    source: str
    timestamp: datetime
    data: Dict[str, Any]
    version: str = "1.0"
    
    def __post_init__(self):
        if not self.event_id:
            self.event_id = str(uuid.uuid4())
        if not self.timestamp:
            self.timestamp = datetime.utcnow()

# 具体的なイベント例
@dataclass
class UserRegisteredEvent(Event):
    """ユーザー登録イベント"""
    def __init__(self, user_id: str, email: str, name: str):
        super().__init__(
            event_id=str(uuid.uuid4()),
            event_type="user.registered",
            source="user-service",
            timestamp=datetime.utcnow(),
            data={
                "user_id": user_id,
                "email": email,
                "name": name
            }
        )

@dataclass  
class OrderPlacedEvent(Event):
    """注文作成イベント"""
    def __init__(self, order_id: str, customer_id: str, amount: float, items: list):
        super().__init__(
            event_id=str(uuid.uuid4()),
            event_type="order.placed",
            source="order-service", 
            timestamp=datetime.utcnow(),
            data={
                "order_id": order_id,
                "customer_id": customer_id,
                "amount": amount,
                "items": items
            }
        )

@dataclass
class PaymentProcessedEvent(Event):
    """決済処理イベント"""
    def __init__(self, payment_id: str, order_id: str, amount: float, status: str):
        super().__init__(
            event_id=str(uuid.uuid4()),
            event_type="payment.processed",
            source="payment-service",
            timestamp=datetime.utcnow(),
            data={
                "payment_id": payment_id,
                "order_id": order_id,
                "amount": amount,
                "status": status
            }
        )
```

## アーキテクチャコンポーネント

### 1. イベントプロデューサー（Event Producer）
```python
from abc import ABC, abstractmethod
from typing import List

class EventProducer(ABC):
    """イベント生成者の抽象クラス"""
    
    @abstractmethod
    async def publish_event(self, event: Event) -> None:
        """イベントを発行"""
        pass

class UserService(EventProducer):
    """ユーザーサービス（イベント生成者）"""
    
    def __init__(self, event_bus: 'EventBus', user_repository: 'UserRepository'):
        self.event_bus = event_bus
        self.user_repository = user_repository
        
    async def register_user(self, email: str, name: str, password: str) -> str:
        """ユーザー登録"""
        # ユーザー作成
        user_id = await self.user_repository.create_user(email, name, password)
        
        # イベント発行
        event = UserRegisteredEvent(
            user_id=user_id,
            email=email,
            name=name
        )
        await self.publish_event(event)
        
        return user_id
        
    async def publish_event(self, event: Event) -> None:
        """イベント発行"""
        await self.event_bus.publish(event)

class OrderService(EventProducer):
    """注文サービス（イベント生成者）"""
    
    def __init__(self, event_bus: 'EventBus', order_repository: 'OrderRepository'):
        self.event_bus = event_bus
        self.order_repository = order_repository
        
    async def place_order(self, customer_id: str, items: List[dict]) -> str:
        """注文作成"""
        # 注文作成
        order_id = await self.order_repository.create_order(customer_id, items)
        amount = sum(item['price'] * item['quantity'] for item in items)
        
        # イベント発行
        event = OrderPlacedEvent(
            order_id=order_id,
            customer_id=customer_id,
            amount=amount,
            items=items
        )
        await self.publish_event(event)
        
        return order_id
        
    async def publish_event(self, event: Event) -> None:
        await self.event_bus.publish(event)
```

### 2. イベントコンシューマー（Event Consumer）
```python
class EventHandler(ABC):
    """イベントハンドラーの抽象クラス"""
    
    @abstractmethod
    async def handle(self, event: Event) -> None:
        """イベント処理"""
        pass
        
    @abstractmethod
    def can_handle(self, event: Event) -> bool:
        """このハンドラーがイベントを処理できるか"""
        pass

class EmailNotificationHandler(EventHandler):
    """メール通知ハンドラー"""
    
    def __init__(self, email_service: 'EmailService'):
        self.email_service = email_service
        self.handled_event_types = ["user.registered", "order.placed"]
        
    async def handle(self, event: Event) -> None:
        """イベント処理"""
        if event.event_type == "user.registered":
            await self._handle_user_registered(event)
        elif event.event_type == "order.placed":
            await self._handle_order_placed(event)
            
    def can_handle(self, event: Event) -> bool:
        return event.event_type in self.handled_event_types
        
    async def _handle_user_registered(self, event: Event):
        """ユーザー登録イベント処理"""
        user_data = event.data
        await self.email_service.send_welcome_email(
            email=user_data['email'],
            name=user_data['name']
        )
        
    async def _handle_order_placed(self, event: Event):
        """注文作成イベント処理"""
        order_data = event.data
        await self.email_service.send_order_confirmation(
            customer_id=order_data['customer_id'],
            order_id=order_data['order_id'],
            amount=order_data['amount']
        )

class InventoryHandler(EventHandler):
    """在庫管理ハンドラー"""
    
    def __init__(self, inventory_service: 'InventoryService'):
        self.inventory_service = inventory_service
        
    async def handle(self, event: Event) -> None:
        if event.event_type == "order.placed":
            await self._handle_order_placed(event)
            
    def can_handle(self, event: Event) -> bool:
        return event.event_type == "order.placed"
        
    async def _handle_order_placed(self, event: Event):
        """注文作成時の在庫処理"""
        order_data = event.data
        
        for item in order_data['items']:
            await self.inventory_service.reserve_stock(
                product_id=item['product_id'],
                quantity=item['quantity']
            )

class PaymentProcessingHandler(EventHandler):
    """決済処理ハンドラー"""
    
    def __init__(self, payment_service: 'PaymentService', event_bus: 'EventBus'):
        self.payment_service = payment_service
        self.event_bus = event_bus
        
    async def handle(self, event: Event) -> None:
        if event.event_type == "order.placed":
            await self._handle_order_placed(event)
            
    def can_handle(self, event: Event) -> bool:
        return event.event_type == "order.placed"
        
    async def _handle_order_placed(self, event: Event):
        """注文作成時の決済処理"""
        order_data = event.data
        
        try:
            payment_result = await self.payment_service.process_payment(
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
            await self.event_bus.publish(payment_event)
            
        except Exception as e:
            # 決済失敗イベント発行
            failure_event = PaymentFailedEvent(
                order_id=order_data['order_id'],
                reason=str(e)
            )
            await self.event_bus.publish(failure_event)
```

### 3. イベントバス（Event Bus）
```python
from asyncio import Queue
from collections import defaultdict
from typing import List, Callable
import asyncio
import logging

class EventBus:
    """イベントバス - イベントの配信を管理"""
    
    def __init__(self):
        self.handlers = defaultdict(list)
        self.event_queue = Queue()
        self.running = False
        self.logger = logging.getLogger(__name__)
        
    def subscribe(self, event_type: str, handler: EventHandler):
        """イベントハンドラーの登録"""
        self.handlers[event_type].append(handler)
        self.logger.info(f"Handler {handler.__class__.__name__} subscribed to {event_type}")
        
    def subscribe_to_all(self, handler: EventHandler):
        """全イベントタイプへの登録"""
        self.handlers['*'].append(handler)
        
    async def publish(self, event: Event):
        """イベント発行"""
        await self.event_queue.put(event)
        self.logger.info(f"Event published: {event.event_type} ({event.event_id})")
        
    async def start(self):
        """イベント処理開始"""
        self.running = True
        self.logger.info("Event bus started")
        
        while self.running:
            try:
                # イベント取得（タイムアウト付き）
                event = await asyncio.wait_for(self.event_queue.get(), timeout=1.0)
                await self._process_event(event)
                
            except asyncio.TimeoutError:
                # タイムアウト時は継続
                continue
            except Exception as e:
                self.logger.error(f"Error processing event: {e}")
                
    async def stop(self):
        """イベント処理停止"""
        self.running = False
        self.logger.info("Event bus stopped")
        
    async def _process_event(self, event: Event):
        """イベント処理"""
        # 特定のイベントタイプのハンドラー
        specific_handlers = self.handlers.get(event.event_type, [])
        
        # 全イベント対応ハンドラー
        global_handlers = self.handlers.get('*', [])
        
        all_handlers = specific_handlers + global_handlers
        
        if not all_handlers:
            self.logger.warning(f"No handlers for event type: {event.event_type}")
            return
            
        # 並行処理でハンドラー実行
        tasks = []
        for handler in all_handlers:
            if handler.can_handle(event):
                task = asyncio.create_task(self._handle_event_safely(handler, event))
                tasks.append(task)
                
        if tasks:
            await asyncio.gather(*tasks, return_exceptions=True)
            
    async def _handle_event_safely(self, handler: EventHandler, event: Event):
        """安全なイベント処理（エラーハンドリング付き）"""
        try:
            await handler.handle(event)
            self.logger.info(
                f"Event {event.event_type} handled by {handler.__class__.__name__}"
            )
        except Exception as e:
            self.logger.error(
                f"Handler {handler.__class__.__name__} failed for event "
                f"{event.event_type}: {e}"
            )
```

### 4. イベントストア（Event Store）
```python
from typing import Optional, List
from datetime import datetime

class EventStore:
    """イベント永続化ストア"""
    
    def __init__(self, database_connection):
        self.db = database_connection
        
    async def save_event(self, event: Event) -> None:
        """イベント保存"""
        query = """
            INSERT INTO events (event_id, event_type, source, timestamp, data, version)
            VALUES (%s, %s, %s, %s, %s, %s)
        """
        
        await self.db.execute(query, (
            event.event_id,
            event.event_type, 
            event.source,
            event.timestamp,
            json.dumps(event.data),
            event.version
        ))
        
    async def get_events(self, 
                        event_types: Optional[List[str]] = None,
                        from_timestamp: Optional[datetime] = None,
                        to_timestamp: Optional[datetime] = None,
                        limit: int = 1000) -> List[Event]:
        """イベント取得"""
        
        query = "SELECT * FROM events WHERE 1=1"
        params = []
        
        if event_types:
            placeholders = ','.join(['%s'] * len(event_types))
            query += f" AND event_type IN ({placeholders})"
            params.extend(event_types)
            
        if from_timestamp:
            query += " AND timestamp >= %s"
            params.append(from_timestamp)
            
        if to_timestamp:
            query += " AND timestamp <= %s"
            params.append(to_timestamp)
            
        query += " ORDER BY timestamp DESC LIMIT %s"
        params.append(limit)
        
        rows = await self.db.fetch_all(query, params)
        
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
        
    async def get_events_by_aggregate(self, aggregate_id: str) -> List[Event]:
        """集約IDでイベント取得"""
        query = """
            SELECT * FROM events 
            WHERE JSON_EXTRACT(data, '$.aggregate_id') = %s
            ORDER BY timestamp ASC
        """
        
        rows = await self.db.fetch_all(query, (aggregate_id,))
        return [self._row_to_event(row) for row in rows]

class EventSourcingRepository:
    """イベントソーシングリポジトリ"""
    
    def __init__(self, event_store: EventStore):
        self.event_store = event_store
        
    async def save_aggregate(self, aggregate_id: str, events: List[Event]) -> None:
        """集約のイベント保存"""
        for event in events:
            # 集約IDをイベントデータに追加
            event.data['aggregate_id'] = aggregate_id
            await self.event_store.save_event(event)
            
    async def load_aggregate(self, aggregate_id: str, aggregate_class) -> Optional[object]:
        """集約の復元"""
        events = await self.event_store.get_events_by_aggregate(aggregate_id)
        
        if not events:
            return None
            
        # 集約をイベントから復元
        aggregate = aggregate_class()
        for event in events:
            aggregate.apply_event(event)
            
        return aggregate
```

## イベントパターン

### 1. コマンドクエリ責任分離（CQRS）との組み合わせ
```python
class OrderAggregate:
    """注文集約（イベントソーシング）"""
    
    def __init__(self):
        self.order_id = None
        self.customer_id = None
        self.status = None
        self.items = []
        self.uncommitted_events = []
        
    def place_order(self, order_id: str, customer_id: str, items: List[dict]):
        """注文作成コマンド"""
        if self.order_id is not None:
            raise ValueError("Order already exists")
            
        # イベント生成
        event = OrderPlacedEvent(
            order_id=order_id,
            customer_id=customer_id,
            amount=sum(item['price'] * item['quantity'] for item in items),
            items=items
        )
        
        self.apply_event(event)
        self.uncommitted_events.append(event)
        
    def confirm_order(self):
        """注文確定コマンド"""
        if self.status != "PENDING":
            raise ValueError("Order cannot be confirmed")
            
        event = OrderConfirmedEvent(
            order_id=self.order_id,
            confirmed_at=datetime.utcnow()
        )
        
        self.apply_event(event)
        self.uncommitted_events.append(event)
        
    def apply_event(self, event: Event):
        """イベント適用"""
        if event.event_type == "order.placed":
            self._apply_order_placed(event)
        elif event.event_type == "order.confirmed":
            self._apply_order_confirmed(event)
            
    def _apply_order_placed(self, event: Event):
        """注文作成イベント適用"""
        data = event.data
        self.order_id = data['order_id']
        self.customer_id = data['customer_id']
        self.items = data['items']
        self.status = "PENDING"
        
    def _apply_order_confirmed(self, event: Event):
        """注文確定イベント適用"""
        self.status = "CONFIRMED"
        
    def get_uncommitted_events(self) -> List[Event]:
        """未コミットイベント取得"""
        return self.uncommitted_events.copy()
        
    def mark_events_as_committed(self):
        """イベントをコミット済みにマーク"""
        self.uncommitted_events.clear()

# 読み取りモデル更新
class OrderProjectionHandler(EventHandler):
    """注文読み取りモデル更新ハンドラー"""
    
    def __init__(self, read_model_repository: 'OrderReadModelRepository'):
        self.read_model_repository = read_model_repository
        
    async def handle(self, event: Event) -> None:
        if event.event_type == "order.placed":
            await self._handle_order_placed(event)
        elif event.event_type == "order.confirmed":
            await self._handle_order_confirmed(event)
            
    def can_handle(self, event: Event) -> bool:
        return event.event_type in ["order.placed", "order.confirmed"]
        
    async def _handle_order_placed(self, event: Event):
        """注文作成時の読み取りモデル更新"""
        data = event.data
        
        order_view = OrderReadModel(
            order_id=data['order_id'],
            customer_id=data['customer_id'],
            total_amount=data['amount'],
            status="PENDING",
            created_at=event.timestamp
        )
        
        await self.read_model_repository.save(order_view)
        
    async def _handle_order_confirmed(self, event: Event):
        """注文確定時の読み取りモデル更新"""
        data = event.data
        
        order_view = await self.read_model_repository.get_by_id(data['order_id'])
        if order_view:
            order_view.status = "CONFIRMED"
            order_view.confirmed_at = event.timestamp
            await self.read_model_repository.save(order_view)
```

### 2. サガパターン（Saga Pattern）
```python
class OrderProcessingSaga:
    """注文処理サガ"""
    
    def __init__(self, event_bus: EventBus):
        self.event_bus = event_bus
        self.state = {}
        
        # イベントハンドラー登録
        event_bus.subscribe("order.placed", self)
        event_bus.subscribe("payment.processed", self)
        event_bus.subscribe("payment.failed", self)
        event_bus.subscribe("inventory.reserved", self)
        event_bus.subscribe("inventory.reservation.failed", self)
        
    async def handle(self, event: Event) -> None:
        """サガイベント処理"""
        saga_id = event.data.get('order_id')
        
        if event.event_type == "order.placed":
            await self._start_saga(saga_id, event)
        elif event.event_type == "payment.processed":
            await self._handle_payment_success(saga_id, event)
        elif event.event_type == "payment.failed":
            await self._handle_payment_failure(saga_id, event)
        elif event.event_type == "inventory.reserved":
            await self._handle_inventory_success(saga_id, event)
        elif event.event_type == "inventory.reservation.failed":
            await self._handle_inventory_failure(saga_id, event)
            
    def can_handle(self, event: Event) -> bool:
        return event.event_type in [
            "order.placed", "payment.processed", "payment.failed",
            "inventory.reserved", "inventory.reservation.failed"
        ]
        
    async def _start_saga(self, saga_id: str, event: Event):
        """サガ開始"""
        self.state[saga_id] = {
            "step": "payment_processing",
            "order_data": event.data,
            "completed_steps": []
        }
        
        # 決済処理イベント発行
        payment_event = ProcessPaymentCommand(
            order_id=saga_id,
            customer_id=event.data['customer_id'],
            amount=event.data['amount']
        )
        await self.event_bus.publish(payment_event)
        
    async def _handle_payment_success(self, saga_id: str, event: Event):
        """決済成功処理"""
        if saga_id not in self.state:
            return
            
        saga_state = self.state[saga_id]
        saga_state["completed_steps"].append("payment")
        saga_state["step"] = "inventory_processing"
        
        # 在庫予約イベント発行
        order_data = saga_state["order_data"]
        inventory_event = ReserveInventoryCommand(
            order_id=saga_id,
            items=order_data['items']
        )
        await self.event_bus.publish(inventory_event)
        
    async def _handle_payment_failure(self, saga_id: str, event: Event):
        """決済失敗処理（補償アクション）"""
        if saga_id not in self.state:
            return
            
        # 注文キャンセルイベント発行
        cancel_event = CancelOrderCommand(
            order_id=saga_id,
            reason="Payment failed"
        )
        await self.event_bus.publish(cancel_event)
        
        # サガ状態クリア
        del self.state[saga_id]
        
    async def _handle_inventory_success(self, saga_id: str, event: Event):
        """在庫予約成功処理"""
        if saga_id not in self.state:
            return
            
        saga_state = self.state[saga_id]
        saga_state["completed_steps"].append("inventory")
        
        # 注文確定イベント発行
        confirm_event = ConfirmOrderCommand(order_id=saga_id)
        await self.event_bus.publish(confirm_event)
        
        # サガ完了
        del self.state[saga_id]
        
    async def _handle_inventory_failure(self, saga_id: str, event: Event):
        """在庫予約失敗処理（補償アクション）"""
        if saga_id not in self.state:
            return
            
        saga_state = self.state[saga_id]
        
        # 決済を返金
        if "payment" in saga_state["completed_steps"]:
            refund_event = RefundPaymentCommand(order_id=saga_id)
            await self.event_bus.publish(refund_event)
            
        # 注文キャンセル
        cancel_event = CancelOrderCommand(
            order_id=saga_id,
            reason="Inventory unavailable"
        )
        await self.event_bus.publish(cancel_event)
        
        # サガ状態クリア
        del self.state[saga_id]
```

## 実装パターン

### 1. 非同期メッセージング
```python
import aioredis
import json

class RedisEventBus(EventBus):
    """Redis を使った分散イベントバス"""
    
    def __init__(self, redis_url: str):
        super().__init__()
        self.redis_url = redis_url
        self.redis = None
        self.pubsub = None
        
    async def connect(self):
        """Redis接続"""
        self.redis = await aioredis.from_url(self.redis_url)
        self.pubsub = self.redis.pubsub()
        
    async def publish(self, event: Event):
        """イベント発行（Redis Pub/Sub）"""
        if not self.redis:
            await self.connect()
            
        event_data = {
            'event_id': event.event_id,
            'event_type': event.event_type,
            'source': event.source,
            'timestamp': event.timestamp.isoformat(),
            'data': event.data,
            'version': event.version
        }
        
        await self.redis.publish(
            f"events.{event.event_type}",
            json.dumps(event_data)
        )
        
    async def subscribe_to_events(self, event_types: List[str]):
        """イベント購読"""
        if not self.pubsub:
            await self.connect()
            
        for event_type in event_types:
            await self.pubsub.subscribe(f"events.{event_type}")
            
        async for message in self.pubsub.listen():
            if message['type'] == 'message':
                await self._handle_redis_message(message)
                
    async def _handle_redis_message(self, message):
        """Redisメッセージ処理"""
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
            
            await self._process_event(event)
            
        except Exception as e:
            self.logger.error(f"Error processing Redis message: {e}")

# Apache Kafka を使った実装
class KafkaEventBus(EventBus):
    """Kafka を使った分散イベントバス"""
    
    def __init__(self, bootstrap_servers: str):
        super().__init__()
        self.bootstrap_servers = bootstrap_servers
        self.producer = None
        self.consumer = None
        
    async def publish(self, event: Event):
        """イベント発行（Kafka）"""
        if not self.producer:
            from aiokafka import AIOKafkaProducer
            self.producer = AIOKafkaProducer(
                bootstrap_servers=self.bootstrap_servers
            )
            await self.producer.start()
            
        event_data = json.dumps({
            'event_id': event.event_id,
            'event_type': event.event_type,
            'source': event.source,
            'timestamp': event.timestamp.isoformat(),
            'data': event.data,
            'version': event.version
        })
        
        await self.producer.send(
            topic=f"events-{event.event_type}",
            value=event_data.encode('utf-8'),
            key=event.event_id.encode('utf-8')
        )
```

### 2. イベント再試行とデッドレターキュー
```python
class ReliableEventHandler:
    """信頼性のあるイベントハンドラー"""
    
    def __init__(self, handler: EventHandler, max_retries: int = 3):
        self.handler = handler
        self.max_retries = max_retries
        self.retry_count = {}
        
    async def handle(self, event: Event) -> None:
        """再試行付きイベント処理"""
        event_key = f"{event.event_id}:{self.handler.__class__.__name__}"
        retry_count = self.retry_count.get(event_key, 0)
        
        try:
            await self.handler.handle(event)
            # 成功時は再試行カウントをクリア
            self.retry_count.pop(event_key, None)
            
        except Exception as e:
            retry_count += 1
            self.retry_count[event_key] = retry_count
            
            if retry_count <= self.max_retries:
                # 指数バックオフで再試行
                delay = 2 ** (retry_count - 1)
                await asyncio.sleep(delay)
                
                await self.handle(event)
            else:
                # 最大再試行数を超えた場合はデッドレターキューに送信
                await self._send_to_dead_letter_queue(event, e)
                self.retry_count.pop(event_key, None)
                
    def can_handle(self, event: Event) -> bool:
        return self.handler.can_handle(event)
        
    async def _send_to_dead_letter_queue(self, event: Event, error: Exception):
        """デッドレターキューへの送信"""
        dead_letter_event = DeadLetterEvent(
            original_event=event,
            handler_class=self.handler.__class__.__name__,
            error_message=str(error),
            retry_count=self.max_retries
        )
        
        # デッドレターキューに送信（実装は省略）
        await self._publish_to_dead_letter_queue(dead_letter_event)

class EventProcessingMonitor:
    """イベント処理監視"""
    
    def __init__(self):
        self.processing_stats = defaultdict(dict)
        
    def record_processing_start(self, event: Event, handler_name: str):
        """処理開始記録"""
        key = f"{event.event_id}:{handler_name}"
        self.processing_stats[key] = {
            'start_time': datetime.utcnow(),
            'event_type': event.event_type,
            'handler_name': handler_name
        }
        
    def record_processing_end(self, event: Event, handler_name: str, success: bool):
        """処理終了記録"""
        key = f"{event.event_id}:{handler_name}"
        if key in self.processing_stats:
            stats = self.processing_stats[key]
            stats['end_time'] = datetime.utcnow()
            stats['success'] = success
            stats['duration'] = (stats['end_time'] - stats['start_time']).total_seconds()
            
    def get_processing_metrics(self) -> dict:
        """処理メトリクス取得"""
        total_events = len(self.processing_stats)
        successful_events = sum(1 for s in self.processing_stats.values() if s.get('success'))
        
        return {
            'total_events': total_events,
            'successful_events': successful_events,
            'failure_rate': (total_events - successful_events) / total_events if total_events > 0 else 0,
            'average_duration': sum(s.get('duration', 0) for s in self.processing_stats.values()) / total_events if total_events > 0 else 0
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
```python
# イベントの後方互換性
class EventVersioning:
    """イベントバージョニング"""
    
    @staticmethod
    def migrate_event(event: Event) -> Event:
        """イベント移行"""
        if event.event_type == "user.registered" and event.version == "1.0":
            # v1.0 から v2.0 への移行
            event.data['full_name'] = event.data.pop('name', '')
            event.version = "2.0"
        return event

# イベントスキーマ定義
class EventSchema:
    """イベントスキーマ"""
    
    USER_REGISTERED_V1 = {
        "type": "object",
        "properties": {
            "user_id": {"type": "string"},
            "email": {"type": "string", "format": "email"},
            "name": {"type": "string"}
        },
        "required": ["user_id", "email", "name"]
    }
    
    USER_REGISTERED_V2 = {
        "type": "object", 
        "properties": {
            "user_id": {"type": "string"},
            "email": {"type": "string", "format": "email"},
            "full_name": {"type": "string"},
            "registration_source": {"type": "string"}
        },
        "required": ["user_id", "email", "full_name"]
    }
```

### 冪等性の保証
```python
class IdempotentEventHandler:
    """冪等性を保証するイベントハンドラー"""
    
    def __init__(self, handler: EventHandler, idempotency_store: 'IdempotencyStore'):
        self.handler = handler
        self.idempotency_store = idempotency_store
        
    async def handle(self, event: Event) -> None:
        """冪等性チェック付きイベント処理"""
        handler_key = f"{event.event_id}:{self.handler.__class__.__name__}"
        
        # 既に処理済みかチェック
        if await self.idempotency_store.is_processed(handler_key):
            return
            
        # イベント処理
        await self.handler.handle(event)
        
        # 処理済みマーク
        await self.idempotency_store.mark_processed(handler_key)
        
    def can_handle(self, event: Event) -> bool:
        return self.handler.can_handle(event)
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
