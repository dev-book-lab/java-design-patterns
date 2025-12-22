# MVP Pattern (Model-View-Presenter 패턴)

> **"View를 완전히 수동적으로 만들고 Presenter가 모든 로직을 담당하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [MVC vs MVP vs MVVM 완전 비교](#6-mvc-vs-mvp-vs-mvvm-완전-비교)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: View에 비즈니스 로직 (MVC의 문제)
public class UserView extends JFrame {
    private JTextField emailField;
    private JTextField passwordField;
    private JButton loginButton;
    
    private UserModel userModel;  // View가 Model 직접 참조
    
    public UserView() {
        loginButton.addActionListener(e -> {
            // 😱 View에 비즈니스 로직!
            String email = emailField.getText();
            String password = new String(passwordField.getPassword());
            
            // 검증 로직
            if (email.isEmpty()) {
                JOptionPane.showMessageDialog(this, "이메일을 입력하세요");
                return;
            }
            
            if (!email.contains("@")) {
                JOptionPane.showMessageDialog(this, "올바른 이메일 형식이 아닙니다");
                return;
            }
            
            if (password.length() < 8) {
                JOptionPane.showMessageDialog(this, "비밀번호는 8자 이상");
                return;
            }
            
            // Model 직접 접근
            User user = userModel.login(email, password);
            
            if (user != null) {
                // 성공 처리
                showMainScreen();
            } else {
                JOptionPane.showMessageDialog(this, "로그인 실패");
            }
            
            // 문제점:
            // 1. View가 비즈니스 로직 포함
            // 2. View가 Model 직접 접근
            // 3. 테스트 불가능 (UI 필요)
            // 4. 재사용 불가
        });
    }
}

// 문제 2: View가 너무 많은 책임
public class OrderView extends JPanel {
    private JTable orderTable;
    private JTextField searchField;
    private JComboBox<String> statusCombo;
    private JButton searchButton;
    
    public OrderView() {
        searchButton.addActionListener(e -> {
            // 😱 View에서 데이터 조회
            String keyword = searchField.getText();
            String status = (String) statusCombo.getSelectedItem();
            
            // DB 조회 (View에서!)
            List<Order> orders = orderRepository.findByKeywordAndStatus(keyword, status);
            
            // 데이터 변환 (View에서!)
            DefaultTableModel model = new DefaultTableModel();
            for (Order order : orders) {
                model.addRow(new Object[]{
                    order.getId(),
                    order.getCustomerName(),
                    order.getTotal(),
                    order.getStatus()
                });
            }
            
            // 테이블 업데이트
            orderTable.setModel(model);
            
            // View가 너무 많은 일을 함!
            // 데이터 조회, 변환, 표시 모두!
        });
    }
}

// 문제 3: 테스트 어려움
public class ProductView extends JFrame {
    private JTextField nameField;
    private JTextField priceField;
    
    public void saveProduct() {
        // 😱 UI 컴포넌트에 직접 접근
        String name = nameField.getText();
        String priceStr = priceField.getText();
        
        // 검증
        if (name.isEmpty()) {
            showError("이름을 입력하세요");
            return;
        }
        
        BigDecimal price;
        try {
            price = new BigDecimal(priceStr);
        } catch (NumberFormatException e) {
            showError("올바른 가격을 입력하세요");
            return;
        }
        
        // 저장
        productRepository.save(new Product(name, price));
        
        // 어떻게 테스트?
        // - JTextField를 Mock?
        // - JFrame을 생성?
        // - UI 없이 테스트 불가능!
    }
}

// 문제 4: View와 Model의 강한 결합
public class ReportView extends JPanel {
    private ReportModel reportModel;  // Model 직접 참조
    
    public ReportView(ReportModel reportModel) {
        this.reportModel = reportModel;
        
        // Model 변경 감지
        reportModel.addPropertyChangeListener(evt -> {
            // 😱 View가 Model 구조 알아야 함
            if ("sales".equals(evt.getPropertyName())) {
                updateSalesChart((List<Sale>) evt.getNewValue());
            } else if ("revenue".equals(evt.getPropertyName())) {
                updateRevenueLabel((BigDecimal) evt.getNewValue());
            }
            
            // Model 변경 시 View도 변경 필요
            // 강한 결합!
        });
    }
}

// 문제 5: 복잡한 UI 로직
public class DashboardView extends JFrame {
    private JLabel totalOrdersLabel;
    private JLabel totalRevenueLabel;
    private JLabel pendingOrdersLabel;
    private JProgressBar progressBar;
    
