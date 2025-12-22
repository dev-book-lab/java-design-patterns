# Double-Checked Locking Pattern (이중 확인 잠금 패턴)

> **"Singleton 초기화를 최적화하면서도 Thread-Safe를 보장하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [volatile 완전 가이드](#6-volatile-완전-가이드)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: Eager Initialization (성능 낭비)
public class DatabaseConnection {
    // 😱 프로그램 시작 시 무조건 생성!
    private static final DatabaseConnection INSTANCE = new DatabaseConnection();
    
    private DatabaseConnection() {
        // 무거운 초기화 (5초 걸림)
        connectToDatabase();
        loadConfiguration();
        initializePool();
    }
    
    public static DatabaseConnection getInstance() {
        return INSTANCE;
    }
    
    // 문제:
    // - 사용 안 해도 생성됨
    // - 시작 시간 증가
    // - 메모리 낭비
}

// 문제 2: Lazy Initialization (Thread-Safe 아님)
public class ConfigManager {
    private static ConfigManager instance;
    
    // 😱 Thread-Safe 아님!
    public static ConfigManager getInstance() {
        if (instance == null) {
            // 여러 스레드가 동시에 이 코드 실행 가능!
            instance = new ConfigManager();
        }
        return instance;
    }
    
    // 시나리오:
    // Thread 1: if (instance == null) → true
    // Thread 2: if (instance == null) → true (동시!)
    // Thread 1: new ConfigManager() → 인스턴스 1
    // Thread 2: new ConfigManager() → 인스턴스 2 (중복!)
}

// 문제 3: Synchronized (성능 저하)
public class CacheManager {
    private static CacheManager instance;
    
    // 😱 매번 Lock!
    public static synchronized CacheManager getInstance() {
        if (instance == null) {
            instance = new CacheManager();
        }
        return instance;
    }
    
    // 문제:
    // - 초기화 후에도 매번 synchronized
    // - 불필요한 Lock (성능 저하)
    // - 1000번 호출 = 1000번 Lock
    // - 실제 필요한 Lock은 첫 번째뿐!
}

// 성능 비교:
// Eager:        프로그램 시작 시 5초 지연
// Lazy (Unsafe): 빠르지만 Thread-Safe 아님
// Synchronized:  매번 Lock (느림)

// 문제 4: 잘못된 Double-Checked Locking (Java 5 이전)
public class ResourceManager {
    private static ResourceManager instance;
    
    public static ResourceManager getInstance() {
        // 😱 volatile 없이는 동작 안 함!
        if (instance == null) {
            synchronized (ResourceManager.class) {
                if (instance == null) {
                    instance = new ResourceManager();
                    // 문제: 재배치(Reordering) 발생 가능!
                    // 1. 메모리 할당
                    // 2. instance 참조 설정 (완성 전!)
                    // 3. 생성자 실행
                    // → 다른 스레드가 미완성 객체 접근!
                }
            }
        }
        return instance;
    }
}

// 문제 5: 복잡한 초기화
public class HeavyResource {
    private static HeavyResource instance;
    
    public static synchronized HeavyResource getInstance() {
        if (instance == null) {
            // 😱 복잡한 초기화가 Lock 안에!
            instance = new HeavyResource();
            instance.loadLargeFile();        // 10초
            instance.connectToServices();    // 5초
            instance.buildCache();           // 8초
        }
        return instance;
        
        // 첫 번째 호출: 23초 동안 다른 스레드 Blocking!
    }
}
```

### ⚡ 핵심 문제

1. **Eager**: 사용 안 해도 생성 (메모리 낭비)
2. **Lazy (Unsafe)**: Thread-Safe 아님 (중복 생성)
3. **Synchronized**: 매번 Lock (성능 저하)
4. **Reordering**: 미완성 객체 접근 (버그)
5. **Blocking**: 긴 초기화 동안 대기

---

## 2. 패턴 정의

### 📖 정의

> **Lazy Initialization + Thread-Safety + 성능 최적화를 모두 만족하는 Singleton 구현 패턴**

### 🎯 목적

- **Lazy**: 사용 시점에 생성
- **Thread-Safe**: 중복 생성 방지
- **성능**: 초기화 후 Lock 없음
- **안전성**: volatile로 Reordering 방지

### 💡 핵심 아이디어

```java
// Before: Synchronized (느림)
public static synchronized Singleton getInstance() {
    if (instance == null) {
        instance = new Singleton();
    }
    return instance;  // 매번 Lock!
}

