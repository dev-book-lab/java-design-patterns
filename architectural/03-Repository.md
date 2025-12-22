# Repository Pattern (리포지토리 패턴)

> **"데이터 접근 로직을 추상화하여 도메인과 데이터 저장소를 분리하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [Repository vs DAO](#6-repository-vs-dao)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 도메인 로직에 데이터 접근 코드가 섞임
public class OrderService {
    public void createOrder(Order order) {
        // 비즈니스 로직
        order.validate();
        order.calculateTotal();
        
        // 😱 데이터 접근 코드가 서비스에!
        try {
            Connection conn = DriverManager.getConnection(DB_URL, USER, PASSWORD);
            
            // 주문 저장
            PreparedStatement orderStmt = conn.prepareStatement(
                "INSERT INTO orders (customer_id, total, status) VALUES (?, ?, ?)",
                Statement.RETURN_GENERATED_KEYS
            );
            orderStmt.setLong(1, order.getCustomerId());
            orderStmt.setBigDecimal(2, order.getTotal());
            orderStmt.setString(3, order.getStatus());
            orderStmt.executeUpdate();
            
            ResultSet rs = orderStmt.getGeneratedKeys();
            if (rs.next()) {
                order.setId(rs.getLong(1));
            }
            
            // 주문 항목 저장
            PreparedStatement itemStmt = conn.prepareStatement(
                "INSERT INTO order_items (order_id, product_id, quantity, price) VALUES (?, ?, ?, ?)"
            );
            for (OrderItem item : order.getItems()) {
                itemStmt.setLong(1, order.getId());
                itemStmt.setLong(2, item.getProductId());
                itemStmt.setInt(3, item.getQuantity());
                itemStmt.setBigDecimal(4, item.getPrice());
                itemStmt.addBatch();
            }
            itemStmt.executeBatch();
            
            conn.commit();
            conn.close();
            
        } catch (SQLException e) {
            throw new RuntimeException("주문 저장 실패", e);
        }
        
        // 서비스가 SQL, JDBC에 종속!
        // 비즈니스 로직과 데이터 접근 로직이 뒤섞임!
    }
}

// 문제 2: 데이터 접근 코드 중복
public class UserService {
    public User findByEmail(String email) {
        // 😱 매번 같은 JDBC 코드 반복
        Connection conn = null;
        try {
            conn = DriverManager.getConnection(DB_URL);
            PreparedStatement stmt = conn.prepareStatement(
                "SELECT * FROM users WHERE email = ?"
            );
            stmt.setString(1, email);
            ResultSet rs = stmt.executeQuery();
            
            if (rs.next()) {
                User user = new User();
                user.setId(rs.getLong("id"));
                user.setName(rs.getString("name"));
                user.setEmail(rs.getString("email"));
                // ...
                return user;
            }
            return null;
            
        } catch (SQLException e) {
            throw new RuntimeException(e);
        } finally {
            if (conn != null) {
                try { conn.close(); } catch (SQLException e) {}
            }
        }
    }
}

public class ProductService {
    public Product findById(Long id) {
        // 😱 또 똑같은 패턴 반복!
        Connection conn = null;
        try {
            conn = DriverManager.getConnection(DB_URL);
            PreparedStatement stmt = conn.prepareStatement(
                "SELECT * FROM products WHERE id = ?"
            );
            // ... 중복 코드
        } catch (SQLException e) {
            // ...
        }
    }
}

// 문제 3: 데이터베이스 변경 시 전체 수정
public class CustomerService {
    public List<Customer> findActiveCustomers() {
        // MySQL 특화 쿼리
        String sql = "SELECT * FROM customers WHERE status = 'ACTIVE' LIMIT 100";
        
        Connection conn = DriverManager.getConnection(
            "jdbc:mysql://localhost:3306/mydb", // MySQL 특화
            USER, PASSWORD
        );
        
        // PostgreSQL로 바꾸면?
        // → 모든 서비스 클래스 수정!
        // → SQL 문법 차이 처리!
        // → 연결 방식 변경!
    }
}

// 문제 4: 테스트 불가능
public class OrderService {
    public void processOrder(Long orderId) {
        // 실제 DB에 의존
        Connection conn = DriverManager.getConnection(DB_URL);
        // ...
        
        // 어떻게 테스트?
        // - DB 없이 테스트 불가능
        // - Mock 불가능 (직접 JDBC 사용)
        // - 통합 테스트만 가능 (느림)
    }
}

// 문제 5: 쿼리 최적화 어려움
public class ReportService {
    public List<SalesReport> generateReport() {
        // N+1 문제 발생
        List<Order> orders = findAllOrders();
        
        for (Order order : orders) {
            // 각 주문마다 개별 쿼리!
            Customer customer = findCustomerById(order.getCustomerId());
            List<OrderItem> items = findOrderItemsByOrderId(order.getId());
            // ...
        }
        
        // 100개 주문 = 201개 쿼리!
        // 쿼리 최적화를 어디서?
    }
}