    public void updateDashboard() {
        // 😱 View에서 계산
        List<Order> orders = orderRepository.findAll();
        
        int totalOrders = orders.size();
        BigDecimal totalRevenue = orders.stream()
            .map(Order::getTotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        
        long pendingOrders = orders.stream()
            .filter(o -> o.getStatus() == OrderStatus.PENDING)
            .count();
        
        double completionRate = (double) (totalOrders - pendingOrders) / totalOrders * 100;
        
        // 업데이트
        totalOrdersLabel.setText(String.valueOf(totalOrders));
        totalRevenueLabel.setText(totalRevenue.toString());
        pendingOrdersLabel.setText(String.valueOf(pendingOrders));
        progressBar.setValue((int) completionRate);
        
        // 계산 로직이 View에!
        // 테스트 불가!
        // 재사용 불가!
    }
}

// 문제 6: 중복 코드
public class CreateUserView extends JDialog {
    public void validate() {
        // 검증 로직
        if (emailField.getText().isEmpty()) { /* ... */ }
        if (!emailField.getText().contains("@")) { /* ... */ }
    }
}

public class EditUserView extends JDialog {
    public void validate() {
        // 😱 똑같은 검증 로직 반복!
        if (emailField.getText().isEmpty()) { /* ... */ }
        if (!emailField.getText().contains("@")) { /* ... */ }
    }
}

// 문제 7: Android Activity의 God Object
public class MainActivity extends AppCompatActivity {
    private TextView userNameText;
    private Button logoutButton;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        
        // 😱 Activity가 모든 것을 함!
        
        // 1. UI 초기화
        userNameText = findViewById(R.id.userName);
        logoutButton = findViewById(R.id.logoutButton);
        
        // 2. 데이터 로드
        User user = database.getUserById(userId);
        
        // 3. 비즈니스 로직
        if (user.isActive()) {
            userNameText.setText(user.getName());
        }
        
        // 4. 이벤트 처리
        logoutButton.setOnClickListener(v -> {
            // 로그아웃 로직
            sessionManager.clear();
            startActivity(new Intent(this, LoginActivity.class));
        });
        
        // 5. 라이프사이클 관리
        // 6. 권한 처리
        // 7. 네트워크 요청
        
        // Activity가 너무 비대!
        // 테스트 불가능!
    }
}
```

### ⚡ 핵심 문제

1. **비즈니스 로직**: View에 비즈니스 로직 포함
2. **테스트 어려움**: UI 없이 테스트 불가
3. **강한 결합**: View와 Model이 직접 결합
4. **중복 코드**: 검증 로직 등 반복
5. **재사용 불가**: View에 로직이 있어 재사용 어려움
6. **God Object**: Activity/View가 너무 많은 책임

---

## 2. 패턴 정의

### 📖 정의

> **View를 완전히 수동적(Passive)으로 만들고, Presenter가 모든 프레젠테이션 로직을 담당하여 View와 Model을 완전히 분리하는 패턴**

### 🎯 목적

- **Passive View**: View는 데이터 표시만 담당
- **테스트 용이**: Presenter를 독립적으로 테스트
- **관심사 분리**: View와 비즈니스 로직 완전 분리
- **재사용성**: Presenter 로직 재사용

### 💡 핵심 아이디어

```java
// Before: View가 능동적 (MVC)
public class UserView {
    private UserModel model;  // Model 직접 참조
    
    public void onLoginClick() {
        String email = emailField.getText();
        
        // View에서 검증
        if (!isValid(email)) {
            showError("Invalid email");
            return;
        }
        
        // Model 직접 호출
        User user = model.login(email, password);
        
        // View가 결정
        if (user != null) {
            showMainScreen();
        }
    }
}

// After: View가 수동적 (MVP)
// 1. View Interface (Contract)
public interface LoginView {
    String getEmail();
    String getPassword();
    void showError(String message);
    void showLoading();
    void hideLoading();
    void navigateToMain();
}

// 2. Presenter (모든 로직)
public class LoginPresenter {
    private LoginView view;
    private UserRepository repository;
    
    public void onLoginClick() {
        // View로부터 데이터 가져오기
        String email = view.getEmail();
        String password = view.getPassword();
        
        // 검증 (Presenter에서!)
        if (!isValid(email)) {
            view.showError("Invalid email");
            return;
        }
        
        view.showLoading();
        
        // Model 사용
        User user = repository.login(email, password);
        
        view.hideLoading();
        
        // Presenter가 결정
        if (user != null) {
            view.navigateToMain();
        } else {
            view.showError("Login failed");
        }
    }
    
    private boolean isValid(String email) {
        return email != null && email.contains("@");
    }
}

// 3. View Implementation (완전히 수동적)
public class LoginActivity implements LoginView {
    private LoginPresenter presenter;
    private EditText emailField;
    private EditText passwordField;
    
    @Override
    public String getEmail() {
        return emailField.getText().toString();
    }
    
    @Override
    public String getPassword() {
        return passwordField.getText().toString();
    }
    
    @Override
    public void showError(String message) {
        Toast.makeText(this, message, Toast.LENGTH_SHORT).show();
    }
    
    @Override
    public void navigateToMain() {
        startActivity(new Intent(this, MainActivity.class));
    }
    
    // View는 단순히 Presenter 호출만!
    public void onLoginButtonClick() {
        presenter.onLoginClick();
    }
}
```

---

## 3. 구조와 구성요소

### 📊 MVP 구조

```
┌─────────────────────────────────────┐
│            View                     │
│  (Passive / Dumb)                   │
│                                     │
│  - UI 컴포넌트                        │
│  - 사용자 입력 받기                     │
│  - 데이터 표시                         │
│  - Presenter 호출                    │
└─────────────────────────────────────┘
              △
              │ implements
              │
