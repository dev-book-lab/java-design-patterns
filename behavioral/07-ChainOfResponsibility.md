# Chain of Responsibility Pattern (책임 연쇄 패턴)

> **"요청을 처리할 수 있는 객체를 찾을 때까지 전달하자"**

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
// 문제 1: 처리자가 강하게 결합
public class SupportSystem {
    public void handleRequest(Request request) {
        if (request.getPriority() == Priority.LOW) {
            juniorSupport.handle(request);
        } else if (request.getPriority() == Priority.MEDIUM) {
            seniorSupport.handle(request);
        } else if (request.getPriority() == Priority.HIGH) {
            manager.handle(request);
        } else {
            ceo.handle(request);
        }
        // 모든 처리자를 직접 알아야 함!
    }
}

// 문제 2: 처리 순서 변경이 어려움
public class RequestHandler {
    public void process(Request request) {
        // 1. 인증 확인
        if (!authCheck(request)) return;
        
        // 2. 권한 확인
        if (!permissionCheck(request)) return;
        
        // 3. 로깅
        log(request);
        
        // 4. 비즈니스 로직
        execute(request);
        
        // 순서를 바꾸거나 단계를 추가하려면 코드 수정!
    }
}

// 문제 3: 여러 처리자 시도
public class PaymentProcessor {
    public void pay(Payment payment) {
        try {
            creditCardProcessor.process(payment);
        } catch (Exception e) {
            try {
                paypalProcessor.process(payment);
            } catch (Exception e2) {
                try {
                    bankTransferProcessor.process(payment);
                } catch (Exception e3) {
                    System.out.println("모든 결제 실패");
                }
            }
        }
        // try-catch 지옥!
    }
}

// 문제 4: 동적 처리자 추가 불가
public class Logger {
    public void log(String message, Level level) {
        if (level == Level.DEBUG) {
            debugLogger.log(message);
        } else if (level == Level.INFO) {
            infoLogger.log(message);
        } else if (level == Level.ERROR) {
            errorLogger.log(message);
        }
        // 새 로그 레벨 추가? 코드 수정!
    }
}
```

### ⚡ 핵심 문제

1. **강한 결합**: 요청자가 모든 처리자를 알아야 함
2. **순서 변경 어려움**: 처리 순서가 코드에 고정
3. **확장 어려움**: 새 처리자 추가 시 코드 수정
4. **책임 분리 불명확**: 누가 처리할지 모호

---

## 2. 패턴 정의

### 📖 정의

> **요청을 보내는 객체와 받는 객체를 분리하고, 여러 객체에게 요청을 처리할 기회를 주는 패턴. 요청을 처리할 수 있는 객체를 찾을 때까지 체인을 따라 전달한다.**

### 🎯 목적

- **디커플링**: 요청자와 처리자 분리
- **동적 체인**: 런타임에 체인 구성
- **책임 분산**: 여러 객체가 처리 기회
- **유연성**: 순서 변경 용이

### 💡 핵심 아이디어

```java
// Before: 직접 호출
if (condition1) {
    handler1.handle();
} else if (condition2) {
    handler2.handle();
}

// After: 체인으로 전달
handler1.setNext(handler2).setNext(handler3);
handler1.handle(request); // 체인을 따라 전달!
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌─────────────┐
│   Client    │
└─────────────┘
       │
       │ sends request
       ▼
┌──────────────────┐
│    Handler       │  ← 추상 핸들러
├──────────────────┤
│ - next: Handler  │───┐
│ + setNext()      │   │ chain
│ + handleRequest()│   │
└──────────────────┘   │
       △               │
       │ extends       │
       │               │
┌──────────────────┐   │
│ConcreteHandler1  │   │
├──────────────────┤   │
│+ handleRequest() │◄──┘
└──────────────────┘
       │
       │ has next
       ▼
┌──────────────────┐
│ConcreteHandler2  │
├──────────────────┤
│+ handleRequest() │
└──────────────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **Handler** | 추상 핸들러 | `SupportHandler` |
| **ConcreteHandler** | 구체적 핸들러 | `JuniorSupport` |
| **Client** | 요청 발송 | `Customer` |
| **Request** | 요청 객체 | `SupportRequest` |

---

## 4. 구현 방법

### 기본 구현: 고객 지원 시스템 ⭐⭐⭐

