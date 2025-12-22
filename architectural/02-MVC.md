# MVC Pattern (Model-View-Controller 패턴)

> **"UI, 비즈니스 로직, 데이터를 분리하여 독립적으로 개발하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [MVC 변형들](#6-mvc-변형들)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: UI와 로직이 뒤섞임
public class UserManagementScreen extends JFrame {
    private JTextField nameField;
    private JTextField emailField;
    private JButton saveButton;
    
    public UserManagementScreen() {
        // UI 구성
        nameField = new JTextField(20);
        emailField = new JTextField(20);
        saveButton = new JButton("저장");
        
        // 이벤트 핸들러에 모든 로직!
        saveButton.addActionListener(e -> {
            // 1. 입력 검증
            String name = nameField.getText();
            if (name.isEmpty()) {
                JOptionPane.showMessageDialog(this, "이름을 입력하세요");
                return;
            }
            
            // 2. 비즈니스 로직
            String email = emailField.getText();
            if (!email.contains("@")) {
                JOptionPane.showMessageDialog(this, "잘못된 이메일");
                return;
            }
            
            // 3. 데이터베이스 저장
            try {
                Connection conn = DriverManager.getConnection(DB_URL);
                PreparedStatement stmt = conn.prepareStatement(
                    "INSERT INTO users (name, email) VALUES (?, ?)"
                );
                stmt.setString(1, name);
                stmt.setString(2, email);
                stmt.executeUpdate();
                
                // 4. UI 업데이트
                JOptionPane.showMessageDialog(this, "저장 완료");
                nameField.setText("");
                emailField.setText("");
                
            } catch (SQLException ex) {
                JOptionPane.showMessageDialog(this, "저장 실패: " + ex.getMessage());
            }
        });
        
        // UI + 검증 + 비즈니스 로직 + DB + UI 업데이트
        // 모두 한 곳에! 😱
    }
}

// 문제 2: 중복 코드 폭발
public class WebUserController {
    public void saveUser(HttpServletRequest request, HttpServletResponse response) {
        String name = request.getParameter("name");
        String email = request.getParameter("email");
        
        // 똑같은 검증 로직 반복
        if (name.isEmpty()) { /* ... */ }
        if (!email.contains("@")) { /* ... */ }
        
        // 똑같은 저장 로직 반복
        Connection conn = DriverManager.getConnection(DB_URL);
        // ...
    }
}

public class MobileUserController {
    public void saveUser(UserRequest request) {
        // 또 똑같은 로직 반복! 😱
        // Web과 Mobile에서 같은 코드 복붙
    }
}

// 문제 3: 테스트 불가능
public class ProductListScreen extends JFrame {
    private JTable productTable;
    
    public ProductListScreen() {
        // UI 초기화
        productTable = new JTable();
        
        // 데이터 로드
        loadProducts();
    }
    
