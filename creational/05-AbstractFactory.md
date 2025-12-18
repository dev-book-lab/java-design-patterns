# Abstract Factory Pattern (추상 팩토리 패턴)

> **"관련된 객체들의 집합을 생성하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [장단점](#6-장단점)
7. [Factory Method vs Abstract Factory](#7-factory-method-vs-abstract-factory)
8. [핵심 정리](#8-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 플랫폼별 UI 컴포넌트
public class Application {
    public void createUI(String platform) {
        if (platform.equals("Windows")) {
            Button button = new WindowsButton();
            Checkbox checkbox = new WindowsCheckbox();
            TextField textField = new WindowsTextField();
            // Windows 스타일로 통일
        } else if (platform.equals("Mac")) {
            Button button = new MacButton();
            Checkbox checkbox = new MacCheckbox();
            TextField textField = new MacTextField();
            // Mac 스타일로 통일
        }
        // 플랫폼마다 반복! 새 플랫폼 추가 시 모두 수정!
    }
}

// 문제 2: 관련 없는 조합이 생길 수 있음
public class UICreator {
    public void createMixedUI() {
        Button button = new WindowsButton();
        Checkbox checkbox = new MacCheckbox();  // ⚠️ 섞임!
        TextField textField = new LinuxTextField();  // ⚠️ 더 섞임!
        // UI가 일관성 없음!
    }
}

// 문제 3: 제품군 확장이 어려움
public class FurnitureStore {
    public void createFurniture(String style, String type) {
        if (style.equals("Modern")) {
            if (type.equals("Chair")) {
                return new ModernChair();
            } else if (type.equals("Sofa")) {
                return new ModernSofa();
            } else if (type.equals("Table")) {
                return new ModernTable();
            }
        } else if (style.equals("Victorian")) {
            if (type.equals("Chair")) {
                return new VictorianChair();
            } else if (type.equals("Sofa")) {
                return new VictorianSofa();
            } else if (type.equals("Table")) {
                return new VictorianTable();
            }
        }
        // 새 스타일 추가 시 모든 타입마다 코드 추가!
    }
}

// 문제 4: 테마별 UI 일관성
public class Website {
    private String theme; // "Light", "Dark"
    
    public void render() {
        // 수동으로 테마에 맞는 컴포넌트 생성
        if (theme.equals("Light")) {
            Header header = new LightHeader();
            Footer footer = new LightFooter();
            Sidebar sidebar = new LightSidebar();
        } else {
            Header header = new DarkHeader();
            Footer footer = new DarkFooter();
            Sidebar sidebar = new DarkSidebar();
        }
        // 실수로 Light와 Dark를 섞을 수 있음!
    }
}
```

### ⚡ 핵심 문제

1. **제품군 일관성**: 관련 객체들이 일관성 있게 생성되어야 함
2. **플랫폼 독립성**: 구체 클래스에 의존하지 않아야 함
3. **확장의 어려움**: 새로운 제품군 추가 시 모든 코드 수정
4. **잘못된 조합**: 서로 맞지 않는 객체 조합 가능

---

## 2. 패턴 정의

### 📖 정의

> **관련성 있는 객체들의 집합(제품군)을 생성하기 위한 인터페이스를 제공하되, 구체적인 클래스는 명시하지 않는 패턴**

### 🎯 목적

- **제품군 생성**: 관련 객체들을 함께 생성
- **일관성 보장**: 같은 제품군의 객체만 조합
- **구체 클래스 숨김**: 인터페이스에만 의존
- **쉬운 교체**: 제품군 전체를 쉽게 교체

### 💡 핵심 아이디어

```java
// Before: 직접 생성 (섞일 수 있음)
Button button = new WindowsButton();
Checkbox checkbox = new MacCheckbox(); // 섞임!

// After: 팩토리로 일관성 보장
GUIFactory factory = new WindowsFactory();
Button button = factory.createButton();
Checkbox checkbox = factory.createCheckbox(); // Windows로 통일!
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
                ┌─────────────────┐
                │ AbstractFactory │ ← 추상 팩토리
                ├─────────────────┤
                │ +createProductA()│
                │ +createProductB()│
                └─────────────────┘
                        △
           ┌────────────┴────────────┐
           │                         │
    ┌──────────────┐         ┌──────────────┐
    │ConcreteFactory1│       │ConcreteFactory2│ ← 구체 팩토리
    ├──────────────┤         ├──────────────┤
    │createProductA()│       │createProductA()│
    │createProductB()│       │createProductB()│
    └──────────────┘         └──────────────┘
           │                         │
           │ creates                 │ creates
           ▼                         ▼
    ┌──────────────┐         ┌──────────────┐
    │  ProductA1   │         │  ProductA2   │ ← 제품 A
    └──────────────┘         └──────────────┘
           │                         │
           ▼                         ▼
    ┌──────────────┐         ┌──────────────┐
    │  ProductB1   │         │  ProductB2   │ ← 제품 B
    └──────────────┘         └──────────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **AbstractFactory** | 제품 생성 메서드 선언 | `GUIFactory` |
| **ConcreteFactory** | 구체적인 제품 생성 | `WindowsFactory`, `MacFactory` |
| **AbstractProduct** | 제품 인터페이스 | `Button`, `Checkbox` |
| **ConcreteProduct** | 구체적인 제품 | `WindowsButton`, `MacButton` |
| **Client** | 추상 팩토리와 제품 사용 | `Application` |

---

## 4. 구현 방법

### 기본 구현: GUI 컴포넌트 ⭐⭐⭐

#### Step 1: AbstractProduct 정의

```java
/**
 * 제품 A: Button
 */
public interface Button {
    void render();
    void onClick();
}

/**
 * 제품 B: Checkbox
 */
public interface Checkbox {
    void render();
    void check();
}

/**
 * 제품 C: TextField
 */
public interface TextField {
    void render();
    void setText(String text);
}
```

#### Step 2: ConcreteProduct 구현

```java
// Windows 제품군
public class WindowsButton implements Button {
    @Override
    public void render() {
        System.out.println("🪟 Windows 스타일 버튼 렌더링");
    }
    
    @Override
    public void onClick() {
        System.out.println("Windows 버튼 클릭!");
    }
}

public class WindowsCheckbox implements Checkbox {
    @Override
    public void render() {
        System.out.println("🪟 Windows 스타일 체크박스 렌더링");
    }
    
    @Override
    public void check() {
        System.out.println("Windows 체크박스 체크!");
    }
}

public class WindowsTextField implements TextField {
    @Override
    public void render() {
        System.out.println("🪟 Windows 스타일 텍스트필드 렌더링");
    }
    
    @Override
    public void setText(String text) {
        System.out.println("Windows 텍스트필드: " + text);
    }
}

// Mac 제품군
public class MacButton implements Button {
    @Override
    public void render() {
        System.out.println("🍎 Mac 스타일 버튼 렌더링");
    }
    
    @Override
    public void onClick() {
        System.out.println("Mac 버튼 클릭!");
    }
}

public class MacCheckbox implements Checkbox {
    @Override
    public void render() {
        System.out.println("🍎 Mac 스타일 체크박스 렌더링");
    }
    
    @Override
    public void check() {
        System.out.println("Mac 체크박스 체크!");
    }
}

public class MacTextField implements TextField {
    @Override
    public void render() {
        System.out.println("🍎 Mac 스타일 텍스트필드 렌더링");
    }
    
    @Override
    public void setText(String text) {
        System.out.println("Mac 텍스트필드: " + text);
    }
}

// Linux 제품군
public class LinuxButton implements Button {
    @Override
    public void render() {
        System.out.println("🐧 Linux 스타일 버튼 렌더링");
    }
    
    @Override
    public void onClick() {
        System.out.println("Linux 버튼 클릭!");
    }
}

public class LinuxCheckbox implements Checkbox {
    @Override
    public void render() {
        System.out.println("🐧 Linux 스타일 체크박스 렌더링");
    }
    
    @Override
    public void check() {
        System.out.println("Linux 체크박스 체크!");
    }
}

public class LinuxTextField implements TextField {
    @Override
    public void render() {
        System.out.println("🐧 Linux 스타일 텍스트필드 렌더링");
    }
    
    @Override
    public void setText(String text) {
        System.out.println("Linux 텍스트필드: " + text);
    }
}
```

#### Step 3: AbstractFactory 정의

```java
/**
 * 추상 팩토리: GUI 컴포넌트 생성
 */
public interface GUIFactory {
    Button createButton();
    Checkbox createCheckbox();
    TextField createTextField();
}
```

#### Step 4: ConcreteFactory 구현

```java
/**
 * Windows 팩토리
 */
public class WindowsFactory implements GUIFactory {
    @Override
    public Button createButton() {
        return new WindowsButton();
    }
    
    @Override
    public Checkbox createCheckbox() {
        return new WindowsCheckbox();
    }
    
    @Override
    public TextField createTextField() {
        return new WindowsTextField();
    }
}

/**
 * Mac 팩토리
 */
public class MacFactory implements GUIFactory {
    @Override
    public Button createButton() {
        return new MacButton();
    }
    
    @Override
    public Checkbox createCheckbox() {
        return new MacCheckbox();
    }
    
    @Override
    public TextField createTextField() {
        return new MacTextField();
    }
}

/**
 * Linux 팩토리
 */
public class LinuxFactory implements GUIFactory {
    @Override
    public Button createButton() {
        return new LinuxButton();
    }
    
    @Override
    public Checkbox createCheckbox() {
        return new LinuxCheckbox();
    }
    
    @Override
    public TextField createTextField() {
        return new LinuxTextField();
    }
}
```

#### Step 5: Client 코드

```java
/**
 * 애플리케이션 (클라이언트)
 */
public class Application {
    private Button button;
    private Checkbox checkbox;
    private TextField textField;
    
    // 팩토리를 주입받음
    public Application(GUIFactory factory) {
        this.button = factory.createButton();
        this.checkbox = factory.createCheckbox();
        this.textField = factory.createTextField();
    }
    
    public void render() {
        System.out.println("=== UI 렌더링 ===");
        button.render();
        checkbox.render();
        textField.render();
    }
    
    public void interact() {
        System.out.println("\n=== 사용자 인터랙션 ===");
        button.onClick();
        checkbox.check();
        textField.setText("Hello World");
    }
}

/**
 * 사용 예제
 */
public class AbstractFactoryExample {
    public static void main(String[] args) {
        // 1. Windows 애플리케이션
        System.out.println("### Windows 애플리케이션 ###");
        GUIFactory windowsFactory = new WindowsFactory();
        Application windowsApp = new Application(windowsFactory);
        windowsApp.render();
        windowsApp.interact();
        
        // 2. Mac 애플리케이션
        System.out.println("\n\n### Mac 애플리케이션 ###");
        GUIFactory macFactory = new MacFactory();
        Application macApp = new Application(macFactory);
        macApp.render();
        macApp.interact();
        
        // 3. Linux 애플리케이션
        System.out.println("\n\n### Linux 애플리케이션 ###");
        GUIFactory linuxFactory = new LinuxFactory();
        Application linuxApp = new Application(linuxFactory);
        linuxApp.render();
        linuxApp.interact();
        
        // 4. 런타임에 플랫폼 선택
        System.out.println("\n\n### 런타임 선택 ###");
        String os = System.getProperty("os.name").toLowerCase();
        GUIFactory factory = getFactory(os);
        Application app = new Application(factory);
        app.render();
    }
    
    private static GUIFactory getFactory(String os) {
        if (os.contains("win")) {
            return new WindowsFactory();
        } else if (os.contains("mac")) {
            return new MacFactory();
        } else {
            return new LinuxFactory();
        }
    }
}
```

**실행 결과:**
```
### Windows 애플리케이션 ###
=== UI 렌더링 ===
🪟 Windows 스타일 버튼 렌더링
🪟 Windows 스타일 체크박스 렌더링
🪟 Windows 스타일 텍스트필드 렌더링

=== 사용자 인터랙션 ===
Windows 버튼 클릭!
Windows 체크박스 체크!
Windows 텍스트필드: Hello World


### Mac 애플리케이션 ###
=== UI 렌더링 ===
🍎 Mac 스타일 버튼 렌더링
🍎 Mac 스타일 체크박스 렌더링
🍎 Mac 스타일 텍스트필드 렌더링

=== 사용자 인터랙션 ===
Mac 버튼 클릭!
Mac 체크박스 체크!
Mac 텍스트필드: Hello World

...
```

---

## 5. 실전 예제

### 예제 1: 가구 제품군 ⭐⭐⭐

```java
// AbstractProduct
public interface Chair {
    void sitOn();
    boolean hasLegs();
}

public interface Sofa {
    void lieOn();
    int getSeats();
}

public interface CoffeeTable {
    void placeItem(String item);
}

// ConcreteProduct - Modern 스타일
public class ModernChair implements Chair {
    @Override
    public void sitOn() {
        System.out.println("🪑 모던 의자에 앉습니다 (심플하고 세련됨)");
    }
    
    @Override
    public boolean hasLegs() {
        return true;
    }
}

public class ModernSofa implements Sofa {
    @Override
    public void lieOn() {
        System.out.println("🛋️ 모던 소파에 눕습니다 (미니멀 디자인)");
    }
    
    @Override
    public int getSeats() {
        return 3;
    }
}

public class ModernCoffeeTable implements CoffeeTable {
    @Override
    public void placeItem(String item) {
        System.out.println("☕ 모던 커피 테이블에 " + item + " 올림 (유리 상판)");
    }
}

// ConcreteProduct - Victorian 스타일
public class VictorianChair implements Chair {
    @Override
    public void sitOn() {
        System.out.println("🪑 빅토리안 의자에 앉습니다 (화려한 장식)");
    }
    
    @Override
    public boolean hasLegs() {
        return true;
    }
}

public class VictorianSofa implements Sofa {
    @Override
    public void lieOn() {
        System.out.println("🛋️ 빅토리안 소파에 눕습니다 (벨벳 소재)");
    }
    
    @Override
    public int getSeats() {
        return 2;
    }
}

public class VictorianCoffeeTable implements CoffeeTable {
    @Override
    public void placeItem(String item) {
        System.out.println("☕ 빅토리안 커피 테이블에 " + item + " 올림 (나무 조각)");
    }
}

// AbstractFactory
public interface FurnitureFactory {
    Chair createChair();
    Sofa createSofa();
    CoffeeTable createCoffeeTable();
}

// ConcreteFactory
public class ModernFurnitureFactory implements FurnitureFactory {
    @Override
    public Chair createChair() {
        return new ModernChair();
    }
    
    @Override
    public Sofa createSofa() {
        return new ModernSofa();
    }
    
    @Override
    public CoffeeTable createCoffeeTable() {
        return new ModernCoffeeTable();
    }
}

public class VictorianFurnitureFactory implements FurnitureFactory {
    @Override
    public Chair createChair() {
        return new VictorianChair();
    }
    
    @Override
    public Sofa createSofa() {
        return new VictorianSofa();
    }
    
    @Override
    public CoffeeTable createCoffeeTable() {
        return new VictorianCoffeeTable();
    }
}

// Client
public class FurnitureShop {
    private Chair chair;
    private Sofa sofa;
    private CoffeeTable table;
    
    public FurnitureShop(FurnitureFactory factory) {
        this.chair = factory.createChair();
        this.sofa = factory.createSofa();
        this.table = factory.createCoffeeTable();
    }
    
    public void showroom() {
        System.out.println("=== 가구 쇼룸 ===");
        chair.sitOn();
        sofa.lieOn();
        table.placeItem("커피");
        System.out.println("소파 좌석 수: " + sofa.getSeats());
    }
}

// 사용 예제
public class FurnitureExample {
    public static void main(String[] args) {
        // Modern 스타일 매장
        System.out.println("### 모던 가구 매장 ###");
        FurnitureFactory modernFactory = new ModernFurnitureFactory();
        FurnitureShop modernShop = new FurnitureShop(modernFactory);
        modernShop.showroom();
        
        // Victorian 스타일 매장
        System.out.println("\n### 빅토리안 가구 매장 ###");
        FurnitureFactory victorianFactory = new VictorianFurnitureFactory();
        FurnitureShop victorianShop = new FurnitureShop(victorianFactory);
        victorianShop.showroom();
    }
}
```

---

### 예제 2: 데이터베이스 연결 ⭐⭐⭐

```java
// AbstractProduct
public interface Connection {
    void connect();
    void disconnect();
    String getConnectionString();
}

public interface Command {
    void execute(String sql);
}

public interface DataReader {
    List<String> readData(String query);
}

// ConcreteProduct - MySQL
public class MySQLConnection implements Connection {
    @Override
    public void connect() {
        System.out.println("🐬 MySQL 데이터베이스 연결");
    }
    
    @Override
    public void disconnect() {
        System.out.println("🐬 MySQL 연결 종료");
    }
    
    @Override
    public String getConnectionString() {
        return "jdbc:mysql://localhost:3306/mydb";
    }
}

public class MySQLCommand implements Command {
    @Override
    public void execute(String sql) {
        System.out.println("🐬 MySQL 명령 실행: " + sql);
    }
}

public class MySQLDataReader implements DataReader {
    @Override
    public List<String> readData(String query) {
        System.out.println("🐬 MySQL 데이터 읽기: " + query);
        return Arrays.asList("MySQL Row 1", "MySQL Row 2");
    }
}

// ConcreteProduct - PostgreSQL
public class PostgreSQLConnection implements Connection {
    @Override
    public void connect() {
        System.out.println("🐘 PostgreSQL 데이터베이스 연결");
    }
    
    @Override
    public void disconnect() {
        System.out.println("🐘 PostgreSQL 연결 종료");
    }
    
    @Override
    public String getConnectionString() {
        return "jdbc:postgresql://localhost:5432/mydb";
    }
}

public class PostgreSQLCommand implements Command {
    @Override
    public void execute(String sql) {
        System.out.println("🐘 PostgreSQL 명령 실행: " + sql);
    }
}

public class PostgreSQLDataReader implements DataReader {
    @Override
    public List<String> readData(String query) {
        System.out.println("🐘 PostgreSQL 데이터 읽기: " + query);
        return Arrays.asList("PostgreSQL Row 1", "PostgreSQL Row 2");
    }
}

// AbstractFactory
public interface DatabaseFactory {
    Connection createConnection();
    Command createCommand();
    DataReader createDataReader();
}

// ConcreteFactory
public class MySQLFactory implements DatabaseFactory {
    @Override
    public Connection createConnection() {
        return new MySQLConnection();
    }
    
    @Override
    public Command createCommand() {
        return new MySQLCommand();
    }
    
    @Override
    public DataReader createDataReader() {
        return new MySQLDataReader();
    }
}

public class PostgreSQLFactory implements DatabaseFactory {
    @Override
    public Connection createConnection() {
        return new PostgreSQLConnection();
    }
    
    @Override
    public Command createCommand() {
        return new PostgreSQLCommand();
    }
    
    @Override
    public DataReader createDataReader() {
        return new PostgreSQLDataReader();
    }
}

// Client
public class DatabaseClient {
    private Connection connection;
    private Command command;
    private DataReader reader;
    
    public DatabaseClient(DatabaseFactory factory) {
        this.connection = factory.createConnection();
        this.command = factory.createCommand();
        this.reader = factory.createDataReader();
    }
    
    public void performOperations() {
        System.out.println("\n=== DB 작업 시작 ===");
        System.out.println("연결 문자열: " + connection.getConnectionString());
        
        connection.connect();
        command.execute("INSERT INTO users VALUES (1, 'John')");
        List<String> data = reader.readData("SELECT * FROM users");
        System.out.println("읽은 데이터: " + data);
        connection.disconnect();
    }
}

// 사용 예제
public class DatabaseExample {
    public static void main(String[] args) {
        // MySQL 사용
        System.out.println("### MySQL 데이터베이스 ###");
        DatabaseFactory mysqlFactory = new MySQLFactory();
        DatabaseClient mysqlClient = new DatabaseClient(mysqlFactory);
        mysqlClient.performOperations();
        
        // PostgreSQL로 쉽게 교체
        System.out.println("\n\n### PostgreSQL 데이터베이스 ###");
        DatabaseFactory postgresFactory = new PostgreSQLFactory();
        DatabaseClient postgresClient = new DatabaseClient(postgresFactory);
        postgresClient.performOperations();
        
        // 설정 파일로 동적 선택
        System.out.println("\n\n### 설정 기반 선택 ###");
        String dbType = "PostgreSQL"; // 설정에서 읽음
        DatabaseFactory factory = getDatabaseFactory(dbType);
        DatabaseClient client = new DatabaseClient(factory);
        client.performOperations();
    }
    
    private static DatabaseFactory getDatabaseFactory(String dbType) {
        switch (dbType) {
            case "MySQL":
                return new MySQLFactory();
            case "PostgreSQL":
                return new PostgreSQLFactory();
            default:
                throw new IllegalArgumentException("Unknown DB: " + dbType);
        }
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **제품군 일관성** | 관련 객체들이 함께 사용됨 보장 | Windows 컴포넌트끼리만 |
| **구체 클래스 분리** | 클라이언트는 인터페이스만 사용 | GUIFactory만 알면 됨 |
| **쉬운 교체** | 팩토리만 바꾸면 전체 교체 | Windows ↔ Mac |
| **OCP 준수** | 새 제품군 추가 시 기존 코드 불변 | Linux 추가 용이 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **새 제품 추가 어려움** | 모든 팩토리 수정 필요 | 신중한 설계 |
| **복잡도 증가** | 많은 클래스와 인터페이스 | 필요시에만 사용 |

---

## 7. Factory Method vs Abstract Factory

### 🔍 비교표

| 특징 | Factory Method | Abstract Factory |
|------|---------------|------------------|
| **목적** | 단일 객체 생성 | 관련 객체군 생성 |
| **복잡도** | 낮음 | 높음 |
| **제품 수** | 1개 | 여러 개 (제품군) |
| **구조** | 상속 기반 | 조합 기반 |
| **예시** | `createTransport()` | `createButton(), createCheckbox()` |

### 💡 선택 가이드

```java
// Factory Method: 하나의 제품만
Transport transport = logistics.createTransport();

// Abstract Factory: 여러 제품을 함께
Button button = factory.createButton();
Checkbox checkbox = factory.createCheckbox();
TextField field = factory.createTextField();
```

---

## 8. 핵심 정리

### 📌 Abstract Factory 패턴 체크리스트

```
✅ AbstractProduct 인터페이스 정의
✅ ConcreteProduct 구현
✅ AbstractFactory 인터페이스 정의
✅ ConcreteFactory 구현
✅ 클라이언트는 추상 타입만 사용
✅ 제품군 일관성 보장
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 관련 제품군이 있음 | ⭐⭐⭐ | 일관성 보장 |
| 플랫폼 독립적 설계 | ⭐⭐⭐ | 구체 클래스 숨김 |
| 제품군 교체 필요 | ⭐⭐⭐ | 쉬운 교체 |
| Look & Feel 통일 | ⭐⭐⭐ | UI 일관성 |

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Prototype](./04-Prototype.md) | [다음: Object Pool →](./06-ObjectPool.md)**

</div>
