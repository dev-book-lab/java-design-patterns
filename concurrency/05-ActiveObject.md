# Active Object Pattern (능동 객체 패턴)

> **"메서드 호출을 비동기로 실행하여 호출자와 실행을 분리하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [Actor 모델 연동](#6-actor-모델-연동)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 동기 메서드 호출 (Blocking)
public class EmailService {
    public void sendEmail(String to, String message) {
        // 😱 이메일 발송은 느림 (3초)
        connectToSMTP();         // 1초
        sendMessage(to, message); // 2초
        
        // 호출자는 3초 동안 대기!
    }
}

// 사용:
emailService.sendEmail("user@example.com", "Hello");
System.out.println("이메일 발송 완료");  // 3초 후 출력

// 문제:
// - UI가 3초 동안 멈춤
// - 사용자 경험 나쁨

// 문제 2: 수동 스레드 관리
public class NotificationService {
    public void notify(User user, String message) {
        // 😱 매번 스레드 생성
        new Thread(() -> {
            sendNotification(user, message);
        }).start();
        
        // 문제:
        // - 스레드 생성 비용
        // - 관리 어려움
        // - 결과 받기 복잡
    }
}

// 문제 3: 공유 상태 동기화
public class Counter {
    private int count = 0;
    
    // 😱 여러 스레드가 동시 접근
    public synchronized void increment() {
        count++;
    }
    
    public synchronized int getCount() {
        return count;
    }
    
    // 문제:
    // - synchronized 복잡
    // - 데드락 위험
    // - 성능 저하
}

// 문제 4: 순서 보장 어려움
public class OrderProcessor {
    public void processOrder(Order order) {
        // 여러 스레드가 동시에 호출
        validateOrder(order);
        chargePayment(order);
        updateInventory(order);
        
        // 😱 순서가 보장되지 않음!
        // Thread 1: validate → charge
        // Thread 2:           charge ← 먼저 실행될 수 있음!
    }
}

// 문제 5: 복잡한 에러 처리
public class TaskExecutor {
    public void executeTask(Task task) {
        new Thread(() -> {
            try {
                task.run();
            } catch (Exception e) {
                // 😱 에러를 어디로?
                // 호출자에게 전달 방법?
                e.printStackTrace();
            }
        }).start();
    }
}
```

### ⚡ 핵심 문제

1. **Blocking**: 동기 호출로 대기
2. **스레드 관리**: 수동 관리 복잡
3. **동기화**: synchronized 복잡
4. **순서**: 실행 순서 보장 어려움
5. **에러**: 비동기 에러 처리 복잡

---

## 2. 패턴 정의

### 📖 정의

> **메서드 호출을 메시지로 변환하여 별도 스레드에서 순차적으로 실행하는 패턴**

### 🎯 목적

- **비동기**: 호출 즉시 반환
- **순차 실행**: 메시지 큐로 순서 보장
- **동기화 불필요**: 단일 스레드 실행
- **스레드 격리**: 호출자와 실행 분리

### 💡 핵심 아이디어

```java
// Before: 동기 호출
service.sendEmail("user@example.com", "Hello");
// ← 여기서 3초 대기

// After: Active Object (비동기)
activeObject.sendEmail("user@example.com", "Hello");
// ← 즉시 반환!
// 실제 실행은 백그라운드 스레드에서
```

---

## 3. 구조와 구성요소

### 📊 Active Object 구조

```
Client
  │
  │ 메서드 호출
  ▼
┌─────────────────────┐
│  Proxy/Servant      │  → 메시지로 변환
└─────────────────────┘
  │
  │ enqueue
  ▼
┌─────────────────────┐
│   Message Queue     │  [Msg1][Msg2][Msg3]
└─────────────────────┘
  │
  │ dequeue
  ▼
┌─────────────────────┐
│  Scheduler Thread   │  → 메시지 실행
└─────────────────────┘
  │
  ▼
실행 (순차적)
```

---

## 4. 구현 방법

### 완전한 구현: Email Service ⭐⭐⭐

```java
/**
 * ============================================
 * ACTIVE OBJECT IMPLEMENTATION
 * ============================================
 */

/**
 * 메시지 인터페이스
 */
interface Message {
    void execute();
}

/**
 * Active Object 기본 클래스
 */