    private void loadProducts() {
        try {
            Connection conn = DriverManager.getConnection(DB_URL);
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("SELECT * FROM products");
            
            DefaultTableModel model = new DefaultTableModel();
            // ResultSet → TableModel 변환
            
            productTable.setModel(model);
            
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
    
    // 어떻게 테스트?
    // DB 없이 테스트 불가능!
    // UI 없이 비즈니스 로직만 테스트 불가능!
}

// 문제 4: 변경 취약성
public class OrderForm extends JFrame {
    private JLabel totalLabel;
    private List<OrderItem> items = new ArrayList<>();
    
    public void addItem(Product product) {
        items.add(new OrderItem(product));
        
        // 총액 계산 (비즈니스 로직)
        double total = 0;
        for (OrderItem item : items) {
            total += item.getPrice();
        }
        
        // UI 업데이트
        totalLabel.setText("총액: " + total);
    }
    
    // 총액 계산 로직이 UI에 종속!
    // 다른 곳에서 재사용 불가!
    // 계산 로직 변경 시 UI 코드 수정!
}
```

### ⚡ 핵심 문제

1. **관심사 혼재**: UI, 비즈니스 로직, 데이터 접근이 섞임
2. **중복 코드**: 같은 로직이 여러 곳에 반복
3. **테스트 어려움**: UI와 로직이 분리 안 됨
4. **변경 취약**: 한 부분 수정이 전체에 영향
5. **재사용 불가**: 로직이 UI에 종속

---

## 2. 패턴 정의

### 📖 정의

> **애플리케이션을 Model(데이터/비즈니스 로직), View(UI), Controller(입력 처리)로 분리하여, 각 컴포넌트가 독립적으로 변경될 수 있게 하는 아키텍처 패턴**

### 🎯 목적

- **관심사 분리**: 데이터, UI, 제어 로직 분리
- **독립적 개발**: 각 컴포넌트 독립 개발
- **재사용성**: 같은 Model을 여러 View에서 사용
- **테스트 용이**: Model과 Controller 독립 테스트

### 💡 핵심 아이디어

```java
// Before: 모든 게 섞임
public class Screen {
    void onButtonClick() {
        String data = textField.getText();
        validate(data);              // 검증
        saveToDatabase(data);        // 저장
        label.setText("완료");        // UI 업데이트
    }
}

// After: MVC로 분리
// Model: 데이터 + 비즈니스 로직
public class User {
    private String name;
    void validate() { /* ... */ }
    void save() { /* ... */ }
}

// View: UI만
public class UserView {
    void display(User user) { /* ... */ }
    void showMessage(String msg) { /* ... */ }
}

// Controller: 입력 처리 + 조정
public class UserController {
    void handleSave(String name) {
        user.setName(name);
        user.validate();
        user.save();
        view.display(user);
    }
}
```

---

## 3. 구조와 구성요소

### 📊 전통적 MVC (Passive MVC)

```
     ┌──────────┐
     │   User   │
     └──────────┘
          │
          │ 입력
          ▼
    ┌──────────┐
    │Controller│ ◄───────────────┐
    └──────────┘                 │
          │                      │
          │ 업데이트               │ 통지
          ▼                      │
     ┌─────────┐                 │
     │  Model  │─────────────────┘
     └─────────┘
          │
          │ 데이터 요청
          ▼
     ┌─────────┐
     │  View   │
     └─────────┘
          │
          │ 표시
          ▼
     ┌──────────┐
     │   User   │
     └──────────┘
```

### 🔄 Observer Pattern을 이용한 MVC

```
     ┌──────────┐
     │   User   │
     └──────────┘
          │
          │ 입력
          ▼
    ┌──────────┐
    │Controller│
    └──────────┘
          │
          │ 업데이트
          ▼
     ┌─────────┐
     │  Model  │ (Observable)
     └─────────┘
          │
          │ notify()
          ▼
     ┌─────────┐
     │  View   │ (Observer)
     │  View2  │ (Observer)
     │  View3  │ (Observer)
     └─────────┘
```

### 🌐 Web MVC (Spring MVC)

```
┌────────────────────────────────────┐
│          Browser (Client)          │
└────────────────────────────────────┘
              │
              │ HTTP Request
              ▼
┌────────────────────────────────────┐
│      DispatcherServlet             │
│      (Front Controller)            │
└────────────────────────────────────┘
              │
              ▼
┌────────────────────────────────────┐
│         @Controller                │  ← Controller
│      - @RequestMapping             │
│      - @GetMapping                 │
└────────────────────────────────────┘
              │
              │ 호출
              ▼
┌────────────────────────────────────┐
│         @Service                   │  ← Model (비즈니스)
│      - 비즈니스 로직                  │
└────────────────────────────────────┘
              │
              ▼
┌────────────────────────────────────┐
│         @Repository                │  ← Model (데이터)
│      - 데이터 접근                    │
└────────────────────────────────────┘
              │
              ▼
┌────────────────────────────────────┐
│          Database                  │
└────────────────────────────────────┘

              ▲
              │ ModelAndView
              │
┌────────────────────────────────────┐
│      View (JSP, Thymeleaf)         │  ← View
│      - HTML 렌더링                   │
└────────────────────────────────────┘
              │
              │ HTML Response
              ▼
┌────────────────────────────────────┐
│          Browser                   │
└────────────────────────────────────┘
```

### 🔧 구성요소

| 컴포넌트 | 역할 | 책임 | 예시 |
|---------|------|------|------|
| **Model** | 데이터 + 비즈니스 로직 | - 데이터 관리<br>- 비즈니스 규칙<br>- 상태 변경 통지 | `User`, `Order` |
| **View** | UI 표현 | - 데이터 표시<br>- 사용자 입력 수집<br>- Model 관찰 | `UserView`, JSP |
| **Controller** | 입력 처리 + 조정 | - 사용자 입력 해석<br>- Model 업데이트<br>- View 선택 | `UserController` |

---

## 4. 구현 방법

### 기본 구현: Desktop Todo 애플리케이션 ⭐⭐⭐

```java
/**
 * ============================================
 * MODEL (데이터 + 비즈니스 로직)
 * ============================================
 */

/**
 * Todo 엔티티
 */
public class Todo {
    private Long id;
    private String title;
    private String description;
    private boolean completed;
    private LocalDateTime createdAt;
    
    public Todo(String title, String description) {
        this.title = title;
        this.description = description;
        this.completed = false;
        this.createdAt = LocalDateTime.now();
    }
    
    /**
     * 비즈니스 로직: 완료 토글
     */
    public void toggleComplete() {
        this.completed = !this.completed;
        System.out.println("✓ Todo 상태 변경: " + title + " → " + 
            (completed ? "완료" : "미완료"));
    }
    
    /**
     * 비즈니스 로직: 수정
     */
    public void update(String title, String description) {
        if (title == null || title.trim().isEmpty()) {
            throw new IllegalArgumentException("제목은 필수입니다");
        }
        
        this.title = title;
        this.description = description;
        System.out.println("✏️ Todo 수정: " + title);
    }
    
    /**
     * 검증 로직
     */
    public void validate() {
        if (title == null || title.trim().isEmpty()) {
            throw new IllegalArgumentException("제목은 필수입니다");
        }
        
        if (title.length() > 100) {
            throw new IllegalArgumentException("제목은 100자 이하");
        }
    }
    
    // Getters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getTitle() { return title; }
    public String getDescription() { return description; }
    public boolean isCompleted() { return completed; }
    public LocalDateTime getCreatedAt() { return createdAt; }
}

/**
 * TodoModel: Observable 역할
 * - Todo 관리
 * - Observer에게 변경 통지
 */
public class TodoModel {
    private List<Todo> todos;
    private List<TodoObserver> observers;
    private Long nextId = 1L;
    
    public TodoModel() {
        this.todos = new ArrayList<>();
        this.observers = new ArrayList<>();
    }
    
    /**
     * Observer 등록
     */
    public void addObserver(TodoObserver observer) {
        observers.add(observer);
        System.out.println("👀 Observer 등록: " + observer.getClass().getSimpleName());
    }
    
    /**
     * Observer에게 변경 통지
     */
    private void notifyObservers() {
        System.out.println("📢 변경 통지 → " + observers.size() + "개 Observer");
        for (TodoObserver observer : observers) {
            observer.update(new ArrayList<>(todos));
        }
    }
    
    /**
     * Todo 추가
     */
    public void addTodo(Todo todo) {
        todo.validate(); // 검증
        todo.setId(nextId++);
        todos.add(todo);
        
        System.out.println("➕ Todo 추가: " + todo.getTitle());
        notifyObservers();
    }
    
    /**
     * Todo 수정
     */
    public void updateTodo(Long id, String title, String description) {
        Todo todo = findById(id);
        todo.update(title, description); // 비즈니스 로직
        
        notifyObservers();
    }
    
    /**
     * Todo 삭제
     */
    public void removeTodo(Long id) {
        Todo todo = findById(id);
        todos.remove(todo);
        
        System.out.println("🗑️ Todo 삭제: " + todo.getTitle());
        notifyObservers();
    }
    
    /**
     * Todo 완료 토글
     */
    public void toggleComplete(Long id) {
        Todo todo = findById(id);
        todo.toggleComplete(); // 비즈니스 로직
        
        notifyObservers();
    }
    
    /**
     * Todo 조회
     */
    public Todo findById(Long id) {
        return todos.stream()
            .filter(t -> t.getId().equals(id))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("Todo not found: " + id));
    }
    
    /**
     * 전체 Todo 조회
     */
    public List<Todo> getAllTodos() {
        return new ArrayList<>(todos);
    }
}

/**
 * Observer 인터페이스
 */
public interface TodoObserver {
    void update(List<Todo> todos);
}

/**
 * ============================================
 * VIEW (UI)
 * ============================================
 */

/**
 * Console View (텍스트 기반 UI)
 */
public class TodoConsoleView implements TodoObserver {
    private TodoController controller;
    
