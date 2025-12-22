# Producer-Consumer Pattern (생산자-소비자 패턴)

> **"생산자와 소비자를 분리하여 비동기적으로 데이터를 처리하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [BlockingQueue 완전 가이드](#6-blockingqueue-완전-가이드)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 생산자와 소비자가 강하게 결합
public class OrderService {
    public void processOrder(Order order) {
        // 😱 주문 처리와 결제가 동기적!
        validateOrder(order);      // 100ms
        processPayment(order);     // 2000ms  (느림!)
        updateInventory(order);    // 100ms
        sendEmail(order);          // 1000ms
        
        // 총 3200ms 대기!
        // 사용자는 3초 이상 기다림!
    }
}

// 문제 2: 처리 속도 불균형
public class ImageProcessor {
    public void processImages(List<Image> images) {
        for (Image image : images) {
            // 😱 업로드는 빠른데 처리는 느림!
            Image uploaded = uploadImage(image);     // 10ms
            Image processed = processImage(uploaded); // 5000ms
            saveImage(processed);                     // 100ms
        }
        
        // 업로드 속도 >> 처리 속도
        // 병목 현상!
    }
}

// 문제 3: 동기화 문제
public class DataProcessor {
    private List<Data> buffer = new ArrayList<>();
    
    // Producer
    public synchronized void produce(Data data) {
        buffer.add(data);
        notify();  // 😱 잘못된 사용
    }
    
    // Consumer
    public synchronized Data consume() {
        while (buffer.isEmpty()) {
            try {
                wait();  // 😱 복잡하고 에러 발생 쉬움
            } catch (InterruptedException e) {
                return null;
            }
        }
        return buffer.remove(0);
    }
}

// 문제 4: 버퍼 관리 복잡
public class MessageQueue {
    private Queue<Message> queue = new LinkedList<>();
    private final int capacity = 100;
    
    public void put(Message msg) throws InterruptedException {
        synchronized (queue) {
            // 😱 큐가 가득 찰 때까지 대기
            while (queue.size() >= capacity) {
                queue.wait();
            }
            
            queue.add(msg);
            queue.notifyAll();
        }
    }
    
    // 복잡하고 실수하기 쉬움!
}

// 문제 5: 생산/소비 속도 차이
public class LogProcessor {
    private List<LogEntry> logs = new ArrayList<>();
    
    // Producer: 초당 1000개 로그 생성
    public void log(String message) {
        logs.add(new LogEntry(message));
    }
    
    // Consumer: 초당 100개 처리
    public void processLogs() {
        for (LogEntry log : logs) {
            // 😱 느린 처리 (DB 저장 등)
            saveToDB(log);  // 10ms
        }
    }
    
    // 메모리 고갈!
}

// 문제 6: 순서 보장 어려움
public class EventProcessor {
    private List<Event> events = new ArrayList<>();
    
    // 여러 Producer가 동시에 추가
    public void addEvent(Event event) {
        // 😱 순서가 보장되지 않음!
        events.add(event);
    }
}
```

### ⚡ 핵심 문제

1. **강한 결합**: 생산자-소비자가 직접 연결
2. **속도 불균형**: 처리 속도 차이
3. **동기화 복잡**: wait/notify 어려움
4. **버퍼 관리**: 수동 관리 복잡
5. **메모리 문제**: 버퍼 무한 증가

---

## 2. 패턴 정의

### 📖 정의

> **생산자와 소비자를 큐로 분리하여 비동기적으로 데이터를 주고받는 패턴**

### 🎯 목적

- **결합도 감소**: 생산자-소비자 분리
- **속도 조절**: 버퍼로 속도 차이 완화
- **병렬 처리**: 동시에 생산/소비
- **안정성**: 동기화 자동 처리

### 💡 핵심 아이디어

```java
// Before: 직접 연결
producer → consumer (Blocking!)

// After: Queue로 분리
producer → [Queue] → consumer
(비동기!)
```

---

## 3. 구조와 구성요소

### 📊 Producer-Consumer 구조

```
┌─────────────┐
│  Producer 1 │───┐
└─────────────┘   │
                  │
