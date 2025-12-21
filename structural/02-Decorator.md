# Decorator Pattern (데코레이터 패턴)

> **"객체에 동적으로 기능을 추가하자"**

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
// 문제 1: 상속으로 기능 추가 → 클래스 폭발
public class SimpleCoffee {
    public int cost() { return 10; }
}

public class MilkCoffee extends SimpleCoffee {
    public int cost() { return super.cost() + 5; }
}

public class SugarCoffee extends SimpleCoffee {
    public int cost() { return super.cost() + 2; }
}

public class MilkSugarCoffee extends SimpleCoffee {
    public int cost() { return super.cost() + 7; }
}

// 휘핑크림, 시럽, 초콜릿... 조합 폭발! 💥
// 우유+설탕+휘핑+시럽 = 별도 클래스 필요!

// 문제 2: 런타임에 기능 추가/제거 불가
public class TextEditor {
    private boolean bold = false;
    private boolean italic = false;
    private boolean underline = false;
    
    public void setBold(boolean b) { this.bold = b; }
    public void setItalic(boolean i) { this.italic = i; }
    public void setUnderline(boolean u) { this.underline = u; }
    
    // 모든 조합을 미리 구현해야 함!
    public String format(String text) {
        if (bold && italic && underline) {
            return "<b><i><u>" + text + "</u></i></b>";
        } else if (bold && italic) {
            return "<b><i>" + text + "</i></b>";
        }
        // ... 8가지 조합!
    }
}

// 문제 3: 기존 클래스 수정 필요
public class FileReader {
    public String read(String path) {
        // 파일 읽기
    }
}

// 암호화 기능 추가하려면?
// → FileReader 수정 (OCP 위반!)
// → 상속 (단일 상속 제약)

// 문제 4: 기능의 순서가 중요한 경우
public class DataProcessor {
    public String process(String data) {
        // 1. 압축
        // 2. 암호화
        // 3. Base64 인코딩
        // 순서 바꾸기 어려움!
    }
}
```

### ⚡ 핵심 문제

1. **조합 폭발**: 기능 조합마다 클래스 필요
2. **정적 구조**: 컴파일 타임에 기능 고정
3. **단일 상속**: 다중 기능 추가 어려움
4. **OCP 위반**: 기존 코드 수정 필요

---

## 2. 패턴 정의

### 📖 정의

> **객체에 동적으로 새로운 책임을 추가하는 패턴. 기능 확장을 위해 서브클래스 대신 유연한 대안을 제공한다.**

### 🎯 목적

- **동적 기능 추가**: 런타임에 객체에 기능 추가/제거
- **조합 자유**: 여러 기능을 자유롭게 조합
- **OCP 준수**: 기존 코드 수정 없이 확장
- **단일 책임**: 각 데코레이터는 하나의 기능만

### 💡 핵심 아이디어

```java
// Before: 상속으로 기능 추가
Coffee milkSugarCoffee = new MilkSugarCoffee();

// After: 데코레이터로 동적 조합
Coffee coffee = new SimpleCoffee();
coffee = new MilkDecorator(coffee);
coffee = new SugarDecorator(coffee);
// 런타임에 기능 추가! ✨
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌───────────────────────┐
│   Component(interface)│  ← 공통 인터페이스
├───────────────────────┤
│ + operation()         │
└───────────────────────┘
          △
          │
    ┌─────┴─────┐
    │           │
┌───────────┐ ┌──────────────────┐
│Concrete   │ │  Decorator       │ ← 추상 데코레이터
│Component  │ ├──────────────────┤
└───────────┘ │ - component      │────┐
              │ + operation()    │    │ wraps
              └──────────────────┘    │
                      △               │
                      │               │
              ┌───────┴────────┐      │
              │                │      │
      ┌───────────────┐ ┌──────────────┐
      │ConcreteA      │ │ConcreteB     │
      │Decorator      │ │Decorator     │
      ├───────────────┤ ├──────────────┤
      │+ operation()  │ │+ operation() │
      └───────────────┘ └──────────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **Component** | 공통 인터페이스 | `Coffee` |
