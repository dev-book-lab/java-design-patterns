# Event-Driven Architecture Pattern (이벤트 주도 아키텍처)

> **"이벤트를 통해 느슨하게 결합된 시스템을 만들자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [이벤트 패턴 변형](#6-이벤트-패턴-변형)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 강한 결합 (Tight Coupling)
public class OrderService {
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final EmailService emailService;
    private final SmsService smsService;
    private final LoggingService loggingService;
    private final AnalyticsService analyticsService;
    
    public void createOrder(Order order) {
        // 주문 저장
        orderRepository.save(order);
        
        // 😱 모든 서비스를 직접 호출!
        inventoryService.decreaseStock(order.getItems());
        paymentService.processPayment(order);
        emailService.sendOrderConfirmation(order);
        smsService.sendSmsNotification(order);
        loggingService.logOrderCreated(order);
        analyticsService.trackOrderCreated(order);
        
        // 새 기능 추가 시마다 이 코드 수정!
        // 강한 결합으로 테스트 어려움!
        // 한 서비스 실패 시 전체 실패!
    }
}

// 문제 2: 동기 처리로 인한 성능 저하
public class UserRegistrationService {
    public void registerUser(User user) {
        // 사용자 저장
        userRepository.save(user);
        
        // 😱 모든 작업을 순차적으로 대기!
        emailService.sendWelcomeEmail(user);        // 2초 대기
        smsService.sendVerificationCode(user);      // 1초 대기
        analyticsService.trackSignup(user);         // 0.5초 대기
        crmService.createCrmEntry(user);            // 1초 대기
        
        // 총 4.5초 대기!
        // 사용자는 4.5초 동안 응답 대기!
    }
}

// 문제 3: 트랜잭션 문제
public class PaymentService {
    @Transactional
    public void processPayment(Payment payment) {
        // 결제 처리
        paymentRepository.save(payment);
        
        // 😱 외부 API 호출이 트랜잭션 안에!
        stripeApi.charge(payment);  // 실패하면 롤백?
        
        // 외부 서비스 호출
        emailService.sendReceipt(payment);  // 실패하면?
        
        // 트랜잭션이 너무 길어짐
        // 외부 서비스 장애 시 DB 트랜잭션도 영향
    }
}

// 문제 4: 변경 전파의 어려움
public class ProductService {
    public void updatePrice(Long productId, BigDecimal newPrice) {
        Product product = productRepository.findById(productId);
        product.setPrice(newPrice);
        productRepository.save(product);
        
        // 😱 가격 변경을 관련 시스템에 알려야 함
        // 어떻게?
        // 1. 직접 호출? → 강한 결합
        // 2. 폴링? → 비효율적
        // 3. 웹훅? → 복잡함
        
        // 새로운 구독자 추가 시 코드 수정 필요
    }
}

// 문제 5: 확장성 문제
public class NotificationService {
    public void sendNotifications(Event event) {
        // 😱 동시에 수천 명에게 알림?
        for (User user : getInterestedUsers(event)) {
            emailService.send(user.getEmail(), event);
            // 동기적으로 처리하면 너무 느림!
            // 서버 리소스 고갈!
        }
    }
}

// 문제 6: 복잡한 비즈니스 흐름
public class OrderProcessingService {
    public void processOrder(Order order) {
        // 주문 처리 단계가 많고 복잡
        
        // 1. 재고 확인
        if (!inventoryService.checkStock(order)) {
            return;
        }
        
        // 2. 결제 처리
        Payment payment = paymentService.process(order);
        if (!payment.isSuccess()) {
            return;
        }
        
        // 3. 배송 준비
        shippingService.prepare(order);
        
        // 4. 알림 발송
        notificationService.send(order);
        
        // 😱 문제점:
        // - 순서가 하드코딩됨
        // - 중간에 실패 시 처리 복잡
        // - 조건부 로직 추가 어려움
        // - 병렬 처리 불가
    }
}

// 문제 7: 모니터링과 추적의 어려움
public class SystemIntegration {
    public void integrateOrder(Order order) {
        // 여러 시스템에 데이터 전파
        erpSystem.sync(order);
        crmSystem.sync(order);
        analyticsSystem.track(order);
        
        // 😱 어떻게 추적?
        // - 어느 시스템까지 전파됐는지?
        // - 실패한 시스템은?
        // - 재시도 필요한가?
        // - 전체 흐름 모니터링?
    }
}
```

### ⚡ 핵심 문제

1. **강한 결합**: 컴포넌트 간 직접 의존성
2. **동기 처리**: 모든 작업이 순차적
3. **확장성 부족**: 부하 분산 어려움
4. **트랜잭션 복잡**: 분산 트랜잭션 문제
5. **변경 전파**: 상태 변경 알림 어려움
6. **모니터링**: 전체 흐름 추적 어려움

---

## 2. 패턴 정의

### 📖 정의

> **시스템의 상태 변화를 이벤트로 표현하고, 이벤트를 통해 컴포넌트 간 통신하여 느슨하게 결합된 확장 가능한 시스템을 만드는 아키텍처 패턴**

### 🎯 목적

- **느슨한 결합**: 이벤트로 통신하여 의존성 제거
- **비동기 처리**: 이벤트를 비동기로 처리
- **확장성**: 이벤트 구독자를 자유롭게 추가
- **탄력성**: 일부 실패가 전체에 영향 안 줌

### 💡 핵심 아이디어

```java
// Before: 직접 호출 (강한 결합)
public class OrderService {
    private EmailService emailService;
    private SmsService smsService;
    
