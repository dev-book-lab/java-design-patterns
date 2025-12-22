# Hexagonal Architecture Pattern (육각형 아키텍처 / Ports & Adapters)

> **"도메인을 중심에 두고 외부 세계로부터 독립시키자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [Clean Architecture와의 관계](#6-clean-architecture와의-관계)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 도메인이 프레임워크에 종속
@Entity  // JPA 애노테이션
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne
    @JoinColumn(name = "customer_id")
    private Customer customer;  // JPA 관계
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> items;
    
    // 도메인 로직
    public void place() {
        // 비즈니스 규칙
    }
    
    // 😱 문제점:
    // - 도메인이 JPA에 강하게 결합
    // - ORM 변경 시 도메인 코드 수정
    // - 테스트 시 JPA 필요
    // - 도메인 로직이 기술에 오염됨
}

// 문제 2: 비즈니스 로직이 컨트롤러에
@RestController
@RequestMapping("/orders")
public class OrderController {
    @Autowired
    private OrderRepository orderRepository;
    
    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
        // 😱 비즈니스 로직이 컨트롤러에!
        Order order = new Order();
        order.setCustomerId(request.getCustomerId());
        
        // 재고 확인
        for (OrderItemRequest item : request.getItems()) {
            Product product = productRepository.findById(item.getProductId());
            if (product.getStock() < item.getQuantity()) {
                throw new OutOfStockException();
            }
        }
        
        // 총액 계산
        BigDecimal total = BigDecimal.ZERO;
        for (OrderItemRequest item : request.getItems()) {
            total = total.add(item.getPrice().multiply(...));
        }
        order.setTotal(total);
        
        // 저장
        orderRepository.save(order);
        
        // 이메일 발송
        emailService.sendOrderConfirmation(order);
        
        return ResponseEntity.ok(order);
    }
    
    // 😱 문제점:
    // - 비즈니스 로직이 HTTP에 종속
    // - 재사용 불가 (CLI, 배치에서 못 씀)
    // - 테스트 어려움 (웹 서버 필요)
    // - 비즈니스 규칙이 흩어짐
}

// 문제 3: 데이터베이스 변경 불가
public class OrderService {
    @Autowired
    private JdbcTemplate jdbcTemplate;  // JDBC에 직접 의존
    
    public void createOrder(Order order) {
        // 😱 SQL이 비즈니스 로직에 섞임
        String sql = "INSERT INTO orders (customer_id, total) VALUES (?, ?)";
        jdbcTemplate.update(sql, order.getCustomerId(), order.getTotal());
        
        // MySQL → PostgreSQL 변경 시?
        // → 전체 비즈니스 로직 수정!
    }
}

// 문제 4: 외부 서비스에 강하게 결합
public class PaymentService {
    public void processPayment(Order order) {
        // 😱 외부 API 직접 호출
        StripeAPI stripe = new StripeAPI(API_KEY);
        ChargeResult result = stripe.charge(
            order.getTotal(),
            order.getCustomerId()
        );
        
        if (result.isSuccess()) {
            order.markAsPaid();
        }
        
        // Stripe → PayPal 변경 시?
        // → 비즈니스 로직 전체 수정!
        // → 테스트 시 실제 Stripe 필요!
    }
}

// 문제 5: 테스트 불가능
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;  // JPA
    
    @Autowired
    private EmailService emailService;  // SMTP
    
    @Autowired
    private PaymentGateway paymentGateway;  // 외부 API
    
    public void processOrder(Long orderId) {
        // 😱 테스트하려면:
        // - 실제 데이터베이스 필요
        // - 실제 메일 서버 필요
        // - 실제 결제 API 필요
        // → 단위 테스트 불가능!
        // → 통합 테스트만 가능 (느림, 불안정)
    }
}

// 문제 6: 프레임워크 의존성
@Service
@Transactional
public class UserService {
    @Autowired  // Spring 의존
    private UserRepository userRepository;
    
    @Cacheable("users")  // Spring Cache 의존
    public User findUser(Long id) {
        return userRepository.findById(id);
    }
    
    // 😱 문제점:
    // - Spring 없이 실행 불가
    // - 다른 프레임워크로 전환 불가
    // - 순수 Java로 테스트 불가
}
```

### ⚡ 핵심 문제

1. **강한 결합**: 도메인이 프레임워크/DB/외부 서비스에 종속
2. **테스트 어려움**: 실제 인프라 없이 테스트 불가
3. **변경 취약**: 기술 스택 변경 시 도메인 수정
4. **재사용 불가**: 비즈니스 로직이 특정 기술에 묶임
5. **관심사 혼재**: 비즈니스 규칙과 기술 세부사항이 섞임

---

## 2. 패턴 정의

### 📖 정의

> **애플리케이션을 내부(도메인)와 외부(인프라)로 분리하고, Port(인터페이스)를 통해 연결하여 도메인이 외부 세계로부터 독립적이도록 하는 아키텍처 패턴**

### 🎯 목적

- **도메인 독립성**: 비즈니스 로직을 기술로부터 격리
- **테스트 용이**: 도메인을 순수 Java로 테스트
- **교체 가능**: 외부 어댑터 쉽게 교체
- **프레임워크 독립**: 특정 프레임워크에 종속되지 않음

### 💡 핵심 아이디어

```java
// Before: 도메인이 외부에 의존
public class OrderService {
    @Autowired
    private JpaOrderRepository orderRepository;  // JPA에 의존
    
    @Autowired
    private StripePaymentGateway paymentGateway;  // Stripe에 의존
    
    public void createOrder(Order order) {
        orderRepository.save(order);  // JPA 직접 사용
        paymentGateway.charge(order);  // Stripe 직접 사용
    }
}

// After: Port를 통해 도메인 독립
// 1. Port 정의 (도메인 계층의 인터페이스)
public interface OrderPort {
    void save(Order order);
}

public interface PaymentPort {
    PaymentResult charge(Money amount);
}

// 2. 도메인 서비스 (Port만 의존)
public class OrderService {
    private final OrderPort orderPort;  // 인터페이스에만 의존
    private final PaymentPort paymentPort;
    
    public OrderService(OrderPort orderPort, PaymentPort paymentPort) {
        this.orderPort = orderPort;
        this.paymentPort = paymentPort;
    }
    
    public void createOrder(Order order) {
        orderPort.save(order);  // 구현체 몰라도 됨
        paymentPort.charge(order.getTotal());
    }
}

// 3. Adapter 구현 (인프라 계층)
public class JpaOrderAdapter implements OrderPort {
    private JpaOrderRepository repository;
    
    @Override
    public void save(Order order) {
        // JPA 구현
    }
}

public class StripePaymentAdapter implements PaymentPort {
    @Override
    public PaymentResult charge(Money amount) {
        // Stripe 구현
    }
}

