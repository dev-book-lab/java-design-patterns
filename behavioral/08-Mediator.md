# Mediator Pattern (중재자 패턴)

> **"객체 간 복잡한 통신을 중재자가 관리하자"**

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
// 문제 1: 객체 간 복잡한 참조
public class ChatRoom {
    private User user1;
    private User user2;
    private User user3;
    
    public void sendMessage(String from, String to, String message) {
        // 모든 사용자를 알아야 함
        if (from.equals(user1.getName())) {
            if (to.equals(user2.getName())) {
                user2.receive(message);
            } else if (to.equals(user3.getName())) {
                user3.receive(message);
            }
        }
        // 사용자 추가마다 코드 수정!
    }
}

public class User {
    private User[] friends;
    
    public void sendMessage(String msg, User recipient) {
        recipient.receive(msg); // 직접 참조!
    }
}

// 문제 2: 다대다 관계의 복잡성
public class UIComponent {
    private Button button;
    private TextField textField;
    private CheckBox checkBox;
    
    public Button getButton() {
        button.addListener(() -> {
            // 텍스트 필드 업데이트
            textField.setText("...");
            // 체크박스 체크
            checkBox.setChecked(true);
        });
    }
    
    public TextField getTextField() {
        textField.addListener(() -> {
            // 버튼 활성화
            button.setEnabled(true);
            // 체크박스 업데이트
            checkBox.setEnabled(true);
        });
    }
    
    // 모든 컴포넌트가 서로를 알아야 함!
    // N개 컴포넌트 = N*(N-1) 관계!
}

// 문제 3: 의존성 스파게티
public class Flight {
    private List<Flight> allFlights;
    
    public void requestLanding() {
        // 모든 비행기에 직접 확인
        for (Flight f : allFlights) {
            if (f.isLanding()) {
                // 대기
            }
        }
    }
}

// 문제 4: 변경 전파 어려움
public class Stock {
    private List<Trader> traders;
    
    public void updatePrice(double price) {
        // 모든 거래자에게 직접 알림
        for (Trader trader : traders) {
            trader.notifyPriceChange(price);
        }
        // 새 거래자 추가? 코드 수정!
    }
}
```

### ⚡ 핵심 문제

1. **강한 결합**: 객체들이 서로를 직접 참조
2. **복잡한 관계**: 다대다 관계로 인한 복잡도
3. **재사용 어려움**: 의존성 때문에 독립 사용 불가
4. **변경 취약**: 한 객체 변경이 다른 객체에 영향

---

## 2. 패턴 정의

### 📖 정의

> **객체 간의 복잡한 통신과 제어를 캡슐화하여 객체들이 직접 참조하지 않고 중재자를 통해 상호작용하게 하는 패턴**

### 🎯 목적

- **느슨한 결합**: 객체 간 직접 참조 제거
- **통신 중앙화**: 모든 통신을 중재자가 관리
- **재사용성**: 객체를 독립적으로 사용
- **복잡도 감소**: 다대다 → 일대다

### 💡 핵심 아이디어

```java
// Before: 직접 참조
user1.sendMessage(user2, "Hello");
user2.sendMessage(user3, "Hi");
// N개 객체 = N*(N-1) 연결

// After: Mediator 통해 통신
mediator.send(user1, "Hello", user2);
// N개 객체 = N개 연결 (중재자에게만)
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌──────────────────┐
│    Mediator      │  ← 중재자 인터페이스
├──────────────────┤
│ + notify()       │
└──────────────────┘
         △
         │ implements
┌──────────────────┐
│ConcreteMediator  │
├──────────────────┤
│ - colleagues     │◄─────┐
│ + notify()       │      │
└──────────────────┘      │
         │                │
         │ manages        │
         ▼                │
┌──────────────────┐      │
│   Colleague      │      │
├──────────────────┤      │
│ - mediator       │──────┘ knows
│ + send()         │
│ + receive()      │
└──────────────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **Mediator** | 중재자 인터페이스 | `ChatMediator` |
| **ConcreteMediator** | 구체적 중재자 | `ChatRoom` |
| **Colleague** | 동료 객체 | `User` |

---

## 4. 구현 방법

### 기본 구현: 채팅방 시스템 ⭐⭐⭐

