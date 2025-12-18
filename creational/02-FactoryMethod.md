# Factory Method Pattern (팩토리 메서드 패턴)

> **"객체 생성을 서브클래스에 위임하여 확장성을 높이자"**

[← 이전: Singleton](./01-Singleton.md) | [목차로 돌아가기](../README.md) | [다음: Builder →](./03-Builder.md)

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
// 문제 1: 새로운 타입 추가 시 모든 곳을 수정해야 함
public class LogisticsApp {
    public void deliver(String type, String destination) {
        if (type.equals("TRUCK")) {
            Truck truck = new Truck();
            truck.deliver(destination);
        } else if (type.equals("SHIP")) {
            Ship ship = new Ship();
            ship.deliver(destination);
        } else if (type.equals("PLANE")) {  // 새로 추가!
            Plane plane = new Plane();      // 여기도 수정!
            plane.deliver(destination);
        }
        // 새 운송 수단 추가할 때마다 이 코드 수정 필요!
    }
}

// 문제 2: 클라이언트가 구체 클래스를 직접 알아야 함
public class NotificationService {
    public void send(String type, String message) {
        if (type.equals("EMAIL")) {
            EmailNotification email = new EmailNotification();  // 직접 생성
            email.send(message);
        } else if (type.equals("SMS")) {
            SMSNotification sms = new SMSNotification();        // 직접 생성
            sms.send(message);
        } else if (type.equals("PUSH")) {
            PushNotification push = new PushNotification();     // 직접 생성
            push.send(message);
        }
        // 구체 클래스에 강하게 결합됨!
    }
}

// 문제 3: 생성 로직이 여러 곳에 중복됨
public class PaymentProcessor {
    public void process(String method) {
        if (method.equals("CARD")) {
            CardPayment payment = new CardPayment();  // 중복 1
            payment.pay();
        }
    }
}

public class RefundProcessor {
    public void refund(String method) {
        if (method.equals("CARD")) {
            CardPayment payment = new CardPayment();  // 중복 2
            payment.refund();
        }
    }
}
```

### ⚡ 핵심 문제

1. **OCP 위반**: 새로운 타입 추가 시 기존 코드 수정 필요
2. **높은 결합도**: 클라이언트가 구체 클래스를 직접 알아야 함
3. **중복 코드**: 객체 생성 로직이 여러 곳에 산재

---

## 2. 패턴 정의

### 📖 정의

> **객체를 생성하는 인터페이스를 정의하되, 어떤 클래스의 인스턴스를 생성할지는 서브클래스가 결정하도록 하는 패턴**

### 🎯 목적

- 객체 생성 로직을 **캡슐화**
- 구체 클래스가 아닌 **인터페이스에 의존**
- **확장에는 열려있고 수정에는 닫힌** 구조 (OCP)

### 💡 핵심 아이디어

```java
// Before: 클라이언트가 직접 생성
Product product = new ConcreteProduct();  // 구체 클래스에 의존

// After: 팩토리 메서드로 생성
Product product = creator.createProduct();  // 인터페이스에 의존
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
           ┌──────────────────┐
           │   Creator        │  ← 추상 클래스/인터페이스
           ├──────────────────┤
           │ + factoryMethod()│  ← 추상 메서드 (팩토리 메서드)
           │ + someOperation()│  ← 템플릿 메서드
           └──────────────────┘
                    △
                    │
       ┌────────────┴────────────┐
       │                         │
┌──────────────┐         ┌──────────────┐
│ConcreteA     │         │ConcreteB     │  ← 구체 Creator
├──────────────┤         ├──────────────┤
│factoryMethod │         │factoryMethod │  ← 구체 Product 생성
└──────────────┘         └──────────────┘
       │                         │
       │ creates                 │ creates
       ▼                         ▼
┌──────────────┐         ┌──────────────┐
│ProductA      │         │ProductB      │  ← 구체 Product
└──────────────┘         └──────────────┘
           △
           │
    ┌──────────────┐
    │   Product    │  ← 제품 인터페이스
    └──────────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **Product** | 생성될 객체의 인터페이스 | `Transport` |
| **ConcreteProduct** | Product 구현체 | `Truck`, `Ship` |
| **Creator** | 팩토리 메서드 선언 | `Logistics` |
| **ConcreteCreator** | 팩토리 메서드 구현 | `RoadLogistics`, `SeaLogistics` |