    public void setController(TodoController controller) {
        this.controller = controller;
    }
    
    /**
     * Model 변경 시 호출됨 (Observer)
     */
    @Override
    public void update(List<Todo> todos) {
        System.out.println("\n╔════════════════════════════════════╗");
        System.out.println("║         📝 TODO LIST               ║");
        System.out.println("╠════════════════════════════════════╣");
        
        if (todos.isEmpty()) {
            System.out.println("║  (할 일이 없습니다)                  ║");
        } else {
            for (Todo todo : todos) {
                String status = todo.isCompleted() ? "✓" : " ";
                String line = String.format("║ [%s] %d. %s", 
                    status, todo.getId(), todo.getTitle());
                System.out.println(line);
            }
        }
        
        System.out.println("╚════════════════════════════════════╝\n");
    }
    
    /**
     * 메뉴 표시
     */
    public void showMenu() {
        System.out.println("1. 추가  2. 완료  3. 삭제  4. 종료");
        System.out.print("선택: ");
    }
    
    /**
     * 사용자 입력 처리 (간단한 UI)
     */
    public void run() {
        Scanner scanner = new Scanner(System.in);
        
        while (true) {
            showMenu();
            String choice = scanner.nextLine();
            
            try {
                switch (choice) {
                    case "1": // 추가
                        System.out.print("제목: ");
                        String title = scanner.nextLine();
                        System.out.print("설명: ");
                        String desc = scanner.nextLine();
                        controller.addTodo(title, desc);
                        break;
                        
                    case "2": // 완료
                        System.out.print("완료할 Todo ID: ");
                        Long completeId = Long.parseLong(scanner.nextLine());
                        controller.toggleComplete(completeId);
                        break;
                        
                    case "3": // 삭제
                        System.out.print("삭제할 Todo ID: ");
                        Long removeId = Long.parseLong(scanner.nextLine());
                        controller.removeTodo(removeId);
                        break;
                        
                    case "4": // 종료
                        System.out.println("👋 종료합니다");
                        return;
                        
                    default:
                        System.out.println("❌ 잘못된 선택");
                }
            } catch (Exception e) {
                System.out.println("❌ 오류: " + e.getMessage());
            }
        }
    }
}

/**
 * Swing GUI View (그래픽 UI)
 */
public class TodoSwingView extends JFrame implements TodoObserver {
    private TodoController controller;
    private DefaultListModel<String> listModel;
    private JList<String> todoList;
    private JTextField titleField;
    private JTextArea descArea;
    private JButton addButton;
    private JButton completeButton;
    private JButton deleteButton;
    