```java
/**
 * Mediator: 채팅 중재자
 */
public interface ChatMediator {
    void sendMessage(String message, User user);
    void addUser(User user);
}

/**
 * Colleague: 사용자
 */
public abstract class User {
    protected ChatMediator mediator;
    protected String name;
    
    public User(ChatMediator mediator, String name) {
        this.mediator = mediator;
        this.name = name;
    }
    
    public abstract void send(String message);
    public abstract void receive(String message);
    
    public String getName() {
        return name;
    }
}

/**
 * ConcreteMediator: 채팅방
 */
public class ChatRoom implements ChatMediator {
    private List<User> users;
    
    public ChatRoom() {
        this.users = new ArrayList<>();
    }
    
    @Override
    public void addUser(User user) {
        users.add(user);
        System.out.println("✅ " + user.getName() + "님이 입장했습니다");
    }
    
    @Override
    public void sendMessage(String message, User sender) {
        System.out.println("\n💬 [" + sender.getName() + "]: " + message);
        
        for (User user : users) {
            // 발신자 제외하고 전송
            if (user != sender) {
                user.receive(message);
            }
        }
    }
}

/**
 * ConcreteColleague: 일반 사용자
 */
public class ChatUser extends User {
    
    public ChatUser(ChatMediator mediator, String name) {
        super(mediator, name);
    }
    
    @Override
    public void send(String message) {
        System.out.println("📤 " + name + "님이 메시지 전송");
        mediator.sendMessage(message, this);
    }
    
    @Override
    public void receive(String message) {
        System.out.println("   📥 " + name + "님이 수신");
    }
}

/**
 * 사용 예제
 */
public class MediatorExample {
    public static void main(String[] args) {
        // 중재자 생성
        ChatMediator chatRoom = new ChatRoom();
        
        // 사용자 생성 및 입장
        System.out.println("=== 채팅방 입장 ===");
        User alice = new ChatUser(chatRoom, "Alice");
        User bob = new ChatUser(chatRoom, "Bob");
        User charlie = new ChatUser(chatRoom, "Charlie");
        
        chatRoom.addUser(alice);
        chatRoom.addUser(bob);
        chatRoom.addUser(charlie);
        
        // 메시지 전송
        System.out.println("\n=== 채팅 시작 ===");
        alice.send("안녕하세요!");
        
        bob.send("반갑습니다~");
        
        charlie.send("좋은 하루 되세요!");
    }
}
```

**실행 결과:**
```
=== 채팅방 입장 ===
✅ Alice님이 입장했습니다
✅ Bob님이 입장했습니다
✅ Charlie님이 입장했습니다

=== 채팅 시작 ===
📤 Alice님이 메시지 전송

💬 [Alice]: 안녕하세요!
   📥 Bob님이 수신
   📥 Charlie님이 수신
📤 Bob님이 메시지 전송

💬 [Bob]: 반갑습니다~
   📥 Alice님이 수신
   📥 Charlie님이 수신
📤 Charlie님이 메시지 전송

💬 [Charlie]: 좋은 하루 되세요!
   📥 Alice님이 수신
   📥 Bob님이 수신
```

---

## 5. 실전 예제

### 예제 1: 항공 관제 시스템 ⭐⭐⭐

