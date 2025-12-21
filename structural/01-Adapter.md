# Adapter Pattern (어댑터 패턴)

> **"호환되지 않는 인터페이스를 연결하자"**

[목차로 돌아가기](../README.md) | [다음: Decorator →](02-Decorator.md)

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
// 문제 1: 기존 시스템과 새 라이브러리가 호환 안 됨
public class LegacySystem {
    public void processRectangle(Rectangle rect) {
        // 기존 시스템은 Rectangle만 처리
    }
}

public class NewLibrary {
    public Shape getShape() {
        return new Circle(); // 새 라이브러리는 Shape 반환
    }
}

// 어떻게 연결할까? 🤔

// 문제 2: 서드파티 API와 내부 인터페이스 불일치
public interface PaymentProcessor {
    void processPayment(String orderId, double amount);
}

public class StripeAPI {
    // 완전히 다른 메서드 시그니처!
    public boolean charge(String token, int cents, String currency) {
        // Stripe 결제 처리
    }
}

// PaymentProcessor 인터페이스로 Stripe를 어떻게 사용할까?

// 문제 3: 레거시 코드 수정 불가
public class OldPrinter {
    public void printDocument(String text) {
        System.out.println("Old Printer: " + text);
    }
}

public interface ModernPrinter {
    void print(Document doc);
}

// OldPrinter는 수정할 수 없음 (외부 라이브러리)
// 하지만 ModernPrinter 인터페이스를 사용해야 함!

// 문제 4: XML과 JSON 데이터 소스
public class XMLDataSource {
    public String getXMLData() {
        return "<user><name>John</name></user>";
    }
}

public interface DataSource {
    Map<String, Object> getData(); // JSON 형식 기대
}

// XMLDataSource를 DataSource로 어떻게 사용할까?
```

### ⚡ 핵심 문제

1. **인터페이스 불일치**: 기존 코드와 새 코드의 인터페이스가 다름
2. **레거시 통합**: 수정 불가능한 레거시 시스템 통합 필요
3. **서드파티 라이브러리**: 외부 API를 내부 인터페이스에 맞춰야 함
4. **재사용 제약**: 호환성 문제로 코드 재사용 어려움

---

## 2. 패턴 정의

### 📖 정의

> **한 클래스의 인터페이스를 클라이언트가 기대하는 다른 인터페이스로 변환하는 패턴. 호환되지 않는 인터페이스 때문에 함께 동작할 수 없던 클래스들을 함께 작동시킨다.**

### 🎯 목적

- **인터페이스 변환**: 기존 인터페이스를 새 인터페이스로 변환
- **레거시 통합**: 수정 불가능한 코드 통합
- **재사용성 향상**: 기존 클래스를 새로운 환경에서 재사용
- **의존성 분리**: 클라이언트와 서비스 간 결합도 감소

### 💡 핵심 아이디어

```java
// Before: 호환 안 됨
ModernSystem system = new ModernSystem();
OldComponent old = new OldComponent();
// system.process(old); // 컴파일 에러!

// After: Adapter로 변환
ModernSystem system = new ModernSystem();
OldComponent old = new OldComponent();
Adapter adapter = new Adapter(old);
system.process(adapter); // 작동! ✅
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

#### 객체 어댑터 (Object Adapter) - 조합 사용

```
┌─────────────────┐
│     Client      │
└─────────────────┘
         │
         │ uses
         ▼
┌───────────────────┐
│  Target(interface)│
├───────────────────┤
│ + request()       │
└───────────────────┘
         △
         │ implements
         │
┌─────────────────┐
│     Adapter     │───────┐
├─────────────────┤       │ has-a
│ - adaptee       │       │
│ + request()     │◄──────┘
└─────────────────┘
         │
         │ delegates to
         ▼
┌─────────────────┐
│    Adaptee      │
├─────────────────┤
│ + specificReq() │
└─────────────────┘
```

#### 클래스 어댑터 (Class Adapter) - 상속 사용