// 문제 6: 복잡한 쿼리 구성
public class ProductService {
    public List<Product> searchProducts(String keyword, 
                                       String category,
                                       BigDecimal minPrice,
                                       BigDecimal maxPrice,
                                       String sortBy) {
        // 😱 동적 쿼리 구성
        StringBuilder sql = new StringBuilder("SELECT * FROM products WHERE 1=1");
        
        if (keyword != null) {
            sql.append(" AND name LIKE ?");
        }
        if (category != null) {
            sql.append(" AND category = ?");
        }
        if (minPrice != null) {
            sql.append(" AND price >= ?");
        }
        if (maxPrice != null) {
            sql.append(" AND price <= ?");
        }
        if (sortBy != null) {
            sql.append(" ORDER BY ").append(sortBy);
        }
        
        // 복잡하고 오류 발생 쉬움!
        // SQL Injection 위험!
    }
}
```

### ⚡ 핵심 문제

1. **관심사 혼재**: 비즈니스 로직과 데이터 접근 로직이 섞임
2. **중복 코드**: JDBC 코드가 모든 서비스에 반복
3. **강한 결합**: 비즈니스 로직이 DB에 직접 의존
4. **테스트 어려움**: Mock이나 Stub 사용 불가
5. **변경 취약**: DB 변경 시 전체 코드 수정
6. **쿼리 분산**: 쿼리가 코드 전체에 흩어짐

---

## 2. 패턴 정의

### 📖 정의

> **도메인 객체에 접근하기 위한 컬렉션과 유사한 인터페이스를 사용하여 도메인과 데이터 매핑 계층 사이를 중재하는 패턴**

### 🎯 목적

- **추상화**: 데이터 접근 로직을 캡슐화
- **도메인 중심**: 도메인 언어로 데이터 접근
- **테스트 용이**: 인터페이스로 Mock 가능
- **유연성**: 데이터 소스 변경 용이

### 💡 핵심 아이디어

```java
// Before: 직접 데이터 접근
public class OrderService {
    public void createOrder(Order order) {
        Connection conn = getConnection();
        PreparedStatement stmt = conn.prepareStatement(...);
        // SQL, JDBC 코드...
    }
}

// After: Repository로 추상화
public class OrderService {
    private OrderRepository orderRepository;
    
    public void createOrder(Order order) {
        // 컬렉션처럼 사용!
        orderRepository.save(order);
    }
}

public interface OrderRepository {
    // 도메인 언어로 메서드 정의
    Order save(Order order);
    Optional<Order> findById(Long id);
    List<Order> findByCustomerId(Long customerId);
    void delete(Order order);
}
```

---

## 3. 구조와 구성요소

### 📊 Repository 패턴 구조

```
┌─────────────────────────────────────┐
│     Domain Layer (도메인 계층)         │
│                                     │
│  ┌──────────┐      ┌─────────────┐  │
│  │  Order   │      │OrderRepository │ ← 인터페이스만
│  │  (Entity)│      │ (Interface) │  │
│  └──────────┘      └─────────────┘  │
└─────────────────────────────────────┘
                           △
                           │ implements
┌──────────────────────────┼──────────┐
│   Infrastructure Layer   │          │
│                          │          │
│              ┌───────────────────┐  │
│              │JpaOrderRepository │  │ ← 구현체
│              │  (Implementation) │  │
│              └───────────────────┘  │
│                     │               │
│                     ▼               │
│              ┌───────────┐          │
│              │ Database  │          │
│              └───────────┘          │
└─────────────────────────────────────┘
```

### 🔄 컬렉션 지향 인터페이스

```java
// Repository는 컬렉션처럼 동작
public interface OrderRepository {
    // Collection.add()처럼
    Order save(Order order);
    
    // Collection.remove()처럼
    void delete(Order order);
    
    // Collection.contains()처럼
    boolean exists(Long id);
    
    // Collection.stream().filter()처럼
    List<Order> findByCustomerId(Long customerId);
    
    // 전체 조회
    List<Order> findAll();
}

// 사용: 마치 컬렉션처럼!
Order order = new Order();
orderRepository.save(order);           // add
List<Order> orders = orderRepository.findAll();  // getAll
orderRepository.delete(order);         // remove
```

### 🏗️ Repository vs DAO 구조 비교

```
=== DAO (Data Access Object) ===
┌─────────────────────────────┐
│     Application Layer       │
│  ┌───────────────────────┐  │
│  │   OrderService        │  │
│  └───────────────────────┘  │
└─────────────────────────────┘
            │
            ▼
┌─────────────────────────────┐
│     Data Access Layer       │
│  ┌───────────────────────┐  │
│  │   OrderDAO            │  │  ← 데이터베이스 중심
│  │ - insertOrder()       │  │     (SQL 메서드명)
│  │ - updateOrder()       │  │
│  │ - deleteOrder()       │  │
│  │ - selectOrderById()   │  │
│  └───────────────────────┘  │
└─────────────────────────────┘

=== Repository ===
┌─────────────────────────────┐
│     Domain Layer            │
│  ┌───────────────────────┐  │
│  │   Order (Entity)      │  │
│  └───────────────────────┘  │
│  ┌───────────────────────┐  │
│  │ OrderRepository(I/F)  │  │  ← 도메인 중심
│  │ - save()              │  │     (도메인 메서드명)
│  │ - findById()          │  │
│  │ - findByCustomer()    │  │
│  └───────────────────────┘  │
└─────────────────────────────┘
            △
            │ implements
┌─────────────────────────────┐
│   Infrastructure Layer      │
│  ┌───────────────────────┐  │
│  │JpaOrderRepository     │  │
│  └───────────────────────┘  │
└─────────────────────────────┘
```

### 🔧 구성요소

| 컴포넌트 | 역할 | 위치 | 예시 |
|---------|------|------|------|
| **Repository Interface** | 데이터 접근 계약 | Domain Layer | `OrderRepository` |
| **Repository Implementation** | 실제 데이터 접근 | Infrastructure | `JpaOrderRepository` |
| **Entity** | 도메인 객체 | Domain Layer | `Order` |
| **Specification** | 쿼리 조건 (선택) | Domain Layer | `OrderSpecification` |

---

## 4. 구현 방법

### 기본 구현: E-Commerce 주문 시스템 ⭐⭐⭐

```java
/**
 * ============================================
 * DOMAIN LAYER (도메인 계층)
 * ============================================
 */

/**
 * Entity: 주문
 */
