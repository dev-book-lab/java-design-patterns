# Stream Pipeline Pattern (스트림 파이프라인 패턴)

> **"데이터를 선언적으로 처리하는 파이프라인을 만들자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [성능 최적화](#6-성능-최적화)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 중첩된 반복문
List<Order> orders = getOrders();
List<String> result = new ArrayList<>();

// 😱 복잡한 중첩 루프
for (Order order : orders) {
    if (order.getStatus() == OrderStatus.COMPLETED) {
        for (OrderItem item : order.getItems()) {
            if (item.getPrice().compareTo(new BigDecimal("10000")) > 0) {
                result.add(item.getProductName());
            }
        }
    }
}

// 문제: 의도가 불명확, 읽기 어려움

// 문제 2: 중간 변수 남발
List<Product> products = getProducts();

// 😱 단계마다 중간 리스트
List<Product> activeProducts = new ArrayList<>();
for (Product p : products) {
    if (p.isActive()) {
        activeProducts.add(p);
    }
}

List<Product> affordableProducts = new ArrayList<>();
for (Product p : activeProducts) {
    if (p.getPrice().compareTo(new BigDecimal("100000")) < 0) {
        affordableProducts.add(p);
    }
}

List<String> names = new ArrayList<>();
for (Product p : affordableProducts) {
    names.add(p.getName());
}

// 중간 변수가 3개나!

// 문제 3: 가독성 부족
// 😱 이게 뭘 하는 코드인지?
List<Integer> result = new ArrayList<>();
for (User user : users) {
    if (user.getAge() >= 18 && user.isActive()) {
        for (Order order : user.getOrders()) {
            if (order.getTotal().compareTo(new BigDecimal("50000")) >= 0) {
                result.add(order.getId());
            }
        }
    }
}

// 문제 4: 수동 그룹핑
Map<String, List<Product>> byCategory = new HashMap<>();

for (Product product : products) {
    String category = product.getCategory();
    
    if (!byCategory.containsKey(category)) {
        byCategory.put(category, new ArrayList<>());
    }
    
    byCategory.get(category).add(product);
}

// 😱 간단한 그룹핑인데 너무 복잡!

// 문제 5: 집계 연산
BigDecimal total = BigDecimal.ZERO;
int count = 0;

for (Order order : orders) {
    if (order.getStatus() == OrderStatus.COMPLETED) {
        total = total.add(order.getTotal());
        count++;
    }
}

BigDecimal average = count > 0 ? total.divide(BigDecimal.valueOf(count)) : BigDecimal.ZERO;

// 😱 평균 계산인데 코드가 김

// 문제 6: 정렬과 제한
List<Product> topProducts = new ArrayList<>(products);

// 정렬
Collections.sort(topProducts, new Comparator<Product>() {
    @Override
    public int compare(Product p1, Product p2) {
        return p2.getPrice().compareTo(p1.getPrice());
    }
});

// 상위 5개만
List<Product> top5 = new ArrayList<>();
for (int i = 0; i < Math.min(5, topProducts.size()); i++) {
    top5.add(topProducts.get(i));
}

// 😱 정렬 + 제한인데 복잡!

// 문제 7: 조건부 처리
List<String> names = new ArrayList<>();

for (User user : users) {
    if (user.isActive()) {
        String name = user.getName();
        if (name != null && !name.isEmpty()) {
            names.add(name.toUpperCase());
        }
    }
}

// 중첩된 if문으로 복잡

// 문제 8: 중복 제거
Set<String> uniqueCategories = new HashSet<>();

for (Product product : products) {
    uniqueCategories.add(product.getCategory());
}

List<String> result = new ArrayList<>(uniqueCategories);

