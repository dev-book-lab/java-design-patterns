# Unit of Work Pattern (작업 단위 패턴)

> **"여러 비즈니스 작업을 하나의 트랜잭션으로 묶어 일관성을 보장하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [Spring과의 통합](#6-spring과의-통합)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 여러 Repository 호출 시 트랜잭션 문제
public class OrderService {
    private OrderRepository orderRepository;
    private ProductRepository productRepository;
    private UserRepository userRepository;
    
    public void createOrder(CreateOrderRequest request) {
        // 😱 각 저장이 별도 트랜잭션!
        
        // 1. 주문 생성
        Order order = new Order();
        orderRepository.save(order);  // 트랜잭션 1
        
        // 2. 재고 차감
        Product product = productRepository.findById(request.getProductId());
        product.decreaseStock(request.getQuantity());
        productRepository.save(product);  // 트랜잭션 2
        
        // 3. 포인트 차감
        User user = userRepository.findById(request.getUserId());
        user.usePoints(request.getPoints());
        userRepository.save(user);  // 트랜잭션 3
        
        // 문제점:
        // - 주문은 생성되었는데 재고 차감 실패?
        // - 재고는 차감되었는데 포인트 차감 실패?
        // - 데이터 불일치!
        // - 롤백 불가능!
    }
}

// 문제 2: 중복 저장 (Same Object)
public class ProductService {
    private ProductRepository productRepository;
    
    public void updateProduct(Long id) {
        // 😱 같은 객체를 여러 번 저장!
        
        Product product = productRepository.findById(id);
        
        // 1차 수정
        product.setName("수정된 이름");
        productRepository.save(product);  // DB UPDATE 1
        
        // 2차 수정
        product.setPrice(new BigDecimal("10000"));
        productRepository.save(product);  // DB UPDATE 2 (불필요!)
        
        // 3차 수정
        product.setStock(100);
        productRepository.save(product);  // DB UPDATE 3 (불필요!)
        
        // 문제점:
        // - 3번의 UPDATE 쿼리 실행
        // - 성능 저하
        // - 마지막 한 번만 저장하면 되는데!
    }
}

// 문제 3: 변경 추적 어려움
public class OrderService {
    public void processOrder(Order order) {
        // 😱 무엇이 변경되었는지 모름!
        
        order.setStatus(OrderStatus.CONFIRMED);
        order.setConfirmedAt(LocalDateTime.now());
        
        // 뭐가 변경되었는지?
        // - status만? 
        // - confirmedAt만?
        // - 둘 다?
        
        // 전체 필드를 다시 저장? (비효율)
        // 변경된 필드만 저장? (어떻게?)
    }
}

// 문제 4: Cascade 처리 복잡
public class OrderService {
    private OrderRepository orderRepository;
    private OrderItemRepository orderItemRepository;
    
    public void createOrder(Order order) {
        // 😱 연관 객체 저장 순서 신경써야 함!
        
        // 1. Order 먼저 저장 (FK 필요)
        orderRepository.save(order);
        
        // 2. OrderItem 저장 (Order ID 필요)
        for (OrderItem item : order.getItems()) {
            item.setOrderId(order.getId());
            orderItemRepository.save(item);  // 각각 저장!
        }
        
        // 문제점:
        // - 저장 순서 관리
        // - 수동으로 FK 설정
        // - N+1 쿼리
    }
}

// 문제 5: 트랜잭션 범위 불명확
public class UserService {
    public void registerUser(User user) {
        // 트랜잭션 시작은 언제?
        userRepository.save(user);
        
        // 여기서 예외 발생하면?
        sendWelcomeEmail(user);
        
        // 트랜잭션 종료는 언제?
        // 롤백은 언제?
    }
}

