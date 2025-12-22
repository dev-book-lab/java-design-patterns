# DAO Pattern (Data Access Object 패턴)

> **"데이터 접근 로직을 캡슐화하여 비즈니스 로직과 분리하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [DAO vs Repository](#6-dao-vs-repository)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 비즈니스 로직에 SQL이 섞임
public class UserService {
    private Connection connection;
    
    public User createUser(String email, String name) {
        // 😱 비즈니스 로직에 SQL!
        String sql = "INSERT INTO users (email, name) VALUES (?, ?)";
        
        try {
            PreparedStatement stmt = connection.prepareStatement(sql);
            stmt.setString(1, email);
            stmt.setString(2, name);
            stmt.executeUpdate();
            
            // ResultSet 처리...
            
        } catch (SQLException e) {
            // 😱 SQLException을 Service에서 처리!
            e.printStackTrace();
        }
        
        // 문제점:
        // 1. 비즈니스 로직과 데이터 접근 로직이 섞임
        // 2. SQL 중복 (같은 쿼리를 여러 곳에서)
        // 3. 테스트 어려움 (실제 DB 필요)
        // 4. DB 변경 시 전체 수정
        // 5. SQLException 처리가 Service 책임
    }
}

// 문제 2: 중복된 JDBC 코드
public class OrderService {
    public Order getOrder(Long id) {
        Connection conn = null;
        PreparedStatement stmt = null;
        ResultSet rs = null;
        
        try {
            conn = dataSource.getConnection();
            stmt = conn.prepareStatement("SELECT * FROM orders WHERE id = ?");
            stmt.setLong(1, id);
            rs = stmt.executeQuery();
            
            if (rs.next()) {
                Order order = new Order();
                order.setId(rs.getLong("id"));
                order.setCustomerId(rs.getLong("customer_id"));
                // ... 필드 매핑 코드 반복
                return order;
            }
            
        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            // 😱 리소스 정리 코드도 반복!
            try {
                if (rs != null) rs.close();
                if (stmt != null) stmt.close();
                if (conn != null) conn.close();
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
        
        return null;
    }
    
    public List<Order> getAllOrders() {
        // 😱 위와 거의 똑같은 코드 반복!
        Connection conn = null;
        PreparedStatement stmt = null;
        ResultSet rs = null;
        
        try {
            conn = dataSource.getConnection();
            // ... 반복되는 코드
        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            // ... 반복되는 정리 코드
        }
    }
}

// 문제 3: 테스트 불가능
public class ProductService {
    public void deleteProduct(Long id) {
        // 😱 직접 SQL 실행
        String sql = "DELETE FROM products WHERE id = ?";
        
        try {
            PreparedStatement stmt = connection.prepareStatement(sql);
            stmt.setLong(1, id);
            stmt.executeUpdate();
        } catch (SQLException e) {
            e.printStackTrace();
        }
        
        // 어떻게 테스트?
        // - 실제 DB가 필요
        // - Mock 불가능
        // - 단위 테스트 어려움
    }
}

// 문제 4: DB 변경 시 전체 수정
public class UserService {
    public User findByEmail(String email) {
        // MySQL 쿼리
        String sql = "SELECT * FROM users WHERE email = ?";
        
        // PostgreSQL로 변경 시?
        // → 모든 Service의 SQL 수정!
        // → 수백 개 파일 수정!
    }
}

// 문제 5: 트랜잭션 관리 어려움
public class OrderService {
    public void createOrderWithItems(Order order, List<OrderItem> items) {
        Connection conn = null;
        
        try {
            conn = dataSource.getConnection();
            conn.setAutoCommit(false);  // 😱 수동 트랜잭션
            
            // Order 저장
            PreparedStatement stmt1 = conn.prepareStatement("INSERT INTO orders ...");
            stmt1.executeUpdate();
            
            // OrderItem 저장
            for (OrderItem item : items) {
                PreparedStatement stmt2 = conn.prepareStatement("INSERT INTO order_items ...");
                stmt2.executeUpdate();
            }
            
            conn.commit();  // 😱 수동 커밋
            
        } catch (SQLException e) {
            if (conn != null) {
                try {
                    conn.rollback();  // 😱 수동 롤백
                } catch (SQLException ex) {
                    ex.printStackTrace();
                }
            }
        }
        
        // 복잡하고 오류 발생 쉬움!
    }
}

// 문제 6: 예외 처리 일관성 없음
public class ProductService {
    public Product getProduct(Long id) {
        try {
            // DB 조회
        } catch (SQLException e) {
            // 😱 어떻게 처리?
            e.printStackTrace();  // 그냥 출력?
            throw new RuntimeException(e);  // RuntimeException?
            return null;  // null 반환?
        }
        
        // 예외 처리가 제각각!
    }
}

// 문제 7: 캐싱 어려움
public class UserService {
    public User getUser(Long id) {
        // 😱 매번 DB 조회
        String sql = "SELECT * FROM users WHERE id = ?";
        // ...
        
        // 캐싱을 어디에?
        // - Service에? 너무 복잡
        // - 별도 계층? 관리 어려움
    }
}

// 문제 8: 쿼리 최적화 어려움
public class OrderService {
    public List<Order> getOrdersWithCustomer() {
        // 😱 N+1 문제 발생!
        List<Order> orders = findAllOrders();
        
        for (Order order : orders) {
            // 각 주문마다 Customer 조회
            Customer customer = findCustomer(order.getCustomerId());
            order.setCustomer(customer);
        }
        
        // 1 (orders) + N (customers) = N+1 쿼리!
    }
}
```

### ⚡ 핵심 문제

1. **관심사 혼재**: 비즈니스 로직과 데이터 접근 로직이 섞임
2. **코드 중복**: JDBC 코드가 반복됨
3. **테스트 어려움**: 실제 DB 없이 테스트 불가
4. **DB 종속**: SQL이 여기저기 흩어짐
5. **트랜잭션**: 수동 관리로 복잡함
6. **예외 처리**: SQLException 처리가 제각각
7. **성능**: 캐싱, 최적화 어려움

---

## 2. 패턴 정의

### 📖 정의

> **데이터베이스 접근 로직을 별도의 객체로 캡슐화하여, 비즈니스 로직과 데이터 접근 로직을 분리하는 패턴**

### 🎯 목적

- **관심사 분리**: 비즈니스 로직과 데이터 접근 분리
- **재사용성**: 데이터 접근 코드 재사용
- **테스트 용이**: Mock DAO로 테스트
- **DB 독립성**: DB 변경 시 DAO만 수정

### 💡 핵심 아이디어

```java
// Before: Service에 SQL 직접
public class UserService {
    public User getUser(Long id) {
        // 😱 SQL이 Service에!
        String sql = "SELECT * FROM users WHERE id = ?";
        PreparedStatement stmt = connection.prepareStatement(sql);
        // ...
    }
}

// After: DAO로 분리
// 1. DAO Interface
public interface UserDAO {
    User findById(Long id);
    List<User> findAll();
    void save(User user);
    void update(User user);
    void delete(Long id);
}

// 2. DAO Implementation
public class UserDAOImpl implements UserDAO {
    private DataSource dataSource;
    
    @Override
    public User findById(Long id) {
        // SQL 캡슐화
        String sql = "SELECT * FROM users WHERE id = ?";
        // JDBC 코드...
    }
    
    // 모든 데이터 접근 로직 여기에!
}

// 3. Service는 DAO만 사용
public class UserService {
    private UserDAO userDAO;
    
    public User getUser(Long id) {
        // 😊 SQL 없음! DAO만 사용!
        return userDAO.findById(id);
    }
}
```

---

## 3. 구조와 구성요소

### 📊 DAO 구조

```
┌─────────────────────────────────────┐
│       Service Layer                 │
│   (Business Logic)                  │
│                                     │
│   - UserService                     │
│   - OrderService                    │
└─────────────────────────────────────┘
              │
              │ uses
              ▼
┌─────────────────────────────────────┐
│        DAO Interface                │
│                                     │
│   interface UserDAO {               │
│     User findById(Long id);         │
│     void save(User user);           │
│   }                                 │
└─────────────────────────────────────┘
              △
              │ implements
              │
┌─────────────────────────────────────┐
│      DAO Implementation             │
│   (Data Access Logic)               │
│                                     │
│   class UserDAOImpl {               │
│     - SQL queries                   │
│     - JDBC code                     │
│     - Transaction handling          │
│   }                                 │
└─────────────────────────────────────┘
              │
              │ uses
              ▼
┌─────────────────────────────────────┐
│         Database                    │
│   (MySQL, PostgreSQL, etc.)         │
└─────────────────────────────────────┘
```

### 🔄 데이터 흐름

```
Client
  │
  ▼
┌──────────┐
│ Service  │ → userDAO.findById(1L)
└──────────┘
  │
  ▼
┌──────────┐
│   DAO    │ → SQL: SELECT * FROM users WHERE id = 1
└──────────┘
  │
  ▼
┌──────────┐
│    DB    │ → Result Set
└──────────┘
  │
  ▼
┌──────────┐
│   DAO    │ → ResultSet → User entity
└──────────┘
  │
  ▼
┌──────────┐
│ Service  │ ← User entity
└──────────┘
  │
  ▼
Client ← User
```

### 🔧 구성요소

| 컴포넌트 | 역할 | 책임 |
|---------|------|------|
| **DAO Interface** | 계약 정의 | CRUD 메서드 정의 |
| **DAO Implementation** | 실제 구현 | SQL 실행, ResultSet 매핑 |
| **Entity** | 데이터 객체 | DB 테이블과 매핑 |
| **Service** | 비즈니스 로직 | DAO 사용 |

---

## 4. 구현 방법

### 완전한 구현: User DAO ⭐⭐⭐

```java
/**
 * ============================================
 * ENTITY (도메인 객체)
 * ============================================
 */
public class User {
    private Long id;
    private String email;
    private String name;
    private String password;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    // Constructors
    public User() {}
    
    public User(String email, String name, String password) {
        this.email = email;
        this.name = name;
        this.password = password;
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }
    
    // Getters, Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(LocalDateTime updatedAt) { this.updatedAt = updatedAt; }
}

/**
 * ============================================
 * DAO INTERFACE (계약)
 * ============================================
 */
public interface UserDAO {
    /**
     * 사용자 저장
     */
    User save(User user);
    
    /**
     * ID로 조회
     */
    Optional<User> findById(Long id);
    
    /**
     * 이메일로 조회
     */
    Optional<User> findByEmail(String email);
    
    /**
     * 전체 조회
     */
    List<User> findAll();
    
    /**
     * 수정
     */
    void update(User user);
    
    /**
     * 삭제
     */
    void delete(Long id);
    
    /**
     * 이메일 존재 여부
     */
    boolean existsByEmail(String email);
}

/**
 * ============================================
 * DAO IMPLEMENTATION (JDBC)
 * ============================================
 */
public class UserDAOImpl implements UserDAO {
    private final DataSource dataSource;
    
    public UserDAOImpl(DataSource dataSource) {
        this.dataSource = dataSource;
    }
    
    @Override
    public User save(User user) {
        String sql = "INSERT INTO users (email, name, password, created_at, updated_at) " +
                    "VALUES (?, ?, ?, ?, ?)";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            
            stmt.setString(1, user.getEmail());
            stmt.setString(2, user.getName());
            stmt.setString(3, user.getPassword());
            stmt.setTimestamp(4, Timestamp.valueOf(user.getCreatedAt()));
            stmt.setTimestamp(5, Timestamp.valueOf(user.getUpdatedAt()));
            
            int affected = stmt.executeUpdate();
            
            if (affected == 0) {
                throw new DAOException("사용자 저장 실패");
            }
            
            // Generated Key 조회
            try (ResultSet generatedKeys = stmt.getGeneratedKeys()) {
                if (generatedKeys.next()) {
                    user.setId(generatedKeys.getLong(1));
                }
            }
            
            System.out.println("💾 UserDAO: 저장 완료 - ID=" + user.getId());
            
            return user;
            
        } catch (SQLException e) {
            throw new DAOException("사용자 저장 중 오류", e);
        }
    }
    
    @Override
    public Optional<User> findById(Long id) {
        String sql = "SELECT * FROM users WHERE id = ?";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            stmt.setLong(1, id);
            
            try (ResultSet rs = stmt.executeQuery()) {
                if (rs.next()) {
                    User user = mapResultSetToUser(rs);
                    System.out.println("🔍 UserDAO: 조회 완료 - ID=" + id);
                    return Optional.of(user);
                }
            }
            
            return Optional.empty();
            
        } catch (SQLException e) {
            throw new DAOException("사용자 조회 중 오류", e);
        }
    }
    
