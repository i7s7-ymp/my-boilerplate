# Domain-Driven Design（ドメイン駆動設計）

## 概要
ドメイン駆動設計（DDD）は、複雑なソフトウェアの設計における戦略的な設計手法です。ビジネスドメインの専門知識を中心に据え、ドメインエキスパートと開発者が共通の言語（ユビキタス言語）を使って協働することで、ビジネス価値の高いソフトウェアを構築します。

## 核心概念

### ユビキタス言語（Ubiquitous Language）
```go
package main

# ドメインエキスパートと開発者が共有する言語で実装
type Order struct {
    // 注文 - ビジネス用語をそのまま使用
    func (receiver) __init__(customer: Customer, order_lines []OrderLine) {
        receiver.customer = customer
        receiver.order_lines = order_lines
        receiver.status = OrderStatus.PENDING
        
    func (receiver) calculate_total_amount() {
        // 合計金額を計算する - ドメインロジック
        total = Money.zero()
        for line in receiver.order_lines:
            total = total.add(line.calculate_line_total())
        return total
        
    func (receiver) confirm() {
        // 注文を確定する
        if not receiver.can_be_confirmed():
            raise OrderCannotBeConfirmedException(// 注文を確定できません)
        receiver.status = OrderStatus.CONFIRMED
        
    func (receiver) can_be_confirmed() {
        // 注文が確定可能かどうか
        return (receiver.status == OrderStatus.PENDING and 
                len(receiver.order_lines) > 0 and
                receiver.customer.is_active())

type OrderLine struct {
    // 注文明細
    func (receiver) __init__(product: Product, quantity: Quantity, unit_price: Money) {
        receiver.product = product
        receiver.quantity = quantity
        receiver.unit_price = unit_price
        
    func (receiver) calculate_line_total() {
        // 明細の合計金額を計算
        return receiver.unit_price.multiply(receiver.quantity.value)
```

### 境界づけられたコンテキスト（Bounded Context）
```go
package main

# 販売コンテキスト
type SalesContext struct {
    // 販売に関する境界づけられたコンテキスト
    
    type Customer struct {
        // 販売コンテキストでの顧客
        func (receiver) __init__(customer_id: CustomerId, name string, credit_limit: Money) {
            receiver.customer_id = customer_id
            receiver.name = name
            receiver.credit_limit = credit_limit
            
        func (receiver) can_place_order(order_amount: Money) {
            // 注文可能かどうか
            return order_amount.is_less_than_or_equal(receiver.credit_limit)
    
    type Order struct {
        // 販売コンテキストでの注文
        func (receiver) place_order(customer: Customer, items []OrderItem) {
            if not customer.can_place_order(receiver.calculate_total(items)):
                raise InsufficientCreditException()
            # 注文処理...

# 配送コンテキスト  
type ShippingContext struct {
    // 配送に関する境界づけられたコンテキスト
    
    type Customer struct {
        // 配送コンテキストでの顧客（異なる観点）
        func (receiver) __init__(customer_id: CustomerId, shipping_address: Address) {
            receiver.customer_id = customer_id  # 同じID、異なる属性
            receiver.shipping_address = shipping_address
            
        func (receiver) get_preferred_delivery_time() {
            // 希望配送時間を取得
            return receiver.shipping_preferences.preferred_time_slot
    
    type Shipment struct {
        // 配送
        func (receiver) __init__(order_id: OrderId, customer: Customer) {
            receiver.shipment_id = ShipmentId.generate()
            receiver.order_id = order_id
            receiver.delivery_address = customer.shipping_address
            receiver.status = ShipmentStatus.PREPARING
```

## 戦略的設計