```
┌───────────────────┐
│  Target(interface)│
└───────────────────┘
         △
         │ implements
         │
┌─────────────────┐
│     Adapter     │
├─────────────────┤      extends
│ + request()     │◄──────────┐
└─────────────────┘           │
                              │
                    ┌─────────────────┐
                    │    Adaptee      │
                    ├─────────────────┤
                    │ + specificReq() │
                    └─────────────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **Target** | 클라이언트가 사용할 인터페이스 | `MediaPlayer` |
| **Adapter** | Target 구현 & Adaptee 호출 | `MediaAdapter` |
| **Adaptee** | 변환이 필요한 기존 클래스 | `AdvancedMediaPlayer` |
| **Client** | Target 인터페이스 사용 | `AudioPlayer` |

---

## 4. 구현 방법

### 방법 1: 객체 어댑터 (Object Adapter) ⭐⭐⭐ 추천!

```java
/**
 * Target: 클라이언트가 사용하는 인터페이스
 */
public interface MediaPlayer {
    void play(String audioType, String fileName);
}

/**
 * Adaptee: 기존 클래스 (호환되지 않음)
 */
public interface AdvancedMediaPlayer {
    void playVlc(String fileName);
    void playMp4(String fileName);
}

/**
 * Adaptee 구현체 1: VLC 플레이어
 */
public class VlcPlayer implements AdvancedMediaPlayer {
    @Override
    public void playVlc(String fileName) {
        System.out.println("🎬 Playing VLC file: " + fileName);
    }
    
    @Override
    public void playMp4(String fileName) {
        // 지원 안 함
    }
}

/**
 * Adaptee 구현체 2: MP4 플레이어
 */
public class Mp4Player implements AdvancedMediaPlayer {
    @Override
    public void playVlc(String fileName) {
        // 지원 안 함
    }
    
    @Override
    public void playMp4(String fileName) {
        System.out.println("🎥 Playing MP4 file: " + fileName);
    }
}

/**
 * Adapter: AdvancedMediaPlayer를 MediaPlayer로 변환
 */
public class MediaAdapter implements MediaPlayer {
    private AdvancedMediaPlayer advancedPlayer;
    
    public MediaAdapter(String audioType) {
        // 타입에 따라 적절한 Adaptee 생성
        if (audioType.equalsIgnoreCase("vlc")) {
            advancedPlayer = new VlcPlayer();
        } else if (audioType.equalsIgnoreCase("mp4")) {
            advancedPlayer = new Mp4Player();
        }
    }
    
    @Override
    public void play(String audioType, String fileName) {
        // MediaPlayer 인터페이스를 AdvancedMediaPlayer로 변환
        if (audioType.equalsIgnoreCase("vlc")) {
            advancedPlayer.playVlc(fileName);
        } else if (audioType.equalsIgnoreCase("mp4")) {
            advancedPlayer.playMp4(fileName);
        }
    }
}

/**
 * Client: MediaPlayer를 사용
 */
public class AudioPlayer implements MediaPlayer {
    private MediaAdapter mediaAdapter;
    
    @Override
    public void play(String audioType, String fileName) {
        // 기본 지원: mp3
        if (audioType.equalsIgnoreCase("mp3")) {
            System.out.println("🎵 Playing MP3 file: " + fileName);
        }
        // Adapter를 통해 확장 포맷 지원
        else if (audioType.equalsIgnoreCase("vlc") || 
                 audioType.equalsIgnoreCase("mp4")) {
            mediaAdapter = new MediaAdapter(audioType);
            mediaAdapter.play(audioType, fileName);
        }
        else {
            System.out.println("❌ Invalid format: " + audioType);
        }
    }
}

/**
 * 사용 예제
 */
public class ObjectAdapterExample {
    public static void main(String[] args) {
        AudioPlayer player = new AudioPlayer();
        
        System.out.println("=== 오디오 재생 ===");
        player.play("mp3", "song.mp3");
        player.play("vlc", "movie.vlc");
        player.play("mp4", "video.mp4");
        player.play("avi", "video.avi");
    }
}
```

**실행 결과:**
```
=== 오디오 재생 ===
🎵 Playing MP3 file: song.mp3
🎬 Playing VLC file: movie.vlc
🎥 Playing MP4 file: video.mp4
❌ Invalid format: avi
```

---

### 방법 2: 클래스 어댑터 (Class Adapter) ⭐⭐

```java
/**
 * Target
 */
public interface TemperatureSensor {
    double getTemperature(); // 섭씨 온도
}

/**
 * Adaptee: 화씨 온도 센서
 */
public class FahrenheitSensor {
    public double getFahrenheitTemperature() {
        // 실제로는 하드웨어에서 읽음
        return 98.6; // 화씨 98.6도
    }
}

/**
 * Adapter: 다중 상속 불가능하므로 단일 상속 + 구현
 */
