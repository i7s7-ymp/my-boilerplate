# Interpreter Pattern（インタープリターパターン）

## 概要
インタープリターパターンは、特定の言語やドメイン固有言語（DSL）の文法を定義し、その言語で書かれた文や式を解釈・実行するためのアーキテクチャパターンです。言語の文法規則を表現するクラス階層を定義し、構文木を構築して実行時に解釈を行います。

## 基本構造

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

## 核心コンポーネント

### 1. 抽象表現（Abstract Expression）
```go
package main

package interpreter

    // fmt
    // sync
)

// Context インタープリターのコンテキスト
type Context struct {
    variables map[string]interface{}
    functions map[string]func([]interface{}) (interface{}, error)
    mu        sync.RWMutex
}

// NewContext コンテキスト作成
func NewContext() *Context {
    return &Context{
        variables: make(map[string]interface{}),
        functions: make(map[string]func([]interface{}) (interface{}, error)),
    }
}

// SetVariable 変数設定
func (c *Context) SetVariable(name string, value interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.variables[name] = value
}

// GetVariable 変数取得
func (c *Context) GetVariable(name string) (interface{}, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    value, exists := c.variables[name]
    return value, exists
}

// SetFunction 関数設定
func (c *Context) SetFunction(name string, fn func([]interface{}) (interface{}, error)) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.functions[name] = fn
}

// GetFunction 関数取得
func (c *Context) GetFunction(name string) (func([]interface{}) (interface{}, error), bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    fn, exists := c.functions[name]
    return fn, exists
}

// Expression 抽象表現インターフェース
type Expression interface {
    Interpret(context *Context) (interface{}, error)
    String() string
}

// BooleanExpression ブール式インターフェース
type BooleanExpression interface {
    Expression
    InterpretBoolean(context *Context) (bool, error)
}

// NumericExpression 数値式インターフェース
type NumericExpression interface {
    Expression
    InterpretNumeric(context *Context) (float64, error)
        // 数値の解釈実行
        // TODO: implement
```

### 2. 終端表現（Terminal Expression）
```go
package main

type Variable struct {
    // 変数表現
    
    func (receiver) __init__(name string) {
        receiver.name = name
        
    func (receiver) interpret(context: Context) {
        // 変数値の取得
        value = context.get_variable(receiver.name)
        if value is nil:
            raise ValueError(f// Undefined variable: {receiver.name})
        return value
        
    func (receiver) __str__() {
        return receiver.name

type Constant struct {
    // 定数表現
    
    func (receiver) __init__(value float64) {
        receiver.value = value
        
    func (receiver) interpret(context: Context) {
        // 定数値の返却
        return receiver.value
        
    func (receiver) __str__() {
        return str(receiver.value)

type StringLiteral struct {
    // 文字列リテラル表現
    
    func (receiver) __init__(value string) {
        receiver.value = value
        
    func (receiver) interpret(context: Context) {
        // 文字列値の返却
        return receiver.value
        
    func (receiver) __str__() {
        return f'// {receiver.value}'

type BooleanLiteral struct {
    // ブールリテラル表現
    
    func (receiver) __init__(value bool) {
        receiver.value = value
        
    func (receiver) interpret(context: Context) {
        // ブール値の返却
        return receiver.value
        
    func (receiver) __str__() {
        return // true if receiver.value else // false
```

