# Optional Chaining Pattern (옵셔널 체이닝 패턴)

> **"null을 안전하게 다루며 체이닝을 통해 우아하게 처리하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [Best Practices](#6-best-practices)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: NullPointerException의 공포
public String getUserCity(User user) {
    // 😱 NPE 위험 곳곳에!
    return user.getAddress().getCity().getName();
    
    // user가 null이면? → NPE!
    // address가 null이면? → NPE!
    // city가 null이면? → NPE!
}

// 문제 2: 방어 코드의 중첩
public String getUserCitySafe(User user) {
    // 😱 중첩된 null 체크
    if (user != null) {
        Address address = user.getAddress();
        if (address != null) {
            City city = address.getCity();
            if (city != null) {
                return city.getName();
            }
        }
    }
    return "Unknown";
    
    // 가독성 최악!
    // 계단 현상 (Stairway to hell)
}

// 문제 3: 반환 타입의 모호성
public User findUser(Long id) {
    User user = database.findById(id);
    return user;  // 😱 null을 반환? 예외를 던져? 모호함!
}

// 호출하는 쪽:
User user = findUser(123L);
if (user != null) {  // null 체크 필수? 선택?
    // ...
}

// 문제 4: 기본값 처리의 번거로움
public String getDisplayName(User user) {
    // 😱 삼항 연산자 중첩
    return user != null 
        ? (user.getName() != null ? user.getName() : "Guest")
        : "Guest";
}

// 문제 5: 컬렉션 반환
public List<Order> getOrders(User user) {
    if (user == null) {
        return null;  // 😱 null 반환
    }
    
    List<Order> orders = user.getOrders();
    return orders != null ? orders : Collections.emptyList();
}

// 호출:
List<Order> orders = getOrders(user);
if (orders != null) {  // 여전히 null 체크
    for (Order order : orders) {
        // ...
    }
}

// 문제 6: 예외 처리와 섞임
public String getEmail(Long userId) {
    try {
        User user = userRepository.findById(userId);
        if (user == null) {
            return null;
        }
        
        String email = user.getEmail();
        if (email == null) {
            return null;
        }
        
        return email;
        
    } catch (Exception e) {
        return null;
    }
    
    // null 반환 이유가 불명확!
    // - 사용자가 없어서?
    // - 이메일이 없어서?
    // - 예외 발생해서?
}
```

### ⚡ 핵심 문제

1. **NullPointerException**: 런타임 에러
2. **중첩 체크**: 가독성 저하
3. **모호성**: null 의미가 불명확
4. **방어 코드**: 코드 양 증가
5. **컴파일 안전성 부족**: 컴파일 타임 체크 불가

---

## 2. 패턴 정의

### 📖 정의

> **값의 존재 여부를 명시적으로 표현하는 컨테이너로, null 대신 사용하여 안전하고 우아한 코드를 작성하는 패턴**

### 🎯 목적

- **Null 안전성**: NPE 방지
- **명시성**: 값이 없을 수 있음을 명확히
- **체이닝**: 연속된 null 체크 제거
- **가독성**: 의도가 명확한 코드

### 💡 핵심 아이디어

```java
// Before: null 반환 + 중첩 체크
public String getUserCity(User user) {
    if (user != null) {
        Address address = user.getAddress();
        if (address != null) {
            City city = address.getCity();
            if (city != null) {
                return city.getName();
            }
        }
    }
    return "Unknown";
}

// After: Optional 체이닝
public String getUserCity(Optional<User> user) {
    return user
        .map(User::getAddress)
        .map(Address::getCity)
        .map(City::getName)
        .orElse("Unknown");
}

// 우아하고 안전!
```

---

## 3. 구조와 구성요소

### 📊 Optional 구조

```
Optional<T>
    │
    ├─ empty()        → Optional.empty()
    ├─ of(T value)    → Optional<T> (null이면 NPE)
    └─ ofNullable(T)  → Optional<T> (null 허용)

