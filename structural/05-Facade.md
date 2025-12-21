# Facade Pattern (퍼사드 패턴)

> **"복잡한 서브시스템을 간단한 인터페이스로 제공하자"**

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
// 문제 1: 복잡한 서브시스템 호출
public class VideoConverter {
    public void convertVideo(String fileName) {
        // 1. 비디오 파일 읽기
        VideoFile video = new VideoFile(fileName);
        
        // 2. 코덱 확인
        Codec codec = CodecFactory.extract(video);
        
        // 3. 비디오 버퍼 생성
        VideoBuffer buffer;
        if (codec instanceof MPEG4Codec) {
            buffer = new MPEG4Buffer(video);
        } else {
            buffer = new OggBuffer(video);
        }
        
        // 4. 오디오 믹서 설정
        AudioMixer mixer = new AudioMixer();
        mixer.fix(buffer);
        
        // 5. 비트레이트 계산
        BitrateCalculator calculator = new BitrateCalculator();
        int bitrate = calculator.calculate(buffer);
        
        // 6. 압축
        Compressor compressor = new Compressor();
        buffer = compressor.compress(buffer, bitrate);
        
        // 7. 파일 저장
        FileWriter writer = new FileWriter();
        writer.write(buffer, "output.mp4");
        
        // 너무 복잡! 클라이언트가 알아야 할 게 너무 많음!
    }
}

// 문제 2: 여러 서브시스템의 복잡한 초기화
public class Application {
    public void start() {
        // 데이터베이스 초기화
        Database db = new Database();
        db.connect("localhost", 3306);
        db.authenticate("user", "pass");
        db.selectDatabase("mydb");
        
        // 캐시 초기화
        Cache cache = new Cache();
        cache.setMaxSize(1000);
        cache.setEvictionPolicy("LRU");
        cache.connect();
        
        // 로거 초기화
        Logger logger = new Logger();
        logger.setLevel("INFO");
        logger.setOutput("file");
        logger.setPath("/var/log/app.log");
        logger.initialize();
        
        // 모니터링 초기화
        Monitor monitor = new Monitor();
        monitor.setInterval(60);
        monitor.setMetrics("cpu", "memory", "disk");
        monitor.start();
        
        // 초기화 코드가 너무 복잡!
    }
}

// 문제 3: 의존성이 클라이언트에 노출
public class OrderService {
    public void createOrder(Order order) {
        // 재고 확인
        InventoryService inventory = new InventoryService();
        if (!inventory.checkStock(order.getItems())) {
            throw new Exception("재고 부족");
        }
        
        // 결제 처리
        PaymentGateway gateway = new PaymentGateway();
        gateway.initialize();
        gateway.setMerchantId("MERCHANT123");
        Transaction tx = gateway.createTransaction(order.getAmount());
        tx.process();
        
        // 배송 시작
        ShippingService shipping = new ShippingService();
        shipping.calculateShippingFee(order);
        shipping.createShipment(order);
        
        // 알림 발송
        NotificationService notifier = new NotificationService();
        notifier.sendEmail(order.getCustomer());
        notifier.sendSMS(order.getCustomer());
        
        // 클라이언트가 너무 많은 클래스를 알아야 함!
    }
}
```

### ⚡ 핵심 문제

1. **복잡한 호출**: 여러 서브시스템 조합 필요
2. **높은 결합도**: 클라이언트가 서브시스템에 강결합
3. **어려운 사용**: 사용법이 복잡하고 어려움
4. **변경 취약**: 서브시스템 변경 시 클라이언트도 수정

---

## 2. 패턴 정의

### 📖 정의

> **서브시스템의 인터페이스 집합에 대해 통합된 하나의 인터페이스를 제공하는 패턴. Facade는 서브시스템을 더 쉽게 사용할 수 있도록 상위 수준의 인터페이스를 정의한다.**

### 🎯 목적

- **단순화**: 복잡한 서브시스템을 간단히
- **낮은 결합도**: 클라이언트와 서브시스템 분리
- **사용 편의성**: 고수준 인터페이스 제공
- **계층화**: 서브시스템 계층 구조화

### 💡 핵심 아이디어

```java
// Before: 직접 서브시스템 호출
SubsystemA a = new SubsystemA();
SubsystemB b = new SubsystemB();
SubsystemC c = new SubsystemC();
a.init();
b.configure();
c.setup();
a.process();
b.execute();
c.finalize();