```java
/**
 * Request: 지원 요청
 */
public class SupportRequest {
    private String description;
    private Priority priority;
    
    public enum Priority {
        LOW, MEDIUM, HIGH, CRITICAL
    }
    
    public SupportRequest(String description, Priority priority) {
        this.description = description;
        this.priority = priority;
    }
    
    public String getDescription() {
        return description;
    }
    
    public Priority getPriority() {
        return priority;
    }
}

/**
 * Handler: 추상 핸들러
 */
public abstract class SupportHandler {
    protected SupportHandler next;
    
    public SupportHandler setNext(SupportHandler next) {
        this.next = next;
        return next; // 체이닝을 위해
    }
    
    public void handle(SupportRequest request) {
        if (canHandle(request)) {
            processRequest(request);
        } else if (next != null) {
            System.out.println("   ↪️ " + getHandlerName() + " → 다음 핸들러로 전달");
            next.handle(request);
        } else {
            System.out.println("   ❌ 처리할 수 있는 핸들러가 없습니다");
        }
    }
    
    protected abstract boolean canHandle(SupportRequest request);
    protected abstract void processRequest(SupportRequest request);
    protected abstract String getHandlerName();
}

/**
 * ConcreteHandler 1: 주니어 지원
 */
public class JuniorSupport extends SupportHandler {
    
    @Override
    protected boolean canHandle(SupportRequest request) {
        return request.getPriority() == SupportRequest.Priority.LOW;
    }
    
    @Override
    protected void processRequest(SupportRequest request) {
        System.out.println("   👨‍💼 주니어 지원: " + request.getDescription() + " 처리 완료");
    }
    
    @Override
    protected String getHandlerName() {
        return "주니어 지원";
    }
}

/**
 * ConcreteHandler 2: 시니어 지원
 */
public class SeniorSupport extends SupportHandler {
    
    @Override
    protected boolean canHandle(SupportRequest request) {
        return request.getPriority() == SupportRequest.Priority.MEDIUM;
    }
    
    @Override
    protected void processRequest(SupportRequest request) {
        System.out.println("   👨‍💻 시니어 지원: " + request.getDescription() + " 처리 완료");
    }
    
    @Override
    protected String getHandlerName() {
        return "시니어 지원";
    }
}

/**
 * ConcreteHandler 3: 매니저
 */
public class ManagerSupport extends SupportHandler {
    
    @Override
    protected boolean canHandle(SupportRequest request) {
        return request.getPriority() == SupportRequest.Priority.HIGH;
    }
    
    @Override
    protected void processRequest(SupportRequest request) {
        System.out.println("   👔 매니저: " + request.getDescription() + " 처리 완료");
    }
    
    @Override
    protected String getHandlerName() {
        return "매니저";
    }
}

/**
 * ConcreteHandler 4: CEO
 */
public class CEOSupport extends SupportHandler {
    
    @Override
    protected boolean canHandle(SupportRequest request) {
        return request.getPriority() == SupportRequest.Priority.CRITICAL;
    }
    
    @Override
    protected void processRequest(SupportRequest request) {
        System.out.println("   🎩 CEO: " + request.getDescription() + " 직접 처리!");
    }
    
    @Override
    protected String getHandlerName() {
        return "CEO";
    }
}

/**
 * 사용 예제
 */
public class ChainOfResponsibilityExample {
    public static void main(String[] args) {
        // 체인 구성
        SupportHandler junior = new JuniorSupport();
        SupportHandler senior = new SeniorSupport();
        SupportHandler manager = new ManagerSupport();
        SupportHandler ceo = new CEOSupport();
        
        junior.setNext(senior)
              .setNext(manager)
              .setNext(ceo);
        
        System.out.println("=== 고객 지원 시스템 ===\n");
        
        // 다양한 우선순위의 요청
        System.out.println("📞 요청 1: LOW");
        junior.handle(new SupportRequest("로그인 문의", SupportRequest.Priority.LOW));
        
        System.out.println("\n📞 요청 2: MEDIUM");
        junior.handle(new SupportRequest("결제 오류", SupportRequest.Priority.MEDIUM));
        
        System.out.println("\n📞 요청 3: HIGH");
        junior.handle(new SupportRequest("시스템 장애", SupportRequest.Priority.HIGH));
        
        System.out.println("\n📞 요청 4: CRITICAL");
        junior.handle(new SupportRequest("보안 침해", SupportRequest.Priority.CRITICAL));
    }
}
```