```java
/**
 * Mediator: 항공 관제탑
 */
public interface ControlTower {
    void registerAircraft(Aircraft aircraft);
    void requestLanding(Aircraft aircraft);
    void requestTakeoff(Aircraft aircraft);
}

/**
 * Colleague: 항공기
 */
public abstract class Aircraft {
    protected ControlTower tower;
    protected String callSign;
    protected boolean isLanding = false;
    
    public Aircraft(ControlTower tower, String callSign) {
        this.tower = tower;
        this.callSign = callSign;
        tower.registerAircraft(this);
    }
    
    public String getCallSign() {
        return callSign;
    }
    
    public boolean isLanding() {
        return isLanding;
    }
    
    public void setLanding(boolean landing) {
        isLanding = landing;
    }
    
    public abstract void land();
    public abstract void takeoff();
    public abstract void receiveInstruction(String instruction);
}

/**
 * ConcreteMediator: 공항 관제탑
 */
public class AirportControlTower implements ControlTower {
    private List<Aircraft> aircrafts;
    private boolean runwayAvailable = true;
    
    public AirportControlTower() {
        this.aircrafts = new ArrayList<>();
    }
    
    @Override
    public void registerAircraft(Aircraft aircraft) {
        aircrafts.add(aircraft);
        System.out.println("🛫 " + aircraft.getCallSign() + " 등록됨");
    }
    
    @Override
    public void requestLanding(Aircraft aircraft) {
        System.out.println("\n📡 " + aircraft.getCallSign() + " 착륙 요청");
        
        if (runwayAvailable) {
            runwayAvailable = false;
            aircraft.setLanding(true);
            aircraft.receiveInstruction("착륙 허가");
            
            // 다른 항공기에 대기 지시
            for (Aircraft other : aircrafts) {
                if (other != aircraft) {
                    other.receiveInstruction("대기 - " + aircraft.getCallSign() + " 착륙 중");
                }
            }
        } else {
            aircraft.receiveInstruction("활주로 사용 중 - 대기");
        }
    }
    
    @Override
    public void requestTakeoff(Aircraft aircraft) {
        System.out.println("\n📡 " + aircraft.getCallSign() + " 이륙 요청");
        
        if (runwayAvailable) {
            runwayAvailable = false;
            aircraft.receiveInstruction("이륙 허가");
            
            // 이륙 완료 후 활주로 사용 가능
            runwayAvailable = true;
            
            // 다른 항공기에 알림
            for (Aircraft other : aircrafts) {
                if (other != aircraft) {
                    other.receiveInstruction("활주로 사용 가능");
                }
            }
        } else {
            aircraft.receiveInstruction("활주로 사용 중 - 대기");
        }
    }
    
    public void landingComplete(Aircraft aircraft) {
        aircraft.setLanding(false);
        runwayAvailable = true;
        System.out.println("✅ " + aircraft.getCallSign() + " 착륙 완료");
        
        // 대기 중인 항공기에 알림
        for (Aircraft other : aircrafts) {
            if (other != aircraft && other.isLanding()) {
                other.receiveInstruction("활주로 사용 가능");
            }
        }
    }
}

/**
 * ConcreteColleague: 여객기
 */
public class PassengerPlane extends Aircraft {
    
    public PassengerPlane(ControlTower tower, String callSign) {
        super(tower, callSign);
    }
    
    @Override
    public void land() {
        tower.requestLanding(this);
    }
    
    @Override
    public void takeoff() {
        tower.requestTakeoff(this);
    }
    
    @Override
    public void receiveInstruction(String instruction) {
        System.out.println("   ✈️ " + callSign + " 수신: " + instruction);
    }
}

/**
 * 사용 예제
 */
public class AirTrafficExample {
    public static void main(String[] args) {
        // 관제탑 생성
        AirportControlTower tower = new AirportControlTower();
        
        // 항공기 생성
        System.out.println("=== 항공기 등록 ===");
        Aircraft flight1 = new PassengerPlane(tower, "KE001");
        Aircraft flight2 = new PassengerPlane(tower, "OZ002");
        Aircraft flight3 = new PassengerPlane(tower, "AA003");
        
        // 착륙 시나리오
        System.out.println("\n=== 착륙 시나리오 ===");
        flight1.land();
        flight2.land(); // 대기
        
        // 착륙 완료
        tower.landingComplete(flight1);
        
        // 이륙 시나리오
        System.out.println("\n=== 이륙 시나리오 ===");
        flight3.takeoff();
    }
}
```

---

### 예제 2: UI 다이얼로그 ⭐⭐⭐

