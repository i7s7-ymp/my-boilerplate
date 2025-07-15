# Interpreter Pattern

## Overview
The Interpreter Pattern is an architectural pattern that defines grammar for a specific language or Domain-Specific Language (DSL) and interprets/executes sentences or expressions written in that language. It defines a class hierarchy representing grammatical rules and builds syntax trees for runtime interpretation.

## Basic Structure

```
┌─────────────────────────────────────────────────────┐
│                   Client                            │
│              (Expression Builder)                   │
├─────────────────────────────────────────────────────┤
│                      ↓                              │
├─────────────────────────────────────────────────────┤
│               AbstractExpression                    │
│                 (interpret())                       │
├─────────────────────────────────────────────────────┤
│         ↙               ↓               ↘           │
├─────────────────┬─────────────────┬─────────────────┤
│ TerminalExpr    │ NonterminalExpr │   CompoundExpr  │
│  (Variables,    │  (Operations,   │   (Complex      │
│   Constants)    │   Functions)    │    Expressions) │
└─────────────────┴─────────────────┴─────────────────┘
```

## Core Components

### 1. Abstract Expression
```go
package interpreter

import (
    "fmt"
    "sync"
)

// Context represents the interpreter context
type Context struct {
    variables map[string]interface{}
    functions map[string]func([]interface{}) (interface{}, error)
    mu        sync.RWMutex
}

// NewContext creates a new context
func NewContext() *Context {
    return &Context{
        variables: make(map[string]interface{}),
        functions: make(map[string]func([]interface{}) (interface{}, error)),
    }
}

// SetVariable sets a variable value
func (c *Context) SetVariable(name string, value interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.variables[name] = value
}

// GetVariable gets a variable value
func (c *Context) GetVariable(name string) (interface{}, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    value, exists := c.variables[name]
    return value, exists
}

// SetFunction sets a function
func (c *Context) SetFunction(name string, fn func([]interface{}) (interface{}, error)) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.functions[name] = fn
}

// GetFunction gets a function
func (c *Context) GetFunction(name string) (func([]interface{}) (interface{}, error), bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    fn, exists := c.functions[name]
    return fn, exists
}

// Expression abstract expression interface
type Expression interface {
    Interpret(context *Context) (interface{}, error)
    String() string
}

// BooleanExpression boolean expression interface
type BooleanExpression interface {
    Expression
    InterpretBoolean(context *Context) (bool, error)
}

// NumericExpression numeric expression interface
type NumericExpression interface {
    Expression
    InterpretNumeric(context *Context) (float64, error)
}
```

### 2. Terminal Expressions
```go
package interpreter

import (
    "fmt"
    "strconv"
)

// ConstantExpression represents constant values
type ConstantExpression struct {
    value interface{}
}

// NewConstantExpression creates a constant expression
func NewConstantExpression(value interface{}) *ConstantExpression {
    return &ConstantExpression{value: value}
}

// Interpret interprets the constant expression
func (e *ConstantExpression) Interpret(context *Context) (interface{}, error) {
    return e.value, nil
}

// String returns string representation
func (e *ConstantExpression) String() string {
    return fmt.Sprintf("%v", e.value)
}

// InterpretBoolean interprets as boolean
func (e *ConstantExpression) InterpretBoolean(context *Context) (bool, error) {
    if val, ok := e.value.(bool); ok {
        return val, nil
    }
    return false, fmt.Errorf("constant %v is not a boolean", e.value)
}

// InterpretNumeric interprets as numeric
func (e *ConstantExpression) InterpretNumeric(context *Context) (float64, error) {
    switch v := e.value.(type) {
    case float64:
        return v, nil
    case int:
        return float64(v), nil
    case string:
        return strconv.ParseFloat(v, 64)
    default:
        return 0, fmt.Errorf("constant %v is not numeric", e.value)
    }
}

// VariableExpression represents variable references
type VariableExpression struct {
    name string
}

// NewVariableExpression creates a variable expression
func NewVariableExpression(name string) *VariableExpression {
    return &VariableExpression{name: name}
}

// Interpret interprets the variable expression
func (e *VariableExpression) Interpret(context *Context) (interface{}, error) {
    value, exists := context.GetVariable(e.name)
    if !exists {
        return nil, fmt.Errorf("variable '%s' not found", e.name)
    }
    return value, nil
}

// String returns string representation
func (e *VariableExpression) String() string {
    return e.name
}

// InterpretBoolean interprets variable as boolean
func (e *VariableExpression) InterpretBoolean(context *Context) (bool, error) {
    value, err := e.Interpret(context)
    if err != nil {
        return false, err
    }
    
    if val, ok := value.(bool); ok {
        return val, nil
    }
    return false, fmt.Errorf("variable '%s' is not a boolean", e.name)
}

// InterpretNumeric interprets variable as numeric
func (e *VariableExpression) InterpretNumeric(context *Context) (float64, error) {
    value, err := e.Interpret(context)
    if err != nil {
        return 0, err
    }
    
    switch v := value.(type) {
    case float64:
        return v, nil
    case int:
        return float64(v), nil
    case string:
        return strconv.ParseFloat(v, 64)
    default:
        return 0, fmt.Errorf("variable '%s' is not numeric", e.name)
    }
}
```