// After: Double-Checked Locking (빠름)
public static Singleton getInstance() {
    if (instance == null) {  // 1차 체크 (Lock 없음)
        synchronized (Singleton.class) {
            if (instance == null) {  // 2차 체크 (Lock 안에서)
                instance = new Singleton();
            }
        }
    }
    return instance;  // 초기화 후 Lock 없음!
}
```

---

## 3. 구조와 구성요소

### 📊 Double-Checked Locking 구조

```
getInstance() 호출
    ↓
1차 체크: instance == null?
    ↓ No (이미 초기화됨)
    return instance  (Lock 없음! 빠름!)
    
    ↓ Yes (아직 초기화 안 됨)
synchronized 진입
    ↓
2차 체크: instance == null?
    ↓ No (다른 스레드가 초기화함)
    return instance
    
    ↓ Yes (초기화 필요)
instance = new Singleton()
    ↓
return instance
```

### 🔄 동시 접근 시나리오

```
Thread 1:
1차 체크 → null
synchronized 진입
2차 체크 → null
객체 생성 ✅
synchronized 해제

Thread 2:
1차 체크 → null (동시!)
synchronized 대기...
synchronized 진입
2차 체크 → NOT null (Thread 1이 생성함)
return ✅

Thread 3:
1차 체크 → NOT null (이미 초기화됨)
return ✅ (Lock 없음!)
```

---

## 4. 구현 방법

### 완전한 구현: Connection Pool Manager ⭐⭐⭐

```java
/**
 * ============================================
 * EVOLUTION OF SINGLETON
 * ============================================
 */

/**
 * 1. Eager Initialization (간단하지만 비효율)
 */
class EagerSingleton {
    // 클래스 로딩 시 생성
    private static final EagerSingleton INSTANCE = new EagerSingleton();
    
    private EagerSingleton() {
        System.out.println("🏗️ EagerSingleton 생성 (프로그램 시작 시)");
    }
    
    public static EagerSingleton getInstance() {
        return INSTANCE;
    }
}

/**
 * 2. Lazy Initialization (Thread-Safe 아님)
 */
class UnsafeLazySingleton {
    private static UnsafeLazySingleton instance;
    
    private UnsafeLazySingleton() {
        System.out.println("🏗️ UnsafeLazySingleton 생성");
    }
    
    // ❌ Thread-Safe 아님!
    public static UnsafeLazySingleton getInstance() {
        if (instance == null) {
            instance = new UnsafeLazySingleton();
        }
        return instance;
    }
}

/**
 * 3. Synchronized (Thread-Safe하지만 느림)
 */
class SynchronizedSingleton {
    private static SynchronizedSingleton instance;
    
    private SynchronizedSingleton() {
        System.out.println("🏗️ SynchronizedSingleton 생성");
    }
    
    // 😱 매번 Lock
    public static synchronized SynchronizedSingleton getInstance() {
        if (instance == null) {
            instance = new SynchronizedSingleton();
        }
        return instance;
    }
}

/**
 * 4. Double-Checked Locking (최적!)
 */
class DoubleCheckedSingleton {
    // ✅ volatile 필수!
    private static volatile DoubleCheckedSingleton instance;
    
    private DoubleCheckedSingleton() {
        System.out.println("🏗️ DoubleCheckedSingleton 생성");
        
        // 무거운 초기화
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    public static DoubleCheckedSingleton getInstance() {
        // 1차 체크 (Lock 없음)
        if (instance == null) {
            synchronized (DoubleCheckedSingleton.class) {
                // 2차 체크 (Lock 안에서)
                if (instance == null) {
                    System.out.println("🔨 새 인스턴스 생성 중...");
                    instance = new DoubleCheckedSingleton();
                }
            }
        }
        return instance;
    }
}

/**
 * 5. Bill Pugh Singleton (추천!)
 */
class BillPughSingleton {
    private BillPughSingleton() {
        System.out.println("🏗️ BillPughSingleton 생성");
    }
    