public class TemperatureAdapter extends FahrenheitSensor 
                               implements TemperatureSensor {
    @Override
    public double getTemperature() {
        // 화씨를 섭씨로 변환
        double fahrenheit = getFahrenheitTemperature();
        double celsius = (fahrenheit - 32) * 5.0 / 9.0;
        System.out.println("🌡️ 변환: " + fahrenheit + "°F → " + 
                String.format("%.1f", celsius) + "°C");
        return celsius;
    }
}

/**
 * 사용 예제
 */
public class ClassAdapterExample {
    public static void main(String[] args) {
        System.out.println("=== 온도 센서 ===");
        TemperatureSensor sensor = new TemperatureAdapter();
        double temp = sensor.getTemperature();
        System.out.println("현재 온도: " + String.format("%.1f", temp) + "°C");
    }
}
```

**실행 결과:**
```
=== 온도 센서 ===
🌡️ 변환: 98.6°F → 37.0°C
현재 온도: 37.0°C
```

---

### 방법 3: 양방향 어댑터 (Two-Way Adapter) ⭐⭐

```java
/**
 * 인터페이스 A
 */
public interface RectangleInterface {
    void drawRectangle(int x1, int y1, int x2, int y2);
}

/**
 * 인터페이스 B
 */
public interface ShapeInterface {
    void draw(int x, int y, int width, int height);
}

/**
 * 구현 A
 */
public class Rectangle implements RectangleInterface {
    @Override
    public void drawRectangle(int x1, int y1, int x2, int y2) {
        System.out.println("📐 Rectangle: (" + x1 + "," + y1 + 
                ") to (" + x2 + "," + y2 + ")");
    }
}

/**
 * 구현 B
 */
public class Shape implements ShapeInterface {
    @Override
    public void draw(int x, int y, int width, int height) {
        System.out.println("🔷 Shape: at (" + x + "," + y + 
                ") size " + width + "x" + height);
    }
}

/**
 * 양방향 어댑터: 두 인터페이스 모두 구현
 */
public class TwoWayAdapter implements RectangleInterface, ShapeInterface {
    private RectangleInterface rectangle;
    private ShapeInterface shape;
    
    public TwoWayAdapter(RectangleInterface rectangle) {
        this.rectangle = rectangle;
    }
    
    public TwoWayAdapter(ShapeInterface shape) {
        this.shape = shape;
    }
    
    // Rectangle → Shape
    @Override
    public void draw(int x, int y, int width, int height) {
        if (rectangle != null) {
            rectangle.drawRectangle(x, y, x + width, y + height);
        } else {
            shape.draw(x, y, width, height);
        }
    }
    
    // Shape → Rectangle
    @Override
    public void drawRectangle(int x1, int y1, int x2, int y2) {
        if (shape != null) {
            shape.draw(x1, y1, x2 - x1, y2 - y1);
        } else {
            rectangle.drawRectangle(x1, y1, x2, y2);
        }
    }
}

/**
 * 사용 예제
 */
public class TwoWayAdapterExample {
    public static void main(String[] args) {
        System.out.println("=== Rectangle을 Shape처럼 사용 ===");
        Rectangle rect = new Rectangle();
        ShapeInterface adapter1 = new TwoWayAdapter(rect);
        adapter1.draw(10, 20, 100, 50);
        
        System.out.println("\n=== Shape을 Rectangle처럼 사용 ===");
        Shape shape = new Shape();
        RectangleInterface adapter2 = new TwoWayAdapter(shape);
        adapter2.drawRectangle(10, 20, 110, 70);
    }
}
```

---

## 5. 실전 예제

### 예제 1: 결제 시스템 통합 ⭐⭐⭐

```java
/**
 * Target: 내부 결제 인터페이스
 */
public interface PaymentProcessor {
    boolean processPayment(String orderId, double amount);
    boolean refund(String orderId, double amount);
}

/**
 * Adaptee 1: Stripe API (서드파티)
 */
public class StripeAPI {
    public boolean charge(String token, int cents) {
        System.out.println("💳 Stripe charge: $" + (cents / 100.0));
        // 실제 Stripe API 호출
        return true;
    }
    
    public boolean refundCharge(String chargeId, int cents) {
        System.out.println("↩️ Stripe refund: $" + (cents / 100.0));
        return true;
    }
}

/**
 * Adaptee 2: PayPal SDK (서드파티)
 */
