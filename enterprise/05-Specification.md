# Specification Pattern (명세 패턴)

> **"비즈니스 규칙을 재사용 가능한 객체로 캡슐화하여 동적 쿼리를 만들자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [Spring Data JPA 통합](#6-spring-data-jpa-통합)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 복잡한 조건문이 여기저기 중복
public class ProductRepository {
    public List<Product> findAvailableProducts() {
        return entityManager.createQuery(
            "SELECT p FROM Product p " +
            "WHERE p.stock > 0 " +
            "AND p.status = 'ACTIVE' " +
            "AND p.deletedAt IS NULL",
            Product.class
        ).getResultList();
    }
    
    public List<Product> findDiscountedProducts() {
        return entityManager.createQuery(
            "SELECT p FROM Product p " +
            "WHERE p.stock > 0 " +              // 😱 중복!
            "AND p.status = 'ACTIVE' " +         // 😱 중복!
            "AND p.deletedAt IS NULL " +         // 😱 중복!
            "AND p.discountRate > 0",
            Product.class
        ).getResultList();
    }
    
    public List<Product> findPremiumProducts() {
        return entityManager.createQuery(
            "SELECT p FROM Product p " +
            "WHERE p.stock > 0 " +              // 😱 또 중복!
            "AND p.status = 'ACTIVE' " +         // 😱 또 중복!
            "AND p.deletedAt IS NULL " +         // 😱 또 중복!
            "AND p.price > 100000",
            Product.class
        ).getResultList();
    }
    
    // "판매 가능한 상품" 조건이 변경되면?
    // → 모든 메서드를 수정해야 함!
}

// 문제 2: 동적 쿼리 작성 어려움
public class ProductSearchService {
    public List<Product> search(ProductSearchCriteria criteria) {
        // 😱 동적 쿼리를 문자열로!
        
        StringBuilder sql = new StringBuilder("SELECT p FROM Product p WHERE 1=1");
        
        if (criteria.getName() != null) {
            sql.append(" AND p.name LIKE '%").append(criteria.getName()).append("%'");
        }
        
        if (criteria.getMinPrice() != null) {
            sql.append(" AND p.price >= ").append(criteria.getMinPrice());
        }
        
        if (criteria.getMaxPrice() != null) {
            sql.append(" AND p.price <= ").append(criteria.getMaxPrice());
        }
        
        if (criteria.getCategory() != null) {
            sql.append(" AND p.category = '").append(criteria.getCategory()).append("'");
        }
        
        // 문제점:
        // - SQL Injection 위험!
        // - 타입 안전성 없음!
        // - 읽기 어려움!
        // - 테스트 어려움!
        
        return entityManager.createQuery(sql.toString(), Product.class).getResultList();
    }
}

// 문제 3: 비즈니스 규칙 재사용 불가
public class OrderService {
    public List<Order> getPendingOrders(Long userId) {
        // 비즈니스 규칙: "대기 중인 주문"
        return orderRepository.findByUserIdAndStatus(userId, OrderStatus.PENDING);
    }
}

public class StatisticsService {
    public long countPendingOrders(Long userId) {
        // 😱 같은 비즈니스 규칙을 다시 정의!
        return orderRepository.countByUserIdAndStatus(userId, OrderStatus.PENDING);
    }
}

public class NotificationService {
    public void sendPendingOrderReminders() {
        // 😱 또 같은 규칙!
        List<Order> orders = orderRepository.findByStatus(OrderStatus.PENDING);
        // ...
    }
}

// "대기 중인 주문"의 정의가 변경되면?
// → 모든 곳을 수정해야 함!

// 문제 4: 복잡한 조건 조합
public class ProductRepository {
    // 😱 조합 가능한 모든 경우의 수를 메서드로!
    
    List<Product> findByCategory(String category);
    List<Product> findByPrice(BigDecimal minPrice, BigDecimal maxPrice);
    List<Product> findByCategoryAndPrice(String category, BigDecimal minPrice, BigDecimal maxPrice);
    List<Product> findByName(String name);
    List<Product> findByNameAndCategory(String name, String category);
    List<Product> findByNameAndPrice(String name, BigDecimal minPrice, BigDecimal maxPrice);
    List<Product> findByNameAndCategoryAndPrice(String name, String category, BigDecimal minPrice, BigDecimal maxPrice);
    // ... 무한 증가!
    
    // 조건이 10개면?
    // → 2^10 = 1024개 메서드!
}

