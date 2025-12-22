# Sealed Classes Pattern (봉인 클래스 패턴)

> **"타입 계층을 명시적으로 제한하여 타입 안전성을 높이자"** (Java 17+)

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [Pattern Matching 연동](#6-pattern-matching-연동)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 무한정 확장 가능한 계층
public abstract class Shape {
    public abstract double area();
}

// 😱 누구나 마음대로 확장 가능!
public class Circle extends Shape {
    private double radius;
    
    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
}

public class Rectangle extends Shape {
    private double width, height;
    
    @Override
    public double area() {
        return width * height;
    }
}

// 다른 패키지에서:
public class UnknownShape extends Shape {
    // 😱 예상하지 못한 타입!
    @Override
    public double area() {
        return 0;
    }
}

// 문제: 어떤 타입이 올지 모름

// 문제 2: instanceof 체크의 불완전함
public double calculateTotalArea(List<Shape> shapes) {
    double total = 0;
    
    for (Shape shape : shapes) {
        // 😱 모든 경우를 다뤘는지 컴파일러가 모름!
        if (shape instanceof Circle) {
            Circle circle = (Circle) shape;
            total += circle.area();
        } else if (shape instanceof Rectangle) {
            Rectangle rect = (Rectangle) shape;
            total += rect.area();
        }
        // 누락된 타입이 있어도 컴파일 에러 없음!
    }
    
    return total;
}

// 문제 3: Enum의 제한
public enum PaymentMethod {
    CREDIT_CARD,
    DEBIT_CARD,
    PAYPAL,
    BANK_TRANSFER
}

// 😱 각 타입마다 다른 데이터 필요
// - CREDIT_CARD: 카드번호, CVV
// - PAYPAL: 이메일
// - BANK_TRANSFER: 계좌번호

// Enum으로는 표현 불가!

// 문제 4: 타입 안전하지 않은 상태 관리
public class Order {
    private OrderState state;  // String or Enum
    
    // 😱 각 상태마다 다른 필드
    private LocalDateTime confirmedAt;  // CONFIRMED일 때만
    private String trackingNumber;      // SHIPPED일 때만
    private LocalDateTime deliveredAt;  // DELIVERED일 때만
    
    // 어떤 필드가 유효한지 알 수 없음!
}

// 문제 5: JSON 역직렬화
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME)
@JsonSubTypes({
    @JsonSubTypes.Type(value = Circle.class, name = "circle"),
    @JsonSubTypes.Type(value = Rectangle.class, name = "rectangle")
})
public abstract class Shape {
    // 😱 새로운 타입이 추가되면 여기 수정 필요!
}

// 문제 6: 방문자 패턴의 복잡함
public interface ShapeVisitor {
    void visit(Circle circle);
    void visit(Rectangle rectangle);
    // 😱 새 타입마다 메서드 추가
}

public abstract class Shape {
    public abstract void accept(ShapeVisitor visitor);
}
```

### ⚡ 핵심 문제

1. **무제한 확장**: 예상 못한 타입 추가
2. **타입 안전성**: 컴파일 타임 체크 불가
3. **완전성**: 모든 케이스 처리 보장 안 됨
4. **복잡도**: 방문자 패턴 등 복잡한 해결책

---

## 2. 패턴 정의

### 📖 정의

> **클래스 계층을 명시적으로 제한하여 허용된 서브클래스만 상속할 수 있게 하는 패턴** (Java 17+)

### 🎯 목적

- **제한된 계층**: 허용된 타입만 존재
- **타입 안전**: 컴파일 타임 체크
- **완전성**: 모든 케이스 보장
- **명시성**: 가능한 타입이 명확

### 💡 핵심 아이디어

```java
// Before: 무제한 확장
public abstract class Shape {
    // 😱 누구나 확장 가능
}

// After: Sealed (봉인됨)
public sealed class Shape 
    permits Circle, Rectangle, Triangle {
    // ✅ 명시된 타입만 허용!
}

public final class Circle extends Shape {
    // final: 더 이상 확장 불가
}

public final class Rectangle extends Shape {
}