// After: Facade로 간단히
Facade facade = new Facade();
facade.simpleOperation(); // 내부에서 모든 처리!
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
┌─────────────────┐
│     Facade      │  ← 통합 인터페이스
├─────────────────┤
│ + operation()   │
└─────────────────┘
         │
         │ delegates to
         ▼
    ┌────┴────┬────────┬────────┐
    │         │        │        │
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│Subsys A│ │Subsys B│ │Subsys C│ │Subsys D│
└────────┘ └────────┘ └────────┘ └────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **Facade** | 통합 인터페이스 제공 | `HomeTheaterFacade` |
| **Subsystems** | 실제 기능 구현 | `Amplifier`, `DVD` |
| **Client** | Facade 사용 | `User` |

---

## 4. 구현 방법

### 기본 구현: 홈시어터 시스템 ⭐⭐⭐

```java
/**
 * Subsystem 1: 앰프
 */
public class Amplifier {
    public void on() {
        System.out.println("🔊 앰프 켜기");
    }
    
    public void off() {
        System.out.println("🔊 앰프 끄기");
    }
    
    public void setVolume(int level) {
        System.out.println("🔊 볼륨 설정: " + level);
    }
    
    public void setSurroundSound() {
        System.out.println("🔊 서라운드 사운드 모드");
    }
}

/**
 * Subsystem 2: DVD 플레이어
 */
public class DvdPlayer {
    public void on() {
        System.out.println("📀 DVD 플레이어 켜기");
    }
    
    public void off() {
        System.out.println("📀 DVD 플레이어 끄기");
    }
    
    public void play(String movie) {
        System.out.println("📀 영화 재생: " + movie);
    }
    
    public void stop() {
        System.out.println("📀 재생 정지");
    }
    
    public void eject() {
        System.out.println("📀 디스크 꺼내기");
    }
}

/**
 * Subsystem 3: 프로젝터
 */
public class Projector {
    public void on() {
        System.out.println("📽️ 프로젝터 켜기");
    }
    
    public void off() {
        System.out.println("📽️ 프로젝터 끄기");
    }
    
    public void wideScreenMode() {
        System.out.println("📽️ 와이드스크린 모드 (16:9)");
    }
}

/**
 * Subsystem 4: 조명
 */
public class Lights {
    public void dim(int level) {
        System.out.println("💡 조명 밝기: " + level + "%");
    }
    
    public void on() {
        System.out.println("💡 조명 켜기");
    }
}

/**
 * Subsystem 5: 스크린
 */
public class Screen {
    public void down() {
        System.out.println("🎬 스크린 내리기");
    }
    
    public void up() {
        System.out.println("🎬 스크린 올리기");
    }
}

/**
 * Facade: 홈시어터 파사드
 */
public class HomeTheaterFacade {
    private Amplifier amp;
    private DvdPlayer dvd;
    private Projector projector;
    private Lights lights;
    private Screen screen;
    
    public HomeTheaterFacade() {
        this.amp = new Amplifier();
        this.dvd = new DvdPlayer();
        this.projector = new Projector();
        this.lights = new Lights();
        this.screen = new Screen();
    }
    
    /**
     * 영화 시청 시작 - 복잡한 과정을 간단히!
     */
    public void watchMovie(String movie) {
        System.out.println("\n🍿 영화 감상 모드 시작...\n");
        
        lights.dim(10);
        screen.down();
        projector.on();
        projector.wideScreenMode();
        amp.on();
        amp.setSurroundSound();
        amp.setVolume(5);
        dvd.on();
        dvd.play(movie);
        
        System.out.println("\n✨ 준비 완료! 영화를 즐기세요!\n");
    }
    
    /**
     * 영화 시청 종료
     */
    public void endMovie() {
        System.out.println("\n🛑 영화 감상 모드 종료...\n");
        
        dvd.stop();
        dvd.eject();
        dvd.off();
        amp.off();
        projector.off();
        screen.up();
        lights.on();
        
        System.out.println("\n✅ 모든 장비 꺼짐\n");
    }
    
    /**
     * 음악 듣기 모드
     */
    public void listenToMusic(String song) {
        System.out.println("\n🎵 음악 감상 모드 시작...\n");
        
        lights.dim(30);
        amp.on();
        amp.setVolume(7);
        dvd.on();
        dvd.play(song);
        
        System.out.println("\n✨ 음악을 즐기세요!\n");
    }
}

/**
 * 사용 예제
 */
public class FacadeExample {
    public static void main(String[] args) {
        HomeTheaterFacade homeTheater = new HomeTheaterFacade();
        
        // Before Facade: 복잡한 과정
        System.out.println("### Before Facade ###");
        System.out.println("클라이언트가 각 시스템을 일일이 제어해야 함...");
        
        // After Facade: 간단한 호출
        System.out.println("\n### After Facade ###");
        homeTheater.watchMovie("Inception");
        
        System.out.println("\n" + "=".repeat(50));
        
        homeTheater.endMovie();
        
        System.out.println("\n" + "=".repeat(50));
        
        homeTheater.listenToMusic("Bohemian Rhapsody");
    }
}
```