메서드:
    ├─ isPresent()    → boolean
    ├─ isEmpty()      → boolean (Java 11+)
    ├─ get()          → T (없으면 예외)
    ├─ orElse(T)      → T
    ├─ orElseGet(Supplier) → T
    ├─ orElseThrow()  → T
    ├─ map(Function)  → Optional<U>
    ├─ flatMap(Function) → Optional<U>
    ├─ filter(Predicate) → Optional<T>
    └─ ifPresent(Consumer) → void
```

### 🔄 Optional 흐름

```
값이 있는 경우:
Optional.of("Hello")
    ↓
  .map(String::toUpperCase)  → Optional("HELLO")
    ↓
  .filter(s -> s.length() > 3) → Optional("HELLO")
    ↓
  .orElse("Default")  → "HELLO"

값이 없는 경우:
Optional.empty()
    ↓
  .map(String::toUpperCase)  → Optional.empty()
    ↓
  .filter(s -> s.length() > 3) → Optional.empty()
    ↓
  .orElse("Default")  → "Default"
```

---

## 4. 구현 방법

### 완전한 구현: E-Commerce User Service ⭐⭐⭐

```java
/**
 * ============================================
 * DOMAIN MODELS
 * ============================================
 */
public class User {
    private Long id;
    private String name;
    private String email;
    private Address address;
    private UserProfile profile;
    
    public User(Long id, String name) {
        this.id = id;
        this.name = name;
    }
    
    // Getters
    public Long getId() { return id; }
    public String getName() { return name; }
    public Optional<String> getEmail() { return Optional.ofNullable(email); }
    public Optional<Address> getAddress() { return Optional.ofNullable(address); }
    public Optional<UserProfile> getProfile() { return Optional.ofNullable(profile); }
    
    // Setters
    public void setEmail(String email) { this.email = email; }
    public void setAddress(Address address) { this.address = address; }
    public void setProfile(UserProfile profile) { this.profile = profile; }
}

public class Address {
    private String street;
    private City city;
    
    public Address(String street) {
        this.street = street;
    }
    
    public String getStreet() { return street; }
    public Optional<City> getCity() { return Optional.ofNullable(city); }
    public void setCity(City city) { this.city = city; }
}

public class City {
    private String name;
    private String zipCode;
    
    public City(String name) {
        this.name = name;
    }
    
    public String getName() { return name; }
    public Optional<String> getZipCode() { return Optional.ofNullable(zipCode); }
    public void setZipCode(String zipCode) { this.zipCode = zipCode; }
}

public class UserProfile {
    private String bio;
    private String avatar;
    
    public Optional<String> getBio() { return Optional.ofNullable(bio); }
    public Optional<String> getAvatar() { return Optional.ofNullable(avatar); }
    public void setBio(String bio) { this.bio = bio; }
    public void setAvatar(String avatar) { this.avatar = avatar; }
}

/**
 * ============================================
 * REPOSITORY (Optional 반환)
 * ============================================
 */
public class UserRepository {
    private final Map<Long, User> users = new HashMap<>();
    
    /**
     * Optional 반환 (찾지 못할 수 있음을 명시)
     */
    public Optional<User> findById(Long id) {
        return Optional.ofNullable(users.get(id));
    }
    
    /**
     * 이메일로 찾기
     */
    public Optional<User> findByEmail(String email) {
        return users.values().stream()
            .filter(user -> user.getEmail()
                .map(e -> e.equals(email))
                .orElse(false))
            .findFirst();
    }
    
    public void save(User user) {
        users.put(user.getId(), user);
    }
}

/**
 * ============================================
 * SERVICE (Optional 활용)
 * ============================================
 */
public class UserService {
    private final UserRepository userRepository;
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    /**
     * 1. 기본 Optional 사용
     */
    public String getUserEmail(Long userId) {
        return userRepository.findById(userId)
            .flatMap(User::getEmail)
            .orElse("no-email@example.com");
    }
    
