# Orchestration Pattern

## Overview
The Orchestration Pattern is a service composition pattern that coordinates multiple services or components through a central orchestrator. The orchestrator acts as a conductor, managing the execution flow, handling service interactions, and ensuring proper sequencing of operations in complex business processes.

## Basic Structure

```
┌─────────────────────────────────────────────────────┐
│                 Orchestrator                        │
│              (Central Coordinator)                  │
├─────────────────────────────────────────────────────┤
│         ↓           ↓           ↓           ↓       │
├─────────────┬─────────────┬─────────────┬───────────┤
│  Service A  │  Service B  │  Service C  │ Service D │
│             │             │             │           │
└─────────────┴─────────────┴─────────────┴───────────┘
```

## Core Components

### 1. Orchestrator Interface
```go
package orchestration

import (
    "context"
    "time"
)

// Orchestrator defines the contract for orchestrating workflows
type Orchestrator interface {
    ExecuteWorkflow(ctx context.Context, workflow Workflow, input WorkflowInput) (*WorkflowResult, error)
    GetWorkflowStatus(workflowID string) (*WorkflowStatus, error)
    CancelWorkflow(workflowID string) error
}

// Workflow represents a business process workflow
type Workflow struct {
    ID          string            `json:"id"`
    Name        string            `json:"name"`
    Description string            `json:"description"`
    Steps       []WorkflowStep    `json:"steps"`
    Config      WorkflowConfig    `json:"config"`
}

// WorkflowStep represents a single step in the workflow
type WorkflowStep struct {
    ID           string                 `json:"id"`
    Name         string                 `json:"name"`
    Type         StepType               `json:"type"`
    ServiceName  string                 `json:"service_name"`
    Method       string                 `json:"method"`
    Input        map[string]interface{} `json:"input"`
    Output       map[string]interface{} `json:"output"`
    Dependencies []string               `json:"dependencies"`
    Timeout      time.Duration          `json:"timeout"`
    Retry        RetryConfig            `json:"retry"`
    Condition    string                 `json:"condition,omitempty"`
}

// StepType defines the type of workflow step
type StepType string

const (
    StepTypeService     StepType = "service"
    StepTypeCondition   StepType = "condition"
    StepTypeParallel    StepType = "parallel"
    StepTypeLoop        StepType = "loop"
    StepTypeDelay       StepType = "delay"
    StepTypeNotification StepType = "notification"
)

// WorkflowConfig contains workflow configuration
type WorkflowConfig struct {
    Timeout         time.Duration `json:"timeout"`
    MaxRetries      int           `json:"max_retries"`
    ParallelLimit   int           `json:"parallel_limit"`
    FailurePolicy   string        `json:"failure_policy"`
    LogLevel        string        `json:"log_level"`
}

// WorkflowInput contains input data for workflow execution
type WorkflowInput struct {
    Data     map[string]interface{} `json:"data"`
    Metadata map[string]string      `json:"metadata"`
}

// WorkflowResult contains the result of workflow execution
type WorkflowResult struct {
    WorkflowID    string                 `json:"workflow_id"`
    Status        WorkflowStatusType     `json:"status"`
    Output        map[string]interface{} `json:"output"`
    Error         string                 `json:"error,omitempty"`
    ExecutedSteps []StepResult           `json:"executed_steps"`
    StartTime     time.Time              `json:"start_time"`
    EndTime       time.Time              `json:"end_time"`
    Duration      time.Duration          `json:"duration"`
}

// WorkflowStatus contains current workflow status
type WorkflowStatus struct {
    WorkflowID    string             `json:"workflow_id"`
    Status        WorkflowStatusType `json:"status"`
    CurrentStep   string             `json:"current_step"`
    Progress      float64            `json:"progress"`
    ExecutedSteps []StepResult       `json:"executed_steps"`
    LastUpdated   time.Time          `json:"last_updated"`
}

// WorkflowStatusType defines workflow status
type WorkflowStatusType string

const (
    StatusPending   WorkflowStatusType = "pending"
    StatusRunning   WorkflowStatusType = "running"
    StatusCompleted WorkflowStatusType = "completed"
    StatusFailed    WorkflowStatusType = "failed"
    StatusCancelled WorkflowStatusType = "cancelled"
    StatusPaused    WorkflowStatusType = "paused"
)

// StepResult contains the result of a workflow step
type StepResult struct {
    StepID    string                 `json:"step_id"`
    Status    StepStatusType         `json:"status"`
    Output    map[string]interface{} `json:"output"`
    Error     string                 `json:"error,omitempty"`
    StartTime time.Time              `json:"start_time"`
    EndTime   time.Time              `json:"end_time"`
    Duration  time.Duration          `json:"duration"`
    Retries   int                    `json:"retries"`
}

// StepStatusType defines step status
type StepStatusType string

const (
    StepStatusPending   StepStatusType = "pending"
    StepStatusRunning   StepStatusType = "running"
    StepStatusCompleted StepStatusType = "completed"
    StepStatusFailed    StepStatusType = "failed"
    StepStatusSkipped   StepStatusType = "skipped"
)

// RetryConfig defines retry configuration for steps
type RetryConfig struct {
    MaxAttempts int           `json:"max_attempts"`
    Delay       time.Duration `json:"delay"`
    Backoff     string        `json:"backoff"` // linear, exponential
}
```