### 1. ドメインマップ
```go
package main

type DomainMap struct {
    // ドメイン全体のマップ定義
    
    bounded_contexts = {
        // sales: {
            // description: // 販売・注文管理,
            // team: // Sales Team,
            // entities: [// Customer, // Order, // Product],
            // relationships: {
                // shipping: // Customer Supplier,  # 顧客データを提供
                // inventory: // Conformist         # 在庫情報に従う
            }
        },
        // shipping: {
            // description: // 配送・物流管理, 
            // team: // Logistics Team,
            // entities: [// Shipment, // Customer, // Address],
            // relationships: {
                // sales: // Customer,              # 顧客データを受け取る
            }
        },
        // inventory: {
            // description: // 在庫管理,
            // team: // Inventory Team, 
            // entities: [// Product, // Stock, // Warehouse],
            // relationships: {
                // sales: // Supplier               # 在庫情報を提供
            }
        }
    }

# コンテキスト間の関係パターン
type ContextRelationship struct {
    // コンテキスト間の関係
    
    # パートナーシップ
    type Partnership struct {
        // 相互に協力する関係
        // TODO: implement
        
    # 共有カーネル
    type SharedKernel struct {
        // 共通のコードを共有
        // TODO: implement
        
    # 顧客・供給者
    type CustomerSupplier struct {
        // 上流・下流の関係
        func (receiver) __init__(upstream_context string, downstream_context string) {
            receiver.upstream = upstream_context
            receiver.downstream = downstream_context
            
    # 準拠者
    type Conformist struct {
        // 上流コンテキストに完全に従う
        // TODO: implement
```

### 2. コンテキストマッピング
```go
package main

# Anti-Corruption Layer（腐敗防止層）
type LegacySystemAdapter struct {
    // レガシーシステムとの間の腐敗防止層
    
    func (receiver) __init__(legacy_customer_service) {
        receiver.legacy_service = legacy_customer_service
        
    func (receiver) get_customer(customer_id: CustomerId) {
        // レガシーデータをドメインモデルに変換
        legacy_data = receiver.legacy_service.getCustomerData(customer_id.value)
        
        # レガシーデータ構造からドメインモデルへの変換
        return Customer(
            customer_id=CustomerId(legacy_data['cust_id']),
            name=legacy_data['cust_name'],
            email=Email(legacy_data['email_addr']),
            # レガシーシステムの複雑なロジックをシンプルなドメインロジックに変換
            status=receiver._convert_legacy_status(legacy_data['status_code'])
        )
        
    func (receiver) _convert_legacy_status(legacy_status string) {
        // レガシーステータスをドメインステータスに変換
        status_mapping = {
            'ACT': CustomerStatus.ACTIVE,
            'INA': CustomerStatus.INACTIVE, 
            'SUS': CustomerStatus.SUSPENDED
        }
        return status_mapping.get(legacy_status, CustomerStatus.UNKNOWN)

# Published Language（公開言語）
type CustomerPublishedLanguage struct {
    // 顧客情報の公開言語定義
    
        type CustomerDTO struct {
        // 外部システム向けの顧客データ転送オブジェクト
        customer_id string
        name string
        email string
        status string
        registration_date string
        
    func (receiver) to_published_language(customer: Customer) {
        // ドメインモデルから公開言語へ変換
        return receiver.CustomerDTO(
            customer_id=customer.customer_id.value,
            name=customer.name,
            email=customer.email.value,
            status=customer.status.value,
            registration_date=customer.registration_date.isoformat()
        )
```

## 戦術的設計

### 1. エンティティ（Entity）
```go
package main

type Customer struct {
    // 顧客エンティティ - 一意な識別子を持つ
    
    func (receiver) __init__(customer_id: CustomerId, name string, email: Email) {
        receiver._customer_id = customer_id  # 不変の識別子
        receiver._name = name
        receiver._email = email
        receiver._registration_date = datetime.now()
        receiver._domain_events = []
        
        func (receiver) customer_id() {
        // 顧客ID（不変）
        return receiver._customer_id
        
    func (receiver) change_email(new_email: Email) {
        // メールアドレス変更
        if receiver._email != new_email:
            old_email = receiver._email
            receiver._email = new_email
            
            # ドメインイベント発行
            receiver._domain_events.append(
                CustomerEmailChangedEvent(receiver._customer_id, old_email, new_email)
            )
            
    func (receiver) __eq__(other) {
        // エンティティの同一性は識別子で判断
        if not isinstance(other, Customer):
            return false
        return receiver._customer_id == other._customer_id
        
    func (receiver) __hash__() {
        return hash(receiver._customer_id)

type CustomerId struct {
    // 顧客ID値オブジェクト
    func (receiver) __init__(value string) {
        if not value or not value.strip():
            raise ValueError(// 顧客IDは必須です)
        receiver._value = value.strip()
        
        func (receiver) value() {
        return receiver._value
        
    func (receiver) __eq__(other) {
        return isinstance(other, CustomerId) and receiver._value == other._value
        
    func (receiver) __hash__() {
        return hash(receiver._value)
```