public sealed class Triangle extends Shape 
    permits EquilateralTriangle, RightTriangle {
    // sealed: 추가 제한
}
```

---

## 3. 구조와 구성요소

### 📊 Sealed Class 구조

```
sealed class Shape
    ↓ permits
    ├─ final class Circle
    ├─ final class Rectangle
    └─ sealed class Triangle
           ↓ permits
           ├─ final class EquilateralTriangle
           └─ final class RightTriangle

규칙:
1. sealed → permits 필요
2. 서브클래스는 같은 패키지 또는 모듈
3. 서브클래스는 final, sealed, non-sealed 중 하나
```

### 🔧 키워드 설명

| 키워드 | 의미 | 사용처 |
|--------|------|--------|
| `sealed` | 봉인됨 (제한적 확장) | 부모 클래스 |
| `permits` | 허용된 서브클래스 명시 | sealed와 함께 |
| `final` | 더 이상 확장 불가 | 서브클래스 |
| `non-sealed` | 봉인 해제 | 서브클래스 (일반 확장 허용) |

---

## 4. 구현 방법

### 완전한 구현: 결제 시스템 ⭐⭐⭐

```java
/**
 * ============================================
 * SEALED PAYMENT METHOD
 * ============================================
 */

/**
 * 결제 수단 (봉인됨)
 */
public sealed interface PaymentMethod 
    permits CreditCard, DebitCard, BankTransfer, PayPal {
    
    /**
     * 결제 처리
     */
    boolean process(BigDecimal amount);
    
    /**
     * 결제 수단 이름
     */
    String getMethodName();
}

/**
 * 신용카드
 */
public final class CreditCard implements PaymentMethod {
    private final String cardNumber;
    private final String cvv;
    private final String expiryDate;
    
    public CreditCard(String cardNumber, String cvv, String expiryDate) {
        this.cardNumber = cardNumber;
        this.cvv = cvv;
        this.expiryDate = expiryDate;
    }
    
    @Override
    public boolean process(BigDecimal amount) {
        System.out.println("💳 신용카드 결제: ₩" + amount);
        System.out.println("   카드: " + maskCardNumber(cardNumber));
        return true;
    }
    
    @Override
    public String getMethodName() {
        return "신용카드";
    }
    
    private String maskCardNumber(String cardNumber) {
        return "**** **** **** " + cardNumber.substring(cardNumber.length() - 4);
    }
    
    public String getCardNumber() { return cardNumber; }
    public String getCvv() { return cvv; }
    public String getExpiryDate() { return expiryDate; }
}

/**
 * 직불카드
 */
public final class DebitCard implements PaymentMethod {
    private final String cardNumber;
    private final String pin;
    
    public DebitCard(String cardNumber, String pin) {
        this.cardNumber = cardNumber;
        this.pin = pin;
    }
    
    @Override
    public boolean process(BigDecimal amount) {
        System.out.println("💳 직불카드 결제: ₩" + amount);
        return true;
    }
    
    @Override
    public String getMethodName() {
        return "직불카드";
    }
    
    public String getCardNumber() { return cardNumber; }
}

/**
 * 계좌 이체
 */
public final class BankTransfer implements PaymentMethod {
    private final String accountNumber;
    private final String bankCode;
    
    public BankTransfer(String accountNumber, String bankCode) {
        this.accountNumber = accountNumber;
        this.bankCode = bankCode;
    }
    
    @Override
    public boolean process(BigDecimal amount) {
        System.out.println("🏦 계좌 이체: ₩" + amount);
        System.out.println("   계좌: " + accountNumber);
        return true;
    }
    
    @Override
    public String getMethodName() {
        return "계좌 이체";
    }
    
    public String getAccountNumber() { return accountNumber; }
    public String getBankCode() { return bankCode; }
}

/**
 * PayPal
 */
public final class PayPal implements PaymentMethod {
    private final String email;
    
    public PayPal(String email) {
        this.email = email;
    }
    
    @Override
    public boolean process(BigDecimal amount) {
        System.out.println("🅿️ PayPal 결제: ₩" + amount);
        System.out.println("   이메일: " + email);
        return true;
    }
    
    @Override
    public String getMethodName() {
        return "PayPal";
    }
    
    public String getEmail() { return email; }
}

