# Singleton Pattern (싱글톤 패턴)

> **"애플리케이션 전체에서 단 하나의 인스턴스만 존재해야 할 때"**

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
// 문제 1: 설정 파일을 여러 번 읽어야 하는 상황
public class Application {
    public static void main(String[] args) {
        Config config1 = new Config();  // 파일 읽기 1
        Config config2 = new Config();  // 파일 읽기 2 (중복!)
        Config config3 = new Config();  // 파일 읽기 3 (중복!)
        
        // 설정이 달라질 수 있음!
        config1.setDatabaseUrl("localhost");
        // config2, config3는 여전히 이전 값
    }
}

// 문제 2: 데이터베이스 커넥션 풀이 여러 개 생성됨
public class Service {
    private ConnectionPool pool = new ConnectionPool(10); // 10개 연결
}

public class Repository {
    private ConnectionPool pool = new ConnectionPool(10); // 또 10개 연결!
    // 총 20개 연결... 비효율적!
}

// 문제 3: 로거가 여러 개 생성되어 로그가 섞임
public class OrderService {
    private Logger logger = new Logger("app.log");
}

public class UserService {
    private Logger logger = new Logger("app.log"); // 동일한 파일에 동시 쓰기!
}
```

### ⚡ 핵심 문제

1. **리소스 낭비**: 무거운 객체가 여러 번 생성됨
2. **상태 불일치**: 여러 인스턴스가 다른 상태를 가질 수 있음
3. **동기화 문제**: 공유 리소스 접근 시 충돌 가능

---

## 2. 패턴 정의

### 📖 정의

> **클래스의 인스턴스가 오직 하나만 생성되도록 보장하고, 그 인스턴스에 대한 전역 접근점을 제공하는 패턴**

### 🎯 목적

- 클래스의 인스턴스를 **단 하나만** 생성
- 어디서든 동일한 인스턴스에 **접근 가능**
- **Lazy Initialization** 지원 (필요할 때만 생성)

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌─────────────────────────┐
│      Singleton          │
├─────────────────────────┤
│ - instance: Singleton   │  ← private static (단 하나)
├─────────────────────────┤
│ - Singleton()           │  ← private 생성자 (외부 생성 차단)
│ + getInstance(): Single │  ← public static (전역 접근점)
└─────────────────────────┘
```

### 🔧 구성요소

| 요소 | 설명 | 중요도 |
|------|------|--------|
| **private static instance** | 유일한 인스턴스를 저장 | ⭐⭐⭐ |
| **private 생성자** | 외부에서 new 사용 불가 | ⭐⭐⭐ |
| **public static getInstance()** | 인스턴스 접근 메서드 | ⭐⭐⭐ |

---

## 4. 구현 방법

### 방법 1: Eager Initialization (즉시 초기화) ⭐

```java
/**
 * 가장 간단하고 안전한 방법
 * 클래스 로딩 시점에 인스턴스 생성
 */
public class EagerSingleton {
    // 1. private static final로 인스턴스 즉시 생성
    private static final EagerSingleton INSTANCE = new EagerSingleton();
    
    // 2. private 생성자로 외부 생성 차단
    private EagerSingleton() {
        System.out.println("EagerSingleton 인스턴스 생성!");
    }
    
    // 3. public static으로 전역 접근점 제공
    public static EagerSingleton getInstance() {
        return INSTANCE;
    }
    
    // 비즈니스 메서드
    public void doSomething() {
        System.out.println("작업 수행: " + this);
    }
}

// 사용 예제
public class EagerExample {
    public static void main(String[] args) {
        EagerSingleton s1 = EagerSingleton.getInstance();
        EagerSingleton s2 = EagerSingleton.getInstance();
        
        System.out.println("s1 == s2: " + (s1 == s2)); // true
        System.out.println("s1 주소: " + s1);
        System.out.println("s2 주소: " + s2);
        
        s1.doSomething();
        s2.doSomething(); // 동일한 객체
    }
}
```

**실행 결과:**
```
EagerSingleton 인스턴스 생성!
s1 == s2: true
s1 주소: EagerSingleton@15db9742
s2 주소: EagerSingleton@15db9742
작업 수행: EagerSingleton@15db9742
작업 수행: EagerSingleton@15db9742
```