public class PayPalSDK {
    public void makePayment(String email, double dollars) {
        System.out.println("💰 PayPal payment: $" + dollars + " to " + email);
    }
    
    public void issueRefund(String transactionId, double dollars) {
        System.out.println("💸 PayPal refund: $" + dollars);
    }
}

/**
 * Adapter 1: Stripe → PaymentProcessor
 */
public class StripeAdapter implements PaymentProcessor {
    private StripeAPI stripe;
    
    public StripeAdapter() {
        this.stripe = new StripeAPI();
    }
    
    @Override
    public boolean processPayment(String orderId, double amount) {
        System.out.println("🔄 Stripe Adapter: Converting order " + orderId);
        // 달러를 센트로 변환
        int cents = (int) (amount * 100);
        return stripe.charge("token_" + orderId, cents);
    }
    
    @Override
    public boolean refund(String orderId, double amount) {
        System.out.println("🔄 Stripe Adapter: Refunding order " + orderId);
        int cents = (int) (amount * 100);
        return stripe.refundCharge("charge_" + orderId, cents);
    }
}

/**
 * Adapter 2: PayPal → PaymentProcessor
 */
public class PayPalAdapter implements PaymentProcessor {
    private PayPalSDK paypal;
    
    public PayPalAdapter() {
        this.paypal = new PayPalSDK();
    }
    
    @Override
    public boolean processPayment(String orderId, double amount) {
        System.out.println("🔄 PayPal Adapter: Converting order " + orderId);
        // 이메일은 설정에서 가져온다고 가정
        paypal.makePayment("customer@example.com", amount);
        return true;
    }
    
    @Override
    public boolean refund(String orderId, double amount) {
        System.out.println("🔄 PayPal Adapter: Refunding order " + orderId);
        paypal.issueRefund("txn_" + orderId, amount);
        return true;
    }
}

/**
 * Client: 결제 서비스
 */
public class PaymentService {
    private PaymentProcessor processor;
    
    public PaymentService(PaymentProcessor processor) {
        this.processor = processor;
    }
    
    public void checkout(String orderId, double amount) {
        System.out.println("\n=== 주문 결제 ===");
        System.out.println("주문 ID: " + orderId);
        System.out.println("금액: $" + amount);
        
        boolean success = processor.processPayment(orderId, amount);
        
        if (success) {
            System.out.println("✅ 결제 성공!");
        } else {
            System.out.println("❌ 결제 실패!");
        }
    }
    
    public void processRefund(String orderId, double amount) {
        System.out.println("\n=== 환불 처리 ===");
        System.out.println("주문 ID: " + orderId);
        System.out.println("금액: $" + amount);
        
        boolean success = processor.refund(orderId, amount);
        
        if (success) {
            System.out.println("✅ 환불 성공!");
        } else {
            System.out.println("❌ 환불 실패!");
        }
    }
}

/**
 * 사용 예제
 */
public class PaymentAdapterExample {
    public static void main(String[] args) {
        // Stripe 사용
        System.out.println("### Stripe로 결제 ###");
        PaymentProcessor stripeProcessor = new StripeAdapter();
        PaymentService stripeService = new PaymentService(stripeProcessor);
        stripeService.checkout("ORDER-001", 99.99);
        stripeService.processRefund("ORDER-001", 99.99);
        
        // PayPal로 쉽게 교체
        System.out.println("\n\n### PayPal로 결제 ###");
        PaymentProcessor paypalProcessor = new PayPalAdapter();
        PaymentService paypalService = new PaymentService(paypalProcessor);
        paypalService.checkout("ORDER-002", 149.99);
        paypalService.processRefund("ORDER-002", 149.99);
    }
}
```

**실행 결과:**
```
### Stripe로 결제 ###

=== 주문 결제 ===
주문 ID: ORDER-001
금액: $99.99
🔄 Stripe Adapter: Converting order ORDER-001
💳 Stripe charge: $99.99
✅ 결제 성공!

=== 환불 처리 ===
주문 ID: ORDER-001
금액: $99.99
🔄 Stripe Adapter: Refunding order ORDER-001
↩️ Stripe refund: $99.99
✅ 환불 성공!


### PayPal로 결제 ###

