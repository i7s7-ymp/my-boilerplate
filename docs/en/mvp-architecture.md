# MVP Architecture (Model-View-Presenter)

## Overview
MVP (Model-View-Presenter) Architecture is a derivative of the MVC pattern where the Presenter handles all UI logic and acts as an intermediary between the Model and View. This pattern promotes better separation of concerns and testability by making the View passive and moving all logic to the Presenter.

## Basic Structure

```
┌─────────────────────────────────────────────────────┐
│                    View                             │
│              (Passive Interface)                    │
├─────────────────────────────────────────────────────┤
│                      ↕                              │
├─────────────────────────────────────────────────────┤
│                  Presenter                          │
│               (UI Logic Handler)                    │
├─────────────────────────────────────────────────────┤
│                      ↕                              │
├─────────────────────────────────────────────────────┤
│                   Model                             │
│              (Business Logic & Data)                │
└─────────────────────────────────────────────────────┘
```

## Core Components

### 1. View Interface
```go
package mvp

// UserView defines the contract for user interface
type UserView interface {
    // Display methods
    ShowUsers(users []UserViewModel)
    ShowUser(user UserViewModel)
    ShowError(message string)
    ShowSuccess(message string)
    ShowLoading()
    HideLoading()
    
    // Input methods
    GetUserInput() UserInputModel
    
    // Validation feedback
    ShowValidationErrors(errors map[string]string)
    ClearValidationErrors()
    
    // Navigation
    NavigateToUserDetail(userID int)
    NavigateToUserEdit(userID int)
    Close()
}

// UserViewModel represents data for view display
type UserViewModel struct {
    ID       int    `json:"id"`
    Name     string `json:"name"`
    Email    string `json:"email"`
    Status   string `json:"status"`
    JoinDate string `json:"join_date"`
}

// UserInputModel represents user input data
type UserInputModel struct {
    Name     string `json:"name"`
    Email    string `json:"email"`
    Password string `json:"password"`
}

// OrderView defines the contract for order interface
type OrderView interface {
    ShowOrders(orders []OrderViewModel)
    ShowOrder(order OrderViewModel)
    ShowError(message string)
    ShowSuccess(message string)
    ShowLoading()
    HideLoading()
    
    GetOrderInput() OrderInputModel
    ShowValidationErrors(errors map[string]string)
    ClearValidationErrors()
    
    NavigateToOrderDetail(orderID int)
    UpdateOrderStatus(orderID int, status string)
}

// OrderViewModel represents order data for view
type OrderViewModel struct {
    ID          int                `json:"id"`
    CustomerName string            `json:"customer_name"`
    TotalAmount string             `json:"total_amount"`
    Status      string             `json:"status"`
    OrderDate   string             `json:"order_date"`
    Items       []OrderItemViewModel `json:"items"`
}

// OrderItemViewModel represents order item for view
type OrderItemViewModel struct {
    ProductName string `json:"product_name"`
    Quantity    int    `json:"quantity"`
    UnitPrice   string `json:"unit_price"`
    TotalPrice  string `json:"total_price"`
}

// OrderInputModel represents order input data
type OrderInputModel struct {
    CustomerID int               `json:"customer_id"`
    Items      []OrderItemInput  `json:"items"`
}

// OrderItemInput represents order item input
type OrderItemInput struct {
    ProductID int `json:"product_id"`
    Quantity  int `json:"quantity"`
}
```