// 도메인은 Port만 알고, Adapter는 모름!
// JPA → MongoDB 변경? Adapter만 교체!
// Stripe → PayPal 변경? Adapter만 교체!
```

---

## 3. 구조와 구성요소

### 📊 육각형 구조 (원형)

```
                    Outside World
                         │
        ┌────────────────┼────────────────┐
        │                │                │
    [Adapter]        [Adapter]       [Adapter]
    (REST API)       (CLI)           (Scheduler)
        │                │                │
        └────────────────┼────────────────┘
                         │
                    [Input Port]
                  (Use Case Interface)
                         │
                         ▼
        ┌─────────────────────────────────┐
        │                                 │
        │         DOMAIN CORE             │
        │   (Business Logic / Entities)   │
        │                                 │
        │   - Pure Java                   │
        │   - Framework Independent       │
        │   - Testable                    │
        │                                 │
        └─────────────────────────────────┘
                         │
                    [Output Port]
                  (Repository Interface)
                         │
        ┌────────────────┼────────────────┐
        │                │                │
    [Adapter]        [Adapter]       [Adapter]
    (JPA)            (MongoDB)       (InMemory)
        │                │                │
        └────────────────┼────────────────┘
                         │
                    Outside World
```

### 🔷 계층별 의존성 방향

```
┌───────────────────────────────────────────────┐
│         Adapters (Infrastructure)             │  ← 외부 세계
│  - REST Controller                            │
│  - JPA Repository                             │
│  - External API Client                        │
└───────────────────────────────────────────────┘
                    │
                    │ implements
                    ▼
┌───────────────────────────────────────────────┐
│              Ports (Interfaces)               │  ← 경계
│  - Input Ports (Use Case)                     │
│  - Output Ports (Repository)                  │
└───────────────────────────────────────────────┘
                    △
                    │ depends on
                    │
┌───────────────────────────────────────────────┐
│            Domain Core                        │  ← 중심
│  - Entities                                   │
│  - Value Objects                              │
│  - Domain Services                            │
│  - Business Rules                             │
│                                               │
│  ✅ Pure Java (no frameworks)                 │
│  ✅ No external dependencies                  │
└───────────────────────────────────────────────┘
```

### 🎭 Port와 Adapter의 역할

```
╔═══════════════════════════════════════════════╗
║              PRIMARY (DRIVING)                ║
║         "애플리케이션을 호출하는 쪽"                 ║
╠═══════════════════════════════════════════════╣
║                                               ║
║  Primary Adapter (입력 어댑터)                   ║
║  ┌─────────────────────────────────────┐      ║
║  │ REST Controller                     │      ║
║  │ CLI                                 │      ║
║  │ GraphQL Resolver                    │      ║
║  │ Message Consumer                    │      ║
║  └─────────────────────────────────────┘      ║
║              │                                ║
║              │ calls                          ║
║              ▼                                ║
║  Primary Port (입력 포트)                       ║
║  ┌─────────────────────────────────────┐      ║
║  │ Use Case Interface                  │      ║
║  │ - CreateOrderUseCase                │      ║
║  │ - GetOrderUseCase                   │      ║
║  └─────────────────────────────────────┘      ║
║                                               ║
╠═══════════════════════════════════════════════╣
║                DOMAIN CORE                    ║
║  ┌─────────────────────────────────────┐      ║
║  │ Use Case Implementation             │      ║
║  │ Entities                            │      ║
║  │ Domain Services                     │      ║
║  └─────────────────────────────────────┘      ║
║              │                                ║
║              │ uses                           ║
║              ▼                                ║
║  Secondary Port (출력 포트)                     ║
║  ┌─────────────────────────────────────┐      ║
║  │ Repository Interface                │      ║
║  │ - OrderRepository                   │      ║
║  │ - PaymentGateway                    │      ║
║  │ - NotificationService               │      ║
║  └─────────────────────────────────────┘      ║
║                                               ║
╠═══════════════════════════════════════════════╣
║            SECONDARY (DRIVEN)                 ║
║          "애플리케이션이 호출하는 쪽"                ║
╠═══════════════════════════════════════════════╣
║              △                                ║
║              │ implements                     ║
║  Secondary Adapter (출력 어댑터)                 ║
║  ┌─────────────────────────────────────┐      ║
║  │ JPA Repository                      │      ║
║  │ MongoDB Repository                  │      ║
║  │ Stripe Payment Adapter              │      ║
║  │ Email Service Adapter               │      ║
║  └─────────────────────────────────────┘      ║
║                                               ║
╚═══════════════════════════════════════════════╝
```

### 🔧 구성요소

| 컴포넌트 | 역할 | 위치 | 예시 |
|---------|------|------|------|
| **Domain Core** | 비즈니스 로직 | 중심 | `Order`, `OrderService` |
| **Primary Port** | Use Case 인터페이스 | Domain | `CreateOrderUseCase` |
| **Primary Adapter** | 입력 처리 | Infrastructure | `OrderController` |
| **Secondary Port** | 외부 서비스 인터페이스 | Domain | `OrderRepository` |
| **Secondary Adapter** | 외부 서비스 구현 | Infrastructure | `JpaOrderRepository` |

---

## 4. 구현 방법

### 완전한 구현: E-Commerce 주문 시스템 ⭐⭐⭐

```java
/**
 * ============================================
 * DOMAIN CORE (도메인 핵심)
 * ============================================
 * - Pure Java (프레임워크 독립)
 * - 외부 의존성 없음
 * - 비즈니스 규칙만 포함
 */

/**
 * Entity: 주문
 */
public class Order {
    private OrderId id;
    private CustomerId customerId;
    private List<OrderLine> orderLines;
    private Money totalAmount;
    private OrderStatus status;
    private LocalDateTime createdAt;
    
    public enum OrderStatus {
        PENDING, CONFIRMED, PAID, SHIPPED, DELIVERED, CANCELLED
    }
    
    // Factory Method
    public static Order create(CustomerId customerId) {
        Order order = new Order();
        order.id = OrderId.generate();
        order.customerId = customerId;
        order.orderLines = new ArrayList<>();
        order.status = OrderStatus.PENDING;
        order.totalAmount = Money.ZERO;
        order.createdAt = LocalDateTime.now();
        return order;
    }
    
    /**
     * 도메인 로직: 상품 추가
     */
    public void addProduct(Product product, Quantity quantity) {
        if (status != OrderStatus.PENDING) {
            throw new OrderAlreadyConfirmedException(
                "주문이 이미 확정되었습니다: " + id
            );
        }
        
        // 비즈니스 규칙: 재고 확인
        if (!product.hasStock(quantity)) {
            throw new InsufficientStockException(
                "재고 부족: " + product.getName()
            );
        }
        
        OrderLine orderLine = new OrderLine(product.getId(), product.getPrice(), quantity);
        orderLines.add(orderLine);
        
        recalculateTotal();
    }
    