### 3. Non-terminal Expressions
```go
package interpreter

import (
    "fmt"
    "math"
)

// BinaryOperationExpression represents binary operations
type BinaryOperationExpression struct {
    left     Expression
    right    Expression
    operator string
}

// NewBinaryOperationExpression creates a binary operation expression
func NewBinaryOperationExpression(left, right Expression, operator string) *BinaryOperationExpression {
    return &BinaryOperationExpression{
        left:     left,
        right:    right,
        operator: operator,
    }
}

// Interpret interprets the binary operation
func (e *BinaryOperationExpression) Interpret(context *Context) (interface{}, error) {
    switch e.operator {
    case "+", "-", "*", "/", "%", "**":
        return e.interpretNumericOperation(context)
    case "==", "!=", "<", ">", "<=", ">=":
        return e.interpretComparisonOperation(context)
    case "&&", "||":
        return e.interpretLogicalOperation(context)
    default:
        return nil, fmt.Errorf("unknown operator: %s", e.operator)
    }
}

// interpretNumericOperation interprets numeric operations
func (e *BinaryOperationExpression) interpretNumericOperation(context *Context) (interface{}, error) {
    leftNumeric, ok := e.left.(NumericExpression)
    if !ok {
        return nil, fmt.Errorf("left operand is not numeric")
    }
    
    rightNumeric, ok := e.right.(NumericExpression)
    if !ok {
        return nil, fmt.Errorf("right operand is not numeric")
    }
    
    leftVal, err := leftNumeric.InterpretNumeric(context)
    if err != nil {
        return nil, err
    }
    
    rightVal, err := rightNumeric.InterpretNumeric(context)
    if err != nil {
        return nil, err
    }
    
    switch e.operator {
    case "+":
        return leftVal + rightVal, nil
    case "-":
        return leftVal - rightVal, nil
    case "*":
        return leftVal * rightVal, nil
    case "/":
        if rightVal == 0 {
            return nil, fmt.Errorf("division by zero")
        }
        return leftVal / rightVal, nil
    case "%":
        if rightVal == 0 {
            return nil, fmt.Errorf("modulo by zero")
        }
        return math.Mod(leftVal, rightVal), nil
    case "**":
        return math.Pow(leftVal, rightVal), nil
    default:
        return nil, fmt.Errorf("unknown numeric operator: %s", e.operator)
    }
}

// interpretComparisonOperation interprets comparison operations
func (e *BinaryOperationExpression) interpretComparisonOperation(context *Context) (interface{}, error) {
    leftVal, err := e.left.Interpret(context)
    if err != nil {
        return nil, err
    }
    
    rightVal, err := e.right.Interpret(context)
    if err != nil {
        return nil, err
    }
    
    // Convert to comparable values
    leftFloat, leftIsNumeric := toFloat64(leftVal)
    rightFloat, rightIsNumeric := toFloat64(rightVal)
    
    if leftIsNumeric && rightIsNumeric {
        switch e.operator {
        case "==":
            return leftFloat == rightFloat, nil
        case "!=":
            return leftFloat != rightFloat, nil
        case "<":
            return leftFloat < rightFloat, nil
        case ">":
            return leftFloat > rightFloat, nil
        case "<=":
            return leftFloat <= rightFloat, nil
        case ">=":
            return leftFloat >= rightFloat, nil
        }
    }
    
    // String comparison
    leftStr := fmt.Sprintf("%v", leftVal)
    rightStr := fmt.Sprintf("%v", rightVal)
    
    switch e.operator {
    case "==":
        return leftStr == rightStr, nil
    case "!=":
        return leftStr != rightStr, nil
    case "<":
        return leftStr < rightStr, nil
    case ">":
        return leftStr > rightStr, nil
    case "<=":
        return leftStr <= rightStr, nil
    case ">=":
        return leftStr >= rightStr, nil
    default:
        return nil, fmt.Errorf("unknown comparison operator: %s", e.operator)
    }
}

// interpretLogicalOperation interprets logical operations
func (e *BinaryOperationExpression) interpretLogicalOperation(context *Context) (interface{}, error) {
    leftBoolean, ok := e.left.(BooleanExpression)
    if !ok {
        return nil, fmt.Errorf("left operand is not boolean")
    }
    
    rightBoolean, ok := e.right.(BooleanExpression)
    if !ok {
        return nil, fmt.Errorf("right operand is not boolean")
    }
    
    switch e.operator {
    case "&&":
        leftVal, err := leftBoolean.InterpretBoolean(context)
        if err != nil {
            return nil, err
        }
        if !leftVal {
            return false, nil // Short-circuit evaluation
        }
        return rightBoolean.InterpretBoolean(context)
        
    case "||":
        leftVal, err := leftBoolean.InterpretBoolean(context)
        if err != nil {
            return nil, err
        }
        if leftVal {
            return true, nil // Short-circuit evaluation
        }
        return rightBoolean.InterpretBoolean(context)
        
    default:
        return nil, fmt.Errorf("unknown logical operator: %s", e.operator)
    }
}

// String returns string representation
func (e *BinaryOperationExpression) String() string {
    return fmt.Sprintf("(%s %s %s)", e.left.String(), e.operator, e.right.String())
}

// InterpretBoolean interprets as boolean
func (e *BinaryOperationExpression) InterpretBoolean(context *Context) (bool, error) {
    result, err := e.Interpret(context)
    if err != nil {
        return false, err
    }
    
    if val, ok := result.(bool); ok {
        return val, nil
    }
    return false, fmt.Errorf("operation result is not boolean")
}

// InterpretNumeric interprets as numeric
func (e *BinaryOperationExpression) InterpretNumeric(context *Context) (float64, error) {
    result, err := e.Interpret(context)
    if err != nil {
        return 0, err
    }
    
    if val, ok := result.(float64); ok {
        return val, nil
    }
    return 0, fmt.Errorf("operation result is not numeric")
}

// toFloat64 converts interface{} to float64 if possible
func toFloat64(value interface{}) (float64, bool) {
    switch v := value.(type) {
    case float64:
        return v, true
    case int:
        return float64(v), true
    case int64:
        return float64(v), true
    case int32:
        return float64(v), true
    default:
        return 0, false
    }
}
```