    @Override
    public Optional<User> findByEmail(String email) {
        String sql = "SELECT * FROM users WHERE email = ?";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            stmt.setString(1, email);
            
            try (ResultSet rs = stmt.executeQuery()) {
                if (rs.next()) {
                    User user = mapResultSetToUser(rs);
                    System.out.println("🔍 UserDAO: 이메일로 조회 - " + email);
                    return Optional.of(user);
                }
            }
            
            return Optional.empty();
            
        } catch (SQLException e) {
            throw new DAOException("사용자 조회 중 오류", e);
        }
    }
    
    @Override
    public List<User> findAll() {
        String sql = "SELECT * FROM users ORDER BY created_at DESC";
        List<User> users = new ArrayList<>();
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql);
             ResultSet rs = stmt.executeQuery()) {
            
            while (rs.next()) {
                users.add(mapResultSetToUser(rs));
            }
            
            System.out.println("📋 UserDAO: 전체 조회 - " + users.size() + "명");
            
            return users;
            
        } catch (SQLException e) {
            throw new DAOException("사용자 목록 조회 중 오류", e);
        }
    }
    
    @Override
    public void update(User user) {
        String sql = "UPDATE users SET email = ?, name = ?, password = ?, updated_at = ? " +
                    "WHERE id = ?";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            user.setUpdatedAt(LocalDateTime.now());
            
            stmt.setString(1, user.getEmail());
            stmt.setString(2, user.getName());
            stmt.setString(3, user.getPassword());
            stmt.setTimestamp(4, Timestamp.valueOf(user.getUpdatedAt()));
            stmt.setLong(5, user.getId());
            
            int affected = stmt.executeUpdate();
            
            if (affected == 0) {
                throw new DAOException("사용자 수정 실패 - ID=" + user.getId());
            }
            
            System.out.println("✏️ UserDAO: 수정 완료 - ID=" + user.getId());
            
        } catch (SQLException e) {
            throw new DAOException("사용자 수정 중 오류", e);
        }
    }
    
    @Override
    public void delete(Long id) {
        String sql = "DELETE FROM users WHERE id = ?";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            stmt.setLong(1, id);
            
            int affected = stmt.executeUpdate();
            
            if (affected == 0) {
                throw new DAOException("사용자 삭제 실패 - ID=" + id);
            }
            
            System.out.println("🗑️ UserDAO: 삭제 완료 - ID=" + id);
            
        } catch (SQLException e) {
            throw new DAOException("사용자 삭제 중 오류", e);
        }
    }
    
    @Override
    public boolean existsByEmail(String email) {
        String sql = "SELECT COUNT(*) FROM users WHERE email = ?";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            stmt.setString(1, email);
            
            try (ResultSet rs = stmt.executeQuery()) {
                if (rs.next()) {
                    return rs.getInt(1) > 0;
                }
            }
            
            return false;
            
        } catch (SQLException e) {
            throw new DAOException("이메일 존재 확인 중 오류", e);
        }
    }
    
    /**
     * ResultSet → User 매핑 (헬퍼 메서드)
     */
    private User mapResultSetToUser(ResultSet rs) throws SQLException {
        User user = new User();
        user.setId(rs.getLong("id"));
        user.setEmail(rs.getString("email"));
        user.setName(rs.getString("name"));
        user.setPassword(rs.getString("password"));
        user.setCreatedAt(rs.getTimestamp("created_at").toLocalDateTime());
        user.setUpdatedAt(rs.getTimestamp("updated_at").toLocalDateTime());
        return user;
    }
}