┌─────────────┐   │    ┌──────────────┐
│  Producer 2 │───┼───→│ BlockingQueue│
└─────────────┘   │    │ [  items   ] │
                  │    └──────────────┘
┌─────────────┐   │           │
│  Producer N │───┘           │
└─────────────┘               │
                              ▼
                     ┌─────────────┐
                     │  Consumer 1 │
                     └─────────────┘
                     ┌─────────────┐
                     │  Consumer 2 │
                     └─────────────┘
                     ┌─────────────┐
                     │  Consumer M │
                     └─────────────┘
```

### 🔄 데이터 흐름

```
1. Producer: queue.put(item)
   → 큐가 가득 차면 대기
   
2. Queue: item 저장

3. Consumer: item = queue.take()
   → 큐가 비어있으면 대기
   
4. 반복
```

---

## 4. 구현 방법

### 완전한 구현: 로그 처리 시스템 ⭐⭐⭐

```java
/**
 * ============================================
 * DOMAIN MODEL
 * ============================================
 */

/**
 * 로그 엔트리
 */
class LogEntry {
    private final String level;
    private final String message;
    private final LocalDateTime timestamp;
    
    public LogEntry(String level, String message) {
        this.level = level;
        this.message = message;
        this.timestamp = LocalDateTime.now();
    }
    
    public String getLevel() { return level; }
    public String getMessage() { return message; }
    public LocalDateTime getTimestamp() { return timestamp; }
    
    @Override
    public String toString() {
        return String.format("[%s] %s: %s", 
            timestamp.format(DateTimeFormatter.ofPattern("HH:mm:ss")),
            level,
            message
        );
    }
}

/**
 * ============================================
 * BASIC IMPLEMENTATION
 * ============================================
 */

/**
 * 직접 구현한 Producer-Consumer (교육용)
 */
class ManualProducerConsumer {
    private final Queue<String> queue = new LinkedList<>();
    private final int capacity;
    private final Object lock = new Object();
    
    public ManualProducerConsumer(int capacity) {
        this.capacity = capacity;
    }
    
    /**
     * Producer
     */
    public void produce(String item) throws InterruptedException {
        synchronized (lock) {
            // 큐가 가득 찰 때까지 대기
            while (queue.size() >= capacity) {
                System.out.println("⏸️ Producer 대기 (큐 가득 참)");
                lock.wait();
            }
            
            queue.add(item);
            System.out.println("➕ 생산: " + item + " (크기: " + queue.size() + ")");
            
            // Consumer 깨우기
            lock.notifyAll();
        }
    }
    
    /**
     * Consumer
     */
    public String consume() throws InterruptedException {
        synchronized (lock) {
            // 큐가 비어있으면 대기
            while (queue.isEmpty()) {
                System.out.println("⏸️ Consumer 대기 (큐 비어있음)");
                lock.wait();
            }
            
            String item = queue.poll();
            System.out.println("➖ 소비: " + item + " (크기: " + queue.size() + ")");
            
            // Producer 깨우기
            lock.notifyAll();
            
            return item;
        }
    }
}

/**
 * ============================================
 * BLOCKINGQUEUE IMPLEMENTATION
 * ============================================
 */

/**
 * Log Producer
 */
class LogProducer implements Runnable {
    private final BlockingQueue<LogEntry> queue;
    private final String producerId;
    private final int count;
    
    public LogProducer(BlockingQueue<LogEntry> queue, String producerId, int count) {
        this.queue = queue;
        this.producerId = producerId;
        this.count = count;
    }
    