### 2. Presenter Implementation
```go
package mvp

import (
    "fmt"
    "strconv"
    "time"
)

// UserPresenter handles user-related presentation logic
type UserPresenter struct {
    view        UserView
    userService UserService
    validator   UserValidator
}

// NewUserPresenter creates a new user presenter
func NewUserPresenter(view UserView, userService UserService, validator UserValidator) *UserPresenter {
    return &UserPresenter{
        view:        view,
        userService: userService,
        validator:   validator,
    }
}

// LoadUsers loads and displays users
func (p *UserPresenter) LoadUsers() {
    p.view.ShowLoading()
    
    users, err := p.userService.GetAllUsers()
    if err != nil {
        p.view.HideLoading()
        p.view.ShowError("Failed to load users: " + err.Error())
        return
    }
    
    viewModels := p.convertUsersToViewModels(users)
    p.view.HideLoading()
    p.view.ShowUsers(viewModels)
}

// LoadUser loads and displays a specific user
func (p *UserPresenter) LoadUser(userID int) {
    p.view.ShowLoading()
    
    user, err := p.userService.GetUserByID(userID)
    if err != nil {
        p.view.HideLoading()
        p.view.ShowError("Failed to load user: " + err.Error())
        return
    }
    
    viewModel := p.convertUserToViewModel(user)
    p.view.HideLoading()
    p.view.ShowUser(viewModel)
}

// CreateUser handles user creation
func (p *UserPresenter) CreateUser() {
    input := p.view.GetUserInput()
    
    // Validate input
    if errors := p.validator.ValidateUserInput(input); len(errors) > 0 {
        p.view.ShowValidationErrors(errors)
        return
    }
    
    p.view.ClearValidationErrors()
    p.view.ShowLoading()
    
    // Create user through service
    user, err := p.userService.CreateUser(input.Name, input.Email, input.Password)
    if err != nil {
        p.view.HideLoading()
        p.view.ShowError("Failed to create user: " + err.Error())
        return
    }
    
    p.view.HideLoading()
    p.view.ShowSuccess("User created successfully")
    p.view.NavigateToUserDetail(user.ID)
}

// UpdateUser handles user updates
func (p *UserPresenter) UpdateUser(userID int) {
    input := p.view.GetUserInput()
    
    // Validate input
    if errors := p.validator.ValidateUserUpdateInput(input); len(errors) > 0 {
        p.view.ShowValidationErrors(errors)
        return
    }
    
    p.view.ClearValidationErrors()
    p.view.ShowLoading()
    
    // Update user through service
    user, err := p.userService.UpdateUser(userID, input.Name, input.Email)
    if err != nil {
        p.view.HideLoading()
        p.view.ShowError("Failed to update user: " + err.Error())
        return
    }
    
    viewModel := p.convertUserToViewModel(user)
    p.view.HideLoading()
    p.view.ShowSuccess("User updated successfully")
    p.view.ShowUser(viewModel)
}

// DeleteUser handles user deletion
func (p *UserPresenter) DeleteUser(userID int) {
    p.view.ShowLoading()
    
    err := p.userService.DeleteUser(userID)
    if err != nil {
        p.view.HideLoading()
        p.view.ShowError("Failed to delete user: " + err.Error())
        return
    }
    
    p.view.HideLoading()
    p.view.ShowSuccess("User deleted successfully")
    p.LoadUsers() // Refresh the list
}

// OnUserSelected handles user selection
func (p *UserPresenter) OnUserSelected(userID int) {
    p.view.NavigateToUserDetail(userID)
}

// OnEditUser handles edit user action
func (p *UserPresenter) OnEditUser(userID int) {
    p.view.NavigateToUserEdit(userID)
}

// convertUsersToViewModels converts domain models to view models
func (p *UserPresenter) convertUsersToViewModels(users []*User) []UserViewModel {
    viewModels := make([]UserViewModel, len(users))
    for i, user := range users {
        viewModels[i] = p.convertUserToViewModel(user)
    }
    return viewModels
}

// convertUserToViewModel converts domain model to view model
func (p *UserPresenter) convertUserToViewModel(user *User) UserViewModel {
    return UserViewModel{
        ID:       user.ID,
        Name:     user.Name,
        Email:    user.Email,
        Status:   p.formatUserStatus(user.Active),
        JoinDate: user.CreatedAt.Format("2006-01-02"),
    }
}

// formatUserStatus formats user status for display
func (p *UserPresenter) formatUserStatus(active bool) string {
    if active {
        return "Active"
    }
    return "Inactive"
}

// OrderPresenter handles order-related presentation logic
type OrderPresenter struct {
    view         OrderView
    orderService OrderService
    validator    OrderValidator
}

// NewOrderPresenter creates a new order presenter
func NewOrderPresenter(view OrderView, orderService OrderService, validator OrderValidator) *OrderPresenter {
    return &OrderPresenter{
        view:         view,
        orderService: orderService,
        validator:    validator,
    }
}

// LoadOrders loads and displays orders
func (p *OrderPresenter) LoadOrders() {
    p.view.ShowLoading()
    
    orders, err := p.orderService.GetAllOrders()
    if err != nil {
        p.view.HideLoading()
        p.view.ShowError("Failed to load orders: " + err.Error())
        return
    }
    
    viewModels := p.convertOrdersToViewModels(orders)
    p.view.HideLoading()
    p.view.ShowOrders(viewModels)
}

// LoadOrder loads and displays a specific order
func (p *OrderPresenter) LoadOrder(orderID int) {
    p.view.ShowLoading()
    
    order, err := p.orderService.GetOrderByID(orderID)
    if err != nil {
        p.view.HideLoading()
        p.view.ShowError("Failed to load order: " + err.Error())
        return
    }
    
    viewModel := p.convertOrderToViewModel(order)
    p.view.HideLoading()
    p.view.ShowOrder(viewModel)
}

// CreateOrder handles order creation
func (p *OrderPresenter) CreateOrder() {
    input := p.view.GetOrderInput()
    
    // Validate input
    if errors := p.validator.ValidateOrderInput(input); len(errors) > 0 {
        p.view.ShowValidationErrors(errors)
        return
    }
    
    p.view.ClearValidationErrors()
    p.view.ShowLoading()
    
    // Convert input to service format
    orderItems := make([]OrderItem, len(input.Items))
    for i, item := range input.Items {
        orderItems[i] = OrderItem{
            ProductID: item.ProductID,
            Quantity:  item.Quantity,
        }
    }
    
    // Create order through service
    order, err := p.orderService.CreateOrder(input.CustomerID, orderItems)
    if err != nil {
        p.view.HideLoading()
        p.view.ShowError("Failed to create order: " + err.Error())
        return
    }
    
    p.view.HideLoading()
    p.view.ShowSuccess("Order created successfully")
    p.view.NavigateToOrderDetail(order.ID)
}

// UpdateOrderStatus handles order status updates
func (p *OrderPresenter) UpdateOrderStatus(orderID int, newStatus string) {
    p.view.ShowLoading()
    
    err := p.orderService.UpdateOrderStatus(orderID, newStatus)
    if err != nil {
        p.view.HideLoading()
        p.view.ShowError("Failed to update order status: " + err.Error())
        return
    }
    
    p.view.HideLoading()
    p.view.ShowSuccess("Order status updated successfully")
    p.view.UpdateOrderStatus(orderID, newStatus)
}

// OnOrderSelected handles order selection
func (p *OrderPresenter) OnOrderSelected(orderID int) {
    p.view.NavigateToOrderDetail(orderID)
}

// convertOrdersToViewModels converts domain models to view models
func (p *OrderPresenter) convertOrdersToViewModels(orders []*Order) []OrderViewModel {
    viewModels := make([]OrderViewModel, len(orders))
    for i, order := range orders {
        viewModels[i] = p.convertOrderToViewModel(order)
    }
    return viewModels
}

// convertOrderToViewModel converts domain model to view model
func (p *OrderPresenter) convertOrderToViewModel(order *Order) OrderViewModel {
    return OrderViewModel{
        ID:           order.ID,
        CustomerName: order.CustomerName,
        TotalAmount:  p.formatCurrency(order.TotalAmount),
        Status:       order.Status,
        OrderDate:    order.CreatedAt.Format("2006-01-02 15:04"),
        Items:        p.convertOrderItemsToViewModels(order.Items),
    }
}

// convertOrderItemsToViewModels converts order items to view models
func (p *OrderPresenter) convertOrderItemsToViewModels(items []OrderItem) []OrderItemViewModel {
    viewModels := make([]OrderItemViewModel, len(items))
    for i, item := range items {
        viewModels[i] = OrderItemViewModel{
            ProductName: item.ProductName,
            Quantity:    item.Quantity,
            UnitPrice:   p.formatCurrency(item.UnitPrice),
            TotalPrice:  p.formatCurrency(item.TotalPrice),
        }
    }
    return viewModels
}

// formatCurrency formats currency for display
func (p *OrderPresenter) formatCurrency(amount float64) string {
    return fmt.Sprintf("$%.2f", amount)
}
```