// 문제 5: 조건문 중복
public class OrderService {
    public List<Order> getVipOrders() {
        List<Order> orders = orderRepository.findAll();
        
        return orders.stream()
            .filter(order -> {
                // 😱 복잡한 VIP 조건
                return order.getTotalAmount().compareTo(new BigDecimal("100000")) >= 0
                    && order.getUser().getGrade() == UserGrade.VIP
                    && order.getStatus() == OrderStatus.COMPLETED;
            })
            .collect(Collectors.toList());
    }
    
    public boolean isVipOrder(Order order) {
        // 😱 같은 조건을 또 작성!
        return order.getTotalAmount().compareTo(new BigDecimal("100000")) >= 0
            && order.getUser().getGrade() == UserGrade.VIP
            && order.getStatus() == OrderStatus.COMPLETED;
    }
}

// 문제 6: 테스트 어려움
public class ProductService {
    public List<Product> getRecommendedProducts(User user) {
        // 😱 복잡한 추천 로직이 메서드 안에!
        
        List<Product> products = productRepository.findAll();
        
        return products.stream()
            .filter(product -> {
                // 복잡한 추천 규칙
                if (user.getAge() < 20) {
                    return product.getCategory().equals("TEEN");
                } else if (user.getAge() < 40) {
                    return product.getCategory().equals("ADULT") 
                        && product.getPrice().compareTo(new BigDecimal("50000")) <= 0;
                } else {
                    return product.getCategory().equals("SENIOR")
                        || product.getDiscountRate() > 0.3;
                }
            })
            .collect(Collectors.toList());
        
        // 이 로직만 테스트하려면?
        // → 불가능! 전체 메서드를 테스트해야 함!
    }
}

// 문제 7: AND/OR 조합 복잡
public class ProductSearchService {
    public List<Product> searchAdvanced(AdvancedSearchCriteria criteria) {
        // 😱 복잡한 AND/OR 조건
        
        // (category = 'A' AND price > 10000)
        // OR
        // (category = 'B' AND discount > 0.2)
        // OR
        // (brand = 'Premium' AND stock > 100)
        
        // 어떻게 구현?
        // 문자열 SQL? → 위험!
        // Criteria API? → 너무 복잡!
    }
}
```

### ⚡ 핵심 문제

1. **코드 중복**: 같은 조건이 여러 곳에 반복
2. **재사용 불가**: 비즈니스 규칙을 재사용할 수 없음
3. **조합 어려움**: 조건을 동적으로 조합하기 어려움
4. **테스트 어려움**: 조건만 따로 테스트 불가
5. **유지보수**: 규칙 변경 시 여러 곳 수정
6. **가독성**: 복잡한 조건문 이해 어려움

---

## 2. 패턴 정의

### 📖 정의

> **비즈니스 규칙을 독립적인 객체로 캡슐화하여, 런타임에 조합하고 재사용할 수 있도록 하는 패턴**

### 🎯 목적

- **재사용**: 비즈니스 규칙을 여러 곳에서 사용
- **조합**: 규칙을 AND/OR로 조합
- **테스트**: 규칙을 독립적으로 테스트
- **명확성**: 규칙의 의도를 명확하게 표현

### 💡 핵심 아이디어

```java
// Before: 조건이 여기저기 중복
public List<Product> findAvailableProducts() {
    return products.stream()
        .filter(p -> p.getStock() > 0 && p.getStatus() == Status.ACTIVE)
        .collect(Collectors.toList());
}

public List<Product> findDiscountedProducts() {
    return products.stream()
        .filter(p -> p.getStock() > 0 && p.getStatus() == Status.ACTIVE && p.getDiscount() > 0)
        .collect(Collectors.toList());
}

// After: Specification으로 재사용
// 1. Specification 정의
public interface Specification<T> {
    boolean isSatisfiedBy(T candidate);
}

// 2. 구체적인 Specification
public class AvailableProductSpec implements Specification<Product> {
    @Override
    public boolean isSatisfiedBy(Product product) {
        return product.getStock() > 0 && product.getStatus() == Status.ACTIVE;
    }
}

// 3. 재사용
Specification<Product> availableSpec = new AvailableProductSpec();
Specification<Product> discountedSpec = new DiscountedProductSpec();

// 조합
Specification<Product> availableAndDiscounted = availableSpec.and(discountedSpec);

