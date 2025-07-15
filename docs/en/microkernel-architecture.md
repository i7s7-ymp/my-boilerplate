# Microkernel Architecture

## Overview
Microkernel Architecture (also known as Plug-in Architecture) is a software architectural pattern that consists of a minimal core system with extended functionality provided through plug-in modules. The core system provides basic services and manages plug-ins, while plug-ins implement specific business logic or features. This architecture promotes modularity, extensibility, and maintainability.

## Basic Structure

```
┌─────────────────────────────────────────────────────┐
│                 Application                         │
│                   Layer                             │
├─────────────────────────────────────────────────────┤
│    Plugin A    │    Plugin B    │    Plugin C      │
│   (Feature 1)  │   (Feature 2)  │   (Feature 3)    │
├─────────────────────────────────────────────────────┤
│              Plugin Registry                        │
│            (Plugin Manager)                         │
├─────────────────────────────────────────────────────┤
│                Microkernel Core                     │
│              (Minimal System)                       │
├─────────────────────────────────────────────────────┤
│             System Services                         │
│        (Database, Logging, etc.)                    │
└─────────────────────────────────────────────────────┘
```

## Core Components

### 1. Microkernel Core
```go
package microkernel

import (
    "context"
    "fmt"
    "log"
    "reflect"
    "sync"
)

// MicrokernelCore represents the minimal core system
type MicrokernelCore struct {
    pluginRegistry *PluginRegistry
    eventBus       *EventBus
    serviceManager *ServiceManager
    logger         Logger
    config         *Configuration
    mu             sync.RWMutex
    running        bool
}

// NewMicrokernelCore creates a new microkernel core
func NewMicrokernelCore(config *Configuration, logger Logger) *MicrokernelCore {
    return &MicrokernelCore{
        pluginRegistry: NewPluginRegistry(),
        eventBus:       NewEventBus(),
        serviceManager: NewServiceManager(),
        logger:         logger,
        config:         config,
    }
}

// Start initializes and starts the microkernel
func (core *MicrokernelCore) Start(ctx context.Context) error {
    core.mu.Lock()
    defer core.mu.Unlock()
    
    if core.running {
        return fmt.Errorf("microkernel is already running")
    }
    
    // Initialize core services
    if err := core.initializeCoreServices(); err != nil {
        return fmt.Errorf("failed to initialize core services: %w", err)
    }
    
    // Start event bus
    if err := core.eventBus.Start(ctx); err != nil {
        return fmt.Errorf("failed to start event bus: %w", err)
    }
    
    // Load and initialize plugins
    if err := core.loadPlugins(); err != nil {
        return fmt.Errorf("failed to load plugins: %w", err)
    }
    
    core.running = true
    core.logger.Info("Microkernel started successfully")
    
    return nil
}

// Stop gracefully shuts down the microkernel
func (core *MicrokernelCore) Stop(ctx context.Context) error {
    core.mu.Lock()
    defer core.mu.Unlock()
    
    if !core.running {
        return fmt.Errorf("microkernel is not running")
    }
    
    // Stop all plugins
    if err := core.pluginRegistry.StopAllPlugins(ctx); err != nil {
        core.logger.Error("Error stopping plugins", "error", err)
    }
    
    // Stop event bus
    if err := core.eventBus.Stop(ctx); err != nil {
        core.logger.Error("Error stopping event bus", "error", err)
    }
    
    // Stop core services
    if err := core.serviceManager.StopAllServices(ctx); err != nil {
        core.logger.Error("Error stopping core services", "error", err)
    }
    
    core.running = false
    core.logger.Info("Microkernel stopped successfully")
    
    return nil
}

// RegisterPlugin registers a new plugin
func (core *MicrokernelCore) RegisterPlugin(plugin Plugin) error {
    core.mu.Lock()
    defer core.mu.Unlock()
    
    return core.pluginRegistry.RegisterPlugin(plugin)
}

// UnregisterPlugin unregisters a plugin
func (core *MicrokernelCore) UnregisterPlugin(pluginID string) error {
    core.mu.Lock()
    defer core.mu.Unlock()
    
    return core.pluginRegistry.UnregisterPlugin(pluginID)
}

// GetService retrieves a core service
func (core *MicrokernelCore) GetService(serviceName string) (interface{}, error) {
    return core.serviceManager.GetService(serviceName)
}

// PublishEvent publishes an event to the event bus
func (core *MicrokernelCore) PublishEvent(event Event) error {
    return core.eventBus.Publish(event)
}

// SubscribeToEvent subscribes to events
func (core *MicrokernelCore) SubscribeToEvent(eventType string, handler EventHandler) error {
    return core.eventBus.Subscribe(eventType, handler)
}

// initializeCoreServices initializes essential core services
func (core *MicrokernelCore) initializeCoreServices() error {
    // Database service
    dbService := NewDatabaseService(core.config.Database)
    if err := core.serviceManager.RegisterService("database", dbService); err != nil {
        return err
    }
    
    // Configuration service
    configService := NewConfigurationService(core.config)
    if err := core.serviceManager.RegisterService("config", configService); err != nil {
        return err
    }
    
    // Security service
    securityService := NewSecurityService(core.config.Security)
    if err := core.serviceManager.RegisterService("security", securityService); err != nil {
        return err
    }
    
    // Start all core services
    return core.serviceManager.StartAllServices()
}

// loadPlugins loads plugins from configuration
func (core *MicrokernelCore) loadPlugins() error {
    for _, pluginConfig := range core.config.Plugins {
        plugin, err := core.createPlugin(pluginConfig)
        if err != nil {
            core.logger.Error("Failed to create plugin", "plugin", pluginConfig.Name, "error", err)
            continue
        }
        
        if err := core.pluginRegistry.RegisterPlugin(plugin); err != nil {
            core.logger.Error("Failed to register plugin", "plugin", pluginConfig.Name, "error", err)
            continue
        }
        
        core.logger.Info("Plugin loaded successfully", "plugin", pluginConfig.Name)
    }
    
    return nil
}

// createPlugin creates a plugin instance from configuration
func (core *MicrokernelCore) createPlugin(config PluginConfig) (Plugin, error) {
    // Plugin factory pattern
    factory, exists := pluginFactories[config.Type]
    if !exists {
        return nil, fmt.Errorf("unknown plugin type: %s", config.Type)
    }
    
    context := &PluginContext{
        Core:   core,
        Config: config,
        Logger: core.logger,
    }
    
    return factory.CreatePlugin(context)
}

// Plugin factory registry
var pluginFactories = make(map[string]PluginFactory)

// RegisterPluginFactory registers a plugin factory
func RegisterPluginFactory(pluginType string, factory PluginFactory) {
    pluginFactories[pluginType] = factory
}
```

