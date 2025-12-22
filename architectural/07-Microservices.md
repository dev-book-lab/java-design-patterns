# Microservices Architecture Pattern (마이크로서비스 아키텍처)

> **"작고 독립적인 서비스들의 조합으로 시스템을 구축하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [핵심 패턴들](#6-핵심-패턴들)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: Monolithic 애플리케이션 (모놀리스)
@SpringBootApplication
public class ECommerceApplication {
    // 😱 모든 기능이 하나의 애플리케이션에!
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private ProductService productService;
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private ShippingService shippingService;
    
    @Autowired
    private NotificationService notificationService;
    
    @Autowired
    private AnalyticsService analyticsService;
    
    // 문제점:
    // 1. 배포 단위가 너무 큼 (전체 재배포)
    // 2. 확장이 어려움 (전체를 스케일)
    // 3. 한 부분 장애가 전체 영향
    // 4. 기술 스택 변경 어려움
    // 5. 팀 간 코드 충돌
}

// 문제 2: 강한 결합 (Tight Coupling)
public class OrderService {
    @Autowired
    private ProductService productService;  // 직접 의존
    
    @Autowired
    private InventoryService inventoryService;  // 직접 의존
    
    @Autowired
    private PaymentService paymentService;  // 직접 의존
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 😱 모든 서비스가 같은 트랜잭션에!
        Product product = productService.getProduct(request.getProductId());
        
        if (!inventoryService.checkStock(product.getId())) {
            throw new OutOfStockException();
        }
        
        Order order = new Order(request);
        orderRepository.save(order);
        
        paymentService.processPayment(order);
        
        // 하나의 서비스 실패 시 전체 롤백!
        // 서비스 간 강한 결합!
    }
}

// 문제 3: 단일 데이터베이스
@Entity
@Table(name = "orders")
public class Order {
    @Id
    private Long id;
    
    @ManyToOne  // 😱 다른 도메인 직접 참조
    @JoinColumn(name = "user_id")
    private User user;
    
    @ManyToOne
    @JoinColumn(name = "product_id")
    private Product product;
    
    // 문제점:
    // - 모든 데이터가 하나의 DB에
    // - 도메인 간 강한 결합
    // - DB 확장 어려움
    // - 스키마 변경 시 영향 범위 큼
}

// 문제 4: 전체 배포 (Big Bang Deployment)
public class DeploymentProcess {
    public void deploy() {
        // 😱 전체 애플리케이션 배포
        
        // 1. 서비스 중단
        stopApplication();
        
        // 2. 새 버전 배포
        deployNewVersion();
        
        // 3. 서비스 시작
        startApplication();
        
        // 문제점:
        // - 다운타임 발생
        // - 롤백 어려움
        // - 위험도 높음
        // - 작은 변경도 전체 배포
    }
}

// 문제 5: 확장성 문제
public class ScalingProblem {
    // 😱 특정 기능만 확장 불가
    
    // 주문 서비스는 트래픽이 많음
    // 사용자 서비스는 트래픽이 적음
    
    // 하지만 전체를 스케일해야 함!
    // → 비용 낭비
    // → 리소스 비효율
}

// 문제 6: 기술 스택 고정
public class TechnologyStack {
    // 😱 한번 선택한 기술을 계속 사용
    
    // Spring Boot로 시작
    // → 모든 기능이 Spring Boot
    // → 새로운 기술 도입 어려움
    
    // 예시:
    // - 추천 엔진: Python이 더 적합
    // - 실시간 분석: Node.js가 더 적합
    // - 이미지 처리: Go가 더 적합
    
    // 하지만 모놀리스에서는 불가능!
}

// 문제 7: 팀 협업 어려움
public class TeamCollaboration {
    // 😱 여러 팀이 하나의 코드베이스
    
    // 팀 A: 사용자 기능 개발
    // 팀 B: 주문 기능 개발
    // 팀 C: 결제 기능 개발
    
    // 문제점:
    // - Git 충돌 빈번
    // - 서로의 코드에 영향
    // - 배포 조율 필요
    // - 책임 범위 불명확
}

// 문제 8: 장애 전파
public class FailurePropagation {
    public void processOrder() {
        // 😱 하나의 기능 장애가 전체 영향
        
        try {
            // 추천 시스템 호출
            List<Product> recommendations = 
                recommendationService.getRecommendations();
            
        } catch (Exception e) {
            // 추천 시스템 장애
            // → 전체 주문 프로세스 중단!
            // → 사용자는 주문 불가!
        }
        
        // 중요하지 않은 기능의 장애가
        // 핵심 기능까지 영향!
    }
}