### 3. Model Layer
```go
package mvp

import (
    "time"
    "errors"
)

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
    ID           int         `json:"id"`
    CustomerID   int         `json:"customer_id"`
    CustomerName string      `json:"customer_name"`
    Items        []OrderItem `json:"items"`
    TotalAmount  float64     `json:"total_amount"`
    Status       string      `json:"status"`
    CreatedAt    time.Time   `json:"created_at"`
    UpdatedAt    time.Time   `json:"updated_at"`
}

// OrderItem represents an item in an order
type OrderItem struct {
    ID          int     `json:"id"`
    ProductID   int     `json:"product_id"`
    ProductName string  `json:"product_name"`
    Quantity    int     `json:"quantity"`
    UnitPrice   float64 `json:"unit_price"`
    TotalPrice  float64 `json:"total_price"`
}

// Product domain model
type Product struct {
    ID          int     `json:"id"`
    Name        string  `json:"name"`
    Description string  `json:"description"`
    Price       float64 `json:"price"`
    Stock       int     `json:"stock"`
    Active      bool    `json:"active"`
}

// UserService interface for user business logic
type UserService interface {
    GetAllUsers() ([]*User, error)
    GetUserByID(id int) (*User, error)
    CreateUser(name, email, password string) (*User, error)
    UpdateUser(id int, name, email string) (*User, error)
    DeleteUser(id int) error
    DeactivateUser(id int) error
}

// OrderService interface for order business logic
type OrderService interface {
    GetAllOrders() ([]*Order, error)
    GetOrderByID(id int) (*Order, error)
    GetOrdersByCustomerID(customerID int) ([]*Order, error)
    CreateOrder(customerID int, items []OrderItem) (*Order, error)
    UpdateOrderStatus(id int, status string) error
    CancelOrder(id int) error
}

// UserServiceImpl implements UserService
type UserServiceImpl struct {
    userRepository UserRepository
    emailService   EmailService
}

// NewUserServiceImpl creates a new user service implementation
func NewUserServiceImpl(userRepo UserRepository, emailService EmailService) *UserServiceImpl {
    return &UserServiceImpl{
        userRepository: userRepo,
        emailService:   emailService,
    }
}

// GetAllUsers retrieves all users
func (s *UserServiceImpl) GetAllUsers() ([]*User, error) {
    return s.userRepository.FindAll()
}

// GetUserByID retrieves user by ID
func (s *UserServiceImpl) GetUserByID(id int) (*User, error) {
    user, err := s.userRepository.FindByID(id)
    if err != nil {
        return nil, err
    }
    
    if !user.Active {
        return nil, errors.New("user is inactive")
    }
    
    return user, nil
}

// CreateUser creates a new user
func (s *UserServiceImpl) CreateUser(name, email, password string) (*User, error) {
    // Check if email already exists
    existingUser, _ := s.userRepository.FindByEmail(email)
    if existingUser != nil {
        return nil, errors.New("email already exists")
    }
    
    // Hash password (simplified)
    hashedPassword := "hashed_" + password
    
    user := &User{
        Name:      name,
        Email:     email,
        Password:  hashedPassword,
        Active:    true,
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
    }
    
    if err := s.userRepository.Save(user); err != nil {
        return nil, err
    }
    
    // Send welcome email (asynchronously)
    go s.emailService.SendWelcomeEmail(user.Email, user.Name)
    
    return user, nil
}

// UpdateUser updates an existing user
func (s *UserServiceImpl) UpdateUser(id int, name, email string) (*User, error) {
    user, err := s.userRepository.FindByID(id)
    if err != nil {
        return nil, err
    }
    
    // Check email uniqueness if changed
    if user.Email != email {
        existingUser, _ := s.userRepository.FindByEmail(email)
        if existingUser != nil && existingUser.ID != id {
            return nil, errors.New("email already exists")
        }
    }
    
    user.Name = name
    user.Email = email
    user.UpdatedAt = time.Now()
    
    if err := s.userRepository.Save(user); err != nil {
        return nil, err
    }
    
    return user, nil
}

// DeleteUser deletes a user
func (s *UserServiceImpl) DeleteUser(id int) error {
    return s.userRepository.Delete(id)
}

// DeactivateUser deactivates a user
func (s *UserServiceImpl) DeactivateUser(id int) error {
    user, err := s.userRepository.FindByID(id)
    if err != nil {
        return err
    }
    
    user.Active = false
    user.UpdatedAt = time.Now()
    
    return s.userRepository.Save(user)
}

// OrderServiceImpl implements OrderService
type OrderServiceImpl struct {
    orderRepository   OrderRepository
    productRepository ProductRepository
}

// NewOrderServiceImpl creates a new order service implementation
func NewOrderServiceImpl(orderRepo OrderRepository, productRepo ProductRepository) *OrderServiceImpl {
    return &OrderServiceImpl{
        orderRepository:   orderRepo,
        productRepository: productRepo,
    }
}

// GetAllOrders retrieves all orders
func (s *OrderServiceImpl) GetAllOrders() ([]*Order, error) {
    return s.orderRepository.FindAll()
}

// GetOrderByID retrieves order by ID
func (s *OrderServiceImpl) GetOrderByID(id int) (*Order, error) {
    return s.orderRepository.FindByID(id)
}

// GetOrdersByCustomerID retrieves orders by customer ID
func (s *OrderServiceImpl) GetOrdersByCustomerID(customerID int) ([]*Order, error) {
    return s.orderRepository.FindByCustomerID(customerID)
}

// CreateOrder creates a new order
func (s *OrderServiceImpl) CreateOrder(customerID int, items []OrderItem) (*Order, error) {
    var totalAmount float64
    
    // Validate and price items
    for i, item := range items {
        product, err := s.productRepository.FindByID(item.ProductID)
        if err != nil {
            return nil, err
        }
        
        if product.Stock < item.Quantity {
            return nil, errors.New("insufficient stock for product: " + product.Name)
        }
        
        items[i].ProductName = product.Name
        items[i].UnitPrice = product.Price
        items[i].TotalPrice = product.Price * float64(item.Quantity)
        totalAmount += items[i].TotalPrice
    }
    
    order := &Order{
        CustomerID:  customerID,
        Items:       items,
        TotalAmount: totalAmount,
        Status:      "pending",
        CreatedAt:   time.Now(),
        UpdatedAt:   time.Now(),
    }
    
    if err := s.orderRepository.Save(order); err != nil {
        return nil, err
    }
    
    // Reserve stock
    for _, item := range items {
        s.productRepository.ReserveStock(item.ProductID, item.Quantity)
    }
    
    return order, nil
}

// UpdateOrderStatus updates order status
func (s *OrderServiceImpl) UpdateOrderStatus(id int, status string) error {
    order, err := s.orderRepository.FindByID(id)
    if err != nil {
        return err
    }
    
    order.Status = status
    order.UpdatedAt = time.Now()
    
    return s.orderRepository.Save(order)
}

// CancelOrder cancels an order
func (s *OrderServiceImpl) CancelOrder(id int) error {
    order, err := s.orderRepository.FindByID(id)
    if err != nil {
        return err
    }
    
    if order.Status != "pending" {
        return errors.New("can only cancel pending orders")
    }
    
    order.Status = "cancelled"
    order.UpdatedAt = time.Now()
    
    if err := s.orderRepository.Save(order); err != nil {
        return err
    }
    
    // Release reserved stock
    for _, item := range order.Items {
        s.productRepository.ReleaseStock(item.ProductID, item.Quantity)
    }
    
    return nil
}

// Repository interfaces
type UserRepository interface {
    FindAll() ([]*User, error)
    FindByID(id int) (*User, error)
    FindByEmail(email string) (*User, error)
    Save(user *User) error
    Delete(id int) error
}

type OrderRepository interface {
    FindAll() ([]*Order, error)
    FindByID(id int) (*Order, error)
    FindByCustomerID(customerID int) ([]*Order, error)
    Save(order *Order) error
    Delete(id int) error
}

type ProductRepository interface {
    FindByID(id int) (*Product, error)
    ReserveStock(productID, quantity int) error
    ReleaseStock(productID, quantity int) error
}

// External service interfaces
type EmailService interface {
    SendWelcomeEmail(email, name string) error
}
```

