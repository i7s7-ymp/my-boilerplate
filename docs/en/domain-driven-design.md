# Domain-Driven Design (DDD)

## Overview
Domain-Driven Design (DDD) is a strategic design approach for complex software. It centers on business domain expertise and facilitates collaboration between domain experts and developers using a common language (Ubiquitous Language) to build software with high business value.

## Core Concepts

### Ubiquitous Language
```go
package domain

import (
    "errors"
    "time"
)

// Order represents an order in business terms
type Order struct {
    customer    *Customer
    orderLines  []OrderLine
    status      OrderStatus
    orderDate   time.Time
}

// NewOrder creates a new order
func NewOrder(customer *Customer, orderLines []OrderLine) *Order {
    return &Order{
        customer:   customer,
        orderLines: orderLines,
        status:     OrderStatusPending,
        orderDate:  time.Now(),
    }
}

// CalculateTotalAmount calculates total amount - domain logic
func (o *Order) CalculateTotalAmount() Money {
    total := ZeroMoney()
    for _, line := range o.orderLines {
        total = total.Add(line.CalculateLineTotal())
    }
    return total
}

// Confirm confirms the order
func (o *Order) Confirm() error {
    if !o.CanBeConfirmed() {
        return errors.New("order cannot be confirmed")
    }
    o.status = OrderStatusConfirmed
    return nil
}

// CanBeConfirmed checks if order can be confirmed
func (o *Order) CanBeConfirmed() bool {
    return o.status == OrderStatusPending &&
           len(o.orderLines) > 0 &&
           o.customer.IsActive()
}

// OrderLine represents order line item
type OrderLine struct {
    product   *Product
    quantity  Quantity
    unitPrice Money
}

// NewOrderLine creates a new order line
func NewOrderLine(product *Product, quantity Quantity, unitPrice Money) *OrderLine {
    return &OrderLine{
        product:   product,
        quantity:  quantity,
        unitPrice: unitPrice,
    }
}

// CalculateLineTotal calculates line total amount
func (ol *OrderLine) CalculateLineTotal() Money {
    return ol.unitPrice.Multiply(ol.quantity.Value())
}

// OrderStatus represents order status
type OrderStatus int

const (
    OrderStatusPending OrderStatus = iota
    OrderStatusConfirmed
    OrderStatusShipped
    OrderStatusDelivered
    OrderStatusCancelled
)
```

### Bounded Context
```go
package sales

// SalesContext - Sales bounded context
import (
    "errors"
)

// Customer in sales context
type Customer struct {
    customerID  CustomerID
    name        string
    creditLimit Money
}

// NewCustomer creates a customer in sales context
func NewCustomer(customerID CustomerID, name string, creditLimit Money) *Customer {
    return &Customer{
        customerID:  customerID,
        name:        name,
        creditLimit: creditLimit,
    }
}

// CanPlaceOrder checks if customer can place order
func (c *Customer) CanPlaceOrder(orderAmount Money) bool {
    return orderAmount.IsLessThanOrEqual(c.creditLimit)
}

// Order in sales context
type Order struct {
    orderID    OrderID
    customerID CustomerID
    items      []OrderItem
    status     OrderStatus
}

// PlaceOrder places an order
func (o *Order) PlaceOrder(customer *Customer, items []OrderItem) error {
    totalAmount := o.calculateTotal(items)
    if !customer.CanPlaceOrder(totalAmount) {
        return errors.New("insufficient credit limit")
    }
    // Order processing logic...
    return nil
}

func (o *Order) calculateTotal(items []OrderItem) Money {
    total := ZeroMoney()
    for _, item := range items {
        total = total.Add(item.Price.Multiply(item.Quantity))
    }
    return total
}

// OrderItem represents an item in order
type OrderItem struct {
    ProductID ProductID
    Quantity  int
    Price     Money
}
```