**장점:**
- ✅ 구현이 간단
- ✅ Thread-safe (클래스 로더가 보장)
- ✅ static final로 불변성 보장

**단점:**
- ❌ 사용하지 않아도 무조건 생성됨 (메모리 낭비 가능)

---

### 방법 2: Lazy Initialization (지연 초기화) - Thread Unsafe ⚠️

```java
/**
 * 필요할 때만 생성하지만, Thread-safe하지 않음
 * 단일 스레드 환경에서만 사용!
 */
public class LazySingleton {
    private static LazySingleton instance;
    
    private LazySingleton() {
        System.out.println("LazySingleton 인스턴스 생성!");
    }
    
    // 호출될 때 생성
    public static LazySingleton getInstance() {
        if (instance == null) {  // ⚠️ Thread-unsafe
            instance = new LazySingleton();
        }
        return instance;
    }
}

// 멀티스레드 문제 재현
public class LazyThreadProblem {
    public static void main(String[] args) {
        // 10개 스레드가 동시에 getInstance() 호출
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                LazySingleton instance = LazySingleton.getInstance();
                System.out.println(Thread.currentThread().getName() 
                    + ": " + instance);
            }).start();
        }
    }
}
```

**문제 상황:**
```
Thread-0: LazySingleton@1a2b3c4d  ← 인스턴스 1
Thread-1: LazySingleton@5e6f7g8h  ← 인스턴스 2 (문제!)
Thread-2: LazySingleton@1a2b3c4d  ← 인스턴스 1
...
```

**왜 문제가 발생하나?**
```java
// Thread A와 Thread B가 동시 실행
Thread A: if (instance == null) { // true
Thread B: if (instance == null) { // true (동시에!)
Thread A:     instance = new LazySingleton(); // 인스턴스 1 생성
Thread B:     instance = new LazySingleton(); // 인스턴스 2 생성 (덮어씀!)
```

---

### 방법 3: Synchronized Method (동기화) 🔒

```java
/**
 * synchronized로 Thread-safe 보장
 * 하지만 성능 이슈 존재
 */
public class SynchronizedSingleton {
    private static SynchronizedSingleton instance;
    
    private SynchronizedSingleton() {
        System.out.println("SynchronizedSingleton 인스턴스 생성!");
    }
    
    // synchronized로 한 번에 한 스레드만 접근
    public static synchronized SynchronizedSingleton getInstance() {
        if (instance == null) {
            instance = new SynchronizedSingleton();
        }
        return instance;
    }
}

// 성능 테스트
public class SynchronizedPerformanceTest {
    public static void main(String[] args) throws InterruptedException {
        int threads = 1000;
        CountDownLatch latch = new CountDownLatch(threads);
        
        long start = System.nanoTime();
        
        for (int i = 0; i < threads; i++) {
            new Thread(() -> {
                SynchronizedSingleton.getInstance(); // 매번 synchronized
                latch.countDown();
            }).start();
        }
        
        latch.await();
        long end = System.nanoTime();
        
        System.out.println("실행 시간: " + (end - start) / 1_000_000 + "ms");
    }
}
```

**장점:**
- ✅ Thread-safe 보장
- ✅ Lazy Initialization

**단점:**
- ❌ **성능 저하**: 이미 생성된 후에도 매번 synchronized 오버헤드
- ❌ getInstance() 호출할 때마다 락 대기

---

### 방법 4: Double-Checked Locking (DCL) ⭐⭐

```java
/**
 * synchronized 오버헤드를 최소화한 방법
 * volatile 키워드 필수!
 */
public class DCLSingleton {
    // volatile: 가시성 보장 (CPU 캐시 무효화)
    private static volatile DCLSingleton instance;
    
    private DCLSingleton() {
        System.out.println("DCLSingleton 인스턴스 생성!");
    }
    
    public static DCLSingleton getInstance() {
        // 1차 체크: 인스턴스가 있으면 바로 반환 (빠름!)
        if (instance == null) {
            // 2차 체크: synchronized 블록 (생성 시에만)
            synchronized (DCLSingleton.class) {
                if (instance == null) {
                    instance = new DCLSingleton();
                }
            }
        }
        return instance;
    }
}

// volatile이 왜 필요한가?
public class VolatileExample {
    public static void main(String[] args) {
        /*
         * instance = new DCLSingleton() 내부 동작:
         * 1. 메모리 할당
         * 2. 생성자 실행
         * 3. instance에 메모리 주소 할당
         * 
         * volatile 없으면:
         * - 순서가 2-3-1로 재배치될 수 있음 (명령어 재배치)
         * - Thread A가 3번만 실행 후, Thread B가 1차 체크 통과
         * - Thread B가 초기화 안 된 객체 사용 (문제!)
         * 
         * volatile 있으면:
         * - 명령어 재배치 방지
         * - 가시성 보장 (CPU 캐시가 아닌 메인 메모리 접근)
         */
    }
}
```