/**
 * ============================================
 * DAO EXCEPTION (커스텀 예외)
 * ============================================
 */
public class DAOException extends RuntimeException {
    public DAOException(String message) {
        super(message);
    }
    
    public DAOException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * ============================================
 * SERVICE (비즈니스 로직)
 * ============================================
 * DAO를 사용, SQL 몰라도 됨!
 */
public class UserService {
    private final UserDAO userDAO;
    
    public UserService(UserDAO userDAO) {
        this.userDAO = userDAO;
    }
    
    /**
     * 사용자 생성
     */
    public User createUser(String email, String name, String password) {
        System.out.println("\n👤 UserService: 사용자 생성");
        
        // 중복 체크 (DAO 사용)
        if (userDAO.existsByEmail(email)) {
            throw new IllegalArgumentException("이미 존재하는 이메일입니다");
        }
        
        // User 생성
        User user = new User(email, name, password);
        
        // 저장 (DAO 사용)
        return userDAO.save(user);
    }
    
    /**
     * 사용자 조회
     */
    public User getUser(Long id) {
        System.out.println("\n🔍 UserService: 사용자 조회");
        
        return userDAO.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("사용자를 찾을 수 없습니다"));
    }
    
    /**
     * 프로필 수정
     */
    public void updateProfile(Long id, String name) {
        System.out.println("\n✏️ UserService: 프로필 수정");
        
        User user = getUser(id);
        user.setName(name);
        
        userDAO.update(user);
    }
    