public class Order {
    private Long id;
    private Long customerId;
    private List<OrderItem> items;
    private OrderStatus status;
    private BigDecimal totalAmount;
    private LocalDateTime createdAt;
    
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
        OrderItem item = new OrderItem(product.getId(), product.getPrice(), quantity);
        items.add(item);
        calculateTotal();
    }
    
    /**
     * 도메인 로직: 총액 계산
     */
    private void calculateTotal() {
        this.totalAmount = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
    
    /**
     * 도메인 로직: 결제
     */
    public void pay() {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("이미 결제되었습니다");
        }
        this.status = OrderStatus.PAID;
    }
    
    // Getters, Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public Long getCustomerId() { return customerId; }
    public List<OrderItem> getItems() { return items; }
    public OrderStatus getStatus() { return status; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public LocalDateTime getCreatedAt() { return createdAt; }
}

/**
 * Value Object: 주문 항목
 */
public class OrderItem {
    private Long productId;
    private BigDecimal price;
    private int quantity;
    
    public OrderItem(Long productId, BigDecimal price, int quantity) {
        this.productId = productId;
        this.price = price;
        this.quantity = quantity;
    }
    
    public BigDecimal getSubtotal() {
        return price.multiply(BigDecimal.valueOf(quantity));
    }
    
    // Getters
    public Long getProductId() { return productId; }
    public BigDecimal getPrice() { return price; }
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
    
    // Getters
    public Long getId() { return id; }
    public String getName() { return name; }
    public BigDecimal getPrice() { return price; }
    public int getStock() { return stock; }
}

/**
 * ============================================
 * REPOSITORY INTERFACE (도메인 계층)
 * ============================================
 * 도메인 언어로 정의, 구현은 인프라 계층에
 */

/**
 * OrderRepository: 주문 리포지토리 인터페이스
 */
public interface OrderRepository {
    /**
     * 주문 저장 (생성 또는 수정)
     */
    Order save(Order order);
    
    /**
     * ID로 주문 조회
     */
    Optional<Order> findById(Long id);
    
    /**
     * 고객의 모든 주문 조회
     */
    List<Order> findByCustomerId(Long customerId);
    
    /**
     * 상태별 주문 조회
     */
    List<Order> findByStatus(Order.OrderStatus status);
    
    /**
     * 기간별 주문 조회
     */
    List<Order> findByDateRange(LocalDateTime start, LocalDateTime end);
    
    /**
     * 전체 주문 조회
     */
    List<Order> findAll();
    
    /**
     * 주문 삭제
     */
    void delete(Order order);
    
    /**
     * 주문 존재 여부
     */
    boolean exists(Long id);
    
    /**
     * 주문 개수
     */
    long count();
}

/**
 * ProductRepository: 상품 리포지토리 인터페이스
 */
public interface ProductRepository {
    Optional<Product> findById(Long id);
    List<Product> findAll();
    Product save(Product product);
    void delete(Product product);
}

/**
 * ============================================
 * REPOSITORY IMPLEMENTATION (인프라 계층)
 * ============================================
 */

/**
 * InMemoryOrderRepository: 메모리 기반 구현
 * (프로토타입, 테스트용)
 */
public class InMemoryOrderRepository implements OrderRepository {
    private final Map<Long, Order> storage = new ConcurrentHashMap<>();
    private final AtomicLong idGenerator = new AtomicLong(1);
    
    @Override
    public Order save(Order order) {
        if (order.getId() == null) {
            // 신규 저장
            order.setId(idGenerator.getAndIncrement());
            storage.put(order.getId(), order);
            System.out.println("💾 주문 저장: ID=" + order.getId());
        } else {
            // 업데이트
            storage.put(order.getId(), order);
            System.out.println("💾 주문 업데이트: ID=" + order.getId());
        }
        return order;
    }
    
    @Override
    public Optional<Order> findById(Long id) {
        System.out.println("🔍 주문 조회: ID=" + id);
        return Optional.ofNullable(storage.get(id));
    }
    
    @Override
    public List<Order> findByCustomerId(Long customerId) {
        System.out.println("🔍 고객 주문 조회: Customer=" + customerId);
        return storage.values().stream()
            .filter(order -> order.getCustomerId().equals(customerId))
            .collect(Collectors.toList());
    }
    
    @Override
    public List<Order> findByStatus(Order.OrderStatus status) {
        System.out.println("🔍 상태별 주문 조회: Status=" + status);
        return storage.values().stream()
            .filter(order -> order.getStatus() == status)
            .collect(Collectors.toList());
    }
    
    @Override
    public List<Order> findByDateRange(LocalDateTime start, LocalDateTime end) {
        System.out.println("🔍 기간별 주문 조회: " + start + " ~ " + end);
        return storage.values().stream()
            .filter(order -> {
                LocalDateTime created = order.getCreatedAt();
                return !created.isBefore(start) && !created.isAfter(end);
            })
            .collect(Collectors.toList());
    }
    
    @Override
    public List<Order> findAll() {
        System.out.println("🔍 전체 주문 조회");
        return new ArrayList<>(storage.values());
    }
    
    @Override
    public void delete(Order order) {
        storage.remove(order.getId());
        System.out.println("🗑️ 주문 삭제: ID=" + order.getId());
    }
    
    @Override
    public boolean exists(Long id) {
        return storage.containsKey(id);
    }
    
    @Override
    public long count() {
        return storage.size();
    }
}

/**
 * JdbcOrderRepository: JDBC 기반 구현
 */
public class JdbcOrderRepository implements OrderRepository {
    private final DataSource dataSource;
    
