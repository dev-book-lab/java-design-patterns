# Functional Interface Pattern (함수형 인터페이스 패턴)

> **"메서드를 일급 객체로 취급하여 함수형 프로그래밍을 가능하게 하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [Built-in 함수형 인터페이스](#6-built-in-함수형-인터페이스)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 익명 클래스의 장황함
public class UserService {
    public List<User> filterActiveUsers(List<User> users) {
        // 😱 간단한 필터링에 너무 많은 코드!
        List<User> result = new ArrayList<>();
        for (User user : users) {
            if (user.isActive()) {
                result.add(user);
            }
        }
        return result;
    }
    
    public List<User> filterAdultUsers(List<User> users) {
        // 😱 거의 똑같은 코드 반복!
        List<User> result = new ArrayList<>();
        for (User user : users) {
            if (user.getAge() >= 18) {
                result.add(user);
            }
        }
        return result;
    }
    
    // 필터 조건마다 메서드 생성?
    // → 코드 폭발!
}

// 문제 2: 전략 패턴의 복잡함
public interface ValidationStrategy {
    boolean validate(String input);
}

public class EmailValidator implements ValidationStrategy {
    @Override
    public boolean validate(String input) {
        return input.contains("@");
    }
}

public class LengthValidator implements ValidationStrategy {
    @Override
    public boolean validate(String input) {
        return input.length() >= 8;
    }
}

// 😱 간단한 검증 로직인데 클래스 2개 필요!
// 사용:
ValidationStrategy emailValidator = new EmailValidator();
ValidationStrategy lengthValidator = new LengthValidator();

// 문제 3: 콜백 패턴의 장황함
public class AsyncService {
    public void processData(DataCallback callback) {
        // 비동기 처리
        new Thread(new Runnable() {  // 😱 익명 클래스!
            @Override
            public void run() {
                String result = "processed";
                callback.onSuccess(result);
            }
        }).start();
    }
}

interface DataCallback {
    void onSuccess(String result);
    void onError(Exception e);
}

// 사용:
service.processData(new DataCallback() {  // 😱 너무 장황!
    @Override
    public void onSuccess(String result) {
        System.out.println(result);
    }
    
    @Override
    public void onError(Exception e) {
        e.printStackTrace();
    }
});

// 문제 4: 정렬 비교자
List<Product> products = getProducts();

// 😱 간단한 정렬인데 너무 복잡!
Collections.sort(products, new Comparator<Product>() {
    @Override
    public int compare(Product p1, Product p2) {
        return p1.getPrice().compareTo(p2.getPrice());
    }
});

// 문제 5: 이벤트 리스너
button.addActionListener(new ActionListener() {  // 😱
    @Override
    public void actionPerformed(ActionEvent e) {
        System.out.println("Button clicked!");
    }
});

// 문제 6: 템플릿 메서드 패턴
public abstract class DataProcessor {
    public final void process() {
        open();
        doProcess();  // 상속받아서 구현
        close();
    }
    
    protected abstract void doProcess();
}

// 😱 간단한 로직인데 상속 필요!
public class CsvProcessor extends DataProcessor {
    @Override
    protected void doProcess() {
        System.out.println("Process CSV");
    }
}
```

### ⚡ 핵심 문제

1. **장황함**: 익명 클래스가 너무 김
2. **중복**: 비슷한 코드 반복
3. **가독성**: 의도가 명확하지 않음
4. **재사용 어려움**: 로직을 변수로 전달 불가
5. **테스트 복잡**: 간단한 로직도 클래스 필요

---

## 2. 패턴 정의

### 📖 정의

> **단 하나의 추상 메서드를 가진 인터페이스로, Lambda 표현식을 통해 간결하게 구현할 수 있는 패턴**

### 🎯 목적

- **간결성**: 익명 클래스 대신 Lambda
- **일급 함수**: 함수를 변수처럼 전달
- **재사용성**: 로직을 쉽게 재사용
- **가독성**: 의도가 명확한 코드

### 💡 핵심 아이디어

```java
// Before: 익명 클래스 (장황함)
button.addActionListener(new ActionListener() {
    @Override
    public void actionPerformed(ActionEvent e) {
        System.out.println("Clicked!");
    }
});

// After: Lambda (간결함)
button.addActionListener(e -> System.out.println("Clicked!"));

