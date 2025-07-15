# Orchestration Pattern（オーケストレーションパターン）

## 概要
オーケストレーションパターンは、複数の独立したサービスやプロセスを調整し、統合された業務フローを実現するアーキテクチャパターンです。中央のオーケストレーターが全体のワークフローを制御し、各サービスの実行順序、エラーハンドリング、補償処理を管理します。

## 基本概念

### オーケストレーション vs コレオグラフィー

```
オーケストレーション（中央集権型）:
┌─────────────────────────────────────────────────────┐
│                Orchestrator                         │
├─────────────────────────────────────────────────────┤
│     ↓           ↓           ↓           ↓           │
├─────────────────────────────────────────────────────┤
│ Service A   Service B   Service C   Service D       │
└─────────────────────────────────────────────────────┘

コレオグラフィー（分散型）:
┌─────────────────────────────────────────────────────┐
│ Service A ↔ Service B ↔ Service C ↔ Service D      │
│     ↕           ↕           ↕           ↕           │
│   Event     Event       Event       Event          │
│   Bus       Bus         Bus         Bus             │
└─────────────────────────────────────────────────────┘
```

## 核心コンポーネント

### 1. ワークフロー定義
```go
package main

package orchestration

    // context
    // encoding/json
    // errors
    // fmt
    // sync
    // time

    // github.com/google/uuid
)

// TaskStatus タスク状態
type TaskStatus string

const (
    TaskStatusPending   TaskStatus = // pending
    TaskStatusRunning   TaskStatus = // running
    TaskStatusCompleted TaskStatus = // completed
    TaskStatusFailed    TaskStatus = // failed
    TaskStatusSkipped   TaskStatus = // skipped
    TaskStatusCancelled TaskStatus = // cancelled
)

// WorkflowStatus ワークフロー状態
type WorkflowStatus string

const (
    WorkflowStatusCreated   WorkflowStatus = // created
    WorkflowStatusRunning   WorkflowStatus = // running
    WorkflowStatusCompleted WorkflowStatus = // completed
    WorkflowStatusFailed    WorkflowStatus = // failed
    WorkflowStatusCancelled WorkflowStatus = // cancelled
)

// ErrorHandlingStrategy エラーハンドリング戦略
type ErrorHandlingStrategy string

const (
    ErrorHandlingFailFast   ErrorHandlingStrategy = // fail_fast
    ErrorHandlingContinue   ErrorHandlingStrategy = // continue
    ErrorHandlingCompensate ErrorHandlingStrategy = // compensate
)

// TaskDefinition タスク定義
type TaskDefinition struct {
    TaskID         string                 `json:// task_id`
    Name           string                 `json:// name`
    ServiceName    string                 `json:// service_name`
    Method         string                 `json:// method`
    Parameters     map[string]interface{} `json:// parameters`
    Timeout        *time.Duration         `json:// timeout,omitempty`
    RetryCount     int                    `json:// retry_count`
    RetryDelay     time.Duration          `json:// retry_delay`
    DependsOn      []string               `json:// depends_on`
    Condition      *string                `json:// condition,omitempty`
    CompensateWith *string                `json:// compensate_with,omitempty`
}

// NewTaskDefinition タスク定義作成
func NewTaskDefinition(taskID, name, serviceName, method string) *TaskDefinition {
    return &TaskDefinition{
        TaskID:      taskID,
        Name:        name,
        ServiceName: serviceName,
        Method:      method,
        Parameters:  make(map[string]interface{}),
        RetryCount:  0,
        RetryDelay:  time.Second,
        DependsOn:   make([]string, 0),
    }
}

// WorkflowDefinition ワークフロー定義
type WorkflowDefinition struct {
    WorkflowID     string                 `json:// workflow_id`
    Name           string                 `json:// name`
    Description    string                 `json:// description`
    Tasks          []*TaskDefinition      `json:// tasks`
    GlobalTimeout  *time.Duration         `json:// global_timeout,omitempty`
    ErrorHandling  ErrorHandlingStrategy  `json:// error_handling`
    mu             sync.RWMutex
}

// NewWorkflowDefinition ワークフロー定義作成
func NewWorkflowDefinition(workflowID, name, description string) *WorkflowDefinition {
    return &WorkflowDefinition{
        WorkflowID:    workflowID,
        Name:          name,
        Description:   description,
        Tasks:         make([]*TaskDefinition, 0),
        ErrorHandling: ErrorHandlingFailFast,
    }
}

// GetTask タスク取得
func (wd *WorkflowDefinition) GetTask(taskID string) *TaskDefinition {
    wd.mu.RLock()
    defer wd.mu.RUnlock()
    
    for _, task := range wd.Tasks {
        if task.TaskID == taskID {
            return task
        }
    }
    return nil
}

// GetDependencies 依存関係取得
func (wd *WorkflowDefinition) GetDependencies(taskID string) []string {
    task := wd.GetTask(taskID)
    if task == nil {
        return []string{}
    }
    return task.DependsOn
}

// GetDependentTasks 依存されているタスク一覧
func (wd *WorkflowDefinition) GetDependentTasks(taskID string) []string {
    wd.mu.RLock()
    defer wd.mu.RUnlock()
    
    var dependents []string
    for _, task := range wd.Tasks {
        for _, dep := range task.DependsOn {
            if dep == taskID {
                dependents = append(dependents, task.TaskID)
                break
            }
        }
    }
    return dependents
}

// AddTask タスク追加
func (wd *WorkflowDefinition) AddTask(task *TaskDefinition) {
    wd.mu.Lock()
    defer wd.mu.Unlock()
    wd.Tasks = append(wd.Tasks, task)
}

// TaskResult タスク実行結果
type TaskResult struct {
    TaskID     string      `json:// task_id`
    Status     TaskStatus  `json:// status`
    Result     interface{} `json:// result,omitempty`
    Error      string      `json:// error,omitempty`
    StartTime  *time.Time  `json:// start_time,omitempty`
    EndTime    *time.Time  `json:// end_time,omitempty`
    RetryCount int         `json:// retry_count`
    mu         sync.RWMutex
}

// NewTaskResult タスク結果作成
func NewTaskResult(taskID string, status TaskStatus) *TaskResult {
    return &TaskResult{
        TaskID:     taskID,
        Status:     status,
        RetryCount: 0,
    }
}

// Duration 実行時間
func (tr *TaskResult) Duration() *time.Duration {
    tr.mu.RLock()
    defer tr.mu.RUnlock()
    
    if tr.StartTime != nil && tr.EndTime != nil {
        duration := tr.EndTime.Sub(*tr.StartTime)
        return &duration
    }
    return nil
}

// SetStatus ステータス設定
func (tr *TaskResult) SetStatus(status TaskStatus) {
    tr.mu.Lock()
    defer tr.mu.Unlock()
    tr.Status = status
}

// SetResult 結果設定
func (tr *TaskResult) SetResult(result interface{}) {
    tr.mu.Lock()
    defer tr.mu.Unlock()
    tr.Result = result
}

// SetError エラー設定
func (tr *TaskResult) SetError(err string) {
    tr.mu.Lock()
    defer tr.mu.Unlock()
    tr.Error = err
}

// WorkflowExecution ワークフロー実行状態
type WorkflowExecution struct {
    ExecutionID        string                    `json:// execution_id`
    WorkflowDefinition *WorkflowDefinition       `json:// workflow_definition`
    Status             WorkflowStatus            `json:// status`
    TaskResults        map[string]*TaskResult    `json:// task_results`
    Context            map[string]interface{}    `json:// context`
    StartTime          *time.Time                `json:// start_time,omitempty`
    EndTime            *time.Time                `json:// end_time,omitempty`
    Error              string                    `json:// error,omitempty`
    mu                 sync.RWMutex
    cancelFunc         context.CancelFunc
}

// NewWorkflowExecution ワークフロー実行作成
func NewWorkflowExecution(workflowDef *WorkflowDefinition) *WorkflowExecution {
    return &WorkflowExecution{
        ExecutionID:        uuid.New().String(),
        WorkflowDefinition: workflowDef,
        Status:             WorkflowStatusCreated,
        TaskResults:        make(map[string]*TaskResult),
        Context:            make(map[string]interface{}),
    }
}

// GetTaskResult タスク結果取得
func (we *WorkflowExecution) GetTaskResult(taskID string) *TaskResult {
    we.mu.RLock()
    defer we.mu.RUnlock()
    return we.TaskResults[taskID]
}

// SetTaskResult タスク結果設定
func (we *WorkflowExecution) SetTaskResult(taskResult *TaskResult) {
    we.mu.Lock()
    defer we.mu.Unlock()
    we.TaskResults[taskResult.TaskID] = taskResult
}