### 2. 値オブジェクト（Value Object）
```go
package main

type Money struct {
    // 金額値オブジェクト - 不変
    
    func (receiver) __init__(amount: Decimal, currency string = // JPY) {
        if amount < 0:
            raise ValueError(// 金額は0以上である必要があります)
        receiver._amount = amount
        receiver._currency = currency
        
        func (receiver) amount() {
        return receiver._amount
        
        func (receiver) currency() {
        return receiver._currency
        
    func (receiver) add(other: 'Money') {
        // 加算
        if receiver._currency != other._currency:
            raise ValueError(// 異なる通貨同士は計算できません)
        return Money(receiver._amount + other._amount, receiver._currency)
        
    func (receiver) multiply(multiplier int) {
        // 乗算
        return Money(receiver._amount * multiplier, receiver._currency)
        
    func (receiver) is_greater_than(other: 'Money') {
        // 比較
        receiver._ensure_same_currency(other)
        return receiver._amount > other._amount
        
    func (receiver) _ensure_same_currency(other: 'Money') {
        if receiver._currency != other._currency:
            raise ValueError(// 異なる通貨同士は比較できません)
            
    func (receiver) __eq__(other) {
        return (isinstance(other, Money) and 
                receiver._amount == other._amount and 
                receiver._currency == other._currency)
                
    func (receiver) __hash__() {
        return hash((receiver._amount, receiver._currency))
        
        func zero(cls, currency string = // JPY) {
        // ゼロ金額
        return cls(Decimal('0'), currency)

type Email struct {
    // メールアドレス値オブジェクト
    
    func (receiver) __init__(value string) {
        if not receiver._is_valid_email(value):
            raise ValueError(f// 無効なメールアドレス: {value})
        receiver._value = value.lower()
        
        func (receiver) value() {
        return receiver._value
        
    func (receiver) _is_valid_email(email string) {
        // メールアドレスのバリデーション
                pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return re.match(pattern, email) is not nil
        
    func (receiver) __eq__(other) {
        return isinstance(other, Email) and receiver._value == other._value
        
    func (receiver) __hash__() {
        return hash(receiver._value)
```

### 3. 集約（Aggregate）
```go
package main

type Order struct {
    // 注文集約ルート
    
    func (receiver) __init__(customer_id: CustomerId) {
        receiver._order_id = OrderId.generate()
        receiver._customer_id = customer_id
        receiver._order_lines = []
        receiver._status = OrderStatus.DRAFT
        receiver._order_date = datetime.now()
        receiver._domain_events = []
        
    func (receiver) add_order_line(product: Product, quantity int, unit_price: Money) {
        // 注文明細追加
        if receiver._status != OrderStatus.DRAFT:
            raise InvalidOperationException(// 確定済みの注文は変更できません)
            
        # ビジネスルール: 同じ商品の重複チェック
        existing_line = receiver._find_order_line_by_product(product.product_id)
        if existing_line:
            existing_line.change_quantity(existing_line.quantity + quantity)
        else:
            new_line = OrderLine(product, quantity, unit_price)
            receiver._order_lines.append(new_line)
            
    func (receiver) remove_order_line(product_id: ProductId) {
        // 注文明細削除
        if receiver._status != OrderStatus.DRAFT:
            raise InvalidOperationException(// 確定済みの注文は変更できません)
            
        receiver._order_lines = [line for line in receiver._order_lines 
                           if line.product_id != product_id]
                           
    func (receiver) confirm() {
        // 注文確定
        receiver._validate_for_confirmation()
        
        receiver._status = OrderStatus.CONFIRMED
        
        # ドメインイベント発行
        receiver._domain_events.append(
            OrderConfirmedEvent(
                order_id=receiver._order_id,
                customer_id=receiver._customer_id,
                total_amount=receiver.calculate_total_amount(),
                order_lines=receiver._order_lines.copy()
            )
        )
        
    func (receiver) _validate_for_confirmation() {
        // 確定時のビジネスルール検証
        if not receiver._order_lines:
            raise DomainException(// 注文明細がありません)
            
        if receiver.calculate_total_amount().amount <= 0:
            raise DomainException(// 注文金額は0より大きい必要があります)
            
    func (receiver) calculate_total_amount() {
        // 合計金額計算
        total = Money.zero()
        for line in receiver._order_lines:
            total = total.add(line.calculate_line_total())
        return total
        
    func (receiver) _find_order_line_by_product(product_id: ProductId) {
        // 商品IDで注文明細を検索
        for line in receiver._order_lines:
            if line.product_id == product_id:
                return line
        return nil
        
        func (receiver) domain_events() {
        // ドメインイベント取得
        return receiver._domain_events.copy()
        
    func (receiver) clear_domain_events() {
        // ドメインイベントクリア
        receiver._domain_events.clear()

type OrderLine struct {
    // 注文明細エンティティ（集約内部）
    
    func (receiver) __init__(product: Product, quantity int, unit_price: Money) {
        receiver._product_id = product.product_id
        receiver._product_name = product.name
        receiver._quantity = quantity
        receiver._unit_price = unit_price
        
        if quantity <= 0:
            raise ValueError(// 数量は1以上である必要があります)
            
    func (receiver) change_quantity(new_quantity int) {
        // 数量変更
        if new_quantity <= 0:
            raise ValueError(// 数量は1以上である必要があります)
        receiver._quantity = new_quantity
        
    func (receiver) calculate_line_total() {
        // 明細合計金額
        return receiver._unit_price.multiply(receiver._quantity)
        
        func (receiver) product_id() {
        return receiver._product_id
        
        func (receiver) quantity() {
        return receiver._quantity
```

