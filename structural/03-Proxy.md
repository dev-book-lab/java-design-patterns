# Proxy Pattern (프록시 패턴)

> **"객체 접근을 제어하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [장단점](#6-장단점)
7. [Decorator vs Proxy](#7-decorator-vs-proxy)
8. [핵심 정리](#8-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 무거운 객체를 항상 생성
public class HighResolutionImage {
    private String fileName;
    private byte[] imageData; // 수백 MB!
    
    public HighResolutionImage(String fileName) {
        this.fileName = fileName;
        loadFromDisk(); // 항상 로드 (느림!)
    }
    
    private void loadFromDisk() {
        System.out.println("Loading " + fileName + " from disk...");
        // 수백 MB 로드... 시간 오래 걸림
    }
    
    public void display() {
        System.out.println("Displaying " + fileName);
    }
}

// 100개 이미지 객체 생성 = 수십 초!
List<HighResolutionImage> images = new ArrayList<>();
for (int i = 0; i < 100; i++) {
    images.add(new HighResolutionImage("image" + i + ".jpg"));
    // 모두 즉시 로드... 너무 느림!
}

// 문제 2: 민감한 객체에 무제한 접근
public class BankAccount {
    private double balance;
    
    public void withdraw(double amount) {
        balance -= amount; // 누구나 접근 가능!
    }
    
    public double getBalance() {
        return balance; // 권한 체크 없음!
    }
}

// 문제 3: 원격 객체 접근 복잡도
public class Client {
    public void callRemoteService() {
        // 네트워크 연결
        Socket socket = new Socket("server.com", 8080);
        // 직렬화
        ObjectOutputStream out = new ObjectOutputStream(socket.getOutputStream());
        // 요청 전송
        out.writeObject(request);
        // 응답 수신
        ObjectInputStream in = new ObjectInputStream(socket.getInputStream());
        Response response = (Response) in.readObject();
        // 연결 종료
        socket.close();
        
        // 매번 이렇게 복잡한 코드 작성!
    }
}

// 문제 4: 로깅/캐싱 추가하려면?
public class DatabaseService {
    public User getUser(int id) {
        // DB 조회
        return database.query("SELECT * FROM users WHERE id=" + id);
        // 캐싱이나 로깅 추가하려면 코드 수정 필요!
    }
}
```

### ⚡ 핵심 문제

1. **지연 로딩 부재**: 무거운 객체를 즉시 생성
2. **접근 제어 없음**: 민감한 객체에 무제한 접근
3. **복잡한 원격 호출**: 네트워크 통신 코드 반복
4. **부가 기능 추가 어려움**: 캐싱, 로깅 등 추가 힘듦

---

## 2. 패턴 정의

### 📖 정의

> **다른 객체에 대한 접근을 제어하기 위해 대리자(Proxy)를 제공하는 패턴. 실제 객체의 인터페이스와 동일한 인터페이스를 제공하여 투명하게 접근을 제어한다.**

### 🎯 목적

- **지연 로딩**: 필요할 때만 객체 생성
- **접근 제어**: 권한 체크, 보안 강화
- **원격 프록시**: 네트워크 통신 단순화
- **부가 기능**: 로깅, 캐싱, 트랜잭션

### 💡 핵심 아이디어

```java
// Before: 직접 접근
RealObject obj = new RealObject(); // 즉시 생성
obj.request();

// After: Proxy로 제어
Subject proxy = new Proxy(); // 가벼운 객체
proxy.request(); // 필요할 때 RealObject 생성
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌─────────────────┐
│     Client      │
└─────────────────┘
         │
         │ uses
         ▼
┌──────────────────┐
│Subject(interface)│  ← 공통 인터페이스
├──────────────────┤
│ + request()      │
└──────────────────┘
         △
         │
    ┌────┴────┐
    │         │
┌───────┐ ┌───────────┐
│ Proxy │ │RealSubject│
├───────┤ └───────────┘
│-real  │────┐
│request│    │ has-a
└───────┘    │
             └─→ 실제 객체 참조
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **Subject** | 공통 인터페이스 | `Image` |
| **RealSubject** | 실제 객체 | `RealImage` |
| **Proxy** | 대리자 | `ProxyImage` |
| **Client** | 프록시 사용 | `ImageViewer` |

---

## 4. 구현 방법

### 방법 1: Virtual Proxy (가상 프록시) ⭐⭐⭐

```java
/**
 * Subject: 이미지 인터페이스
 */
public interface Image {
    void display();
    String getFileName();
}

/**
 * RealSubject: 실제 고해상도 이미지
 */
public class RealImage implements Image {
    private String fileName;
    private byte[] imageData;
    
    public RealImage(String fileName) {
        this.fileName = fileName;
        loadFromDisk();
    }
    
    private void loadFromDisk() {
        System.out.println("🔄 Loading " + fileName + " from disk...");
        try {
            Thread.sleep(1000); // 로딩 시뮬레이션
            imageData = new byte[1024 * 1024]; // 1MB
            System.out.println("✅ " + fileName + " loaded successfully!");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    
    @Override
    public void display() {
        System.out.println("🖼️ Displaying " + fileName);
    }
    
    @Override
    public String getFileName() {
        return fileName;
    }
}

/**
 * Proxy: 지연 로딩 프록시
 */
public class ProxyImage implements Image {
    private String fileName;
    private RealImage realImage; // 실제 객체 (지연 생성)
    
    public ProxyImage(String fileName) {
        this.fileName = fileName;
        System.out.println("📋 ProxyImage created for " + fileName);
    }
    
    @Override
    public void display() {
        // 실제 필요할 때만 RealImage 생성
        if (realImage == null) {
            System.out.println("💡 First access - creating RealImage");
            realImage = new RealImage(fileName);
        }
        realImage.display();
    }
    
    @Override
    public String getFileName() {
        return fileName;
    }
}

/**
 * 사용 예제
 */
public class VirtualProxyExample {
    public static void main(String[] args) {
        System.out.println("=== 프록시 생성 (빠름!) ===");
        long start = System.currentTimeMillis();
        
        Image image1 = new ProxyImage("photo1.jpg");
        Image image2 = new ProxyImage("photo2.jpg");
        Image image3 = new ProxyImage("photo3.jpg");
        
        long end = System.currentTimeMillis();
        System.out.println("⏱️ 생성 시간: " + (end - start) + "ms\n");
        
        // 실제 표시할 때만 로드
        System.out.println("=== 첫 번째 이미지 표시 ===");
        image1.display();
        
        System.out.println("\n=== 두 번째 이미지 표시 ===");
        image2.display();
        
        System.out.println("\n=== 첫 번째 이미지 재표시 (캐시됨) ===");
        image1.display();
    }
}
```

**실행 결과:**
```
=== 프록시 생성 (빠름!) ===
📋 ProxyImage created for photo1.jpg
📋 ProxyImage created for photo2.jpg
📋 ProxyImage created for photo3.jpg
⏱️ 생성 시간: 2ms

=== 첫 번째 이미지 표시 ===
💡 First access - creating RealImage
🔄 Loading photo1.jpg from disk...
✅ photo1.jpg loaded successfully!
🖼️ Displaying photo1.jpg

=== 두 번째 이미지 표시 ===
💡 First access - creating RealImage
🔄 Loading photo2.jpg from disk...
✅ photo2.jpg loaded successfully!
🖼️ Displaying photo2.jpg

=== 첫 번째 이미지 재표시 (캐시됨) ===
🖼️ Displaying photo1.jpg
```

---

### 방법 2: Protection Proxy (보호 프록시) ⭐⭐⭐

```java
/**
 * Subject: 문서 인터페이스
 */
public interface Document {
    void view();
    void edit(String content);
    void delete();
}

/**
 * RealSubject: 실제 문서
 */
public class RealDocument implements Document {
    private String title;
    private String content;
    
    public RealDocument(String title) {
        this.title = title;
        this.content = "Original content";
    }
    
    @Override
    public void view() {
        System.out.println("📄 Viewing document: " + title);
        System.out.println("   Content: " + content);
    }
    
    @Override
    public void edit(String content) {
        this.content = content;
        System.out.println("✏️ Document edited: " + title);
    }
    
    @Override
    public void delete() {
        System.out.println("🗑️ Document deleted: " + title);
    }
}

/**
 * User: 사용자 권한
 */
enum UserRole {
    VIEWER, EDITOR, ADMIN
}

class User {
    private String name;
    private UserRole role;
    
    public User(String name, UserRole role) {
        this.name = name;
        this.role = role;
    }
    
    public String getName() { return name; }
    public UserRole getRole() { return role; }
}

/**
 * Proxy: 권한 체크 프록시
 */
public class ProtectedDocument implements Document {
    private RealDocument document;
    private User user;
    
    public ProtectedDocument(String title, User user) {
        this.document = new RealDocument(title);
        this.user = user;
    }
    
    @Override
    public void view() {
        // 모든 사용자 가능
        System.out.println("👤 User: " + user.getName() + " (" + user.getRole() + ")");
        document.view();
    }
    
    @Override
    public void edit(String content) {
        // EDITOR 이상만 가능
        System.out.println("👤 User: " + user.getName() + " (" + user.getRole() + ")");
        
        if (user.getRole() == UserRole.EDITOR || user.getRole() == UserRole.ADMIN) {
            document.edit(content);
        } else {
            System.out.println("❌ Access Denied! EDITOR role required.");
        }
    }
    
    @Override
    public void delete() {
        // ADMIN만 가능
        System.out.println("👤 User: " + user.getName() + " (" + user.getRole() + ")");
        
        if (user.getRole() == UserRole.ADMIN) {
            document.delete();
        } else {
            System.out.println("❌ Access Denied! ADMIN role required.");
        }
    }
}

/**
 * 사용 예제
 */
public class ProtectionProxyExample {
    public static void main(String[] args) {
        // 사용자 생성
        User viewer = new User("Alice", UserRole.VIEWER);
        User editor = new User("Bob", UserRole.EDITOR);
        User admin = new User("Charlie", UserRole.ADMIN);
        
        // VIEWER 권한
        System.out.println("=== VIEWER (Alice) ===");
        Document doc1 = new ProtectedDocument("Report.docx", viewer);
        doc1.view();
        doc1.edit("Modified content");
        doc1.delete();
        
        // EDITOR 권한
        System.out.println("\n=== EDITOR (Bob) ===");
        Document doc2 = new ProtectedDocument("Report.docx", editor);
        doc2.view();
        doc2.edit("Modified content");
        doc2.delete();
        
        // ADMIN 권한
        System.out.println("\n=== ADMIN (Charlie) ===");
        Document doc3 = new ProtectedDocument("Report.docx", admin);
        doc3.view();
        doc3.edit("Modified content");
        doc3.delete();
    }
}
```

**실행 결과:**
```
=== VIEWER (Alice) ===
👤 User: Alice (VIEWER)
📄 Viewing document: Report.docx
   Content: Original content
👤 User: Alice (VIEWER)
❌ Access Denied! EDITOR role required.
👤 User: Alice (VIEWER)
❌ Access Denied! ADMIN role required.

=== EDITOR (Bob) ===
👤 User: Bob (EDITOR)
📄 Viewing document: Report.docx
   Content: Original content
👤 User: Bob (EDITOR)
✏️ Document edited: Report.docx
👤 User: Bob (EDITOR)
❌ Access Denied! ADMIN role required.

=== ADMIN (Charlie) ===
👤 User: Charlie (ADMIN)
📄 Viewing document: Report.docx
   Content: Original content
👤 User: Charlie (ADMIN)
✏️ Document edited: Report.docx
👤 User: Charlie (ADMIN)
🗑️ Document deleted: Report.docx
```

---

### 방법 3: Caching Proxy (캐싱 프록시) ⭐⭐

```java
/**
 * Subject: 데이터베이스 인터페이스
 */
public interface Database {
    User getUser(int id);
}

/**
 * User 클래스
 */
class User {
    private int id;
    private String name;
    
    public User(int id, String name) {
        this.id = id;
        this.name = name;
    }
    
    public int getId() { return id; }
    public String getName() { return name; }
    
    @Override
    public String toString() {
        return "User{id=" + id + ", name='" + name + "'}";
    }
}

/**
 * RealSubject: 실제 데이터베이스
 */
public class RealDatabase implements Database {
    @Override
    public User getUser(int id) {
        System.out.println("🔍 Querying database for user " + id + "...");
        try {
            Thread.sleep(1000); // DB 조회 시뮬레이션
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return new User(id, "User" + id);
    }
}

/**
 * Proxy: 캐싱 프록시
 */
public class CachingDatabaseProxy implements Database {
    private RealDatabase database;
    private Map<Integer, User> cache;
    
    public CachingDatabaseProxy() {
        this.database = new RealDatabase();
        this.cache = new HashMap<>();
    }
    
    @Override
    public User getUser(int id) {
        // 캐시 확인
        if (cache.containsKey(id)) {
            System.out.println("✨ Cache hit for user " + id);
            return cache.get(id);
        }
        
        // 캐시 미스 - DB 조회
        System.out.println("❌ Cache miss for user " + id);
        User user = database.getUser(id);
        
        // 캐시에 저장
        cache.put(id, user);
        System.out.println("💾 User " + id + " cached");
        
        return user;
    }
    
    public void clearCache() {
        cache.clear();
        System.out.println("🗑️ Cache cleared");
    }
}

/**
 * 사용 예제
 */
public class CachingProxyExample {
    public static void main(String[] args) {
        Database db = new CachingDatabaseProxy();
        
        System.out.println("=== 첫 번째 조회 ===");
        User user1 = db.getUser(1);
        System.out.println("Result: " + user1);
        
        System.out.println("\n=== 두 번째 조회 (캐시됨) ===");
        User user2 = db.getUser(1);
        System.out.println("Result: " + user2);
        
        System.out.println("\n=== 다른 사용자 조회 ===");
        User user3 = db.getUser(2);
        System.out.println("Result: " + user3);
        
        System.out.println("\n=== 캐시된 사용자 재조회 ===");
        User user4 = db.getUser(2);
        System.out.println("Result: " + user4);
    }
}
```

**실행 결과:**
```
=== 첫 번째 조회 ===
❌ Cache miss for user 1
🔍 Querying database for user 1...
💾 User 1 cached
Result: User{id=1, name='User1'}

=== 두 번째 조회 (캐시됨) ===
✨ Cache hit for user 1
Result: User{id=1, name='User1'}

=== 다른 사용자 조회 ===
❌ Cache miss for user 2
🔍 Querying database for user 2...
💾 User 2 cached
Result: User{id=2, name='User2'}

=== 캐시된 사용자 재조회 ===
✨ Cache hit for user 2
Result: User{id=2, name='User2'}
```

---

## 5. 실전 예제

### 예제 1: 로깅 프록시 ⭐⭐⭐

```java
/**
 * Subject: 계산기
 */
public interface Calculator {
    int add(int a, int b);
    int subtract(int a, int b);
    int multiply(int a, int b);
    int divide(int a, int b);
}

/**
 * RealSubject: 실제 계산기
 */
public class RealCalculator implements Calculator {
    @Override
    public int add(int a, int b) {
        return a + b;
    }
    
    @Override
    public int subtract(int a, int b) {
        return a - b;
    }
    
    @Override
    public int multiply(int a, int b) {
        return a * b;
    }
    
    @Override
    public int divide(int a, int b) {
        return a / b;
    }
}

/**
 * Proxy: 로깅 프록시
 */
public class LoggingCalculatorProxy implements Calculator {
    private RealCalculator calculator;
    
    public LoggingCalculatorProxy() {
        this.calculator = new RealCalculator();
    }
    
    @Override
    public int add(int a, int b) {
        long start = System.nanoTime();
        int result = calculator.add(a, b);
        long end = System.nanoTime();
        
        System.out.println("➕ add(" + a + ", " + b + ") = " + result + 
                " [" + (end - start) + "ns]");
        return result;
    }
    
    @Override
    public int subtract(int a, int b) {
        long start = System.nanoTime();
        int result = calculator.subtract(a, b);
        long end = System.nanoTime();
        
        System.out.println("➖ subtract(" + a + ", " + b + ") = " + result + 
                " [" + (end - start) + "ns]");
        return result;
    }
    
    @Override
    public int multiply(int a, int b) {
        long start = System.nanoTime();
        int result = calculator.multiply(a, b);
        long end = System.nanoTime();
        
        System.out.println("✖️ multiply(" + a + ", " + b + ") = " + result + 
                " [" + (end - start) + "ns]");
        return result;
    }
    
    @Override
    public int divide(int a, int b) {
        long start = System.nanoTime();
        
        try {
            int result = calculator.divide(a, b);
            long end = System.nanoTime();
            
            System.out.println("➗ divide(" + a + ", " + b + ") = " + result + 
                    " [" + (end - start) + "ns]");
            return result;
        } catch (ArithmeticException e) {
            long end = System.nanoTime();
            System.out.println("❌ divide(" + a + ", " + b + ") = ERROR: " + 
                    e.getMessage() + " [" + (end - start) + "ns]");
            throw e;
        }
    }
}

/**
 * 사용 예제
 */
public class LoggingProxyExample {
    public static void main(String[] args) {
        Calculator calc = new LoggingCalculatorProxy();
        
        System.out.println("=== 계산 시작 ===");
        calc.add(10, 5);
        calc.subtract(10, 5);
        calc.multiply(10, 5);
        calc.divide(10, 5);
        
        try {
            calc.divide(10, 0);
        } catch (ArithmeticException e) {
            System.out.println("예외 발생 처리됨");
        }
    }
}
```

---

### 예제 2: Remote Proxy (원격 프록시) ⭐⭐

```java
/**
 * Subject: 원격 서비스
 */
public interface WeatherService {
    String getWeather(String city);
}

/**
 * RealSubject: 실제 원격 서비스 (서버)
 */
public class RealWeatherService implements WeatherService {
    @Override
    public String getWeather(String city) {
        // 실제 날씨 API 호출
        return "Sunny, 25°C"; // 간단히 시뮬레이션
    }
}

/**
 * Proxy: 원격 프록시
 */
public class RemoteWeatherProxy implements WeatherService {
    
    @Override
    public String getWeather(String city) {
        System.out.println("🌐 Connecting to weather server...");
        
        try {
            // 네트워크 지연 시뮬레이션
            Thread.sleep(500);
            
            // 실제로는 HTTP 요청
            System.out.println("📡 Sending request for " + city);
            
            // 응답 수신 시뮬레이션
            Thread.sleep(500);
            String weather = "Sunny, 25°C in " + city;
            
            System.out.println("📥 Received response");
            return weather;
            
        } catch (InterruptedException e) {
            return "Error: Connection failed";
        }
    }
}

/**
 * 사용 예제
 */
public class RemoteProxyExample {
    public static void main(String[] args) {
        WeatherService service = new RemoteWeatherProxy();
        
        System.out.println("=== 날씨 조회 ===");
        String weather = service.getWeather("Seoul");
        System.out.println("Result: " + weather);
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **지연 로딩** | 필요할 때만 생성 | 이미지 프록시 |
| **접근 제어** | 권한 체크 | 보호 프록시 |
| **성능 향상** | 캐싱, 최적화 | 캐싱 프록시 |
| **OCP 준수** | 기존 코드 수정 없이 확장 | 로깅 추가 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **응답 지연** | 프록시 레이어 추가 | 성능 측정 |
| **복잡도 증가** | 클래스 수 증가 | 필요시에만 사용 |

---

## 7. Decorator vs Proxy

### 🔍 비교표

| 특징 | Decorator | Proxy |
|------|-----------|-------|
| **목적** | 기능 추가 | 접근 제어 |
| **생성 시점** | 클라이언트가 조합 | 프록시가 생성 |
| **투명성** | 클라이언트가 인지 | 클라이언트 모름 |
| **예시** | 커피 옵션 추가 | 이미지 지연 로딩 |

---

## 8. 핵심 정리

### 📌 Proxy 패턴 체크리스트

```
✅ Subject 인터페이스 정의
✅ RealSubject 구현
✅ Proxy 구현
✅ 접근 제어 로직 추가
✅ 클라이언트는 Proxy 사용
```

### 🎯 프록시 종류

| 종류 | 목적 | 예시 |
|------|------|------|
| **Virtual Proxy** | 지연 로딩 | 이미지, 문서 |
| **Protection Proxy** | 권한 체크 | 보안, 인증 |
| **Remote Proxy** | 원격 접근 | RMI, 웹 서비스 |
| **Caching Proxy** | 결과 캐싱 | DB 프록시 |
| **Logging Proxy** | 로깅 추가 | AOP |

### 💡 핵심 포인트

1. **접근 제어가 핵심**
2. **실제 객체를 감쌈**
3. **투명하게 동작**
4. **다양한 프록시 조합 가능**

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Decorator](02-Decorator.md) | [다음: Facade →](04-Composite.md)**

</div>
