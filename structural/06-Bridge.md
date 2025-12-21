# Bridge Pattern (브릿지 패턴)

> **"추상화와 구현을 분리하여 독립적으로 확장하자"**

[← 이전: Facade](05-Facade.md) | [목차로 돌아가기](../README.md) | [다음: Flyweight →](07-Flyweight.md)

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
// 문제 1: 클래스 폭발 (Cartesian Product)
// Shape (도형) x Color (색상) = 조합 폭발!

public abstract class Shape {
    public abstract void draw();
}

public class RedCircle extends Shape { }
public class BlueCircle extends Shape { }
public class RedRectangle extends Shape { }
public class BlueRectangle extends Shape { }
// Red/Blue/Green x Circle/Rectangle/Triangle
// = 3 x 3 = 9개 클래스!
// 새 색상 추가 시 모든 도형마다 클래스 추가!

// 문제 2: 다중 차원 확장이 어려움
public abstract class RemoteControl {
    public abstract void turnOn();
    public abstract void turnOff();
}

public class SamsungTVRemote extends RemoteControl { }
public class LGTV Remote extends RemoteControl { }
// 기본 리모콘, 고급 리모콘도 추가하려면?
// Samsung기본, Samsung고급, LG기본, LG고급...
// 2개 차원 = 클래스 폭발!

// 문제 3: 상속으로 묶여있어 변경이 어려움
public class WindowsButton extends Button {
    @Override
    public void render() {
        // Windows 스타일 렌더링
    }
}

public class MacButton extends Button {
    @Override
    public void render() {
        // Mac 스타일 렌더링
    }
}

// 렌더링 방식만 바꾸고 싶은데 전체를 상속해야 함!

// 문제 4: 확장 시 기존 코드 수정
public class Database {
    private String type; // "MySQL", "PostgreSQL"
    
    public void connect() {
        if (type.equals("MySQL")) {
            // MySQL 연결
        } else if (type.equals("PostgreSQL")) {
            // PostgreSQL 연결
        }
        // 새 DB 추가 시 이 메서드 수정!
    }
}
```

### ⚡ 핵심 문제

1. **클래스 폭발**: 조합마다 클래스 생성
2. **다중 차원 확장**: 여러 방향으로 확장 어려움
3. **강한 결합**: 추상화와 구현이 묶임
4. **OCP 위반**: 확장 시 기존 코드 수정

---

## 2. 패턴 정의

### 📖 정의

> **추상화(Abstraction)와 구현(Implementation)을 분리하여 각각 독립적으로 변경할 수 있도록 하는 패턴**

### 🎯 목적

- **분리**: 추상화와 구현을 독립적으로
- **확장성**: 두 계층을 독립적으로 확장
- **조합 자유**: 런타임에 구현 교체 가능
- **클래스 폭발 방지**: 조합 수 감소

### 💡 핵심 아이디어

```java
// Before: 상속으로 결합
RedCircle, BlueCircle, RedRectangle, BlueRectangle...
// 2개 차원 x 2개 옵션 = 4개 클래스

// After: 조합으로 분리
Shape shape = new Circle(new RedColor());
// 2개 차원 + 2개 옵션 = 4개 클래스 (합)
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌────────────────────┐
│   Abstraction      │  ← 추상화 계층
├────────────────────┤
│ - impl: Implementor│───┐
│ + operation()      │   │ has-a (bridge)
└────────────────────┘   │
         △               │
         │               │
┌────────────────────┐   │
│RefinedAbstraction  │   │
└────────────────────┘   │
                         │
                         ▼
              ┌──────────────────┐
              │  Implementor     │  ← 구현 계층
              ├──────────────────┤
              │ + operationImpl()│
              └──────────────────┘
                       △
                       │
              ┌────────┴────────┐
              │                 │
    ┌─────────────────┐ ┌──────────────────┐
    │ConcreteImplA    │ │ConcreteImplB     │
    └─────────────────┘ └──────────────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **Abstraction** | 추상화 인터페이스 | `Shape` |
| **RefinedAbstraction** | 확장된 추상화 | `Circle`, `Rectangle` |
| **Implementor** | 구현 인터페이스 | `Color` |
| **ConcreteImplementor** | 구체적 구현 | `Red`, `Blue` |

---

## 4. 구현 방법

### 기본 구현: 도형과 색상 ⭐⭐⭐