    public void createOrder(Order order) {
        orderRepository.save(order);
        
        // 직접 호출
        emailService.send(order);
        smsService.send(order);
        
        // 새 서비스 추가 시 여기 수정!
    }
}

// After: 이벤트 발행 (느슨한 결합)
public class OrderService {
    private EventPublisher eventPublisher;
    
    public void createOrder(Order order) {
        orderRepository.save(order);
        
        // 이벤트 발행만!
        eventPublisher.publish(new OrderCreatedEvent(order));
        
        // 누가 구독하는지 몰라도 됨!
        // 새 구독자 추가해도 이 코드는 변경 없음!
    }
}

// 구독자들 (독립적으로 추가/제거 가능)
@EventListener
public class EmailNotificationListener {
    public void on(OrderCreatedEvent event) {
        emailService.send(event.getOrder());
    }
}

@EventListener
public class SmsNotificationListener {
    public void on(OrderCreatedEvent event) {
        smsService.send(event.getOrder());
    }
}

// 나중에 추가 (기존 코드 수정 없이!)
@EventListener
public class AnalyticsListener {
    public void on(OrderCreatedEvent event) {
        analyticsService.track(event.getOrder());
    }
}
```

---

## 3. 구조와 구성요소

### 📊 Event-Driven 구조

```
┌────────────────────────────────────────┐
│         Event Producer                 │
│  (Publisher / Event Source)            │
│                                        │
│  - 상태 변경 감지                         │
│  - 이벤트 생성                            │
│  - 이벤트 발행                            │
└────────────────────────────────────────┘
                │
                │ publishes
                ▼
┌────────────────────────────────────────┐
│          Event Channel                 │
│     (Event Bus / Message Queue)        │
│                                        │
│  - 이벤트 라우팅                          │
│  - 이벤트 저장 (선택)                      │
│  - 순서 보장 (선택)                       │
└────────────────────────────────────────┘
                │
                │ delivers
                ▼
┌────────────────────────────────────────┐
│        Event Consumers                 │
│    (Subscribers / Event Handlers)      │
│                                        │
│  - 이벤트 수신                            │
│  - 비즈니스 로직 실행                       │
│  - 독립적으로 실행                         │
└────────────────────────────────────────┘
```

### 🔄 이벤트 흐름

```
User Action
    │
    ▼
┌─────────────┐
│  Service    │ → orderRepository.save()
└─────────────┘
    │
    │ publish(OrderCreatedEvent)
    ▼
┌─────────────────┐
│   Event Bus     │
└─────────────────┘
    │
    ├─────────────────┬─────────────────┬─────────────────┐
    │                 │                 │                 │
    ▼                 ▼                 ▼                 ▼
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│ Email   │     │   SMS   │     │Analytics│     │Inventory│
│Listener │     │Listener │     │Listener │     │Listener │
└─────────┘     └─────────┘     └─────────┘     └─────────┘
    │                 │                 │                 │
    │                 │                 │                 │
(비동기 실행)      (비동기 실행)        (비동기 실행)        (비동기 실행)
```

### 🏗️ 이벤트 패턴 종류

```
1. Point-to-Point (Queue)
   Producer → [Queue] → Single Consumer

2. Publish-Subscribe (Topic)
   Producer → [Topic] → Multiple Consumers

3. Event Sourcing
   Command → [Event Store] → Event Stream → Read Models

4. CQRS
   Write Model → Events → Read Model
```

### 🔧 구성요소

| 컴포넌트 | 역할 | 책임 | 예시 |
|---------|------|------|------|
| **Event** | 상태 변화 표현 | - 불변 객체<br>- 과거형 이름<br>- 충분한 정보 | `OrderCreatedEvent` |
| **Event Producer** | 이벤트 발행 | - 상태 변경<br>- 이벤트 생성<br>- 이벤트 발행 | `OrderService` |
| **Event Bus** | 이벤트 전달 | - 라우팅<br>- 필터링<br>- 저장 (선택) | `EventPublisher` |
| **Event Consumer** | 이벤트 처리 | - 이벤트 수신<br>- 비즈니스 로직<br>- 독립 실행 | `EmailListener` |

---

## 4. 구현 방법

### 기본 구현: E-Commerce 주문 시스템 ⭐⭐⭐

```java
/**
 * ============================================
 * DOMAIN EVENTS (도메인 이벤트)
 * ============================================
 */

/**
 * Base Event
 */
public abstract class DomainEvent {
    private final String eventId;
    private final LocalDateTime occurredOn;
    
    protected DomainEvent() {
        this.eventId = UUID.randomUUID().toString();
        this.occurredOn = LocalDateTime.now();
    }
    
    public String getEventId() { return eventId; }
    public LocalDateTime getOccurredOn() { return occurredOn; }
}

/**
 * 주문 생성 이벤트
 */
public class OrderCreatedEvent extends DomainEvent {
    private final Long orderId;
    private final Long customerId;
    private final BigDecimal totalAmount;
    private final List<OrderItem> items;
    
