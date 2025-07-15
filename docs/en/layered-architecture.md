# Layered Architecture

## Overview
Layered Architecture (also known as n-tier architecture) is a structural pattern that separates software into logical layers. Each layer has specific responsibilities and communicates only with adjacent layers, achieving separation of concerns and maintainability.

## Basic Structure

### Typical 4-Layer Structure
```
┌─────────────────────────────────┐
│     Presentation Layer          │ ← UI/Controllers
├─────────────────────────────────┤
│     Business Logic Layer        │ ← Domain/Service Logic
├─────────────────────────────────┤
│     Data Access Layer           │ ← Repository/DAO
├─────────────────────────────────┤
│     Database Layer              │ ← Data Storage
└─────────────────────────────────┘
```

## Layer Responsibilities

### 1. Presentation Layer
- **Responsibility**: User interface and user interaction handling
- **Contains**:
  - Web controllers
  - API endpoints
  - View templates
  - Input validation
  - Response formatting

```go
package presentation

import (
    "encoding/json"
    "net/http"
    "strconv"
    
    "github.com/gorilla/mux"
)

// UserController handles user-related HTTP requests
type UserController struct {
    userService UserService
}

// NewUserController creates a new user controller
func NewUserController(userService UserService) *UserController {
    return &UserController{
        userService: userService,
    }
}

// CreateUser handles user creation requests
func (c *UserController) CreateUser(w http.ResponseWriter, r *http.Request) {
    var request CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }
    
    // Input validation
    if err := c.validateCreateUserRequest(request); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    // Call business layer
    user, err := c.userService.CreateUser(request.Name, request.Email, request.Password)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    // Format response
    response := UserResponse{
        ID:    user.ID,
        Name:  user.Name,
        Email: user.Email,
    }
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(response)
}

// GetUser handles user retrieval requests
func (c *UserController) GetUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    userID, err := strconv.Atoi(vars["id"])
    if err != nil {
        http.Error(w, "Invalid user ID", http.StatusBadRequest)
        return
    }
    
    user, err := c.userService.GetUserByID(userID)
    if err != nil {
        if err == ErrUserNotFound {
            http.Error(w, "User not found", http.StatusNotFound)
            return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    response := UserResponse{
        ID:    user.ID,
        Name:  user.Name,
        Email: user.Email,
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}

// validateCreateUserRequest validates user creation request
func (c *UserController) validateCreateUserRequest(request CreateUserRequest) error {
    if request.Name == "" {
        return errors.New("name is required")
    }
    if request.Email == "" {
        return errors.New("email is required")
    }
    if request.Password == "" {
        return errors.New("password is required")
    }
    if len(request.Password) < 8 {
        return errors.New("password must be at least 8 characters")
    }
    return nil
}

// CreateUserRequest represents user creation request
type CreateUserRequest struct {
    Name     string `json:"name"`
    Email    string `json:"email"`
    Password string `json:"password"`
}

// UserResponse represents user response
type UserResponse struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

// OrderController handles order-related HTTP requests
type OrderController struct {
    orderService OrderService
}

// NewOrderController creates a new order controller
func NewOrderController(orderService OrderService) *OrderController {
    return &OrderController{
        orderService: orderService,
    }
}

// CreateOrder handles order creation requests
func (c *OrderController) CreateOrder(w http.ResponseWriter, r *http.Request) {
    var request CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }
    
    order, err := c.orderService.CreateOrder(request.UserID, request.Items)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    response := OrderResponse{
        ID:          order.ID,
        UserID:      order.UserID,
        TotalAmount: order.TotalAmount,
        Status:      order.Status,
        Items:       order.Items,
    }
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(response)
}

// CreateOrderRequest represents order creation request
type CreateOrderRequest struct {
    UserID int         `json:"user_id"`
    Items  []OrderItem `json:"items"`
}

// OrderResponse represents order response
type OrderResponse struct {
    ID          int         `json:"id"`
    UserID      int         `json:"user_id"`
    TotalAmount float64     `json:"total_amount"`
    Status      string      `json:"status"`
    Items       []OrderItem `json:"items"`
}
```