### 3. 非終端表現（Nonterminal Expression）
```go
package main

type BinaryOperatorExpression struct {
    // 二項演算子表現の基底クラス
    
    func (receiver) __init__(left: NumericExpression, right: NumericExpression) {
        receiver.left = left
        receiver.right = right
        
        func (receiver) operate(left_val float64, right_val float64) {
        // 演算実行
        // TODO: implement
        
    func (receiver) interpret(context: Context) {
        // 二項演算の解釈実行
        left_val = receiver.left.interpret(context)
        right_val = receiver.right.interpret(context)
        return receiver.operate(left_val, right_val)

type AddExpression struct {
    // 加算表現
    
    func (receiver) operate(left_val float64, right_val float64) {
        return left_val + right_val
        
    func (receiver) __str__() {
        return f// ({receiver.left} + {receiver.right})

type SubtractExpression struct {
    // 減算表現
    
    func (receiver) operate(left_val float64, right_val float64) {
        return left_val - right_val
        
    func (receiver) __str__() {
        return f// ({receiver.left} - {receiver.right})

type MultiplyExpression struct {
    // 乗算表現
    
    func (receiver) operate(left_val float64, right_val float64) {
        return left_val * right_val
        
    func (receiver) __str__() {
        return f// ({receiver.left} * {receiver.right})

type DivideExpression struct {
    // 除算表現
    
    func (receiver) operate(left_val float64, right_val float64) {
        if right_val == 0:
            raise ValueError(// Division by zero)
        return left_val / right_val
        
    func (receiver) __str__() {
        return f// ({receiver.left} / {receiver.right})

type PowerExpression struct {
    // 累乗表現
    
    func (receiver) operate(left_val float64, right_val float64) {
        return left_val ** right_val
        
    func (receiver) __str__() {
        return f// ({receiver.left} ^ {receiver.right})

# ブール演算
type BooleanBinaryExpression struct {
    // ブール二項演算の基底クラス
    
    func (receiver) __init__(left: BooleanExpression, right: BooleanExpression) {
        receiver.left = left
        receiver.right = right

type AndExpression struct {
    // 論理積表現
    
    func (receiver) interpret(context: Context) {
        return receiver.left.interpret(context) and receiver.right.interpret(context)
        
    func (receiver) __str__() {
        return f// ({receiver.left} AND {receiver.right})

type OrExpression struct {
    // 論理和表現
    
    func (receiver) interpret(context: Context) {
        return receiver.left.interpret(context) or receiver.right.interpret(context)
        
    func (receiver) __str__() {
        return f// ({receiver.left} OR {receiver.right})

type NotExpression struct {
    // 論理否定表現
    
    func (receiver) __init__(expression: BooleanExpression) {
        receiver.expression = expression
        
    func (receiver) interpret(context: Context) {
        return not receiver.expression.interpret(context)
        
    func (receiver) __str__() {
        return f// (NOT {receiver.expression})

# 比較演算
type ComparisonExpression struct {
    // 比較演算の基底クラス
    
    func (receiver) __init__(left: NumericExpression, right: NumericExpression) {
        receiver.left = left
        receiver.right = right
        
        func (receiver) compare(left_val float64, right_val float64) {
        // 比較実行
        // TODO: implement
        
    func (receiver) interpret(context: Context) {
        left_val = receiver.left.interpret(context)
        right_val = receiver.right.interpret(context)
        return receiver.compare(left_val, right_val)

type EqualsExpression struct {
    // 等価比較表現
    
    func (receiver) compare(left_val float64, right_val float64) {
        return abs(left_val - right_val) < 1e-10  # 浮動小数点の比較
        
    func (receiver) __str__() {
        return f// ({receiver.left} == {receiver.right})

type GreaterThanExpression struct {
    // 大なり比較表現
    
    func (receiver) compare(left_val float64, right_val float64) {
        return left_val > right_val
        
    func (receiver) __str__() {
        return f// ({receiver.left} > {receiver.right})

type LessThanExpression struct {
    // 小なり比較表現
    
    func (receiver) compare(left_val float64, right_val float64) {
        return left_val < right_val
        
    func (receiver) __str__() {
        return f// ({receiver.left} < {receiver.right})
```