**실행 결과:**
```
### Before Facade ###
클라이언트가 각 시스템을 일일이 제어해야 함...

### After Facade ###

🍿 영화 감상 모드 시작...

💡 조명 밝기: 10%
🎬 스크린 내리기
📽️ 프로젝터 켜기
📽️ 와이드스크린 모드 (16:9)
🔊 앰프 켜기
🔊 서라운드 사운드 모드
🔊 볼륨 설정: 5
📀 DVD 플레이어 켜기
📀 영화 재생: Inception

✨ 준비 완료! 영화를 즐기세요!

==================================================

🛑 영화 감상 모드 종료...

📀 재생 정지
📀 디스크 꺼내기
📀 DVD 플레이어 끄기
🔊 앰프 끄기
📽️ 프로젝터 끄기
🎬 스크린 올리기
💡 조명 켜기

✅ 모든 장비 꺼짐

==================================================

🎵 음악 감상 모드 시작...

💡 조명 밝기: 30%
🔊 앰프 켜기
🔊 볼륨 설정: 7
📀 DVD 플레이어 켜기
📀 영화 재생: Bohemian Rhapsody

✨ 음악을 즐기세요!
```

---

## 5. 실전 예제

### 예제 1: 컴퓨터 시스템 ⭐⭐⭐

```java
/**
 * Subsystem: CPU
 */
public class CPU {
    public void freeze() {
        System.out.println("💻 CPU: Freezing...");
    }
    
    public void jump(long position) {
        System.out.println("💻 CPU: Jumping to " + position);
    }
    
    public void execute() {
        System.out.println("💻 CPU: Executing...");
    }
}

/**
 * Subsystem: Memory
 */
public class Memory {
    public void load(long position, byte[] data) {
        System.out.println("💾 Memory: Loading data to position " + position);
    }
}

/**
 * Subsystem: HardDrive
 */
public class HardDrive {
    public byte[] read(long lba, int size) {
        System.out.println("💿 HardDrive: Reading " + size + " bytes from sector " + lba);
        return new byte[size];
    }
}

/**
 * Facade: 컴퓨터
 */
public class ComputerFacade {
    private static final long BOOT_ADDRESS = 0x00;
    private static final long BOOT_SECTOR = 0x00;
    private static final int SECTOR_SIZE = 512;
    
    private CPU cpu;
    private Memory memory;
    private HardDrive hardDrive;
    
    public ComputerFacade() {
        this.cpu = new CPU();
        this.memory = new Memory();
        this.hardDrive = new HardDrive();
    }
    
    public void start() {
        System.out.println("\n🖥️ 컴퓨터 부팅 시작...\n");
        
        cpu.freeze();
        memory.load(BOOT_ADDRESS, hardDrive.read(BOOT_SECTOR, SECTOR_SIZE));
        cpu.jump(BOOT_ADDRESS);
        cpu.execute();
        
        System.out.println("\n✅ 부팅 완료!\n");
    }
}

/**
 * 사용 예제
 */
public class ComputerExample {
    public static void main(String[] args) {
        ComputerFacade computer = new ComputerFacade();
        
        // 간단한 한 줄 호출!
        computer.start();
        
        System.out.println("Facade 없이는 CPU, Memory, HardDrive를 직접 제어해야 함!");
    }
}
```