// 사용
List<Product> products = productRepository.findAll().stream()
    .filter(availableAndDiscounted::isSatisfiedBy)
    .collect(Collectors.toList());
```

---

## 3. 구조와 구성요소

### 📊 Specification 구조

```
┌─────────────────────────────────────┐
│     Specification<T>                │
│                                     │
│  + isSatisfiedBy(T): boolean        │
│  + and(Specification<T>)            │
│  + or(Specification<T>)             │
│  + not()                            │
└─────────────────────────────────────┘
              △
              │ implements
              │
    ┌─────────┴─────────┬──────────────┐
    │                   │              │
┌─────────┐      ┌──────────┐    ┌─────────┐
│Available│      │Discounted│    │Premium  │
│  Spec   │      │   Spec   │    │  Spec   │
└─────────┘      └──────────┘    └─────────┘

// 조합
AvailableSpec.and(DiscountedSpec)
AvailableSpec.or(PremiumSpec)
AvailableSpec.not()
```

### 🔄 조합 방식

```
Spec A = new AvailableProductSpec()
Spec B = new DiscountedProductSpec()
Spec C = new PremiumProductSpec()

// AND 조합
Spec A AND B = A.and(B)

// OR 조합
Spec A OR C = A.or(C)

// NOT
Spec NOT A = A.not()

// 복잡한 조합
((A AND B) OR C) AND (NOT D)
= A.and(B).or(C).and(D.not())
```

---

## 4. 구현 방법

### 완전한 구현: Product Specification ⭐⭐⭐

```java
/**
 * ============================================
 * SPECIFICATION INTERFACE
 * ============================================
 */
public interface Specification<T> {
    /**
     * 조건 만족 여부
     */
    boolean isSatisfiedBy(T candidate);
    
    /**
     * AND 조합
     */
    default Specification<T> and(Specification<T> other) {
        return new AndSpecification<>(this, other);
    }
    
    /**
     * OR 조합
     */
    default Specification<T> or(Specification<T> other) {
        return new OrSpecification<>(this, other);
    }
    
    /**
     * NOT (부정)
     */
    default Specification<T> not() {
        return new NotSpecification<>(this);
    }
}

/**
 * ============================================
 * COMPOSITE SPECIFICATIONS
 * ============================================
 */

/**
 * AND Specification
 */
class AndSpecification<T> implements Specification<T> {
    private final Specification<T> left;
    private final Specification<T> right;
    
    public AndSpecification(Specification<T> left, Specification<T> right) {
        this.left = left;
        this.right = right;
    }
    
    @Override
    public boolean isSatisfiedBy(T candidate) {
        return left.isSatisfiedBy(candidate) && right.isSatisfiedBy(candidate);
    }
}

/**
 * OR Specification
 */
class OrSpecification<T> implements Specification<T> {
    private final Specification<T> left;
    private final Specification<T> right;
    
    public OrSpecification(Specification<T> left, Specification<T> right) {
        this.left = left;
        this.right = right;
    }
    
    @Override
    public boolean isSatisfiedBy(T candidate) {
        return left.isSatisfiedBy(candidate) || right.isSatisfiedBy(candidate);
    }
}

/**
 * NOT Specification
 */
class NotSpecification<T> implements Specification<T> {
    private final Specification<T> spec;
    
    public NotSpecification(Specification<T> spec) {
        this.spec = spec;
    }
    
    @Override
    public boolean isSatisfiedBy(T candidate) {
        return !spec.isSatisfiedBy(candidate);
    }
}

/**
 * ============================================
 * DOMAIN MODEL
 * ============================================
 */
public class Product {
    private Long id;
    private String name;
    private BigDecimal price;
    private int stock;
    private ProductStatus status;
    private String category;
    private double discountRate;
    
    public enum ProductStatus {
        ACTIVE, INACTIVE, OUT_OF_STOCK
    }
    
    // Constructors
    public Product(Long id, String name, BigDecimal price, int stock, ProductStatus status) {
        this.id = id;
        this.name = name;
        this.price = price;
        this.stock = stock;
        this.status = status;
    }
    
    // Getters, Setters
    public Long getId() { return id; }
    public String getName() { return name; }
    public BigDecimal getPrice() { return price; }
    public int getStock() { return stock; }
    public ProductStatus getStatus() { return status; }
    public String getCategory() { return category; }
    public void setCategory(String category) { this.category = category; }
    public double getDiscountRate() { return discountRate; }
    public void setDiscountRate(double discountRate) { this.discountRate = discountRate; }
}