// Before: 별도 클래스
public class PriceComparator implements Comparator<Product> {
    @Override
    public int compare(Product p1, Product p2) {
        return p1.getPrice().compareTo(p2.getPrice());
    }
}
products.sort(new PriceComparator());

// After: Lambda
products.sort((p1, p2) -> p1.getPrice().compareTo(p2.getPrice()));
```

---

## 3. 구조와 구성요소

### 📊 Functional Interface 구조

```java
@FunctionalInterface  // ← 선택적 애노테이션
public interface Predicate<T> {
    boolean test(T t);  // ← 단 하나의 추상 메서드 (SAM)
    
    // default 메서드는 여러 개 가능
    default Predicate<T> and(Predicate<T> other) {
        return t -> test(t) && other.test(t);
    }
    
    default Predicate<T> or(Predicate<T> other) {
        return t -> test(t) || other.test(t);
    }
    
    // static 메서드도 가능
    static <T> Predicate<T> isEqual(Object target) {
        return t -> t.equals(target);
    }
}
```

### 🔄 Lambda 표현식 형태

```java
// 1. 매개변수 없음
() -> System.out.println("Hello")

// 2. 매개변수 1개
x -> x * 2
(x) -> x * 2  // 괄호 선택적

// 3. 매개변수 2개 이상
(x, y) -> x + y

// 4. 타입 명시
(Integer x, Integer y) -> x + y

// 5. 여러 문장 (블록)
(x, y) -> {
    int sum = x + y;
    return sum * 2;
}

// 6. 메서드 참조
String::toUpperCase
System.out::println
```

### 🔧 함수형 인터페이스 카테고리

| 카테고리 | 인터페이스 | 메서드 | 용도 |
|---------|-----------|--------|------|
| **Predicate** | `Predicate<T>` | `boolean test(T)` | 조건 검사 |
| **Function** | `Function<T,R>` | `R apply(T)` | 변환 |
| **Consumer** | `Consumer<T>` | `void accept(T)` | 소비 (출력 등) |
| **Supplier** | `Supplier<T>` | `T get()` | 공급 (생성) |
| **BiFunction** | `BiFunction<T,U,R>` | `R apply(T,U)` | 2개 입력 변환 |
| **UnaryOperator** | `UnaryOperator<T>` | `T apply(T)` | 단항 연산 |
| **BinaryOperator** | `BinaryOperator<T>` | `T apply(T,T)` | 이항 연산 |

---

## 4. 구현 방법

### 완전한 구현: E-Commerce 필터링 시스템 ⭐⭐⭐

```java
/**
 * ============================================
 * DOMAIN MODEL
 * ============================================
 */
public class Product {
    private Long id;
    private String name;
    private BigDecimal price;
    private String category;
    private int stock;
    private boolean onSale;
    
    public Product(Long id, String name, BigDecimal price, String category, int stock) {
        this.id = id;
        this.name = name;
        this.price = price;
        this.category = category;
        this.stock = stock;
        this.onSale = false;
    }
    
    // Getters, Setters
    public Long getId() { return id; }
    public String getName() { return name; }
    public BigDecimal getPrice() { return price; }
    public String getCategory() { return category; }
    public int getStock() { return stock; }
    public boolean isOnSale() { return onSale; }
    public void setOnSale(boolean onSale) { this.onSale = onSale; }
    
    @Override
    public String toString() {
        return String.format("%s (₩%,d) [%s]", name, price.intValue(), category);
    }
}

/**
 * ============================================
 * CUSTOM FUNCTIONAL INTERFACES
 * ============================================
 */

/**
 * 상품 검증 인터페이스
 */
@FunctionalInterface
public interface ProductPredicate {
    boolean test(Product product);
    
    // default 메서드로 조합 가능
    default ProductPredicate and(ProductPredicate other) {
        return product -> test(product) && other.test(product);
    }
    
    default ProductPredicate or(ProductPredicate other) {
        return product -> test(product) || other.test(product);
    }
    
    default ProductPredicate negate() {
        return product -> !test(product);
    }
}

/**
 * 상품 변환 인터페이스
 */
@FunctionalInterface
public interface ProductMapper<R> {
    R apply(Product product);
}

/**
 * 상품 처리 인터페이스
 */
@FunctionalInterface
public interface ProductConsumer {
    void accept(Product product);
}

/**
 * ============================================
 * PRODUCT SERVICE (함수형 인터페이스 활용)
 * ============================================
 */