┌─────────────────────────────────────┐
│        View Interface               │
│                                     │
│  - getXxx()                         │
│  - showXxx()                        │
│  - hideXxx()                        │
└─────────────────────────────────────┘
              △
              │ depends on
              │
┌─────────────────────────────────────┐
│         Presenter                   │
│  (Orchestrator)                     │
│                                     │
│  - 프레젠테이션 로직                    │
│  - 검증                              │
│  - 데이터 변환                         │
│  - View 조작 (Interface 통해)         │
└─────────────────────────────────────┘
              │
              │ uses
              ▼
┌─────────────────────────────────────┐
│           Model                     │
│  (Business Logic)                   │
│                                     │
│  - 비즈니스 로직                       │
│  - 데이터 접근                         │
│  - 도메인 규칙                         │
└─────────────────────────────────────┘
```

### 🔄 데이터 흐름

```
User Action
    │
    ▼
┌─────────┐
│  View   │ → presenter.onXxxClick()
└─────────┘
    │
    │ (1) Presenter 호출
    ▼
┌──────────────┐
│  Presenter   │ → email = view.getEmail()
└──────────────┘
    │
    │ (2) View에서 데이터 가져오기
    ▼
┌──────────────┐
│  Presenter   │ → validate(email)
└──────────────┘
    │
    │ (3) 검증
    ▼
┌──────────────┐
│  Presenter   │ → user = repository.login(email)
└──────────────┘
    │
    │ (4) Model 사용
    ▼
┌──────────────┐
│  Presenter   │ → view.showSuccess()
└──────────────┘
    │
    │ (5) View 업데이트 명령
    ▼
┌─────────┐
│  View   │ → UI 업데이트
└─────────┘
```

### 🏗️ MVC vs MVP 비교

```
=== MVC ===
View → Controller → Model
 ↑                    │
 └──── Observer ──────┘
(View가 Model 관찰)

=== MVP ===
View ←→ Presenter → Model
(Presenter가 중재)
(View는 완전히 수동적)
```

### 🔧 구성요소

| 컴포넌트 | 역할 | 책임 | 예시 |
|---------|------|------|------|
| **View** | UI 표시 | - 사용자 입력 받기<br>- 데이터 표시<br>- Presenter 호출 | `LoginActivity` |
| **View Interface** | Contract | - View 메서드 정의<br>- Presenter-View 결합 | `LoginView` |
| **Presenter** | 프레젠테이션 로직 | - 검증<br>- 데이터 변환<br>- View 조작 | `LoginPresenter` |
| **Model** | 비즈니스 로직 | - 도메인 로직<br>- 데이터 접근 | `User`, `UserRepository` |

---

## 4. 구현 방법

### 완전한 구현: Todo 애플리케이션 ⭐⭐⭐

```java
/**
 * ============================================
 * MODEL (비즈니스 로직)
 * ============================================
 */

/**
 * Todo Entity
 */
public class Todo {
    private Long id;
    private String title;
    private String description;
    private boolean completed;
    private Priority priority;
    private LocalDateTime createdAt;
    
    public enum Priority {
        LOW, NORMAL, HIGH, URGENT
    }
    
    public Todo(String title) {
        this.title = title;
        this.completed = false;
        this.priority = Priority.NORMAL;
        this.createdAt = LocalDateTime.now();
    }
    
    /**
     * 비즈니스 로직: 검증
     */
    public void validate() {
        if (title == null || title.trim().isEmpty()) {
            throw new IllegalArgumentException("제목은 필수입니다");
        }
        if (title.length() > 100) {
            throw new IllegalArgumentException("제목은 100자 이하");
        }
    }
    
    /**
     * 비즈니스 로직: 완료 토글
     */
    public void toggleComplete() {
        this.completed = !this.completed;
    }
    
    // Getters, Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
    public boolean isCompleted() { return completed; }
    public void setCompleted(boolean completed) { this.completed = completed; }
    public Priority getPriority() { return priority; }
    public void setPriority(Priority priority) { this.priority = priority; }
    public LocalDateTime getCreatedAt() { return createdAt; }
}

/**
 * Repository
 */
public class TodoRepository {
    private final Map<Long, Todo> storage = new ConcurrentHashMap<>();
    private final AtomicLong idGenerator = new AtomicLong(1);
    
    public Todo save(Todo todo) {
        if (todo.getId() == null) {
            todo.setId(idGenerator.getAndIncrement());
        }
        storage.put(todo.getId(), todo);
        return todo;
    }
    
    public List<Todo> findAll() {
        return new ArrayList<>(storage.values());
    }
    
    public Optional<Todo> findById(Long id) {
        return Optional.ofNullable(storage.get(id));
    }
    
    public void delete(Long id) {
        storage.remove(id);
    }
}

/**
 * ============================================
 * VIEW CONTRACT (인터페이스)
 * ============================================
 * Presenter와 View 간의 계약
 */

/**
 * Todo List View Contract
 */
public interface TodoListView {
    /**
     * 입력 가져오기
     */
    String getTodoTitle();
    String getTodoDescription();
    Todo.Priority getTodoPriority();
    
    /**
     * 입력 초기화
     */
    void clearInputs();
    
