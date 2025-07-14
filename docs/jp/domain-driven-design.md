# Domain-Driven Design（ドメイン駆動設計）

## 概要
ドメイン駆動設計（DDD）は、複雑なソフトウェアの設計における戦略的な設計手法です。ビジネスドメインの専門知識を中心に据え、ドメインエキスパートと開発者が共通の言語（ユビキタス言語）を使って協働することで、ビジネス価値の高いソフトウェアを構築します。

## 核心概念

### ユビキタス言語（Ubiquitous Language）
```python
# ドメインエキスパートと開発者が共有する言語で実装
class Order:
    """注文 - ビジネス用語をそのまま使用"""
    def __init__(self, customer: Customer, order_lines: List[OrderLine]):
        self.customer = customer
        self.order_lines = order_lines
        self.status = OrderStatus.PENDING
        
    def calculate_total_amount(self) -> Money:
        """合計金額を計算する - ドメインロジック"""
        total = Money.zero()
        for line in self.order_lines:
            total = total.add(line.calculate_line_total())
        return total
        
    def confirm(self):
        """注文を確定する"""
        if not self.can_be_confirmed():
            raise OrderCannotBeConfirmedException("注文を確定できません")
        self.status = OrderStatus.CONFIRMED
        
    def can_be_confirmed(self) -> bool:
        """注文が確定可能かどうか"""
        return (self.status == OrderStatus.PENDING and 
                len(self.order_lines) > 0 and
                self.customer.is_active())

class OrderLine:
    """注文明細"""
    def __init__(self, product: Product, quantity: Quantity, unit_price: Money):
        self.product = product
        self.quantity = quantity
        self.unit_price = unit_price
        
    def calculate_line_total(self) -> Money:
        """明細の合計金額を計算"""
        return self.unit_price.multiply(self.quantity.value)
```

### 境界づけられたコンテキスト（Bounded Context）
```python
# 販売コンテキスト
class SalesContext:
    """販売に関する境界づけられたコンテキスト"""
    
    class Customer:
        """販売コンテキストでの顧客"""
        def __init__(self, customer_id: CustomerId, name: str, credit_limit: Money):
            self.customer_id = customer_id
            self.name = name
            self.credit_limit = credit_limit
            
        def can_place_order(self, order_amount: Money) -> bool:
            """注文可能かどうか"""
            return order_amount.is_less_than_or_equal(self.credit_limit)
    
    class Order:
        """販売コンテキストでの注文"""
        def place_order(self, customer: Customer, items: List[OrderItem]):
            if not customer.can_place_order(self.calculate_total(items)):
                raise InsufficientCreditException()
            # 注文処理...

# 配送コンテキスト  
class ShippingContext:
    """配送に関する境界づけられたコンテキスト"""
    
    class Customer:
        """配送コンテキストでの顧客（異なる観点）"""
        def __init__(self, customer_id: CustomerId, shipping_address: Address):
            self.customer_id = customer_id  # 同じID、異なる属性
            self.shipping_address = shipping_address
            
        def get_preferred_delivery_time(self) -> TimeSlot:
            """希望配送時間を取得"""
            return self.shipping_preferences.preferred_time_slot
    
    class Shipment:
        """配送"""
        def __init__(self, order_id: OrderId, customer: Customer):
            self.shipment_id = ShipmentId.generate()
            self.order_id = order_id
            self.delivery_address = customer.shipping_address
            self.status = ShipmentStatus.PREPARING
```

## 戦略的設計

