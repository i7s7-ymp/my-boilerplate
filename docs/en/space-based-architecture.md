# Space-Based Architecture

## Overview
Space-Based Architecture (SBA) is a distributed computing architectural pattern that achieves high scalability by removing the central database as a synchronous constraint and instead replicating data across multiple processing units. The architecture is named after the tuple space concept and is designed to handle extremely high concurrent user loads. It separates application processing from data persistence, allowing the system to scale linearly with user demand.

## Basic Structure

```
┌─────────────────────────────────────────────────────┐
│                 Load Balancer                       │
│               (Entry Point)                         │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│  │ Processing  │ │ Processing  │ │ Processing  │   │
│  │   Unit 1    │ │   Unit 2    │ │   Unit 3    │   │
│  │             │ │             │ │             │   │
│  │ ┌─────────┐ │ │ ┌─────────┐ │ │ ┌─────────┐ │   │
│  │ │ In-Mem  │ │ │ │ In-Mem  │ │ │ │ In-Mem  │ │   │
│  │ │  Cache  │ │ │ │  Cache  │ │ │ │  Cache  │ │   │
│  │ └─────────┘ │ │ └─────────┘ │ │ └─────────┘ │   │
│  └─────────────┘ └─────────────┘ └─────────────┘   │
│                                                     │
├─────────────────────────────────────────────────────┤
│                Data Replication                     │
│                   Manager                           │
├─────────────────────────────────────────────────────┤
│                 Data Pumps                          │
│              (Async Writers)                        │
├─────────────────────────────────────────────────────┤
│              Persistent Store                       │
│                (Database)                           │
└─────────────────────────────────────────────────────┘
```

## Core Components

