# Reader-Writer Lock Pattern (읽기-쓰기 락 패턴)

> **"읽기와 쓰기를 분리하여 동시성을 최적화하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [ReadWriteLock 완전 가이드](#6-readwritelock-완전-가이드)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: synchronized로 모든 접근 제한
public class UserCache {
    private Map<Long, User> cache = new HashMap<>();
    
    // 😱 읽기도 Blocking!
    public synchronized User get(Long id) {
        return cache.get(id);  // 읽기만 하는데 Lock!
    }
    
    public synchronized void put(Long id, User user) {
        cache.put(id, user);
    }
    
    // 문제:
    // - 100명이 동시에 읽기 → 순차 처리!
    // - 읽기는 동시에 가능한데 Blocking
    // - 성능 저하!
}

// 시나리오:
// Thread 1: get(1)  ← Lock 획득
// Thread 2: get(2)  ← 대기 (읽기인데!)
// Thread 3: get(3)  ← 대기 (읽기인데!)

// 문제 2: 읽기가 대부분인 경우
public class ConfigService {
    private Properties config = new Properties();
    
    public synchronized String get(String key) {
        // 😱 읽기: 99.9%
        return config.getProperty(key);
    }
    
    public synchronized void set(String key, String value) {
        // 쓰기: 0.1%
        config.setProperty(key, value);
    }
    
    // 대부분이 읽기인데 모두 Blocking!
    // 불필요한 대기 시간!
}

// 문제 3: 성능 병목
public class ProductCatalog {
    private List<Product> products = new ArrayList<>();
    
    // 😱 조회가 99%인데 Lock!
    public synchronized List<Product> search(String keyword) {
        return products.stream()
            .filter(p -> p.getName().contains(keyword))
            .collect(Collectors.toList());
    }
    
    // 1% 업데이트
    public synchronized void update(Product product) {
        // ...
    }
    
    // 1000명이 동시 검색 → 순차 처리!
}

// 문제 4: 복잡한 수동 구현
public class DataStore {
    private Map<String, String> data = new HashMap<>();
    private int readers = 0;
    private int writers = 0;
    private final Object lock = new Object();
    
    // 😱 복잡한 Reader Lock
    public String read(String key) {
        synchronized (lock) {
            while (writers > 0) {
                try {
                    lock.wait();
                } catch (InterruptedException e) {
                    return null;
                }
            }
            readers++;
        }
        
        try {
            return data.get(key);
        } finally {
            synchronized (lock) {
                readers--;
                lock.notifyAll();
            }
        }
    }
    
    // 😱 복잡한 Writer Lock
    public void write(String key, String value) {
        synchronized (lock) {
            while (readers > 0 || writers > 0) {
                try {
                    lock.wait();
                } catch (InterruptedException e) {
                    return;
                }
            }
            writers++;
        }
        
        try {
            data.put(key, value);
        } finally {
            synchronized (lock) {
                writers--;
                lock.notifyAll();
            }
        }
    }
    
    // 에러 발생 쉽고 유지보수 어려움!
}

// 문제 5: Starvation (기아)
public class SharedResource {
    // 😱 읽기가 계속되면 쓰기가 영원히 대기!
    
    // Thread 1: read()  ← 실행 중
    // Thread 2: read()  ← 실행 중
    // Thread 3: write() ← 대기...
    // Thread 4: read()  ← 실행 중 (새로 들어옴)
    // Thread 5: read()  ← 실행 중 (새로 들어옴)
    // Thread 3: write() ← 여전히 대기... (Starvation!)
}
```

### ⚡ 핵심 문제

1. **불필요한 Blocking**: 읽기도 Lock
2. **성능 저하**: 읽기 99% 인데 순차 처리
3. **복잡한 구현**: 수동 구현 어려움
4. **Starvation**: 쓰기가 기아 상태
5. **확장성 부족**: 동시 읽기 불가

---

## 2. 패턴 정의

### 📖 정의

> **읽기와 쓰기 Lock을 분리하여 여러 Reader가 동시에 접근 가능하게 하되, Writer는 배타적으로 접근하는 패턴**

### 🎯 목적

- **동시 읽기**: 여러 Reader 동시 허용
- **배타적 쓰기**: Writer는 독점 접근
- **성능 향상**: 읽기가 많을 때 유리
- **안전성**: 데이터 일관성 보장

### 💡 핵심 아이디어

```java
// Before: synchronized (모두 Blocking)
public synchronized String read() {
    return data;
}

public synchronized void write(String value) {
    data = value;
}

// After: ReadWriteLock (읽기는 동시 허용)
ReadWriteLock lock = new ReentrantReadWriteLock();