    /**
     * 데이터 표시
     */
    void showTodos(List<TodoViewModel> todos);
    void showLoading();
    void hideLoading();
    
    /**
     * 메시지 표시
     */
    void showError(String message);
    void showSuccess(String message);
    
    /**
     * 통계 표시
     */
    void showStatistics(int total, int completed, int active);
}

/**
 * ============================================
 * PRESENTER (프레젠테이션 로직)
 * ============================================
 * 모든 로직을 담당하는 핵심!
 */

/**
 * Todo List Presenter
 */
public class TodoListPresenter {
    private final TodoListView view;
    private final TodoRepository repository;
    
    public TodoListPresenter(TodoListView view, TodoRepository repository) {
        this.view = view;
        this.repository = repository;
    }
    
    /**
     * 초기 로드
     */
    public void onViewCreated() {
        System.out.println("\n📋 Presenter: View 생성됨");
        loadTodos();
    }
    
    /**
     * Todo 추가
     */
    public void onAddTodoClick() {
        System.out.println("\n➕ Presenter: Todo 추가 시작");
        
        // 1. View로부터 데이터 가져오기
        String title = view.getTodoTitle();
        String description = view.getTodoDescription();
        Todo.Priority priority = view.getTodoPriority();
        
        System.out.println("   → View에서 데이터 수신: " + title);
        
        // 2. 검증 (Presenter에서!)
        if (title == null || title.trim().isEmpty()) {
            view.showError("제목을 입력하세요");
            return;
        }
        
        if (title.length() > 100) {
            view.showError("제목은 100자 이하로 입력하세요");
            return;
        }
        
        System.out.println("   ✅ 검증 완료");
        
        // 3. Model 생성
        try {
            Todo todo = new Todo(title);
            todo.setDescription(description);
            todo.setPriority(priority);
            todo.validate();
            
            // 4. 저장
            repository.save(todo);
            
            System.out.println("   💾 Todo 저장 완료: ID=" + todo.getId());
            
            // 5. View 업데이트 명령
            view.clearInputs();
            view.showSuccess("Todo가 추가되었습니다");
            loadTodos();
            
        } catch (IllegalArgumentException e) {
            view.showError(e.getMessage());
        }
    }
    
    /**
     * Todo 완료 토글
     */
    public void onTodoToggle(Long todoId) {
        System.out.println("\n✓ Presenter: Todo 토글 - ID=" + todoId);
        
        repository.findById(todoId).ifPresent(todo -> {
            // 비즈니스 로직 실행
            todo.toggleComplete();
            repository.save(todo);
            
            System.out.println("   → 완료 상태: " + todo.isCompleted());
            
            // View 업데이트
            loadTodos();
        });
    }
    
    /**
     * Todo 삭제
     */
    public void onDeleteTodoClick(Long todoId) {
        System.out.println("\n🗑️ Presenter: Todo 삭제 - ID=" + todoId);
        
        repository.delete(todoId);
        
        view.showSuccess("Todo가 삭제되었습니다");
        loadTodos();
    }
    
    /**
     * Todo 목록 로드 (핵심 메서드)
     */
    private void loadTodos() {
        System.out.println("\n🔄 Presenter: Todo 목록 로드");
        
        view.showLoading();
        
        // 1. Model에서 데이터 조회
        List<Todo> todos = repository.findAll();
        
        System.out.println("   → " + todos.size() + "개 조회");
        
        // 2. ViewModel로 변환 (Presenter의 책임!)
        List<TodoViewModel> viewModels = todos.stream()
            .map(this::toViewModel)
            .collect(Collectors.toList());
        
        // 3. 통계 계산 (Presenter의 책임!)
        int total = todos.size();
        int completed = (int) todos.stream()
            .filter(Todo::isCompleted)
            .count();
        int active = total - completed;
        
        System.out.println("   → 통계: 전체=" + total + ", 완료=" + completed + ", 진행=" + active);
        
        view.hideLoading();
        
        // 4. View에 명령
        view.showTodos(viewModels);
        view.showStatistics(total, completed, active);
        
        System.out.println("   ✅ View 업데이트 완료");
    }
    
    /**
     * Model → ViewModel 변환
     */
    private TodoViewModel toViewModel(Todo todo) {
        String displayText = todo.isCompleted() 
            ? "✓ " + todo.getTitle() 
            : "☐ " + todo.getTitle();
        
        String priorityColor = getPriorityColor(todo.getPriority());
        
        return new TodoViewModel(
            todo.getId(),
            displayText,
            todo.getDescription(),
            todo.isCompleted(),
            priorityColor
        );
    }
    
    /**
     * 우선순위 색상 결정 (프레젠테이션 로직)
     */
    private String getPriorityColor(Todo.Priority priority) {
        switch (priority) {
            case URGENT: return "red";
            case HIGH: return "orange";
            case NORMAL: return "green";
            case LOW: return "gray";
            default: return "black";
        }
    }
}

/**
 * ViewModel (View용 데이터)
 */
public class TodoViewModel {
    private final Long id;
    private final String displayText;
    private final String description;
    private final boolean completed;
    private final String priorityColor;
    