### 1. Processing Unit
```go
package spacebased

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "sync"
    "time"
    
    "github.com/gorilla/mux"
)

// ProcessingUnit represents a self-contained processing unit
type ProcessingUnit struct {
    id                string
    cache             InMemoryCache
    replicationManager ReplicationManager
    dataPump          DataPump
    httpServer        *http.Server
    router            *mux.Router
    businessLogic     BusinessLogicContainer
    loadBalancer      LoadBalancer
    logger            Logger
    config            ProcessingUnitConfig
    running           bool
    mu                sync.RWMutex
}

// NewProcessingUnit creates a new processing unit
func NewProcessingUnit(
    id string,
    config ProcessingUnitConfig,
    cache InMemoryCache,
    replicationManager ReplicationManager,
    dataPump DataPump,
    businessLogic BusinessLogicContainer,
    logger Logger,
) *ProcessingUnit {
    
    pu := &ProcessingUnit{
        id:                id,
        cache:             cache,
        replicationManager: replicationManager,
        dataPump:          dataPump,
        businessLogic:     businessLogic,
        logger:            logger,
        config:            config,
        router:            mux.NewRouter(),
    }
    
    pu.setupRoutes()
    return pu
}

// Start starts the processing unit
func (pu *ProcessingUnit) Start(ctx context.Context) error {
    pu.mu.Lock()
    defer pu.mu.Unlock()
    
    if pu.running {
        return fmt.Errorf("processing unit %s is already running", pu.id)
    }
    
    // Initialize in-memory cache
    if err := pu.cache.Initialize(); err != nil {
        return fmt.Errorf("failed to initialize cache: %w", err)
    }
    
    // Start data replication
    if err := pu.replicationManager.Start(ctx); err != nil {
        return fmt.Errorf("failed to start replication manager: %w", err)
    }
    
    // Start data pump
    if err := pu.dataPump.Start(ctx); err != nil {
        return fmt.Errorf("failed to start data pump: %w", err)
    }
    
    // Start HTTP server
    pu.httpServer = &http.Server{
        Addr:    fmt.Sprintf(":%d", pu.config.Port),
        Handler: pu.router,
    }
    
    go func() {
        if err := pu.httpServer.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            pu.logger.Error("HTTP server error", "unit", pu.id, "error", err)
        }
    }()
    
    pu.running = true
    pu.logger.Info("Processing unit started", "unit", pu.id, "port", pu.config.Port)
    
    return nil
}

// Stop stops the processing unit
func (pu *ProcessingUnit) Stop(ctx context.Context) error {
    pu.mu.Lock()
    defer pu.mu.Unlock()
    
    if !pu.running {
        return fmt.Errorf("processing unit %s is not running", pu.id)
    }
    
    // Stop HTTP server
    if err := pu.httpServer.Shutdown(ctx); err != nil {
        pu.logger.Error("Failed to shutdown HTTP server", "error", err)
    }
    
    // Stop data pump
    if err := pu.dataPump.Stop(ctx); err != nil {
        pu.logger.Error("Failed to stop data pump", "error", err)
    }
    
    // Stop replication manager
    if err := pu.replicationManager.Stop(ctx); err != nil {
        pu.logger.Error("Failed to stop replication manager", "error", err)
    }
    
    // Persist remaining cache data
    if err := pu.cache.Flush(); err != nil {
        pu.logger.Error("Failed to flush cache", "error", err)
    }
    
    pu.running = false
    pu.logger.Info("Processing unit stopped", "unit", pu.id)
    
    return nil
}

// setupRoutes configures HTTP routes
func (pu *ProcessingUnit) setupRoutes() {
    // Business logic routes
    pu.router.HandleFunc("/api/users", pu.handleUsers).Methods("GET", "POST")
    pu.router.HandleFunc("/api/users/{id}", pu.handleUser).Methods("GET", "PUT", "DELETE")
    pu.router.HandleFunc("/api/orders", pu.handleOrders).Methods("GET", "POST")
    pu.router.HandleFunc("/api/orders/{id}", pu.handleOrder).Methods("GET", "PUT", "DELETE")
    
    // Health and status routes
    pu.router.HandleFunc("/health", pu.handleHealthCheck).Methods("GET")
    pu.router.HandleFunc("/status", pu.handleStatus).Methods("GET")
    pu.router.HandleFunc("/metrics", pu.handleMetrics).Methods("GET")
    
    // Cache management routes
    pu.router.HandleFunc("/cache/stats", pu.handleCacheStats).Methods("GET")
    pu.router.HandleFunc("/cache/clear", pu.handleCacheClear).Methods("POST")
}

// handleUsers handles user-related requests
func (pu *ProcessingUnit) handleUsers(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case "GET":
        pu.handleGetUsers(w, r)
    case "POST":
        pu.handleCreateUser(w, r)
    }
}

// handleGetUsers retrieves users from cache
func (pu *ProcessingUnit) handleGetUsers(w http.ResponseWriter, r *http.Request) {
    // Try to get from cache first
    users, err := pu.cache.GetUsers()
    if err != nil {
        pu.logger.Error("Failed to get users from cache", "error", err)
        pu.writeErrorResponse(w, "Internal server error", http.StatusInternalServerError)
        return
    }
    
    pu.writeJSONResponse(w, users, http.StatusOK)
}

// handleCreateUser creates a new user
func (pu *ProcessingUnit) handleCreateUser(w http.ResponseWriter, r *http.Request) {
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        pu.writeErrorResponse(w, "Invalid request body", http.StatusBadRequest)
        return
    }
    
    // Validate user data
    if err := pu.validateUser(user); err != nil {
        pu.writeErrorResponse(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    // Process through business logic
    processedUser, err := pu.businessLogic.ProcessUserCreation(user)
    if err != nil {
        pu.logger.Error("Failed to process user creation", "error", err)
        pu.writeErrorResponse(w, "Processing failed", http.StatusInternalServerError)
        return
    }
    
    // Store in local cache
    if err := pu.cache.StoreUser(processedUser); err != nil {
        pu.logger.Error("Failed to store user in cache", "error", err)
        pu.writeErrorResponse(w, "Storage failed", http.StatusInternalServerError)
        return
    }
    
    // Replicate to other processing units
    if err := pu.replicationManager.ReplicateUserCreation(processedUser); err != nil {
        pu.logger.Error("Failed to replicate user creation", "error", err)
        // Continue execution - eventual consistency
    }
    
    // Queue for persistent storage
    if err := pu.dataPump.QueueUserWrite(processedUser); err != nil {
        pu.logger.Error("Failed to queue user for persistence", "error", err)
        // Continue execution - eventual persistence
    }
    
    pu.writeJSONResponse(w, processedUser, http.StatusCreated)
}

// handleOrders handles order-related requests
func (pu *ProcessingUnit) handleOrders(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case "GET":
        pu.handleGetOrders(w, r)
    case "POST":
        pu.handleCreateOrder(w, r)
    }
}

// handleCreateOrder creates a new order
func (pu *ProcessingUnit) handleCreateOrder(w http.ResponseWriter, r *http.Request) {
    var order Order
    if err := json.NewDecoder(r.Body).Decode(&order); err != nil {
        pu.writeErrorResponse(w, "Invalid request body", http.StatusBadRequest)
        return
    }
    
    // Validate order data
    if err := pu.validateOrder(order); err != nil {
        pu.writeErrorResponse(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    // Check inventory in cache
    available, err := pu.cache.CheckInventory(order.ProductID, order.Quantity)
    if err != nil {
        pu.logger.Error("Failed to check inventory", "error", err)
        pu.writeErrorResponse(w, "Inventory check failed", http.StatusInternalServerError)
        return
    }
    
    if !available {
        pu.writeErrorResponse(w, "Insufficient inventory", http.StatusBadRequest)
        return
    }
    
    // Process through business logic
    processedOrder, err := pu.businessLogic.ProcessOrderCreation(order)
    if err != nil {
        pu.logger.Error("Failed to process order creation", "error", err)
        pu.writeErrorResponse(w, "Processing failed", http.StatusInternalServerError)
        return
    }
    
    // Update inventory in cache
    if err := pu.cache.UpdateInventory(order.ProductID, -order.Quantity); err != nil {
        pu.logger.Error("Failed to update inventory", "error", err)
        pu.writeErrorResponse(w, "Inventory update failed", http.StatusInternalServerError)
        return
    }
    
    // Store order in cache
    if err := pu.cache.StoreOrder(processedOrder); err != nil {
        pu.logger.Error("Failed to store order in cache", "error", err)
        pu.writeErrorResponse(w, "Storage failed", http.StatusInternalServerError)
        return
    }
    
    // Replicate to other processing units
    if err := pu.replicationManager.ReplicateOrderCreation(processedOrder); err != nil {
        pu.logger.Error("Failed to replicate order creation", "error", err)
    }
    
    // Queue for persistent storage
    if err := pu.dataPump.QueueOrderWrite(processedOrder); err != nil {
        pu.logger.Error("Failed to queue order for persistence", "error", err)
    }
    
    pu.writeJSONResponse(w, processedOrder, http.StatusCreated)
}

// handleHealthCheck handles health check requests
func (pu *ProcessingUnit) handleHealthCheck(w http.ResponseWriter, r *http.Request) {
    health := HealthStatus{
        ProcessingUnitID: pu.id,
        Status:          "healthy",
        Timestamp:       time.Now(),
        CacheStatus:     pu.cache.GetStatus(),
        ReplicationStatus: pu.replicationManager.GetStatus(),
        DataPumpStatus:  pu.dataPump.GetStatus(),
    }
    
    pu.writeJSONResponse(w, health, http.StatusOK)
}

// handleStatus handles status requests
func (pu *ProcessingUnit) handleStatus(w http.ResponseWriter, r *http.Request) {
    status := ProcessingUnitStatus{
        ID:          pu.id,
        Running:     pu.running,
        StartTime:   pu.config.StartTime,
        CacheStats:  pu.cache.GetStatistics(),
        Connections: pu.getActiveConnections(),
        Load:        pu.getCurrentLoad(),
    }
    
    pu.writeJSONResponse(w, status, http.StatusOK)
}

// validateUser validates user data
func (pu *ProcessingUnit) validateUser(user User) error {
    if user.Email == "" {
        return fmt.Errorf("email is required")
    }
    if user.Name == "" {
        return fmt.Errorf("name is required")
    }
    return nil
}

// validateOrder validates order data
func (pu *ProcessingUnit) validateOrder(order Order) error {
    if order.CustomerID <= 0 {
        return fmt.Errorf("customer ID is required")
    }
    if order.ProductID <= 0 {
        return fmt.Errorf("product ID is required")
    }
    if order.Quantity <= 0 {
        return fmt.Errorf("quantity must be positive")
    }
    return nil
}

// getActiveConnections returns current active connections
func (pu *ProcessingUnit) getActiveConnections() int {
    // Implementation would track active HTTP connections
    return 0
}

// getCurrentLoad returns current processing load
func (pu *ProcessingUnit) getCurrentLoad() float64 {
    // Implementation would calculate current load based on metrics
    return 0.5
}

// Helper methods
func (pu *ProcessingUnit) writeJSONResponse(w http.ResponseWriter, data interface{}, statusCode int) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(statusCode)
    json.NewEncoder(w).Encode(data)
}

func (pu *ProcessingUnit) writeErrorResponse(w http.ResponseWriter, message string, statusCode int) {
    response := map[string]string{
        "error":  message,
        "status": "error",
    }
    pu.writeJSONResponse(w, response, statusCode)
}
```