### 4. 複合表現と関数呼び出し
```go
package main

type FunctionCallExpression struct {
    // 関数呼び出し表現
    
    func (receiver) __init__(function_name string, arguments []Expression) {
        receiver.function_name = function_name
        receiver.arguments = arguments
        
    func (receiver) interpret(context: Context) {
        // 関数呼び出しの解釈実行
        func = context.get_function(receiver.function_name)
        if func is nil:
            raise ValueError(f// Undefined function: {receiver.function_name})
            
        # 引数を評価
        arg_values = [arg.interpret(context) for arg in receiver.arguments]
        
        try:
            return func(*arg_values)
        except Exception as e:
            raise ValueError(f// Error calling function {receiver.function_name}: {str(e)})
            
    func (receiver) __str__() {
        args_str = // , .join(str(arg) for arg in receiver.arguments)
        return f// {receiver.function_name}({args_str})

type ConditionalExpression struct {
    // 条件表現（三項演算子）
    
    def __init__(self, condition: BooleanExpression, 
                 true_expr: Expression, false_expr: Expression):
        receiver.condition = condition
        receiver.true_expr = true_expr
        receiver.false_expr = false_expr
        
    func (receiver) interpret(context: Context) {
        // 条件分岐の解釈実行
        if receiver.condition.interpret(context):
            return receiver.true_expr.interpret(context)
        else:
            return receiver.false_expr.interpret(context)
            
    func (receiver) __str__() {
        return f// ({receiver.condition} ? {receiver.true_expr} : {receiver.false_expr})

type SequenceExpression struct {
    // シーケンス表現（複数の式を順次実行）
    
    func (receiver) __init__(expressions []Expression) {
        receiver.expressions = expressions
        
    func (receiver) interpret(context: Context) {
        // シーケンス実行
        result = nil
        for expr in receiver.expressions:
            result = expr.interpret(context)
        return result
        
    func (receiver) __str__() {
        return // ; .join(str(expr) for expr in receiver.expressions)

type AssignmentExpression struct {
    // 代入表現
    
    func (receiver) __init__(variable_name string, value_expr: Expression) {
        receiver.variable_name = variable_name
        receiver.value_expr = value_expr
        
    func (receiver) interpret(context: Context) {
        // 代入の解釈実行
        value = receiver.value_expr.interpret(context)
        context.set_variable(receiver.variable_name, value)
        return value
        
    func (receiver) __str__() {
        return f// {receiver.variable_name} = {receiver.value_expr}
```

## パーサー（構文解析器）

### 1. トークナイザー（字句解析）
```go
package main


type TokenType struct {
    // トークンタイプ
    NUMBER = // NUMBER
    IDENTIFIER = // IDENTIFIER
    STRING = // STRING
    PLUS = // PLUS
    MINUS = // MINUS
    MULTIPLY = // MULTIPLY
    DIVIDE = // DIVIDE
    POWER = // POWER
    LPAREN = // LPAREN
    RPAREN = // RPAREN
    ASSIGN = // ASSIGN
    EQUALS = // EQUALS
    GREATER = // GREATER
    LESS = // LESS
    AND = // AND
    OR = // OR
    NOT = // NOT
    IF = // IF
    THEN = // THEN
    ELSE = // ELSE
    COMMA = // COMMA
    SEMICOLON = // SEMICOLON
    EOF = // EOF
    WHITESPACE = // WHITESPACE

type Token struct {
    // トークン
    type: TokenType
    value string
    position int

type Tokenizer struct {
    // 字句解析器
    
    TOKEN_PATTERNS = [
        (TokenType.NUMBER, r'\d+(\.\d+)?'),
        (TokenType.STRING, r'// ([^\\\\]|\\\\.)*"'),
        (TokenType.IDENTIFIER, r'[a-zA-Z_][a-zA-Z0-9_]*'),
        (TokenType.ASSIGN, r'='),
        (TokenType.EQUALS, r'=='),
        (TokenType.GREATER, r'>'),
        (TokenType.LESS, r'<'),
        (TokenType.PLUS, r'\+'),
        (TokenType.MINUS, r'-'),
        (TokenType.MULTIPLY, r'\*'),
        (TokenType.DIVIDE, r'/'),
        (TokenType.POWER, r'\^'),
        (TokenType.LPAREN, r'\('),
        (TokenType.RPAREN, r'\)'),
        (TokenType.COMMA, r','),
        (TokenType.SEMICOLON, r';'),
        (TokenType.WHITESPACE, r'\s+'),
    ]
    
    KEYWORDS = {
        'AND': TokenType.AND,
        'OR': TokenType.OR,
        'NOT': TokenType.NOT,
        'IF': TokenType.IF,
        'THEN': TokenType.THEN,
        'ELSE': TokenType.ELSE,
    }
    
    func (receiver) __init__(text string) {
        receiver.text = text
        receiver.position = 0
        receiver.tokens []Token = []
        
    func (receiver) tokenize() {
        // 字句解析実行
        while receiver.position < len(receiver.text):
            match_found = false
            
            for token_type, pattern in receiver.TOKEN_PATTERNS:
                regex = re.compile(pattern)
                match = regex.match(receiver.text, receiver.position)
                
                if match:
                    value = match.group(0)
                    
                    # 空白は無視
                    if token_type != TokenType.WHITESPACE:
                        # キーワードチェック
                        if token_type == TokenType.IDENTIFIER:
                            token_type = receiver.KEYWORDS.get(value.upper(), TokenType.IDENTIFIER)
                            
                        receiver.tokens.append(Token(token_type, value, receiver.position))
                        
                    receiver.position = match.end()
                    match_found = true
                    break
                    
            if not match_found:
                raise ValueError(f// Invalid character at position {receiver.position}: {receiver.text[receiver.position]})
                
        receiver.tokens.append(Token(TokenType.EOF, "", receiver.position))
        return receiver.tokens
```