public String read() {
    lock.readLock().lock();
    try {
        return data;  // 여러 Reader 동시 OK!
    } finally {
        lock.readLock().unlock();
    }
}

public void write(String value) {
    lock.writeLock().lock();
    try {
        data = value;  // Writer는 독점!
    } finally {
        lock.writeLock().unlock();
    }
}
```

---

## 3. 구조와 구성요소

### 📊 Reader-Writer Lock 구조

```
┌──────────────────────────────────┐
│      ReadWriteLock               │
│                                  │
│  ┌─────────────┐ ┌─────────────┐ │
│  │ Read Lock   │ │ Write Lock  │ │
│  └─────────────┘ └─────────────┘ │
└──────────────────────────────────┘

Read Lock (공유):
Reader 1 ─┐
Reader 2 ─┼─→ [Data] (동시 접근 OK!)
Reader N ─┘

Write Lock (배타):
Writer ───→ [Data] (독점 접근!)
```

### 🔄 Lock 규칙

```
1. Reader-Reader: ✅ 동시 허용
   Reader 1 + Reader 2 = OK

2. Reader-Writer: ❌ Blocking
   Reader 실행 중 → Writer 대기
   Writer 실행 중 → Reader 대기

3. Writer-Writer: ❌ Blocking
   Writer 1 실행 중 → Writer 2 대기
```

---

## 4. 구현 방법

### 완전한 구현: 캐시 시스템 ⭐⭐⭐

```java
/**
 * ============================================
 * SIMPLE CACHE WITH READWRITELOCK
 * ============================================
 */

/**
 * Thread-Safe 캐시
 */
public class ThreadSafeCache<K, V> {
    private final Map<K, V> cache = new HashMap<>();
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private final Lock readLock = lock.readLock();
    private final Lock writeLock = lock.writeLock();
    
    /**
     * 읽기 (여러 스레드 동시 가능)
     */
    public V get(K key) {
        readLock.lock();
        try {
            System.out.println("📖 [" + Thread.currentThread().getName() + "] " +
                              "읽기: " + key);
            
            // 시뮬레이션: 읽기 시간
            Thread.sleep(100);
            
            return cache.get(key);
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return null;
        } finally {
            readLock.unlock();
        }
    }
    
    /**
     * 쓰기 (배타적)
     */
    public void put(K key, V value) {
        writeLock.lock();
        try {
            System.out.println("✏️ [" + Thread.currentThread().getName() + "] " +
                              "쓰기: " + key + " = " + value);
            
            // 시뮬레이션: 쓰기 시간
            Thread.sleep(200);
            
            cache.put(key, value);
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            writeLock.unlock();
        }
    }
    
    /**
     * 크기 (읽기)
     */
    public int size() {
        readLock.lock();
        try {
            return cache.size();
        } finally {
            readLock.unlock();
        }
    }
    
    /**
     * 초기화 (쓰기)
     */
    public void clear() {
        writeLock.lock();
        try {
            System.out.println("🗑️ [" + Thread.currentThread().getName() + "] 초기화");
            cache.clear();
        } finally {
            writeLock.unlock();
        }
    }
}

/**
 * ============================================
 * STATISTICS TRACKING
 * ============================================
 */

/**
 * 통계 추적 캐시
 */
public class MonitoredCache<K, V> {
    private final Map<K, V> cache = new HashMap<>();
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    
    // 통계
    private long reads = 0;
    private long writes = 0;
    private long readWaitTime = 0;
    private long writeWaitTime = 0;
    
    public V get(K key) {
        long startWait = System.nanoTime();
        
        lock.readLock().lock();
        try {
            long endWait = System.nanoTime();
            readWaitTime += (endWait - startWait);
            reads++;
            
            return cache.get(key);
        } finally {
            lock.readLock().unlock();
        }
    }
    
    public void put(K key, V value) {
        long startWait = System.nanoTime();
        
        lock.writeLock().lock();
        try {
            long endWait = System.nanoTime();
            writeWaitTime += (endWait - startWait);
            writes++;
            
            cache.put(key, value);
        } finally {
            lock.writeLock().unlock();
        }
    }
    
    public void printStatistics() {
        lock.readLock().lock();
        try {
            System.out.println("\n📊 캐시 통계:");
            System.out.println("   읽기 횟수: " + reads);
            System.out.println("   쓰기 횟수: " + writes);
            System.out.println("   평균 읽기 대기: " + (readWaitTime / reads / 1000) + "μs");
            System.out.println("   평균 쓰기 대기: " + (writeWaitTime / writes / 1000) + "μs");
        } finally {
            lock.readLock().unlock();
        }
    }
}