**장점:**
- ✅ Thread-safe
- ✅ Lazy Initialization
- ✅ **성능 우수**: 생성 후에는 synchronized 없음

**단점:**
- ❌ 코드가 복잡함
- ❌ volatile 이해 필요

---

### 방법 5: Bill Pugh Solution (Initialization-on-demand holder) ⭐⭐⭐ 추천!

```java
/**
 * 가장 권장되는 방법!
 * - Lazy Initialization
 * - Thread-safe (JVM이 보장)
 * - 성능 우수
 */
public class BillPughSingleton {
    
    // private 생성자
    private BillPughSingleton() {
        System.out.println("BillPughSingleton 인스턴스 생성!");
    }
    
    // static inner class
    // getInstance() 호출 전까지 로딩되지 않음!
    private static class SingletonHolder {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }
    
    public static BillPughSingleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
    
    public void doSomething() {
        System.out.println("작업 수행!");
    }
}

// 동작 원리 이해
public class BillPughExample {
    public static void main(String[] args) {
        System.out.println("=== 프로그램 시작 ===");
        // 아직 SingletonHolder 클래스 로딩 안 됨
        
        System.out.println("=== getInstance() 호출 ===");
        BillPughSingleton s1 = BillPughSingleton.getInstance();
        // 이 시점에 SingletonHolder 로딩 → INSTANCE 생성
        
        System.out.println("=== 두 번째 호출 ===");
        BillPughSingleton s2 = BillPughSingleton.getInstance();
        // 이미 생성됨, 바로 반환
        
        System.out.println("s1 == s2: " + (s1 == s2));
    }
}
```

**실행 결과:**
```
=== 프로그램 시작 ===
=== getInstance() 호출 ===
BillPughSingleton 인스턴스 생성!
=== 두 번째 호출 ===
s1 == s2: true
```

**왜 Thread-safe한가?**
```java
/*
 * JVM이 클래스 초기화 과정에서 동기화를 보장
 * 
 * 1. SingletonHolder 클래스는 getInstance() 호출 시 로딩
 * 2. 클래스 로딩은 JVM이 synchronized로 보호
 * 3. INSTANCE는 static final이므로 한 번만 초기화
 * 4. 개발자가 synchronized 쓸 필요 없음!
 */
```

**장점:**
- ✅ **최고의 선택!**
- ✅ Lazy Initialization
- ✅ Thread-safe (JVM 보장)
- ✅ 성능 우수 (synchronized 없음)
- ✅ 코드 간결

---

### 방법 6: Enum Singleton ⭐⭐⭐ 최강!

```java
/**
 * Joshua Bloch가 제안한 방법
 * - 가장 안전하고 간결
 * - Serialization 공격 방어
 * - Reflection 공격 방어
 */
public enum EnumSingleton {
    INSTANCE;  // 유일한 인스턴스
    
    // 인스턴스 변수
    private int value;
    
    // 생성자 (자동으로 private)
    EnumSingleton() {
        System.out.println("EnumSingleton 인스턴스 생성!");
    }
    
    // 비즈니스 메서드
    public void setValue(int value) {
        this.value = value;
    }
    
    public int getValue() {
        return value;
    }
    
    public void doSomething() {
        System.out.println("작업 수행! value=" + value);
    }
}

// 사용 예제
public class EnumSingletonExample {
    public static void main(String[] args) {
        // INSTANCE로 직접 접근
        EnumSingleton s1 = EnumSingleton.INSTANCE;
        EnumSingleton s2 = EnumSingleton.INSTANCE;
        
        System.out.println("s1 == s2: " + (s1 == s2)); // true
        
        s1.setValue(100);
        System.out.println("s2.getValue(): " + s2.getValue()); // 100
        
        s1.doSomething();
        s2.doSomething(); // 동일한 객체
    }
}
```