```go
package shipping

// ShippingContext - Shipping bounded context
import "time"

// Customer in shipping context (different perspective)
type Customer struct {
    customerID       CustomerID // Same ID, different attributes
    shippingAddress  Address
    deliveryPrefs    DeliveryPreferences
}

// NewShippingCustomer creates a customer in shipping context
func NewShippingCustomer(customerID CustomerID, address Address) *Customer {
    return &Customer{
        customerID:      customerID,
        shippingAddress: address,
    }
}

// GetPreferredDeliveryTime gets preferred delivery time
func (c *Customer) GetPreferredDeliveryTime() time.Time {
    return c.deliveryPrefs.PreferredTimeSlot
}

// Shipment represents a shipment
type Shipment struct {
    shipmentID      ShipmentID
    orderID         OrderID
    deliveryAddress Address
    status          ShipmentStatus
}

// NewShipment creates a new shipment
func NewShipment(orderID OrderID, customer *Customer) *Shipment {
    return &Shipment{
        shipmentID:      GenerateShipmentID(),
        orderID:         orderID,
        deliveryAddress: customer.shippingAddress,
        status:          ShipmentStatusPreparing,
    }
}

// ShipmentStatus represents shipment status
type ShipmentStatus int

const (
    ShipmentStatusPreparing ShipmentStatus = iota
    ShipmentStatusShipped
    ShipmentStatusInTransit
    ShipmentStatusDelivered
)

// Address represents delivery address
type Address struct {
    Street   string
    City     string
    ZipCode  string
    Country  string
}

// DeliveryPreferences represents delivery preferences
type DeliveryPreferences struct {
    PreferredTimeSlot time.Time
    SpecialInstructions string
}
```

## Strategic Design

### 1. Domain Map
```go
package strategy

// DomainMap defines the overall domain map
type DomainMap struct {
    BoundedContexts map[string]BoundedContext
}

// BoundedContext represents a bounded context
type BoundedContext struct {
    Name          string
    Description   string
    Team          string
    Entities      []string
    Relationships map[string]RelationshipType
}

// RelationshipType represents context relationship patterns
type RelationshipType int

const (
    Partnership RelationshipType = iota
    SharedKernel
    CustomerSupplier
    Conformist
    AntiCorruptionLayer
)

// NewDomainMap creates a new domain map
func NewDomainMap() *DomainMap {
    return &DomainMap{
        BoundedContexts: map[string]BoundedContext{
            "sales": {
                Name:        "sales",
                Description: "Sales and order management",
                Team:        "Sales Team",
                Entities:    []string{"Customer", "Order", "Product"},
                Relationships: map[string]RelationshipType{
                    "shipping":  CustomerSupplier, // Provides customer data
                    "inventory": Conformist,       // Follows inventory information
                },
            },
            "shipping": {
                Name:        "shipping",
                Description: "Shipping and logistics management",
                Team:        "Logistics Team",
                Entities:    []string{"Shipment", "Customer", "Address"},
                Relationships: map[string]RelationshipType{
                    "sales": CustomerSupplier, // Receives customer data
                },
            },
            "inventory": {
                Name:        "inventory",
                Description: "Inventory management",
                Team:        "Inventory Team",
                Entities:    []string{"Product", "Stock", "Warehouse"},
                Relationships: map[string]RelationshipType{
                    "sales": CustomerSupplier, // Provides inventory information
                },
            },
        },
    }
}
```

### 2. Context Mapping
```go
package integration

import (
    "fmt"
    "regexp"
)

// LegacySystemAdapter - Anti-Corruption Layer
type LegacySystemAdapter struct {
    legacyService LegacyCustomerService
}

// NewLegacySystemAdapter creates a new adapter
func NewLegacySystemAdapter(legacyService LegacyCustomerService) *LegacySystemAdapter {
    return &LegacySystemAdapter{
        legacyService: legacyService,
    }
}

// GetCustomer converts legacy data to domain model
func (a *LegacySystemAdapter) GetCustomer(customerID CustomerID) (*Customer, error) {
    legacyData, err := a.legacyService.GetCustomerData(customerID.Value())
    if err != nil {
        return nil, err
    }

    // Convert legacy data structure to domain model
    email, err := NewEmail(legacyData.EmailAddr)
    if err != nil {
        return nil, err
    }

    return NewCustomer(
        NewCustomerID(legacyData.CustID),
        legacyData.CustName,
        email,
        a.convertLegacyStatus(legacyData.StatusCode),
    ), nil
}

// convertLegacyStatus converts legacy status to domain status
func (a *LegacySystemAdapter) convertLegacyStatus(legacyStatus string) CustomerStatus {
    statusMapping := map[string]CustomerStatus{
        "ACT": CustomerStatusActive,
        "INA": CustomerStatusInactive,
        "SUS": CustomerStatusSuspended,
    }
    
    if status, exists := statusMapping[legacyStatus]; exists {
        return status
    }
    return CustomerStatusUnknown
}

// LegacyCustomerData represents legacy system data structure
type LegacyCustomerData struct {
    CustID     string
    CustName   string
    EmailAddr  string
    StatusCode string
}

// LegacyCustomerService interface for legacy system
type LegacyCustomerService interface {
    GetCustomerData(customerID string) (*LegacyCustomerData, error)
}

// CustomerPublishedLanguage - Published Language for customer information
type CustomerPublishedLanguage struct{}

// CustomerDTO - Data transfer object for external systems
type CustomerDTO struct {
    CustomerID       string `json:"customer_id"`
    Name            string `json:"name"`
    Email           string `json:"email"`
    Status          string `json:"status"`
    RegistrationDate string `json:"registration_date"`
}

// ToPublishedLanguage converts domain model to published language
func (cpl *CustomerPublishedLanguage) ToPublishedLanguage(customer *Customer) *CustomerDTO {
    return &CustomerDTO{
        CustomerID:       customer.CustomerID().Value(),
        Name:            customer.Name(),
        Email:           customer.Email().Value(),
        Status:          customer.Status().String(),
        RegistrationDate: customer.RegistrationDate().Format("2006-01-02T15:04:05Z"),
    }
}
```

