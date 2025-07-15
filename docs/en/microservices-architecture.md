# Microservices Architecture

## Overview
Microservices Architecture is an approach to developing software applications as a suite of small, independently deployable services. Each service is focused on a specific business function, runs in its own process, and communicates via well-defined APIs. This architecture enables organizations to build scalable, resilient, and maintainable distributed systems.

## Basic Structure

```
┌─────────────────────────────────────────────────────┐
│                 API Gateway                         │
│               (Entry Point)                         │
├─────────────────────────────────────────────────────┤
│         ↓           ↓           ↓           ↓       │
├─────────────┬─────────────┬─────────────┬───────────┤
│   User      │   Order     │   Payment   │ Inventory │
│  Service    │   Service   │   Service   │  Service  │
├─────────────┼─────────────┼─────────────┼───────────┤
│     ↓       │      ↓      │      ↓      │     ↓     │
├─────────────┼─────────────┼─────────────┼───────────┤
│   User      │   Order     │   Payment   │ Inventory │
│ Database    │  Database   │  Database   │ Database  │
└─────────────┴─────────────┴─────────────┴───────────┘
```

## Core Components

### 1. API Gateway
```go
package gateway

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "net/http/httputil"
    "net/url"
    "strings"
    "time"
    
    "github.com/gorilla/mux"
)

// APIGateway handles routing and cross-cutting concerns
type APIGateway struct {
    router           *mux.Router
    serviceRegistry  ServiceRegistry
    authService      AuthService
    rateLimiter      RateLimiter
    circuitBreaker   CircuitBreaker
    logger          Logger
    metricsCollector MetricsCollector
}

// NewAPIGateway creates a new API gateway
func NewAPIGateway(
    serviceRegistry ServiceRegistry,
    authService AuthService,
    rateLimiter RateLimiter,
    circuitBreaker CircuitBreaker,
    logger Logger,
    metricsCollector MetricsCollector,
) *APIGateway {
    
    gateway := &APIGateway{
        router:           mux.NewRouter(),
        serviceRegistry:  serviceRegistry,
        authService:      authService,
        rateLimiter:      rateLimiter,
        circuitBreaker:   circuitBreaker,
        logger:          logger,
        metricsCollector: metricsCollector,
    }
    
    gateway.setupRoutes()
    return gateway
}

// setupRoutes configures API routes
func (g *APIGateway) setupRoutes() {
    // User service routes
    g.router.PathPrefix("/api/users").Handler(
        g.createServiceProxy("user-service", "/api/users"),
    )
    
    // Order service routes
    g.router.PathPrefix("/api/orders").Handler(
        g.createServiceProxy("order-service", "/api/orders"),
    )
    
    // Payment service routes
    g.router.PathPrefix("/api/payments").Handler(
        g.createServiceProxy("payment-service", "/api/payments"),
    )
    
    // Inventory service routes
    g.router.PathPrefix("/api/inventory").Handler(
        g.createServiceProxy("inventory-service", "/api/inventory"),
    )
    
    // Health check endpoint
    g.router.HandleFunc("/health", g.healthCheck).Methods("GET")
    
    // Metrics endpoint
    g.router.HandleFunc("/metrics", g.metricsHandler).Methods("GET")
}

// createServiceProxy creates a reverse proxy for a service
func (g *APIGateway) createServiceProxy(serviceName, pathPrefix string) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        startTime := time.Now()
        
        // Apply middleware chain
        if err := g.applyMiddleware(r, serviceName); err != nil {
            g.writeErrorResponse(w, err, http.StatusUnauthorized)
            return
        }
        
        // Get service URL from registry
        serviceURL, err := g.serviceRegistry.GetServiceURL(serviceName)
        if err != nil {
            g.logger.Error("Service not found", "service", serviceName, "error", err)
            g.writeErrorResponse(w, err, http.StatusServiceUnavailable)
            return
        }
        
        // Execute with circuit breaker
        err = g.circuitBreaker.Execute(serviceName, func() error {
            return g.proxyRequest(w, r, serviceURL, pathPrefix)
        })
        
        if err != nil {
            g.logger.Error("Service call failed", "service", serviceName, "error", err)
            g.writeErrorResponse(w, err, http.StatusInternalServerError)
            return
        }
        
        // Record metrics
        duration := time.Since(startTime)
        g.metricsCollector.RecordDuration("gateway_request_duration", duration, map[string]string{
            "service": serviceName,
            "method":  r.Method,
            "path":    r.URL.Path,
        })
    })
}

// applyMiddleware applies cross-cutting concerns
func (g *APIGateway) applyMiddleware(r *http.Request, serviceName string) error {
    // Rate limiting
    if err := g.rateLimiter.Allow(r); err != nil {
        return fmt.Errorf("rate limit exceeded: %w", err)
    }
    
    // Authentication (skip for public endpoints)
    if !g.isPublicEndpoint(r.URL.Path) {
        if err := g.authService.Authenticate(r); err != nil {
            return fmt.Errorf("authentication failed: %w", err)
        }
    }
    
    // Request logging
    g.logger.Info("Gateway request", "method", r.Method, "path", r.URL.Path, "service", serviceName)
    
    return nil
}

// proxyRequest proxies the request to the target service
func (g *APIGateway) proxyRequest(w http.ResponseWriter, r *http.Request, serviceURL *url.URL, pathPrefix string) error {
    // Modify request path
    r.URL.Path = strings.TrimPrefix(r.URL.Path, pathPrefix)
    
    // Create reverse proxy
    proxy := httputil.NewSingleHostReverseProxy(serviceURL)
    
    // Add custom headers
    r.Header.Set("X-Gateway-Request-ID", generateRequestID())
    r.Header.Set("X-Gateway-Timestamp", time.Now().Format(time.RFC3339))
    
    // Proxy the request
    proxy.ServeHTTP(w, r)
    return nil
}

// isPublicEndpoint checks if endpoint requires authentication
func (g *APIGateway) isPublicEndpoint(path string) bool {
    publicPaths := []string{"/health", "/metrics", "/api/users/register", "/api/users/login"}
    for _, publicPath := range publicPaths {
        if strings.HasPrefix(path, publicPath) {
            return true
        }
    }
    return false
}

// writeErrorResponse writes error response
func (g *APIGateway) writeErrorResponse(w http.ResponseWriter, err error, statusCode int) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(statusCode)
    
    response := map[string]string{
        "error":   err.Error(),
        "status":  "error",
        "code":    fmt.Sprintf("%d", statusCode),
    }
    
    json.NewEncoder(w).Encode(response)
}

// healthCheck handles health check requests
func (g *APIGateway) healthCheck(w http.ResponseWriter, r *http.Request) {
    status := map[string]interface{}{
        "status":    "healthy",
        "timestamp": time.Now(),
        "services":  g.checkServicesHealth(),
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(status)
}

// checkServicesHealth checks health of registered services
func (g *APIGateway) checkServicesHealth() map[string]string {
    services := g.serviceRegistry.ListServices()
    health := make(map[string]string)
    
    for _, service := range services {
        if g.serviceRegistry.IsHealthy(service) {
            health[service] = "healthy"
        } else {
            health[service] = "unhealthy"
        }
    }
    
    return health
}

// metricsHandler handles metrics requests
func (g *APIGateway) metricsHandler(w http.ResponseWriter, r *http.Request) {
    metrics := g.metricsCollector.GetMetrics()
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(metrics)
}

// generateRequestID generates unique request ID
func generateRequestID() string {
    return fmt.Sprintf("req-%d", time.Now().UnixNano())
}
```