### 2. 再帰下降パーサー
```go
package main

type Parser struct {
    // 再帰下降パーサー
    
    func (receiver) __init__(tokens []Token) {
        receiver.tokens = tokens
        receiver.current = 0
        
    func (receiver) parse() {
        // 構文解析実行
        expr = receiver.parse_sequence()
        if not receiver.is_at_end():
            raise ValueError(f// Unexpected token: {receiver.peek().value})
        return expr
        
    func (receiver) parse_sequence() {
        // シーケンス解析
        expressions = []
        expressions.append(receiver.parse_assignment())
        
        while receiver.match(TokenType.SEMICOLON):
            expressions.append(receiver.parse_assignment())
            
        if len(expressions) == 1:
            return expressions[0]
        return SequenceExpression(expressions)
        
    func (receiver) parse_assignment() {
        // 代入解析
        expr = receiver.parse_or()
        
        if receiver.match(TokenType.ASSIGN):
            if not isinstance(expr, Variable):
                raise ValueError(// Invalid assignment target)
            value = receiver.parse_assignment()
            return AssignmentExpression(expr.name, value)
            
        return expr
        
    func (receiver) parse_or() {
        // 論理和解析
        expr = receiver.parse_and()
        
        while receiver.match(TokenType.OR):
            right = receiver.parse_and()
            expr = OrExpression(expr, right)
            
        return expr
        
    func (receiver) parse_and() {
        // 論理積解析
        expr = receiver.parse_equality()
        
        while receiver.match(TokenType.AND):
            right = receiver.parse_equality()
            expr = AndExpression(expr, right)
            
        return expr
        
    func (receiver) parse_equality() {
        // 等価比較解析
        expr = receiver.parse_comparison()
        
        while receiver.match(TokenType.EQUALS):
            right = receiver.parse_comparison()
            expr = EqualsExpression(expr, right)
            
        return expr
        
    func (receiver) parse_comparison() {
        // 比較解析
        expr = receiver.parse_addition()
        
        while true:
            if receiver.match(TokenType.GREATER):
                right = receiver.parse_addition()
                expr = GreaterThanExpression(expr, right)
            elif receiver.match(TokenType.LESS):
                right = receiver.parse_addition()
                expr = LessThanExpression(expr, right)
            else:
                break
                
        return expr
        
    func (receiver) parse_addition() {
        // 加減算解析
        expr = receiver.parse_multiplication()
        
        while true:
            if receiver.match(TokenType.PLUS):
                right = receiver.parse_multiplication()
                expr = AddExpression(expr, right)
            elif receiver.match(TokenType.MINUS):
                right = receiver.parse_multiplication()
                expr = SubtractExpression(expr, right)
            else:
                break
                
        return expr
        
    func (receiver) parse_multiplication() {
        // 乗除算解析
        expr = receiver.parse_power()
        
        while true:
            if receiver.match(TokenType.MULTIPLY):
                right = receiver.parse_power()
                expr = MultiplyExpression(expr, right)
            elif receiver.match(TokenType.DIVIDE):
                right = receiver.parse_power()
                expr = DivideExpression(expr, right)
            else:
                break
                
        return expr
        
    func (receiver) parse_power() {
        // 累乗解析
        expr = receiver.parse_unary()
        
        if receiver.match(TokenType.POWER):
            right = receiver.parse_power()  # 右結合
            expr = PowerExpression(expr, right)
            
        return expr
        
    func (receiver) parse_unary() {
        // 単項演算解析
        if receiver.match(TokenType.NOT):
            expr = receiver.parse_unary()
            return NotExpression(expr)
            
        if receiver.match(TokenType.MINUS):
            expr = receiver.parse_unary()
            return MultiplyExpression(Constant(-1), expr)
            
        return receiver.parse_primary()
        
    func (receiver) parse_primary() {
        // 基本要素解析
        if receiver.match(TokenType.NUMBER):
            return Constant(float(receiver.previous().value))
            
        if receiver.match(TokenType.STRING):
            # 引用符を除去
            value = receiver.previous().value[1:-1]
            return StringLiteral(value)
            
        if receiver.match(TokenType.IDENTIFIER):
            name = receiver.previous().value
            
            # 関数呼び出しチェック
            if receiver.match(TokenType.LPAREN):
                arguments = []
                
                if not receiver.check(TokenType.RPAREN):
                    arguments.append(receiver.parse_assignment())
                    while receiver.match(TokenType.COMMA):
                        arguments.append(receiver.parse_assignment())
                        
                receiver.consume(TokenType.RPAREN, // Expected ')' after function arguments)
                return FunctionCallExpression(name, arguments)
                
            return Variable(name)
            
        if receiver.match(TokenType.LPAREN):
            expr = receiver.parse_assignment()
            receiver.consume(TokenType.RPAREN, // Expected ')' after expression)
            return expr
            
        raise ValueError(f// Unexpected token: {receiver.peek().value})
        
    func (receiver) match(*types: TokenType) {
        // トークンタイプマッチング
        for token_type in types:
            if receiver.check(token_type):
                receiver.advance()
                return true
        return false
        
    func (receiver) check(token_type: TokenType) {
        // 現在のトークンタイプチェック
        if receiver.is_at_end():
            return false
        return receiver.peek().type == token_type
        
    func (receiver) advance() {
        // 次のトークンに進む
        if not receiver.is_at_end():
            receiver.current += 1
        return receiver.previous()
        
    func (receiver) is_at_end() {
        // 終端チェック
        return receiver.peek().type == TokenType.EOF
        
    func (receiver) peek() {
        // 現在のトークン取得
        return receiver.tokens[receiver.current]
        
    func (receiver) previous() {
        // 前のトークン取得
        return receiver.tokens[receiver.current - 1]
        
    func (receiver) consume(token_type: TokenType, message string) {
        // 期待されるトークンの消費
        if receiver.check(token_type):
            return receiver.advance()
        raise ValueError(message)
```