---

## 4. 구현 방법

### 기본 구현: 물류 배송 시스템 ⭐⭐⭐

#### Step 1: Product 인터페이스 정의

```java
/**
 * 운송 수단 인터페이스
 * - 모든 운송 수단이 구현해야 할 공통 메서드 정의
 */
public interface Transport {
    void deliver(String destination);
    int getCapacity();
    double getCost(int distance);
}
```

#### Step 2: ConcreteProduct 구현

```java
/**
 * 트럭 운송
 */
public class Truck implements Transport {
    @Override
    public void deliver(String destination) {
        System.out.println("🚚 트럭으로 " + destination + "까지 육로 배송");
    }
    
    @Override
    public int getCapacity() {
        return 1000; // kg
    }
    
    @Override
    public double getCost(int distance) {
        return distance * 1.5; // km당 1.5원
    }
}

/**
 * 선박 운송
 */
public class Ship implements Transport {
    @Override
    public void deliver(String destination) {
        System.out.println("🚢 선박으로 " + destination + "까지 해상 배송");
    }
    
    @Override
    public int getCapacity() {
        return 10000; // kg
    }
    
    @Override
    public double getCost(int distance) {
        return distance * 0.5; // km당 0.5원
    }
}

/**
 * 항공 운송
 */
public class Plane implements Transport {
    @Override
    public void deliver(String destination) {
        System.out.println("✈️ 항공기로 " + destination + "까지 항공 배송");
    }
    
    @Override
    public int getCapacity() {
        return 500; // kg
    }
    
    @Override
    public double getCost(int distance) {
        return distance * 5.0; // km당 5.0원
    }
}
```

#### Step 3: Creator 추상 클래스

```java
/**
 * 물류 관리 추상 클래스
 * - 팩토리 메서드(createTransport)는 서브클래스에서 구현
 * - 템플릿 메서드(planDelivery)는 공통 알고리즘 정의
 */
public abstract class Logistics {
    
    // 팩토리 메서드: 서브클래스가 구현
    protected abstract Transport createTransport();
    
    // 템플릿 메서드: 공통 비즈니스 로직
    public void planDelivery(String origin, String destination, int distance) {
        System.out.println("\n=== 배송 계획 수립 ===");
        System.out.println("출발지: " + origin);
        System.out.println("목적지: " + destination);
        System.out.println("거리: " + distance + "km");
        
        // 팩토리 메서드로 운송 수단 생성
        Transport transport = createTransport();
        
        // 운송 수단 정보 출력
        System.out.println("운송 능력: " + transport.getCapacity() + "kg");
        System.out.println("예상 비용: " + transport.getCost(distance) + "원");
        
        // 배송 실행
        transport.deliver(destination);
    }
}
```

#### Step 4: ConcreteCreator 구현

```java
/**
 * 육로 물류
 */
public class RoadLogistics extends Logistics {
    @Override
    protected Transport createTransport() {
        return new Truck();  // 트럭 생성
    }
}

/**
 * 해상 물류
 */
public class SeaLogistics extends Logistics {
    @Override
    protected Transport createTransport() {
        return new Ship();   // 선박 생성
    }
}

/**
 * 항공 물류
 */
public class AirLogistics extends Logistics {
    @Override
    protected Transport createTransport() {
        return new Plane();  // 항공기 생성
    }
}
```

#### Step 5: 클라이언트 코드

```java
/**
 * 물류 시스템 사용
 */
public class FactoryMethodExample {
    public static void main(String[] args) {
        // 1. 육로 배송
        Logistics roadLogistics = new RoadLogistics();
        roadLogistics.planDelivery("서울", "부산", 400);
        
        // 2. 해상 배송
        Logistics seaLogistics = new SeaLogistics();
        seaLogistics.planDelivery("부산", "제주", 300);
        
        // 3. 항공 배송
        Logistics airLogistics = new AirLogistics();
        airLogistics.planDelivery("인천", "뉴욕", 11000);
        
        // 4. 동적 선택
        System.out.println("\n=== 거리에 따른 최적 운송 수단 ===");
        planOptimalDelivery("서울", "대전", 150);
        planOptimalDelivery("서울", "제주", 450);
        planOptimalDelivery("서울", "LA", 10000);
    }
    
    // 거리에 따라 최적의 물류 방식 선택
    private static void planOptimalDelivery(String origin, String dest, int distance) {
        Logistics logistics;
        
        if (distance < 300) {
            logistics = new RoadLogistics();      // 단거리: 육로
        } else if (distance < 1000) {
            logistics = new SeaLogistics();       // 중거리: 해상
        } else {
            logistics = new AirLogistics();       // 장거리: 항공
        }
        
        logistics.planDelivery(origin, dest, distance);
    }
}
```