    // Static Inner Class (Lazy + Thread-Safe)
    private static class SingletonHolder {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }
    
    public static BillPughSingleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}

/**
 * ============================================
 * COMPLETE EXAMPLE: CONNECTION POOL
 * ============================================
 */

/**
 * Connection Pool Manager
 */
class ConnectionPoolManager {
    private static volatile ConnectionPoolManager instance;
    
    private final int poolSize;
    private final List<Connection> connections;
    private final LocalDateTime createdAt;
    
    private ConnectionPoolManager() {
        System.out.println("🔧 ConnectionPoolManager 초기화 시작...");
        
        this.poolSize = 10;
        this.connections = new ArrayList<>();
        this.createdAt = LocalDateTime.now();
        
        // 무거운 초기화 (5초)
        initializeConnections();
        
        System.out.println("✅ ConnectionPoolManager 초기화 완료");
    }
    
    private void initializeConnections() {
        try {
            for (int i = 0; i < poolSize; i++) {
                Thread.sleep(500);  // 연결 시뮬레이션
                connections.add(new Connection("conn-" + i));
                System.out.println("   📡 Connection " + i + " 생성");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    /**
     * Double-Checked Locking
     */
    public static ConnectionPoolManager getInstance() {
        // 1차 체크 (빠른 경로)
        if (instance == null) {
            System.out.println("⚠️ [" + Thread.currentThread().getName() + 
                              "] 인스턴스 없음, Lock 진입");
            
            synchronized (ConnectionPoolManager.class) {
                // 2차 체크
                if (instance == null) {
                    System.out.println("🔨 [" + Thread.currentThread().getName() + 
                                      "] 인스턴스 생성 시작");
                    instance = new ConnectionPoolManager();
                } else {
                    System.out.println("⏭️ [" + Thread.currentThread().getName() + 
                                      "] 다른 스레드가 이미 생성함");
                }
            }
        }
        return instance;
    }
    
    public Connection getConnection() {
        if (connections.isEmpty()) {
            throw new RuntimeException("No available connections");
        }
        return connections.get(0);
    }
    
    public void printInfo() {
        System.out.println("\n📊 Connection Pool 정보:");
        System.out.println("   Pool Size: " + poolSize);
        System.out.println("   생성 시각: " + createdAt.format(DateTimeFormatter.ofPattern("HH:mm:ss")));
        System.out.println("   스레드: " + Thread.currentThread().getName());
    }
}

/**
 * Connection 클래스 (시뮬레이션)
 */
class Connection {
    private final String id;
    
    public Connection(String id) {
        this.id = id;
    }
    
    public String getId() {
        return id;
    }
}

/**
 * ============================================
 * PERFORMANCE COMPARISON
 * ============================================
 */

/**
 * 성능 비교 테스트
 */
class PerformanceTest {
    
    /**
     * Synchronized Singleton
     */
    static class SyncSingleton {
        private static SyncSingleton instance;
        
        private SyncSingleton() {}
        
        public static synchronized SyncSingleton getInstance() {
            if (instance == null) {
                instance = new SyncSingleton();
            }
            return instance;
        }
    }
    
    /**
     * Double-Checked Locking Singleton
     */
    static class DCLSingleton {
        private static volatile DCLSingleton instance;
        
        private DCLSingleton() {}
        
        public static DCLSingleton getInstance() {
            if (instance == null) {
                synchronized (DCLSingleton.class) {
                    if (instance == null) {
                        instance = new DCLSingleton();
                    }
                }
            }
            return instance;
        }
    }
    
    /**
     * 성능 측정
     */
    public static void runPerformanceTest() throws InterruptedException {
        System.out.println("\n=== 성능 비교 테스트 ===");
        
        int threadCount = 100;
        int iterations = 10000;
        
        // 1. Synchronized
        long syncStart = System.nanoTime();
        
        Thread[] syncThreads = new Thread[threadCount];
        for (int i = 0; i < threadCount; i++) {
            syncThreads[i] = new Thread(() -> {
                for (int j = 0; j < iterations; j++) {
                    SyncSingleton.getInstance();
                }
            });
            syncThreads[i].start();
        }
        
        for (Thread thread : syncThreads) {
            thread.join();
        }
        
        long syncTime = System.nanoTime() - syncStart;
        
        // 2. Double-Checked Locking
        long dclStart = System.nanoTime();
        
        Thread[] dclThreads = new Thread[threadCount];
        for (int i = 0; i < threadCount; i++) {
            dclThreads[i] = new Thread(() -> {
                for (int j = 0; j < iterations; j++) {
                    DCLSingleton.getInstance();
                }
            });
            dclThreads[i].start();
        }
        
        for (Thread thread : dclThreads) {
            thread.join();
        }
        
        long dclTime = System.nanoTime() - dclStart;
        
        // 결과 출력
        System.out.println("\n📊 성능 결과 (100 스레드 × 10,000 호출):");
        System.out.println("   Synchronized:          " + (syncTime / 1_000_000) + "ms");
        System.out.println("   Double-Checked Locking: " + (dclTime / 1_000_000) + "ms");
        System.out.println("   개선율: " + ((syncTime - dclTime) * 100 / syncTime) + "%");
    }
}

/**
 * ============================================
 * VOLATILE DEMONSTRATION
 * ============================================
 */

/**
 * volatile 없이는 문제 발생!
 */
class VolatileDemo {
    
    /**
     * volatile 없음 (문제 가능!)
     */
    static class WithoutVolatile {
        private static WithoutVolatile instance;  // ❌ volatile 없음
        
        public static WithoutVolatile getInstance() {
            if (instance == null) {
                synchronized (WithoutVolatile.class) {
                    if (instance == null) {
                        instance = new WithoutVolatile();
                        // Reordering 발생 가능:
                        // 1. 메모리 할당
                        // 2. instance = 할당된 메모리 (생성자 실행 전!)
                        // 3. 생성자 실행
                        // → 다른 스레드가 미완성 객체 접근!
                    }
                }
            }
            return instance;
        }
    }
    
    /**
     * volatile 있음 (안전!)
     */
    static class WithVolatile {
        private static volatile WithVolatile instance;  // ✅ volatile
        
        public static WithVolatile getInstance() {
            if (instance == null) {
                synchronized (WithVolatile.class) {
                    if (instance == null) {
                        instance = new WithVolatile();
                        // volatile이 Reordering 방지:
                        // 1. 메모리 할당
                        // 2. 생성자 실행
                        // 3. instance = 완성된 객체 ✅
                    }
                }
            }
            return instance;
        }
    }
}

/**
 * ============================================
 * DEMO
 * ============================================
 */
public class DoubleCheckedLockingDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Double-Checked Locking Pattern 예제 ===");
        
        // 1. 동시 접근 테스트
        System.out.println("\n" + "=".repeat(60));
        System.out.println("1️⃣ 동시 접근 테스트");
        System.out.println("=".repeat(60));
        
        // 10개 스레드가 동시에 getInstance() 호출
        Thread[] threads = new Thread[10];
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(() -> {
                ConnectionPoolManager manager = ConnectionPoolManager.getInstance();
                manager.printInfo();
            }, "Thread-" + i);
        }
        
        // 동시 시작
        for (Thread thread : threads) {
            thread.start();
        }
        
        for (Thread thread : threads) {
            thread.join();
        }
        
        // 2. 성능 비교
        System.out.println("\n" + "=".repeat(60));
        System.out.println("2️⃣ 성능 비교");
        System.out.println("=".repeat(60));
        
        PerformanceTest.runPerformanceTest();
        
        // 3. Singleton 종류별 비교
        System.out.println("\n" + "=".repeat(60));
        System.out.println("3️⃣ Singleton 구현 비교");
        System.out.println("=".repeat(60));
        
        System.out.println("\n1. Eager Initialization:");
        EagerSingleton.getInstance();
        
        System.out.println("\n2. Lazy (Unsafe):");
        UnsafeLazySingleton.getInstance();
        
        System.out.println("\n3. Synchronized:");
        SynchronizedSingleton.getInstance();
        
        System.out.println("\n4. Double-Checked Locking:");
        DoubleCheckedSingleton.getInstance();
        
        System.out.println("\n5. Bill Pugh:");
        BillPughSingleton.getInstance();
        
        System.out.println("\n✅ 모든 예제 완료!");
    }
}
```

**실행 결과:**
```
=== Double-Checked Locking Pattern 예제 ===

============================================================
1️⃣ 동시 접근 테스트
============================================================
⚠️ [Thread-0] 인스턴스 없음, Lock 진입
🔨 [Thread-0] 인스턴스 생성 시작
🔧 ConnectionPoolManager 초기화 시작...
   📡 Connection 0 생성
   📡 Connection 1 생성
   ...
   📡 Connection 9 생성
✅ ConnectionPoolManager 초기화 완료

⚠️ [Thread-1] 인스턴스 없음, Lock 진입
⏭️ [Thread-1] 다른 스레드가 이미 생성함

📊 Connection Pool 정보:
   Pool Size: 10
   생성 시각: 12:34:56
   스레드: Thread-0

============================================================
2️⃣ 성능 비교
============================================================

=== 성능 비교 테스트 ===

📊 성능 결과 (100 스레드 × 10,000 호출):
   Synchronized:          523ms
   Double-Checked Locking: 87ms
   개선율: 83%

============================================================
3️⃣ Singleton 구현 비교
============================================================

1. Eager Initialization:
🏗️ EagerSingleton 생성 (프로그램 시작 시)

2. Lazy (Unsafe):
🏗️ UnsafeLazySingleton 생성

3. Synchronized:
🏗️ SynchronizedSingleton 생성

4. Double-Checked Locking:
⚠️ [main] 인스턴스 없음, Lock 진입
🔨 [main] 인스턴스 생성 시작
🏗️ DoubleCheckedSingleton 생성

5. Bill Pugh:
🏗️ BillPughSingleton 생성

✅ 모든 예제 완료!
```

---

## 5. 실전 예제

### 예제 1: 설정 관리자 ⭐⭐⭐

```java
public class ConfigurationManager {
    private static volatile ConfigurationManager instance;
    private final Properties config;
    
