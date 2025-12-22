# Service Layer Pattern (서비스 계층 패턴)

> **"비즈니스 로직을 독립적인 계층으로 분리하여 재사용과 테스트를 용이하게 하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [트랜잭션 관리](#6-트랜잭션-관리)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: Controller에 비즈니스 로직
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private UserRepository userRepository;
    
    @PostMapping
    public Order createOrder(@RequestBody CreateOrderRequest request) {
        // 😱 Controller에 비즈니스 로직!
        
        // 1. 사용자 확인
        User user = userRepository.findById(request.getUserId())
            .orElseThrow(() -> new RuntimeException("사용자 없음"));
        
        if (!user.isActive()) {
            throw new RuntimeException("비활성 사용자");
        }
        
        // 2. 상품 확인
        Product product = productRepository.findById(request.getProductId())
            .orElseThrow(() -> new RuntimeException("상품 없음"));
        
        if (product.getStock() < request.getQuantity()) {
            throw new RuntimeException("재고 부족");
        }
        
        // 3. 재고 차감
        product.setStock(product.getStock() - request.getQuantity());
        productRepository.save(product);
        
        // 4. 주문 생성
        Order order = new Order();
        order.setUserId(user.getId());
        order.setProductId(product.getId());
        order.setQuantity(request.getQuantity());
        order.setTotalAmount(product.getPrice() * request.getQuantity());
        order.setStatus(OrderStatus.PENDING);
        
        Order savedOrder = orderRepository.save(order);
        
        // 5. 이메일 발송
        sendOrderConfirmationEmail(user, order);
        
        return savedOrder;
        
        // 문제점:
        // 1. Controller가 너무 많은 책임
        // 2. 비즈니스 로직 재사용 불가
        // 3. 테스트 어려움 (HTTP 필요)
        // 4. 트랜잭션 관리 복잡
    }
}

// 문제 2: 중복된 비즈니스 로직
@RestController
public class OrderController {
    @PostMapping("/api/orders")
    public Order createOrderFromWeb(@RequestBody CreateOrderRequest request) {
        // 재고 확인
        if (product.getStock() < request.getQuantity()) {
            throw new RuntimeException("재고 부족");
        }
        
        // 주문 생성
        // ...
    }
}

@RestController
public class MobileOrderController {
    @PostMapping("/api/mobile/orders")
    public Order createOrderFromMobile(@RequestBody CreateOrderRequest request) {
        // 😱 똑같은 로직 반복!
        if (product.getStock() < request.getQuantity()) {
            throw new RuntimeException("재고 부족");
        }
        
        // 주문 생성
        // ...
    }
}

public class BatchOrderProcessor {
    public void processBatchOrders(List<CreateOrderRequest> requests) {
        // 😱 또 똑같은 로직 반복!
        for (CreateOrderRequest request : requests) {
            if (product.getStock() < request.getQuantity()) {
                throw new RuntimeException("재고 부족");
            }
            // ...
        }
    }
}

// 문제 3: 트랜잭션 관리 어려움
public class OrderController {
    @PostMapping("/api/orders")
    public Order createOrder(@RequestBody CreateOrderRequest request) {
        // 😱 트랜잭션 없음!
        
        // 1. 주문 생성
        Order order = orderRepository.save(new Order());
        
        // 2. 재고 차감 (별도 트랜잭션)
        product.setStock(product.getStock() - quantity);
        productRepository.save(product);
        
        // 3. 포인트 차감 (별도 트랜잭션)
        user.setPoints(user.getPoints() - usedPoints);
        userRepository.save(user);
        
        // 문제점:
        // - 주문은 생성되었는데 재고 차감 실패?
        // - 재고는 차감되었는데 포인트 차감 실패?
        // - 데이터 불일치!
    }
}