### 2. Orchestrator Implementation
```go
package orchestration

import (
    "context"
    "fmt"
    "log"
    "sync"
    "time"
    
    "github.com/google/uuid"
)

// DefaultOrchestrator implements the Orchestrator interface
type DefaultOrchestrator struct {
    serviceRegistry ServiceRegistry
    workflowStore   WorkflowStore
    eventBus        EventBus
    logger          Logger
    executions      map[string]*WorkflowExecution
    mu              sync.RWMutex
}

// NewDefaultOrchestrator creates a new orchestrator
func NewDefaultOrchestrator(
    serviceRegistry ServiceRegistry,
    workflowStore WorkflowStore,
    eventBus EventBus,
    logger Logger,
) *DefaultOrchestrator {
    return &DefaultOrchestrator{
        serviceRegistry: serviceRegistry,
        workflowStore:   workflowStore,
        eventBus:        eventBus,
        logger:          logger,
        executions:      make(map[string]*WorkflowExecution),
    }
}

// ExecuteWorkflow executes a workflow
func (o *DefaultOrchestrator) ExecuteWorkflow(
    ctx context.Context,
    workflow Workflow,
    input WorkflowInput,
) (*WorkflowResult, error) {
    
    workflowID := uuid.New().String()
    o.logger.Info("Starting workflow execution", "workflowID", workflowID, "workflow", workflow.Name)
    
    // Create workflow execution context
    execution := &WorkflowExecution{
        ID:         workflowID,
        Workflow:   workflow,
        Input:      input,
        Status:     StatusRunning,
        StartTime:  time.Now(),
        Context:    ctx,
        Variables:  make(map[string]interface{}),
        StepResults: make(map[string]StepResult),
    }
    
    // Initialize variables with input data
    for key, value := range input.Data {
        execution.Variables[key] = value
    }
    
    // Store execution
    o.mu.Lock()
    o.executions[workflowID] = execution
    o.mu.Unlock()
    
    // Publish workflow started event
    o.eventBus.Publish(&WorkflowEvent{
        Type:       "workflow.started",
        WorkflowID: workflowID,
        Timestamp:  time.Now(),
    })
    
    // Execute workflow steps
    result, err := o.executeWorkflowSteps(execution)
    
    // Update execution status
    o.mu.Lock()
    if err != nil {
        execution.Status = StatusFailed
        execution.Error = err.Error()
    } else {
        execution.Status = StatusCompleted
    }
    execution.EndTime = time.Now()
    execution.Duration = execution.EndTime.Sub(execution.StartTime)
    o.mu.Unlock()
    
    // Publish workflow completed/failed event
    eventType := "workflow.completed"
    if err != nil {
        eventType = "workflow.failed"
    }
    
    o.eventBus.Publish(&WorkflowEvent{
        Type:       eventType,
        WorkflowID: workflowID,
        Timestamp:  time.Now(),
        Error:      err,
    })
    
    o.logger.Info("Workflow execution finished", "workflowID", workflowID, "status", execution.Status)
    
    return result, err
}

// executeWorkflowSteps executes all workflow steps
func (o *DefaultOrchestrator) executeWorkflowSteps(execution *WorkflowExecution) (*WorkflowResult, error) {
    stepDependencies := o.buildDependencyGraph(execution.Workflow.Steps)
    
    var executedSteps []StepResult
    
    for len(executedSteps) < len(execution.Workflow.Steps) {
        // Find steps that can be executed (dependencies satisfied)
        readySteps := o.findReadySteps(execution.Workflow.Steps, stepDependencies, executedSteps)
        
        if len(readySteps) == 0 {
            return nil, fmt.Errorf("circular dependency or missing dependencies detected")
        }
        
        // Execute ready steps (potentially in parallel)
        stepResults, err := o.executeSteps(execution, readySteps)
        if err != nil {
            return nil, err
        }
        
        executedSteps = append(executedSteps, stepResults...)
        
        // Update execution variables with step outputs
        for _, result := range stepResults {
            for key, value := range result.Output {
                execution.Variables[key] = value
            }
        }
    }
    
    // Build final result
    result := &WorkflowResult{
        WorkflowID:    execution.ID,
        Status:        execution.Status,
        Output:        execution.Variables,
        ExecutedSteps: executedSteps,
        StartTime:     execution.StartTime,
        EndTime:       execution.EndTime,
        Duration:      execution.Duration,
    }
    
    return result, nil
}

// executeSteps executes a batch of steps
func (o *DefaultOrchestrator) executeSteps(execution *WorkflowExecution, steps []WorkflowStep) ([]StepResult, error) {
    var results []StepResult
    var wg sync.WaitGroup
    var mu sync.Mutex
    var stepError error
    
    semaphore := make(chan struct{}, execution.Workflow.Config.ParallelLimit)
    
    for _, step := range steps {
        wg.Add(1)
        go func(s WorkflowStep) {
            defer wg.Done()
            
            semaphore <- struct{}{} // Acquire
            defer func() { <-semaphore }() // Release
            
            result, err := o.executeStep(execution, s)
            
            mu.Lock()
            results = append(results, result)
            if err != nil && stepError == nil {
                stepError = err
            }
            mu.Unlock()
        }(step)
    }
    
    wg.Wait()
    
    return results, stepError
}

// executeStep executes a single workflow step
func (o *DefaultOrchestrator) executeStep(execution *WorkflowExecution, step WorkflowStep) (StepResult, error) {
    o.logger.Info("Executing step", "workflowID", execution.ID, "stepID", step.ID, "stepName", step.Name)
    
    result := StepResult{
        StepID:    step.ID,
        Status:    StepStatusRunning,
        StartTime: time.Now(),
    }
    
    // Publish step started event
    o.eventBus.Publish(&StepEvent{
        Type:       "step.started",
        WorkflowID: execution.ID,
        StepID:     step.ID,
        Timestamp:  time.Now(),
    })
    
    var err error
    
    switch step.Type {
    case StepTypeService:
        result, err = o.executeServiceStep(execution, step)
    case StepTypeCondition:
        result, err = o.executeConditionStep(execution, step)
    case StepTypeDelay:
        result, err = o.executeDelayStep(execution, step)
    case StepTypeNotification:
        result, err = o.executeNotificationStep(execution, step)
    default:
        err = fmt.Errorf("unknown step type: %s", step.Type)
    }
    
    result.EndTime = time.Now()
    result.Duration = result.EndTime.Sub(result.StartTime)
    
    if err != nil {
        result.Status = StepStatusFailed
        result.Error = err.Error()
        o.logger.Error("Step execution failed", "workflowID", execution.ID, "stepID", step.ID, "error", err)
    } else {
        result.Status = StepStatusCompleted
        o.logger.Info("Step execution completed", "workflowID", execution.ID, "stepID", step.ID)
    }
    
    // Store step result
    execution.StepResults[step.ID] = result
    
    // Publish step completed/failed event
    eventType := "step.completed"
    if err != nil {
        eventType = "step.failed"
    }
    
    o.eventBus.Publish(&StepEvent{
        Type:       eventType,
        WorkflowID: execution.ID,
        StepID:     step.ID,
        Timestamp:  time.Now(),
        Error:      err,
    })
    
    return result, err
}

// executeServiceStep executes a service call step
func (o *DefaultOrchestrator) executeServiceStep(execution *WorkflowExecution, step WorkflowStep) (StepResult, error) {
    service, err := o.serviceRegistry.GetService(step.ServiceName)
    if err != nil {
        return StepResult{}, fmt.Errorf("service not found: %s", step.ServiceName)
    }
    
    // Prepare input by resolving variables
    input := o.resolveVariables(step.Input, execution.Variables)
    
    // Create timeout context
    ctx, cancel := context.WithTimeout(execution.Context, step.Timeout)
    defer cancel()
    
    // Execute service call with retry
    var output map[string]interface{}
    var serviceErr error
    
    for attempt := 0; attempt <= step.Retry.MaxAttempts; attempt++ {
        output, serviceErr = service.Call(ctx, step.Method, input)
        if serviceErr == nil {
            break
        }
        
        if attempt < step.Retry.MaxAttempts {
            time.Sleep(step.Retry.Delay)
        }
    }
    
    result := StepResult{
        StepID:  step.ID,
        Output:  output,
        Retries: len(execution.StepResults), // Count of retry attempts
    }
    
    return result, serviceErr
}

// executeConditionStep executes a conditional step
func (o *DefaultOrchestrator) executeConditionStep(execution *WorkflowExecution, step WorkflowStep) (StepResult, error) {
    // Evaluate condition expression
    conditionResult, err := o.evaluateCondition(step.Condition, execution.Variables)
    if err != nil {
        return StepResult{}, err
    }
    
    result := StepResult{
        StepID: step.ID,
        Output: map[string]interface{}{
            "condition_result": conditionResult,
            "executed":        conditionResult,
        },
    }
    
    if !conditionResult {
        result.Status = StepStatusSkipped
    }
    
    return result, nil
}

// executeDelayStep executes a delay step
func (o *DefaultOrchestrator) executeDelayStep(execution *WorkflowExecution, step WorkflowStep) (StepResult, error) {
    delayDuration := step.Timeout
    if delayDuration == 0 {
        delayDuration = 1 * time.Second
    }
    
    select {
    case <-time.After(delayDuration):
        // Delay completed
    case <-execution.Context.Done():
        return StepResult{}, execution.Context.Err()
    }
    
    result := StepResult{
        StepID: step.ID,
        Output: map[string]interface{}{
            "delay_duration": delayDuration.String(),
        },
    }
    
    return result, nil
}

// executeNotificationStep executes a notification step
func (o *DefaultOrchestrator) executeNotificationStep(execution *WorkflowExecution, step WorkflowStep) (StepResult, error) {
    message := o.resolveVariables(step.Input, execution.Variables)
    
    // Send notification (implementation depends on notification service)
    err := o.sendNotification(message)
    
    result := StepResult{
        StepID: step.ID,
        Output: map[string]interface{}{
            "notification_sent": err == nil,
        },
    }
    
    return result, err
}

// buildDependencyGraph builds a dependency graph for workflow steps
func (o *DefaultOrchestrator) buildDependencyGraph(steps []WorkflowStep) map[string][]string {
    dependencies := make(map[string][]string)
    for _, step := range steps {
        dependencies[step.ID] = step.Dependencies
    }
    return dependencies
}

// findReadySteps finds steps that can be executed (dependencies satisfied)
func (o *DefaultOrchestrator) findReadySteps(
    allSteps []WorkflowStep,
    dependencies map[string][]string,
    executedSteps []StepResult,
) []WorkflowStep {
    
    executed := make(map[string]bool)
    for _, result := range executedSteps {
        executed[result.StepID] = true
    }
    
    var readySteps []WorkflowStep
    for _, step := range allSteps {
        if executed[step.ID] {
            continue
        }
        
        // Check if all dependencies are satisfied
        dependenciesSatisfied := true
        for _, dep := range dependencies[step.ID] {
            if !executed[dep] {
                dependenciesSatisfied = false
                break
            }
        }
        
        if dependenciesSatisfied {
            readySteps = append(readySteps, step)
        }
    }
    
    return readySteps
}

// resolveVariables resolves variables in input data
func (o *DefaultOrchestrator) resolveVariables(
    input map[string]interface{},
    variables map[string]interface{},
) map[string]interface{} {
    
    resolved := make(map[string]interface{})
    for key, value := range input {
        if strVal, ok := value.(string); ok {
            // Simple variable substitution (could be enhanced with templating)
            if varValue, exists := variables[strVal]; exists {
                resolved[key] = varValue
            } else {
                resolved[key] = value
            }
        } else {
            resolved[key] = value
        }
    }
    return resolved
}

// evaluateCondition evaluates a condition expression
func (o *DefaultOrchestrator) evaluateCondition(condition string, variables map[string]interface{}) (bool, error) {
    // Simple condition evaluation (could be enhanced with expression parser)
    // For now, just check if variable exists and is true
    if varValue, exists := variables[condition]; exists {
        if boolVal, ok := varValue.(bool); ok {
            return boolVal, nil
        }
    }
    return false, nil
}

// sendNotification sends a notification
func (o *DefaultOrchestrator) sendNotification(message map[string]interface{}) error {
    // Implementation depends on notification service
    o.logger.Info("Sending notification", "message", message)
    return nil
}

// GetWorkflowStatus returns the current status of a workflow
func (o *DefaultOrchestrator) GetWorkflowStatus(workflowID string) (*WorkflowStatus, error) {
    o.mu.RLock()
    execution, exists := o.executions[workflowID]
    o.mu.RUnlock()
    
    if !exists {
        return nil, fmt.Errorf("workflow not found: %s", workflowID)
    }
    
    var executedSteps []StepResult
    for _, result := range execution.StepResults {
        executedSteps = append(executedSteps, result)
    }
    
    progress := float64(len(executedSteps)) / float64(len(execution.Workflow.Steps))
    
    status := &WorkflowStatus{
        WorkflowID:    workflowID,
        Status:        execution.Status,
        Progress:      progress,
        ExecutedSteps: executedSteps,
        LastUpdated:   time.Now(),
    }
    
    return status, nil
}

// CancelWorkflow cancels a running workflow
func (o *DefaultOrchestrator) CancelWorkflow(workflowID string) error {
    o.mu.Lock()
    execution, exists := o.executions[workflowID]
    if exists {
        execution.Status = StatusCancelled
        // Cancel context to stop running steps
        if execution.cancelFunc != nil {
            execution.cancelFunc()
        }
    }
    o.mu.Unlock()
    
    if !exists {
        return fmt.Errorf("workflow not found: %s", workflowID)
    }
    
    o.eventBus.Publish(&WorkflowEvent{
        Type:       "workflow.cancelled",
        WorkflowID: workflowID,
        Timestamp:  time.Now(),
    })
    
    return nil
}

// WorkflowExecution represents a running workflow execution
type WorkflowExecution struct {
    ID          string
    Workflow    Workflow
    Input       WorkflowInput
    Status      WorkflowStatusType
    Error       string
    StartTime   time.Time
    EndTime     time.Time
    Duration    time.Duration
    Context     context.Context
    cancelFunc  context.CancelFunc
    Variables   map[string]interface{}
    StepResults map[string]StepResult
}
```