| **ConcreteComponent** | 기본 구현 | `SimpleCoffee` |
| **Decorator** | 데코레이터 기반 클래스 | `CoffeeDecorator` |
| **ConcreteDecorator** | 구체적인 기능 추가 | `MilkDecorator` |

---

## 4. 구현 방법

### 기본 구현: 커피 주문 시스템 ⭐⭐⭐

```java
/**
 * Component: 커피 인터페이스
 */
public interface Coffee {
    String getDescription();
    int cost();
}

/**
 * ConcreteComponent: 기본 커피
 */
public class SimpleCoffee implements Coffee {
    @Override
    public String getDescription() {
        return "Simple Coffee";
    }
    
    @Override
    public int cost() {
        return 10;
    }
}

/**
 * Decorator: 추상 데코레이터
 */
public abstract class CoffeeDecorator implements Coffee {
    protected Coffee coffee;
    
    public CoffeeDecorator(Coffee coffee) {
        this.coffee = coffee;
    }
    
    @Override
    public String getDescription() {
        return coffee.getDescription();
    }
    
    @Override
    public int cost() {
        return coffee.cost();
    }
}

/**
 * ConcreteDecorator 1: 우유 추가
 */
public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public String getDescription() {
        return coffee.getDescription() + ", Milk";
    }
    
    @Override
    public int cost() {
        return coffee.cost() + 5;
    }
}

/**
 * ConcreteDecorator 2: 설탕 추가
 */
public class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public String getDescription() {
        return coffee.getDescription() + ", Sugar";
    }
    
    @Override
    public int cost() {
        return coffee.cost() + 2;
    }
}

/**
 * ConcreteDecorator 3: 휘핑크림 추가
 */
public class WhipDecorator extends CoffeeDecorator {
    public WhipDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public String getDescription() {
        return coffee.getDescription() + ", Whip Cream";
    }
    
    @Override
    public int cost() {
        return coffee.cost() + 7;
    }
}

/**
 * ConcreteDecorator 4: 바닐라 시럽 추가
 */
public class VanillaDecorator extends CoffeeDecorator {
    public VanillaDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public String getDescription() {
        return coffee.getDescription() + ", Vanilla Syrup";
    }
    
    @Override
    public int cost() {
        return coffee.cost() + 3;
    }
}

/**
 * 사용 예제
 */
public class DecoratorExample {
    public static void main(String[] args) {
        // 1. 기본 커피
        System.out.println("=== 주문 1: 기본 커피 ===");
        Coffee coffee1 = new SimpleCoffee();
        printOrder(coffee1);
        
        // 2. 우유 커피
        System.out.println("\n=== 주문 2: 우유 커피 ===");
        Coffee coffee2 = new SimpleCoffee();
        coffee2 = new MilkDecorator(coffee2);
        printOrder(coffee2);
        
        // 3. 우유 + 설탕 커피
        System.out.println("\n=== 주문 3: 우유 설탕 커피 ===");
        Coffee coffee3 = new SimpleCoffee();
        coffee3 = new MilkDecorator(coffee3);
        coffee3 = new SugarDecorator(coffee3);
        printOrder(coffee3);
        
        // 4. 풀옵션 커피
        System.out.println("\n=== 주문 4: 풀옵션 커피 ===");
        Coffee coffee4 = new SimpleCoffee();
        coffee4 = new MilkDecorator(coffee4);
        coffee4 = new SugarDecorator(coffee4);
        coffee4 = new WhipDecorator(coffee4);
        coffee4 = new VanillaDecorator(coffee4);
        printOrder(coffee4);
        
        // 5. 한 줄로 작성
        System.out.println("\n=== 주문 5: 카라멜 마키아또 스타일 ===");
        Coffee coffee5 = new VanillaDecorator(
                            new WhipDecorator(
                                new MilkDecorator(
                                    new SimpleCoffee()
                                )
                            )
                         );
        printOrder(coffee5);
    }
    
    private static void printOrder(Coffee coffee) {
        System.out.println("☕ " + coffee.getDescription());
        System.out.println("💰 가격: " + coffee.cost() + "원");
    }
}
```