    public OrderCreatedEvent(Long orderId, Long customerId, 
                            BigDecimal totalAmount, List<OrderItem> items) {
        super();
        this.orderId = orderId;
        this.customerId = customerId;
        this.totalAmount = totalAmount;
        this.items = new ArrayList<>(items);
    }
    
    public Long getOrderId() { return orderId; }
    public Long getCustomerId() { return customerId; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public List<OrderItem> getItems() { return Collections.unmodifiableList(items); }
}

/**
 * 결제 완료 이벤트
 */
public class PaymentCompletedEvent extends DomainEvent {
    private final Long orderId;
    private final Long paymentId;
    private final BigDecimal amount;
    private final String paymentMethod;
    
    public PaymentCompletedEvent(Long orderId, Long paymentId, 
                                BigDecimal amount, String paymentMethod) {
        super();
        this.orderId = orderId;
        this.paymentId = paymentId;
        this.amount = amount;
        this.paymentMethod = paymentMethod;
    }
    
    public Long getOrderId() { return orderId; }
    public Long getPaymentId() { return paymentId; }
    public BigDecimal getAmount() { return amount; }
    public String getPaymentMethod() { return paymentMethod; }
}

/**
 * 재고 부족 이벤트
 */
public class StockDepletedEvent extends DomainEvent {
    private final Long productId;
    private final String productName;
    private final int currentStock;
    private final int threshold;
    
    public StockDepletedEvent(Long productId, String productName, 
                             int currentStock, int threshold) {
        super();
        this.productId = productId;
        this.productName = productName;
        this.currentStock = currentStock;
        this.threshold = threshold;
    }
    
    public Long getProductId() { return productId; }
    public String getProductName() { return productName; }
    public int getCurrentStock() { return currentStock; }
    public int getThreshold() { return threshold; }
}

/**
 * 배송 시작 이벤트
 */
public class ShippingStartedEvent extends DomainEvent {
    private final Long orderId;
    private final String trackingNumber;
    private final String carrier;
    
    public ShippingStartedEvent(Long orderId, String trackingNumber, String carrier) {
        super();
        this.orderId = orderId;
        this.trackingNumber = trackingNumber;
        this.carrier = carrier;
    }
    
    public Long getOrderId() { return orderId; }
    public String getTrackingNumber() { return trackingNumber; }
    public String getCarrier() { return carrier; }
}

/**
 * ============================================
 * EVENT BUS (이벤트 버스)
 * ============================================
 */

/**
 * Event Listener 인터페이스
 */
@FunctionalInterface
public interface EventListener<T extends DomainEvent> {
    void handle(T event);
}

/**
 * 간단한 InMemory Event Bus
 */
public class InMemoryEventBus {
    private final Map<Class<? extends DomainEvent>, List<EventListener<?>>> listeners;
    private final ExecutorService executor;
    
    public InMemoryEventBus() {
        this.listeners = new ConcurrentHashMap<>();
        this.executor = Executors.newFixedThreadPool(10);
    }
    
    /**
     * 리스너 등록
     */
    public <T extends DomainEvent> void subscribe(
            Class<T> eventType, 
            EventListener<T> listener) {
        
        listeners.computeIfAbsent(eventType, k -> new CopyOnWriteArrayList<>())
                 .add(listener);
        
        System.out.println("✅ 구독 등록: " + eventType.getSimpleName() + 
                          " → " + listener.getClass().getSimpleName());
    }
    
    /**
     * 이벤트 발행 (비동기)
     */
    public <T extends DomainEvent> void publish(T event) {
        System.out.println("\n📢 이벤트 발행: " + event.getClass().getSimpleName());
        System.out.println("   ID: " + event.getEventId());
        System.out.println("   발생 시간: " + event.getOccurredOn());
        
        List<EventListener<?>> eventListeners = listeners.get(event.getClass());
        
        if (eventListeners == null || eventListeners.isEmpty()) {
            System.out.println("   ⚠️ 구독자 없음");
            return;
        }
        
        System.out.println("   → " + eventListeners.size() + "개 구독자에게 전달");
        
        // 각 리스너를 비동기로 실행
        for (EventListener<?> listener : eventListeners) {
            executor.submit(() -> {
                try {
                    @SuppressWarnings("unchecked")
                    EventListener<T> typedListener = (EventListener<T>) listener;
                    typedListener.handle(event);
                    
                } catch (Exception e) {
                    System.err.println("❌ 이벤트 처리 실패: " + 
                        listener.getClass().getSimpleName());
                    System.err.println("   오류: " + e.getMessage());
                }
            });
        }
    }
    
    /**
     * 종료
     */
    public void shutdown() {
        executor.shutdown();
        try {
            if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
        }
    }
}

/**
 * ============================================
 * EVENT PRODUCERS (이벤트 발행자)
 * ============================================
 */

/**
 * 주문 서비스 (이벤트 발행자)
 */
public class OrderService {
    private final OrderRepository orderRepository;
    private final InMemoryEventBus eventBus;
    
    public OrderService(OrderRepository orderRepository, InMemoryEventBus eventBus) {
        this.orderRepository = orderRepository;
        this.eventBus = eventBus;
    }
    