### 2. Plugin Interface and Registry
```go
package microkernel

import (
    "context"
    "fmt"
    "sync"
)

// Plugin interface that all plugins must implement
type Plugin interface {
    GetID() string
    GetName() string
    GetVersion() string
    GetDescription() string
    GetDependencies() []string
    Initialize(ctx *PluginContext) error
    Start(ctx context.Context) error
    Stop(ctx context.Context) error
    IsRunning() bool
    GetCapabilities() []string
    HandleRequest(request PluginRequest) (PluginResponse, error)
}

// PluginFactory creates plugin instances
type PluginFactory interface {
    CreatePlugin(context *PluginContext) (Plugin, error)
    GetSupportedTypes() []string
}

// PluginContext provides access to core services
type PluginContext struct {
    Core   *MicrokernelCore
    Config PluginConfig
    Logger Logger
}

// PluginRegistry manages plugin lifecycle
type PluginRegistry struct {
    plugins        map[string]Plugin
    pluginOrder    []string
    dependencies   map[string][]string
    capabilities   map[string][]Plugin
    mu             sync.RWMutex
}

// NewPluginRegistry creates a new plugin registry
func NewPluginRegistry() *PluginRegistry {
    return &PluginRegistry{
        plugins:      make(map[string]Plugin),
        pluginOrder:  make([]string, 0),
        dependencies: make(map[string][]string),
        capabilities: make(map[string][]Plugin),
    }
}

// RegisterPlugin registers a new plugin
func (registry *PluginRegistry) RegisterPlugin(plugin Plugin) error {
    registry.mu.Lock()
    defer registry.mu.Unlock()
    
    pluginID := plugin.GetID()
    
    // Check if plugin already exists
    if _, exists := registry.plugins[pluginID]; exists {
        return fmt.Errorf("plugin %s is already registered", pluginID)
    }
    
    // Validate dependencies
    if err := registry.validateDependencies(plugin); err != nil {
        return fmt.Errorf("dependency validation failed for plugin %s: %w", pluginID, err)
    }
    
    // Register plugin
    registry.plugins[pluginID] = plugin
    registry.dependencies[pluginID] = plugin.GetDependencies()
    
    // Register capabilities
    for _, capability := range plugin.GetCapabilities() {
        registry.capabilities[capability] = append(registry.capabilities[capability], plugin)
    }
    
    // Update plugin order based on dependencies
    registry.updatePluginOrder()
    
    return nil
}

// UnregisterPlugin unregisters a plugin
func (registry *PluginRegistry) UnregisterPlugin(pluginID string) error {
    registry.mu.Lock()
    defer registry.mu.Unlock()
    
    plugin, exists := registry.plugins[pluginID]
    if !exists {
        return fmt.Errorf("plugin %s not found", pluginID)
    }
    
    // Check if other plugins depend on this one
    if err := registry.checkDependents(pluginID); err != nil {
        return err
    }
    
    // Stop plugin if running
    if plugin.IsRunning() {
        if err := plugin.Stop(context.Background()); err != nil {
            return fmt.Errorf("failed to stop plugin %s: %w", pluginID, err)
        }
    }
    
    // Remove from capabilities
    for _, capability := range plugin.GetCapabilities() {
        registry.removePluginFromCapability(capability, plugin)
    }
    
    // Remove plugin
    delete(registry.plugins, pluginID)
    delete(registry.dependencies, pluginID)
    
    // Update plugin order
    registry.updatePluginOrder()
    
    return nil
}

// StartAllPlugins starts all registered plugins in dependency order
func (registry *PluginRegistry) StartAllPlugins(ctx context.Context) error {
    registry.mu.RLock()
    defer registry.mu.RUnlock()
    
    for _, pluginID := range registry.pluginOrder {
        plugin := registry.plugins[pluginID]
        if !plugin.IsRunning() {
            if err := plugin.Start(ctx); err != nil {
                return fmt.Errorf("failed to start plugin %s: %w", pluginID, err)
            }
        }
    }
    
    return nil
}

// StopAllPlugins stops all running plugins in reverse dependency order
func (registry *PluginRegistry) StopAllPlugins(ctx context.Context) error {
    registry.mu.RLock()
    defer registry.mu.RUnlock()
    
    // Stop in reverse order
    for i := len(registry.pluginOrder) - 1; i >= 0; i-- {
        pluginID := registry.pluginOrder[i]
        plugin := registry.plugins[pluginID]
        if plugin.IsRunning() {
            if err := plugin.Stop(ctx); err != nil {
                return fmt.Errorf("failed to stop plugin %s: %w", pluginID, err)
            }
        }
    }
    
    return nil
}

// GetPlugin retrieves a plugin by ID
func (registry *PluginRegistry) GetPlugin(pluginID string) (Plugin, error) {
    registry.mu.RLock()
    defer registry.mu.RUnlock()
    
    plugin, exists := registry.plugins[pluginID]
    if !exists {
        return nil, fmt.Errorf("plugin %s not found", pluginID)
    }
    
    return plugin, nil
}

// GetPluginsByCapability retrieves plugins that provide a specific capability
func (registry *PluginRegistry) GetPluginsByCapability(capability string) []Plugin {
    registry.mu.RLock()
    defer registry.mu.RUnlock()
    
    return registry.capabilities[capability]
}

// ListPlugins returns list of all registered plugins
func (registry *PluginRegistry) ListPlugins() []Plugin {
    registry.mu.RLock()
    defer registry.mu.RUnlock()
    
    plugins := make([]Plugin, 0, len(registry.plugins))
    for _, plugin := range registry.plugins {
        plugins = append(plugins, plugin)
    }
    
    return plugins
}

// validateDependencies validates plugin dependencies
func (registry *PluginRegistry) validateDependencies(plugin Plugin) error {
    for _, dependency := range plugin.GetDependencies() {
        if _, exists := registry.plugins[dependency]; !exists {
            return fmt.Errorf("dependency %s not found", dependency)
        }
    }
    return nil
}

// checkDependents checks if other plugins depend on this plugin
func (registry *PluginRegistry) checkDependents(pluginID string) error {
    for id, dependencies := range registry.dependencies {
        if id == pluginID {
            continue
        }
        for _, dependency := range dependencies {
            if dependency == pluginID {
                return fmt.Errorf("plugin %s is depended upon by %s", pluginID, id)
            }
        }
    }
    return nil
}

// updatePluginOrder updates plugin loading order based on dependencies
func (registry *PluginRegistry) updatePluginOrder() {
    visited := make(map[string]bool)
    tempMark := make(map[string]bool)
    var order []string
    
    var visit func(string) error
    visit = func(pluginID string) error {
        if tempMark[pluginID] {
            return fmt.Errorf("circular dependency detected involving %s", pluginID)
        }
        if visited[pluginID] {
            return nil
        }
        
        tempMark[pluginID] = true
        
        for _, dependency := range registry.dependencies[pluginID] {
            if err := visit(dependency); err != nil {
                return err
            }
        }
        
        tempMark[pluginID] = false
        visited[pluginID] = true
        order = append(order, pluginID)
        
        return nil
    }
    
    for pluginID := range registry.plugins {
        if !visited[pluginID] {
            if err := visit(pluginID); err != nil {
                // Handle circular dependency error
                continue
            }
        }
    }
    
    registry.pluginOrder = order
}

// removePluginFromCapability removes plugin from capability list
func (registry *PluginRegistry) removePluginFromCapability(capability string, targetPlugin Plugin) {
    plugins := registry.capabilities[capability]
    for i, plugin := range plugins {
        if plugin.GetID() == targetPlugin.GetID() {
            registry.capabilities[capability] = append(plugins[:i], plugins[i+1:]...)
            break
        }
    }
}
```