### 1. ドメインマップ
```python
class DomainMap:
    """ドメイン全体のマップ定義"""
    
    bounded_contexts = {
        "sales": {
            "description": "販売・注文管理",
            "team": "Sales Team",
            "entities": ["Customer", "Order", "Product"],
            "relationships": {
                "shipping": "Customer Supplier",  # 顧客データを提供
                "inventory": "Conformist"         # 在庫情報に従う
            }
        },
        "shipping": {
            "description": "配送・物流管理", 
            "team": "Logistics Team",
            "entities": ["Shipment", "Customer", "Address"],
            "relationships": {
                "sales": "Customer",              # 顧客データを受け取る
            }
        },
        "inventory": {
            "description": "在庫管理",
            "team": "Inventory Team", 
            "entities": ["Product", "Stock", "Warehouse"],
            "relationships": {
                "sales": "Supplier"               # 在庫情報を提供
            }
        }
    }

# コンテキスト間の関係パターン
class ContextRelationship:
    """コンテキスト間の関係"""
    
    # パートナーシップ
    class Partnership:
        """相互に協力する関係"""
        pass
        
    # 共有カーネル
    class SharedKernel:
        """共通のコードを共有"""
        pass
        
    # 顧客・供給者
    class CustomerSupplier:
        """上流・下流の関係"""
        def __init__(self, upstream_context: str, downstream_context: str):
            self.upstream = upstream_context
            self.downstream = downstream_context
            
    # 準拠者
    class Conformist:
        """上流コンテキストに完全に従う"""
        pass
```

### 2. コンテキストマッピング
```python
# Anti-Corruption Layer（腐敗防止層）
class LegacySystemAdapter:
    """レガシーシステムとの間の腐敗防止層"""
    
    def __init__(self, legacy_customer_service):
        self.legacy_service = legacy_customer_service
        
    def get_customer(self, customer_id: CustomerId) -> Customer:
        """レガシーデータをドメインモデルに変換"""
        legacy_data = self.legacy_service.getCustomerData(customer_id.value)
        
        # レガシーデータ構造からドメインモデルへの変換
        return Customer(
            customer_id=CustomerId(legacy_data['cust_id']),
            name=legacy_data['cust_name'],
            email=Email(legacy_data['email_addr']),
            # レガシーシステムの複雑なロジックをシンプルなドメインロジックに変換
            status=self._convert_legacy_status(legacy_data['status_code'])
        )
        
    def _convert_legacy_status(self, legacy_status: str) -> CustomerStatus:
        """レガシーステータスをドメインステータスに変換"""
        status_mapping = {
            'ACT': CustomerStatus.ACTIVE,
            'INA': CustomerStatus.INACTIVE, 
            'SUS': CustomerStatus.SUSPENDED
        }
        return status_mapping.get(legacy_status, CustomerStatus.UNKNOWN)

# Published Language（公開言語）
class CustomerPublishedLanguage:
    """顧客情報の公開言語定義"""
    
    @dataclass
    class CustomerDTO:
        """外部システム向けの顧客データ転送オブジェクト"""
        customer_id: str
        name: str
        email: str
        status: str
        registration_date: str
        
    def to_published_language(self, customer: Customer) -> CustomerDTO:
        """ドメインモデルから公開言語へ変換"""
        return self.CustomerDTO(
            customer_id=customer.customer_id.value,
            name=customer.name,
            email=customer.email.value,
            status=customer.status.value,
            registration_date=customer.registration_date.isoformat()
        )
```

## 戦術的設計

### 1. エンティティ（Entity）
```python
class Customer:
    """顧客エンティティ - 一意な識別子を持つ"""
    
    def __init__(self, customer_id: CustomerId, name: str, email: Email):
        self._customer_id = customer_id  # 不変の識別子
        self._name = name
        self._email = email
        self._registration_date = datetime.now()
        self._domain_events = []
        
    @property
    def customer_id(self) -> CustomerId:
        """顧客ID（不変）"""
        return self._customer_id
        
    def change_email(self, new_email: Email):
        """メールアドレス変更"""
        if self._email != new_email:
            old_email = self._email
            self._email = new_email
            
            # ドメインイベント発行
            self._domain_events.append(
                CustomerEmailChangedEvent(self._customer_id, old_email, new_email)
            )
            
    def __eq__(self, other):
        """エンティティの同一性は識別子で判断"""
        if not isinstance(other, Customer):
            return False
        return self._customer_id == other._customer_id
        
    def __hash__(self):
        return hash(self._customer_id)

class CustomerId:
    """顧客ID値オブジェクト"""
    def __init__(self, value: str):
        if not value or not value.strip():
            raise ValueError("顧客IDは必須です")
        self._value = value.strip()
        
    @property
    def value(self) -> str:
        return self._value
        
    def __eq__(self, other):
        return isinstance(other, CustomerId) and self._value == other._value
        
    def __hash__(self):
        return hash(self._value)
```