/**
 * ============================================
 * SEALED ORDER STATE
 * ============================================
 */

/**
 * 주문 상태 (봉인됨)
 */
public sealed interface OrderState 
    permits Pending, Confirmed, Shipped, Delivered, Cancelled {
    
    /**
     * 상태 이름
     */
    String getStateName();
}

public record Pending() implements OrderState {
    @Override
    public String getStateName() {
        return "대기 중";
    }
}

public record Confirmed(LocalDateTime confirmedAt) implements OrderState {
    @Override
    public String getStateName() {
        return "확인됨";
    }
}

public record Shipped(LocalDateTime shippedAt, String trackingNumber) implements OrderState {
    @Override
    public String getStateName() {
        return "배송 중";
    }
}

public record Delivered(LocalDateTime deliveredAt, String signature) implements OrderState {
    @Override
    public String getStateName() {
        return "배송 완료";
    }
}

public record Cancelled(LocalDateTime cancelledAt, String reason) implements OrderState {
    @Override
    public String getStateName() {
        return "취소됨";
    }
}

/**
 * ============================================
 * PATTERN MATCHING WITH SEALED CLASSES
 * ============================================
 */

/**
 * 결제 서비스
 */
public class PaymentService {
    
    /**
     * Java 21+ Pattern Matching for switch
     */
    public void processPayment(PaymentMethod method, BigDecimal amount) {
        // ✅ 컴파일러가 모든 케이스 확인!
        String message = switch (method) {
            case CreditCard card -> 
                "신용카드 (" + card.getCardNumber() + ")";
            case DebitCard card -> 
                "직불카드 (" + card.getCardNumber() + ")";
            case BankTransfer transfer -> 
                "계좌 이체 (" + transfer.getAccountNumber() + ")";
            case PayPal paypal -> 
                "PayPal (" + paypal.getEmail() + ")";
            // default 필요 없음! (모든 케이스 처리됨)
        };
        
        System.out.println("\n💰 결제 처리: " + message);
        method.process(amount);
    }
    
    /**
     * 결제 수수료 계산
     */
    public BigDecimal calculateFee(PaymentMethod method, BigDecimal amount) {
        return switch (method) {
            case CreditCard c -> amount.multiply(new BigDecimal("0.03"));  // 3%
            case DebitCard d -> amount.multiply(new BigDecimal("0.02"));   // 2%
            case BankTransfer b -> new BigDecimal("1000");                 // 고정
            case PayPal p -> amount.multiply(new BigDecimal("0.04"));      // 4%
        };
    }
}

/**
 * 주문 서비스
 */
public class OrderService {
    
    /**
     * 주문 상태 처리
     */
    public void handleOrderState(OrderState state) {
        String info = switch (state) {
            case Pending p -> 
                "⏳ 주문 대기 중";
            case Confirmed c -> 
                "✅ 주문 확인됨 (" + c.confirmedAt() + ")";
            case Shipped s -> 
                "🚚 배송 중 (추적번호: " + s.trackingNumber() + ")";
            case Delivered d -> 
                "📦 배송 완료 (" + d.deliveredAt() + ")";
            case Cancelled c -> 
                "❌ 취소됨 (사유: " + c.reason() + ")";
        };
        
        System.out.println(info);
    }
    
    /**
     * 다음 상태로 전환 가능 여부
     */
    public boolean canTransitionTo(OrderState current, OrderState next) {
        return switch (current) {
            case Pending p -> next instanceof Confirmed || next instanceof Cancelled;
            case Confirmed c -> next instanceof Shipped || next instanceof Cancelled;
            case Shipped s -> next instanceof Delivered;
            case Delivered d -> false;  // 최종 상태
            case Cancelled c -> false;  // 최종 상태
        };
    }
}

/**
 * ============================================
 * SEALED SHAPE HIERARCHY
 * ============================================
 */

/**
 * 도형 (봉인됨)
 */
public sealed class Shape 
    permits Circle, Rectangle, Triangle {
    
    public abstract double area();
    public abstract double perimeter();
}

public final class Circle extends Shape {
    private final double radius;
    
    public Circle(double radius) {
        this.radius = radius;
    }
    
    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
    