### 2. In-Memory Cache
```go
package spacebased

import (
    "fmt"
    "sync"
    "time"
)

// InMemoryCache provides high-speed data access
type InMemoryCache interface {
    Initialize() error
    GetUsers() ([]User, error)
    GetUser(id int) (*User, error)
    StoreUser(user User) error
    UpdateUser(user User) error
    DeleteUser(id int) error
    
    GetOrders() ([]Order, error)
    GetOrder(id int) (*Order, error)
    StoreOrder(order Order) error
    UpdateOrder(order Order) error
    DeleteOrder(id int) error
    
    CheckInventory(productID, quantity int) (bool, error)
    UpdateInventory(productID, delta int) error
    
    GetStatus() CacheStatus
    GetStatistics() CacheStatistics
    Flush() error
}

// DefaultInMemoryCache implements InMemoryCache
type DefaultInMemoryCache struct {
    users     map[int]User
    orders    map[int]Order
    inventory map[int]int
    
    usersMu     sync.RWMutex
    ordersMu    sync.RWMutex
    inventoryMu sync.RWMutex
    
    statistics CacheStatistics
    statsMu    sync.RWMutex
    
    lastAccess time.Time
    config     CacheConfig
}

// NewDefaultInMemoryCache creates a new in-memory cache
func NewDefaultInMemoryCache(config CacheConfig) *DefaultInMemoryCache {
    return &DefaultInMemoryCache{
        users:     make(map[int]User),
        orders:    make(map[int]Order),
        inventory: make(map[int]int),
        config:    config,
        statistics: CacheStatistics{
            StartTime: time.Now(),
        },
    }
}

// Initialize initializes the cache
func (cache *DefaultInMemoryCache) Initialize() error {
    // Pre-load critical data if configured
    if cache.config.PreloadData {
        if err := cache.preloadData(); err != nil {
            return fmt.Errorf("failed to preload data: %w", err)
        }
    }
    
    // Start cache maintenance if configured
    if cache.config.MaintenanceInterval > 0 {
        go cache.startMaintenance()
    }
    
    return nil
}

// GetUsers retrieves all users from cache
func (cache *DefaultInMemoryCache) GetUsers() ([]User, error) {
    cache.usersMu.RLock()
    defer cache.usersMu.RUnlock()
    
    cache.recordCacheAccess("users", "read")
    cache.lastAccess = time.Now()
    
    users := make([]User, 0, len(cache.users))
    for _, user := range cache.users {
        users = append(users, user)
    }
    
    return users, nil
}

// GetUser retrieves a specific user from cache
func (cache *DefaultInMemoryCache) GetUser(id int) (*User, error) {
    cache.usersMu.RLock()
    defer cache.usersMu.RUnlock()
    
    cache.recordCacheAccess("user", "read")
    cache.lastAccess = time.Now()
    
    user, exists := cache.users[id]
    if !exists {
        cache.recordCacheMiss("user")
        return nil, fmt.Errorf("user %d not found in cache", id)
    }
    
    cache.recordCacheHit("user")
    return &user, nil
}

// StoreUser stores a user in cache
func (cache *DefaultInMemoryCache) StoreUser(user User) error {
    cache.usersMu.Lock()
    defer cache.usersMu.Unlock()
    
    cache.recordCacheAccess("user", "write")
    cache.users[user.ID] = user
    cache.lastAccess = time.Now()
    
    return nil
}

// UpdateUser updates a user in cache
func (cache *DefaultInMemoryCache) UpdateUser(user User) error {
    cache.usersMu.Lock()
    defer cache.usersMu.Unlock()
    
    cache.recordCacheAccess("user", "update")
    
    if _, exists := cache.users[user.ID]; !exists {
        return fmt.Errorf("user %d not found in cache", user.ID)
    }
    
    cache.users[user.ID] = user
    cache.lastAccess = time.Now()
    
    return nil
}

// DeleteUser removes a user from cache
func (cache *DefaultInMemoryCache) DeleteUser(id int) error {
    cache.usersMu.Lock()
    defer cache.usersMu.Unlock()
    
    cache.recordCacheAccess("user", "delete")
    
    if _, exists := cache.users[id]; !exists {
        return fmt.Errorf("user %d not found in cache", id)
    }
    
    delete(cache.users, id)
    cache.lastAccess = time.Now()
    
    return nil
}

// StoreOrder stores an order in cache
func (cache *DefaultInMemoryCache) StoreOrder(order Order) error {
    cache.ordersMu.Lock()
    defer cache.ordersMu.Unlock()
    
    cache.recordCacheAccess("order", "write")
    cache.orders[order.ID] = order
    cache.lastAccess = time.Now()
    
    return nil
}

// CheckInventory checks if enough inventory is available
func (cache *DefaultInMemoryCache) CheckInventory(productID, quantity int) (bool, error) {
    cache.inventoryMu.RLock()
    defer cache.inventoryMu.RUnlock()
    
    cache.recordCacheAccess("inventory", "read")
    cache.lastAccess = time.Now()
    
    available, exists := cache.inventory[productID]
    if !exists {
        cache.recordCacheMiss("inventory")
        return false, fmt.Errorf("product %d not found in inventory cache", productID)
    }
    
    cache.recordCacheHit("inventory")
    return available >= quantity, nil
}

// UpdateInventory updates inventory levels
func (cache *DefaultInMemoryCache) UpdateInventory(productID, delta int) error {
    cache.inventoryMu.Lock()
    defer cache.inventoryMu.Unlock()
    
    cache.recordCacheAccess("inventory", "update")
    
    current, exists := cache.inventory[productID]
    if !exists {
        return fmt.Errorf("product %d not found in inventory cache", productID)
    }
    
    newLevel := current + delta
    if newLevel < 0 {
        return fmt.Errorf("insufficient inventory for product %d", productID)
    }
    
    cache.inventory[productID] = newLevel
    cache.lastAccess = time.Now()
    
    return nil
}

// GetStatus returns cache status
func (cache *DefaultInMemoryCache) GetStatus() CacheStatus {
    cache.statsMu.RLock()
    defer cache.statsMu.RUnlock()
    
    return CacheStatus{
        Status:      "healthy",
        LastAccess:  cache.lastAccess,
        UserCount:   len(cache.users),
        OrderCount:  len(cache.orders),
        InventoryCount: len(cache.inventory),
        Statistics:  cache.statistics,
    }
}

// GetStatistics returns cache statistics
func (cache *DefaultInMemoryCache) GetStatistics() CacheStatistics {
    cache.statsMu.RLock()
    defer cache.statsMu.RUnlock()
    
    return cache.statistics
}

// Flush persists all cached data
func (cache *DefaultInMemoryCache) Flush() error {
    // In a real implementation, this would write data to persistent storage
    // For this example, we'll just log the action
    fmt.Println("Flushing cache data to persistent storage")
    return nil
}

// recordCacheAccess records cache access statistics
func (cache *DefaultInMemoryCache) recordCacheAccess(entityType, operation string) {
    cache.statsMu.Lock()
    defer cache.statsMu.Unlock()
    
    cache.statistics.TotalAccesses++
    
    if cache.statistics.AccessesByType == nil {
        cache.statistics.AccessesByType = make(map[string]int)
    }
    cache.statistics.AccessesByType[entityType]++
    
    if cache.statistics.OperationsByType == nil {
        cache.statistics.OperationsByType = make(map[string]int)
    }
    cache.statistics.OperationsByType[operation]++
}

// recordCacheHit records a cache hit
func (cache *DefaultInMemoryCache) recordCacheHit(entityType string) {
    cache.statsMu.Lock()
    defer cache.statsMu.Unlock()
    
    cache.statistics.TotalHits++
    
    if cache.statistics.HitsByType == nil {
        cache.statistics.HitsByType = make(map[string]int)
    }
    cache.statistics.HitsByType[entityType]++
}

// recordCacheMiss records a cache miss
func (cache *DefaultInMemoryCache) recordCacheMiss(entityType string) {
    cache.statsMu.Lock()
    defer cache.statsMu.Unlock()
    
    cache.statistics.TotalMisses++
    
    if cache.statistics.MissesByType == nil {
        cache.statistics.MissesByType = make(map[string]int)
    }
    cache.statistics.MissesByType[entityType]++
}

// preloadData pre-loads critical data into cache
func (cache *DefaultInMemoryCache) preloadData() error {
    // Implementation would load critical data from persistent storage
    // For this example, we'll just initialize with some default data
    
    // Initialize inventory for common products
    cache.inventoryMu.Lock()
    cache.inventory[1] = 1000 // Product 1: 1000 units
    cache.inventory[2] = 500  // Product 2: 500 units
    cache.inventory[3] = 750  // Product 3: 750 units
    cache.inventoryMu.Unlock()
    
    return nil
}

// startMaintenance starts cache maintenance routines
func (cache *DefaultInMemoryCache) startMaintenance() {
    ticker := time.NewTicker(cache.config.MaintenanceInterval)
    defer ticker.Stop()
    
    for range ticker.C {
        cache.performMaintenance()
    }
}

// performMaintenance performs cache maintenance tasks
func (cache *DefaultInMemoryCache) performMaintenance() {
    // Cleanup expired entries, optimize memory usage, etc.
    fmt.Println("Performing cache maintenance")
}
```

