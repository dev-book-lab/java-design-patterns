# Interpreter Pattern (인터프리터 패턴)

> **"언어의 문법을 정의하고 해석하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [장단점](#6-장단점)
7. [안티패턴](#7-안티패턴)
8. [핵심 정리](#8-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 표현식 파싱이 복잡
public class Calculator {
    public int calculate(String expression) {
        // "2 + 3 * 4" 를 어떻게 계산?
        
        String[] tokens = expression.split(" ");
        // 연산자 우선순위는?
        // 괄호 처리는?
        // if-else 지옥!
        
        if (tokens[1].equals("+")) {
            return Integer.parseInt(tokens[0]) + 
                   Integer.parseInt(tokens[2]);
        } else if (tokens[1].equals("*")) {
            return Integer.parseInt(tokens[0]) * 
                   Integer.parseInt(tokens[2]);
        }
        // 복잡한 표현식은?
    }
}

// 문제 2: 규칙 평가가 어려움
public class AccessControl {
    public boolean checkAccess(User user, String rule) {
        // "role = admin AND department = IT"
        // 어떻게 파싱하고 평가?
        
        if (rule.contains("AND")) {
            String[] parts = rule.split("AND");
            // 재귀적으로?
            // 복잡!
        }
    }
}

// 문제 3: DSL 구현
public class QueryBuilder {
    public List<User> query(String dsl) {
        // "SELECT users WHERE age > 18 AND city = 'Seoul'"
        // SQL 파서를 직접 만들기?
        // 너무 복잡!
        
        // 정규식으로?
        // 유지보수 어려움!
    }
}

// 문제 4: 설정 파일 해석
public class ConfigParser {
    public void parse(String config) {
        // "if (env == 'prod') then timeout = 30"
        // 조건문, 변수 할당...
        // 파서 라이브러리 의존?
    }
}
```

### ⚡ 핵심 문제

1. **복잡한 파싱**: 문법 규칙을 코드로 표현 어려움
2. **확장성 부족**: 새 문법 추가 시 전체 수정
3. **재사용 불가**: 파싱 로직이 흩어짐
4. **유지보수 어려움**: 문법 변경 시 파급 효과

---

## 2. 패턴 정의

### 📖 정의

> **간단한 언어의 문법을 정의하고, 그 언어로 작성된 문장을 해석하는 인터프리터를 제공하는 패턴**

### 🎯 목적

- **문법 정의**: 언어의 문법을 클래스로 표현
- **해석 제공**: 문장을 파싱하고 실행
- **확장 용이**: 새 문법 추가 쉬움
- **재사용**: 문법 규칙 재사용

### 💡 핵심 아이디어

```java
// Before: 문자열 파싱
if (expr.contains("+")) {
    String[] parts = expr.split("+");
    return parse(parts[0]) + parse(parts[1]);
}

// After: Expression 객체로
Expression expr = new Add(
    new Number(2),
    new Number(3)
);
int result = expr.interpret(context);
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌──────────────────┐
│AbstractExpression│  ← 표현식 인터페이스
├──────────────────┤
│ + interpret()    │
└──────────────────┘
         △
         │ extends
         │
    ┌────┴─────┐
    │          │
┌───────────┐ ┌──────────────┐
│Terminal   │ │NonTerminal   │
│Expression │ │Expression    │
├───────────┤ ├──────────────┤
│interpret()│ │- expr1, expr2│
└───────────┘ │+ interpret() │
              └──────────────┘

┌──────────────────┐
│    Context       │  ← 전역 정보
├──────────────────┤
│ - variables      │
└──────────────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **AbstractExpression** | 표현식 인터페이스 | `Expression` |
| **TerminalExpression** | 말단 표현식 | `Number` |
| **NonterminalExpression** | 비말단 표현식 | `Add`, `Multiply` |
| **Context** | 해석에 필요한 정보 | `변수 저장소` |

---

## 4. 구현 방법

### 기본 구현: 간단한 계산기 ⭐⭐⭐

```java
/**
 * Context: 변수 저장소
 */
public class Context {
    private Map<String, Integer> variables;
    
    public Context() {
        this.variables = new HashMap<>();
    }
    
    public void setVariable(String name, int value) {
        variables.put(name, value);
        System.out.println("✅ 변수 설정: " + name + " = " + value);
    }
    
    public int getVariable(String name) {
        return variables.getOrDefault(name, 0);
    }
}

/**
 * AbstractExpression: 표현식
 */
public interface Expression {
    int interpret(Context context);
}

/**
 * TerminalExpression: 숫자
 */
public class NumberExpression implements Expression {
    private int number;
    
    public NumberExpression(int number) {
        this.number = number;
    }
    
    @Override
    public int interpret(Context context) {
        return number;
    }
    
    @Override
    public String toString() {
        return String.valueOf(number);
    }
}

/**
 * TerminalExpression: 변수
 */
public class VariableExpression implements Expression {
    private String name;
    
    public VariableExpression(String name) {
        this.name = name;
    }
    
    @Override
    public int interpret(Context context) {
        return context.getVariable(name);
    }
    
    @Override
    public String toString() {
        return name;
    }
}

/**
 * NonterminalExpression: 덧셈
 */
public class AddExpression implements Expression {
    private Expression left;
    private Expression right;
    
    public AddExpression(Expression left, Expression right) {
        this.left = left;
        this.right = right;
    }
    
    @Override
    public int interpret(Context context) {
        int leftValue = left.interpret(context);
        int rightValue = right.interpret(context);
        int result = leftValue + rightValue;
        System.out.println("  " + leftValue + " + " + rightValue + " = " + result);
        return result;
    }
    
    @Override
    public String toString() {
        return "(" + left + " + " + right + ")";
    }
}

/**
 * NonterminalExpression: 뺄셈
 */
public class SubtractExpression implements Expression {
    private Expression left;
    private Expression right;
    
    public SubtractExpression(Expression left, Expression right) {
        this.left = left;
        this.right = right;
    }
    
    @Override
    public int interpret(Context context) {
        int leftValue = left.interpret(context);
        int rightValue = right.interpret(context);
        int result = leftValue - rightValue;
        System.out.println("  " + leftValue + " - " + rightValue + " = " + result);
        return result;
    }
    
    @Override
    public String toString() {
        return "(" + left + " - " + right + ")";
    }
}

/**
 * NonterminalExpression: 곱셈
 */
public class MultiplyExpression implements Expression {
    private Expression left;
    private Expression right;
    
    public MultiplyExpression(Expression left, Expression right) {
        this.left = left;
        this.right = right;
    }
    
    @Override
    public int interpret(Context context) {
        int leftValue = left.interpret(context);
        int rightValue = right.interpret(context);
        int result = leftValue * rightValue;
        System.out.println("  " + leftValue + " * " + rightValue + " = " + result);
        return result;
    }
    
    @Override
    public String toString() {
        return "(" + left + " * " + right + ")";
    }
}

/**
 * Parser: 표현식 파서
 */
public class ExpressionParser {
    
    public static Expression parse(String expression) {
        // 간단한 후위 표기법 파서
        // "3 4 + 5 *" → (3 + 4) * 5
        
        Stack<Expression> stack = new Stack<>();
        String[] tokens = expression.split(" ");
        
        for (String token : tokens) {
            if (isOperator(token)) {
                Expression right = stack.pop();
                Expression left = stack.pop();
                
                switch (token) {
                    case "+":
                        stack.push(new AddExpression(left, right));
                        break;
                    case "-":
                        stack.push(new SubtractExpression(left, right));
                        break;
                    case "*":
                        stack.push(new MultiplyExpression(left, right));
                        break;
                }
            } else if (isNumber(token)) {
                stack.push(new NumberExpression(Integer.parseInt(token)));
            } else {
                stack.push(new VariableExpression(token));
            }
        }
        
        return stack.pop();
    }
    
    private static boolean isOperator(String token) {
        return token.equals("+") || token.equals("-") || token.equals("*");
    }
    
    private static boolean isNumber(String token) {
        try {
            Integer.parseInt(token);
            return true;
        } catch (NumberFormatException e) {
            return false;
        }
    }
}

/**
 * 사용 예제
 */
public class InterpreterExample {
    public static void main(String[] args) {
        Context context = new Context();
        
        System.out.println("=== 간단한 계산기 ===\n");
        
        // 1. 직접 생성
        System.out.println("--- 직접 생성 ---");
        Expression expr1 = new AddExpression(
            new NumberExpression(3),
            new NumberExpression(4)
        );
        System.out.println("표현식: " + expr1);
        int result1 = expr1.interpret(context);
        System.out.println("결과: " + result1);
        
        // 2. 복잡한 표현식: (3 + 4) * 5
        System.out.println("\n--- 복잡한 표현식 ---");
        Expression expr2 = new MultiplyExpression(
            new AddExpression(
                new NumberExpression(3),
                new NumberExpression(4)
            ),
            new NumberExpression(5)
        );
        System.out.println("표현식: " + expr2);
        int result2 = expr2.interpret(context);
        System.out.println("결과: " + result2);
        
        // 3. 변수 사용
        System.out.println("\n--- 변수 사용 ---");
        context.setVariable("x", 10);
        context.setVariable("y", 20);
        
        Expression expr3 = new AddExpression(
            new VariableExpression("x"),
            new VariableExpression("y")
        );
        System.out.println("표현식: " + expr3);
        int result3 = expr3.interpret(context);
        System.out.println("결과: " + result3);
        
        // 4. 파서 사용 (후위 표기법)
        System.out.println("\n--- 파서 사용 ---");
        String postfix = "3 4 + 5 *";
        System.out.println("후위 표기법: " + postfix);
        Expression expr4 = ExpressionParser.parse(postfix);
        System.out.println("표현식: " + expr4);
        int result4 = expr4.interpret(context);
        System.out.println("결과: " + result4);
    }
}
```

**실행 결과:**
```
=== 간단한 계산기 ===

--- 직접 생성 ---
표현식: (3 + 4)
  3 + 4 = 7
결과: 7

--- 복잡한 표현식 ---
표현식: ((3 + 4) * 5)
  3 + 4 = 7
  7 * 5 = 35
결과: 35

--- 변수 사용 ---
✅ 변수 설정: x = 10
✅ 변수 설정: y = 20
표현식: (x + y)
  10 + 20 = 30
결과: 30

--- 파서 사용 ---
후위 표기법: 3 4 + 5 *
표현식: ((3 + 4) * 5)
  3 + 4 = 7
  7 * 5 = 35
결과: 35
```

---

## 5. 실전 예제

### 예제 1: 불린 표현식 평가 ⭐⭐⭐

```java
/**
 * Context: 변수 저장
 */
public class BooleanContext {
    private Map<String, Boolean> variables;
    
    public BooleanContext() {
        this.variables = new HashMap<>();
    }
    
    public void assign(String name, boolean value) {
        variables.put(name, value);
        System.out.println("✅ " + name + " = " + value);
    }
    
    public boolean lookup(String name) {
        return variables.getOrDefault(name, false);
    }
}

/**
 * AbstractExpression: 불린 표현식
 */
public interface BooleanExpression {
    boolean interpret(BooleanContext context);
}

/**
 * TerminalExpression: 상수
 */
public class Constant implements BooleanExpression {
    private boolean value;
    
    public Constant(boolean value) {
        this.value = value;
    }
    
    @Override
    public boolean interpret(BooleanContext context) {
        return value;
    }
    
    @Override
    public String toString() {
        return String.valueOf(value);
    }
}

/**
 * TerminalExpression: 변수
 */
public class Variable implements BooleanExpression {
    private String name;
    
    public Variable(String name) {
        this.name = name;
    }
    
    @Override
    public boolean interpret(BooleanContext context) {
        return context.lookup(name);
    }
    
    @Override
    public String toString() {
        return name;
    }
}

/**
 * NonterminalExpression: AND
 */
public class And implements BooleanExpression {
    private BooleanExpression left;
    private BooleanExpression right;
    
    public And(BooleanExpression left, BooleanExpression right) {
        this.left = left;
        this.right = right;
    }
    
    @Override
    public boolean interpret(BooleanContext context) {
        boolean leftValue = left.interpret(context);
        boolean rightValue = right.interpret(context);
        boolean result = leftValue && rightValue;
        System.out.println("  " + leftValue + " AND " + rightValue + " = " + result);
        return result;
    }
    
    @Override
    public String toString() {
        return "(" + left + " AND " + right + ")";
    }
}

/**
 * NonterminalExpression: OR
 */
public class Or implements BooleanExpression {
    private BooleanExpression left;
    private BooleanExpression right;
    
    public Or(BooleanExpression left, BooleanExpression right) {
        this.left = left;
        this.right = right;
    }
    
    @Override
    public boolean interpret(BooleanContext context) {
        boolean leftValue = left.interpret(context);
        boolean rightValue = right.interpret(context);
        boolean result = leftValue || rightValue;
        System.out.println("  " + leftValue + " OR " + rightValue + " = " + result);
        return result;
    }
    
    @Override
    public String toString() {
        return "(" + left + " OR " + right + ")";
    }
}

/**
 * NonterminalExpression: NOT
 */
public class Not implements BooleanExpression {
    private BooleanExpression expression;
    
    public Not(BooleanExpression expression) {
        this.expression = expression;
    }
    
    @Override
    public boolean interpret(BooleanContext context) {
        boolean value = expression.interpret(context);
        boolean result = !value;
        System.out.println("  NOT " + value + " = " + result);
        return result;
    }
    
    @Override
    public String toString() {
        return "(NOT " + expression + ")";
    }
}

/**
 * 사용 예제
 */
public class BooleanInterpreterExample {
    public static void main(String[] args) {
        BooleanContext context = new BooleanContext();
        
        System.out.println("=== 불린 표현식 평가 ===\n");
        
        // 변수 할당
        System.out.println("--- 변수 할당 ---");
        context.assign("isAdmin", true);
        context.assign("isActive", true);
        context.assign("isBanned", false);
        
        // 1. (isAdmin AND isActive)
        System.out.println("\n--- 표현식 1 ---");
        BooleanExpression expr1 = new And(
            new Variable("isAdmin"),
            new Variable("isActive")
        );
        System.out.println("표현식: " + expr1);
        boolean result1 = expr1.interpret(context);
        System.out.println("결과: " + result1);
        
        // 2. (isAdmin AND isActive) AND (NOT isBanned)
        System.out.println("\n--- 표현식 2 ---");
        BooleanExpression expr2 = new And(
            new And(
                new Variable("isAdmin"),
                new Variable("isActive")
            ),
            new Not(new Variable("isBanned"))
        );
        System.out.println("표현식: " + expr2);
        boolean result2 = expr2.interpret(context);
        System.out.println("결과: " + result2);
        
        // 3. 권한 체크
        System.out.println("\n--- 권한 체크 ---");
        context.assign("hasPermission", false);
        
        BooleanExpression accessCheck = new Or(
            new Variable("isAdmin"),
            new Variable("hasPermission")
        );
        System.out.println("표현식: " + accessCheck);
        boolean hasAccess = accessCheck.interpret(context);
        System.out.println("접근 가능: " + hasAccess);
    }
}
```

---

### 예제 2: 간단한 SQL 인터프리터 ⭐⭐

```java
/**
 * Context: 데이터베이스
 */
public class Database {
    private List<Map<String, Object>> table;
    
    public Database() {
        this.table = new ArrayList<>();
    }
    
    public void addRow(Map<String, Object> row) {
        table.add(row);
    }
    
    public List<Map<String, Object>> getTable() {
        return table;
    }
}

/**
 * AbstractExpression: SQL 표현식
 */
public interface SqlExpression {
    List<Map<String, Object>> interpret(Database db);
}

/**
 * TerminalExpression: SELECT ALL
 */
public class SelectAll implements SqlExpression {
    
    @Override
    public List<Map<String, Object>> interpret(Database db) {
        System.out.println("📋 SELECT ALL 실행");
        return new ArrayList<>(db.getTable());
    }
}

/**
 * NonterminalExpression: WHERE
 */
public class Where implements SqlExpression {
    private SqlExpression baseQuery;
    private String column;
    private Object value;
    
    public Where(SqlExpression baseQuery, String column, Object value) {
        this.baseQuery = baseQuery;
        this.column = column;
        this.value = value;
    }
    
    @Override
    public List<Map<String, Object>> interpret(Database db) {
        System.out.println("🔍 WHERE " + column + " = " + value);
        List<Map<String, Object>> baseResult = baseQuery.interpret(db);
        List<Map<String, Object>> filtered = new ArrayList<>();
        
        for (Map<String, Object> row : baseResult) {
            if (value.equals(row.get(column))) {
                filtered.add(row);
            }
        }
        
        System.out.println("   결과: " + filtered.size() + " 행");
        return filtered;
    }
}

/**
 * 사용 예제
 */
public class SqlInterpreterExample {
    public static void main(String[] args) {
        // 데이터베이스 준비
        Database db = new Database();
        
        Map<String, Object> row1 = new HashMap<>();
        row1.put("id", 1);
        row1.put("name", "Alice");
        row1.put("age", 25);
        row1.put("city", "Seoul");
        
        Map<String, Object> row2 = new HashMap<>();
        row2.put("id", 2);
        row2.put("name", "Bob");
        row2.put("age", 30);
        row2.put("city", "Busan");
        
        Map<String, Object> row3 = new HashMap<>();
        row3.put("id", 3);
        row3.put("name", "Charlie");
        row3.put("age", 25);
        row3.put("city", "Seoul");
        
        db.addRow(row1);
        db.addRow(row2);
        db.addRow(row3);
        
        System.out.println("=== SQL 인터프리터 ===\n");
        
        // SELECT * FROM users
        System.out.println("--- Query 1 ---");
        SqlExpression query1 = new SelectAll();
        List<Map<String, Object>> result1 = query1.interpret(db);
        printResults(result1);
        
        // SELECT * FROM users WHERE city = 'Seoul'
        System.out.println("\n--- Query 2 ---");
        SqlExpression query2 = new Where(
            new SelectAll(),
            "city",
            "Seoul"
        );
        List<Map<String, Object>> result2 = query2.interpret(db);
        printResults(result2);
        
        // SELECT * FROM users WHERE age = 25
        System.out.println("\n--- Query 3 ---");
        SqlExpression query3 = new Where(
            new SelectAll(),
            "age",
            25
        );
        List<Map<String, Object>> result3 = query3.interpret(db);
        printResults(result3);
    }
    
    private static void printResults(List<Map<String, Object>> results) {
        System.out.println("\n결과:");
        for (Map<String, Object> row : results) {
            System.out.println("  " + row);
        }
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **문법 확장** | 새 문법 추가 쉬움 | Expression 추가 |
| **재사용** | 표현식 재사용 | And, Or 조합 |
| **유지보수** | 문법이 클래스로 명확 | 각 연산자 클래스 |
| **복합 가능** | 표현식 조합 | Composite 패턴 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **클래스 증가** | 문법마다 클래스 | 간단한 문법만 |
| **복잡한 문법 어려움** | 파서 구현 복잡 | ANTLR 등 사용 |
| **성능** | 트리 순회 비용 | 캐싱 |

---

## 7. 안티패턴

### ❌ 안티패턴: 복잡한 문법에 사용

```java
// 잘못된 예: 너무 복잡한 문법
// SQL 전체 구현? 클래스 폭발!
class Select { }
class From { }
class Where { }
class Join { }
class GroupBy { }
class Having { }
class OrderBy { }
// ...수백 개의 클래스!
```

**해결:**
```java
// 간단한 DSL만 Interpreter로
// 복잡한 언어는 파서 생성기 사용
// ANTLR, JavaCC 등
```

---

## 8. 핵심 정리

### 📌 Interpreter 패턴 체크리스트

```
✅ AbstractExpression 정의
✅ TerminalExpression 구현
✅ NonterminalExpression 구현
✅ Context 정의
✅ interpret() 메서드
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 간단한 문법 | ⭐⭐⭐ | 클래스 관리 |
| DSL 구현 | ⭐⭐⭐ | 재사용 |
| 규칙 평가 | ⭐⭐⭐ | 조합 가능 |
| 표현식 계산 | ⭐⭐⭐ | 트리 구조 |

### 💡 핵심 포인트

1. **문법을 클래스로**
2. **재귀적 해석**
3. **Composite 패턴**
4. **간단한 언어만**

### 🔥 실무 활용

```java
// 간단한 DSL에만 사용
// - 설정 파일
// - 간단한 쿼리
// - 비즈니스 규칙

// 복잡한 언어는 파서 생성기 사용
// - ANTLR
// - JavaCC
// - Parboiled
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Visitor](10-Visitor.md)**

</div>