### 2. Business Logic Layer
- **Responsibility**: Core business logic and rules
- **Contains**:
  - Business services
  - Domain models
  - Business rules
  - Workflow orchestration

```go
package business

import (
    "errors"
    "time"
)

// UserService handles user business logic
type UserService struct {
    userRepository UserRepository
    emailService   EmailService
    logger         Logger
}

// NewUserService creates a new user service
func NewUserService(userRepo UserRepository, emailService EmailService, logger Logger) *UserService {
    return &UserService{
        userRepository: userRepo,
        emailService:   emailService,
        logger:         logger,
    }
}

// CreateUser creates a new user with business rules
func (s *UserService) CreateUser(name, email, password string) (*User, error) {
    s.logger.Info("Creating user", "email", email)
    
    // Business rule: Check if email already exists
    existingUser, err := s.userRepository.FindByEmail(email)
    if err != nil && err != ErrUserNotFound {
        return nil, err
    }
    if existingUser != nil {
        return nil, errors.New("email already exists")
    }
    
    // Business rule: Password encryption
    hashedPassword, err := s.hashPassword(password)
    if err != nil {
        return nil, err
    }
    
    // Create user entity
    user := &User{
        Name:      name,
        Email:     email,
        Password:  hashedPassword,
        CreatedAt: time.Now(),
        Active:    true,
    }
    
    // Save user
    if err := s.userRepository.Save(user); err != nil {
        return nil, err
    }
    
    // Business rule: Send welcome email
    if err := s.emailService.SendWelcomeEmail(user.Email, user.Name); err != nil {
        s.logger.Error("Failed to send welcome email", "error", err)
        // Don't fail user creation if email fails
    }
    
    s.logger.Info("User created successfully", "userID", user.ID)
    return user, nil
}

// GetUserByID retrieves user by ID
func (s *UserService) GetUserByID(id int) (*User, error) {
    user, err := s.userRepository.FindByID(id)
    if err != nil {
        return nil, err
    }
    
    // Business rule: Don't return inactive users
    if !user.Active {
        return nil, ErrUserNotFound
    }
    
    return user, nil
}

// UpdateUser updates user information
func (s *UserService) UpdateUser(id int, name, email string) (*User, error) {
    user, err := s.userRepository.FindByID(id)
    if err != nil {
        return nil, err
    }
    
    // Business rule: Email uniqueness check (if changed)
    if user.Email != email {
        existingUser, err := s.userRepository.FindByEmail(email)
        if err != nil && err != ErrUserNotFound {
            return nil, err
        }
        if existingUser != nil && existingUser.ID != id {
            return nil, errors.New("email already exists")
        }
    }
    
    // Update user
    user.Name = name
    user.Email = email
    user.UpdatedAt = time.Now()
    
    if err := s.userRepository.Save(user); err != nil {
        return nil, err
    }
    
    return user, nil
}

// DeactivateUser deactivates a user
func (s *UserService) DeactivateUser(id int) error {
    user, err := s.userRepository.FindByID(id)
    if err != nil {
        return err
    }
    
    // Business rule: Cancel all active orders before deactivation
    activeOrders, err := s.userRepository.FindActiveOrdersByUserID(id)
    if err != nil {
        return err
    }
    
    if len(activeOrders) > 0 {
        return errors.New("cannot deactivate user with active orders")
    }
    
    user.Active = false
    user.UpdatedAt = time.Now()
    
    return s.userRepository.Save(user)
}

// hashPassword hashes password using bcrypt
func (s *UserService) hashPassword(password string) (string, error) {
    // Implementation would use bcrypt or similar
    return "hashed_" + password, nil
}

// OrderService handles order business logic
type OrderService struct {
    orderRepository   OrderRepository
    productRepository ProductRepository
    userRepository    UserRepository
    paymentService    PaymentService
    logger           Logger
}

// NewOrderService creates a new order service
func NewOrderService(
    orderRepo OrderRepository,
    productRepo ProductRepository,
    userRepo UserRepository,
    paymentService PaymentService,
    logger Logger,
) *OrderService {
    return &OrderService{
        orderRepository:   orderRepo,
        productRepository: productRepo,
        userRepository:    userRepo,
        paymentService:    paymentService,
        logger:           logger,
    }
}

// CreateOrder creates a new order with business validation
func (s *OrderService) CreateOrder(userID int, items []OrderItem) (*Order, error) {
    s.logger.Info("Creating order", "userID", userID)
    
    // Business rule: Validate user exists and is active
    user, err := s.userRepository.FindByID(userID)
    if err != nil {
        return nil, err
    }
    if !user.Active {
        return nil, errors.New("user is not active")
    }
    
    // Business rule: Validate items and calculate total
    var totalAmount float64
    for i, item := range items {
        product, err := s.productRepository.FindByID(item.ProductID)
        if err != nil {
            return nil, err
        }
        
        // Business rule: Check stock availability
        if product.Stock < item.Quantity {
            return nil, errors.New("insufficient stock for product: " + product.Name)
        }
        
        // Update item with current price
        items[i].UnitPrice = product.Price
        items[i].TotalPrice = product.Price * float64(item.Quantity)
        totalAmount += items[i].TotalPrice
    }
    
    // Business rule: Minimum order amount
    if totalAmount < 10.0 {
        return nil, errors.New("minimum order amount is $10.00")
    }
    
    // Create order
    order := &Order{
        UserID:      userID,
        Items:       items,
        TotalAmount: totalAmount,
        Status:      "pending",
        CreatedAt:   time.Now(),
    }
    
    // Save order
    if err := s.orderRepository.Save(order); err != nil {
        return nil, err
    }
    
    // Business rule: Reserve stock
    if err := s.reserveStock(items); err != nil {
        // Rollback order creation
        s.orderRepository.Delete(order.ID)
        return nil, err
    }
    
    s.logger.Info("Order created successfully", "orderID", order.ID)
    return order, nil
}

// ProcessPayment processes payment for an order
func (s *OrderService) ProcessPayment(orderID int, paymentDetails PaymentDetails) error {
    order, err := s.orderRepository.FindByID(orderID)
    if err != nil {
        return err
    }
    
    // Business rule: Only pending orders can be paid
    if order.Status != "pending" {
        return errors.New("order is not in pending status")
    }
    
    // Process payment
    paymentResult, err := s.paymentService.ProcessPayment(order.TotalAmount, paymentDetails)
    if err != nil {
        return err
    }
    
    if paymentResult.Success {
        order.Status = "paid"
        order.PaymentID = paymentResult.PaymentID
        order.UpdatedAt = time.Now()
    } else {
        order.Status = "payment_failed"
        order.UpdatedAt = time.Now()
        
        // Release reserved stock
        s.releaseStock(order.Items)
    }
    
    return s.orderRepository.Save(order)
}

// reserveStock reserves stock for order items
func (s *OrderService) reserveStock(items []OrderItem) error {
    for _, item := range items {
        if err := s.productRepository.ReserveStock(item.ProductID, item.Quantity); err != nil {
            // Rollback previous reservations
            for i := range items {
                if i >= len(items) {
                    break
                }
                s.productRepository.ReleaseStock(items[i].ProductID, items[i].Quantity)
            }
            return err
        }
    }
    return nil
}

// releaseStock releases reserved stock
func (s *OrderService) releaseStock(items []OrderItem) {
    for _, item := range items {
        s.productRepository.ReleaseStock(item.ProductID, item.Quantity)
    }
}

// User domain model
type User struct {
    ID        int       `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    Password  string    `json:"-"`
    Active    bool      `json:"active"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

// Order domain model
type Order struct {
    ID          int         `json:"id"`
    UserID      int         `json:"user_id"`
    Items       []OrderItem `json:"items"`
    TotalAmount float64     `json:"total_amount"`
    Status      string      `json:"status"`
    PaymentID   string      `json:"payment_id,omitempty"`
    CreatedAt   time.Time   `json:"created_at"`
    UpdatedAt   time.Time   `json:"updated_at"`
}

// OrderItem represents an item in an order
type OrderItem struct {
    ProductID  int     `json:"product_id"`
    Quantity   int     `json:"quantity"`
    UnitPrice  float64 `json:"unit_price"`
    TotalPrice float64 `json:"total_price"`
}

// Product domain model
type Product struct {
    ID    int     `json:"id"`
    Name  string  `json:"name"`
    Price float64 `json:"price"`
    Stock int     `json:"stock"`
}
```