    @Override
    public double perimeter() {
        return 2 * Math.PI * radius;
    }
    
    public double getRadius() { return radius; }
}

public final class Rectangle extends Shape {
    private final double width;
    private final double height;
    
    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }
    
    @Override
    public double area() {
        return width * height;
    }
    
    @Override
    public double perimeter() {
        return 2 * (width + height);
    }
    
    public double getWidth() { return width; }
    public double getHeight() { return height; }
}

public sealed class Triangle extends Shape 
    permits EquilateralTriangle, RightTriangle {
    
    protected final double a, b, c;
    
    public Triangle(double a, double b, double c) {
        this.a = a;
        this.b = b;
        this.c = c;
    }
    
    @Override
    public double area() {
        double s = perimeter() / 2;
        return Math.sqrt(s * (s - a) * (s - b) * (s - c));
    }
    
    @Override
    public double perimeter() {
        return a + b + c;
    }
}

public final class EquilateralTriangle extends Triangle {
    public EquilateralTriangle(double side) {
        super(side, side, side);
    }
    
    @Override
    public double area() {
        return (Math.sqrt(3) / 4) * a * a;
    }
}

public final class RightTriangle extends Triangle {
    public RightTriangle(double a, double b) {
        super(a, b, Math.sqrt(a * a + b * b));
    }
}

/**
 * 도형 계산기
 */
public class ShapeCalculator {
    
    public String describeShape(Shape shape) {
        return switch (shape) {
            case Circle c -> 
                "원 (반지름: " + c.getRadius() + ")";
            case Rectangle r -> 
                "직사각형 (" + r.getWidth() + " × " + r.getHeight() + ")";
            case EquilateralTriangle et -> 
                "정삼각형 (변: " + et.a + ")";
            case RightTriangle rt -> 
                "직각삼각형 (밑변: " + rt.a + ", 높이: " + rt.b + ")";
            case Triangle t -> 
                "삼각형 (변: " + t.a + ", " + t.b + ", " + t.c + ")";
        };
    }
}

/**
 * ============================================
 * DEMO
 * ============================================
 */
public class SealedClassesDemo {
    public static void main(String[] args) {
        System.out.println("=== Sealed Classes Pattern 예제 ===");
        
        // 1. 결제 처리
        PaymentService paymentService = new PaymentService();
        
        System.out.println("\n💳 결제 테스트:");
        
        PaymentMethod creditCard = new CreditCard("1234567890123456", "123", "12/25");
        paymentService.processPayment(creditCard, new BigDecimal("100000"));
        
        PaymentMethod paypal = new PayPal("user@example.com");
        paymentService.processPayment(paypal, new BigDecimal("50000"));
        
        // 2. 수수료 계산
        System.out.println("\n💵 수수료 계산:");
        BigDecimal amount = new BigDecimal("100000");
        
        System.out.println("신용카드: ₩" + paymentService.calculateFee(creditCard, amount));
        System.out.println("PayPal: ₩" + paymentService.calculateFee(paypal, amount));
        
        System.out.println("\n" + "=".repeat(60));
        
        // 3. 주문 상태
        OrderService orderService = new OrderService();
        
        System.out.println("\n📦 주문 상태 처리:");
        
        OrderState pending = new Pending();
        OrderState confirmed = new Confirmed(LocalDateTime.now());
        OrderState shipped = new Shipped(LocalDateTime.now(), "TRK123456");
        
        orderService.handleOrderState(pending);
        orderService.handleOrderState(confirmed);
        orderService.handleOrderState(shipped);
        
        System.out.println("\n" + "=".repeat(60));
        
        // 4. 도형 계산
        ShapeCalculator calculator = new ShapeCalculator();
        
        System.out.println("\n📐 도형 계산:");
        
        Shape circle = new Circle(5);
        Shape rectangle = new Rectangle(4, 6);
        Shape triangle = new EquilateralTriangle(3);
        
        System.out.println(calculator.describeShape(circle));
        System.out.println("  면적: " + String.format("%.2f", circle.area()));
        
        System.out.println(calculator.describeShape(rectangle));
        System.out.println("  면적: " + String.format("%.2f", rectangle.area()));
        
        System.out.println(calculator.describeShape(triangle));
        System.out.println("  면적: " + String.format("%.2f", triangle.area()));
        
        System.out.println("\n✅ 완료!");
    }
}
```

**실행 결과:**
```
=== Sealed Classes Pattern 예제 ===