### 4. Validation Layer
```go
package mvp

import (
    "regexp"
    "strings"
)

// UserValidator validates user input
type UserValidator struct{}

// NewUserValidator creates a new user validator
func NewUserValidator() *UserValidator {
    return &UserValidator{}
}

// ValidateUserInput validates user creation input
func (v *UserValidator) ValidateUserInput(input UserInputModel) map[string]string {
    errors := make(map[string]string)
    
    // Validate name
    if strings.TrimSpace(input.Name) == "" {
        errors["name"] = "Name is required"
    } else if len(input.Name) < 2 {
        errors["name"] = "Name must be at least 2 characters"
    } else if len(input.Name) > 100 {
        errors["name"] = "Name must be less than 100 characters"
    }
    
    // Validate email
    if strings.TrimSpace(input.Email) == "" {
        errors["email"] = "Email is required"
    } else if !v.isValidEmail(input.Email) {
        errors["email"] = "Invalid email format"
    }
    
    // Validate password
    if strings.TrimSpace(input.Password) == "" {
        errors["password"] = "Password is required"
    } else if len(input.Password) < 8 {
        errors["password"] = "Password must be at least 8 characters"
    } else if !v.isStrongPassword(input.Password) {
        errors["password"] = "Password must contain uppercase, lowercase, number, and special character"
    }
    
    return errors
}

// ValidateUserUpdateInput validates user update input
func (v *UserValidator) ValidateUserUpdateInput(input UserInputModel) map[string]string {
    errors := make(map[string]string)
    
    // Validate name
    if strings.TrimSpace(input.Name) == "" {
        errors["name"] = "Name is required"
    } else if len(input.Name) < 2 {
        errors["name"] = "Name must be at least 2 characters"
    } else if len(input.Name) > 100 {
        errors["name"] = "Name must be less than 100 characters"
    }
    
    // Validate email
    if strings.TrimSpace(input.Email) == "" {
        errors["email"] = "Email is required"
    } else if !v.isValidEmail(input.Email) {
        errors["email"] = "Invalid email format"
    }
    
    // Password is optional for updates
    if input.Password != "" && len(input.Password) < 8 {
        errors["password"] = "Password must be at least 8 characters"
    } else if input.Password != "" && !v.isStrongPassword(input.Password) {
        errors["password"] = "Password must contain uppercase, lowercase, number, and special character"
    }
    
    return errors
}

// isValidEmail validates email format
func (v *UserValidator) isValidEmail(email string) bool {
    emailRegex := regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)
    return emailRegex.MatchString(email)
}

// isStrongPassword validates password strength
func (v *UserValidator) isStrongPassword(password string) bool {
    hasUpper := regexp.MustCompile(`[A-Z]`).MatchString(password)
    hasLower := regexp.MustCompile(`[a-z]`).MatchString(password)
    hasNumber := regexp.MustCompile(`[0-9]`).MatchString(password)
    hasSpecial := regexp.MustCompile(`[!@#$%^&*(),.?":{}|<>]`).MatchString(password)
    
    return hasUpper && hasLower && hasNumber && hasSpecial
}

