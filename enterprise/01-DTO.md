# DTO Pattern (Data Transfer Object 패턴)

> **"계층 간 데이터 전송을 위한 순수한 데이터 객체를 만들자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [DTO vs Entity vs VO](#6-dto-vs-entity-vs-vo)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: Entity를 직접 노출
@RestController
@RequestMapping("/api/users")
public class UserController {
    @Autowired
    private UserRepository userRepository;
    
    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        // 😱 Entity를 그대로 반환!
        return userRepository.findById(id).orElseThrow();
    }
    
    // 문제점:
    // 1. password 같은 민감 정보 노출
    // 2. JPA 연관관계로 인한 무한 순환 참조
    // 3. Lazy Loading 예외 발생
    // 4. Entity 변경 시 API 스펙 변경
}

// Entity (문제 있음!)
@Entity
@Table(name = "users")
public class User {
    @Id
    private Long id;
    
    private String email;
    
    private String password;  // 😱 비밀번호 노출!
    
    private String name;
    
    @OneToMany(mappedBy = "user")
    private List<Order> orders;  // 😱 무한 순환 참조!
    
    @ManyToOne(fetch = FetchType.LAZY)
    private Company company;  // 😱 LazyInitializationException!
    
    // Getters, Setters
}

// 클라이언트가 받는 JSON (위험!)
{
    "id": 1,
    "email": "user@example.com",
    "password": "hashed_password",  // 😱 비밀번호 노출!
    "name": "홍길동",
    "orders": [
        {
            "id": 100,
            "user": {  // 😱 무한 순환!
                "id": 1,
                "orders": [...]
            }
        }
    ],
    "company": null  // 😱 또는 LazyInitializationException!
}

// 문제 2: 여러 Entity 조합이 필요한 경우
@GetMapping("/dashboard")
public ??? getDashboard() {
    User user = userRepository.findById(1L).orElseThrow();
    List<Order> orders = orderRepository.findByUserId(1L);
    Statistics stats = statisticsService.calculate(1L);
    
    // 😱 이걸 어떻게 반환?
    // Map? 타입 안전성 없음!
    // Entity? 여러 Entity를 하나로 묶을 수 없음!
    
    Map<String, Object> result = new HashMap<>();
    result.put("user", user);
    result.put("orders", orders);
    result.put("stats", stats);
    
    return result;  // 😱 타입 안전성 없음!
}

// 문제 3: 네트워크 오버헤드
public class Product {
    private Long id;
    private String name;
    private String description;  // 5000자
    private byte[] image;  // 5MB 이미지
    private String fullSpecification;  // 10000자
    
    // ... 100개 필드
}

@GetMapping("/products")
public List<Product> getProducts() {
    // 😱 목록 조회인데 모든 데이터 전송!
    // - 5MB 이미지도 전송
    // - 상세 설명도 전송
    // - 목록에서는 id, name만 필요한데!
    
    return productRepository.findAll();
}

// 문제 4: 클라이언트 요구사항과 Entity 불일치
// Entity
public class Order {
    private Long id;
    private Long userId;
    private LocalDateTime createdAt;
    private OrderStatus status;
}

// 클라이언트가 원하는 형식
{
    "orderId": 123,  // ← id가 아닌 orderId
    "customerName": "홍길동",  // ← User 조인 필요
    "orderDate": "2024-01-01",  // ← LocalDateTime을 String으로
    "statusText": "배송중"  // ← Enum을 한글로
}

// 😱 Entity로는 표현 불가!

// 문제 5: 입력 검증
@PostMapping("/users")
public User createUser(@RequestBody User user) {
    // 😱 Entity를 직접 받음!
    
    // 문제점:
    // 1. id를 클라이언트가 설정하면?
    // 2. createdAt을 조작하면?
    // 3. 필수 필드 검증은?
    // 4. @Email, @NotNull 같은 검증을?
    
    return userRepository.save(user);
}

// 문제 6: 여러 요청에 대한 다른 응답
// 회원가입 요청
{
    "email": "user@example.com",
    "password": "1234",
    "name": "홍길동"
}