## 高度な機能実装

### 1. ドメイン固有言語（DSL）
```go
package main

type RuleEngineInterpreter struct {
    // ルールエンジンのインタープリター
    
    func (receiver) __init__() {
        receiver.context = Context()
        receiver.setup_builtin_functions()
        
    func (receiver) setup_builtin_functions() {
        // 組み込み関数の設定
        receiver.context.set_function(// max, max)
        receiver.context.set_function(// min, min)
        receiver.context.set_function(// abs, abs)
        receiver.context.set_function(// round, round)
        receiver.context.set_function(// len, len)
        
        # カスタム関数
        receiver.context.set_function(// contains, lambda text, substring: substring in text)
        receiver.context.set_function(// startswith, lambda text, prefix: text.startswith(prefix))
        receiver.context.set_function(// endswith, lambda text, suffix: text.endswith(suffix))
        
    func (receiver) evaluate_rule(rule_text string, data map[str]Any) {
        // ルールの評価
        # データをコンテキストに設定
        for key, value in data.items():
            receiver.context.set_variable(key, value)
            
        # パースして実行
        tokenizer = Tokenizer(rule_text)
        tokens = tokenizer.tokenize()
        parser = Parser(tokens)
        expression = parser.parse()
        
        result = expression.interpret(receiver.context)
        return bool(result)

# 使用例
rule_engine = RuleEngineInterpreter()

# 顧客データ
customer_data = {
    // age: 25,
    // income: 50000,
    // credit_score: 750,
    // employment_years: 3,
    // name: // John Doe
}

# ルール定義
rules = [
    // age >= 18 AND age <= 65,                                    # 年齢制限
    // income > 30000,                                             # 収入制限
    // credit_score >= 700,                                        # 信用スコア
    // employment_years >= 2,                                      # 勤続年数
    // contains(name, \John\// ),                                   # 名前チェック
]

# ルール評価
for i, rule in enumerate(rules):
    result = rule_engine.evaluate_rule(rule, customer_data)
    print(f// Rule {i+1}: {rule} -> {result})
```