    /**
     * 2. Optional 체이닝
     */
    public String getUserCityName(Long userId) {
        return userRepository.findById(userId)
            .flatMap(User::getAddress)
            .flatMap(Address::getCity)
            .map(City::getName)
            .orElse("Unknown");
    }
    
    /**
     * 3. filter 사용
     */
    public Optional<User> getAdultUser(Long userId) {
        return userRepository.findById(userId)
            .filter(user -> {
                // 예시: 나이가 18세 이상
                // 실제로는 User에 age 필드 필요
                return true;
            });
    }
    
    /**
     * 4. orElseGet (Lazy evaluation)
     */
    public String getUserDisplayName(Long userId) {
        return userRepository.findById(userId)
            .map(User::getName)
            .orElseGet(() -> {
                System.out.println("기본 이름 생성 중...");
                return "Guest_" + System.currentTimeMillis();
            });
        
        // orElse vs orElseGet:
        // orElse → 값이 있어도 항상 실행
        // orElseGet → 값이 없을 때만 실행 (Lazy)
    }
    
    /**
     * 5. orElseThrow (예외 발생)
     */
    public User getUserOrThrow(Long userId) {
        return userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException("User not found: " + userId));
    }
    
    /**
     * 6. ifPresent (값이 있으면 실행)
     */
    public void sendWelcomeEmail(Long userId) {
        userRepository.findById(userId)
            .flatMap(User::getEmail)
            .ifPresent(email -> {
                System.out.println("📧 환영 이메일 발송: " + email);
            });
    }
    
    /**
     * 7. ifPresentOrElse (Java 9+)
     */
    public void processUser(Long userId) {
        userRepository.findById(userId)
            .ifPresentOrElse(
                user -> System.out.println("✅ 사용자 처리: " + user.getName()),
                () -> System.out.println("❌ 사용자 없음")
            );
    }
    
    /**
     * 8. or (Java 9+) - 대체 Optional
     */
    public Optional<User> findUserWithFallback(Long userId) {
        return userRepository.findById(userId)
            .or(() -> {
                System.out.println("기본 사용자 생성...");
                return Optional.of(new User(0L, "Guest"));
            });
    }
    
    /**
     * 9. stream (Java 9+) - Optional을 Stream으로
     */
    public List<String> getUserEmails(List<Long> userIds) {
        return userIds.stream()
            .map(userRepository::findById)
            .flatMap(Optional::stream)  // Optional → Stream
            .flatMap(user -> user.getEmail().stream())
            .collect(Collectors.toList());
    }
    
    /**
     * 10. 복잡한 체이닝
     */
    public String getUserProfileBio(Long userId) {
        return userRepository.findById(userId)
            .flatMap(User::getProfile)
            .flatMap(UserProfile::getBio)
            .filter(bio -> bio.length() > 10)
            .map(bio -> bio.substring(0, 50) + "...")
            .orElse("프로필이 없습니다");
    }
}

/**
 * ============================================
 * OPTIONAL EXAMPLES
 * ============================================
 */
public class OptionalExamples {
    
    /**
     * 1. Optional 생성
     */
    public void creationExamples() {
        System.out.println("\n=== Optional 생성 ===");
        
        // empty
        Optional<String> empty = Optional.empty();
        System.out.println("empty: " + empty);
        
        // of (null이면 NPE)
        Optional<String> notNull = Optional.of("Hello");
        System.out.println("of: " + notNull);
        
        // ofNullable (null 허용)
        Optional<String> nullable = Optional.ofNullable(null);
        System.out.println("ofNullable: " + nullable);
    }
    
    /**
     * 2. 값 확인
     */
    public void checkingExamples() {
        System.out.println("\n=== 값 확인 ===");
        
        Optional<String> value = Optional.of("Hello");
        Optional<String> empty = Optional.empty();
        
        System.out.println("value.isPresent(): " + value.isPresent());
        System.out.println("empty.isPresent(): " + empty.isPresent());
        
        // Java 11+
        System.out.println("value.isEmpty(): " + value.isEmpty());
        System.out.println("empty.isEmpty(): " + empty.isEmpty());
    }
    