### 3. Data Access Layer
- **Responsibility**: Data persistence and retrieval
- **Contains**:
  - Repository implementations
  - Data mappers
  - Database queries
  - Transaction management

```go
package dataaccess

import (
    "database/sql"
    "errors"
    "time"
    
    _ "github.com/lib/pq"
)

// UserRepository implements user data access
type UserRepository struct {
    db *sql.DB
}

// NewUserRepository creates a new user repository
func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

// Save saves a user to database
func (r *UserRepository) Save(user *User) error {
    if user.ID == 0 {
        return r.create(user)
    }
    return r.update(user)
}

// create creates a new user
func (r *UserRepository) create(user *User) error {
    query := `
        INSERT INTO users (name, email, password, active, created_at, updated_at)
        VALUES ($1, $2, $3, $4, $5, $6)
        RETURNING id`
    
    row := r.db.QueryRow(
        query,
        user.Name,
        user.Email,
        user.Password,
        user.Active,
        user.CreatedAt,
        user.UpdatedAt,
    )
    
    return row.Scan(&user.ID)
}

// update updates an existing user
func (r *UserRepository) update(user *User) error {
    query := `
        UPDATE users 
        SET name = $1, email = $2, active = $3, updated_at = $4
        WHERE id = $5`
    
    result, err := r.db.Exec(
        query,
        user.Name,
        user.Email,
        user.Active,
        time.Now(),
        user.ID,
    )
    if err != nil {
        return err
    }
    
    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return err
    }
    
    if rowsAffected == 0 {
        return ErrUserNotFound
    }
    
    return nil
}

// FindByID finds user by ID
func (r *UserRepository) FindByID(id int) (*User, error) {
    query := `
        SELECT id, name, email, password, active, created_at, updated_at
        FROM users 
        WHERE id = $1`
    
    user := &User{}
    row := r.db.QueryRow(query, id)
    
    err := row.Scan(
        &user.ID,
        &user.Name,
        &user.Email,
        &user.Password,
        &user.Active,
        &user.CreatedAt,
        &user.UpdatedAt,
    )
    
    if err == sql.ErrNoRows {
        return nil, ErrUserNotFound
    }
    if err != nil {
        return nil, err
    }
    
    return user, nil
}

// FindByEmail finds user by email
func (r *UserRepository) FindByEmail(email string) (*User, error) {
    query := `
        SELECT id, name, email, password, active, created_at, updated_at
        FROM users 
        WHERE email = $1`
    
    user := &User{}
    row := r.db.QueryRow(query, email)
    
    err := row.Scan(
        &user.ID,
        &user.Name,
        &user.Email,
        &user.Password,
        &user.Active,
        &user.CreatedAt,
        &user.UpdatedAt,
    )
    
    if err == sql.ErrNoRows {
        return nil, ErrUserNotFound
    }
    if err != nil {
        return nil, err
    }
    
    return user, nil
}

// FindActiveOrdersByUserID finds active orders for user
func (r *UserRepository) FindActiveOrdersByUserID(userID int) ([]*Order, error) {
    query := `
        SELECT id, user_id, total_amount, status, payment_id, created_at, updated_at
        FROM orders 
        WHERE user_id = $1 AND status IN ('pending', 'paid', 'processing')`
    
    rows, err := r.db.Query(query, userID)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var orders []*Order
    for rows.Next() {
        order := &Order{}
        var paymentID sql.NullString
        
        err := rows.Scan(
            &order.ID,
            &order.UserID,
            &order.TotalAmount,
            &order.Status,
            &paymentID,
            &order.CreatedAt,
            &order.UpdatedAt,
        )
        if err != nil {
            return nil, err
        }
        
        if paymentID.Valid {
            order.PaymentID = paymentID.String
        }
        
        // Load order items
        items, err := r.findOrderItems(order.ID)
        if err != nil {
            return nil, err
        }
        order.Items = items
        
        orders = append(orders, order)
    }
    
    return orders, nil
}

// OrderRepository implements order data access
type OrderRepository struct {
    db *sql.DB
}

// NewOrderRepository creates a new order repository
func NewOrderRepository(db *sql.DB) *OrderRepository {
    return &OrderRepository{db: db}
}

// Save saves an order to database
func (r *OrderRepository) Save(order *Order) error {
    tx, err := r.db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    if order.ID == 0 {
        if err := r.createOrder(tx, order); err != nil {
            return err
        }
    } else {
        if err := r.updateOrder(tx, order); err != nil {
            return err
        }
    }
    
    // Save order items
    if err := r.saveOrderItems(tx, order); err != nil {
        return err
    }
    
    return tx.Commit()
}

// createOrder creates a new order
func (r *OrderRepository) createOrder(tx *sql.Tx, order *Order) error {
    query := `
        INSERT INTO orders (user_id, total_amount, status, created_at, updated_at)
        VALUES ($1, $2, $3, $4, $5)
        RETURNING id`
    
    row := tx.QueryRow(
        query,
        order.UserID,
        order.TotalAmount,
        order.Status,
        order.CreatedAt,
        order.UpdatedAt,
    )
    
    return row.Scan(&order.ID)
}

// updateOrder updates an existing order
func (r *OrderRepository) updateOrder(tx *sql.Tx, order *Order) error {
    query := `
        UPDATE orders 
        SET total_amount = $1, status = $2, payment_id = $3, updated_at = $4
        WHERE id = $5`
    
    _, err := tx.Exec(
        query,
        order.TotalAmount,
        order.Status,
        order.PaymentID,
        time.Now(),
        order.ID,
    )
    
    return err
}

// saveOrderItems saves order items
func (r *OrderRepository) saveOrderItems(tx *sql.Tx, order *Order) error {
    // Delete existing items
    _, err := tx.Exec("DELETE FROM order_items WHERE order_id = $1", order.ID)
    if err != nil {
        return err
    }
    
    // Insert new items
    for _, item := range order.Items {
        query := `
            INSERT INTO order_items (order_id, product_id, quantity, unit_price, total_price)
            VALUES ($1, $2, $3, $4, $5)`
        
        _, err := tx.Exec(
            query,
            order.ID,
            item.ProductID,
            item.Quantity,
            item.UnitPrice,
            item.TotalPrice,
        )
        if err != nil {
            return err
        }
    }
    
    return nil
}

// FindByID finds order by ID
func (r *OrderRepository) FindByID(id int) (*Order, error) {
    query := `
        SELECT id, user_id, total_amount, status, payment_id, created_at, updated_at
        FROM orders 
        WHERE id = $1`
    
    order := &Order{}
    var paymentID sql.NullString
    row := r.db.QueryRow(query, id)
    
    err := row.Scan(
        &order.ID,
        &order.UserID,
        &order.TotalAmount,
        &order.Status,
        &paymentID,
        &order.CreatedAt,
        &order.UpdatedAt,
    )
    
    if err == sql.ErrNoRows {
        return nil, ErrOrderNotFound
    }
    if err != nil {
        return nil, err
    }
    
    if paymentID.Valid {
        order.PaymentID = paymentID.String
    }
    
    // Load order items
    items, err := r.findOrderItems(order.ID)
    if err != nil {
        return nil, err
    }
    order.Items = items
    
    return order, nil
}

// findOrderItems finds items for an order
func (r *OrderRepository) findOrderItems(orderID int) ([]OrderItem, error) {
    query := `
        SELECT product_id, quantity, unit_price, total_price
        FROM order_items 
        WHERE order_id = $1`
    
    rows, err := r.db.Query(query, orderID)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var items []OrderItem
    for rows.Next() {
        item := OrderItem{}
        err := rows.Scan(
            &item.ProductID,
            &item.Quantity,
            &item.UnitPrice,
            &item.TotalPrice,
        )
        if err != nil {
            return nil, err
        }
        items = append(items, item)
    }
    
    return items, nil
}

// Delete deletes an order
func (r *OrderRepository) Delete(id int) error {
    tx, err := r.db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    // Delete order items first
    _, err = tx.Exec("DELETE FROM order_items WHERE order_id = $1", id)
    if err != nil {
        return err
    }
    
    // Delete order
    _, err = tx.Exec("DELETE FROM orders WHERE id = $1", id)
    if err != nil {
        return err
    }
    
    return tx.Commit()
}

// Common errors
var (
    ErrUserNotFound  = errors.New("user not found")
    ErrOrderNotFound = errors.New("order not found")
)
```