### 2. 計算式エンジン
```go
package main

type CalculationEngine struct {
    // 計算式エンジン
    
    func (receiver) __init__() {
        receiver.context = Context()
        receiver.setup_math_functions()
        
    func (receiver) setup_math_functions() {
        // 数学関数の設定
                
        receiver.context.set_function(// sin, math.sin)
        receiver.context.set_function(// cos, math.cos)
        receiver.context.set_function(// tan, math.tan)
        receiver.context.set_function(// log, math.log)
        receiver.context.set_function(// log10, math.log10)
        receiver.context.set_function(// sqrt, math.sqrt)
        receiver.context.set_function(// exp, math.exp)
        
        # 定数
        receiver.context.set_variable(// PI, math.pi)
        receiver.context.set_variable(// E, math.e)
        
    func (receiver) calculate(formula string, variables map[str]float = nil) {
        // 数式の計算
        if variables:
            for name, value in variables.items():
                receiver.context.set_variable(name, value)
                
        tokenizer = Tokenizer(formula)
        tokens = tokenizer.tokenize()
        parser = Parser(tokens)
        expression = parser.parse()
        
        result = expression.interpret(receiver.context)
        return float(result)

# 使用例
calc_engine = CalculationEngine()

# 基本計算
result1 = calc_engine.calculate(// 2 + 3 * 4)
print(f// 2 + 3 * 4 = {result1})

# 変数を使った計算
variables = {// x: 10, // y: 5}
result2 = calc_engine.calculate(// x^2 + y^2, variables)
print(f// x^2 + y^2 = {result2})

# 関数を使った計算
result3 = calc_engine.calculate(// sin(PI/2) + cos(0))
print(f// sin(π/2) + cos(0) = {result3})
```

### 3. クエリ言語インタープリター
```go
package main

type QueryLanguageInterpreter struct {
    // クエリ言語インタープリター
    
    func (receiver) __init__(data_source []Dict[str, Any]) {
        receiver.data_source = data_source
        receiver.context = Context()
        receiver.setup_query_functions()
        
    func (receiver) setup_query_functions() {
        // クエリ関数の設定
        receiver.context.set_function(// count, len)
        receiver.context.set_function(// sum, sum)
        receiver.context.set_function(// avg, lambda lst: sum(lst) / len(lst) if lst else 0)
        receiver.context.set_function(// max, max)
        receiver.context.set_function(// min, min)
        
    func (receiver) execute_query(query_expr string) {
        // クエリの実行
        results = []
        
        for record in receiver.data_source:
            # レコードをコンテキストに設定
            for key, value in record.items():
                receiver.context.set_variable(key, value)
                
            # クエリ評価
            tokenizer = Tokenizer(query_expr)
            tokens = tokenizer.tokenize()
            parser = Parser(tokens)
            expression = parser.parse()
            
            if expression.interpret(receiver.context):
                results.append(record.copy())
                
        return results

# 使用例
data = [
    {// name: // Alice, // age: 30, // salary: 70000, // department: // Engineering},
    {// name: // Bob, // age: 25, // salary: 50000, // department: // Marketing},
    {// name: // Charlie, // age: 35, // salary: 80000, // department: // Engineering},
    {// name: // Diana, // age: 28, // salary: 60000, // department: // Sales},
]

query_engine = QueryLanguageInterpreter(data)

# クエリ実行例
queries = [
    // age > 25,                                    # 年齢が25歳超
    // salary >= 60000,                            # 給与が60000以上
    // age > 25 AND department == \Engineering\"", # エンジニアリング部門の25歳超
]

for query in queries:
    print(f// Query: {query})
    results = query_engine.execute_query(query)
    for result in results:
        print(f//   {result})
    print()
```