## Tactical Design

### 1. Entity
```go
package domain

import (
    "time"
    "github.com/google/uuid"
)

// Customer entity - has unique identity
type Customer struct {
    customerID       CustomerID    // Immutable identifier
    name            string
    email           Email
    registrationDate time.Time
    status          CustomerStatus
    domainEvents    []DomainEvent
}

// NewCustomer creates a new customer
func NewCustomer(customerID CustomerID, name string, email Email) *Customer {
    return &Customer{
        customerID:       customerID,
        name:            name,
        email:           email,
        registrationDate: time.Now(),
        status:          CustomerStatusActive,
        domainEvents:    make([]DomainEvent, 0),
    }
}

// CustomerID returns customer ID (immutable)
func (c *Customer) CustomerID() CustomerID {
    return c.customerID
}

// ChangeEmail changes email address
func (c *Customer) ChangeEmail(newEmail Email) {
    if c.email != newEmail {
        oldEmail := c.email
        c.email = newEmail
        
        // Raise domain event
        event := NewCustomerEmailChangedEvent(c.customerID, oldEmail, newEmail)
        c.domainEvents = append(c.domainEvents, event)
    }
}

// Equals checks entity equality by identifier
func (c *Customer) Equals(other *Customer) bool {
    if other == nil {
        return false
    }
    return c.customerID.Equals(other.customerID)
}

// GetDomainEvents returns domain events
func (c *Customer) GetDomainEvents() []DomainEvent {
    return c.domainEvents
}

// ClearDomainEvents clears domain events
func (c *Customer) ClearDomainEvents() {
    c.domainEvents = make([]DomainEvent, 0)
}

// CustomerID value object
type CustomerID struct {
    value string
}

// NewCustomerID creates a new customer ID
func NewCustomerID(value string) CustomerID {
    if value == "" {
        panic("Customer ID is required")
    }
    return CustomerID{value: value}
}

// GenerateCustomerID generates a new customer ID
func GenerateCustomerID() CustomerID {
    return CustomerID{value: uuid.New().String()}
}

// Value returns the ID value
func (id CustomerID) Value() string {
    return id.value
}

// Equals checks ID equality
func (id CustomerID) Equals(other CustomerID) bool {
    return id.value == other.value
}

// CustomerStatus represents customer status
type CustomerStatus int

const (
    CustomerStatusActive CustomerStatus = iota
    CustomerStatusInactive
    CustomerStatusSuspended
    CustomerStatusUnknown
)

// String returns string representation
func (s CustomerStatus) String() string {
    switch s {
    case CustomerStatusActive:
        return "active"
    case CustomerStatusInactive:
        return "inactive"
    case CustomerStatusSuspended:
        return "suspended"
    default:
        return "unknown"
    }
}
```