### 3. Service Registry
```go
package orchestration

import (
    "context"
    "fmt"
    "sync"
)

// Service represents a service that can be called by the orchestrator
type Service interface {
    Call(ctx context.Context, method string, input map[string]interface{}) (map[string]interface{}, error)
    GetName() string
    GetMethods() []string
    IsHealthy() bool
}

// ServiceRegistry manages available services
type ServiceRegistry interface {
    RegisterService(service Service) error
    UnregisterService(name string) error
    GetService(name string) (Service, error)
    ListServices() []string
    GetServiceHealth(name string) (bool, error)
}

// DefaultServiceRegistry implements ServiceRegistry
type DefaultServiceRegistry struct {
    services map[string]Service
    mu       sync.RWMutex
}

// NewDefaultServiceRegistry creates a new service registry
func NewDefaultServiceRegistry() *DefaultServiceRegistry {
    return &DefaultServiceRegistry{
        services: make(map[string]Service),
    }
}

// RegisterService registers a service
func (r *DefaultServiceRegistry) RegisterService(service Service) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    name := service.GetName()
    if _, exists := r.services[name]; exists {
        return fmt.Errorf("service already registered: %s", name)
    }
    
    r.services[name] = service
    return nil
}

// UnregisterService unregisters a service
func (r *DefaultServiceRegistry) UnregisterService(name string) error {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    if _, exists := r.services[name]; !exists {
        return fmt.Errorf("service not found: %s", name)
    }
    
    delete(r.services, name)
    return nil
}

// GetService retrieves a service by name
func (r *DefaultServiceRegistry) GetService(name string) (Service, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    service, exists := r.services[name]
    if !exists {
        return nil, fmt.Errorf("service not found: %s", name)
    }
    
    return service, nil
}

// ListServices returns list of registered services
func (r *DefaultServiceRegistry) ListServices() []string {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    var names []string
    for name := range r.services {
        names = append(names, name)
    }
    return names
}

// GetServiceHealth returns health status of a service
func (r *DefaultServiceRegistry) GetServiceHealth(name string) (bool, error) {
    service, err := r.GetService(name)
    if err != nil {
        return false, err
    }
    
    return service.IsHealthy(), nil
}

// HTTPService represents an HTTP-based service
type HTTPService struct {
    name    string
    baseURL string
    client  HTTPClient
}

// NewHTTPService creates a new HTTP service
func NewHTTPService(name, baseURL string, client HTTPClient) *HTTPService {
    return &HTTPService{
        name:    name,
        baseURL: baseURL,
        client:  client,
    }
}

// Call makes an HTTP call to the service
func (s *HTTPService) Call(
    ctx context.Context,
    method string,
    input map[string]interface{},
) (map[string]interface{}, error) {
    
    url := s.baseURL + "/" + method
    return s.client.Post(ctx, url, input)
}

// GetName returns service name
func (s *HTTPService) GetName() string {
    return s.name
}

// GetMethods returns available methods
func (s *HTTPService) GetMethods() []string {
    // Could be discovered dynamically
    return []string{"process", "validate", "transform"}
}

// IsHealthy checks if service is healthy
func (s *HTTPService) IsHealthy() bool {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    _, err := s.client.Get(ctx, s.baseURL+"/health")
    return err == nil
}

// HTTPClient interface for HTTP operations
type HTTPClient interface {
    Get(ctx context.Context, url string) (map[string]interface{}, error)
    Post(ctx context.Context, url string, data map[string]interface{}) (map[string]interface{}, error)
}
```