### 2. 値オブジェクト（Value Object）
```python
class Money:
    """金額値オブジェクト - 不変"""
    
    def __init__(self, amount: Decimal, currency: str = "JPY"):
        if amount < 0:
            raise ValueError("金額は0以上である必要があります")
        self._amount = amount
        self._currency = currency
        
    @property
    def amount(self) -> Decimal:
        return self._amount
        
    @property 
    def currency(self) -> str:
        return self._currency
        
    def add(self, other: 'Money') -> 'Money':
        """加算"""
        if self._currency != other._currency:
            raise ValueError("異なる通貨同士は計算できません")
        return Money(self._amount + other._amount, self._currency)
        
    def multiply(self, multiplier: int) -> 'Money':
        """乗算"""
        return Money(self._amount * multiplier, self._currency)
        
    def is_greater_than(self, other: 'Money') -> bool:
        """比較"""
        self._ensure_same_currency(other)
        return self._amount > other._amount
        
    def _ensure_same_currency(self, other: 'Money'):
        if self._currency != other._currency:
            raise ValueError("異なる通貨同士は比較できません")
            
    def __eq__(self, other):
        return (isinstance(other, Money) and 
                self._amount == other._amount and 
                self._currency == other._currency)
                
    def __hash__(self):
        return hash((self._amount, self._currency))
        
    @classmethod
    def zero(cls, currency: str = "JPY") -> 'Money':
        """ゼロ金額"""
        return cls(Decimal('0'), currency)

class Email:
    """メールアドレス値オブジェクト"""
    
    def __init__(self, value: str):
        if not self._is_valid_email(value):
            raise ValueError(f"無効なメールアドレス: {value}")
        self._value = value.lower()
        
    @property
    def value(self) -> str:
        return self._value
        
    def _is_valid_email(self, email: str) -> bool:
        """メールアドレスのバリデーション"""
        import re
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return re.match(pattern, email) is not None
        
    def __eq__(self, other):
        return isinstance(other, Email) and self._value == other._value
        
    def __hash__(self):
        return hash(self._value)
```