    /**
     * 도메인 로직: 총액 재계산
     */
    private void recalculateTotal() {
        this.totalAmount = orderLines.stream()
            .map(OrderLine::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
    
    /**
     * 도메인 로직: 주문 확정
     */
    public void confirm() {
        if (orderLines.isEmpty()) {
            throw new EmptyOrderException("주문 항목이 없습니다");
        }
        
        if (status != OrderStatus.PENDING) {
            throw new OrderAlreadyConfirmedException("이미 확정된 주문입니다");
        }
        
        this.status = OrderStatus.CONFIRMED;
    }
    
    /**
     * 도메인 로직: 결제 완료
     */
    public void markAsPaid() {
        if (status != OrderStatus.CONFIRMED) {
            throw new IllegalStateException("확정된 주문만 결제 가능합니다");
        }
        
        this.status = OrderStatus.PAID;
    }
    
    /**
     * 도메인 로직: 배송 시작
     */
    public void ship() {
        if (status != OrderStatus.PAID) {
            throw new IllegalStateException("결제된 주문만 배송 가능합니다");
        }
        
        this.status = OrderStatus.SHIPPED;
    }
    
    // Getters
    public OrderId getId() { return id; }
    public CustomerId getCustomerId() { return customerId; }
    public List<OrderLine> getOrderLines() { return Collections.unmodifiableList(orderLines); }
    public Money getTotalAmount() { return totalAmount; }
    public OrderStatus getStatus() { return status; }
    public LocalDateTime getCreatedAt() { return createdAt; }
}

/**
 * Value Object: 주문 라인
 */
public class OrderLine {
    private final ProductId productId;
    private final Money price;
    private final Quantity quantity;
    
    public OrderLine(ProductId productId, Money price, Quantity quantity) {
        this.productId = productId;
        this.price = price;
        this.quantity = quantity;
    }
    
    public Money getSubtotal() {
        return price.multiply(quantity.getValue());
    }
    
    public ProductId getProductId() { return productId; }
    public Money getPrice() { return price; }
    public Quantity getQuantity() { return quantity; }
}

/**
 * Value Object: 주문 ID
 */
public class OrderId {
    private final Long value;
    
    private OrderId(Long value) {
        this.value = value;
    }
    
    public static OrderId of(Long value) {
        if (value == null || value <= 0) {
            throw new IllegalArgumentException("Invalid order ID");
        }
        return new OrderId(value);
    }
    
    public static OrderId generate() {
        // 실제로는 UUID나 Sequence 사용
        return new OrderId(System.currentTimeMillis());
    }
    
    public Long getValue() { return value; }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof OrderId)) return false;
        OrderId orderId = (OrderId) o;
        return Objects.equals(value, orderId.value);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(value);
    }
}

/**
 * Value Object: 금액
 */
public class Money {
    public static final Money ZERO = new Money(BigDecimal.ZERO);
    
    private final BigDecimal amount;
    
    private Money(BigDecimal amount) {
        this.amount = amount;
    }
    
    public static Money of(BigDecimal amount) {
        if (amount == null || amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Invalid amount");
        }
        return new Money(amount);
    }
    
    public Money add(Money other) {
        return new Money(this.amount.add(other.amount));
    }
    
    public Money multiply(int multiplier) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(multiplier)));
    }
    
    public BigDecimal getAmount() { return amount; }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money)) return false;
        Money money = (Money) o;
        return amount.compareTo(money.amount) == 0;
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(amount);
    }
}

/**
 * Value Object: 수량
 */
public class Quantity {
    private final int value;
    
    private Quantity(int value) {
        this.value = value;
    }
    
    public static Quantity of(int value) {
        if (value <= 0) {
            throw new IllegalArgumentException("수량은 1 이상이어야 합니다");
        }
        return new Quantity(value);
    }
    
    public int getValue() { return value; }
}

/**
 * Entity: 상품
 */
public class Product {
    private ProductId id;
    private String name;
    private Money price;
    private Stock stock;
    
    public Product(ProductId id, String name, Money price, Stock stock) {
        this.id = id;
        this.name = name;
        this.price = price;
        this.stock = stock;
    }
    
    /**
     * 도메인 로직: 재고 확인
     */
    public boolean hasStock(Quantity quantity) {
        return stock.isAvailable(quantity);
    }
    
    /**
     * 도메인 로직: 재고 차감
     */
    public void decreaseStock(Quantity quantity) {
        this.stock = stock.decrease(quantity);
    }
    
    // Getters
    public ProductId getId() { return id; }
    public String getName() { return name; }
    public Money getPrice() { return price; }
    public Stock getStock() { return stock; }
}

/**
 * Value Object: 재고
 */
public class Stock {
    private final int quantity;
    
    private Stock(int quantity) {
        this.quantity = quantity;
    }
    
    public static Stock of(int quantity) {
        if (quantity < 0) {
            throw new IllegalArgumentException("재고는 0 이상이어야 합니다");
        }
        return new Stock(quantity);
    }
    
    public boolean isAvailable(Quantity required) {
        return quantity >= required.getValue();
    }
    
    public Stock decrease(Quantity amount) {
        if (!isAvailable(amount)) {
            throw new InsufficientStockException("재고 부족");
        }
        return new Stock(quantity - amount.getValue());
    }
    
    public int getQuantity() { return quantity; }
}

/**
 * ============================================
 * PRIMARY PORTS (입력 포트 - Use Case)
 * ============================================
 * 도메인 계층에 정의
 */

/**
 * Use Case: 주문 생성
 */
public interface CreateOrderUseCase {
    OrderId execute(CreateOrderCommand command);
}

/**
 * Command (입력 데이터)
 */
public class CreateOrderCommand {
    private final CustomerId customerId;
    private final List<OrderItemDto> items;
    
    public CreateOrderCommand(CustomerId customerId, List<OrderItemDto> items) {
        this.customerId = customerId;
        this.items = items;
    }
    
    public CustomerId getCustomerId() { return customerId; }
    public List<OrderItemDto> getItems() { return items; }
    
    public static class OrderItemDto {
        private final ProductId productId;
        private final Quantity quantity;
        
        public OrderItemDto(ProductId productId, Quantity quantity) {
            this.productId = productId;
            this.quantity = quantity;
        }
        
        public ProductId getProductId() { return productId; }
        public Quantity getQuantity() { return quantity; }
    }
}

/**
 * Use Case: 주문 조회
 */
public interface GetOrderUseCase {
    Order execute(OrderId orderId);
}

/**
 * Use Case: 주문 결제
 */
public interface PayOrderUseCase {
    void execute(PayOrderCommand command);
}

public class PayOrderCommand {
    private final OrderId orderId;
    private final PaymentMethod paymentMethod;
    
    public PayOrderCommand(OrderId orderId, PaymentMethod paymentMethod) {
        this.orderId = orderId;
        this.paymentMethod = paymentMethod;
    }
    
    public OrderId getOrderId() { return orderId; }
    public PaymentMethod getPaymentMethod() { return paymentMethod; }
}

/**
 * ============================================
 * SECONDARY PORTS (출력 포트 - Repository)
 * ============================================
 * 도메인 계층에 정의
 */

/**
 * Repository Port: 주문
 */
public interface OrderRepository {
    Order save(Order order);
    Order findById(OrderId id);
    List<Order> findByCustomerId(CustomerId customerId);
}

/**
 * Repository Port: 상품
 */