### 4. Event System
```go
package orchestration

import (
    "sync"
    "time"
)

// EventBus handles workflow and step events
type EventBus interface {
    Publish(event Event) error
    Subscribe(eventType string, handler EventHandler) error
    Unsubscribe(eventType string, handler EventHandler) error
}

// Event represents a workflow or step event
type Event interface {
    GetType() string
    GetTimestamp() time.Time
}

// EventHandler handles workflow events
type EventHandler interface {
    Handle(event Event) error
}

// WorkflowEvent represents workflow-level events
type WorkflowEvent struct {
    Type       string    `json:"type"`
    WorkflowID string    `json:"workflow_id"`
    Timestamp  time.Time `json:"timestamp"`
    Error      error     `json:"error,omitempty"`
}

// GetType returns event type
func (e *WorkflowEvent) GetType() string {
    return e.Type
}

// GetTimestamp returns event timestamp
func (e *WorkflowEvent) GetTimestamp() time.Time {
    return e.Timestamp
}

// StepEvent represents step-level events
type StepEvent struct {
    Type       string    `json:"type"`
    WorkflowID string    `json:"workflow_id"`
    StepID     string    `json:"step_id"`
    Timestamp  time.Time `json:"timestamp"`
    Error      error     `json:"error,omitempty"`
}

// GetType returns event type
func (e *StepEvent) GetType() string {
    return e.Type
}

// GetTimestamp returns event timestamp
func (e *StepEvent) GetTimestamp() time.Time {
    return e.Timestamp
}

// DefaultEventBus implements EventBus
type DefaultEventBus struct {
    handlers map[string][]EventHandler
    mu       sync.RWMutex
}

// NewDefaultEventBus creates a new event bus
func NewDefaultEventBus() *DefaultEventBus {
    return &DefaultEventBus{
        handlers: make(map[string][]EventHandler),
    }
}

// Publish publishes an event
func (b *DefaultEventBus) Publish(event Event) error {
    b.mu.RLock()
    handlers := b.handlers[event.GetType()]
    b.mu.RUnlock()
    
    for _, handler := range handlers {
        go handler.Handle(event) // Async handling
    }
    
    return nil
}

// Subscribe subscribes to events
func (b *DefaultEventBus) Subscribe(eventType string, handler EventHandler) error {
    b.mu.Lock()
    defer b.mu.Unlock()
    
    b.handlers[eventType] = append(b.handlers[eventType], handler)
    return nil
}

// Unsubscribe unsubscribes from events
func (b *DefaultEventBus) Unsubscribe(eventType string, handler EventHandler) error {
    b.mu.Lock()
    defer b.mu.Unlock()
    
    handlers := b.handlers[eventType]
    for i, h := range handlers {
        if h == handler {
            b.handlers[eventType] = append(handlers[:i], handlers[i+1:]...)
            break
        }
    }
    
    return nil
}

// LoggingEventHandler logs all events
type LoggingEventHandler struct {
    logger Logger
}

// NewLoggingEventHandler creates a logging event handler
func NewLoggingEventHandler(logger Logger) *LoggingEventHandler {
    return &LoggingEventHandler{logger: logger}
}

// Handle handles events by logging them
func (h *LoggingEventHandler) Handle(event Event) error {
    h.logger.Info("Event received", "type", event.GetType(), "timestamp", event.GetTimestamp())
    return nil
}

// MetricsEventHandler collects metrics from events
type MetricsEventHandler struct {
    metrics MetricsCollector
}

// NewMetricsEventHandler creates a metrics event handler
func NewMetricsEventHandler(metrics MetricsCollector) *MetricsEventHandler {
    return &MetricsEventHandler{metrics: metrics}
}

// Handle handles events by collecting metrics
func (h *MetricsEventHandler) Handle(event Event) error {
    switch e := event.(type) {
    case *WorkflowEvent:
        h.metrics.IncrementCounter("workflow_events_total", map[string]string{
            "type":        e.Type,
            "workflow_id": e.WorkflowID,
        })
    case *StepEvent:
        h.metrics.IncrementCounter("step_events_total", map[string]string{
            "type":        e.Type,
            "workflow_id": e.WorkflowID,
            "step_id":     e.StepID,
        })
    }
    return nil
}

// Supporting interfaces
type Logger interface {
    Info(msg string, fields ...interface{})
    Error(msg string, fields ...interface{})
    Debug(msg string, fields ...interface{})
}

type MetricsCollector interface {
    IncrementCounter(name string, tags map[string]string)
    RecordDuration(name string, duration time.Duration, tags map[string]string)
}

type WorkflowStore interface {
    SaveWorkflow(workflow Workflow) error
    GetWorkflow(id string) (*Workflow, error)
    SaveExecution(execution *WorkflowExecution) error
    GetExecution(id string) (*WorkflowExecution, error)
}
```