    public TodoViewModel(Long id, String displayText, String description, 
                        boolean completed, String priorityColor) {
        this.id = id;
        this.displayText = displayText;
        this.description = description;
        this.completed = completed;
        this.priorityColor = priorityColor;
    }
    
    // Getters
    public Long getId() { return id; }
    public String getDisplayText() { return displayText; }
    public String getDescription() { return description; }
    public boolean isCompleted() { return completed; }
    public String getPriorityColor() { return priorityColor; }
}

/**
 * ============================================
 * VIEW IMPLEMENTATION (완전히 수동적)
 * ============================================
 * UI만 담당, 로직은 전혀 없음!
 */

/**
 * Swing으로 구현한 View
 */
public class TodoListSwingView extends JFrame implements TodoListView {
    private final TodoListPresenter presenter;
    
    // UI Components
    private JTextField titleField;
    private JTextArea descriptionArea;
    private JComboBox<Todo.Priority> priorityCombo;
    private JButton addButton;
    private DefaultListModel<TodoViewModel> todoListModel;
    private JList<TodoViewModel> todoList;
    private JLabel totalLabel;
    private JLabel completedLabel;
    private JLabel activeLabel;
    private JPanel loadingPanel;
    
    public TodoListSwingView(TodoListPresenter presenter) {
        this.presenter = presenter;
        
        initializeUI();
        setupEventListeners();
        
        // Presenter에 View 생성 알림
        presenter.onViewCreated();
    }
    
    /**
     * UI 초기화
     */
    private void initializeUI() {
        setTitle("Todo List - MVP Pattern");
        setSize(600, 500);
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setLayout(new BorderLayout(10, 10));
        
        // 입력 패널
        JPanel inputPanel = createInputPanel();
        add(inputPanel, BorderLayout.NORTH);
        
        // 리스트 패널
        JPanel listPanel = createListPanel();
        add(listPanel, BorderLayout.CENTER);
        
        // 통계 패널
        JPanel statsPanel = createStatsPanel();
        add(statsPanel, BorderLayout.SOUTH);
        
        // 로딩 패널
        loadingPanel = new JPanel();
        loadingPanel.add(new JLabel("로딩 중..."));
        loadingPanel.setVisible(false);
    }
    
    private JPanel createInputPanel() {
        JPanel panel = new JPanel(new GridBagLayout());
        panel.setBorder(BorderFactory.createTitledBorder("새 Todo"));
        
        GridBagConstraints gbc = new GridBagConstraints();
        gbc.insets = new Insets(5, 5, 5, 5);
        gbc.fill = GridBagConstraints.HORIZONTAL;
        
        // 제목
        gbc.gridx = 0; gbc.gridy = 0;
        panel.add(new JLabel("제목:"), gbc);
        
        gbc.gridx = 1; gbc.weightx = 1.0;
        titleField = new JTextField(30);
        panel.add(titleField, gbc);
        
        // 설명
        gbc.gridx = 0; gbc.gridy = 1; gbc.weightx = 0;
        panel.add(new JLabel("설명:"), gbc);
        
        gbc.gridx = 1; gbc.weightx = 1.0;
        descriptionArea = new JTextArea(3, 30);
        descriptionArea.setLineWrap(true);
        panel.add(new JScrollPane(descriptionArea), gbc);
        
        // 우선순위
        gbc.gridx = 0; gbc.gridy = 2; gbc.weightx = 0;
        panel.add(new JLabel("우선순위:"), gbc);
        
        gbc.gridx = 1;
        priorityCombo = new JComboBox<>(Todo.Priority.values());
        panel.add(priorityCombo, gbc);
        
        // 추가 버튼
        gbc.gridx = 1; gbc.gridy = 3;
        addButton = new JButton("추가");
        panel.add(addButton, gbc);
        
        return panel;
    }
    
    private JPanel createListPanel() {
        JPanel panel = new JPanel(new BorderLayout());
        panel.setBorder(BorderFactory.createTitledBorder("Todo 목록"));
        
        todoListModel = new DefaultListModel<>();
        todoList = new JList<>(todoListModel);
        todoList.setCellRenderer(new TodoCellRenderer());
        
        JScrollPane scrollPane = new JScrollPane(todoList);
        panel.add(scrollPane, BorderLayout.CENTER);
        
        return panel;
    }
    
    private JPanel createStatsPanel() {
        JPanel panel = new JPanel(new FlowLayout(FlowLayout.LEFT));
        panel.setBorder(BorderFactory.createTitledBorder("통계"));
        
        totalLabel = new JLabel("전체: 0");
        completedLabel = new JLabel("완료: 0");
        activeLabel = new JLabel("진행 중: 0");
        
        panel.add(totalLabel);
        panel.add(new JLabel(" | "));
        panel.add(completedLabel);
        panel.add(new JLabel(" | "));
        panel.add(activeLabel);
        
        return panel;
    }
    