public interface ProductRepository {
    Product findById(ProductId id);
    void save(Product product);
}

/**
 * External Service Port: 결제
 */
public interface PaymentGateway {
    PaymentResult charge(Money amount, PaymentMethod method);
}

public class PaymentResult {
    private final boolean success;
    private final String transactionId;
    
    public PaymentResult(boolean success, String transactionId) {
        this.success = success;
        this.transactionId = transactionId;
    }
    
    public boolean isSuccess() { return success; }
    public String getTransactionId() { return transactionId; }
}

/**
 * External Service Port: 알림
 */
public interface NotificationService {
    void sendOrderConfirmation(Order order);
}

/**
 * ============================================
 * USE CASE IMPLEMENTATION (도메인 서비스)
 * ============================================
 * Port만 의존 (구현체 모름)
 */

/**
 * 주문 생성 Use Case 구현
 */
public class CreateOrderService implements CreateOrderUseCase {
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final NotificationService notificationService;
    
    // Port만 의존 (DI)
    public CreateOrderService(
            OrderRepository orderRepository,
            ProductRepository productRepository,
            NotificationService notificationService) {
        this.orderRepository = orderRepository;
        this.productRepository = productRepository;
        this.notificationService = notificationService;
    }
    
    @Override
    public OrderId execute(CreateOrderCommand command) {
        System.out.println("\n🛒 === 주문 생성 Use Case 실행 ===");
        
        // 1. 도메인 객체 생성
        Order order = Order.create(command.getCustomerId());
        
        // 2. 상품 추가 (도메인 로직)
        for (CreateOrderCommand.OrderItemDto item : command.getItems()) {
            Product product = productRepository.findById(item.getProductId());
            
            // 도메인 로직 실행 (재고 확인 포함)
            order.addProduct(product, item.getQuantity());
            
            // 재고 차감 (도메인 로직)
            product.decreaseStock(item.getQuantity());
            productRepository.save(product);
        }
        
        // 3. 주문 확정 (도메인 로직)
        order.confirm();
        
        // 4. 저장 (Port 사용)
        Order savedOrder = orderRepository.save(order);
        
        // 5. 알림 발송 (Port 사용)
        notificationService.sendOrderConfirmation(savedOrder);
        
        System.out.println("✅ 주문 생성 완료: " + savedOrder.getId().getValue());
        System.out.println("   총액: " + savedOrder.getTotalAmount().getAmount());
        
        return savedOrder.getId();
    }
}

/**
 * 주문 결제 Use Case 구현
 */
public class PayOrderService implements PayOrderUseCase {
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    
    public PayOrderService(OrderRepository orderRepository, 
                          PaymentGateway paymentGateway) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
    }
    
    @Override
    public void execute(PayOrderCommand command) {
        System.out.println("\n💳 === 주문 결제 Use Case 실행 ===");
        
        // 1. 주문 조회
        Order order = orderRepository.findById(command.getOrderId());
        
        // 2. 결제 처리 (Port 사용)
        PaymentResult result = paymentGateway.charge(
            order.getTotalAmount(),
            command.getPaymentMethod()
        );
        
        if (!result.isSuccess()) {
            throw new PaymentFailedException("결제 실패");
        }
        
        // 3. 주문 상태 변경 (도메인 로직)
        order.markAsPaid();
        
        // 4. 저장
        orderRepository.save(order);
        
        System.out.println("✅ 결제 완료: " + result.getTransactionId());
    }
}

/**
 * 주문 조회 Use Case 구현
 */
public class GetOrderService implements GetOrderUseCase {
    private final OrderRepository orderRepository;
    
    public GetOrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
    
    @Override
    public Order execute(OrderId orderId) {
        return orderRepository.findById(orderId);
    }
}

/**
 * ============================================
 * PRIMARY ADAPTERS (입력 어댑터)
 * ============================================
 * 인프라 계층
 */

/**
 * REST API Adapter
 */
public class OrderRestController {
    private final CreateOrderUseCase createOrderUseCase;
    private final GetOrderUseCase getOrderUseCase;
    private final PayOrderUseCase payOrderUseCase;
    
    public OrderRestController(
            CreateOrderUseCase createOrderUseCase,
            GetOrderUseCase getOrderUseCase,
            PayOrderUseCase payOrderUseCase) {
        this.createOrderUseCase = createOrderUseCase;
        this.getOrderUseCase = getOrderUseCase;
        this.payOrderUseCase = payOrderUseCase;
    }
    
    /**
     * POST /orders
     */
    public OrderResponse createOrder(CreateOrderRequest request) {
        System.out.println("📥 REST API: POST /orders");
        
        // DTO → Command 변환
        CreateOrderCommand command = toCommand(request);
        
        // Use Case 실행
        OrderId orderId = createOrderUseCase.execute(command);
        
        // 조회 후 응답
        Order order = getOrderUseCase.execute(orderId);
        
        return toResponse(order);
    }
    
    /**
     * GET /orders/{id}
     */
    public OrderResponse getOrder(Long id) {
        System.out.println("📥 REST API: GET /orders/" + id);
        
        OrderId orderId = OrderId.of(id);
        Order order = getOrderUseCase.execute(orderId);
        
        return toResponse(order);
    }
    
    /**
     * POST /orders/{id}/pay
     */
    public void payOrder(Long id, PayOrderRequest request) {
        System.out.println("📥 REST API: POST /orders/" + id + "/pay");
        
        PayOrderCommand command = new PayOrderCommand(
            OrderId.of(id),
            request.getPaymentMethod()
        );
        
        payOrderUseCase.execute(command);
    }
    
    // DTO 변환
    private CreateOrderCommand toCommand(CreateOrderRequest request) {
        List<CreateOrderCommand.OrderItemDto> items = request.getItems().stream()
            .map(item -> new CreateOrderCommand.OrderItemDto(
                ProductId.of(item.getProductId()),
                Quantity.of(item.getQuantity())
            ))
            .collect(Collectors.toList());
        
        return new CreateOrderCommand(
            CustomerId.of(request.getCustomerId()),
            items
        );
    }
    
    private OrderResponse toResponse(Order order) {
        return new OrderResponse(
            order.getId().getValue(),
            order.getCustomerId().getValue(),
            order.getTotalAmount().getAmount(),
            order.getStatus().name()
        );
    }
}

/**
 * Request DTO
 */
public class CreateOrderRequest {
    private Long customerId;
    private List<OrderItemRequest> items;
    
    // Getters, Setters
    public Long getCustomerId() { return customerId; }
    public void setCustomerId(Long customerId) { this.customerId = customerId; }
    public List<OrderItemRequest> getItems() { return items; }
    public void setItems(List<OrderItemRequest> items) { this.items = items; }
    
    public static class OrderItemRequest {
        private Long productId;
        private int quantity;
        
        public Long getProductId() { return productId; }
        public void setProductId(Long productId) { this.productId = productId; }
        public int getQuantity() { return quantity; }
        public void setQuantity(int quantity) { this.quantity = quantity; }
    }
}

/**
 * Response DTO
 */