**실행 결과:**
```
EnumSingleton 인스턴스 생성!
s1 == s2: true
s2.getValue(): 100
작업 수행! value=100
작업 수행! value=100
```

**Serialization 공격 방어**

```java
// 일반 Singleton의 문제
public class NormalSingleton implements Serializable {
    private static final NormalSingleton INSTANCE = new NormalSingleton();
    
    private NormalSingleton() {}
    
    public static NormalSingleton getInstance() {
        return INSTANCE;
    }
}

// Serialization 공격
public class SerializationAttack {
    public static void main(String[] args) throws Exception {
        NormalSingleton s1 = NormalSingleton.getInstance();
        
        // 직렬화
        ObjectOutputStream oos = new ObjectOutputStream(
            new FileOutputStream("singleton.ser")
        );
        oos.writeObject(s1);
        oos.close();
        
        // 역직렬화 → 새로운 인스턴스 생성됨!
        ObjectInputStream ois = new ObjectInputStream(
            new FileInputStream("singleton.ser")
        );
        NormalSingleton s2 = (NormalSingleton) ois.readObject();
        ois.close();
        
        System.out.println("s1 == s2: " + (s1 == s2)); // false (문제!)
    }
}

// Enum은 자동으로 방어
public class EnumSerializationSafe {
    public static void main(String[] args) throws Exception {
        EnumSingleton s1 = EnumSingleton.INSTANCE;
        
        // 직렬화
        ObjectOutputStream oos = new ObjectOutputStream(
            new FileOutputStream("enum.ser")
        );
        oos.writeObject(s1);
        oos.close();
        
        // 역직렬화 → 동일한 인스턴스!
        ObjectInputStream ois = new ObjectInputStream(
            new FileInputStream("enum.ser")
        );
        EnumSingleton s2 = (EnumSingleton) ois.readObject();
        ois.close();
        
        System.out.println("s1 == s2: " + (s1 == s2)); // true (안전!)
    }
}
```

**Reflection 공격 방어**

```java
// 일반 Singleton 공격 가능
public class ReflectionAttack {
    public static void main(String[] args) throws Exception {
        BillPughSingleton s1 = BillPughSingleton.getInstance();
        
        // Reflection으로 private 생성자 호출
        Constructor<BillPughSingleton> constructor = 
            BillPughSingleton.class.getDeclaredConstructor();
        constructor.setAccessible(true);
        BillPughSingleton s2 = constructor.newInstance(); // 새 인스턴스!
        
        System.out.println("s1 == s2: " + (s1 == s2)); // false (공격 성공)
    }
}

// Enum은 Reflection 방어
public class EnumReflectionSafe {
    public static void main(String[] args) {
        try {
            Constructor<EnumSingleton> constructor = 
                EnumSingleton.class.getDeclaredConstructor();
            constructor.setAccessible(true);
            EnumSingleton instance = constructor.newInstance();
        } catch (Exception e) {
            System.out.println("예외 발생: " + e.getMessage());
            // Cannot reflectively create enum objects
        }
    }
}
```

**장점:**
- ✅ **가장 안전한 방법**
- ✅ 코드 간결 (3줄!)
- ✅ Serialization 안전
- ✅ Reflection 안전
- ✅ Thread-safe

**단점:**
- ❌ 상속 불가 (enum은 상속 불가)
- ❌ Lazy Initialization 불가

---

## 5. 실전 예제

### 예제 1: 애플리케이션 설정 관리자 ⭐⭐⭐