### 2. Service Registry
```go
package registry

import (
    "context"
    "fmt"
    "net/http"
    "net/url"
    "sync"
    "time"
)

// ServiceRegistry manages service discovery and health checking
type ServiceRegistry interface {
    RegisterService(service ServiceInfo) error
    UnregisterService(serviceName string) error
    GetServiceURL(serviceName string) (*url.URL, error)
    ListServices() []string
    IsHealthy(serviceName string) bool
    StartHealthChecking(ctx context.Context)
}

// ServiceInfo contains service registration information
type ServiceInfo struct {
    Name        string            `json:"name"`
    URL         string            `json:"url"`
    HealthCheck string            `json:"health_check"`
    Version     string            `json:"version"`
    Metadata    map[string]string `json:"metadata"`
    RegisteredAt time.Time         `json:"registered_at"`
}

// DefaultServiceRegistry implements ServiceRegistry
type DefaultServiceRegistry struct {
    services    map[string]ServiceInfo
    healthStatus map[string]bool
    client      *http.Client
    mu          sync.RWMutex
    logger      Logger
}

// NewDefaultServiceRegistry creates a new service registry
func NewDefaultServiceRegistry(logger Logger) *DefaultServiceRegistry {
    return &DefaultServiceRegistry{
        services:     make(map[string]ServiceInfo),
        healthStatus: make(map[string]bool),
        client:       &http.Client{Timeout: 5 * time.Second},
        logger:       logger,
    }
}

// RegisterService registers a new service
func (r *DefaultServiceRegistry) RegisterService(service ServiceInfo) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    service.RegisteredAt = time.Now()
    r.services[service.Name] = service
    r.healthStatus[service.Name] = true
    
    r.logger.Info("Service registered", "name", service.Name, "url", service.URL)
    return nil
}

// UnregisterService unregisters a service
func (r *DefaultServiceRegistry) UnregisterService(serviceName string) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    delete(r.services, serviceName)
    delete(r.healthStatus, serviceName)
    
    r.logger.Info("Service unregistered", "name", serviceName)
    return nil
}

// GetServiceURL returns the URL for a service
func (r *DefaultServiceRegistry) GetServiceURL(serviceName string) (*url.URL, error) {
    r.mu.RLock()
    service, exists := r.services[serviceName]
    r.mu.RUnlock()
    
    if !exists {
        return nil, fmt.Errorf("service not found: %s", serviceName)
    }
    
    return url.Parse(service.URL)
}

// ListServices returns list of registered services
func (r *DefaultServiceRegistry) ListServices() []string {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    var services []string
    for name := range r.services {
        services = append(services, name)
    }
    return services
}

// IsHealthy returns health status of a service
func (r *DefaultServiceRegistry) IsHealthy(serviceName string) bool {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    return r.healthStatus[serviceName]
}

// StartHealthChecking starts periodic health checking
func (r *DefaultServiceRegistry) StartHealthChecking(ctx context.Context) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            r.performHealthChecks()
        }
    }
}

// performHealthChecks checks health of all registered services
func (r *DefaultServiceRegistry) performHealthChecks() {
    r.mu.RLock()
    services := make(map[string]ServiceInfo)
    for name, service := range r.services {
        services[name] = service
    }
    r.mu.RUnlock()
    
    for name, service := range services {
        healthy := r.checkServiceHealth(service)
        
        r.mu.Lock()
        r.healthStatus[name] = healthy
        r.mu.Unlock()
        
        if !healthy {
            r.logger.Error("Service health check failed", "name", name, "url", service.URL)
        }
    }
}

// checkServiceHealth performs health check for a single service
func (r *DefaultServiceRegistry) checkServiceHealth(service ServiceInfo) bool {
    if service.HealthCheck == "" {
        return true // No health check configured
    }
    
    resp, err := r.client.Get(service.HealthCheck)
    if err != nil {
        return false
    }
    defer resp.Body.Close()
    
    return resp.StatusCode == http.StatusOK
}
```