### 4. ドメインサービス（Domain Service）
```go
package main

type OrderDomainService struct {
    // 注文に関するドメインサービス
    
    def __init__(self, customer_repository: CustomerRepository,
                 product_repository: ProductRepository):
        receiver._customer_repository = customer_repository
        receiver._product_repository = product_repository
        
    def can_customer_place_order(self, customer_id: CustomerId, 
                               order_amount: Money) -> bool:
        // 顧客が注文可能かどうかの判定
        customer = receiver._customer_repository.find_by_id(customer_id)
        if not customer:
            return false
            
        # 複数のビジネスルールを組み合わせた複雑な判定
        return (customer.is_active() and 
                customer.has_sufficient_credit_limit(order_amount) and
                not customer.has_overdue_payments())
                
    def calculate_shipping_cost(self, order: Order, 
                              shipping_address: Address) -> Money:
        // 配送料計算
        base_cost = Money(Decimal('500'))  # 基本料金
        
        # 重量ベースの計算
        total_weight = receiver._calculate_total_weight(order)
        weight_cost = receiver._calculate_weight_cost(total_weight)
        
        # 距離ベースの計算
        distance_cost = receiver._calculate_distance_cost(shipping_address)
        
        return base_cost.add(weight_cost).add(distance_cost)
        
    func (receiver) _calculate_total_weight(order: Order) {
        // 注文の総重量計算
        total_weight = Weight.zero()
        for line in order.order_lines:
            product = receiver._product_repository.find_by_id(line.product_id)
            line_weight = product.weight.multiply(line.quantity)
            total_weight = total_weight.add(line_weight)
        return total_weight

type ProductCatalogService struct {
    // 商品カタログサービス
    
    func (receiver) __init__(product_repository: ProductRepository) {
        receiver._product_repository = product_repository
        
    def is_product_available_for_order(self, product_id: ProductId, 
                                     quantity int) -> bool:
        // 商品が注文可能かどうか
        product = receiver._product_repository.find_by_id(product_id)
        if not product:
            return false
            
        return (product.is_active() and 
                product.has_sufficient_stock(quantity) and
                not product.is_discontinued())
```