**실행 결과:**
```
=== 배송 계획 수립 ===
출발지: 서울
목적지: 부산
거리: 400km
운송 능력: 1000kg
예상 비용: 600.0원
🚚 트럭으로 부산까지 육로 배송

=== 배송 계획 수립 ===
출발지: 부산
목적지: 제주
거리: 300km
운송 능력: 10000kg
예상 비용: 150.0원
🚢 선박으로 제주까지 해상 배송

=== 배송 계획 수립 ===
출발지: 인천
목적지: 뉴욕
거리: 11000km
운송 능력: 500kg
예상 비용: 55000.0원
✈️ 항공기로 뉴욕까지 항공 배송

=== 거리에 따른 최적 운송 수단 ===
...
```

---

### 확장: 새로운 운송 수단 추가 ⭐⭐

```java
/**
 * 드론 배송 추가
 * - 기존 코드 수정 없이 확장!
 */
public class Drone implements Transport {
    @Override
    public void deliver(String destination) {
        System.out.println("🚁 드론으로 " + destination + "까지 배송");
    }
    
    @Override
    public int getCapacity() {
        return 5; // kg
    }
    
    @Override
    public double getCost(int distance) {
        return distance * 3.0; // km당 3.0원
    }
}

/**
 * 드론 물류
 */
public class DroneLogistics extends Logistics {
    @Override
    protected Transport createTransport() {
        return new Drone();  // 드론 생성
    }
}

// 사용
public class ExtensionExample {
    public static void main(String[] args) {
        Logistics droneLogistics = new DroneLogistics();
        droneLogistics.planDelivery("서울", "강남", 20);
        // 기존 코드 수정 없이 새 기능 추가 완료!
    }
}
```

---

## 5. 실전 예제

### 예제 1: 알림(Notification) 시스템 ⭐⭐⭐