### 3. Example Plugin Implementation
```go
package plugins

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "time"
    
    "github.com/gorilla/mux"
)

// UserManagementPlugin implements user management functionality
type UserManagementPlugin struct {
    id          string
    name        string
    version     string
    description string
    
    context    *PluginContext
    repository UserRepository
    server     *http.Server
    router     *mux.Router
    running    bool
}

// NewUserManagementPlugin creates a new user management plugin
func NewUserManagementPlugin() *UserManagementPlugin {
    return &UserManagementPlugin{
        id:          "user-management",
        name:        "User Management Plugin",
        version:     "1.0.0",
        description: "Provides user registration, authentication, and profile management",
        router:      mux.NewRouter(),
    }
}

// GetID returns plugin ID
func (p *UserManagementPlugin) GetID() string {
    return p.id
}

// GetName returns plugin name
func (p *UserManagementPlugin) GetName() string {
    return p.name
}

// GetVersion returns plugin version
func (p *UserManagementPlugin) GetVersion() string {
    return p.version
}

// GetDescription returns plugin description
func (p *UserManagementPlugin) GetDescription() string {
    return p.description
}

// GetDependencies returns plugin dependencies
func (p *UserManagementPlugin) GetDependencies() []string {
    return []string{"database", "security"}
}

// GetCapabilities returns plugin capabilities
func (p *UserManagementPlugin) GetCapabilities() []string {
    return []string{"user-authentication", "user-management", "profile-management"}
}

// Initialize initializes the plugin
func (p *UserManagementPlugin) Initialize(ctx *PluginContext) error {
    p.context = ctx
    
    // Get database service
    dbService, err := ctx.Core.GetService("database")
    if err != nil {
        return fmt.Errorf("failed to get database service: %w", err)
    }
    
    // Initialize repository
    p.repository = NewUserRepository(dbService.(*DatabaseService))
    
    // Setup HTTP routes
    p.setupRoutes()
    
    // Subscribe to events
    if err := p.subscribeToEvents(); err != nil {
        return fmt.Errorf("failed to subscribe to events: %w", err)
    }
    
    p.context.Logger.Info("User management plugin initialized")
    return nil
}

// Start starts the plugin
func (p *UserManagementPlugin) Start(ctx context.Context) error {
    if p.running {
        return fmt.Errorf("plugin is already running")
    }
    
    // Get HTTP port from configuration
    port := p.context.Config.Parameters["http_port"].(string)
    
    // Create HTTP server
    p.server = &http.Server{
        Addr:    ":" + port,
        Handler: p.router,
    }
    
    // Start HTTP server in goroutine
    go func() {
        if err := p.server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            p.context.Logger.Error("HTTP server error", "error", err)
        }
    }()
    
    p.running = true
    p.context.Logger.Info("User management plugin started", "port", port)
    
    return nil
}

// Stop stops the plugin
func (p *UserManagementPlugin) Stop(ctx context.Context) error {
    if !p.running {
        return fmt.Errorf("plugin is not running")
    }
    
    // Shutdown HTTP server
    if err := p.server.Shutdown(ctx); err != nil {
        return fmt.Errorf("failed to shutdown HTTP server: %w", err)
    }
    
    p.running = false
    p.context.Logger.Info("User management plugin stopped")
    
    return nil
}

// IsRunning returns whether the plugin is running
func (p *UserManagementPlugin) IsRunning() bool {
    return p.running
}

// HandleRequest handles plugin-specific requests
func (p *UserManagementPlugin) HandleRequest(request PluginRequest) (PluginResponse, error) {
    switch request.Action {
    case "authenticate":
        return p.handleAuthentication(request)
    case "get-user":
        return p.handleGetUser(request)
    case "create-user":
        return p.handleCreateUser(request)
    default:
        return PluginResponse{}, fmt.Errorf("unknown action: %s", request.Action)
    }
}

// setupRoutes configures HTTP routes
func (p *UserManagementPlugin) setupRoutes() {
    // Authentication routes
    p.router.HandleFunc("/api/auth/login", p.handleLogin).Methods("POST")
    p.router.HandleFunc("/api/auth/logout", p.handleLogout).Methods("POST")
    p.router.HandleFunc("/api/auth/refresh", p.handleRefreshToken).Methods("POST")
    
    // User management routes
    p.router.HandleFunc("/api/users", p.handleCreateUserHTTP).Methods("POST")
    p.router.HandleFunc("/api/users", p.handleGetUsers).Methods("GET")
    p.router.HandleFunc("/api/users/{id}", p.handleGetUserHTTP).Methods("GET")
    p.router.HandleFunc("/api/users/{id}", p.handleUpdateUser).Methods("PUT")
    p.router.HandleFunc("/api/users/{id}", p.handleDeleteUser).Methods("DELETE")
    
    // Profile routes
    p.router.HandleFunc("/api/profile", p.handleGetProfile).Methods("GET")
    p.router.HandleFunc("/api/profile", p.handleUpdateProfile).Methods("PUT")
    
    // Health check
    p.router.HandleFunc("/health", p.handleHealthCheck).Methods("GET")
}

// subscribeToEvents subscribes to relevant events
func (p *UserManagementPlugin) subscribeToEvents() error {
    // Subscribe to user creation events
    if err := p.context.Core.SubscribeToEvent("user.created", p.handleUserCreatedEvent); err != nil {
        return err
    }
    
    // Subscribe to authentication events
    if err := p.context.Core.SubscribeToEvent("auth.failed", p.handleAuthFailedEvent); err != nil {
        return err
    }
    
    return nil
}

// handleLogin handles user login
func (p *UserManagementPlugin) handleLogin(w http.ResponseWriter, r *http.Request) {
    var loginReq LoginRequest
    if err := json.NewDecoder(r.Body).Decode(&loginReq); err != nil {
        p.writeErrorResponse(w, "Invalid request body", http.StatusBadRequest)
        return
    }
    
    // Authenticate user
    user, err := p.authenticateUser(loginReq.Email, loginReq.Password)
    if err != nil {
        p.writeErrorResponse(w, "Authentication failed", http.StatusUnauthorized)
        return
    }
    
    // Generate JWT token
    token, err := p.generateJWTToken(user)
    if err != nil {
        p.writeErrorResponse(w, "Token generation failed", http.StatusInternalServerError)
        return
    }
    
    // Publish authentication event
    event := AuthenticationEvent{
        UserID:    user.ID,
        Email:     user.Email,
        Success:   true,
        Timestamp: time.Now(),
    }
    p.context.Core.PublishEvent(event)
    
    response := LoginResponse{
        Token: token,
        User:  user,
    }
    
    p.writeJSONResponse(w, response, http.StatusOK)
}

// handleAuthentication handles authentication requests from other plugins
func (p *UserManagementPlugin) handleAuthentication(request PluginRequest) (PluginResponse, error) {
    token, ok := request.Data["token"].(string)
    if !ok {
        return PluginResponse{}, fmt.Errorf("token not provided")
    }
    
    user, err := p.validateJWTToken(token)
    if err != nil {
        return PluginResponse{}, fmt.Errorf("token validation failed: %w", err)
    }
    
    return PluginResponse{
        Success: true,
        Data: map[string]interface{}{
            "user_id": user.ID,
            "email":   user.Email,
        },
    }, nil
}

// authenticateUser authenticates a user with email and password
func (p *UserManagementPlugin) authenticateUser(email, password string) (*User, error) {
    user, err := p.repository.GetByEmail(email)
    if err != nil {
        return nil, err
    }
    
    if !p.verifyPassword(password, user.PasswordHash) {
        return nil, fmt.Errorf("invalid password")
    }
    
    return user, nil
}

// generateJWTToken generates JWT token for user
func (p *UserManagementPlugin) generateJWTToken(user *User) (string, error) {
    // Get security service
    securityService, err := p.context.Core.GetService("security")
    if err != nil {
        return "", err
    }
    
    return securityService.(*SecurityService).GenerateJWT(user.ID, user.Email)
}

// validateJWTToken validates JWT token
func (p *UserManagementPlugin) validateJWTToken(token string) (*User, error) {
    // Get security service
    securityService, err := p.context.Core.GetService("security")
    if err != nil {
        return nil, err
    }
    
    claims, err := securityService.(*SecurityService).ValidateJWT(token)
    if err != nil {
        return nil, err
    }
    
    userID := int(claims["user_id"].(float64))
    return p.repository.GetByID(userID)
}

// verifyPassword verifies password against hash
func (p *UserManagementPlugin) verifyPassword(password, hash string) bool {
    // In production, use bcrypt or similar
    return hash == "hashed_"+password
}

// handleUserCreatedEvent handles user creation events
func (p *UserManagementPlugin) handleUserCreatedEvent(event Event) {
    userEvent := event.(UserCreatedEvent)
    p.context.Logger.Info("User created", "user_id", userEvent.UserID, "email", userEvent.Email)
    
    // Send welcome email, initialize preferences, etc.
}

// handleAuthFailedEvent handles authentication failure events
func (p *UserManagementPlugin) handleAuthFailedEvent(event Event) {
    authEvent := event.(AuthFailedEvent)
    p.context.Logger.Warn("Authentication failed", "email", authEvent.Email, "ip", authEvent.IPAddress)
    
    // Implement rate limiting, logging, etc.
}

// handleHealthCheck handles health check requests
func (p *UserManagementPlugin) handleHealthCheck(w http.ResponseWriter, r *http.Request) {
    status := map[string]interface{}{
        "plugin":  p.name,
        "version": p.version,
        "status":  "healthy",
        "timestamp": time.Now(),
    }
    
    p.writeJSONResponse(w, status, http.StatusOK)
}

// Helper methods
func (p *UserManagementPlugin) writeJSONResponse(w http.ResponseWriter, data interface{}, statusCode int) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(statusCode)
    json.NewEncoder(w).Encode(data)
}

func (p *UserManagementPlugin) writeErrorResponse(w http.ResponseWriter, message string, statusCode int) {
    response := map[string]string{
        "error":  message,
        "status": "error",
    }
    p.writeJSONResponse(w, response, statusCode)
}

// Domain models and DTOs
type User struct {
    ID           int       `json:"id"`
    Email        string    `json:"email"`
    Name         string    `json:"name"`
    PasswordHash string    `json:"-"`
    CreatedAt    time.Time `json:"created_at"`
    UpdatedAt    time.Time `json:"updated_at"`
}

type LoginRequest struct {
    Email    string `json:"email"`
    Password string `json:"password"`
}

type LoginResponse struct {
    Token string `json:"token"`
    User  *User  `json:"user"`
}

type AuthenticationEvent struct {
    UserID    int       `json:"user_id"`
    Email     string    `json:"email"`
    Success   bool      `json:"success"`
    Timestamp time.Time `json:"timestamp"`
}

type UserCreatedEvent struct {
    UserID    int       `json:"user_id"`
    Email     string    `json:"email"`
    Name      string    `json:"name"`
    Timestamp time.Time `json:"timestamp"`
}

type AuthFailedEvent struct {
    Email     string    `json:"email"`
    IPAddress string    `json:"ip_address"`
    Reason    string    `json:"reason"`
    Timestamp time.Time `json:"timestamp"`
}

// Plugin factory for user management plugin
type UserManagementPluginFactory struct{}

func (f *UserManagementPluginFactory) CreatePlugin(context *PluginContext) (Plugin, error) {
    plugin := NewUserManagementPlugin()
    if err := plugin.Initialize(context); err != nil {
        return nil, err
    }
    return plugin, nil
}

func (f *UserManagementPluginFactory) GetSupportedTypes() []string {
    return []string{"user-management"}
}

// Register plugin factory
func init() {
    RegisterPluginFactory("user-management", &UserManagementPluginFactory{})
}
```