public class ProductService {
    private final List<Product> products = new ArrayList<>();
    
    public ProductService() {
        // 테스트 데이터
        products.add(new Product(1L, "노트북", new BigDecimal("1200000"), "전자기기", 10));
        products.add(new Product(2L, "마우스", new BigDecimal("30000"), "액세서리", 50));
        products.add(new Product(3L, "키보드", new BigDecimal("80000"), "액세서리", 30));
        products.add(new Product(4L, "모니터", new BigDecimal("300000"), "전자기기", 5));
        products.add(new Product(5L, "헤드셋", new BigDecimal("150000"), "액세서리", 0));
        
        products.get(1).setOnSale(true);
        products.get(2).setOnSale(true);
    }
    
    /**
     * 필터링 (Predicate 사용)
     */
    public List<Product> filter(ProductPredicate predicate) {
        List<Product> result = new ArrayList<>();
        for (Product product : products) {
            if (predicate.test(product)) {
                result.add(product);
            }
        }
        return result;
    }
    
    /**
     * 변환 (Function 사용)
     */
    public <R> List<R> map(ProductMapper<R> mapper) {
        List<R> result = new ArrayList<>();
        for (Product product : products) {
            result.add(mapper.apply(product));
        }
        return result;
    }
    
    /**
     * 각 상품에 대해 처리 (Consumer 사용)
     */
    public void forEach(ProductConsumer consumer) {
        for (Product product : products) {
            consumer.accept(product);
        }
    }
    
    /**
     * 조건에 맞는 첫 상품 찾기
     */
    public Product findFirst(ProductPredicate predicate) {
        for (Product product : products) {
            if (predicate.test(product)) {
                return product;
            }
        }
        return null;
    }
    
    /**
     * 모든 상품이 조건 만족?
     */
    public boolean allMatch(ProductPredicate predicate) {
        for (Product product : products) {
            if (!predicate.test(product)) {
                return false;
            }
        }
        return true;
    }
    
    /**
     * 하나라도 조건 만족?
     */
    public boolean anyMatch(ProductPredicate predicate) {
        for (Product product : products) {
            if (predicate.test(product)) {
                return true;
            }
        }
        return false;
    }
}

/**
 * ============================================
 * PREDICATE FACTORY (재사용 가능한 조건들)
 * ============================================
 */
public class ProductPredicates {
    
    /**
     * 재고 있는 상품
     */
    public static ProductPredicate hasStock() {
        return product -> product.getStock() > 0;
    }
    
    /**
     * 세일 중인 상품
     */
    public static ProductPredicate onSale() {
        return product -> product.isOnSale();
    }
    
    /**
     * 특정 카테고리
     */
    public static ProductPredicate inCategory(String category) {
        return product -> category.equals(product.getCategory());
    }
    
    /**
     * 가격 범위
     */
    public static ProductPredicate priceBetween(BigDecimal min, BigDecimal max) {
        return product -> {
            BigDecimal price = product.getPrice();
            return price.compareTo(min) >= 0 && price.compareTo(max) <= 0;
        };
    }
    
    /**
     * 최소 재고
     */
    public static ProductPredicate minStock(int minStock) {
        return product -> product.getStock() >= minStock;
    }
}

/**
 * ============================================
 * BUILT-IN FUNCTIONAL INTERFACES 활용
 * ============================================
 */
public class FunctionalInterfaceExamples {
    
    /**
     * 1. Predicate<T> - 조건 검사
     */
    public void predicateExample() {
        System.out.println("\n=== Predicate 예제 ===");
        
        // 짝수 체크
        Predicate<Integer> isEven = n -> n % 2 == 0;
        System.out.println("4는 짝수? " + isEven.test(4));  // true
        System.out.println("5는 짝수? " + isEven.test(5));  // false
        
        // 조합
        Predicate<Integer> isPositive = n -> n > 0;
        Predicate<Integer> isPositiveEven = isEven.and(isPositive);
        System.out.println("-4는 양수이면서 짝수? " + isPositiveEven.test(-4));  // false
        System.out.println("4는 양수이면서 짝수? " + isPositiveEven.test(4));   // true
    }
    
    /**
     * 2. Function<T, R> - 변환
     */
    public void functionExample() {
        System.out.println("\n=== Function 예제 ===");
        
        // 문자열 길이
        Function<String, Integer> length = String::length;
        System.out.println("'Hello'의 길이: " + length.apply("Hello"));  // 5
        
        // 체이닝
        Function<String, String> trim = String::trim;
        Function<String, Integer> trimAndLength = trim.andThen(length);
        System.out.println("'  Hello  ' trim 후 길이: " + trimAndLength.apply("  Hello  "));  // 5
    }
    