```java
/**
 * 설정 파일을 한 번만 읽고, 전역에서 공유
 */
public enum AppConfig {
    INSTANCE;
    
    private Properties properties;
    
    AppConfig() {
        properties = new Properties();
        try (InputStream input = getClass()
                .getResourceAsStream("/config.properties")) {
            properties.load(input);
            System.out.println("설정 파일 로드 완료!");
        } catch (IOException e) {
            throw new RuntimeException("설정 파일 로드 실패", e);
        }
    }
    
    public String get(String key) {
        return properties.getProperty(key);
    }
    
    public int getInt(String key) {
        return Integer.parseInt(properties.getProperty(key));
    }
    
    public void set(String key, String value) {
        properties.setProperty(key, value);
    }
}

// 사용 예제
public class ConfigExample {
    public static void main(String[] args) {
        // 어디서든 동일한 설정 접근
        String dbUrl = AppConfig.INSTANCE.get("database.url");
        int maxConnections = AppConfig.INSTANCE.getInt("database.max.connections");
        
        System.out.println("DB URL: " + dbUrl);
        System.out.println("최대 연결: " + maxConnections);
        
        // 런타임에 설정 변경
        AppConfig.INSTANCE.set("app.mode", "production");
        
        // 다른 클래스에서도 동일한 설정
        new OrderService().processOrder();
        new UserService().createUser();
    }
}

class OrderService {
    void processOrder() {
        String mode = AppConfig.INSTANCE.get("app.mode");
        System.out.println("주문 처리 (모드: " + mode + ")");
    }
}

class UserService {
    void createUser() {
        String mode = AppConfig.INSTANCE.get("app.mode");
        System.out.println("사용자 생성 (모드: " + mode + ")");
    }
}
```

**config.properties:**
```properties
database.url=jdbc:mysql://localhost:3306/mydb
database.max.connections=20
app.mode=development
```

**실행 결과:**
```
설정 파일 로드 완료!
DB URL: jdbc:mysql://localhost:3306/mydb
최대 연결: 20
주문 처리 (모드: production)
사용자 생성 (모드: production)
```

---

### 예제 2: 데이터베이스 커넥션 풀 ⭐⭐⭐

```java
/**
 * 커넥션 풀을 Singleton으로 관리
 * 리소스 재사용으로 성능 향상
 */
public class ConnectionPool {
    private static volatile ConnectionPool instance;
    private BlockingQueue<Connection> pool;
    private final int MAX_POOL_SIZE = 10;
    
    private ConnectionPool() {
        pool = new LinkedBlockingQueue<>(MAX_POOL_SIZE);
        initializePool();
        System.out.println("커넥션 풀 초기화 완료 (크기: " + MAX_POOL_SIZE + ")");
    }
    
    public static ConnectionPool getInstance() {
        if (instance == null) {
            synchronized (ConnectionPool.class) {
                if (instance == null) {
                    instance = new ConnectionPool();
                }
            }
        }
        return instance;
    }
    
    private void initializePool() {
        try {
            for (int i = 0; i < MAX_POOL_SIZE; i++) {
                Connection conn = DriverManager.getConnection(
                    "jdbc:mysql://localhost:3306/mydb",
                    "user", "password"
                );
                pool.offer(conn);
            }
        } catch (SQLException e) {
            throw new RuntimeException("풀 초기화 실패", e);
        }
    }
    
    // 커넥션 가져오기
    public Connection getConnection() throws InterruptedException {
        Connection conn = pool.take(); // 비어있으면 대기
        System.out.println("커넥션 대여 (남은 개수: " + pool.size() + ")");
        return conn;
    }
    
    // 커넥션 반환
    public void releaseConnection(Connection conn) {
        if (conn != null) {
            pool.offer(conn);
            System.out.println("커넥션 반환 (남은 개수: " + pool.size() + ")");
        }
    }
    
    public int getAvailableConnections() {
        return pool.size();
    }
}

// 사용 예제
public class ConnectionPoolExample {
    public static void main(String[] args) {
        ConnectionPool pool = ConnectionPool.getInstance();
        
        // 여러 스레드가 동일한 풀 사용
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                try {
                    Connection conn = pool.getConnection();
                    
                    // DB 작업 시뮬레이션
                    Thread.sleep(1000);
                    
                    pool.releaseConnection(conn);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }, "Worker-" + i).start();
        }
        
        // 다른 곳에서도 동일한 풀 사용
        ConnectionPool pool2 = ConnectionPool.getInstance();
        System.out.println("pool == pool2: " + (pool == pool2)); // true
    }
}
```

**실행 결과:**
```
커넥션 풀 초기화 완료 (크기: 10)
pool == pool2: true
커넥션 대여 (남은 개수: 9)
커넥션 대여 (남은 개수: 8)
커넥션 대여 (남은 개수: 7)
커넥션 대여 (남은 개수: 6)
커넥션 대여 (남은 개수: 5)
커넥션 반환 (남은 개수: 6)
커넥션 반환 (남은 개수: 7)
...
```

---

### 예제 3: 로거(Logger) 시스템 ⭐⭐