// GetCompletedTasks 完了タスク一覧
func (we *WorkflowExecution) GetCompletedTasks() []string {
    we.mu.RLock()
    defer we.mu.RUnlock()
    
    var completed []string
    for taskID, result := range we.TaskResults {
        if result.Status == TaskStatusCompleted {
            completed = append(completed, taskID)
        }
    }
    return completed
}

// GetFailedTasks 失敗タスク一覧
func (we *WorkflowExecution) GetFailedTasks() []string {
    we.mu.RLock()
    defer we.mu.RUnlock()
    
    var failed []string
    for taskID, result := range we.TaskResults {
        if result.Status == TaskStatusFailed {
            failed = append(failed, taskID)
        }
    }
    return failed
}

// SetStatus ステータス設定
func (we *WorkflowExecution) SetStatus(status WorkflowStatus) {
    we.mu.Lock()
    defer we.mu.Unlock()
    we.Status = status
}

// SetError エラー設定
func (we *WorkflowExecution) SetError(err string) {
    we.mu.Lock()
    defer we.mu.Unlock()
    we.Error = err
}

// UpdateContext コンテキスト更新
func (we *WorkflowExecution) UpdateContext(key string, value interface{}) {
    we.mu.Lock()
    defer we.mu.Unlock()
    we.Context[key] = value
}

// GetContext コンテキスト取得
func (we *WorkflowExecution) GetContext(key string) (interface{}, bool) {
    we.mu.RLock()
    defer we.mu.RUnlock()
    value, exists := we.Context[key]
    return value, exists
}
```

### 2. サービスインターフェース
```go
// ServiceInterface サービスインターフェース
type ServiceInterface interface {
    Execute(ctx context.Context, method string, parameters map[string]interface{}, context map[string]interface{}) (interface{}, error)
    HealthCheck(ctx context.Context) error
    GetName() string
}

// ServiceRegistry サービスレジストリ
type ServiceRegistry struct {
    services map[string]ServiceInterface
    mu       sync.RWMutex
}

// NewServiceRegistry サービスレジストリ作成
func NewServiceRegistry() *ServiceRegistry {
    return &ServiceRegistry{
        services: make(map[string]ServiceInterface),
    }
}

// RegisterService サービス登録
func (sr *ServiceRegistry) RegisterService(name string, service ServiceInterface) {
    sr.mu.Lock()
    defer sr.mu.Unlock()
    sr.services[name] = service
}

// GetService サービス取得
func (sr *ServiceRegistry) GetService(name string) ServiceInterface {
    sr.mu.RLock()
    defer sr.mu.RUnlock()
    return sr.services[name]
}

// UnregisterService サービス登録解除
func (sr *ServiceRegistry) UnregisterService(name string) {
    sr.mu.Lock()
    defer sr.mu.Unlock()
    delete(sr.services, name)
}

// UserService ユーザーサービス実装
type UserService struct {
    users map[string]map[string]interface{}
    mu    sync.RWMutex
}

// NewUserService ユーザーサービス作成
func NewUserService() *UserService {
    return &UserService{
        users: make(map[string]map[string]interface{}),
    }
}

// Execute メソッド実行
func (us *UserService) Execute(ctx context.Context, method string, parameters map[string]interface{}, context map[string]interface{}) (interface{}, error) {
    switch method {
    case "create_user":
        return us.createUser(ctx, parameters)
    case "get_user":
        return us.getUser(ctx, parameters)
    case "update_user":
        return us.updateUser(ctx, parameters)
    case "delete_user":
        return us.deleteUser(ctx, parameters)
    default:
        return nil, fmt.Errorf("unknown method: %s", method)
    }
}

// createUser ユーザー作成
func (us *UserService) createUser(ctx context.Context, params map[string]interface{}) (interface{}, error) {
    userID := uuid.New().String()
    
    name, _ := params["name"].(string)
    email, _ := params["email"].(string)
    
    userData := map[string]interface{}{
        "user_id":    userID,
        "name":       name,
        "email":      email,
        "created_at": time.Now().Format(time.RFC3339),
    }
    
    us.mu.Lock()
    us.users[userID] = userData
    us.mu.Unlock()
    
    return userData, nil
}

// getUser ユーザー取得
func (us *UserService) getUser(ctx context.Context, params map[string]interface{}) (interface{}, error) {
    userID, ok := params["user_id"].(string)
    if !ok {
        return nil, errors.New("user_id is required")
    }
    
    us.mu.RLock()
    user, exists := us.users[userID]
    us.mu.RUnlock()
    
    if !exists {
        return nil, fmt.Errorf("user not found: %s", userID)
    }
    
    return user, nil
}

// updateUser ユーザー更新
func (us *UserService) updateUser(ctx context.Context, params map[string]interface{}) (interface{}, error) {
    userID, ok := params["user_id"].(string)
    if !ok {
        return nil, errors.New("user_id is required")
    }
    
    updates, ok := params["updates"].(map[string]interface{})
    if !ok {
        return nil, errors.New("updates is required")
    }
    
    us.mu.Lock()
    defer us.mu.Unlock()
    
    user, exists := us.users[userID]
    if !exists {
        return nil, fmt.Errorf("user not found: %s", userID)
    }
    
    for key, value := range updates {
        user[key] = value
    }
    
    return user, nil
}

// deleteUser ユーザー削除
func (us *UserService) deleteUser(ctx context.Context, params map[string]interface{}) (interface{}, error) {
    userID, ok := params["user_id"].(string)
    if !ok {
        return nil, errors.New("user_id is required")
    }
    
    us.mu.Lock()
    defer us.mu.Unlock()
    
    if _, exists := us.users[userID]; !exists {
        return false, fmt.Errorf("user not found: %s", userID)
    }
    
    delete(us.users, userID)
    return true, nil
}

// HealthCheck ヘルスチェック
func (us *UserService) HealthCheck(ctx context.Context) error {
    return nil
}

// GetName サービス名取得
func (us *UserService) GetName() string {
    return "UserService"
}

// EmailService メールサービス実装
type EmailService struct {
    sentEmails []map[string]interface{}
    mu         sync.Mutex
}

// NewEmailService メールサービス作成
func NewEmailService() *EmailService {
    return &EmailService{
        sentEmails: make([]map[string]interface{}, 0),
    }
}

// Execute メソッド実行
func (es *EmailService) Execute(ctx context.Context, method string, parameters map[string]interface{}, context map[string]interface{}) (interface{}, error) {
    switch method {
    case "send_email":
        return es.sendEmail(ctx, parameters)
    case "send_welcome_email":
        return es.sendWelcomeEmail(ctx, parameters, context)
    default:
        return nil, fmt.Errorf("unknown method: %s", method)
    }
}

// sendEmail メール送信
func (es *EmailService) sendEmail(ctx context.Context, params map[string]interface{}) (interface{}, error) {
    to, _ := params["to"].(string)
    subject, _ := params["subject"].(string)
    body, _ := params["body"].(string)
    
    emailData := map[string]interface{}{
        "to":      to,
        "subject": subject,
        "body":    body,
        "sent_at": time.Now().Format(time.RFC3339),
    }
    
    es.mu.Lock()
    es.sentEmails = append(es.sentEmails, emailData)
    es.mu.Unlock()
    
    // シミュレートした遅延
    time.Sleep(100 * time.Millisecond)
    
    return true, nil
}

// sendWelcomeEmail ウェルカムメール送信
func (es *EmailService) sendWelcomeEmail(ctx context.Context, params map[string]interface{}, context map[string]interface{}) (interface{}, error) {
    userData, exists := context["user_data"]
    if !exists {
        return nil, errors.New("user data not found in context")
    }
    
    userMap, ok := userData.(map[string]interface{})
    if !ok {
        return nil, errors.New("invalid user data format")
    }
    
    email, _ := userMap["email"].(string)
    name, _ := userMap["name"].(string)
    
    emailParams := map[string]interface{}{
        "to":      email,
        "subject": "Welcome!",
        "body":    fmt.Sprintf("Welcome %s!", name),
    }
    
    return es.sendEmail(ctx, emailParams)
}

// HealthCheck ヘルスチェック
func (es *EmailService) HealthCheck(ctx context.Context) error {
    return nil
}

// GetName サービス名取得
func (es *EmailService) GetName() string {
    return "EmailService"
}
```

### 3. オーケストレーションエンジン
```go
// ConditionEvaluator 条件評価器
type ConditionEvaluator struct{}

// NewConditionEvaluator 条件評価器作成
func NewConditionEvaluator() *ConditionEvaluator {
    return &ConditionEvaluator{}
}

