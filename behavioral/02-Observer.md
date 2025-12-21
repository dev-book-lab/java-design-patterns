# Observer Pattern (옵저버 패턴)

> **"객체의 상태 변화를 관찰자들에게 자동으로 알리자"**

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
// 문제 1: 수동으로 상태 확인 (Polling)
public class WeatherStation {
    private float temperature;
    
    public float getTemperature() {
        return temperature;
    }
}

public class Display {
    private WeatherStation station;
    
    public void update() {
        // 1초마다 확인
        while (true) {
            float temp = station.getTemperature();
            System.out.println("현재 온도: " + temp);
            Thread.sleep(1000);
        }
        // 비효율적! 변화 없어도 계속 확인
    }
}

// 문제 2: 강한 결합
public class StockMarket {
    private double stockPrice;
    
    public void setStockPrice(double price) {
        this.stockPrice = price;
        
        // 직접 호출 (강한 결합!)
        MobileApp mobileApp = new MobileApp();
        mobileApp.updatePrice(price);
        
        WebDashboard dashboard = new WebDashboard();
        dashboard.updatePrice(price);
        
        EmailNotifier notifier = new EmailNotifier();
        notifier.sendAlert(price);
        
        // 새 클라이언트 추가 시 여기 수정!
    }
}

// 문제 3: 일대다 관계 관리 어려움
public class YouTubeChannel {
    private List<Subscriber> subscribers = new ArrayList<>();
    
    public void uploadVideo(String title) {
        // 모든 구독자에게 일일이 알림
        for (Subscriber sub : subscribers) {
            sub.notify(title);
        }
        
        // 구독 취소는? 새 구독자는?
        // 관리가 복잡!
    }
}

// 문제 4: 이벤트 전파가 비효율적
public class Button {
    public void click() {
        // 클릭 이벤트 발생
        // 누가 관심 있는지 모름
        // 모든 객체에 물어봐야 함?
        
        if (listener1 != null) {
            listener1.onClick();
        }
        if (listener2 != null) {
            listener2.onClick();
        }
        // 리스너 추가마다 코드 수정!
    }
}
```

### ⚡ 핵심 문제

1. **강한 결합**: Subject가 Observer를 직접 알아야 함
2. **비효율적 Polling**: 변화 확인을 위해 계속 체크
3. **확장성 부족**: 새 Observer 추가 시 코드 수정
4. **알림 누락**: 수동 호출로 인한 실수 가능

---

## 2. 패턴 정의

### 📖 정의

> **객체 간의 일대다 의존 관계를 정의하여, 한 객체의 상태가 변하면 그 객체에 의존하는 모든 객체에게 자동으로 알림이 가고 갱신되게 하는 패턴**

### 🎯 목적

- **느슨한 결합**: Subject와 Observer 분리
- **자동 알림**: 상태 변경 시 자동 통지
- **일대다 관계**: 한 객체 → 여러 객체
- **동적 구독**: 런타임에 Observer 추가/제거

### 💡 핵심 아이디어

```java
// Before: 직접 호출 (강한 결합)
public void setState(int state) {
    this.state = state;
    display1.update(state);
    display2.update(state);
    display3.update(state);
}

// After: 자동 알림 (느슨한 결합)
public void setState(int state) {
    this.state = state;
    notifyObservers(); // 등록된 모든 Observer에게 자동 통지!
}
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌─────────────────┐
│    Subject      │  ← 관찰 대상
├─────────────────┤
│ - observers     │
│ + attach(obs)   │
│ + detach(obs)   │
│ + notify()      │
└─────────────────┘
         △
         │
┌─────────────────┐
│ConcreteSubject  │
├─────────────────┤
│ - state         │
│ + getState()    │
│ + setState()    │
└─────────────────┘
         │
         │ notifies
         ▼
┌─────────────────┐
│   Observer      │  ← 관찰자
├─────────────────┤
│ + update()      │
└─────────────────┘
         △
         │ implements
┌─────────────────┐
│ConcreteObserver │
├─────────────────┤
│ + update()      │
└─────────────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **Subject** | 관찰 대상 인터페이스 | `WeatherData` |
| **ConcreteSubject** | 구체적 관찰 대상 | `WeatherStation` |
| **Observer** | 관찰자 인터페이스 | `Display` |
| **ConcreteObserver** | 구체적 관찰자 | `CurrentDisplay` |

---

## 4. 구현 방법

### 기본 구현: 날씨 모니터링 시스템 ⭐⭐⭐