// 문제 4: 여러 Repository 조합
public class StatisticsController {
    @GetMapping("/api/stats/dashboard")
    public DashboardStats getDashboard() {
        // 😱 여러 Repository를 직접 조합!
        
        long totalUsers = userRepository.count();
        long totalOrders = orderRepository.count();
        BigDecimal totalRevenue = orderRepository.sumTotalAmount();
        List<Order> recentOrders = orderRepository.findTop10ByOrderByCreatedAtDesc();
        List<Product> popularProducts = productRepository.findTopSellingProducts();
        
        // 이 로직을 다른 곳에서도 사용한다면?
        // → 중복 코드 발생!
    }
}

// 문제 5: 테스트 불가능
public class OrderController {
    @PostMapping("/api/orders")
    public Order createOrder(@RequestBody CreateOrderRequest request) {
        // 비즈니스 로직이 Controller에 있어서
        // 테스트하려면:
        // - MockMvc 필요
        // - HTTP 요청 만들어야 함
        // - 느린 통합 테스트만 가능
        // - 단위 테스트 불가능!
    }
}

// 문제 6: 복잡한 비즈니스 규칙
public class OrderController {
    @PostMapping("/api/orders")
    public Order createOrder(@RequestBody CreateOrderRequest request) {
        // 😱 복잡한 비즈니스 규칙이 Controller에!
        
        // 할인 계산
        BigDecimal discountRate = BigDecimal.ZERO;
        if (user.getGrade() == UserGrade.VIP) {
            discountRate = new BigDecimal("0.1");
        } else if (order.getTotalAmount().compareTo(new BigDecimal("100000")) >= 0) {
            discountRate = new BigDecimal("0.05");
        }
        
        // 배송비 계산
        BigDecimal shippingFee = BigDecimal.ZERO;
        if (order.getTotalAmount().compareTo(new BigDecimal("30000")) < 0) {
            shippingFee = new BigDecimal("3000");
        }
        
        // 포인트 적립
        int earnedPoints = order.getTotalAmount().intValue() / 100;
        if (user.getGrade() == UserGrade.VIP) {
            earnedPoints *= 2;
        }
        
        // 이런 복잡한 규칙이 Controller에!
        // 재사용? 테스트? 불가능!
    }
}

// 문제 7: 여러 시스템 통합
public class OrderController {
    @PostMapping("/api/orders")
    public Order createOrder(@RequestBody CreateOrderRequest request) {
        // 😱 여러 외부 시스템 호출이 Controller에!
        
        // 1. 주문 생성
        Order order = orderRepository.save(new Order());
        
        // 2. 결제 시스템 호출
        paymentSystem.processPayment(order);
        
        // 3. 재고 시스템 호출
        inventorySystem.decreaseStock(product.getId(), quantity);
        
        // 4. 배송 시스템 호출
        shippingSystem.createShipment(order);
        
        // 5. 이메일 시스템 호출
        emailSystem.sendConfirmation(user.getEmail(), order);
        
        // 6. SMS 시스템 호출
        smsSystem.sendNotification(user.getPhone(), order);
        
        // Controller가 너무 많은 일을 함!
        // 오케스트레이션 로직이 필요!
    }
}
```

### ⚡ 핵심 문제

1. **관심사 혼재**: Controller에 비즈니스 로직
2. **중복 코드**: 같은 로직이 여러 곳에
3. **테스트 어려움**: HTTP 없이 테스트 불가
4. **트랜잭션**: 트랜잭션 관리 복잡
5. **재사용 불가**: 비즈니스 로직 재사용 어려움
6. **복잡도**: Controller가 너무 비대

---

## 2. 패턴 정의

### 📖 정의

> **비즈니스 로직을 독립적인 서비스 계층으로 분리하여, 여러 클라이언트에서 재사용 가능하고 트랜잭션을 관리하는 패턴**

### 🎯 목적

- **비즈니스 로직 캡슐화**: 독립적인 계층으로 분리
- **재사용성**: 여러 곳에서 사용 가능
- **트랜잭션 관리**: 일관된 트랜잭션 처리
- **테스트 용이**: 단위 테스트 가능

### 💡 핵심 아이디어

```java
// Before: Controller에 비즈니스 로직
@RestController
public class OrderController {
    @PostMapping("/api/orders")
    public Order createOrder(@RequestBody CreateOrderRequest request) {
        // 😱 비즈니스 로직이 Controller에!
        User user = userRepository.findById(request.getUserId()).orElseThrow();
        Product product = productRepository.findById(request.getProductId()).orElseThrow();
        
        if (product.getStock() < request.getQuantity()) {
            throw new RuntimeException("재고 부족");
        }
        
        // ... 복잡한 로직
    }
}

