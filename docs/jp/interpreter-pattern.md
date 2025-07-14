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
```python
from abc import ABC, abstractmethod
from typing import Dict, Any, List, Optional

class Context:
    """インタープリターのコンテキスト"""
    def __init__(self):
        self.variables: Dict[str, Any] = {}
        self.functions: Dict[str, callable] = {}
        
    def set_variable(self, name: str, value: Any) -> None:
        """変数設定"""
        self.variables[name] = value
        
    def get_variable(self, name: str) -> Any:
        """変数取得"""
        return self.variables.get(name)
        
    def set_function(self, name: str, func: callable) -> None:
        """関数設定"""
        self.functions[name] = func
        
    def get_function(self, name: str) -> Optional[callable]:
        """関数取得"""
        return self.functions.get(name)

class Expression(ABC):
    """抽象表現クラス"""
    
    @abstractmethod
    def interpret(self, context: Context) -> Any:
        """式の解釈実行"""
        pass
        
    @abstractmethod
    def __str__(self) -> str:
        """文字列表現"""
        pass

class BooleanExpression(Expression):
    """ブール式の抽象クラス"""
    
    @abstractmethod
    def interpret(self, context: Context) -> bool:
        """ブール値の解釈実行"""
        pass

class NumericExpression(Expression):
    """数値式の抽象クラス"""
    
    @abstractmethod
    def interpret(self, context: Context) -> float:
        """数値の解釈実行"""
        pass
```

### 2. 終端表現（Terminal Expression）
```python
class Variable(Expression):
    """変数表現"""
    
    def __init__(self, name: str):
        self.name = name
        
    def interpret(self, context: Context) -> Any:
        """変数値の取得"""
        value = context.get_variable(self.name)
        if value is None:
            raise ValueError(f"Undefined variable: {self.name}")
        return value
        
    def __str__(self) -> str:
        return self.name

class Constant(NumericExpression):
    """定数表現"""
    
    def __init__(self, value: float):
        self.value = value
        
    def interpret(self, context: Context) -> float:
        """定数値の返却"""
        return self.value
        
    def __str__(self) -> str:
        return str(self.value)

class StringLiteral(Expression):
    """文字列リテラル表現"""
    
    def __init__(self, value: str):
        self.value = value
        
    def interpret(self, context: Context) -> str:
        """文字列値の返却"""
        return self.value
        
    def __str__(self) -> str:
        return f'"{self.value}"'

class BooleanLiteral(BooleanExpression):
    """ブールリテラル表現"""
    
    def __init__(self, value: bool):
        self.value = value
        
    def interpret(self, context: Context) -> bool:
        """ブール値の返却"""
        return self.value
        
    def __str__(self) -> str:
        return "true" if self.value else "false"
```