    /**
     * 3. Consumer<T> - 소비 (반환 없음)
     */
    public void consumerExample() {
        System.out.println("\n=== Consumer 예제 ===");
        
        Consumer<String> print = System.out::println;
        Consumer<String> upperPrint = s -> System.out.println(s.toUpperCase());
        
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
        
        System.out.println("일반 출력:");
        names.forEach(print);
        
        System.out.println("\n대문자 출력:");
        names.forEach(upperPrint);
        
        // 조합
        Consumer<String> printTwice = print.andThen(print);
        System.out.println("\n두 번 출력:");
        printTwice.accept("Hello");
    }
    
    /**
     * 4. Supplier<T> - 공급 (입력 없음)
     */
    public void supplierExample() {
        System.out.println("\n=== Supplier 예제 ===");
        
        Supplier<Double> randomValue = Math::random;
        System.out.println("랜덤 값: " + randomValue.get());
        
        Supplier<LocalDateTime> now = LocalDateTime::now;
        System.out.println("현재 시간: " + now.get());
        
        // Lazy Evaluation
        Supplier<String> expensiveOperation = () -> {
            System.out.println("  → 비싼 연산 실행!");
            return "결과";
        };
        
        System.out.println("Supplier 생성 (아직 실행 안 됨)");
        System.out.println("get() 호출:");
        System.out.println(expensiveOperation.get());
    }
    
    /**
     * 5. BiFunction<T, U, R> - 2개 입력, 1개 출력
     */
    public void biFunctionExample() {
        System.out.println("\n=== BiFunction 예제 ===");
        
        BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;
        BiFunction<Integer, Integer, Integer> multiply = (a, b) -> a * b;
        
        System.out.println("10 + 5 = " + add.apply(10, 5));
        System.out.println("10 * 5 = " + multiply.apply(10, 5));
        
        // andThen
        BiFunction<Integer, Integer, Integer> addAndDouble = 
            add.andThen(result -> result * 2);
        System.out.println("(10 + 5) * 2 = " + addAndDouble.apply(10, 5));
    }
    
    /**
     * 6. UnaryOperator<T> - 입력과 출력 타입 동일
     */
    public void unaryOperatorExample() {
        System.out.println("\n=== UnaryOperator 예제 ===");
        
        UnaryOperator<Integer> square = x -> x * x;
        UnaryOperator<Integer> increment = x -> x + 1;
        
        System.out.println("5의 제곱: " + square.apply(5));
        
        // 조합
        UnaryOperator<Integer> squareAndIncrement = square.andThen(increment);
        System.out.println("5를 제곱 후 +1: " + squareAndIncrement.apply(5));  // 26
    }
    
    /**
     * 7. BinaryOperator<T> - 2개 입력, 같은 타입 출력
     */
    public void binaryOperatorExample() {
        System.out.println("\n=== BinaryOperator 예제 ===");
        
        BinaryOperator<Integer> max = (a, b) -> a > b ? a : b;
        BinaryOperator<Integer> min = (a, b) -> a < b ? a : b;
        
        System.out.println("max(10, 20) = " + max.apply(10, 20));
        System.out.println("min(10, 20) = " + min.apply(10, 20));
        
        // BinaryOperator.maxBy / minBy
        BinaryOperator<String> longest = BinaryOperator.maxBy(
            Comparator.comparingInt(String::length)
        );
        System.out.println("더 긴 문자열: " + longest.apply("Hello", "World!"));
    }
}

/**
 * ============================================
 * DEMO
 * ============================================
 */
