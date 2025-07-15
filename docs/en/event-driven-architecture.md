# Event-Driven Architecture (EDA)

## Overview
Event-Driven Architecture (EDA) is an architectural pattern designed around the generation, detection, consumption, and reaction to events. It achieves loose coupling between system components and features event-based communication with emphasis on real-time processing and scalability.

## Core Concepts

### Event
```go
package events

import (
    "time"
    "github.com/google/uuid"
)

// Event basic event structure
type Event struct {
    EventID   string                 `json:"event_id"`
    EventType string                 `json:"event_type"`
    Source    string                 `json:"source"`
    Timestamp time.Time              `json:"timestamp"`
    Data      map[string]interface{} `json:"data"`
    Version   string                 `json:"version"`
}

// NewEvent creates a basic event
func NewEvent(eventType, source string, data map[string]interface{}) *Event {
    return &Event{
        EventID:   uuid.New().String(),
        EventType: eventType,
        Source:    source,
        Timestamp: time.Now().UTC(),
        Data:      data,
        Version:   "1.0",
    }
}

// UserRegisteredEvent user registration event
type UserRegisteredEvent struct {
    *Event
    UserID string `json:"user_id"`
    Email  string `json:"email"`
    Name   string `json:"name"`
}

// NewUserRegisteredEvent creates a user registration event
func NewUserRegisteredEvent(userID, email, name string) *UserRegisteredEvent {
    data := map[string]interface{}{
        "user_id": userID,
        "email":   email,
        "name":    name,
    }
    
    return &UserRegisteredEvent{
        Event:  NewEvent("user.registered", "user-service", data),
        UserID: userID,
        Email:  email,
        Name:   name,
    }
}

// OrderPlacedEvent order placement event
type OrderPlacedEvent struct {
    *Event
    OrderID    string      `json:"order_id"`
    CustomerID string      `json:"customer_id"`
    Amount     float64     `json:"amount"`
    Items      []OrderItem `json:"items"`
}

type OrderItem struct {
    ProductID string  `json:"product_id"`
    Quantity  int     `json:"quantity"`
    Price     float64 `json:"price"`
}

// NewOrderPlacedEvent creates an order placement event
func NewOrderPlacedEvent(orderID, customerID string, amount float64, items []OrderItem) *OrderPlacedEvent {
    data := map[string]interface{}{
        "order_id":    orderID,
        "customer_id": customerID,
        "amount":      amount,
        "items":       items,
    }
    
    return &OrderPlacedEvent{
        Event:      NewEvent("order.placed", "order-service", data),
        OrderID:    orderID,
        CustomerID: customerID,
        Amount:     amount,
        Items:      items,
    }
}

// PaymentProcessedEvent payment processing event
type PaymentProcessedEvent struct {
    *Event
    PaymentID string  `json:"payment_id"`
    OrderID   string  `json:"order_id"`
    Amount    float64 `json:"amount"`
    Status    string  `json:"status"`
}

// NewPaymentProcessedEvent creates a payment processing event
func NewPaymentProcessedEvent(paymentID, orderID string, amount float64, status string) *PaymentProcessedEvent {
    data := map[string]interface{}{
        "payment_id": paymentID,
        "order_id":   orderID,
        "amount":     amount,
        "status":     status,
    }
    
    return &PaymentProcessedEvent{
        Event:     NewEvent("payment.processed", "payment-service", data),
        PaymentID: paymentID,
        OrderID:   orderID,
        Amount:    amount,
        Status:    status,
    }
}
```

## Architecture Components