### 3. 非終端表現（Nonterminal Expression）
```python
class BinaryOperatorExpression(NumericExpression):
    """二項演算子表現の基底クラス"""
    
    def __init__(self, left: NumericExpression, right: NumericExpression):
        self.left = left
        self.right = right
        
    @abstractmethod
    def operate(self, left_val: float, right_val: float) -> float:
        """演算実行"""
        pass
        
    def interpret(self, context: Context) -> float:
        """二項演算の解釈実行"""
        left_val = self.left.interpret(context)
        right_val = self.right.interpret(context)
        return self.operate(left_val, right_val)

class AddExpression(BinaryOperatorExpression):
    """加算表現"""
    
    def operate(self, left_val: float, right_val: float) -> float:
        return left_val + right_val
        
    def __str__(self) -> str:
        return f"({self.left} + {self.right})"

class SubtractExpression(BinaryOperatorExpression):
    """減算表現"""
    
    def operate(self, left_val: float, right_val: float) -> float:
        return left_val - right_val
        
    def __str__(self) -> str:
        return f"({self.left} - {self.right})"

class MultiplyExpression(BinaryOperatorExpression):
    """乗算表現"""
    
    def operate(self, left_val: float, right_val: float) -> float:
        return left_val * right_val
        
    def __str__(self) -> str:
        return f"({self.left} * {self.right})"

class DivideExpression(BinaryOperatorExpression):
    """除算表現"""
    
    def operate(self, left_val: float, right_val: float) -> float:
        if right_val == 0:
            raise ValueError("Division by zero")
        return left_val / right_val
        
    def __str__(self) -> str:
        return f"({self.left} / {self.right})"

class PowerExpression(BinaryOperatorExpression):
    """累乗表現"""
    
    def operate(self, left_val: float, right_val: float) -> float:
        return left_val ** right_val
        
    def __str__(self) -> str:
        return f"({self.left} ^ {self.right})"

# ブール演算
class BooleanBinaryExpression(BooleanExpression):
    """ブール二項演算の基底クラス"""
    
    def __init__(self, left: BooleanExpression, right: BooleanExpression):
        self.left = left
        self.right = right

class AndExpression(BooleanBinaryExpression):
    """論理積表現"""
    
    def interpret(self, context: Context) -> bool:
        return self.left.interpret(context) and self.right.interpret(context)
        
    def __str__(self) -> str:
        return f"({self.left} AND {self.right})"

class OrExpression(BooleanBinaryExpression):
    """論理和表現"""
    
    def interpret(self, context: Context) -> bool:
        return self.left.interpret(context) or self.right.interpret(context)
        
    def __str__(self) -> str:
        return f"({self.left} OR {self.right})"

class NotExpression(BooleanExpression):
    """論理否定表現"""
    
    def __init__(self, expression: BooleanExpression):
        self.expression = expression
        
    def interpret(self, context: Context) -> bool:
        return not self.expression.interpret(context)
        
    def __str__(self) -> str:
        return f"(NOT {self.expression})"

# 比較演算
class ComparisonExpression(BooleanExpression):
    """比較演算の基底クラス"""
    
    def __init__(self, left: NumericExpression, right: NumericExpression):
        self.left = left
        self.right = right
        
    @abstractmethod
    def compare(self, left_val: float, right_val: float) -> bool:
        """比較実行"""
        pass
        
    def interpret(self, context: Context) -> bool:
        left_val = self.left.interpret(context)
        right_val = self.right.interpret(context)
        return self.compare(left_val, right_val)

class EqualsExpression(ComparisonExpression):
    """等価比較表現"""
    
    def compare(self, left_val: float, right_val: float) -> bool:
        return abs(left_val - right_val) < 1e-10  # 浮動小数点の比較
        
    def __str__(self) -> str:
        return f"({self.left} == {self.right})"

class GreaterThanExpression(ComparisonExpression):
    """大なり比較表現"""
    
    def compare(self, left_val: float, right_val: float) -> bool:
        return left_val > right_val
        
    def __str__(self) -> str:
        return f"({self.left} > {self.right})"

class LessThanExpression(ComparisonExpression):
    """小なり比較表現"""
    
    def compare(self, left_val: float, right_val: float) -> bool:
        return left_val < right_val
        
    def __str__(self) -> str:
        return f"({self.left} < {self.right})"
```

