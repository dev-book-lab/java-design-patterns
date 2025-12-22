# MVVM Pattern (Model-View-ViewModel 패턴)

> **"데이터 바인딩으로 View와 ViewModel을 자동 동기화하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [MVC vs MVP vs MVVM](#6-mvc-vs-mvp-vs-mvvm)
7. [장단점](#7-장단점)
8. [안티패턴](#8-안티패턴)
9. [심화 주제](#9-심화-주제)
10. [핵심 정리](#10-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: View가 Model을 직접 업데이트
public class UserView extends JPanel {
    private JTextField nameField;
    private JButton saveButton;
    private UserModel model;  // View가 Model 직접 참조
    
    public UserView(UserModel model) {
        this.model = model;
        
        saveButton.addActionListener(e -> {
            // 😱 View가 Model을 직접 수정!
            model.setName(nameField.getText());
            model.save();
            
            // 다른 View는 변경을 모름!
            // 수동으로 업데이트 필요!
        });
    }
}

// 문제 2: UI 업데이트를 수동으로 해야 함
public class OrderController {
    private OrderView view;
    private Order order;
    
    public void addItem(Product product) {
        // 모델 업데이트
        order.addItem(product);
        
        // 😱 View를 수동으로 업데이트!
        view.updateTotalLabel(order.getTotal());
        view.updateItemsList(order.getItems());
        view.updateSubmitButton(order.canSubmit());
        
        // 새 필드 추가 시 여기도 수정!
        // 실수로 빠뜨리면 UI가 안 맞음!
    }
}

// 문제 3: UI 로직이 View에 흩어짐
public class ProductListView extends JFrame {
    private JTextField searchField;
    private JComboBox<String> categoryCombo;
    private JTable productTable;
    private JLabel resultLabel;
    
    public ProductListView() {
        searchField.addKeyListener(new KeyAdapter() {
            @Override
            public void keyReleased(KeyEvent e) {
                // 😱 UI 로직이 View에!
                String keyword = searchField.getText();
                String category = (String) categoryCombo.getSelectedItem();
                
                // 검색 로직
                List<Product> results = searchProducts(keyword, category);
                
                // 결과 표시
                updateTable(results);
                
                // 개수 표시
                resultLabel.setText(results.size() + "개 상품");
                
                // 비즈니스 로직과 UI 로직이 섞임!
            }
        });
    }
}

// 문제 4: 테스트 불가능
public class LoginView extends JDialog {
    private JTextField emailField;
    private JPasswordField passwordField;
    private JButton loginButton;
    
    public LoginView() {
        loginButton.addActionListener(e -> {
            String email = emailField.getText();
            String password = new String(passwordField.getPassword());
            
            // 😱 검증 로직이 View에!
            if (email.isEmpty()) {
                JOptionPane.showMessageDialog(this, "이메일을 입력하세요");
                return;
            }
            
            if (!email.contains("@")) {
                JOptionPane.showMessageDialog(this, "올바른 이메일을 입력하세요");
                return;
            }
            
            if (password.length() < 8) {
                JOptionPane.showMessageDialog(this, "비밀번호는 8자 이상");
                return;
            }
            
            // 로그인 처리
            doLogin(email, password);
        });
        
        // 어떻게 테스트?
        // - JOptionPane Mock 필요
        // - JTextField Mock 필요
        // - UI 없이 검증 로직만 테스트 불가!
    }
}

// 문제 5: 복잡한 상태 관리
public class ShoppingCartView extends JPanel {
    private JList<CartItem> itemList;
    private JLabel totalLabel;
    private JLabel discountLabel;
    private JButton checkoutButton;
    
    private List<CartItem> items = new ArrayList<>();
    private double total = 0;
    private double discount = 0;
    private boolean canCheckout = false;
    
    public void addItem(Product product) {
        CartItem item = new CartItem(product);
        items.add(item);
        
        // 😱 상태 동기화를 수동으로!
        updateTotal();
        updateDiscount();
        updateCheckoutButton();
        updateItemList();
        
        // 하나라도 빠뜨리면 버그!
    }
    
    private void updateTotal() {
        total = items.stream()
            .mapToDouble(CartItem::getPrice)
            .sum();
        totalLabel.setText(String.format("%.0f원", total));
    }
    
    private void updateDiscount() {
        if (total >= 100000) {
            discount = total * 0.1;
        } else {
            discount = 0;
        }
        discountLabel.setText(String.format("%.0f원", discount));
    }
    
    private void updateCheckoutButton() {
        canCheckout = !items.isEmpty() && total > 0;
        checkoutButton.setEnabled(canCheckout);
    }
    
    private void updateItemList() {
        DefaultListModel<CartItem> model = new DefaultListModel<>();
        items.forEach(model::addElement);
        itemList.setModel(model);
    }
    
    // 복잡하고 오류 발생 쉬움!
}

// 문제 6: View와 비즈니스 로직의 강한 결합
public class ReportView extends JFrame {
    private JComboBox<String> periodCombo;
    private JTable dataTable;
    
    public ReportView() {
        periodCombo.addActionListener(e -> {
            String period = (String) periodCombo.getSelectedItem();
            
            // 😱 비즈니스 로직이 View에!
            LocalDate startDate;
            LocalDate endDate = LocalDate.now();
            
            switch (period) {
                case "오늘":
                    startDate = LocalDate.now();
                    break;
                case "이번 주":
                    startDate = LocalDate.now().with(DayOfWeek.MONDAY);
                    break;
                case "이번 달":
                    startDate = LocalDate.now().withDayOfMonth(1);
                    break;
                default:
                    startDate = LocalDate.now().minusMonths(1);
            }
            
            // DB 조회
            List<Report> reports = fetchReports(startDate, endDate);
            
            // 통계 계산
            double total = reports.stream()
                .mapToDouble(Report::getAmount)
                .sum();
            
            // 표시
            updateTable(reports);
            
            // View와 비즈니스 로직이 분리 안 됨!
        });
    }
}
```

### ⚡ 핵심 문제

1. **수동 동기화**: View와 Model 상태를 수동으로 동기화
2. **강한 결합**: View가 Model을 직접 참조
3. **테스트 어려움**: UI 로직과 비즈니스 로직 분리 안 됨
4. **코드 중복**: 상태 업데이트 코드가 반복
5. **오류 발생**: 동기화 누락 시 버그
6. **유지보수 어려움**: 변경 시 여러 곳 수정

---

## 2. 패턴 정의

### 📖 정의

> **View와 Model 사이에 ViewModel을 두어 데이터 바인딩을 통해 자동으로 동기화하고, View의 상태와 로직을 ViewModel로 분리하는 패턴**

### 🎯 목적

- **자동 동기화**: 데이터 바인딩으로 View ↔ ViewModel 자동 동기화
- **관심사 분리**: UI 로직을 ViewModel로 분리
- **테스트 용이**: ViewModel을 독립적으로 테스트
- **재사용성**: ViewModel을 여러 View에서 재사용

### 💡 핵심 아이디어

```java
// Before: View가 Model 직접 조작
public class UserView {
    private User user;  // Model 직접 참조
    
    public void onSaveClick() {
        user.setName(nameField.getText());  // 수동 동기화
        updateUI();  // 수동 업데이트
    }
}

// After: ViewModel + 데이터 바인딩
public class UserViewModel {
    // Observable 프로퍼티 (자동 알림)
    private StringProperty name = new SimpleStringProperty();
    private BooleanProperty canSave = new SimpleBooleanProperty();
    
    public UserViewModel() {
        // name이 변경되면 자동으로 canSave 재계산
        name.addListener((obs, old, newVal) -> {
            canSave.set(isValid(newVal));
        });
    }
    
    public StringProperty nameProperty() { return name; }
    public BooleanProperty canSaveProperty() { return canSave; }
}

public class UserView {
    private UserViewModel viewModel;
    
    public UserView(UserViewModel viewModel) {
        this.viewModel = viewModel;
        
        // 데이터 바인딩 (자동 동기화!)
        nameField.textProperty().bindBidirectional(viewModel.nameProperty());
        saveButton.disableProperty().bind(viewModel.canSaveProperty().not());
        
        // 이제 수동 동기화 불필요!
    }
}
```

---

## 3. 구조와 구성요소

### 📊 MVVM 구조

```
┌─────────────────────────────────────┐
│            View                     │
│  - UI Components                    │
│  - FXML / Layout                    │
│  - Event Handlers (최소)             │
└─────────────────────────────────────┘
              ↕ Data Binding
              ↕ (자동 동기화)
┌─────────────────────────────────────┐
│         ViewModel                   │
│  - Observable Properties            │
│  - Commands                         │
│  - Presentation Logic               │
│  - Validation                       │
└─────────────────────────────────────┘
              │
              │ uses (단방향)
              ▼
┌─────────────────────────────────────┐
│           Model                     │
│  - Business Logic                   │
│  - Domain Entities                  │
│  - Data Access                      │
└─────────────────────────────────────┘
```

### 🔄 데이터 흐름

```
User Input
    │
    ▼
┌─────────┐
│  View   │
└─────────┘
    │
    │ Data Binding (자동)
    ▼
┌──────────────┐
│ ViewModel    │
└──────────────┘
    │
    │ Property Change (자동)
    ▼
┌──────────────┐
│ ViewModel    │ → Validation
│              │ → Business Logic
└──────────────┘
    │
    │ Model Update
    ▼
┌─────────┐
│  Model  │
└─────────┘
    │
    │ Data Changed
    ▼
┌──────────────┐
│ ViewModel    │ ← Observable 업데이트
└──────────────┘
    │
    │ Property Change (자동)
    ▼
┌─────────┐
│  View   │ ← 자동으로 UI 업데이트!
└─────────┘
```

### 🔧 구성요소

| 컴포넌트 | 역할 | 책임 | 예시 |
|---------|------|------|------|
| **View** | UI 표현 | - UI 렌더링<br>- 데이터 바인딩<br>- 사용자 입력 수집 | FXML, JavaFX |
| **ViewModel** | Presentation 로직 | - Observable Properties<br>- Commands<br>- Validation<br>- Formatting | `UserViewModel` |
| **Model** | 비즈니스 로직 | - Domain Logic<br>- Data Access<br>- Business Rules | `User`, `UserService` |

---

## 4. 구현 방법

### 완전한 구현: JavaFX Todo 애플리케이션 ⭐⭐⭐

```java
/**
 * ============================================
 * MODEL (도메인 모델)
 * ============================================
 */

/**
 * Domain Entity: Todo
 */
public class Todo {
    private Long id;
    private String title;
    private String description;
    private boolean completed;
    private LocalDateTime createdAt;
    private Priority priority;
    
    public enum Priority {
        LOW, NORMAL, HIGH, URGENT
    }
    
    public Todo(String title) {
        this.title = title;
        this.completed = false;
        this.createdAt = LocalDateTime.now();
        this.priority = Priority.NORMAL;
    }
    
    /**
     * 비즈니스 로직: 완료 토글
     */
    public void toggleComplete() {
        this.completed = !this.completed;
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
    
    // Getters, Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
    public boolean isCompleted() { return completed; }
    public void setCompleted(boolean completed) { this.completed = completed; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public Priority getPriority() { return priority; }
    public void setPriority(Priority priority) { this.priority = priority; }
}

/**
 * Service: Todo 서비스
 */
public class TodoService {
    private final TodoRepository repository;
    
    public TodoService(TodoRepository repository) {
        this.repository = repository;
    }
    
    public Todo createTodo(String title, String description, Todo.Priority priority) {
        Todo todo = new Todo(title);
        todo.setDescription(description);
        todo.setPriority(priority);
        todo.validate();
        
        return repository.save(todo);
    }
    
    public void toggleComplete(Long id) {
        Todo todo = repository.findById(id);
        todo.toggleComplete();
        repository.save(todo);
    }
    
    public List<Todo> getAllTodos() {
        return repository.findAll();
    }
    
    public void deleteTodo(Long id) {
        repository.delete(id);
    }
}

/**
 * Repository (간단한 InMemory 구현)
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
    
    public Todo findById(Long id) {
        Todo todo = storage.get(id);
        if (todo == null) {
            throw new IllegalArgumentException("Todo not found: " + id);
        }
        return todo;
    }
    
    public List<Todo> findAll() {
        return new ArrayList<>(storage.values());
    }
    
    public void delete(Long id) {
        storage.remove(id);
    }
}

/**
 * ============================================
 * VIEWMODEL (Presentation 로직)
 * ============================================
 */

/**
 * ViewModel: Todo 항목
 */
public class TodoItemViewModel {
    private final Todo todo;
    
    // Observable Properties (자동 알림)
    private final LongProperty id;
    private final StringProperty title;
    private final StringProperty description;
    private final BooleanProperty completed;
    private final ObjectProperty<Todo.Priority> priority;
    private final StringProperty displayText;
    private final StringProperty styleClass;
    
    public TodoItemViewModel(Todo todo) {
        this.todo = todo;
        
        // Observable 프로퍼티 초기화
        this.id = new SimpleLongProperty(todo.getId());
        this.title = new SimpleStringProperty(todo.getTitle());
        this.description = new SimpleStringProperty(todo.getDescription());
        this.completed = new SimpleBooleanProperty(todo.isCompleted());
        this.priority = new SimpleObjectProperty<>(todo.getPriority());
        this.displayText = new SimpleStringProperty();
        this.styleClass = new SimpleStringProperty();
        
        // 계산된 프로퍼티 설정
        setupComputedProperties();
    }
    
    /**
     * 계산된 프로퍼티 설정 (자동 업데이트)
     */
    private void setupComputedProperties() {
        // title이나 completed가 변경되면 displayText 자동 업데이트
        InvalidationListener updateDisplayText = obs -> {
            String prefix = completed.get() ? "✓ " : "☐ ";
            displayText.set(prefix + title.get());
        };
        
        title.addListener(updateDisplayText);
        completed.addListener(updateDisplayText);
        updateDisplayText.invalidated(null);  // 초기값 설정
        
        // priority에 따라 styleClass 자동 업데이트
        priority.addListener((obs, oldVal, newVal) -> {
            switch (newVal) {
                case URGENT:
                    styleClass.set("todo-urgent");
                    break;
                case HIGH:
                    styleClass.set("todo-high");
                    break;
                case NORMAL:
                    styleClass.set("todo-normal");
                    break;
                case LOW:
                    styleClass.set("todo-low");
                    break;
            }
        });
        styleClass.set("todo-normal");  // 초기값
    }
    
    /**
     * Command: 완료 토글
     */
    public void toggleComplete() {
        completed.set(!completed.get());
        todo.setCompleted(completed.get());
    }
    
    /**
     * Model 업데이트
     */
    public void updateModel() {
        todo.setTitle(title.get());
        todo.setDescription(description.get());
        todo.setCompleted(completed.get());
        todo.setPriority(priority.get());
    }
    
    // Property Getters (데이터 바인딩용)
    public LongProperty idProperty() { return id; }
    public StringProperty titleProperty() { return title; }
    public StringProperty descriptionProperty() { return description; }
    public BooleanProperty completedProperty() { return completed; }
    public ObjectProperty<Todo.Priority> priorityProperty() { return priority; }
    public StringProperty displayTextProperty() { return displayText; }
    public StringProperty styleClassProperty() { return styleClass; }
    
    public Todo getTodo() { return todo; }
}

/**
 * ViewModel: Todo 리스트
 */
public class TodoListViewModel {
    private final TodoService todoService;
    
    // Observable Collections (자동 알림)
    private final ObservableList<TodoItemViewModel> todos;
    private final ObservableList<TodoItemViewModel> filteredTodos;
    
    // Observable Properties
    private final StringProperty searchText;
    private final ObjectProperty<FilterMode> filterMode;
    private final IntegerProperty totalCount;
    private final IntegerProperty completedCount;
    private final IntegerProperty activeCount;
    private final StringProperty statusText;
    
    // Commands
    private final ObjectProperty<Consumer<String>> onError;
    
    public enum FilterMode {
        ALL, ACTIVE, COMPLETED
    }
    
    public TodoListViewModel(TodoService todoService) {
        this.todoService = todoService;
        
        // Observable 초기화
        this.todos = FXCollections.observableArrayList();
        this.filteredTodos = FXCollections.observableArrayList();
        this.searchText = new SimpleStringProperty("");
        this.filterMode = new SimpleObjectProperty<>(FilterMode.ALL);
        this.totalCount = new SimpleIntegerProperty(0);
        this.completedCount = new SimpleIntegerProperty(0);
        this.activeCount = new SimpleIntegerProperty(0);
        this.statusText = new SimpleStringProperty();
        this.onError = new SimpleObjectProperty<>();
        
        // 자동 필터링 설정
        setupAutoFiltering();
        
        // 자동 통계 계산
        setupAutoStatistics();
        
        // 초기 데이터 로드
        loadTodos();
    }
    
    /**
     * 자동 필터링 설정
     */
    private void setupAutoFiltering() {
        // searchText나 filterMode가 변경되면 자동으로 필터링
        InvalidationListener updateFilter = obs -> applyFilter();
        searchText.addListener(updateFilter);
        filterMode.addListener(updateFilter);
        
        // todos가 변경되어도 자동 필터링
        todos.addListener((ListChangeListener<TodoItemViewModel>) c -> applyFilter());
    }
    
    /**
     * 자동 통계 계산 설정
     */
    private void setupAutoStatistics() {
        // todos가 변경되면 자동으로 통계 재계산
        todos.addListener((ListChangeListener<TodoItemViewModel>) c -> {
            totalCount.set(todos.size());
            
            long completed = todos.stream()
                .filter(vm -> vm.completedProperty().get())
                .count();
            completedCount.set((int) completed);
            
            activeCount.set(todos.size() - (int) completed);
            
            statusText.set(String.format(
                "전체 %d개 | 완료 %d개 | 진행 중 %d개",
                totalCount.get(),
                completedCount.get(),
                activeCount.get()
            ));
        });
    }
    
    /**
     * 필터 적용
     */
    private void applyFilter() {
        filteredTodos.clear();
        
        todos.stream()
            .filter(this::matchesFilter)
            .forEach(filteredTodos::add);
    }
    
    /**
     * 필터 조건 확인
     */
    private boolean matchesFilter(TodoItemViewModel vm) {
        // 검색어 필터
        String search = searchText.get().toLowerCase();
        if (!search.isEmpty()) {
            String title = vm.titleProperty().get().toLowerCase();
            if (!title.contains(search)) {
                return false;
            }
        }
        
        // 상태 필터
        switch (filterMode.get()) {
            case ACTIVE:
                return !vm.completedProperty().get();
            case COMPLETED:
                return vm.completedProperty().get();
            case ALL:
            default:
                return true;
        }
    }
    
    /**
     * Command: Todo 추가
     */
    public void addTodo(String title, String description, Todo.Priority priority) {
        try {
            Todo todo = todoService.createTodo(title, description, priority);
            TodoItemViewModel vm = new TodoItemViewModel(todo);
            todos.add(vm);
            
            System.out.println("✅ Todo 추가: " + title);
            
        } catch (Exception e) {
            if (onError.get() != null) {
                onError.get().accept(e.getMessage());
            }
        }
    }
    
    /**
     * Command: Todo 삭제
     */
    public void deleteTodo(TodoItemViewModel vm) {
        try {
            todoService.deleteTodo(vm.idProperty().get());
            todos.remove(vm);
            
            System.out.println("🗑️ Todo 삭제: " + vm.titleProperty().get());
            
        } catch (Exception e) {
            if (onError.get() != null) {
                onError.get().accept(e.getMessage());
            }
        }
    }
    
    /**
     * Command: Todo 완료 토글
     */
    public void toggleComplete(TodoItemViewModel vm) {
        try {
            todoService.toggleComplete(vm.idProperty().get());
            vm.toggleComplete();
            
            System.out.println("✓ 완료 토글: " + vm.titleProperty().get());
            
        } catch (Exception e) {
            if (onError.get() != null) {
                onError.get().accept(e.getMessage());
            }
        }
    }
    
    /**
     * 데이터 로드
     */
    public void loadTodos() {
        List<Todo> todoList = todoService.getAllTodos();
        
        todos.clear();
        todoList.stream()
            .map(TodoItemViewModel::new)
            .forEach(todos::add);
        
        System.out.println("📋 Todo 로드: " + todos.size() + "개");
    }
    
    // Property Getters
    public ObservableList<TodoItemViewModel> getTodos() { return todos; }
    public ObservableList<TodoItemViewModel> getFilteredTodos() { return filteredTodos; }
    public StringProperty searchTextProperty() { return searchText; }
    public ObjectProperty<FilterMode> filterModeProperty() { return filterMode; }
    public IntegerProperty totalCountProperty() { return totalCount; }
    public IntegerProperty completedCountProperty() { return completedCount; }
    public IntegerProperty activeCountProperty() { return activeCount; }
    public StringProperty statusTextProperty() { return statusText; }
    public ObjectProperty<Consumer<String>> onErrorProperty() { return onError; }
}

/**
 * ============================================
 * VIEW (JavaFX UI)
 * ============================================
 */

/**
 * View: Todo 리스트 화면
 */
public class TodoListView extends VBox {
    private final TodoListViewModel viewModel;
    
    // UI Components
    private TextField titleField;
    private TextArea descriptionArea;
    private ComboBox<Todo.Priority> priorityCombo;
    private Button addButton;
    private TextField searchField;
    private ComboBox<TodoListViewModel.FilterMode> filterCombo;
    private ListView<TodoItemViewModel> todoListView;
    private Label statusLabel;
    
    public TodoListView(TodoListViewModel viewModel) {
        this.viewModel = viewModel;
        
        initializeUI();
        setupBindings();
        setupEventHandlers();
    }
    
    /**
     * UI 초기화
     */
    private void initializeUI() {
        setPadding(new Insets(20));
        setSpacing(10);
        
        // 입력 영역
        HBox inputBox = new HBox(10);
        titleField = new TextField();
        titleField.setPromptText("할 일 제목");
        titleField.setPrefWidth(200);
        
        descriptionArea = new TextArea();
        descriptionArea.setPromptText("설명 (선택)");
        descriptionArea.setPrefRowCount(2);
        descriptionArea.setPrefWidth(300);
        
        priorityCombo = new ComboBox<>();
        priorityCombo.getItems().addAll(Todo.Priority.values());
        priorityCombo.setValue(Todo.Priority.NORMAL);
        
        addButton = new Button("추가");
        
        inputBox.getChildren().addAll(
            new Label("제목:"), titleField,
            new Label("우선순위:"), priorityCombo,
            addButton
        );
        
        // 필터 영역
        HBox filterBox = new HBox(10);
        searchField = new TextField();
        searchField.setPromptText("검색...");
        searchField.setPrefWidth(200);
        
        filterCombo = new ComboBox<>();
        filterCombo.getItems().addAll(TodoListViewModel.FilterMode.values());
        filterCombo.setValue(TodoListViewModel.FilterMode.ALL);
        
        filterBox.getChildren().addAll(
            new Label("검색:"), searchField,
            new Label("필터:"), filterCombo
        );
        
        // 리스트 영역
        todoListView = new ListView<>();
        todoListView.setPrefHeight(400);
        todoListView.setCellFactory(lv -> new TodoListCell(viewModel));
        
        // 상태 표시
        statusLabel = new Label();
        
        // 추가
        getChildren().addAll(
            new Label("📝 Todo List"),
            new Separator(),
            inputBox,
            descriptionArea,
            new Separator(),
            filterBox,
            todoListView,
            new Separator(),
            statusLabel
        );
    }
    
    /**
     * 데이터 바인딩 설정 (핵심!)
     */
    private void setupBindings() {
        // 양방향 바인딩
        searchField.textProperty().bindBidirectional(
            viewModel.searchTextProperty()
        );
        
        filterCombo.valueProperty().bindBidirectional(
            viewModel.filterModeProperty()
        );
        
        // 단방향 바인딩 (ViewModel → View)
        todoListView.itemsProperty().bind(
            new SimpleObjectProperty<>(viewModel.getFilteredTodos())
        );
        
        statusLabel.textProperty().bind(
            viewModel.statusTextProperty()
        );
        
        // 에러 핸들러 바인딩
        viewModel.onErrorProperty().set(this::showError);
    }
    
    /**
     * 이벤트 핸들러 설정
     */
    private void setupEventHandlers() {
        // 추가 버튼
        addButton.setOnAction(e -> {
            String title = titleField.getText();
            String description = descriptionArea.getText();
            Todo.Priority priority = priorityCombo.getValue();
            
            if (title.trim().isEmpty()) {
                showError("제목을 입력하세요");
                return;
            }
            
            // ViewModel 커맨드 호출
            viewModel.addTodo(title, description, priority);
            
            // 입력 필드 초기화
            titleField.clear();
            descriptionArea.clear();
            priorityCombo.setValue(Todo.Priority.NORMAL);
        });
        
        // Enter 키로 추가
        titleField.setOnAction(e -> addButton.fire());
    }
    
    /**
     * 에러 표시
     */
    private void showError(String message) {
        Alert alert = new Alert(Alert.AlertType.ERROR);
        alert.setTitle("오류");
        alert.setHeaderText(null);
        alert.setContentText(message);
        alert.showAndWait();
    }
}

/**
 * Custom ListCell (데이터 바인딩)
 */
class TodoListCell extends ListCell<TodoItemViewModel> {
    private final TodoListViewModel listViewModel;
    private HBox container;
    private CheckBox completedCheckBox;
    private Label titleLabel;
    private Label priorityLabel;
    private Button deleteButton;
    
    public TodoListCell(TodoListViewModel listViewModel) {
        this.listViewModel = listViewModel;
        
        // UI 구성
        completedCheckBox = new CheckBox();
        titleLabel = new Label();
        priorityLabel = new Label();
        deleteButton = new Button("삭제");
        deleteButton.setStyle("-fx-background-color: #ff4444; -fx-text-fill: white;");
        
        Region spacer = new Region();
        HBox.setHgrow(spacer, Priority.ALWAYS);
        
        container = new HBox(10);
        container.setAlignment(Pos.CENTER_LEFT);
        container.setPadding(new Insets(5));
        container.getChildren().addAll(
            completedCheckBox,
            titleLabel,
            spacer,
            priorityLabel,
            deleteButton
        );
    }
    
    @Override
    protected void updateItem(TodoItemViewModel item, boolean empty) {
        super.updateItem(item, empty);
        
        if (empty || item == null) {
            setGraphic(null);
            return;
        }
        
        // 데이터 바인딩 (핵심!)
        completedCheckBox.selectedProperty().bindBidirectional(
            item.completedProperty()
        );
        
        titleLabel.textProperty().bind(
            item.titleProperty()
        );
        
        priorityLabel.textProperty().bind(
            item.priorityProperty().asString()
        );
        
        // 스타일 바인딩
        titleLabel.styleProperty().bind(
            Bindings.when(item.completedProperty())
                .then("-fx-text-fill: gray; -fx-strikethrough: true;")
                .otherwise("-fx-text-fill: black;")
        );
        
        // 이벤트 핸들러
        completedCheckBox.setOnAction(e -> {
            listViewModel.toggleComplete(item);
        });
        
        deleteButton.setOnAction(e -> {
            listViewModel.deleteTodo(item);
        });
        
        setGraphic(container);
    }
}

/**
 * ============================================
 * APPLICATION (메인)
 * ============================================
 */
public class MVVMTodoApp extends Application {
    
    @Override
    public void start(Stage primaryStage) {
        System.out.println("=== MVVM 패턴 Todo 애플리케이션 ===\n");
        
        // Model
        TodoRepository repository = new TodoRepository();
        TodoService todoService = new TodoService(repository);
        
        // ViewModel
        TodoListViewModel viewModel = new TodoListViewModel(todoService);
        
        // View
        TodoListView view = new TodoListView(viewModel);
        
        // 초기 데이터
        viewModel.addTodo("MVVM 패턴 학습", "데이터 바인딩 이해하기", Todo.Priority.HIGH);
        viewModel.addTodo("JavaFX 복습", "Observable 프로퍼티 실습", Todo.Priority.NORMAL);
        viewModel.addTodo("예제 프로젝트", "Todo 앱 완성", Todo.Priority.URGENT);
        
        // Scene
        Scene scene = new Scene(view, 800, 600);
        
        primaryStage.setTitle("📝 Todo List - MVVM");
        primaryStage.setScene(scene);
        primaryStage.show();
    }
    
    public static void main(String[] args) {
        launch(args);
    }
}
```

**실행 결과 (콘솔):**
```
=== MVVM 패턴 Todo 애플리케이션 ===

✅ Todo 추가: MVVM 패턴 학습
✅ Todo 추가: JavaFX 복습
✅ Todo 추가: 예제 프로젝트
📋 Todo 로드: 3개

[사용자가 첫 번째 Todo 체크]
✓ 완료 토글: MVVM 패턴 학습

[사용자가 검색창에 "JavaFX" 입력]
→ 자동으로 필터링됨 (바인딩!)

[사용자가 "완료" 필터 선택]
→ 자동으로 완료된 항목만 표시 (바인딩!)

[Todo 삭제]
🗑️ Todo 삭제: 예제 프로젝트
```

---

## 5. 실전 예제

### 예제 1: 폼 검증 (Validation) ⭐⭐⭐

```java
/**
 * ============================================
 * 실시간 폼 검증 ViewModel
 * ============================================
 */
public class UserRegistrationViewModel {
    
    // Input Properties
    private final StringProperty email;
    private final StringProperty password;
    private final StringProperty passwordConfirm;
    private final StringProperty name;
    
    // Validation Properties (자동 계산)
    private final BooleanProperty emailValid;
    private final BooleanProperty passwordValid;
    private final BooleanProperty passwordMatch;
    private final BooleanProperty nameValid;
    private final BooleanProperty formValid;
    
    // Error Message Properties
    private final StringProperty emailError;
    private final StringProperty passwordError;
    private final StringProperty passwordConfirmError;
    private final StringProperty nameError;
    
    public UserRegistrationViewModel() {
        // Input 초기화
        email = new SimpleStringProperty("");
        password = new SimpleStringProperty("");
        passwordConfirm = new SimpleStringProperty("");
        name = new SimpleStringProperty("");
        
        // Validation 초기화
        emailValid = new SimpleBooleanProperty(false);
        passwordValid = new SimpleBooleanProperty(false);
        passwordMatch = new SimpleBooleanProperty(false);
        nameValid = new SimpleBooleanProperty(false);
        formValid = new SimpleBooleanProperty(false);
        
        // Error 초기화
        emailError = new SimpleStringProperty("");
        passwordError = new SimpleStringProperty("");
        passwordConfirmError = new SimpleStringProperty("");
        nameError = new SimpleStringProperty("");
        
        // 자동 검증 설정
        setupValidation();
    }
    
    /**
     * 자동 검증 설정 (실시간 검증!)
     */
    private void setupValidation() {
        // 이메일 검증
        email.addListener((obs, oldVal, newVal) -> {
            boolean valid = validateEmail(newVal);
            emailValid.set(valid);
            
            if (newVal.isEmpty()) {
                emailError.set("");
            } else if (!valid) {
                emailError.set("올바른 이메일 형식이 아닙니다");
            } else {
                emailError.set("");
            }
        });
        
        // 비밀번호 검증
        password.addListener((obs, oldVal, newVal) -> {
            boolean valid = validatePassword(newVal);
            passwordValid.set(valid);
            
            if (newVal.isEmpty()) {
                passwordError.set("");
            } else if (newVal.length() < 8) {
                passwordError.set("비밀번호는 8자 이상이어야 합니다");
            } else if (!newVal.matches(".*[A-Z].*")) {
                passwordError.set("대문자를 포함해야 합니다");
            } else if (!newVal.matches(".*[0-9].*")) {
                passwordError.set("숫자를 포함해야 합니다");
            } else {
                passwordError.set("");
            }
            
            // 비밀번호 확인도 재검증
            checkPasswordMatch();
        });
        
        // 비밀번호 확인 검증
        passwordConfirm.addListener((obs, oldVal, newVal) -> {
            checkPasswordMatch();
        });
        
        // 이름 검증
        name.addListener((obs, oldVal, newVal) -> {
            boolean valid = !newVal.trim().isEmpty() && newVal.length() >= 2;
            nameValid.set(valid);
            
            if (newVal.isEmpty()) {
                nameError.set("");
            } else if (!valid) {
                nameError.set("이름은 2자 이상이어야 합니다");
            } else {
                nameError.set("");
            }
        });
        
        // 전체 폼 검증 (모든 필드가 유효할 때만 true)
        formValid.bind(
            emailValid
                .and(passwordValid)
                .and(passwordMatch)
                .and(nameValid)
        );
    }
    
    /**
     * 비밀번호 일치 확인
     */
    private void checkPasswordMatch() {
        boolean match = password.get().equals(passwordConfirm.get()) 
            && !password.get().isEmpty();
        passwordMatch.set(match);
        
        if (passwordConfirm.get().isEmpty()) {
            passwordConfirmError.set("");
        } else if (!match) {
            passwordConfirmError.set("비밀번호가 일치하지 않습니다");
        } else {
            passwordConfirmError.set("");
        }
    }
    
    /**
     * 이메일 검증
     */
    private boolean validateEmail(String email) {
        return email.matches("^[A-Za-z0-9+_.-]+@(.+)$");
    }
    
    /**
     * 비밀번호 검증
     */
    private boolean validatePassword(String password) {
        return password.length() >= 8
            && password.matches(".*[A-Z].*")
            && password.matches(".*[0-9].*");
    }
    
    /**
     * Command: 회원가입
     */
    public void register() {
        if (!formValid.get()) {
            throw new IllegalStateException("유효하지 않은 폼");
        }
        
        System.out.println("✅ 회원가입 성공!");
        System.out.println("   이메일: " + email.get());
        System.out.println("   이름: " + name.get());
    }
    
    // Property Getters
    public StringProperty emailProperty() { return email; }
    public StringProperty passwordProperty() { return password; }
    public StringProperty passwordConfirmProperty() { return passwordConfirm; }
    public StringProperty nameProperty() { return name; }
    
    public BooleanProperty emailValidProperty() { return emailValid; }
    public BooleanProperty formValidProperty() { return formValid; }
    
    public StringProperty emailErrorProperty() { return emailError; }
    public StringProperty passwordErrorProperty() { return passwordError; }
    public StringProperty passwordConfirmErrorProperty() { return passwordConfirmError; }
    public StringProperty nameErrorProperty() { return nameError; }
}

/**
 * View with Binding
 */
public class UserRegistrationView extends VBox {
    private UserRegistrationViewModel viewModel;
    
    private TextField emailField;
    private Label emailErrorLabel;
    private PasswordField passwordField;
    private Label passwordErrorLabel;
    private PasswordField passwordConfirmField;
    private Label passwordConfirmErrorLabel;
    private TextField nameField;
    private Label nameErrorLabel;
    private Button registerButton;
    
    public UserRegistrationView(UserRegistrationViewModel viewModel) {
        this.viewModel = viewModel;
        
        initializeUI();
        setupBindings();
    }
    
    private void initializeUI() {
        setPadding(new Insets(20));
        setSpacing(10);
        
        // Email
        emailField = new TextField();
        emailField.setPromptText("이메일");
        emailErrorLabel = new Label();
        emailErrorLabel.setStyle("-fx-text-fill: red; -fx-font-size: 11px;");
        
        // Password
        passwordField = new PasswordField();
        passwordField.setPromptText("비밀번호");
        passwordErrorLabel = new Label();
        passwordErrorLabel.setStyle("-fx-text-fill: red; -fx-font-size: 11px;");
        
        // Password Confirm
        passwordConfirmField = new PasswordField();
        passwordConfirmField.setPromptText("비밀번호 확인");
        passwordConfirmErrorLabel = new Label();
        passwordConfirmErrorLabel.setStyle("-fx-text-fill: red; -fx-font-size: 11px;");
        
        // Name
        nameField = new TextField();
        nameField.setPromptText("이름");
        nameErrorLabel = new Label();
        nameErrorLabel.setStyle("-fx-text-fill: red; -fx-font-size: 11px;");
        
        // Register Button
        registerButton = new Button("회원가입");
        
        getChildren().addAll(
            new Label("회원가입"),
            emailField, emailErrorLabel,
            passwordField, passwordErrorLabel,
            passwordConfirmField, passwordConfirmErrorLabel,
            nameField, nameErrorLabel,
            registerButton
        );
    }
    
    /**
     * 데이터 바인딩 (핵심!)
     */
    private void setupBindings() {
        // 양방향 바인딩
        emailField.textProperty().bindBidirectional(viewModel.emailProperty());
        passwordField.textProperty().bindBidirectional(viewModel.passwordProperty());
        passwordConfirmField.textProperty().bindBidirectional(viewModel.passwordConfirmProperty());
        nameField.textProperty().bindBidirectional(viewModel.nameProperty());
        
        // 에러 메시지 바인딩 (단방향)
        emailErrorLabel.textProperty().bind(viewModel.emailErrorProperty());
        passwordErrorLabel.textProperty().bind(viewModel.passwordErrorProperty());
        passwordConfirmErrorLabel.textProperty().bind(viewModel.passwordConfirmErrorProperty());
        nameErrorLabel.textProperty().bind(viewModel.nameErrorProperty());
        
        // 버튼 활성화 바인딩
        registerButton.disableProperty().bind(
            viewModel.formValidProperty().not()
        );
        
        // 필드 스타일 바인딩 (유효성에 따라)
        emailField.styleProperty().bind(
            Bindings.when(viewModel.emailValidProperty().or(viewModel.emailProperty().isEmpty()))
                .then("-fx-border-color: transparent;")
                .otherwise("-fx-border-color: red;")
        );
        
        // 이벤트
        registerButton.setOnAction(e -> viewModel.register());
    }
}
```

---

## 6. MVC vs MVP vs MVVM

### 📊 비교표

| 특징 | MVC | MVP | MVVM |
|------|-----|-----|------|
| **View-Model 관계** | View가 Model 관찰 | Presenter가 중재 | ViewModel이 바인딩 |
| **데이터 흐름** | View → Controller → Model | View ↔ Presenter ↔ Model | View ↔ ViewModel → Model |
| **View 역할** | 능동적 (Model 관찰) | 수동적 (Presenter 제어) | 선언적 (바인딩) |
| **테스트 용이성** | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| **코드 복잡도** | 낮음 | 중간 | 중간 |
| **자동 동기화** | ❌ (수동) | ❌ (수동) | ✅ (자동) |
| **적합한 플랫폼** | Web (Spring MVC) | Android | WPF, JavaFX, Angular |

### 🔄 구조 비교

```
=== MVC ===
View → Controller → Model
 ↑                    │
 └─── Observer ───────┘

=== MVP ===
View ←─── Presenter ───→ Model
(View는 완전히 수동적)

=== MVVM ===
View ←── Data Binding ──→ ViewModel ───→ Model
(자동 동기화!)
```

---

## 7. 장단점

### ✅ 장점

| 장점 | 설명 | 실무 효과 |
|------|------|-----------|
| **자동 동기화** | 데이터 바인딩 | 코드 간결 |
| **테스트 용이** | ViewModel 독립 테스트 | 빠른 단위 테스트 |
| **관심사 분리** | UI 로직 분리 | 유지보수 용이 |
| **재사용성** | ViewModel 재사용 | 여러 View에서 공유 |
| **선언적 UI** | 바인딩으로 명확 | 가독성 향상 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **학습 곡선** | Observable, 바인딩 개념 | 문서화, 예제 |
| **초기 복잡도** | 코드량 증가 | 복잡한 UI만 적용 |
| **디버깅 어려움** | 바인딩 추적 어려움 | 로깅 |
| **메모리 관리** | 바인딩 해제 필요 | Lifecycle 관리 |

---

## 8. 안티패턴

### ❌ 안티패턴 1: View에 비즈니스 로직

```java
// 잘못된 예: View에 비즈니스 로직
public class ProductView {
    private TextField priceField;
    
    public void onSaveClick() {
        // ❌ 비즈니스 로직이 View에!
        double price = Double.parseDouble(priceField.getText());
        
        if (price > 1000000) {
            price *= 0.9;  // 할인
        }
        
        product.setPrice(price);
    }
}
```

**해결:**
```java
// 올바른 예: ViewModel에 로직
public class ProductViewModel {
    private DoubleProperty price = new SimpleDoubleProperty();
    private DoubleProperty finalPrice = new SimpleDoubleProperty();
    
    public ProductViewModel() {
        // 가격 변경 시 자동 계산
        price.addListener((obs, old, newVal) -> {
            double value = newVal.doubleValue();
            if (value > 1000000) {
                finalPrice.set(value * 0.9);
            } else {
                finalPrice.set(value);
            }
        });
    }
}
```

### ❌ 안티패턴 2: ViewModel이 View 참조

```java
// 잘못된 예: ViewModel이 View를 참조
public class UserViewModel {
    private UserView view;  // ❌
    
    public void save() {
        // ❌ View 직접 조작!
        view.showSuccessMessage();
    }
}
```

**해결:**
```java
// 올바른 예: Event나 Command로
public class UserViewModel {
    private ObjectProperty<Consumer<String>> onSuccess;
    
    public void save() {
        // 저장 로직
        
        // View에 알림 (간접적)
        if (onSuccess.get() != null) {
            onSuccess.get().accept("저장 완료");
        }
    }
}
```

---

## 9. 심화 주제

### 🎯 RxJava와 통합

```java
/**
 * Reactive ViewModel
 */
public class SearchViewModel {
    private final PublishSubject<String> searchSubject;
    private final Observable<List<Product>> searchResults;
    
    public SearchViewModel(ProductService service) {
        searchSubject = PublishSubject.create();
        
        // Reactive 검색 (디바운싱, 중복 제거)
        searchResults = searchSubject
            .debounce(300, TimeUnit.MILLISECONDS)  // 300ms 대기
            .distinctUntilChanged()  // 중복 제거
            .switchMap(keyword -> 
                Observable.fromCallable(() -> service.search(keyword))
                    .subscribeOn(Schedulers.io())
                    .observeOn(JavaFxScheduler.platform())
            )
            .share();
    }
    
    public void search(String keyword) {
        searchSubject.onNext(keyword);
    }
    
    public Observable<List<Product>> getSearchResults() {
        return searchResults;
    }
}
```

### 🔥 Command 패턴 통합

```java
/**
 * RelayCommand (WPF 스타일)
 */
public class RelayCommand {
    private final Runnable execute;
    private final Supplier<Boolean> canExecute;
    private final List<Runnable> canExecuteChangedListeners;
    
    public RelayCommand(Runnable execute, Supplier<Boolean> canExecute) {
        this.execute = execute;
        this.canExecute = canExecute;
        this.canExecuteChangedListeners = new ArrayList<>();
    }
    
    public void execute() {
        if (canExecute()) {
            execute.run();
        }
    }
    
    public boolean canExecute() {
        return canExecute == null || canExecute.get();
    }
    
    public void raiseCanExecuteChanged() {
        canExecuteChangedListeners.forEach(Runnable::run);
    }
    
    public void addCanExecuteChangedListener(Runnable listener) {
        canExecuteChangedListeners.add(listener);
    }
}

/**
 * ViewModel with Commands
 */
public class OrderViewModel {
    private final RelayCommand submitCommand;
    private final BooleanProperty canSubmit;
    
    public OrderViewModel() {
        canSubmit = new SimpleBooleanProperty(false);
        
        submitCommand = new RelayCommand(
            this::submit,
            () -> canSubmit.get()
        );
        
        // canSubmit 변경 시 Command도 업데이트
        canSubmit.addListener((obs, old, newVal) -> {
            submitCommand.raiseCanExecuteChanged();
        });
    }
    
    public RelayCommand getSubmitCommand() {
        return submitCommand;
    }
}
```

---

## 10. 핵심 정리

### 📌 MVVM 패턴 체크리스트

```
✅ Observable Properties 사용
✅ 데이터 바인딩 (양방향/단방향)
✅ ViewModel은 View 모름
✅ View는 UI만 담당
✅ Command 패턴으로 액션
✅ Validation은 ViewModel에
✅ 계산된 프로퍼티 활용
✅ 자동 동기화
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| JavaFX 애플리케이션 | ⭐⭐⭐ | 네이티브 지원 |
| 복잡한 UI | ⭐⭐⭐ | 자동 동기화 |
| 실시간 검증 | ⭐⭐⭐ | Observable |
| 간단한 CRUD | ⭐⭐ | 오버엔지니어링 |

### 💡 핵심 원칙

1. **데이터 바인딩**
2. **Observable Properties**
3. **ViewModel은 View 독립**
4. **자동 동기화**

### 🔥 실무 팁

```java
// ✅ DO: Observable 프로퍼티
private StringProperty name = new SimpleStringProperty();
public StringProperty nameProperty() { return name; }

// ✅ DO: 계산된 프로퍼티
fullName.bind(
    firstName.concat(" ").concat(lastName)
);

// ✅ DO: 양방향 바인딩
textField.textProperty().bindBidirectional(
    viewModel.nameProperty()
);

// ❌ DON'T: ViewModel이 View 참조
public class ViewModel {
    private View view;  // ❌
}

// ❌ DON'T: 수동 동기화
textField.setText(viewModel.getName());  // ❌
```

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Hexagonal](./04-Hexagonal.md) | [다음: Event-Driven →](./06-EventDriven.md)**

</div>