    /**
     * 전체 사용자 조회
     */
    public List<User> getAllUsers() {
        System.out.println("\n📋 UserService: 전체 조회");
        
        return userDAO.findAll();
    }
}

/**
 * ============================================
 * DEMO
 * ============================================
 */
public class DAOPatternDemo {
    public static void main(String[] args) {
        System.out.println("=== DAO Pattern 예제 ===\n");
        
        // 1. DataSource 설정 (H2 인메모리)
        DataSource dataSource = createDataSource();
        
        // 2. 테이블 생성
        createTable(dataSource);
        
        // 3. DAO 생성
        UserDAO userDAO = new UserDAOImpl(dataSource);
        
        // 4. Service 생성
        UserService userService = new UserService(userDAO);
        
        try {
            // 5. 사용자 생성
            User user1 = userService.createUser(
                "user1@example.com",
                "홍길동",
                "password123"
            );
            
            User user2 = userService.createUser(
                "user2@example.com",
                "김철수",
                "password456"
            );
            
            System.out.println("\n" + "=".repeat(50));
            
            // 6. 사용자 조회
            User found = userService.getUser(user1.getId());
            System.out.println("조회된 사용자: " + found.getName());
            
            System.out.println("\n" + "=".repeat(50));
            
            // 7. 프로필 수정
            userService.updateProfile(user1.getId(), "홍길동(수정)");
            
            System.out.println("\n" + "=".repeat(50));
            
            // 8. 전체 조회
            List<User> allUsers = userService.getAllUsers();
            System.out.println("전체 사용자: " + allUsers.size() + "명");
            
        } catch (Exception e) {
            System.err.println("오류 발생: " + e.getMessage());
            e.printStackTrace();
        }
    }
    