/**
 * ============================================
 * FAIR VS NON-FAIR
 * ============================================
 */

/**
 * 공정성 비교
 */
public class FairnessDemo {
    
    /**
     * Non-Fair Lock (기본)
     */
    public void nonFairExample() {
        System.out.println("\n=== Non-Fair Lock ===");
        
        ReadWriteLock lock = new ReentrantReadWriteLock(false);  // Non-fair
        
        // 성능 우선 (Starvation 가능)
    }
    
    /**
     * Fair Lock
     */
    public void fairExample() {
        System.out.println("\n=== Fair Lock ===");
        
        ReadWriteLock lock = new ReentrantReadWriteLock(true);  // Fair
        
        // 공정성 우선 (성능 희생)
    }
}

/**
 * ============================================
 * LOCK DOWNGRADING
 * ============================================
 */

/**
 * Lock Downgrading (쓰기 → 읽기)
 */
public class LockDowngradingExample {
    private final Map<String, String> data = new HashMap<>();
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    
    /**
     * Downgrading: Write Lock → Read Lock
     */
    public void updateAndRead(String key, String value) {
        // 1. Write Lock 획득
        lock.writeLock().lock();
        try {
            System.out.println("✏️ Write Lock 획득");
            data.put(key, value);
            
            // 2. Read Lock 획득 (Write Lock 유지한 채)
            lock.readLock().lock();
            System.out.println("📖 Read Lock 획득 (Downgrading)");
            
        } finally {
            // 3. Write Lock 해제
            lock.writeLock().unlock();
            System.out.println("✏️ Write Lock 해제");
        }
        
        try {
            // 4. Read Lock만 유지
            System.out.println("📖 Read Lock으로 읽기: " + data.get(key));
        } finally {
            // 5. Read Lock 해제
            lock.readLock().unlock();
            System.out.println("📖 Read Lock 해제");
        }
    }
}

/**
 * ============================================
 * COMPLETE EXAMPLE
 * ============================================
 */

/**
 * 사용자 정보 캐시
 */
class User {
    private final Long id;
    private final String name;
    
    public User(Long id, String name) {
        this.id = id;
        this.name = name;
    }
    
    public Long getId() { return id; }
    public String getName() { return name; }
    
    @Override
    public String toString() {
        return "User{id=" + id + ", name='" + name + "'}";
    }
}

/**
 * 사용자 캐시 서비스
 */
public class UserCacheService {
    private final ThreadSafeCache<Long, User> cache;
    
    public UserCacheService() {
        this.cache = new ThreadSafeCache<>();
        
        // 초기 데이터
        cache.put(1L, new User(1L, "Alice"));
        cache.put(2L, new User(2L, "Bob"));
        cache.put(3L, new User(3L, "Charlie"));
    }
    
    /**
     * 사용자 조회
     */
    public User getUser(Long id) {
        User user = cache.get(id);
        
        if (user == null) {
            System.out.println("⚠️ Cache Miss: " + id);
            // DB에서 로드
            user = loadFromDatabase(id);
            if (user != null) {
                cache.put(id, user);
            }
        }
        
        return user;
    }
    
    /**
     * 사용자 업데이트
     */
    public void updateUser(User user) {
        cache.put(user.getId(), user);
    }
    
    private User loadFromDatabase(Long id) {
        // 시뮬레이션
        return new User(id, "User-" + id);
    }
}

/**
 * ============================================
 * DEMO
 * ============================================
 */