```java
/**
 * Observer: 관찰자 인터페이스
 */
public interface Observer {
    void update(float temperature, float humidity, float pressure);
}

/**
 * Subject: 관찰 대상 인터페이스
 */
public interface Subject {
    void registerObserver(Observer o);
    void removeObserver(Observer o);
    void notifyObservers();
}

/**
 * ConcreteSubject: 날씨 데이터
 */
public class WeatherData implements Subject {
    private List<Observer> observers;
    private float temperature;
    private float humidity;
    private float pressure;
    
    public WeatherData() {
        observers = new ArrayList<>();
    }
    
    @Override
    public void registerObserver(Observer o) {
        observers.add(o);
        System.out.println("✅ 관찰자 등록: " + o.getClass().getSimpleName());
    }
    
    @Override
    public void removeObserver(Observer o) {
        observers.remove(o);
        System.out.println("❌ 관찰자 제거: " + o.getClass().getSimpleName());
    }
    
    @Override
    public void notifyObservers() {
        System.out.println("\n📢 모든 관찰자에게 알림 전송...");
        for (Observer observer : observers) {
            observer.update(temperature, humidity, pressure);
        }
    }
    
    public void measurementsChanged() {
        notifyObservers();
    }
    
    public void setMeasurements(float temperature, float humidity, float pressure) {
        System.out.println("\n🌡️ 날씨 데이터 변경:");
        System.out.println("   온도: " + temperature + "°C");
        System.out.println("   습도: " + humidity + "%");
        System.out.println("   기압: " + pressure + "hPa");
        
        this.temperature = temperature;
        this.humidity = humidity;
        this.pressure = pressure;
        measurementsChanged();
    }
}

/**
 * ConcreteObserver 1: 현재 날씨 디스플레이
 */
public class CurrentConditionsDisplay implements Observer {
    private float temperature;
    private float humidity;
    
    @Override
    public void update(float temperature, float humidity, float pressure) {
        this.temperature = temperature;
        this.humidity = humidity;
        display();
    }
    
    public void display() {
        System.out.println("  📱 현재 날씨: " + temperature + "°C, 습도 " + humidity + "%");
    }
}

/**
 * ConcreteObserver 2: 통계 디스플레이
 */
public class StatisticsDisplay implements Observer {
    private float maxTemp = 0.0f;
    private float minTemp = 200.0f;
    private float tempSum = 0.0f;
    private int numReadings;
    
    @Override
    public void update(float temperature, float humidity, float pressure) {
        tempSum += temperature;
        numReadings++;
        
        if (temperature > maxTemp) {
            maxTemp = temperature;
        }
        
        if (temperature < minTemp) {
            minTemp = temperature;
        }
        
        display();
    }
    
    public void display() {
        System.out.println("  📊 통계: 평균 " + (tempSum / numReadings) + 
                "°C, 최고 " + maxTemp + "°C, 최저 " + minTemp + "°C");
    }
}

/**
 * ConcreteObserver 3: 예보 디스플레이
 */
public class ForecastDisplay implements Observer {
    private float currentPressure = 29.92f;
    private float lastPressure;
    
    @Override
    public void update(float temperature, float humidity, float pressure) {
        lastPressure = currentPressure;
        currentPressure = pressure;
        display();
    }
    
    public void display() {
        System.out.print("  🌤️ 예보: ");
        if (currentPressure > lastPressure) {
            System.out.println("날씨가 좋아지고 있습니다!");
        } else if (currentPressure == lastPressure) {
            System.out.println("현재와 비슷합니다");
        } else {
            System.out.println("악화될 것 같습니다");
        }
    }
}

/**
 * 사용 예제
 */
public class ObserverExample {
    public static void main(String[] args) {
        // Subject 생성
        WeatherData weatherData = new WeatherData();
        
        // Observer 생성 및 등록
        System.out.println("=== 관찰자 등록 ===");
        CurrentConditionsDisplay currentDisplay = new CurrentConditionsDisplay();
        StatisticsDisplay statisticsDisplay = new StatisticsDisplay();
        ForecastDisplay forecastDisplay = new ForecastDisplay();
        
        weatherData.registerObserver(currentDisplay);
        weatherData.registerObserver(statisticsDisplay);
        weatherData.registerObserver(forecastDisplay);
        
        // 날씨 데이터 업데이트
        System.out.println("\n" + "=".repeat(50));
        weatherData.setMeasurements(25, 65, 30.4f);
        
        System.out.println("\n" + "=".repeat(50));
        weatherData.setMeasurements(27, 70, 29.2f);
        
        // 관찰자 제거
        System.out.println("\n" + "=".repeat(50));
        System.out.println("\n=== 통계 디스플레이 제거 ===");
        weatherData.removeObserver(statisticsDisplay);
        
        System.out.println("\n" + "=".repeat(50));
        weatherData.setMeasurements(23, 90, 29.2f);
    }
}
```