### 3. 集約（Aggregate）
```python
class Order:
    """注文集約ルート"""
    
    def __init__(self, customer_id: CustomerId):
        self._order_id = OrderId.generate()
        self._customer_id = customer_id
        self._order_lines = []
        self._status = OrderStatus.DRAFT
        self._order_date = datetime.now()
        self._domain_events = []
        
    def add_order_line(self, product: Product, quantity: int, unit_price: Money):
        """注文明細追加"""
        if self._status != OrderStatus.DRAFT:
            raise InvalidOperationException("確定済みの注文は変更できません")
            
        # ビジネスルール: 同じ商品の重複チェック
        existing_line = self._find_order_line_by_product(product.product_id)
        if existing_line:
            existing_line.change_quantity(existing_line.quantity + quantity)
        else:
            new_line = OrderLine(product, quantity, unit_price)
            self._order_lines.append(new_line)
            
    def remove_order_line(self, product_id: ProductId):
        """注文明細削除"""
        if self._status != OrderStatus.DRAFT:
            raise InvalidOperationException("確定済みの注文は変更できません")
            
        self._order_lines = [line for line in self._order_lines 
                           if line.product_id != product_id]
                           
    def confirm(self):
        """注文確定"""
        self._validate_for_confirmation()
        
        self._status = OrderStatus.CONFIRMED
        
        # ドメインイベント発行
        self._domain_events.append(
            OrderConfirmedEvent(
                order_id=self._order_id,
                customer_id=self._customer_id,
                total_amount=self.calculate_total_amount(),
                order_lines=self._order_lines.copy()
            )
        )
        
    def _validate_for_confirmation(self):
        """確定時のビジネスルール検証"""
        if not self._order_lines:
            raise DomainException("注文明細がありません")
            
        if self.calculate_total_amount().amount <= 0:
            raise DomainException("注文金額は0より大きい必要があります")
            
    def calculate_total_amount(self) -> Money:
        """合計金額計算"""
        total = Money.zero()
        for line in self._order_lines:
            total = total.add(line.calculate_line_total())
        return total
        
    def _find_order_line_by_product(self, product_id: ProductId) -> Optional[OrderLine]:
        """商品IDで注文明細を検索"""
        for line in self._order_lines:
            if line.product_id == product_id:
                return line
        return None
        
    @property
    def domain_events(self) -> List[DomainEvent]:
        """ドメインイベント取得"""
        return self._domain_events.copy()
        
    def clear_domain_events(self):
        """ドメインイベントクリア"""
        self._domain_events.clear()

class OrderLine:
    """注文明細エンティティ（集約内部）"""
    
    def __init__(self, product: Product, quantity: int, unit_price: Money):
        self._product_id = product.product_id
        self._product_name = product.name
        self._quantity = quantity
        self._unit_price = unit_price
        
        if quantity <= 0:
            raise ValueError("数量は1以上である必要があります")
            
    def change_quantity(self, new_quantity: int):
        """数量変更"""
        if new_quantity <= 0:
            raise ValueError("数量は1以上である必要があります")
        self._quantity = new_quantity
        
    def calculate_line_total(self) -> Money:
        """明細合計金額"""
        return self._unit_price.multiply(self._quantity)
        
    @property
    def product_id(self) -> ProductId:
        return self._product_id
        
    @property
    def quantity(self) -> int:
        return self._quantity
```

### 4. ドメインサービス（Domain Service）
```python
class OrderDomainService:
    """注文に関するドメインサービス"""
    
    def __init__(self, customer_repository: CustomerRepository,
                 product_repository: ProductRepository):
        self._customer_repository = customer_repository
        self._product_repository = product_repository
        
    def can_customer_place_order(self, customer_id: CustomerId, 
                               order_amount: Money) -> bool:
        """顧客が注文可能かどうかの判定"""
        customer = self._customer_repository.find_by_id(customer_id)
        if not customer:
            return False
            
        # 複数のビジネスルールを組み合わせた複雑な判定
        return (customer.is_active() and 
                customer.has_sufficient_credit_limit(order_amount) and
                not customer.has_overdue_payments())
                
    def calculate_shipping_cost(self, order: Order, 
                              shipping_address: Address) -> Money:
        """配送料計算"""
        base_cost = Money(Decimal('500'))  # 基本料金
        
        # 重量ベースの計算
        total_weight = self._calculate_total_weight(order)
        weight_cost = self._calculate_weight_cost(total_weight)
        
        # 距離ベースの計算
        distance_cost = self._calculate_distance_cost(shipping_address)
        
        return base_cost.add(weight_cost).add(distance_cost)
        
    def _calculate_total_weight(self, order: Order) -> Weight:
        """注文の総重量計算"""
        total_weight = Weight.zero()
        for line in order.order_lines:
            product = self._product_repository.find_by_id(line.product_id)
            line_weight = product.weight.multiply(line.quantity)
            total_weight = total_weight.add(line_weight)
        return total_weight

class ProductCatalogService:
    """商品カタログサービス"""
    
    def __init__(self, product_repository: ProductRepository):
        self._product_repository = product_repository
        
    def is_product_available_for_order(self, product_id: ProductId, 
                                     quantity: int) -> bool:
        """商品が注文可能かどうか"""
        product = self._product_repository.find_by_id(product_id)
        if not product:
            return False
            
        return (product.is_active() and 
                product.has_sufficient_stock(quantity) and
                not product.is_discontinued())
```