## 実装時の考慮事項

### 1. エラーハンドリング
```go
package main

type InterpreterError struct {
    // インタープリターエラー
    // TODO: implement

type SyntaxError struct {
    // 構文エラー
    func (receiver) __init__(message string, position int) {
        receiver.message = message
        receiver.position = position
        super().__init__(f// Syntax error at position {position}: {message})

type RuntimeError struct {
    // 実行時エラー
    func (receiver) __init__(message string, expression: Expression) {
        receiver.message = message
        receiver.expression = expression
        super().__init__(f// Runtime error in '{expression}': {message})

type SafeInterpreter struct {
    // 安全なインタープリター
    
    func (receiver) __init__(max_depth int = 100, timeout_seconds int = 10) {
        receiver.max_depth = max_depth
        receiver.timeout_seconds = timeout_seconds
        receiver.current_depth = 0
        
    func (receiver) safe_interpret(expression: Expression, context: Context) {
        // 安全な解釈実行
                
        func timeout_handler(signum, frame) {
            raise TimeoutError(// Interpretation timeout)
            
        # タイムアウト設定
        signal.signal(signal.SIGALRM, timeout_handler)
        signal.alarm(receiver.timeout_seconds)
        
        try:
            receiver.current_depth = 0
            return receiver._safe_interpret_recursive(expression, context)
        finally:
            signal.alarm(0)  # タイムアウト解除
            
    func (receiver) _safe_interpret_recursive(expression: Expression, context: Context) {
        // 再帰深度チェック付き解釈
        receiver.current_depth += 1
        
        if receiver.current_depth > receiver.max_depth:
            raise RuntimeError(// Maximum recursion depth exceeded, expression)
            
        try:
            result = expression.interpret(context)
            return result
        except Exception as e:
            raise RuntimeError(str(e), expression)
        finally:
            receiver.current_depth -= 1
```

### 2. パフォーマンス最適化
```go
package main

type OptimizedExpression struct {
    // 最適化された表現
    
    func (receiver) __init__(original: Expression) {
        receiver.original = original
        receiver.cached_result = nil
        receiver.is_constant = receiver._check_if_constant()
        
    func (receiver) _check_if_constant() {
        // 定数式かどうかチェック
        return isinstance(receiver.original, (Constant, StringLiteral, BooleanLiteral))
        
    func (receiver) interpret(context: Context) {
        // 最適化された解釈実行
        if receiver.is_constant and receiver.cached_result is not nil:
            return receiver.cached_result
            
        result = receiver.original.interpret(context)
        
        if receiver.is_constant:
            receiver.cached_result = result
            
        return result
        
    func (receiver) __str__() {
        return str(receiver.original)

type ExpressionOptimizer struct {
    // 式の最適化器
    
    func (receiver) optimize(expression: Expression) {
        // 式の最適化
        if isinstance(expression, BinaryOperatorExpression):
            return receiver._optimize_binary_operation(expression)
        elif isinstance(expression, BooleanBinaryExpression):
            return receiver._optimize_boolean_operation(expression)
        else:
            return expression
            
    func (receiver) _optimize_binary_operation(expr: BinaryOperatorExpression) {
        // 二項演算の最適化
        left = receiver.optimize(expr.left)
        right = receiver.optimize(expr.right)
        
        # 定数畳み込み
        if isinstance(left, Constant) and isinstance(right, Constant):
            try:
                result = expr.operate(left.value, right.value)
                return Constant(result)
            except:
                // TODO: implement  # エラーの場合は最適化しない
                
        # ゼロとの演算最適化
        if isinstance(expr, AddExpression):
            if isinstance(left, Constant) and left.value == 0:
                return right
            if isinstance(right, Constant) and right.value == 0:
                return left
                
        if isinstance(expr, MultiplyExpression):
            if isinstance(left, Constant):
                if left.value == 0:
                    return Constant(0)
                if left.value == 1:
                    return right
            if isinstance(right, Constant):
                if right.value == 0:
                    return Constant(0)
                if right.value == 1:
                    return left
                    
        # 最適化されたノードを返す
        expr.left = left
        expr.right = right
        return expr
        
    func (receiver) _optimize_boolean_operation(expr: BooleanBinaryExpression) {
        // ブール演算の最適化
        left = receiver.optimize(expr.left)
        right = receiver.optimize(expr.right)
        
        if isinstance(expr, AndExpression):
            # false AND x = false
            if isinstance(left, BooleanLiteral) and not left.value:
                return BooleanLiteral(false)
            if isinstance(right, BooleanLiteral) and not right.value:
                return BooleanLiteral(false)
            # true AND x = x
            if isinstance(left, BooleanLiteral) and left.value:
                return right
            if isinstance(right, BooleanLiteral) and right.value:
                return left
                
        if isinstance(expr, OrExpression):
            # true OR x = true
            if isinstance(left, BooleanLiteral) and left.value:
                return BooleanLiteral(true)
            if isinstance(right, BooleanLiteral) and right.value:
                return BooleanLiteral(true)
            # false OR x = x
            if isinstance(left, BooleanLiteral) and not left.value:
                return right
            if isinstance(right, BooleanLiteral) and not right.value:
                return left
                
        expr.left = left
        expr.right = right
        return expr
```