---

### 예제 2: 온라인 쇼핑몰 주문 ⭐⭐⭐

```java
/**
 * Subsystem: 재고 관리
 */
public class InventorySystem {
    public boolean checkStock(String productId, int quantity) {
        System.out.println("📦 재고 확인: " + productId + " x " + quantity);
        return true; // 간단히 항상 true
    }
    
    public void reserveStock(String productId, int quantity) {
        System.out.println("📦 재고 예약: " + productId + " x " + quantity);
    }
    
    public void releaseStock(String productId, int quantity) {
        System.out.println("📦 재고 해제: " + productId + " x " + quantity);
    }
}

/**
 * Subsystem: 결제 시스템
 */
public class PaymentSystem {
    public boolean processPayment(String cardNumber, double amount) {
        System.out.println("💳 결제 처리: " + amount + "원 (카드: " + cardNumber + ")");
        return true;
    }
    
    public void refund(String transactionId, double amount) {
        System.out.println("💸 환불 처리: " + amount + "원 (거래: " + transactionId + ")");
    }
}

/**
 * Subsystem: 배송 시스템
 */
public class ShippingSystem {
    public String createShipment(String address, String productId) {
        System.out.println("🚚 배송 생성: " + productId + " → " + address);
        return "SHIP-" + System.currentTimeMillis();
    }
    
    public void trackShipment(String shipmentId) {
        System.out.println("🔍 배송 추적: " + shipmentId);
    }
}

/**
 * Subsystem: 알림 시스템
 */
public class NotificationSystem {
    public void sendEmail(String email, String message) {
        System.out.println("📧 이메일 발송: " + email);
        System.out.println("   내용: " + message);
    }
    
    public void sendSMS(String phone, String message) {
        System.out.println("📱 SMS 발송: " + phone);
        System.out.println("   내용: " + message);
    }
}

/**
 * Facade: 주문 시스템
 */
public class OrderFacade {
    private InventorySystem inventory;
    private PaymentSystem payment;
    private ShippingSystem shipping;
    private NotificationSystem notification;
    
    public OrderFacade() {
        this.inventory = new InventorySystem();
        this.payment = new PaymentSystem();
        this.shipping = new ShippingSystem();
        this.notification = new NotificationSystem();
    }
    
    /**
     * 주문 처리 - 복잡한 프로세스를 한 번에!
     */
    public boolean placeOrder(String productId, int quantity, 
                              String cardNumber, double amount,
                              String address, String email, String phone) {
        
        System.out.println("\n🛒 주문 처리 시작...\n");
        
        try {
            // 1. 재고 확인 및 예약
            if (!inventory.checkStock(productId, quantity)) {
                System.out.println("❌ 재고 부족!");
                return false;
            }
            inventory.reserveStock(productId, quantity);
            
            // 2. 결제 처리
            if (!payment.processPayment(cardNumber, amount)) {
                System.out.println("❌ 결제 실패!");
                inventory.releaseStock(productId, quantity);
                return false;
            }
            
            // 3. 배송 생성
            String shipmentId = shipping.createShipment(address, productId);
            
            // 4. 알림 발송
            notification.sendEmail(email, "주문이 완료되었습니다. 배송번호: " + shipmentId);
            notification.sendSMS(phone, "주문 완료! 배송번호: " + shipmentId);
            
            System.out.println("\n✅ 주문 성공!\n");
            return true;
            
        } catch (Exception e) {
            System.out.println("❌ 주문 실패: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * 주문 취소
     */
    public void cancelOrder(String productId, int quantity, 
                           String transactionId, double amount) {
        
        System.out.println("\n🚫 주문 취소 처리...\n");
        
        inventory.releaseStock(productId, quantity);
        payment.refund(transactionId, amount);
        
        System.out.println("\n✅ 취소 완료!\n");
    }
}

/**
 * 사용 예제
 */
public class OrderExample {
    public static void main(String[] args) {
        OrderFacade orderSystem = new OrderFacade();
        
        // 복잡한 주문 프로세스를 간단한 한 줄로!
        boolean success = orderSystem.placeOrder(
            "PROD-001",              // 상품 ID
            2,                       // 수량
            "1234-5678-9012-3456",  // 카드 번호
            50000,                   // 금액
            "서울시 강남구",          // 주소
            "user@email.com",        // 이메일
            "010-1234-5678"          // 전화번호
        );
        
        if (success) {
            System.out.println("주문이 성공적으로 완료되었습니다!");
        }
        
        // 취소도 간단히
        System.out.println("\n" + "=".repeat(50));
        orderSystem.cancelOrder("PROD-001", 2, "TXN-123", 50000);
    }
}
```