// OrderValidator validates order input
type OrderValidator struct{}

// NewOrderValidator creates a new order validator
func NewOrderValidator() *OrderValidator {
    return &OrderValidator{}
}

// ValidateOrderInput validates order creation input
func (v *OrderValidator) ValidateOrderInput(input OrderInputModel) map[string]string {
    errors := make(map[string]string)
    
    // Validate customer ID
    if input.CustomerID <= 0 {
        errors["customer_id"] = "Valid customer ID is required"
    }
    
    // Validate items
    if len(input.Items) == 0 {
        errors["items"] = "At least one item is required"
    } else {
        for i, item := range input.Items {
            if item.ProductID <= 0 {
                errors[fmt.Sprintf("items[%d].product_id", i)] = "Valid product ID is required"
            }
            if item.Quantity <= 0 {
                errors[fmt.Sprintf("items[%d].quantity", i)] = "Quantity must be greater than 0"
            }
        }
    }
    
    return errors
}
```

## Web Implementation Example

### 1. HTTP Handlers (View Implementation)
```go
package web

import (
    "encoding/json"
    "html/template"
    "net/http"
    "strconv"
    
    "github.com/gorilla/mux"
)

// UserWebView implements UserView for web interface
type UserWebView struct {
    templates map[string]*template.Template
    writer    http.ResponseWriter
    request   *http.Request
}