```java
// Product 인터페이스
public interface Notification {
    void send(String recipient, String message);
    boolean isDelivered();
    String getType();
}

// ConcreteProduct 1: 이메일
public class EmailNotification implements Notification {
    private boolean delivered = false;
    
    @Override
    public void send(String recipient, String message) {
        System.out.println("📧 이메일 발송");
        System.out.println("수신자: " + recipient);
        System.out.println("내용: " + message);
        // 실제 이메일 전송 로직
        delivered = true;
    }
    
    @Override
    public boolean isDelivered() {
        return delivered;
    }
    
    @Override
    public String getType() {
        return "EMAIL";
    }
}

// ConcreteProduct 2: SMS
public class SMSNotification implements Notification {
    private boolean delivered = false;
    
    @Override
    public void send(String recipient, String message) {
        System.out.println("📱 SMS 발송");
        System.out.println("전화번호: " + recipient);
        System.out.println("내용: " + message.substring(0, Math.min(message.length(), 80)));
        // 실제 SMS 전송 로직
        delivered = true;
    }
    
    @Override
    public boolean isDelivered() {
        return delivered;
    }
    
    @Override
    public String getType() {
        return "SMS";
    }
}

// ConcreteProduct 3: Push
public class PushNotification implements Notification {
    private boolean delivered = false;
    
    @Override
    public void send(String recipient, String message) {
        System.out.println("🔔 푸시 알림 발송");
        System.out.println("디바이스 ID: " + recipient);
        System.out.println("내용: " + message);
        // 실제 푸시 전송 로직
        delivered = true;
    }
    
    @Override
    public boolean isDelivered() {
        return delivered;
    }
    
    @Override
    public String getType() {
        return "PUSH";
    }
}

// Creator 추상 클래스
public abstract class NotificationService {
    
    // 팩토리 메서드
    protected abstract Notification createNotification();
    
    // 템플릿 메서드: 알림 전송 프로세스
    public void notify(String recipient, String message) {
        System.out.println("\n=== 알림 발송 시작 ===");
        
        // 1. 알림 생성
        Notification notification = createNotification();
        System.out.println("알림 타입: " + notification.getType());
        
        // 2. 메시지 유효성 검증
        if (message == null || message.isEmpty()) {
            System.out.println("❌ 메시지가 비어있습니다.");
            return;
        }
        
        // 3. 알림 발송
        notification.send(recipient, message);
        
        // 4. 발송 결과 확인
        if (notification.isDelivered()) {
            System.out.println("✅ 알림이 성공적으로 전송되었습니다.");
        } else {
            System.out.println("❌ 알림 전송에 실패했습니다.");
        }
    }
}

// ConcreteCreator 1: 이메일 서비스
public class EmailNotificationService extends NotificationService {
    @Override
    protected Notification createNotification() {
        return new EmailNotification();
    }
}

// ConcreteCreator 2: SMS 서비스
public class SMSNotificationService extends NotificationService {
    @Override
    protected Notification createNotification() {
        return new SMSNotification();
    }
}

// ConcreteCreator 3: 푸시 서비스
public class PushNotificationService extends NotificationService {
    @Override
    protected Notification createNotification() {
        return new PushNotification();
    }
}

// 사용 예제
public class NotificationExample {
    public static void main(String[] args) {
        // 1. 이메일 알림
        NotificationService emailService = new EmailNotificationService();
        emailService.notify("user@example.com", "회원가입을 환영합니다!");
        
        // 2. SMS 알림
        NotificationService smsService = new SMSNotificationService();
        smsService.notify("010-1234-5678", "인증번호: 123456");
        
        // 3. 푸시 알림
        NotificationService pushService = new PushNotificationService();
        pushService.notify("device-token-abc123", "새로운 메시지가 도착했습니다.");
        
        // 4. 사용자 선호도에 따른 알림
        notifyUser("user@example.com", "주문이 완료되었습니다.", "EMAIL");
        notifyUser("010-9876-5432", "배송이 시작되었습니다.", "SMS");
    }
    
    private static void notifyUser(String contact, String message, String preferredType) {
        NotificationService service;
        
        switch (preferredType) {
            case "EMAIL":
                service = new EmailNotificationService();
                break;
            case "SMS":
                service = new SMSNotificationService();
                break;
            case "PUSH":
                service = new PushNotificationService();
                break;
            default:
                service = new EmailNotificationService(); // 기본값
        }
        
        service.notify(contact, message);
    }
}
```

**실행 결과:**
```
=== 알림 발송 시작 ===
알림 타입: EMAIL
📧 이메일 발송
수신자: user@example.com
내용: 회원가입을 환영합니다!
✅ 알림이 성공적으로 전송되었습니다.

=== 알림 발송 시작 ===
알림 타입: SMS
📱 SMS 발송
전화번호: 010-1234-5678
내용: 인증번호: 123456
✅ 알림이 성공적으로 전송되었습니다.

=== 알림 발송 시작 ===
알림 타입: PUSH
🔔 푸시 알림 발송
디바이스 ID: device-token-abc123
내용: 새로운 메시지가 도착했습니다.
✅ 알림이 성공적으로 전송되었습니다.
```

---

### 예제 2: 결제(Payment) 시스템 ⭐⭐⭐