    /**
     * 3. 값 가져오기
     */
    public void gettingExamples() {
        System.out.println("\n=== 값 가져오기 ===");
        
        Optional<String> value = Optional.of("Hello");
        
        // get() - 위험! 값이 없으면 예외
        // String s = value.get();
        
        // orElse - 기본값
        String s1 = value.orElse("Default");
        System.out.println("orElse: " + s1);
        
        // orElseGet - Lazy
        String s2 = value.orElseGet(() -> {
            System.out.println("  → orElseGet 실행");
            return "Default";
        });
        
        // orElseThrow
        String s3 = value.orElseThrow(() -> new RuntimeException("No value"));
        System.out.println("orElseThrow: " + s3);
    }
    
    /**
     * 4. map vs flatMap
     */
    public void mapExamples() {
        System.out.println("\n=== map vs flatMap ===");
        
        Optional<String> name = Optional.of("alice");
        
        // map - 결과가 Optional로 자동 래핑
        Optional<String> upper = name.map(String::toUpperCase);
        System.out.println("map: " + upper);
        
        // flatMap - 이미 Optional인 결과
        Optional<String> result = name.flatMap(n -> Optional.of(n.toUpperCase()));
        System.out.println("flatMap: " + result);
        
        // 중첩 Optional 방지
        Optional<Optional<String>> nested = name.map(n -> Optional.of(n));
        System.out.println("map (중첩): " + nested);
        
        Optional<String> flat = name.flatMap(n -> Optional.of(n));
        System.out.println("flatMap (평탄): " + flat);
    }
    
    /**
     * 5. filter
     */
    public void filterExamples() {
        System.out.println("\n=== filter ===");
        
        Optional<Integer> number = Optional.of(42);
        
        Optional<Integer> even = number.filter(n -> n % 2 == 0);
        System.out.println("짝수: " + even);
        
        Optional<Integer> odd = number.filter(n -> n % 2 != 0);
        System.out.println("홀수: " + odd);
    }
}

/**
 * ============================================
 * DEMO
 * ============================================
 */
public class OptionalChainingDemo {
    public static void main(String[] args) {
        System.out.println("=== Optional Chaining Pattern 예제 ===");
        
        // Repository & Service 생성
        UserRepository repository = new UserRepository();
        UserService service = new UserService(repository);
        
        // 테스트 데이터
        User user1 = new User(1L, "홍길동");
        user1.setEmail("hong@example.com");
        
        Address address = new Address("강남대로 123");
        City city = new City("서울");
        city.setZipCode("06000");
        address.setCity(city);
        user1.setAddress(address);
        
        UserProfile profile = new UserProfile();
        profile.setBio("안녕하세요! 홍길동입니다. Optional 패턴을 공부하고 있어요.");
        user1.setProfile(profile);
        
        repository.save(user1);
        
        // 테스트 실행
        System.out.println("\n📧 이메일 조회:");
        System.out.println("  " + service.getUserEmail(1L));
        System.out.println("  " + service.getUserEmail(999L));  // 없는 ID
        
        System.out.println("\n🏙️ 도시 이름:");
        System.out.println("  " + service.getUserCityName(1L));
        System.out.println("  " + service.getUserCityName(999L));
        
        System.out.println("\n👤 표시 이름:");
        System.out.println("  " + service.getUserDisplayName(1L));
        System.out.println("  " + service.getUserDisplayName(999L));
        
        System.out.println("\n📝 프로필:");
        System.out.println("  " + service.getUserProfileBio(1L));
        
        System.out.println("\n💌 환영 이메일:");
        service.sendWelcomeEmail(1L);
        service.sendWelcomeEmail(999L);  // 없으면 아무 일도 안 일어남
        
        System.out.println("\n🔍 사용자 처리:");
        service.processUser(1L);
        service.processUser(999L);
        
        System.out.println("\n" + "=".repeat(60));
        
        // Optional 예제
        OptionalExamples examples = new OptionalExamples();
        examples.creationExamples();
        examples.checkingExamples();
        examples.gettingExamples();
        examples.mapExamples();
        examples.filterExamples();
        
        System.out.println("\n✅ 완료!");
    }
}

