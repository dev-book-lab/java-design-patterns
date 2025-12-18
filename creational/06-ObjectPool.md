# Object Pool Pattern (객체 풀 패턴)

> **"생성 비용이 큰 객체를 재사용하자"**

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
// 문제 1: 데이터베이스 커넥션 매번 생성
public class DatabaseService {
    public void queryData() {
        // 매번 새로운 연결 생성 (비용 큼!)
        Connection conn = DriverManager.getConnection(
            "jdbc:mysql://localhost:3306/db", "user", "pass"
        );
        
        // 쿼리 실행
        Statement stmt = conn.createStatement();
        ResultSet rs = stmt.executeQuery("SELECT * FROM users");
        
        // 연결 종료
        conn.close();
        
        // 다음 요청 시 또 생성... 반복!
    }
}

// 문제 2: 스레드 매번 생성/삭제
public class TaskProcessor {
    public void processTasks(List<Task> tasks) {
        for (Task task : tasks) {
            // 매번 새 스레드 생성 (비용 큼!)
            Thread thread = new Thread(task);
            thread.start();
            // 스레드 종료 후 버려짐...
        }
        // 1000개 작업 = 1000개 스레드 생성/삭제!
    }
}

// 문제 3: 무거운 객체 반복 생성
public class ImageProcessor {
    public void processImages(List<String> imagePaths) {
        for (String path : imagePaths) {
            // 매번 새 프로세서 생성 (초기화 비용 큼!)
            ImageEditor editor = new ImageEditor();
            editor.loadPlugins();       // 플러그인 로드 (느림)
            editor.initializeFilters(); // 필터 초기화 (느림)
            
            editor.process(path);
            
            // 사용 후 버려짐...
        }
    }
}

// 문제 4: 메모리 압박
public class GameObjectManager {
    public void spawnEnemies() {
        for (int i = 0; i < 100; i++) {
            // 100개의 적 생성
            Enemy enemy = new Enemy();
            enemy.loadModel();    // 3D 모델 로드 (메모리 큼)
            enemy.loadTexture();  // 텍스처 로드 (메모리 큼)
            enemy.loadSound();    // 사운드 로드 (메모리 큼)
            
            // 적 사망 시 메모리 해제
            // 또 생성할 때 다시 로드... GC 압박!
        }
    }
}
```

### ⚡ 핵심 문제

1. **생성 비용**: 객체 생성에 많은 시간 소요
2. **리소스 낭비**: 동일한 초기화 반복
3. **메모리 압박**: 빈번한 생성/삭제로 GC 부담
4. **성능 저하**: 시스템 리소스 부족

---

## 2. 패턴 정의

### 📖 정의

> **생성 비용이 큰 객체들을 미리 생성해두고 재사용하는 패턴. 객체를 풀(Pool)에 보관하여 필요할 때 대여하고 사용 후 반납한다.**

### 🎯 목적

- **성능 향상**: 객체 재사용으로 생성 비용 절감
- **리소스 관리**: 제한된 리소스 효율적 사용
- **메모리 최적화**: GC 부담 감소
- **응답 시간 개선**: 즉시 사용 가능한 객체 제공

### 💡 핵심 아이디어

```java
// Before: 매번 생성
Connection conn = new Connection(); // 느림!
conn.query();
conn.close();

// After: 풀에서 빌림
Connection conn = pool.acquire(); // 빠름!
conn.query();
pool.release(conn); // 반납 (재사용 가능)
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌─────────────────────┐
│    ObjectPool       │  ← 풀 관리자
├─────────────────────┤
│ - available: Queue  │  ← 사용 가능 객체
│ - inUse: Set        │  ← 사용 중 객체
├─────────────────────┤
│ + acquire(): Object │  ← 객체 대여
│ + release(Object)   │  ← 객체 반납
│ - create(): Object  │  ← 새 객체 생성
│ - validate(Object)  │  ← 유효성 검증
└─────────────────────┘
           │
           │ manages
           ▼
    ┌─────────────┐
    │   Object    │  ← 재사용될 객체
    └─────────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **ObjectPool** | 객체 풀 관리 | `ConnectionPool` |
| **Reusable Object** | 재사용될 객체 | `Connection`, `Thread` |
| **Client** | 풀 사용자 | `DatabaseService` |
| **available** | 사용 가능 객체 큐 | `Queue<Object>` |
| **inUse** | 사용 중 객체 집합 | `Set<Object>` |

---

## 4. 구현 방법

### 방법 1: 기본 Object Pool ⭐⭐⭐