**실행 결과:**
```
=== 관찰자 등록 ===
✅ 관찰자 등록: CurrentConditionsDisplay
✅ 관찰자 등록: StatisticsDisplay
✅ 관찰자 등록: ForecastDisplay

==================================================

🌡️ 날씨 데이터 변경:
   온도: 25.0°C
   습도: 65.0%
   기압: 30.4hPa

📢 모든 관찰자에게 알림 전송...
  📱 현재 날씨: 25.0°C, 습도 65.0%
  📊 통계: 평균 25.0°C, 최고 25.0°C, 최저 25.0°C
  🌤️ 예보: 날씨가 좋아지고 있습니다!

==================================================

🌡️ 날씨 데이터 변경:
   온도: 27.0°C
   습도: 70.0%
   기압: 29.2hPa

📢 모든 관찰자에게 알림 전송...
  📱 현재 날씨: 27.0°C, 습도 70.0%
  📊 통계: 평균 26.0°C, 최고 27.0°C, 최저 25.0°C
  🌤️ 예보: 악화될 것 같습니다

==================================================

=== 통계 디스플레이 제거 ===
❌ 관찰자 제거: StatisticsDisplay

==================================================

🌡️ 날씨 데이터 변경:
   온도: 23.0°C
   습도: 90.0%
   기압: 29.2hPa

📢 모든 관찰자에게 알림 전송...
  📱 현재 날씨: 23.0°C, 습도 90.0%
  🌤️ 예보: 현재와 비슷합니다
```

---

## 5. 실전 예제

### 예제 1: 주식 시장 모니터링 ⭐⭐⭐

```java
/**
 * Subject: 주식
 */
public class Stock implements Subject {
    private List<Observer> observers;
    private String symbol;
    private double price;
    
    public Stock(String symbol, double price) {
        this.observers = new ArrayList<>();
        this.symbol = symbol;
        this.price = price;
    }
    
    @Override
    public void registerObserver(Observer o) {
        observers.add(o);
    }
    
    @Override
    public void removeObserver(Observer o) {
        observers.remove(o);
    }
    
    @Override
    public void notifyObservers() {
        for (Observer observer : observers) {
            observer.update(symbol, price);
        }
    }
    
    public void setPrice(double price) {
        System.out.println("\n📈 " + symbol + " 주가 변동: $" + this.price + " → $" + price);
        this.price = price;
        notifyObservers();
    }
    
    public String getSymbol() {
        return symbol;
    }
    
    public double getPrice() {
        return price;
    }
}

/**
 * Observer 1: 모바일 앱
 */
public class MobileApp implements Observer {
    private String userName;
    
    public MobileApp(String userName) {
        this.userName = userName;
    }
    
    @Override
    public void update(String symbol, double price) {
        System.out.println("  📱 [모바일 앱 - " + userName + "] " + 
                symbol + " 주가: $" + price);
        sendPushNotification(symbol, price);
    }
    
    private void sendPushNotification(String symbol, double price) {
        System.out.println("     🔔 푸시 알림 전송!");
    }
}

/**
 * Observer 2: 이메일 알림
 */
public class EmailAlert implements Observer {
    private String email;
    
    public EmailAlert(String email) {
        this.email = email;
    }
    
    @Override
    public void update(String symbol, double price) {
        System.out.println("  📧 [이메일 - " + email + "] " + 
                symbol + " 주가 업데이트: $" + price);
        sendEmail(symbol, price);
    }
    
    private void sendEmail(String symbol, double price) {
        System.out.println("     ✉️ 이메일 발송!");
    }
}

/**
 * Observer 3: 웹 대시보드
 */
public class WebDashboard implements Observer {
    @Override
    public void update(String symbol, double price) {
        System.out.println("  💻 [웹 대시보드] " + symbol + " 차트 업데이트: $" + price);
        updateChart(symbol, price);
    }
    
    private void updateChart(String symbol, double price) {
        System.out.println("     📊 실시간 차트 갱신!");
    }
}

/**
 * 사용 예제
 */
public class StockMarketExample {
    public static void main(String[] args) {
        // 주식 생성
        Stock apple = new Stock("AAPL", 150.00);
        
        // 관찰자 등록
        System.out.println("=== 관찰자 등록 ===");
        apple.registerObserver(new MobileApp("John"));
        apple.registerObserver(new EmailAlert("john@email.com"));
        apple.registerObserver(new WebDashboard());
        
        // 주가 변동
        apple.setPrice(152.50);
        apple.setPrice(155.00);
        apple.setPrice(148.75);
    }
}
```

---

### 예제 2: 이벤트 시스템 ⭐⭐⭐