/**
 * ============================================
 * CONCRETE SPECIFICATIONS
 * ============================================
 */

/**
 * 판매 가능한 상품 (재사용 가능한 비즈니스 규칙!)
 */
public class AvailableProductSpecification implements Specification<Product> {
    
    @Override
    public boolean isSatisfiedBy(Product product) {
        return product.getStock() > 0 
            && product.getStatus() == Product.ProductStatus.ACTIVE;
    }
    
    @Override
    public String toString() {
        return "판매 가능한 상품";
    }
}

/**
 * 할인 중인 상품
 */
public class DiscountedProductSpecification implements Specification<Product> {
    
    @Override
    public boolean isSatisfiedBy(Product product) {
        return product.getDiscountRate() > 0;
    }
    
    @Override
    public String toString() {
        return "할인 중인 상품";
    }
}

/**
 * 프리미엄 상품 (10만원 이상)
 */
public class PremiumProductSpecification implements Specification<Product> {
    private static final BigDecimal PREMIUM_THRESHOLD = new BigDecimal("100000");
    
    @Override
    public boolean isSatisfiedBy(Product product) {
        return product.getPrice().compareTo(PREMIUM_THRESHOLD) >= 0;
    }
    
    @Override
    public String toString() {
        return "프리미엄 상품 (10만원 이상)";
    }
}

/**
 * 카테고리별 상품 (파라미터 있는 Specification)
 */
public class CategorySpecification implements Specification<Product> {
    private final String category;
    
    public CategorySpecification(String category) {
        this.category = category;
    }
    
    @Override
    public boolean isSatisfiedBy(Product product) {
        return category.equals(product.getCategory());
    }
    
    @Override
    public String toString() {
        return "카테고리: " + category;
    }
}

/**
 * 가격 범위 상품
 */
public class PriceRangeSpecification implements Specification<Product> {
    private final BigDecimal minPrice;
    private final BigDecimal maxPrice;
    
    public PriceRangeSpecification(BigDecimal minPrice, BigDecimal maxPrice) {
        this.minPrice = minPrice;
        this.maxPrice = maxPrice;
    }
    
    @Override
    public boolean isSatisfiedBy(Product product) {
        return product.getPrice().compareTo(minPrice) >= 0
            && product.getPrice().compareTo(maxPrice) <= 0;
    }
    
    @Override
    public String toString() {
        return "가격: " + minPrice + "원 ~ " + maxPrice + "원";
    }
}

/**
 * ============================================
 * REPOSITORY (Specification 사용)
 * ============================================
 */
public class ProductRepository {
    private final List<Product> products = new ArrayList<>();
    
    /**
     * Specification으로 조회 (핵심!)
     */
    public List<Product> find(Specification<Product> spec) {
        return products.stream()
            .filter(spec::isSatisfiedBy)
            .collect(Collectors.toList());
    }
    
    /**
     * 첫 번째 하나만 조회
     */
    public Product findFirst(Specification<Product> spec) {
        return products.stream()
            .filter(spec::isSatisfiedBy)
            .findFirst()
            .orElse(null);
    }
    
    /**
     * 개수 세기
     */
    public long count(Specification<Product> spec) {
        return products.stream()
            .filter(spec::isSatisfiedBy)
            .count();
    }
    
    /**
     * 존재 여부
     */
    public boolean exists(Specification<Product> spec) {
        return products.stream()
            .anyMatch(spec::isSatisfiedBy);
    }
    
    // 테스트용 데이터 추가
    public void add(Product product) {
        products.add(product);
    }
}

/**
 * ============================================
 * SERVICE (Specification 활용)
 * ============================================
 */