### 5. リポジトリ（Repository）
```go
package main


type CustomerRepository struct {
    // 顧客リポジトリインターフェース
    
        func (receiver) find_by_id(customer_id: CustomerId) {
        // IDで顧客を検索
        // TODO: implement
        
        func (receiver) find_by_email(email: Email) {
        // メールアドレスで顧客を検索
        // TODO: implement
        
        func (receiver) save(customer: Customer) {
        // 顧客を保存
        // TODO: implement
        
        func (receiver) delete(customer: Customer) {
        // 顧客を削除
        // TODO: implement

type SqlCustomerRepository struct {
    // SQL実装の顧客リポジトリ
    
    func (receiver) __init__(db_session) {
        receiver._db_session = db_session
        
    func (receiver) find_by_id(customer_id: CustomerId) {
        // IDで顧客を検索
        query = // 
            SELECT customer_id, name, email, registration_date, status
            FROM customers 
            WHERE customer_id = %s
        
        
        result = receiver._db_session.execute(query, (customer_id.value,)).fetchone()
        if not result:
            return nil
            
        return receiver._map_to_domain(result)
        
    func (receiver) save(customer: Customer) {
        // 顧客を保存
        # 既存顧客の確認
        existing = receiver.find_by_id(customer.customer_id)
        
        if existing:
            receiver._update_customer(customer)
        else:
            receiver._insert_customer(customer)
            
        # ドメインイベントの処理
        receiver._publish_domain_events(customer)
        
    func (receiver) _map_to_domain(db_record) {
        // データベースレコードからドメインオブジェクトへの変換
        return Customer(
            customer_id=CustomerId(db_record['customer_id']),
            name=db_record['name'],
            email=Email(db_record['email'])
        )
        
    func (receiver) _insert_customer(customer: Customer) {
        // 新規顧客挿入
        query = // 
            INSERT INTO customers (customer_id, name, email, registration_date, status)
            VALUES (%s, %s, %s, %s, %s)
        
        
        receiver._db_session.execute(query, (
            customer.customer_id.value,
            customer.name,
            customer.email.value,
            customer.registration_date,
            customer.status.value
        ))
        
    func (receiver) _update_customer(customer: Customer) {
        // 既存顧客更新
        query = // 
            UPDATE customers 
            SET name = %s, email = %s, status = %s
            WHERE customer_id = %s
        
        
        receiver._db_session.execute(query, (
            customer.name,
            customer.email.value,
            customer.status.value,
            customer.customer_id.value
        ))
```

## アプリケーション層

### アプリケーションサービス
```go
package main

type OrderApplicationService struct {
    // 注文アプリケーションサービス
    
    def __init__(self, 
                 order_repository: OrderRepository,
                 customer_repository: CustomerRepository,
                 product_repository: ProductRepository,
                 order_domain_service: OrderDomainService,
                 unit_of_work: UnitOfWork):
        receiver._order_repository = order_repository
        receiver._customer_repository = customer_repository
        receiver._product_repository = product_repository
        receiver._order_domain_service = order_domain_service
        receiver._unit_of_work = unit_of_work
        
    func (receiver) create_order(command: CreateOrderCommand) {
        // 注文作成
        with receiver._unit_of_work:
            # ドメインオブジェクト取得
            customer = receiver._customer_repository.find_by_id(
                CustomerId(command.customer_id)
            )
            if not customer:
                raise CustomerNotFoundException()
                
            # 新しい注文作成
            order = Order(customer.customer_id)
            
            # 注文明細追加
            for item in command.order_items:
                product = receiver._product_repository.find_by_id(
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
            if not receiver._order_domain_service.can_customer_place_order(
                customer.customer_id, order.calculate_total_amount()
            ):
                raise CustomerCannotPlaceOrderException()
                
            # 注文保存
            receiver._order_repository.save(order)
            
            return order.order_id
            
    func (receiver) confirm_order(command: ConfirmOrderCommand) {
        // 注文確定
        with receiver._unit_of_work:
            order = receiver._order_repository.find_by_id(
                OrderId(command.order_id)
            )
            if not order:
                raise OrderNotFoundException()
                
            order.confirm()
            receiver._order_repository.save(order)

type CreateOrderCommand struct {
    // 注文作成コマンド
    customer_id string
    order_items []'OrderItemCommand'
    
type OrderItemCommand struct {
    // 注文明細コマンド
    product_id string
    quantity int
    unit_price string  # Decimal文字列
```

## ドメインイベント