```java
/**
 * Observer: 이벤트 리스너
 */
public interface EventListener {
    void onEvent(String eventType, String data);
}

/**
 * Subject: 이벤트 관리자
 */
public class EventManager {
    private Map<String, List<EventListener>> listeners;
    
    public EventManager() {
        listeners = new HashMap<>();
    }
    
    public void subscribe(String eventType, EventListener listener) {
        listeners.computeIfAbsent(eventType, k -> new ArrayList<>())
                .add(listener);
        System.out.println("✅ 구독: " + eventType);
    }
    
    public void unsubscribe(String eventType, EventListener listener) {
        List<EventListener> users = listeners.get(eventType);
        if (users != null) {
            users.remove(listener);
            System.out.println("❌ 구독 취소: " + eventType);
        }
    }
    
    public void notify(String eventType, String data) {
        List<EventListener> users = listeners.get(eventType);
        if (users != null) {
            System.out.println("\n📢 이벤트 발생: " + eventType);
            for (EventListener listener : users) {
                listener.onEvent(eventType, data);
            }
        }
    }
}

/**
 * ConcreteSubject: 파일 에디터
 */
public class Editor {
    private EventManager events;
    private String fileName;
    
    public Editor() {
        this.events = new EventManager();
    }
    
    public EventManager getEvents() {
        return events;
    }
    
    public void openFile(String path) {
        this.fileName = path;
        events.notify("open", path);
    }
    
    public void saveFile() {
        events.notify("save", fileName);
    }
}

/**
 * ConcreteObserver: 로거
 */
public class LoggingListener implements EventListener {
    private String logFile;
    
    public LoggingListener(String logFile) {
        this.logFile = logFile;
    }
    
    @Override
    public void onEvent(String eventType, String data) {
        System.out.println("  📝 로그 기록: " + eventType + " - " + data);
        System.out.println("     파일: " + logFile);
    }
}

/**
 * ConcreteObserver: 이메일 알림
 */
public class EmailNotificationListener implements EventListener {
    private String email;
    
    public EmailNotificationListener(String email) {
        this.email = email;
    }
    
    @Override
    public void onEvent(String eventType, String data) {
        System.out.println("  📧 이메일 발송: " + email);
        System.out.println("     내용: 파일 " + eventType + " - " + data);
    }
}

/**
 * 사용 예제
 */
public class EventSystemExample {
    public static void main(String[] args) {
        Editor editor = new Editor();
        
        // 리스너 등록
        System.out.println("=== 리스너 등록 ===");
        LoggingListener logger = new LoggingListener("/var/log/editor.log");
        EmailNotificationListener emailNotifier = 
                new EmailNotificationListener("admin@example.com");
        
        editor.getEvents().subscribe("open", logger);
        editor.getEvents().subscribe("save", logger);
        editor.getEvents().subscribe("save", emailNotifier);
        
        // 이벤트 발생
        System.out.println("\n" + "=".repeat(50));
        editor.openFile("/home/user/document.txt");
        
        System.out.println("\n" + "=".repeat(50));
        editor.saveFile();
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **느슨한 결합** | Subject-Observer 독립적 | 날씨 시스템 |
| **동적 관계** | 런타임에 Observer 추가/제거 | 구독/구독 취소 |
| **OCP 준수** | 새 Observer 추가 시 기존 코드 불변 | 새 디스플레이 |
| **브로드캐스트** | 한 번에 여러 객체 통지 | 이벤트 시스템 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **순서 보장 안 됨** | 알림 순서 랜덤 | 우선순위 큐 사용 |
| **메모리 누수** | Observer 제거 안 하면 | WeakReference 사용 |
| **복잡도 증가** | 이벤트 체인 추적 어려움 | 로깅 강화 |

---

## 7. 안티패턴

### ❌ 안티패턴: Observer 제거 안 함

```java
// 잘못된 예: 메모리 누수
Subject subject = new Subject();
Observer observer = new Observer();
subject.attach(observer);
// observer 제거 안 함 → 메모리 누수!
```

**해결:**
```java
subject.attach(observer);
// 사용 완료 후
subject.detach(observer);
```

---

## 8. 핵심 정리

### 📌 Observer 패턴 체크리스트

```
✅ Subject 인터페이스 정의
✅ Observer 인터페이스 정의
✅ ConcreteSubject 구현
✅ ConcreteObserver 구현
✅ attach/detach 메서드
✅ notify 메커니즘
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 일대다 관계 | ⭐⭐⭐ | 자동 알림 |
| 이벤트 시스템 | ⭐⭐⭐ | 느슨한 결합 |
| 상태 동기화 | ⭐⭐⭐ | 자동 갱신 |
| 데이터 바인딩 | ⭐⭐⭐ | MVC/MVVM |

### 💡 핵심 포인트

1. **일대다 의존성**
2. **자동 알림**
3. **느슨한 결합**
4. **Publish-Subscribe**

### 🔥 실무 활용

```java
// Java 내장 Observable (deprecated)
// 대신 PropertyChangeListener 사용

// 또는 라이브러리
// RxJava, EventBus, Spring Events
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Strategy](01-Strategy.md) | [다음: Template Method →](03-TemplateMethod.md)**

</div>