    @Override
    public void run() {
        try {
            for (int i = 1; i <= count; i++) {
                // 로그 생성
                String level = (i % 3 == 0) ? "ERROR" : (i % 2 == 0) ? "WARN" : "INFO";
                LogEntry log = new LogEntry(level, producerId + " - 메시지 " + i);
                
                // 큐에 추가 (가득 차면 자동 대기)
                queue.put(log);
                
                System.out.println("📝 [" + producerId + "] 로그 생성: " + log.getMessage());
                
                // 랜덤 딜레이
                Thread.sleep((long) (Math.random() * 100));
            }
            
            System.out.println("✅ [" + producerId + "] 생산 완료");
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}

/**
 * Log Consumer
 */
class LogConsumer implements Runnable {
    private final BlockingQueue<LogEntry> queue;
    private final String consumerId;
    private volatile boolean running = true;
    
    public LogConsumer(BlockingQueue<LogEntry> queue, String consumerId) {
        this.queue = queue;
        this.consumerId = consumerId;
    }
    
    @Override
    public void run() {
        try {
            while (running) {
                // 큐에서 가져오기 (비어있으면 자동 대기)
                LogEntry log = queue.poll(1, TimeUnit.SECONDS);
                
                if (log != null) {
                    // 로그 처리 (시뮬레이션)
                    processLog(log);
                }
            }
            
            System.out.println("🛑 [" + consumerId + "] 종료");
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    private void processLog(LogEntry log) throws InterruptedException {
        System.out.println("⚙️ [" + consumerId + "] 처리 중: " + log);
        
        // 처리 시간 시뮬레이션 (ERROR는 더 오래 걸림)
        int delay = log.getLevel().equals("ERROR") ? 300 : 100;
        Thread.sleep(delay);
        
        System.out.println("✅ [" + consumerId + "] 처리 완료: " + log.getMessage());
    }
    
    public void stop() {
        running = false;
    }
}

/**
 * ============================================
 * DIFFERENT QUEUE TYPES
 * ============================================
 */

/**
 * Queue 타입별 예제
 */
public class QueueTypesDemo {
    
    /**
     * 1. ArrayBlockingQueue (고정 크기)
     */
    public void arrayBlockingQueueExample() throws InterruptedException {
        System.out.println("\n=== ArrayBlockingQueue (고정 크기) ===");
        
        BlockingQueue<String> queue = new ArrayBlockingQueue<>(3);
        
        // Producer
        new Thread(() -> {
            try {
                for (int i = 1; i <= 5; i++) {
                    String item = "Item-" + i;
                    System.out.println("➕ 생산 시도: " + item);
                    queue.put(item);
                    System.out.println("✅ 생산 완료: " + item);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }).start();
        
        // Consumer (느림)
        Thread.sleep(500);
        new Thread(() -> {
            try {
                while (true) {
                    String item = queue.take();
                    System.out.println("➖ 소비: " + item);
                    Thread.sleep(1000);  // 느린 처리
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }).start();
        
        Thread.sleep(6000);
    }
    
    /**
     * 2. LinkedBlockingQueue (무제한 또는 제한)
     */
    public void linkedBlockingQueueExample() {
        System.out.println("\n=== LinkedBlockingQueue ===");
        
        // 무제한
        BlockingQueue<String> unlimited = new LinkedBlockingQueue<>();
        
        // 제한
        BlockingQueue<String> limited = new LinkedBlockingQueue<>(100);
    }
    
    /**
     * 3. PriorityBlockingQueue (우선순위)
     */
    public void priorityBlockingQueueExample() throws InterruptedException {
        System.out.println("\n=== PriorityBlockingQueue (우선순위) ===");
        
        // ERROR > WARN > INFO 순서
        BlockingQueue<LogEntry> queue = new PriorityBlockingQueue<>(
            10,
            Comparator.comparing(LogEntry::getLevel).reversed()
        );
        
        // 랜덤 순서로 추가
        queue.put(new LogEntry("INFO", "정보 메시지"));
        queue.put(new LogEntry("ERROR", "에러 발생!"));
        queue.put(new LogEntry("WARN", "경고"));
        queue.put(new LogEntry("INFO", "또 다른 정보"));
        
        // 우선순위 순서로 꺼내짐
        while (!queue.isEmpty()) {
            LogEntry log = queue.take();
            System.out.println("처리: " + log);
        }
    }
    
    /**
     * 4. SynchronousQueue (직접 전달)
     */
    public void synchronousQueueExample() throws InterruptedException {
        System.out.println("\n=== SynchronousQueue (직접 전달) ===");
        
        BlockingQueue<String> queue = new SynchronousQueue<>();
        
        // Producer
        new Thread(() -> {
            try {
                System.out.println("➕ 생산 시도...");
                queue.put("Item");  // Consumer가 받을 때까지 Blocking
                System.out.println("✅ 전달 완료");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }).start();
        
        // Consumer (2초 후)
        Thread.sleep(2000);
        new Thread(() -> {
            try {
                System.out.println("➖ 수신 준비...");
                String item = queue.take();
                System.out.println("✅ 수신: " + item);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }).start();
        
        Thread.sleep(1000);
    }
}

/**
 * ============================================
 * COMPLETE EXAMPLE
 * ============================================
 */

/**
 * 로그 처리 시스템
 */
public class LogProcessingSystem {
    private final BlockingQueue<LogEntry> queue;
    private final List<LogConsumer> consumers;
    private final ExecutorService executorService;
    
    public LogProcessingSystem(int queueSize, int consumerCount) {
        this.queue = new ArrayBlockingQueue<>(queueSize);
        this.consumers = new ArrayList<>();
        this.executorService = Executors.newCachedThreadPool();
        
        // Consumer 생성
        for (int i = 1; i <= consumerCount; i++) {
            LogConsumer consumer = new LogConsumer(queue, "Consumer-" + i);
            consumers.add(consumer);
            executorService.submit(consumer);
        }
        
        System.out.println("✅ 로그 처리 시스템 시작 (큐: " + queueSize + 
                          ", Consumer: " + consumerCount + ")");
    }
    
    /**
     * Producer 추가
     */
    public void addProducer(String producerId, int logCount) {
        LogProducer producer = new LogProducer(queue, producerId, logCount);
        executorService.submit(producer);
    }
    
    /**
     * 시스템 종료
     */
    public void shutdown() throws InterruptedException {
        System.out.println("\n🛑 시스템 종료 시작...");
        
        // Consumer 종료
        for (LogConsumer consumer : consumers) {
            consumer.stop();
        }
        
        executorService.shutdown();
        executorService.awaitTermination(10, TimeUnit.SECONDS);
        
        System.out.println("✅ 시스템 종료 완료");
    }
    
    /**
     * 통계
     */
    public void printStats() {
        System.out.println("\n📊 큐 상태:");
        System.out.println("   크기: " + queue.size());
        System.out.println("   남은 용량: " + queue.remainingCapacity());
    }
}

/**
 * ============================================
 * DEMO
 * ============================================
 */
public class ProducerConsumerDemo {
    public static void main(String[] args) throws Exception {
        System.out.println("=== Producer-Consumer Pattern 예제 ===");
        
        // 1. Queue 타입 예제
        System.out.println("\n" + "=".repeat(60));
        System.out.println("1️⃣ BlockingQueue 타입별 예제");
        System.out.println("=".repeat(60));
        
        QueueTypesDemo queueDemo = new QueueTypesDemo();
        queueDemo.arrayBlockingQueueExample();
        queueDemo.priorityBlockingQueueExample();
        queueDemo.synchronousQueueExample();
        
        System.out.println("\n" + "=".repeat(60));
        System.out.println("2️⃣ 로그 처리 시스템");
        System.out.println("=".repeat(60));
        
        // 2. 로그 처리 시스템
        LogProcessingSystem system = new LogProcessingSystem(10, 3);
        
        // Producer 추가
        system.addProducer("App-Server", 5);
        system.addProducer("DB-Server", 5);
        system.addProducer("Cache-Server", 5);
        
        // 5초 대기
        Thread.sleep(5000);
        
        system.printStats();
        
        // 종료
        system.shutdown();
        
        System.out.println("\n✅ 모든 예제 완료!");
    }
}
```

**실행 결과:**
```
=== Producer-Consumer Pattern 예제 ===

============================================================
1️⃣ BlockingQueue 타입별 예제
============================================================

=== ArrayBlockingQueue (고정 크기) ===
➕ 생산 시도: Item-1
✅ 생산 완료: Item-1
➕ 생산 시도: Item-2
✅ 생산 완료: Item-2
➕ 생산 시도: Item-3
✅ 생산 완료: Item-3
➕ 생산 시도: Item-4
[대기... 큐가 가득 참]
➖ 소비: Item-1
✅ 생산 완료: Item-4

=== PriorityBlockingQueue (우선순위) ===
처리: [12:34:56] ERROR: 에러 발생!
처리: [12:34:56] WARN: 경고
처리: [12:34:56] INFO: 정보 메시지
처리: [12:34:56] INFO: 또 다른 정보

============================================================
2️⃣ 로그 처리 시스템
============================================================
✅ 로그 처리 시스템 시작 (큐: 10, Consumer: 3)
📝 [App-Server] 로그 생성: App-Server - 메시지 1
📝 [DB-Server] 로그 생성: DB-Server - 메시지 1
⚙️ [Consumer-1] 처리 중: [12:34:56] INFO: App-Server - 메시지 1
⚙️ [Consumer-2] 처리 중: [12:34:56] INFO: DB-Server - 메시지 1
✅ [Consumer-1] 처리 완료: App-Server - 메시지 1

📊 큐 상태:
   크기: 3
   남은 용량: 7

🛑 시스템 종료 시작...
✅ 시스템 종료 완료

✅ 모든 예제 완료!
```

---

## 5. 실전 예제

### 예제 1: 이미지 업로드 시스템 ⭐⭐⭐

```java
public class ImageUploadSystem {
    private final BlockingQueue<Image> uploadQueue;
    private final BlockingQueue<Image> processQueue;
    
    public ImageUploadSystem() {
        this.uploadQueue = new LinkedBlockingQueue<>(100);
        this.processQueue = new LinkedBlockingQueue<>(50);
        
        // Upload Worker
        startUploadWorker();
        
        // Process Worker
        startProcessWorker();
    }
    
    private void startUploadWorker() {
        new Thread(() -> {
            while (true) {
                Image image = uploadQueue.take();
                uploadToS3(image);
                processQueue.put(image);
            }
        }).start();
    }
    
    private void startProcessWorker() {
        new Thread(() -> {
            while (true) {
                Image image = processQueue.take();
                resizeImage(image);
                generateThumbnail(image);
            }
        }).start();
    }
}
```

---

## 6. BlockingQueue 완전 가이드

### 📊 주요 메서드

| 메서드 | Blocking | Timeout | Exception |
|--------|----------|---------|-----------|
| `put()` | ✅ | ❌ | ❌ |
| `take()` | ✅ | ❌ | ❌ |
| `offer(e, time)` | ✅ | ✅ | ❌ |
| `poll(time)` | ✅ | ✅ | ❌ |
| `add()` | ❌ | ❌ | ✅ |
| `remove()` | ❌ | ❌ | ✅ |

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 |
|------|------|
| **결합도 감소** | 생산자-소비자 분리 |
| **속도 조절** | 버퍼로 완충 |
| **병렬 처리** | 동시 실행 |

### ❌ 단점

| 단점 | 설명 |
|------|------|
| **메모리** | 큐 크기 |
| **복잡도** | 디버깅 어려움 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: 무한정 큐

```java
// 잘못된 예
BlockingQueue<Task> queue = new LinkedBlockingQueue<>();  // ❌ 무제한

// 올바른 예
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(1000);  // ✅ 제한
```

---

## 9. 심화 주제

### 🎯 Backpressure 처리

```java
// Bounded Queue + Reject Policy
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(100);

try {
    boolean added = queue.offer(task, 1, TimeUnit.SECONDS);
    if (!added) {
        // Backpressure 처리
        handleRejection(task);
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

---

## 10. 핵심 정리

### 📌 체크리스트

```
✅ BlockingQueue 사용
✅ 적절한 큐 크기
✅ Producer/Consumer 비율
✅ 예외 처리
✅ Graceful Shutdown
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Thread Pool](./01-ThreadPool.md) | [다음: Reader-Writer Lock →](./03-ReaderWriterLock.md)**

</div>