```java
/**
 * 재사용 가능한 객체 인터페이스
 */
public interface Reusable {
    void reset(); // 초기화
}

/**
 * 제네릭 ObjectPool
 */
public class ObjectPool<T extends Reusable> {
    private final Queue<T> available;
    private final Set<T> inUse;
    private final int maxPoolSize;
    private final ObjectFactory<T> factory;
    
    public ObjectPool(ObjectFactory<T> factory, int maxPoolSize) {
        this.factory = factory;
        this.maxPoolSize = maxPoolSize;
        this.available = new ConcurrentLinkedQueue<>();
        this.inUse = ConcurrentHashMap.newKeySet();
        
        // 초기 객체 생성
        initializePool(maxPoolSize / 2);
    }
    
    private void initializePool(int initialSize) {
        for (int i = 0; i < initialSize; i++) {
            available.offer(factory.create());
        }
        System.out.println("✅ 풀 초기화 완료: " + initialSize + "개 객체");
    }
    
    /**
     * 객체 대여
     */
    public synchronized T acquire() {
        // 사용 가능한 객체가 있으면 반환
        T object = available.poll();
        
        if (object == null) {
            // 풀이 비었으면 새로 생성 (최대 크기 제한)
            if (inUse.size() < maxPoolSize) {
                object = factory.create();
                System.out.println("⚠️ 풀이 비어 새 객체 생성");
            } else {
                throw new RuntimeException("풀이 고갈됨! 최대: " + maxPoolSize);
            }
        }
        
        inUse.add(object);
        System.out.println("📤 객체 대여 (사용중: " + inUse.size() + 
                ", 대기: " + available.size() + ")");
        return object;
    }
    
    /**
     * 객체 반납
     */
    public synchronized void release(T object) {
        if (!inUse.remove(object)) {
            throw new IllegalArgumentException("이 객체는 풀에 속하지 않습니다");
        }
        
        // 재사용 가능하도록 초기화
        object.reset();
        
        // 사용 가능한 큐에 추가
        available.offer(object);
        System.out.println("📥 객체 반납 (사용중: " + inUse.size() + 
                ", 대기: " + available.size() + ")");
    }
    
    /**
     * 풀 정보 출력
     */
    public void printStatus() {
        System.out.println("=== 풀 상태 ===");
        System.out.println("총 크기: " + (available.size() + inUse.size()));
        System.out.println("사용 가능: " + available.size());
        System.out.println("사용 중: " + inUse.size());
    }
}

/**
 * 객체 팩토리 인터페이스
 */
public interface ObjectFactory<T> {
    T create();
}
```

---

### 방법 2: 데이터베이스 커넥션 풀 ⭐⭐⭐

```java
/**
 * 데이터베이스 커넥션 (재사용 가능)
 */
public class PooledConnection implements Reusable {
    private final int id;
    private boolean connected;
    private int queryCount;
    
    public PooledConnection(int id) {
        this.id = id;
        this.connected = false;
        this.queryCount = 0;
        
        // 무거운 초기화 작업 시뮬레이션
        try {
            Thread.sleep(100); // 실제로는 DB 연결
            this.connected = true;
            System.out.println("🔌 Connection #" + id + " 생성 완료");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    
    public void executeQuery(String sql) {
        if (!connected) {
            throw new IllegalStateException("연결되지 않음");
        }
        queryCount++;
        System.out.println("  🔍 Connection #" + id + " 쿼리 실행: " + sql);
    }
    
    @Override
    public void reset() {
        // 연결 상태 초기화
        System.out.println("  🔄 Connection #" + id + " 초기화 (쿼리 수: " + queryCount + ")");
        queryCount = 0;
    }
    
    public int getId() {
        return id;
    }
    
    public boolean isConnected() {
        return connected;
    }
}

/**
 * 커넥션 풀
 */
public class ConnectionPool extends ObjectPool<PooledConnection> {
    private static int idCounter = 0;
    
    public ConnectionPool(int maxPoolSize) {
        super(new ObjectFactory<PooledConnection>() {
            @Override
            public PooledConnection create() {
                return new PooledConnection(++idCounter);
            }
        }, maxPoolSize);
    }
}

/**
 * 사용 예제
 */
public class ConnectionPoolExample {
    public static void main(String[] args) throws InterruptedException {
        // 1. 커넥션 풀 생성 (최대 5개)
        System.out.println("### 커넥션 풀 생성 ###");
        ConnectionPool pool = new ConnectionPool(5);
        
        // 2. 순차적 사용
        System.out.println("\n### 순차적 사용 ###");
        PooledConnection conn1 = pool.acquire();
        conn1.executeQuery("SELECT * FROM users");
        pool.release(conn1);
        
        PooledConnection conn2 = pool.acquire();
        conn2.executeQuery("SELECT * FROM orders");
        pool.release(conn2);
        
        pool.printStatus();
        
        // 3. 동시 사용 (멀티스레드)
        System.out.println("\n### 멀티스레드 사용 ###");
        ExecutorService executor = Executors.newFixedThreadPool(10);
        
        for (int i = 0; i < 20; i++) {
            final int taskId = i;
            executor.submit(() -> {
                try {
                    PooledConnection conn = pool.acquire();
                    conn.executeQuery("Task " + taskId);
                    Thread.sleep(50); // 작업 시뮬레이션
                    pool.release(conn);
                } catch (Exception e) {
                    System.err.println("❌ Task " + taskId + " 실패: " + e.getMessage());
                }
            });
        }
        
        executor.shutdown();
        executor.awaitTermination(10, TimeUnit.SECONDS);
        
        System.out.println("\n### 최종 상태 ###");
        pool.printStatus();
    }
}
```