// 로그인 요청
{
    "email": "user@example.com",
    "password": "1234"
}

// 프로필 수정 요청
{
    "name": "김철수",
    "phone": "010-1234-5678"
}

// 😱 모두 User Entity 사용?
// - 각 요청마다 필요한 필드가 다름!
// - 검증 규칙도 다름!
// - 하나의 Entity로 불가능!

// 문제 7: 성능 문제
public class OrderWithDetails {
    // 주문 정보
    private Long orderId;
    
    // User 테이블 조인
    private String userName;
    private String userEmail;
    
    // Product 테이블 조인
    private String productName;
    private BigDecimal productPrice;
    
    // OrderItem 테이블 조인
    private int quantity;
    
    // 😱 4개 테이블 조인 필요!
    // N+1 문제 발생!
}

@GetMapping("/orders")
public List<OrderWithDetails> getOrders() {
    // 😱 Entity로는 효율적인 조회 불가!
    List<Order> orders = orderRepository.findAll();
    
    // N+1 문제!
    return orders.stream()
        .map(order -> {
            User user = userRepository.findById(order.getUserId());  // +N
            Product product = productRepository.findById(order.getProductId());  // +N
            // ...
        })
        .collect(Collectors.toList());
}
```

### ⚡ 핵심 문제

1. **보안**: Entity를 직접 노출 시 민감 정보 유출
2. **순환 참조**: JPA 연관관계로 무한 순환
3. **성능**: 불필요한 데이터 전송
4. **결합도**: Entity 변경 시 API 영향
5. **표현 불일치**: 클라이언트 요구와 Entity 구조 차이
6. **검증**: 입력 데이터 검증 어려움
7. **N+1 문제**: 비효율적인 쿼리

---

## 2. 패턴 정의

### 📖 정의

> **계층 간 데이터 전송을 위한 순수한 데이터 객체로, Entity와 분리하여 API 요청/응답에 최적화된 구조를 제공하는 패턴**

### 🎯 목적

- **캡슐화**: Entity 내부 구조 숨김
- **보안**: 민감 정보 제외
- **성능**: 필요한 데이터만 전송
- **독립성**: Entity 변경이 API에 영향 안 줌

### 💡 핵심 아이디어

```java
// Before: Entity 직접 사용
@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userRepository.findById(id).orElseThrow();
        // 😱 password, orders 등 모두 노출!
    }
}

// After: DTO 사용
@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        User user = userRepository.findById(id).orElseThrow();
        
        // Entity → DTO 변환
        return UserResponse.from(user);
    }
}

// Response DTO (필요한 정보만!)
public class UserResponse {
    private Long id;
    private String email;
    private String name;
    // password 제외!
    // orders 제외!
    
    public static UserResponse from(User user) {
        UserResponse dto = new UserResponse();
        dto.id = user.getId();
        dto.email = user.getEmail();
        dto.name = user.getName();
        return dto;
    }
}

// 클라이언트가 받는 JSON (안전!)
{
    "id": 1,
    "email": "user@example.com",
    "name": "홍길동"
    // password 없음! ✅
    // orders 없음! ✅
}
```

---

## 3. 구조와 구성요소

### 📊 DTO 구조

```
┌─────────────────────────────────────┐
│      Presentation Layer             │
│      (Controller)                   │
└─────────────────────────────────────┘
              ▲
              │ UserResponse (DTO)
              │
┌─────────────────────────────────────┐
│       Service Layer                 │
│                                     │
│  User entity = repository.find()    │
│  UserResponse dto = from(entity)    │
│  return dto;                        │
└─────────────────────────────────────┘
              │
              │ User (Entity)
              ▼
┌─────────────────────────────────────┐
│      Persistence Layer              │
│      (Repository)                   │
└─────────────────────────────────────┘
```

### 🔄 데이터 흐름

```
Client Request (JSON)
    │
    ▼
┌──────────────┐
│CreateUserDTO │ (Request DTO)
└──────────────┘
    │
    │ Controller
    ▼