public class FunctionalInterfaceDemo {
    public static void main(String[] args) {
        System.out.println("=== Functional Interface Pattern 예제 ===");
        
        ProductService service = new ProductService();
        
        // 1. 재고 있는 상품 (Lambda)
        System.out.println("\n📦 재고 있는 상품:");
        List<Product> inStock = service.filter(p -> p.getStock() > 0);
        inStock.forEach(p -> System.out.println("   " + p));
        
        System.out.println("\n" + "=".repeat(60));
        
        // 2. 세일 상품 (메서드 참조)
        System.out.println("\n💰 세일 중인 상품:");
        List<Product> onSale = service.filter(ProductPredicates.onSale());
        onSale.forEach(p -> System.out.println("   " + p));
        
        System.out.println("\n" + "=".repeat(60));
        
        // 3. 조합 (재고 있으면서 세일)
        System.out.println("\n✨ 재고 있으면서 세일 중:");
        List<Product> stockAndSale = service.filter(
            ProductPredicates.hasStock().and(ProductPredicates.onSale())
        );
        stockAndSale.forEach(p -> System.out.println("   " + p));
        
        System.out.println("\n" + "=".repeat(60));
        
        // 4. 복잡한 조합
        System.out.println("\n🎯 전자기기 또는 (액세서리이면서 세일):");
        ProductPredicate complex = ProductPredicates.inCategory("전자기기")
            .or(ProductPredicates.inCategory("액세서리").and(ProductPredicates.onSale()));
        
        List<Product> complexResult = service.filter(complex);
        complexResult.forEach(p -> System.out.println("   " + p));
        
        System.out.println("\n" + "=".repeat(60));
        
        // 5. 변환 (상품 이름만 추출)
        System.out.println("\n📝 모든 상품 이름:");
        List<String> names = service.map(Product::getName);
        names.forEach(name -> System.out.println("   - " + name));
        
        System.out.println("\n" + "=".repeat(60));
        
        // 6. 가격 인상 (Consumer)
        System.out.println("\n💵 가격 10% 인상:");
        service.forEach(p -> {
            BigDecimal newPrice = p.getPrice().multiply(new BigDecimal("1.1"));
            System.out.println("   " + p.getName() + ": " + 
                p.getPrice() + " → " + newPrice.intValue());
        });
        
        System.out.println("\n" + "=".repeat(60));
        
        // 7. Built-in 함수형 인터페이스 예제
        FunctionalInterfaceExamples examples = new FunctionalInterfaceExamples();
        examples.predicateExample();
        examples.functionExample();
        examples.consumerExample();
        examples.supplierExample();
        examples.biFunctionExample();
        examples.unaryOperatorExample();
        examples.binaryOperatorExample();
        
        System.out.println("\n✅ 완료!");
    }
}
```

**실행 결과:**
```
=== Functional Interface Pattern 예제 ===

📦 재고 있는 상품:
   노트북 (₩1,200,000) [전자기기]
   마우스 (₩30,000) [액세서리]
   키보드 (₩80,000) [액세서리]
   모니터 (₩300,000) [전자기기]

============================================================

💰 세일 중인 상품:
   마우스 (₩30,000) [액세서리]
   키보드 (₩80,000) [액세서리]

============================================================

✨ 재고 있으면서 세일 중:
   마우스 (₩30,000) [액세서리]
   키보드 (₩80,000) [액세서리]

============================================================

🎯 전자기기 또는 (액세서리이면서 세일):
   노트북 (₩1,200,000) [전자기기]
   마우스 (₩30,000) [액세서리]
   키보드 (₩80,000) [액세서리]
   모니터 (₩300,000) [전자기기]

============================================================

📝 모든 상품 이름:
   - 노트북
   - 마우스
   - 키보드
   - 모니터
   - 헤드셋

============================================================

💵 가격 10% 인상:
   노트북: 1200000 → 1320000
   마우스: 30000 → 33000
   키보드: 80000 → 88000
   모니터: 300000 → 330000
   헤드셋: 150000 → 165000

============================================================

=== Predicate 예제 ===
4는 짝수? true
5는 짝수? false
-4는 양수이면서 짝수? false
4는 양수이면서 짝수? true

=== Function 예제 ===
'Hello'의 길이: 5
'  Hello  ' trim 후 길이: 5

=== Consumer 예제 ===
일반 출력:
Alice
Bob
Charlie

대문자 출력:
ALICE
BOB
CHARLIE

두 번 출력:
Hello
Hello

=== Supplier 예제 ===
랜덤 값: 0.123...
현재 시간: 2024-12-22T...
Supplier 생성 (아직 실행 안 됨)
get() 호출:
  → 비싼 연산 실행!
결과

=== BiFunction 예제 ===
10 + 5 = 15
10 * 5 = 50
(10 + 5) * 2 = 30

=== UnaryOperator 예제 ===
5의 제곱: 25
5를 제곱 후 +1: 26

=== BinaryOperator 예제 ===
max(10, 20) = 20
min(10, 20) = 10
더 긴 문자열: World!