**실행 결과:**
```
=== 주문 1: 기본 커피 ===
☕ Simple Coffee
💰 가격: 10원

=== 주문 2: 우유 커피 ===
☕ Simple Coffee, Milk
💰 가격: 15원

=== 주문 3: 우유 설탕 커피 ===
☕ Simple Coffee, Milk, Sugar
💰 가격: 17원

=== 주문 4: 풀옵션 커피 ===
☕ Simple Coffee, Milk, Sugar, Whip Cream, Vanilla Syrup
💰 가격: 27원

=== 주문 5: 카라멜 마키아또 스타일 ===
☕ Simple Coffee, Milk, Whip Cream, Vanilla Syrup
💰 가격: 25원
```

---

## 5. 실전 예제

### 예제 1: 스트림 처리 (Java I/O) ⭐⭐⭐

```java
/**
 * Component: 데이터 스트림
 */
public interface DataStream {
    void write(String data);
    String read();
}

/**
 * ConcreteComponent: 기본 스트림
 */
public class FileDataStream implements DataStream {
    private String fileName;
    private String data;
    
    public FileDataStream(String fileName) {
        this.fileName = fileName;
    }
    
    @Override
    public void write(String data) {
        this.data = data;
        System.out.println("📁 파일에 쓰기: " + fileName);
        System.out.println("   데이터: " + data);
    }
    
    @Override
    public String read() {
        System.out.println("📖 파일에서 읽기: " + fileName);
        return data;
    }
}

/**
 * Decorator: 스트림 데코레이터
 */
public abstract class DataStreamDecorator implements DataStream {
    protected DataStream stream;
    
    public DataStreamDecorator(DataStream stream) {
        this.stream = stream;
    }
    
    @Override
    public void write(String data) {
        stream.write(data);
    }
    
    @Override
    public String read() {
        return stream.read();
    }
}

/**
 * ConcreteDecorator 1: 압축
 */
public class CompressionDecorator extends DataStreamDecorator {
    public CompressionDecorator(DataStream stream) {
        super(stream);
    }
    
    @Override
    public void write(String data) {
        String compressed = compress(data);
        System.out.println("🗜️ 압축: " + data.length() + " → " + compressed.length() + " bytes");
        stream.write(compressed);
    }
    
    @Override
    public String read() {
        String data = stream.read();
        String decompressed = decompress(data);
        System.out.println("📦 압축 해제: " + data.length() + " → " + decompressed.length() + " bytes");
        return decompressed;
    }
    
    private String compress(String data) {
        // 실제로는 GZIP 등 사용
        return "[COMPRESSED]" + data;
    }
    
    private String decompress(String data) {
        return data.replace("[COMPRESSED]", "");
    }
}

/**
 * ConcreteDecorator 2: 암호화
 */
public class EncryptionDecorator extends DataStreamDecorator {
    public EncryptionDecorator(DataStream stream) {
        super(stream);
    }
    
    @Override
    public void write(String data) {
        String encrypted = encrypt(data);
        System.out.println("🔒 암호화: " + data + " → " + encrypted);
        stream.write(encrypted);
    }
    
    @Override
    public String read() {
        String data = stream.read();
        String decrypted = decrypt(data);
        System.out.println("🔓 복호화: " + data + " → " + decrypted);
        return decrypted;
    }
    
    private String encrypt(String data) {
        // 실제로는 AES 등 사용
        return new StringBuilder(data).reverse().toString();
    }
    
    private String decrypt(String data) {
        return new StringBuilder(data).reverse().toString();
    }
}

/**
 * ConcreteDecorator 3: Base64 인코딩
 */
public class Base64Decorator extends DataStreamDecorator {
    public Base64Decorator(DataStream stream) {
        super(stream);
    }
    
    @Override
    public void write(String data) {
        String encoded = Base64.getEncoder().encodeToString(data.getBytes());
        System.out.println("🔤 Base64 인코딩");
        stream.write(encoded);
    }
    
    @Override
    public String read() {
        String data = stream.read();
        String decoded = new String(Base64.getDecoder().decode(data));
        System.out.println("🔡 Base64 디코딩");
        return decoded;
    }
}

/**
 * 사용 예제
 */
public class StreamDecoratorExample {
    public static void main(String[] args) {
        String message = "Hello, Decorator Pattern!";
        
        // 1. 기본 스트림
        System.out.println("### 기본 파일 스트림 ###");
        DataStream stream1 = new FileDataStream("file1.txt");
        stream1.write(message);
        System.out.println("읽은 데이터: " + stream1.read());
        
        // 2. 압축 스트림
        System.out.println("\n### 압축 스트림 ###");
        DataStream stream2 = new CompressionDecorator(
            new FileDataStream("file2.txt.gz")
        );
        stream2.write(message);
        System.out.println("읽은 데이터: " + stream2.read());
        
        // 3. 암호화 + 압축 스트림
        System.out.println("\n### 암호화 + 압축 스트림 ###");
        DataStream stream3 = new CompressionDecorator(
            new EncryptionDecorator(
                new FileDataStream("file3.enc.gz")
            )
        );
        stream3.write(message);
        System.out.println("읽은 데이터: " + stream3.read());
        
        // 4. Base64 + 암호화 + 압축 스트림
        System.out.println("\n### Base64 + 암호화 + 압축 스트림 ###");
        DataStream stream4 = new Base64Decorator(
            new EncryptionDecorator(
                new CompressionDecorator(
                    new FileDataStream("file4.b64.enc.gz")
                )
            )
        );
        stream4.write(message);
        System.out.println("읽은 데이터: " + stream4.read());
    }
}
```