    /**
     * 주문 생성
     */
    public Order createOrder(Long customerId, List<OrderItem> items) {
        System.out.println("\n🛒 주문 생성 시작");
        
        // 1. 주문 생성
        Order order = new Order(customerId, items);
        Order savedOrder = orderRepository.save(order);
        
        System.out.println("✅ 주문 저장 완료: ID=" + savedOrder.getId());
        
        // 2. 이벤트 발행 (핵심!)
        OrderCreatedEvent event = new OrderCreatedEvent(
            savedOrder.getId(),
            savedOrder.getCustomerId(),
            savedOrder.getTotalAmount(),
            savedOrder.getItems()
        );
        
        eventBus.publish(event);
        
        return savedOrder;
    }
}

/**
 * 결제 서비스 (이벤트 발행자)
 */
public class PaymentService {
    private final PaymentRepository paymentRepository;
    private final InMemoryEventBus eventBus;
    
    public PaymentService(PaymentRepository paymentRepository, InMemoryEventBus eventBus) {
        this.paymentRepository = paymentRepository;
        this.eventBus = eventBus;
    }
    
    /**
     * 결제 처리
     */
    public void processPayment(Long orderId, BigDecimal amount, String method) {
        System.out.println("\n💳 결제 처리 시작");
        
        // 1. 결제 처리
        Payment payment = new Payment(orderId, amount, method);
        Payment savedPayment = paymentRepository.save(payment);
        
        System.out.println("✅ 결제 완료: ID=" + savedPayment.getId());
        
        // 2. 이벤트 발행
        PaymentCompletedEvent event = new PaymentCompletedEvent(
            orderId,
            savedPayment.getId(),
            amount,
            method
        );
        
        eventBus.publish(event);
    }
}

/**
 * 재고 서비스 (이벤트 발행자)
 */
public class InventoryService {
    private final InMemoryEventBus eventBus;
    private final Map<Long, Integer> stock = new ConcurrentHashMap<>();
    private final int STOCK_THRESHOLD = 10;
    
    public InventoryService(InMemoryEventBus eventBus) {
        this.eventBus = eventBus;
        
        // 초기 재고
        stock.put(1L, 15);
        stock.put(2L, 5);
    }
    
    /**
     * 재고 차감
     */
    public void decreaseStock(Long productId, int quantity) {
        System.out.println("\n📦 재고 차감: Product=" + productId + ", Qty=" + quantity);
        
        int current = stock.getOrDefault(productId, 0);
        int newStock = current - quantity;
        
        if (newStock < 0) {
            throw new IllegalStateException("재고 부족");
        }
        
        stock.put(productId, newStock);
        System.out.println("✅ 재고 차감 완료: " + current + " → " + newStock);
        
        // 재고 부족 시 이벤트 발행
        if (newStock < STOCK_THRESHOLD) {
            StockDepletedEvent event = new StockDepletedEvent(
                productId,
                "상품-" + productId,
                newStock,
                STOCK_THRESHOLD
            );
            
            eventBus.publish(event);
        }
    }
}

/**
 * ============================================
 * EVENT CONSUMERS (이벤트 구독자)
 * ============================================
 */

/**
 * 이메일 알림 리스너
 */
public class EmailNotificationListener implements EventListener<OrderCreatedEvent> {
    