### 2. Value Object
```go
package domain

import (
    "errors"
    "regexp"
    "github.com/shopspring/decimal"
)

// Money value object - immutable
type Money struct {
    amount   decimal.Decimal
    currency string
}

// NewMoney creates a new money value object
func NewMoney(amount decimal.Decimal, currency string) (Money, error) {
    if amount.IsNegative() {
        return Money{}, errors.New("amount must be non-negative")
    }
    return Money{
        amount:   amount,
        currency: currency,
    }, nil
}

// ZeroMoney creates zero money
func ZeroMoney() Money {
    return Money{
        amount:   decimal.Zero,
        currency: "JPY",
    }
}

// Amount returns the amount
func (m Money) Amount() decimal.Decimal {
    return m.amount
}

// Currency returns the currency
func (m Money) Currency() string {
    return m.currency
}

// Add adds two money values
func (m Money) Add(other Money) (Money, error) {
    if m.currency != other.currency {
        return Money{}, errors.New("cannot add different currencies")
    }
    return Money{
        amount:   m.amount.Add(other.amount),
        currency: m.currency,
    }, nil
}

// Multiply multiplies money by a factor
func (m Money) Multiply(multiplier int) Money {
    return Money{
        amount:   m.amount.Mul(decimal.NewFromInt(int64(multiplier))),
        currency: m.currency,
    }
}

// IsGreaterThan compares two money values
func (m Money) IsGreaterThan(other Money) (bool, error) {
    if err := m.ensureSameCurrency(other); err != nil {
        return false, err
    }
    return m.amount.GreaterThan(other.amount), nil
}

// IsLessThanOrEqual compares two money values
func (m Money) IsLessThanOrEqual(other Money) bool {
    if m.currency != other.currency {
        return false
    }
    return m.amount.LessThanOrEqual(other.amount)
}

// ensureSameCurrency ensures same currency for operations
func (m Money) ensureSameCurrency(other Money) error {
    if m.currency != other.currency {
        return errors.New("cannot compare different currencies")
    }
    return nil
}

// Equals checks money equality
func (m Money) Equals(other Money) bool {
    return m.amount.Equal(other.amount) && m.currency == other.currency
}

// Email value object
type Email struct {
    value string
}

// NewEmail creates a new email value object
func NewEmail(value string) (Email, error) {
    if !isValidEmail(value) {
        return Email{}, errors.New("invalid email address: " + value)
    }
    return Email{value: value}, nil
}

// Value returns email value
func (e Email) Value() string {
    return e.value
}

// isValidEmail validates email format
func isValidEmail(email string) bool {
    pattern := `^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`
    matched, _ := regexp.MatchString(pattern, email)
    return matched
}

// Equals checks email equality
func (e Email) Equals(other Email) bool {
    return e.value == other.value
}

// Quantity value object
type Quantity struct {
    value int
}

// NewQuantity creates a new quantity
func NewQuantity(value int) (Quantity, error) {
    if value < 0 {
        return Quantity{}, errors.New("quantity must be non-negative")
    }
    return Quantity{value: value}, nil
}

// Value returns quantity value
func (q Quantity) Value() int {
    return q.value
}

// Add adds quantities
func (q Quantity) Add(other Quantity) Quantity {
    return Quantity{value: q.value + other.value}
}

// Equals checks quantity equality
func (q Quantity) Equals(other Quantity) bool {
    return q.value == other.value
}
```