// 문제 9: 긴 빌드 시간
public class BuildTime {
    public void build() {
        // 😱 전체 애플리케이션 빌드
        
        // 코드베이스가 커질수록
        // 빌드 시간 증가
        
        // 작은 변경에도
        // 전체 빌드 + 테스트
        
        // 개발자 생산성 저하!
    }
}

// 문제 10: 모니터링과 디버깅
public class MonitoringProblem {
    public void monitor() {
        // 😱 어디서 문제가 발생했는지?
        
        // 로그가 모두 섞임
        // 성능 병목 지점 찾기 어려움
        // 특정 기능만 모니터링 불가
        
        // 문제 원인 파악에 시간 소요
    }
}
```

### ⚡ 핵심 문제

1. **배포 단위**: 전체 애플리케이션 재배포
2. **확장성**: 부분 확장 불가
3. **장애 격리**: 한 부분 장애가 전체 영향
4. **기술 선택**: 단일 기술 스택 고정
5. **팀 협업**: 코드베이스 충돌
6. **빌드 시간**: 전체 빌드 필요
7. **복잡도**: 코드베이스 거대화

---

## 2. 패턴 정의

### 📖 정의

> **애플리케이션을 작고 독립적인 서비스들로 분해하여, 각 서비스가 자체 프로세스로 실행되고 경량 메커니즘(HTTP/메시지)으로 통신하는 아키텍처 패턴**

### 🎯 목적

- **독립 배포**: 각 서비스를 독립적으로 배포
- **기술 다양성**: 서비스마다 최적 기술 선택
- **확장성**: 필요한 서비스만 확장
- **장애 격리**: 한 서비스 장애가 전체에 영향 안 줌

### 💡 핵심 아이디어

```
// Before: Monolithic
┌─────────────────────────────────┐
│     E-Commerce Application      │
│                                 │
│  - User Management              │
│  - Product Catalog              │
│  - Order Processing             │
│  - Payment                      │
│  - Inventory                    │
│  - Shipping                     │
│                                 │
│  (하나의 거대한 애플리케이션)          │
└─────────────────────────────────┘

// After: Microservices
┌──────────┐  ┌──────────┐  ┌──────────┐
│  User    │  │ Product  │  │  Order   │
│ Service  │  │ Service  │  │ Service  │
└──────────┘  └──────────┘  └──────────┘
     │             │             │
     └─────────────┴─────────────┘
              │
         (HTTP/메시지로 통신)
              │
     ┌─────────────────────┐
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Payment  │  │Inventory │  │ Shipping │
│ Service  │  │ Service  │  │ Service  │
└──────────┘  └──────────┘  └──────────┘

// 각 서비스:
// - 독립적으로 배포
// - 자체 데이터베이스
// - 독립적으로 확장
// - 다른 기술 스택 가능
```

---

## 3. 구조와 구성요소

### 📊 Microservices 구조

```
                    ┌─────────────┐
                    │   Client    │
                    └─────────────┘
                          │
                          │ HTTPS
                          ▼
                  ┌───────────────┐
                  │  API Gateway  │  ← 단일 진입점
                  └───────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   User       │  │   Product    │  │   Order      │
│   Service    │  │   Service    │  │   Service    │
│              │  │              │  │              │
│ - Spring     │  │ - Node.js    │  │ - Spring     │
│ - PostgreSQL │  │ - MongoDB    │  │ - MySQL      │
└──────────────┘  └──────────────┘  └──────────────┘
        │                 │                 │
        │                 │                 │
        │  (각 서비스는 독립적으로 실행, 배포, 확장) │
        │                 │                 │
        └─────────────────┼─────────────────┘
                          │
                  ┌───────────────┐
                  │ Message Queue │  ← 비동기 통신
                  │  (RabbitMQ)   │
                  └───────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Payment    │  │  Inventory   │  │  Shipping    │
│   Service    │  │   Service    │  │   Service    │
└──────────────┘  └──────────────┘  └──────────────┘
```

### 🔄 서비스 간 통신

```
1. 동기 통신 (Synchronous)
   Service A ──HTTP REST──> Service B
   Service A ──gRPC──────> Service B

2. 비동기 통신 (Asynchronous)
   Service A ──Message──> [Queue] ──> Service B
   Service A ──Event───> [Event Bus] ──> Service B