### 4. Event Bus System
```go
package microkernel

import (
    "context"
    "fmt"
    "reflect"
    "sync"
)

// Event represents a system event
type Event interface {
    GetType() string
    GetTimestamp() time.Time
    GetSource() string
}

// EventHandler handles events
type EventHandler func(event Event)

// EventBus manages event publishing and subscription
type EventBus struct {
    subscribers map[string][]EventHandler
    eventQueue  chan Event
    mu          sync.RWMutex
    running     bool
    ctx         context.Context
    cancel      context.CancelFunc
}

// NewEventBus creates a new event bus
func NewEventBus() *EventBus {
    return &EventBus{
        subscribers: make(map[string][]EventHandler),
        eventQueue:  make(chan Event, 1000),
    }
}

// Start starts the event bus
func (bus *EventBus) Start(ctx context.Context) error {
    bus.mu.Lock()
    defer bus.mu.Unlock()
    
    if bus.running {
        return fmt.Errorf("event bus is already running")
    }
    
    bus.ctx, bus.cancel = context.WithCancel(ctx)
    bus.running = true
    
    // Start event processing goroutine
    go bus.processEvents()
    
    return nil
}

// Stop stops the event bus
func (bus *EventBus) Stop(ctx context.Context) error {
    bus.mu.Lock()
    defer bus.mu.Unlock()
    
    if !bus.running {
        return fmt.Errorf("event bus is not running")
    }
    
    bus.cancel()
    bus.running = false
    close(bus.eventQueue)
    
    return nil
}

// Publish publishes an event
func (bus *EventBus) Publish(event Event) error {
    if !bus.running {
        return fmt.Errorf("event bus is not running")
    }
    
    select {
    case bus.eventQueue <- event:
        return nil
    case <-bus.ctx.Done():
        return fmt.Errorf("event bus is shutting down")
    default:
        return fmt.Errorf("event queue is full")
    }
}

// Subscribe subscribes to events of a specific type
func (bus *EventBus) Subscribe(eventType string, handler EventHandler) error {
    bus.mu.Lock()
    defer bus.mu.Unlock()
    
    bus.subscribers[eventType] = append(bus.subscribers[eventType], handler)
    return nil
}

// Unsubscribe removes a subscription
func (bus *EventBus) Unsubscribe(eventType string, handler EventHandler) error {
    bus.mu.Lock()
    defer bus.mu.Unlock()
    
    handlers := bus.subscribers[eventType]
    for i, h := range handlers {
        if reflect.ValueOf(h).Pointer() == reflect.ValueOf(handler).Pointer() {
            bus.subscribers[eventType] = append(handlers[:i], handlers[i+1:]...)
            break
        }
    }
    
    return nil
}

// processEvents processes events from the queue
func (bus *EventBus) processEvents() {
    for {
        select {
        case event, ok := <-bus.eventQueue:
            if !ok {
                return // Channel closed
            }
            bus.handleEvent(event)
        case <-bus.ctx.Done():
            return
        }
    }
}

// handleEvent handles a single event
func (bus *EventBus) handleEvent(event Event) {
    bus.mu.RLock()
    handlers := bus.subscribers[event.GetType()]
    bus.mu.RUnlock()
    
    for _, handler := range handlers {
        go func(h EventHandler) {
            defer func() {
                if r := recover(); r != nil {
                    // Log panic but don't crash
                    fmt.Printf("Event handler panic: %v\n", r)
                }
            }()
            h(event)
        }(handler)
    }
}
```