```java
/**
 * 파일 로거를 Singleton으로 구현
 * 여러 곳에서 동일한 파일에 로그 기록
 */
public enum Logger {
    INSTANCE;
    
    private PrintWriter writer;
    private final String LOG_FILE = "application.log";
    
    Logger() {
        try {
            writer = new PrintWriter(new FileWriter(LOG_FILE, true));
            System.out.println("로거 초기화: " + LOG_FILE);
        } catch (IOException e) {
            throw new RuntimeException("로거 초기화 실패", e);
        }
    }
    
    public synchronized void log(Level level, String message) {
        String timestamp = LocalDateTime.now()
            .format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
        String logMessage = String.format("[%s] [%s] %s", 
            timestamp, level, message);
        
        System.out.println(logMessage); // 콘솔 출력
        writer.println(logMessage);     // 파일 출력
        writer.flush();
    }
    
    public void info(String message) {
        log(Level.INFO, message);
    }
    
    public void warn(String message) {
        log(Level.WARN, message);
    }
    
    public void error(String message) {
        log(Level.ERROR, message);
    }
    
    public void close() {
        if (writer != null) {
            writer.close();
        }
    }
    
    enum Level {
        INFO, WARN, ERROR
    }
}

// 사용 예제
public class LoggerExample {
    public static void main(String[] args) {
        // 여러 클래스에서 동일한 로거 사용
        OrderService orderService = new OrderService();
        UserService userService = new UserService();
        PaymentService paymentService = new PaymentService();
        
        orderService.createOrder(1001);
        userService.registerUser("john@example.com");
        paymentService.processPayment(1001, 50000);
        
        // 로거 종료
        Logger.INSTANCE.close();
    }
}

class OrderService {
    void createOrder(int orderId) {
        Logger.INSTANCE.info("주문 생성 시작: " + orderId);
        // 주문 처리 로직
        Logger.INSTANCE.info("주문 생성 완료: " + orderId);
    }
}

class UserService {
    void registerUser(String email) {
        Logger.INSTANCE.info("사용자 등록: " + email);
        // 사용자 등록 로직
    }
}

class PaymentService {
    void processPayment(int orderId, int amount) {
        Logger.INSTANCE.info("결제 처리 시작: 주문=" + orderId + ", 금액=" + amount);
        // 결제 처리 로직
        Logger.INSTANCE.info("결제 완료");
    }
}
```

**실행 결과 (콘솔 & application.log):**
```
로거 초기화: application.log
[2024-12-18 10:30:45] [INFO] 주문 생성 시작: 1001
[2024-12-18 10:30:45] [INFO] 주문 생성 완료: 1001
[2024-12-18 10:30:45] [INFO] 사용자 등록: john@example.com
[2024-12-18 10:30:45] [INFO] 결제 처리 시작: 주문=1001, 금액=50000
[2024-12-18 10:30:45] [INFO] 결제 완료
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **단일 인스턴스 보장** | 메모리 절약, 상태 공유 | 설정, 캐시, 로거 |
| **전역 접근점** | 어디서든 접근 가능 | getInstance() |
| **Lazy Initialization** | 필요할 때만 생성 | Bill Pugh, DCL |
| **리소스 공유** | DB 커넥션, 파일 핸들 | ConnectionPool |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **전역 상태** | 암묵적 의존성 증가 | DI(Dependency Injection) 고려 |
| **테스트 어려움** | 상태 공유로 독립적 테스트 힘듦 | Mock 객체 사용 |
| **멀티스레딩** | 동기화 문제 가능 | Thread-safe 구현 필수 |
| **단일 책임 위반** | 인스턴스 관리 + 비즈니스 로직 | 책임 분리 고려 |

---

## 7. 안티패턴

### ❌ 안티패턴 1: public 생성자

```java
// 잘못된 예
public class BadSingleton {
    private static BadSingleton instance;
    
    // public 생성자 → 외부에서 생성 가능!
    public BadSingleton() {
        System.out.println("생성자 호출");
    }
    
    public static BadSingleton getInstance() {
        if (instance == null) {
            instance = new BadSingleton();
        }
        return instance;
    }
}