### 4. Function Call Expression
```go
package interpreter

import (
    "fmt"
)

// FunctionCallExpression represents function calls
type FunctionCallExpression struct {
    functionName string
    arguments    []Expression
}

// NewFunctionCallExpression creates a function call expression
func NewFunctionCallExpression(functionName string, arguments []Expression) *FunctionCallExpression {
    return &FunctionCallExpression{
        functionName: functionName,
        arguments:    arguments,
    }
}

// Interpret interprets the function call
func (e *FunctionCallExpression) Interpret(context *Context) (interface{}, error) {
    fn, exists := context.GetFunction(e.functionName)
    if !exists {
        return nil, fmt.Errorf("function '%s' not found", e.functionName)
    }
    
    // Evaluate arguments
    args := make([]interface{}, len(e.arguments))
    for i, arg := range e.arguments {
        value, err := arg.Interpret(context)
        if err != nil {
            return nil, fmt.Errorf("error evaluating argument %d: %v", i, err)
        }
        args[i] = value
    }
    
    // Call function
    return fn(args)
}

// String returns string representation
func (e *FunctionCallExpression) String() string {
    argStrs := make([]string, len(e.arguments))
    for i, arg := range e.arguments {
        argStrs[i] = arg.String()
    }
    return fmt.Sprintf("%s(%s)", e.functionName, fmt.Sprintf("%v", argStrs))
}

// InterpretBoolean interprets function result as boolean
func (e *FunctionCallExpression) InterpretBoolean(context *Context) (bool, error) {
    result, err := e.Interpret(context)
    if err != nil {
        return false, err
    }
    
    if val, ok := result.(bool); ok {
        return val, nil
    }
    return false, fmt.Errorf("function '%s' does not return boolean", e.functionName)
}

// InterpretNumeric interprets function result as numeric
func (e *FunctionCallExpression) InterpretNumeric(context *Context) (float64, error) {
    result, err := e.Interpret(context)
    if err != nil {
        return 0, err
    }
    
    if val, ok := toFloat64(result); ok {
        return val, true
    }
    return 0, fmt.Errorf("function '%s' does not return numeric value", e.functionName)
}
```