## Benefits

### Modularity
- Clear separation of concerns between core and plugins
- Independent development and testing of features
- Reduced complexity through modular design

### Extensibility
- Easy addition of new features through plugins
- No need to modify core system for new functionality
- Support for third-party extensions

### Maintainability
- Isolated feature implementations
- Easier debugging and troubleshooting
- Simplified testing of individual components

### Flexibility
- Runtime plugin loading and unloading
- Configurable feature sets
- Customizable application behavior

## Drawbacks

### Performance Overhead
- Plugin loading and communication costs
- Indirect method calls through interfaces
- Event system overhead

### Complexity Management
- Plugin dependency management
- Version compatibility issues
- Integration testing challenges

### Security Concerns
- Plugin sandboxing requirements
- Trust boundaries between core and plugins
- Potential security vulnerabilities in plugins

### Limited Shared State
- Difficulty sharing data between plugins
- Event-based communication complexity
- Potential data consistency issues

## When to Use

### Suitable Scenarios
- **Extensible applications**: Need for third-party plugins
- **Feature-rich software**: Multiple optional features
- **Product families**: Shared core with different feature sets
- **IDE and editors**: Plugin-based functionality

### Not Suitable Scenarios
- **Simple applications**: Overhead outweighs benefits
- **High-performance systems**: Plugin overhead unacceptable
- **Tightly coupled features**: Extensive data sharing needed
- **Security-critical systems**: Plugin trust issues