### 1. Event Producer
```go
// EventProducer event producer interface
type EventProducer interface {
    PublishEvent(event *Event) error
}

// UserService user service (event producer)
type UserService struct {
    eventBus       EventBus
    userRepository UserRepository
}

// NewUserService creates a user service
func NewUserService(eventBus EventBus, userRepository UserRepository) *UserService {
    return &UserService{
        eventBus:       eventBus,
        userRepository: userRepository,
    }
}

// RegisterUser registers a user
func (s *UserService) RegisterUser(email, name, password string) (string, error) {
    // Create user
    userID, err := s.userRepository.CreateUser(email, name, password)
    if err != nil {
        return "", err
    }
    
    // Publish event
    event := NewUserRegisteredEvent(userID, email, name)
    if err := s.PublishEvent(event.Event); err != nil {
        return "", err
    }
    
    return userID, nil
}

// PublishEvent publishes an event
func (s *UserService) PublishEvent(event *Event) error {
    return s.eventBus.Publish(event)
}

// OrderService order service (event producer)
type OrderService struct {
    eventBus        EventBus
    orderRepository OrderRepository
}

// NewOrderService creates an order service
func NewOrderService(eventBus EventBus, orderRepository OrderRepository) *OrderService {
    return &OrderService{
        eventBus:        eventBus,
        orderRepository: orderRepository,
    }
}

// PlaceOrder places an order
func (s *OrderService) PlaceOrder(customerID string, items []OrderItem) (string, error) {
    // Create order
    orderID, err := s.orderRepository.CreateOrder(customerID, items)
    if err != nil {
        return "", err
    }
    
    // Calculate total amount
    var amount float64
    for _, item := range items {
        amount += item.Price * float64(item.Quantity)
    }
    
    // Publish event
    event := NewOrderPlacedEvent(orderID, customerID, amount, items)
    if err := s.PublishEvent(event.Event); err != nil {
        return "", err
    }
    
    return orderID, nil
}

// PublishEvent publishes an event
func (s *OrderService) PublishEvent(event *Event) error {
    return s.eventBus.Publish(event)
}
```

### 2. Event Consumer
```go
// EventHandler event handler interface
type EventHandler interface {
    Handle(event *Event) error
    GetEventType() string
}

// EmailNotificationHandler email notification handler
type EmailNotificationHandler struct {
    emailService EmailService
    logger       Logger
}

// NewEmailNotificationHandler creates an email notification handler
func NewEmailNotificationHandler(emailService EmailService, logger Logger) *EmailNotificationHandler {
    return &EmailNotificationHandler{
        emailService: emailService,
        logger:       logger,
    }
}

// Handle processes an event
func (h *EmailNotificationHandler) Handle(event *Event) error {
    switch event.EventType {
    case "user.registered":
        return h.handleUserRegistered(event)
    case "order.placed":
        return h.handleOrderPlaced(event)
    case "payment.failed":
        return h.handlePaymentFailed(event)
    default:
        h.logger.Info("Unhandled event type: " + event.EventType)
        return nil
    }
}

// GetEventType returns the event type this handler processes
func (h *EmailNotificationHandler) GetEventType() string {
    return "*" // Handle all events
}

func (h *EmailNotificationHandler) handleUserRegistered(event *Event) error {
    h.logger.Info("Sending welcome email for user registration")
    // Welcome email sending logic
    return h.emailService.SendWelcomeEmail(event.Data)
}

func (h *EmailNotificationHandler) handleOrderPlaced(event *Event) error {
    h.logger.Info("Sending order confirmation email")
    // Order confirmation email sending logic
    return h.emailService.SendOrderConfirmationEmail(event.Data)
}

func (h *EmailNotificationHandler) handlePaymentFailed(event *Event) error {
    h.logger.Error("Sending payment failure notification")
    // Payment failure notification email sending logic
    return h.emailService.SendPaymentFailureEmail(event.Data)
}
```