    public JdbcOrderRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }
    
    @Override
    public Order save(Order order) {
        if (order.getId() == null) {
            return insert(order);
        } else {
            return update(order);
        }
    }
    
    private Order insert(Order order) {
        String sql = "INSERT INTO orders (customer_id, status, total_amount, created_at) " +
                    "VALUES (?, ?, ?, ?)";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            
            stmt.setLong(1, order.getCustomerId());
            stmt.setString(2, order.getStatus().name());
            stmt.setBigDecimal(3, order.getTotalAmount());
            stmt.setTimestamp(4, Timestamp.valueOf(order.getCreatedAt()));
            
            stmt.executeUpdate();
            
            // 생성된 ID 조회
            try (ResultSet rs = stmt.getGeneratedKeys()) {
                if (rs.next()) {
                    order.setId(rs.getLong(1));
                }
            }
            
            // 주문 항목 저장
            saveOrderItems(conn, order);
            
            System.out.println("💾 주문 저장: ID=" + order.getId());
            return order;
            
        } catch (SQLException e) {
            throw new RepositoryException("주문 저장 실패", e);
        }
    }
    
    private void saveOrderItems(Connection conn, Order order) throws SQLException {
        String sql = "INSERT INTO order_items (order_id, product_id, price, quantity) " +
                    "VALUES (?, ?, ?, ?)";
        
        try (PreparedStatement stmt = conn.prepareStatement(sql)) {
            for (OrderItem item : order.getItems()) {
                stmt.setLong(1, order.getId());
                stmt.setLong(2, item.getProductId());
                stmt.setBigDecimal(3, item.getPrice());
                stmt.setInt(4, item.getQuantity());
                stmt.addBatch();
            }
            stmt.executeBatch();
        }
    }
    
    private Order update(Order order) {
        String sql = "UPDATE orders SET status = ?, total_amount = ? WHERE id = ?";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            stmt.setString(1, order.getStatus().name());
            stmt.setBigDecimal(2, order.getTotalAmount());
            stmt.setLong(3, order.getId());
            
            stmt.executeUpdate();
            
            System.out.println("💾 주문 업데이트: ID=" + order.getId());
            return order;
            
        } catch (SQLException e) {
            throw new RepositoryException("주문 업데이트 실패", e);
        }
    }
    
    @Override
    public Optional<Order> findById(Long id) {
        String sql = "SELECT * FROM orders WHERE id = ?";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            stmt.setLong(1, id);
            
            try (ResultSet rs = stmt.executeQuery()) {
                if (rs.next()) {
                    Order order = mapToOrder(rs);
                    loadOrderItems(conn, order);
                    return Optional.of(order);
                }
            }
            
            return Optional.empty();
            
        } catch (SQLException e) {
            throw new RepositoryException("주문 조회 실패", e);
        }
    }
    
    private Order mapToOrder(ResultSet rs) throws SQLException {
        Long customerId = rs.getLong("customer_id");
        Order order = new Order(customerId);
        order.setId(rs.getLong("id"));
        // status, totalAmount, createdAt 설정...
        return order;
    }
    
    private void loadOrderItems(Connection conn, Order order) throws SQLException {
        String sql = "SELECT * FROM order_items WHERE order_id = ?";
        
        try (PreparedStatement stmt = conn.prepareStatement(sql)) {
            stmt.setLong(1, order.getId());
            
            try (ResultSet rs = stmt.executeQuery()) {
                while (rs.next()) {
                    // OrderItem 생성 및 추가
                }
            }
        }
    }
    
    @Override
    public List<Order> findByCustomerId(Long customerId) {
        String sql = "SELECT * FROM orders WHERE customer_id = ?";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            stmt.setLong(1, customerId);
            
            List<Order> orders = new ArrayList<>();
            try (ResultSet rs = stmt.executeQuery()) {
                while (rs.next()) {
                    Order order = mapToOrder(rs);
                    loadOrderItems(conn, order);
                    orders.add(order);
                }
            }
            
            return orders;
            
        } catch (SQLException e) {
            throw new RepositoryException("고객 주문 조회 실패", e);
        }
    }
    
    @Override
    public List<Order> findByStatus(Order.OrderStatus status) {
        // 구현...
        return null;
    }
    
    @Override
    public List<Order> findByDateRange(LocalDateTime start, LocalDateTime end) {
        // 구현...
        return null;
    }
    
    @Override
    public List<Order> findAll() {
        // 구현...
        return null;
    }
    
    @Override
    public void delete(Order order) {
        // 구현...
    }
    
    @Override
    public boolean exists(Long id) {
        // 구현...
        return false;
    }
    
    @Override
    public long count() {
        // 구현...
        return 0;
    }
}

/**
 * Repository 예외
 */
public class RepositoryException extends RuntimeException {
    public RepositoryException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * ============================================
 * APPLICATION LAYER (응용 계층)
 * ============================================
 */

/**
 * OrderService: 비즈니스 로직
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
     * 주문 생성
     */
    public Order createOrder(Long customerId, Map<Long, Integer> productQuantities) {
        System.out.println("\n🛒 === 주문 생성 ===");
        
        // 1. 도메인 객체 생성
        Order order = new Order(customerId);
        
        // 2. 상품 추가
        for (Map.Entry<Long, Integer> entry : productQuantities.entrySet()) {
            Product product = productRepository.findById(entry.getKey())
                .orElseThrow(() -> new IllegalArgumentException("상품 없음: " + entry.getKey()));
            
            order.addItem(product, entry.getValue());
        }
        
        // 3. Repository를 통해 저장 (컬렉션처럼!)
        Order savedOrder = orderRepository.save(order);
        
        System.out.println("✅ 주문 생성 완료: ID=" + savedOrder.getId() + 
                ", 총액=" + savedOrder.getTotalAmount());
        
        return savedOrder;
    }
    
    /**
     * 주문 결제
     */
    public void payOrder(Long orderId) {
        System.out.println("\n💳 === 주문 결제 ===");
        
        // Repository를 통해 조회 (컬렉션처럼!)
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new IllegalArgumentException("주문 없음: " + orderId));
        