**실행 결과:**
```
### 커넥션 풀 생성 ###
🔌 Connection #1 생성 완료
🔌 Connection #2 생성 완료
✅ 풀 초기화 완료: 2개 객체

### 순차적 사용 ###
📤 객체 대여 (사용중: 1, 대기: 1)
  🔍 Connection #1 쿼리 실행: SELECT * FROM users
  🔄 Connection #1 초기화 (쿼리 수: 1)
📥 객체 반납 (사용중: 0, 대기: 2)
📤 객체 대여 (사용중: 1, 대기: 1)
  🔍 Connection #1 쿼리 실행: SELECT * FROM orders
  🔄 Connection #1 초기화 (쿼리 수: 1)
📥 객체 반납 (사용중: 0, 대기: 2)

=== 풀 상태 ===
총 크기: 2
사용 가능: 2
사용 중: 0

### 멀티스레드 사용 ###
📤 객체 대여 (사용중: 1, 대기: 1)
📤 객체 대여 (사용중: 2, 대기: 0)
⚠️ 풀이 비어 새 객체 생성
🔌 Connection #3 생성 완료
📤 객체 대여 (사용중: 3, 대기: 0)
...
```

---

## 5. 실전 예제

### 예제 1: 스레드 풀 ⭐⭐⭐

```java
/**
 * Worker 스레드
 */
public class WorkerThread extends Thread implements Reusable {
    private final int id;
    private BlockingQueue<Runnable> taskQueue;
    private volatile boolean running;
    
    public WorkerThread(int id, BlockingQueue<Runnable> taskQueue) {
        this.id = id;
        this.taskQueue = taskQueue;
        this.running = true;
        System.out.println("🧵 Worker #" + id + " 생성");
    }
    
    @Override
    public void run() {
        while (running) {
            try {
                Runnable task = taskQueue.poll(1, TimeUnit.SECONDS);
                if (task != null) {
                    System.out.println("  ⚙️ Worker #" + id + " 작업 실행 중...");
                    task.run();
                    System.out.println("  ✅ Worker #" + id + " 작업 완료");
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
    
    @Override
    public void reset() {
        System.out.println("  🔄 Worker #" + id + " 초기화");
    }
    
    public void shutdown() {
        this.running = false;
    }
    
    public int getWorkerId() {
        return id;
    }
}

/**
 * 스레드 풀
 */
public class ThreadPool {
    private final List<WorkerThread> workers;
    private final BlockingQueue<Runnable> taskQueue;
    private final int poolSize;
    
    public ThreadPool(int poolSize) {
        this.poolSize = poolSize;
        this.taskQueue = new LinkedBlockingQueue<>();
        this.workers = new ArrayList<>();
        
        // Worker 스레드 생성 및 시작
        for (int i = 0; i < poolSize; i++) {
            WorkerThread worker = new WorkerThread(i + 1, taskQueue);
            workers.add(worker);
            worker.start();
        }
        
        System.out.println("✅ 스레드 풀 생성 완료: " + poolSize + "개 워커\n");
    }
    
    /**
     * 작업 제출
     */
    public void submit(Runnable task) {
        try {
            taskQueue.put(task);
            System.out.println("📨 작업 제출 (대기열: " + taskQueue.size() + ")");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    /**
     * 종료
     */
    public void shutdown() {
        System.out.println("\n🛑 스레드 풀 종료 중...");
        for (WorkerThread worker : workers) {
            worker.shutdown();
        }
        for (WorkerThread worker : workers) {
            try {
                worker.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.println("✅ 스레드 풀 종료 완료");
    }
}

/**
 * 사용 예제
 */
public class ThreadPoolExample {
    public static void main(String[] args) throws InterruptedException {
        // 스레드 풀 생성 (3개 워커)
        ThreadPool pool = new ThreadPool(3);
        
        // 10개 작업 제출
        for (int i = 1; i <= 10; i++) {
            final int taskId = i;
            pool.submit(() -> {
                try {
                    System.out.println("    💼 Task #" + taskId + " 실행");
                    Thread.sleep(500); // 작업 시뮬레이션
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
        
        // 모든 작업 완료 대기
        Thread.sleep(5000);
        
        // 풀 종료
        pool.shutdown();
    }
}
```