### 3. Aggregate
```go
package domain

import (
    "errors"
    "time"
)

// Order aggregate root
type Order struct {
    orderID      OrderID
    customerID   CustomerID
    orderLines   []OrderLine
    status       OrderStatus
    orderDate    time.Time
    domainEvents []DomainEvent
}

// NewOrder creates a new order
func NewOrder(customerID CustomerID) *Order {
    return &Order{
        orderID:      GenerateOrderID(),
        customerID:   customerID,
        orderLines:   make([]OrderLine, 0),
        status:       OrderStatusDraft,
        orderDate:    time.Now(),
        domainEvents: make([]DomainEvent, 0),
    }
}

// AddOrderLine adds an order line
func (o *Order) AddOrderLine(product *Product, quantity int, unitPrice Money) error {
    if o.status != OrderStatusDraft {
        return errors.New("confirmed orders cannot be modified")
    }
    
    // Business rule: check for duplicate products
    existingLine := o.findOrderLineByProduct(product.ProductID())
    if existingLine != nil {
        return existingLine.ChangeQuantity(existingLine.Quantity() + quantity)
    }
    
    newLine, err := NewOrderLine(product, quantity, unitPrice)
    if err != nil {
        return err
    }
    
    o.orderLines = append(o.orderLines, *newLine)
    return nil
}

// RemoveOrderLine removes an order line
func (o *Order) RemoveOrderLine(productID ProductID) error {
    if o.status != OrderStatusDraft {
        return errors.New("confirmed orders cannot be modified")
    }
    
    for i, line := range o.orderLines {
        if line.ProductID().Equals(productID) {
            o.orderLines = append(o.orderLines[:i], o.orderLines[i+1:]...)
            break
        }
    }
    return nil
}

// Confirm confirms the order
func (o *Order) Confirm() error {
    if err := o.validateForConfirmation(); err != nil {
        return err
    }
    
    o.status = OrderStatusConfirmed
    
    // Raise domain event
    event := NewOrderConfirmedEvent(
        o.orderID,
        o.customerID,
        o.CalculateTotalAmount(),
        o.orderLines,
    )
    o.domainEvents = append(o.domainEvents, event)
    
    return nil
}

// validateForConfirmation validates order for confirmation
func (o *Order) validateForConfirmation() error {
    if len(o.orderLines) == 0 {
        return errors.New("order must have at least one line item")
    }
    
    totalAmount := o.CalculateTotalAmount()
    if totalAmount.Amount().IsZero() {
        return errors.New("order total must be greater than zero")
    }
    
    return nil
}

// CalculateTotalAmount calculates total order amount
func (o *Order) CalculateTotalAmount() Money {
    total := ZeroMoney()
    for _, line := range o.orderLines {
        lineTotal := line.CalculateLineTotal()
        if newTotal, err := total.Add(lineTotal); err == nil {
            total = newTotal
        }
    }
    return total
}

// findOrderLineByProduct finds order line by product ID
func (o *Order) findOrderLineByProduct(productID ProductID) *OrderLine {
    for i := range o.orderLines {
        if o.orderLines[i].ProductID().Equals(productID) {
            return &o.orderLines[i]
        }
    }
    return nil
}

// OrderID returns order ID
func (o *Order) OrderID() OrderID {
    return o.orderID
}

// CustomerID returns customer ID
func (o *Order) CustomerID() CustomerID {
    return o.customerID
}

// GetDomainEvents returns domain events
func (o *Order) GetDomainEvents() []DomainEvent {
    return o.domainEvents
}

// ClearDomainEvents clears domain events
func (o *Order) ClearDomainEvents() {
    o.domainEvents = make([]DomainEvent, 0)
}

// OrderID value object
type OrderID struct {
    value string
}

// NewOrderID creates a new order ID
func NewOrderID(value string) OrderID {
    return OrderID{value: value}
}

// GenerateOrderID generates a new order ID
func GenerateOrderID() OrderID {
    return OrderID{value: uuid.New().String()}
}

// Value returns order ID value
func (id OrderID) Value() string {
    return id.value
}

// Equals checks order ID equality
func (id OrderID) Equals(other OrderID) bool {
    return id.value == other.value
}
```

### 4. Domain Service
```go
package domain

import "errors"

// TransferService domain service for money transfer
type TransferService struct {
    feePolicy FeePolicy
}

// NewTransferService creates a transfer service
func NewTransferService(feePolicy FeePolicy) *TransferService {
    return &TransferService{
        feePolicy: feePolicy,
    }
}

// Transfer performs money transfer between accounts
func (s *TransferService) Transfer(from *Account, to *Account, amount Money) (*TransferResult, error) {
    // Domain logic that doesn't belong to either account
    if err := s.validateTransfer(from, to, amount); err != nil {
        return nil, err
    }
    
    // Calculate transfer fee
    fee := s.feePolicy.CalculateFee(amount, from.AccountType(), to.AccountType())
    totalAmount, err := amount.Add(fee)
    if err != nil {
        return nil, err
    }
    
    // Perform transfer
    if err := from.Withdraw(totalAmount); err != nil {
        return nil, err
    }
    
    if err := to.Deposit(amount); err != nil {
        // Rollback withdrawal
        from.Deposit(totalAmount)
        return nil, err
    }
    
    return &TransferResult{
        TransferID: GenerateTransferID(),
        Amount:     amount,
        Fee:        fee,
        FromAccount: from.AccountID(),
        ToAccount:   to.AccountID(),
    }, nil
}

// validateTransfer validates transfer operation
func (s *TransferService) validateTransfer(from *Account, to *Account, amount Money) error {
    if from.AccountID().Equals(to.AccountID()) {
        return errors.New("cannot transfer to the same account")
    }
    
    if !from.CanWithdraw(amount) {
        return errors.New("insufficient balance")
    }
    
    return nil
}

// FeePolicy interface for fee calculation
type FeePolicy interface {
    CalculateFee(amount Money, fromType AccountType, toType AccountType) Money
}

// TransferResult represents transfer result
type TransferResult struct {
    TransferID  TransferID
    Amount      Money
    Fee         Money
    FromAccount AccountID
    ToAccount   AccountID
}
```