// 중복 제거하려고 Set 거쳐야 함
```

### ⚡ 핵심 문제

1. **명령형**: "어떻게(How)" 구현할지 작성
2. **가독성**: 의도가 명확하지 않음
3. **중간 상태**: 불필요한 중간 변수
4. **장황함**: 간단한 작업도 코드 많음
5. **에러 발생**: 수동 인덱스 관리 위험

---

## 2. 패턴 정의

### 📖 정의

> **데이터 소스에서 일련의 연산을 체이닝하여 선언적으로 데이터를 처리하는 패턴**

### 🎯 목적

- **선언적**: "무엇을(What)" 할지 작성
- **간결성**: 코드가 짧고 명확
- **조합성**: 연산을 체이닝
- **Lazy**: 필요한 만큼만 처리

### 💡 핵심 아이디어

```java
// Before: 명령형
List<String> result = new ArrayList<>();
for (Order order : orders) {
    if (order.getStatus() == OrderStatus.COMPLETED) {
        if (order.getTotal().compareTo(new BigDecimal("50000")) >= 0) {
            result.add(order.getCustomerName());
        }
    }
}

// After: 선언형 (Stream)
List<String> result = orders.stream()
    .filter(order -> order.getStatus() == OrderStatus.COMPLETED)
    .filter(order -> order.getTotal().compareTo(new BigDecimal("50000")) >= 0)
    .map(Order::getCustomerName)
    .collect(Collectors.toList());

// 의도가 명확: 완료된 주문 중 5만원 이상인 고객 이름
```

---

## 3. 구조와 구성요소

### 📊 Stream Pipeline 구조

```
Source → Intermediate Operations → Terminal Operation
(소스)    (중간 연산)                (최종 연산)

예시:
orders.stream()
    ↓
  .filter(...)  ← 중간 연산
    ↓
  .map(...)     ← 중간 연산
    ↓
  .sorted()     ← 중간 연산
    ↓
  .collect(...) ← 최종 연산
    ↓
  Result
```

### 🔄 연산 종류

| 카테고리 | 연산 | 설명 | 예시 |
|---------|------|------|------|
| **중간 연산** | `filter` | 조건 필터링 | `filter(p -> p.getPrice() > 1000)` |
| | `map` | 변환 | `map(User::getName)` |
| | `flatMap` | 평탄화 | `flatMap(Order::getItems)` |
| | `sorted` | 정렬 | `sorted(Comparator.comparing(Product::getPrice))` |
| | `distinct` | 중복 제거 | `distinct()` |
| | `limit` | 개수 제한 | `limit(10)` |
| | `skip` | 건너뛰기 | `skip(5)` |
| **최종 연산** | `collect` | 수집 | `collect(Collectors.toList())` |
| | `forEach` | 각 요소 처리 | `forEach(System.out::println)` |
| | `reduce` | 리듀스 | `reduce(0, Integer::sum)` |
| | `count` | 개수 | `count()` |
| | `anyMatch` | 하나라도 매치 | `anyMatch(p -> p.getPrice() > 1000)` |
| | `allMatch` | 모두 매치 | `allMatch(Product::isActive)` |
| | `findFirst` | 첫 요소 | `findFirst()` |

---

## 4. 구현 방법

### 완전한 구현: E-Commerce 데이터 처리 ⭐⭐⭐

```java
/**
 * ============================================
 * DOMAIN MODELS
 * ============================================
 */
public class Order {
    private Long id;
    private String customerName;
    private BigDecimal total;
    private OrderStatus status;
    private List<OrderItem> items;
    private LocalDateTime createdAt;
    
    public enum OrderStatus {
        PENDING, CONFIRMED, COMPLETED, CANCELLED
    }
    
    public Order(Long id, String customerName, BigDecimal total, OrderStatus status) {
        this.id = id;
        this.customerName = customerName;
        this.total = total;
        this.status = status;
        this.items = new ArrayList<>();
        this.createdAt = LocalDateTime.now();
    }
    
    public void addItem(OrderItem item) {
        items.add(item);
    }
    