### 5. リポジトリ（Repository）
```python
from abc import ABC, abstractmethod

class CustomerRepository(ABC):
    """顧客リポジトリインターフェース"""
    
    @abstractmethod
    def find_by_id(self, customer_id: CustomerId) -> Optional[Customer]:
        """IDで顧客を検索"""
        pass
        
    @abstractmethod
    def find_by_email(self, email: Email) -> Optional[Customer]:
        """メールアドレスで顧客を検索"""
        pass
        
    @abstractmethod
    def save(self, customer: Customer) -> None:
        """顧客を保存"""
        pass
        
    @abstractmethod
    def delete(self, customer: Customer) -> None:
        """顧客を削除"""
        pass

class SqlCustomerRepository(CustomerRepository):
    """SQL実装の顧客リポジトリ"""
    
    def __init__(self, db_session):
        self._db_session = db_session
        
    def find_by_id(self, customer_id: CustomerId) -> Optional[Customer]:
        """IDで顧客を検索"""
        query = """
            SELECT customer_id, name, email, registration_date, status
            FROM customers 
            WHERE customer_id = %s
        """
        
        result = self._db_session.execute(query, (customer_id.value,)).fetchone()
        if not result:
            return None
            
        return self._map_to_domain(result)
        
    def save(self, customer: Customer) -> None:
        """顧客を保存"""
        # 既存顧客の確認
        existing = self.find_by_id(customer.customer_id)
        
        if existing:
            self._update_customer(customer)
        else:
            self._insert_customer(customer)
            
        # ドメインイベントの処理
        self._publish_domain_events(customer)
        
    def _map_to_domain(self, db_record) -> Customer:
        """データベースレコードからドメインオブジェクトへの変換"""
        return Customer(
            customer_id=CustomerId(db_record['customer_id']),
            name=db_record['name'],
            email=Email(db_record['email'])
        )
        
    def _insert_customer(self, customer: Customer):
        """新規顧客挿入"""
        query = """
            INSERT INTO customers (customer_id, name, email, registration_date, status)
            VALUES (%s, %s, %s, %s, %s)
        """
        
        self._db_session.execute(query, (
            customer.customer_id.value,
            customer.name,
            customer.email.value,
            customer.registration_date,
            customer.status.value
        ))
        
    def _update_customer(self, customer: Customer):
        """既存顧客更新"""
        query = """
            UPDATE customers 
            SET name = %s, email = %s, status = %s
            WHERE customer_id = %s
        """
        
        self._db_session.execute(query, (
            customer.name,
            customer.email.value,
            customer.status.value,
            customer.customer_id.value
        ))
```

## アプリケーション層

### アプリケーションサービス
```python
class OrderApplicationService:
    """注文アプリケーションサービス"""
    
    def __init__(self, 
                 order_repository: OrderRepository,
                 customer_repository: CustomerRepository,
                 product_repository: ProductRepository,
                 order_domain_service: OrderDomainService,
                 unit_of_work: UnitOfWork):
        self._order_repository = order_repository
        self._customer_repository = customer_repository
        self._product_repository = product_repository
        self._order_domain_service = order_domain_service
        self._unit_of_work = unit_of_work
        
    def create_order(self, command: CreateOrderCommand) -> OrderId:
        """注文作成"""
        with self._unit_of_work:
            # ドメインオブジェクト取得
            customer = self._customer_repository.find_by_id(
                CustomerId(command.customer_id)
            )
            if not customer:
                raise CustomerNotFoundException()
                
            # 新しい注文作成
            order = Order(customer.customer_id)
            
            # 注文明細追加
            for item in command.order_items:
                product = self._product_repository.find_by_id(
                    ProductId(item.product_id)
                )
                if not product:
                    raise ProductNotFoundException()
                    
                order.add_order_line(
                    product=product,
                    quantity=item.quantity,
                    unit_price=Money(item.unit_price)
                )
                
            # ビジネスルール検証
            if not self._order_domain_service.can_customer_place_order(
                customer.customer_id, order.calculate_total_amount()
            ):
                raise CustomerCannotPlaceOrderException()
                
            # 注文保存
            self._order_repository.save(order)
            
            return order.order_id
            
    def confirm_order(self, command: ConfirmOrderCommand) -> None:
        """注文確定"""
        with self._unit_of_work:
            order = self._order_repository.find_by_id(
                OrderId(command.order_id)
            )
            if not order:
                raise OrderNotFoundException()
                
            order.confirm()
            self._order_repository.save(order)

@dataclass
class CreateOrderCommand:
    """注文作成コマンド"""
    customer_id: str
    order_items: List['OrderItemCommand']
    
@dataclass 
class OrderItemCommand:
    """注文明細コマンド"""
    product_id: str
    quantity: int
    unit_price: str  # Decimal文字列
```