┌──────────────┐
│   Service    │ → DTO를 Entity로 변환
└──────────────┘
    │
    ▼
┌──────────────┐
│ User Entity  │ → DB 저장
└──────────────┘
    │
    ▼
┌──────────────┐
│   Service    │ → Entity를 DTO로 변환
└──────────────┘
    │
    ▼
┌──────────────┐
│ UserResponse │ (Response DTO)
└──────────────┘
    │
    │ Controller
    ▼
Client Response (JSON)
```

### 🔧 DTO 종류

| 종류 | 용도 | 예시 |
|------|------|------|
| **Request DTO** | 클라이언트 → 서버 | `CreateUserRequest` |
| **Response DTO** | 서버 → 클라이언트 | `UserResponse` |
| **Internal DTO** | 서비스 간 통신 | `UserSummary` |
| **List DTO** | 목록 조회 | `UserListResponse` |

---

## 4. 구현 방법

### 완전한 구현: E-Commerce User API ⭐⭐⭐

```java
/**
 * ============================================
 * ENTITY (도메인 모델)
 * ============================================
 * DB 테이블과 매핑, 비즈니스 로직 포함
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
    private String password;  // 해시된 비밀번호
    
    @Column(nullable = false)
    private String name;
    
    private String phone;
    
    @Enumerated(EnumType.STRING)
    private UserStatus status;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders = new ArrayList<>();
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "company_id")
    private Company company;
    
    public enum UserStatus {
        ACTIVE, INACTIVE, SUSPENDED
    }
    
    // 비즈니스 로직
    public void activate() {
        this.status = UserStatus.ACTIVE;
    }
    
    public void suspend() {
        this.status = UserStatus.SUSPENDED;
    }
    
    public boolean canOrder() {
        return status == UserStatus.ACTIVE;
    }
    
    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }
    
    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
    
    // Getters, Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getPhone() { return phone; }
    public void setPhone(String phone) { this.phone = phone; }
    public UserStatus getStatus() { return status; }
    public void setStatus(UserStatus status) { this.status = status; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public List<Order> getOrders() { return orders; }
    public Company getCompany() { return company; }
    public void setCompany(Company company) { this.company = company; }
}

/**
 * ============================================
 * REQUEST DTOs (입력용)
 * ============================================
 * 클라이언트 → 서버
 */

/**
 * 회원가입 요청 DTO
 */
public class SignupRequest {
    @NotBlank(message = "이메일은 필수입니다")
    @Email(message = "올바른 이메일 형식이 아닙니다")
    private String email;
    
    @NotBlank(message = "비밀번호는 필수입니다")
    @Size(min = 8, message = "비밀번호는 8자 이상이어야 합니다")
    @Pattern(
        regexp = "^(?=.*[A-Z])(?=.*[a-z])(?=.*\\d).*$",
        message = "비밀번호는 대문자, 소문자, 숫자를 포함해야 합니다"
    )
    private String password;
    
    @NotBlank(message = "이름은 필수입니다")
    @Size(min = 2, max = 50, message = "이름은 2-50자여야 합니다")
    private String name;
    
    @Pattern(regexp = "^01[0-9]-[0-9]{4}-[0-9]{4}$", message = "올바른 휴대폰 형식이 아닙니다")
    private String phone;
    
    // Getters, Setters
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getPhone() { return phone; }
    public void setPhone(String phone) { this.phone = phone; }
    
    /**
     * DTO → Entity 변환
     */
    public User toEntity(PasswordEncoder passwordEncoder) {
        User user = new User();
        user.setEmail(this.email);
        user.setPassword(passwordEncoder.encode(this.password));
        user.setName(this.name);
        user.setPhone(this.phone);
        user.setStatus(User.UserStatus.ACTIVE);
        return user;
    }
}

/**
 * 로그인 요청 DTO
 */
public class LoginRequest {
    @NotBlank(message = "이메일은 필수입니다")
    @Email
    private String email;
    
    @NotBlank(message = "비밀번호는 필수입니다")
    private String password;
    