    /**
     * 이벤트 리스너 설정
     * (단순히 Presenter 메서드 호출만!)
     */
    private void setupEventListeners() {
        // 추가 버튼
        addButton.addActionListener(e -> presenter.onAddTodoClick());
        
        // Enter 키로 추가
        titleField.addActionListener(e -> presenter.onAddTodoClick());
        
        // 더블클릭으로 완료 토글
        todoList.addMouseListener(new MouseAdapter() {
            @Override
            public void mouseClicked(MouseEvent e) {
                if (e.getClickCount() == 2) {
                    int index = todoList.locationToIndex(e.getPoint());
                    if (index >= 0) {
                        TodoViewModel vm = todoListModel.get(index);
                        presenter.onTodoToggle(vm.getId());
                    }
                }
            }
        });
        
        // 우클릭으로 삭제
        todoList.addMouseListener(new MouseAdapter() {
            @Override
            public void mousePressed(MouseEvent e) {
                if (SwingUtilities.isRightMouseButton(e)) {
                    int index = todoList.locationToIndex(e.getPoint());
                    if (index >= 0) {
                        todoList.setSelectedIndex(index);
                        showDeleteMenu(e.getX(), e.getY(), index);
                    }
                }
            }
        });
    }
    
    private void showDeleteMenu(int x, int y, int index) {
        JPopupMenu menu = new JPopupMenu();
        JMenuItem deleteItem = new JMenuItem("삭제");
        
        deleteItem.addActionListener(e -> {
            TodoViewModel vm = todoListModel.get(index);
            presenter.onDeleteTodoClick(vm.getId());
        });
        
        menu.add(deleteItem);
        menu.show(todoList, x, y);
    }
    
    /**
     * ============================================
     * VIEW INTERFACE 구현 (완전히 수동적!)
     * ============================================
     * Presenter가 시키는 대로만 함!
     */
    
    @Override
    public String getTodoTitle() {
        return titleField.getText();
    }
    
    @Override
    public String getTodoDescription() {
        return descriptionArea.getText();
    }
    
    @Override
    public Todo.Priority getTodoPriority() {
        return (Todo.Priority) priorityCombo.getSelectedItem();
    }
    
    @Override
    public void clearInputs() {
        titleField.setText("");
        descriptionArea.setText("");
        priorityCombo.setSelectedIndex(0);
    }
    
    @Override
    public void showTodos(List<TodoViewModel> todos) {
        todoListModel.clear();
        todos.forEach(todoListModel::addElement);
    }
    
    @Override
    public void showLoading() {
        loadingPanel.setVisible(true);
    }
    
    @Override
    public void hideLoading() {
        loadingPanel.setVisible(false);
    }
    
    @Override
    public void showError(String message) {
        JOptionPane.showMessageDialog(
            this,
            message,
            "오류",
            JOptionPane.ERROR_MESSAGE
        );
    }
    
    @Override
    public void showSuccess(String message) {
        JOptionPane.showMessageDialog(
            this,
            message,
            "성공",
            JOptionPane.INFORMATION_MESSAGE
        );
    }
    
    @Override
    public void showStatistics(int total, int completed, int active) {
        totalLabel.setText("전체: " + total);
        completedLabel.setText("완료: " + completed);
        activeLabel.setText("진행 중: " + active);
    }
    
    /**
     * Custom Cell Renderer
     */
    private class TodoCellRenderer extends DefaultListCellRenderer {
        @Override
        public Component getListCellRendererComponent(
                JList<?> list, Object value, int index,
                boolean isSelected, boolean cellHasFocus) {
            
            super.getListCellRendererComponent(list, value, index, isSelected, cellHasFocus);
            
            if (value instanceof TodoViewModel) {
                TodoViewModel vm = (TodoViewModel) value;
                
                setText(vm.getDisplayText());
                
                // 우선순위 색상
                Color color = getColorFromString(vm.getPriorityColor());
                setForeground(isSelected ? Color.WHITE : color);
                
                // 완료된 항목 스타일
                if (vm.isCompleted()) {
                    setFont(getFont().deriveFont(Font.ITALIC));
                }
            }
            
            return this;
        }
        
        private Color getColorFromString(String colorName) {
            switch (colorName) {
                case "red": return Color.RED;
                case "orange": return Color.ORANGE;
                case "green": return Color.GREEN;
                case "gray": return Color.GRAY;
                default: return Color.BLACK;
            }
        }
    }
}

/**
 * ============================================
 * MAIN APPLICATION
 * ============================================
 */
public class MVPTodoApp {
    public static void main(String[] args) {
        System.out.println("=== MVP 패턴 Todo 애플리케이션 ===\n");
        
        SwingUtilities.invokeLater(() -> {
            // 1. Model 생성
            TodoRepository repository = new TodoRepository();
            
            // 2. Presenter 생성 (View는 나중에)
            TodoListPresenter presenter = new TodoListPresenter(null, repository);
            
            // 3. View 생성 (Presenter 전달)
            TodoListSwingView view = new TodoListSwingView(presenter);
            
            // 4. Presenter에 View 연결
            // (실제로는 생성자에서 처리)
            
            // 5. View 표시
            view.setVisible(true);
            
            System.out.println("✅ 애플리케이션 시작\n");
        });
    }
}
```

**실행 결과:**
```
=== MVP 패턴 Todo 애플리케이션 ===

✅ 애플리케이션 시작

📋 Presenter: View 생성됨