### 3. Event Bus
```go
import (
    "context"
    "fmt"
    "log"
    "sync"
    "time"
)

// EventBus event bus - manages event distribution
type EventBus interface {
    Subscribe(eventType string, handler EventHandler) error
    SubscribeToAll(handler EventHandler) error
    Publish(event *Event) error
    Start(ctx context.Context) error
    Stop() error
}

// DefaultEventBus default event bus implementation
type DefaultEventBus struct {
    handlers   map[string][]EventHandler
    eventQueue chan *Event
    running    bool
    logger     *log.Logger
    mu         sync.RWMutex
    wg         sync.WaitGroup
}

// NewDefaultEventBus creates a default event bus
func NewDefaultEventBus(bufferSize int) *DefaultEventBus {
    return &DefaultEventBus{
        handlers:   make(map[string][]EventHandler),
        eventQueue: make(chan *Event, bufferSize),
        logger:     log.New(log.Writer(), "[EventBus] ", log.LstdFlags),
    }
}

// Subscribe registers an event handler
func (eb *DefaultEventBus) Subscribe(eventType string, handler EventHandler) error {
    eb.mu.Lock()
    defer eb.mu.Unlock()
    
    eb.handlers[eventType] = append(eb.handlers[eventType], handler)
    eb.logger.Printf("Handler subscribed to %s", eventType)
    return nil
}

// SubscribeToAll registers a handler for all event types
func (eb *DefaultEventBus) SubscribeToAll(handler EventHandler) error {
    return eb.Subscribe("*", handler)
}

// Publish publishes an event
func (eb *DefaultEventBus) Publish(event *Event) error {
    select {
    case eb.eventQueue <- event:
        eb.logger.Printf("Event published: %s (%s)", event.EventType, event.EventID)
        return nil
    default:
        return fmt.Errorf("event queue is full")
    }
}

// Start starts event processing
func (eb *DefaultEventBus) Start(ctx context.Context) error {
    eb.mu.Lock()
    eb.running = true
    eb.mu.Unlock()
    
    eb.logger.Println("Event bus started")
    
    eb.wg.Add(1)
    go func() {
        defer eb.wg.Done()
        eb.processEvents(ctx)
    }()
    
    return nil
}

// processEvents event processing loop
func (eb *DefaultEventBus) processEvents(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            eb.logger.Println("Event bus stopping...")
            return
        case event := <-eb.eventQueue:
            if err := eb.processEvent(event); err != nil {
                eb.logger.Printf("Error processing event: %v", err)
            }
        case <-time.After(1 * time.Second):
            // Continue on timeout
            continue
        }
    }
}

// processEvent processes a single event
func (eb *DefaultEventBus) processEvent(event *Event) error {
    eb.mu.RLock()
    specificHandlers := eb.handlers[event.EventType]
    globalHandlers := eb.handlers["*"]
    eb.mu.RUnlock()
    
    allHandlers := append(specificHandlers, globalHandlers...)
    
    if len(allHandlers) == 0 {
        eb.logger.Printf("No handlers for event type: %s", event.EventType)
        return nil
    }
    
    // Execute handlers concurrently
    errChan := make(chan error, len(allHandlers))
    for _, handler := range allHandlers {
        go func(h EventHandler) {
            errChan <- h.Handle(event)
        }(handler)
    }
    
    // Collect results
    var errors []error
    for i := 0; i < len(allHandlers); i++ {
        if err := <-errChan; err != nil {
            errors = append(errors, err)
        }
    }
    
    if len(errors) > 0 {
        return fmt.Errorf("handler errors: %v", errors)
    }
    
    return nil
}

// Stop stops event processing
func (eb *DefaultEventBus) Stop() error {
    eb.mu.Lock()
    eb.running = false
    eb.mu.Unlock()
    
    close(eb.eventQueue)
    eb.wg.Wait()
    
    eb.logger.Println("Event bus stopped")
    return nil
}
```

## Benefits

### Loose Coupling
- Services don't need to know about each other directly
- Easy to add new event consumers
- Flexible system extension

### Scalability
- Independent scaling of event producers and consumers
- Horizontal scaling support
- Load distribution through event queues

### Real-time Processing
- Immediate response to system changes
- Support for real-time notifications
- Fast data propagation

### Fault Tolerance
- System continues operating even if some components fail
- Event replay capability
- Resilient error handling

## Drawbacks

### Complexity
- Difficult to track event flow
- Debugging challenges
- Event ordering issues

### Eventual Consistency
- Data may be temporarily inconsistent
- Need to handle delayed processing
- Complex state management

### Performance Overhead
- Additional network communication
- Event serialization/deserialization costs
- Message queue overhead

## Implementation Patterns