    // Getters, Setters
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
}

/**
 * 프로필 수정 요청 DTO
 */
public class UpdateProfileRequest {
    @NotBlank(message = "이름은 필수입니다")
    @Size(min = 2, max = 50)
    private String name;
    
    @Pattern(regexp = "^01[0-9]-[0-9]{4}-[0-9]{4}$")
    private String phone;
    
    // Getters, Setters
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getPhone() { return phone; }
    public void setPhone(String phone) { this.phone = phone; }
    
    /**
     * Entity 업데이트
     */
    public void updateEntity(User user) {
        user.setName(this.name);
        user.setPhone(this.phone);
    }
}

/**
 * ============================================
 * RESPONSE DTOs (출력용)
 * ============================================
 * 서버 → 클라이언트
 */

/**
 * 사용자 상세 응답 DTO
 */
public class UserResponse {
    private Long id;
    private String email;
    private String name;
    private String phone;
    private String status;
    private String createdAt;
    
    // password 제외! ✅
    // orders 제외! ✅
    // company 제외! ✅
    
    // Getters, Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getPhone() { return phone; }
    public void setPhone(String phone) { this.phone = phone; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public String getCreatedAt() { return createdAt; }
    public void setCreatedAt(String createdAt) { this.createdAt = createdAt; }
    
    /**
     * Entity → DTO 변환 (정적 팩토리 메서드)
     */
    public static UserResponse from(User user) {
        UserResponse dto = new UserResponse();
        dto.id = user.getId();
        dto.email = user.getEmail();
        dto.name = user.getName();
        dto.phone = user.getPhone();
        dto.status = user.getStatus().name();
        
        // LocalDateTime → String 변환
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        dto.createdAt = user.getCreatedAt().format(formatter);
        
        return dto;
    }
}

/**
 * 사용자 목록 응답 DTO (간략 정보)
 */
public class UserListResponse {
    private Long id;
    private String email;
    private String name;
    private String status;
    
    // 목록에서는 phone, createdAt 제외 (더 간략!)
    
    // Getters, Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    
    public static UserListResponse from(User user) {
        UserListResponse dto = new UserListResponse();
        dto.id = user.getId();
        dto.email = user.getEmail();
        dto.name = user.getName();
        dto.status = user.getStatus().name();
        return dto;
    }
}

/**
 * 로그인 응답 DTO
 */
public class LoginResponse {
    private String token;
    private UserResponse user;
    
    public LoginResponse(String token, UserResponse user) {
        this.token = token;
        this.user = user;
    }
    
    // Getters, Setters
    public String getToken() { return token; }
    public void setToken(String token) { this.token = token; }
    public UserResponse getUser() { return user; }
    public void setUser(UserResponse user) { this.user = user; }
}

/**
 * ============================================
 * SERVICE (비즈니스 로직)
 * ============================================
 */

@Service
public class UserService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    
    @Autowired
    public UserService(UserRepository userRepository, PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }
    
    /**
     * 회원가입
     */
    @Transactional
    public UserResponse signup(SignupRequest request) {
        System.out.println("\n👤 회원가입 시작");
        System.out.println("   이메일: " + request.getEmail());
        
        // 1. 중복 체크
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new DuplicateEmailException("이미 사용 중인 이메일입니다");
        }
        
        // 2. DTO → Entity 변환
        User user = request.toEntity(passwordEncoder);
        
        // 3. 저장
        User savedUser = userRepository.save(user);
        
        System.out.println("   ✅ 회원가입 완료: ID=" + savedUser.getId());
        