## Usage Examples

### 1. E-commerce Order Processing
```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    // Create orchestrator
    serviceRegistry := NewDefaultServiceRegistry()
    eventBus := NewDefaultEventBus()
    logger := &ConsoleLogger{}
    
    orchestrator := NewDefaultOrchestrator(
        serviceRegistry,
        nil, // workflow store
        eventBus,
        logger,
    )
    
    // Register services
    serviceRegistry.RegisterService(NewPaymentService())
    serviceRegistry.RegisterService(NewInventoryService())
    serviceRegistry.RegisterService(NewShippingService())
    serviceRegistry.RegisterService(NewNotificationService())
    
    // Define order processing workflow
    orderWorkflow := Workflow{
        ID:          "order-processing",
        Name:        "Order Processing Workflow",
        Description: "Process customer orders end-to-end",
        Steps: []WorkflowStep{
            {
                ID:          "validate-order",
                Name:        "Validate Order",
                Type:        StepTypeService,
                ServiceName: "inventory",
                Method:      "validate",
                Input: map[string]interface{}{
                    "order_id": "order_id",
                    "items":    "order_items",
                },
                Timeout: 30 * time.Second,
                Retry: RetryConfig{
                    MaxAttempts: 3,
                    Delay:       2 * time.Second,
                },
            },
            {
                ID:          "reserve-inventory",
                Name:        "Reserve Inventory",
                Type:        StepTypeService,
                ServiceName: "inventory",
                Method:      "reserve",
                Input: map[string]interface{}{
                    "order_id": "order_id",
                    "items":    "order_items",
                },
                Dependencies: []string{"validate-order"},
                Timeout:      30 * time.Second,
            },
            {
                ID:          "process-payment",
                Name:        "Process Payment",
                Type:        StepTypeService,
                ServiceName: "payment",
                Method:      "charge",
                Input: map[string]interface{}{
                    "order_id":       "order_id",
                    "amount":         "total_amount",
                    "payment_method": "payment_method",
                },
                Dependencies: []string{"reserve-inventory"},
                Timeout:      60 * time.Second,
            },
            {
                ID:          "create-shipment",
                Name:        "Create Shipment",
                Type:        StepTypeService,
                ServiceName: "shipping",
                Method:      "create",
                Input: map[string]interface{}{
                    "order_id": "order_id",
                    "address":  "shipping_address",
                },
                Dependencies: []string{"process-payment"},
                Timeout:      30 * time.Second,
            },
            {
                ID:          "send-confirmation",
                Name:        "Send Order Confirmation",
                Type:        StepTypeService,
                ServiceName: "notification",
                Method:      "send",
                Input: map[string]interface{}{
                    "type":       "order_confirmation",
                    "customer":   "customer_email",
                    "order_id":   "order_id",
                    "shipment_id": "shipment_id",
                },
                Dependencies: []string{"create-shipment"},
                Timeout:      15 * time.Second,
            },
        },
        Config: WorkflowConfig{
            Timeout:       5 * time.Minute,
            MaxRetries:    1,
            ParallelLimit: 3,
            FailurePolicy: "fail-fast",
        },
    }
    
    // Execute workflow
    input := WorkflowInput{
        Data: map[string]interface{}{
            "order_id":         "ORD-12345",
            "customer_email":   "customer@example.com",
            "order_items":      []map[string]interface{}{
                {"product_id": "PROD-001", "quantity": 2},
                {"product_id": "PROD-002", "quantity": 1},
            },
            "total_amount":     99.99,
            "payment_method":   "credit_card",
            "shipping_address": map[string]interface{}{
                "street": "123 Main St",
                "city":   "Anytown",
                "zip":    "12345",
            },
        },
    }
    
    result, err := orchestrator.ExecuteWorkflow(context.Background(), orderWorkflow, input)
    if err != nil {
        fmt.Printf("Workflow execution failed: %v\n", err)
        return
    }
    
    fmt.Printf("Order processing completed successfully!\n")
    fmt.Printf("Workflow ID: %s\n", result.WorkflowID)
    fmt.Printf("Duration: %v\n", result.Duration)
    fmt.Printf("Steps executed: %d\n", len(result.ExecutedSteps))
}

// Example service implementations
type PaymentService struct{}

func NewPaymentService() *PaymentService {
    return &PaymentService{}
}

func (s *PaymentService) Call(ctx context.Context, method string, input map[string]interface{}) (map[string]interface{}, error) {
    switch method {
    case "charge":
        // Simulate payment processing
        time.Sleep(2 * time.Second)
        return map[string]interface{}{
            "payment_id":     "PAY-" + generateID(),
            "status":         "completed",
            "transaction_id": "TXN-" + generateID(),
        }, nil
    }
    return nil, fmt.Errorf("unknown method: %s", method)
}

func (s *PaymentService) GetName() string {
    return "payment"
}

func (s *PaymentService) GetMethods() []string {
    return []string{"charge", "refund"}
}

func (s *PaymentService) IsHealthy() bool {
    return true
}

type InventoryService struct{}

func NewInventoryService() *InventoryService {
    return &InventoryService{}
}

func (s *InventoryService) Call(ctx context.Context, method string, input map[string]interface{}) (map[string]interface{}, error) {
    switch method {
    case "validate":
        // Simulate inventory validation
        time.Sleep(1 * time.Second)
        return map[string]interface{}{
            "valid":     true,
            "available": true,
        }, nil
    case "reserve":
        // Simulate inventory reservation
        time.Sleep(1 * time.Second)
        return map[string]interface{}{
            "reservation_id": "RES-" + generateID(),
            "reserved":       true,
        }, nil
    }
    return nil, fmt.Errorf("unknown method: %s", method)
}

func (s *InventoryService) GetName() string {
    return "inventory"
}

func (s *InventoryService) GetMethods() []string {
    return []string{"validate", "reserve", "release"}
}

func (s *InventoryService) IsHealthy() bool {
    return true
}

// Similar implementations for ShippingService and NotificationService...

func generateID() string {
    return fmt.Sprintf("%d", time.Now().UnixNano())
}

type ConsoleLogger struct{}

func (l *ConsoleLogger) Info(msg string, fields ...interface{}) {
    fmt.Printf("[INFO] %s %v\n", msg, fields)
}

func (l *ConsoleLogger) Error(msg string, fields ...interface{}) {
    fmt.Printf("[ERROR] %s %v\n", msg, fields)
}

func (l *ConsoleLogger) Debug(msg string, fields ...interface{}) {
    fmt.Printf("[DEBUG] %s %v\n", msg, fields)
}
```