---

### 예제 2: 게임 오브젝트 풀 ⭐⭐⭐

```java
/**
 * 게임 오브젝트 (총알)
 */
public class Bullet implements Reusable {
    private static int idCounter = 0;
    private final int id;
    private int x, y;
    private int speed;
    private boolean active;
    
    public Bullet() {
        this.id = ++idCounter;
        this.active = false;
        System.out.println("🎯 Bullet #" + id + " 생성 (무거운 초기화)");
    }
    
    public void fire(int x, int y, int speed) {
        this.x = x;
        this.y = y;
        this.speed = speed;
        this.active = true;
        System.out.println("  💥 Bullet #" + id + " 발사! (" + x + "," + y + ")");
    }
    
    public void update() {
        if (active) {
            x += speed;
            // 화면 밖으로 나가면 비활성화
            if (x > 1000) {
                active = false;
                System.out.println("  ⚪ Bullet #" + id + " 비활성화");
            }
        }
    }
    
    @Override
    public void reset() {
        this.x = 0;
        this.y = 0;
        this.speed = 0;
        this.active = false;
        System.out.println("  🔄 Bullet #" + id + " 리셋");
    }
    
    public boolean isActive() {
        return active;
    }
    
    public int getId() {
        return id;
    }
}

/**
 * 게임 오브젝트 풀
 */
public class BulletPool extends ObjectPool<Bullet> {
    public BulletPool(int maxPoolSize) {
        super(new ObjectFactory<Bullet>() {
            @Override
            public Bullet create() {
                return new Bullet();
            }
        }, maxPoolSize);
    }
}

/**
 * 게임 예제
 */
public class GameObjectPoolExample {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("### 게임 시작 ###");
        BulletPool bulletPool = new BulletPool(5);
        
        // 첫 번째 발사 (풀에서 빌림)
        System.out.println("\n--- 첫 번째 발사 ---");
        Bullet bullet1 = bulletPool.acquire();
        bullet1.fire(0, 100, 10);
        
        // 두 번째 발사
        System.out.println("\n--- 두 번째 발사 ---");
        Bullet bullet2 = bulletPool.acquire();
        bullet2.fire(0, 200, 10);
        
        // 첫 번째 총알이 화면 밖으로
        System.out.println("\n--- 업데이트 (총알 이동) ---");
        for (int i = 0; i < 110; i++) {
            bullet1.update();
        }
        
        // 첫 번째 총알 반납
        System.out.println("\n--- 총알 반납 ---");
        bulletPool.release(bullet1);
        
        // 세 번째 발사 (재사용!)
        System.out.println("\n--- 세 번째 발사 (재사용) ---");
        Bullet bullet3 = bulletPool.acquire();
        bullet3.fire(0, 300, 10);
        System.out.println("💡 bullet3 ID: " + bullet3.getId() + 
                " (bullet1과 동일 = 재사용!)");
        
        bulletPool.printStatus();
    }
}
```

---

### 예제 3: 버퍼 풀 ⭐⭐