```java
/**
 * Implementor: 색상 인터페이스
 */
public interface Color {
    void applyColor();
}

/**
 * ConcreteImplementor 1: 빨간색
 */
public class RedColor implements Color {
    @Override
    public void applyColor() {
        System.out.println("🔴 빨간색으로 칠하기");
    }
}

/**
 * ConcreteImplementor 2: 파란색
 */
public class BlueColor implements Color {
    @Override
    public void applyColor() {
        System.out.println("🔵 파란색으로 칠하기");
    }
}

/**
 * ConcreteImplementor 3: 초록색
 */
public class GreenColor implements Color {
    @Override
    public void applyColor() {
        System.out.println("🟢 초록색으로 칠하기");
    }
}

/**
 * Abstraction: 도형
 */
public abstract class Shape {
    protected Color color; // Bridge!
    
    public Shape(Color color) {
        this.color = color;
    }
    
    public abstract void draw();
}

/**
 * RefinedAbstraction 1: 원
 */
public class Circle extends Shape {
    public Circle(Color color) {
        super(color);
    }
    
    @Override
    public void draw() {
        System.out.print("⭕ 원 그리기 - ");
        color.applyColor();
    }
}

/**
 * RefinedAbstraction 2: 사각형
 */
public class Rectangle extends Shape {
    public Rectangle(Color color) {
        super(color);
    }
    
    @Override
    public void draw() {
        System.out.print("⬜ 사각형 그리기 - ");
        color.applyColor();
    }
}

/**
 * RefinedAbstraction 3: 삼각형
 */
public class Triangle extends Shape {
    public Triangle(Color color) {
        super(color);
    }
    
    @Override
    public void draw() {
        System.out.print("🔺 삼각형 그리기 - ");
        color.applyColor();
    }
}

/**
 * 사용 예제
 */
public class BridgeExample {
    public static void main(String[] args) {
        System.out.println("=== Bridge 패턴 - 조합의 자유! ===\n");
        
        // 다양한 조합 생성
        Shape redCircle = new Circle(new RedColor());
        Shape blueCircle = new Circle(new BlueColor());
        Shape greenRectangle = new Rectangle(new GreenColor());
        Shape redTriangle = new Triangle(new RedColor());
        
        // 그리기
        redCircle.draw();
        blueCircle.draw();
        greenRectangle.draw();
        redTriangle.draw();
        
        System.out.println("\n=== 클래스 개수 비교 ===");
        System.out.println("Before Bridge: 3 shapes x 3 colors = 9 classes");
        System.out.println("After Bridge: 3 shapes + 3 colors = 6 classes");
        System.out.println("절반으로 감소! ✨");
    }
}
```

**실행 결과:**
```
=== Bridge 패턴 - 조합의 자유! ===

⭕ 원 그리기 - 🔴 빨간색으로 칠하기
⭕ 원 그리기 - 🔵 파란색으로 칠하기
⬜ 사각형 그리기 - 🟢 초록색으로 칠하기
🔺 삼각형 그리기 - 🔴 빨간색으로 칠하기

=== 클래스 개수 비교 ===
Before Bridge: 3 shapes x 3 colors = 9 classes
After Bridge: 3 shapes + 3 colors = 6 classes
절반으로 감소! ✨
```

---

## 5. 실전 예제

### 예제 1: 리모콘과 TV ⭐⭐⭐

```java
/**
 * Implementor: TV 인터페이스
 */
public interface TV {
    void on();
    void off();
    void setChannel(int channel);
    void setVolume(int volume);
}

/**
 * ConcreteImplementor 1: Samsung TV
 */
public class SamsungTV implements TV {
    private int channel = 1;
    private int volume = 10;
    
    @Override
    public void on() {
        System.out.println("📺 Samsung TV 켜기");
    }
    
    @Override
    public void off() {
        System.out.println("📺 Samsung TV 끄기");
    }
    
    @Override
    public void setChannel(int channel) {
        this.channel = channel;
        System.out.println("📺 Samsung TV 채널: " + channel);
    }
    
    @Override
    public void setVolume(int volume) {
        this.volume = volume;
        System.out.println("📺 Samsung TV 볼륨: " + volume);
    }
}

/**
 * ConcreteImplementor 2: LG TV
 */
public class LGTV implements TV {
    private int channel = 1;
    private int volume = 10;
    
    @Override
    public void on() {
        System.out.println("📺 LG TV 켜기");
    }
    
    @Override
    public void off() {
        System.out.println("📺 LG TV 끄기");
    }
    
    @Override
    public void setChannel(int channel) {
        this.channel = channel;
        System.out.println("📺 LG TV 채널: " + channel);
    }
    
    @Override
    public void setVolume(int volume) {
        this.volume = volume;
        System.out.println("📺 LG TV 볼륨: " + volume);
    }
}

/**
 * Abstraction: 리모콘
 */
public abstract class RemoteControl {
    protected TV tv; // Bridge!
    
    public RemoteControl(TV tv) {
        this.tv = tv;
    }
    
    public void turnOn() {
        tv.on();
    }
    
    public void turnOff() {
        tv.off();
    }
    
    public void setChannel(int channel) {
        tv.setChannel(channel);
    }
}

/**
 * RefinedAbstraction 1: 기본 리모콘
 */
public class BasicRemote extends RemoteControl {
    public BasicRemote(TV tv) {
        super(tv);
    }
    
    public void channelUp() {
        System.out.println("🔼 채널 올리기");
    }
    
    public void channelDown() {
        System.out.println("🔽 채널 내리기");
    }
}

/**
 * RefinedAbstraction 2: 고급 리모콘
 */
public class AdvancedRemote extends RemoteControl {
    public AdvancedRemote(TV tv) {
        super(tv);
    }
    
    public void mute() {
        System.out.println("🔇 음소거");
        tv.setVolume(0);
    }
    
    public void setVolume(int volume) {
        System.out.println("🔊 볼륨 조절");
        tv.setVolume(volume);
    }
}

/**
 * 사용 예제
 */
public class RemoteExample {
    public static void main(String[] args) {
        // Samsung TV + 기본 리모콘
        System.out.println("=== Samsung TV + 기본 리모콘 ===");
        TV samsungTV = new SamsungTV();
        RemoteControl basicRemote = new BasicRemote(samsungTV);
        basicRemote.turnOn();
        basicRemote.setChannel(7);
        ((BasicRemote) basicRemote).channelUp();
        basicRemote.turnOff();
        
        // LG TV + 고급 리모콘
        System.out.println("\n=== LG TV + 고급 리모콘 ===");
        TV lgTV = new LGTV();
        AdvancedRemote advancedRemote = new AdvancedRemote(lgTV);
        advancedRemote.turnOn();
        advancedRemote.setChannel(11);
        advancedRemote.setVolume(15);
        advancedRemote.mute();
        advancedRemote.turnOff();
        
        System.out.println("\n✨ 리모콘과 TV를 독립적으로 확장 가능!");
    }
}
```