// Evaluate 条件評価（簡単な実装）
func (ce *ConditionEvaluator) Evaluate(condition string, context map[string]interface{}) bool {
    // 実際の実装ではより安全で高機能な条件評価器を使用
    // ここでは簡単な例として文字列比較を行う
    switch condition {
    case "available == True", "available == true":
        if val, ok := context["available"]; ok {
            if bval, ok := val.(bool); ok {
                return bval
            }
        }
    }
    return false
}

// OrchestrationEngine オーケストレーションエンジン
type OrchestrationEngine struct {
    serviceRegistry     *ServiceRegistry
    executions          map[string]*WorkflowExecution
    conditionEvaluator  *ConditionEvaluator
    mu                  sync.RWMutex
}

// NewOrchestrationEngine オーケストレーションエンジン作成
func NewOrchestrationEngine(serviceRegistry *ServiceRegistry) *OrchestrationEngine {
    return &OrchestrationEngine{
        serviceRegistry:    serviceRegistry,
        executions:         make(map[string]*WorkflowExecution),
        conditionEvaluator: NewConditionEvaluator(),
    }
}

// ExecuteWorkflow ワークフロー実行開始
func (oe *OrchestrationEngine) ExecuteWorkflow(ctx context.Context, workflowDef *WorkflowDefinition, initialContext map[string]interface{}) (string, error) {
    execution := NewWorkflowExecution(workflowDef)
    
    if initialContext != nil {
        for k, v := range initialContext {
            execution.Context[k] = v
        }
    }
    
    now := time.Now()
    execution.StartTime = &now
    
    oe.mu.Lock()
    oe.executions[execution.ExecutionID] = execution
    oe.mu.Unlock()
    
    // Goroutineでワークフロー実行
    go oe.executeWorkflowAsync(ctx, execution)
    
    return execution.ExecutionID, nil
}

// executeWorkflowAsync ワークフロー非同期実行
func (oe *OrchestrationEngine) executeWorkflowAsync(ctx context.Context, execution *WorkflowExecution) {
    defer func() {
        now := time.Now()
        execution.EndTime = &now
    }()
    
    // グローバルタイムアウト設定
    workflowCtx := ctx
    if execution.WorkflowDefinition.GlobalTimeout != nil {
        var cancel context.CancelFunc
        workflowCtx, cancel = context.WithTimeout(ctx, *execution.WorkflowDefinition.GlobalTimeout)
        defer cancel()
    }
    
    execution.SetStatus(WorkflowStatusRunning)
    
    for {
        select {
        case <-workflowCtx.Done():
            execution.SetStatus(WorkflowStatusCancelled)
            execution.SetError("workflow timeout or cancelled")
            return
        default:
        }
        
        readyTasks := oe.getReadyTasks(execution)
        
        if len(readyTasks) == 0 {
            // 実行中のタスクがあるかチェック
            if oe.hasRunningTasks(execution) {
                time.Sleep(100 * time.Millisecond)
                continue
            }
            break
        }
        
        // 並行実行
        var wg sync.WaitGroup
        for _, taskID := range readyTasks {
            wg.Add(1)
            go func(id string) {
                defer wg.Done()
                oe.executeTask(workflowCtx, execution, id)
            }(taskID)
        }
        wg.Wait()
        
        // エラーチェック
        if oe.shouldStopExecution(execution) {
            break
        }
    }
    
    // 完了判定
    if oe.allTasksCompleted(execution) {
        execution.SetStatus(WorkflowStatusCompleted)
    } else if execution.Status == WorkflowStatusRunning {
        execution.SetStatus(WorkflowStatusFailed)
    }
}

// getReadyTasks 実行可能タスク取得
func (oe *OrchestrationEngine) getReadyTasks(execution *WorkflowExecution) []string {
    var readyTasks []string
    
    for _, task := range execution.WorkflowDefinition.Tasks {
        // 既に実行済みまたは実行中はスキップ
        taskResult := execution.GetTaskResult(task.TaskID)
        if taskResult != nil {
            if taskResult.Status == TaskStatusCompleted || 
               taskResult.Status == TaskStatusRunning || 
               taskResult.Status == TaskStatusFailed {
                continue
            }
        }
        
        // 依存関係チェック
        dependenciesMet := true
        for _, depID := range task.DependsOn {
            depResult := execution.GetTaskResult(depID)
            if depResult == nil || depResult.Status != TaskStatusCompleted {
                dependenciesMet = false
                break
            }
        }
        
        if dependenciesMet {
            // 条件チェック
            if task.Condition == nil || oe.conditionEvaluator.Evaluate(*task.Condition, execution.Context) {
                readyTasks = append(readyTasks, task.TaskID)
            }
        }
    }
    
    return readyTasks
}

// hasRunningTasks 実行中タスクの存在確認
func (oe *OrchestrationEngine) hasRunningTasks(execution *WorkflowExecution) bool {
    execution.mu.RLock()
    defer execution.mu.RUnlock()
    
    for _, result := range execution.TaskResults {
        if result.Status == TaskStatusRunning {
            return true
        }
    }
    return false
}

// executeTask タスク実行
func (oe *OrchestrationEngine) executeTask(ctx context.Context, execution *WorkflowExecution, taskID string) {
    taskDef := execution.WorkflowDefinition.GetTask(taskID)
    if taskDef == nil {
        return
    }
    
    // タスク結果初期化
    taskResult := NewTaskResult(taskID, TaskStatusRunning)
    now := time.Now()
    taskResult.StartTime = &now
    execution.SetTaskResult(taskResult)
    
    retryCount := 0
    
    for retryCount <= taskDef.RetryCount {
        // タイムアウト設定
        taskCtx := ctx
        if taskDef.Timeout != nil {
            var cancel context.CancelFunc
            taskCtx, cancel = context.WithTimeout(ctx, *taskDef.Timeout)
            defer cancel()
        }
        
        // サービス取得
        service := oe.serviceRegistry.GetService(taskDef.ServiceName)
        if service == nil {
            taskResult.SetStatus(TaskStatusFailed)
            taskResult.SetError(fmt.Sprintf("Service not found: %s", taskDef.ServiceName))
            now := time.Now()
            taskResult.EndTime = &now
            return
        }
        
        // タスク実行
        result, err := service.Execute(taskCtx, taskDef.Method, taskDef.Parameters, execution.Context)
        
        if err != nil {
            retryCount++
            taskResult.RetryCount = retryCount
            
            if retryCount > taskDef.RetryCount {
                // 最終的に失敗
                taskResult.SetStatus(TaskStatusFailed)
                taskResult.SetError(err.Error())
                now := time.Now()
                taskResult.EndTime = &now
                
                // エラーハンドリング
                oe.handleTaskFailure(ctx, execution, taskDef, taskResult)
                return
            } else {
                // リトライ
                time.Sleep(taskDef.RetryDelay)
                continue
            }
        }
        
        // 成功
        taskResult.SetStatus(TaskStatusCompleted)
        taskResult.SetResult(result)
        now := time.Now()
        taskResult.EndTime = &now
        
        // コンテキスト更新
        if resultMap, ok := result.(map[string]interface{}); ok {
            for k, v := range resultMap {
                execution.UpdateContext(k, v)
            }
        } else {
            execution.UpdateContext(fmt.Sprintf("%s_result", taskID), result)
        }
        
        break
    }
}

// handleTaskFailure タスク失敗処理
func (oe *OrchestrationEngine) handleTaskFailure(ctx context.Context, execution *WorkflowExecution, taskDef *TaskDefinition, taskResult *TaskResult) {
    errorHandling := execution.WorkflowDefinition.ErrorHandling
    
    switch errorHandling {
    case ErrorHandlingFailFast:
        execution.SetStatus(WorkflowStatusFailed)
        execution.SetError(fmt.Sprintf("Task %s failed: %s", taskDef.TaskID, taskResult.Error))
        
    case ErrorHandlingCompensate:
        if taskDef.CompensateWith != nil {
            // 補償処理実行
            oe.executeCompensation(ctx, execution, *taskDef.CompensateWith)
        }
    }
}

// executeCompensation 補償処理実行
func (oe *OrchestrationEngine) executeCompensation(ctx context.Context, execution *WorkflowExecution, compensateTaskID string) {
    oe.executeTask(ctx, execution, compensateTaskID)
}

// shouldStopExecution 実行停止判定
func (oe *OrchestrationEngine) shouldStopExecution(execution *WorkflowExecution) bool {
    if execution.Status != WorkflowStatusRunning {
        return true
    }
    
    errorHandling := execution.WorkflowDefinition.ErrorHandling
    
    if errorHandling == ErrorHandlingFailFast {
        return len(execution.GetFailedTasks()) > 0
    }
    
    return false
}