3. 하이브리드
   Service A ──HTTP──> Service B
       │
       └──Event──> [Event Bus] ──> Service C
```

### 🏗️ 핵심 컴포넌트

```
┌────────────────────────────────────────┐
│         Infrastructure                 │
│                                        │
│  - API Gateway                         │
│  - Service Discovery                   │
│  - Config Server                       │
│  - Load Balancer                       │
└────────────────────────────────────────┘
                  │
                  │ uses
                  ▼
┌────────────────────────────────────────┐
│         Microservices                  │
│                                        │
│  ┌──────────┐  ┌──────────┐            │
│  │Service A │  │Service B │            │
│  │          │  │          │            │
│  │ - API    │  │ - API    │            │
│  │ - Logic  │  │ - Logic  │            │
│  │ - DB     │  │ - DB     │            │
│  └──────────┘  └──────────┘            │
└────────────────────────────────────────┘
                  │
                  │ monitored by
                  ▼
┌────────────────────────────────────────┐
│      Observability                     │
│                                        │
│  - Logging (ELK)                       │
│  - Monitoring (Prometheus)             │
│  - Tracing (Zipkin)                    │
└────────────────────────────────────────┘
```

### 🔧 구성요소

| 컴포넌트 | 역할 | 기술 예시 |
|---------|------|-----------|
| **API Gateway** | 단일 진입점, 라우팅 | Spring Cloud Gateway, Kong |
| **Service Discovery** | 서비스 위치 찾기 | Eureka, Consul |
| **Config Server** | 중앙 설정 관리 | Spring Cloud Config |
| **Message Queue** | 비동기 통신 | RabbitMQ, Kafka |
| **Circuit Breaker** | 장애 격리 | Resilience4j, Hystrix |
| **Load Balancer** | 부하 분산 | Ribbon, Nginx |

---

## 4. 구현 방법

### 완전한 구현: E-Commerce 마이크로서비스 ⭐⭐⭐

```java
/**
 * ============================================
 * SERVICE 1: USER SERVICE (사용자 서비스)
 * ============================================
 * Port: 8081
 * DB: PostgreSQL
 * Tech: Spring Boot
 */

/**
 * User Entity
 */
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String email;
    
    @Column(nullable = false)
    private String name;
    
    private String phone;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    // Getters, Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getPhone() { return phone; }
    public void setPhone(String phone) { this.phone = phone; }
}

/**
 * User Repository
 */
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
}

/**
 * User Service
 */
@Service
public class UserService {
    private final UserRepository userRepository;
    
    @Autowired
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    public User createUser(String email, String name, String phone) {
        User user = new User();
        user.setEmail(email);
        user.setName(name);
        user.setPhone(phone);
        user.setCreatedAt(LocalDateTime.now());
        
        return userRepository.save(user);
    }
    
    public User getUser(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
}

/**
 * User REST Controller
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
    public ResponseEntity<UserResponse> createUser(@RequestBody CreateUserRequest request) {
        User user = userService.createUser(
            request.getEmail(),
            request.getName(),
            request.getPhone()
        );
        
        return ResponseEntity.ok(new UserResponse(user));
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        User user = userService.getUser(id);
        return ResponseEntity.ok(new UserResponse(user));
    }
}

/**
 * User Service Application
 */
@SpringBootApplication
@EnableDiscoveryClient  // Service Discovery 활성화
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}

/**
 * application.yml (User Service)
 */
/*
spring:
  application:
    name: user-service
  datasource:
    url: jdbc:postgresql://localhost:5432/userdb
    username: postgres
    password: postgres

server:
  port: 8081

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
*/

/**
 * ============================================
 * SERVICE 2: PRODUCT SERVICE (상품 서비스)
 * ============================================
 * Port: 8082
 * DB: MongoDB
 * Tech: Spring Boot
 */

/**
 * Product Document (MongoDB)
 */
@Document(collection = "products")
public class Product {
    @Id
    private String id;
    
    private String name;
    private String description;
    private BigDecimal price;
    private Integer stock;
    private String category;
    
    // Getters, Setters
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public BigDecimal getPrice() { return price; }
    public void setPrice(BigDecimal price) { this.price = price; }
    public Integer getStock() { return stock; }
    public void setStock(Integer stock) { this.stock = stock; }
}

/**
 * Product Repository (MongoDB)
 */
public interface ProductRepository extends MongoRepository<Product, String> {
    List<Product> findByCategory(String category);
}