### 3. Data Replication Manager
```go
package spacebased

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "sync"
    "time"
)

// ReplicationManager handles data replication across processing units
type ReplicationManager interface {
    Start(ctx context.Context) error
    Stop(ctx context.Context) error
    ReplicateUserCreation(user User) error
    ReplicateUserUpdate(user User) error
    ReplicateUserDeletion(userID int) error
    ReplicateOrderCreation(order Order) error
    ReplicateOrderUpdate(order Order) error
    ReplicateInventoryUpdate(productID, quantity int) error
    GetStatus() ReplicationStatus
}

// DefaultReplicationManager implements ReplicationManager
type DefaultReplicationManager struct {
    processingUnits []ProcessingUnitEndpoint
    httpClient      *http.Client
    replicationQueue chan ReplicationEvent
    ctx             context.Context
    cancel          context.CancelFunc
    running         bool
    mu              sync.RWMutex
    statistics      ReplicationStatistics
    config          ReplicationConfig
    logger          Logger
}

// NewDefaultReplicationManager creates a new replication manager
func NewDefaultReplicationManager(
    processingUnits []ProcessingUnitEndpoint,
    config ReplicationConfig,
    logger Logger,
) *DefaultReplicationManager {
    return &DefaultReplicationManager{
        processingUnits:  processingUnits,
        httpClient:       &http.Client{Timeout: 5 * time.Second},
        replicationQueue: make(chan ReplicationEvent, config.QueueSize),
        config:          config,
        logger:          logger,
        statistics: ReplicationStatistics{
            StartTime: time.Now(),
        },
    }
}

// Start starts the replication manager
func (rm *DefaultReplicationManager) Start(ctx context.Context) error {
    rm.mu.Lock()
    defer rm.mu.Unlock()
    
    if rm.running {
        return fmt.Errorf("replication manager is already running")
    }
    
    rm.ctx, rm.cancel = context.WithCancel(ctx)
    rm.running = true
    
    // Start replication workers
    for i := 0; i < rm.config.WorkerCount; i++ {
        go rm.replicationWorker(i)
    }
    
    rm.logger.Info("Replication manager started", "workers", rm.config.WorkerCount)
    return nil
}

// Stop stops the replication manager
func (rm *DefaultReplicationManager) Stop(ctx context.Context) error {
    rm.mu.Lock()
    defer rm.mu.Unlock()
    
    if !rm.running {
        return fmt.Errorf("replication manager is not running")
    }
    
    rm.cancel()
    rm.running = false
    close(rm.replicationQueue)
    
    rm.logger.Info("Replication manager stopped")
    return nil
}

// ReplicateUserCreation replicates user creation to other processing units
func (rm *DefaultReplicationManager) ReplicateUserCreation(user User) error {
    event := ReplicationEvent{
        Type:      "user_created",
        Data:      user,
        Timestamp: time.Now(),
    }
    
    return rm.queueReplicationEvent(event)
}

// ReplicateUserUpdate replicates user update to other processing units
func (rm *DefaultReplicationManager) ReplicateUserUpdate(user User) error {
    event := ReplicationEvent{
        Type:      "user_updated",
        Data:      user,
        Timestamp: time.Now(),
    }
    
    return rm.queueReplicationEvent(event)
}

// ReplicateOrderCreation replicates order creation to other processing units
func (rm *DefaultReplicationManager) ReplicateOrderCreation(order Order) error {
    event := ReplicationEvent{
        Type:      "order_created",
        Data:      order,
        Timestamp: time.Now(),
    }
    
    return rm.queueReplicationEvent(event)
}

// ReplicateInventoryUpdate replicates inventory update to other processing units
func (rm *DefaultReplicationManager) ReplicateInventoryUpdate(productID, quantity int) error {
    inventoryUpdate := InventoryUpdate{
        ProductID: productID,
        Quantity:  quantity,
        Timestamp: time.Now(),
    }
    
    event := ReplicationEvent{
        Type:      "inventory_updated",
        Data:      inventoryUpdate,
        Timestamp: time.Now(),
    }
    
    return rm.queueReplicationEvent(event)
}

// queueReplicationEvent queues a replication event
func (rm *DefaultReplicationManager) queueReplicationEvent(event ReplicationEvent) error {
    select {
    case rm.replicationQueue <- event:
        rm.recordStatistic("events_queued")
        return nil
    default:
        rm.recordStatistic("queue_full_errors")
        return fmt.Errorf("replication queue is full")
    }
}

// replicationWorker processes replication events
func (rm *DefaultReplicationManager) replicationWorker(workerID int) {
    rm.logger.Info("Replication worker started", "worker_id", workerID)
    
    for {
        select {
        case event, ok := <-rm.replicationQueue:
            if !ok {
                rm.logger.Info("Replication worker stopped", "worker_id", workerID)
                return
            }
            
            rm.processReplicationEvent(event)
            
        case <-rm.ctx.Done():
            rm.logger.Info("Replication worker stopped", "worker_id", workerID)
            return
        }
    }
}

// processReplicationEvent processes a single replication event
func (rm *DefaultReplicationManager) processReplicationEvent(event ReplicationEvent) {
    for _, unit := range rm.processingUnits {
        if err := rm.sendReplicationEvent(unit, event); err != nil {
            rm.logger.Error("Failed to replicate event", 
                "event_type", event.Type,
                "unit", unit.ID,
                "error", err)
            rm.recordStatistic("replication_errors")
        } else {
            rm.recordStatistic("successful_replications")
        }
    }
}

// sendReplicationEvent sends replication event to a processing unit
func (rm *DefaultReplicationManager) sendReplicationEvent(unit ProcessingUnitEndpoint, event ReplicationEvent) error {
    payload, err := json.Marshal(event)
    if err != nil {
        return fmt.Errorf("failed to marshal event: %w", err)
    }
    
    url := fmt.Sprintf("%s/api/replication", unit.URL)
    req, err := http.NewRequestWithContext(rm.ctx, "POST", url, bytes.NewBuffer(payload))
    if err != nil {
        return fmt.Errorf("failed to create request: %w", err)
    }
    
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("X-Replication-Source", rm.config.SourceID)
    
    resp, err := rm.httpClient.Do(req)
    if err != nil {
        return fmt.Errorf("failed to send request: %w", err)
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        return fmt.Errorf("replication failed with status: %d", resp.StatusCode)
    }
    
    return nil
}

// GetStatus returns replication status
func (rm *DefaultReplicationManager) GetStatus() ReplicationStatus {
    rm.mu.RLock()
    defer rm.mu.RUnlock()
    
    return ReplicationStatus{
        Running:           rm.running,
        QueueLength:       len(rm.replicationQueue),
        ProcessingUnits:   len(rm.processingUnits),
        Statistics:        rm.statistics,
        LastReplication:   rm.statistics.LastReplication,
    }
}

// recordStatistic records a statistic
func (rm *DefaultReplicationManager) recordStatistic(statType string) {
    rm.mu.Lock()
    defer rm.mu.Unlock()
    
    if rm.statistics.EventCounts == nil {
        rm.statistics.EventCounts = make(map[string]int)
    }
    
    rm.statistics.EventCounts[statType]++
    rm.statistics.LastReplication = time.Now()
}
```