### 4. 複合表現と関数呼び出し
```python
class FunctionCallExpression(Expression):
    """関数呼び出し表現"""
    
    def __init__(self, function_name: str, arguments: List[Expression]):
        self.function_name = function_name
        self.arguments = arguments
        
    def interpret(self, context: Context) -> Any:
        """関数呼び出しの解釈実行"""
        func = context.get_function(self.function_name)
        if func is None:
            raise ValueError(f"Undefined function: {self.function_name}")
            
        # 引数を評価
        arg_values = [arg.interpret(context) for arg in self.arguments]
        
        try:
            return func(*arg_values)
        except Exception as e:
            raise ValueError(f"Error calling function {self.function_name}: {str(e)}")
            
    def __str__(self) -> str:
        args_str = ", ".join(str(arg) for arg in self.arguments)
        return f"{self.function_name}({args_str})"

class ConditionalExpression(Expression):
    """条件表現（三項演算子）"""
    
    def __init__(self, condition: BooleanExpression, 
                 true_expr: Expression, false_expr: Expression):
        self.condition = condition
        self.true_expr = true_expr
        self.false_expr = false_expr
        
    def interpret(self, context: Context) -> Any:
        """条件分岐の解釈実行"""
        if self.condition.interpret(context):
            return self.true_expr.interpret(context)
        else:
            return self.false_expr.interpret(context)
            
    def __str__(self) -> str:
        return f"({self.condition} ? {self.true_expr} : {self.false_expr})"

class SequenceExpression(Expression):
    """シーケンス表現（複数の式を順次実行）"""
    
    def __init__(self, expressions: List[Expression]):
        self.expressions = expressions
        
    def interpret(self, context: Context) -> Any:
        """シーケンス実行"""
        result = None
        for expr in self.expressions:
            result = expr.interpret(context)
        return result
        
    def __str__(self) -> str:
        return "; ".join(str(expr) for expr in self.expressions)

class AssignmentExpression(Expression):
    """代入表現"""
    
    def __init__(self, variable_name: str, value_expr: Expression):
        self.variable_name = variable_name
        self.value_expr = value_expr
        
    def interpret(self, context: Context) -> Any:
        """代入の解釈実行"""
        value = self.value_expr.interpret(context)
        context.set_variable(self.variable_name, value)
        return value
        
    def __str__(self) -> str:
        return f"{self.variable_name} = {self.value_expr}"
```

## パーサー（構文解析器）

### 1. トークナイザー（字句解析）
```python
import re
from enum import Enum
from dataclasses import dataclass
from typing import List, Iterator, Optional

class TokenType(Enum):
    """トークンタイプ"""
    NUMBER = "NUMBER"
    IDENTIFIER = "IDENTIFIER"
    STRING = "STRING"
    PLUS = "PLUS"
    MINUS = "MINUS"
    MULTIPLY = "MULTIPLY"
    DIVIDE = "DIVIDE"
    POWER = "POWER"
    LPAREN = "LPAREN"
    RPAREN = "RPAREN"
    ASSIGN = "ASSIGN"
    EQUALS = "EQUALS"
    GREATER = "GREATER"
    LESS = "LESS"
    AND = "AND"
    OR = "OR"
    NOT = "NOT"
    IF = "IF"
    THEN = "THEN"
    ELSE = "ELSE"
    COMMA = "COMMA"
    SEMICOLON = "SEMICOLON"
    EOF = "EOF"
    WHITESPACE = "WHITESPACE"

@dataclass
class Token:
    """トークン"""
    type: TokenType
    value: str
    position: int

class Tokenizer:
    """字句解析器"""
    
    TOKEN_PATTERNS = [
        (TokenType.NUMBER, r'\d+(\.\d+)?'),
        (TokenType.STRING, r'"([^"\\\\]|\\\\.)*"'),
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
    
    def __init__(self, text: str):
        self.text = text
        self.position = 0
        self.tokens: List[Token] = []
        
    def tokenize(self) -> List[Token]:
        """字句解析実行"""
        while self.position < len(self.text):
            match_found = False
            
            for token_type, pattern in self.TOKEN_PATTERNS:
                regex = re.compile(pattern)
                match = regex.match(self.text, self.position)
                
                if match:
                    value = match.group(0)
                    
                    # 空白は無視
                    if token_type != TokenType.WHITESPACE:
                        # キーワードチェック
                        if token_type == TokenType.IDENTIFIER:
                            token_type = self.KEYWORDS.get(value.upper(), TokenType.IDENTIFIER)
                            
                        self.tokens.append(Token(token_type, value, self.position))
                        
                    self.position = match.end()
                    match_found = True
                    break
                    
            if not match_found:
                raise ValueError(f"Invalid character at position {self.position}: {self.text[self.position]}")
                
        self.tokens.append(Token(TokenType.EOF, "", self.position))
        return self.tokens
```