### 5. Conditional Expression
```go
package interpreter

import "fmt"

// ConditionalExpression represents if-then-else expressions
type ConditionalExpression struct {
    condition Expression
    thenExpr  Expression
    elseExpr  Expression
}

// NewConditionalExpression creates a conditional expression
func NewConditionalExpression(condition, thenExpr, elseExpr Expression) *ConditionalExpression {
    return &ConditionalExpression{
        condition: condition,
        thenExpr:  thenExpr,
        elseExpr:  elseExpr,
    }
}

// Interpret interprets the conditional expression
func (e *ConditionalExpression) Interpret(context *Context) (interface{}, error) {
    conditionBoolean, ok := e.condition.(BooleanExpression)
    if !ok {
        return nil, fmt.Errorf("condition is not a boolean expression")
    }
    
    conditionResult, err := conditionBoolean.InterpretBoolean(context)
    if err != nil {
        return nil, err
    }
    
    if conditionResult {
        return e.thenExpr.Interpret(context)
    } else {
        return e.elseExpr.Interpret(context)
    }
}

// String returns string representation
func (e *ConditionalExpression) String() string {
    return fmt.Sprintf("if (%s) then (%s) else (%s)", 
        e.condition.String(), e.thenExpr.String(), e.elseExpr.String())
}

// InterpretBoolean interprets as boolean
func (e *ConditionalExpression) InterpretBoolean(context *Context) (bool, error) {
    result, err := e.Interpret(context)
    if err != nil {
        return false, err
    }
    
    if val, ok := result.(bool); ok {
        return val, nil
    }
    return false, fmt.Errorf("conditional expression does not result in boolean")
}

// InterpretNumeric interprets as numeric
func (e *ConditionalExpression) InterpretNumeric(context *Context) (float64, error) {
    result, err := e.Interpret(context)
    if err != nil {
        return 0, err
    }
    
    if val, ok := toFloat64(result); ok {
        return val, true
    }
    return 0, fmt.Errorf("conditional expression does not result in numeric value")
}
```

## Expression Builder and Parser