// After: Service Layer로 분리
// 1. Service Interface
public interface OrderService {
    Order createOrder(CreateOrderCommand command);
}

// 2. Service Implementation
@Service
@Transactional
public class OrderServiceImpl implements OrderService {
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final UserRepository userRepository;
    
    @Override
    public Order createOrder(CreateOrderCommand command) {
        // 비즈니스 로직은 Service에!
        User user = userRepository.findById(command.getUserId()).orElseThrow();
        Product product = productRepository.findById(command.getProductId()).orElseThrow();
        
        // 검증, 계산, 저장 등
        validateOrder(user, product, command.getQuantity());
        return processOrder(user, product, command);
    }
}

// 3. Controller는 Service만 호출
@RestController
public class OrderController {
    private final OrderService orderService;
    
    @PostMapping("/api/orders")
    public Order createOrder(@RequestBody CreateOrderRequest request) {
        // 😊 Controller는 간결!
        CreateOrderCommand command = toCommand(request);
        return orderService.createOrder(command);
    }
}
```

---

## 3. 구조와 구성요소

### 📊 Service Layer 구조

```
┌─────────────────────────────────────┐
│      Presentation Layer             │
│   (Controller, View)                │
│                                     │
│   - REST Controller                 │
│   - GraphQL Resolver                │
│   - Message Consumer                │
└─────────────────────────────────────┘
              │
              │ uses
              ▼
┌─────────────────────────────────────┐
│       Service Layer                 │
│   (Business Logic)                  │
│                                     │
│   @Service                          │
│   @Transactional                    │
│                                     │
│   - 비즈니스 로직                      │
│   - 트랜잭션 관리                      │
│   - 여러 Repository 조합              │
└─────────────────────────────────────┘
              │
              │ uses
              ▼
┌─────────────────────────────────────┐
│      Data Access Layer              │
│   (Repository, DAO)                 │
│                                     │
│   - CRUD 작업                        │
│   - 쿼리 실행                         │
└─────────────────────────────────────┘
```

### 🔄 계층 간 호출 흐름

```
HTTP Request
    │
    ▼
┌──────────────┐
│ Controller   │ → DTO 변환
└──────────────┘
    │
    ▼
┌──────────────┐
│   Service    │ → 비즈니스 로직
│              │ → 트랜잭션 시작
└──────────────┘
    │
    ├─→ Repository A
    ├─→ Repository B
    └─→ Repository C
    │
    ▼
┌──────────────┐
│   Service    │ → 트랜잭션 커밋/롤백
└──────────────┘
    │
    ▼
┌──────────────┐
│ Controller   │ → DTO 반환
└──────────────┘
    │
    ▼
HTTP Response
```

### 🔧 Service Layer 책임

| 책임 | 설명 | 예시 |
|------|------|------|
| **비즈니스 로직** | 도메인 규칙 구현 | 할인 계산, 재고 확인 |
| **트랜잭션 관리** | 원자성 보장 | @Transactional |
| **Repository 조합** | 여러 Repository 사용 | User + Order + Product |
| **데이터 변환** | Entity ↔ DTO | 매핑 |
| **예외 변환** | 기술 예외 → 비즈니스 예외 | SQLException → OrderException |

---

## 4. 구현 방법

### 완전한 구현: E-Commerce Order Service ⭐⭐⭐

```java
/**
 * ============================================
 * DOMAIN MODELS
 * ============================================
 */
// User, Order, Product (이전 예제에서 정의됨)

/**
 * ============================================
 * SERVICE INTERFACE
 * ============================================
 */
public interface OrderService {
    /**
     * 주문 생성
     */
    Order createOrder(CreateOrderCommand command);
    