/**
 * Product Service
 */
@Service
public class ProductService {
    private final ProductRepository productRepository;
    
    @Autowired
    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }
    
    public Product createProduct(String name, BigDecimal price, Integer stock) {
        Product product = new Product();
        product.setName(name);
        product.setPrice(price);
        product.setStock(stock);
        
        return productRepository.save(product);
    }
    
    public Product getProduct(String id) {
        return productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
    }
    
    public boolean checkStock(String productId, int quantity) {
        Product product = getProduct(productId);
        return product.getStock() >= quantity;
    }
    
    public void decreaseStock(String productId, int quantity) {
        Product product = getProduct(productId);
        
        if (product.getStock() < quantity) {
            throw new InsufficientStockException(productId);
        }
        
        product.setStock(product.getStock() - quantity);
        productRepository.save(product);
    }
}

/**
 * Product REST Controller
 */
@RestController
@RequestMapping("/api/products")
public class ProductController {
    private final ProductService productService;
    
    @Autowired
    public ProductController(ProductService productService) {
        this.productService = productService;
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getProduct(@PathVariable String id) {
        Product product = productService.getProduct(id);
        return ResponseEntity.ok(new ProductResponse(product));
    }
    
    @PostMapping("/{id}/check-stock")
    public ResponseEntity<Boolean> checkStock(
            @PathVariable String id,
            @RequestParam int quantity) {
        
        boolean available = productService.checkStock(id, quantity);
        return ResponseEntity.ok(available);
    }
}

/**
 * ============================================
 * SERVICE 3: ORDER SERVICE (주문 서비스)
 * ============================================
 * Port: 8083
 * DB: MySQL
 * Tech: Spring Boot
 */

/**
 * Order Entity
 */
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "user_id", nullable = false)
    private Long userId;  // ← User 객체가 아닌 ID만! (느슨한 결합)
    
    @Column(name = "product_id", nullable = false)
    private String productId;  // ← Product ID만!
    
    private Integer quantity;
    
    @Column(name = "total_amount")
    private BigDecimal totalAmount;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    public enum OrderStatus {
        PENDING, CONFIRMED, PAID, SHIPPED, DELIVERED, CANCELLED
    }
    
    // Getters, Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public Long getUserId() { return userId; }
    public void setUserId(Long userId) { this.userId = userId; }
    public String getProductId() { return productId; }
    public void setProductId(String productId) { this.productId = productId; }
    public Integer getQuantity() { return quantity; }
    public void setQuantity(Integer quantity) { this.quantity = quantity; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public void setTotalAmount(BigDecimal totalAmount) { this.totalAmount = totalAmount; }
    public OrderStatus getStatus() { return status; }
    public void setStatus(OrderStatus status) { this.status = status; }
}

/**
 * Order Repository
 */
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByUserId(Long userId);
}

/**
 * External Service Clients (Feign)
 */
@FeignClient(name = "user-service")  // Service Discovery로 찾음
public interface UserServiceClient {
    @GetMapping("/api/users/{id}")
    UserResponse getUser(@PathVariable Long id);
}

@FeignClient(name = "product-service")
public interface ProductServiceClient {
    @GetMapping("/api/products/{id}")
    ProductResponse getProduct(@PathVariable String id);
    
    @PostMapping("/api/products/{id}/check-stock")
    Boolean checkStock(@PathVariable String id, @RequestParam int quantity);
}