### 1. Expression Builder
```go
package interpreter

import (
    "fmt"
    "math"
)

// ExpressionBuilder builds expressions using fluent interface
type ExpressionBuilder struct {
    context *Context
}

// NewExpressionBuilder creates a new expression builder
func NewExpressionBuilder() *ExpressionBuilder {
    context := NewContext()
    
    // Register built-in functions
    context.SetFunction("abs", func(args []interface{}) (interface{}, error) {
        if len(args) != 1 {
            return nil, fmt.Errorf("abs expects 1 argument, got %d", len(args))
        }
        
        if val, ok := toFloat64(args[0]); ok {
            return math.Abs(val), nil
        }
        return nil, fmt.Errorf("abs expects numeric argument")
    })
    
    context.SetFunction("max", func(args []interface{}) (interface{}, error) {
        if len(args) < 2 {
            return nil, fmt.Errorf("max expects at least 2 arguments, got %d", len(args))
        }
        
        max := math.Inf(-1)
        for i, arg := range args {
            if val, ok := toFloat64(arg); ok {
                if val > max {
                    max = val
                }
            } else {
                return nil, fmt.Errorf("argument %d is not numeric", i)
            }
        }
        return max, nil
    })
    
    context.SetFunction("min", func(args []interface{}) (interface{}, error) {
        if len(args) < 2 {
            return nil, fmt.Errorf("min expects at least 2 arguments, got %d", len(args))
        }
        
        min := math.Inf(1)
        for i, arg := range args {
            if val, ok := toFloat64(arg); ok {
                if val < min {
                    min = val
                }
            } else {
                return nil, fmt.Errorf("argument %d is not numeric", i)
            }
        }
        return min, nil
    })
    
    return &ExpressionBuilder{context: context}
}

// GetContext returns the context
func (b *ExpressionBuilder) GetContext() *Context {
    return b.context
}

// Constant creates a constant expression
func (b *ExpressionBuilder) Constant(value interface{}) Expression {
    return NewConstantExpression(value)
}

// Variable creates a variable expression
func (b *ExpressionBuilder) Variable(name string) Expression {
    return NewVariableExpression(name)
}

// Add creates an addition expression
func (b *ExpressionBuilder) Add(left, right Expression) Expression {
    return NewBinaryOperationExpression(left, right, "+")
}

// Subtract creates a subtraction expression
func (b *ExpressionBuilder) Subtract(left, right Expression) Expression {
    return NewBinaryOperationExpression(left, right, "-")
}

// Multiply creates a multiplication expression
func (b *ExpressionBuilder) Multiply(left, right Expression) Expression {
    return NewBinaryOperationExpression(left, right, "*")
}

// Divide creates a division expression
func (b *ExpressionBuilder) Divide(left, right Expression) Expression {
    return NewBinaryOperationExpression(left, right, "/")
}

// Equal creates an equality expression
func (b *ExpressionBuilder) Equal(left, right Expression) Expression {
    return NewBinaryOperationExpression(left, right, "==")
}

// GreaterThan creates a greater than expression
func (b *ExpressionBuilder) GreaterThan(left, right Expression) Expression {
    return NewBinaryOperationExpression(left, right, ">")
}

// And creates a logical AND expression
func (b *ExpressionBuilder) And(left, right Expression) Expression {
    return NewBinaryOperationExpression(left, right, "&&")
}

// Or creates a logical OR expression
func (b *ExpressionBuilder) Or(left, right Expression) Expression {
    return NewBinaryOperationExpression(left, right, "||")
}

// If creates a conditional expression
func (b *ExpressionBuilder) If(condition, thenExpr, elseExpr Expression) Expression {
    return NewConditionalExpression(condition, thenExpr, elseExpr)
}

// Function creates a function call expression
func (b *ExpressionBuilder) Function(name string, args ...Expression) Expression {
    return NewFunctionCallExpression(name, args)
}
```