public class ProductService {
    private final ProductRepository productRepository;
    
    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }
    
    /**
     * 판매 가능한 상품 조회 (Specification 재사용!)
     */
    public List<Product> getAvailableProducts() {
        Specification<Product> spec = new AvailableProductSpecification();
        return productRepository.find(spec);
    }
    
    /**
     * 판매 가능하면서 할인 중인 상품 (조합!)
     */
    public List<Product> getAvailableAndDiscountedProducts() {
        Specification<Product> available = new AvailableProductSpecification();
        Specification<Product> discounted = new DiscountedProductSpecification();
        
        // AND 조합
        Specification<Product> spec = available.and(discounted);
        
        return productRepository.find(spec);
    }
    
    /**
     * 프리미엄 상품 또는 할인 중인 상품 (OR 조합!)
     */
    public List<Product> getPremiumOrDiscountedProducts() {
        Specification<Product> premium = new PremiumProductSpecification();
        Specification<Product> discounted = new DiscountedProductSpecification();
        
        // OR 조합
        Specification<Product> spec = premium.or(discounted);
        
        return productRepository.find(spec);
    }
    
    /**
     * 복잡한 조건 조합
     * (판매 가능하면서 할인 중) 또는 (프리미엄 상품)
     */
    public List<Product> getSpecialProducts() {
        Specification<Product> available = new AvailableProductSpecification();
        Specification<Product> discounted = new DiscountedProductSpecification();
        Specification<Product> premium = new PremiumProductSpecification();
        
        // (available AND discounted) OR premium
        Specification<Product> spec = available.and(discounted).or(premium);
        
        return productRepository.find(spec);
    }
    
    /**
     * 동적 검색 (여러 조건 조합)
     */
    public List<Product> search(ProductSearchCriteria criteria) {
        // 기본: 판매 가능한 상품
        Specification<Product> spec = new AvailableProductSpecification();
        
        // 카테고리 필터
        if (criteria.getCategory() != null) {
            spec = spec.and(new CategorySpecification(criteria.getCategory()));
        }
        
        // 가격 범위 필터
        if (criteria.getMinPrice() != null && criteria.getMaxPrice() != null) {
            spec = spec.and(new PriceRangeSpecification(
                criteria.getMinPrice(),
                criteria.getMaxPrice()
            ));
        }
        
        // 할인 상품만
        if (criteria.isDiscountedOnly()) {
            spec = spec.and(new DiscountedProductSpecification());
        }
        
        return productRepository.find(spec);
    }
}

/**
 * 검색 조건
 */
class ProductSearchCriteria {
    private String category;
    private BigDecimal minPrice;
    private BigDecimal maxPrice;
    private boolean discountedOnly;
    
    // Getters, Setters
    public String getCategory() { return category; }
    public void setCategory(String category) { this.category = category; }
    public BigDecimal getMinPrice() { return minPrice; }
    public void setMinPrice(BigDecimal minPrice) { this.minPrice = minPrice; }
    public BigDecimal getMaxPrice() { return maxPrice; }
    public void setMaxPrice(BigDecimal maxPrice) { this.maxPrice = maxPrice; }
    public boolean isDiscountedOnly() { return discountedOnly; }
    public void setDiscountedOnly(boolean discountedOnly) { this.discountedOnly = discountedOnly; }
}

/**
 * ============================================
 * DEMO
 * ============================================
 */