    /**
     * 주문 취소
     */
    void cancelOrder(Long orderId);
    
    /**
     * 주문 조회
     */
    Order getOrder(Long orderId);
    
    /**
     * 사용자별 주문 목록
     */
    List<Order> getUserOrders(Long userId);
    
    /**
     * 주문 통계
     */
    OrderStatistics getOrderStatistics(Long userId);
}

/**
 * Command Objects (입력 데이터)
 */
public class CreateOrderCommand {
    private Long userId;
    private Long productId;
    private int quantity;
    private int usedPoints;  // 사용 포인트
    
    public CreateOrderCommand(Long userId, Long productId, int quantity, int usedPoints) {
        this.userId = userId;
        this.productId = productId;
        this.quantity = quantity;
        this.usedPoints = usedPoints;
    }
    
    // Getters
    public Long getUserId() { return userId; }
    public Long getProductId() { return productId; }
    public int getQuantity() { return quantity; }
    public int getUsedPoints() { return usedPoints; }
}

/**
 * ============================================
 * SERVICE IMPLEMENTATION
 * ============================================
 */
@Service
@Transactional  // 클래스 레벨 트랜잭션
public class OrderServiceImpl implements OrderService {
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final UserRepository userRepository;
    private final NotificationService notificationService;
    
    @Autowired
    public OrderServiceImpl(
            OrderRepository orderRepository,
            ProductRepository productRepository,
            UserRepository userRepository,
            NotificationService notificationService) {
        this.orderRepository = orderRepository;
        this.productRepository = productRepository;
        this.userRepository = userRepository;
        this.notificationService = notificationService;
    }
    
    /**
     * 주문 생성 (핵심 비즈니스 로직)
     */
    @Override
    public Order createOrder(CreateOrderCommand command) {
        System.out.println("\n🛒 OrderService: 주문 생성 시작");
        System.out.println("   사용자 ID: " + command.getUserId());
        System.out.println("   상품 ID: " + command.getProductId());
        System.out.println("   수량: " + command.getQuantity());
        
        // 1. 사용자 조회 및 검증
        User user = findAndValidateUser(command.getUserId());
        
        // 2. 상품 조회 및 재고 확인
        Product product = findAndValidateProduct(command.getProductId(), command.getQuantity());
        
        // 3. 포인트 검증
        validatePoints(user, command.getUsedPoints());
        
        // 4. 금액 계산
        OrderAmount amount = calculateOrderAmount(product, command.getQuantity(), command.getUsedPoints(), user);
        
        System.out.println("   총액: " + amount.getTotalAmount());
        System.out.println("   할인: " + amount.getDiscount());
        System.out.println("   최종 금액: " + amount.getFinalAmount());
        
        // 5. 주문 생성
        Order order = createOrderEntity(user, product, command, amount);
        Order savedOrder = orderRepository.save(order);
        
        // 6. 재고 차감
        decreaseStock(product, command.getQuantity());
        
        // 7. 포인트 차감/적립
        processPoints(user, command.getUsedPoints(), amount.getEarnedPoints());
        
        // 8. 알림 발송 (비동기)
        sendOrderNotification(user, savedOrder);
        
        System.out.println("   ✅ 주문 생성 완료: ID=" + savedOrder.getId());
        
        return savedOrder;
    }
    
    /**
     * 주문 취소
     */
    @Override
    public void cancelOrder(Long orderId) {
        System.out.println("\n❌ OrderService: 주문 취소 - ID=" + orderId);
        
        // 1. 주문 조회
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException("주문을 찾을 수 없습니다"));
        
        // 2. 취소 가능 여부 확인
        validateCancellable(order);
        
        // 3. 상품 조회
        Product product = productRepository.findById(order.getProductId())
            .orElseThrow(() -> new ProductNotFoundException("상품을 찾을 수 없습니다"));
        
        // 4. 재고 복구
        product.setStock(product.getStock() + order.getQuantity());
        productRepository.save(product);
        