// NewUserWebView creates a new web view
func NewUserWebView(w http.ResponseWriter, r *http.Request) *UserWebView {
    return &UserWebView{
        templates: loadTemplates(),
        writer:    w,
        request:   r,
    }
}

// ShowUsers displays list of users
func (v *UserWebView) ShowUsers(users []UserViewModel) {
    data := struct {
        Users []UserViewModel
        Title string
    }{
        Users: users,
        Title: "User Management",
    }
    
    v.templates["users_list"].Execute(v.writer, data)
}

// ShowUser displays a single user
func (v *UserWebView) ShowUser(user UserViewModel) {
    data := struct {
        User  UserViewModel
        Title string
    }{
        User:  user,
        Title: "User Details",
    }
    
    v.templates["user_detail"].Execute(v.writer, data)
}

// ShowError displays error message
func (v *UserWebView) ShowError(message string) {
    http.Error(v.writer, message, http.StatusInternalServerError)
}

// ShowSuccess displays success message
func (v *UserWebView) ShowSuccess(message string) {
    v.writer.Header().Set("Content-Type", "application/json")
    json.NewEncoder(v.writer).Encode(map[string]string{
        "status":  "success",
        "message": message,
    })
}

// ShowLoading shows loading indicator (handled by frontend)
func (v *UserWebView) ShowLoading() {
    // Implementation depends on frontend framework
}