// 문제 발생
public class Problem {
    public static void main(String[] args) {
        BadSingleton s1 = BadSingleton.getInstance();
        BadSingleton s2 = new BadSingleton(); // 생성 가능! (문제!)
        
        System.out.println("s1 == s2: " + (s1 == s2)); // false
    }
}
```

**해결:**
```java
private BadSingleton() {  // private으로 변경!
    System.out.println("생성자 호출");
}
```

---

### ❌ 안티패턴 2: Serialization 미처리

```java
// 잘못된 예
public class BadSerializable implements Serializable {
    private static final BadSerializable INSTANCE = new BadSerializable();
    
    private BadSerializable() {}
    
    public static BadSerializable getInstance() {
        return INSTANCE;
    }
    
    // readResolve() 없음 → 역직렬화 시 새 인스턴스 생성!
}
```

**해결:**
```java
public class GoodSerializable implements Serializable {
    private static final GoodSerializable INSTANCE = new GoodSerializable();
    
    private GoodSerializable() {}
    
    public static GoodSerializable getInstance() {
        return INSTANCE;
    }
    
    // 역직렬화 시 동일한 인스턴스 반환
    protected Object readResolve() {
        return INSTANCE;
    }
}
```

---

### ❌ 안티패턴 3: 상속 가능한 Singleton

```java
// 잘못된 예
public class BadInheritance {
    private static BadInheritance instance;
    
    protected BadInheritance() {}  // protected → 상속 가능!
    
    public static BadInheritance getInstance() {
        if (instance == null) {
            instance = new BadInheritance();
        }
        return instance;
    }
}

// 문제 발생
class SubSingleton extends BadInheritance {
    // 새로운 Singleton 생성 가능 (문제!)
}
```

**해결:**
```java
public final class GoodSingleton {  // final로 상속 차단
    private static GoodSingleton instance;
    
    private GoodSingleton() {}
    
    public static GoodSingleton getInstance() {
        if (instance == null) {
            instance = new GoodSingleton();
        }
        return instance;
    }
}
```

---

## 8. 핵심 정리

### 📌 Singleton 패턴 체크리스트

```
✅ private 생성자로 외부 생성 차단
✅ static 메서드로 전역 접근점 제공
✅ Thread-safe 구현 선택
✅ Lazy vs Eager 결정
✅ Serialization 고려 (필요 시)
✅ final 클래스로 상속 방지
```

### 🎯 구현 방법 선택 가이드

| 상황 | 추천 방법 | 이유 |
|------|-----------|------|
| **일반적인 경우** | **Enum** | 가장 안전하고 간결 |
| **복잡한 초기화** | **Bill Pugh** | Lazy + Thread-safe |
| **즉시 초기화** | **Eager** | 단순하고 안전 |
| **레거시 코드** | **DCL** | 기존 코드 유지 |

### 💡 핵심 포인트

1. **private 생성자는 필수!**
2. **Thread-safe 고려 필수!**
3. **Enum이 가장 안전!**
4. **Bill Pugh가 실무에서 많이 사용됨**
5. **전역 상태는 신중하게 사용**

### 🔥 실무 팁

```java
// 1. 설정 관리 → Enum
public enum Config { INSTANCE; }

// 2. 리소스 풀 → DCL or Bill Pugh
public class ConnectionPool { /* DCL */ }

// 3. 상태를 가진 객체 → 가급적 피하기
// Singleton보다 DI(Dependency Injection) 고려
```

---

## 🎓 연습 문제

### 문제 1: Thread-safe Singleton 구현
```java
/**
 * 요구사항:
 * 1. Bill Pugh 방식으로 구현
 * 2. 카운터 기능 추가 (increment, getCount)
 * 3. 멀티스레드 환경에서 안전하게 동작
 */
public class Counter {
    // 여기에 구현
}
```

### 문제 2: 캐시 시스템 구현
```java
/**
 * 요구사항:
 * 1. Enum으로 구현
 * 2. put(key, value), get(key) 메서드
 * 3. 최대 100개 저장, LRU 방식
 */
public enum Cache {
    // 여기에 구현
}
```

### 문제 3: 문제점 찾기
```java
// 이 코드의 문제점을 찾고 수정하세요
public class Database {
    private static Database instance;
    
    public Database() {}
    
    public Database getInstance() {
        if (instance == null) {
            instance = new Database();
        }
        return instance;
    }
}
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[다음: Factory Method →](./02-FactoryMethod.md)**

</div>