**실행 결과:**
```
🛒 주문 처리 시작...

📦 재고 확인: PROD-001 x 2
📦 재고 예약: PROD-001 x 2
💳 결제 처리: 50000.0원 (카드: 1234-5678-9012-3456)
🚚 배송 생성: PROD-001 → 서울시 강남구
📧 이메일 발송: user@email.com
   내용: 주문이 완료되었습니다. 배송번호: SHIP-1234567890
📱 SMS 발송: 010-1234-5678
   내용: 주문 완료! 배송번호: SHIP-1234567890

✅ 주문 성공!

주문이 성공적으로 완료되었습니다!

==================================================

🚫 주문 취소 처리...

📦 재고 해제: PROD-001 x 2
💸 환불 처리: 50000.0원 (거래: TXN-123)

✅ 취소 완료!
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **단순화** | 복잡한 시스템을 간단히 | 홈시어터 제어 |
| **낮은 결합도** | 클라이언트와 서브시스템 분리 | 주문 시스템 |
| **사용 편의성** | 고수준 인터페이스 | watchMovie() |
| **계층화** | 서브시스템 조직화 | MVC 패턴 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **God Object 가능성** | Facade가 너무 커질 수 있음 | 책임 분리 |
| **유연성 감소** | 고급 기능 접근 어려움 | 직접 접근도 허용 |

---

## 7. 안티패턴

### ❌ 안티패턴 1: God Facade

```java
// 잘못된 예: 모든 기능을 Facade에
public class GodFacade {
    public void doEverything() {
        // 너무 많은 책임!
    }
}
```

**해결:**
```java
// 여러 Facade로 분리
public class OrderFacade { }
public class UserFacade { }
public class ProductFacade { }
```

---

## 8. 핵심 정리

### 📌 Facade 패턴 체크리스트

```
✅ 복잡한 서브시스템 파악
✅ Facade 클래스 생성
✅ 고수준 메서드 정의
✅ 서브시스템 호출 캡슐화
✅ 클라이언트는 Facade만 사용
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 복잡한 라이브러리 | ⭐⭐⭐ | 사용 간소화 |
| 레이어 분리 | ⭐⭐⭐ | 계층 구조 |
| 레거시 통합 | ⭐⭐⭐ | 인터페이스 단순화 |

### 💡 핵심 포인트

1. **복잡함을 숨김**
2. **사용 편의성 극대화**
3. **서브시스템 독립성 유지**
4. **계층화 핵심 패턴**

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Composite](04-Composite.md) | [다음: Bridge →](06-Bridge.md)**

</div>