```java
// Product 인터페이스
public interface Payment {
    boolean processPayment(double amount);
    boolean refund(double amount);
    String getPaymentMethod();
    double getTransactionFee(double amount);
}

// ConcreteProduct 1: 신용카드
public class CreditCardPayment implements Payment {
    
    @Override
    public boolean processPayment(double amount) {
        System.out.println("💳 신용카드 결제 처리");
        System.out.println("금액: " + amount + "원");
        System.out.println("수수료: " + getTransactionFee(amount) + "원");
        // 실제 결제 로직
        return true;
    }
    
    @Override
    public boolean refund(double amount) {
        System.out.println("💳 신용카드 환불 처리");
        System.out.println("환불 금액: " + amount + "원");
        return true;
    }
    
    @Override
    public String getPaymentMethod() {
        return "신용카드";
    }
    
    @Override
    public double getTransactionFee(double amount) {
        return amount * 0.03; // 3% 수수료
    }
}

// ConcreteProduct 2: 계좌이체
public class BankTransferPayment implements Payment {
    
    @Override
    public boolean processPayment(double amount) {
        System.out.println("🏦 계좌이체 결제 처리");
        System.out.println("금액: " + amount + "원");
        System.out.println("수수료: " + getTransactionFee(amount) + "원");
        return true;
    }
    
    @Override
    public boolean refund(double amount) {
        System.out.println("🏦 계좌이체 환불 처리");
        System.out.println("환불 금액: " + amount + "원");
        return true;
    }
    
    @Override
    public String getPaymentMethod() {
        return "계좌이체";
    }
    
    @Override
    public double getTransactionFee(double amount) {
        return 500.0; // 고정 수수료
    }
}

// ConcreteProduct 3: 간편결제
public class SimplePayment implements Payment {
    
    @Override
    public boolean processPayment(double amount) {
        System.out.println("📱 간편결제 처리");
        System.out.println("금액: " + amount + "원");
        System.out.println("수수료: " + getTransactionFee(amount) + "원");
        return true;
    }
    
    @Override
    public boolean refund(double amount) {
        System.out.println("📱 간편결제 환불 처리");
        System.out.println("환불 금액: " + amount + "원");
        return true;
    }
    
    @Override
    public String getPaymentMethod() {
        return "간편결제";
    }
    
    @Override
    public double getTransactionFee(double amount) {
        return amount * 0.015; // 1.5% 수수료
    }
}

// Creator 추상 클래스
public abstract class PaymentProcessor {
    
    // 팩토리 메서드
    protected abstract Payment createPayment();
    
    // 템플릿 메서드: 결제 처리 프로세스
    public boolean processOrder(String orderId, double amount) {
        System.out.println("\n=== 주문 결제 처리 ===");
        System.out.println("주문 번호: " + orderId);
        
        // 1. 결제 수단 생성
        Payment payment = createPayment();
        System.out.println("결제 방법: " + payment.getPaymentMethod());
        
        // 2. 금액 검증
        if (amount <= 0) {
            System.out.println("❌ 올바르지 않은 금액입니다.");
            return false;
        }
        
        // 3. 총 결제 금액 계산
        double totalAmount = amount + payment.getTransactionFee(amount);
        System.out.println("총 결제 금액: " + totalAmount + "원");
        
        // 4. 결제 처리
        boolean success = payment.processPayment(amount);
        
        if (success) {
            System.out.println("✅ 결제가 완료되었습니다.");
        } else {
            System.out.println("❌ 결제에 실패했습니다.");
        }
        
        return success;
    }
    
    // 템플릿 메서드: 환불 처리 프로세스
    public boolean processRefund(String orderId, double amount) {
        System.out.println("\n=== 주문 환불 처리 ===");
        System.out.println("주문 번호: " + orderId);
        
        Payment payment = createPayment();
        System.out.println("환불 방법: " + payment.getPaymentMethod());
        
        boolean success = payment.refund(amount);
        
        if (success) {
            System.out.println("✅ 환불이 완료되었습니다.");
        } else {
            System.out.println("❌ 환불에 실패했습니다.");
        }
        
        return success;
    }
}

// ConcreteCreator
public class CreditCardProcessor extends PaymentProcessor {
    @Override
    protected Payment createPayment() {
        return new CreditCardPayment();
    }
}

public class BankTransferProcessor extends PaymentProcessor {
    @Override
    protected Payment createPayment() {
        return new BankTransferPayment();
    }
}

public class SimplePaymentProcessor extends PaymentProcessor {
    @Override
    protected Payment createPayment() {
        return new SimplePayment();
    }
}

// 사용 예제
public class PaymentExample {
    public static void main(String[] args) {
        // 1. 신용카드 결제
        PaymentProcessor cardProcessor = new CreditCardProcessor();
        cardProcessor.processOrder("ORDER-001", 50000);
        
        // 2. 계좌이체 결제
        PaymentProcessor bankProcessor = new BankTransferProcessor();
        bankProcessor.processOrder("ORDER-002", 100000);
        
        // 3. 간편결제
        PaymentProcessor simpleProcessor = new SimplePaymentProcessor();
        simpleProcessor.processOrder("ORDER-003", 30000);
        
        // 4. 환불 처리
        cardProcessor.processRefund("ORDER-001", 50000);
    }
}
```