// allTasksCompleted 全タスク完了判定
func (oe *OrchestrationEngine) allTasksCompleted(execution *WorkflowExecution) bool {
    for _, task := range execution.WorkflowDefinition.Tasks {
        taskResult := execution.GetTaskResult(task.TaskID)
        if taskResult == nil || (taskResult.Status != TaskStatusCompleted && taskResult.Status != TaskStatusSkipped) {
            return false
        }
    }
    return true
}

// GetExecutionStatus 実行状態取得
func (oe *OrchestrationEngine) GetExecutionStatus(executionID string) *WorkflowExecution {
    oe.mu.RLock()
    defer oe.mu.RUnlock()
    return oe.executions[executionID]
}

// CancelWorkflow ワークフローキャンセル
func (oe *OrchestrationEngine) CancelWorkflow(executionID string) bool {
    oe.mu.Lock()
    defer oe.mu.Unlock()
    
    execution, exists := oe.executions[executionID]
    if !exists {
        return false
    }
    
    if execution.Status == WorkflowStatusRunning {
        execution.SetStatus(WorkflowStatusCancelled)
        now := time.Now()
        execution.EndTime = &now
        if execution.cancelFunc != nil {
            execution.cancelFunc()
        }
        return true
    }
    
    return false
}
```

### 4. ワークフロー定義ビルダー
```go
// WorkflowBuilder ワークフロー定義ビルダー
type WorkflowBuilder struct {
    workflowDefinition *WorkflowDefinition
}

// NewWorkflowBuilder ワークフロービルダー作成
func NewWorkflowBuilder(workflowID, name string) *WorkflowBuilder {
    return &WorkflowBuilder{
        workflowDefinition: NewWorkflowDefinition(workflowID, name, ""),
    }
}

// Description 説明設定
func (wb *WorkflowBuilder) Description(desc string) *WorkflowBuilder {
    wb.workflowDefinition.Description = desc
    return wb
}

// GlobalTimeout グローバルタイムアウト設定
func (wb *WorkflowBuilder) GlobalTimeout(timeout time.Duration) *WorkflowBuilder {
    wb.workflowDefinition.GlobalTimeout = &timeout
    return wb
}

// ErrorHandling エラーハンドリング戦略設定
func (wb *WorkflowBuilder) ErrorHandling(strategy ErrorHandlingStrategy) *WorkflowBuilder {
    wb.workflowDefinition.ErrorHandling = strategy
    return wb
}

// Task タスク追加
func (wb *WorkflowBuilder) Task(taskID, name, serviceName, method string) *TaskBuilder {
    taskDef := NewTaskDefinition(taskID, name, serviceName, method)
    wb.workflowDefinition.AddTask(taskDef)
    return NewTaskBuilder(wb, taskDef)
}

// Build ワークフロー定義構築
func (wb *WorkflowBuilder) Build() *WorkflowDefinition {
    return wb.workflowDefinition
}

// TaskBuilder タスクビルダー
type TaskBuilder struct {
    workflowBuilder *WorkflowBuilder
    taskDef         *TaskDefinition
}

// NewTaskBuilder タスクビルダー作成
func NewTaskBuilder(workflowBuilder *WorkflowBuilder, taskDef *TaskDefinition) *TaskBuilder {
    return &TaskBuilder{
        workflowBuilder: workflowBuilder,
        taskDef:         taskDef,
    }
}

// Parameters パラメータ設定
func (tb *TaskBuilder) Parameters(params map[string]interface{}) *TaskBuilder {
    tb.taskDef.Parameters = params
    return tb
}

// Timeout タイムアウト設定
func (tb *TaskBuilder) Timeout(timeout time.Duration) *TaskBuilder {
    tb.taskDef.Timeout = &timeout
    return tb
}

// Retry リトライ設定
func (tb *TaskBuilder) Retry(count int, delay time.Duration) *TaskBuilder {
    tb.taskDef.RetryCount = count
    tb.taskDef.RetryDelay = delay
    return tb
}

// DependsOn 依存関係設定
func (tb *TaskBuilder) DependsOn(taskIDs ...string) *TaskBuilder {
    tb.taskDef.DependsOn = append(tb.taskDef.DependsOn, taskIDs...)
    return tb
}

// Condition 実行条件設定
func (tb *TaskBuilder) Condition(cond string) *TaskBuilder {
    tb.taskDef.Condition = &cond
    return tb
}

// CompensateWith 補償処理設定
func (tb *TaskBuilder) CompensateWith(taskID string) *TaskBuilder {
    tb.taskDef.CompensateWith = &taskID
    return tb
}

// Task 次のタスク追加
func (tb *TaskBuilder) Task(taskID, name, serviceName, method string) *TaskBuilder {
    return tb.workflowBuilder.Task(taskID, name, serviceName, method)
}

// Build ワークフロー定義構築
func (tb *TaskBuilder) Build() *WorkflowDefinition {
    return tb.workflowBuilder.Build()
}
```

## 実装例

### 1. ユーザー登録ワークフロー
```go
package main

package main

    // context
    // fmt
    // log
    // time
)