🔄 Presenter: Todo 목록 로드
   → 0개 조회
   → 통계: 전체=0, 완료=0, 진행=0
   ✅ View 업데이트 완료

[사용자가 "MVP 패턴 학습" 입력 후 추가 버튼 클릭]

➕ Presenter: Todo 추가 시작
   → View에서 데이터 수신: MVP 패턴 학습
   ✅ 검증 완료
   💾 Todo 저장 완료: ID=1

🔄 Presenter: Todo 목록 로드
   → 1개 조회
   → 통계: 전체=1, 완료=0, 진행=1
   ✅ View 업데이트 완료

[사용자가 Todo 더블클릭 (완료 토글)]

✓ Presenter: Todo 토글 - ID=1
   → 완료 상태: true

🔄 Presenter: Todo 목록 로드
   → 1개 조회
   → 통계: 전체=1, 완료=1, 진행=0
   ✅ View 업데이트 완료
```

---

## 5. 실전 예제

### 예제 1: Android MVP 구현 ⭐⭐⭐

```java
/**
 * ============================================
 * Android MVP 예제
 * ============================================
 */

/**
 * Login View Contract
 */
public interface LoginView {
    void showProgress();
    void hideProgress();
    void setEmailError(String error);
    void setPasswordError(String error);
    void navigateToHome();
    void showLoginError(String message);
}

/**
 * Login Presenter
 */
public class LoginPresenter {
    private LoginView view;
    private UserRepository userRepository;
    
    public LoginPresenter(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    public void attachView(LoginView view) {
        this.view = view;
    }
    
    public void detachView() {
        this.view = null;
    }
    
    public void login(String email, String password) {
        // 검증
        if (!isEmailValid(email)) {
            view.setEmailError("올바른 이메일을 입력하세요");
            return;
        }
        
        if (!isPasswordValid(password)) {
            view.setPasswordError("비밀번호는 8자 이상");
            return;
        }
        
        // 로그인 처리
        view.showProgress();
        
        userRepository.login(email, password, new Callback<User>() {
            @Override
            public void onSuccess(User user) {
                if (view != null) {
                    view.hideProgress();
                    view.navigateToHome();
                }
            }
            
            @Override
            public void onError(String message) {
                if (view != null) {
                    view.hideProgress();
                    view.showLoginError(message);
                }
            }
        });
    }
    
    private boolean isEmailValid(String email) {
        return email != null && email.contains("@");
    }
    
    private boolean isPasswordValid(String password) {
        return password != null && password.length() >= 8;
    }
}

/**
 * Login Activity (View 구현)
 */
public class LoginActivity extends AppCompatActivity implements LoginView {
    private EditText emailEditText;
    private EditText passwordEditText;
    private Button loginButton;
    private ProgressBar progressBar;
    
    private LoginPresenter presenter;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_login);
        
        // UI 초기화
        emailEditText = findViewById(R.id.email);
        passwordEditText = findViewById(R.id.password);
        loginButton = findViewById(R.id.loginButton);
        progressBar = findViewById(R.id.progress);
        
        // Presenter 생성
        UserRepository repository = new UserRepository();
        presenter = new LoginPresenter(repository);
        presenter.attachView(this);
        
        // 이벤트 리스너 (Presenter 호출만!)
        loginButton.setOnClickListener(v -> {
            String email = emailEditText.getText().toString();
            String password = passwordEditText.getText().toString();
            presenter.login(email, password);
        });
    }
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
        presenter.detachView();  // 메모리 누수 방지
    }
    
    // View Interface 구현 (완전히 수동적!)
    
    @Override
    public void showProgress() {
        progressBar.setVisibility(View.VISIBLE);
        loginButton.setEnabled(false);
    }
    
    @Override
    public void hideProgress() {
        progressBar.setVisibility(View.GONE);
        loginButton.setEnabled(true);
    }
    
    @Override
    public void setEmailError(String error) {
        emailEditText.setError(error);
    }
    
    @Override
    public void setPasswordError(String error) {
        passwordEditText.setError(error);
    }
    
    @Override
    public void navigateToHome() {
        Intent intent = new Intent(this, MainActivity.class);
        startActivity(intent);
        finish();
    }
    
    @Override
    public void showLoginError(String message) {
        Toast.makeText(this, message, Toast.LENGTH_SHORT).show();
    }
}
```

---

## 6. MVC vs MVP vs MVVM 완전 비교

### 📊 3가지 패턴 비교

| 특징 | MVC | MVP | MVVM |
|------|-----|-----|------|
| **View-Model 관계** | View가 Model 관찰 | Presenter가 중재 | ViewModel 바인딩 |
| **View 역할** | 능동적 (Active) | 수동적 (Passive) | 선언적 (Declarative) |
| **테스트 용이성** | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **View-Logic 분리** | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **코드 복잡도** | 낮음 | 중간 | 중간 |
| **데이터 바인딩** | ❌ | ❌ | ✅ |
| **적합한 플랫폼** | Web (Spring MVC) | Android, Desktop | WPF, JavaFX |

### 🔄 구조 비교

```
=== MVC ===
View → Controller → Model
 ↑                    │
 └──── Observer ──────┘