```java
/**
 * 재사용 가능한 버퍼
 */
public class ByteBuffer implements Reusable {
    private final byte[] buffer;
    private final int capacity;
    private int position;
    
    public ByteBuffer(int capacity) {
        this.capacity = capacity;
        this.buffer = new byte[capacity];
        this.position = 0;
        System.out.println("📦 Buffer 생성 (크기: " + capacity + " bytes)");
    }
    
    public void write(byte[] data) {
        int length = Math.min(data.length, capacity - position);
        System.arraycopy(data, 0, buffer, position, length);
        position += length;
        System.out.println("  ✍️ " + length + " bytes 기록 (총: " + position + "/" + capacity + ")");
    }
    
    public byte[] read() {
        byte[] data = new byte[position];
        System.arraycopy(buffer, 0, data, 0, position);
        System.out.println("  📖 " + position + " bytes 읽음");
        return data;
    }
    
    @Override
    public void reset() {
        Arrays.fill(buffer, (byte) 0);
        position = 0;
        System.out.println("  🔄 Buffer 리셋");
    }
    
    public int getCapacity() {
        return capacity;
    }
}

/**
 * 버퍼 풀
 */
public class BufferPool extends ObjectPool<ByteBuffer> {
    public BufferPool(int bufferSize, int maxPoolSize) {
        super(new ObjectFactory<ByteBuffer>() {
            @Override
            public ByteBuffer create() {
                return new ByteBuffer(bufferSize);
            }
        }, maxPoolSize);
    }
}

/**
 * 사용 예제
 */
public class BufferPoolExample {
    public static void main(String[] args) {
        System.out.println("### 버퍼 풀 생성 ###");
        BufferPool pool = new BufferPool(1024, 3);
        
        // 파일 처리 시뮬레이션
        System.out.println("\n### 파일 처리 ###");
        for (int i = 1; i <= 5; i++) {
            System.out.println("\n--- 파일 " + i + " 처리 ---");
            ByteBuffer buffer = pool.acquire();
            
            // 데이터 쓰기
            byte[] data = ("File " + i + " content").getBytes();
            buffer.write(data);
            
            // 데이터 읽기
            byte[] read = buffer.read();
            System.out.println("  내용: " + new String(read));
            
            // 버퍼 반납
            pool.release(buffer);
        }
        
        pool.printStatus();
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **성능 향상** | 생성 비용 절감 | DB 커넥션 재사용 |
| **리소스 제어** | 최대 개수 제한 | 최대 100개 스레드 |
| **메모리 효율** | GC 부담 감소 | 게임 오브젝트 재사용 |
| **응답 시간** | 즉시 사용 가능 | 대기 시간 없음 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **메모리 사용** | 미리 할당으로 메모리 사용 | 적절한 풀 크기 |
| **복잡도** | 풀 관리 로직 추가 | 라이브러리 사용 |
| **동기화** | 멀티스레드 환경에서 동기화 필요 | Thread-safe 구현 |
| **상태 관리** | 객체 초기화 필수 | reset() 철저히 |

---

## 7. 안티패턴

### ❌ 안티패턴 1: 초기화 누락

```java
// 잘못된 예: reset() 구현 안 함
public class BadObject implements Reusable {
    private String data;
    
    @Override
    public void reset() {
        // 아무것도 안 함! (위험)
    }
}

// 문제
BadObject obj = pool.acquire();
obj.data = "Old Data";
pool.release(obj);

BadObject obj2 = pool.acquire(); // 같은 객체
System.out.println(obj2.data); // "Old Data" (문제!)
```

**해결:**
```java
@Override
public void reset() {
    this.data = null; // 초기화 필수!
}
```

---

### ❌ 안티패턴 2: 풀 고갈 미처리

```java
// 잘못된 예: 풀이 비었을 때 처리 안 함
public T acquire() {
    return available.poll(); // null 반환 가능!
}
```

**해결:**
```java
public T acquire() {
    T object = available.poll();
    if (object == null) {
        // 대기 또는 새로 생성
        object = factory.create();
    }
    return object;
}
```

---

## 8. 핵심 정리

### 📌 Object Pool 패턴 체크리스트

```
✅ 재사용 가능한 객체 정의
✅ reset() 메서드 구현
✅ 풀 크기 제한
✅ Thread-safe 구현
✅ 객체 유효성 검증
✅ 풀 고갈 처리
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 객체 생성 비용이 큼 | ⭐⭐⭐ | DB 커넥션, 스레드 |
| 빈번한 생성/삭제 | ⭐⭐⭐ | 게임 오브젝트 |
| 리소스 제한 필요 | ⭐⭐⭐ | 최대 개수 제어 |
| GC 압박 | ⭐⭐⭐ | 메모리 효율 |

### 💡 핵심 포인트

1. **생성 비용이 큰 객체에 사용**
2. **reset()으로 재사용 준비**
3. **풀 크기 적절히 설정**
4. **Thread-safe 구현 필수**

### 🔥 실무 팁

```java
// Java 제공 라이브러리 사용
// 1. ExecutorService (스레드 풀)
ExecutorService executor = Executors.newFixedThreadPool(10);

// 2. HikariCP (DB 커넥션 풀)
HikariConfig config = new HikariConfig();
config.setMaximumPoolSize(10);
HikariDataSource dataSource = new HikariDataSource(config);

// 3. Apache Commons Pool2
GenericObjectPool<MyObject> pool = 
    new GenericObjectPool<>(factory);
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Abstract Factory](./05-AbstractFactory.md)**

</div>