### 3. User Microservice
```go
package userservice

import (
    "context"
    "encoding/json"
    "net/http"
    "strconv"
    "time"
    
    "github.com/gorilla/mux"
    "github.com/golang-jwt/jwt/v4"
)

// UserService handles user-related operations
type UserService struct {
    repository UserRepository
    jwtSecret  []byte
    logger     Logger
}

// NewUserService creates a new user service
func NewUserService(repository UserRepository, jwtSecret []byte, logger Logger) *UserService {
    return &UserService{
        repository: repository,
        jwtSecret:  jwtSecret,
        logger:     logger,
    }
}

// SetupRoutes configures HTTP routes
func (s *UserService) SetupRoutes() *mux.Router {
    router := mux.NewRouter()
    
    // Public endpoints
    router.HandleFunc("/api/users/register", s.register).Methods("POST")
    router.HandleFunc("/api/users/login", s.login).Methods("POST")
    router.HandleFunc("/health", s.healthCheck).Methods("GET")
    
    // Protected endpoints
    protected := router.PathPrefix("/api/users").Subrouter()
    protected.Use(s.authMiddleware)
    protected.HandleFunc("", s.getUsers).Methods("GET")
    protected.HandleFunc("/{id}", s.getUser).Methods("GET")
    protected.HandleFunc("/{id}", s.updateUser).Methods("PUT")
    protected.HandleFunc("/{id}", s.deleteUser).Methods("DELETE")
    
    return router
}

// register handles user registration
func (s *UserService) register(w http.ResponseWriter, r *http.Request) {
    var req RegisterRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        s.writeErrorResponse(w, "Invalid request body", http.StatusBadRequest)
        return
    }
    
    // Validate request
    if err := s.validateRegisterRequest(req); err != nil {
        s.writeErrorResponse(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    // Check if user already exists
    existing, _ := s.repository.GetByEmail(req.Email)
    if existing != nil {
        s.writeErrorResponse(w, "User already exists", http.StatusConflict)
        return
    }
    
    // Create user
    user := &User{
        Name:      req.Name,
        Email:     req.Email,
        Password:  s.hashPassword(req.Password),
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
    }
    
    if err := s.repository.Create(user); err != nil {
        s.logger.Error("Failed to create user", "error", err)
        s.writeErrorResponse(w, "Internal server error", http.StatusInternalServerError)
        return
    }
    
    // Generate JWT token
    token, err := s.generateJWT(user.ID)
    if err != nil {
        s.logger.Error("Failed to generate token", "error", err)
        s.writeErrorResponse(w, "Internal server error", http.StatusInternalServerError)
        return
    }
    
    response := RegisterResponse{
        ID:    user.ID,
        Name:  user.Name,
        Email: user.Email,
        Token: token,
    }
    
    s.writeJSONResponse(w, response, http.StatusCreated)
}

// login handles user authentication
func (s *UserService) login(w http.ResponseWriter, r *http.Request) {
    var req LoginRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        s.writeErrorResponse(w, "Invalid request body", http.StatusBadRequest)
        return
    }
    
    // Find user by email
    user, err := s.repository.GetByEmail(req.Email)
    if err != nil {
        s.writeErrorResponse(w, "Invalid credentials", http.StatusUnauthorized)
        return
    }
    
    // Verify password
    if !s.verifyPassword(req.Password, user.Password) {
        s.writeErrorResponse(w, "Invalid credentials", http.StatusUnauthorized)
        return
    }
    
    // Generate JWT token
    token, err := s.generateJWT(user.ID)
    if err != nil {
        s.logger.Error("Failed to generate token", "error", err)
        s.writeErrorResponse(w, "Internal server error", http.StatusInternalServerError)
        return
    }
    
    response := LoginResponse{
        ID:    user.ID,
        Name:  user.Name,
        Email: user.Email,
        Token: token,
    }
    
    s.writeJSONResponse(w, response, http.StatusOK)
}

// getUsers handles getting all users
func (s *UserService) getUsers(w http.ResponseWriter, r *http.Request) {
    users, err := s.repository.GetAll()
    if err != nil {
        s.logger.Error("Failed to get users", "error", err)
        s.writeErrorResponse(w, "Internal server error", http.StatusInternalServerError)
        return
    }
    
    var response []UserResponse
    for _, user := range users {
        response = append(response, UserResponse{
            ID:        user.ID,
            Name:      user.Name,
            Email:     user.Email,
            CreatedAt: user.CreatedAt,
        })
    }
    
    s.writeJSONResponse(w, response, http.StatusOK)
}

// getUser handles getting a specific user
func (s *UserService) getUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    userID, err := strconv.Atoi(vars["id"])
    if err != nil {
        s.writeErrorResponse(w, "Invalid user ID", http.StatusBadRequest)
        return
    }
    
    user, err := s.repository.GetByID(userID)
    if err != nil {
        s.writeErrorResponse(w, "User not found", http.StatusNotFound)
        return
    }
    
    response := UserResponse{
        ID:        user.ID,
        Name:      user.Name,
        Email:     user.Email,
        CreatedAt: user.CreatedAt,
    }
    
    s.writeJSONResponse(w, response, http.StatusOK)
}

// authMiddleware validates JWT tokens
func (s *UserService) authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            s.writeErrorResponse(w, "Authorization header required", http.StatusUnauthorized)
            return
        }
        
        tokenString := authHeader[7:] // Remove "Bearer " prefix
        
        token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
            return s.jwtSecret, nil
        })
        
        if err != nil || !token.Valid {
            s.writeErrorResponse(w, "Invalid token", http.StatusUnauthorized)
            return
        }
        
        claims, ok := token.Claims.(jwt.MapClaims)
        if !ok {
            s.writeErrorResponse(w, "Invalid token claims", http.StatusUnauthorized)
            return
        }
        
        userID := int(claims["user_id"].(float64))
        ctx := context.WithValue(r.Context(), "user_id", userID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// generateJWT generates a JWT token for a user
func (s *UserService) generateJWT(userID int) (string, error) {
    claims := jwt.MapClaims{
        "user_id": userID,
        "exp":     time.Now().Add(24 * time.Hour).Unix(),
        "iat":     time.Now().Unix(),
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(s.jwtSecret)
}

// hashPassword hashes a password (simplified - use bcrypt in production)
func (s *UserService) hashPassword(password string) string {
    return "hashed_" + password
}

// verifyPassword verifies a password against hash
func (s *UserService) verifyPassword(password, hash string) bool {
    return s.hashPassword(password) == hash
}

// healthCheck handles health check requests
func (s *UserService) healthCheck(w http.ResponseWriter, r *http.Request) {
    response := map[string]interface{}{
        "status":    "healthy",
        "service":   "user-service",
        "timestamp": time.Now(),
    }
    
    s.writeJSONResponse(w, response, http.StatusOK)
}

// Helper methods for JSON responses
func (s *UserService) writeJSONResponse(w http.ResponseWriter, data interface{}, statusCode int) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(statusCode)
    json.NewEncoder(w).Encode(data)
}

func (s *UserService) writeErrorResponse(w http.ResponseWriter, message string, statusCode int) {
    response := map[string]string{
        "error":  message,
        "status": "error",
    }
    s.writeJSONResponse(w, response, statusCode)
}

// validateRegisterRequest validates registration request
func (s *UserService) validateRegisterRequest(req RegisterRequest) error {
    if req.Name == "" {
        return fmt.Errorf("name is required")
    }
    if req.Email == "" {
        return fmt.Errorf("email is required")
    }
    if req.Password == "" {
        return fmt.Errorf("password is required")
    }
    if len(req.Password) < 6 {
        return fmt.Errorf("password must be at least 6 characters")
    }
    return nil
}

// Domain models and DTOs
type User struct {
    ID        int       `json:"id" db:"id"`
    Name      string    `json:"name" db:"name"`
    Email     string    `json:"email" db:"email"`
    Password  string    `json:"-" db:"password"`
    CreatedAt time.Time `json:"created_at" db:"created_at"`
    UpdatedAt time.Time `json:"updated_at" db:"updated_at"`
}

type RegisterRequest struct {
    Name     string `json:"name"`
    Email    string `json:"email"`
    Password string `json:"password"`
}

type RegisterResponse struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
    Token string `json:"token"`
}

type LoginRequest struct {
    Email    string `json:"email"`
    Password string `json:"password"`
}

type LoginResponse struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
    Token string `json:"token"`
}

type UserResponse struct {
    ID        int       `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
}