public class SpecificationDemo {
    public static void main(String[] args) {
        System.out.println("=== Specification Pattern 예제 ===\n");
        
        // 1. Repository 생성
        ProductRepository repository = new ProductRepository();
        
        // 2. 테스트 데이터
        repository.add(new Product(1L, "노트북", new BigDecimal("1200000"), 10, Product.ProductStatus.ACTIVE));
        repository.add(new Product(2L, "마우스", new BigDecimal("30000"), 50, Product.ProductStatus.ACTIVE));
        repository.add(new Product(3L, "키보드", new BigDecimal("80000"), 0, Product.ProductStatus.OUT_OF_STOCK));
        repository.add(new Product(4L, "모니터", new BigDecimal("300000"), 5, Product.ProductStatus.ACTIVE));
        repository.add(new Product(5L, "헤드셋", new BigDecimal("150000"), 20, Product.ProductStatus.INACTIVE));
        
        Product p1 = repository.find(new Specification<Product>() {
            @Override
            public boolean isSatisfiedBy(Product p) {
                return p.getId() == 1L;
            }
        }).get(0);
        p1.setCategory("전자기기");
        p1.setDiscountRate(0.1);
        
        Product p2 = repository.find(new Specification<Product>() {
            @Override
            public boolean isSatisfiedBy(Product p) {
                return p.getId() == 2L;
            }
        }).get(0);
        p2.setCategory("액세서리");
        p2.setDiscountRate(0.2);
        
        Product p4 = repository.find(new Specification<Product>() {
            @Override
            public boolean isSatisfiedBy(Product p) {
                return p.getId() == 4L;
            }
        }).get(0);
        p4.setCategory("전자기기");
        
        // 3. Service 생성
        ProductService service = new ProductService(repository);
        
        // 4. 판매 가능한 상품
        System.out.println("📦 판매 가능한 상품:");
        Specification<Product> availableSpec = new AvailableProductSpecification();
        List<Product> available = repository.find(availableSpec);
        available.forEach(p -> System.out.println("   - " + p.getName() + " (" + p.getPrice() + "원)"));
        
        System.out.println("\n" + "=".repeat(60));
        
        // 5. 판매 가능하면서 할인 중
        System.out.println("\n💰 판매 가능하면서 할인 중인 상품:");
        List<Product> discounted = service.getAvailableAndDiscountedProducts();
        discounted.forEach(p -> System.out.println("   - " + p.getName() + " (할인율: " + (p.getDiscountRate() * 100) + "%)"));
        
        System.out.println("\n" + "=".repeat(60));
        
        // 6. 프리미엄 또는 할인
        System.out.println("\n⭐ 프리미엄 또는 할인 중인 상품:");
        List<Product> special = service.getPremiumOrDiscountedProducts();
        special.forEach(p -> System.out.println("   - " + p.getName() + " (" + p.getPrice() + "원)"));
        
        System.out.println("\n" + "=".repeat(60));
        
        // 7. 동적 검색
        System.out.println("\n🔍 동적 검색 (전자기기 + 할인):");
        ProductSearchCriteria criteria = new ProductSearchCriteria();
        criteria.setCategory("전자기기");
        criteria.setDiscountedOnly(true);
        
        List<Product> searched = service.search(criteria);
        searched.forEach(p -> System.out.println("   - " + p.getName()));
        
        System.out.println("\n" + "=".repeat(60));
        
        // 8. NOT 조합
        System.out.println("\n🚫 프리미엄이 아닌 상품:");
        Specification<Product> notPremium = new PremiumProductSpecification().not();
        List<Product> notPremiumProducts = repository.find(availableSpec.and(notPremium));
        notPremiumProducts.forEach(p -> System.out.println("   - " + p.getName() + " (" + p.getPrice() + "원)"));
        
        System.out.println("\n✅ 완료!");
    }
}
```

**실행 결과:**
```
=== Specification Pattern 예제 ===

📦 판매 가능한 상품:
   - 노트북 (1200000원)
   - 마우스 (30000원)
   - 모니터 (300000원)

============================================================

💰 판매 가능하면서 할인 중인 상품:
   - 노트북 (할인율: 10.0%)
   - 마우스 (할인율: 20.0%)

============================================================

⭐ 프리미엄 또는 할인 중인 상품:
   - 노트북 (1200000원)
   - 마우스 (30000원)
   - 모니터 (300000원)

============================================================

🔍 동적 검색 (전자기기 + 할인):
   - 노트북

============================================================

🚫 프리미엄이 아닌 상품:
   - 마우스 (30000원)

✅ 완료!
```

---

## 5. 실전 예제

### 예제 1: Order Specification ⭐⭐⭐

```java
/**
 * 주문 Specifications
 */
public class OrderSpecifications {
    
    /**
     * 대기 중인 주문 (재사용 가능!)
     */
    public static Specification<Order> isPending() {
        return order -> order.getStatus() == OrderStatus.PENDING;
    }
    
    /**
     * 특정 사용자의 주문
     */
    public static Specification<Order> byUser(Long userId) {
        return order -> order.getUserId().equals(userId);
    }
    
    /**
     * 금액 이상
     */
    public static Specification<Order> amountGreaterThan(BigDecimal amount) {
        return order -> order.getTotalAmount().compareTo(amount) >= 0;
    }
    
    /**
     * 날짜 범위
     */
    public static Specification<Order> createdBetween(LocalDateTime start, LocalDateTime end) {
        return order -> {
            LocalDateTime created = order.getCreatedAt();
            return created.isAfter(start) && created.isBefore(end);
        };
    }
}

/**
 * 사용 예
 */
public class OrderService {
    public List<Order> getVipOrders(Long userId) {
        // 조합: 특정 사용자 AND 10만원 이상 AND 대기 중
        Specification<Order> spec = OrderSpecifications.byUser(userId)
            .and(OrderSpecifications.amountGreaterThan(new BigDecimal("100000")))
            .and(OrderSpecifications.isPending());
        
        return orderRepository.find(spec);
    }
}
```

---

## 6. Spring Data JPA 통합

### 🔄 JpaSpecificationExecutor

```java
/**
 * Spring Data JPA Specification
 */