### 2. Data Processing Pipeline
```go
package main

func createDataProcessingWorkflow() Workflow {
    return Workflow{
        ID:          "data-processing",
        Name:        "Data Processing Pipeline",
        Description: "Extract, transform, and load data",
        Steps: []WorkflowStep{
            {
                ID:          "extract-data",
                Name:        "Extract Data",
                Type:        StepTypeService,
                ServiceName: "data-extractor",
                Method:      "extract",
                Input: map[string]interface{}{
                    "source": "data_source",
                    "format": "csv",
                },
                Timeout: 5 * time.Minute,
            },
            {
                ID:          "validate-data",
                Name:        "Validate Data",
                Type:        StepTypeService,
                ServiceName: "data-validator",
                Method:      "validate",
                Input: map[string]interface{}{
                    "data": "extracted_data",
                },
                Dependencies: []string{"extract-data"},
                Timeout:      2 * time.Minute,
            },
            {
                ID:          "transform-data",
                Name:        "Transform Data",
                Type:        StepTypeService,
                ServiceName: "data-transformer",
                Method:      "transform",
                Input: map[string]interface{}{
                    "data":  "validated_data",
                    "rules": "transformation_rules",
                },
                Dependencies: []string{"validate-data"},
                Timeout:      10 * time.Minute,
            },
            {
                ID:          "load-data",
                Name:        "Load Data",
                Type:        StepTypeService,
                ServiceName: "data-loader",
                Method:      "load",
                Input: map[string]interface{}{
                    "data":        "transformed_data",
                    "destination": "target_database",
                },
                Dependencies: []string{"transform-data"},
                Timeout:      15 * time.Minute,
            },
            {
                ID:          "generate-report",
                Name:        "Generate Processing Report",
                Type:        StepTypeService,
                ServiceName: "report-generator",
                Method:      "generate",
                Input: map[string]interface{}{
                    "metrics": "processing_metrics",
                    "summary": "processing_summary",
                },
                Dependencies: []string{"load-data"},
                Timeout:      2 * time.Minute,
            },
        },
        Config: WorkflowConfig{
            Timeout:       30 * time.Minute,
            MaxRetries:    2,
            ParallelLimit: 2,
            FailurePolicy: "retry",
        },
    }
}
```