// 문제 6: 부분 실패 처리
public class BulkUpdateService {
    public void updateProducts(List<Product> products) {
        for (Product product : products) {
            try {
                // 😱 하나씩 별도 트랜잭션
                productRepository.save(product);
                
            } catch (Exception e) {
                // 일부만 성공, 일부는 실패
                // 전체를 롤백하려면?
            }
        }
        
        // 문제점:
        // - All or Nothing이 보장 안 됨
        // - 일관성 없음
    }
}

// 문제 7: 동일 객체 여러 번 조회
public class OrderService {
    public void processOrder(Long orderId) {
        // 😱 같은 Order를 여러 번 조회!
        
        Order order1 = orderRepository.findById(orderId);  // SELECT 1
        order1.confirm();
        
        // 다른 메서드에서 또 조회
        Order order2 = orderRepository.findById(orderId);  // SELECT 2 (같은 객체!)
        order2.prepare();
        
        // 문제점:
        // - 중복 쿼리
        // - 메모리 낭비
        // - Identity Map 없음
    }
}

// 문제 8: Lazy Loading 문제
public class OrderService {
    @Transactional
    public void processOrder(Long orderId) {
        Order order = orderRepository.findById(orderId);
        
        // 트랜잭션 종료 후
    }
    
    public void displayOrder(Order order) {
        // 😱 LazyInitializationException!
        System.out.println(order.getUser().getName());  // Error!
        
        // 문제점:
        // - 트랜잭션 밖에서 Lazy Loading
        // - Session 없음
    }
}
```

### ⚡ 핵심 문제

1. **트랜잭션 불일치**: 여러 작업이 원자적으로 처리 안 됨
2. **중복 저장**: 같은 객체를 여러 번 저장
3. **변경 추적**: 무엇이 변경되었는지 모름
4. **성능**: 불필요한 쿼리 반복
5. **일관성**: 부분 실패 시 데이터 불일치
6. **복잡도**: 수동 트랜잭션 관리

---

## 2. 패턴 정의

### 📖 정의

> **비즈니스 트랜잭션 동안 영향받은 객체들의 변경사항을 추적하고, 작업이 완료될 때 모든 변경사항을 한 번에 데이터베이스에 반영하는 패턴**

### 🎯 목적

- **원자성**: 여러 작업을 하나의 트랜잭션으로
- **변경 추적**: 변경된 객체만 저장
- **중복 제거**: 같은 객체는 한 번만 저장
- **성능**: 배치 처리로 최적화

### 💡 핵심 아이디어

```java
// Before: Repository 직접 호출
public class OrderService {
    private OrderRepository orderRepository;
    private ProductRepository productRepository;
    
    public void createOrder(Order order, Product product) {
        // 😱 각각 저장
        orderRepository.save(order);
        productRepository.save(product);
        
        // 트랜잭션 관리?
        // 롤백?
    }
}

// After: Unit of Work 사용
public class OrderService {
    private UnitOfWork unitOfWork;
    
    public void createOrder(Order order, Product product) {
        // 😊 Unit of Work에 등록만
        unitOfWork.registerNew(order);
        unitOfWork.registerDirty(product);
        
        // 나중에 한 번에 커밋!
        unitOfWork.commit();
        
        // 또는 자동 커밋 (@Transactional)
    }
}

// Unit of Work가 하는 일:
// 1. 변경된 객체 추적
// 2. 순서대로 저장 (Order → OrderItem)
// 3. 한 번의 트랜잭션으로
// 4. 실패 시 전체 롤백
```

---

## 3. 구조와 구성요소

### 📊 Unit of Work 구조

```
┌─────────────────────────────────────┐
│       Business Logic                │
│                                     │
│  unitOfWork.registerNew(order)      │
│  unitOfWork.registerDirty(product)  │
│  unitOfWork.commit()                │ 
└─────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│        Unit of Work                 │
│                                     │
│  - New Objects: []                  │
│  - Dirty Objects: []                │
│  - Removed Objects: []              │
│                                     │
│  commit() {                         │
│    BEGIN TRANSACTION                │
│    INSERT New Objects               │
│    UPDATE Dirty Objects             │
│    DELETE Removed Objects           │
│    COMMIT TRANSACTION               │
│  }                                  │
└─────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│         Database                    │
└─────────────────────────────────────┘
```

### 🔄 객체 생명주기

```
New Object
    │
    │ registerNew()
    ▼