// UserRepository interface
type UserRepository interface {
    Create(user *User) error
    GetByID(id int) (*User, error)
    GetByEmail(email string) (*User, error)
    GetAll() ([]*User, error)
    Update(user *User) error
    Delete(id int) error
}
```

### 4. Order Microservice
```go
package orderservice

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "strconv"
    "time"
    
    "github.com/gorilla/mux"
)

// OrderService handles order-related operations
type OrderService struct {
    repository      OrderRepository
    paymentClient   PaymentClient
    inventoryClient InventoryClient
    eventBus        EventBus
    logger          Logger
}

// NewOrderService creates a new order service
func NewOrderService(
    repository OrderRepository,
    paymentClient PaymentClient,
    inventoryClient InventoryClient,
    eventBus EventBus,
    logger Logger,
) *OrderService {
    return &OrderService{
        repository:      repository,
        paymentClient:   paymentClient,
        inventoryClient: inventoryClient,
        eventBus:        eventBus,
        logger:          logger,
    }
}

// SetupRoutes configures HTTP routes
func (s *OrderService) SetupRoutes() *mux.Router {
    router := mux.NewRouter()
    
    router.HandleFunc("/api/orders", s.createOrder).Methods("POST")
    router.HandleFunc("/api/orders", s.getOrders).Methods("GET")
    router.HandleFunc("/api/orders/{id}", s.getOrder).Methods("GET")
    router.HandleFunc("/api/orders/{id}/status", s.updateOrderStatus).Methods("PUT")
    router.HandleFunc("/api/orders/{id}/cancel", s.cancelOrder).Methods("POST")
    router.HandleFunc("/health", s.healthCheck).Methods("GET")
    
    return router
}