### 4. Data Pump (Async Writer)
```go
package spacebased

import (
    "context"
    "database/sql"
    "fmt"
    "sync"
    "time"
)

// DataPump handles asynchronous writes to persistent storage
type DataPump interface {
    Start(ctx context.Context) error
    Stop(ctx context.Context) error
    QueueUserWrite(user User) error
    QueueOrderWrite(order Order) error
    QueueInventoryWrite(productID, quantity int) error
    GetStatus() DataPumpStatus
}

// DefaultDataPump implements DataPump
type DefaultDataPump struct {
    db              *sql.DB
    writeQueue      chan WriteOperation
    batchQueue      []WriteOperation
    batchSize       int
    flushInterval   time.Duration
    ctx             context.Context
    cancel          context.CancelFunc
    running         bool
    mu              sync.RWMutex
    statistics      DataPumpStatistics
    logger          Logger
}

// NewDefaultDataPump creates a new data pump
func NewDefaultDataPump(
    db *sql.DB,
    batchSize int,
    flushInterval time.Duration,
    logger Logger,
) *DefaultDataPump {
    return &DefaultDataPump{
        db:            db,
        writeQueue:    make(chan WriteOperation, 10000),
        batchQueue:    make([]WriteOperation, 0, batchSize),
        batchSize:     batchSize,
        flushInterval: flushInterval,
        logger:        logger,
        statistics: DataPumpStatistics{
            StartTime: time.Now(),
        },
    }
}

// Start starts the data pump
func (dp *DefaultDataPump) Start(ctx context.Context) error {
    dp.mu.Lock()
    defer dp.mu.Unlock()
    
    if dp.running {
        return fmt.Errorf("data pump is already running")
    }
    
    dp.ctx, dp.cancel = context.WithCancel(ctx)
    dp.running = true
    
    // Start write worker
    go dp.writeWorker()
    
    // Start batch flusher
    go dp.batchFlusher()
    
    dp.logger.Info("Data pump started")
    return nil
}

// Stop stops the data pump
func (dp *DefaultDataPump) Stop(ctx context.Context) error {
    dp.mu.Lock()
    defer dp.mu.Unlock()
    
    if !dp.running {
        return fmt.Errorf("data pump is not running")
    }
    
    dp.cancel()
    dp.running = false
    close(dp.writeQueue)
    
    // Flush remaining operations
    if len(dp.batchQueue) > 0 {
        dp.flushBatch()
    }
    
    dp.logger.Info("Data pump stopped")
    return nil
}

// QueueUserWrite queues a user write operation
func (dp *DefaultDataPump) QueueUserWrite(user User) error {
    operation := WriteOperation{
        Type:      "user",
        Operation: "insert_or_update",
        Data:      user,
        Timestamp: time.Now(),
    }
    
    return dp.queueOperation(operation)
}

// QueueOrderWrite queues an order write operation
func (dp *DefaultDataPump) QueueOrderWrite(order Order) error {
    operation := WriteOperation{
        Type:      "order",
        Operation: "insert_or_update",
        Data:      order,
        Timestamp: time.Now(),
    }
    
    return dp.queueOperation(operation)
}

// QueueInventoryWrite queues an inventory write operation
func (dp *DefaultDataPump) QueueInventoryWrite(productID, quantity int) error {
    inventoryUpdate := InventoryUpdate{
        ProductID: productID,
        Quantity:  quantity,
        Timestamp: time.Now(),
    }
    
    operation := WriteOperation{
        Type:      "inventory",
        Operation: "update",
        Data:      inventoryUpdate,
        Timestamp: time.Now(),
    }
    
    return dp.queueOperation(operation)
}

// queueOperation queues a write operation
func (dp *DefaultDataPump) queueOperation(operation WriteOperation) error {
    select {
    case dp.writeQueue <- operation:
        dp.recordStatistic("operations_queued")
        return nil
    default:
        dp.recordStatistic("queue_full_errors")
        return fmt.Errorf("write queue is full")
    }
}

// writeWorker processes write operations
func (dp *DefaultDataPump) writeWorker() {
    for {
        select {
        case operation, ok := <-dp.writeQueue:
            if !ok {
                return // Channel closed
            }
            
            dp.addToBatch(operation)
            
        case <-dp.ctx.Done():
            return
        }
    }
}

// addToBatch adds operation to batch
func (dp *DefaultDataPump) addToBatch(operation WriteOperation) {
    dp.mu.Lock()
    defer dp.mu.Unlock()
    
    dp.batchQueue = append(dp.batchQueue, operation)
    
    if len(dp.batchQueue) >= dp.batchSize {
        dp.flushBatch()
    }
}

// batchFlusher periodically flushes batches
func (dp *DefaultDataPump) batchFlusher() {
    ticker := time.NewTicker(dp.flushInterval)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            dp.mu.Lock()
            if len(dp.batchQueue) > 0 {
                dp.flushBatch()
            }
            dp.mu.Unlock()
            
        case <-dp.ctx.Done():
            return
        }
    }
}

// flushBatch flushes current batch to database
func (dp *DefaultDataPump) flushBatch() {
    if len(dp.batchQueue) == 0 {
        return
    }
    
    batch := make([]WriteOperation, len(dp.batchQueue))
    copy(batch, dp.batchQueue)
    dp.batchQueue = dp.batchQueue[:0] // Clear batch
    
    go dp.processBatch(batch)
}

// processBatch processes a batch of write operations
func (dp *DefaultDataPump) processBatch(batch []WriteOperation) {
    tx, err := dp.db.Begin()
    if err != nil {
        dp.logger.Error("Failed to begin transaction", "error", err)
        dp.recordStatistic("transaction_errors")
        return
    }
    defer tx.Rollback()
    
    for _, operation := range batch {
        if err := dp.executeOperation(tx, operation); err != nil {
            dp.logger.Error("Failed to execute operation", 
                "type", operation.Type,
                "operation", operation.Operation,
                "error", err)
            dp.recordStatistic("operation_errors")
            return
        }
    }
    
    if err := tx.Commit(); err != nil {
        dp.logger.Error("Failed to commit transaction", "error", err)
        dp.recordStatistic("commit_errors")
        return
    }
    
    dp.recordStatistic("successful_batches")
    dp.recordBatchSize(len(batch))
}

// executeOperation executes a single write operation
func (dp *DefaultDataPump) executeOperation(tx *sql.Tx, operation WriteOperation) error {
    switch operation.Type {
    case "user":
        return dp.executeUserOperation(tx, operation)
    case "order":
        return dp.executeOrderOperation(tx, operation)
    case "inventory":
        return dp.executeInventoryOperation(tx, operation)
    default:
        return fmt.Errorf("unknown operation type: %s", operation.Type)
    }
}

// executeUserOperation executes user-related database operation
func (dp *DefaultDataPump) executeUserOperation(tx *sql.Tx, operation WriteOperation) error {
    user := operation.Data.(User)
    
    query := `
        INSERT INTO users (id, name, email, created_at, updated_at)
        VALUES (?, ?, ?, ?, ?)
        ON DUPLICATE KEY UPDATE
        name = VALUES(name),
        email = VALUES(email),
        updated_at = VALUES(updated_at)
    `
    
    _, err := tx.Exec(query, user.ID, user.Name, user.Email, user.CreatedAt, user.UpdatedAt)
    return err
}

// executeOrderOperation executes order-related database operation
func (dp *DefaultDataPump) executeOrderOperation(tx *sql.Tx, operation WriteOperation) error {
    order := operation.Data.(Order)
    
    query := `
        INSERT INTO orders (id, customer_id, product_id, quantity, total_amount, status, created_at, updated_at)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?)
        ON DUPLICATE KEY UPDATE
        status = VALUES(status),
        updated_at = VALUES(updated_at)
    `
    
    _, err := tx.Exec(query, order.ID, order.CustomerID, order.ProductID, 
        order.Quantity, order.TotalAmount, order.Status, order.CreatedAt, order.UpdatedAt)
    return err
}

// executeInventoryOperation executes inventory-related database operation
func (dp *DefaultDataPump) executeInventoryOperation(tx *sql.Tx, operation WriteOperation) error {
    inventoryUpdate := operation.Data.(InventoryUpdate)
    
    query := `
        INSERT INTO inventory (product_id, quantity, updated_at)
        VALUES (?, ?, ?)
        ON DUPLICATE KEY UPDATE
        quantity = ?,
        updated_at = VALUES(updated_at)
    `
    
    _, err := tx.Exec(query, inventoryUpdate.ProductID, inventoryUpdate.Quantity, 
        inventoryUpdate.Timestamp, inventoryUpdate.Quantity)
    return err
}

// GetStatus returns data pump status
func (dp *DefaultDataPump) GetStatus() DataPumpStatus {
    dp.mu.RLock()
    defer dp.mu.RUnlock()
    
    return DataPumpStatus{
        Running:      dp.running,
        QueueLength:  len(dp.writeQueue),
        BatchLength:  len(dp.batchQueue),
        Statistics:   dp.statistics,
    }
}

// recordStatistic records a statistic
func (dp *DefaultDataPump) recordStatistic(statType string) {
    dp.mu.Lock()
    defer dp.mu.Unlock()
    
    if dp.statistics.OperationCounts == nil {
        dp.statistics.OperationCounts = make(map[string]int)
    }
    
    dp.statistics.OperationCounts[statType]++
    dp.statistics.LastWrite = time.Now()
}

// recordBatchSize records batch size statistic
func (dp *DefaultDataPump) recordBatchSize(size int) {
    dp.mu.Lock()
    defer dp.mu.Unlock()
    
    dp.statistics.TotalBatches++
    dp.statistics.TotalOperations += size
    
    if size > dp.statistics.MaxBatchSize {
        dp.statistics.MaxBatchSize = size
    }
    
    if dp.statistics.MinBatchSize == 0 || size < dp.statistics.MinBatchSize {
        dp.statistics.MinBatchSize = size
    }
    
    dp.statistics.AverageBatchSize = float64(dp.statistics.TotalOperations) / float64(dp.statistics.TotalBatches)
}
```