    private static DataSource createDataSource() {
        // H2 인메모리 DB 설정
        org.h2.jdbcx.JdbcDataSource ds = new org.h2.jdbcx.JdbcDataSource();
        ds.setURL("jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1");
        ds.setUser("sa");
        ds.setPassword("");
        return ds;
    }
    
    private static void createTable(DataSource dataSource) {
        String sql = """
            CREATE TABLE users (
                id BIGINT AUTO_INCREMENT PRIMARY KEY,
                email VARCHAR(255) NOT NULL UNIQUE,
                name VARCHAR(100) NOT NULL,
                password VARCHAR(255) NOT NULL,
                created_at TIMESTAMP NOT NULL,
                updated_at TIMESTAMP NOT NULL
            )
            """;
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            stmt.executeUpdate();
            System.out.println("📊 테이블 생성 완료\n");
            
        } catch (SQLException e) {
            throw new RuntimeException("테이블 생성 실패", e);
        }
    }
}
```

**실행 결과:**
```
=== DAO Pattern 예제 ===

📊 테이블 생성 완료

👤 UserService: 사용자 생성
💾 UserDAO: 저장 완료 - ID=1

👤 UserService: 사용자 생성
💾 UserDAO: 저장 완료 - ID=2

==================================================

🔍 UserService: 사용자 조회
🔍 UserDAO: 조회 완료 - ID=1
조회된 사용자: 홍길동

==================================================

✏️ UserService: 프로필 수정
🔍 UserDAO: 조회 완료 - ID=1
✏️ UserDAO: 수정 완료 - ID=1

==================================================

📋 UserService: 전체 조회
📋 UserDAO: 전체 조회 - 2명
전체 사용자: 2명
```

---

## 5. 실전 예제

### 예제 1: Spring JdbcTemplate 활용 ⭐⭐⭐

```java
/**
 * Spring JdbcTemplate로 DAO 구현
 */