### 2. 再帰下降パーサー
```python
class Parser:
    """再帰下降パーサー"""
    
    def __init__(self, tokens: List[Token]):
        self.tokens = tokens
        self.current = 0
        
    def parse(self) -> Expression:
        """構文解析実行"""
        expr = self.parse_sequence()
        if not self.is_at_end():
            raise ValueError(f"Unexpected token: {self.peek().value}")
        return expr
        
    def parse_sequence(self) -> Expression:
        """シーケンス解析"""
        expressions = []
        expressions.append(self.parse_assignment())
        
        while self.match(TokenType.SEMICOLON):
            expressions.append(self.parse_assignment())
            
        if len(expressions) == 1:
            return expressions[0]
        return SequenceExpression(expressions)
        
    def parse_assignment(self) -> Expression:
        """代入解析"""
        expr = self.parse_or()
        
        if self.match(TokenType.ASSIGN):
            if not isinstance(expr, Variable):
                raise ValueError("Invalid assignment target")
            value = self.parse_assignment()
            return AssignmentExpression(expr.name, value)
            
        return expr
        
    def parse_or(self) -> Expression:
        """論理和解析"""
        expr = self.parse_and()
        
        while self.match(TokenType.OR):
            right = self.parse_and()
            expr = OrExpression(expr, right)
            
        return expr
        
    def parse_and(self) -> Expression:
        """論理積解析"""
        expr = self.parse_equality()
        
        while self.match(TokenType.AND):
            right = self.parse_equality()
            expr = AndExpression(expr, right)
            
        return expr
        
    def parse_equality(self) -> Expression:
        """等価比較解析"""
        expr = self.parse_comparison()
        
        while self.match(TokenType.EQUALS):
            right = self.parse_comparison()
            expr = EqualsExpression(expr, right)
            
        return expr
        
    def parse_comparison(self) -> Expression:
        """比較解析"""
        expr = self.parse_addition()
        
        while True:
            if self.match(TokenType.GREATER):
                right = self.parse_addition()
                expr = GreaterThanExpression(expr, right)
            elif self.match(TokenType.LESS):
                right = self.parse_addition()
                expr = LessThanExpression(expr, right)
            else:
                break
                
        return expr
        
    def parse_addition(self) -> Expression:
        """加減算解析"""
        expr = self.parse_multiplication()
        
        while True:
            if self.match(TokenType.PLUS):
                right = self.parse_multiplication()
                expr = AddExpression(expr, right)
            elif self.match(TokenType.MINUS):
                right = self.parse_multiplication()
                expr = SubtractExpression(expr, right)
            else:
                break
                
        return expr
        
    def parse_multiplication(self) -> Expression:
        """乗除算解析"""
        expr = self.parse_power()
        
        while True:
            if self.match(TokenType.MULTIPLY):
                right = self.parse_power()
                expr = MultiplyExpression(expr, right)
            elif self.match(TokenType.DIVIDE):
                right = self.parse_power()
                expr = DivideExpression(expr, right)
            else:
                break
                
        return expr
        
    def parse_power(self) -> Expression:
        """累乗解析"""
        expr = self.parse_unary()
        
        if self.match(TokenType.POWER):
            right = self.parse_power()  # 右結合
            expr = PowerExpression(expr, right)
            
        return expr
        
    def parse_unary(self) -> Expression:
        """単項演算解析"""
        if self.match(TokenType.NOT):
            expr = self.parse_unary()
            return NotExpression(expr)
            
        if self.match(TokenType.MINUS):
            expr = self.parse_unary()
            return MultiplyExpression(Constant(-1), expr)
            
        return self.parse_primary()
        
    def parse_primary(self) -> Expression:
        """基本要素解析"""
        if self.match(TokenType.NUMBER):
            return Constant(float(self.previous().value))
            
        if self.match(TokenType.STRING):
            # 引用符を除去
            value = self.previous().value[1:-1]
            return StringLiteral(value)
            
        if self.match(TokenType.IDENTIFIER):
            name = self.previous().value
            
            # 関数呼び出しチェック
            if self.match(TokenType.LPAREN):
                arguments = []
                
                if not self.check(TokenType.RPAREN):
                    arguments.append(self.parse_assignment())
                    while self.match(TokenType.COMMA):
                        arguments.append(self.parse_assignment())
                        
                self.consume(TokenType.RPAREN, "Expected ')' after function arguments")
                return FunctionCallExpression(name, arguments)
                
            return Variable(name)
            
        if self.match(TokenType.LPAREN):
            expr = self.parse_assignment()
            self.consume(TokenType.RPAREN, "Expected ')' after expression")
            return expr
            
        raise ValueError(f"Unexpected token: {self.peek().value}")
        
    def match(self, *types: TokenType) -> bool:
        """トークンタイプマッチング"""
        for token_type in types:
            if self.check(token_type):
                self.advance()
                return True
        return False
        
    def check(self, token_type: TokenType) -> bool:
        """現在のトークンタイプチェック"""
        if self.is_at_end():
            return False
        return self.peek().type == token_type
        
    def advance(self) -> Token:
        """次のトークンに進む"""
        if not self.is_at_end():
            self.current += 1
        return self.previous()
        
    def is_at_end(self) -> bool:
        """終端チェック"""
        return self.peek().type == TokenType.EOF
        
    def peek(self) -> Token:
        """現在のトークン取得"""
        return self.tokens[self.current]
        
    def previous(self) -> Token:
        """前のトークン取得"""
        return self.tokens[self.current - 1]
        
    def consume(self, token_type: TokenType, message: str) -> Token:
        """期待されるトークンの消費"""
        if self.check(token_type):
            return self.advance()
        raise ValueError(message)
```