    public TodoSwingView() {
        setTitle("Todo List");
        setSize(500, 400);
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        
        // UI 컴포넌트 초기화
        listModel = new DefaultListModel<>();
        todoList = new JList<>(listModel);
        
        titleField = new JTextField(20);
        descArea = new JTextArea(3, 20);
        
        addButton = new JButton("추가");
        completeButton = new JButton("완료");
        deleteButton = new JButton("삭제");
        
        // 레이아웃 구성
        setLayout(new BorderLayout());
        
        // 왼쪽: Todo 리스트
        add(new JScrollPane(todoList), BorderLayout.CENTER);
        
        // 오른쪽: 입력 폼
        JPanel inputPanel = new JPanel();
        inputPanel.setLayout(new BoxLayout(inputPanel, BoxLayout.Y_AXIS));
        inputPanel.add(new JLabel("제목:"));
        inputPanel.add(titleField);
        inputPanel.add(new JLabel("설명:"));
        inputPanel.add(new JScrollPane(descArea));
        
        JPanel buttonPanel = new JPanel();
        buttonPanel.add(addButton);
        buttonPanel.add(completeButton);
        buttonPanel.add(deleteButton);
        inputPanel.add(buttonPanel);
        
        add(inputPanel, BorderLayout.EAST);
    }
    
    public void setController(TodoController controller) {
        this.controller = controller;
        
        // 버튼 이벤트 → Controller로 전달
        addButton.addActionListener(e -> {
            String title = titleField.getText();
            String desc = descArea.getText();
            controller.addTodo(title, desc);
            
            // 입력 필드 초기화
            titleField.setText("");
            descArea.setText("");
        });
        
        completeButton.addActionListener(e -> {
            int index = todoList.getSelectedIndex();
            if (index >= 0) {
                Long id = (long) (index + 1); // 간단히 index+1을 ID로
                controller.toggleComplete(id);
            }
        });
        
        deleteButton.addActionListener(e -> {
            int index = todoList.getSelectedIndex();
            if (index >= 0) {
                Long id = (long) (index + 1);
                controller.removeTodo(id);
            }
        });
    }
    
    /**
     * Model 변경 시 호출됨 (Observer)
     */
    @Override
    public void update(List<Todo> todos) {
        // UI 업데이트
        SwingUtilities.invokeLater(() -> {
            listModel.clear();
            for (Todo todo : todos) {
                String status = todo.isCompleted() ? "✓" : " ";
                listModel.addElement(String.format("[%s] %s", status, todo.getTitle()));
            }
        });
    }
}

/**
 * ============================================
 * CONTROLLER (입력 처리 + 조정)
 * ============================================
 */

/**
 * TodoController
 * - 사용자 입력 처리
 * - Model 업데이트
 * - View는 직접 업데이트 안 함 (Observer 패턴으로 자동)
 */
public class TodoController {
    private TodoModel model;
    
    public TodoController(TodoModel model) {
        this.model = model;
    }
    
    /**
     * Todo 추가 처리
     */
    public void addTodo(String title, String description) {
        try {
            // Model 생성 및 추가
            Todo todo = new Todo(title, description);
            model.addTodo(todo);
            
            System.out.println("✅ Controller: Todo 추가 완료");
            
        } catch (IllegalArgumentException e) {
            System.out.println("❌ Controller: 검증 실패 - " + e.getMessage());
            throw e; // View에서 처리하도록
        }
    }
    
    /**
     * Todo 완료 토글
     */
    public void toggleComplete(Long id) {
        try {
            model.toggleComplete(id);
            System.out.println("✅ Controller: 완료 토글");
            
        } catch (Exception e) {
            System.out.println("❌ Controller: 오류 - " + e.getMessage());
            throw e;
        }
    }
    
    /**
     * Todo 삭제
     */
    public void removeTodo(Long id) {
        try {
            model.removeTodo(id);
            System.out.println("✅ Controller: Todo 삭제");
            
        } catch (Exception e) {
            System.out.println("❌ Controller: 오류 - " + e.getMessage());
            throw e;
        }
    }
    