public class OrderResponse {
    private Long id;
    private Long customerId;
    private BigDecimal totalAmount;
    private String status;
    
    public OrderResponse(Long id, Long customerId, BigDecimal totalAmount, String status) {
        this.id = id;
        this.customerId = customerId;
        this.totalAmount = totalAmount;
        this.status = status;
    }
    
    // Getters
    public Long getId() { return id; }
    public Long getCustomerId() { return customerId; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public String getStatus() { return status; }
}

/**
 * ============================================
 * SECONDARY ADAPTERS (출력 어댑터)
 * ============================================
 * 인프라 계층
 */

/**
 * InMemory Repository Adapter
 */
public class InMemoryOrderRepository implements OrderRepository {
    private final Map<OrderId, Order> storage = new ConcurrentHashMap<>();
    
    @Override
    public Order save(Order order) {
        storage.put(order.getId(), order);
        System.out.println("💾 InMemory: 주문 저장 - " + order.getId().getValue());
        return order;
    }
    
    @Override
    public Order findById(OrderId id) {
        Order order = storage.get(id);
        if (order == null) {
            throw new OrderNotFoundException("주문 없음: " + id.getValue());
        }
        System.out.println("🔍 InMemory: 주문 조회 - " + id.getValue());
        return order;
    }
    
    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        return storage.values().stream()
            .filter(order -> order.getCustomerId().equals(customerId))
            .collect(Collectors.toList());
    }
}

/**
 * InMemory Product Repository Adapter
 */
public class InMemoryProductRepository implements ProductRepository {
    private final Map<ProductId, Product> storage = new ConcurrentHashMap<>();
    
    public InMemoryProductRepository() {
        // 초기 데이터
        Product product1 = new Product(
            ProductId.of(1L),
            "노트북",
            Money.of(new BigDecimal("1200000")),
            Stock.of(10)
        );
        Product product2 = new Product(
            ProductId.of(2L),
            "마우스",
            Money.of(new BigDecimal("30000")),
            Stock.of(50)
        );
        
        storage.put(product1.getId(), product1);
        storage.put(product2.getId(), product2);
    }
    
    @Override
    public Product findById(ProductId id) {
        Product product = storage.get(id);
        if (product == null) {
            throw new ProductNotFoundException("상품 없음: " + id.getValue());
        }
        return product;
    }
    
    @Override
    public void save(Product product) {
        storage.put(product.getId(), product);
        System.out.println("💾 InMemory: 상품 저장 - " + product.getName());
    }
}

/**
 * Mock Payment Gateway Adapter
 */
public class MockPaymentGateway implements PaymentGateway {
    
    @Override
    public PaymentResult charge(Money amount, PaymentMethod method) {
        System.out.println("💳 MockPayment: 결제 처리");
        System.out.println("   금액: " + amount.getAmount());
        System.out.println("   방법: " + method);
        
        // 실제로는 외부 API 호출
        // 여기서는 항상 성공
        String transactionId = "TXN-" + System.currentTimeMillis();
        
        return new PaymentResult(true, transactionId);
    }
}

/**
 * Console Notification Adapter
 */
public class ConsoleNotificationService implements NotificationService {
    
    @Override
    public void sendOrderConfirmation(Order order) {
        System.out.println("📧 알림: 주문 확인 이메일 발송");
        System.out.println("   주문번호: " + order.getId().getValue());
        System.out.println("   금액: " + order.getTotalAmount().getAmount());
    }
}

/**
 * ============================================
 * CONFIGURATION (의존성 조립)
 * ============================================
 */
public class OrderConfiguration {
    
    public static OrderRestController createRestController() {
        // Secondary Adapters (출력)
        OrderRepository orderRepository = new InMemoryOrderRepository();
        ProductRepository productRepository = new InMemoryProductRepository();
        PaymentGateway paymentGateway = new MockPaymentGateway();
        NotificationService notificationService = new ConsoleNotificationService();
        
        // Use Cases (도메인)
        CreateOrderUseCase createOrderUseCase = new CreateOrderService(
            orderRepository,
            productRepository,
            notificationService
        );
        
        GetOrderUseCase getOrderUseCase = new GetOrderService(orderRepository);
        
        PayOrderUseCase payOrderUseCase = new PayOrderService(
            orderRepository,
            paymentGateway
        );
        
        // Primary Adapter (입력)
        return new OrderRestController(
            createOrderUseCase,
            getOrderUseCase,
            payOrderUseCase
        );
    }
}

/**
 * ============================================
 * MAIN APPLICATION
 * ============================================
 */
public class HexagonalArchitectureExample {
    public static void main(String[] args) {
        System.out.println("=== Hexagonal Architecture 예제 ===");
        System.out.println("Port & Adapters 패턴\n");
        
        // 의존성 조립
        OrderRestController controller = OrderConfiguration.createRestController();
        
        try {
            // 1. 주문 생성
            CreateOrderRequest request = new CreateOrderRequest();
            request.setCustomerId(100L);
            
            List<CreateOrderRequest.OrderItemRequest> items = new ArrayList<>();
            CreateOrderRequest.OrderItemRequest item1 = new CreateOrderRequest.OrderItemRequest();
            item1.setProductId(1L);
            item1.setQuantity(1);
            items.add(item1);
            
            CreateOrderRequest.OrderItemRequest item2 = new CreateOrderRequest.OrderItemRequest();
            item2.setProductId(2L);
            item2.setQuantity(2);
            items.add(item2);
            
            request.setItems(items);
            
            OrderResponse orderResponse = controller.createOrder(request);
            System.out.println("\n📦 생성된 주문: " + orderResponse.getId());
            
            // 2. 주문 조회
            OrderResponse retrievedOrder = controller.getOrder(orderResponse.getId());
            System.out.println("\n📋 조회된 주문:");
            System.out.println("   ID: " + retrievedOrder.getId());
            System.out.println("   상태: " + retrievedOrder.getStatus());
            System.out.println("   총액: " + retrievedOrder.getTotalAmount());
            
            // 3. 결제
            PayOrderRequest payRequest = new PayOrderRequest();
            payRequest.setPaymentMethod(PaymentMethod.CREDIT_CARD);
            
            controller.payOrder(orderResponse.getId(), payRequest);
            
            // 4. 결제 후 조회
            OrderResponse paidOrder = controller.getOrder(orderResponse.getId());
            System.out.println("\n📋 결제 후 주문 상태: " + paidOrder.getStatus());
            
        } catch (Exception e) {
            System.err.println("❌ 오류: " + e.getMessage());
            e.printStackTrace();
        }
    }
}

/**
 * 필요한 Value Objects 및 예외
 */
class CustomerId {
    private final Long value;
    
    private CustomerId(Long value) { this.value = value; }
    
    public static CustomerId of(Long value) {
        return new CustomerId(value);
    }
    
    public Long getValue() { return value; }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof CustomerId)) return false;
        CustomerId that = (CustomerId) o;
        return Objects.equals(value, that.value);
    }
    
    @Override
    public int hashCode() { return Objects.hash(value); }
}