---

### 예제 2: 메시지 전송 시스템 ⭐⭐

```java
/**
 * Implementor: 메시지 발신자
 */
public interface MessageSender {
    void sendMessage(String message);
}

/**
 * ConcreteImplementor: SMS 발신자
 */
public class SMSSender implements MessageSender {
    @Override
    public void sendMessage(String message) {
        System.out.println("📱 SMS 발송: " + message);
    }
}

/**
 * ConcreteImplementor: Email 발신자
 */
public class EmailSender implements MessageSender {
    @Override
    public void sendMessage(String message) {
        System.out.println("📧 Email 발송: " + message);
    }
}

/**
 * Abstraction: 메시지
 */
public abstract class Message {
    protected MessageSender sender;
    
    public Message(MessageSender sender) {
        this.sender = sender;
    }
    
    public abstract void send(String content);
}

/**
 * RefinedAbstraction: 긴급 메시지
 */
public class UrgentMessage extends Message {
    public UrgentMessage(MessageSender sender) {
        super(sender);
    }
    
    @Override
    public void send(String content) {
        String urgentContent = "[긴급] " + content;
        sender.sendMessage(urgentContent);
    }
}

/**
 * RefinedAbstraction: 일반 메시지
 */
public class NormalMessage extends Message {
    public NormalMessage(MessageSender sender) {
        super(sender);
    }
    
    @Override
    public void send(String content) {
        sender.sendMessage(content);
    }
}

/**
 * 사용 예제
 */
public class MessageExample {
    public static void main(String[] args) {
        Message urgentSMS = new UrgentMessage(new SMSSender());
        urgentSMS.send("서버 다운!");
        
        Message normalEmail = new NormalMessage(new EmailSender());
        normalEmail.send("회의 일정 공유");
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **독립적 확장** | 추상화와 구현을 분리 | TV와 리모콘 |
| **클래스 감소** | 조합 대신 구성 | n+m < n×m |
| **런타임 교체** | 구현 동적 변경 | 색상 바꾸기 |
| **OCP 준수** | 기존 코드 수정 없이 확장 | 새 TV 추가 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **복잡도** | 설계가 복잡해짐 | 필요시에만 사용 |
| **간접 호출** | 레이어 추가 | 성능 크리티컬하면 고려 |

---

## 7. 안티패턴

### ❌ 안티패턴: 불필요한 Bridge

```java
// 한 방향으로만 확장되면 불필요
public abstract class Shape {
    protected Color color; // 색상만 바뀜
}
// 도형은 추가 안 되면 Bridge 과함!
```

---

## 8. 핵심 정리

### 📌 Bridge 패턴 체크리스트

```
✅ 두 차원의 변화 파악
✅ Implementor 인터페이스
✅ ConcreteImplementor 구현
✅ Abstraction 정의
✅ RefinedAbstraction 구현
✅ 조합으로 사용
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 다차원 확장 | ⭐⭐⭐ | 독립적 변화 |
| 클래스 폭발 | ⭐⭐⭐ | 조합 감소 |
| 런타임 교체 | ⭐⭐⭐ | 유연성 |

### 💡 핵심 포인트

1. **상속보다 조합**
2. **두 계층 분리**
3. **독립적 확장**
4. **조합 폭발 방지**

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Facade](05-Facade.md) | [다음: Flyweight →](07-Flyweight.md)**

</div>