public interface ProductRepository extends JpaRepository<Product, Long>, 
                                          JpaSpecificationExecutor<Product> {
    // JpaSpecificationExecutor가 제공:
    // - findAll(Specification<Product> spec)
    // - findOne(Specification<Product> spec)
    // - count(Specification<Product> spec)
}

/**
 * Specification 정의
 */
public class ProductSpecs {
    
    public static Specification<Product> isAvailable() {
        return (root, query, cb) -> cb.and(
            cb.greaterThan(root.get("stock"), 0),
            cb.equal(root.get("status"), ProductStatus.ACTIVE)
        );
    }
    
    public static Specification<Product> hasCategory(String category) {
        return (root, query, cb) -> cb.equal(root.get("category"), category);
    }
    
    public static Specification<Product> priceBetween(BigDecimal min, BigDecimal max) {
        return (root, query, cb) -> cb.between(root.get("price"), min, max);
    }
}

/**
 * 사용
 */
@Service
public class ProductService {
    @Autowired
    private ProductRepository productRepository;
    
    public List<Product> search(String category, BigDecimal minPrice, BigDecimal maxPrice) {
        // Specification 조합
        Specification<Product> spec = ProductSpecs.isAvailable()
            .and(ProductSpecs.hasCategory(category))
            .and(ProductSpecs.priceBetween(minPrice, maxPrice));
        
        // 실행 (JPA 쿼리로 변환!)
        return productRepository.findAll(spec);
    }
}
```

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 |
|------|------|
| **재사용성** | 비즈니스 규칙 재사용 |
| **조합 가능** | AND/OR로 동적 조합 |
| **테스트 용이** | 규칙을 독립적으로 테스트 |
| **명확성** | 의도가 명확 |
| **유지보수** | 규칙 변경 시 한 곳만 수정 |

### ❌ 단점

| 단점 | 설명 |
|------|------|
| **클래스 증가** | Specification 클래스 많아짐 |
| **복잡도** | 초기 학습 필요 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: 너무 많은 파라미터

```java
// 잘못된 예
public class ComplexSpecification implements Specification<Product> {
    private String category;
    private BigDecimal minPrice;
    private BigDecimal maxPrice;
    private boolean discounted;
    private String brand;
    private int minStock;
    // ... 10개 이상
    
    public ComplexSpecification(String category, BigDecimal minPrice, ...) {
        // 😱 생성자가 너무 복잡!
    }
}
```

**해결:**
```java
// 올바른 예: 작은 Specification 조합
Specification<Product> spec = new CategorySpec(category)
    .and(new PriceRangeSpec(minPrice, maxPrice))
    .and(new DiscountedSpec());
```

---

## 9. 심화 주제

### 🎯 Fluent API

```java
/**
 * Fluent Specification Builder
 */
public class ProductSpecBuilder {
    private Specification<Product> spec;
    
    public ProductSpecBuilder() {
        this.spec = (p) -> true;  // 항상 true
    }
    
    public ProductSpecBuilder available() {
        spec = spec.and(new AvailableProductSpecification());
        return this;
    }
    
    public ProductSpecBuilder category(String category) {
        spec = spec.and(new CategorySpecification(category));
        return this;
    }
    
    public ProductSpecBuilder priceBetween(BigDecimal min, BigDecimal max) {
        spec = spec.and(new PriceRangeSpecification(min, max));
        return this;
    }
    
    public Specification<Product> build() {
        return spec;
    }
}

/**
 * 사용
 */
Specification<Product> spec = new ProductSpecBuilder()
    .available()
    .category("전자기기")
    .priceBetween(new BigDecimal("10000"), new BigDecimal("100000"))
    .build();
```

---

## 10. 핵심 정리

### 📌 Specification 체크리스트

```
✅ 비즈니스 규칙을 Specification으로
✅ AND/OR/NOT 조합 가능
✅ 재사용 가능하게 설계
✅ 파라미터는 최소화
✅ Spring Data JPA 활용
✅ 독립적으로 테스트
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 복잡한 조회 조건 | ⭐⭐⭐ | 동적 조합 |
| 비즈니스 규칙 재사용 | ⭐⭐⭐ | 중복 제거 |
| 간단한 CRUD | ⭐ | 오버엔지니어링 |

---

<div align="center">

**[← 이전: Unit of Work](./04-UnitOfWork.md)**

</div>