    /**
     * Todo 수정
     */
    public void updateTodo(Long id, String title, String description) {
        try {
            model.updateTodo(id, title, description);
            System.out.println("✅ Controller: Todo 수정");
            
        } catch (Exception e) {
            System.out.println("❌ Controller: 오류 - " + e.getMessage());
            throw e;
        }
    }
}

/**
 * ============================================
 * APPLICATION (메인)
 * ============================================
 */
public class MVCExample {
    public static void main(String[] args) {
        System.out.println("=== MVC 패턴 Todo 애플리케이션 ===\n");
        
        // 1. Model 생성
        TodoModel model = new TodoModel();
        
        // 2. Controller 생성
        TodoController controller = new TodoController(model);
        
        // 3. View 생성 및 연결
        TodoConsoleView consoleView = new TodoConsoleView();
        consoleView.setController(controller);
        
        // 4. View를 Observer로 등록
        model.addObserver(consoleView);
        
        // 5. 초기 데이터
        System.out.println("\n--- 초기 데이터 추가 ---");
        controller.addTodo("MVC 패턴 학습", "Model, View, Controller 이해");
        controller.addTodo("Observer 패턴 복습", "MVC와의 관계 정리");
        
        // 6. View 실행
        System.out.println("\n--- 애플리케이션 시작 ---");
        consoleView.run();
    }
}
```

**실행 결과:**
```
=== MVC 패턴 Todo 애플리케이션 ===

👀 Observer 등록: TodoConsoleView

--- 초기 데이터 추가 ---
➕ Todo 추가: MVC 패턴 학습
📢 변경 통지 → 1개 Observer

╔════════════════════════════════════╗
║         📝 TODO LIST               ║
╠════════════════════════════════════╣
║ [ ] 1. MVC 패턴 학습                 ║
╚════════════════════════════════════╝

✅ Controller: Todo 추가 완료
➕ Todo 추가: Observer 패턴 복습
📢 변경 통지 → 1개 Observer

╔════════════════════════════════════╗
║         📝 TODO LIST               ║
╠════════════════════════════════════╣
║ [ ] 1. MVC 패턴 학습                 ║
║ [ ] 2. Observer 패턴 복습            ║
╚════════════════════════════════════╝

--- 애플리케이션 시작 ---
1. 추가  2. 완료  3. 삭제  4. 종료
선택: 2
완료할 Todo ID: 1
✓ Todo 상태 변경: MVC 패턴 학습 → 완료
📢 변경 통지 → 1개 Observer

╔════════════════════════════════════╗
║         📝 TODO LIST               ║
╠════════════════════════════════════╣
║ [✓] 1. MVC 패턴 학습                 ║
║ [ ] 2. Observer 패턴 복습            ║
╚════════════════════════════════════╝

✅ Controller: 완료 토글
```

---

## 5. 실전 예제

### 예제 1: Spring MVC 웹 애플리케이션 ⭐⭐⭐

```java
/**
 * ============================================
 * MODEL (Domain + Service + Repository)
 * ============================================
 */

/**
 * Entity: 게시글
 */
@Entity
@Table(name = "posts")
public class Post {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String title;
    
    @Column(columnDefinition = "TEXT")
    private String content;
    
    @Column(nullable = false)
    private String author;
    
    @Column(nullable = false)
    private LocalDateTime createdAt;
    
    @Column(nullable = false)
    private int viewCount;
    
    protected Post() {}
    
    public Post(String title, String content, String author) {
        this.title = title;
        this.content = content;
        this.author = author;
        this.createdAt = LocalDateTime.now();
        this.viewCount = 0;
    }
    
    /**
     * 비즈니스 로직: 조회수 증가
     */
    public void increaseViewCount() {
        this.viewCount++;
    }
    
    /**
     * 비즈니스 로직: 수정
     */
    public void update(String title, String content) {
        if (title != null && !title.trim().isEmpty()) {
            this.title = title;
        }
        if (content != null) {
            this.content = content;
        }
    }
    
    // Getters
    public Long getId() { return id; }
    public String getTitle() { return title; }
    public String getContent() { return content; }
    public String getAuthor() { return author; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public int getViewCount() { return viewCount; }
}

/**
 * Repository: 데이터 접근
 */
public interface PostRepository extends JpaRepository<Post, Long> {
    List<Post> findByAuthor(String author);
    
    @Query("SELECT p FROM Post p ORDER BY p.createdAt DESC")
    List<Post> findAllOrderByCreatedAtDesc();
}

/**
 * Service: 비즈니스 로직
 */
@Service
@Transactional
public class PostService {
    private final PostRepository postRepository;
    
    @Autowired
    public PostService(PostRepository postRepository) {
        this.postRepository = postRepository;
    }
    
    /**
     * 게시글 생성
     */
    public Post createPost(String title, String content, String author) {
        Post post = new Post(title, content, author);
        return postRepository.save(post);
    }
    
    /**
     * 게시글 조회 (조회수 증가)
     */
    public Post getPost(Long id) {
        Post post = postRepository.findById(id)
            .orElseThrow(() -> new PostNotFoundException(id));
        
        post.increaseViewCount(); // 비즈니스 로직
        
        return post;
    }
    
    /**
     * 전체 게시글 조회
     */
    @Transactional(readOnly = true)
    public List<Post> getAllPosts() {
        return postRepository.findAllOrderByCreatedAtDesc();
    }
    
    /**
     * 게시글 수정
     */
    public Post updatePost(Long id, String title, String content) {
        Post post = postRepository.findById(id)
            .orElseThrow(() -> new PostNotFoundException(id));
        
        post.update(title, content); // 비즈니스 로직
        
        return post;
    }
    