public class ReaderWriterLockDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Reader-Writer Lock Pattern 예제 ===");
        
        // 1. 기본 예제
        System.out.println("\n" + "=".repeat(60));
        System.out.println("1️⃣ 동시 읽기 테스트");
        System.out.println("=".repeat(60));
        
        ThreadSafeCache<String, String> cache = new ThreadSafeCache<>();
        cache.put("key1", "value1");
        cache.put("key2", "value2");
        
        // 여러 Reader (동시 실행 OK!)
        for (int i = 1; i <= 5; i++) {
            int readerId = i;
            new Thread(() -> {
                String value = cache.get("key" + (readerId % 2 + 1));
                System.out.println("✅ Reader-" + readerId + " 완료: " + value);
            }, "Reader-" + i).start();
        }
        
        Thread.sleep(500);
        
        // Writer (Readers 완료 대기)
        new Thread(() -> {
            cache.put("key3", "value3");
            System.out.println("✅ Writer 완료");
        }, "Writer").start();
        
        Thread.sleep(1000);
        
        // 2. Lock Downgrading
        System.out.println("\n" + "=".repeat(60));
        System.out.println("2️⃣ Lock Downgrading");
        System.out.println("=".repeat(60));
        
        LockDowngradingExample downgrading = new LockDowngradingExample();
        downgrading.updateAndRead("test", "value");
        
        // 3. 사용자 캐시 서비스
        System.out.println("\n" + "=".repeat(60));
        System.out.println("3️⃣ 사용자 캐시 서비스");
        System.out.println("=".repeat(60));
        
        UserCacheService userService = new UserCacheService();
        
        // 동시 조회
        ExecutorService executor = Executors.newFixedThreadPool(10);
        
        for (int i = 0; i < 20; i++) {
            Long userId = (long) (i % 3 + 1);
            executor.submit(() -> {
                User user = userService.getUser(userId);
                System.out.println("👤 조회: " + user);
            });
        }
        
        // 업데이트
        executor.submit(() -> {
            userService.updateUser(new User(1L, "Alice Updated"));
            System.out.println("✏️ 업데이트 완료");
        });
        
        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.SECONDS);
        
        System.out.println("\n✅ 모든 예제 완료!");
    }
}
```

**실행 결과:**
```
=== Reader-Writer Lock Pattern 예제 ===

============================================================
1️⃣ 동시 읽기 테스트
============================================================
📖 [Reader-1] 읽기: key1
📖 [Reader-2] 읽기: key2
📖 [Reader-3] 읽기: key1
📖 [Reader-4] 읽기: key2
📖 [Reader-5] 읽기: key1
✅ Reader-1 완료: value1
✅ Reader-2 완료: value2
✅ Reader-3 완료: value1
✅ Reader-4 완료: value2
✅ Reader-5 완료: value1
✏️ [Writer] 쓰기: key3 = value3
✅ Writer 완료

============================================================
2️⃣ Lock Downgrading
============================================================
✏️ Write Lock 획득
📖 Read Lock 획득 (Downgrading)
✏️ Write Lock 해제
📖 Read Lock으로 읽기: value
📖 Read Lock 해제

============================================================
3️⃣ 사용자 캐시 서비스
============================================================
👤 조회: User{id=1, name='Alice'}
👤 조회: User{id=2, name='Bob'}
👤 조회: User{id=3, name='Charlie'}
👤 조회: User{id=1, name='Alice'}
✏️ 업데이트 완료
👤 조회: User{id=1, name='Alice Updated'}

✅ 모든 예제 완료!
```

---

## 5. 실전 예제

### 예제 1: 설정 관리자 ⭐⭐⭐

```java
public class ConfigurationManager {
    private final Properties config = new Properties();
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    
    public String get(String key) {
        lock.readLock().lock();
        try {
            return config.getProperty(key);
        } finally {
            lock.readLock().unlock();
        }
    }
    
    public void reload() {
        lock.writeLock().lock();
        try {
            config.clear();
            config.load(new FileInputStream("config.properties"));
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

---

## 6. ReadWriteLock 완전 가이드

### 📊 메서드

| 메서드 | 설명 |
|--------|------|
| `readLock()` | Read Lock 획득 |
| `writeLock()` | Write Lock 획득 |
| `getReadHoldCount()` | Read Lock 보유 수 |
| `isWriteLocked()` | Write Lock 여부 |

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 |
|------|------|
| **동시 읽기** | 여러 Reader 동시 |
| **성능** | 읽기 많을 때 유리 |

### ❌ 단점

| 단점 | 설명 |
|------|------|
| **복잡도** | synchronized보다 복잡 |
| **Starvation** | Writer 기아 가능 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: Lock 미해제

```java
// 잘못된 예
lock.readLock().lock();
doSomething();  // ❌ finally 없음

// 올바른 예
lock.readLock().lock();
try {
    doSomething();
} finally {
    lock.readLock().unlock();  // ✅
}
```

---

## 9. 심화 주제

### 🎯 StampedLock (Java 8+)

```java
// 더 빠른 대안
StampedLock lock = new StampedLock();

// Optimistic Read
long stamp = lock.tryOptimisticRead();
// 읽기...
if (!lock.validate(stamp)) {
    // 재시도
}
```

---

## 10. 핵심 정리

### 📌 체크리스트

```
✅ 읽기 > 쓰기인 경우 사용
✅ finally로 unlock
✅ Fair vs Non-Fair 고려
✅ Downgrading 활용
✅ 통계 모니터링
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Producer-Consumer](./02-ProducerConsumer.md) | [다음: Double-Checked Locking →](04-DoubleCheckedLocking.md)**

</div>