[New Objects List]
    │
    │ commit()
    ▼
INSERT INTO DB
    │
    ▼
Persistent Object
    │
    │ modify()
    │ registerDirty()
    ▼
[Dirty Objects List]
    │
    │ commit()
    ▼
UPDATE DB
    │
    ▼
[Clean Objects]
    │
    │ registerRemoved()
    ▼
[Removed Objects List]
    │
    │ commit()
    ▼
DELETE FROM DB
```

### 🔧 구성요소

| 컴포넌트 | 역할 | 책임 |
|---------|------|------|
| **Unit of Work** | 작업 단위 관리 | 변경 추적, 커밋/롤백 |
| **Identity Map** | 객체 캐시 | 중복 조회 방지 |
| **Change Tracker** | 변경 감지 | Dirty Checking |
| **Transaction** | 트랜잭션 관리 | BEGIN/COMMIT/ROLLBACK |

---

## 4. 구현 방법

### 완전한 구현: Unit of Work ⭐⭐⭐

```java
/**
 * ============================================
 * UNIT OF WORK INTERFACE
 * ============================================
 */
public interface UnitOfWork {
    /**
     * 새 객체 등록
     */
    void registerNew(Object entity);
    
    /**
     * 변경된 객체 등록
     */
    void registerDirty(Object entity);
    
    /**
     * 삭제할 객체 등록
     */
    void registerRemoved(Object entity);
    
    /**
     * 변경사항 커밋
     */
    void commit();
    
    /**
     * 변경사항 롤백
     */
    void rollback();
    
    /**
     * Identity Map에서 조회
     */
    <T> T find(Class<T> entityClass, Object id);
}

/**
 * ============================================
 * UNIT OF WORK IMPLEMENTATION
 * ============================================
 */
public class UnitOfWorkImpl implements UnitOfWork {
    
    // 변경 추적 리스트
    private final List<Object> newObjects = new ArrayList<>();
    private final List<Object> dirtyObjects = new ArrayList<>();
    private final List<Object> removedObjects = new ArrayList<>();
    
    // Identity Map (캐시)
    private final Map<String, Object> identityMap = new HashMap<>();
    
    // Repositories
    private final Map<Class<?>, Repository<?>> repositories = new HashMap<>();
    
    // Transaction
    private Connection connection;
    private boolean inTransaction = false;
    
    public UnitOfWorkImpl(Connection connection) {
        this.connection = connection;
    }
    
    /**
     * Repository 등록
     */
    public <T> void registerRepository(Class<T> entityClass, Repository<T> repository) {
        repositories.put(entityClass, repository);
    }
    
    @Override
    public void registerNew(Object entity) {
        if (entity == null) {
            throw new IllegalArgumentException("Entity cannot be null");
        }
        
        System.out.println("➕ UnitOfWork: New 등록 - " + entity.getClass().getSimpleName());
        
        // 중복 등록 방지
        if (!newObjects.contains(entity) && 
            !dirtyObjects.contains(entity) && 
            !removedObjects.contains(entity)) {
            newObjects.add(entity);
        }
    }
    
    @Override
    public void registerDirty(Object entity) {
        if (entity == null) {
            throw new IllegalArgumentException("Entity cannot be null");
        }
        
        System.out.println("✏️ UnitOfWork: Dirty 등록 - " + entity.getClass().getSimpleName());
        
        // New나 Removed 상태면 무시
        if (!newObjects.contains(entity) && !removedObjects.contains(entity)) {
            if (!dirtyObjects.contains(entity)) {
                dirtyObjects.add(entity);
            }
        }
    }
    