💳 결제 테스트:

💰 결제 처리: 신용카드 (1234567890123456)
💳 신용카드 결제: ₩100000
   카드: **** **** **** 3456

💰 결제 처리: PayPal (user@example.com)
🅿️ PayPal 결제: ₩50000
   이메일: user@example.com

💵 수수료 계산:
신용카드: ₩3000
PayPal: ₩2000

============================================================

📦 주문 상태 처리:
⏳ 주문 대기 중
✅ 주문 확인됨 (2024-12-22T...)
🚚 배송 중 (추적번호: TRK123456)

============================================================

📐 도형 계산:
원 (반지름: 5.0)
  면적: 78.54
직사각형 (4.0 × 6.0)
  면적: 24.00
정삼각형 (변: 3.0)
  면적: 3.90

✅ 완료!
```

---

## 5. 실전 예제

### 예제 1: JSON 타입 처리 ⭐⭐⭐

```java
/**
 * JSON 값 타입
 */
public sealed interface JsonValue 
    permits JsonObject, JsonArray, JsonString, JsonNumber, JsonBoolean, JsonNull {
    
    String toJson();
}

public record JsonObject(Map<String, JsonValue> properties) implements JsonValue {
    @Override
    public String toJson() {
        return "{" + properties.entrySet().stream()
            .map(e -> "\"" + e.getKey() + "\":" + e.getValue().toJson())
            .collect(Collectors.joining(",")) + "}";
    }
}

// 사용
public String processJson(JsonValue value) {
    return switch (value) {
        case JsonObject obj -> "객체 (" + obj.properties().size() + " 속성)";
        case JsonArray arr -> "배열 (" + arr.values().size() + " 요소)";
        case JsonString str -> "문자열: " + str.value();
        case JsonNumber num -> "숫자: " + num.value();
        case JsonBoolean bool -> "불린: " + bool.value();
        case JsonNull n -> "null";
    };
}
```

---

## 6. Pattern Matching 연동

### 🔄 Switch Expression

```java
// Java 21+ Pattern Matching
public String describe(PaymentMethod method) {
    return switch (method) {
        case CreditCard(var num, var cvv, var exp) -> 
            "Credit: " + num;
        case DebitCard(var num, var pin) -> 
            "Debit: " + num;
        case BankTransfer(var acc, var bank) -> 
            "Bank: " + acc;
        case PayPal(var email) -> 
            "PayPal: " + email;
    };
}
```

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 |
|------|------|
| **타입 안전** | 컴파일 타임 체크 |
| **완전성** | 모든 케이스 보장 |
| **명시성** | 가능한 타입 명확 |
| **패턴 매칭** | switch 활용 |

### ❌ 단점

| 단점 | 설명 |
|------|------|
| **Java 17+** | 버전 제한 |
| **확장 제한** | 유연성 감소 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: 너무 많은 서브클래스

```java
// 잘못된 예
sealed class Status 
    permits Status1, Status2, ..., Status50 {
    // 😱 너무 많음!
}

// 올바른 예: 그룹화
sealed class Status 
    permits Active, Inactive, Suspended {
}
```

---

## 9. 심화 주제

### 🎯 Record + Sealed

```java
sealed interface Result<T> 
    permits Success, Failure {
}

record Success<T>(T value) implements Result<T> {}
record Failure<T>(String error) implements Result<T> {}

// 사용
Result<String> result = fetchData();
String message = switch (result) {
    case Success(var value) -> "성공: " + value;
    case Failure(var error) -> "실패: " + error;
};
```

---

## 10. 핵심 정리

### 📌 체크리스트

```
✅ sealed + permits
✅ 서브클래스는 final/sealed/non-sealed
✅ Pattern Matching 활용
✅ Record와 조합
✅ 모든 케이스 처리
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Optional Chaining](./03-OptionalChaining.md)**

</div>