## Implementation Patterns

### Plugin Discovery
```go
// Plugin discovery through filesystem scanning
type PluginDiscovery struct {
    pluginPaths []string
    loader      PluginLoader
}

func (pd *PluginDiscovery) DiscoverPlugins() ([]PluginInfo, error) {
    var plugins []PluginInfo
    
    for _, path := range pd.pluginPaths {
        found, err := pd.scanDirectory(path)
        if err != nil {
            continue
        }
        plugins = append(plugins, found...)
    }
    
    return plugins, nil
}
```

### Plugin Communication
```go
// Inter-plugin communication through message passing
type PluginMessage struct {
    FromPlugin string
    ToPlugin   string
    Type       string
    Data       map[string]interface{}
}

type PluginMessenger interface {
    SendMessage(message PluginMessage) error
    ReceiveMessage() <-chan PluginMessage
}
```

### Plugin Configuration
```go
// Plugin configuration management
type PluginConfig struct {
    Name         string                 `json:"name"`
    Type         string                 `json:"type"`
    Version      string                 `json:"version"`
    Enabled      bool                   `json:"enabled"`
    Dependencies []string               `json:"dependencies"`
    Parameters   map[string]interface{} `json:"parameters"`
}
```

## Related Patterns
- **Plugin Pattern**: Core plugin management
- **Observer Pattern**: Event notification system
- **Factory Pattern**: Plugin instantiation
- **Registry Pattern**: Plugin registration and discovery
- **Service Locator**: Core service access

## Best Practices

### 1. Plugin Design
- Well-defined plugin interfaces
- Clear capability definitions
- Proper dependency management
- Version compatibility handling

### 2. Core System Design
- Minimal core functionality
- Stable plugin APIs
- Comprehensive service layer
- Robust error handling

### 3. Security
- Plugin sandboxing
- Permission-based access control
- Input validation and sanitization
- Secure plugin loading mechanisms

### 4. Performance
- Lazy plugin loading
- Efficient event system
- Minimal overhead design
- Performance monitoring

## References
- [Pattern-Oriented Software Architecture Volume 1](https://www.wiley.com/en-us/Pattern+Oriented+Software+Architecture%2C+Volume+1%3A+A+System+of+Patterns-p-9780471958697)
- [Software Architecture Patterns - Mark Richards](https://www.oreilly.com/library/view/software-architecture-patterns/9781491971437/)
- [Eclipse Plugin Development](https://www.eclipse.org/articles/)
- [OSGi Framework](https://www.osgi.org/)