        // 5. 포인트 복구
        if (order.getUsedPoints() > 0) {
            User user = userRepository.findById(order.getUserId()).orElseThrow();
            user.setPoints(user.getPoints() + order.getUsedPoints());
            userRepository.save(user);
        }
        
        // 6. 주문 상태 변경
        order.setStatus(OrderStatus.CANCELLED);
        orderRepository.save(order);
        
        System.out.println("   ✅ 주문 취소 완료");
    }
    
    /**
     * 주문 조회
     */
    @Override
    @Transactional(readOnly = true)  // 읽기 전용
    public Order getOrder(Long orderId) {
        return orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException("주문을 찾을 수 없습니다"));
    }
    
    /**
     * 사용자별 주문 목록
     */
    @Override
    @Transactional(readOnly = true)
    public List<Order> getUserOrders(Long userId) {
        // 사용자 존재 확인
        userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException("사용자를 찾을 수 없습니다"));
        
        return orderRepository.findByUserIdOrderByCreatedAtDesc(userId);
    }
    
    /**
     * 주문 통계
     */
    @Override
    @Transactional(readOnly = true)
    public OrderStatistics getOrderStatistics(Long userId) {
        System.out.println("\n📊 OrderService: 주문 통계 조회");
        
        List<Order> orders = orderRepository.findByUserId(userId);
        
        long totalCount = orders.size();
        BigDecimal totalAmount = orders.stream()
            .map(Order::getTotalAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        
        long completedCount = orders.stream()
            .filter(o -> o.getStatus() == OrderStatus.COMPLETED)
            .count();
        
        OrderStatistics stats = new OrderStatistics(totalCount, totalAmount, completedCount);
        
        System.out.println("   총 주문: " + totalCount);
        System.out.println("   총 금액: " + totalAmount);
        System.out.println("   완료: " + completedCount);
        
        return stats;
    }
    
    /**
     * ============================================
     * PRIVATE HELPER METHODS (비즈니스 로직)
     * ============================================
     */
    
    /**
     * 사용자 조회 및 검증
     */
    private User findAndValidateUser(Long userId) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException("사용자를 찾을 수 없습니다"));
        
        if (!user.isActive()) {
            throw new UserNotActiveException("비활성화된 사용자입니다");
        }
        
        System.out.println("   ✅ 사용자 확인: " + user.getName());
        
        return user;
    }
    
    /**
     * 상품 조회 및 재고 확인
     */
    private Product findAndValidateProduct(Long productId, int quantity) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException("상품을 찾을 수 없습니다"));
        
        if (product.getStock() < quantity) {
            throw new InsufficientStockException(
                "재고가 부족합니다. 재고: " + product.getStock() + ", 요청: " + quantity
            );
        }
        
        System.out.println("   ✅ 상품 확인: " + product.getName() + " (재고: " + product.getStock() + ")");
        
        return product;
    }
    
    /**
     * 포인트 검증
     */
    private void validatePoints(User user, int usedPoints) {
        if (usedPoints < 0) {
            throw new InvalidPointsException("포인트는 0 이상이어야 합니다");
        }
        
        if (user.getPoints() < usedPoints) {
            throw new InsufficientPointsException(
                "포인트가 부족합니다. 보유: " + user.getPoints() + ", 사용: " + usedPoints
            );
        }
    }
    
    /**
     * 주문 금액 계산 (복잡한 비즈니스 로직!)
     */
    private OrderAmount calculateOrderAmount(Product product, int quantity, int usedPoints, User user) {
        // 기본 금액
        BigDecimal baseAmount = product.getPrice().multiply(BigDecimal.valueOf(quantity));
        
        // 할인 계산
        BigDecimal discount = calculateDiscount(baseAmount, user);
        
        // 포인트 차감
        BigDecimal pointsDiscount = BigDecimal.valueOf(usedPoints);
        
        // 최종 금액
        BigDecimal finalAmount = baseAmount.subtract(discount).subtract(pointsDiscount);
        
        // 적립 포인트 (최종 금액의 1%)
        int earnedPoints = finalAmount.multiply(new BigDecimal("0.01")).intValue();
        if (user.getGrade() == UserGrade.VIP) {
            earnedPoints *= 2;  // VIP는 2배
        }
        
        return new OrderAmount(baseAmount, discount, pointsDiscount, finalAmount, earnedPoints);
    }
    
    /**
     * 할인 계산
     */
    private BigDecimal calculateDiscount(BigDecimal amount, User user) {
        BigDecimal discountRate = BigDecimal.ZERO;
        
        // VIP 회원 10% 할인
        if (user.getGrade() == UserGrade.VIP) {
            discountRate = new BigDecimal("0.10");
        }
        // 10만원 이상 5% 할인
        else if (amount.compareTo(new BigDecimal("100000")) >= 0) {
            discountRate = new BigDecimal("0.05");
        }
        
        return amount.multiply(discountRate);
    }
    
    /**
     * 주문 Entity 생성
     */
    private Order createOrderEntity(User user, Product product, CreateOrderCommand command, OrderAmount amount) {
        Order order = new Order();
        order.setUserId(user.getId());
        order.setProductId(product.getId());
        order.setQuantity(command.getQuantity());
        order.setUsedPoints(command.getUsedPoints());
        order.setTotalAmount(amount.getFinalAmount());
        order.setStatus(OrderStatus.PENDING);
        order.setCreatedAt(LocalDateTime.now());
        return order;
    }
    
    /**
     * 재고 차감
     */
    private void decreaseStock(Product product, int quantity) {
        product.setStock(product.getStock() - quantity);
        productRepository.save(product);
        
        System.out.println("   📦 재고 차감: " + product.getName() + " (남은 재고: " + product.getStock() + ")");
    }
    
    /**
     * 포인트 처리
     */
    private void processPoints(User user, int usedPoints, int earnedPoints) {
        // 포인트 차감
        if (usedPoints > 0) {
            user.setPoints(user.getPoints() - usedPoints);
        }
        
        // 포인트 적립
        user.setPoints(user.getPoints() + earnedPoints);
        
        userRepository.save(user);
        
        System.out.println("   💰 포인트 처리: 사용=" + usedPoints + ", 적립=" + earnedPoints + ", 잔여=" + user.getPoints());
    }
    
    /**
     * 알림 발송
     */
    private void sendOrderNotification(User user, Order order) {
        try {
            notificationService.sendOrderConfirmation(user.getEmail(), order);
            System.out.println("   📧 알림 발송 완료");
        } catch (Exception e) {
            // 알림 실패는 주문에 영향 안 줌
            System.err.println("   ⚠️ 알림 발송 실패: " + e.getMessage());
        }
    }
    
    /**
     * 취소 가능 여부 확인
     */
    private void validateCancellable(Order order) {
        if (order.getStatus() == OrderStatus.CANCELLED) {
            throw new OrderAlreadyCancelledException("이미 취소된 주문입니다");
        }
        
        if (order.getStatus() == OrderStatus.SHIPPED || order.getStatus() == OrderStatus.COMPLETED) {
            throw new OrderNotCancellableException("배송 중이거나 완료된 주문은 취소할 수 없습니다");
        }
    }
}