        // 도메인 로직 실행
        order.pay();
        
        // Repository를 통해 저장
        orderRepository.save(order);
        
        System.out.println("✅ 결제 완료");
    }
    
    /**
     * 고객 주문 조회
     */
    public List<Order> getCustomerOrders(Long customerId) {
        System.out.println("\n📋 === 고객 주문 조회 ===");
        
        // Repository를 통해 조회 (도메인 언어!)
        List<Order> orders = orderRepository.findByCustomerId(customerId);
        
        System.out.println("✅ 조회 완료: " + orders.size() + "건");
        
        return orders;
    }
    
    /**
     * 대기 중인 주문 조회
     */
    public List<Order> getPendingOrders() {
        return orderRepository.findByStatus(Order.OrderStatus.PENDING);
    }
}

/**
 * ============================================
 * MAIN APPLICATION
 * ============================================
 */
public class RepositoryExample {
    public static void main(String[] args) {
        System.out.println("=== Repository 패턴 예제 ===\n");
        
        // Repository 구현체 선택 (의존성 주입)
        OrderRepository orderRepository = new InMemoryOrderRepository();
        ProductRepository productRepository = createProductRepository();
        
        // Service 생성
        OrderService orderService = new OrderService(orderRepository, productRepository);
        
        try {
            // 1. 주문 생성
            Map<Long, Integer> items = new HashMap<>();
            items.put(1L, 2);  // 상품1 2개
            items.put(2L, 1);  // 상품2 1개
            
            Order order1 = orderService.createOrder(100L, items);
            
            // 2. 다른 주문 생성
            Map<Long, Integer> items2 = new HashMap<>();
            items2.put(1L, 1);
            
            Order order2 = orderService.createOrder(100L, items2);
            
            // 3. 결제
            orderService.payOrder(order1.getId());
            
            // 4. 고객 주문 조회
            List<Order> customerOrders = orderService.getCustomerOrders(100L);
            
            // 5. 대기 중인 주문 조회
            System.out.println("\n📋 === 대기 주문 조회 ===");
            List<Order> pendingOrders = orderService.getPendingOrders();
            System.out.println("대기 주문: " + pendingOrders.size() + "건");
            
            // 6. Repository 통계
            System.out.println("\n📊 === Repository 통계 ===");
            System.out.println("전체 주문 수: " + orderRepository.count());
            
        } catch (Exception e) {
            System.err.println("❌ 오류: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    private static ProductRepository createProductRepository() {
        // 간단한 InMemory 구현
        return new ProductRepository() {
            private Map<Long, Product> products = Map.of(
                1L, new Product(1L, "노트북", new BigDecimal("1200000"), 10),
                2L, new Product(2L, "마우스", new BigDecimal("30000"), 50)
            );
            
            @Override
            public Optional<Product> findById(Long id) {
                return Optional.ofNullable(products.get(id));
            }
            
            @Override
            public List<Product> findAll() {
                return new ArrayList<>(products.values());
            }
            
            @Override
            public Product save(Product product) {
                return product;
            }
            
            @Override
            public void delete(Product product) {}
        };
    }
}
```

**실행 결과:**
```
=== Repository 패턴 예제 ===

🛒 === 주문 생성 ===
💾 주문 저장: ID=1
✅ 주문 생성 완료: ID=1, 총액=2430000

🛒 === 주문 생성 ===
💾 주문 저장: ID=2
✅ 주문 생성 완료: ID=2, 총액=1200000

💳 === 주문 결제 ===
🔍 주문 조회: ID=1
💾 주문 업데이트: ID=1
✅ 결제 완료

📋 === 고객 주문 조회 ===
🔍 고객 주문 조회: Customer=100
✅ 조회 완료: 2건

📋 === 대기 주문 조회 ===
🔍 상태별 주문 조회: Status=PENDING
대기 주문: 1건

📊 === Repository 통계 ===
전체 주문 수: 2
```

---

## 5. 실전 예제

### 예제 1: Spring Data JPA Repository ⭐⭐⭐

```java
/**
 * ============================================
 * Spring Data JPA 실전 예제
 * ============================================
 */

/**
 * Entity (JPA)
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
    
    @Column(nullable = false)
    private String password;
    
    @Enumerated(EnumType.STRING)
    private UserStatus status;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @Column(name = "last_login_at")
    private LocalDateTime lastLoginAt;
    
    public enum UserStatus {
        ACTIVE, INACTIVE, SUSPENDED
    }
    
    // 도메인 로직
    public void activate() {
        this.status = UserStatus.ACTIVE;
    }
    
    public void suspend() {
        this.status = UserStatus.SUSPENDED;
    }
    
    public void updateLastLogin() {
        this.lastLoginAt = LocalDateTime.now();
    }
    
    // Getters, Setters
}

/**
 * Spring Data JPA Repository
 * - 인터페이스만 정의하면 구현체 자동 생성!
 */
public interface UserRepository extends JpaRepository<User, Long> {
    
    // 1. 메서드 이름으로 쿼리 자동 생성
    Optional<User> findByEmail(String email);
    
    List<User> findByStatus(User.UserStatus status);
    
    List<User> findByNameContaining(String keyword);
    
    List<User> findByCreatedAtBetween(LocalDateTime start, LocalDateTime end);
    
    // 2. @Query로 직접 쿼리 작성 (JPQL)
    @Query("SELECT u FROM User u WHERE u.status = :status ORDER BY u.createdAt DESC")
    List<User> findActiveUsersOrderByCreated(@Param("status") User.UserStatus status);
    
    // 3. Native SQL 사용
    @Query(value = "SELECT * FROM users WHERE DATE(created_at) = CURRENT_DATE", 
           nativeQuery = true)
    List<User> findTodayUsers();
    
    // 4. 커스텀 쿼리 메서드
    @Query("SELECT u FROM User u WHERE u.lastLoginAt < :date")
    List<User> findInactiveUsersSince(@Param("date") LocalDateTime date);
    
    // 5. 수정 쿼리
    @Modifying
    @Query("UPDATE User u SET u.status = :status WHERE u.lastLoginAt < :date")
    int suspendInactiveUsers(@Param("status") User.UserStatus status, 
                            @Param("date") LocalDateTime date);
    
    // 6. 존재 여부 체크
    boolean existsByEmail(String email);
    
    // 7. 카운트
    long countByStatus(User.UserStatus status);
    
    // 8. 삭제
    void deleteByEmail(String email);
    
    // 9. Pageable 지원
    Page<User> findByStatus(User.UserStatus status, Pageable pageable);
    
    // 10. Projection (특정 필드만 조회)
    @Query("SELECT u.email, u.name FROM User u WHERE u.status = :status")
    List<UserSummary> findUserSummaries(@Param("status") User.UserStatus status);
    
    interface UserSummary {
        String getEmail();
        String getName();
    }
}

/**
 * Service에서 사용
 */
@Service
@Transactional
public class UserService {
    private final UserRepository userRepository;
    
    @Autowired
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    /**
     * 사용자 생성
     */
    public User createUser(String email, String name, String password) {
        // 중복 체크
        if (userRepository.existsByEmail(email)) {
            throw new DuplicateEmailException("이미 존재하는 이메일: " + email);
        }
        
        // 도메인 객체 생성
        User user = new User();
        user.setEmail(email);
        user.setName(name);
        user.setPassword(encryptPassword(password));
        user.setStatus(User.UserStatus.ACTIVE);
        user.setCreatedAt(LocalDateTime.now());
        
        // Repository를 통해 저장
        return userRepository.save(user);
    }
    
    /**
     * 로그인
     */
    public User login(String email, String password) {
        User user = userRepository.findByEmail(email)
            .orElseThrow(() -> new UserNotFoundException("사용자 없음: " + email));
        
        if (!verifyPassword(password, user.getPassword())) {
            throw new InvalidPasswordException("잘못된 비밀번호");
        }
        
        // 도메인 로직
        user.updateLastLogin();
        
        return userRepository.save(user);
    }
    
    /**
     * 비활성 사용자 정지
     */
    public int suspendInactiveUsers(int days) {
        LocalDateTime threshold = LocalDateTime.now().minusDays(days);
        
        return userRepository.suspendInactiveUsers(
            User.UserStatus.SUSPENDED,
            threshold
        );
    }
    
    /**
     * 활성 사용자 목록 (페이징)
     */
    @Transactional(readOnly = true)
    public Page<User> getActiveUsers(int page, int size) {
        Pageable pageable = PageRequest.of(page, size, 
            Sort.by("createdAt").descending());
        
        return userRepository.findByStatus(User.UserStatus.ACTIVE, pageable);
    }
    
    /**
     * 사용자 검색
     */
    @Transactional(readOnly = true)
    public List<User> searchUsers(String keyword) {
        return userRepository.findByNameContaining(keyword);
    }
    
    private String encryptPassword(String password) {
        // BCrypt 등 사용
        return password; // 예시
    }
    
    private boolean verifyPassword(String raw, String encrypted) {
        return raw.equals(encrypted); // 예시
    }
}
```

---

### 예제 2: Specification Pattern (동적 쿼리) ⭐⭐⭐

```java
/**
 * ============================================
 * Specification Pattern으로 동적 쿼리 구성
 * ============================================
 */

/**
 * Spring Data JPA Specification 사용
 */
public interface UserRepository extends JpaRepository<User, Long>, 
                                       JpaSpecificationExecutor<User> {
    // JpaSpecificationExecutor 상속으로 Specification 지원
}

/**
 * Specification 빌더
 */
public class UserSpecifications {
    
    /**
     * 이메일로 검색
     */
    public static Specification<User> hasEmail(String email) {
        return (root, query, cb) -> 
            email == null ? null : cb.equal(root.get("email"), email);
    }
    
    /**
     * 이름 포함
     */
    public static Specification<User> nameContains(String keyword) {
        return (root, query, cb) -> 
            keyword == null ? null : cb.like(root.get("name"), "%" + keyword + "%");
    }
    
    /**
     * 상태
     */
    public static Specification<User> hasStatus(User.UserStatus status) {
        return (root, query, cb) -> 
            status == null ? null : cb.equal(root.get("status"), status);
    }
    
    /**
     * 생성일 범위
     */
    public static Specification<User> createdBetween(LocalDateTime start, 
                                                     LocalDateTime end) {
        return (root, query, cb) -> {
            if (start == null && end == null) return null;
            if (start == null) return cb.lessThanOrEqualTo(root.get("createdAt"), end);
            if (end == null) return cb.greaterThanOrEqualTo(root.get("createdAt"), start);
            return cb.between(root.get("createdAt"), start, end);
        };
    }
    
    /**
     * 마지막 로그인 이후
     */
    public static Specification<User> lastLoginAfter(LocalDateTime date) {
        return (root, query, cb) -> 
            date == null ? null : cb.greaterThan(root.get("lastLoginAt"), date);
    }
}

/**
 * Service에서 Specification 조합 사용
 */
@Service
public class UserSearchService {
    private final UserRepository userRepository;
    
    @Autowired
    public UserSearchService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    /**
     * 동적 검색
     */
    public List<User> searchUsers(UserSearchCriteria criteria) {
        // Specification 조합
        Specification<User> spec = Specification
            .where(UserSpecifications.hasEmail(criteria.getEmail()))
            .and(UserSpecifications.nameContains(criteria.getKeyword()))
            .and(UserSpecifications.hasStatus(criteria.getStatus()))
            .and(UserSpecifications.createdBetween(
                criteria.getStartDate(), 
                criteria.getEndDate()
            ))
            .and(UserSpecifications.lastLoginAfter(criteria.getLastLoginAfter()));
        
        // Repository를 통해 조회
        return userRepository.findAll(spec);
    }
    
    /**
     * 페이징 + 정렬
     */
    public Page<User> searchUsersWithPaging(UserSearchCriteria criteria, 
                                            Pageable pageable) {
        Specification<User> spec = buildSpecification(criteria);
        
        return userRepository.findAll(spec, pageable);
    }
    
    private Specification<User> buildSpecification(UserSearchCriteria criteria) {
        return Specification
            .where(UserSpecifications.hasEmail(criteria.getEmail()))
            .and(UserSpecifications.nameContains(criteria.getKeyword()))
            .and(UserSpecifications.hasStatus(criteria.getStatus()));
    }
}

/**
 * 검색 조건 DTO
 */
public class UserSearchCriteria {
    private String email;
    private String keyword;
    private User.UserStatus status;
    private LocalDateTime startDate;
    private LocalDateTime endDate;
    private LocalDateTime lastLoginAfter;
    
    // Getters, Setters
}

/**
 * 사용 예제
 */
public class SpecificationExample {
    public static void main(String[] args) {
        // 검색 조건 구성
        UserSearchCriteria criteria = new UserSearchCriteria();
        criteria.setKeyword("홍길동");
        criteria.setStatus(User.UserStatus.ACTIVE);
        criteria.setStartDate(LocalDateTime.now().minusMonths(1));
        
        // 동적 쿼리 실행
        List<User> users = userSearchService.searchUsers(criteria);
        
        // 생성되는 SQL:
        // SELECT * FROM users 
        // WHERE name LIKE '%홍길동%' 
        //   AND status = 'ACTIVE' 
        //   AND created_at >= ?
    }
}
```

---

## 6. Repository vs DAO

### 🔍 핵심 차이

| 특징 | Repository | DAO |
|------|-----------|-----|
| **초점** | 도메인 중심 | 데이터베이스 중심 |
| **인터페이스** | 컬렉션 지향 | 데이터 접근 메서드 |
| **메서드명** | `save()`, `findById()` | `insert()`, `selectById()` |
| **위치** | Domain Layer | Data Access Layer |
| **추상화 수준** | 높음 (도메인 언어) | 낮음 (SQL 언어) |
| **구현** | 여러 DAO 조합 가능 | 단일 테이블 |

### 📝 메서드 이름 비교

```java
// DAO (데이터베이스 중심)
public interface UserDAO {
    void insertUser(User user);
    void updateUser(User user);
    void deleteUser(Long id);
    User selectUserById(Long id);
    List<User> selectAllUsers();
    List<User> selectUsersByStatus(String status);
}

// Repository (도메인 중심)
public interface UserRepository {
    User save(User user);              // 생성/수정 통합
    void delete(User user);            // 엔티티로 삭제
    Optional<User> findById(Long id);  // Optional 반환
    List<User> findAll();
    List<User> findByStatus(UserStatus status); // 도메인 타입
}
```

### 🎯 Repository가 여러 DAO를 사용하는 예

```java
/**
 * Repository는 도메인 개념
 * 내부에서 여러 DAO를 조합할 수 있음
 */
public class JpaOrderRepository implements OrderRepository {
    private final OrderDAO orderDAO;
    private final OrderItemDAO orderItemDAO;
    private final CustomerDAO customerDAO;
    
    @Override
    public Order save(Order order) {
        // 1. 주문 저장 (OrderDAO)
        orderDAO.insert(order);
        
        // 2. 주문 항목 저장 (OrderItemDAO)
        for (OrderItem item : order.getItems()) {
            orderItemDAO.insert(item);
        }
        
        // 3. 고객 정보 업데이트 (CustomerDAO)
        customerDAO.incrementOrderCount(order.getCustomerId());
        
        return order;
    }
    
    // Repository는 도메인 관점의 단일 인터페이스 제공
    // 내부 구현은 여러 DAO를 조합
}
```

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 | 실무 효과 |
|------|------|-----------|
| **관심사 분리** | 데이터 접근 로직 캡슐화 | 유지보수 용이 |
| **테스트 용이** | Mock Repository 사용 | 단위 테스트 |
| **도메인 중심** | 도메인 언어로 데이터 접근 | 가독성 향상 |
| **교체 가능** | 구현체 교체 쉬움 | DB 변경 용이 |
| **재사용성** | 쿼리 로직 재사용 | 중복 제거 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **추상화 비용** | 인터페이스 계층 추가 | 복잡한 경우만 |
| **과도한 추상화** | 간단한 CRUD도 복잡 | 상황에 맞게 |
| **N+1 문제** | 지연 로딩 시 성능 저하 | Fetch Join |

---

## 8. 안티패턴

### ❌ 안티패턴 1: Generic Repository (만능 리포지토리)

```java
// 잘못된 예: 모든 엔티티에 동일한 메서드
public interface GenericRepository<T, ID> {
    T save(T entity);
    T findById(ID id);
    List<T> findAll();
    void delete(T entity);
}

public interface UserRepository extends GenericRepository<User, Long> {
    // 도메인 특화 메서드가 없음!
}

public interface OrderRepository extends GenericRepository<Order, Long> {
    // 모든 Repository가 똑같음!
}
```

**해결:**
```java
// 올바른 예: 도메인 특화 메서드 정의
public interface UserRepository {
    // 기본 메서드
    User save(User user);
    Optional<User> findById(Long id);
    
    // 도메인 특화 메서드 (중요!)
    Optional<User> findByEmail(String email);
    List<User> findActiveUsers();
    List<User> findByRole(UserRole role);
    int countNewUsersThisMonth();
}

public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(Long id);
    
    // 주문 도메인 특화
    List<Order> findByCustomerId(Long customerId);
    List<Order> findPendingOrders();
    BigDecimal calculateTotalSales(LocalDateTime start, LocalDateTime end);
}
```

### ❌ 안티패턴 2: Repository에 비즈니스 로직

```java
// 잘못된 예: Repository에 비즈니스 로직
public class OrderRepositoryImpl implements OrderRepository {
    