// createOrder handles order creation
func (s *OrderService) createOrder(w http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        s.writeErrorResponse(w, "Invalid request body", http.StatusBadRequest)
        return
    }
    
    // Validate request
    if err := s.validateCreateOrderRequest(req); err != nil {
        s.writeErrorResponse(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    // Check inventory availability
    for _, item := range req.Items {
        available, err := s.inventoryClient.CheckAvailability(item.ProductID, item.Quantity)
        if err != nil {
            s.logger.Error("Failed to check inventory", "error", err)
            s.writeErrorResponse(w, "Inventory service unavailable", http.StatusServiceUnavailable)
            return
        }
        
        if !available {
            s.writeErrorResponse(w, fmt.Sprintf("Insufficient inventory for product %d", item.ProductID), http.StatusBadRequest)
            return
        }
    }
    
    // Create order
    order := &Order{
        CustomerID:  req.CustomerID,
        Items:       req.Items,
        TotalAmount: s.calculateTotal(req.Items),
        Status:      "pending",
        CreatedAt:   time.Now(),
        UpdatedAt:   time.Now(),
    }
    
    if err := s.repository.Create(order); err != nil {
        s.logger.Error("Failed to create order", "error", err)
        s.writeErrorResponse(w, "Internal server error", http.StatusInternalServerError)
        return
    }
    
    // Reserve inventory
    for _, item := range req.Items {
        if err := s.inventoryClient.ReserveItem(item.ProductID, item.Quantity); err != nil {
            s.logger.Error("Failed to reserve inventory", "error", err)
            // In production, implement saga pattern for compensation
        }
    }
    
    // Publish order created event
    event := OrderCreatedEvent{
        OrderID:     order.ID,
        CustomerID:  order.CustomerID,
        TotalAmount: order.TotalAmount,
        Items:       order.Items,
        Timestamp:   time.Now(),
    }
    
    if err := s.eventBus.Publish("order.created", event); err != nil {
        s.logger.Error("Failed to publish event", "error", err)
    }
    
    response := OrderResponse{
        ID:          order.ID,
        CustomerID:  order.CustomerID,
        Items:       order.Items,
        TotalAmount: order.TotalAmount,
        Status:      order.Status,
        CreatedAt:   order.CreatedAt,
    }
    
    s.writeJSONResponse(w, response, http.StatusCreated)
}

// getOrders handles getting orders
func (s *OrderService) getOrders(w http.ResponseWriter, r *http.Request) {
    customerIDStr := r.URL.Query().Get("customer_id")
    var orders []*Order
    var err error
    
    if customerIDStr != "" {
        customerID, parseErr := strconv.Atoi(customerIDStr)
        if parseErr != nil {
            s.writeErrorResponse(w, "Invalid customer ID", http.StatusBadRequest)
            return
        }
        orders, err = s.repository.GetByCustomerID(customerID)
    } else {
        orders, err = s.repository.GetAll()
    }
    
    if err != nil {
        s.logger.Error("Failed to get orders", "error", err)
        s.writeErrorResponse(w, "Internal server error", http.StatusInternalServerError)
        return
    }
    
    var response []OrderResponse
    for _, order := range orders {
        response = append(response, OrderResponse{
            ID:          order.ID,
            CustomerID:  order.CustomerID,
            Items:       order.Items,
            TotalAmount: order.TotalAmount,
            Status:      order.Status,
            CreatedAt:   order.CreatedAt,
        })
    }
    
    s.writeJSONResponse(w, response, http.StatusOK)
}

// processPayment handles payment processing for an order
func (s *OrderService) processPayment(orderID int, paymentInfo PaymentInfo) error {
    order, err := s.repository.GetByID(orderID)
    if err != nil {
        return err
    }
    
    // Process payment through payment service
    paymentRequest := PaymentRequest{
        OrderID:       orderID,
        Amount:        order.TotalAmount,
        PaymentMethod: paymentInfo.PaymentMethod,
        CardToken:     paymentInfo.CardToken,
    }
    
    paymentResult, err := s.paymentClient.ProcessPayment(paymentRequest)
    if err != nil {
        s.logger.Error("Payment processing failed", "orderID", orderID, "error", err)
        
        // Update order status to payment failed
        order.Status = "payment_failed"
        s.repository.Update(order)
        
        // Release reserved inventory
        for _, item := range order.Items {
            s.inventoryClient.ReleaseReservation(item.ProductID, item.Quantity)
        }
        
        return err
    }
    
    // Update order with payment information
    order.PaymentID = paymentResult.PaymentID
    order.Status = "paid"
    order.UpdatedAt = time.Now()
    
    if err := s.repository.Update(order); err != nil {
        s.logger.Error("Failed to update order", "error", err)
        return err
    }
    
    // Publish payment processed event
    event := PaymentProcessedEvent{
        OrderID:   orderID,
        PaymentID: paymentResult.PaymentID,
        Amount:    order.TotalAmount,
        Status:    "success",
        Timestamp: time.Now(),
    }
    
    s.eventBus.Publish("payment.processed", event)
    
    return nil
}

// calculateTotal calculates order total amount
func (s *OrderService) calculateTotal(items []OrderItem) float64 {
    var total float64
    for _, item := range items {
        total += item.UnitPrice * float64(item.Quantity)
    }
    return total
}

// validateCreateOrderRequest validates order creation request
func (s *OrderService) validateCreateOrderRequest(req CreateOrderRequest) error {
    if req.CustomerID <= 0 {
        return fmt.Errorf("customer ID is required")
    }
    if len(req.Items) == 0 {
        return fmt.Errorf("order items are required")
    }
    for _, item := range req.Items {
        if item.ProductID <= 0 {
            return fmt.Errorf("product ID is required")
        }
        if item.Quantity <= 0 {
            return fmt.Errorf("quantity must be positive")
        }
        if item.UnitPrice <= 0 {
            return fmt.Errorf("unit price must be positive")
        }
    }
    return nil
}

// healthCheck handles health check requests
func (s *OrderService) healthCheck(w http.ResponseWriter, r *http.Request) {
    response := map[string]interface{}{
        "status":    "healthy",
        "service":   "order-service",
        "timestamp": time.Now(),
    }
    
    s.writeJSONResponse(w, response, http.StatusOK)
}

// Helper methods
func (s *OrderService) writeJSONResponse(w http.ResponseWriter, data interface{}, statusCode int) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(statusCode)
    json.NewEncoder(w).Encode(data)
}