## ドメインイベント

### イベント定義と発行
```python
class DomainEvent(ABC):
    """ドメインイベント基底クラス"""
    def __init__(self):
        self.occurred_at = datetime.utcnow()
        self.event_id = str(uuid.uuid4())

class OrderConfirmedEvent(DomainEvent):
    """注文確定イベント"""
    def __init__(self, order_id: OrderId, customer_id: CustomerId, 
                 total_amount: Money, order_lines: List[OrderLine]):
        super().__init__()
        self.order_id = order_id
        self.customer_id = customer_id
        self.total_amount = total_amount
        self.order_lines = order_lines

class CustomerEmailChangedEvent(DomainEvent):
    """顧客メールアドレス変更イベント"""
    def __init__(self, customer_id: CustomerId, old_email: Email, new_email: Email):
        super().__init__()
        self.customer_id = customer_id
        self.old_email = old_email
        self.new_email = new_email

# イベントハンドラー
class OrderConfirmedEventHandler:
    """注文確定イベントハンドラー"""
    
    def __init__(self, inventory_service: InventoryService,
                 notification_service: NotificationService):
        self._inventory_service = inventory_service
        self._notification_service = notification_service
        
    def handle(self, event: OrderConfirmedEvent):
        """イベント処理"""
        # 在庫予約
        for line in event.order_lines:
            self._inventory_service.reserve_stock(
                line.product_id, line.quantity
            )
            
        # 顧客通知
        self._notification_service.send_order_confirmation(
            event.customer_id, event.order_id
        )

# イベント発行
class DomainEventPublisher:
    """ドメインイベント発行者"""
    
    def __init__(self):
        self._handlers = defaultdict(list)
        
    def subscribe(self, event_type: Type[DomainEvent], handler):
        """イベントハンドラー登録"""
        self._handlers[event_type].append(handler)
        
    def publish(self, event: DomainEvent):
        """イベント発行"""
        event_type = type(event)
        for handler in self._handlers[event_type]:
            try:
                handler.handle(event)
            except Exception as e:
                # エラーログ記録
                logger.error(f"Event handler failed: {e}")
```

## 実装時の考慮事項

### 1. 永続化の無知（Persistence Ignorance）
```python
# ドメインモデルはデータベースの詳細を知らない
class Customer:
    """永続化技術に依存しないドメインモデル"""
    
    def __init__(self, customer_id: CustomerId, name: str, email: Email):
        # データベースのアノテーションや依存性はない
        self._customer_id = customer_id
        self._name = name
        self._email = email
        
    # ビジネスロジックのみに集中
    def change_email(self, new_email: Email):
        if self._email != new_email:
            self._email = new_email
            # ドメインイベント発行

# インフラストラクチャ層で永続化の詳細を処理
class CustomerEntity:
    """ORMエンティティ"""
    __tablename__ = 'customers'
    
    id = Column(String, primary_key=True)
    name = Column(String, nullable=False)
    email = Column(String, nullable=False)
    registration_date = Column(DateTime)
    
    def to_domain(self) -> Customer:
        """ドメインモデルへの変換"""
        return Customer(
            customer_id=CustomerId(self.id),
            name=self.name,
            email=Email(self.email)
        )
        
    @classmethod
    def from_domain(cls, customer: Customer) -> 'CustomerEntity':
        """ドメインモデルからの変換"""
        return cls(
            id=customer.customer_id.value,
            name=customer.name,
            email=customer.email.value
        )
```