func main() {
    // サービス登録
    serviceRegistry := NewServiceRegistry()
    serviceRegistry.RegisterService(// UserService, NewUserService())
    serviceRegistry.RegisterService(// EmailService, NewEmailService())
    
    // オーケストレーションエンジン作成
    engine := NewOrchestrationEngine(serviceRegistry)
    
    // ワークフロー定義
    workflow := NewWorkflowBuilder(// user_registration, // User Registration Workflow).
        Description(// New user registration with email verification).
        GlobalTimeout(5 * time.Minute).
        ErrorHandling(ErrorHandlingCompensate).
        
        Task(// create_user, // Create User, // UserService, // create_user).
        Parameters(map[string]interface{}{
            // name:  // John Doe,
            // email: "john        }).
        Timeout(30 * time.Second).
        Retry(2, 2*time.Second).
        CompensateWith(// delete_user).
        
        Task(// send_welcome_email, // Send Welcome Email, // EmailService, // send_welcome_email).
        DependsOn(// create_user).
        Timeout(10 * time.Second).
        Retry(3, 1*time.Second).
        
        Task(// delete_user, // Delete User (Compensation), // UserService, // delete_user).
        Parameters(map[string]interface{}{
            // user_id: // ${user_id}, // 動的パラメータ
        }).
        
        Build()
    
    // ワークフロー実行
    ctx := context.Background()
    executionID, err := engine.ExecuteWorkflow(ctx, workflow, nil)
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Printf(// Workflow started: %s\n, executionID)
    
    // 実行状態監視
    for {
        execution := engine.GetExecutionStatus(executionID)
        if execution == nil {
            break
        }
        
        fmt.Printf(// Status: %s\n, execution.Status)
        
        if execution.Status == WorkflowStatusCompleted || 
           execution.Status == WorkflowStatusFailed || 
           execution.Status == WorkflowStatusCancelled {
            break
        }
        
        // タスクの状態表示
        execution.mu.RLock()
        for taskID, result := range execution.TaskResults {
            fmt.Printf(//   Task %s: %s, taskID, result.Status)
            if result.Error != "" {
                fmt.Printf(//  (Error: %s), result.Error)
            }
            fmt.Println()
        }
        execution.mu.RUnlock()
        
        time.Sleep(1 * time.Second)
    }
    
    // 最終結果
    finalExecution := engine.GetExecutionStatus(executionID)
    if finalExecution != nil {
        fmt.Printf(// Final status: %s\n, finalExecution.Status)
        if finalExecution.StartTime != nil && finalExecution.EndTime != nil {
            duration := finalExecution.EndTime.Sub(*finalExecution.StartTime)
            fmt.Printf(// Duration: %v\n, duration)
        }
    }
}
```

### 2. E コマース注文処理ワークフロー
```go
// InventoryService 在庫サービス実装
type InventoryService struct {
    inventory map[string]int
    mu        sync.RWMutex
}

// NewInventoryService 在庫サービス作成
func NewInventoryService() *InventoryService {
    return &InventoryService{
        inventory: map[string]int{
            "product_1": 10,
            "product_2": 5,
        },
    }
}

// Execute メソッド実行
func (is *InventoryService) Execute(ctx context.Context, method string, parameters map[string]interface{}, context map[string]interface{}) (interface{}, error) {
    switch method {
    case "check_availability":
        return is.checkAvailability(ctx, parameters)
    case "reserve_stock":
        return is.reserveStock(ctx, parameters)
    case "release_stock":
        return is.releaseStock(ctx, parameters)
    default:
        return nil, fmt.Errorf("unknown method: %s", method)
    }
}

// checkAvailability 在庫確認
func (is *InventoryService) checkAvailability(ctx context.Context, params map[string]interface{}) (interface{}, error) {
    productID, _ := params["product_id"].(string)
    quantity, _ := params["quantity"].(float64)
    if quantity == 0 {
        quantity = 1
    }
    
    is.mu.RLock()
    stock := is.inventory[productID]
    is.mu.RUnlock()
    
    available := stock >= int(quantity)
    
    return map[string]interface{}{
        "available": available,
        "stock":     stock,
    }, nil
}

// reserveStock 在庫予約
func (is *InventoryService) reserveStock(ctx context.Context, params map[string]interface{}) (interface{}, error) {
    productID, _ := params["product_id"].(string)
    quantity, _ := params["quantity"].(float64)
    if quantity == 0 {
        quantity = 1
    }
    
    is.mu.Lock()
    defer is.mu.Unlock()
    
    if is.inventory[productID] >= int(quantity) {
        is.inventory[productID] -= int(quantity)
        return map[string]interface{}{
            "reserved":        true,
            "remaining_stock": is.inventory[productID],
        }, nil
    }
    
    return nil, errors.New("insufficient stock")
}

// releaseStock 在庫解放
func (is *InventoryService) releaseStock(ctx context.Context, params map[string]interface{}) (interface{}, error) {
    productID, _ := params["product_id"].(string)
    quantity, _ := params["quantity"].(float64)
    if quantity == 0 {
        quantity = 1
    }
    
    is.mu.Lock()
    defer is.mu.Unlock()
    
    is.inventory[productID] += int(quantity)
    
    return map[string]interface{}{
        "released":      true,
        "current_stock": is.inventory[productID],
    }, nil
}

// HealthCheck ヘルスチェック
func (is *InventoryService) HealthCheck(ctx context.Context) error {
    return nil
}

// GetName サービス名取得
func (is *InventoryService) GetName() string {
    return "InventoryService"
}

// PaymentService 決済サービス実装
type PaymentService struct {
    transactions map[string]map[string]interface{}
    mu           sync.RWMutex
}

// NewPaymentService 決済サービス作成
func NewPaymentService() *PaymentService {
    return &PaymentService{
        transactions: make(map[string]map[string]interface{}),
    }
}

// Execute メソッド実行
func (ps *PaymentService) Execute(ctx context.Context, method string, parameters map[string]interface{}, context map[string]interface{}) (interface{}, error) {
    switch method {
    case "process_payment":
        return ps.processPayment(ctx, parameters)
    case "refund_payment":
        return ps.refundPayment(ctx, parameters)
    default:
        return nil, fmt.Errorf("unknown method: %s", method)
    }
}

// processPayment 決済処理
func (ps *PaymentService) processPayment(ctx context.Context, params map[string]interface{}) (interface{}, error) {
    transactionID := uuid.New().String()
    amount, _ := params["amount"].(float64)
    
    // 決済処理シミュレーション
    if amount <= 0 {
        return nil, errors.New("invalid amount")
    }
    
    ps.mu.Lock()
    ps.transactions[transactionID] = map[string]interface{}{
        "amount":       amount,
        "status":       "completed",
        "processed_at": time.Now().Format(time.RFC3339),
    }
    ps.mu.Unlock()
    
    return map[string]interface{}{
        "transaction_id": transactionID,
        "status":         "completed",
        "amount":         amount,
    }, nil
}

// refundPayment 決済返金
func (ps *PaymentService) refundPayment(ctx context.Context, params map[string]interface{}) (interface{}, error) {
    transactionID, _ := params["transaction_id"].(string)
    
    ps.mu.Lock()
    defer ps.mu.Unlock()
    
    if transaction, exists := ps.transactions[transactionID]; exists {
        transaction["status"] = "refunded"
        return map[string]interface{}{
            "refund_status": "completed",
        }, nil
    }
    
    return nil, errors.New("transaction not found")
}

// HealthCheck ヘルスチェック
func (ps *PaymentService) HealthCheck(ctx context.Context) error {
    return nil
}

// GetName サービス名取得
func (ps *PaymentService) GetName() string {
    return "PaymentService"
}

// createEcommerceWorkflow E コマース注文処理ワークフロー作成
func createEcommerceWorkflow() (*OrchestrationEngine, *WorkflowDefinition) {
    // サービス登録
    serviceRegistry := NewServiceRegistry()
    serviceRegistry.RegisterService("InventoryService", NewInventoryService())
    serviceRegistry.RegisterService("PaymentService", NewPaymentService())
    serviceRegistry.RegisterService("EmailService", NewEmailService())
    
    engine := NewOrchestrationEngine(serviceRegistry)
    
    // 複雑なワークフロー定義
    workflow := NewWorkflowBuilder("ecommerce_order", "E-commerce Order Processing").
        Description("Complete order processing with inventory, payment, and notifications").
        ErrorHandling(ErrorHandlingCompensate).
        
        // 在庫確認
        Task("check_inventory", "Check Product Availability", "InventoryService", "check_availability").
        Parameters(map[string]interface{}{
            "product_id": "product_1",
            "quantity":   2,
        }).
        Timeout(10 * time.Second).
        Retry(1, time.Second).
        
        // 在庫予約（在庫が十分な場合のみ）
        Task("reserve_stock", "Reserve Stock", "InventoryService", "reserve_stock").
        Parameters(map[string]interface{}{
            "product_id": "product_1",
            "quantity":   2,
        }).
        DependsOn("check_inventory").
        Condition("available == true"). // 在庫確認結果をチェック
        CompensateWith("release_stock").
        Timeout(10 * time.Second).
        
        // 決済処理
        Task("process_payment", "Process Payment", "PaymentService", "process_payment").
        Parameters(map[string]interface{}{
            "amount": 199.99,
        }).
        DependsOn("reserve_stock").
        CompensateWith("refund_payment").
        Timeout(30 * time.Second).
        Retry(2, 3*time.Second).
        
        // 注文確認メール送信
        Task("send_confirmation", "Send Order Confirmation", "EmailService", "send_email").
        Parameters(map[string]interface{}{
            "to":      "customer@example.com",
            "subject": "Order Confirmation",
            "body":    "Your order has been processed successfully.",
        }).
        DependsOn("process_payment").
        Timeout(15 * time.Second).
        Retry(3, 2*time.Second).
        
        // 補償タスク
        Task("release_stock", "Release Reserved Stock", "InventoryService", "release_stock").
        Parameters(map[string]interface{}{
            "product_id": "product_1",
            "quantity":   2,
        }).
        
        Task("refund_payment", "Refund Payment", "PaymentService", "refund_payment").
        Parameters(map[string]interface{}{
            "transaction_id": "${transaction_id}",
        }).
        
        Build()
    
    return engine, workflow
}

func runEcommerceExample() {
    engine, workflow := createEcommerceWorkflow()
    
    // 初期コンテキスト
    initialContext := map[string]interface{}{
        "customer_id": "cust_123",
        "order_id":    uuid.New().String(),
    }
    
    ctx := context.Background()
    executionID, err := engine.ExecuteWorkflow(ctx, workflow, initialContext)
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Printf("E-commerce workflow started: %s\n", executionID)
    
    // 監視
    for {
        execution := engine.GetExecutionStatus(executionID)
        if execution == nil {
            break
        }
        
        fmt.Printf("\nWorkflow Status: %s\n", execution.Status)
        fmt.Println("Task Results:")
        
        execution.mu.RLock()
        for taskID, result := range execution.TaskResults {
            statusSymbol := map[TaskStatus]string{
                TaskStatusPending:   "⏳",
                TaskStatusRunning:   "🔄",
                TaskStatusCompleted: "✅",
                TaskStatusFailed:    "❌",
                TaskStatusSkipped:   "⏭️",
            }[result.Status]
            
            if statusSymbol == "" {
                statusSymbol = "❓"
            }
            
            fmt.Printf("  %s %s: %s\n", statusSymbol, taskID, result.Status)
            
            if duration := result.Duration(); duration != nil {
                fmt.Printf("    Duration: %v\n", *duration)
            }
            if result.Error != "" {
                fmt.Printf("    Error: %s\n", result.Error)
            }
            if result.Result != nil {
                fmt.Printf("    Result: %v\n", result.Result)
            }
        }
        execution.mu.RUnlock()
        
        if execution.Status == WorkflowStatusCompleted || 
           execution.Status == WorkflowStatusFailed || 
           execution.Status == WorkflowStatusCancelled {
            break
        }
        
        time.Sleep(1 * time.Second)
    }
}

func main() {
    runEcommerceExample()
}
```

## 高度な機能

### 1. 並列実行とバリア同期
```go
// ParallelExecutionEngine 並列実行対応オーケストレーションエンジン
type ParallelExecutionEngine struct {
    *OrchestrationEngine
}

// NewParallelExecutionEngine 並列実行エンジン作成
func NewParallelExecutionEngine(serviceRegistry *ServiceRegistry) *ParallelExecutionEngine {
    return &ParallelExecutionEngine{
        OrchestrationEngine: NewOrchestrationEngine(serviceRegistry),
    }
}

// executeWorkflowAsync 並列実行対応ワークフロー実行
func (pee *ParallelExecutionEngine) executeWorkflowAsync(ctx context.Context, execution *WorkflowExecution) {
    defer func() {
        now := time.Now()
        execution.EndTime = &now
    }()
    
    workflowCtx := ctx
    if execution.WorkflowDefinition.GlobalTimeout != nil {
        var cancel context.CancelFunc
        workflowCtx, cancel = context.WithTimeout(ctx, *execution.WorkflowDefinition.GlobalTimeout)
        defer cancel()
    }
    
    execution.SetStatus(WorkflowStatusRunning)
    
    for {
        select {
        case <-workflowCtx.Done():
            execution.SetStatus(WorkflowStatusCancelled)
            execution.SetError("workflow timeout or cancelled")
            return
        default:
        }
        
        readyTasks := pee.getReadyTasks(execution)
        
        if len(readyTasks) == 0 {
            if pee.hasRunningTasks(execution) {
                time.Sleep(100 * time.Millisecond)
                continue
            }
            break
        }
        
        // 並列グループの特定
        parallelGroups := pee.groupParallelTasks(execution, readyTasks)
        
        for _, group := range parallelGroups {
            // グループ内タスクを並列実行
            var wg sync.WaitGroup
            for _, taskID := range group {
                wg.Add(1)
                go func(id string) {
                    defer wg.Done()
                    pee.executeTask(workflowCtx, execution, id)
                }(taskID)
            }
            wg.Wait()
            
            // エラーチェック
            if pee.shouldStopExecution(execution) {
                break
            }
        }
    }
    
    // 完了判定
    if pee.allTasksCompleted(execution) {
        execution.SetStatus(WorkflowStatusCompleted)
    } else if execution.Status == WorkflowStatusRunning {
        execution.SetStatus(WorkflowStatusFailed)
    }
}

// groupParallelTasks 並列実行可能なタスクをグループ化
func (pee *ParallelExecutionEngine) groupParallelTasks(execution *WorkflowExecution, readyTasks []string) [][]string {
    // 同じ依存関係を持つタスクは並列実行可能
    var groups [][]string
    remainingTasks := make([]string, len(readyTasks))
    copy(remainingTasks, readyTasks)
    
    for len(remainingTasks) > 0 {
        currentGroup := []string{remainingTasks[0]}
        currentDeps := execution.WorkflowDefinition.GetDependencies(remainingTasks[0])
        remainingTasks = remainingTasks[1:]
        
        // 同じ依存関係を持つタスクを同一グループに追加
        i := 0
        for i < len(remainingTasks) {
            taskDeps := execution.WorkflowDefinition.GetDependencies(remainingTasks[i])
            if pee.slicesEqual(taskDeps, currentDeps) {
                currentGroup = append(currentGroup, remainingTasks[i])
                remainingTasks = append(remainingTasks[:i], remainingTasks[i+1:]...)
            } else {
                i++
            }
        }
        
        groups = append(groups, currentGroup)
    }
    
    return groups
}

// slicesEqual スライス比較
func (pee *ParallelExecutionEngine) slicesEqual(a, b []string) bool {
    if len(a) != len(b) {
        return false
    }
    
    aMap := make(map[string]bool)
    for _, item := range a {
        aMap[item] = true
    }
    
    for _, item := range b {
        if !aMap[item] {
            return false
        }
    }
    
    return true
}

// BarrierTask バリアタスク（複数タスクの完了を待機）
type BarrierTask struct {
    *TaskDefinition
    WaitForTasks []string
}

// NewBarrierTask バリアタスク作成
func NewBarrierTask(taskID string, waitForTasks []string) *BarrierTask {
    taskDef := NewTaskDefinition(taskID, fmt.Sprintf("Barrier for %v", waitForTasks), "internal", "barrier")
    taskDef.DependsOn = waitForTasks
    
    return &BarrierTask{
        TaskDefinition: taskDef,
        WaitForTasks:   waitForTasks,
    }
}

// InternalService 内部サービス（バリア等）
type InternalService struct{}

// NewInternalService 内部サービス作成
func NewInternalService() *InternalService {
    return &InternalService{}
}

// Execute メソッド実行
func (is *InternalService) Execute(ctx context.Context, method string, parameters map[string]interface{}, context map[string]interface{}) (interface{}, error) {
    switch method {
    case "barrier":
        return map[string]interface{}{"barrier_passed": true}, nil
    case "delay":
        delaySeconds, _ := parameters["seconds"].(float64)
        if delaySeconds == 0 {
            delaySeconds = 1
        }
        time.Sleep(time.Duration(delaySeconds) * time.Second)
        return map[string]interface{}{"delayed": delaySeconds}, nil
    default:
        return nil, fmt.Errorf("unknown internal method: %s", method)
    }
}

// HealthCheck ヘルスチェック
func (is *InternalService) HealthCheck(ctx context.Context) error {
    return nil
}

// GetName サービス名取得
func (is *InternalService) GetName() string {
    return "InternalService"
}
```

### 2. 動的ワークフロー
```go
// DynamicWorkflowEngine 動的ワークフロー対応エンジン
type DynamicWorkflowEngine struct {
    *OrchestrationEngine
}

// NewDynamicWorkflowEngine 動的ワークフローエンジン作成
func NewDynamicWorkflowEngine(serviceRegistry *ServiceRegistry) *DynamicWorkflowEngine {
    return &DynamicWorkflowEngine{
        OrchestrationEngine: NewOrchestrationEngine(serviceRegistry),
    }
}

// AddDynamicTask 実行時のタスク追加
func (dwe *DynamicWorkflowEngine) AddDynamicTask(executionID string, taskDef *TaskDefinition) bool {
    dwe.mu.Lock()
    defer dwe.mu.Unlock()
    
    execution, exists := dwe.executions[executionID]
    if !exists || execution.Status != WorkflowStatusRunning {
        return false
    }
    
    // タスク定義に追加
    execution.WorkflowDefinition.AddTask(taskDef)
    return true
}

// SkipTask タスクのスキップ
func (dwe *DynamicWorkflowEngine) SkipTask(executionID, taskID string) bool {
    dwe.mu.Lock()
    defer dwe.mu.Unlock()
    
    execution, exists := dwe.executions[executionID]
    if !exists {
        return false
    }
    
    taskResult := execution.GetTaskResult(taskID)
    if taskResult != nil && taskResult.Status == TaskStatusPending {
        taskResult.SetStatus(TaskStatusSkipped)
        now := time.Now()
        taskResult.EndTime = &now
        return true
    }
    
    return false
}

// ModifyTaskParameters タスクパラメータの動的変更
func (dwe *DynamicWorkflowEngine) ModifyTaskParameters(executionID, taskID string, newParameters map[string]interface{}) bool {
    dwe.mu.Lock()
    defer dwe.mu.Unlock()
    
    execution, exists := dwe.executions[executionID]
    if !exists {
        return false
    }
    
    taskDef := execution.WorkflowDefinition.GetTask(taskID)
    if taskDef != nil {
        taskResult := execution.GetTaskResult(taskID)
        if taskResult == nil || taskResult.Status == TaskStatusPending {
            for k, v := range newParameters {
                taskDef.Parameters[k] = v
            }
            return true
        }
    }
    
    return false
}

// ConditionalWorkflowBuilder 条件分岐対応ワークフロービルダー
type ConditionalWorkflowBuilder struct {
    *WorkflowBuilder
}

// NewConditionalWorkflowBuilder 条件分岐ワークフロービルダー作成
func NewConditionalWorkflowBuilder(workflowID, name string) *ConditionalWorkflowBuilder {
    return &ConditionalWorkflowBuilder{
        WorkflowBuilder: NewWorkflowBuilder(workflowID, name),
    }
}

// ConditionalBranch 条件分岐開始
func (cwb *ConditionalWorkflowBuilder) ConditionalBranch(condition string) *ConditionalBranchBuilder {
    return NewConditionalBranchBuilder(cwb.WorkflowBuilder, condition)
}

// ConditionalBranchBuilder 条件分岐ビルダー
type ConditionalBranchBuilder struct {
    parentBuilder   *WorkflowBuilder
    condition       string
    ifTasks         []*TaskDefinition
    elseTasks       []*TaskDefinition
    currentBranch   string
}

// NewConditionalBranchBuilder 条件分岐ビルダー作成
func NewConditionalBranchBuilder(parentBuilder *WorkflowBuilder, condition string) *ConditionalBranchBuilder {
    return &ConditionalBranchBuilder{
        parentBuilder: parentBuilder,
        condition:     condition,
        ifTasks:       make([]*TaskDefinition, 0),
        elseTasks:     make([]*TaskDefinition, 0),
        currentBranch: "if",
    }
}

// IfTask if分岐のタスク
func (cbb *ConditionalBranchBuilder) IfTask(taskID, name, serviceName, method string) *TaskBuilder {
    cbb.currentBranch = "if"
    taskDef := NewTaskDefinition(taskID, name, serviceName, method)
    taskDef.Condition = &cbb.condition
    
    cbb.ifTasks = append(cbb.ifTasks, taskDef)
    cbb.parentBuilder.workflowDefinition.AddTask(taskDef)
    
    return NewTaskBuilder(cbb.parentBuilder, taskDef)
}

// ElseTask else分岐のタスク
func (cbb *ConditionalBranchBuilder) ElseTask(taskID, name, serviceName, method string) *TaskBuilder {
    cbb.currentBranch = "else"
    negatedCondition := fmt.Sprintf("not (%s)", cbb.condition)
    taskDef := NewTaskDefinition(taskID, name, serviceName, method)
    taskDef.Condition = &negatedCondition
    
    cbb.elseTasks = append(cbb.elseTasks, taskDef)
    cbb.parentBuilder.workflowDefinition.AddTask(taskDef)
    
    return NewTaskBuilder(cbb.parentBuilder, taskDef)
}

// EndBranch 分岐終了
func (cbb *ConditionalBranchBuilder) EndBranch() *WorkflowBuilder {
    return cbb.parentBuilder
}

// ループワークフロー
type LoopWorkflowBuilder struct {
    *WorkflowBuilder
    loopCounter int
}

// NewLoopWorkflowBuilder ループワークフロービルダー作成
func NewLoopWorkflowBuilder(workflowID, name string) *LoopWorkflowBuilder {
    return &LoopWorkflowBuilder{
        WorkflowBuilder: NewWorkflowBuilder(workflowID, name),
        loopCounter:     0,
    }
}

// ForEachTask 反復タスク
func (lwb *LoopWorkflowBuilder) ForEachTask(taskID, name, serviceName, method string, items []interface{}) *WorkflowBuilder {
    for i, item := range items {
        iterTaskID := fmt.Sprintf("%s_%d", taskID, i)
        iterName := fmt.Sprintf("%s (Item %d)", name, i)
        
        taskDef := NewTaskDefinition(iterTaskID, iterName, serviceName, method)
        taskDef.Parameters["item"] = item
        taskDef.Parameters["index"] = i
        
        lwb.workflowDefinition.AddTask(taskDef)
    }
    
    return lwb.WorkflowBuilder
}

// WhileTask 条件付きループタスク
func (lwb *LoopWorkflowBuilder) WhileTask(taskID, name, serviceName, method string, condition string, maxIterations int) *WorkflowBuilder {
    for i := 0; i < maxIterations; i++ {
        iterTaskID := fmt.Sprintf("%s_%d", taskID, i)
        iterName := fmt.Sprintf("%s (Iteration %d)", name, i)
        
        taskDef := NewTaskDefinition(iterTaskID, iterName, serviceName, method)
        taskDef.Condition = &condition
        taskDef.Parameters["iteration"] = i
        
        // 前の反復に依存
        if i > 0 {
            prevTaskID := fmt.Sprintf("%s_%d", taskID, i-1)
            taskDef.DependsOn = []string{prevTaskID}
        }
        
        lwb.workflowDefinition.AddTask(taskDef)
    }
    
    return lwb.WorkflowBuilder
}
```

### 3. モニタリングと観測可能性
```go
// WorkflowEventListener ワークフローイベントリスナー
type WorkflowEventListener interface {
    OnWorkflowStarted(ctx context.Context, execution *WorkflowExecution) error
    OnWorkflowCompleted(ctx context.Context, execution *WorkflowExecution) error
    OnWorkflowFailed(ctx context.Context, execution *WorkflowExecution) error
    OnTaskStarted(ctx context.Context, execution *WorkflowExecution, taskID string) error
    OnTaskCompleted(ctx context.Context, execution *WorkflowExecution, taskID string, result *TaskResult) error
    OnTaskFailed(ctx context.Context, execution *WorkflowExecution, taskID string, result *TaskResult) error
}

// MetricsCollector メトリクス収集器
type MetricsCollector struct {
    workflowMetrics map[string]map[string]interface{}
    taskMetrics     map[string]map[string]interface{}
    mu              sync.RWMutex
}

// NewMetricsCollector メトリクス収集器作成
func NewMetricsCollector() *MetricsCollector {
    return &MetricsCollector{
        workflowMetrics: make(map[string]map[string]interface{}),
        taskMetrics:     make(map[string]map[string]interface{}),
    }
}

// OnWorkflowStarted ワークフロー開始メトリクス
func (mc *MetricsCollector) OnWorkflowStarted(ctx context.Context, execution *WorkflowExecution) error {
    mc.mu.Lock()
    defer mc.mu.Unlock()
    
    mc.workflowMetrics[execution.ExecutionID] = map[string]interface{}{
        "start_time":   execution.StartTime,
        "workflow_id":  execution.WorkflowDefinition.WorkflowID,
        "task_count":   len(execution.WorkflowDefinition.Tasks),
    }
    
    return nil
}

// OnWorkflowCompleted ワークフロー完了メトリクス
func (mc *MetricsCollector) OnWorkflowCompleted(ctx context.Context, execution *WorkflowExecution) error {
    mc.mu.Lock()
    defer mc.mu.Unlock()
    
    metrics := mc.workflowMetrics[execution.ExecutionID]
    if metrics == nil {
        metrics = make(map[string]interface{})
        mc.workflowMetrics[execution.ExecutionID] = metrics
    }
    
    var duration *time.Duration
    if execution.StartTime != nil && execution.EndTime != nil {
        d := execution.EndTime.Sub(*execution.StartTime)
        duration = &d
    }
    
    metrics["end_time"] = execution.EndTime
    metrics["duration"] = duration
    metrics["status"] = execution.Status
    metrics["completed_tasks"] = len(execution.GetCompletedTasks())
    metrics["failed_tasks"] = len(execution.GetFailedTasks())
    
    return nil
}

// OnWorkflowFailed ワークフロー失敗メトリクス
func (mc *MetricsCollector) OnWorkflowFailed(ctx context.Context, execution *WorkflowExecution) error {
    return mc.OnWorkflowCompleted(ctx, execution)
}

// OnTaskStarted タスク開始メトリクス
func (mc *MetricsCollector) OnTaskStarted(ctx context.Context, execution *WorkflowExecution, taskID string) error {
    // タスク開始のメトリクス収集
    return nil
}

// OnTaskCompleted タスク完了メトリクス
func (mc *MetricsCollector) OnTaskCompleted(ctx context.Context, execution *WorkflowExecution, taskID string, result *TaskResult) error {
    mc.mu.Lock()
    defer mc.mu.Unlock()
    
    key := fmt.Sprintf("%s:%s", execution.ExecutionID, taskID)
    mc.taskMetrics[key] = map[string]interface{}{
        "duration":     result.Duration(),
        "retry_count":  result.RetryCount,
        "status":       result.Status,
    }
    
    return nil
}

// OnTaskFailed タスク失敗メトリクス
func (mc *MetricsCollector) OnTaskFailed(ctx context.Context, execution *WorkflowExecution, taskID string, result *TaskResult) error {
    return mc.OnTaskCompleted(ctx, execution, taskID, result)
}

// GetWorkflowMetrics ワークフローメトリクス取得
func (mc *MetricsCollector) GetWorkflowMetrics(executionID string) map[string]interface{} {
    mc.mu.RLock()
    defer mc.mu.RUnlock()
    
    metrics := mc.workflowMetrics[executionID]
    if metrics == nil {
        return make(map[string]interface{})
    }
    
    // コピーを返す
    result := make(map[string]interface{})
    for k, v := range metrics {
        result[k] = v
    }
    return result
}

// GetAverageTaskDuration タスクの平均実行時間
func (mc *MetricsCollector) GetAverageTaskDuration(taskID string) *time.Duration {
    mc.mu.RLock()
    defer mc.mu.RUnlock()
    
    var durations []time.Duration
    for key, metrics := range mc.taskMetrics {
        if duration, ok := metrics["duration"].(*time.Duration); ok && duration != nil {
            durations = append(durations, *duration)
        }
    }
    
    if len(durations) > 0 {
        var total time.Duration
        for _, d := range durations {
            total += d
        }
        average := total / time.Duration(len(durations))
        return &average
    }
    
    return nil
}

// ObservableOrchestrationEngine 観測可能なオーケストレーションエンジン
type ObservableOrchestrationEngine struct {
    *OrchestrationEngine
    eventListeners []WorkflowEventListener
    mu             sync.RWMutex
}

// NewObservableOrchestrationEngine 観測可能なオーケストレーションエンジン作成
func NewObservableOrchestrationEngine(serviceRegistry *ServiceRegistry) *ObservableOrchestrationEngine {
    return &ObservableOrchestrationEngine{
        OrchestrationEngine: NewOrchestrationEngine(serviceRegistry),
        eventListeners:      make([]WorkflowEventListener, 0),
    }
}

// AddEventListener イベントリスナー追加
func (ooe *ObservableOrchestrationEngine) AddEventListener(listener WorkflowEventListener) {
    ooe.mu.Lock()
    defer ooe.mu.Unlock()
    ooe.eventListeners = append(ooe.eventListeners, listener)
}

// executeWorkflowAsync 観測可能なワークフロー実行
func (ooe *ObservableOrchestrationEngine) executeWorkflowAsync(ctx context.Context, execution *WorkflowExecution) {
    // 開始イベント
    ooe.mu.RLock()
    listeners := make([]WorkflowEventListener, len(ooe.eventListeners))
    copy(listeners, ooe.eventListeners)
    ooe.mu.RUnlock()
    
    for _, listener := range listeners {
        listener.OnWorkflowStarted(ctx, execution)
    }
    
    defer func() {
        now := time.Now()
        execution.EndTime = &now
        
        // 完了イベント
        if execution.Status == WorkflowStatusCompleted {
            for _, listener := range listeners {
                listener.OnWorkflowCompleted(ctx, execution)
            }
        } else if execution.Status == WorkflowStatusFailed {
            for _, listener := range listeners {
                listener.OnWorkflowFailed(ctx, execution)
            }
        }
    }()
    
    // 元のワークフロー実行ロジック
    ooe.OrchestrationEngine.executeWorkflowAsync(ctx, execution)
}

// executeTask 観測可能なタスク実行
func (ooe *ObservableOrchestrationEngine) executeTask(ctx context.Context, execution *WorkflowExecution, taskID string) {
    // タスク開始イベント
    ooe.mu.RLock()
    listeners := make([]WorkflowEventListener, len(ooe.eventListeners))
    copy(listeners, ooe.eventListeners)
    ooe.mu.RUnlock()
    
    for _, listener := range listeners {
        listener.OnTaskStarted(ctx, execution, taskID)
    }
    
    // 元のタスク実行ロジック
    ooe.OrchestrationEngine.executeTask(ctx, execution, taskID)
    
    // タスク完了イベント
    taskResult := execution.GetTaskResult(taskID)
    if taskResult != nil {
        if taskResult.Status == TaskStatusCompleted {
            for _, listener := range listeners {
                listener.OnTaskCompleted(ctx, execution, taskID, taskResult)
            }
        } else if taskResult.Status == TaskStatusFailed {
            for _, listener := range listeners {
                listener.OnTaskFailed(ctx, execution, taskID, taskResult)
            }
        }
    }
}

// LoggingEventListener ログ出力イベントリスナー
type LoggingEventListener struct {
    logger *log.Logger
}

// NewLoggingEventListener ログイベントリスナー作成
func NewLoggingEventListener(logger *log.Logger) *LoggingEventListener {
    return &LoggingEventListener{logger: logger}
}

// OnWorkflowStarted ワークフロー開始ログ
func (lel *LoggingEventListener) OnWorkflowStarted(ctx context.Context, execution *WorkflowExecution) error {
    lel.logger.Printf("Workflow started: %s (%s)", execution.ExecutionID, execution.WorkflowDefinition.Name)
    return nil
}

// OnWorkflowCompleted ワークフロー完了ログ
func (lel *LoggingEventListener) OnWorkflowCompleted(ctx context.Context, execution *WorkflowExecution) error {
    duration := ""
    if execution.StartTime != nil && execution.EndTime != nil {
        duration = fmt.Sprintf(" in %v", execution.EndTime.Sub(*execution.StartTime))
    }
    lel.logger.Printf("Workflow completed: %s%s", execution.ExecutionID, duration)
    return nil
}

// OnWorkflowFailed ワークフロー失敗ログ
func (lel *LoggingEventListener) OnWorkflowFailed(ctx context.Context, execution *WorkflowExecution) error {
    lel.logger.Printf("Workflow failed: %s - %s", execution.ExecutionID, execution.Error)
    return nil
}

// OnTaskStarted タスク開始ログ
func (lel *LoggingEventListener) OnTaskStarted(ctx context.Context, execution *WorkflowExecution, taskID string) error {
    lel.logger.Printf("Task started: %s in workflow %s", taskID, execution.ExecutionID)
    return nil
}

// OnTaskCompleted タスク完了ログ
func (lel *LoggingEventListener) OnTaskCompleted(ctx context.Context, execution *WorkflowExecution, taskID string, result *TaskResult) error {
    duration := ""
    if d := result.Duration(); d != nil {
        duration = fmt.Sprintf(" in %v", *d)
    }
    lel.logger.Printf("Task completed: %s%s", taskID, duration)
    return nil
}

// OnTaskFailed タスク失敗ログ
func (lel *LoggingEventListener) OnTaskFailed(ctx context.Context, execution *WorkflowExecution, taskID string, result *TaskResult) error {
    lel.logger.Printf("Task failed: %s - %s", taskID, result.Error)
    return nil
}
```

## メリット

### 中央集権的制御
- **統一された制御**: 全体のワークフローを一箇所で管理
- **可視性**: ワークフローの状態と進行状況の完全な把握
- **デバッグ容易性**: 実行履歴とエラートレースの集中管理

### 複雑な制御フロー
- **条件分岐**: 動的な実行パス決定
- **並列実行**: 複数タスクの同時実行
- **エラーハンドリング**: 統一された例外処理と回復メカニズム

### ビジネスプロセス表現
- **宣言的定義**: ワークフローの明確な定義
- **再利用性**: ワークフロー定義の再利用
- **保守性**: ビジネスロジックの変更容易性

## デメリット

### 単一障害点
- **中央集権リスク**: オーケストレーターの障害が全体に影響
- **ボトルネック**: 全ての通信がオーケストレーター経由
- **スケーラビリティ制限**: 中央制御による処理能力の上限

### 密結合
- **サービス依存**: オーケストレーターがすべてのサービスを知る必要
- **変更の影響**: 一つのサービス変更が全体に波及
- **テスト複雑性**: 全体の統合テストが必要

### 運用複雑性
- **状態管理**: ワークフロー状態の永続化と管理
- **監視**: 複雑な実行状況の監視
- **パフォーマンス**: 多数のワークフロー同時実行時の性能

## 適用場面

### 適している場合
- **複雑なビジネスプロセス**: 多段階の業務フロー
- **トランザクション管理**: 分散トランザクションの制御
- **バッチ処理**: 大量データの段階的処理
- **承認ワークフロー**: 複数段階の承認プロセス

### 適していない場合
- **シンプルな処理**: 単純な一対一の通信
- **高スループット**: 極めて高い処理性能が必要
- **リアルタイム**: 低レイテンシが必須
- **独立性重視**: サービス間の完全な分離が必要

## 関連パターン
- **Saga Pattern**: 分散トランザクション管理
- **Command Pattern**: 実行可能な処理の抽象化
- **State Machine**: 状態遷移の管理
- **Pipeline Pattern**: 順次処理パイプライン
- **Workflow Pattern**: ビジネスプロセス管理

## 参考資料
- [Enterprise Integration Patterns - Gregor Hohpe](https://www.enterpriseintegrationpatterns.com/)
- [Microservices Patterns - Chris Richardson](https://microservices.io/patterns/)
- [Workflow Patterns](http://www.workflowpatterns.com/)
- [BPMN 2.0 Specification](https://www.omg.org/spec/BPMN/2.0/)
- [Camunda BPMN](https://camunda.com/bpmn/)