func (s *OrderService) writeErrorResponse(w http.ResponseWriter, message string, statusCode int) {
    response := map[string]string{
        "error":  message,
        "status": "error",
    }
    s.writeJSONResponse(w, response, statusCode)
}

// Domain models and DTOs
type Order struct {
    ID          int         `json:"id" db:"id"`
    CustomerID  int         `json:"customer_id" db:"customer_id"`
    Items       []OrderItem `json:"items" db:"items"`
    TotalAmount float64     `json:"total_amount" db:"total_amount"`
    Status      string      `json:"status" db:"status"`
    PaymentID   string      `json:"payment_id" db:"payment_id"`
    CreatedAt   time.Time   `json:"created_at" db:"created_at"`
    UpdatedAt   time.Time   `json:"updated_at" db:"updated_at"`
}

type OrderItem struct {
    ProductID int     `json:"product_id"`
    Quantity  int     `json:"quantity"`
    UnitPrice float64 `json:"unit_price"`
}

type CreateOrderRequest struct {
    CustomerID int         `json:"customer_id"`
    Items      []OrderItem `json:"items"`
}

type OrderResponse struct {
    ID          int         `json:"id"`
    CustomerID  int         `json:"customer_id"`
    Items       []OrderItem `json:"items"`
    TotalAmount float64     `json:"total_amount"`
    Status      string      `json:"status"`
    CreatedAt   time.Time   `json:"created_at"`
}