### 4. Database Layer
- **Responsibility**: Data storage and database management
- **Contains**:
  - Database schema
  - Stored procedures
  - Database configuration
  - Connection management

```go
package database

import (
    "database/sql"
    "fmt"
    "log"
    
    _ "github.com/lib/pq"
)

// DatabaseConfig holds database configuration
type DatabaseConfig struct {
    Host     string
    Port     int
    DBName   string
    Username string
    Password string
    SSLMode  string
}

// DatabaseManager manages database connections and schema
type DatabaseManager struct {
    db     *sql.DB
    config DatabaseConfig
}

// NewDatabaseManager creates a new database manager
func NewDatabaseManager(config DatabaseConfig) *DatabaseManager {
    return &DatabaseManager{
        config: config,
    }
}

// Connect establishes database connection
func (m *DatabaseManager) Connect() error {
    dsn := fmt.Sprintf(
        "host=%s port=%d user=%s password=%s dbname=%s sslmode=%s",
        m.config.Host,
        m.config.Port,
        m.config.Username,
        m.config.Password,
        m.config.DBName,
        m.config.SSLMode,
    )
    
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return fmt.Errorf("failed to open database: %w", err)
    }
    
    if err := db.Ping(); err != nil {
        return fmt.Errorf("failed to ping database: %w", err)
    }
    
    m.db = db
    log.Println("Database connected successfully")
    return nil
}

// GetDB returns database connection
func (m *DatabaseManager) GetDB() *sql.DB {
    return m.db
}

// Close closes database connection
func (m *DatabaseManager) Close() error {
    if m.db != nil {
        return m.db.Close()
    }
    return nil
}

// Migrate runs database migrations
func (m *DatabaseManager) Migrate() error {
    log.Println("Running database migrations...")
    
    // Create users table
    if err := m.createUsersTable(); err != nil {
        return err
    }
    
    // Create orders table
    if err := m.createOrdersTable(); err != nil {
        return err
    }
    
    // Create order_items table
    if err := m.createOrderItemsTable(); err != nil {
        return err
    }
    
    // Create products table
    if err := m.createProductsTable(); err != nil {
        return err
    }
    
    log.Println("Database migrations completed")
    return nil
}

// createUsersTable creates users table
func (m *DatabaseManager) createUsersTable() error {
    query := `
        CREATE TABLE IF NOT EXISTS users (
            id SERIAL PRIMARY KEY,
            name VARCHAR(255) NOT NULL,
            email VARCHAR(255) UNIQUE NOT NULL,
            password VARCHAR(255) NOT NULL,
            active BOOLEAN DEFAULT true,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )`
    
    _, err := m.db.Exec(query)
    if err != nil {
        return fmt.Errorf("failed to create users table: %w", err)
    }
    
    return nil
}

// createOrdersTable creates orders table
func (m *DatabaseManager) createOrdersTable() error {
    query := `
        CREATE TABLE IF NOT EXISTS orders (
            id SERIAL PRIMARY KEY,
            user_id INTEGER NOT NULL REFERENCES users(id),
            total_amount DECIMAL(10,2) NOT NULL,
            status VARCHAR(50) NOT NULL,
            payment_id VARCHAR(255),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )`
    
    _, err := m.db.Exec(query)
    if err != nil {
        return fmt.Errorf("failed to create orders table: %w", err)
    }
    
    return nil
}

// createOrderItemsTable creates order_items table
func (m *DatabaseManager) createOrderItemsTable() error {
    query := `
        CREATE TABLE IF NOT EXISTS order_items (
            id SERIAL PRIMARY KEY,
            order_id INTEGER NOT NULL REFERENCES orders(id),
            product_id INTEGER NOT NULL,
            quantity INTEGER NOT NULL,
            unit_price DECIMAL(10,2) NOT NULL,
            total_price DECIMAL(10,2) NOT NULL
        )`
    
    _, err := m.db.Exec(query)
    if err != nil {
        return fmt.Errorf("failed to create order_items table: %w", err)
    }
    
    return nil
}

// createProductsTable creates products table
func (m *DatabaseManager) createProductsTable() error {
    query := `
        CREATE TABLE IF NOT EXISTS products (
            id SERIAL PRIMARY KEY,
            name VARCHAR(255) NOT NULL,
            price DECIMAL(10,2) NOT NULL,
            stock INTEGER DEFAULT 0,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )`
    
    _, err := m.db.Exec(query)
    if err != nil {
        return fmt.Errorf("failed to create products table: %w", err)
    }
    
    return nil
}
```