✅ 완료!
```

---

## 5. 실전 예제

### 예제 1: 전략 패턴 대체 ⭐⭐⭐

```java
/**
 * Before: 클래스 기반 전략 패턴
 */
interface DiscountStrategy {
    BigDecimal apply(BigDecimal price);
}

class NoDiscount implements DiscountStrategy {
    public BigDecimal apply(BigDecimal price) {
        return price;
    }
}

class TenPercentDiscount implements DiscountStrategy {
    public BigDecimal apply(BigDecimal price) {
        return price.multiply(new BigDecimal("0.9"));
    }
}

/**
 * After: 함수형 인터페이스
 */
@FunctionalInterface
interface DiscountFunction {
    BigDecimal apply(BigDecimal price);
}

// 사용
DiscountFunction noDiscount = price -> price;
DiscountFunction tenPercent = price -> price.multiply(new BigDecimal("0.9"));
DiscountFunction seasonal = price -> price.multiply(new BigDecimal("0.8"));

// 조합
DiscountFunction vipDiscount = tenPercent.andThen(
    price -> price.multiply(new BigDecimal("0.95"))
);
```

---

## 6. Built-in 함수형 인터페이스

### 📊 주요 인터페이스 요약

```java
// 1. Predicate<T>
Predicate<String> isEmpty = String::isEmpty;
Predicate<Integer> isEven = n -> n % 2 == 0;

// 2. Function<T, R>
Function<String, Integer> length = String::length;
Function<Integer, String> toString = Object::toString;

// 3. Consumer<T>
Consumer<String> print = System.out::println;
Consumer<List<String>> clear = List::clear;

// 4. Supplier<T>
Supplier<Double> random = Math::random;
Supplier<UUID> uuid = UUID::randomUUID;

// 5. BiPredicate<T, U>
BiPredicate<String, String> equals = String::equals;

// 6. BiConsumer<T, U>
BiConsumer<String, Integer> printPair = (s, i) -> 
    System.out.println(s + ": " + i);

// 7. Primitive 특화
IntPredicate isPositive = n -> n > 0;
IntFunction<String> intToString = Integer::toString;
IntConsumer printInt = System.out::println;
IntSupplier randomInt = () -> (int)(Math.random() * 100);
```

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 |
|------|------|
| **간결성** | 익명 클래스 대비 간결 |
| **가독성** | 의도가 명확 |
| **재사용** | 함수를 변수로 전달 |
| **조합** | 함수 조합 가능 |

### ❌ 단점

| 단점 | 설명 |
|------|------|
| **디버깅** | 스택 트레이스 복잡 |
| **학습 곡선** | 함수형 사고 필요 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: 복잡한 Lambda

```java
// 잘못된 예
Function<String, String> complex = s -> {
    // 😱 Lambda가 너무 길고 복잡!
    String result = s.trim();
    if (result.isEmpty()) {
        return "empty";
    }
    result = result.toUpperCase();
    if (result.length() > 10) {
        result = result.substring(0, 10);
    }
    return result + "...";
};
```

**해결:**
```java
// 올바른 예: 별도 메서드로
private String processString(String s) {
    String result = s.trim();
    if (result.isEmpty()) return "empty";
    result = result.toUpperCase();
    if (result.length() > 10) {
        result = result.substring(0, 10);
    }
    return result + "...";
}

Function<String, String> processor = this::processString;
```

---

## 9. 심화 주제

### 🎯 메서드 참조 (Method Reference)

```java
// 1. 정적 메서드 참조
Function<String, Integer> parseInt = Integer::parseInt;

// 2. 인스턴스 메서드 참조
String str = "Hello";
Supplier<String> upperCase = str::toUpperCase;

// 3. 임의 객체의 인스턴스 메서드 참조
Function<String, String> toUpper = String::toUpperCase;

// 4. 생성자 참조
Supplier<ArrayList<String>> listFactory = ArrayList::new;
Function<Integer, int[]> arrayFactory = int[]::new;
```

---

## 10. 핵심 정리

### 📌 체크리스트

```
✅ 단 하나의 추상 메서드
✅ @FunctionalInterface 애노테이션
✅ Lambda로 간결하게
✅ 메서드 참조 활용
✅ Built-in 인터페이스 우선 사용
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[다음: Stream Pipeline →](./02-StreamPipeline.md)**

</div>