**실행 결과:**
```
=== 주문 결제 처리 ===
주문 번호: ORDER-001
결제 방법: 신용카드
총 결제 금액: 51500.0원
💳 신용카드 결제 처리
금액: 50000.0원
수수료: 1500.0원
✅ 결제가 완료되었습니다.

=== 주문 결제 처리 ===
주문 번호: ORDER-002
결제 방법: 계좌이체
총 결제 금액: 100500.0원
🏦 계좌이체 결제 처리
금액: 100000.0원
수수료: 500.0원
✅ 결제가 완료되었습니다.
...
```

---

### 예제 3: 문서(Document) 생성 시스템 ⭐⭐

```java
// Product 인터페이스
public interface Document {
    void open();
    void save(String content);
    void close();
    String getFormat();
}

// ConcreteProduct: PDF
public class PDFDocument implements Document {
    private String content;
    
    @Override
    public void open() {
        System.out.println("📄 PDF 문서 열기");
    }
    
    @Override
    public void save(String content) {
        this.content = content;
        System.out.println("💾 PDF 문서 저장: " + content.length() + " bytes");
    }
    
    @Override
    public void close() {
        System.out.println("❌ PDF 문서 닫기");
    }
    
    @Override
    public String getFormat() {
        return "PDF";
    }
}

// ConcreteProduct: Word
public class WordDocument implements Document {
    private String content;
    
    @Override
    public void open() {
        System.out.println("📝 Word 문서 열기");
    }
    
    @Override
    public void save(String content) {
        this.content = content;
        System.out.println("💾 Word 문서 저장: " + content.length() + " bytes");
    }
    
    @Override
    public void close() {
        System.out.println("❌ Word 문서 닫기");
    }
    
    @Override
    public String getFormat() {
        return "DOCX";
    }
}

// Creator
public abstract class Application {
    
    protected abstract Document createDocument();
    
    public void newDocument(String initialContent) {
        System.out.println("\n=== 새 문서 생성 ===");
        
        Document doc = createDocument();
        System.out.println("문서 형식: " + doc.getFormat());
        
        doc.open();
        doc.save(initialContent);
        doc.close();
    }
}

// ConcreteCreator
public class PDFApplication extends Application {
    @Override
    protected Document createDocument() {
        return new PDFDocument();
    }
}

public class WordApplication extends Application {
    @Override
    protected Document createDocument() {
        return new WordDocument();
    }
}

// 사용
public class DocumentExample {
    public static void main(String[] args) {
        Application pdfApp = new PDFApplication();
        pdfApp.newDocument("This is a PDF document.");
        
        Application wordApp = new WordApplication();
        wordApp.newDocument("This is a Word document.");
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **OCP 준수** | 확장에 열려있고 수정에 닫힘 | 새 운송 수단 추가 시 기존 코드 수정 불필요 |
| **SRP 준수** | 생성 책임 분리 | Creator는 생성만, Product는 비즈니스 로직만 |
| **느슨한 결합** | 구체 클래스 대신 인터페이스 의존 | `Transport transport = createTransport()` |
| **재사용성** | 공통 로직 재사용 (템플릿 메서드) | `planDelivery()`는 모든 Logistics 공유 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **클래스 증가** | Product마다 Creator 필요 | 단순한 경우 Simple Factory 고려 |
| **복잡도 증가** | 상속 구조 이해 필요 | 문서화, 네이밍 규칙 |
| **과도한 추상화** | 간단한 생성에는 과함 | 규모에 맞게 선택 |

---

## 7. 안티패턴

### ❌ 안티패턴 1: if-else로 팩토리 메서드 구현

```java
// 잘못된 예: 팩토리 메서드 안에서 분기
public abstract class BadLogistics {
    protected Transport createTransport(String type) {  // 추상 메서드가 아님!
        if (type.equals("TRUCK")) {
            return new Truck();
        } else if (type.equals("SHIP")) {
            return new Ship();
        } else {
            return new Plane();
        }
        // 새 타입 추가 시 이 메서드 수정 필요 (OCP 위반!)
    }
}
```

**해결:**
```java
// 올바른 예: 서브클래스가 결정
public abstract class GoodLogistics {
    protected abstract Transport createTransport();  // 추상 메서드
}