/**
 * Order Service (핵심 오케스트레이션)
 */
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final UserServiceClient userServiceClient;
    private final ProductServiceClient productServiceClient;
    
    @Autowired
    public OrderService(
            OrderRepository orderRepository,
            UserServiceClient userServiceClient,
            ProductServiceClient productServiceClient) {
        this.orderRepository = orderRepository;
        this.userServiceClient = userServiceClient;
        this.productServiceClient = productServiceClient;
    }
    
    /**
     * 주문 생성 (여러 마이크로서비스 조합)
     */
    public Order createOrder(Long userId, String productId, int quantity) {
        System.out.println("\n🛒 주문 생성 시작");
        
        // 1. User Service 호출 (HTTP)
        System.out.println("→ User Service 호출");
        UserResponse user = userServiceClient.getUser(userId);
        System.out.println("  ✅ 사용자 확인: " + user.getName());
        
        // 2. Product Service 호출 (HTTP)
        System.out.println("→ Product Service 호출");
        ProductResponse product = productServiceClient.getProduct(productId);
        System.out.println("  ✅ 상품 확인: " + product.getName());
        
        // 3. 재고 확인
        System.out.println("→ 재고 확인");
        Boolean stockAvailable = productServiceClient.checkStock(productId, quantity);
        
        if (!stockAvailable) {
            throw new InsufficientStockException(productId);
        }
        System.out.println("  ✅ 재고 충분");
        
        // 4. 주문 생성
        Order order = new Order();
        order.setUserId(userId);
        order.setProductId(productId);
        order.setQuantity(quantity);
        order.setTotalAmount(product.getPrice().multiply(BigDecimal.valueOf(quantity)));
        order.setStatus(Order.OrderStatus.PENDING);
        order.setCreatedAt(LocalDateTime.now());
        
        Order savedOrder = orderRepository.save(order);
        System.out.println("✅ 주문 생성 완료: ID=" + savedOrder.getId());
        
        return savedOrder;
    }
    
    public List<Order> getUserOrders(Long userId) {
        return orderRepository.findByUserId(userId);
    }
}

/**
 * Order REST Controller
 */
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final OrderService orderService;
    
    @Autowired
    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }
    
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(@RequestBody CreateOrderRequest request) {
        Order order = orderService.createOrder(
            request.getUserId(),
            request.getProductId(),
            request.getQuantity()
        );
        
        return ResponseEntity.ok(new OrderResponse(order));
    }
    
    @GetMapping("/user/{userId}")
    public ResponseEntity<List<OrderResponse>> getUserOrders(@PathVariable Long userId) {
        List<Order> orders = orderService.getUserOrders(userId);
        
        List<OrderResponse> responses = orders.stream()
            .map(OrderResponse::new)
            .collect(Collectors.toList());
        
        return ResponseEntity.ok(responses);
    }
}

/**
 * ============================================
 * INFRASTRUCTURE: API GATEWAY
 * ============================================
 * Port: 8080
 * Tech: Spring Cloud Gateway
 */

/**
 * API Gateway Configuration
 */
@Configuration
public class GatewayConfig {
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            // User Service 라우팅
            .route("user-service", r -> r
                .path("/api/users/**")
                .uri("lb://user-service"))  // Load Balanced
            
            // Product Service 라우팅
            .route("product-service", r -> r
                .path("/api/products/**")
                .uri("lb://product-service"))
            
            // Order Service 라우팅
            .route("order-service", r -> r
                .path("/api/orders/**")
                .uri("lb://order-service"))
            
            .build();
    }
}

/**
 * API Gateway Application
 */
@SpringBootApplication
@EnableDiscoveryClient
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}

/**
 * ============================================
 * INFRASTRUCTURE: SERVICE DISCOVERY
 * ============================================
 * Port: 8761
 * Tech: Netflix Eureka
 */

/**
 * Eureka Server
 */
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}

/**
 * application.yml (Eureka Server)
 */
/*
spring:
  application:
    name: eureka-server

server:
  port: 8761

eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
*/

/**
 * ============================================
 * DTOs (Data Transfer Objects)
 * ============================================
 */

/**
 * User Response
 */
public class UserResponse {
    private Long id;
    private String email;
    private String name;
    private String phone;
    
    public UserResponse(User user) {
        this.id = user.getId();
        this.email = user.getEmail();
        this.name = user.getName();
        this.phone = user.getPhone();
    }
    
    // Getters
    public Long getId() { return id; }
    public String getEmail() { return email; }
    public String getName() { return name; }
    public String getPhone() { return phone; }
}

/**
 * Product Response
 */
public class ProductResponse {
    private String id;
    private String name;
    private BigDecimal price;
    private Integer stock;
    
    public ProductResponse(Product product) {
        this.id = product.getId();
        this.name = product.getName();
        this.price = product.getPrice();
        this.stock = product.getStock();
    }
    
    // Getters
    public String getId() { return id; }
    public String getName() { return name; }
    public BigDecimal getPrice() { return price; }
    public Integer getStock() { return stock; }
}

/**
 * Order Response
 */
public class OrderResponse {
    private Long id;
    private Long userId;
    private String productId;
    private Integer quantity;
    private BigDecimal totalAmount;
    private String status;
    
    public OrderResponse(Order order) {
        this.id = order.getId();
        this.userId = order.getUserId();
        this.productId = order.getProductId();
        this.quantity = order.getQuantity();
        this.totalAmount = order.getTotalAmount();
        this.status = order.getStatus().name();
    }
    