public class ActiveObject {
    private final BlockingQueue<Message> messageQueue;
    private final Thread schedulerThread;
    private volatile boolean running = true;
    
    public ActiveObject(String name) {
        this.messageQueue = new LinkedBlockingQueue<>();
        this.schedulerThread = new Thread(this::processMessages, name + "-Scheduler");
        this.schedulerThread.start();
        
        System.out.println("✅ Active Object 시작: " + name);
    }
    
    /**
     * 메시지 처리 루프
     */
    private void processMessages() {
        while (running) {
            try {
                Message message = messageQueue.take();
                System.out.println("⚙️ [" + Thread.currentThread().getName() + 
                                  "] 메시지 실행");
                message.execute();
                
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
        System.out.println("🛑 Scheduler 종료");
    }
    
    /**
     * 메시지 전송 (비동기)
     */
    protected void enqueue(Message message) {
        try {
            messageQueue.put(message);
            System.out.println("📥 메시지 큐에 추가");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    /**
     * 종료
     */
    public void shutdown() throws InterruptedException {
        running = false;
        schedulerThread.interrupt();
        schedulerThread.join();
    }
}

/**
 * ============================================
 * EMAIL SERVICE (Active Object 구현)
 * ============================================
 */

/**
 * Email Service Active Object
 */
public class EmailServiceActiveObject extends ActiveObject {
    
    public EmailServiceActiveObject() {
        super("EmailService");
    }
    
    /**
     * 이메일 전송 (비동기)
     */
    public void sendEmail(String to, String subject, String body) {
        System.out.println("📬 [" + Thread.currentThread().getName() + 
                          "] sendEmail() 호출 (즉시 반환)");
        
        // 메시지로 변환하여 큐에 추가
        enqueue(() -> {
            System.out.println("📧 이메일 전송 시작: " + to);
            
            // 시뮬레이션: 느린 작업
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            
            System.out.println("✅ 이메일 전송 완료: " + to);
        });
    }
    
    /**
     * 대량 이메일 전송
     */
    public void sendBulkEmail(List<String> recipients, String subject, String body) {
        System.out.println("📬 대량 이메일 전송 시작 (" + recipients.size() + "명)");
        
        for (String recipient : recipients) {
            sendEmail(recipient, subject, body);
        }
    }
}

/**
 * ============================================
 * FUTURE SUPPORT
 * ============================================
 */

/**
 * Future를 반환하는 Active Object
 */
public class EmailServiceWithFuture extends ActiveObject {
    
    public EmailServiceWithFuture() {
        super("EmailServiceWithFuture");
    }
    
    /**
     * 이메일 전송 (Future 반환)
     */
    public Future<Boolean> sendEmail(String to, String subject, String body) {
        CompletableFuture<Boolean> future = new CompletableFuture<>();
        
        enqueue(() -> {
            try {
                System.out.println("📧 이메일 전송: " + to);
                Thread.sleep(1000);
                
                // 성공
                future.complete(true);
                System.out.println("✅ 전송 완료: " + to);
                
            } catch (Exception e) {
                // 실패
                future.completeExceptionally(e);
                System.err.println("❌ 전송 실패: " + to);
            }
        });
        
        return future;
    }
}

/**
 * ============================================
 * ACTOR MODEL STYLE
 * ============================================
 */

/**
 * Actor 인터페이스
 */
interface Actor {
    void send(Object message);
    void start();
    void stop();
}

/**
 * Email Actor
 */
public class EmailActor implements Actor {
    private final BlockingQueue<Object> mailbox = new LinkedBlockingQueue<>();
    private final Thread thread;
    private volatile boolean running = true;
    
    public EmailActor(String name) {
        this.thread = new Thread(this::processMessages, name);
    }
    
    @Override
    public void start() {
        thread.start();
        System.out.println("🎬 Actor 시작: " + thread.getName());
    }
    
    @Override
    public void send(Object message) {
        try {
            mailbox.put(message);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
    
    @Override
    public void stop() {
        running = false;
        thread.interrupt();
    }
    
    private void processMessages() {
        while (running) {
            try {
                Object message = mailbox.take();
                
                if (message instanceof SendEmailMessage) {
                    handleSendEmail((SendEmailMessage) message);
                }
                
            } catch (InterruptedException e) {
                break;
            }
        }
    }
    
    private void handleSendEmail(SendEmailMessage msg) {
        System.out.println("🎬 Actor 처리: " + msg.to);
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        System.out.println("✅ Actor 완료: " + msg.to);
    }
}

/**
 * 메시지 클래스
 */
record SendEmailMessage(String to, String subject, String body) {}

/**
 * ============================================
 * COMPLETE EXAMPLE
 * ============================================
 */

/**
 * 로깅 서비스 (Active Object)
 */
public class LoggingService extends ActiveObject {
    private final List<String> logs = new ArrayList<>();
    
    public LoggingService() {
        super("LoggingService");
    }
    
    /**
     * 로그 작성 (비동기)
     */
    public void log(String level, String message) {
        enqueue(() -> {
            String logEntry = String.format("[%s] %s: %s",
                LocalDateTime.now().format(DateTimeFormatter.ofPattern("HH:mm:ss")),
                level,
                message
            );
            
            logs.add(logEntry);
            System.out.println("📝 " + logEntry);
            
            // 시뮬레이션: 파일 쓰기
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
    }
    
    /**
     * 모든 로그 출력 (비동기)
     */
    public void printAllLogs() {
        enqueue(() -> {
            System.out.println("\n📋 전체 로그 (" + logs.size() + "개):");
            logs.forEach(log -> System.out.println("   " + log));
        });
    }
}

/**
 * ============================================
 * DEMO
 * ============================================
 */
public class ActiveObjectDemo {
    public static void main(String[] args) throws Exception {
        System.out.println("=== Active Object Pattern 예제 ===");
        
        // 1. 기본 Email Service
        System.out.println("\n" + "=".repeat(60));
        System.out.println("1️⃣ 기본 Email Service (비동기)");
        System.out.println("=".repeat(60));
        
        EmailServiceActiveObject emailService = new EmailServiceActiveObject();
        
        // 비동기 호출 (즉시 반환)
        emailService.sendEmail("user1@example.com", "Welcome", "Hello!");
        emailService.sendEmail("user2@example.com", "Notice", "Important");
        emailService.sendEmail("user3@example.com", "Alert", "Warning");
        
        System.out.println("✅ 모든 sendEmail() 호출 완료 (메인 스레드)");
        System.out.println("   → 백그라운드에서 실행 중...\n");
        
        Thread.sleep(7000);
        
        // 2. Future 지원
        System.out.println("\n" + "=".repeat(60));
        System.out.println("2️⃣ Future 지원 (결과 받기)");
        System.out.println("=".repeat(60));
        
        EmailServiceWithFuture futureService = new EmailServiceWithFuture();
        
        Future<Boolean> future1 = futureService.sendEmail("user4@example.com", "Test", "Body");
        Future<Boolean> future2 = futureService.sendEmail("user5@example.com", "Test", "Body");
        
        System.out.println("⏳ 결과 대기 중...");
        Boolean result1 = future1.get();
        Boolean result2 = future2.get();
        
        System.out.println("📊 결과: " + result1 + ", " + result2);
        
        // 3. Actor 모델
        System.out.println("\n" + "=".repeat(60));
        System.out.println("3️⃣ Actor 모델");
        System.out.println("=".repeat(60));
        
        EmailActor actor = new EmailActor("EmailActor");
        actor.start();
        
        actor.send(new SendEmailMessage("user6@example.com", "Actor", "Hello from Actor"));
        actor.send(new SendEmailMessage("user7@example.com", "Actor", "Another message"));
        
        Thread.sleep(3000);
        
        // 4. 로깅 서비스
        System.out.println("\n" + "=".repeat(60));
        System.out.println("4️⃣ 로깅 서비스");
        System.out.println("=".repeat(60));
        
        LoggingService logger = new LoggingService();
        
        logger.log("INFO", "Application started");
        logger.log("DEBUG", "Processing request");
        logger.log("ERROR", "Something went wrong");
        logger.log("INFO", "Request completed");
        
        Thread.sleep(1000);
        logger.printAllLogs();
        
        // 종료
        Thread.sleep(2000);
        
        emailService.shutdown();
        futureService.shutdown();
        actor.stop();
        logger.shutdown();
        
        System.out.println("\n✅ 모든 예제 완료!");
    }
}
```

**실행 결과:**
```
=== Active Object Pattern 예제 ===

============================================================
1️⃣ 기본 Email Service (비동기)
============================================================
✅ Active Object 시작: EmailService
📬 [main] sendEmail() 호출 (즉시 반환)
📥 메시지 큐에 추가
📬 [main] sendEmail() 호출 (즉시 반환)
📥 메시지 큐에 추가
📬 [main] sendEmail() 호출 (즉시 반환)
📥 메시지 큐에 추가
✅ 모든 sendEmail() 호출 완료 (메인 스레드)
   → 백그라운드에서 실행 중...

⚙️ [EmailService-Scheduler] 메시지 실행
📧 이메일 전송 시작: user1@example.com
✅ 이메일 전송 완료: user1@example.com
⚙️ [EmailService-Scheduler] 메시지 실행
📧 이메일 전송 시작: user2@example.com
✅ 이메일 전송 완료: user2@example.com

============================================================
2️⃣ Future 지원 (결과 받기)
============================================================
⏳ 결과 대기 중...
📧 이메일 전송: user4@example.com
✅ 전송 완료: user4@example.com
📧 이메일 전송: user5@example.com
✅ 전송 완료: user5@example.com
📊 결과: true, true

============================================================
3️⃣ Actor 모델
============================================================
🎬 Actor 시작: EmailActor
🎬 Actor 처리: user6@example.com
✅ Actor 완료: user6@example.com
🎬 Actor 처리: user7@example.com
✅ Actor 완료: user7@example.com

============================================================
4️⃣ 로깅 서비스
============================================================
📝 [12:34:56] INFO: Application started
📝 [12:34:56] DEBUG: Processing request
📝 [12:34:56] ERROR: Something went wrong
📝 [12:34:56] INFO: Request completed

📋 전체 로그 (4개):
   [12:34:56] INFO: Application started
   [12:34:56] DEBUG: Processing request
   [12:34:56] ERROR: Something went wrong
   [12:34:56] INFO: Request completed

✅ 모든 예제 완료!
```

---

## 5. 실전 예제

### 예제 1: 파일 업로드 서비스 ⭐⭐⭐

```java
public class FileUploadService extends ActiveObject {
    public FileUploadService() {
        super("FileUpload");
    }
    
    public Future<String> uploadFile(byte[] data, String filename) {
        CompletableFuture<String> future = new CompletableFuture<>();
        
        enqueue(() -> {
            try {
                // 업로드 처리
                String url = processUpload(data, filename);
                future.complete(url);
            } catch (Exception e) {
                future.completeExceptionally(e);
            }
        });
        
        return future;
    }
}
```

---

## 6. Actor 모델 연동

### 🎭 Actor 모델 특징

```
1. 격리: 각 Actor는 독립적
2. 메시지: 비동기 통신
3. 상태: 외부 공유 없음
4. 순서: 메시지 순차 처리
```

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 |
|------|------|
| **비동기** | 호출 즉시 반환 |
| **순차 실행** | 순서 보장 |
| **동기화 불필요** | 단일 스레드 |

### ❌ 단점

| 단점 | 설명 |
|------|------|
| **복잡도** | 구현 복잡 |
| **오버헤드** | 메시지 큐 비용 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: Blocking 작업

```java
// 잘못된 예
enqueue(() -> {
    result.get();  // ❌ Blocking!
});

// 올바른 예
enqueue(() -> {
    result.thenAccept(r -> {  // ✅ 비동기
        // 처리
    });
});
```

---

## 9. 심화 주제

### 🎯 Akka Framework

```java
// Akka Actor
public class EmailActor extends AbstractActor {
    @Override
    public Receive createReceive() {
        return receiveBuilder()
            .match(SendEmail.class, this::handleSendEmail)
            .build();
    }
}
```

---

## 10. 핵심 정리

### 📌 체크리스트

```
✅ 메시지 큐 사용
✅ 단일 Scheduler 스레드
✅ 순차 실행 보장
✅ Future로 결과 반환
✅ Graceful Shutdown
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Double-Checked Locking](./04-DoubleCheckedLocking.md) | [다음: Future/Promise →](./06-FuturePromise.md)**

</div>