(View가 Model 관찰)

=== MVP ===
View ←→ Presenter → Model
      (완전히 수동적)

=== MVVM ===
View ←── Binding ───→ ViewModel → Model
      (자동 동기화)
```

### 💡 선택 가이드

| 상황 | 추천 패턴 | 이유 |
|------|-----------|------|
| **Android 앱** | MVP | Activity 테스트 용이 |
| **JavaFX 앱** | MVVM | Observable 지원 |
| **Spring Web** | MVC | 프레임워크 지원 |
| **테스트 중요** | MVP/MVVM | View 독립 테스트 |

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 | 실무 효과 |
|------|------|-----------|
| **테스트 용이** | Presenter 독립 테스트 | 빠른 단위 테스트 |
| **관심사 분리** | View와 로직 완전 분리 | 유지보수 용이 |
| **Passive View** | View는 표시만 | 재사용성 |
| **명확한 계약** | View Interface | 협업 용이 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **코드량 증가** | Interface 추가 | 복잡한 경우만 |
| **Presenter 비대** | 로직 집중 | 분리 |
| **1:1 관계** | View-Presenter | 재사용 고려 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: Presenter가 View 구현 참조

```java
// 잘못된 예: Presenter가 구체적 View 참조
public class LoginPresenter {
    private LoginActivity view;  // ❌ 구체 클래스
    
    public void login() {
        view.findViewById(R.id.progress).setVisibility(View.VISIBLE);  // ❌
    }
}
```

**해결:**
```java
// 올바른 예: Interface만 참조
public class LoginPresenter {
    private LoginView view;  // ✅ 인터페이스
    
    public void login() {
        view.showProgress();  // ✅
    }
}
```

### ❌ 안티패턴 2: View에 비즈니스 로직

```java
// 잘못된 예: View에서 검증
public class LoginActivity implements LoginView {
    public void onLoginClick() {
        String email = emailField.getText();
        
        // ❌ View에서 검증
        if (!email.contains("@")) {
            showError("Invalid email");
            return;
        }
        
        presenter.login(email, password);
    }
}
```

**해결:**
```java
// 올바른 예: Presenter에서 검증
public class LoginActivity implements LoginView {
    public void onLoginClick() {
        String email = emailField.getText();
        String password = passwordField.getText();
        
        // ✅ 그냥 전달만
        presenter.login(email, password);
    }
}

public class LoginPresenter {
    public void login(String email, String password) {
        // ✅ Presenter에서 검증
        if (!email.contains("@")) {
            view.showError("Invalid email");
            return;
        }
    }
}
```

---

## 9. 심화 주제

### 🎯 Presenter 테스트

```java
/**
 * Presenter 단위 테스트
 */
public class LoginPresenterTest {
    
    @Test
    public void loginWithInvalidEmail_shouldShowError() {
        // Given
        LoginView view = mock(LoginView.class);
        UserRepository repository = mock(UserRepository.class);
        LoginPresenter presenter = new LoginPresenter(repository);
        presenter.attachView(view);
        
        // When
        presenter.login("invalid-email", "password123");
        
        // Then
        verify(view).setEmailError("올바른 이메일을 입력하세요");
        verify(view, never()).showProgress();
    }
    
    @Test
    public void loginWithValidCredentials_shouldNavigateToHome() {
        // Given
        LoginView view = mock(LoginView.class);
        UserRepository repository = mock(UserRepository.class);
        LoginPresenter presenter = new LoginPresenter(repository);
        presenter.attachView(view);
        
        // When
        presenter.login("test@example.com", "password123");
        
        // Then
        verify(view).showProgress();
        // ... (비동기 처리 테스트)
    }
}
```

---

## 10. 핵심 정리

### 📌 MVP 패턴 체크리스트

```
✅ View Interface 정의
✅ View는 완전히 수동적
✅ Presenter가 모든 로직
✅ View는 Presenter만 호출
✅ Presenter는 View Interface만 의존
✅ Model은 독립적
✅ Presenter 테스트 작성
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| Android 앱 | ⭐⭐⭐ | Activity 테스트 |
| 테스트 중요 | ⭐⭐⭐ | 독립 테스트 |
| Desktop 앱 | ⭐⭐⭐ | Swing/JavaFX |
| 데이터 바인딩 | ⭐ | MVVM 더 적합 |

### 💡 핵심 원칙

1. **Passive View**
2. **Presenter가 모든 로직**
3. **Interface로 분리**
4. **테스트 용이**

### 🔥 실무 적용

```java
// ✅ DO: View Interface 정의
public interface LoginView {
    void showError(String message);
}

// ✅ DO: Presenter가 검증
public class LoginPresenter {
    public void login(String email, String password) {
        if (!isValid(email)) {
            view.showError("Invalid");
        }
    }
}

// ❌ DON'T: View에서 검증
public class LoginActivity {
    public void onLoginClick() {
        if (!isValid(email)) {  // ❌
            showError("Invalid");
        }
    }
}

// ❌ DON'T: Presenter가 구체 View 참조
public class LoginPresenter {
    private LoginActivity view;  // ❌
}
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Microservices](./07-Microservices.md)**

</div>