## Benefits

### Centralized Control
- Single point of coordination for complex processes
- Centralized monitoring and logging
- Easier debugging and troubleshooting

### Service Decoupling
- Services don't need to know about each other
- Easy to replace or upgrade individual services
- Clear separation of concerns

### Process Visibility
- Complete view of workflow execution
- Step-by-step monitoring and tracking
- Comprehensive audit trails

### Error Handling
- Centralized error handling and recovery
- Retry mechanisms for failed steps
- Graceful degradation strategies

## Drawbacks

### Single Point of Failure
- Orchestrator failure affects entire workflow
- Need for high availability orchestrator
- Potential bottleneck for all processes

### Complexity
- Complex orchestrator logic
- Difficult to debug distributed workflows
- State management challenges

### Performance Overhead
- Additional network hops through orchestrator
- Latency introduced by central coordination
- Potential scalability limitations

## When to Use

### Suitable Scenarios
- **Complex business processes**: Multi-step workflows with dependencies
- **Service coordination**: Multiple services need coordination
- **Process monitoring**: Need for comprehensive workflow visibility
- **Error recovery**: Complex error handling and retry requirements

### Not Suitable Scenarios
- **Simple processes**: Basic sequential operations
- **High-performance systems**: Latency-sensitive applications
- **Loosely coupled systems**: When services should remain independent
- **Real-time processing**: When immediate response is required

## Comparison with Choreography

### Orchestration
- Centralized control
- Explicit workflow definition
- Single point of monitoring
- Potential single point of failure

### Choreography
- Distributed control
- Event-driven coordination
- No central coordinator
- Better fault tolerance

## Related Patterns
- **Saga Pattern**: For distributed transaction management
- **Process Manager**: For long-running business processes
- **Choreography Pattern**: Alternative coordination approach
- **Command Pattern**: For workflow step execution

## References
- [Microservices Patterns - Chris Richardson](https://microservices.io/patterns/data/saga.html)
- [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/)
- [Workflow Patterns](http://www.workflowpatterns.com/)