## Domain Models and Configuration

```go
package spacebased

import "time"

// Domain Models
type User struct {
    ID        int       `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

type Order struct {
    ID          int       `json:"id"`
    CustomerID  int       `json:"customer_id"`
    ProductID   int       `json:"product_id"`
    Quantity    int       `json:"quantity"`
    TotalAmount float64   `json:"total_amount"`
    Status      string    `json:"status"`
    CreatedAt   time.Time `json:"created_at"`
    UpdatedAt   time.Time `json:"updated_at"`
}

type InventoryUpdate struct {
    ProductID int       `json:"product_id"`
    Quantity  int       `json:"quantity"`
    Timestamp time.Time `json:"timestamp"`
}

// Configuration Types
type ProcessingUnitConfig struct {
    ID               string        `json:"id"`
    Port             int           `json:"port"`
    StartTime        time.Time     `json:"start_time"`
    CacheConfig      CacheConfig   `json:"cache_config"`
    ReplicationConfig ReplicationConfig `json:"replication_config"`
}

type CacheConfig struct {
    PreloadData         bool          `json:"preload_data"`
    MaintenanceInterval time.Duration `json:"maintenance_interval"`
}

type ReplicationConfig struct {
    SourceID    string `json:"source_id"`
    QueueSize   int    `json:"queue_size"`
    WorkerCount int    `json:"worker_count"`
}