    @Override
    public Order save(Order order) {
        // ❌ 검증 로직
        if (order.getTotal().compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("총액은 0보다 커야 합니다");
        }
        
        // ❌ 비즈니스 로직
        if (order.getItems().size() > 10) {
            order.applyBulkDiscount(); // 대량 구매 할인
        }
        
        // 데이터 저장
        return entityManager.persist(order);
    }
}
```

**해결:**
```java
// 올바른 예: 비즈니스 로직은 Service/Domain에
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    
    public Order createOrder(Order order) {
        // ✅ 비즈니스 로직은 Service에
        order.validate();
        
        if (order.getItems().size() > 10) {
            order.applyBulkDiscount();
        }
        
        // Repository는 단순히 저장만
        return orderRepository.save(order);
    }
}

public class OrderRepositoryImpl implements OrderRepository {
    
    @Override
    public Order save(Order order) {
        // ✅ 데이터 저장만
        return entityManager.persist(order);
    }
}
```

---

## 9. 심화 주제

### 🎯 Unit of Work 패턴과 통합

```java
/**
 * Unit of Work: 트랜잭션 내에서 변경 추적
 */
@Service
@Transactional
public class OrderService {
    private final OrderRepository orderRepository;
    
    public void processOrder(Long orderId) {
        // 1. 조회 (Persistence Context에 로드)
        Order order = orderRepository.findById(orderId)
            .orElseThrow();
        
        // 2. 도메인 로직 (변경 감지)
        order.process();
        order.ship();
        
        // 3. 명시적 save() 불필요!
        // @Transactional 종료 시 자동 flush
        // (Dirty Checking)
    }
}
```

### 🔥 CQRS (Command Query Responsibility Segregation)

```java
/**
 * Command용 Repository
 */