### イベント定義と発行
```go
package main

type DomainEvent struct {
    // ドメインイベント基底クラス
    func (receiver) __init__() {
        receiver.occurred_at = datetime.utcnow()
        receiver.event_id = str(uuid.uuid4())

type OrderConfirmedEvent struct {
    // 注文確定イベント
    def __init__(self, order_id: OrderId, customer_id: CustomerId, 
                 total_amount: Money, order_lines []OrderLine):
        super().__init__()
        receiver.order_id = order_id
        receiver.customer_id = customer_id
        receiver.total_amount = total_amount
        receiver.order_lines = order_lines

type CustomerEmailChangedEvent struct {
    // 顧客メールアドレス変更イベント
    func (receiver) __init__(customer_id: CustomerId, old_email: Email, new_email: Email) {
        super().__init__()
        receiver.customer_id = customer_id
        receiver.old_email = old_email
        receiver.new_email = new_email

# イベントハンドラー
type OrderConfirmedEventHandler struct {
    // 注文確定イベントハンドラー
    
    def __init__(self, inventory_service: InventoryService,
                 notification_service: NotificationService):
        receiver._inventory_service = inventory_service
        receiver._notification_service = notification_service
        
    func (receiver) handle(event: OrderConfirmedEvent) {
        // イベント処理
        # 在庫予約
        for line in event.order_lines:
            receiver._inventory_service.reserve_stock(
                line.product_id, line.quantity
            )
            
        # 顧客通知
        receiver._notification_service.send_order_confirmation(
            event.customer_id, event.order_id
        )

# イベント発行
type DomainEventPublisher struct {
    // ドメインイベント発行者
    
    func (receiver) __init__() {
        receiver._handlers = defaultdict(list)
        
    func (receiver) subscribe(event_type: Type[DomainEvent], handler) {
        // イベントハンドラー登録
        receiver._handlers[event_type].append(handler)
        
    func (receiver) publish(event: DomainEvent) {
        // イベント発行
        event_type = type(event)
        for handler in receiver._handlers[event_type]:
            try:
                handler.handle(event)
            except Exception as e:
                # エラーログ記録
                logger.error(f// Event handler failed: {e})
```

## 実装時の考慮事項

### 1. 永続化の無知（Persistence Ignorance）
```go
package main

# ドメインモデルはデータベースの詳細を知らない
type Customer struct {
    // 永続化技術に依存しないドメインモデル
    
    func (receiver) __init__(customer_id: CustomerId, name string, email: Email) {
        # データベースのアノテーションや依存性はない
        receiver._customer_id = customer_id
        receiver._name = name
        receiver._email = email
        
    # ビジネスロジックのみに集中
    func (receiver) change_email(new_email: Email) {
        if receiver._email != new_email:
            receiver._email = new_email
            # ドメインイベント発行

# インフラストラクチャ層で永続化の詳細を処理
type CustomerEntity struct {
    // ORMエンティティ
    __tablename__ = 'customers'
    
    id = Column(String, primary_key=true)
    name = Column(String, nullable=false)
    email = Column(String, nullable=false)
    registration_date = Column(DateTime)
    
    func (receiver) to_domain() {
        // ドメインモデルへの変換
        return Customer(
            customer_id=CustomerId(receiver.id),
            name=receiver.name,
            email=Email(receiver.email)
        )
        
        func from_domain(cls, customer: Customer) {
        // ドメインモデルからの変換
        return cls(
            id=customer.customer_id.value,
            name=customer.name,
            email=customer.email.value
        )
```

### 2. 集約設計のガイドライン
```go
package main

# 小さな集約を設計
type Order struct {
    // 注文集約 - 注文と注文明細のみを含む
    
    func (receiver) __init__(customer_id: CustomerId) {
        receiver._order_id = OrderId.generate()
        receiver._customer_id = customer_id  # 参照のみ、Customerオブジェクトは含まない
        receiver._order_lines = []
        
    func (receiver) add_order_line(product_id: ProductId, quantity int, unit_price: Money) {
        # Productオブジェクトは集約に含めず、IDのみ保持
        new_line = OrderLine(product_id, quantity, unit_price)
        receiver._order_lines.append(new_line)

# IDによる参照
type OrderLine struct {
    // 注文明細
    
    func (receiver) __init__(product_id: ProductId, quantity int, unit_price: Money) {
        receiver._product_id = product_id  # Product集約への参照
        receiver._quantity = quantity
        receiver._unit_price = unit_price
        
    # 商品情報が必要な場合はリポジトリから取得
    func (receiver) get_product_name(product_repository: ProductRepository) {
        product = product_repository.find_by_id(receiver._product_id)
        return product.name if product else // Unknown Product
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