// Status Types
type HealthStatus struct {
    ProcessingUnitID  string              `json:"processing_unit_id"`
    Status           string              `json:"status"`
    Timestamp        time.Time           `json:"timestamp"`
    CacheStatus      CacheStatus         `json:"cache_status"`
    ReplicationStatus ReplicationStatus   `json:"replication_status"`
    DataPumpStatus   DataPumpStatus      `json:"data_pump_status"`
}

type CacheStatus struct {
    Status         string           `json:"status"`
    LastAccess     time.Time        `json:"last_access"`
    UserCount      int              `json:"user_count"`
    OrderCount     int              `json:"order_count"`
    InventoryCount int              `json:"inventory_count"`
    Statistics     CacheStatistics  `json:"statistics"`
}

type CacheStatistics struct {
    StartTime         time.Time         `json:"start_time"`
    TotalAccesses     int               `json:"total_accesses"`
    TotalHits         int               `json:"total_hits"`
    TotalMisses       int               `json:"total_misses"`
    AccessesByType    map[string]int    `json:"accesses_by_type"`
    HitsByType        map[string]int    `json:"hits_by_type"`
    MissesByType      map[string]int    `json:"misses_by_type"`
    OperationsByType  map[string]int    `json:"operations_by_type"`
}

type ReplicationStatus struct {
    Running           bool                   `json:"running"`
    QueueLength       int                    `json:"queue_length"`
    ProcessingUnits   int                    `json:"processing_units"`
    Statistics        ReplicationStatistics  `json:"statistics"`
    LastReplication   time.Time              `json:"last_replication"`
}

type ReplicationStatistics struct {
    StartTime       time.Time         `json:"start_time"`
    EventCounts     map[string]int    `json:"event_counts"`
    LastReplication time.Time         `json:"last_replication"`
}