    /**
     * 게시글 삭제
     */
    public void deletePost(Long id) {
        postRepository.deleteById(id);
    }
}

/**
 * ============================================
 * VIEW (DTO)
 * ============================================
 */

/**
 * Request DTO: 게시글 생성
 */
public class CreatePostRequest {
    @NotBlank(message = "제목은 필수입니다")
    @Size(max = 100, message = "제목은 100자 이하")
    private String title;
    
    @NotBlank(message = "내용은 필수입니다")
    private String content;
    
    @NotBlank(message = "작성자는 필수입니다")
    private String author;
    
    // Getters, Setters
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    public String getContent() { return content; }
    public void setContent(String content) { this.content = content; }
    public String getAuthor() { return author; }
    public void setAuthor(String author) { this.author = author; }
}

/**
 * Response DTO: 게시글 응답
 */
public class PostResponse {
    private Long id;
    private String title;
    private String content;
    private String author;
    private LocalDateTime createdAt;
    private int viewCount;
    
    public PostResponse(Post post) {
        this.id = post.getId();
        this.title = post.getTitle();
        this.content = post.getContent();
        this.author = post.getAuthor();
        this.createdAt = post.getCreatedAt();
        this.viewCount = post.getViewCount();
    }
    
    // Getters
    public Long getId() { return id; }
    public String getTitle() { return title; }
    public String getContent() { return content; }
    public String getAuthor() { return author; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public int getViewCount() { return viewCount; }
}

/**
 * ============================================
 * CONTROLLER (입력 처리)
 * ============================================
 */

/**
 * REST Controller
 */
@RestController
@RequestMapping("/api/posts")
public class PostController {
    private final PostService postService;
    
    @Autowired
    public PostController(PostService postService) {
        this.postService = postService;
    }
    