/**
 * ============================================
 * VALUE OBJECTS
 * ============================================
 */

/**
 * 주문 금액 정보
 */
class OrderAmount {
    private final BigDecimal baseAmount;
    private final BigDecimal discount;
    private final BigDecimal pointsDiscount;
    private final BigDecimal finalAmount;
    private final int earnedPoints;
    
    public OrderAmount(BigDecimal baseAmount, BigDecimal discount, BigDecimal pointsDiscount,
                      BigDecimal finalAmount, int earnedPoints) {
        this.baseAmount = baseAmount;
        this.discount = discount;
        this.pointsDiscount = pointsDiscount;
        this.finalAmount = finalAmount;
        this.earnedPoints = earnedPoints;
    }
    
    public BigDecimal getTotalAmount() { return baseAmount; }
    public BigDecimal getDiscount() { return discount; }
    public BigDecimal getFinalAmount() { return finalAmount; }
    public int getEarnedPoints() { return earnedPoints; }
}

/**
 * 주문 통계
 */
public class OrderStatistics {
    private final long totalCount;
    private final BigDecimal totalAmount;
    private final long completedCount;
    
    public OrderStatistics(long totalCount, BigDecimal totalAmount, long completedCount) {
        this.totalCount = totalCount;
        this.totalAmount = totalAmount;
        this.completedCount = completedCount;
    }
    