## 高度な機能実装

### 1. ドメイン固有言語（DSL）
```python
class RuleEngineInterpreter:
    """ルールエンジンのインタープリター"""
    
    def __init__(self):
        self.context = Context()
        self.setup_builtin_functions()
        
    def setup_builtin_functions(self):
        """組み込み関数の設定"""
        self.context.set_function("max", max)
        self.context.set_function("min", min)
        self.context.set_function("abs", abs)
        self.context.set_function("round", round)
        self.context.set_function("len", len)
        
        # カスタム関数
        self.context.set_function("contains", lambda text, substring: substring in text)
        self.context.set_function("startswith", lambda text, prefix: text.startswith(prefix))
        self.context.set_function("endswith", lambda text, suffix: text.endswith(suffix))
        
    def evaluate_rule(self, rule_text: str, data: Dict[str, Any]) -> bool:
        """ルールの評価"""
        # データをコンテキストに設定
        for key, value in data.items():
            self.context.set_variable(key, value)
            
        # パースして実行
        tokenizer = Tokenizer(rule_text)
        tokens = tokenizer.tokenize()
        parser = Parser(tokens)
        expression = parser.parse()
        
        result = expression.interpret(self.context)
        return bool(result)

# 使用例
rule_engine = RuleEngineInterpreter()

# 顧客データ
customer_data = {
    "age": 25,
    "income": 50000,
    "credit_score": 750,
    "employment_years": 3,
    "name": "John Doe"
}

# ルール定義
rules = [
    "age >= 18 AND age <= 65",                                    # 年齢制限
    "income > 30000",                                             # 収入制限
    "credit_score >= 700",                                        # 信用スコア
    "employment_years >= 2",                                      # 勤続年数
    "contains(name, \"John\")",                                   # 名前チェック
]

# ルール評価
for i, rule in enumerate(rules):
    result = rule_engine.evaluate_rule(rule, customer_data)
    print(f"Rule {i+1}: {rule} -> {result}")
```