### 2. Simple Parser
```go
package interpreter

import (
    "fmt"
    "regexp"
    "strconv"
    "strings"
)

// SimpleParser parses simple expressions
type SimpleParser struct {
    builder *ExpressionBuilder
}

// NewSimpleParser creates a new simple parser
func NewSimpleParser() *SimpleParser {
    return &SimpleParser{
        builder: NewExpressionBuilder(),
    }
}

// GetBuilder returns the expression builder
func (p *SimpleParser) GetBuilder() *ExpressionBuilder {
    return p.builder
}

// ParseExpression parses a simple expression
func (p *SimpleParser) ParseExpression(expr string) (Expression, error) {
    expr = strings.TrimSpace(expr)
    
    // Handle parentheses
    if strings.HasPrefix(expr, "(") && strings.HasSuffix(expr, ")") {
        return p.ParseExpression(expr[1 : len(expr)-1])
    }
    
    // Try to parse as number
    if val, err := strconv.ParseFloat(expr, 64); err == nil {
        return p.builder.Constant(val), nil
    }
    
    // Try to parse as boolean
    if expr == "true" {
        return p.builder.Constant(true), nil
    }
    if expr == "false" {
        return p.builder.Constant(false), nil
    }
    
    // Try to parse as string literal
    if strings.HasPrefix(expr, "\"") && strings.HasSuffix(expr, "\"") {
        return p.builder.Constant(expr[1 : len(expr)-1]), nil
    }
    
    // Try to parse binary operations
    if binExpr, err := p.parseBinaryOperation(expr); err == nil {
        return binExpr, nil
    }
    
    // Try to parse function call
    if funcExpr, err := p.parseFunctionCall(expr); err == nil {
        return funcExpr, nil
    }
    
    // Assume it's a variable
    if p.isValidIdentifier(expr) {
        return p.builder.Variable(expr), nil
    }
    
    return nil, fmt.Errorf("unable to parse expression: %s", expr)
}

// parseBinaryOperation parses binary operations
func (p *SimpleParser) parseBinaryOperation(expr string) (Expression, error) {
    operators := []string{"==", "!=", "<=", ">=", "<", ">", "&&", "||", "+", "-", "*", "/"}
    
    for _, op := range operators {
        if idx := strings.Index(expr, op); idx > 0 {
            leftExpr := strings.TrimSpace(expr[:idx])
            rightExpr := strings.TrimSpace(expr[idx+len(op):])
            
            left, err := p.ParseExpression(leftExpr)
            if err != nil {
                continue
            }
            
            right, err := p.ParseExpression(rightExpr)
            if err != nil {
                continue
            }
            
            return NewBinaryOperationExpression(left, right, op), nil
        }
    }
    
    return nil, fmt.Errorf("no binary operation found")
}

// parseFunctionCall parses function calls
func (p *SimpleParser) parseFunctionCall(expr string) (Expression, error) {
    pattern := regexp.MustCompile(`^(\w+)\((.*)\)$`)
    matches := pattern.FindStringSubmatch(expr)
    if len(matches) != 3 {
        return nil, fmt.Errorf("not a function call")
    }
    
    functionName := matches[1]
    argsStr := matches[2]
    
    var args []Expression
    if argsStr != "" {
        argStrings := strings.Split(argsStr, ",")
        for _, argStr := range argStrings {
            argExpr, err := p.ParseExpression(strings.TrimSpace(argStr))
            if err != nil {
                return nil, err
            }
            args = append(args, argExpr)
        }
    }
    
    return NewFunctionCallExpression(functionName, args), nil
}

// isValidIdentifier checks if string is valid identifier
func (p *SimpleParser) isValidIdentifier(s string) bool {
    pattern := regexp.MustCompile(`^[a-zA-Z_][a-zA-Z0-9_]*$`)
    return pattern.MatchString(s)
}
```

## Usage Examples

### 1. Basic Expression Evaluation
```go
package main

import (
    "fmt"
    "log"
)

func main() {
    // Create expression builder
    builder := NewExpressionBuilder()
    context := builder.GetContext()
    
    // Set variables
    context.SetVariable("x", 10.0)
    context.SetVariable("y", 5.0)
    context.SetVariable("name", "Alice")
    
    // Build expressions
    expr1 := builder.Add(
        builder.Variable("x"),
        builder.Multiply(builder.Variable("y"), builder.Constant(2.0)),
    )
    
    expr2 := builder.GreaterThan(
        builder.Variable("x"),
        builder.Constant(5.0),
    )
    
    expr3 := builder.If(
        expr2,
        builder.Constant("x is greater than 5"),
        builder.Constant("x is not greater than 5"),
    )
    
    // Evaluate expressions
    result1, err := expr1.Interpret(context)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("x + (y * 2) = %v\n", result1) // Output: 20
    
    result2, err := expr2.Interpret(context)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("x > 5 = %v\n", result2) // Output: true
    
    result3, err := expr3.Interpret(context)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Conditional result: %v\n", result3) // Output: x is greater than 5
}
```