public interface OrderCommandRepository {
    Order save(Order order);
    void delete(Order order);
}

/**
 * Query용 Repository
 */
public interface OrderQueryRepository {
    Optional<Order> findById(Long id);
    List<OrderDTO> findOrderSummaries();
    Page<OrderDTO> searchOrders(SearchCriteria criteria, Pageable pageable);
}

/**
 * Service에서 분리 사용
 */
@Service
public class OrderService {
    private final OrderCommandRepository commandRepo;
    private final OrderQueryRepository queryRepo;
    
    // 명령과 조회 분리
    public void createOrder(Order order) {
        commandRepo.save(order); // Write Model
    }
    
    public List<OrderDTO> getOrders() {
        return queryRepo.findOrderSummaries(); // Read Model
    }
}
```

### 💾 Repository 캐싱

```java
/**
 * Repository에 캐싱 적용
 */
public class CachedUserRepository implements UserRepository {
    private final UserRepository delegate;
    private final Cache<Long, User> cache;
    
    public CachedUserRepository(UserRepository delegate) {
        this.delegate = delegate;
        this.cache = CacheBuilder.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .build();
    }
    
    @Override
    public Optional<User> findById(Long id) {
        // 캐시 확인
        User cached = cache.getIfPresent(id);
        if (cached != null) {
            System.out.println("💨 캐시 히트: " + id);
            return Optional.of(cached);
        }
        
        // DB 조회
        Optional<User> user = delegate.findById(id);
        user.ifPresent(u -> cache.put(id, u));
        
        return user;
    }
    