### 2. 計算式エンジン
```python
class CalculationEngine:
    """計算式エンジン"""
    
    def __init__(self):
        self.context = Context()
        self.setup_math_functions()
        
    def setup_math_functions(self):
        """数学関数の設定"""
        import math
        
        self.context.set_function("sin", math.sin)
        self.context.set_function("cos", math.cos)
        self.context.set_function("tan", math.tan)
        self.context.set_function("log", math.log)
        self.context.set_function("log10", math.log10)
        self.context.set_function("sqrt", math.sqrt)
        self.context.set_function("exp", math.exp)
        
        # 定数
        self.context.set_variable("PI", math.pi)
        self.context.set_variable("E", math.e)
        
    def calculate(self, formula: str, variables: Dict[str, float] = None) -> float:
        """数式の計算"""
        if variables:
            for name, value in variables.items():
                self.context.set_variable(name, value)
                
        tokenizer = Tokenizer(formula)
        tokens = tokenizer.tokenize()
        parser = Parser(tokens)
        expression = parser.parse()
        
        result = expression.interpret(self.context)
        return float(result)

# 使用例
calc_engine = CalculationEngine()

# 基本計算
result1 = calc_engine.calculate("2 + 3 * 4")
print(f"2 + 3 * 4 = {result1}")

# 変数を使った計算
variables = {"x": 10, "y": 5}
result2 = calc_engine.calculate("x^2 + y^2", variables)
print(f"x^2 + y^2 = {result2}")

# 関数を使った計算
result3 = calc_engine.calculate("sin(PI/2) + cos(0)")
print(f"sin(π/2) + cos(0) = {result3}")
```

### 3. クエリ言語インタープリター
```python
class QueryLanguageInterpreter:
    """クエリ言語インタープリター"""
    
    def __init__(self, data_source: List[Dict[str, Any]]):
        self.data_source = data_source
        self.context = Context()
        self.setup_query_functions()
        
    def setup_query_functions(self):
        """クエリ関数の設定"""
        self.context.set_function("count", len)
        self.context.set_function("sum", sum)
        self.context.set_function("avg", lambda lst: sum(lst) / len(lst) if lst else 0)
        self.context.set_function("max", max)
        self.context.set_function("min", min)
        
    def execute_query(self, query_expr: str) -> List[Dict[str, Any]]:
        """クエリの実行"""
        results = []
        
        for record in self.data_source:
            # レコードをコンテキストに設定
            for key, value in record.items():
                self.context.set_variable(key, value)
                
            # クエリ評価
            tokenizer = Tokenizer(query_expr)
            tokens = tokenizer.tokenize()
            parser = Parser(tokens)
            expression = parser.parse()
            
            if expression.interpret(self.context):
                results.append(record.copy())
                
        return results

# 使用例
data = [
    {"name": "Alice", "age": 30, "salary": 70000, "department": "Engineering"},
    {"name": "Bob", "age": 25, "salary": 50000, "department": "Marketing"},
    {"name": "Charlie", "age": 35, "salary": 80000, "department": "Engineering"},
    {"name": "Diana", "age": 28, "salary": 60000, "department": "Sales"},
]

query_engine = QueryLanguageInterpreter(data)

# クエリ実行例
queries = [
    "age > 25",                                    # 年齢が25歳超
    "salary >= 60000",                            # 給与が60000以上
    "age > 25 AND department == \"Engineering\"", # エンジニアリング部門の25歳超
]

for query in queries:
    print(f"Query: {query}")
    results = query_engine.execute_query(query)
    for result in results:
        print(f"  {result}")
    print()
```

## 実装時の考慮事項