        // 4. Entity → DTO 변환
        return UserResponse.from(savedUser);
    }
    
    /**
     * 로그인
     */
    public LoginResponse login(LoginRequest request) {
        System.out.println("\n🔐 로그인 시도: " + request.getEmail());
        
        // 1. 사용자 조회
        User user = userRepository.findByEmail(request.getEmail())
            .orElseThrow(() -> new InvalidCredentialsException("이메일 또는 비밀번호가 일치하지 않습니다"));
        
        // 2. 비밀번호 확인
        if (!passwordEncoder.matches(request.getPassword(), user.getPassword())) {
            throw new InvalidCredentialsException("이메일 또는 비밀번호가 일치하지 않습니다");
        }
        
        // 3. 상태 확인
        if (user.getStatus() != User.UserStatus.ACTIVE) {
            throw new UserNotActiveException("활성화되지 않은 계정입니다");
        }
        
        System.out.println("   ✅ 로그인 성공");
        
        // 4. 토큰 생성 (JWT 등)
        String token = generateToken(user);
        
        // 5. 응답 DTO 생성
        return new LoginResponse(token, UserResponse.from(user));
    }
    
    /**
     * 프로필 조회
     */
    public UserResponse getUserProfile(Long userId) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException("사용자를 찾을 수 없습니다"));
        
        return UserResponse.from(user);
    }
    
    /**
     * 프로필 수정
     */
    @Transactional
    public UserResponse updateProfile(Long userId, UpdateProfileRequest request) {
        System.out.println("\n✏️ 프로필 수정: ID=" + userId);
        
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException("사용자를 찾을 수 없습니다"));
        
        // DTO로 Entity 업데이트
        request.updateEntity(user);
        
        System.out.println("   ✅ 프로필 수정 완료");
        
        return UserResponse.from(user);
    }
    
    /**
     * 사용자 목록 조회
     */
    public List<UserListResponse> getAllUsers() {
        return userRepository.findAll().stream()
            .map(UserListResponse::from)
            .collect(Collectors.toList());
    }
    
    private String generateToken(User user) {
        // JWT 토큰 생성 (간단히 표현)
        return "jwt_token_" + user.getId();
    }
}

/**
 * ============================================
 * CONTROLLER (API)
 * ============================================
 */

@RestController
@RequestMapping("/api/users")
public class UserController {
    private final UserService userService;
    
    @Autowired
    public UserController(UserService userService) {
        this.userService = userService;
    }
    
    /**
     * 회원가입
     */
    @PostMapping("/signup")
    public ResponseEntity<UserResponse> signup(@Valid @RequestBody SignupRequest request) {
        UserResponse response = userService.signup(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
    
    /**
     * 로그인
     */
    @PostMapping("/login")
    public ResponseEntity<LoginResponse> login(@Valid @RequestBody LoginRequest request) {
        LoginResponse response = userService.login(request);
        return ResponseEntity.ok(response);
    }
    
    /**
     * 프로필 조회
     */
    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        UserResponse response = userService.getUserProfile(id);
        return ResponseEntity.ok(response);
    }
    
    /**
     * 프로필 수정
     */
    @PutMapping("/{id}")
    public ResponseEntity<UserResponse> updateProfile(
            @PathVariable Long id,
            @Valid @RequestBody UpdateProfileRequest request) {
        
        UserResponse response = userService.updateProfile(id, request);
        return ResponseEntity.ok(response);
    }
    
    /**
     * 사용자 목록
     */
    @GetMapping
    public ResponseEntity<List<UserListResponse>> getAllUsers() {
        List<UserListResponse> response = userService.getAllUsers();
        return ResponseEntity.ok(response);
    }
}

/**
 * ============================================
 * REPOSITORY
 * ============================================
 */

public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    boolean existsByEmail(String email);
}

/**
 * ============================================
 * EXCEPTIONS
 * ============================================
 */

class DuplicateEmailException extends RuntimeException {
    public DuplicateEmailException(String message) {
        super(message);
    }
}

class InvalidCredentialsException extends RuntimeException {
    public InvalidCredentialsException(String message) {
        super(message);
    }
}

class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(String message) {
        super(message);
    }
}

class UserNotActiveException extends RuntimeException {
    public UserNotActiveException(String message) {
        super(message);
    }
}

/**
 * ============================================
 * DEMO
 * ============================================
 */