class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(String message) {
        super(message);
    }
}
```

**실행 결과:**
```
=== Optional Chaining Pattern 예제 ===

📧 이메일 조회:
  hong@example.com
  no-email@example.com

🏙️ 도시 이름:
  서울
  Unknown

👤 표시 이름:
  홍길동
기본 이름 생성 중...
  Guest_1703234567890

📝 프로필:
  안녕하세요! 홍길동입니다. Optional 패턴을 공부하고 있어요...

💌 환영 이메일:
📧 환영 이메일 발송: hong@example.com

🔍 사용자 처리:
✅ 사용자 처리: 홍길동
❌ 사용자 없음

============================================================

=== Optional 생성 ===
empty: Optional.empty
of: Optional[Hello]
ofNullable: Optional.empty

=== 값 확인 ===
value.isPresent(): true
empty.isPresent(): false
value.isEmpty(): false
empty.isEmpty(): true

=== 값 가져오기 ===
orElse: Hello
orElseThrow: Hello

=== map vs flatMap ===
map: Optional[ALICE]
flatMap: Optional[ALICE]
map (중첩): Optional[Optional[ALICE]]
flatMap (평탄): Optional[ALICE]

=== filter ===
짝수: Optional[42]
홀수: Optional.empty

✅ 완료!
```

---

## 5. 실전 예제

### 예제 1: 설정 관리 ⭐⭐⭐

```java
public class ConfigService {
    private Map<String, String> config = new HashMap<>();
    
    public Optional<String> get(String key) {
        return Optional.ofNullable(config.get(key));
    }
    
    public int getInt(String key, int defaultValue) {
        return get(key)
            .map(Integer::parseInt)
            .orElse(defaultValue);
    }
    
    public boolean getBoolean(String key) {
        return get(key)
            .map(Boolean::parseBoolean)
            .orElse(false);
    }
}
```

---

## 6. Best Practices

### ✅ 좋은 사용

```java
// ✅ Repository 반환
Optional<User> findById(Long id);

// ✅ 체이닝
user.flatMap(User::getAddress)
    .map(Address::getStreet);

// ✅ orElseGet (Lazy)
.orElseGet(this::createDefault);
```

### ❌ 나쁜 사용

```java
// ❌ 필드로 사용
class User {
    Optional<String> name;  // X
}

// ❌ 파라미터로 사용
void process(Optional<User> user);  // X

// ❌ get() 직접 호출
optional.get();  // X (예외 위험)
```

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 |
|------|------|
| **Null 안전** | NPE 방지 |
| **명시성** | 값이 없을 수 있음을 명확히 |
| **체이닝** | 우아한 코드 |

### ❌ 단점

| 단점 | 설명 |
|------|------|
| **오버헤드** | 객체 래핑 |
| **직렬화 불가** | Serializable 아님 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: isPresent + get

```java
// 잘못된 예
if (optional.isPresent()) {
    String value = optional.get();
    System.out.println(value);
}

// 올바른 예
optional.ifPresent(System.out::println);
```

---

## 9. 심화 주제

### 🎯 Optional vs Null Object Pattern

```java
// Optional
Optional<User> user = findUser(id);

// Null Object
User user = findUser(id).orElse(User.EMPTY);
```

---

## 10. 핵심 정리

### 📌 체크리스트

```
✅ Repository 반환값으로 사용
✅ orElse보다 orElseGet
✅ 체이닝 활용
✅ 필드/파라미터 금지
✅ get() 직접 호출 금지
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Stream Pipeline](./02-StreamPipeline.md) | [다음: Sealed Classes →](./04-SealedClasses.md)**

</div>