### 1. エラーハンドリング
```python
class InterpreterError(Exception):
    """インタープリターエラー"""
    pass

class SyntaxError(InterpreterError):
    """構文エラー"""
    def __init__(self, message: str, position: int):
        self.message = message
        self.position = position
        super().__init__(f"Syntax error at position {position}: {message}")

class RuntimeError(InterpreterError):
    """実行時エラー"""
    def __init__(self, message: str, expression: Expression):
        self.message = message
        self.expression = expression
        super().__init__(f"Runtime error in '{expression}': {message}")

class SafeInterpreter:
    """安全なインタープリター"""
    
    def __init__(self, max_depth: int = 100, timeout_seconds: int = 10):
        self.max_depth = max_depth
        self.timeout_seconds = timeout_seconds
        self.current_depth = 0
        
    def safe_interpret(self, expression: Expression, context: Context) -> Any:
        """安全な解釈実行"""
        import signal
        
        def timeout_handler(signum, frame):
            raise TimeoutError("Interpretation timeout")
            
        # タイムアウト設定
        signal.signal(signal.SIGALRM, timeout_handler)
        signal.alarm(self.timeout_seconds)
        
        try:
            self.current_depth = 0
            return self._safe_interpret_recursive(expression, context)
        finally:
            signal.alarm(0)  # タイムアウト解除
            
    def _safe_interpret_recursive(self, expression: Expression, context: Context) -> Any:
        """再帰深度チェック付き解釈"""
        self.current_depth += 1
        
        if self.current_depth > self.max_depth:
            raise RuntimeError("Maximum recursion depth exceeded", expression)
            
        try:
            result = expression.interpret(context)
            return result
        except Exception as e:
            raise RuntimeError(str(e), expression)
        finally:
            self.current_depth -= 1
```

### 2. パフォーマンス最適化
```python
class OptimizedExpression(Expression):
    """最適化された表現"""
    
    def __init__(self, original: Expression):
        self.original = original
        self.cached_result = None
        self.is_constant = self._check_if_constant()
        
    def _check_if_constant(self) -> bool:
        """定数式かどうかチェック"""
        return isinstance(self.original, (Constant, StringLiteral, BooleanLiteral))
        
    def interpret(self, context: Context) -> Any:
        """最適化された解釈実行"""
        if self.is_constant and self.cached_result is not None:
            return self.cached_result
            
        result = self.original.interpret(context)
        
        if self.is_constant:
            self.cached_result = result
            
        return result
        
    def __str__(self) -> str:
        return str(self.original)

class ExpressionOptimizer:
    """式の最適化器"""
    
    def optimize(self, expression: Expression) -> Expression:
        """式の最適化"""
        if isinstance(expression, BinaryOperatorExpression):
            return self._optimize_binary_operation(expression)
        elif isinstance(expression, BooleanBinaryExpression):
            return self._optimize_boolean_operation(expression)
        else:
            return expression
            
    def _optimize_binary_operation(self, expr: BinaryOperatorExpression) -> Expression:
        """二項演算の最適化"""
        left = self.optimize(expr.left)
        right = self.optimize(expr.right)
        
        # 定数畳み込み
        if isinstance(left, Constant) and isinstance(right, Constant):
            try:
                result = expr.operate(left.value, right.value)
                return Constant(result)
            except:
                pass  # エラーの場合は最適化しない
                
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
        
    def _optimize_boolean_operation(self, expr: BooleanBinaryExpression) -> Expression:
        """ブール演算の最適化"""
        left = self.optimize(expr.left)
        right = self.optimize(expr.right)
        
        if isinstance(expr, AndExpression):
            # false AND x = false
            if isinstance(left, BooleanLiteral) and not left.value:
                return BooleanLiteral(False)
            if isinstance(right, BooleanLiteral) and not right.value:
                return BooleanLiteral(False)
            # true AND x = x
            if isinstance(left, BooleanLiteral) and left.value:
                return right
            if isinstance(right, BooleanLiteral) and right.value:
                return left
                
        if isinstance(expr, OrExpression):
            # true OR x = true
            if isinstance(left, BooleanLiteral) and left.value:
                return BooleanLiteral(True)
            if isinstance(right, BooleanLiteral) and right.value:
                return BooleanLiteral(True)
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