## メリット

### 柔軟性
- **DSL作成**: ドメイン特化の表現力豊かな言語
- **動的実行**: 実行時の式解釈とカスタマイズ
- **拡張性**: 新しい演算子や関数の追加

### 保守性
- **文法の変更**: 言語仕様の独立した管理
- **テスタビリティ**: 各表現クラスの単体テスト
- **デバッグ**: 構文木の可視化とトレース

### 再利用性
- **表現の組み合わせ**: 基本要素から複雑な式を構築
- **言語の再利用**: 異なるアプリケーションでの同一文法利用
- **コンポーネント化**: パーサーやインタープリターの再利用

## デメリット

### パフォーマンス
- **解釈オーバーヘッド**: 実行時の式解釈コスト
- **メモリ使用量**: 構文木の構築とメンテナンス
- **複雑な式**: 深いネストや大きな式での性能劣化

### 複雑性
- **実装コスト**: パーサーとインタープリターの実装
- **デバッグ困難**: 言語の問題とアプリケーションの問題の分離
- **学習コスト**: 独自言語の習得

## 適用場面

### 適している場合
- **ルールエンジン**: ビジネスルールの動的定義
- **設定ファイル**: 複雑な設定の表現
- **数式計算**: 動的な計算式の処理
- **クエリ言語**: 柔軟なデータ検索・フィルタリング

### 適していない場合
- **パフォーマンス重視**: 高速処理が必要なシステム
- **シンプルな処理**: 固定的なロジックで十分な場合
- **大規模言語**: 複雑すぎる言語は既存言語を使用
- **頻繁な変更**: 言語仕様が安定しない場合

## 関連パターン
- **Composite Pattern**: 構文木の構造
- **Visitor Pattern**: 構文木の走査と処理
- **Strategy Pattern**: 異なる解釈戦略
- **Command Pattern**: 実行可能なコマンドの表現
- **Builder Pattern**: 複雑な式の構築

## 参考資料
- [Design Patterns: Elements of Reusable Object-Oriented Software](https://www.oreilly.com/library/view/design-patterns-elements/0201633612/)
- [Crafting Interpreters - Robert Nystrom](https://craftinginterpreters.com/)
- [Language Implementation Patterns - Terence Parr](https://pragprog.com/titles/tpdsl/language-implementation-patterns/)
- [Compilers: Principles, Techniques, and Tools](https://www.pearson.com/store/p/compilers-principles-techniques-and-tools/P100000315333)
