# Layered Architecture Pattern (계층형 아키텍처 패턴)

> **"시스템을 수평적 계층으로 나누어 관심사를 분리하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [장단점](#6-장단점)
7. [안티패턴](#7-안티패턴)
8. [심화 주제](#8-심화-주제)
9. [핵심 정리](#9-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 모든 것이 한 클래스에 (God Object)
public class UserService {
    // UI 로직
    public void showUserList() {
        System.out.println("=== 사용자 목록 ===");
        List<User> users = getAllUsers();
        for (User user : users) {
            System.out.println(user.getName());
        }
    }
    
    // 비즈니스 로직
    public void registerUser(String name, String email, String password) {
        // 검증
        if (name == null || name.isEmpty()) {
            throw new IllegalArgumentException("이름 필수");
        }
        
        // 비밀번호 암호화
        String hashedPassword = BCrypt.hashpw(password, BCrypt.gensalt());
        
        // 데이터베이스 저장
        Connection conn = DriverManager.getConnection(DB_URL);
        PreparedStatement stmt = conn.prepareStatement(
            "INSERT INTO users (name, email, password) VALUES (?, ?, ?)"
        );
        stmt.setString(1, name);
        stmt.setString(2, email);
        stmt.setString(3, hashedPassword);
        stmt.executeUpdate();
        
        // 이메일 발송
        sendWelcomeEmail(email);
    }
    
    // 데이터 접근 로직
    public List<User> getAllUsers() {
        Connection conn = DriverManager.getConnection(DB_URL);
        Statement stmt = conn.createStatement();
        ResultSet rs = stmt.executeQuery("SELECT * FROM users");
        // ...
    }
    
    // UI + 비즈니스 + 데이터 접근이 뒤섞임!
    // 테스트 불가능!
    // 재사용 불가능!
}

// 문제 2: 의존성 스파게티
public class OrderController {
    public void createOrder() {
        // DB 직접 접근
        Connection conn = getConnection();
        
        // 비즈니스 로직
        double total = calculateTotal();
        
        // 결제 처리
        PaymentGateway.charge(total);
        
        // 재고 업데이트
        InventoryService.update();
        
        // 어떤 계층인지 알 수 없음!
        // 모든 게 섞여있음!
    }
}

// 문제 3: 데이터베이스 변경 시 전체 수정
public class ProductService {
    public void saveProduct(Product product) {
        // MySQL 특화 코드
        Connection conn = DriverManager.getConnection(
            "jdbc:mysql://localhost/mydb"
        );
        // ...
    }
    
    // PostgreSQL로 바꾸면?
    // → 모든 서비스 클래스 수정!
    // → DB 의존적인 코드가 전체에 퍼져있음!
}

// 문제 4: 테스트 불가능
public class ReportService {
    public void generateReport() {
        // DB 직접 접근
        Connection conn = DriverManager.getConnection(DB_URL);
        
        // 복잡한 비즈니스 로직
        List<Data> data = fetchData(conn);
        Report report = processData(data);
        
        // 파일 시스템 접근
        FileWriter writer = new FileWriter("report.txt");
        writer.write(report.toString());
        
        // 단위 테스트가 불가능!
        // DB, 파일 시스템에 의존!
    }
}
```

### ⚡ 핵심 문제

1. **관심사 혼재**: UI, 비즈니스, 데이터 접근이 뒤섞임
2. **강한 결합**: 모든 계층이 서로 직접 의존
3. **테스트 어려움**: 의존성 분리 불가
4. **유지보수 어려움**: 한 부분 변경이 전체에 영향
5. **재사용 불가**: 계층 분리가 안 되어 재사용 어려움

---

## 2. 패턴 정의

### 📖 정의

> **시스템을 수평적 계층으로 나누어 각 계층이 명확한 책임을 가지고, 상위 계층이 하위 계층에만 의존하도록 하는 아키텍처 패턴**

### 🎯 목적

- **관심사 분리**: 각 계층이 하나의 관심사만 담당
- **의존성 관리**: 단방향 의존성 (상위 → 하위)
- **교체 가능**: 각 계층을 독립적으로 교체
- **테스트 용이**: 계층별 독립 테스트

### 💡 핵심 아이디어

```java
// Before: 모든 게 섞여있음
public class Service {
    public void doSomething() {
        // UI 코드
        System.out.println("...");
        
        // 비즈니스 로직
        calculate();
        
        // DB 접근
        Connection conn = ...;
    }
}

// After: 계층 분리
// Presentation Layer
public class Controller {
    private Service service;
    public void handle() {
        service.execute();
    }
}

// Business Layer
public class Service {
    private Repository repository;
    public void execute() {
        Data data = repository.find();
        process(data);
    }
}

// Data Access Layer
public class Repository {
    public Data find() {
        // DB 접근
    }
}
```

---

## 3. 구조와 구성요소

### 📊 전통적 3계층 아키텍처

```
┌────────────────────────────────────┐
│   Presentation Layer (표현 계층)     │  ← UI, Controller
│   - HTTP 요청/응답 처리               │
│   - 입력 검증 (형식)                  │
│   - DTO 변환                        │
└────────────────────────────────────┘
              ↓ (의존)
┌────────────────────────────────────┐
│   Business Layer (비즈니스 계층)      │  ← Service, Domain
│   - 비즈니스 로직                     │
│   - 트랜잭션 관리                     │
│   - 비즈니스 규칙 검증                 │
└────────────────────────────────────┘
              ↓ (의존)
┌────────────────────────────────────┐
│   Data Access Layer (데이터 계층)     │  ← Repository, DAO
│   - 데이터 CRUD                      │
│   - 쿼리 실행                        │
│   - 데이터 매핑                       │
└────────────────────────────────────┘
              ↓ (의존)
┌────────────────────────────────────┐
│   Database (데이터베이스)             │
└────────────────────────────────────┘
```

### 🔧 현대적 4계층 아키텍처

```
┌─────────────────────────────────────┐
│   Presentation Layer                │
│   ┌─────────────────────────────┐   │
│   │ Controller / REST API       │   │
│   │ - @RestController           │   │
│   │ - Request/Response DTO      │   │
│   └─────────────────────────────┘   │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│   Application Layer (응용 계층)       │
│   ┌─────────────────────────────┐   │
│   │ Service / Use Case          │   │
│   │ - @Service                  │   │
│   │ - 트랜잭션 경계                │   │
│   │ - 오케스트레이션                │   │
│   └─────────────────────────────┘   │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│   Domain Layer (도메인 계층)           │
│   ┌─────────────────────────────┐   │
│   │ Entity / Domain Model       │   │
│   │ - 비즈니스 규칙                │   │
│   │ - 도메인 로직                  │   │
│   │ - Value Object              │   │
│   └─────────────────────────────┘   │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│   Infrastructure Layer (인프라)       │
│   ┌─────────────────────────────┐   │
│   │ Repository Implementation   │   │
│   │ - @Repository               │   │
│   │ - JPA / JDBC                │   │
│   │ - 외부 서비스 연동              │   │
│   └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

### 🏗️ 계층별 책임

| 계층 | 책임 | 기술 | 예시 |
|------|------|------|------|
| **Presentation** | UI, 요청/응답 처리 | Spring MVC, Thymeleaf | `@RestController` |
| **Application** | Use Case 구현 | Spring Service | `@Service` |
| **Domain** | 비즈니스 규칙 | Pure Java | `Entity`, `ValueObject` |
| **Infrastructure** | 기술적 구현 | JPA, JDBC | `@Repository` |

---

## 4. 구현 방법

### 전체 시스템: E-Commerce 주문 시스템 ⭐⭐⭐

```java
/**
 * ============================================
 * DOMAIN LAYER (도메인 계층)
 * ============================================
 * 비즈니스 규칙과 도메인 로직
 * 다른 계층에 의존하지 않음 (Pure Java)
 */

/**
 * Entity: 주문
 */
public class Order {
    private Long id;
    private Long customerId;
    private List<OrderItem> items;
    private OrderStatus status;
    private LocalDateTime createdAt;
    private BigDecimal totalAmount;
    
    public enum OrderStatus {
        PENDING, PAID, SHIPPED, DELIVERED, CANCELLED
    }
    
    public Order(Long customerId) {
        this.customerId = customerId;
        this.items = new ArrayList<>();
        this.status = OrderStatus.PENDING;
        this.createdAt = LocalDateTime.now();
        this.totalAmount = BigDecimal.ZERO;
    }
    
    /**
     * 도메인 로직: 상품 추가
     */
    public void addItem(Product product, int quantity) {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("주문이 이미 진행 중입니다");
        }
        
        OrderItem item = new OrderItem(product, quantity);
        items.add(item);
        calculateTotal();
        
        System.out.println("📦 상품 추가: " + product.getName() + " x " + quantity);
    }
    
    /**
     * 도메인 로직: 총액 계산
     */
    private void calculateTotal() {
        totalAmount = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
    
    /**
     * 도메인 로직: 결제 처리
     */
    public void pay() {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("이미 결제되었습니다");
        }
        
        if (items.isEmpty()) {
            throw new IllegalStateException("주문 항목이 없습니다");
        }
        
        this.status = OrderStatus.PAID;
        System.out.println("💳 결제 완료: " + totalAmount);
    }
    
    /**
     * 도메인 로직: 배송 시작
     */
    public void ship() {
        if (status != OrderStatus.PAID) {
            throw new IllegalStateException("결제되지 않았습니다");
        }
        
        this.status = OrderStatus.SHIPPED;
        System.out.println("🚚 배송 시작");
    }
    
    // Getters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public OrderStatus getStatus() { return status; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public List<OrderItem> getItems() { return items; }
}

/**
 * Value Object: 주문 항목
 */
public class OrderItem {
    private Product product;
    private int quantity;
    private BigDecimal price;
    
    public OrderItem(Product product, int quantity) {
        this.product = product;
        this.quantity = quantity;
        this.price = product.getPrice();
    }
    
    public BigDecimal getSubtotal() {
        return price.multiply(BigDecimal.valueOf(quantity));
    }
    
    public Product getProduct() { return product; }
    public int getQuantity() { return quantity; }
}

/**
 * Entity: 상품
 */
public class Product {
    private Long id;
    private String name;
    private BigDecimal price;
    private int stock;
    
    public Product(Long id, String name, BigDecimal price, int stock) {
        this.id = id;
        this.name = name;
        this.price = price;
        this.stock = stock;
    }
    
    /**
     * 도메인 로직: 재고 확인
     */
    public boolean hasStock(int quantity) {
        return stock >= quantity;
    }
    
    /**
     * 도메인 로직: 재고 차감
     */
    public void decreaseStock(int quantity) {
        if (!hasStock(quantity)) {
            throw new IllegalStateException("재고 부족: " + name);
        }
        this.stock -= quantity;
        System.out.println("📉 재고 차감: " + name + " (" + stock + "개 남음)");
    }
    
    // Getters
    public Long getId() { return id; }
    public String getName() { return name; }
    public BigDecimal getPrice() { return price; }
    public int getStock() { return stock; }
}

/**
 * ============================================
 * INFRASTRUCTURE LAYER (인프라 계층)
 * ============================================
 * 데이터 접근 및 외부 서비스 연동
 */

/**
 * Repository Interface (도메인 계층에 정의)
 */
public interface OrderRepository {
    Order save(Order order);
    Order findById(Long id);
    List<Order> findAll();
}

/**
 * Repository Implementation (인프라 계층에 구현)
 */
public class JpaOrderRepository implements OrderRepository {
    // 실제로는 EntityManager 사용
    private Map<Long, Order> database = new HashMap<>();
    private Long sequence = 1L;
    
    @Override
    public Order save(Order order) {
        if (order.getId() == null) {
            order.setId(sequence++);
            database.put(order.getId(), order);
            System.out.println("💾 주문 저장: ID=" + order.getId());
        } else {
            database.put(order.getId(), order);
            System.out.println("💾 주문 업데이트: ID=" + order.getId());
        }
        return order;
    }
    
    @Override
    public Order findById(Long id) {
        Order order = database.get(id);
        if (order == null) {
            throw new IllegalArgumentException("주문을 찾을 수 없음: " + id);
        }
        System.out.println("🔍 주문 조회: ID=" + id);
        return order;
    }
    
    @Override
    public List<Order> findAll() {
        return new ArrayList<>(database.values());
    }
}

/**
 * Product Repository
 */
public interface ProductRepository {
    Product findById(Long id);
    List<Product> findAll();
}

public class JpaProductRepository implements ProductRepository {
    private Map<Long, Product> database = new HashMap<>();
    
    public JpaProductRepository() {
        // 초기 데이터
        database.put(1L, new Product(1L, "노트북", new BigDecimal("1200000"), 10));
        database.put(2L, new Product(2L, "마우스", new BigDecimal("30000"), 50));
        database.put(3L, new Product(3L, "키보드", new BigDecimal("80000"), 30));
    }
    
    @Override
    public Product findById(Long id) {
        Product product = database.get(id);
        if (product == null) {
            throw new IllegalArgumentException("상품을 찾을 수 없음: " + id);
        }
        return product;
    }
    
    @Override
    public List<Product> findAll() {
        return new ArrayList<>(database.values());
    }
}

/**
 * ============================================
 * APPLICATION LAYER (응용 계층)
 * ============================================
 * Use Case 구현 및 트랜잭션 관리
 */

/**
 * DTO: 주문 생성 요청
 */
public class CreateOrderRequest {
    private Long customerId;
    private List<OrderItemRequest> items;
    
    public CreateOrderRequest(Long customerId, List<OrderItemRequest> items) {
        this.customerId = customerId;
        this.items = items;
    }
    
    public Long getCustomerId() { return customerId; }
    public List<OrderItemRequest> getItems() { return items; }
}

public class OrderItemRequest {
    private Long productId;
    private int quantity;
    
    public OrderItemRequest(Long productId, int quantity) {
        this.productId = productId;
        this.quantity = quantity;
    }
    
    public Long getProductId() { return productId; }
    public int getQuantity() { return quantity; }
}

/**
 * DTO: 주문 응답
 */
public class OrderResponse {
    private Long orderId;
    private String status;
    private BigDecimal totalAmount;
    private List<String> items;
    
    public OrderResponse(Order order) {
        this.orderId = order.getId();
        this.status = order.getStatus().name();
        this.totalAmount = order.getTotalAmount();
        this.items = order.getItems().stream()
            .map(item -> item.getProduct().getName() + " x " + item.getQuantity())
            .collect(Collectors.toList());
    }
    
    // Getters
    public Long getOrderId() { return orderId; }
    public String getStatus() { return status; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public List<String> getItems() { return items; }
    
    @Override
    public String toString() {
        return "OrderResponse{" +
                "orderId=" + orderId +
                ", status='" + status + '\'' +
                ", totalAmount=" + totalAmount +
                ", items=" + items +
                '}';
    }
}

/**
 * Service: 주문 서비스
 * - Use Case 오케스트레이션
 * - 트랜잭션 경계
 */
public class OrderService {
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    
    public OrderService(OrderRepository orderRepository, 
                       ProductRepository productRepository) {
        this.orderRepository = orderRepository;
        this.productRepository = productRepository;
    }
    
    /**
     * Use Case: 주문 생성
     * @Transactional 트랜잭션 경계
     */
    public OrderResponse createOrder(CreateOrderRequest request) {
        System.out.println("\n🛒 === 주문 생성 시작 ===");
        
        // 1. 도메인 객체 생성
        Order order = new Order(request.getCustomerId());
        
        // 2. 주문 항목 추가 (도메인 로직 호출)
        for (OrderItemRequest itemReq : request.getItems()) {
            Product product = productRepository.findById(itemReq.getProductId());
            
            // 재고 확인 (도메인 로직)
            if (!product.hasStock(itemReq.getQuantity())) {
                throw new IllegalStateException(
                    "재고 부족: " + product.getName()
                );
            }
            
            order.addItem(product, itemReq.getQuantity());
        }
        
        // 3. 저장
        Order savedOrder = orderRepository.save(order);
        
        System.out.println("✅ 주문 생성 완료\n");
        
        // 4. DTO 변환
        return new OrderResponse(savedOrder);
    }
    
    /**
     * Use Case: 주문 결제
     */
    public OrderResponse payOrder(Long orderId) {
        System.out.println("\n💳 === 결제 처리 시작 ===");
        
        // 1. 주문 조회
        Order order = orderRepository.findById(orderId);
        
        // 2. 재고 차감 (도메인 로직)
        for (OrderItem item : order.getItems()) {
            Product product = item.getProduct();
            product.decreaseStock(item.getQuantity());
        }
        
        // 3. 결제 처리 (도메인 로직)
        order.pay();
        
        // 4. 저장
        Order paidOrder = orderRepository.save(order);
        
        System.out.println("✅ 결제 완료\n");
        
        return new OrderResponse(paidOrder);
    }
    
    /**
     * Use Case: 주문 배송
     */
    public OrderResponse shipOrder(Long orderId) {
        System.out.println("\n🚚 === 배송 시작 ===");
        
        Order order = orderRepository.findById(orderId);
        order.ship(); // 도메인 로직
        
        Order shippedOrder = orderRepository.save(order);
        
        System.out.println("✅ 배송 시작 완료\n");
        
        return new OrderResponse(shippedOrder);
    }
    
    /**
     * Query: 주문 조회
     */
    public OrderResponse getOrder(Long orderId) {
        Order order = orderRepository.findById(orderId);
        return new OrderResponse(order);
    }
}

/**
 * ============================================
 * PRESENTATION LAYER (표현 계층)
 * ============================================
 * HTTP 요청/응답 처리, 입력 검증
 */

/**
 * Controller: REST API
 */
public class OrderController {
    private final OrderService orderService;
    
    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }
    
    /**
     * POST /orders - 주문 생성
     */
    public OrderResponse createOrder(CreateOrderRequest request) {
        System.out.println("📥 API 요청: POST /orders");
        
        // 입력 검증 (형식 검증만)
        validateCreateOrderRequest(request);
        
        // 서비스 호출
        OrderResponse response = orderService.createOrder(request);
        
        System.out.println("📤 API 응답: " + response);
        return response;
    }
    
    /**
     * POST /orders/{id}/pay - 결제
     */
    public OrderResponse payOrder(Long orderId) {
        System.out.println("📥 API 요청: POST /orders/" + orderId + "/pay");
        
        OrderResponse response = orderService.payOrder(orderId);
        
        System.out.println("📤 API 응답: " + response);
        return response;
    }
    
    /**
     * POST /orders/{id}/ship - 배송
     */
    public OrderResponse shipOrder(Long orderId) {
        System.out.println("📥 API 요청: POST /orders/" + orderId + "/ship");
        
        OrderResponse response = orderService.shipOrder(orderId);
        
        System.out.println("📤 API 응답: " + response);
        return response;
    }
    
    /**
     * GET /orders/{id} - 조회
     */
    public OrderResponse getOrder(Long orderId) {
        System.out.println("📥 API 요청: GET /orders/" + orderId);
        
        OrderResponse response = orderService.getOrder(orderId);
        
        System.out.println("📤 API 응답: " + response);
        return response;
    }
    
    /**
     * 입력 검증 (형식만)
     */
    private void validateCreateOrderRequest(CreateOrderRequest request) {
        if (request.getCustomerId() == null) {
            throw new IllegalArgumentException("고객 ID 필수");
        }
        
        if (request.getItems() == null || request.getItems().isEmpty()) {
            throw new IllegalArgumentException("주문 항목 필수");
        }
        
        for (OrderItemRequest item : request.getItems()) {
            if (item.getProductId() == null) {
                throw new IllegalArgumentException("상품 ID 필수");
            }
            if (item.getQuantity() <= 0) {
                throw new IllegalArgumentException("수량은 1 이상");
            }
        }
    }
}

/**
 * ============================================
 * APPLICATION (메인)
 * ============================================
 */
public class LayeredArchitectureExample {
    public static void main(String[] args) {
        // 의존성 주입 (DI Container 역할)
        ProductRepository productRepository = new JpaProductRepository();
        OrderRepository orderRepository = new JpaOrderRepository();
        OrderService orderService = new OrderService(orderRepository, productRepository);
        OrderController orderController = new OrderController(orderService);
        
        System.out.println("=== E-Commerce 주문 시스템 ===");
        System.out.println("계층형 아키텍처 데모\n");
        
        try {
            // 1. 주문 생성
            List<OrderItemRequest> items = Arrays.asList(
                new OrderItemRequest(1L, 1),  // 노트북 1개
                new OrderItemRequest(2L, 2)   // 마우스 2개
            );
            CreateOrderRequest createRequest = new CreateOrderRequest(100L, items);
            OrderResponse order = orderController.createOrder(createRequest);
            
            // 2. 결제
            orderController.payOrder(order.getOrderId());
            
            // 3. 배송
            orderController.shipOrder(order.getOrderId());
            
            // 4. 조회
            System.out.println("\n📋 === 최종 주문 상태 ===");
            OrderResponse finalOrder = orderController.getOrder(order.getOrderId());
            System.out.println(finalOrder);
            
        } catch (Exception e) {
            System.err.println("❌ 오류: " + e.getMessage());
            e.printStackTrace();
        }
    }
}
```

---

## 5. 실전 예제

### 예제 1: Spring Boot 실전 구조 ⭐⭐⭐

```java
/**
 * ============================================
 * 실전 Spring Boot 프로젝트 구조
 * ============================================
 */

/*
src/main/java/com/example/app/
├── presentation/           (Presentation Layer)
│   ├── controller/
│   │   └── UserController.java
│   └── dto/
│       ├── UserRequest.java
│       └── UserResponse.java
│
├── application/            (Application Layer)
│   ├── service/
│   │   └── UserService.java
│   └── usecase/           (Optional: Use Case 분리)
│       ├── CreateUserUseCase.java
│       └── UpdateUserUseCase.java
│
├── domain/                 (Domain Layer)
│   ├── model/
│   │   ├── User.java
│   │   └── Email.java
│   ├── repository/        (Interface만)
│   │   └── UserRepository.java
│   └── service/           (Domain Service)
│       └── UserDomainService.java
│
└── infrastructure/         (Infrastructure Layer)
    ├── persistence/
    │   ├── entity/
    │   │   └── UserEntity.java
    │   └── repository/
    │       └── JpaUserRepository.java
    └── external/
        └── EmailService.java
*/

/**
 * Presentation Layer
 */
@RestController
@RequestMapping("/api/users")
public class UserController {
    private final UserService userService;
    
    @Autowired
    public UserController(UserService userService) {
        this.userService = userService;
    }
    
    @PostMapping
    public ResponseEntity<UserResponse> createUser(@Valid @RequestBody UserRequest request) {
        // 1. DTO 검증 (형식)
        // @Valid가 자동 처리
        
        // 2. 서비스 호출
        User user = userService.createUser(
            request.getName(),
            request.getEmail(),
            request.getPassword()
        );
        
        // 3. DTO 변환
        UserResponse response = new UserResponse(user);
        
        return ResponseEntity.ok(response);
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok(new UserResponse(user));
    }
}

/**
 * Request DTO
 */
public class UserRequest {
    @NotBlank(message = "이름 필수")
    private String name;
    
    @Email(message = "이메일 형식")
    @NotBlank(message = "이메일 필수")
    private String email;
    
    @Size(min = 8, message = "비밀번호 8자 이상")
    private String password;
    
    // Getters, Setters
}

/**
 * Response DTO
 */
public class UserResponse {
    private Long id;
    private String name;
    private String email;
    
    public UserResponse(User user) {
        this.id = user.getId();
        this.name = user.getName();
        this.email = user.getEmail().getValue();
    }
    
    // Getters
}

/**
 * Application Layer
 */
@Service
@Transactional
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;
    
    @Autowired
    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
    
    /**
     * Use Case: 사용자 생성
     */
    public User createUser(String name, String emailStr, String password) {
        // 1. 도메인 객체 생성
        Email email = new Email(emailStr);
        User user = new User(name, email);
        
        // 2. 비밀번호 암호화 (도메인 로직)
        user.setPassword(password);
        
        // 3. 중복 체크 (비즈니스 규칙)
        if (userRepository.existsByEmail(email)) {
            throw new DuplicateEmailException("이미 존재하는 이메일");
        }
        
        // 4. 저장
        User savedUser = userRepository.save(user);
        
        // 5. 이메일 발송 (인프라 서비스)
        emailService.sendWelcomeEmail(savedUser);
        
        return savedUser;
    }
    
    @Transactional(readOnly = true)
    public User findById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
}

/**
 * Domain Layer
 */
public class User {
    private Long id;
    private String name;
    private Email email;
    private String hashedPassword;
    private LocalDateTime createdAt;
    
    public User(String name, Email email) {
        this.name = name;
        this.email = email;
        this.createdAt = LocalDateTime.now();
    }
    
    /**
     * 도메인 로직: 비밀번호 설정
     */
    public void setPassword(String rawPassword) {
        validatePassword(rawPassword);
        this.hashedPassword = BCrypt.hashpw(rawPassword, BCrypt.gensalt());
    }
    
    /**
     * 도메인 로직: 비밀번호 검증
     */
    private void validatePassword(String password) {
        if (password.length() < 8) {
            throw new InvalidPasswordException("비밀번호 8자 이상");
        }
    }
    
    // Getters
}

/**
 * Value Object: 이메일
 */
public class Email {
    private final String value;
    
    public Email(String value) {
        validateEmail(value);
        this.value = value;
    }
    
    private void validateEmail(String email) {
        if (!email.matches("^[A-Za-z0-9+_.-]+@(.+)$")) {
            throw new InvalidEmailException("잘못된 이메일 형식");
        }
    }
    
    public String getValue() {
        return value;
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Email)) return false;
        Email email = (Email) o;
        return Objects.equals(value, email.value);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(value);
    }
}

/**
 * Domain Repository Interface
 */
public interface UserRepository {
    User save(User user);
    Optional<User> findById(Long id);
    boolean existsByEmail(Email email);
}

/**
 * Infrastructure Layer
 */
@Repository
public class JpaUserRepository implements UserRepository {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public User save(User user) {
        // User → UserEntity 변환
        UserEntity entity = toEntity(user);
        entityManager.persist(entity);
        
        // UserEntity → User 변환
        return toDomain(entity);
    }
    
    @Override
    public Optional<User> findById(Long id) {
        UserEntity entity = entityManager.find(UserEntity.class, id);
        return Optional.ofNullable(entity)
            .map(this::toDomain);
    }
    
    @Override
    public boolean existsByEmail(Email email) {
        Long count = entityManager.createQuery(
            "SELECT COUNT(u) FROM UserEntity u WHERE u.email = :email",
            Long.class
        )
        .setParameter("email", email.getValue())
        .getSingleResult();
        
        return count > 0;
    }
    
    private UserEntity toEntity(User user) {
        UserEntity entity = new UserEntity();
        entity.setName(user.getName());
        entity.setEmail(user.getEmail().getValue());
        entity.setHashedPassword(user.getHashedPassword());
        return entity;
    }
    
    private User toDomain(UserEntity entity) {
        User user = new User(entity.getName(), new Email(entity.getEmail()));
        // ... 변환 로직
        return user;
    }
}

/**
 * JPA Entity (Infrastructure)
 */
@Entity
@Table(name = "users")
public class UserEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(unique = true, nullable = false)
    private String email;
    
    @Column(nullable = false)
    private String hashedPassword;
    
    @Column(nullable = false)
    private LocalDateTime createdAt;
    
    // Getters, Setters
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 실무 적용 |
|------|------|-----------|
| **관심사 분리** | 각 계층이 하나의 책임 | 유지보수 용이 |
| **테스트 용이** | 계층별 독립 테스트 | Mock 활용 |
| **교체 가능** | DB 변경 시 인프라만 수정 | MySQL → PostgreSQL |
| **이해 쉬움** | 직관적 구조 | 신규 개발자 온보딩 |
| **재사용** | 비즈니스 로직 재사용 | Web + Mobile API |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **성능** | 계층 간 데이터 변환 비용 | DTO 최소화 |
| **복잡도** | 간단한 CRUD도 여러 계층 | 상황에 맞게 조정 |
| **결합도** | 계층 간 순환 의존 가능 | DIP 적용 |

---

## 7. 안티패턴

### ❌ 안티패턴 1: 계층 건너뛰기

```java
// 잘못된 예: Controller가 Repository 직접 접근
@RestController
public class UserController {
    @Autowired
    private UserRepository repository; // ❌
    
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return repository.findById(id); // ❌ Service 건너뜀
    }
}
```

**해결:**
```java
// 올바른 예: Service를 통해 접근
@RestController
public class UserController {
    @Autowired
    private UserService service; // ✅
    
    @GetMapping("/users/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        User user = service.findById(id); // ✅
        return new UserResponse(user);
    }
}
```

### ❌ 안티패턴 2: 도메인 로직이 Service에

```java
// 잘못된 예: 빈약한 도메인 모델 (Anemic Domain Model)
public class User {
    private String name;
    private String email;
    // Getter/Setter만 있음 (로직 없음)
}

@Service
public class UserService {
    public void updateEmail(User user, String newEmail) {
        // ❌ 도메인 로직이 Service에!
        if (!newEmail.contains("@")) {
            throw new Exception("잘못된 이메일");
        }
        user.setEmail(newEmail);
    }
}
```

**해결:**
```java
// 올바른 예: 풍부한 도메인 모델 (Rich Domain Model)
public class User {
    private Email email;
    
    public void changeEmail(String newEmail) {
        // ✅ 도메인 로직이 Entity에!
        this.email = new Email(newEmail); // 검증 포함
    }
}

@Service
public class UserService {
    public void updateEmail(Long userId, String newEmail) {
        User user = repository.findById(userId);
        user.changeEmail(newEmail); // ✅ 도메인 로직 호출
        repository.save(user);
    }
}
```

---

## 8. 심화 주제

### 🎯 DIP (Dependency Inversion Principle) 적용

```java
// 상위 계층(Domain)이 하위 계층(Infrastructure)에 의존 ❌
// → 의존성 역전!

/**
 * Domain Layer에 인터페이스 정의
 */
package com.example.domain.repository;

public interface UserRepository {
    User save(User user);
    Optional<User> findById(Long id);
}

/**
 * Infrastructure Layer에 구현
 */
package com.example.infrastructure.persistence;

public class JpaUserRepository implements UserRepository {
    // JPA 구현
}

// 의존성 방향:
// Domain (interface) ← Infrastructure (implementation)
// 상위 계층이 하위 계층에 의존하지 않음!
```

### 🔥 Cross-Cutting Concerns (횡단 관심사)

```java
/**
 * AOP로 횡단 관심사 처리
 */
@Aspect
@Component
public class LoggingAspect {
    
    // 모든 Service 계층 메서드에 로깅
    @Around("execution(* com.example.application.service.*.*(..))")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        
        Object result = joinPoint.proceed();
        
        long executionTime = System.currentTimeMillis() - start;
        System.out.println(joinPoint.getSignature() + " 실행 시간: " + executionTime + "ms");
        
        return result;
    }
}

// 트랜잭션, 보안, 캐싱 등도 AOP로 처리
```

---

## 9. 핵심 정리

### 📌 계층형 아키텍처 체크리스트

```
✅ 계층 분리 (Presentation, Application, Domain, Infrastructure)
✅ 단방향 의존성 (상위 → 하위)
✅ DTO 변환 (계층 경계에서)
✅ 도메인 로직은 Domain Layer에
✅ Repository 인터페이스는 Domain에, 구현은 Infrastructure에
✅ Service는 오케스트레이션만
✅ 트랜잭션 경계는 Service
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 엔터프라이즈 애플리케이션 | ⭐⭐⭐ | 표준 아키텍처 |
| 중대형 프로젝트 | ⭐⭐⭐ | 유지보수성 |
| 팀 협업 | ⭐⭐⭐ | 역할 분담 |
| 간단한 CRUD | ⭐ | 오버엔지니어링 |

### 💡 핵심 원칙

1. **각 계층은 하나의 책임만**
2. **상위 계층은 하위 계층에만 의존**
3. **도메인 로직은 Domain Layer에**
4. **인프라 의존성 격리**

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[다음: MVC →](./02-MVC.md)**

</div>