    // Getters
    public Long getId() { return id; }
    public Long getUserId() { return userId; }
    public String getProductId() { return productId; }
    public Integer getQuantity() { return quantity; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public String getStatus() { return status; }
}

/**
 * Create Order Request
 */
public class CreateOrderRequest {
    private Long userId;
    private String productId;
    private Integer quantity;
    
    // Getters, Setters
    public Long getUserId() { return userId; }
    public void setUserId(Long userId) { this.userId = userId; }
    public String getProductId() { return productId; }
    public void setProductId(String productId) { this.productId = productId; }
    public Integer getQuantity() { return quantity; }
    public void setQuantity(Integer quantity) { this.quantity = quantity; }
}

/**
 * ============================================
 * EXCEPTIONS
 * ============================================
 */
class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(Long id) {
        super("User not found: " + id);
    }
}

class ProductNotFoundException extends RuntimeException {
    public ProductNotFoundException(String id) {
        super("Product not found: " + id);
    }
}

class InsufficientStockException extends RuntimeException {
    public InsufficientStockException(String productId) {
        super("Insufficient stock: " + productId);
    }
}

/**
 * ============================================
 * DEMO CLIENT
 * ============================================
 */
public class MicroservicesDemo {
    public static void main(String[] args) {
        System.out.println("=== Microservices Architecture 예제 ===\n");
        
        System.out.println("🚀 서비스 시작");
        System.out.println("  - Eureka Server: http://localhost:8761");
        System.out.println("  - API Gateway:    http://localhost:8080");
        System.out.println("  - User Service:   http://localhost:8081");
        System.out.println("  - Product Service: http://localhost:8082");
        System.out.println("  - Order Service:  http://localhost:8083");
        
        System.out.println("\n" + "=".repeat(60));
        
        // API Gateway를 통한 요청
        System.out.println("\n📥 클라이언트 → API Gateway → Order Service");
        System.out.println("POST http://localhost:8080/api/orders");
        System.out.println("Body: {userId: 1, productId: 'prod-1', quantity: 2}");
        
        System.out.println("\n→ API Gateway가 요청 라우팅");
        System.out.println("→ Order Service가 요청 수신");
        System.out.println("→ Order Service가 User Service 호출");
        System.out.println("→ Order Service가 Product Service 호출");
        System.out.println("→ 주문 생성 완료");
        
        System.out.println("\n" + "=".repeat(60));
        System.out.println("\n✅ 마이크로서비스 간 통신 완료!");
    }
}
```

**실행 흐름:**
```
1. Eureka Server 시작 (8761)
   → Service Discovery 준비

2. User Service 시작 (8081)
   → Eureka에 등록
   → PostgreSQL 연결

3. Product Service 시작 (8082)
   → Eureka에 등록
   → MongoDB 연결

4. Order Service 시작 (8083)
   → Eureka에 등록
   → MySQL 연결
   → Feign Client 준비

5. API Gateway 시작 (8080)
   → Eureka에 등록
   → 라우팅 규칙 설정

6. 클라이언트 요청
   POST http://localhost:8080/api/orders
   {
     "userId": 1,
     "productId": "prod-1",
     "quantity": 2
   }

7. 처리 흐름:
   Client → API Gateway (8080)
          → Order Service (8083)
             → User Service (8081) [사용자 확인]
             → Product Service (8082) [상품 확인]
             → Product Service (8082) [재고 확인]
          → Order 생성 완료
   Client ← 응답 반환
```

---

## 5. 실전 예제

### 예제 1: Circuit Breaker (장애 격리) ⭐⭐⭐

```java
/**
 * ============================================
 * Resilience4j Circuit Breaker
 * ============================================
 * 한 서비스 장애가 다른 서비스에 영향 안 주도록
 */

/**
 * Circuit Breaker 설정
 */
@Configuration
public class CircuitBreakerConfig {
    
    @Bean
    public CircuitBreakerRegistry circuitBreakerRegistry() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)  // 실패율 50% 초과 시
            .waitDurationInOpenState(Duration.ofSeconds(30))  // 30초 대기
            .slidingWindowSize(10)  // 최근 10개 요청 기준
            .build();
        
        return CircuitBreakerRegistry.of(config);
    }
}

/**
 * Circuit Breaker 적용
 */
@Service
public class OrderServiceWithCircuitBreaker {
    private final UserServiceClient userServiceClient;
    private final CircuitBreaker circuitBreaker;
    