public class DTOPatternDemo {
    public static void main(String[] args) {
        System.out.println("=== DTO Pattern 예제 ===\n");
        
        // 1. 회원가입 요청
        SignupRequest signupRequest = new SignupRequest();
        signupRequest.setEmail("user@example.com");
        signupRequest.setPassword("Password123");
        signupRequest.setName("홍길동");
        signupRequest.setPhone("010-1234-5678");
        
        System.out.println("📥 회원가입 요청 JSON:");
        System.out.println("{");
        System.out.println("  \"email\": \"" + signupRequest.getEmail() + "\",");
        System.out.println("  \"password\": \"" + signupRequest.getPassword() + "\",");
        System.out.println("  \"name\": \"" + signupRequest.getName() + "\",");
        System.out.println("  \"phone\": \"" + signupRequest.getPhone() + "\"");
        System.out.println("}");
        
        // 2. 로그인 요청
        LoginRequest loginRequest = new LoginRequest();
        loginRequest.setEmail("user@example.com");
        loginRequest.setPassword("Password123");
        
        System.out.println("\n📥 로그인 요청 JSON:");
        System.out.println("{");
        System.out.println("  \"email\": \"" + loginRequest.getEmail() + "\",");
        System.out.println("  \"password\": \"" + loginRequest.getPassword() + "\"");
        System.out.println("}");
        
        // 3. 응답 예시
        System.out.println("\n📤 사용자 응답 JSON:");
        System.out.println("{");
        System.out.println("  \"id\": 1,");
        System.out.println("  \"email\": \"user@example.com\",");
        System.out.println("  \"name\": \"홍길동\",");
        System.out.println("  \"phone\": \"010-1234-5678\",");
        System.out.println("  \"status\": \"ACTIVE\",");
        System.out.println("  \"createdAt\": \"2024-01-01 10:00:00\"");
        System.out.println("}");
        System.out.println("// password 제외! ✅");
        System.out.println("// orders 제외! ✅");
        System.out.println("// company 제외! ✅");
    }
}
```

**실행 결과:**
```
=== DTO Pattern 예제 ===

📥 회원가입 요청 JSON:
{
  "email": "user@example.com",
  "password": "Password123",
  "name": "홍길동",
  "phone": "010-1234-5678"
}

👤 회원가입 시작
   이메일: user@example.com
   ✅ 회원가입 완료: ID=1

📥 로그인 요청 JSON:
{
  "email": "user@example.com",
  "password": "Password123"
}

🔐 로그인 시도: user@example.com
   ✅ 로그인 성공

📤 사용자 응답 JSON:
{
  "id": 1,
  "email": "user@example.com",
  "name": "홍길동",
  "phone": "010-1234-5678",
  "status": "ACTIVE",
  "createdAt": "2024-01-01 10:00:00"
}
// password 제외! ✅
// orders 제외! ✅
// company 제외! ✅
```

---

## 5. 실전 예제

### 예제 1: 여러 Entity 조합 DTO ⭐⭐⭐

```java
/**
 * 주문 상세 조회 (Order + User + Product 조합)
 */
public class OrderDetailResponse {
    // Order 정보
    private Long orderId;
    private String orderNumber;
    private String orderStatus;
    private BigDecimal totalAmount;
    private String orderDate;
    
    // User 정보 (일부만)
    private Long customerId;
    private String customerName;
    private String customerEmail;
    
    // Product 정보 (일부만)
    private String productName;
    private BigDecimal productPrice;
    private int quantity;
    
    // Getters, Setters
    
    /**
     * Entity 조합 → DTO 변환
     */
    public static OrderDetailResponse from(Order order, User user, Product product) {
        OrderDetailResponse dto = new OrderDetailResponse();
        
        // Order
        dto.orderId = order.getId();
        dto.orderNumber = order.getOrderNumber();
        dto.orderStatus = order.getStatus().name();
        dto.totalAmount = order.getTotalAmount();
        dto.orderDate = order.getCreatedAt().format(DateTimeFormatter.ISO_DATE);
        
        // User
        dto.customerId = user.getId();
        dto.customerName = user.getName();
        dto.customerEmail = user.getEmail();
        
        // Product
        dto.productName = product.getName();
        dto.productPrice = product.getPrice();
        dto.quantity = order.getQuantity();
        
        return dto;
    }
}