class ProductId {
    private final Long value;
    
    private ProductId(Long value) { this.value = value; }
    
    public static ProductId of(Long value) {
        return new ProductId(value);
    }
    
    public Long getValue() { return value; }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof ProductId)) return false;
        ProductId productId = (ProductId) o;
        return Objects.equals(value, productId.value);
    }
    
    @Override
    public int hashCode() { return Objects.hash(value); }
}

enum PaymentMethod {
    CREDIT_CARD, DEBIT_CARD, PAYPAL, BANK_TRANSFER
}

class PayOrderRequest {
    private PaymentMethod paymentMethod;
    
    public PaymentMethod getPaymentMethod() { return paymentMethod; }
    public void setPaymentMethod(PaymentMethod paymentMethod) { 
        this.paymentMethod = paymentMethod; 
    }
}

// 도메인 예외
class OrderNotFoundException extends RuntimeException {
    public OrderNotFoundException(String message) { super(message); }
}

class ProductNotFoundException extends RuntimeException {
    public ProductNotFoundException(String message) { super(message); }
}

class InsufficientStockException extends RuntimeException {
    public InsufficientStockException(String message) { super(message); }
}

class OrderAlreadyConfirmedException extends RuntimeException {
    public OrderAlreadyConfirmedException(String message) { super(message); }
}

class EmptyOrderException extends RuntimeException {
    public EmptyOrderException(String message) { super(message); }
}

class PaymentFailedException extends RuntimeException {
    public PaymentFailedException(String message) { super(message); }
}
```

**실행 결과:**
```
=== Hexagonal Architecture 예제 ===
Port & Adapters 패턴

📥 REST API: POST /orders

🛒 === 주문 생성 Use Case 실행 ===
💾 InMemory: 상품 저장 - 노트북
💾 InMemory: 상품 저장 - 마우스
💾 InMemory: 주문 저장 - 1734876543210
📧 알림: 주문 확인 이메일 발송
   주문번호: 1734876543210
   금액: 1260000
✅ 주문 생성 완료: 1734876543210
   총액: 1260000

📦 생성된 주문: 1734876543210

📥 REST API: GET /orders/1734876543210
🔍 InMemory: 주문 조회 - 1734876543210

📋 조회된 주문:
   ID: 1734876543210
   상태: CONFIRMED
   총액: 1260000

📥 REST API: POST /orders/1734876543210/pay

💳 === 주문 결제 Use Case 실행 ===
🔍 InMemory: 주문 조회 - 1734876543210
💳 MockPayment: 결제 처리
   금액: 1260000
   방법: CREDIT_CARD
💾 InMemory: 주문 저장 - 1734876543210
✅ 결제 완료: TXN-1734876543250

📥 REST API: GET /orders/1734876543210
🔍 InMemory: 주문 조회 - 1734876543210

📋 결제 후 주문 상태: PAID
```

---

## 5. 실전 예제

### 예제 1: JPA Adapter 구현 ⭐⭐⭐

```java
/**
 * ============================================
 * JPA Adapter 구현
 * ============================================
 * 도메인 <-> JPA Entity 변환
 */

/**
 * JPA Entity (인프라 계층)
 */
@Entity
@Table(name = "orders")
public class OrderJpaEntity {
    @Id
    private Long id;
    
    @Column(name = "customer_id")
    private Long customerId;
    
    @Column(name = "total_amount")
    private BigDecimal totalAmount;
    
    @Enumerated(EnumType.STRING)
    private Order.OrderStatus status;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderLineJpaEntity> orderLines = new ArrayList<>();
    
    // Getters, Setters
}

@Entity
@Table(name = "order_lines")
public class OrderLineJpaEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private OrderJpaEntity order;
    
    @Column(name = "product_id")
    private Long productId;
    
    private BigDecimal price;
    
    private int quantity;
    
    // Getters, Setters
}

/**
 * Spring Data JPA Repository
 */
public interface OrderJpaRepository extends JpaRepository<OrderJpaEntity, Long> {
    List<OrderJpaEntity> findByCustomerId(Long customerId);
}

/**
 * JPA Adapter (Secondary Adapter)
 */
@Repository
public class JpaOrderRepositoryAdapter implements OrderRepository {
    private final OrderJpaRepository jpaRepository;
    
    @Autowired
    public JpaOrderRepositoryAdapter(OrderJpaRepository jpaRepository) {
        this.jpaRepository = jpaRepository;
    }
    
    @Override
    public Order save(Order order) {
        // Domain → JPA Entity 변환
        OrderJpaEntity entity = toEntity(order);
        
        // JPA 저장
        OrderJpaEntity saved = jpaRepository.save(entity);
        
        // JPA Entity → Domain 변환
        return toDomain(saved);
    }
    
    @Override
    public Order findById(OrderId id) {
        OrderJpaEntity entity = jpaRepository.findById(id.getValue())
            .orElseThrow(() -> new OrderNotFoundException("주문 없음: " + id));
        
        return toDomain(entity);
    }
    
    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        List<OrderJpaEntity> entities = jpaRepository.findByCustomerId(
            customerId.getValue()
        );
        
        return entities.stream()
            .map(this::toDomain)
            .collect(Collectors.toList());
    }
    
    /**
     * Domain → JPA Entity 변환
     */
    private OrderJpaEntity toEntity(Order order) {
        OrderJpaEntity entity = new OrderJpaEntity();
        entity.setId(order.getId().getValue());
        entity.setCustomerId(order.getCustomerId().getValue());
        entity.setTotalAmount(order.getTotalAmount().getAmount());
        entity.setStatus(order.getStatus());
        entity.setCreatedAt(order.getCreatedAt());
        
        List<OrderLineJpaEntity> lineEntities = order.getOrderLines().stream()
            .map(line -> {
                OrderLineJpaEntity lineEntity = new OrderLineJpaEntity();
                lineEntity.setOrder(entity);
                lineEntity.setProductId(line.getProductId().getValue());
                lineEntity.setPrice(line.getPrice().getAmount());
                lineEntity.setQuantity(line.getQuantity().getValue());
                return lineEntity;
            })
            .collect(Collectors.toList());
        
        entity.setOrderLines(lineEntities);
        
        return entity;
    }
    
    /**
     * JPA Entity → Domain 변환
     */
    private Order toDomain(OrderJpaEntity entity) {
        Order order = Order.create(CustomerId.of(entity.getCustomerId()));
        
        // 리플렉션으로 ID 설정 (또는 Order에 setId 메서드 제공)
        setOrderId(order, OrderId.of(entity.getId()));
        
        // OrderLine 복원
        for (OrderLineJpaEntity lineEntity : entity.getOrderLines()) {
            // Product는 별도 조회 필요하거나, 최소 정보만으로 OrderLine 생성
            // 실제로는 Product도 함께 조회하거나 별도 처리
        }
        
        return order;
    }
    
    private void setOrderId(Order order, OrderId id) {
        // 리플렉션 사용 또는 Order에 패키지 private setter 제공
    }
}
```

---

### 예제 2: 여러 Adapter 교체 ⭐⭐⭐

```java
/**
 * ============================================
 * 다양한 Adapter 구현
 * ============================================
 * 도메인 코드 변경 없이 교체 가능
 */