// HideLoading hides loading indicator (handled by frontend)
func (v *UserWebView) HideLoading() {
    // Implementation depends on frontend framework
}

// GetUserInput retrieves user input from request
func (v *UserWebView) GetUserInput() UserInputModel {
    var input UserInputModel
    json.NewDecoder(v.request.Body).Decode(&input)
    return input
}

// ShowValidationErrors displays validation errors
func (v *UserWebView) ShowValidationErrors(errors map[string]string) {
    v.writer.Header().Set("Content-Type", "application/json")
    v.writer.WriteHeader(http.StatusBadRequest)
    json.NewEncoder(v.writer).Encode(map[string]interface{}{
        "status": "error",
        "errors": errors,
    })
}

// ClearValidationErrors clears validation errors (handled by frontend)
func (v *UserWebView) ClearValidationErrors() {
    // Implementation depends on frontend framework
}

// NavigateToUserDetail redirects to user detail page
func (v *UserWebView) NavigateToUserDetail(userID int) {
    http.Redirect(v.writer, v.request, "/users/"+strconv.Itoa(userID), http.StatusSeeOther)
}

// NavigateToUserEdit redirects to user edit page
func (v *UserWebView) NavigateToUserEdit(userID int) {
    http.Redirect(v.writer, v.request, "/users/"+strconv.Itoa(userID)+"/edit", http.StatusSeeOther)
}

// Close closes the view
func (v *UserWebView) Close() {
    // Cleanup if needed
}

// UserHandler handles HTTP requests for users
type UserHandler struct {
    userService UserService
    validator   UserValidator
}

// NewUserHandler creates a new user handler
func NewUserHandler(userService UserService, validator UserValidator) *UserHandler {
    return &UserHandler{
        userService: userService,
        validator:   validator,
    }
}

// HandleListUsers handles GET /users
func (h *UserHandler) HandleListUsers(w http.ResponseWriter, r *http.Request) {
    view := NewUserWebView(w, r)
    presenter := NewUserPresenter(view, h.userService, h.validator)
    presenter.LoadUsers()
}

// HandleGetUser handles GET /users/{id}
func (h *UserHandler) HandleGetUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    userID, err := strconv.Atoi(vars["id"])
    if err != nil {
        http.Error(w, "Invalid user ID", http.StatusBadRequest)
        return
    }
    
    view := NewUserWebView(w, r)
    presenter := NewUserPresenter(view, h.userService, h.validator)
    presenter.LoadUser(userID)
}

// HandleCreateUser handles POST /users
func (h *UserHandler) HandleCreateUser(w http.ResponseWriter, r *http.Request) {
    view := NewUserWebView(w, r)
    presenter := NewUserPresenter(view, h.userService, h.validator)
    presenter.CreateUser()
}

// HandleUpdateUser handles PUT /users/{id}
func (h *UserHandler) HandleUpdateUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    userID, err := strconv.Atoi(vars["id"])
    if err != nil {
        http.Error(w, "Invalid user ID", http.StatusBadRequest)
        return
    }
    
    view := NewUserWebView(w, r)
    presenter := NewUserPresenter(view, h.userService, h.validator)
    presenter.UpdateUser(userID)
}

// HandleDeleteUser handles DELETE /users/{id}
func (h *UserHandler) HandleDeleteUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    userID, err := strconv.Atoi(vars["id"])
    if err != nil {
        http.Error(w, "Invalid user ID", http.StatusBadRequest)
        return
    }
    
    view := NewUserWebView(w, r)
    presenter := NewUserPresenter(view, h.userService, h.validator)
    presenter.DeleteUser(userID)
}