### 2. 集約設計のガイドライン
```python
# 小さな集約を設計
class Order:
    """注文集約 - 注文と注文明細のみを含む"""
    
    def __init__(self, customer_id: CustomerId):
        self._order_id = OrderId.generate()
        self._customer_id = customer_id  # 参照のみ、Customerオブジェクトは含まない
        self._order_lines = []
        
    def add_order_line(self, product_id: ProductId, quantity: int, unit_price: Money):
        # Productオブジェクトは集約に含めず、IDのみ保持
        new_line = OrderLine(product_id, quantity, unit_price)
        self._order_lines.append(new_line)

# IDによる参照
class OrderLine:
    """注文明細"""
    
    def __init__(self, product_id: ProductId, quantity: int, unit_price: Money):
        self._product_id = product_id  # Product集約への参照
        self._quantity = quantity
        self._unit_price = unit_price
        
    # 商品情報が必要な場合はリポジトリから取得
    def get_product_name(self, product_repository: ProductRepository) -> str:
        product = product_repository.find_by_id(self._product_id)
        return product.name if product else "Unknown Product"
```

## メリット

### ビジネス価値の向上
- **ドメインエキスパートとの協働**: 共通言語による効果的なコミュニケーション
- **ビジネスロジックの表現**: ドメインの複雑性を適切にモデル化
- **変更への対応**: ビジネス変更をコードに反映しやすい

### 設計品質の向上
- **関心の分離**: ドメインロジックと技術的詳細の分離
- **テスタビリティ**: ドメインロジックの単体テスト
- **保守性**: 理解しやすく変更しやすいコード

## デメリット

### 学習コスト
- **複雑性**: DDD概念の理解とマスター
- **設計スキル**: 適切な境界設定とモデリング
- **チーム教育**: 開発チーム全体への浸透

### 開発コスト
- **初期コスト**: ドメインモデリングと設計の時間
- **オーバーヘッド**: 単純なCRUD操作でも複雑な実装
- **技術的負債**: 不適切な設計による複雑性の増大

## 適用場面

### 適している場合
- **複雑なビジネスロジック**: 複雑なルールや制約が多い
- **大規模システム**: 複数チームでの開発
- **長期運用**: 継続的な機能追加・変更が予想される
- **ドメインエキスパートの存在**: ビジネス知識を持つステークホルダー

### 適していない場合
- **単純なCRUD**: 基本的なデータ操作のみ
- **小規模システム**: オーバーエンジニアリングのリスク
- **短期プロジェクト**: 設計コストがメリットを上回る
- **技術的制約**: レガシーシステムとの強い結合

## 関連パターン
- **CQRS**: 読み取りと書き込みモデルの分離
- **Event Sourcing**: ドメインイベントによる状態管理
- **Hexagonal Architecture**: ドメインロジックの保護
- **Microservices**: 境界づけられたコンテキストごとのサービス分割

## 参考資料
- [Domain-Driven Design - Eric Evans](https://www.oreilly.com/library/view/domain-driven-design-tackling/0321125215/)
- [Implementing Domain-Driven Design - Vaughn Vernon](https://www.oreilly.com/library/view/implementing-domain-driven-design/9780133039900/)
- [Domain-Driven Design Reference - Eric Evans](https://domainlanguage.com/ddd/reference/)
- [DDD Community](https://dddcommunity.org/)