    @Override
    public User save(User user) {
        User saved = delegate.save(user);
        cache.put(saved.getId(), saved); // 캐시 업데이트
        return saved;
    }
}
```

---

## 10. 핵심 정리

### 📌 Repository 패턴 체크리스트

```
✅ 인터페이스는 Domain Layer에
✅ 구현체는 Infrastructure Layer에
✅ 도메인 언어로 메서드 정의
✅ 컬렉션처럼 사용 (save, findById, delete)
✅ 비즈니스 로직은 포함하지 않음
✅ 테스트 가능하도록 인터페이스로 의존
✅ 도메인 특화 쿼리 메서드 제공
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 도메인 주도 설계 | ⭐⭐⭐ | 필수 패턴 |
| 복잡한 도메인 | ⭐⭐⭐ | 관심사 분리 |
| 테스트 중요 | ⭐⭐⭐ | Mock 가능 |
| 간단한 CRUD | ⭐⭐ | DAO로도 충분 |

### 💡 핵심 원칙

1. **도메인 언어 사용**
2. **컬렉션처럼 사용**
3. **데이터 접근만 담당**
4. **구현 은폐**

### 🔥 Spring Data JPA 활용

```java
// ✅ DO: 메서드 이름으로 쿼리 자동 생성
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    List<User> findByNameContaining(String keyword);
    Page<User> findByStatus(UserStatus status, Pageable pageable);
}

// ✅ DO: @Query로 복잡한 쿼리
@Query("SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = :id")
Optional<User> findByIdWithOrders(@Param("id") Long id);

// ✅ DO: Specification으로 동적 쿼리
Specification<User> spec = Specification
    .where(hasEmail(email))
    .and(nameContains(keyword));
List<User> users = userRepository.findAll(spec);

// ❌ DON'T: Repository에 비즈니스 로직
// ❌ DON'T: 모든 엔티티에 동일한 Generic Repository
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: MVC](./02-MVC.md) | [-> 다음: Hexagonal](./04-Hexagonal.md)**

</div>