**실행 결과:**
```
=== 고객 지원 시스템 ===

📞 요청 1: LOW
   👨‍💼 주니어 지원: 로그인 문의 처리 완료

📞 요청 2: MEDIUM
   ↪️ 주니어 지원 → 다음 핸들러로 전달
   👨‍💻 시니어 지원: 결제 오류 처리 완료

📞 요청 3: HIGH
   ↪️ 주니어 지원 → 다음 핸들러로 전달
   ↪️ 시니어 지원 → 다음 핸들러로 전달
   👔 매니저: 시스템 장애 처리 완료

📞 요청 4: CRITICAL
   ↪️ 주니어 지원 → 다음 핸들러로 전달
   ↪️ 시니어 지원 → 다음 핸들러로 전달
   ↪️ 매니저 → 다음 핸들러로 전달
   🎩 CEO: 보안 침해 직접 처리!
```

---

## 5. 실전 예제

### 예제 1: 로거 체인 ⭐⭐⭐

```java
/**
 * LogLevel
 */
public enum LogLevel {
    DEBUG(1), INFO(2), WARNING(3), ERROR(4);
    
    private int level;
    
    LogLevel(int level) {
        this.level = level;
    }
    
    public int getLevel() {
        return level;
    }
}

/**
 * Handler: 로거
 */
public abstract class Logger {
    protected LogLevel level;
    protected Logger next;
    
    public Logger(LogLevel level) {
        this.level = level;
    }
    
    public Logger setNext(Logger next) {
        this.next = next;
        return next;
    }
    
    public void log(String message, LogLevel messageLevel) {
        if (messageLevel.getLevel() >= level.getLevel()) {
            write(message);
        }
        
        if (next != null) {
            next.log(message, messageLevel);
        }
    }
    
    protected abstract void write(String message);
}

/**
 * ConcreteHandler: 콘솔 로거
 */
public class ConsoleLogger extends Logger {
    
    public ConsoleLogger(LogLevel level) {
        super(level);
    }
    
    @Override
    protected void write(String message) {
        System.out.println("🖥️ Console: " + message);
    }
}

/**
 * ConcreteHandler: 파일 로거
 */
public class FileLogger extends Logger {
    
    public FileLogger(LogLevel level) {
        super(level);
    }
    
    @Override
    protected void write(String message) {
        System.out.println("📄 File: " + message);
    }
}

/**
 * ConcreteHandler: 이메일 로거
 */
public class EmailLogger extends Logger {
    
    public EmailLogger(LogLevel level) {
        super(level);
    }
    
    @Override
    protected void write(String message) {
        System.out.println("📧 Email: " + message);
    }
}

/**
 * 사용 예제
 */
public class LoggerChainExample {
    public static void main(String[] args) {
        // 체인 구성: Console(DEBUG) → File(INFO) → Email(ERROR)
        Logger console = new ConsoleLogger(LogLevel.DEBUG);
        Logger file = new FileLogger(LogLevel.INFO);
        Logger email = new EmailLogger(LogLevel.ERROR);
        
        console.setNext(file).setNext(email);
        
        System.out.println("=== 로거 체인 ===\n");
        
        System.out.println("📝 DEBUG 메시지:");
        console.log("디버그 정보", LogLevel.DEBUG);
        
        System.out.println("\n📝 INFO 메시지:");
        console.log("일반 정보", LogLevel.INFO);
        
        System.out.println("\n📝 ERROR 메시지:");
        console.log("심각한 에러 발생!", LogLevel.ERROR);
    }
}
```

**실행 결과:**
```
=== 로거 체인 ===

📝 DEBUG 메시지:
🖥️ Console: 디버그 정보

📝 INFO 메시지:
🖥️ Console: 일반 정보
📄 File: 일반 정보

📝 ERROR 메시지:
🖥️ Console: 심각한 에러 발생!
📄 File: 심각한 에러 발생!
📧 Email: 심각한 에러 발생!
```

---

### 예제 2: HTTP 미들웨어 ⭐⭐⭐