=== 주문 결제 ===
주문 ID: ORDER-002
금액: $149.99
🔄 PayPal Adapter: Converting order ORDER-002
💰 PayPal payment: $149.99 to customer@example.com
✅ 결제 성공!
...
```

---

### 예제 2: 데이터 소스 어댑터 ⭐⭐⭐

```java
/**
 * Target: 표준 데이터 소스 인터페이스
 */
public interface DataSource {
    Map<String, Object> getData();
    void setData(Map<String, Object> data);
}

/**
 * Adaptee 1: XML 데이터 소스
 */
public class XMLDataSource {
    private String xmlData;
    
    public XMLDataSource() {
        this.xmlData = "<user><name>John</name><age>30</age></user>";
    }
    
    public String getXMLData() {
        System.out.println("📄 Reading XML data");
        return xmlData;
    }
    
    public void setXMLData(String xmlData) {
        System.out.println("💾 Writing XML data");
        this.xmlData = xmlData;
    }
}

/**
 * Adaptee 2: CSV 데이터 소스
 */
public class CSVDataSource {
    private String csvData;
    
    public CSVDataSource() {
        this.csvData = "name,age\nJohn,30";
    }
    
    public String getCSVData() {
        System.out.println("📊 Reading CSV data");
        return csvData;
    }
    
    public void setCSVData(String csvData) {
        System.out.println("💾 Writing CSV data");
        this.csvData = csvData;
    }
}

/**
 * Adapter 1: XML → Map
 */
public class XMLAdapter implements DataSource {
    private XMLDataSource xmlSource;
    
    public XMLAdapter(XMLDataSource xmlSource) {
        this.xmlSource = xmlSource;
    }
    
    @Override
    public Map<String, Object> getData() {
        String xml = xmlSource.getXMLData();
        System.out.println("🔄 Converting XML to Map");
        
        // 간단한 파싱 (실제로는 XML 파서 사용)
        Map<String, Object> data = new HashMap<>();
        data.put("name", "John");
        data.put("age", 30);
        
        return data;
    }
    
    @Override
    public void setData(Map<String, Object> data) {
        System.out.println("🔄 Converting Map to XML");
        
        // 간단한 변환 (실제로는 XML 생성기 사용)
        String xml = "<user>";
        data.forEach((key, value) -> {});
        xml += "</user>";
        
        xmlSource.setXMLData(xml);
    }
}

/**
 * Adapter 2: CSV → Map
 */
public class CSVAdapter implements DataSource {
    private CSVDataSource csvSource;
    
    public CSVAdapter(CSVDataSource csvSource) {
        this.csvSource = csvSource;
    }
    
    @Override
    public Map<String, Object> getData() {
        String csv = csvSource.getCSVData();
        System.out.println("🔄 Converting CSV to Map");
        
        // 간단한 파싱
        Map<String, Object> data = new HashMap<>();
        data.put("name", "John");
        data.put("age", 30);
        
        return data;
    }
    
    @Override
    public void setData(Map<String, Object> data) {
        System.out.println("🔄 Converting Map to CSV");
        
        // 간단한 변환
        String csv = "name,age\n" + data.get("name") + "," + data.get("age");
        csvSource.setCSVData(csv);
    }
}

/**
 * Client: 데이터 프로세서
 */
public class DataProcessor {
    public void process(DataSource source) {
        System.out.println("\n=== 데이터 처리 ===");
        
        // 표준 인터페이스로 데이터 읽기
        Map<String, Object> data = source.getData();
        System.out.println("읽은 데이터: " + data);
        
        // 데이터 수정
        data.put("processed", true);
        
        // 표준 인터페이스로 데이터 쓰기
        source.setData(data);
        System.out.println("✅ 처리 완료!");
    }
}

/**
 * 사용 예제
 */
public class DataSourceAdapterExample {
    public static void main(String[] args) {
        DataProcessor processor = new DataProcessor();
        
        // XML 처리
        System.out.println("### XML 데이터 소스 ###");
        XMLDataSource xmlSource = new XMLDataSource();
        DataSource xmlAdapter = new XMLAdapter(xmlSource);
        processor.process(xmlAdapter);
        
        // CSV 처리
        System.out.println("\n\n### CSV 데이터 소스 ###");
        CSVDataSource csvSource = new CSVDataSource();
        DataSource csvAdapter = new CSVAdapter(csvSource);
        processor.process(csvAdapter);
    }
}
```

---

### 예제 3: 로깅 시스템 통합 ⭐⭐

```java
/**
 * Target: 표준 로거 인터페이스
 */