### 2. Business Rule Engine
```go
package main

import (
    "fmt"
    "log"
)

// BusinessRule represents a business rule
type BusinessRule struct {
    name       string
    condition  Expression
    action     Expression
    description string
}

// NewBusinessRule creates a new business rule
func NewBusinessRule(name string, condition, action Expression, description string) *BusinessRule {
    return &BusinessRule{
        name:        name,
        condition:   condition,
        action:      action,
        description: description,
    }
}

// Evaluate evaluates the business rule
func (r *BusinessRule) Evaluate(context *Context) (interface{}, error) {
    conditionBoolean, ok := r.condition.(BooleanExpression)
    if !ok {
        return nil, fmt.Errorf("rule condition must be boolean")
    }
    
    conditionResult, err := conditionBoolean.InterpretBoolean(context)
    if err != nil {
        return nil, err
    }
    
    if conditionResult {
        return r.action.Interpret(context)
    }
    
    return nil, nil // Rule did not fire
}

// String returns string representation
func (r *BusinessRule) String() string {
    return fmt.Sprintf("Rule '%s': IF %s THEN %s", r.name, r.condition.String(), r.action.String())
}

// RuleEngine manages and executes business rules
type RuleEngine struct {
    rules   []*BusinessRule
    context *Context
}

// NewRuleEngine creates a new rule engine
func NewRuleEngine() *RuleEngine {
    return &RuleEngine{
        rules:   make([]*BusinessRule, 0),
        context: NewContext(),
    }
}

// AddRule adds a rule to the engine
func (e *RuleEngine) AddRule(rule *BusinessRule) {
    e.rules = append(e.rules, rule)
}

// GetContext returns the context
func (e *RuleEngine) GetContext() *Context {
    return e.context
}

// ExecuteRules executes all rules
func (e *RuleEngine) ExecuteRules() []interface{} {
    results := make([]interface{}, 0)
    
    for _, rule := range e.rules {
        result, err := rule.Evaluate(e.context)
        if err != nil {
            fmt.Printf("Error executing rule '%s': %v\n", rule.name, err)
            continue
        }
        
        if result != nil {
            fmt.Printf("Rule '%s' fired: %v\n", rule.name, result)
            results = append(results, result)
        }
    }
    
    return results
}

func main() {
    // Create rule engine
    engine := NewRuleEngine()
    builder := NewExpressionBuilder()
    context := engine.GetContext()
    
    // Set customer data
    context.SetVariable("age", 25)
    context.SetVariable("income", 50000.0)
    context.SetVariable("creditScore", 750)
    context.SetVariable("hasJob", true)
    
    // Define business rules
    
    // Rule 1: High-value customer
    rule1 := NewBusinessRule(
        "HighValueCustomer",
        builder.And(
            builder.GreaterThan(builder.Variable("income"), builder.Constant(80000.0)),
            builder.GreaterThan(builder.Variable("creditScore"), builder.Constant(700)),
        ),
        builder.Constant("PREMIUM_CUSTOMER"),
        "Customer qualifies for premium services",
    )
    
    // Rule 2: Loan eligibility
    rule2 := NewBusinessRule(
        "LoanEligibility",
        builder.And(
            builder.And(
                builder.GreaterThan(builder.Variable("age"), builder.Constant(18)),
                builder.Variable("hasJob"),
            ),
            builder.GreaterThan(builder.Variable("creditScore"), builder.Constant(600)),
        ),
        builder.Constant("LOAN_APPROVED"),
        "Customer is eligible for loan",
    )
    
    // Rule 3: Discount eligibility
    rule3 := NewBusinessRule(
        "DiscountEligibility",
        builder.GreaterThan(builder.Variable("age"), builder.Constant(65)),
        builder.Constant("SENIOR_DISCOUNT"),
        "Senior citizen discount",
    )
    
    // Add rules to engine
    engine.AddRule(rule1)
    engine.AddRule(rule2)
    engine.AddRule(rule3)
    
    // Execute rules
    fmt.Println("Executing business rules...")
    results := engine.ExecuteRules()
    
    fmt.Printf("\nFired rules count: %d\n", len(results))
}
```