@Repository
public class UserDAOJdbcTemplate implements UserDAO {
    private final JdbcTemplate jdbcTemplate;
    
    @Autowired
    public UserDAOJdbcTemplate(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
    
    @Override
    public User save(User user) {
        String sql = "INSERT INTO users (email, name, password, created_at, updated_at) " +
                    "VALUES (?, ?, ?, ?, ?)";
        
        KeyHolder keyHolder = new GeneratedKeyHolder();
        
        jdbcTemplate.update(connection -> {
            PreparedStatement ps = connection.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS);
            ps.setString(1, user.getEmail());
            ps.setString(2, user.getName());
            ps.setString(3, user.getPassword());
            ps.setTimestamp(4, Timestamp.valueOf(user.getCreatedAt()));
            ps.setTimestamp(5, Timestamp.valueOf(user.getUpdatedAt()));
            return ps;
        }, keyHolder);
        
        user.setId(keyHolder.getKey().longValue());
        return user;
    }
    
    @Override
    public Optional<User> findById(Long id) {
        String sql = "SELECT * FROM users WHERE id = ?";
        
        try {
            User user = jdbcTemplate.queryForObject(sql, userRowMapper(), id);
            return Optional.of(user);
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }
    
    @Override
    public List<User> findAll() {
        String sql = "SELECT * FROM users ORDER BY created_at DESC";
        return jdbcTemplate.query(sql, userRowMapper());
    }
    
    /**
     * RowMapper (ResultSet → User 변환)
     */
    private RowMapper<User> userRowMapper() {
        return (rs, rowNum) -> {
            User user = new User();
            user.setId(rs.getLong("id"));
            user.setEmail(rs.getString("email"));
            user.setName(rs.getString("name"));
            user.setPassword(rs.getString("password"));
            user.setCreatedAt(rs.getTimestamp("created_at").toLocalDateTime());
            user.setUpdatedAt(rs.getTimestamp("updated_at").toLocalDateTime());
            return user;
        };
    }
}
```

---

## 6. DAO vs Repository

### 📊 비교표

| 특징 | DAO | Repository |
|------|-----|------------|
| **추상화 수준** | 낮음 (테이블 중심) | 높음 (도메인 중심) |
| **메서드 이름** | `insert()`, `select()` | `save()`, `find()` |
| **관점** | 데이터베이스 | 도메인 컬렉션 |
| **사용** | Java EE | DDD, Spring Data |

### 💡 예시

```java
// DAO (데이터베이스 중심)
public interface UserDAO {
    void insert(User user);
    User selectById(Long id);
    void update(User user);
    void delete(Long id);
}

// Repository (도메인 중심)
public interface UserRepository {
    User save(User user);
    User findById(Long id);
    void remove(User user);
}
```

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 |
|------|------|
| **관심사 분리** | 비즈니스 로직과 데이터 접근 분리 |
| **재사용성** | 동일한 CRUD 코드 재사용 |
| **테스트 용이** | Mock DAO로 단위 테스트 |
| **DB 독립성** | DB 변경 시 DAO만 수정 |

### ❌ 단점

| 단점 | 설명 |
|------|------|
| **코드 증가** | Interface + Impl |
| **복잡도** | 간단한 경우 오버헤드 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: Service에 SQL

```java
// 잘못된 예
public class UserService {
    public User getUser(Long id) {
        String sql = "SELECT * FROM users WHERE id = ?";  // ❌
        // ...
    }
}
```

**해결:**
```java
// 올바른 예
public class UserService {
    private UserDAO userDAO;
    
    public User getUser(Long id) {
        return userDAO.findById(id).orElseThrow();  // ✅
    }
}
```

---

## 9. 심화 주제

### 🎯 Spring Data JPA

```java
/**
 * Spring Data JPA로 DAO 대체
 */
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    boolean existsByEmail(String email);
    
    // 메서드 이름으로 쿼리 자동 생성!
}
```

---

## 10. 핵심 정리

### 📌 DAO 체크리스트

```
✅ Interface 정의
✅ SQL 캡슐화
✅ 예외 변환 (SQLException → DAOException)
✅ 리소스 정리
✅ Service는 DAO만 사용
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: DTO](./01-DTO.md) | [다음: Service Layer →](./03-ServiceLayer.md)**

</div>