    @Override
    public void registerRemoved(Object entity) {
        if (entity == null) {
            throw new IllegalArgumentException("Entity cannot be null");
        }
        
        System.out.println("🗑️ UnitOfWork: Removed 등록 - " + entity.getClass().getSimpleName());
        
        // New 목록에서 제거
        newObjects.remove(entity);
        
        // Dirty 목록에서 제거
        dirtyObjects.remove(entity);
        
        // Removed에 추가
        if (!removedObjects.contains(entity)) {
            removedObjects.add(entity);
        }
    }
    
    @Override
    public void commit() {
        if (inTransaction) {
            throw new IllegalStateException("Already in transaction");
        }
        
        try {
            System.out.println("\n💾 UnitOfWork: Commit 시작");
            
            // 트랜잭션 시작
            connection.setAutoCommit(false);
            inTransaction = true;
            
            // 1. INSERT (새 객체들)
            insertNewObjects();
            
            // 2. UPDATE (변경된 객체들)
            updateDirtyObjects();
            
            // 3. DELETE (삭제할 객체들)
            deleteRemovedObjects();
            
            // 커밋
            connection.commit();
            System.out.println("✅ UnitOfWork: Commit 성공");
            
            // 정리
            clear();
            
        } catch (Exception e) {
            System.err.println("❌ UnitOfWork: Commit 실패 - " + e.getMessage());
            rollback();
            throw new RuntimeException("Unit of Work commit failed", e);
            
        } finally {
            try {
                connection.setAutoCommit(true);
                inTransaction = false;
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }
    
    @Override
    public void rollback() {
        if (!inTransaction) {
            return;
        }
        
        try {
            System.out.println("\n🔄 UnitOfWork: Rollback");
            connection.rollback();
            clear();
            
        } catch (SQLException e) {
            System.err.println("❌ Rollback 실패: " + e.getMessage());
            
        } finally {
            try {
                connection.setAutoCommit(true);
                inTransaction = false;
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }
    
    @Override
    @SuppressWarnings("unchecked")
    public <T> T find(Class<T> entityClass, Object id) {
        String key = getIdentityKey(entityClass, id);
        
        // 1. Identity Map에서 먼저 찾기
        Object cached = identityMap.get(key);
        if (cached != null) {
            System.out.println("🔍 UnitOfWork: Identity Map에서 조회 - " + key);
            return (T) cached;
        }
        
        // 2. Repository에서 조회
        Repository<T> repository = (Repository<T>) repositories.get(entityClass);
        if (repository == null) {
            throw new IllegalStateException("No repository for " + entityClass.getSimpleName());
        }
        
        T entity = repository.findById(id);
        
        // 3. Identity Map에 캐싱
        if (entity != null) {
            identityMap.put(key, entity);
        }
        
        return entity;
    }
    
    /**
     * ============================================
     * PRIVATE METHODS
     * ============================================
     */
    
    /**
     * 새 객체들 INSERT
     */
    @SuppressWarnings("unchecked")
    private void insertNewObjects() {
        System.out.println("   → INSERT " + newObjects.size() + "개");
        
        for (Object entity : newObjects) {
            Repository<Object> repository = (Repository<Object>) repositories.get(entity.getClass());
            if (repository == null) {
                throw new IllegalStateException("No repository for " + entity.getClass().getSimpleName());
            }
            
            Object saved = repository.insert(entity);
            
            // Identity Map에 추가
            String key = getIdentityKey(entity.getClass(), getEntityId(saved));
            identityMap.put(key, saved);
        }
    }
    
    /**
     * 변경된 객체들 UPDATE
     */
    @SuppressWarnings("unchecked")
    private void updateDirtyObjects() {
        System.out.println("   → UPDATE " + dirtyObjects.size() + "개");
        
        for (Object entity : dirtyObjects) {
            Repository<Object> repository = (Repository<Object>) repositories.get(entity.getClass());
            if (repository == null) {
                throw new IllegalStateException("No repository for " + entity.getClass().getSimpleName());
            }
            
            repository.update(entity);
        }
    }
    
    /**
     * 삭제할 객체들 DELETE
     */
    @SuppressWarnings("unchecked")
    private void deleteRemovedObjects() {
        System.out.println("   → DELETE " + removedObjects.size() + "개");
        
        for (Object entity : removedObjects) {
            Repository<Object> repository = (Repository<Object>) repositories.get(entity.getClass());
            if (repository == null) {
                throw new IllegalStateException("No repository for " + entity.getClass().getSimpleName());
            }
            
            repository.delete(getEntityId(entity));
            
            // Identity Map에서 제거
            String key = getIdentityKey(entity.getClass(), getEntityId(entity));
            identityMap.remove(key);
        }
    }
    
    /**
     * 정리
     */
    private void clear() {
        newObjects.clear();
        dirtyObjects.clear();
        removedObjects.clear();
    }
    
    /**
     * Identity Key 생성
     */
    private String getIdentityKey(Class<?> entityClass, Object id) {
        return entityClass.getSimpleName() + "#" + id;
    }
    
    /**
     * Entity ID 가져오기 (리플렉션 사용)
     */
    private Object getEntityId(Object entity) {
        try {
            // getId() 메서드 호출
            return entity.getClass().getMethod("getId").invoke(entity);
        } catch (Exception e) {
            throw new RuntimeException("Cannot get entity ID", e);
        }
    }
}

/**
 * ============================================
 * REPOSITORY INTERFACE
 * ============================================
 */
public interface Repository<T> {
    T findById(Object id);
    T insert(T entity);
    void update(T entity);
    void delete(Object id);
}

/**
 * ============================================
 * ENTITY EXAMPLE
 * ============================================
 */
public class Order {
    private Long id;
    private Long userId;
    private BigDecimal totalAmount;
    private OrderStatus status;
    private LocalDateTime createdAt;
    
    public enum OrderStatus {
        PENDING, CONFIRMED, SHIPPED, COMPLETED, CANCELLED
    }
    
    // Constructors, Getters, Setters
    public Order() {}
    
    public Order(Long userId, BigDecimal totalAmount) {
        this.userId = userId;
        this.totalAmount = totalAmount;
        this.status = OrderStatus.PENDING;
        this.createdAt = LocalDateTime.now();
    }
    
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public Long getUserId() { return userId; }
    public void setUserId(Long userId) { this.userId = userId; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public void setTotalAmount(BigDecimal totalAmount) { this.totalAmount = totalAmount; }
    public OrderStatus getStatus() { return status; }
    public void setStatus(OrderStatus status) { this.status = status; }
    public LocalDateTime getCreatedAt() { return createdAt; }
}

/**
 * ============================================
 * ORDER REPOSITORY
 * ============================================
 */
public class OrderRepository implements Repository<Order> {
    private final Connection connection;
    
    public OrderRepository(Connection connection) {
        this.connection = connection;
    }
    
    @Override
    public Order findById(Object id) {
        String sql = "SELECT * FROM orders WHERE id = ?";
        
        try (PreparedStatement stmt = connection.prepareStatement(sql)) {
            stmt.setLong(1, (Long) id);
            
            try (ResultSet rs = stmt.executeQuery()) {
                if (rs.next()) {
                    return mapResultSetToOrder(rs);
                }
            }
            
            return null;
            
        } catch (SQLException e) {
            throw new RuntimeException("Find failed", e);
        }
    }
    
    @Override
    public Order insert(Order order) {
        String sql = "INSERT INTO orders (user_id, total_amount, status, created_at) VALUES (?, ?, ?, ?)";
        
        try (PreparedStatement stmt = connection.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            stmt.setLong(1, order.getUserId());
            stmt.setBigDecimal(2, order.getTotalAmount());
            stmt.setString(3, order.getStatus().name());
            stmt.setTimestamp(4, Timestamp.valueOf(order.getCreatedAt()));
            
            stmt.executeUpdate();
            
            try (ResultSet generatedKeys = stmt.getGeneratedKeys()) {
                if (generatedKeys.next()) {
                    order.setId(generatedKeys.getLong(1));
                }
            }
            
            System.out.println("      ✓ Order INSERT: ID=" + order.getId());
            return order;
            
        } catch (SQLException e) {
            throw new RuntimeException("Insert failed", e);
        }
    }
    
    @Override
    public void update(Order order) {
        String sql = "UPDATE orders SET user_id = ?, total_amount = ?, status = ? WHERE id = ?";
        
        try (PreparedStatement stmt = connection.prepareStatement(sql)) {
            stmt.setLong(1, order.getUserId());
            stmt.setBigDecimal(2, order.getTotalAmount());
            stmt.setString(3, order.getStatus().name());
            stmt.setLong(4, order.getId());
            
            stmt.executeUpdate();
            System.out.println("      ✓ Order UPDATE: ID=" + order.getId());
            
        } catch (SQLException e) {
            throw new RuntimeException("Update failed", e);
        }
    }
    
    @Override
    public void delete(Object id) {
        String sql = "DELETE FROM orders WHERE id = ?";
        
        try (PreparedStatement stmt = connection.prepareStatement(sql)) {
            stmt.setLong(1, (Long) id);
            stmt.executeUpdate();
            
            System.out.println("      ✓ Order DELETE: ID=" + id);
            
        } catch (SQLException e) {
            throw new RuntimeException("Delete failed", e);
        }
    }
    
    private Order mapResultSetToOrder(ResultSet rs) throws SQLException {
        Order order = new Order();
        order.setId(rs.getLong("id"));
        order.setUserId(rs.getLong("user_id"));
        order.setTotalAmount(rs.getBigDecimal("total_amount"));
        order.setStatus(Order.OrderStatus.valueOf(rs.getString("status")));
        order.setCreatedAt(rs.getTimestamp("created_at").toLocalDateTime());
        return order;
    }
}

/**
 * ============================================
 * SERVICE (Unit of Work 사용)
 * ============================================
 */
public class OrderService {
    private final UnitOfWork unitOfWork;
    
    public OrderService(UnitOfWork unitOfWork) {
        this.unitOfWork = unitOfWork;
    }
    
    /**
     * 주문 생성 (여러 작업을 하나의 트랜잭션으로)
     */
    public Order createOrder(Long userId, BigDecimal amount) {
        System.out.println("\n🛒 OrderService: 주문 생성");
        
        // 1. Order 생성
        Order order = new Order(userId, amount);
        unitOfWork.registerNew(order);
        
        // 2. 다른 객체도 등록 가능
        // Product product = ...
        // unitOfWork.registerDirty(product);
        
        // 3. Commit (모든 변경사항을 한 번에!)
        unitOfWork.commit();
        
        return order;
    }
    
    /**
     * 주문 수정
     */
    public void updateOrder(Long orderId, Order.OrderStatus newStatus) {
        System.out.println("\n✏️ OrderService: 주문 수정");
        
        // 1. 조회 (Identity Map 사용)
        Order order = unitOfWork.find(Order.class, orderId);
        
        // 2. 수정
        order.setStatus(newStatus);
        unitOfWork.registerDirty(order);
        
        // 3. Commit
        unitOfWork.commit();
    }
    
    /**
     * 주문 삭제
     */
    public void deleteOrder(Long orderId) {
        System.out.println("\n🗑️ OrderService: 주문 삭제");
        
        // 1. 조회
        Order order = unitOfWork.find(Order.class, orderId);
        
        // 2. 삭제 등록
        unitOfWork.registerRemoved(order);
        
        // 3. Commit
        unitOfWork.commit();
    }
}

/**
 * ============================================
 * DEMO
 * ============================================
 */
public class UnitOfWorkDemo {
    public static void main(String[] args) {
        System.out.println("=== Unit of Work Pattern 예제 ===\n");
        
        try {
            // 1. DB 연결
            Connection connection = createConnection();
            createTable(connection);
            
            // 2. Unit of Work 생성
            UnitOfWork unitOfWork = new UnitOfWorkImpl(connection);
            
            // 3. Repository 등록
            OrderRepository orderRepository = new OrderRepository(connection);
            ((UnitOfWorkImpl) unitOfWork).registerRepository(Order.class, orderRepository);
            
            // 4. Service 생성
            OrderService orderService = new OrderService(unitOfWork);
            
            // 5. 주문 생성
            Order order = orderService.createOrder(1L, new BigDecimal("50000"));
            
            System.out.println("\n" + "=".repeat(60));
            
            // 6. 주문 수정
            orderService.updateOrder(order.getId(), Order.OrderStatus.CONFIRMED);
            
            System.out.println("\n" + "=".repeat(60));
            
            // 7. Identity Map 테스트 (중복 조회 방지)
            System.out.println("\n🔍 Identity Map 테스트:");
            Order order1 = unitOfWork.find(Order.class, order.getId());
            Order order2 = unitOfWork.find(Order.class, order.getId());
            System.out.println("   Same instance? " + (order1 == order2));  // true!
            
            System.out.println("\n" + "=".repeat(60));
            
            // 8. 주문 삭제
            orderService.deleteOrder(order.getId());
            
            connection.close();
            
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    
    private static Connection createConnection() throws SQLException {
        return DriverManager.getConnection("jdbc:h2:mem:testdb", "sa", "");
    }
    
    private static void createTable(Connection connection) throws SQLException {
        String sql = """
            CREATE TABLE orders (
                id BIGINT AUTO_INCREMENT PRIMARY KEY,
                user_id BIGINT NOT NULL,
                total_amount DECIMAL(10, 2) NOT NULL,
                status VARCHAR(20) NOT NULL,
                created_at TIMESTAMP NOT NULL
            )
            """;
        
        try (PreparedStatement stmt = connection.prepareStatement(sql)) {
            stmt.executeUpdate();
            System.out.println("📊 테이블 생성 완료\n");
        }
    }
}
```

**실행 결과:**
```
=== Unit of Work Pattern 예제 ===

📊 테이블 생성 완료

🛒 OrderService: 주문 생성
➕ UnitOfWork: New 등록 - Order

💾 UnitOfWork: Commit 시작
   → INSERT 1개
      ✓ Order INSERT: ID=1
   → UPDATE 0개
   → DELETE 0개
✅ UnitOfWork: Commit 성공

============================================================

✏️ OrderService: 주문 수정
✏️ UnitOfWork: Dirty 등록 - Order

💾 UnitOfWork: Commit 시작
   → INSERT 0개
   → UPDATE 1개
      ✓ Order UPDATE: ID=1
   → DELETE 0개
✅ UnitOfWork: Commit 성공

============================================================

🔍 Identity Map 테스트:
🔍 UnitOfWork: Identity Map에서 조회 - Order#1
   Same instance? true

============================================================

🗑️ OrderService: 주문 삭제
🗑️ UnitOfWork: Removed 등록 - Order

💾 UnitOfWork: Commit 시작
   → INSERT 0개
   → UPDATE 0개
   → DELETE 1개
      ✓ Order DELETE: ID=1
✅ UnitOfWork: Commit 성공
```

---

## 5. 실전 예제

### 예제 1: 복잡한 비즈니스 트랜잭션 ⭐⭐⭐

```java
/**
 * 주문 + 재고 + 포인트를 하나의 트랜잭션으로
 */
public class ComplexOrderService {
    private final UnitOfWork unitOfWork;
    
    public Order createCompleteOrder(CreateOrderRequest request) {
        System.out.println("\n🛒 복잡한 주문 생성");
        
        // 1. Order 생성
        Order order = new Order(request.getUserId(), request.getAmount());
        unitOfWork.registerNew(order);
        
        // 2. Product 재고 차감
        Product product = unitOfWork.find(Product.class, request.getProductId());
        product.decreaseStock(request.getQuantity());
        unitOfWork.registerDirty(product);
        
        // 3. User 포인트 차감
        User user = unitOfWork.find(User.class, request.getUserId());
        user.usePoints(request.getUsedPoints());
        unitOfWork.registerDirty(user);
        
        // 4. OrderItem 생성
        OrderItem item = new OrderItem(order.getId(), product.getId(), request.getQuantity());
        unitOfWork.registerNew(item);
        
        // 5. 모두 한 번에 커밋!
        // - 실패 시 전부 롤백
        // - 성공 시 전부 저장
        unitOfWork.commit();
        
        System.out.println("   ✅ 모든 작업 완료 (원자적)");
        
        return order;
    }
}
```

---

## 6. Spring과의 통합

### 🔄 Spring @Transactional

```java
/**
 * Spring은 Unit of Work를 자동으로 제공!
 */
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private ProductRepository productRepository;
    
    @Transactional  // ← 이게 Unit of Work!
    public Order createOrder(CreateOrderRequest request) {
        // 1. Order 생성
        Order order = new Order();
        orderRepository.save(order);  // Unit of Work에 등록
        
        // 2. Product 재고 차감
        Product product = productRepository.findById(request.getProductId());
        product.decreaseStock(request.getQuantity());
        // save() 호출 안 해도 됨! (Dirty Checking)
        
        // 3. 메서드 종료 시 자동 commit
        // 예외 발생 시 자동 rollback
        
        return order;
    }
}
```

**Spring의 Unit of Work:**
- `@Transactional` = Unit of Work
- `EntityManager` = Identity Map + Change Tracking
- `Hibernate Session` = Unit of Work 구현

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 |
|------|------|
| **원자성** | All or Nothing |
| **성능** | 배치 처리 |
| **일관성** | 데이터 불일치 방지 |
| **중복 제거** | Identity Map |

### ❌ 단점

| 단점 | 설명 |
|------|------|
| **복잡도** | 구현 복잡 |
| **메모리** | 객체 캐싱 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: 너무 큰 Unit of Work

```java
// 잘못된 예
@Transactional
public void processBigBatch() {
    // 😱 10만 건 처리
    for (int i = 0; i < 100000; i++) {
        Order order = new Order();
        orderRepository.save(order);
    }
    
    // 메모리 부족!
}
```

**해결:**
```java
// 올바른 예
public void processBigBatch() {
    int batchSize = 1000;
    
    for (int i = 0; i < 100; i++) {
        processBatch(i * batchSize, batchSize);
    }
}

@Transactional
public void processBatch(int offset, int size) {
    // 1000건씩 처리
}
```

---

## 9. 심화 주제

### 🎯 Dirty Checking

```java
/**
 * JPA의 Dirty Checking
 */
@Transactional
public void updateProduct(Long id) {
    Product product = productRepository.findById(id);
    
    // 수정만 하면 됨
    product.setName("수정된 이름");
    
    // save() 호출 불필요!
    // Transaction 커밋 시 자동으로 UPDATE
}
```

---

## 10. 핵심 정리

### 📌 Unit of Work 체크리스트

```
✅ 변경 추적 (New, Dirty, Removed)
✅ Identity Map (중복 방지)
✅ 한 번에 커밋
✅ 실패 시 롤백
✅ @Transactional 사용
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Service Layer](./03-ServiceLayer.md) | [다음: Specification →](./05-Specification.md)**

</div>