/**
 * 1. MongoDB Adapter
 */
@Document(collection = "orders")
public class OrderMongoDocument {
    @Id
    private String id;
    private Long customerId;
    private BigDecimal totalAmount;
    // ...
}

public class MongoOrderRepositoryAdapter implements OrderRepository {
    private final MongoTemplate mongoTemplate;
    
    @Override
    public Order save(Order order) {
        OrderMongoDocument doc = toDocument(order);
        mongoTemplate.save(doc);
        return order;
    }
    
    // MongoDB 구현...
}

/**
 * 2. Redis Cache Adapter
 */
public class RedisOrderRepositoryAdapter implements OrderRepository {
    private final OrderRepository delegate;  // 실제 Repository
    private final RedisTemplate<String, Order> redisTemplate;
    
    public RedisOrderRepositoryAdapter(
            OrderRepository delegate,
            RedisTemplate<String, Order> redisTemplate) {
        this.delegate = delegate;
        this.redisTemplate = redisTemplate;
    }
    
    @Override
    public Order findById(OrderId id) {
        // 캐시 확인
        String key = "order:" + id.getValue();
        Order cached = redisTemplate.opsForValue().get(key);
        
        if (cached != null) {
            System.out.println("💨 Redis 캐시 히트");
            return cached;
        }
        
        // DB 조회
        Order order = delegate.findById(id);
        
        // 캐시 저장
        redisTemplate.opsForValue().set(key, order, 10, TimeUnit.MINUTES);
        
        return order;
    }
    
    @Override
    public Order save(Order order) {
        Order saved = delegate.save(order);
        
        // 캐시 무효화
        String key = "order:" + order.getId().getValue();
        redisTemplate.delete(key);
        
        return saved;
    }
}

/**
 * 3. Real Payment Gateway Adapter (Stripe)
 */
public class StripePaymentAdapter implements PaymentGateway {
    private final Stripe stripe;
    
    public StripePaymentAdapter(String apiKey) {
        Stripe.apiKey = apiKey;
        this.stripe = new Stripe();
    }
    
    @Override
    public PaymentResult charge(Money amount, PaymentMethod method) {
        try {
            // 실제 Stripe API 호출
            Map<String, Object> params = new HashMap<>();
            params.put("amount", amount.getAmount().multiply(new BigDecimal("100")).intValue());
            params.put("currency", "krw");
            params.put("source", convertPaymentMethod(method));
            
            Charge charge = Charge.create(params);
            
            return new PaymentResult(
                "succeeded".equals(charge.getStatus()),
                charge.getId()
            );
            
        } catch (StripeException e) {
            throw new PaymentFailedException("Stripe 결제 실패: " + e.getMessage());
        }
    }
    
    private String convertPaymentMethod(PaymentMethod method) {
        // PaymentMethod → Stripe source 변환
        return "tok_visa";  // 테스트 토큰
    }
}

/**
 * 4. Email Notification Adapter (SendGrid)
 */
public class SendGridNotificationAdapter implements NotificationService {
    private final SendGrid sendGrid;
    
    public SendGridNotificationAdapter(String apiKey) {
        this.sendGrid = new SendGrid(apiKey);
    }
    
    @Override
    public void sendOrderConfirmation(Order order) {
        Email from = new Email("noreply@example.com");
        Email to = new Email(getCustomerEmail(order.getCustomerId()));
        String subject = "주문 확인 - " + order.getId().getValue();
        
        Content content = new Content(
            "text/html",
            buildEmailContent(order)
        );
        
        Mail mail = new Mail(from, subject, to, content);
        
        try {
            Request request = new Request();
            request.setMethod(Method.POST);
            request.setEndpoint("mail/send");
            request.setBody(mail.build());
            
            Response response = sendGrid.api(request);
            
            System.out.println("📧 SendGrid: 이메일 발송 완료");
            
        } catch (IOException e) {
            // 로깅만 하고 실패해도 주문 처리는 계속
            System.err.println("이메일 발송 실패: " + e.getMessage());
        }
    }
    
    private String buildEmailContent(Order order) {
        return "<h1>주문이 확정되었습니다</h1>" +
               "<p>주문번호: " + order.getId().getValue() + "</p>" +
               "<p>총액: " + order.getTotalAmount().getAmount() + "원</p>";
    }
}

/**
 * Configuration에서 Adapter 선택
 */
@Configuration
public class AdapterConfiguration {
    
    @Bean
    public OrderRepository orderRepository(
            @Value("${storage.type}") String storageType) {
        
        switch (storageType) {
            case "jpa":
                return new JpaOrderRepositoryAdapter(...);
            
            case "mongo":
                return new MongoOrderRepositoryAdapter(...);
            
            case "inmemory":
                return new InMemoryOrderRepository();
            
            case "redis-cache":
                return new RedisOrderRepositoryAdapter(
                    new JpaOrderRepositoryAdapter(...),
                    redisTemplate()
                );
            
            default:
                throw new IllegalArgumentException("Unknown storage: " + storageType);
        }
    }
    
    @Bean
    public PaymentGateway paymentGateway(
            @Value("${payment.gateway}") String gateway) {
        
        switch (gateway) {
            case "stripe":
                return new StripePaymentAdapter(stripeApiKey);
            
            case "paypal":
                return new PayPalPaymentAdapter(paypalApiKey);
            
            case "mock":
                return new MockPaymentGateway();
            
            default:
                throw new IllegalArgumentException("Unknown gateway: " + gateway);
        }
    }
}
```

**application.yml 설정만으로 Adapter 교체:**
```yaml
# 개발 환경
storage:
  type: inmemory

payment:
  gateway: mock

---
# 운영 환경
storage:
  type: redis-cache

payment:
  gateway: stripe
```

---

## 6. Clean Architecture와의 관계

### 🔷 Clean Architecture 동심원

```
┌───────────────────────────────────────────┐
│     Frameworks & Drivers (Blue)           │  ← 가장 바깥
│  - Web                                    │
│  - DB                                     │
│  - External APIs                          │
└───────────────────────────────────────────┘
                │
                │ depends on
                ▼
┌───────────────────────────────────────────┐
│     Interface Adapters (Green)            │
│  - Controllers                            │
│  - Presenters                             │
│  - Gateways                               │
└───────────────────────────────────────────┘
                │
                │ depends on
                ▼
┌───────────────────────────────────────────┐
│     Application Business Rules (Red)      │
│  - Use Cases                              │
└───────────────────────────────────────────┘
                │
                │ depends on
                ▼