    @Override
    public void handle(OrderCreatedEvent event) {
        System.out.println("\n📧 [이메일 리스너] 주문 확인 이메일 발송");
        System.out.println("   주문 ID: " + event.getOrderId());
        System.out.println("   고객 ID: " + event.getCustomerId());
        System.out.println("   금액: " + event.getTotalAmount());
        
        // 실제로는 이메일 발송
        try {
            Thread.sleep(500);  // 이메일 발송 시뮬레이션
            System.out.println("   ✅ 이메일 발송 완료");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}

/**
 * SMS 알림 리스너
 */
public class SmsNotificationListener implements EventListener<OrderCreatedEvent> {
    
    @Override
    public void handle(OrderCreatedEvent event) {
        System.out.println("\n📱 [SMS 리스너] 주문 확인 SMS 발송");
        System.out.println("   주문 ID: " + event.getOrderId());
        
        try {
            Thread.sleep(300);  // SMS 발송 시뮬레이션
            System.out.println("   ✅ SMS 발송 완료");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}

/**
 * 재고 관리 리스너
 */
public class InventoryListener implements EventListener<OrderCreatedEvent> {
    private final InventoryService inventoryService;
    
    public InventoryListener(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }
    
    @Override
    public void handle(OrderCreatedEvent event) {
        System.out.println("\n📦 [재고 리스너] 재고 차감");
        
        for (OrderItem item : event.getItems()) {
            inventoryService.decreaseStock(item.getProductId(), item.getQuantity());
        }
        
        System.out.println("   ✅ 재고 처리 완료");
    }
}

/**
 * 분석 리스너
 */
public class AnalyticsListener implements EventListener<OrderCreatedEvent> {
    
    @Override
    public void handle(OrderCreatedEvent event) {
        System.out.println("\n📊 [분석 리스너] 주문 데이터 수집");
        System.out.println("   주문 ID: " + event.getOrderId());
        System.out.println("   금액: " + event.getTotalAmount());
        System.out.println("   상품 수: " + event.getItems().size());
        
        // 분석 시스템에 데이터 전송
        System.out.println("   ✅ 분석 데이터 수집 완료");
    }
}

/**
 * 배송 준비 리스너 (결제 완료 후)
 */
public class ShippingListener implements EventListener<PaymentCompletedEvent> {
    private final InMemoryEventBus eventBus;
    
    public ShippingListener(InMemoryEventBus eventBus) {
        this.eventBus = eventBus;
    }
    
    @Override
    public void handle(PaymentCompletedEvent event) {
        System.out.println("\n🚚 [배송 리스너] 배송 준비");
        System.out.println("   주문 ID: " + event.getOrderId());
        System.out.println("   결제 금액: " + event.getAmount());
        
        try {
            Thread.sleep(200);
            
            // 배송 시작 이벤트 발행
            String trackingNumber = "TRK-" + System.currentTimeMillis();
            ShippingStartedEvent shippingEvent = new ShippingStartedEvent(
                event.getOrderId(),
                trackingNumber,
                "CJ대한통운"
            );
            
            eventBus.publish(shippingEvent);
            
            System.out.println("   ✅ 배송 준비 완료");
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}

/**
 * 재입고 알림 리스너
 */
public class RestockAlertListener implements EventListener<StockDepletedEvent> {
    
    @Override
    public void handle(StockDepletedEvent event) {
        System.out.println("\n⚠️  [재입고 리스너] 재고 부족 알림");
        System.out.println("   상품 ID: " + event.getProductId());
        System.out.println("   상품명: " + event.getProductName());
        System.out.println("   현재 재고: " + event.getCurrentStock());
        System.out.println("   임계값: " + event.getThreshold());
        System.out.println("   ✅ 구매팀에 재입고 요청 전송");
    }
}

/**
 * ============================================
 * DOMAIN MODELS
 * ============================================
 */

/**
 * Order Entity
 */
public class Order {
    private Long id;
    private Long customerId;
    private List<OrderItem> items;
    private BigDecimal totalAmount;
    private LocalDateTime createdAt;
    
    public Order(Long customerId, List<OrderItem> items) {
        this.customerId = customerId;
        this.items = new ArrayList<>(items);
        this.totalAmount = calculateTotal(items);
        this.createdAt = LocalDateTime.now();
    }
    
    private BigDecimal calculateTotal(List<OrderItem> items) {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
    
    // Getters, Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public Long getCustomerId() { return customerId; }
    public List<OrderItem> getItems() { return items; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public LocalDateTime getCreatedAt() { return createdAt; }
}

/**
 * OrderItem
 */
public class OrderItem {
    private Long productId;
    private String productName;
    private int quantity;
    private BigDecimal price;
    
    public OrderItem(Long productId, String productName, int quantity, BigDecimal price) {
        this.productId = productId;
        this.productName = productName;
        this.quantity = quantity;
        this.price = price;
    }
    
    public BigDecimal getSubtotal() {
        return price.multiply(BigDecimal.valueOf(quantity));
    }
    
    public Long getProductId() { return productId; }
    public String getProductName() { return productName; }
    public int getQuantity() { return quantity; }
    public BigDecimal getPrice() { return price; }
}

/**
 * Payment Entity
 */
public class Payment {
    private Long id;
    private Long orderId;
    private BigDecimal amount;
    private String method;
    private LocalDateTime paidAt;
    
    public Payment(Long orderId, BigDecimal amount, String method) {
        this.orderId = orderId;
        this.amount = amount;
        this.method = method;
        this.paidAt = LocalDateTime.now();
    }
    
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public Long getOrderId() { return orderId; }
    public BigDecimal getAmount() { return amount; }
    public String getMethod() { return method; }
    public LocalDateTime getPaidAt() { return paidAt; }
}

/**
 * ============================================
 * REPOSITORIES
 * ============================================
 */

public class OrderRepository {
    private final Map<Long, Order> storage = new ConcurrentHashMap<>();
    private final AtomicLong idGenerator = new AtomicLong(1);
    
    public Order save(Order order) {
        if (order.getId() == null) {
            order.setId(idGenerator.getAndIncrement());
        }
        storage.put(order.getId(), order);
        return order;
    }
    
    public Order findById(Long id) {
        return storage.get(id);
    }
}

public class PaymentRepository {
    private final Map<Long, Payment> storage = new ConcurrentHashMap<>();
    private final AtomicLong idGenerator = new AtomicLong(1);
    
    public Payment save(Payment payment) {
        if (payment.getId() == null) {
            payment.setId(idGenerator.getAndIncrement());
        }
        storage.put(payment.getId(), payment);
        return payment;
    }
}

/**
 * ============================================
 * APPLICATION (메인)
 * ============================================
 */
public class EventDrivenExample {
    public static void main(String[] args) {
        System.out.println("=== Event-Driven Architecture 예제 ===\n");
        
        // 1. Event Bus 생성
        InMemoryEventBus eventBus = new InMemoryEventBus();
        
        // 2. 서비스 생성
        OrderRepository orderRepository = new OrderRepository();
        PaymentRepository paymentRepository = new PaymentRepository();
        
        OrderService orderService = new OrderService(orderRepository, eventBus);
        PaymentService paymentService = new PaymentService(paymentRepository, eventBus);
        InventoryService inventoryService = new InventoryService(eventBus);
        
        // 3. 이벤트 리스너 등록
        System.out.println("📝 이벤트 리스너 등록\n");
        
        // OrderCreatedEvent 구독자들
        eventBus.subscribe(OrderCreatedEvent.class, new EmailNotificationListener());
        eventBus.subscribe(OrderCreatedEvent.class, new SmsNotificationListener());
        eventBus.subscribe(OrderCreatedEvent.class, new InventoryListener(inventoryService));
        eventBus.subscribe(OrderCreatedEvent.class, new AnalyticsListener());
        
        // PaymentCompletedEvent 구독자
        eventBus.subscribe(PaymentCompletedEvent.class, new ShippingListener(eventBus));
        
        // StockDepletedEvent 구독자
        eventBus.subscribe(StockDepletedEvent.class, new RestockAlertListener());
        
        System.out.println("\n" + "=".repeat(60));
        
        // 4. 주문 생성 (이벤트 발행!)
        List<OrderItem> items = Arrays.asList(
            new OrderItem(1L, "노트북", 1, new BigDecimal("1200000")),
            new OrderItem(2L, "마우스", 2, new BigDecimal("30000"))
        );
        
        Order order = orderService.createOrder(100L, items);
        
        // 잠시 대기 (비동기 처리 완료를 위해)
        sleep(2000);
        
        System.out.println("\n" + "=".repeat(60));
        
        // 5. 결제 처리 (이벤트 발행!)
        paymentService.processPayment(
            order.getId(),
            order.getTotalAmount(),
            "신용카드"
        );
        
        // 잠시 대기
        sleep(2000);
        
        System.out.println("\n" + "=".repeat(60));
        System.out.println("\n✅ 모든 이벤트 처리 완료");
        
        // 6. 종료
        eventBus.shutdown();
    }
    
    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**실행 결과:**
```
=== Event-Driven Architecture 예제 ===

📝 이벤트 리스너 등록

✅ 구독 등록: OrderCreatedEvent → EmailNotificationListener
✅ 구독 등록: OrderCreatedEvent → SmsNotificationListener
✅ 구독 등록: OrderCreatedEvent → InventoryListener
✅ 구독 등록: OrderCreatedEvent → AnalyticsListener
✅ 구독 등록: PaymentCompletedEvent → ShippingListener
✅ 구독 등록: StockDepletedEvent → RestockAlertListener

============================================================

🛒 주문 생성 시작
✅ 주문 저장 완료: ID=1

📢 이벤트 발행: OrderCreatedEvent
   ID: a1b2c3d4-...
   발생 시간: 2025-12-22T...
   → 4개 구독자에게 전달

📧 [이메일 리스너] 주문 확인 이메일 발송
   주문 ID: 1
   고객 ID: 100
   금액: 1260000
   ✅ 이메일 발송 완료

📱 [SMS 리스너] 주문 확인 SMS 발송
   주문 ID: 1
   ✅ SMS 발송 완료

📦 [재고 리스너] 재고 차감

📦 재고 차감: Product=1, Qty=1
✅ 재고 차감 완료: 15 → 14

📦 재고 차감: Product=2, Qty=2
✅ 재고 차감 완료: 5 → 3

📢 이벤트 발행: StockDepletedEvent
   ID: b2c3d4e5-...
   발생 시간: 2025-12-22T...
   → 1개 구독자에게 전달

⚠️  [재입고 리스너] 재고 부족 알림
   상품 ID: 2
   상품명: 상품-2
   현재 재고: 3
   임계값: 10
   ✅ 구매팀에 재입고 요청 전송

   ✅ 재고 처리 완료

📊 [분석 리스너] 주문 데이터 수집
   주문 ID: 1
   금액: 1260000
   상품 수: 2
   ✅ 분석 데이터 수집 완료

============================================================

💳 결제 처리 시작
✅ 결제 완료: ID=1

📢 이벤트 발행: PaymentCompletedEvent
   ID: c3d4e5f6-...
   발생 시간: 2025-12-22T...
   → 1개 구독자에게 전달

🚚 [배송 리스너] 배송 준비
   주문 ID: 1
   결제 금액: 1260000

📢 이벤트 발행: ShippingStartedEvent
   ID: d4e5f6g7-...
   발생 시간: 2025-12-22T...
   → 0개 구독자에게 전달
   ⚠️ 구독자 없음

   ✅ 배송 준비 완료

============================================================

✅ 모든 이벤트 처리 완료
```

---

## 5. 실전 예제

### 예제 1: Spring Events (Spring Framework) ⭐⭐⭐

```java
/**
 * ============================================
 * Spring Events 구현
 * ============================================
 */

/**
 * Event
 */
public class UserRegisteredEvent {
    private final Long userId;
    private final String email;
    private final LocalDateTime registeredAt;
    
    public UserRegisteredEvent(Long userId, String email) {
        this.userId = userId;
        this.email = email;
        this.registeredAt = LocalDateTime.now();
    }
    
    public Long getUserId() { return userId; }
    public String getEmail() { return email; }
    public LocalDateTime getRegisteredAt() { return registeredAt; }
}

/**
 * Event Publisher (Producer)
 */
@Service
public class UserService {
    private final UserRepository userRepository;
    private final ApplicationEventPublisher eventPublisher;
    
    @Autowired
    public UserService(UserRepository userRepository, 
                      ApplicationEventPublisher eventPublisher) {
        this.userRepository = userRepository;
        this.eventPublisher = eventPublisher;
    }
    
    /**
     * 사용자 등록
     */
    @Transactional
    public User registerUser(String email, String name, String password) {
        // 1. 사용자 생성
        User user = new User(email, name, password);
        User savedUser = userRepository.save(user);
        
        // 2. 이벤트 발행 (핵심!)
        UserRegisteredEvent event = new UserRegisteredEvent(
            savedUser.getId(),
            savedUser.getEmail()
        );
        
        eventPublisher.publishEvent(event);
        
        return savedUser;
    }
}

/**
 * Event Listeners (Consumers)
 */
@Component
public class UserEventListeners {
    
    /**
     * 환영 이메일 발송
     */
    @EventListener
    @Async  // 비동기 처리
    public void handleUserRegistered(UserRegisteredEvent event) {
        System.out.println("📧 환영 이메일 발송: " + event.getEmail());
        
        // 이메일 발송 로직
        try {
            Thread.sleep(1000);  // 시뮬레이션
            System.out.println("✅ 이메일 발송 완료");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    /**
     * CRM 등록
     */
    @EventListener
    @Async
    public void syncToCrm(UserRegisteredEvent event) {
        System.out.println("👤 CRM 동기화: " + event.getEmail());
        
        // CRM API 호출
        try {
            Thread.sleep(500);
            System.out.println("✅ CRM 동기화 완료");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    /**
     * 분석 데이터 수집
     */
    @EventListener
    @Async
    public void trackSignup(UserRegisteredEvent event) {
        System.out.println("📊 가입 추적: User=" + event.getUserId());
        
        // 분석 시스템에 전송
        System.out.println("✅ 분석 데이터 수집 완료");
    }
}

/**
 * 비동기 설정
 */
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}
```

---

### 예제 2: Message Queue (RabbitMQ/Kafka) ⭐⭐⭐

```java
/**
 * ============================================
 * RabbitMQ를 이용한 Event-Driven
 * ============================================
 */

/**
 * RabbitMQ 설정
 */
@Configuration
public class RabbitMQConfig {
    
    public static final String ORDER_EXCHANGE = "order.exchange";
    public static final String ORDER_CREATED_QUEUE = "order.created.queue";
    public static final String ORDER_CREATED_ROUTING_KEY = "order.created";
    
    @Bean
    public DirectExchange orderExchange() {
        return new DirectExchange(ORDER_EXCHANGE);
    }
    
    @Bean
    public Queue orderCreatedQueue() {
        return new Queue(ORDER_CREATED_QUEUE, true);  // durable
    }
    
    @Bean
    public Binding orderCreatedBinding() {
        return BindingBuilder
            .bind(orderCreatedQueue())
            .to(orderExchange())
            .with(ORDER_CREATED_ROUTING_KEY);
    }
}

/**
 * Event Publisher (RabbitMQ)
 */
@Service
public class OrderEventPublisher {
    private final RabbitTemplate rabbitTemplate;
    
    @Autowired
    public OrderEventPublisher(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }
    
    /**
     * 주문 생성 이벤트 발행
     */
    public void publishOrderCreated(OrderCreatedEvent event) {
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.ORDER_EXCHANGE,
            RabbitMQConfig.ORDER_CREATED_ROUTING_KEY,
            event
        );
        
        System.out.println("📢 RabbitMQ에 이벤트 발행: OrderCreated");
    }
}

/**
 * Event Consumer (RabbitMQ)
 */
@Component
public class OrderEventConsumer {
    
    /**
     * 주문 생성 이벤트 수신
     */
    @RabbitListener(queues = RabbitMQConfig.ORDER_CREATED_QUEUE)
    public void handleOrderCreated(OrderCreatedEvent event) {
        System.out.println("📥 RabbitMQ에서 이벤트 수신");
        System.out.println("   주문 ID: " + event.getOrderId());
        
        // 이벤트 처리
        processOrder(event);
    }
    
    private void processOrder(OrderCreatedEvent event) {
        // 비즈니스 로직 실행
        System.out.println("✅ 주문 처리 완료");
    }
}

/**
 * Kafka를 이용한 Event-Driven
 */
@Configuration
public class KafkaConfig {
    
    public static final String ORDER_TOPIC = "order-events";
    
    @Bean
    public NewTopic orderTopic() {
        return TopicBuilder
            .name(ORDER_TOPIC)
            .partitions(3)
            .replicas(1)
            .build();
    }
}

/**
 * Kafka Producer
 */
@Service
public class KafkaEventPublisher {
    private final KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate;
    
    @Autowired
    public KafkaEventPublisher(KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }
    
    /**
     * 이벤트 발행
     */
    public void publish(OrderCreatedEvent event) {
        kafkaTemplate.send(
            KafkaConfig.ORDER_TOPIC,
            event.getOrderId().toString(),  // key (partitioning)
            event
        );
        
        System.out.println("📢 Kafka에 이벤트 발행");
    }
}

/**
 * Kafka Consumer
 */
@Component
public class KafkaEventConsumer {
    
    @KafkaListener(
        topics = KafkaConfig.ORDER_TOPIC,
        groupId = "order-processor-group"
    )
    public void consume(OrderCreatedEvent event) {
        System.out.println("📥 Kafka에서 이벤트 수신");
        System.out.println("   주문 ID: " + event.getOrderId());
        
        // 처리
        processOrder(event);
    }
    
    private void processOrder(OrderCreatedEvent event) {
        System.out.println("✅ 이벤트 처리 완료");
    }
}
```

---

## 6. 이벤트 패턴 변형

### 📊 주요 변형들

| 패턴 | 특징 | 사용처 |
|------|------|--------|
| **Simple Event** | 단순 알림 | 로깅, 모니터링 |
| **Event Sourcing** | 이벤트를 영구 저장 | 감사, 이력 추적 |
| **CQRS** | Command/Query 분리 | 읽기/쓰기 분리 |
| **Saga** | 분산 트랜잭션 | 마이크로서비스 |

### 🔄 Event Sourcing

```java
/**
 * Event Store
 */
public class EventStore {
    private final List<DomainEvent> events = new ArrayList<>();
    
    public void append(DomainEvent event) {
        events.add(event);
    }
    
    public List<DomainEvent> getEventsForAggregate(String aggregateId) {
        // 특정 Aggregate의 이벤트 조회
        return events;
    }
    
    public void replay(String aggregateId, EventHandler handler) {
        getEventsForAggregate(aggregateId).forEach(handler::handle);
    }
}

/**
 * Aggregate 재구성
 */
public class Order {
    private Long id;
    private OrderStatus status;
    
    public static Order reconstruct(List<DomainEvent> events) {
        Order order = new Order();
        
        for (DomainEvent event : events) {
            order.apply(event);
        }
        
        return order;
    }
    
    private void apply(DomainEvent event) {
        if (event instanceof OrderCreatedEvent) {
            this.id = ((OrderCreatedEvent) event).getOrderId();
            this.status = OrderStatus.CREATED;
            
        } else if (event instanceof OrderPaidEvent) {
            this.status = OrderStatus.PAID;
            
        } else if (event instanceof OrderShippedEvent) {
            this.status = OrderStatus.SHIPPED;
        }
    }
}
```

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 | 실무 효과 |
|------|------|-----------|
| **느슨한 결합** | 이벤트로 통신 | 독립 개발 |
| **확장성** | 구독자 추가 쉬움 | 기능 확장 |
| **비동기 처리** | 성능 향상 | 빠른 응답 |
| **탄력성** | 일부 실패 격리 | 안정성 |
| **이력 추적** | 이벤트 로그 | 감사 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **복잡도** | 디버깅 어려움 | 로깅, 모니터링 |
| **순서 보장** | 이벤트 순서 | 메시지 큐 |
| **중복 처리** | 멱등성 필요 | 이벤트 ID |
| **일관성** | Eventually Consistent | Saga 패턴 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: 이벤트에 너무 많은 데이터

```java
// 잘못된 예: 모든 데이터를 포함
public class OrderCreatedEvent {
    private Order order;  // ❌ 전체 Order 객체
    private Customer customer;  // ❌
    private List<Product> products;  // ❌
    // 너무 무거움!
}
```

**해결:**
```java
// 올바른 예: 최소한의 데이터
public class OrderCreatedEvent {
    private Long orderId;  // ✅ ID만
    private Long customerId;
    private BigDecimal totalAmount;
    // 필요한 최소 정보만!
}
```

### ❌ 안티패턴 2: 이벤트 체인이 너무 김

```
// 잘못된 예: 이벤트 체인
Event1 → Event2 → Event3 → Event4 → Event5
// 디버깅 악몽!
```

**해결:**
```
// 올바른 예: 명확한 의도
Event1 → [Process] → Event2
// 간결하고 명확!
```

---

## 9. 심화 주제

### 🎯 Saga 패턴 (분산 트랜잭션)

```java
/**
 * Saga Orchestrator
 */
public class OrderSaga {
    
    public void execute(Order order) {
        try {
            // 1. 결제
            Payment payment = paymentService.charge(order);
            
            // 2. 재고 차감
            inventoryService.decrease(order);
            
            // 3. 배송 준비
            shippingService.prepare(order);
            
        } catch (Exception e) {
            // 보상 트랜잭션 (Compensation)
            compensate(order);
        }
    }
    
    private void compensate(Order order) {
        // 역순으로 취소
        shippingService.cancel(order);
        inventoryService.restore(order);
        paymentService.refund(order);
    }
}
```

---

## 10. 핵심 정리

### 📌 Event-Driven 체크리스트

```
✅ 이벤트는 불변 객체
✅ 과거형 이름 (OrderCreated)
✅ 이벤트에 충분한 정보
✅ 비동기 처리
✅ 멱등성 보장
✅ 에러 처리
✅ 모니터링
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 마이크로서비스 | ⭐⭐⭐ | 느슨한 결합 |
| 비동기 처리 | ⭐⭐⭐ | 성능 향상 |
| 확장 필요 | ⭐⭐⭐ | 구독자 추가 |
| 간단한 CRUD | ⭐ | 오버엔지니어링 |

### 💡 핵심 원칙

1. **이벤트는 과거형**
2. **느슨한 결합**
3. **비동기 처리**
4. **멱등성 보장**

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: MVVM](./05-MVVM.md) | [다음: Microservices →](./07-Microservices.md)**

</div>