    /**
     * POST /api/posts - 게시글 생성
     */
    @PostMapping
    public ResponseEntity<PostResponse> createPost(
            @Valid @RequestBody CreatePostRequest request) {
        
        // Service 호출 (Model)
        Post post = postService.createPost(
            request.getTitle(),
            request.getContent(),
            request.getAuthor()
        );
        
        // Response DTO 변환 (View)
        PostResponse response = new PostResponse(post);
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * GET /api/posts/{id} - 게시글 조회
     */
    @GetMapping("/{id}")
    public ResponseEntity<PostResponse> getPost(@PathVariable Long id) {
        Post post = postService.getPost(id);
        PostResponse response = new PostResponse(post);
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * GET /api/posts - 전체 게시글 조회
     */
    @GetMapping
    public ResponseEntity<List<PostResponse>> getAllPosts() {
        List<Post> posts = postService.getAllPosts();
        
        List<PostResponse> responses = posts.stream()
            .map(PostResponse::new)
            .collect(Collectors.toList());
        
        return ResponseEntity.ok(responses);
    }
    
    /**
     * PUT /api/posts/{id} - 게시글 수정
     */
    @PutMapping("/{id}")
    public ResponseEntity<PostResponse> updatePost(
            @PathVariable Long id,
            @Valid @RequestBody CreatePostRequest request) {
        
        Post post = postService.updatePost(
            id,
            request.getTitle(),
            request.getContent()
        );
        
        PostResponse response = new PostResponse(post);
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * DELETE /api/posts/{id} - 게시글 삭제
     */
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deletePost(@PathVariable Long id) {
        postService.deletePost(id);
        return ResponseEntity.noContent().build();
    }
}

/**
 * Web Controller (HTML 반환)
 */
@Controller
@RequestMapping("/posts")
public class PostWebController {
    private final PostService postService;
    
    @Autowired
    public PostWebController(PostService postService) {
        this.postService = postService;
    }
    
    /**
     * GET /posts - 게시글 목록 페이지
     */
    @GetMapping
    public String listPosts(Model model) {
        List<Post> posts = postService.getAllPosts();
        model.addAttribute("posts", posts);
        
        return "post/list"; // View 이름 (list.html)
    }
    
    /**
     * GET /posts/{id} - 게시글 상세 페이지
     */
    @GetMapping("/{id}")
    public String viewPost(@PathVariable Long id, Model model) {
        Post post = postService.getPost(id);
        model.addAttribute("post", post);
        
        return "post/detail"; // View 이름 (detail.html)
    }
    
    /**
     * GET /posts/new - 게시글 작성 페이지
     */
    @GetMapping("/new")
    public String newPostForm(Model model) {
        model.addAttribute("post", new CreatePostRequest());
        return "post/form"; // View 이름 (form.html)
    }
    
    /**
     * POST /posts - 게시글 생성 처리
     */
    @PostMapping
    public String createPost(@Valid @ModelAttribute CreatePostRequest request,
                           BindingResult result) {
        if (result.hasErrors()) {
            return "post/form";
        }
        
        Post post = postService.createPost(
            request.getTitle(),
            request.getContent(),
            request.getAuthor()
        );
        
        return "redirect:/posts/" + post.getId();
    }
}
```

**Thymeleaf View (list.html):**
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>게시글 목록</title>
</head>
<body>
    <h1>📝 게시글 목록</h1>
    
    <a href="/posts/new">새 글 작성</a>
    
    <table>
        <thead>
            <tr>
                <th>번호</th>
                <th>제목</th>
                <th>작성자</th>
                <th>조회수</th>
                <th>작성일</th>
            </tr>
        </thead>
        <tbody>
            <tr th:each="post : ${posts}">
                <td th:text="${post.id}">1</td>
                <td>
                    <a th:href="@{/posts/{id}(id=${post.id})}" 
                       th:text="${post.title}">제목</a>
                </td>
                <td th:text="${post.author}">작성자</td>
                <td th:text="${post.viewCount}">0</td>
                <td th:text="${#temporals.format(post.createdAt, 'yyyy-MM-dd')}">
                    2025-01-01
                </td>
            </tr>
        </tbody>
    </table>
</body>
</html>
```

---

## 6. MVC 변형들

### 📱 MVC의 주요 변형

| 패턴 | 특징 | 사용처 |
|------|------|--------|
| **MVC (전통적)** | Model이 View에 직접 통지 | Swing, JavaFX |
| **MVP** | View가 수동적, Presenter가 모든 제어 | Android (구형) |
| **MVVM** | ViewModel + 데이터 바인딩 | WPF, Angular, Vue |
| **MVC Model 2** | Front Controller 패턴 | Spring MVC, Struts |

### 🔄 MVC vs MVP vs MVVM

```
=== MVC ===
View ──→ Controller ──→ Model
 ↑                        │
 └────── notify ──────────┘

=== MVP ===
View ←──→ Presenter ──→ Model
(View는 완전 수동)

=== MVVM ===
View ←─ Data Binding ─→ ViewModel ──→ Model
(자동 동기화)
```

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 | 실무 효과 |
|------|------|-----------|
| **관심사 분리** | UI, 로직, 데이터 독립 | 유지보수 용이 |
| **재사용성** | Model을 여러 View에서 재사용 | Web + Mobile |
| **테스트 용이** | Model, Controller 독립 테스트 | 단위 테스트 |
| **병렬 개발** | MVC 각각 독립 개발 | 팀 협업 |
| **확장 용이** | 새 View 추가 쉬움 | UI 변경 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **복잡도** | 간단한 UI도 3개 컴포넌트 | 상황에 맞게 |
| **Controller 비대** | 로직이 Controller에 집중 | Service Layer |
| **View-Model 결합** | View가 Model 구조 알아야 | DTO 사용 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: Massive View Controller (비대한 Controller)

```java
// 잘못된 예: Controller에 모든 로직
@Controller
public class UserController {
    
    @PostMapping("/users")
    public String createUser(@RequestParam String name, 
                           @RequestParam String email) {
        // ❌ 검증 로직이 Controller에
        if (name.isEmpty()) {
            return "error";
        }
        if (!email.contains("@")) {
            return "error";
        }
        
        // ❌ 비즈니스 로직이 Controller에
        String hashedPassword = BCrypt.hashpw(password, BCrypt.gensalt());
        
        // ❌ DB 접근까지 Controller에
        Connection conn = DriverManager.getConnection(DB_URL);
        // ...
        
        return "success";
    }
}
```

**해결:**
```java
// 올바른 예: Controller는 조정만
@Controller
public class UserController {
    @Autowired
    private UserService userService; // ✅ Service에 위임
    
    @PostMapping("/users")
    public String createUser(@Valid UserRequest request) {
        // ✅ Service 호출
        User user = userService.createUser(request);
        return "redirect:/users/" + user.getId();
    }
}

@Service
public class UserService {
    // ✅ 비즈니스 로직은 Service에
    public User createUser(UserRequest request) {
        User user = new User(request.getName(), request.getEmail());
        user.validate(); // 도메인 로직
        return userRepository.save(user);
    }
}
```

### ❌ 안티패턴 2: Anemic Domain Model (빈약한 도메인 모델)

```java
// 잘못된 예: Model이 Getter/Setter만
public class Order {
    private List<OrderItem> items;
    private BigDecimal total;
    
    // Getter/Setter만 있음
    public List<OrderItem> getItems() { return items; }
    public void setItems(List<OrderItem> items) { this.items = items; }
    public BigDecimal getTotal() { return total; }
    public void setTotal(BigDecimal total) { this.total = total; }
}

// ❌ 로직이 Service에
@Service
public class OrderService {
    public void addItem(Order order, OrderItem item) {
        order.getItems().add(item); // ❌ Service가 직접 조작
        
        // ❌ 총액 계산도 Service에
        BigDecimal total = BigDecimal.ZERO;
        for (OrderItem i : order.getItems()) {
            total = total.add(i.getPrice());
        }
        order.setTotal(total);
    }
}
```

**해결:**
```java
// 올바른 예: Rich Domain Model
public class Order {
    private List<OrderItem> items;
    private BigDecimal total;
    
    // ✅ 비즈니스 로직이 Model에
    public void addItem(OrderItem item) {
        items.add(item);
        calculateTotal(); // 도메인 로직
    }
    
    private void calculateTotal() {
        this.total = items.stream()
            .map(OrderItem::getPrice)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
    
    // ✅ 불변성 보호
    public List<OrderItem> getItems() {
        return Collections.unmodifiableList(items);
    }
}

@Service
public class OrderService {
    public void addItemToOrder(Long orderId, OrderItem item) {
        Order order = orderRepository.findById(orderId);
        order.addItem(item); // ✅ 도메인 로직 호출
        orderRepository.save(order);
    }
}
```

---

## 9. 심화 주제

### 🎯 MVC + DDD (Domain-Driven Design)

```java
/**
 * 도메인 주도 설계와 MVC 통합
 */

// Aggregate Root
@Entity
public class Order {
    @Id
    private Long id;
    
    @Embedded
    private OrderStatus status;
    
    @ElementCollection
    private List<OrderLine> orderLines;
    
    // ✅ 도메인 로직
    public void place() {
        if (orderLines.isEmpty()) {
            throw new CannotPlaceEmptyOrderException();
        }
        this.status = OrderStatus.PLACED;
    }
    
    public void cancel() {
        if (!status.canCancel()) {
            throw new CannotCancelOrderException();
        }
        this.status = OrderStatus.CANCELLED;
    }
}

// Value Object
@Embeddable
public class OrderStatus {
    private String value;
    
    public boolean canCancel() {
        return value.equals("PLACED") || value.equals("CONFIRMED");
    }
}

// Repository (인터페이스는 Domain에)
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(Long id);
}

// Application Service
@Service
public class OrderApplicationService {
    private final OrderRepository orderRepository;
    
    @Transactional
    public void placeOrder(PlaceOrderCommand command) {
        // 1. Aggregate 생성
        Order order = new Order(command.getCustomerId());
        
        // 2. 도메인 로직 실행
        for (OrderItemDTO item : command.getItems()) {
            order.addOrderLine(new OrderLine(item));
        }
        order.place();
        
        // 3. 저장
        orderRepository.save(order);
    }
}
```

### 🔥 Front Controller Pattern (Spring MVC)

```java
/**
 * DispatcherServlet (Front Controller)
 */
public class DispatcherServlet extends HttpServlet {
    
    @Override
    protected void doGet(HttpServletRequest request, 
                        HttpServletResponse response) {
        
        // 1. Handler Mapping: URL → Controller 매핑
        HandlerExecutionChain handler = getHandler(request);
        
        // 2. Handler Adapter: Controller 실행
        ModelAndView mav = handleRequest(handler, request, response);
        
        // 3. View Resolver: View 이름 → 실제 View
        View view = resolveViewName(mav.getViewName());
        
        // 4. View Rendering
        view.render(mav.getModel(), request, response);
    }
}

/**
 * 실제 Controller
 */
@Controller
public class ProductController {
    
    @GetMapping("/products/{id}")
    public String getProduct(@PathVariable Long id, Model model) {
        // Model 업데이트
        Product product = productService.findById(id);
        model.addAttribute("product", product);
        
        // View 이름 반환
        return "product/detail";
    }
}
```

---

## 10. 핵심 정리

### 📌 MVC 패턴 체크리스트

```
✅ Model: 데이터 + 비즈니스 로직 (독립적)
✅ View: UI만 (Model 관찰)
✅ Controller: 입력 처리 + 조정 (얇게)
✅ Model ↔ View 직접 의존 없음
✅ Observer 패턴으로 Model → View 통지
✅ 비즈니스 로직은 Model에
✅ Service Layer 분리 (복잡한 경우)
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 웹 애플리케이션 | ⭐⭐⭐ | 표준 패턴 |
| Desktop GUI | ⭐⭐⭐ | Swing, JavaFX |
| 복잡한 UI | ⭐⭐⭐ | 관심사 분리 |
| 간단한 CRUD | ⭐⭐ | 오버엔지니어링 가능 |

### 💡 핵심 원칙

1. **Model은 독립적** (UI 몰라야 함)
2. **View는 수동적** (Model 관찰만)
3. **Controller는 얇게** (조정만)
4. **비즈니스 로직은 Model**

### 🔥 실무 팁

```java
// ✅ DO: Model에 비즈니스 로직
public class User {
    public void changePassword(String newPassword) {
        validatePassword(newPassword);
        this.password = encrypt(newPassword);
    }
}

// ❌ DON'T: Controller에 비즈니스 로직
@Controller
public class UserController {
    public void changePassword(String password) {
        if (password.length() < 8) { /* ... */ } // ❌
        user.setPassword(encrypt(password)); // ❌
    }
}

// ✅ DO: Service Layer 분리 (복잡한 경우)
@Service
public class UserService {
    @Transactional
    public void changePassword(Long userId, String newPassword) {
        User user = userRepository.findById(userId);
        user.changePassword(newPassword); // ✅ 도메인 로직
        userRepository.save(user);
        emailService.sendPasswordChangeNotification(user);
    }
}
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Layered Architecture](./01-LayeredArchitecture.md) | [다음: Repository →](./03-Repository.md)**

</div>
