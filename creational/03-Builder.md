# Builder Pattern (빌더 패턴)

> **"복잡한 객체를 단계별로 구성하자"**

[← 이전: Factory Method](./02-FactoryMethod.md) | [목차로 돌아가기](../README.md)

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [장단점](#6-장단점)
7. [안티패턴](#7-안티패턴)
8. [핵심 정리](#8-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 생성자 매개변수가 너무 많음 (Telescoping Constructor)
public class User {
    public User(String username, String email) { }
    
    public User(String username, String email, int age) { }
    
    public User(String username, String email, int age, String phone) { }
    
    public User(String username, String email, int age, String phone, 
                String address) { }
    
    public User(String username, String email, int age, String phone, 
                String address, String city) { }
    
    public User(String username, String email, int age, String phone, 
                String address, String city, String country) { }
    // 생성자가 계속 늘어남... 어떤 생성자를 써야 할지 혼란!
}

// 사용 시 문제
User user = new User("john", "john@email.com", 25, null, null, null, null);
// null이 뭘 의미하는지 알 수 없음!

// 문제 2: 순서를 틀리면 큰일남
User user = new User("john@email.com", "john", 25, "010-1234-5678");
// 첫 번째가 이메일? 이름? 순서가 바뀌면 컴파일 에러도 안 남!

// 문제 3: 선택적 매개변수 처리가 어려움
public class Pizza {
    private int size;           // 필수
    private boolean cheese;     // 선택
    private boolean pepperoni;  // 선택
    private boolean bacon;      // 선택
    
    // 모든 조합을 위한 생성자가 필요... 2^n개!
    public Pizza(int size) { }
    public Pizza(int size, boolean cheese) { }
    public Pizza(int size, boolean cheese, boolean pepperoni) { }
    public Pizza(int size, boolean cheese, boolean pepperoni, boolean bacon) { }
    // 조합이 폭발적으로 증가!
}

// 문제 4: 불변 객체 생성이 어려움
public class Config {
    private String host;
    private int port;
    private int timeout;
    
    // Setter로 구성하면 불변 객체를 만들 수 없음
    public void setHost(String host) { this.host = host; }
    public void setPort(int port) { this.port = port; }
    public void setTimeout(int timeout) { this.timeout = timeout; }
    
    // 중간 상태에서 사용될 위험!
    Config config = new Config();
    config.setHost("localhost");
    // port와 timeout이 설정 안 된 상태에서 사용 가능 (위험!)
    useConfig(config);
}
```

### ⚡ 핵심 문제

1. **Telescoping Constructor**: 생성자가 너무 많음
2. **가독성 저하**: 매개변수 순서와 의미를 알기 어려움
3. **불변성 보장 어려움**: Setter로는 불변 객체 불가
4. **선택적 매개변수**: 필수/선택 구분이 어려움

---

## 2. 패턴 정의

### 📖 정의

> **복잡한 객체의 생성 과정과 표현 방법을 분리하여, 동일한 생성 절차에서 서로 다른 표현 결과를 만들 수 있게 하는 패턴**

### 🎯 목적

- 복잡한 객체를 **단계별로** 구성
- **가독성 높은** 객체 생성 (Fluent API)
- **불변 객체** 생성 지원
- **선택적 매개변수** 우아하게 처리

### 💡 핵심 아이디어

```java
// Before: 생성자 지옥
User user = new User("john", "john@email.com", 25, "010-1234-5678", 
                     "Seoul", "Korea", null, null, true, false);

// After: Builder 패턴
User user = new User.Builder("john", "john@email.com")
    .age(25)
    .phone("010-1234-5678")
    .address("Seoul")
    .country("Korea")
    .build();
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌──────────────────────┐
│      Director        │  ← 생성 절차 정의 (선택사항)
├──────────────────────┤
│ + construct()        │
└──────────────────────┘
           │ uses
           ▼
┌──────────────────────┐
│   Builder(interface) │  ← 생성 단계 정의
├──────────────────────┤
│ + buildPartA()       │
│ + buildPartB()       │
│ + getResult()        │
└──────────────────────┘
           △
           │ implements
┌──────────────────────┐
│  ConcreteBuilder     │  ← 실제 생성 로직
├──────────────────────┤
│ - product: Product   │
│ + buildPartA()       │
│ + buildPartB()       │
│ + getResult()        │
└──────────────────────┘
           │ creates
           ▼
┌──────────────────────┐
│      Product         │  ← 생성될 복잡한 객체
└──────────────────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **Product** | 생성될 복잡한 객체 | `User`, `Pizza` |
| **Builder** | 객체 생성 인터페이스 | `UserBuilder` |
| **ConcreteBuilder** | 실제 생성 로직 구현 | `User.Builder` |
| **Director** | 생성 절차 정의 (선택) | `UserDirector` |

---

## 4. 구현 방법

### 방법 1: 기본 Builder 패턴 ⭐⭐⭐

```java
/**
 * Product: 복잡한 객체
 */
public class User {
    // 필수 매개변수
    private final String username;
    private final String email;
    
    // 선택적 매개변수
    private final int age;
    private final String phone;
    private final String address;
    private final String city;
    private final String country;
    private final boolean newsletter;
    
    // private 생성자: Builder를 통해서만 생성
    private User(Builder builder) {
        this.username = builder.username;
        this.email = builder.email;
        this.age = builder.age;
        this.phone = builder.phone;
        this.address = builder.address;
        this.city = builder.city;
        this.country = builder.country;
        this.newsletter = builder.newsletter;
    }
    
    // Getter만 제공 (불변 객체)
    public String getUsername() { return username; }
    public String getEmail() { return email; }
    public int getAge() { return age; }
    public String getPhone() { return phone; }
    public String getAddress() { return address; }
    public String getCity() { return city; }
    public String getCountry() { return country; }
    public boolean isNewsletter() { return newsletter; }
    
    @Override
    public String toString() {
        return "User{" +
                "username='" + username + '\'' +
                ", email='" + email + '\'' +
                ", age=" + age +
                ", phone='" + phone + '\'' +
                ", address='" + address + '\'' +
                ", city='" + city + '\'' +
                ", country='" + country + '\'' +
                ", newsletter=" + newsletter +
                '}';
    }
    
    /**
     * Builder: static inner class
     */
    public static class Builder {
        // 필수 매개변수
        private final String username;
        private final String email;
        
        // 선택적 매개변수 - 기본값 초기화
        private int age = 0;
        private String phone = "";
        private String address = "";
        private String city = "";
        private String country = "";
        private boolean newsletter = false;
        
        // 필수 매개변수는 생성자로
        public Builder(String username, String email) {
            this.username = username;
            this.email = email;
        }
        
        // 선택적 매개변수는 메서드로 (Fluent API)
        public Builder age(int age) {
            this.age = age;
            return this;  // 메서드 체이닝
        }
        
        public Builder phone(String phone) {
            this.phone = phone;
            return this;
        }
        
        public Builder address(String address) {
            this.address = address;
            return this;
        }
        
        public Builder city(String city) {
            this.city = city;
            return this;
        }
        
        public Builder country(String country) {
            this.country = country;
            return this;
        }
        
        public Builder newsletter(boolean newsletter) {
            this.newsletter = newsletter;
            return this;
        }
        
        // 최종 객체 생성
        public User build() {
            // 유효성 검증
            validateUser();
            return new User(this);
        }
        
        private void validateUser() {
            if (username == null || username.isEmpty()) {
                throw new IllegalStateException("Username is required");
            }
            if (email == null || !email.contains("@")) {
                throw new IllegalStateException("Valid email is required");
            }
            if (age < 0 || age > 150) {
                throw new IllegalStateException("Age must be between 0 and 150");
            }
        }
    }
}

// 사용 예제
public class BasicBuilderExample {
    public static void main(String[] args) {
        // 1. 필수 매개변수만 사용
        User user1 = new User.Builder("john", "john@email.com")
                .build();
        System.out.println(user1);
        
        // 2. 일부 선택적 매개변수 사용
        User user2 = new User.Builder("jane", "jane@email.com")
                .age(25)
                .phone("010-1234-5678")
                .build();
        System.out.println(user2);
        
        // 3. 모든 매개변수 사용
        User user3 = new User.Builder("alice", "alice@email.com")
                .age(30)
                .phone("010-9876-5432")
                .address("123 Main St")
                .city("Seoul")
                .country("Korea")
                .newsletter(true)
                .build();
        System.out.println(user3);
        
        // 4. 순서는 상관없음! (가독성 UP)
        User user4 = new User.Builder("bob", "bob@email.com")
                .newsletter(true)
                .city("Busan")
                .age(28)
                .country("Korea")
                .build();
        System.out.println(user4);
        
        // 5. 유효성 검증
        try {
            User invalid = new User.Builder("", "invalid-email")
                    .age(200)
                    .build();
        } catch (IllegalStateException e) {
            System.out.println("검증 실패: " + e.getMessage());
        }
    }
}
```

**실행 결과:**
```
User{username='john', email='john@email.com', age=0, phone='', address='', city='', country='', newsletter=false}
User{username='jane', email='jane@email.com', age=25, phone='010-1234-5678', address='', city='', country='', newsletter=false}
User{username='alice', email='alice@email.com', age=30, phone='010-9876-5432', address='123 Main St', city='Seoul', country='Korea', newsletter=true}
User{username='bob', email='bob@email.com', age=28, phone='', address='', city='Busan', country='Korea', newsletter=true}
검증 실패: Username is required
```

---

### 방법 2: Lombok @Builder ⭐⭐⭐ (실무)

```java
import lombok.Builder;
import lombok.Getter;
import lombok.ToString;

/**
 * Lombok을 사용하면 Builder 코드를 자동 생성!
 */
@Getter
@ToString
@Builder
public class Product {
    private final String name;
    private final String description;
    private final double price;
    
    @Builder.Default
    private final int stock = 0;
    
    @Builder.Default
    private final boolean available = true;
    
    private final String category;
    private final String brand;
}

// 사용 예제
public class LombokBuilderExample {
    public static void main(String[] args) {
        // Lombok이 자동으로 Builder 생성
        Product product = Product.builder()
                .name("노트북")
                .description("고성능 노트북")
                .price(1500000)
                .stock(10)
                .available(true)
                .category("전자제품")
                .brand("SamsungSamsung")
                .build();
        
        System.out.println(product);
    }
}
```

---

### 방법 3: Director 패턴 활용 ⭐⭐

```java
/**
 * Product: 복잡한 컴퓨터
 */
public class Computer {
    private String cpu;
    private String ram;
    private String storage;
    private String gpu;
    private String motherboard;
    private String powerSupply;
    private String coolingSystem;
    
    private Computer(Builder builder) {
        this.cpu = builder.cpu;
        this.ram = builder.ram;
        this.storage = builder.storage;
        this.gpu = builder.gpu;
        this.motherboard = builder.motherboard;
        this.powerSupply = builder.powerSupply;
        this.coolingSystem = builder.coolingSystem;
    }
    
    @Override
    public String toString() {
        return "Computer{" +
                "cpu='" + cpu + '\'' +
                ", ram='" + ram + '\'' +
                ", storage='" + storage + '\'' +
                ", gpu='" + gpu + '\'' +
                ", motherboard='" + motherboard + '\'' +
                ", powerSupply='" + powerSupply + '\'' +
                ", coolingSystem='" + coolingSystem + '\'' +
                '}';
    }
    
    public static class Builder {
        private String cpu;
        private String ram;
        private String storage;
        private String gpu;
        private String motherboard;
        private String powerSupply;
        private String coolingSystem;
        
        public Builder cpu(String cpu) {
            this.cpu = cpu;
            return this;
        }
        
        public Builder ram(String ram) {
            this.ram = ram;
            return this;
        }
        
        public Builder storage(String storage) {
            this.storage = storage;
            return this;
        }
        
        public Builder gpu(String gpu) {
            this.gpu = gpu;
            return this;
        }
        
        public Builder motherboard(String motherboard) {
            this.motherboard = motherboard;
            return this;
        }
        
        public Builder powerSupply(String powerSupply) {
            this.powerSupply = powerSupply;
            return this;
        }
        
        public Builder coolingSystem(String coolingSystem) {
            this.coolingSystem = coolingSystem;
            return this;
        }
        
        public Computer build() {
            return new Computer(this);
        }
    }
}

/**
 * Director: 사전 정의된 구성으로 생성
 */
public class ComputerDirector {
    
    // 게이밍 PC 구성
    public Computer constructGamingPC() {
        return new Computer.Builder()
                .cpu("Intel i9-13900K")
                .ram("32GB DDR5")
                .storage("2TB NVMe SSD")
                .gpu("RTX 4090")
                .motherboard("ASUS ROG")
                .powerSupply("1000W 80+ Gold")
                .coolingSystem("수냉 쿨러")
                .build();
    }
    
    // 사무용 PC 구성
    public Computer constructOfficePC() {
        return new Computer.Builder()
                .cpu("Intel i5-12400")
                .ram("16GB DDR4")
                .storage("512GB SSD")
                .gpu("내장 그래픽")
                .motherboard("일반 메인보드")
                .powerSupply("500W")
                .coolingSystem("기본 쿨러")
                .build();
    }
    
    // 개발자용 PC 구성
    public Computer constructDeveloperPC() {
        return new Computer.Builder()
                .cpu("AMD Ryzen 9 7950X")
                .ram("64GB DDR5")
                .storage("4TB NVMe SSD")
                .gpu("RTX 4070")
                .motherboard("고급 메인보드")
                .powerSupply("850W 80+ Platinum")
                .coolingSystem("고성능 공랭 쿨러")
                .build();
    }
}

// 사용 예제
public class DirectorExample {
    public static void main(String[] args) {
        ComputerDirector director = new ComputerDirector();
        
        // 1. 게이밍 PC
        Computer gamingPC = director.constructGamingPC();
        System.out.println("=== 게이밍 PC ===");
        System.out.println(gamingPC);
        
        // 2. 사무용 PC
        Computer officePC = director.constructOfficePC();
        System.out.println("\n=== 사무용 PC ===");
        System.out.println(officePC);
        
        // 3. 개발자용 PC
        Computer devPC = director.constructDeveloperPC();
        System.out.println("\n=== 개발자용 PC ===");
        System.out.println(devPC);
        
        // 4. 커스텀 PC (Director 없이 직접 구성)
        Computer customPC = new Computer.Builder()
                .cpu("AMD Ryzen 7")
                .ram("32GB")
                .storage("1TB SSD")
                .gpu("RTX 4060")
                .build();
        System.out.println("\n=== 커스텀 PC ===");
        System.out.println(customPC);
    }
}
```

**실행 결과:**
```
=== 게이밍 PC ===
Computer{cpu='Intel i9-13900K', ram='32GB DDR5', storage='2TB NVMe SSD', gpu='RTX 4090', motherboard='ASUS ROG', powerSupply='1000W 80+ Gold', coolingSystem='수냉 쿨러'}

=== 사무용 PC ===
Computer{cpu='Intel i5-12400', ram='16GB DDR4', storage='512GB SSD', gpu='내장 그래픽', motherboard='일반 메인보드', powerSupply='500W', coolingSystem='기본 쿨러'}

=== 개발자용 PC ===
Computer{cpu='AMD Ryzen 9 7950X', ram='64GB DDR5', storage='4TB NVMe SSD', gpu='RTX 4070', motherboard='고급 메인보드', powerSupply='850W 80+ Platinum', coolingSystem='고성능 공랭 쿨러'}

=== 커스텀 PC ===
Computer{cpu='AMD Ryzen 7', ram='32GB', storage='1TB SSD', gpu='RTX 4060', motherboard='null', powerSupply='null', coolingSystem='null'}
```

---

## 5. 실전 예제

### 예제 1: HTTP 요청 빌더 ⭐⭐⭐

```java
/**
 * HTTP 요청을 Builder로 구성
 */
public class HttpRequest {
    private final String method;
    private final String url;
    private final Map<String, String> headers;
    private final Map<String, String> queryParams;
    private final String body;
    private final int timeout;
    private final boolean followRedirects;
    
    private HttpRequest(Builder builder) {
        this.method = builder.method;
        this.url = builder.url;
        this.headers = builder.headers;
        this.queryParams = builder.queryParams;
        this.body = builder.body;
        this.timeout = builder.timeout;
        this.followRedirects = builder.followRedirects;
    }
    
    public void send() {
        System.out.println("=== HTTP 요청 전송 ===");
        System.out.println("Method: " + method);
        System.out.println("URL: " + buildFullUrl());
        System.out.println("Headers: " + headers);
        System.out.println("Body: " + body);
        System.out.println("Timeout: " + timeout + "ms");
        System.out.println("Follow Redirects: " + followRedirects);
        
        // 실제 HTTP 요청 전송 로직
    }
    
    private String buildFullUrl() {
        if (queryParams.isEmpty()) {
            return url;
        }
        
        StringBuilder fullUrl = new StringBuilder(url);
        fullUrl.append("?");
        
        queryParams.forEach((key, value) -> 
            fullUrl.append(key).append("=").append(value).append("&")
        );
        
        return fullUrl.substring(0, fullUrl.length() - 1);
    }
    
    public static class Builder {
        private String method = "GET";
        private String url;
        private Map<String, String> headers = new HashMap<>();
        private Map<String, String> queryParams = new HashMap<>();
        private String body = "";
        private int timeout = 30000;
        private boolean followRedirects = true;
        
        public Builder(String url) {
            this.url = url;
        }
        
        public Builder method(String method) {
            this.method = method;
            return this;
        }
        
        public Builder header(String key, String value) {
            this.headers.put(key, value);
            return this;
        }
        
        public Builder queryParam(String key, String value) {
            this.queryParams.put(key, value);
            return this;
        }
        
        public Builder body(String body) {
            this.body = body;
            return this;
        }
        
        public Builder timeout(int timeout) {
            this.timeout = timeout;
            return this;
        }
        
        public Builder followRedirects(boolean followRedirects) {
            this.followRedirects = followRedirects;
            return this;
        }
        
        // 편의 메서드
        public Builder get() {
            return method("GET");
        }
        
        public Builder post() {
            return method("POST");
        }
        
        public Builder put() {
            return method("PUT");
        }
        
        public Builder delete() {
            return method("DELETE");
        }
        
        public Builder jsonBody(String json) {
            return header("Content-Type", "application/json")
                   .body(json);
        }
        
        public Builder basicAuth(String username, String password) {
            String credentials = username + ":" + password;
            String encoded = Base64.getEncoder()
                    .encodeToString(credentials.getBytes());
            return header("Authorization", "Basic " + encoded);
        }
        
        public HttpRequest build() {
            if (url == null || url.isEmpty()) {
                throw new IllegalStateException("URL is required");
            }
            return new HttpRequest(this);
        }
    }
}

// 사용 예제
public class HttpRequestExample {
    public static void main(String[] args) {
        // 1. GET 요청
        HttpRequest getRequest = new HttpRequest.Builder("https://api.example.com/users")
                .get()
                .queryParam("page", "1")
                .queryParam("size", "10")
                .header("Accept", "application/json")
                .timeout(5000)
                .build();
        getRequest.send();
        
        // 2. POST 요청
        String jsonBody = "{\"name\":\"John\",\"email\":\"john@example.com\"}";
        HttpRequest postRequest = new HttpRequest.Builder("https://api.example.com/users")
                .post()
                .jsonBody(jsonBody)
                .basicAuth("admin", "password123")
                .build();
        postRequest.send();
        
        // 3. PUT 요청
        HttpRequest putRequest = new HttpRequest.Builder("https://api.example.com/users/123")
                .put()
                .header("Content-Type", "application/json")
                .body("{\"name\":\"Jane\"}")
                .followRedirects(false)
                .build();
        putRequest.send();
    }
}
```

**실행 결과:**
```
=== HTTP 요청 전송 ===
Method: GET
URL: https://api.example.com/users?page=1&size=10
Headers: {Accept=application/json}
Body: 
Timeout: 5000ms
Follow Redirects: true

=== HTTP 요청 전송 ===
Method: POST
URL: https://api.example.com/users
Headers: {Content-Type=application/json, Authorization=Basic YWRtaW46cGFzc3dvcmQxMjM=}
Body: {"name":"John","email":"john@example.com"}
Timeout: 30000ms
Follow Redirects: true
...
```

---

### 예제 2: SQL Query Builder ⭐⭐⭐

```java
/**
 * SQL 쿼리를 Builder로 구성
 */
public class SqlQuery {
    private final String table;
    private final List<String> columns;
    private final List<String> whereConditions;
    private final String orderBy;
    private final String groupBy;
    private final Integer limit;
    private final Integer offset;
    private final List<String> joins;
    
    private SqlQuery(Builder builder) {
        this.table = builder.table;
        this.columns = builder.columns;
        this.whereConditions = builder.whereConditions;
        this.orderBy = builder.orderBy;
        this.groupBy = builder.groupBy;
        this.limit = builder.limit;
        this.offset = builder.offset;
        this.joins = builder.joins;
    }
    
    public String toSql() {
        StringBuilder sql = new StringBuilder("SELECT ");
        
        // SELECT 절
        if (columns.isEmpty()) {
            sql.append("*");
        } else {
            sql.append(String.join(", ", columns));
        }
        
        // FROM 절
        sql.append(" FROM ").append(table);
        
        // JOIN 절
        for (String join : joins) {
            sql.append(" ").append(join);
        }
        
        // WHERE 절
        if (!whereConditions.isEmpty()) {
            sql.append(" WHERE ");
            sql.append(String.join(" AND ", whereConditions));
        }
        
        // GROUP BY 절
        if (groupBy != null) {
            sql.append(" GROUP BY ").append(groupBy);
        }
        
        // ORDER BY 절
        if (orderBy != null) {
            sql.append(" ORDER BY ").append(orderBy);
        }
        
        // LIMIT 절
        if (limit != null) {
            sql.append(" LIMIT ").append(limit);
        }
        
        // OFFSET 절
        if (offset != null) {
            sql.append(" OFFSET ").append(offset);
        }
        
        return sql.toString();
    }
    
    public static class Builder {
        private String table;
        private List<String> columns = new ArrayList<>();
        private List<String> whereConditions = new ArrayList<>();
        private String orderBy;
        private String groupBy;
        private Integer limit;
        private Integer offset;
        private List<String> joins = new ArrayList<>();
        
        public Builder from(String table) {
            this.table = table;
            return this;
        }
        
        public Builder select(String... columns) {
            this.columns.addAll(Arrays.asList(columns));
            return this;
        }
        
        public Builder where(String condition) {
            this.whereConditions.add(condition);
            return this;
        }
        
        public Builder whereEqual(String column, Object value) {
            if (value instanceof String) {
                this.whereConditions.add(column + " = '" + value + "'");
            } else {
                this.whereConditions.add(column + " = " + value);
            }
            return this;
        }
        
        public Builder whereLike(String column, String pattern) {
            this.whereConditions.add(column + " LIKE '%" + pattern + "%'");
            return this;
        }
        
        public Builder whereIn(String column, Object... values) {
            String valuesList = Arrays.stream(values)
                    .map(v -> v instanceof String ? "'" + v + "'" : v.toString())
                    .collect(Collectors.joining(", "));
            this.whereConditions.add(column + " IN (" + valuesList + ")");
            return this;
        }
        
        public Builder join(String table, String condition) {
            this.joins.add("JOIN " + table + " ON " + condition);
            return this;
        }
        
        public Builder leftJoin(String table, String condition) {
            this.joins.add("LEFT JOIN " + table + " ON " + condition);
            return this;
        }
        
        public Builder orderBy(String orderBy) {
            this.orderBy = orderBy;
            return this;
        }
        
        public Builder groupBy(String groupBy) {
            this.groupBy = groupBy;
            return this;
        }
        
        public Builder limit(int limit) {
            this.limit = limit;
            return this;
        }
        
        public Builder offset(int offset) {
            this.offset = offset;
            return this;
        }
        
        public SqlQuery build() {
            if (table == null || table.isEmpty()) {
                throw new IllegalStateException("Table name is required");
            }
            return new SqlQuery(this);
        }
    }
}

// 사용 예제
public class SqlQueryExample {
    public static void main(String[] args) {
        // 1. 기본 SELECT
        SqlQuery query1 = new SqlQuery.Builder()
                .select("id", "name", "email")
                .from("users")
                .build();
        System.out.println("Query 1:");
        System.out.println(query1.toSql());
        
        // 2. WHERE 조건
        SqlQuery query2 = new SqlQuery.Builder()
                .select("*")
                .from("users")
                .whereEqual("age", 25)
                .whereEqual("city", "Seoul")
                .build();
        System.out.println("\nQuery 2:");
        System.out.println(query2.toSql());
        
        // 3. JOIN과 ORDER BY
        SqlQuery query3 = new SqlQuery.Builder()
                .select("u.name", "o.order_id", "o.total")
                .from("users u")
                .join("orders o", "u.id = o.user_id")
                .whereEqual("u.active", true)
                .orderBy("o.created_at DESC")
                .limit(10)
                .build();
        System.out.println("\nQuery 3:");
        System.out.println(query3.toSql());
        
        // 4. 복잡한 쿼리
        SqlQuery query4 = new SqlQuery.Builder()
                .select("category", "COUNT(*) as count", "AVG(price) as avg_price")
                .from("products")
                .whereLike("name", "phone")
                .whereIn("status", "active", "pending")
                .groupBy("category")
                .orderBy("count DESC")
                .limit(5)
                .offset(10)
                .build();
        System.out.println("\nQuery 4:");
        System.out.println(query4.toSql());
    }
}
```

**실행 결과:**
```
Query 1:
SELECT id, name, email FROM users

Query 2:
SELECT * FROM users WHERE age = 25 AND city = 'Seoul'

Query 3:
SELECT u.name, o.order_id, o.total FROM users u JOIN orders o ON u.id = o.user_id WHERE u.active = true ORDER BY o.created_at DESC LIMIT 10

Query 4:
SELECT category, COUNT(*) as count, AVG(price) as avg_price FROM products WHERE name LIKE '%phone%' AND status IN ('active', 'pending') GROUP BY category ORDER BY count DESC LIMIT 5 OFFSET 10
```

---

### 예제 3: Pizza 주문 시스템 ⭐⭐ (고전 예제)

```java
/**
 * 피자 주문 (GoF 디자인 패턴 책의 예제)
 */
public class Pizza {
    // 필수
    private final int size;  // 인치
    
    // 선택적
    private final boolean cheese;
    private final boolean pepperoni;
    private final boolean bacon;
    private final boolean mushrooms;
    private final boolean onions;
    private final boolean olives;
    
    private Pizza(Builder builder) {
        this.size = builder.size;
        this.cheese = builder.cheese;
        this.pepperoni = builder.pepperoni;
        this.bacon = builder.bacon;
        this.mushrooms = builder.mushrooms;
        this.onions = builder.onions;
        this.olives = builder.olives;
    }
    
    @Override
    public String toString() {
        StringBuilder description = new StringBuilder();
        description.append(size).append("인치 피자");
        
        List<String> toppings = new ArrayList<>();
        if (cheese) toppings.add("치즈");
        if (pepperoni) toppings.add("페퍼로니");
        if (bacon) toppings.add("베이컨");
        if (mushrooms) toppings.add("버섯");
        if (onions) toppings.add("양파");
        if (olives) toppings.add("올리브");
        
        if (!toppings.isEmpty()) {
            description.append(" with ").append(String.join(", ", toppings));
        }
        
        return description.toString();
    }
    
    public static class Builder {
        private final int size;
        
        private boolean cheese = false;
        private boolean pepperoni = false;
        private boolean bacon = false;
        private boolean mushrooms = false;
        private boolean onions = false;
        private boolean olives = false;
        
        public Builder(int size) {
            this.size = size;
        }
        
        public Builder cheese() {
            this.cheese = true;
            return this;
        }
        
        public Builder pepperoni() {
            this.pepperoni = true;
            return this;
        }
        
        public Builder bacon() {
            this.bacon = true;
            return this;
        }
        
        public Builder mushrooms() {
            this.mushrooms = true;
            return this;
        }
        
        public Builder onions() {
            this.onions = true;
            return this;
        }
        
        public Builder olives() {
            this.olives = true;
            return this;
        }
        
        public Pizza build() {
            return new Pizza(this);
        }
    }
}

// 사용 예제
public class PizzaExample {
    public static void main(String[] args) {
        // 1. 치즈 피자
        Pizza cheesePizza = new Pizza.Builder(12)
                .cheese()
                .build();
        System.out.println("주문 1: " + cheesePizza);
        
        // 2. 페퍼로니 피자
        Pizza pepperoniPizza = new Pizza.Builder(14)
                .cheese()
                .pepperoni()
                .build();
        System.out.println("주문 2: " + pepperoniPizza);
        
        // 3. 디럭스 피자 (모든 토핑)
        Pizza deluxePizza = new Pizza.Builder(16)
                .cheese()
                .pepperoni()
                .bacon()
                .mushrooms()
                .onions()
                .olives()
                .build();
        System.out.println("주문 3: " + deluxePizza);
        
        // 4. 베지테리안 피자
        Pizza veggiePizza = new Pizza.Builder(12)
                .cheese()
                .mushrooms()
                .onions()
                .olives()
                .build();
        System.out.println("주문 4: " + veggiePizza);
    }
}
```

**실행 결과:**
```
주문 1: 12인치 피자 with 치즈
주문 2: 14인치 피자 with 치즈, 페퍼로니
주문 3: 16인치 피자 with 치즈, 페퍼로니, 베이컨, 버섯, 양파, 올리브
주문 4: 12인치 피자 with 치즈, 버섯, 양파, 올리브
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **가독성** | 메서드 이름으로 의미 명확 | `.age(25)` vs 생성자의 `25` |
| **불변 객체** | final 필드로 불변성 보장 | Thread-safe |
| **선택적 매개변수** | 필요한 것만 설정 | 3개 vs 10개 생성자 |
| **유효성 검증** | build() 시점에 검증 | 일관성 보장 |
| **메서드 체이닝** | Fluent API로 직관적 | `.a().b().c()` |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **코드량 증가** | Builder 클래스 추가 필요 | Lombok @Builder |
| **메모리 오버헤드** | Builder 객체 생성 비용 | 복잡한 객체에만 사용 |
| **학습 곡선** | 패턴 이해 필요 | 문서화, 예제 제공 |

---

## 7. 안티패턴

### ❌ 안티패턴 1: Setter 남용

```java
// 잘못된 예: Setter로 구성
public class BadUser {
    private String username;
    private String email;
    
    public void setUsername(String username) {
        this.username = username;
    }
    
    public void setEmail(String email) {
        this.email = email;
    }
}

// 문제
BadUser user = new BadUser();
user.setUsername("john");
// email이 설정 안 된 상태에서 사용 가능! (위험)
useUser(user);
```

**해결:**
```java
// 올바른 예: Builder로 불변 객체
User user = new User.Builder("john", "john@email.com")
        .build();
```

---

### ❌ 안티패턴 2: 유효성 검증 누락

```java
// 잘못된 예: 검증 없음
public class BadBuilder {
    public Product build() {
        return new Product(this);  // 유효성 검증 없음!
    }
}
```

**해결:**
```java
// 올바른 예: build()에서 검증
public Product build() {
    if (name == null || name.isEmpty()) {
        throw new IllegalStateException("Name is required");
    }
    if (price < 0) {
        throw new IllegalStateException("Price must be positive");
    }
    return new Product(this);
}
```

---

### ❌ 안티패턴 3: 불변성 깨기

```java
// 잘못된 예: Setter 노출
public class BadProduct {
    private final String name;
    private String description;  // final 아님!
    
    // Setter 제공 (불변성 깨짐)
    public void setDescription(String description) {
        this.description = description;
    }
}
```

**해결:**
```java
// 올바른 예: 모든 필드 final + Getter만
public class GoodProduct {
    private final String name;
    private final String description;
    
    // Getter만 제공
    public String getName() { return name; }
    public String getDescription() { return description; }
}
```

---

## 8. 핵심 정리

### 📌 Builder 패턴 체크리스트

```
✅ Product 클래스의 모든 필드를 final로
✅ Product 생성자는 private
✅ Builder는 static inner class로
✅ Builder 메서드는 this 반환 (메서드 체이닝)
✅ 필수 매개변수는 Builder 생성자로
✅ 선택적 매개변수는 Builder 메서드로
✅ build()에서 유효성 검증
✅ Getter만 제공, Setter 제공 안 함
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 매개변수가 4개 이상 | ⭐⭐⭐ | 가독성 향상 |
| 선택적 매개변수 많음 | ⭐⭐⭐ | Telescoping Constructor 방지 |
| 불변 객체 필요 | ⭐⭐⭐ | final 필드 사용 |
| 복잡한 생성 로직 | ⭐⭐⭐ | 생성 과정 분리 |

### 💡 핵심 포인트

1. **복잡한 객체는 Builder로!**
2. **불변 객체 만들기 최적!**
3. **Fluent API로 가독성 UP!**
4. **Lombok으로 보일러플레이트 제거!**
5. **build()에서 유효성 검증 필수!**

### 🔥 실무 팁

```java
// Tip 1: Lombok 사용 (가장 간편)
@Builder
public class Product {
    private final String name;
    @Builder.Default
    private final int stock = 0;
}

// Tip 2: 필수 매개변수는 Builder 생성자로
public Builder(String requiredParam) {
    this.requiredParam = requiredParam;
}

// Tip 3: 유효성 검증은 build()에서
public Product build() {
    validate();
    return new Product(this);
}

// Tip 4: 편의 메서드 제공
public Builder jsonBody(String json) {
    return contentType("application/json")
           .body(json);
}
```

---

## 🎓 연습 문제

### 문제 1: Email Builder 구현
```java
/**
 * 요구사항:
 * 1. 필수: to, subject, body
 * 2. 선택: cc, bcc, attachments
 * 3. 유효성 검증 (이메일 형식, 내용 길이)
 */
public class Email {
    // 여기에 구현
}
```

### 문제 2: Database Config Builder
```java
/**
 * 요구사항:
 * 1. 필수: host, port, database
 * 2. 선택: username, password, poolSize, timeout
 * 3. Builder Default 값 설정
 */
```

### 문제 3: 확장 과제
```java
/**
 * HttpRequest에 다음 기능 추가:
 * 1. 파일 업로드 (multipart/form-data)
 * 2. 쿠키 설정
 * 3. 재시도 로직 (maxRetries, retryDelay)
 */
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Factory Method](./02-FactoryMethod.md)**

</div>