### 3. Configuration DSL
```go
package main

import (
    "fmt"
    "log"
)

// ConfigurationDSL processes configuration expressions
type ConfigurationDSL struct {
    parser  *SimpleParser
    context *Context
}

// NewConfigurationDSL creates a new configuration DSL
func NewConfigurationDSL() *ConfigurationDSL {
    parser := NewSimpleParser()
    context := parser.GetBuilder().GetContext()
    
    // Set environment variables
    context.SetVariable("ENV", "production")
    context.SetVariable("DEBUG", false)
    context.SetVariable("MAX_CONNECTIONS", 100)
    context.SetVariable("TIMEOUT", 30)
    
    // Add configuration functions
    context.SetFunction("env", func(args []interface{}) (interface{}, error) {
        if len(args) != 2 {
            return nil, fmt.Errorf("env expects 2 arguments: variable name and default value")
        }
        
        varName, ok := args[0].(string)
        if !ok {
            return nil, fmt.Errorf("first argument must be string")
        }
        
        value, exists := context.GetVariable(varName)
        if !exists {
            return args[1], nil // Return default value
        }
        return value, nil
    })
    
    return &ConfigurationDSL{
        parser:  parser,
        context: context,
    }
}

// ProcessConfiguration processes configuration rules
func (dsl *ConfigurationDSL) ProcessConfiguration(configRules []string) map[string]interface{} {
    config := make(map[string]interface{})
    
    for _, rule := range configRules {
        parts := strings.SplitN(rule, "=", 2)
        if len(parts) != 2 {
            fmt.Printf("Invalid configuration rule: %s\n", rule)
            continue
        }
        
        key := strings.TrimSpace(parts[0])
        exprStr := strings.TrimSpace(parts[1])
        
        expr, err := dsl.parser.ParseExpression(exprStr)
        if err != nil {
            fmt.Printf("Error parsing expression '%s': %v\n", exprStr, err)
            continue
        }
        
        value, err := expr.Interpret(dsl.context)
        if err != nil {
            fmt.Printf("Error evaluating expression '%s': %v\n", exprStr, err)
            continue
        }
        
        config[key] = value
        fmt.Printf("Config: %s = %v\n", key, value)
    }
    
    return config
}

func main() {
    dsl := NewConfigurationDSL()
    
    // Define configuration rules
    configRules := []string{
        "database.maxConnections = MAX_CONNECTIONS * 2",
        "database.timeout = TIMEOUT + 10",
        "logging.enabled = DEBUG || ENV == \"development\"",
        "cache.enabled = ENV == \"production\"",
        "api.rateLimit = env(\"API_RATE_LIMIT\", 1000)",
    }
    
    // Process configuration
    config := dsl.ProcessConfiguration(configRules)
    
    fmt.Printf("\nFinal configuration: %+v\n", config)
}
```

## Benefits

### Flexibility and Extensibility
- Easy to add new grammar rules and expressions
- Dynamic expression evaluation at runtime
- Support for domain-specific languages

### Maintainability
- Clear separation between grammar and implementation
- Easy to modify and extend language features
- Well-structured code organization

### Reusability
- Expression components can be reused
- Common patterns can be abstracted
- Modular design

## Drawbacks

### Performance Overhead
- Runtime interpretation can be slower than compiled code
- Tree traversal overhead
- Dynamic dispatch costs

### Complexity
- Can become complex for large grammars
- Debugging interpreted code can be difficult
- Memory usage for expression trees

### Limited Optimization
- Less optimization compared to compiled languages
- No compile-time error checking
- Runtime error discovery

## When to Use

### Suitable Scenarios
- **Business rule engines**: Complex, changing business logic
- **Configuration systems**: Dynamic configuration evaluation
- **Query languages**: Custom query DSLs
- **Formula evaluation**: Mathematical or logical expressions

### Not Suitable Scenarios
- **Performance-critical code**: Where speed is paramount
- **Simple expressions**: Basic calculations don't need interpretation
- **Static logic**: Logic that doesn't change

## Related Patterns
- **Visitor Pattern**: For operations on expression trees
- **Composite Pattern**: Tree structure representation
- **Strategy Pattern**: Different interpretation strategies
- **Builder Pattern**: Expression construction

## References
- [Gang of Four - Design Patterns](https://en.wikipedia.org/wiki/Design_Patterns)
- [Martin Fowler - Domain-Specific Languages](https://martinfowler.com/books/dsl.html)
- [Robert Nystrom - Crafting Interpreters](https://craftinginterpreters.com/)