## Benefits

### Separation of Concerns
- Each layer has a single responsibility
- Changes in one layer don't affect others
- Clear boundaries between different aspects

### Maintainability
- Easy to understand and modify
- Localized changes
- Reduced complexity

### Testability
- Each layer can be tested independently
- Easy to mock dependencies
- Unit testing friendly

### Reusability
- Business logic can be reused across different presentations
- Data access can be shared across services
- Modular components

## Drawbacks

### Performance Overhead
- Multiple layer traversals
- Additional abstraction costs
- Potential inefficiencies

### Tight Coupling Between Adjacent Layers
- Changes in data model affect all upper layers
- Database changes ripple upward
- Interface dependencies

### Development Overhead
- More initial setup required
- Additional abstraction layers
- Increased code volume

## When to Use

### Suitable Scenarios
- **Complex business applications**: Multiple business rules and workflows
- **Enterprise systems**: Large, long-lived applications
- **Team development**: Clear boundaries for different teams
- **Maintainable systems**: Long-term maintenance requirements

### Not Suitable Scenarios
- **Simple applications**: Basic CRUD operations
- **Performance-critical systems**: Where overhead is unacceptable
- **Rapid prototyping**: Quick development needs
- **Small applications**: Overhead outweighs benefits

## Best Practices

### 1. Layer Independence
- Each layer should only depend on the layer directly below it
- Avoid skip-layer dependencies
- Use dependency injection for loose coupling

### 2. Interface Definition
- Define clear interfaces between layers
- Use abstractions to hide implementation details
- Follow contract-first design

### 3. Error Handling
- Handle errors appropriately at each layer
- Don't let lower-layer exceptions bubble up unchanged
- Provide meaningful error messages

### 4. Data Transformation
- Transform data appropriately at layer boundaries
- Don't expose internal data structures
- Use DTOs for layer communication

## Variations

### Clean Architecture
- Dependencies point inward toward business logic
- Business logic at the center
- External concerns (UI, database) at the edges

### Hexagonal Architecture
- Business logic at the center
- Adapters for external interfaces
- Ports define boundaries

### Onion Architecture
- Similar to Clean Architecture
- Dependencies point toward the core
- Infrastructure at the outer layer

## Related Patterns
- **Repository Pattern**: Data access abstraction
- **Service Layer Pattern**: Business logic organization
- **DTO Pattern**: Data transfer between layers
- **Dependency Injection**: Loose coupling between layers

## References
- [Martin Fowler - Enterprise Application Architecture](https://martinfowler.com/eaaCatalog/)
- [Robert C. Martin - Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Microsoft - Layered Architecture](https://docs.microsoft.com/en-us/azure/architecture/guide/architecture-styles/n-tier)