public class RoadLogistics extends GoodLogistics {
    @Override
    protected Transport createTransport() {
        return new Truck();  // 서브클래스가 결정
    }
}
```

---

### ❌ 안티패턴 2: Creator 없이 Product만 사용

```java
// 잘못된 예: 클라이언트가 직접 생성
public class BadClient {
    public void deliver() {
        Transport transport;
        
        String type = getTransportType();
        if (type.equals("TRUCK")) {
            transport = new Truck();  // 구체 클래스 의존
        } else {
            transport = new Ship();
        }
        
        transport.deliver("목적지");
        // 새 타입 추가 시 이 코드 수정 필요!
    }
}
```

**해결:**
```java
// 올바른 예: Creator 사용
public class GoodClient {
    private Logistics logistics;
    
    public GoodClient(Logistics logistics) {
        this.logistics = logistics;
    }
    
    public void deliver() {
        logistics.planDelivery("출발", "목적지", 100);
        // Transport 타입 몰라도 됨!
    }
}
```

---

### ❌ 안티패턴 3: 너무 많은 책임

```java
// 잘못된 예: Creator가 너무 많은 일을 함
public class BadCreator {
    public Product createProduct() {
        Product product = new ConcreteProduct();
        
        // 생성 후 설정까지 다 함 (SRP 위반!)
        product.setConfig(loadConfig());
        product.initialize();
        product.validate();
        product.log();
        
        return product;
    }
}
```

**해결:**
```java
// 올바른 예: 생성만 담당
public abstract class GoodCreator {
    protected abstract Product createProduct();
    
    // 설정은 별도 메서드로 분리
    protected void configureProduct(Product product) {
        product.setConfig(loadConfig());
    }
}
```

---

## 8. 핵심 정리

### 📌 Factory Method 패턴 체크리스트

```
✅ Product 인터페이스 정의
✅ ConcreteProduct 구현
✅ Creator 추상 클래스/인터페이스 정의
✅ factoryMethod()는 추상 메서드로
✅ 템플릿 메서드로 공통 로직 구현
✅ ConcreteCreator에서 factoryMethod() 구현
✅ 클라이언트는 Creator만 사용
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 생성할 객체 타입을 미리 모름 | ⭐⭐⭐ | 런타임에 결정 |
| 객체 생성 로직이 복잡함 | ⭐⭐⭐ | 캡슐화 필요 |
| 확장 가능성이 높음 | ⭐⭐⭐ | OCP 중요 |
| 생성과 사용을 분리하고 싶음 | ⭐⭐⭐ | SRP 준수 |

### 💡 핵심 포인트

1. **객체 생성을 서브클래스에 위임**
2. **인터페이스에 의존, 구현에 의존 X**
3. **확장에 열려있고 수정에 닫힘 (OCP)**
4. **템플릿 메서드와 함께 사용하면 강력**

### 🔥 실무 팁

```java
// 1. Simple Factory가 더 나은 경우
// - 타입이 고정적이고 적을 때
// - 확장 가능성이 낮을 때
public class SimpleFactory {
    public static Product create(String type) {
        switch (type) {
            case "A": return new ProductA();
            case "B": return new ProductB();
            default: throw new IllegalArgumentException();
        }
    }
}

// 2. Factory Method가 더 나은 경우
// - 타입이 동적이거나 많을 때
// - 확장 가능성이 높을 때
// - 생성 로직이 복잡할 때
```

---

## 🎓 연습 문제

### 문제 1: 게임 캐릭터 생성 시스템
```java
/**
 * 요구사항:
 * 1. Character 인터페이스 정의 (attack, defend)
 * 2. Warrior, Mage, Archer 구현
 * 3. CharacterFactory로 팩토리 메서드 패턴 구현
 */
```

### 문제 2: 로그 파서 시스템
```java
/**
 * 요구사항:
 * 1. LogParser 인터페이스 정의
 * 2. JSONParser, XMLParser, CSVParser 구현
 * 3. 파일 확장자에 따라 적절한 파서 생성
 */
```

### 문제 3: 확장 과제
```java
/**
 * 물류 시스템에 다음 기능 추가:
 * 1. 배송 추적 기능
 * 2. 배송 취소 기능
 * 3. 배송 비용 계산 최적화
 */
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Singleton](./01-Singleton.md) | [다음: Builder →](./03-Builder.md)**

</div>
