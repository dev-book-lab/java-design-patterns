# Thread Pool Pattern (스레드 풀 패턴)

> **"스레드를 미리 생성하여 재사용함으로써 성능을 최적화하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [ExecutorService 완전 가이드](#6-executorservice-완전-가이드)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 요청마다 스레드 생성
public class WebServer {
    public void handleRequest(Request request) {
        // 😱 요청마다 새 스레드 생성!
        new Thread(() -> {
            processRequest(request);
        }).start();
        
        // 문제점:
        // 1. 스레드 생성 비용 (시간, 메모리)
        // 2. 무제한 스레드 생성 → 리소스 고갈
        // 3. 컨텍스트 스위칭 오버헤드
    }
}

// 시나리오: 1000개 요청 들어오면?
// → 1000개 스레드 생성!
// → 시스템 다운!

// 문제 2: 스레드 관리 어려움
public class TaskProcessor {
    private List<Thread> threads = new ArrayList<>();
    
    public void processTasks(List<Task> tasks) {
        // 😱 수동으로 스레드 관리
        for (Task task : tasks) {
            Thread thread = new Thread(() -> task.execute());
            threads.add(thread);
            thread.start();
        }
        
        // 모든 스레드 대기
        for (Thread thread : threads) {
            try {
                thread.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        
        // 문제:
        // - 복잡한 코드
        // - 에러 처리 어려움
        // - 스레드 재사용 불가
    }
}

// 문제 3: 리소스 제한 없음
public class EmailService {
    public void sendEmails(List<Email> emails) {
        // 😱 10,000개 이메일 = 10,000개 스레드?
        for (Email email : emails) {
            new Thread(() -> send(email)).start();
        }
        
        // 시스템 리소스 고갈!
        // OutOfMemoryError!
    }
}

// 문제 4: 스레드 재사용 불가
// 스레드 생성 비용:
// - 메모리 할당 (Stack: 1MB)
// - 시스템 호출 (커널 레벨)
// - 컨텍스트 초기화

// 1000개 요청 처리:
// Without Pool: 1000번 생성 = 1000 * 비용
// With Pool: 10개 재사용 = 10 * 비용

// 문제 5: 작업 큐 관리
public class WorkerManager {
    private Queue<Task> tasks = new LinkedList<>();
    private List<Thread> workers = new ArrayList<>();
    
    // 😱 직접 구현해야 함!
    public void start() {
        // Worker 스레드 생성
        for (int i = 0; i < 5; i++) {
            Thread worker = new Thread(() -> {
                while (true) {
                    Task task;
                    synchronized (tasks) {
                        while (tasks.isEmpty()) {
                            try {
                                tasks.wait();
                            } catch (InterruptedException e) {
                                return;
                            }
                        }
                        task = tasks.poll();
                    }
                    task.execute();
                }
            });
            workers.add(worker);
            worker.start();
        }
    }
    
    // 복잡하고 에러 발생 쉬움!
}
```

### ⚡ 핵심 문제

1. **생성 비용**: 스레드 생성은 비쌈
2. **리소스 낭비**: 무제한 생성
3. **관리 복잡**: 수동 관리 어려움
4. **재사용 불가**: 일회용 스레드
5. **제어 어려움**: 동시 실행 수 제한 불가

---

## 2. 패턴 정의

### 📖 정의

> **미리 생성된 스레드들을 풀로 관리하여 작업이 들어올 때 재사용함으로써 성능을 최적화하는 패턴**

### 🎯 목적

- **성능 향상**: 스레드 재사용
- **리소스 제어**: 스레드 개수 제한
- **간편한 관리**: 자동 관리
- **확장성**: 작업 큐 관리

### 💡 핵심 아이디어

```java
// Before: 매번 생성
for (Task task : tasks) {
    new Thread(() -> task.execute()).start();
}

// After: Thread Pool
ExecutorService executor = Executors.newFixedThreadPool(10);
for (Task task : tasks) {
    executor.submit(() -> task.execute());
}
executor.shutdown();

// 10개 스레드로 모든 작업 처리!
```

---

## 3. 구조와 구성요소

### 📊 Thread Pool 구조

```
┌─────────────────────────────────────┐
│         Client                      │
│   executor.submit(task)             │
└─────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│      ExecutorService                │
│   (Thread Pool Manager)             │
└─────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│        Task Queue                   │
│   [Task1][Task2][Task3]...          │
└─────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────┐
│      Worker Threads                 │
│   [Thread1][Thread2]...[Thread10]   │
└─────────────────────────────────────┘
```

### 🔄 작업 흐름

```
1. submit(task) → Task Queue에 추가
2. 유휴 Worker Thread가 Task 가져감
3. Task 실행
4. 완료 후 다시 대기 (재사용!)
5. 반복
```

### 🔧 구성요소

| 컴포넌트 | 역할 | 설명 |
|---------|------|------|
| **ExecutorService** | 풀 관리자 | 작업 제출, 종료 관리 |
| **Worker Threads** | 작업 실행 | 미리 생성된 스레드들 |
| **Task Queue** | 작업 대기열 | 실행 대기 중인 작업들 |
| **ThreadFactory** | 스레드 생성 | 커스텀 스레드 생성 |
| **RejectedExecutionHandler** | 거부 정책 | 큐 가득 찬 경우 처리 |

---

## 4. 구현 방법

### 완전한 구현: 이미지 처리 서비스 ⭐⭐⭐

```java
/**
 * ============================================
 * BASIC THREAD POOL IMPLEMENTATION
 * ============================================
 */

/**
 * 간단한 Thread Pool 직접 구현
 */
public class SimpleThreadPool {
    private final int poolSize;
    private final WorkerThread[] workers;
    private final BlockingQueue<Runnable> taskQueue;
    private volatile boolean isShutdown = false;
    
    public SimpleThreadPool(int poolSize, int queueSize) {
        this.poolSize = poolSize;
        this.taskQueue = new LinkedBlockingQueue<>(queueSize);
        this.workers = new WorkerThread[poolSize];
        
        // Worker 스레드 생성 및 시작
        for (int i = 0; i < poolSize; i++) {
            workers[i] = new WorkerThread("Worker-" + i);
            workers[i].start();
        }
        
        System.out.println("✅ Thread Pool 생성: " + poolSize + "개 스레드");
    }
    
    /**
     * 작업 제출
     */
    public void submit(Runnable task) {
        if (isShutdown) {
            throw new IllegalStateException("Pool is shutdown");
        }
        
        try {
            taskQueue.put(task);
            System.out.println("📥 작업 제출: " + task);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Task submission interrupted", e);
        }
    }
    
    /**
     * Pool 종료
     */
    public void shutdown() {
        isShutdown = true;
        
        // 모든 Worker에게 인터럽트
        for (WorkerThread worker : workers) {
            worker.interrupt();
        }
        
        System.out.println("🛑 Thread Pool 종료");
    }
    
    /**
     * Worker Thread 클래스
     */
    private class WorkerThread extends Thread {
        public WorkerThread(String name) {
            super(name);
        }
        
        @Override
        public void run() {
            System.out.println("🔄 " + getName() + " 시작");
            
            while (!isShutdown) {
                try {
                    // 작업 대기 (Blocking)
                    Runnable task = taskQueue.take();
                    
                    System.out.println("⚙️ " + getName() + " 작업 실행: " + task);
                    
                    // 작업 실행
                    task.run();
                    
                    System.out.println("✅ " + getName() + " 작업 완료");
                    
                } catch (InterruptedException e) {
                    // 종료 신호
                    break;
                }
            }
            
            System.out.println("🛑 " + getName() + " 종료");
        }
    }
}

/**
 * ============================================
 * EXECUTOR SERVICE EXAMPLES
 * ============================================
 */

/**
 * 이미지 처리 작업
 */
class ImageProcessingTask implements Runnable {
    private final String imageName;
    private final int processingTime;
    
    public ImageProcessingTask(String imageName, int processingTime) {
        this.imageName = imageName;
        this.processingTime = processingTime;
    }
    
    @Override
    public void run() {
        System.out.println("🖼️ [" + Thread.currentThread().getName() + "] " +
                          imageName + " 처리 시작");
        
        try {
            // 시뮬레이션: 처리 시간
            Thread.sleep(processingTime);
            
            System.out.println("✅ [" + Thread.currentThread().getName() + "] " +
                              imageName + " 처리 완료");
        } catch (InterruptedException e) {
            System.err.println("❌ " + imageName + " 처리 중단");
            Thread.currentThread().interrupt();
        }
    }
    
    @Override
    public String toString() {
        return "ImageTask(" + imageName + ")";
    }
}

/**
 * Thread Pool 사용 예제들
 */
public class ThreadPoolExamples {
    
    /**
     * 1. Fixed Thread Pool
     */
    public void fixedThreadPoolExample() {
        System.out.println("\n=== Fixed Thread Pool ===");
        
        // 고정 크기 풀 (3개 스레드)
        ExecutorService executor = Executors.newFixedThreadPool(3);
        
        // 10개 작업 제출
        for (int i = 1; i <= 10; i++) {
            executor.submit(new ImageProcessingTask("image" + i + ".jpg", 1000));
        }
        
        // 종료
        executor.shutdown();
        
        try {
            // 모든 작업 완료 대기 (최대 10초)
            if (!executor.awaitTermination(10, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
        }
        
        System.out.println("✅ 모든 작업 완료");
    }
    
    /**
     * 2. Cached Thread Pool
     */
    public void cachedThreadPoolExample() {
        System.out.println("\n=== Cached Thread Pool ===");
        
        // 필요에 따라 스레드 생성 (60초 idle 후 제거)
        ExecutorService executor = Executors.newCachedThreadPool();
        
        // 5개 작업 제출
        for (int i = 1; i <= 5; i++) {
            executor.submit(new ImageProcessingTask("photo" + i + ".png", 500));
        }
        
        executor.shutdown();
    }
    
    /**
     * 3. Single Thread Executor
     */
    public void singleThreadExecutorExample() {
        System.out.println("\n=== Single Thread Executor ===");
        
        // 단일 스레드 (순차 처리)
        ExecutorService executor = Executors.newSingleThreadExecutor();
        
        for (int i = 1; i <= 3; i++) {
            executor.submit(new ImageProcessingTask("doc" + i + ".pdf", 300));
        }
        
        executor.shutdown();
    }
    
    /**
     * 4. Scheduled Thread Pool
     */
    public void scheduledThreadPoolExample() {
        System.out.println("\n=== Scheduled Thread Pool ===");
        
        ScheduledExecutorService executor = Executors.newScheduledThreadPool(2);
        
        // 1초 후 실행
        executor.schedule(
            () -> System.out.println("⏰ 1초 후 실행"),
            1,
            TimeUnit.SECONDS
        );
        
        // 2초 후 시작, 3초마다 반복
        executor.scheduleAtFixedRate(
            () -> System.out.println("🔄 주기적 실행: " + System.currentTimeMillis()),
            2,
            3,
            TimeUnit.SECONDS
        );
        
        // 5초 후 종료
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        
        executor.shutdown();
    }
    
    /**
     * 5. Custom Thread Pool
     */
    public void customThreadPoolExample() {
        System.out.println("\n=== Custom Thread Pool ===");
        
        // 커스텀 ThreadPoolExecutor
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
            2,                              // corePoolSize
            4,                              // maximumPoolSize
            60,                             // keepAliveTime
            TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(10),   // workQueue
            new CustomThreadFactory(),      // threadFactory
            new ThreadPoolExecutor.CallerRunsPolicy()  // handler
        );
        
        // 모니터링
        executor.prestartAllCoreThreads();
        
        for (int i = 1; i <= 5; i++) {
            executor.submit(new ImageProcessingTask("custom" + i + ".jpg", 200));
            
            System.out.println("📊 Pool 상태:");
            System.out.println("   Active: " + executor.getActiveCount());
            System.out.println("   Queue: " + executor.getQueue().size());
            System.out.println("   Completed: " + executor.getCompletedTaskCount());
        }
        
        executor.shutdown();
    }
}

/**
 * 커스텀 Thread Factory
 */
class CustomThreadFactory implements ThreadFactory {
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;
    
    public CustomThreadFactory() {
        this.namePrefix = "CustomPool-";
    }
    
    @Override
    public Thread newThread(Runnable r) {
        Thread thread = new Thread(r, namePrefix + threadNumber.getAndIncrement());
        thread.setDaemon(false);  // Non-daemon
        thread.setPriority(Thread.NORM_PRIORITY);
        
        System.out.println("🔨 새 스레드 생성: " + thread.getName());
        
        return thread;
    }
}

/**
 * ============================================
 * FUTURE AND CALLABLE
 * ============================================
 */

/**
 * Callable 예제 (반환값 있음)
 */
class ImageSizeCalculator implements Callable<Long> {
    private final String imageName;
    
    public ImageSizeCalculator(String imageName) {
        this.imageName = imageName;
    }
    
    @Override
    public Long call() throws Exception {
        System.out.println("📏 [" + Thread.currentThread().getName() + "] " +
                          imageName + " 크기 계산 중");
        
        Thread.sleep(500);  // 시뮬레이션
        
        // 랜덤 크기 반환
        long size = (long) (Math.random() * 10000000);
        
        System.out.println("✅ " + imageName + " 크기: " + size + " bytes");
        
        return size;
    }
}

/**
 * Future 사용 예제
 */
public class FutureExample {
    public void futureExample() throws Exception {
        System.out.println("\n=== Future Example ===");
        
        ExecutorService executor = Executors.newFixedThreadPool(3);
        
        // Callable 제출 (Future 반환)
        List<Future<Long>> futures = new ArrayList<>();
        
        for (int i = 1; i <= 5; i++) {
            Future<Long> future = executor.submit(
                new ImageSizeCalculator("image" + i + ".jpg")
            );
            futures.add(future);
        }
        
        // 결과 수집
        long totalSize = 0;
        for (Future<Long> future : futures) {
            try {
                // get()은 Blocking (결과 나올 때까지 대기)
                Long size = future.get();
                totalSize += size;
                
            } catch (ExecutionException e) {
                System.err.println("❌ 작업 실패: " + e.getCause());
            }
        }
        
        System.out.println("📊 총 크기: " + totalSize + " bytes");
        
        executor.shutdown();
    }
}

/**
 * ============================================
 * DEMO
 * ============================================
 */
public class ThreadPoolDemo {
    public static void main(String[] args) throws Exception {
        System.out.println("=== Thread Pool Pattern 예제 ===");
        
        // 1. 직접 구현한 Simple Thread Pool
        System.out.println("\n" + "=".repeat(60));
        System.out.println("1️⃣ Simple Thread Pool (직접 구현)");
        System.out.println("=".repeat(60));
        
        SimpleThreadPool simplePool = new SimpleThreadPool(3, 10);
        
        for (int i = 1; i <= 5; i++) {
            int taskId = i;
            simplePool.submit(() -> {
                System.out.println("🔧 Task " + taskId + " 실행");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }
        
        Thread.sleep(6000);
        simplePool.shutdown();
        
        // 2. ExecutorService 예제들
        System.out.println("\n" + "=".repeat(60));
        System.out.println("2️⃣ ExecutorService Examples");
        System.out.println("=".repeat(60));
        
        ThreadPoolExamples examples = new ThreadPoolExamples();
        
        examples.fixedThreadPoolExample();
        Thread.sleep(12000);
        
        examples.cachedThreadPoolExample();
        Thread.sleep(3000);
        
        examples.singleThreadExecutorExample();
        Thread.sleep(2000);
        
        examples.customThreadPoolExample();
        Thread.sleep(2000);
        
        // 3. Future 예제
        System.out.println("\n" + "=".repeat(60));
        System.out.println("3️⃣ Future Example");
        System.out.println("=".repeat(60));
        
        FutureExample futureExample = new FutureExample();
        futureExample.futureExample();
        
        System.out.println("\n✅ 모든 예제 완료!");
    }
}
```

**실행 결과:**
```
=== Thread Pool Pattern 예제 ===

============================================================
1️⃣ Simple Thread Pool (직접 구현)
============================================================
✅ Thread Pool 생성: 3개 스레드
🔄 Worker-0 시작
🔄 Worker-1 시작
🔄 Worker-2 시작
📥 작업 제출: Task 1
📥 작업 제출: Task 2
📥 작업 제출: Task 3
⚙️ Worker-0 작업 실행: Task 1
⚙️ Worker-1 작업 실행: Task 2
⚙️ Worker-2 작업 실행: Task 3
🔧 Task 1 실행
🔧 Task 2 실행
🔧 Task 3 실행
✅ Worker-0 작업 완료
✅ Worker-1 작업 완료
✅ Worker-2 작업 완료
🛑 Thread Pool 종료

============================================================
2️⃣ ExecutorService Examples
============================================================

=== Fixed Thread Pool ===
🖼️ [pool-1-thread-1] image1.jpg 처리 시작
🖼️ [pool-1-thread-2] image2.jpg 처리 시작
🖼️ [pool-1-thread-3] image3.jpg 처리 시작
✅ [pool-1-thread-1] image1.jpg 처리 완료
🖼️ [pool-1-thread-1] image4.jpg 처리 시작
✅ 모든 작업 완료

============================================================
3️⃣ Future Example
============================================================

=== Future Example ===
📏 [pool-3-thread-1] image1.jpg 크기 계산 중
📏 [pool-3-thread-2] image2.jpg 크기 계산 중
✅ image1.jpg 크기: 8234567 bytes
✅ image2.jpg 크기: 5678901 bytes
📊 총 크기: 35789012 bytes

✅ 모든 예제 완료!
```

---

## 5. 실전 예제

### 예제 1: 웹 크롤러 ⭐⭐⭐

```java
public class WebCrawler {
    private final ExecutorService executor;
    private final Set<String> visited = ConcurrentHashMap.newKeySet();
    
    public WebCrawler(int poolSize) {
        this.executor = Executors.newFixedThreadPool(poolSize);
    }
    
    public void crawl(String url, int depth) {
        if (depth == 0 || visited.contains(url)) {
            return;
        }
        
        visited.add(url);
        
        executor.submit(() -> {
            List<String> links = fetchLinks(url);
            for (String link : links) {
                crawl(link, depth - 1);
            }
        });
    }
    
    public void shutdown() {
        executor.shutdown();
    }
}
```

---

## 6. ExecutorService 완전 가이드

### 📊 Executor 계층

```
Executor (interface)
    │
    ├─ ExecutorService (interface)
    │       │
    │       ├─ ThreadPoolExecutor
    │       └─ ScheduledThreadPoolExecutor
    │
    └─ Executors (factory)
```

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 |
|------|------|
| **성능** | 스레드 재사용 |
| **리소스 제어** | 개수 제한 |
| **관리 용이** | 자동 관리 |

### ❌ 단점

| 단점 | 설명 |
|------|------|
| **복잡도** | 설정 필요 |
| **메모리** | 유휴 스레드 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: 무한정 큐

```java
// 잘못된 예
ExecutorService executor = Executors.newFixedThreadPool(10);
while (true) {
    executor.submit(task);  // ❌ 큐 무한 증가
}

// 해결: 제한된 큐
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10, 10, 0L, TimeUnit.MILLISECONDS,
    new ArrayBlockingQueue<>(100)  // ✅ 제한
);
```

---

## 9. 심화 주제

### 🎯 ThreadPoolExecutor 파라미터

```java
new ThreadPoolExecutor(
    corePoolSize,      // 기본 스레드 수
    maximumPoolSize,   // 최대 스레드 수
    keepAliveTime,     // 유휴 시간
    unit,              // 시간 단위
    workQueue,         // 작업 큐
    threadFactory,     // 스레드 생성
    handler            // 거부 정책
);
```

---

## 10. 핵심 정리

### 📌 체크리스트

```
✅ 적절한 Pool 크기 설정
✅ shutdown() 호출
✅ 예외 처리
✅ 작업 큐 크기 제한
✅ 모니터링
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[다음: Producer-Consumer →](./02-ProducerConsumer.md)**

</div>