    private ConfigurationManager() {
        config = new Properties();
        loadConfiguration();
    }
    
    public static ConfigurationManager getInstance() {
        if (instance == null) {
            synchronized (ConfigurationManager.class) {
                if (instance == null) {
                    instance = new ConfigurationManager();
                }
            }
        }
        return instance;
    }
    
    public String get(String key) {
        return config.getProperty(key);
    }
}
```

---

## 6. volatile 완전 가이드

### 🔄 volatile의 역할

```
1. 가시성 (Visibility):
   - 변수를 메인 메모리에서 읽기/쓰기
   - CPU 캐시 우회
   
2. 재배치 방지 (Reordering):
   - volatile 쓰기 이전 코드는 이후로 이동 불가
   - volatile 읽기 이후 코드는 이전으로 이동 불가
```

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 |
|------|------|
| **Lazy** | 사용 시점 생성 |
| **Thread-Safe** | 중복 방지 |
| **성능** | 초기화 후 Lock 없음 |

### ❌ 단점

| 단점 | 설명 |
|------|------|
| **복잡도** | 이해 어려움 |
| **Java 5+** | volatile 필요 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: volatile 빠뜨림

```java
// 잘못된 예
private static Instance instance;  // ❌ volatile 없음

// 올바른 예
private static volatile Instance instance;  // ✅
```

---

## 9. 심화 주제

### 🎯 대안: Bill Pugh (추천!)

```java
public class Singleton {
    private Singleton() {}
    
    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
    }
    
    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

---

## 10. 핵심 정리

### 📌 체크리스트

```
✅ volatile 키워드 필수
✅ 1차 체크 (Lock 없음)
✅ synchronized 블록
✅ 2차 체크 (Lock 안에서)
✅ Bill Pugh 고려
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Reader-Writer Lock](./03-ReaderWriterLock.md) | [다음: Active Object →](./05-ActiveObject.md)**

</div>