┌───────────────────────────────────────────┐
│     Enterprise Business Rules (Yellow)    │  ← 가장 안쪽
│  - Entities                               │
│  - Value Objects                          │
└───────────────────────────────────────────┘
```

### 🔄 Hexagonal ↔ Clean Architecture 매핑

| Hexagonal Architecture | Clean Architecture | 역할 |
|------------------------|-------------------|------|
| **Domain Core** | Enterprise Business Rules | 엔티티, Value Object |
| **Use Case (Port)** | Application Business Rules | Use Case 인터페이스 |
| **Use Case Impl** | Application Business Rules | Use Case 구현 |
| **Primary Adapter** | Interface Adapters | Controller, Presenter |
| **Secondary Adapter** | Interface Adapters | Gateway, Repository Impl |
| **External** | Frameworks & Drivers | DB, Web, External API |

### 💡 핵심 원칙

1. **의존성 규칙**: 안쪽은 바깥쪽을 모른다
2. **안정성**: 안쪽일수록 변경 빈도 낮음
3. **추상화**: 경계는 인터페이스로

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 | 실무 효과 |
|------|------|-----------|
| **도메인 독립성** | 프레임워크/DB 독립 | 기술 변경 용이 |
| **테스트 용이** | 순수 Java 테스트 | 빠른 단위 테스트 |
| **교체 가능** | Adapter 교체 쉬움 | 유연한 아키텍처 |
| **명확한 경계** | Port로 경계 명확 | 책임 분리 |
| **재사용성** | 도메인 로직 재사용 | 여러 UI에서 공유 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **초기 복잡도** | 코드량 증가 | 복잡한 도메인만 |
| **학습 곡선** | 개념 이해 필요 | 문서화 |
| **변환 비용** | Domain ↔ DTO 변환 | 매퍼 자동화 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: Port를 건너뛰고 Adapter 직접 사용

```java
// 잘못된 예: UseCase가 Adapter를 직접 의존
public class CreateOrderService implements CreateOrderUseCase {
    private final JpaOrderRepository jpaRepository;  // ❌ 구현체 의존
    
    @Override
    public OrderId execute(CreateOrderCommand command) {
        // JPA에 직접 의존
        jpaRepository.save(...);
    }
}
```

**해결:**
```java
// 올바른 예: Port(인터페이스)에 의존
public class CreateOrderService implements CreateOrderUseCase {
    private final OrderRepository orderRepository;  // ✅ 인터페이스 의존
    
    @Override
    public OrderId execute(CreateOrderCommand command) {
        orderRepository.save(...);
    }
}
```

### ❌ 안티패턴 2: 도메인에 프레임워크 애노테이션

```java
// 잘못된 예: 도메인이 JPA에 오염
@Entity  // ❌ JPA 애노테이션
public class Order {
    @Id
    @GeneratedValue
    private Long id;
    
    @ManyToOne
    private Customer customer;  // ❌ JPA 관계
}
```

**해결:**
```java
// 올바른 예: Pure Java
public class Order {  // ✅ 애노테이션 없음
    private OrderId id;
    private CustomerId customerId;  // ✅ Value Object
    
    // 순수 비즈니스 로직만
}

// JPA Entity는 별도 (Infrastructure)
@Entity
public class OrderJpaEntity {
    @Id
    private Long id;
    // JPA 전용
}
```

---

## 9. 심화 주제

### 🎯 DDD와 통합

```java
/**
 * Aggregate Root
 */
public class Order {  // Aggregate Root
    private OrderId id;
    private List<OrderLine> orderLines;  // Entity
    
    // Aggregate 경계 내에서만 변경
    public void addProduct(Product product, Quantity quantity) {
        // 불변식 검증
        // 이벤트 발행
    }
}

/**
 * Domain Event
 */
public class OrderPlacedEvent {
    private final OrderId orderId;
    private final LocalDateTime occurredOn;
    
    public OrderPlacedEvent(OrderId orderId) {
        this.orderId = orderId;
        this.occurredOn = LocalDateTime.now();
    }
}

/**
 * Event Publisher Port
 */
public interface EventPublisher {
    void publish(DomainEvent event);
}

/**
 * Use Case에서 이벤트 발행
 */
public class CreateOrderService implements CreateOrderUseCase {
    private final OrderRepository orderRepository;
    private final EventPublisher eventPublisher;
    
    @Override
    public OrderId execute(CreateOrderCommand command) {
        Order order = Order.create(...);
        orderRepository.save(order);
        
        // 도메인 이벤트 발행
        eventPublisher.publish(new OrderPlacedEvent(order.getId()));
        
        return order.getId();
    }
}
```

### 🔥 CQRS와 통합

```java
/**
 * Command (쓰기)
 */
public interface CreateOrderCommand extends Command {
    OrderId execute(CreateOrderRequest request);
}

/**
 * Query (읽기)
 */
public interface OrderQuery {
    OrderDTO findById(OrderId id);
    List<OrderSummaryDTO> findByCustomerId(CustomerId customerId);
}

/**
 * 분리된 Read Model
 */
public class OrderQueryService implements OrderQuery {
    private final JdbcTemplate jdbcTemplate;  // 읽기 최적화
    
    @Override
    public OrderDTO findById(OrderId id) {
        // Join 최적화된 쿼리
        String sql = """
            SELECT o.*, c.name as customer_name, ...
            FROM orders o
            JOIN customers c ON o.customer_id = c.id
            WHERE o.id = ?
        """;
        
        return jdbcTemplate.queryForObject(sql, ...);
    }
}
```

---

## 10. 핵심 정리

### 📌 Hexagonal Architecture 체크리스트

```
✅ Domain Core는 Pure Java (프레임워크 독립)
✅ Port는 Domain Layer에 정의
✅ Adapter는 Infrastructure Layer에 구현
✅ 도메인은 Port만 의존 (Adapter 모름)
✅ Primary Adapter → Use Case (Port) 호출
✅ Use Case → Secondary Port 사용
✅ Adapter는 언제든 교체 가능
✅ 테스트는 Mock Port로
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 복잡한 도메인 | ⭐⭐⭐ | 도메인 보호 |
| 장기 프로젝트 | ⭐⭐⭐ | 유지보수성 |
| 기술 변경 예상 | ⭐⭐⭐ | 교체 용이 |
| 간단한 CRUD | ⭐ | 오버엔지니어링 |

### 💡 핵심 원칙

1. **도메인이 중심**
2. **Port로 경계 정의**
3. **Adapter는 교체 가능**
4. **의존성은 안쪽으로**

### 🔥 실무 적용

```java
// ✅ DO: Port를 통해 추상화
public interface PaymentGateway {
    PaymentResult charge(Money amount);
}

// ✅ DO: 도메인은 Pure Java
public class Order {
    public void place() {
        // 비즈니스 규칙
    }
}

// ✅ DO: Adapter는 Infrastructure
public class StripePaymentAdapter implements PaymentGateway {
    // Stripe 구현
}

// ❌ DON'T: 도메인에 프레임워크 의존
@Entity  // ❌
public class Order {
    @ManyToOne  // ❌
    private Customer customer;
}

// ❌ DON'T: UseCase가 Adapter 직접 의존
public class OrderService {
    private JpaOrderRepository repository;  // ❌
}
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Repository](./03-Repository.md) | [다음: MVVM →](./05-MVVM.md)**

</div>