/**
 * 효율적인 조회 (JOIN 활용)
 */
@Service
public class OrderService {
    
    public OrderDetailResponse getOrderDetail(Long orderId) {
        // 한 번의 쿼리로 조회 (JOIN)
        Object[] result = orderRepository.findOrderDetailById(orderId);
        
        Order order = (Order) result[0];
        User user = (User) result[1];
        Product product = (Product) result[2];
        
        return OrderDetailResponse.from(order, user, product);
    }
}
```

---

## 6. DTO vs Entity vs VO

### 📊 비교표

| 특징 | Entity | DTO | Value Object |
|------|--------|-----|--------------|
| **목적** | DB 매핑 | 데이터 전송 | 값 표현 |
| **가변성** | 가변 | 가변 | **불변** |
| **비즈니스 로직** | ✅ 포함 | ❌ 없음 | ✅ 포함 |
| **영속성** | ✅ 저장됨 | ❌ 저장 안됨 | ❌ 저장 안됨 |
| **동등성** | ID 기반 | 필드 기반 | **값 기반** |
| **사용 위치** | Domain Layer | API Layer | Domain Layer |

### 💡 예시

```java
// Entity (영속성)
@Entity
public class User {
    @Id
    private Long id;
    private String name;
    
    // 비즈니스 로직
    public void activate() { }
}

// DTO (전송)
public class UserResponse {
    private Long id;
    private String name;
    
    // 로직 없음, 전송만!
}

// Value Object (값)
public class Money {
    private final BigDecimal amount;  // 불변!
    
    public Money(BigDecimal amount) {
        this.amount = amount;
    }
    
    // 값 기반 동등성
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Money)) return false;
        Money other = (Money) o;
        return amount.equals(other.amount);
    }
    
    // 비즈니스 로직
    public Money add(Money other) {
        return new Money(this.amount.add(other.amount));
    }
}
```

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 | 실무 효과 |
|------|------|-----------|
| **보안** | 민감 정보 제외 | 비밀번호 노출 방지 |
| **성능** | 필요한 데이터만 | 네트워크 트래픽 감소 |
| **독립성** | Entity 변경 영향 없음 | API 안정성 |
| **검증** | 요청별 검증 규칙 | 입력 검증 용이 |
| **명확성** | API 명세 명확 | 문서화 용이 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **코드 중복** | 변환 코드 반복 | MapStruct, ModelMapper |
| **클래스 증가** | DTO 클래스 많아짐 | 적절한 패키지 구조 |
| **메모리** | 객체 생성 비용 | 필요한 경우만 사용 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: Entity를 DTO로 사용

```java
// 잘못된 예
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userRepository.findById(id).orElseThrow();  // ❌
}
```

**해결:**
```java
// 올바른 예
@GetMapping("/users/{id}")
public UserResponse getUser(@PathVariable Long id) {
    User user = userRepository.findById(id).orElseThrow();
    return UserResponse.from(user);  // ✅
}
```

---

## 9. 심화 주제

### 🎯 MapStruct 활용

```java
/**
 * MapStruct로 자동 변환
 */
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserResponse toResponse(User user);
    
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "password", expression = "java(passwordEncoder.encode(dto.getPassword()))")
    User toEntity(SignupRequest dto, @Context PasswordEncoder passwordEncoder);
}
```

---

## 10. 핵심 정리

### 📌 DTO 체크리스트

```
✅ Entity와 DTO 분리
✅ 민감 정보 제외
✅ 검증 애노테이션 사용
✅ 정적 팩토리 메서드 제공
✅ 용도별 DTO 분리 (Request/Response)
✅ 불변성 고려
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| REST API | ⭐⭐⭐ | 필수 |
| 계층 간 통신 | ⭐⭐⭐ | 결합도 감소 |
| 내부 서비스 | ⭐⭐ | Entity 사용 가능 |

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[다음: DAO →](./02-DAO.md)**

</div>