**실행 결과:**
```
### 기본 파일 스트림 ###
📁 파일에 쓰기: file1.txt
   데이터: Hello, Decorator Pattern!
📖 파일에서 읽기: file1.txt
읽은 데이터: Hello, Decorator Pattern!

### 압축 스트림 ###
🗜️ 압축: 25 → 38 bytes
📁 파일에 쓰기: file2.txt.gz
   데이터: [COMPRESSED]Hello, Decorator Pattern!
📖 파일에서 읽기: file2.txt.gz
📦 압축 해제: 38 → 25 bytes
읽은 데이터: Hello, Decorator Pattern!

### 암호화 + 압축 스트림 ###
🔒 암호화: Hello, Decorator Pattern! → !nrettaP rotaroceD ,olleH
🗜️ 압축: 25 → 38 bytes
📁 파일에 쓰기: file3.enc.gz
...
```

---

### 예제 2: 알림 시스템 ⭐⭐⭐

```java
/**
 * Component: 알림 인터페이스
 */
public interface Notifier {
    void send(String message);
}

/**
 * ConcreteComponent: 기본 이메일 알림
 */
public class EmailNotifier implements Notifier {
    private String email;
    
    public EmailNotifier(String email) {
        this.email = email;
    }
    
    @Override
    public void send(String message) {
        System.out.println("📧 Email to " + email + ": " + message);
    }
}

/**
 * Decorator: 알림 데코레이터
 */
public abstract class NotifierDecorator implements Notifier {
    protected Notifier notifier;
    
    public NotifierDecorator(Notifier notifier) {
        this.notifier = notifier;
    }
    
    @Override
    public void send(String message) {
        notifier.send(message);
    }
}

/**
 * ConcreteDecorator 1: SMS 추가
 */
public class SMSDecorator extends NotifierDecorator {
    private String phoneNumber;
    
    public SMSDecorator(Notifier notifier, String phoneNumber) {
        super(notifier);
        this.phoneNumber = phoneNumber;
    }
    
    @Override
    public void send(String message) {
        super.send(message);
        System.out.println("📱 SMS to " + phoneNumber + ": " + message);
    }
}

/**
 * ConcreteDecorator 2: Slack 추가
 */
public class SlackDecorator extends NotifierDecorator {
    private String channel;
    
    public SlackDecorator(Notifier notifier, String channel) {
        super(notifier);
        this.channel = channel;
    }
    
    @Override
    public void send(String message) {
        super.send(message);
        System.out.println("💬 Slack to #" + channel + ": " + message);
    }
}

/**
 * ConcreteDecorator 3: Facebook 추가
 */
public class FacebookDecorator extends NotifierDecorator {
    private String accountName;
    
    public FacebookDecorator(Notifier notifier, String accountName) {
        super(notifier);
        this.accountName = accountName;
    }
    
    @Override
    public void send(String message) {
        super.send(message);
        System.out.println("👥 Facebook to " + accountName + ": " + message);
    }
}

/**
 * 사용 예제
 */
public class NotifierDecoratorExample {
    public static void main(String[] args) {
        // 1. 기본 이메일만
        System.out.println("=== 이메일만 ===");
        Notifier notifier1 = new EmailNotifier("user@example.com");
        notifier1.send("서버 점검 안내");
        
        // 2. 이메일 + SMS
        System.out.println("\n=== 이메일 + SMS ===");
        Notifier notifier2 = new SMSDecorator(
            new EmailNotifier("user@example.com"),
            "010-1234-5678"
        );
        notifier2.send("긴급 알림!");
        
        // 3. 이메일 + SMS + Slack
        System.out.println("\n=== 이메일 + SMS + Slack ===");
        Notifier notifier3 = new SlackDecorator(
            new SMSDecorator(
                new EmailNotifier("user@example.com"),
                "010-1234-5678"
            ),
            "general"
        );
        notifier3.send("배포 완료");
        
        // 4. 모든 채널
        System.out.println("\n=== 모든 채널 ===");
        Notifier notifier4 = new FacebookDecorator(
            new SlackDecorator(
                new SMSDecorator(
                    new EmailNotifier("user@example.com"),
                    "010-1234-5678"
                ),
                "general"
            ),
            "Company Page"
        );
        notifier4.send("중요 공지사항");
    }
}
```