    // Getters
    public Long getId() { return id; }
    public String getCustomerName() { return customerName; }
    public BigDecimal getTotal() { return total; }
    public OrderStatus getStatus() { return status; }
    public List<OrderItem> getItems() { return items; }
    public LocalDateTime getCreatedAt() { return createdAt; }
}

public class OrderItem {
    private String productName;
    private BigDecimal price;
    private int quantity;
    
    public OrderItem(String productName, BigDecimal price, int quantity) {
        this.productName = productName;
        this.price = price;
        this.quantity = quantity;
    }
    
    public BigDecimal getSubtotal() {
        return price.multiply(BigDecimal.valueOf(quantity));
    }
    
    public String getProductName() { return productName; }
    public BigDecimal getPrice() { return price; }
    public int getQuantity() { return quantity; }
}

public class Product {
    private Long id;
    private String name;
    private BigDecimal price;
    private String category;
    private int salesCount;
    
    public Product(Long id, String name, BigDecimal price, String category, int salesCount) {
        this.id = id;
        this.name = name;
        this.price = price;
        this.category = category;
        this.salesCount = salesCount;
    }
    
    public Long getId() { return id; }
    public String getName() { return name; }
    public BigDecimal getPrice() { return price; }
    public String getCategory() { return category; }
    public int getSalesCount() { return salesCount; }
}

/**
 * ============================================
 * STREAM PIPELINE EXAMPLES
 * ============================================
 */
public class StreamPipelineExamples {
    private final List<Order> orders;
    private final List<Product> products;
    
    public StreamPipelineExamples() {
        this.orders = createTestOrders();
        this.products = createTestProducts();
    }
    
    /**
     * 1. 기본 필터링과 매핑
     */
    public void basicFilterAndMap() {
        System.out.println("\n=== 기본 필터링과 매핑 ===");
        
        // 완료된 주문의 고객 이름
        List<String> customerNames = orders.stream()
            .filter(order -> order.getStatus() == Order.OrderStatus.COMPLETED)
            .map(Order::getCustomerName)
            .collect(Collectors.toList());
        
        System.out.println("완료된 주문 고객: " + customerNames);
    }
    
    /**
     * 2. 정렬과 제한
     */
    public void sortAndLimit() {
        System.out.println("\n=== 정렬과 제한 ===");
        
        // 가격 높은 순으로 상위 3개
        List<Product> top3 = products.stream()
            .sorted(Comparator.comparing(Product::getPrice).reversed())
            .limit(3)
            .collect(Collectors.toList());
        
        System.out.println("Top 3 상품:");
        top3.forEach(p -> System.out.println("  " + p.getName() + ": " + p.getPrice()));
    }
    
    /**
     * 3. flatMap (평탄화)
     */
    public void flatMapExample() {
        System.out.println("\n=== flatMap ===");
        
        // 모든 주문의 모든 아이템
        List<OrderItem> allItems = orders.stream()
            .flatMap(order -> order.getItems().stream())
            .collect(Collectors.toList());
        
        System.out.println("전체 아이템 수: " + allItems.size());
        
        // 모든 상품명 (중복 제거)
        List<String> productNames = orders.stream()
            .flatMap(order -> order.getItems().stream())
            .map(OrderItem::getProductName)
            .distinct()
            .collect(Collectors.toList());
        
        System.out.println("주문된 상품: " + productNames);
    }
    