```java
/**
 * Request
 */
public class HttpRequest {
    private String method;
    private String url;
    private String authToken;
    private Map<String, String> headers;
    
    public HttpRequest(String method, String url) {
        this.method = method;
        this.url = url;
        this.headers = new HashMap<>();
    }
    
    public void setAuthToken(String token) {
        this.authToken = token;
    }
    
    public String getAuthToken() {
        return authToken;
    }
    
    public String getMethod() {
        return method;
    }
    
    public String getUrl() {
        return url;
    }
}

/**
 * Handler: 미들웨어
 */
public abstract class Middleware {
    protected Middleware next;
    
    public Middleware setNext(Middleware next) {
        this.next = next;
        return next;
    }
    
    public boolean handle(HttpRequest request) {
        if (!check(request)) {
            return false;
        }
        
        if (next == null) {
            return true;
        }
        
        return next.handle(request);
    }
    
    protected abstract boolean check(HttpRequest request);
}

/**
 * ConcreteHandler: 인증 미들웨어
 */
public class AuthenticationMiddleware extends Middleware {
    
    @Override
    protected boolean check(HttpRequest request) {
        System.out.println("🔐 인증 확인 중...");
        
        if (request.getAuthToken() == null) {
            System.out.println("   ❌ 인증 실패: 토큰 없음");
            return false;
        }
        
        System.out.println("   ✅ 인증 성공");
        return true;
    }
}

/**
 * ConcreteHandler: 권한 미들웨어
 */
public class AuthorizationMiddleware extends Middleware {
    
    @Override
    protected boolean check(HttpRequest request) {
        System.out.println("🛡️ 권한 확인 중...");
        
        // 간단히 admin만 DELETE 허용
        if (request.getMethod().equals("DELETE") && 
            !request.getAuthToken().contains("admin")) {
            System.out.println("   ❌ 권한 없음");
            return false;
        }
        
        System.out.println("   ✅ 권한 확인");
        return true;
    }
}

/**
 * ConcreteHandler: 로깅 미들웨어
 */
public class LoggingMiddleware extends Middleware {
    
    @Override
    protected boolean check(HttpRequest request) {
        System.out.println("📝 요청 로깅:");
        System.out.println("   " + request.getMethod() + " " + request.getUrl());
        return true; // 항상 통과
    }
}

/**
 * ConcreteHandler: Rate Limiting 미들웨어
 */
public class RateLimitMiddleware extends Middleware {
    private int requestCount = 0;
    private static final int MAX_REQUESTS = 3;
    
    @Override
    protected boolean check(HttpRequest request) {
        System.out.println("⏱️ Rate Limit 확인 중...");
        requestCount++;
        
        if (requestCount > MAX_REQUESTS) {
            System.out.println("   ❌ 요청 한도 초과 (" + requestCount + "/" + MAX_REQUESTS + ")");
            return false;
        }
        
        System.out.println("   ✅ 요청 허용 (" + requestCount + "/" + MAX_REQUESTS + ")");
        return true;
    }
}

/**
 * 사용 예제
 */
public class MiddlewareExample {
    public static void main(String[] args) {
        // 미들웨어 체인 구성
        Middleware chain = new RateLimitMiddleware();
        chain.setNext(new AuthenticationMiddleware())
             .setNext(new AuthorizationMiddleware())
             .setNext(new LoggingMiddleware());
        
        System.out.println("=== HTTP 미들웨어 체인 ===\n");
        
        // 요청 1: 정상
        System.out.println("📡 요청 1: GET /users");
        HttpRequest request1 = new HttpRequest("GET", "/users");
        request1.setAuthToken("user-token");
        boolean result1 = chain.handle(request1);
        System.out.println("결과: " + (result1 ? "✅ 성공" : "❌ 실패"));
        
        // 요청 2: 권한 없음
        System.out.println("\n📡 요청 2: DELETE /users/1");
        HttpRequest request2 = new HttpRequest("DELETE", "/users/1");
        request2.setAuthToken("user-token");
        boolean result2 = chain.handle(request2);
        System.out.println("결과: " + (result2 ? "✅ 성공" : "❌ 실패"));
        
        // 요청 3: 인증 실패
        System.out.println("\n📡 요청 3: GET /products");
        HttpRequest request3 = new HttpRequest("GET", "/products");
        boolean result3 = chain.handle(request3);
        System.out.println("결과: " + (result3 ? "✅ 성공" : "❌ 실패"));
    }
}
```

---

### 예제 3: 승인 체인 ⭐⭐