public interface Logger {
    void log(String level, String message);
}

/**
 * Adaptee 1: Log4j (레거시)
 */
public class Log4jLogger {
    public void debug(String msg) {
        System.out.println("[Log4j DEBUG] " + msg);
    }
    
    public void info(String msg) {
        System.out.println("[Log4j INFO] " + msg);
    }
    
    public void error(String msg) {
        System.out.println("[Log4j ERROR] " + msg);
    }
}

/**
 * Adaptee 2: SLF4J
 */
public class SLF4JLogger {
    public void logMessage(int level, String msg) {
        String levelStr = level == 0 ? "DEBUG" : level == 1 ? "INFO" : "ERROR";
        System.out.println("[SLF4J " + levelStr + "] " + msg);
    }
}

/**
 * Adapter 1: Log4j → Logger
 */
public class Log4jAdapter implements Logger {
    private Log4jLogger log4j;
    
    public Log4jAdapter() {
        this.log4j = new Log4jLogger();
    }
    
    @Override
    public void log(String level, String message) {
        switch (level.toUpperCase()) {
            case "DEBUG":
                log4j.debug(message);
                break;
            case "INFO":
                log4j.info(message);
                break;
            case "ERROR":
                log4j.error(message);
                break;
        }
    }
}

/**
 * Adapter 2: SLF4J → Logger
 */
public class SLF4JAdapter implements Logger {
    private SLF4JLogger slf4j;
    
    public SLF4JAdapter() {
        this.slf4j = new SLF4JLogger();
    }
    
    @Override
    public void log(String level, String message) {
        int levelCode;
        switch (level.toUpperCase()) {
            case "DEBUG":
                levelCode = 0;
                break;
            case "INFO":
                levelCode = 1;
                break;
            case "ERROR":
                levelCode = 2;
                break;
            default:
                levelCode = 1;
        }
        slf4j.logMessage(levelCode, message);
    }
}

/**
 * 사용 예제
 */
public class LoggerAdapterExample {
    public static void main(String[] args) {
        System.out.println("### Log4j 사용 ###");
        Logger log4jLogger = new Log4jAdapter();
        log4jLogger.log("DEBUG", "디버그 메시지");
        log4jLogger.log("INFO", "정보 메시지");
        log4jLogger.log("ERROR", "에러 메시지");
        
        System.out.println("\n### SLF4J 사용 ###");
        Logger slf4jLogger = new SLF4JAdapter();
        slf4jLogger.log("DEBUG", "디버그 메시지");
        slf4jLogger.log("INFO", "정보 메시지");
        slf4jLogger.log("ERROR", "에러 메시지");
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **단일 책임 원칙** | 변환 로직을 분리 | Adapter가 변환만 담당 |
| **개방-폐쇄 원칙** | 기존 코드 수정 없이 확장 | 새 Adapter 추가만 |
| **재사용성** | 기존 클래스 재사용 | 레거시 코드 활용 |
| **유연성** | 런타임에 어댑터 교체 가능 | Stripe ↔ PayPal |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **복잡도 증가** | 클래스 수 증가 | 필요한 경우만 사용 |
| **간접 호출** | 성능 오버헤드 약간 | 대부분 무시 가능 |

---

## 7. 안티패턴

### ❌ 안티패턴 1: 과도한 어댑터 체인

```java
// 잘못된 예: 어댑터의 어댑터
Adapter1 -> Adapter2 -> Adapter3 -> Adaptee
// 복잡도만 증가!
```

**해결:**
```java
// 직접 변환하는 단일 어댑터
DirectAdapter -> Adaptee
```

---

## 8. 핵심 정리

### 📌 Adapter 패턴 체크리스트

```
✅ Target 인터페이스 정의
✅ Adaptee 파악
✅ Adapter 구현 (조합 or 상속)
✅ 변환 로직 작성
✅ 클라이언트는 Target만 사용
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 레거시 시스템 통합 | ⭐⭐⭐ | 수정 불가 |
| 서드파티 API 통합 | ⭐⭐⭐ | 인터페이스 불일치 |
| 재사용 필요 | ⭐⭐⭐ | 기존 코드 활용 |

### 💡 핵심 포인트

1. **인터페이스 변환이 핵심**
2. **객체 어댑터 권장** (조합 > 상속)
3. **단방향 변환 기본**
4. **레거시 통합에 최적**

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[다음: Decorator →](02-Decorator.md)**

</div>