type PaymentInfo struct {
    PaymentMethod string `json:"payment_method"`
    CardToken     string `json:"card_token"`
}

type PaymentRequest struct {
    OrderID       int     `json:"order_id"`
    Amount        float64 `json:"amount"`
    PaymentMethod string  `json:"payment_method"`
    CardToken     string  `json:"card_token"`
}

type PaymentResult struct {
    PaymentID string `json:"payment_id"`
    Status    string `json:"status"`
}

// Events
type OrderCreatedEvent struct {
    OrderID     int         `json:"order_id"`
    CustomerID  int         `json:"customer_id"`
    TotalAmount float64     `json:"total_amount"`
    Items       []OrderItem `json:"items"`
    Timestamp   time.Time   `json:"timestamp"`
}

type PaymentProcessedEvent struct {
    OrderID   int       `json:"order_id"`
    PaymentID string    `json:"payment_id"`
    Amount    float64   `json:"amount"`
    Status    string    `json:"status"`
    Timestamp time.Time `json:"timestamp"`
}

// Client interfaces
type PaymentClient interface {
    ProcessPayment(request PaymentRequest) (*PaymentResult, error)
}

type InventoryClient interface {
    CheckAvailability(productID, quantity int) (bool, error)
    ReserveItem(productID, quantity int) error
    ReleaseReservation(productID, quantity int) error
}