// loadTemplates loads HTML templates
func loadTemplates() map[string]*template.Template {
    templates := make(map[string]*template.Template)
    
    templates["users_list"] = template.Must(template.New("users_list").Parse(`
<!DOCTYPE html>
<html>
<head>
    <title>{{.Title}}</title>
</head>
<body>
    <h1>{{.Title}}</h1>
    <table>
        <thead>
            <tr>
                <th>ID</th>
                <th>Name</th>
                <th>Email</th>
                <th>Status</th>
                <th>Join Date</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            {{range .Users}}
            <tr>
                <td>{{.ID}}</td>
                <td>{{.Name}}</td>
                <td>{{.Email}}</td>
                <td>{{.Status}}</td>
                <td>{{.JoinDate}}</td>
                <td>
                    <a href="/users/{{.ID}}">View</a>
                    <a href="/users/{{.ID}}/edit">Edit</a>
                </td>
            </tr>
            {{end}}
        </tbody>
    </table>
    <a href="/users/new">Create New User</a>
</body>
</html>
    `))
    
    templates["user_detail"] = template.Must(template.New("user_detail").Parse(`
<!DOCTYPE html>
<html>
<head>
    <title>{{.Title}}</title>
</head>
<body>
    <h1>User Details</h1>
    <div>
        <p><strong>ID:</strong> {{.User.ID}}</p>
        <p><strong>Name:</strong> {{.User.Name}}</p>
        <p><strong>Email:</strong> {{.User.Email}}</p>
        <p><strong>Status:</strong> {{.User.Status}}</p>
        <p><strong>Join Date:</strong> {{.User.JoinDate}}</p>
    </div>
    <a href="/users/{{.User.ID}}/edit">Edit</a>
    <a href="/users">Back to List</a>
</body>
</html>
    `))
    
    return templates
}
```

## Benefits

### Testability
- Presenter can be unit tested without UI dependencies
- Mock views can be easily created for testing
- Business logic is isolated and testable

### Separation of Concerns
- View is passive and only handles display
- Presenter contains all UI logic
- Model handles business logic and data

### Flexibility
- Multiple view implementations (web, mobile, desktop)
- Easy to change UI without affecting business logic
- Presenter can be reused across different view types

## Drawbacks

### Complexity
- More classes and interfaces than simpler patterns
- Additional abstraction layer
- Learning curve for developers

### Presenter Complexity
- Presenter can become large and complex
- Risk of becoming a "god class"
- May need subdivision for complex UIs

### View-Presenter Coupling
- Tight coupling between view and presenter interfaces
- Changes in view requirements may affect presenter
- Interface evolution challenges

## When to Use

### Suitable Scenarios
- **Complex UI logic**: Rich user interfaces with complex interactions
- **Multiple view types**: Web, mobile, desktop versions
- **High testability requirements**: Extensive unit testing needed
- **Team development**: Clear separation for UI and business logic teams

### Not Suitable Scenarios
- **Simple UIs**: Basic CRUD interfaces
- **Rapid prototyping**: Quick development needs
- **Small applications**: Overhead outweighs benefits
- **Static content**: Mostly read-only displays

## Comparison with Other Patterns

### MVP vs MVC
- **MVP**: View is passive, Presenter handles all UI logic
- **MVC**: View can directly observe Model, Controller handles input

### MVP vs MVVM
- **MVP**: Presenter explicitly updates View
- **MVVM**: View binds to ViewModel properties automatically

### MVP Variants
- **Passive View**: View has no knowledge of Model (shown above)
- **Supervising Controller**: View can directly bind to simple Model properties

## Related Patterns
- **Observer Pattern**: For view updates and event handling
- **Command Pattern**: For user actions and undo/redo
- **Factory Pattern**: For creating presenters and views
- **Dependency Injection**: For loose coupling between components

## References
- [Martin Fowler - GUI Architectures](https://martinfowler.com/eaaDev/uiArchs.html)
- [Microsoft - Model-View-Presenter](https://docs.microsoft.com/en-us/previous-versions/msp-n-p/ff649571(v=pandp.10))
- [MVP Pattern Explained](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter)