    public long getTotalCount() { return totalCount; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public long getCompletedCount() { return completedCount; }
}

/**
 * ============================================
 * CONTROLLER (간결!)
 * ============================================
 */
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final OrderService orderService;
    
    @Autowired
    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }
    
    /**
     * 주문 생성
     */
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(@RequestBody CreateOrderRequest request) {
        // DTO → Command 변환
        CreateOrderCommand command = new CreateOrderCommand(
            request.getUserId(),
            request.getProductId(),
            request.getQuantity(),
            request.getUsedPoints()
        );
        
        // Service 호출 (비즈니스 로직은 Service에!)
        Order order = orderService.createOrder(command);
        
        // Entity → DTO 변환
        return ResponseEntity.ok(OrderResponse.from(order));
    }
    
    /**
     * 주문 취소
     */
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> cancelOrder(@PathVariable Long id) {
        orderService.cancelOrder(id);
        return ResponseEntity.noContent().build();
    }
    
    /**
     * 주문 조회
     */
    @GetMapping("/{id}")
    public ResponseEntity<OrderResponse> getOrder(@PathVariable Long id) {
        Order order = orderService.getOrder(id);
        return ResponseEntity.ok(OrderResponse.from(order));
    }
    
    /**
     * 사용자별 주문 목록
     */
    @GetMapping("/user/{userId}")
    public ResponseEntity<List<OrderResponse>> getUserOrders(@PathVariable Long userId) {
        List<Order> orders = orderService.getUserOrders(userId);
        
        List<OrderResponse> response = orders.stream()
            .map(OrderResponse::from)
            .collect(Collectors.toList());
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * 주문 통계
     */
    @GetMapping("/user/{userId}/statistics")
    public ResponseEntity<OrderStatistics> getStatistics(@PathVariable Long userId) {
        OrderStatistics stats = orderService.getOrderStatistics(userId);
        return ResponseEntity.ok(stats);
    }
}

/**
 * ============================================
 * EXCEPTIONS
 * ============================================
 */
class OrderNotFoundException extends RuntimeException {
    public OrderNotFoundException(String message) { super(message); }
}

class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(String message) { super(message); }
}

class ProductNotFoundException extends RuntimeException {
    public ProductNotFoundException(String message) { super(message); }
}

class UserNotActiveException extends RuntimeException {
    public UserNotActiveException(String message) { super(message); }
}

class InsufficientStockException extends RuntimeException {
    public InsufficientStockException(String message) { super(message); }
}

class InvalidPointsException extends RuntimeException {
    public InvalidPointsException(String message) { super(message); }
}

class InsufficientPointsException extends RuntimeException {
    public InsufficientPointsException(String message) { super(message); }
}

class OrderAlreadyCancelledException extends RuntimeException {
    public OrderAlreadyCancelledException(String message) { super(message); }
}

class OrderNotCancellableException extends RuntimeException {
    public OrderNotCancellableException(String message) { super(message); }
}
```

**실행 시나리오:**
```
=== Service Layer Pattern 예제 ===

🛒 OrderService: 주문 생성 시작
   사용자 ID: 1
   상품 ID: 1
   수량: 2
   ✅ 사용자 확인: 홍길동
   ✅ 상품 확인: 노트북 (재고: 10)
   총액: 2400000
   할인: 240000 (VIP 10% 할인)
   최종 금액: 2160000
   📦 재고 차감: 노트북 (남은 재고: 8)
   💰 포인트 처리: 사용=10000, 적립=43200, 잔여=43200
   📧 알림 발송 완료
   ✅ 주문 생성 완료: ID=1

==================================================