type EventBus interface {
    Publish(eventType string, event interface{}) error
}

type OrderRepository interface {
    Create(order *Order) error
    GetByID(id int) (*Order, error)
    GetByCustomerID(customerID int) ([]*Order, error)
    GetAll() ([]*Order, error)
    Update(order *Order) error
    Delete(id int) error
}
```

## Benefits

### Independent Deployment
- Services can be deployed independently
- Faster release cycles
- Reduced deployment risk

### Technology Diversity
- Different technologies for different services
- Choose the best tool for each job
- Team autonomy in technology decisions

### Scalability
- Scale services independently based on demand
- Resource optimization
- Better performance under load

### Fault Isolation
- Failure in one service doesn't bring down entire system
- Better system resilience
- Graceful degradation

### Team Autonomy
- Teams can work independently
- Clear service boundaries
- Faster development cycles

## Drawbacks

### Distributed System Complexity
- Network latency and reliability issues
- Data consistency challenges
- Debugging distributed transactions

### Operational Overhead
- Multiple deployment pipelines
- Service monitoring and management
- Infrastructure complexity

### Data Management Challenges
- No ACID transactions across services
- Eventual consistency requirements
- Data synchronization issues

### Testing Complexity
- Integration testing across services
- Contract testing requirements
- End-to-end testing challenges

## When to Use

### Suitable Scenarios
- **Large applications**: Complex domains with multiple business areas
- **Large teams**: Multiple development teams working in parallel
- **Independent scalability**: Different services have different load patterns
- **Technology diversity**: Different parts benefit from different technologies

### Not Suitable Scenarios
- **Small applications**: Simple monoliths are often better
- **Small teams**: Overhead outweighs benefits
- **Strong consistency requirements**: ACID transactions needed
- **Simple domains**: Limited business complexity

## Implementation Patterns

### Service Mesh
```go
// Service mesh configuration for inter-service communication
type ServiceMeshConfig struct {
    ProxyPort      int
    MetricsPort    int
    TracingEnabled bool
    mTLSEnabled    bool
}

// Sidecar proxy for service mesh
type SidecarProxy struct {
    config      ServiceMeshConfig
    router      Router
    loadBalancer LoadBalancer
    circuitBreaker CircuitBreaker
}
```

### Event-Driven Communication
```go
// Asynchronous event handling
type EventHandler struct {
    orderService OrderService
}

func (h *EventHandler) HandleUserRegistered(event UserRegisteredEvent) {
    // Send welcome email
    // Create user preferences
    // Initialize user profile
}

func (h *EventHandler) HandlePaymentProcessed(event PaymentProcessedEvent) {
    // Update order status
    // Trigger shipping
    // Send confirmation
}
```

### Circuit Breaker Pattern
```go
// Circuit breaker for service resilience
type CircuitBreaker struct {
    failureThreshold int
    timeout          time.Duration
    state            CircuitState
    failureCount     int
}

func (cb *CircuitBreaker) Execute(serviceName string, operation func() error) error {
    if cb.state == Open {
        return ErrCircuitOpen
    }
    
    err := operation()
    if err != nil {
        cb.onFailure()
        return err
    }
    
    cb.onSuccess()
    return nil
}
```

## Related Patterns
- **API Gateway Pattern**: Single entry point for clients
- **Service Registry Pattern**: Service discovery mechanism
- **Circuit Breaker Pattern**: Fault tolerance
- **Event Sourcing**: Event-driven state management
- **CQRS**: Command and query separation
- **Saga Pattern**: Distributed transaction management

## Best Practices

### 1. Service Design
- Single responsibility principle
- Business-aligned service boundaries
- API-first design approach
- Versioning strategy

### 2. Data Management
- Database per service
- Eventual consistency acceptance
- Event-driven data synchronization
- Compensation patterns for failures

### 3. Communication
- Asynchronous communication preferred
- Synchronous for real-time requirements
- Circuit breaker for resilience
- Timeout and retry strategies

### 4. Monitoring and Observability
- Distributed tracing
- Centralized logging
- Service health monitoring
- Business metrics tracking

## References
- [Microservices.io - Chris Richardson](https://microservices.io/)
- [Building Microservices - Sam Newman](https://samnewman.io/books/building_microservices/)
- [Microservices Patterns - Chris Richardson](https://microservices.io/book)
- [Martin Fowler - Microservices](https://martinfowler.com/articles/microservices.html)