**실행 결과:**
```
=== 이메일만 ===
📧 Email to user@example.com: 서버 점검 안내

=== 이메일 + SMS ===
📧 Email to user@example.com: 긴급 알림!
📱 SMS to 010-1234-5678: 긴급 알림!

=== 이메일 + SMS + Slack ===
📧 Email to user@example.com: 배포 완료
📱 SMS to 010-1234-5678: 배포 완료
💬 Slack to #general: 배포 완료

=== 모든 채널 ===
📧 Email to user@example.com: 중요 공지사항
📱 SMS to 010-1234-5678: 중요 공지사항
💬 Slack to #general: 중요 공지사항
👥 Facebook to Company Page: 중요 공지사항
```

---

### 예제 3: 텍스트 포매터 ⭐⭐

```java
/**
 * Component
 */
public interface Text {
    String getContent();
}

/**
 * ConcreteComponent
 */
public class PlainText implements Text {
    private String content;
    
    public PlainText(String content) {
        this.content = content;
    }
    
    @Override
    public String getContent() {
        return content;
    }
}

/**
 * Decorator
 */
public abstract class TextDecorator implements Text {
    protected Text text;
    
    public TextDecorator(Text text) {
        this.text = text;
    }
    
    @Override
    public String getContent() {
        return text.getContent();
    }
}

/**
 * ConcreteDecorator: Bold
 */
public class BoldDecorator extends TextDecorator {
    public BoldDecorator(Text text) {
        super(text);
    }
    
    @Override
    public String getContent() {
        return "<b>" + text.getContent() + "</b>";
    }
}

/**
 * ConcreteDecorator: Italic
 */
public class ItalicDecorator extends TextDecorator {
    public ItalicDecorator(Text text) {
        super(text);
    }
    
    @Override
    public String getContent() {
        return "<i>" + text.getContent() + "</i>";
    }
}

/**
 * ConcreteDecorator: Underline
 */
public class UnderlineDecorator extends TextDecorator {
    public UnderlineDecorator(Text text) {
        super(text);
    }
    
    @Override
    public String getContent() {
        return "<u>" + text.getContent() + "</u>";
    }
}

/**
 * ConcreteDecorator: Color
 */
public class ColorDecorator extends TextDecorator {
    private String color;
    
    public ColorDecorator(Text text, String color) {
        super(text);
        this.color = color;
    }
    
    @Override
    public String getContent() {
        return "<span style='color:" + color + "'>" + 
               text.getContent() + "</span>";
    }
}

/**
 * 사용 예제
 */
public class TextDecoratorExample {
    public static void main(String[] args) {
        String content = "Hello, World!";
        
        // 다양한 조합
        Text text1 = new BoldDecorator(new PlainText(content));
        System.out.println("Bold: " + text1.getContent());
        
        Text text2 = new ItalicDecorator(
            new BoldDecorator(new PlainText(content))
        );
        System.out.println("Bold + Italic: " + text2.getContent());
        
        Text text3 = new UnderlineDecorator(
            new ItalicDecorator(
                new BoldDecorator(new PlainText(content))
            )
        );
        System.out.println("Bold + Italic + Underline: " + text3.getContent());
        
        Text text4 = new ColorDecorator(
            new UnderlineDecorator(
                new BoldDecorator(new PlainText(content))
            ),
            "red"
        );
        System.out.println("Bold + Underline + Red: " + text4.getContent());
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **유연성** | 런타임에 기능 추가/제거 | 커피 옵션 자유 조합 |
| **OCP 준수** | 기존 코드 수정 없이 확장 | 새 데코레이터 추가 |
| **단일 책임** | 각 데코레이터는 하나의 기능 | MilkDecorator는 우유만 |
| **조합 자유** | 여러 데코레이터 조합 가능 | 압축+암호화+인코딩 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **복잡도** | 작은 객체 많이 생성 | 필요시에만 사용 |
| **디버깅 어려움** | 레이어가 많으면 추적 힘듦 | 명확한 네이밍 |
| **순서 의존** | 데코레이터 순서가 중요 | 문서화 |

---

## 7. 안티패턴

### ❌ 안티패턴 1: 과도한 데코레이터

```java
// 잘못된 예: 너무 많은 레이어
Text text = new ColorDecorator(
    new SizeDecorator(
        new FontDecorator(
            new UnderlineDecorator(
                new ItalicDecorator(
                    new BoldDecorator(
                        new PlainText("Hello")
                    )
                )
            )
        )
    ),
    "red"
);
// 가독성 떨어짐!
```

---

## 8. 핵심 정리

### 📌 Decorator 패턴 체크리스트

```
✅ Component 인터페이스 정의
✅ ConcreteComponent 구현
✅ Decorator 추상 클래스
✅ ConcreteDecorator 구현
✅ 기능 조합 가능
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 동적 기능 추가 | ⭐⭐⭐ | 런타임 조합 |
| 상속 대안 | ⭐⭐⭐ | 조합 폭발 방지 |
| 기능 조합 | ⭐⭐⭐ | 자유로운 조합 |

### 💡 핵심 포인트

1. **상속보다 조합**
2. **런타임에 기능 추가**
3. **투명성 유지** (같은 인터페이스)
4. **Java I/O가 대표 예**

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Adapter](01-Adapter.md) | [다음: Proxy →](03-Proxy.md)**

</div>