    @Autowired
    public OrderServiceWithCircuitBreaker(
            UserServiceClient userServiceClient,
            CircuitBreakerRegistry registry) {
        this.userServiceClient = userServiceClient;
        this.circuitBreaker = registry.circuitBreaker("user-service");
    }
    
    /**
     * Circuit Breaker로 보호된 호출
     */
    public Order createOrder(Long userId, String productId, int quantity) {
        // Circuit Breaker로 감싸기
        Supplier<UserResponse> userSupplier = CircuitBreaker
            .decorateSupplier(circuitBreaker, () -> 
                userServiceClient.getUser(userId)
            );
        
        try {
            UserResponse user = userSupplier.get();
            
            // 정상 처리
            return processOrder(user, productId, quantity);
            
        } catch (CallNotPermittedException e) {
            // Circuit Open! (서비스 장애)
            System.err.println("⚠️ Circuit Breaker OPEN: User Service 불가");
            
            // Fallback 처리
            return createOrderWithFallback(userId, productId, quantity);
        }
    }
    
    /**
     * Fallback 처리
     */
    private Order createOrderWithFallback(Long userId, String productId, int quantity) {
        System.out.println("🔄 Fallback: 기본 사용자 정보로 주문 생성");
        
        // 주문은 생성하되, 사용자 정보는 나중에 업데이트
        Order order = new Order();
        order.setUserId(userId);
        order.setProductId(productId);
        order.setQuantity(quantity);
        order.setStatus(Order.OrderStatus.PENDING);
        
        return orderRepository.save(order);
    }
    
    private Order processOrder(UserResponse user, String productId, int quantity) {
        // 정상 주문 처리
        return null;
    }
}
```

---

### 예제 2: API Gateway Patterns ⭐⭐⭐

```java
/**
 * ============================================
 * API Gateway 고급 패턴
 * ============================================
 */

/**
 * Rate Limiting (요청 제한)
 */
@Configuration
public class RateLimitingConfig {
    
    @Bean
    public RouteLocator rateLimitedRoutes(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("user-service", r -> r
                .path("/api/users/**")
                .filters(f -> f
                    .requestRateLimiter(c -> c
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver()))
                )
                .uri("lb://user-service"))
            .build();
    }
    
    @Bean
    public KeyResolver userKeyResolver() {
        // IP 기반 제한
        return exchange -> Mono.just(
            exchange.getRequest()
                .getRemoteAddress()
                .getAddress()
                .getHostAddress()
        );
    }
    
    @Bean
    public RedisRateLimiter redisRateLimiter() {
        return new RedisRateLimiter(
            10,   // replenishRate: 초당 10개
            20    // burstCapacity: 최대 20개
        );
    }
}

/**
 * Request/Response 변환
 */
@Component
public class RequestResponseFilter implements GlobalFilter {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        
        // 공통 헤더 추가
        ServerHttpRequest modifiedRequest = request.mutate()
            .header("X-Request-Id", UUID.randomUUID().toString())
            .header("X-Gateway-Time", LocalDateTime.now().toString())
            .build();
        
        return chain.filter(exchange.mutate().request(modifiedRequest).build())
            .then(Mono.fromRunnable(() -> {
                // 응답 후처리
                ServerHttpResponse response = exchange.getResponse();
                response.getHeaders().add("X-Response-Time", 
                    LocalDateTime.now().toString());
            }));
    }
}

/**
 * Authentication (인증)
 */
@Component
public class AuthenticationFilter implements GlobalFilter, Ordered {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        
        // JWT 토큰 검증
        String token = request.getHeaders().getFirst("Authorization");
        