```java
/**
 * Request: 구매 요청
 */
public class PurchaseRequest {
    private String item;
    private double amount;
    
    public PurchaseRequest(String item, double amount) {
        this.item = item;
        this.amount = amount;
    }
    
    public String getItem() {
        return item;
    }
    
    public double getAmount() {
        return amount;
    }
}

/**
 * Handler: 승인자
 */
public abstract class Approver {
    protected Approver next;
    protected double approvalLimit;
    
    public Approver(double approvalLimit) {
        this.approvalLimit = approvalLimit;
    }
    
    public Approver setNext(Approver next) {
        this.next = next;
        return next;
    }
    
    public void approve(PurchaseRequest request) {
        if (request.getAmount() <= approvalLimit) {
            processApproval(request);
        } else if (next != null) {
            System.out.println("   ↪️ 상위 결재자에게 전달");
            next.approve(request);
        } else {
            System.out.println("   ❌ 승인 거부: 금액 초과");
        }
    }
    
    protected abstract void processApproval(PurchaseRequest request);
}

/**
 * ConcreteHandler: 팀장
 */
public class TeamLeader extends Approver {
    
    public TeamLeader() {
        super(1000); // $1,000까지 승인
    }
    
    @Override
    protected void processApproval(PurchaseRequest request) {
        System.out.println("   👨‍💼 팀장 승인: " + request.getItem() + 
                " ($" + request.getAmount() + ")");
    }
}

/**
 * ConcreteHandler: 부서장
 */
public class DepartmentHead extends Approver {
    
    public DepartmentHead() {
        super(5000); // $5,000까지 승인
    }
    
    @Override
    protected void processApproval(PurchaseRequest request) {
        System.out.println("   👔 부서장 승인: " + request.getItem() + 
                " ($" + request.getAmount() + ")");
    }
}

/**
 * ConcreteHandler: CEO
 */
public class CEO extends Approver {
    
    public CEO() {
        super(Double.MAX_VALUE); // 무제한
    }
    
    @Override
    protected void processApproval(PurchaseRequest request) {
        System.out.println("   🎩 CEO 승인: " + request.getItem() + 
                " ($" + request.getAmount() + ")");
    }
}

/**
 * 사용 예제
 */
public class ApprovalChainExample {
    public static void main(String[] args) {
        // 승인 체인
        Approver teamLeader = new TeamLeader();
        Approver deptHead = new DepartmentHead();
        Approver ceo = new CEO();
        
        teamLeader.setNext(deptHead).setNext(ceo);
        
        System.out.println("=== 구매 승인 시스템 ===\n");
        
        System.out.println("📝 요청 1: $500");
        teamLeader.approve(new PurchaseRequest("노트북", 500));
        
        System.out.println("\n📝 요청 2: $3000");
        teamLeader.approve(new PurchaseRequest("서버", 3000));
        
        System.out.println("\n📝 요청 3: $10000");
        teamLeader.approve(new PurchaseRequest("사무실", 10000));
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **디커플링** | 요청자와 처리자 분리 | 고객-지원팀 |
| **동적 체인** | 런타임에 체인 변경 | 미들웨어 추가 |
| **단일 책임** | 각 핸들러 독립적 | 인증/권한 분리 |
| **순서 제어** | 처리 순서 유연 | 로거 체인 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **처리 보장 없음** | 끝까지 전달 가능 | 기본 핸들러 |
| **디버깅 어려움** | 체인 추적 힘듦 | 로깅 강화 |
| **성능 오버헤드** | 여러 핸들러 거침 | 필요시에만 |

---

## 7. 안티패턴

### ❌ 안티패턴: 순환 체인

```java
// 잘못된 예
handler1.setNext(handler2);
handler2.setNext(handler1); // 순환!
// 무한 루프 발생!
```

**해결:**
```java
// 체인 검증
public Approver setNext(Approver next) {
    if (isInChain(next)) {
        throw new IllegalArgumentException("순환 체인!");
    }
    this.next = next;
    return next;
}
```

---

## 8. 핵심 정리

### 📌 Chain of Responsibility 체크리스트

```
✅ Handler 추상 클래스
✅ setNext() 메서드
✅ handle() 메서드
✅ ConcreteHandler 구현
✅ 체인 구성
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 다단계 처리 | ⭐⭐⭐ | 순차 처리 |
| 동적 처리자 | ⭐⭐⭐ | 런타임 변경 |
| 미들웨어 | ⭐⭐⭐ | HTTP 파이프라인 |
| 승인 시스템 | ⭐⭐⭐ | 계층적 승인 |

### 💡 핵심 포인트

1. **요청을 체인으로**
2. **처리자 독립적**
3. **동적 구성 가능**
4. **순서 중요**

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: State](./06-State.md) | [다음: Mediator →](08-Mediator.md)**

</div>