❌ OrderService: 주문 취소 - ID=1
   ✅ 주문 취소 완료
```

---

## 5. 실전 예제

### 예제 1: 복잡한 비즈니스 로직 처리 ⭐⭐⭐

```java
/**
 * 재고 관리 서비스
 */
@Service
@Transactional
public class InventoryService {
    private final ProductRepository productRepository;
    private final InventoryHistoryRepository historyRepository;
    
    /**
     * 재고 조정 (여러 비즈니스 규칙)
     */
    public void adjustStock(Long productId, int quantity, String reason) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException("상품 없음"));
        
        // 비즈니스 규칙 1: 재고는 0 이상
        int newStock = product.getStock() + quantity;
        if (newStock < 0) {
            throw new InvalidStockException("재고는 0 이상이어야 합니다");
        }
        
        // 비즈니스 규칙 2: 재고 부족 알림
        if (newStock < product.getMinStock()) {
            sendLowStockAlert(product, newStock);
        }
        
        // 재고 변경
        product.setStock(newStock);
        productRepository.save(product);
        
        // 이력 기록
        InventoryHistory history = new InventoryHistory(
            product.getId(),
            quantity,
            newStock,
            reason
        );
        historyRepository.save(history);
    }
}
```

---

## 6. 트랜잭션 관리

### 🔄 Spring @Transactional

```java
/**
 * 트랜잭션 관리
 */
@Service
public class OrderService {
    
    // 1. 기본 트랜잭션
    @Transactional
    public Order createOrder(CreateOrderCommand command) {
        // 이 메서드 전체가 하나의 트랜잭션
        // 성공 시 커밋, 예외 시 롤백
    }
    
    // 2. 읽기 전용 (성능 최적화)
    @Transactional(readOnly = true)
    public List<Order> getAllOrders() {
        // 읽기 전용 트랜잭션
        // DB 최적화 가능
    }
    
    // 3. 예외별 롤백 규칙
    @Transactional(
        rollbackFor = Exception.class,  // 모든 예외에 롤백
        noRollbackFor = {NotificationException.class}  // 알림 실패는 롤백 안 함
    )
    public Order createOrderWithNotification(CreateOrderCommand command) {
        Order order = createOrder(command);
        sendNotification(order);  // 실패해도 주문은 유지
        return order;
    }
}
```

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 |
|------|------|
| **재사용성** | 여러 곳에서 사용 |
| **테스트 용이** | 단위 테스트 가능 |
| **트랜잭션** | 일관된 관리 |
| **관심사 분리** | Controller 간결 |

### ❌ 단점

| 단점 | 설명 |
|------|------|
| **복잡도** | 계층 증가 |
| **비대화** | God Service 위험 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: God Service

```java
// 잘못된 예: 모든 로직을 하나의 Service에
@Service
public class ApplicationService {
    // 모든 비즈니스 로직!
    public void createUser() {}
    public void createOrder() {}
    public void processPayment() {}
    public void sendEmail() {}
    // ...
}
```

**해결:**
```java
// 올바른 예: 도메인별 Service 분리
@Service
public class UserService { }

@Service
public class OrderService { }

@Service
public class PaymentService { }
```

---

## 9. 심화 주제

### 🎯 Service 간 통신

```java
/**
 * Service 간 협력
 */
@Service
public class OrderService {
    private final PaymentService paymentService;
    private final NotificationService notificationService;
    
    @Transactional
    public Order createOrder(CreateOrderCommand command) {
        Order order = saveOrder(command);
        
        // 다른 Service 호출
        paymentService.processPayment(order);
        notificationService.sendConfirmation(order);
        
        return order;
    }
}
```

---

## 10. 핵심 정리

### 📌 Service Layer 체크리스트

```
✅ 비즈니스 로직은 Service에
✅ @Service, @Transactional 사용
✅ Controller는 Service만 호출
✅ 도메인별 Service 분리
✅ 읽기 전용은 readOnly=true
✅ 예외는 명확하게
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: DAO](./02-DAO.md) | [다음: Unit of Work →](./04-UnitOfWork.md)**

</div>