        if (token == null || !isValidToken(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        
        // 사용자 정보를 헤더에 추가
        String userId = extractUserId(token);
        ServerHttpRequest modifiedRequest = request.mutate()
            .header("X-User-Id", userId)
            .build();
        
        return chain.filter(exchange.mutate().request(modifiedRequest).build());
    }
    
    @Override
    public int getOrder() {
        return -100;  // 가장 먼저 실행
    }
    
    private boolean isValidToken(String token) {
        // JWT 검증 로직
        return true;
    }
    
    private String extractUserId(String token) {
        // JWT에서 사용자 ID 추출
        return "user-123";
    }
}
```

---

## 6. 핵심 패턴들

### 📊 Microservices 핵심 패턴

| 패턴 | 목적 | 구현 |
|------|------|------|
| **API Gateway** | 단일 진입점 | Spring Cloud Gateway |
| **Service Discovery** | 서비스 찾기 | Eureka, Consul |
| **Circuit Breaker** | 장애 격리 | Resilience4j |
| **Saga** | 분산 트랜잭션 | Orchestration/Choreography |
| **CQRS** | 읽기/쓰기 분리 | Event Sourcing |
| **Event Sourcing** | 이벤트 저장 | Event Store |

### 🔄 Database per Service

```java
/**
 * 각 서비스가 독립적인 DB 소유
 */

// User Service → PostgreSQL
@Entity
public class User {
    @Id
    private Long id;
    // ...
}

// Product Service → MongoDB
@Document
public class Product {
    @Id
    private String id;
    // ...
}

// Order Service → MySQL
@Entity
public class Order {
    @Id
    private Long id;
    
    // ❌ User 객체 참조 X
    // ✅ userId만 저장
    private Long userId;
    
    // ❌ Product 객체 참조 X
    // ✅ productId만 저장
    private String productId;
}
```

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 | 실무 효과 |
|------|------|-----------|
| **독립 배포** | 서비스별 배포 | 빠른 릴리즈 |
| **기술 다양성** | 서비스별 기술 선택 | 최적 기술 |
| **확장성** | 필요한 서비스만 확장 | 비용 절감 |
| **장애 격리** | 일부 장애 격리 | 안정성 |
| **팀 자율성** | 팀별 독립 개발 | 생산성 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **복잡도** | 분산 시스템 복잡 | 자동화 |
| **데이터 일관성** | 분산 트랜잭션 | Saga |
| **네트워크 지연** | 서비스 간 통신 | 캐싱 |
| **운영 부담** | 모니터링 어려움 | 중앙화 |
| **테스트 복잡** | 통합 테스트 | Contract Testing |

---

## 8. 안티패턴

### ❌ 안티패턴 1: 너무 세분화 (Nanoservices)

```java
// 잘못된 예: 너무 작은 서비스
UserNameService
UserEmailService
UserPhoneService
UserAddressService

// 올바른 예: 적절한 크기
UserService  // 모든 사용자 관련 기능
```

### ❌ 안티패턴 2: 공유 데이터베이스

```
// 잘못된 예: 여러 서비스가 같은 DB
UserService     ─┐
                 ├─→ [Shared Database]
OrderService    ─┘

// 올바른 예: 서비스별 DB
UserService  → [User DB]
OrderService → [Order DB]
```

---

## 9. 심화 주제

### 🎯 Saga Pattern (분산 트랜잭션)

```java
/**
 * Orchestration Saga
 */
@Service
public class OrderSagaOrchestrator {
    
    public void createOrder(CreateOrderRequest request) {
        // 1. Order 생성
        Order order = orderService.createOrder(request);
        
        try {
            // 2. 재고 차감
            inventoryService.decreaseStock(
                order.getProductId(),
                order.getQuantity()
            );
            
            // 3. 결제 처리
            paymentService.processPayment(order);
            
            // 4. 배송 시작
            shippingService.startShipping(order);
            
            // 성공!
            order.setStatus(OrderStatus.CONFIRMED);
            
        } catch (Exception e) {
            // 보상 트랜잭션 (역순)
            compensate(order);
        }
    }
    
    private void compensate(Order order) {
        shippingService.cancelShipping(order);
        paymentService.refundPayment(order);
        inventoryService.restoreStock(order);
        orderService.cancelOrder(order);
    }
}
```

---

## 10. 핵심 정리

### 📌 Microservices 체크리스트

```
✅ 서비스별 독립 DB
✅ API Gateway 사용
✅ Service Discovery
✅ Circuit Breaker 적용
✅ 비동기 통신 (이벤트)
✅ 분산 트레이싱
✅ 중앙 로깅
✅ 자동화된 배포
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 대규모 시스템 | ⭐⭐⭐ | 확장성 |
| 빠른 릴리즈 | ⭐⭐⭐ | 독립 배포 |
| 다양한 기술 | ⭐⭐⭐ | 기술 선택 |
| 작은 프로젝트 | ⭐ | 오버엔지니어링 |

### 💡 핵심 원칙

1. **서비스 분리**
2. **독립 배포**
3. **느슨한 결합**
4. **장애 격리**

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Event-Driven](./06-EventDriven.md) | [다음: MVP →](./08-MVP.md)**

</div>