### 5. Repository Pattern
```go
package domain

// CustomerRepository domain repository interface
type CustomerRepository interface {
    FindByID(customerID CustomerID) (*Customer, error)
    FindByEmail(email Email) (*Customer, error)
    Save(customer *Customer) error
    Delete(customerID CustomerID) error
    NextIdentity() CustomerID
}

// OrderRepository domain repository interface
type OrderRepository interface {
    FindByID(orderID OrderID) (*Order, error)
    FindByCustomerID(customerID CustomerID) ([]*Order, error)
    Save(order *Order) error
    Delete(orderID OrderID) error
    NextIdentity() OrderID
}

// ProductRepository domain repository interface
type ProductRepository interface {
    FindByID(productID ProductID) (*Product, error)
    FindByCategory(category ProductCategory) ([]*Product, error)
    Save(product *Product) error
    Delete(productID ProductID) error
    NextIdentity() ProductID
}

// Specification pattern for complex queries
type Specification interface {
    IsSatisfiedBy(candidate interface{}) bool
}

// CustomerSpecification for customer queries
type CustomerSpecification struct {
    status CustomerStatus
    minAge int
}

// NewCustomerSpecification creates customer specification
func NewCustomerSpecification(status CustomerStatus, minAge int) *CustomerSpecification {
    return &CustomerSpecification{
        status: status,
        minAge: minAge,
    }
}

// IsSatisfiedBy checks if customer satisfies specification
func (s *CustomerSpecification) IsSatisfiedBy(candidate interface{}) bool {
    customer, ok := candidate.(*Customer)
    if !ok {
        return false
    }
    
    return customer.Status() == s.status && customer.Age() >= s.minAge
}
```

## Benefits

### Clear Business Logic
- Business rules are explicitly expressed in code
- Domain expertise is preserved in the codebase
- Easier maintenance and extension

### Improved Communication
- Common language between technical and business teams
- Reduced misunderstandings
- Better requirement analysis

### Flexibility and Maintainability
- Domain logic is isolated from technical concerns
- Easy to test and modify
- Supports complex business scenarios

## Drawbacks

### Learning Curve
- Requires understanding of DDD concepts
- More complex than simple CRUD applications
- Need for domain modeling skills

### Initial Overhead
- Higher upfront development cost
- More complex architecture
- Requires experienced developers

### Over-engineering Risk
- May be excessive for simple domains
- Can lead to unnecessary complexity
- Need to balance effort with business value

## When to Use DDD

### Suitable Scenarios
- **Complex business domain**: Rich business logic and rules
- **Large development teams**: Need for clear boundaries
- **Long-term projects**: Benefits justify initial investment
- **Domain expert availability**: Close collaboration possible

### Not Suitable Scenarios
- **Simple CRUD applications**: Basic data manipulation
- **Small projects**: Limited complexity
- **Tight deadlines**: No time for proper modeling
- **No domain experts**: Cannot validate business logic

## Implementation Guidelines

### 1. Start with Strategic Design
- Identify bounded contexts
- Map context relationships
- Define ubiquitous language

### 2. Focus on Core Domain
- Identify the most important business areas
- Invest modeling effort appropriately
- Use supporting patterns for less critical areas

### 3. Iterative Modeling
- Start simple and refine
- Validate with domain experts
- Continuously improve the model

### 4. Testing Strategy
- Unit test domain logic thoroughly
- Use specifications for complex business rules
- Test aggregate boundaries and invariants

## Related Patterns
- **Hexagonal Architecture**: Isolate domain from infrastructure
- **CQRS**: Separate read and write models
- **Event Sourcing**: Store state as sequence of events
- **Microservices**: Align services with bounded contexts

## References
- [Eric Evans - Domain-Driven Design](https://domainlanguage.com/ddd/)
- [Vaughn Vernon - Implementing Domain-Driven Design](https://vaughnvernon.co/?page_id=168)
- [Martin Fowler - DDD Reference](https://martinfowler.com/tags/domain%20driven%20design.html)
