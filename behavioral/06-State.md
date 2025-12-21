# State Pattern (상태 패턴)

> **"객체의 상태에 따라 행동을 변경하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [장단점](#6-장단점)
7. [Strategy vs State](#7-strategy-vs-state)
8. [핵심 정리](#8-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 상태 체크가 모든 메서드에
public class GumballMachine {
    private int state; // 0=품절, 1=동전없음, 2=동전있음, 3=판매중
    
    public void insertCoin() {
        if (state == 0) {
            System.out.println("품절입니다");
        } else if (state == 1) {
            state = 2;
            System.out.println("동전 투입됨");
        } else if (state == 2) {
            System.out.println("이미 동전이 있습니다");
        } else if (state == 3) {
            System.out.println("잠시만 기다려주세요");
        }
    }
    
    public void turnCrank() {
        if (state == 0) {
            System.out.println("품절입니다");
        } else if (state == 1) {
            System.out.println("동전을 넣어주세요");
        } else if (state == 2) {
            state = 3;
            dispense();
        } else if (state == 3) {
            System.out.println("한 번만 돌려주세요");
        }
    }
    
    // 모든 메서드마다 상태 체크 반복!
    // 새 상태 추가? 모든 메서드 수정!
}

// 문제 2: 복잡한 조건문
public class TCPConnection {
    private String state = "CLOSED";
    
    public void open() {
        if (state.equals("CLOSED")) {
            state = "LISTEN";
        } else if (state.equals("LISTEN")) {
            // 이미 열림
        } else if (state.equals("ESTABLISHED")) {
            // 이미 연결됨
        }
        // 상태마다 다른 동작!
    }
    
    public void close() {
        if (state.equals("CLOSED")) {
            // 이미 닫힘
        } else if (state.equals("LISTEN")) {
            state = "CLOSED";
        } else if (state.equals("ESTABLISHED")) {
            state = "CLOSED";
        }
    }
    
    // if-else 지옥!
}

// 문제 3: 상태 전환 로직 분산
public class Order {
    private String status = "NEW";
    
    public void pay() {
        if (status.equals("NEW")) {
            status = "PAID";
            sendConfirmationEmail();
        }
    }
    
    public void ship() {
        if (status.equals("PAID")) {
            status = "SHIPPED";
            updateInventory();
        }
    }
    
    public void deliver() {
        if (status.equals("SHIPPED")) {
            status = "DELIVERED";
            notifyCustomer();
        }
    }
    
    // 상태 전환이 여러 메서드에 흩어짐!
}

// 문제 4: 상태별 불가능한 동작 처리
public class AudioPlayer {
    private String state = "STOPPED";
    
    public void play() {
        if (state.equals("PLAYING")) {
            System.out.println("이미 재생 중");
            return;
        }
        // 재생 로직
        state = "PLAYING";
    }
    
    public void pause() {
        if (state.equals("STOPPED")) {
            System.out.println("재생 중이 아닙니다");
            return;
        }
        // 일시정지 로직
        state = "PAUSED";
    }
    
    // 각 메서드마다 상태 유효성 체크!
}
```

### ⚡ 핵심 문제

1. **조건문 폭발**: 모든 메서드에 상태 체크
2. **확장 어려움**: 새 상태 추가 시 모든 메서드 수정
3. **분산된 로직**: 상태 전환 로직이 흩어짐
4. **가독성 저하**: if-else로 인한 복잡도

---

## 2. 패턴 정의

### 📖 정의

> **객체의 내부 상태가 변경될 때 객체의 행동을 변경하는 패턴. 객체가 마치 클래스를 바꾼 것처럼 보인다.**

### 🎯 목적

- **상태 캡슐화**: 각 상태를 별도 클래스로
- **조건문 제거**: if-else 대신 다형성
- **OCP 준수**: 새 상태 추가 시 기존 코드 불변
- **명확한 전환**: 상태 전환 로직 명확화

### 💡 핵심 아이디어

```java
// Before: if-else로 상태 체크
if (state == PLAYING) {
    // 재생 중 동작
} else if (state == PAUSED) {
    // 일시정지 동작
}

// After: State 객체로 위임
currentState.play(); // 현재 상태가 알아서 처리!
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌─────────────┐
│   Context   │  ← 상태를 가진 객체
├─────────────┤
│ - state     │───┐
│ + request() │   │ has-a
└─────────────┘   │
                  │
                  ▼
          ┌──────────────┐
          │    State     │  ← 상태 인터페이스
          ├──────────────┤
          │ + handle()   │
          └──────────────┘
                  △
                  │ implements
         ┌────────┼────────┐
         │        │        │
┌────────────┐ ┌─────────┐ ┌─────────┐
│StateA      │ │StateB   │ │StateC   │
├────────────┤ ├─────────┤ ├─────────┤
│+ handle()  │ │+handle()│ │+handle()│
└────────────┘ └─────────┘ └─────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **State** | 상태 인터페이스 | `State` |
| **ConcreteState** | 구체적 상태 | `PlayingState` |
| **Context** | 현재 상태 관리 | `AudioPlayer` |

---

## 4. 구현 방법

### 기본 구현: 오디오 플레이어 ⭐⭐⭐

```java
/**
 * Context: 오디오 플레이어
 */
public class AudioPlayer {
    private State state;
    private String song;
    
    public AudioPlayer() {
        this.state = new StoppedState(this);
    }
    
    public void setState(State state) {
        System.out.println("🔄 상태 변경: " + this.state.getClass().getSimpleName() + 
                " → " + state.getClass().getSimpleName());
        this.state = state;
    }
    
    public void play() {
        state.play();
    }
    
    public void pause() {
        state.pause();
    }
    
    public void stop() {
        state.stop();
    }
    
    public void setSong(String song) {
        this.song = song;
    }
    
    public String getSong() {
        return song;
    }
}

/**
 * State: 상태 인터페이스
 */
public interface State {
    void play();
    void pause();
    void stop();
}

/**
 * ConcreteState 1: 정지 상태
 */
public class StoppedState implements State {
    private AudioPlayer player;
    
    public StoppedState(AudioPlayer player) {
        this.player = player;
    }
    
    @Override
    public void play() {
        System.out.println("▶️ 재생 시작: " + player.getSong());
        player.setState(new PlayingState(player));
    }
    
    @Override
    public void pause() {
        System.out.println("⚠️ 재생 중이 아닙니다");
    }
    
    @Override
    public void stop() {
        System.out.println("⚠️ 이미 정지 상태입니다");
    }
}

/**
 * ConcreteState 2: 재생 상태
 */
public class PlayingState implements State {
    private AudioPlayer player;
    
    public PlayingState(AudioPlayer player) {
        this.player = player;
    }
    
    @Override
    public void play() {
        System.out.println("⚠️ 이미 재생 중입니다");
    }
    
    @Override
    public void pause() {
        System.out.println("⏸️ 일시정지");
        player.setState(new PausedState(player));
    }
    
    @Override
    public void stop() {
        System.out.println("⏹️ 정지");
        player.setState(new StoppedState(player));
    }
}

/**
 * ConcreteState 3: 일시정지 상태
 */
public class PausedState implements State {
    private AudioPlayer player;
    
    public PausedState(AudioPlayer player) {
        this.player = player;
    }
    
    @Override
    public void play() {
        System.out.println("▶️ 재생 재개");
        player.setState(new PlayingState(player));
    }
    
    @Override
    public void pause() {
        System.out.println("⚠️ 이미 일시정지 상태입니다");
    }
    
    @Override
    public void stop() {
        System.out.println("⏹️ 정지");
        player.setState(new StoppedState(player));
    }
}

/**
 * 사용 예제
 */
public class StateExample {
    public static void main(String[] args) {
        AudioPlayer player = new AudioPlayer();
        player.setSong("Bohemian Rhapsody");
        
        System.out.println("=== 오디오 플레이어 ===\n");
        
        // 정지 → 재생
        player.play();
        
        // 재생 → 일시정지
        System.out.println();
        player.pause();
        
        // 일시정지 → 재생
        System.out.println();
        player.play();
        
        // 재생 → 정지
        System.out.println();
        player.stop();
        
        // 잘못된 동작들
        System.out.println("\n=== 잘못된 동작 ===\n");
        player.pause(); // 정지 상태에서 일시정지
        player.stop();  // 이미 정지 상태
    }
}
```

**실행 결과:**
```
=== 오디오 플레이어 ===

▶️ 재생 시작: Bohemian Rhapsody
🔄 상태 변경: StoppedState → PlayingState

⏸️ 일시정지
🔄 상태 변경: PlayingState → PausedState

▶️ 재생 재개
🔄 상태 변경: PausedState → PlayingState

⏹️ 정지
🔄 상태 변경: PlayingState → StoppedState

=== 잘못된 동작 ===

⚠️ 재생 중이 아닙니다
⚠️ 이미 정지 상태입니다
```

---

## 5. 실전 예제

### 예제 1: 주문 시스템 ⭐⭐⭐

```java
/**
 * Context: 주문
 */
public class Order {
    private State state;
    private String orderId;
    private double amount;
    
    public Order(String orderId, double amount) {
        this.orderId = orderId;
        this.amount = amount;
        this.state = new NewState(this);
    }
    
    public void setState(State state) {
        System.out.println("📦 주문 " + orderId + " 상태: " + 
                state.getClass().getSimpleName());
        this.state = state;
    }
    
    public void pay() {
        state.pay();
    }
    
    public void ship() {
        state.ship();
    }
    
    public void deliver() {
        state.deliver();
    }
    
    public void cancel() {
        state.cancel();
    }
    
    public String getOrderId() {
        return orderId;
    }
    
    public double getAmount() {
        return amount;
    }
}

/**
 * State: 주문 상태
 */
public interface State {
    void pay();
    void ship();
    void deliver();
    void cancel();
}

/**
 * ConcreteState: 신규 주문
 */
public class NewState implements State {
    private Order order;
    
    public NewState(Order order) {
        this.order = order;
    }
    
    @Override
    public void pay() {
        System.out.println("💳 결제 처리 중... $" + order.getAmount());
        order.setState(new PaidState(order));
    }
    
    @Override
    public void ship() {
        System.out.println("❌ 결제가 필요합니다");
    }
    
    @Override
    public void deliver() {
        System.out.println("❌ 결제가 필요합니다");
    }
    
    @Override
    public void cancel() {
        System.out.println("🚫 주문 취소");
        order.setState(new CancelledState(order));
    }
}

/**
 * ConcreteState: 결제 완료
 */
public class PaidState implements State {
    private Order order;
    
    public PaidState(Order order) {
        this.order = order;
    }
    
    @Override
    public void pay() {
        System.out.println("❌ 이미 결제되었습니다");
    }
    
    @Override
    public void ship() {
        System.out.println("🚚 배송 시작");
        order.setState(new ShippedState(order));
    }
    
    @Override
    public void deliver() {
        System.out.println("❌ 배송이 필요합니다");
    }
    
    @Override
    public void cancel() {
        System.out.println("💸 환불 처리 중...");
        order.setState(new CancelledState(order));
    }
}

/**
 * ConcreteState: 배송 중
 */
public class ShippedState implements State {
    private Order order;
    
    public ShippedState(Order order) {
        this.order = order;
    }
    
    @Override
    public void pay() {
        System.out.println("❌ 이미 결제되었습니다");
    }
    
    @Override
    public void ship() {
        System.out.println("❌ 이미 배송 중입니다");
    }
    
    @Override
    public void deliver() {
        System.out.println("📬 배송 완료");
        order.setState(new DeliveredState(order));
    }
    
    @Override
    public void cancel() {
        System.out.println("❌ 배송 중에는 취소할 수 없습니다");
    }
}

/**
 * ConcreteState: 배송 완료
 */
public class DeliveredState implements State {
    private Order order;
    
    public DeliveredState(Order order) {
        this.order = order;
    }
    
    @Override
    public void pay() {
        System.out.println("❌ 이미 결제되었습니다");
    }
    
    @Override
    public void ship() {
        System.out.println("❌ 이미 배송되었습니다");
    }
    
    @Override
    public void deliver() {
        System.out.println("❌ 이미 배송 완료되었습니다");
    }
    
    @Override
    public void cancel() {
        System.out.println("❌ 배송 완료 후에는 취소할 수 없습니다");
    }
}

/**
 * ConcreteState: 취소됨
 */
public class CancelledState implements State {
    private Order order;
    
    public CancelledState(Order order) {
        this.order = order;
    }
    
    @Override
    public void pay() {
        System.out.println("❌ 취소된 주문입니다");
    }
    
    @Override
    public void ship() {
        System.out.println("❌ 취소된 주문입니다");
    }
    
    @Override
    public void deliver() {
        System.out.println("❌ 취소된 주문입니다");
    }
    
    @Override
    public void cancel() {
        System.out.println("❌ 이미 취소되었습니다");
    }
}

/**
 * 사용 예제
 */
public class OrderStateExample {
    public static void main(String[] args) {
        Order order = new Order("ORD-001", 99.99);
        
        System.out.println("=== 정상 주문 흐름 ===\n");
        order.pay();
        order.ship();
        order.deliver();
        
        System.out.println("\n=== 새 주문 (취소) ===\n");
        Order order2 = new Order("ORD-002", 49.99);
        order2.cancel();
        order2.pay(); // 취소된 주문
        
        System.out.println("\n=== 잘못된 동작 ===\n");
        Order order3 = new Order("ORD-003", 149.99);
        order3.ship(); // 결제 안 됨
        order3.deliver(); // 배송 안 됨
    }
}
```

---

### 예제 2: TCP 연결 ⭐⭐

```java
/**
 * Context: TCP 연결
 */
public class TCPConnection {
    private State state;
    
    public TCPConnection() {
        this.state = new ClosedState(this);
    }
    
    public void setState(State state) {
        System.out.println("🔄 상태: " + state.getClass().getSimpleName());
        this.state = state;
    }
    
    public void open() {
        state.open();
    }
    
    public void close() {
        state.close();
    }
    
    public void acknowledge() {
        state.acknowledge();
    }
}

/**
 * State: 연결 상태
 */
public interface State {
    void open();
    void close();
    void acknowledge();
}

/**
 * ConcreteState: 닫힘
 */
public class ClosedState implements State {
    private TCPConnection connection;
    
    public ClosedState(TCPConnection connection) {
        this.connection = connection;
    }
    
    @Override
    public void open() {
        System.out.println("📡 연결 시작...");
        connection.setState(new ListenState(connection));
    }
    
    @Override
    public void close() {
        System.out.println("⚠️ 이미 닫혀있습니다");
    }
    
    @Override
    public void acknowledge() {
        System.out.println("⚠️ 연결이 없습니다");
    }
}

/**
 * ConcreteState: 대기 중
 */
public class ListenState implements State {
    private TCPConnection connection;
    
    public ListenState(TCPConnection connection) {
        this.connection = connection;
    }
    
    @Override
    public void open() {
        System.out.println("⚠️ 이미 연결 중입니다");
    }
    
    @Override
    public void close() {
        System.out.println("🔌 연결 종료");
        connection.setState(new ClosedState(connection));
    }
    
    @Override
    public void acknowledge() {
        System.out.println("✅ 연결 확립");
        connection.setState(new EstablishedState(connection));
    }
}

/**
 * ConcreteState: 연결됨
 */
public class EstablishedState implements State {
    private TCPConnection connection;
    
    public EstablishedState(TCPConnection connection) {
        this.connection = connection;
    }
    
    @Override
    public void open() {
        System.out.println("⚠️ 이미 연결되어 있습니다");
    }
    
    @Override
    public void close() {
        System.out.println("🔌 연결 종료");
        connection.setState(new ClosedState(connection));
    }
    
    @Override
    public void acknowledge() {
        System.out.println("✅ 데이터 전송 중...");
    }
}

/**
 * 사용 예제
 */
public class TCPStateExample {
    public static void main(String[] args) {
        TCPConnection conn = new TCPConnection();
        
        System.out.println("=== TCP 연결 ===\n");
        conn.open();
        conn.acknowledge();
        conn.close();
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **조건문 제거** | if-else 대신 다형성 | 상태별 클래스 |
| **OCP 준수** | 새 상태 추가 용이 | 기존 코드 불변 |
| **SRP 준수** | 각 상태가 독립적 | 명확한 책임 |
| **명확한 전환** | 상태 전환 명시적 | setState() |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **클래스 증가** | 상태마다 클래스 | 필요시에만 |
| **복잡도** | 단순 상태에 과함 | enum 고려 |

---

## 7. Strategy vs State

### 🔍 비교표

| 특징 | Strategy | State |
|------|----------|-------|
| **목적** | 알고리즘 교체 | 상태별 행동 |
| **변경 주체** | 클라이언트 | Context 스스로 |
| **관계** | 독립적 | 상태 전환 |
| **예시** | 정렬 알고리즘 | 주문 상태 |

---

## 8. 핵심 정리

### 📌 State 패턴 체크리스트

```
✅ State 인터페이스 정의
✅ ConcreteState 구현
✅ Context 작성
✅ setState() 메서드
✅ 상태 전환 로직
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 많은 조건문 | ⭐⭐⭐ | 다형성으로 |
| 상태 의존 행동 | ⭐⭐⭐ | 명확한 분리 |
| 상태 전환 | ⭐⭐⭐ | 명시적 관리 |

### 💡 핵심 포인트

1. **상태를 객체로**
2. **조건문 제거**
3. **명확한 전환**
4. **자동 전환 가능**

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Iterator](05-Iterator.md)** | **[다음: Chain of Responsibility →](07-ChainOfResponsibility.md)**

</div>