type DataPumpStatus struct {
    Running     bool               `json:"running"`
    QueueLength int                `json:"queue_length"`
    BatchLength int                `json:"batch_length"`
    Statistics  DataPumpStatistics `json:"statistics"`
}

type DataPumpStatistics struct {
    StartTime        time.Time         `json:"start_time"`
    TotalBatches     int               `json:"total_batches"`
    TotalOperations  int               `json:"total_operations"`
    MaxBatchSize     int               `json:"max_batch_size"`
    MinBatchSize     int               `json:"min_batch_size"`
    AverageBatchSize float64           `json:"average_batch_size"`
    OperationCounts  map[string]int    `json:"operation_counts"`
    LastWrite        time.Time         `json:"last_write"`
}

// Event Types
type ReplicationEvent struct {
    Type      string      `json:"type"`
    Data      interface{} `json:"data"`
    Timestamp time.Time   `json:"timestamp"`
}

type WriteOperation struct {
    Type      string      `json:"type"`
    Operation string      `json:"operation"`
    Data      interface{} `json:"data"`
    Timestamp time.Time   `json:"timestamp"`
}

type ProcessingUnitEndpoint struct {
    ID  string `json:"id"`
    URL string `json:"url"`
}

type ProcessingUnitStatus struct {
    ID          string         `json:"id"`
    Running     bool           `json:"running"`
    StartTime   time.Time      `json:"start_time"`
    CacheStats  CacheStatistics `json:"cache_stats"`
    Connections int            `json:"connections"`
    Load        float64        `json:"load"`
}

// Interface Types
type BusinessLogicContainer interface {
    ProcessUserCreation(user User) (User, error)
    ProcessOrderCreation(order Order) (Order, error)
    ProcessInventoryUpdate(productID, quantity int) error
}

type LoadBalancer interface {
    RouteRequest(request interface{}) (string, error)
    GetHealthyUnits() []string
    RegisterUnit(unitID, url string) error
    UnregisterUnit(unitID string) error
}

type Logger interface {
    Info(msg string, fields ...interface{})
    Error(msg string, fields ...interface{})
    Warn(msg string, fields ...interface{})
}
```

## Benefits

### Extreme Scalability
- Linear scalability by adding processing units
- No database bottleneck during peak loads
- High concurrent user capacity

### High Performance
- In-memory data access for ultra-fast response times
- Parallel processing across multiple units
- Minimal latency for read/write operations

### High Availability
- No single points of failure
- Data replication across processing units
- Graceful degradation during failures

### Elasticity
- Dynamic scaling based on load
- Processing units can be added/removed on demand
- Cloud-friendly architecture

## Drawbacks

### Complexity
- Complex data synchronization
- Difficult debugging and monitoring
- Sophisticated infrastructure requirements

### Data Consistency
- Eventual consistency model
- Potential data conflicts during replication
- Complex conflict resolution mechanisms needed

### Cost
- High memory requirements for in-memory caches
- Multiple processing units increase infrastructure costs
- Operational complexity increases maintenance costs

### Limited Use Cases
- Not suitable for all application types
- Requires specific load patterns to be effective
- ACID transaction limitations

## When to Use

### Suitable Scenarios
- **High-volume applications**: Extremely high concurrent user loads
- **Variable load patterns**: Unpredictable traffic spikes
- **Read-heavy workloads**: Frequent data access requirements
- **Real-time applications**: Low latency requirements

### Not Suitable Scenarios
- **ACID transactions**: Strong consistency requirements
- **Small applications**: Infrastructure overhead too high
- **Batch processing**: Sequential processing patterns
- **Low concurrency**: Minimal concurrent users

## Implementation Considerations

### Data Partitioning
```go
// Data partitioning strategy for distribution across processing units
type DataPartitioner interface {
    PartitionUser(user User) string
    PartitionOrder(order Order) string
    GetPartitionForKey(key string) string
}

type HashBasedPartitioner struct {
    partitions []string
}

func (p *HashBasedPartitioner) PartitionUser(user User) string {
    hash := hashString(user.Email)
    return p.partitions[hash%len(p.partitions)]
}
```

### Load Balancing
```go
// Load balancing strategies for request distribution
type LoadBalancingStrategy interface {
    SelectProcessingUnit(request interface{}) string
}

type RoundRobinLoadBalancer struct {
    units   []string
    current int
    mu      sync.Mutex
}

func (lb *RoundRobinLoadBalancer) SelectProcessingUnit(request interface{}) string {
    lb.mu.Lock()
    defer lb.mu.Unlock()
    
    unit := lb.units[lb.current]
    lb.current = (lb.current + 1) % len(lb.units)
    return unit
}
```

### Conflict Resolution
```go
// Conflict resolution for data synchronization
type ConflictResolver interface {
    ResolveUserConflict(local, remote User) User
    ResolveOrderConflict(local, remote Order) Order
}

type TimestampBasedResolver struct{}

func (r *TimestampBasedResolver) ResolveUserConflict(local, remote User) User {
    if remote.UpdatedAt.After(local.UpdatedAt) {
        return remote
    }
    return local
}
```

## Related Patterns
- **Microservices Architecture**: Service decomposition
- **CQRS**: Command query separation
- **Event Sourcing**: Event-driven state management
- **Distributed Cache**: In-memory data distribution
- **Master-Slave Replication**: Data replication strategies

## Best Practices

### 1. Cache Design
- Optimal cache size configuration
- Efficient eviction policies
- Cache warming strategies
- Data locality optimization

### 2. Replication Strategy
- Asynchronous replication for performance
- Conflict resolution mechanisms
- Network partition handling
- Consistency level configuration

### 3. Data Pump Design
- Batch processing for efficiency
- Failure recovery mechanisms
- Data integrity validation
- Performance monitoring

### 4. Monitoring and Observability
- Real-time performance metrics
- Cache hit/miss ratios
- Replication lag monitoring
- Processing unit health checks

## References
- [Software Architecture Patterns - Mark Richards](https://www.oreilly.com/library/view/software-architecture-patterns/9781491971437/)
- [GigaSpaces Documentation](https://docs.gigaspaces.com/)
- [Apache Ignite In-Memory Platform](https://ignite.apache.org/)
- [Hazelcast In-Memory Data Grid](https://hazelcast.com/)