    /**
     * 4. 집계 (reduce)
     */
    public void reduceExample() {
        System.out.println("\n=== reduce (집계) ===");
        
        // 총 매출액
        BigDecimal totalSales = orders.stream()
            .filter(order -> order.getStatus() == Order.OrderStatus.COMPLETED)
            .map(Order::getTotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        
        System.out.println("총 매출: ₩" + totalSales);
        
        // 최고가 상품
        Optional<Product> mostExpensive = products.stream()
            .reduce((p1, p2) -> p1.getPrice().compareTo(p2.getPrice()) > 0 ? p1 : p2);
        
        mostExpensive.ifPresent(p -> 
            System.out.println("최고가 상품: " + p.getName() + " (₩" + p.getPrice() + ")"));
    }
    
    /**
     * 5. 그룹핑
     */
    public void groupingExample() {
        System.out.println("\n=== 그룹핑 ===");
        
        // 카테고리별 상품 개수
        Map<String, Long> countByCategory = products.stream()
            .collect(Collectors.groupingBy(
                Product::getCategory,
                Collectors.counting()
            ));
        
        System.out.println("카테고리별 상품 수:");
        countByCategory.forEach((category, count) -> 
            System.out.println("  " + category + ": " + count));
        
        // 상태별 주문 그룹핑
        Map<Order.OrderStatus, List<Order>> byStatus = orders.stream()
            .collect(Collectors.groupingBy(Order::getStatus));
        
        System.out.println("\n상태별 주문:");
        byStatus.forEach((status, orderList) -> 
            System.out.println("  " + status + ": " + orderList.size() + "건"));
    }
    
    /**
     * 6. 파티셔닝
     */
    public void partitioningExample() {
        System.out.println("\n=== 파티셔닝 ===");
        
        // 10만원 기준으로 분류
        Map<Boolean, List<Product>> partitioned = products.stream()
            .collect(Collectors.partitioningBy(
                p -> p.getPrice().compareTo(new BigDecimal("100000")) >= 0
            ));
        
        System.out.println("10만원 이상: " + partitioned.get(true).size() + "개");
        System.out.println("10만원 미만: " + partitioned.get(false).size() + "개");
    }
    
    /**
     * 7. 통계
     */
    public void statisticsExample() {
        System.out.println("\n=== 통계 ===");
        
        // 가격 통계
        IntSummaryStatistics priceStats = products.stream()
            .mapToInt(p -> p.getPrice().intValue())
            .summaryStatistics();
        
        System.out.println("가격 통계:");
        System.out.println("  최소: ₩" + priceStats.getMin());
        System.out.println("  최대: ₩" + priceStats.getMax());
        System.out.println("  평균: ₩" + (int)priceStats.getAverage());
        System.out.println("  합계: ₩" + priceStats.getSum());
    }
    
    /**
     * 8. 조건 매칭
     */
    public void matchingExample() {
        System.out.println("\n=== 조건 매칭 ===");
        
        // 모든 상품이 1만원 이상?
        boolean allExpensive = products.stream()
            .allMatch(p -> p.getPrice().compareTo(new BigDecimal("10000")) >= 0);
        System.out.println("모든 상품 1만원 이상? " + allExpensive);
        
        // 50만원 이상 상품이 하나라도?
        boolean anyVeryExpensive = products.stream()
            .anyMatch(p -> p.getPrice().compareTo(new BigDecimal("500000")) >= 0);
        System.out.println("50만원 이상 상품 존재? " + anyVeryExpensive);
        
        // 취소된 주문이 하나도 없는지?
        boolean noCancelled = orders.stream()
            .noneMatch(o -> o.getStatus() == Order.OrderStatus.CANCELLED);
        System.out.println("취소 주문 없음? " + noCancelled);
    }
    
    /**
     * 9. 복잡한 파이프라인
     */
    public void complexPipeline() {
        System.out.println("\n=== 복잡한 파이프라인 ===");
        
        // 완료된 주문 중 5만원 이상인 주문의 상품들을
        // 카테고리별로 그룹핑하여 각 카테고리의 총 판매액 계산
        Map<String, BigDecimal> salesByCategory = orders.stream()
            .filter(order -> order.getStatus() == Order.OrderStatus.COMPLETED)
            .filter(order -> order.getTotal().compareTo(new BigDecimal("50000")) >= 0)
            .flatMap(order -> order.getItems().stream())
            .collect(Collectors.groupingBy(
                OrderItem::getProductName,
                Collectors.reducing(
                    BigDecimal.ZERO,
                    OrderItem::getSubtotal,
                    BigDecimal::add
                )
            ));
        
        System.out.println("고액 주문의 상품별 매출:");
        salesByCategory.forEach((product, total) -> 
            System.out.println("  " + product + ": ₩" + total));
    }
    
    /**
     * 10. 커스텀 Collector
     */
    public void customCollector() {
        System.out.println("\n=== 커스텀 수집 ===");
        
        // 이름을 쉼표로 연결
        String names = products.stream()
            .map(Product::getName)
            .collect(Collectors.joining(", "));
        
        System.out.println("상품 목록: " + names);
        
        // 카테고리별 평균 가격
        Map<String, Double> avgPriceByCategory = products.stream()
            .collect(Collectors.groupingBy(
                Product::getCategory,
                Collectors.averagingDouble(p -> p.getPrice().doubleValue())
            ));
        
        System.out.println("\n카테고리별 평균 가격:");
        avgPriceByCategory.forEach((category, avg) -> 
            System.out.println("  " + category + ": ₩" + avg.intValue()));
    }
    
    /**
     * ============================================
     * TEST DATA
     * ============================================
     */
    private List<Order> createTestOrders() {
        List<Order> orders = new ArrayList<>();
        
        Order order1 = new Order(1L, "홍길동", new BigDecimal("150000"), Order.OrderStatus.COMPLETED);
        order1.addItem(new OrderItem("노트북", new BigDecimal("120000"), 1));
        order1.addItem(new OrderItem("마우스", new BigDecimal("30000"), 1));
        orders.add(order1);
        
        Order order2 = new Order(2L, "김철수", new BigDecimal("80000"), Order.OrderStatus.COMPLETED);
        order2.addItem(new OrderItem("키보드", new BigDecimal("80000"), 1));
        orders.add(order2);
        
        Order order3 = new Order(3L, "이영희", new BigDecimal("30000"), Order.OrderStatus.PENDING);
        order3.addItem(new OrderItem("마우스", new BigDecimal("30000"), 1));
        orders.add(order3);
        
        Order order4 = new Order(4L, "박민수", new BigDecimal("200000"), Order.OrderStatus.COMPLETED);
        order4.addItem(new OrderItem("노트북", new BigDecimal("120000"), 1));
        order4.addItem(new OrderItem("키보드", new BigDecimal("80000"), 1));
        orders.add(order4);
        
        return orders;
    }
    
    private List<Product> createTestProducts() {
        return Arrays.asList(
            new Product(1L, "노트북", new BigDecimal("1200000"), "전자기기", 50),
            new Product(2L, "마우스", new BigDecimal("30000"), "액세서리", 200),
            new Product(3L, "키보드", new BigDecimal("80000"), "액세서리", 150),
            new Product(4L, "모니터", new BigDecimal("300000"), "전자기기", 30),
            new Product(5L, "헤드셋", new BigDecimal("150000"), "액세서리", 80)
        );
    }
}

/**
 * ============================================
 * DEMO
 * ============================================
 */
public class StreamPipelineDemo {
    public static void main(String[] args) {
        System.out.println("=== Stream Pipeline Pattern 예제 ===");
        
        StreamPipelineExamples examples = new StreamPipelineExamples();
        
        examples.basicFilterAndMap();
        examples.sortAndLimit();
        examples.flatMapExample();
        examples.reduceExample();
        examples.groupingExample();
        examples.partitioningExample();
        examples.statisticsExample();
        examples.matchingExample();
        examples.complexPipeline();
        examples.customCollector();
        
        System.out.println("\n✅ 완료!");
    }
}
```

**실행 결과:**
```
=== Stream Pipeline Pattern 예제 ===

=== 기본 필터링과 매핑 ===
완료된 주문 고객: [홍길동, 김철수, 박민수]

=== 정렬과 제한 ===
Top 3 상품:
  노트북: 1200000
  모니터: 300000
  헤드셋: 150000

=== flatMap ===
전체 아이템 수: 6
주문된 상품: [노트북, 마우스, 키보드]

=== reduce (집계) ===
총 매출: ₩430000
최고가 상품: 노트북 (₩1200000)

=== 그룹핑 ===
카테고리별 상품 수:
  전자기기: 2
  액세서리: 3

상태별 주문:
  COMPLETED: 3건
  PENDING: 1건

=== 파티셔닝 ===
10만원 이상: 3개
10만원 미만: 2개

=== 통계 ===
가격 통계:
  최소: ₩30000
  최대: ₩1200000
  평균: ₩352000
  합계: ₩1760000

=== 조건 매칭 ===
모든 상품 1만원 이상? true
50만원 이상 상품 존재? true
취소 주문 없음? true

=== 복잡한 파이프라인 ===
고액 주문의 상품별 매출:
  노트북: 240000
  마우스: 30000
  키보드: 160000

=== 커스텀 수집 ===
상품 목록: 노트북, 마우스, 키보드, 모니터, 헤드셋

카테고리별 평균 가격:
  전자기기: ₩750000
  액세서리: ₩86666

✅ 완료!
```

---

## 5. 실전 예제

### 예제 1: 대시보드 통계 ⭐⭐⭐

```java
public class DashboardStatistics {
    public Map<String, Object> calculateStats(List<Order> orders) {
        return Map.of(
            "totalOrders", orders.size(),
            
            "totalRevenue", orders.stream()
                .map(Order::getTotal)
                .reduce(BigDecimal.ZERO, BigDecimal::add),
            
            "averageOrderValue", orders.stream()
                .mapToDouble(o -> o.getTotal().doubleValue())
                .average()
                .orElse(0.0),
            
            "topCustomer", orders.stream()
                .collect(Collectors.groupingBy(
                    Order::getCustomerName,
                    Collectors.summingDouble(o -> o.getTotal().doubleValue())
                ))
                .entrySet().stream()
                .max(Map.Entry.comparingByValue())
                .map(Map.Entry::getKey)
                .orElse("N/A")
        );
    }
}
```

---

## 6. 성능 최적화

### 🚀 Parallel Stream

```java
// Sequential
long count = products.stream()
    .filter(p -> p.getPrice().compareTo(new BigDecimal("100000")) > 0)
    .count();

// Parallel (대용량 데이터에 유리)
long count = products.parallelStream()
    .filter(p -> p.getPrice().compareTo(new BigDecimal("100000")) > 0)
    .count();
```

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 |
|------|------|
| **가독성** | 의도가 명확 |
| **간결성** | 코드가 짧음 |
| **Lazy** | 필요할 때만 처리 |
| **병렬화** | parallelStream() |

### ❌ 단점

| 단점 | 설명 |
|------|------|
| **디버깅** | 중간 확인 어려움 |
| **학습 곡선** | 함수형 사고 필요 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: forEach에서 외부 상태 변경

```java
// 잘못된 예
List<String> result = new ArrayList<>();
stream.forEach(s -> result.add(s));  // ❌ 외부 상태 변경

// 올바른 예
List<String> result = stream.collect(Collectors.toList());  // ✅
```

---

## 9. 심화 주제

### 🎯 Custom Collector

```java
Collector<Product, ?, String> productSummary = Collector.of(
    StringBuilder::new,
    (sb, p) -> sb.append(p.getName()).append(", "),
    StringBuilder::append,
    StringBuilder::toString
);

String summary = products.stream().collect(productSummary);
```

---

## 10. 핵심 정리

### 📌 체크리스트

```
✅ 선언적 코드 작성
✅ 중간 연산 체이닝
✅ 적절한 Collector 사용
✅ 병렬 처리 고려
✅ 외부 상태 변경 금지
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Functional Interface](./01-FunctionalInterface.md) | [다음: Optional Chaining →](./03-OptionalChaining.md)**

</div>