```java
/**
 * Mediator: 다이얼로그 중재자
 */
public interface DialogMediator {
    void notify(Component sender, String event);
}

/**
 * Colleague: UI 컴포넌트
 */
public abstract class Component {
    protected DialogMediator mediator;
    
    public void setMediator(DialogMediator mediator) {
        this.mediator = mediator;
    }
    
    public abstract String getName();
}

/**
 * ConcreteColleague: 버튼
 */
public class Button extends Component {
    private String name;
    
    public Button(String name) {
        this.name = name;
    }
    
    public void click() {
        System.out.println("🖱️ " + name + " 버튼 클릭");
        mediator.notify(this, "click");
    }
    
    @Override
    public String getName() {
        return name;
    }
}

/**
 * ConcreteColleague: 텍스트 필드
 */
public class TextField extends Component {
    private String name;
    private String text = "";
    
    public TextField(String name) {
        this.name = name;
    }
    
    public void setText(String text) {
        this.text = text;
        System.out.println("📝 " + name + " 텍스트 변경: " + text);
        mediator.notify(this, "textChanged");
    }
    
    public String getText() {
        return text;
    }
    
    @Override
    public String getName() {
        return name;
    }
}

/**
 * ConcreteColleague: 체크박스
 */
public class CheckBox extends Component {
    private String name;
    private boolean checked = false;
    
    public CheckBox(String name) {
        this.name = name;
    }
    
    public void toggle() {
        checked = !checked;
        System.out.println("☑️ " + name + " 체크박스: " + (checked ? "체크됨" : "해제됨"));
        mediator.notify(this, "toggled");
    }
    
    public boolean isChecked() {
        return checked;
    }
    
    @Override
    public String getName() {
        return name;
    }
}

/**
 * ConcreteMediator: 로그인 다이얼로그
 */
public class LoginDialog implements DialogMediator {
    private TextField usernameField;
    private TextField passwordField;
    private CheckBox rememberMe;
    private Button loginButton;
    
    public LoginDialog() {
        usernameField = new TextField("Username");
        passwordField = new TextField("Password");
        rememberMe = new CheckBox("Remember Me");
        loginButton = new Button("Login");
        
        // 중재자 설정
        usernameField.setMediator(this);
        passwordField.setMediator(this);
        rememberMe.setMediator(this);
        loginButton.setMediator(this);
    }
    
    @Override
    public void notify(Component sender, String event) {
        System.out.println("   🔔 다이얼로그가 " + sender.getName() + "의 " + event + " 이벤트 수신");
        
        if (sender == loginButton && event.equals("click")) {
            handleLogin();
        } else if ((sender == usernameField || sender == passwordField) && 
                   event.equals("textChanged")) {
            validateForm();
        }
    }
    
    private void handleLogin() {
        String username = usernameField.getText();
        String password = passwordField.getText();
        boolean remember = rememberMe.isChecked();
        
        System.out.println("\n   🔐 로그인 처리:");
        System.out.println("      사용자: " + username);
        System.out.println("      비밀번호: " + password);
        System.out.println("      자동 로그인: " + remember);
    }
    
    private void validateForm() {
        boolean isValid = !usernameField.getText().isEmpty() && 
                         !passwordField.getText().isEmpty();
        System.out.println("   ✓ 폼 유효성: " + (isValid ? "유효" : "무효"));
    }
    
    public TextField getUsernameField() { return usernameField; }
    public TextField getPasswordField() { return passwordField; }
    public CheckBox getRememberMe() { return rememberMe; }
    public Button getLoginButton() { return loginButton; }
}

/**
 * 사용 예제
 */
public class DialogExample {
    public static void main(String[] args) {
        LoginDialog dialog = new LoginDialog();
        
        System.out.println("=== 로그인 다이얼로그 ===\n");
        
        dialog.getUsernameField().setText("john");
        System.out.println();
        
        dialog.getPasswordField().setText("pass123");
        System.out.println();
        
        dialog.getRememberMe().toggle();
        System.out.println();
        
        dialog.getLoginButton().click();
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **결합도 감소** | 객체 간 직접 참조 제거 | 채팅 사용자 |
| **재사용성** | 객체 독립적 사용 | UI 컴포넌트 |
| **중앙화** | 통신 로직 한 곳에 | 관제탑 |
| **SRP** | 통신 책임 분리 | 다이얼로그 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **God Object** | 중재자가 복잡해질 수 있음 | 책임 분리 |
| **성능** | 모든 통신이 중재자 거침 | 필요시에만 |

---

## 7. 안티패턴

### ❌ 안티패턴: 중재자가 너무 많이 알음

```java
// 잘못된 예
public class BadMediator {
    public void notify(Component sender) {
        // 모든 컴포넌트의 세부 사항을 알음
        if (sender instanceof Button) {
            Button btn = (Button) sender;
            // 버튼 내부 로직까지 처리
        }
    }
}
```

**해결:**
```java
// 이벤트 기반
public void notify(Component sender, String event) {
    // 이벤트만 받아서 처리
}
```

---

## 8. 핵심 정리

### 📌 Mediator 패턴 체크리스트

```
✅ Mediator 인터페이스
✅ ConcreteMediator 구현
✅ Colleague 정의
✅ notify() 메서드
✅ 객체 간 직접 참조 제거
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 다대다 관계 | ⭐⭐⭐ | 복잡도 감소 |
| UI 컴포넌트 | ⭐⭐⭐ | 통신 중앙화 |
| 채팅 시스템 | ⭐⭐⭐ | 메시지 중계 |
| 관제 시스템 | ⭐⭐⭐ | 조정 필요 |

### 💡 핵심 포인트

1. **통신 중앙화**
2. **직접 참조 제거**
3. **느슨한 결합**
4. **재사용성 향상**

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Chain of Responsibility](07-ChainOfResponsibility.md) | [다음: Memento →](09-Memento.md)**

</div>