### 1. Event Sourcing
```go
// EventStore event store interface
type EventStore interface {
    SaveEvents(streamID string, events []Event, expectedVersion int) error
    GetEvents(streamID string, fromVersion int) ([]Event, error)
    GetAllEvents() ([]Event, error)
}

// InMemoryEventStore in-memory event store implementation
type InMemoryEventStore struct {
    streams map[string][]Event
    mu      sync.RWMutex
}

func NewInMemoryEventStore() *InMemoryEventStore {
    return &InMemoryEventStore{
        streams: make(map[string][]Event),
    }
}

func (es *InMemoryEventStore) SaveEvents(streamID string, events []Event, expectedVersion int) error {
    es.mu.Lock()
    defer es.mu.Unlock()
    
    stream := es.streams[streamID]
    if len(stream) != expectedVersion {
        return fmt.Errorf("concurrency conflict: expected version %d, actual %d", 
            expectedVersion, len(stream))
    }
    
    es.streams[streamID] = append(stream, events...)
    return nil
}

func (es *InMemoryEventStore) GetEvents(streamID string, fromVersion int) ([]Event, error) {
    es.mu.RLock()
    defer es.mu.RUnlock()
    
    stream := es.streams[streamID]
    if fromVersion >= len(stream) {
        return []Event{}, nil
    }
    
    return stream[fromVersion:], nil
}

func (es *InMemoryEventStore) GetAllEvents() ([]Event, error) {
    es.mu.RLock()
    defer es.mu.RUnlock()
    
    var allEvents []Event
    for _, stream := range es.streams {
        allEvents = append(allEvents, stream...)
    }
    
    return allEvents, nil
}
```

### 2. CQRS Integration
```go
// CommandHandler command handler interface
type CommandHandler interface {
    Handle(command interface{}) error
}

// QueryHandler query handler interface
type QueryHandler interface {
    Handle(query interface{}) (interface{}, error)
}

// EventProjector event projector for read models
type EventProjector struct {
    readRepository ReadRepository
}

func NewEventProjector(readRepository ReadRepository) *EventProjector {
    return &EventProjector{
        readRepository: readRepository,
    }
}

func (p *EventProjector) Handle(event *Event) error {
    switch event.EventType {
    case "user.registered":
        return p.handleUserRegistered(event)
    case "order.placed":
        return p.handleOrderPlaced(event)
    default:
        return nil
    }
}

func (p *EventProjector) handleUserRegistered(event *Event) error {
    userID := event.Data["user_id"].(string)
    email := event.Data["email"].(string)
    name := event.Data["name"].(string)
    
    readModel := &UserReadModel{
        ID:    userID,
        Email: email,
        Name:  name,
    }
    
    return p.readRepository.SaveUser(readModel)
}

func (p *EventProjector) handleOrderPlaced(event *Event) error {
    orderID := event.Data["order_id"].(string)
    customerID := event.Data["customer_id"].(string)
    amount := event.Data["amount"].(float64)
    
    readModel := &OrderReadModel{
        ID:         orderID,
        CustomerID: customerID,
        Amount:     amount,
        Status:     "placed",
    }
    
    return p.readRepository.SaveOrder(readModel)
}
```

## Applicable Scenarios

### When Suitable
- **Microservices architecture**: Loose coupling between services
- **Real-time systems**: Immediate response to changes required
- **Complex workflows**: Multiple steps with different processing times
- **High scalability requirements**: Independent scaling needed

### When Not Suitable
- **Simple applications**: Synchronous processing is sufficient
- **Strong consistency requirements**: Immediate consistency is critical
- **Small teams**: Complexity outweighs benefits

## Implementation Considerations

### Event Design
- **Event immutability**: Events should never be modified
- **Event versioning**: Support for schema evolution
- **Event naming**: Clear and consistent naming conventions

### Error Handling
- **Retry mechanisms**: Automatic retry for transient failures
- **Dead letter queues**: Handling of failed events
- **Circuit breakers**: Preventing cascade failures

### Monitoring
- **Event tracking**: Monitoring event flow and processing
- **Performance metrics**: Latency and throughput monitoring
- **Error monitoring**: Failed event processing tracking

## Related Patterns
- **CQRS**: Command Query Responsibility Segregation
- **Event Sourcing**: State management through events
- **Saga Pattern**: Distributed transaction management
- **Publisher-Subscriber**: Event notification pattern

## References
- [Martin Fowler - Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)
- [AWS - What is Event-Driven Architecture?](https://aws.amazon.com/event-driven-architecture/)
- [Microsoft - Event-driven architecture style](https://docs.microsoft.com/en-us/azure/architecture/guide/architecture-styles/event-driven)
