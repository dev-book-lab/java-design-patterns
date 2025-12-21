# Memento Pattern (메멘토 패턴)

> **"객체의 상태를 저장하고 복원하자"**

---

## 📌 목차
1. [문제 상황](#1-문제-상황)
2. [패턴 정의](#2-패턴-정의)
3. [구조와 구성요소](#3-구조와-구성요소)
4. [구현 방법](#4-구현-방법)
5. [실전 예제](#5-실전-예제)
6. [장단점](#6-장단점)
7. [Command vs Memento](#7-command-vs-memento)
8. [핵심 정리](#8-핵심-정리)

---

## 1. 문제 상황

### 🤔 이런 경험 있으신가요?

```java
// 문제 1: 캡슐화 위반
public class TextEditor {
    private String text;
    private int cursorPosition;
    private String fontFamily;
    
    // Undo를 위해 모든 필드를 public으로?
    public String getText() { return text; }
    public int getCursorPosition() { return cursorPosition; }
    public String getFontFamily() { return fontFamily; }
    
    // 외부에서 상태 복원
    public void setText(String t) { text = t; }
    public void setCursorPosition(int p) { cursorPosition = p; }
    public void setFontFamily(String f) { fontFamily = f; }
    
    // 캡슐화 깨짐!
}

// 문제 2: 히스토리 관리 복잡
public class Game {
    private int level;
    private int health;
    private int score;
    private Position position;
    
    // 이전 상태를 어떻게 저장?
    private int previousLevel;
    private int previousHealth;
    private int previousScore;
    private Position previousPosition;
    
    // 여러 단계 Undo는?
    // 필드가 더 많아지면?
}

// 문제 3: 상태 저장 로직 분산
public class Drawing {
    private List<Shape> shapes;
    
    public void save() {
        // 어떻게 저장?
        // 각 Shape도 저장해야 함
        for (Shape shape : shapes) {
            // Shape의 내부 상태는?
        }
    }
    
    public void undo() {
        // 어떻게 복원?
        // 복잡한 객체 그래프!
    }
}

// 문제 4: 직렬화 의존
public class Document {
    private transient Connection dbConnection; // 직렬화 제외
    
    public void saveState() {
        // 직렬화하면?
        // → transient 필드는 저장 안 됨
        // → 복원 시 문제 발생
    }
}
```

### ⚡ 핵심 문제

1. **캡슐화 위반**: 상태 저장을 위해 내부 노출
2. **복잡한 저장**: 깊은 복사, 객체 그래프 처리
3. **히스토리 관리**: 여러 단계 Undo/Redo
4. **의존성**: 직렬화에 의존

---

## 2. 패턴 정의

### 📖 정의

> **캡슐화를 위반하지 않고 객체의 내부 상태를 캡처하고 외부에 저장하여, 나중에 이 상태로 객체를 복원할 수 있게 하는 패턴**

### 🎯 목적

- **캡슐화 유지**: 내부 상태를 노출하지 않음
- **상태 저장**: 스냅샷 생성
- **상태 복원**: 이전 상태로 되돌림
- **Undo/Redo**: 히스토리 관리

### 💡 핵심 아이디어

```java
// Before: 외부에서 상태 접근
EditorState state = new EditorState(
    editor.getText(),
    editor.getCursorPosition()
); // 캡슐화 위반!

// After: Memento 사용
Memento memento = editor.save(); // 내부 상태 캡슐화
editor.restore(memento); // 복원
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌──────────────┐
│  Originator  │  ← 원본 객체
├──────────────┤
│ - state      │
│ + save()     │────┐ creates
│ + restore()  │    │
└──────────────┘    │
                    ▼
              ┌──────────────┐
              │   Memento    │  ← 상태 저장
              ├──────────────┤
              │ - state      │
              │ + getState() │
              └──────────────┘
                    △
                    │ stores
              ┌──────────────┐
              │  Caretaker   │  ← 히스토리 관리
              ├──────────────┤
              │ - mementos   │
              │ + save()     │
              │ + undo()     │
              └──────────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **Originator** | 상태를 가진 원본 객체 | `TextEditor` |
| **Memento** | 상태 스냅샷 | `EditorMemento` |
| **Caretaker** | Memento 관리 | `History` |

---

## 4. 구현 방법

### 기본 구현: 텍스트 에디터 ⭐⭐⭐

```java
/**
 * Memento: 에디터 상태
 */
public class EditorMemento {
    private final String text;
    private final int cursorPosition;
    
    public EditorMemento(String text, int cursorPosition) {
        this.text = text;
        this.cursorPosition = cursorPosition;
    }
    
    public String getText() {
        return text;
    }
    
    public int getCursorPosition() {
        return cursorPosition;
    }
}

/**
 * Originator: 텍스트 에디터
 */
public class TextEditor {
    private String text;
    private int cursorPosition;
    
    public TextEditor() {
        this.text = "";
        this.cursorPosition = 0;
    }
    
    public void write(String words) {
        text += words;
        cursorPosition = text.length();
        System.out.println("✍️ 입력: " + words);
        System.out.println("   현재 텍스트: \"" + text + "\"");
    }
    
    public void moveCursor(int position) {
        this.cursorPosition = position;
        System.out.println("🖱️ 커서 이동: " + position);
    }
    
    public EditorMemento save() {
        System.out.println("💾 상태 저장");
        return new EditorMemento(text, cursorPosition);
    }
    
    public void restore(EditorMemento memento) {
        this.text = memento.getText();
        this.cursorPosition = memento.getCursorPosition();
        System.out.println("↩️ 상태 복원: \"" + text + "\"");
    }
    
    public void display() {
        System.out.println("📄 텍스트: \"" + text + "\" (커서: " + cursorPosition + ")");
    }
}

/**
 * Caretaker: 히스토리 관리
 */
public class EditorHistory {
    private Stack<EditorMemento> history;
    
    public EditorHistory() {
        this.history = new Stack<>();
    }
    
    public void save(EditorMemento memento) {
        history.push(memento);
        System.out.println("📚 히스토리 크기: " + history.size());
    }
    
    public EditorMemento undo() {
        if (history.isEmpty()) {
            System.out.println("⚠️ 되돌릴 상태가 없습니다");
            return null;
        }
        return history.pop();
    }
    
    public int size() {
        return history.size();
    }
}

/**
 * 사용 예제
 */
public class MementoExample {
    public static void main(String[] args) {
        TextEditor editor = new TextEditor();
        EditorHistory history = new EditorHistory();
        
        System.out.println("=== 텍스트 에디터 ===\n");
        
        // 작업 1
        editor.write("Hello");
        history.save(editor.save());
        
        System.out.println();
        
        // 작업 2
        editor.write(" World");
        history.save(editor.save());
        
        System.out.println();
        
        // 작업 3
        editor.write("!");
        history.save(editor.save());
        
        System.out.println();
        editor.display();
        
        // Undo
        System.out.println("\n=== Undo ===\n");
        EditorMemento memento = history.undo();
        if (memento != null) {
            editor.restore(memento);
        }
        
        System.out.println();
        
        memento = history.undo();
        if (memento != null) {
            editor.restore(memento);
        }
        
        System.out.println();
        editor.display();
    }
}
```

**실행 결과:**
```
=== 텍스트 에디터 ===

✍️ 입력: Hello
   현재 텍스트: "Hello"
💾 상태 저장
📚 히스토리 크기: 1

✍️ 입력:  World
   현재 텍스트: "Hello World"
💾 상태 저장
📚 히스토리 크기: 2

✍️ 입력: !
   현재 텍스트: "Hello World!"
💾 상태 저장
📚 히스토리 크기: 3

📄 텍스트: "Hello World!" (커서: 12)

=== Undo ===

↩️ 상태 복원: "Hello World!"

↩️ 상태 복원: "Hello World"

📄 텍스트: "Hello World" (커서: 11)
```

---

## 5. 실전 예제

### 예제 1: 게임 체크포인트 ⭐⭐⭐

```java
/**
 * Memento: 게임 상태
 */
public class GameMemento {
    private final int level;
    private final int health;
    private final int score;
    private final Position position;
    
    public GameMemento(int level, int health, int score, Position position) {
        this.level = level;
        this.health = health;
        this.score = score;
        this.position = new Position(position); // 깊은 복사
    }
    
    public int getLevel() { return level; }
    public int getHealth() { return health; }
    public int getScore() { return score; }
    public Position getPosition() { return new Position(position); }
}

/**
 * Position 클래스
 */
class Position {
    private int x;
    private int y;
    
    public Position(int x, int y) {
        this.x = x;
        this.y = y;
    }
    
    public Position(Position other) {
        this.x = other.x;
        this.y = other.y;
    }
    
    @Override
    public String toString() {
        return "(" + x + "," + y + ")";
    }
}

/**
 * Originator: 게임
 */
public class Game {
    private int level;
    private int health;
    private int score;
    private Position position;
    
    public Game() {
        this.level = 1;
        this.health = 100;
        this.score = 0;
        this.position = new Position(0, 0);
    }
    
    public void play() {
        level++;
        health -= 20;
        score += 100;
        position = new Position(position.x + 10, position.y + 5);
        
        System.out.println("🎮 게임 진행:");
        displayStatus();
    }
    
    public void takeDamage(int damage) {
        health -= damage;
        System.out.println("💥 데미지: -" + damage);
        displayStatus();
    }
    
    public GameMemento save() {
        System.out.println("💾 체크포인트 저장");
        return new GameMemento(level, health, score, position);
    }
    
    public void restore(GameMemento memento) {
        this.level = memento.getLevel();
        this.health = memento.getHealth();
        this.score = memento.getScore();
        this.position = memento.getPosition();
        System.out.println("↩️ 체크포인트 복원");
        displayStatus();
    }
    
    public void displayStatus() {
        System.out.println("   레벨: " + level + ", 체력: " + health + 
                ", 점수: " + score + ", 위치: " + position);
    }
}

/**
 * Caretaker: 세이브 매니저
 */
public class SaveManager {
    private List<GameMemento> saves;
    
    public SaveManager() {
        this.saves = new ArrayList<>();
    }
    
    public void save(GameMemento memento) {
        saves.add(memento);
        System.out.println("📁 세이브 슬롯 " + saves.size() + "에 저장");
    }
    
    public GameMemento load(int slot) {
        if (slot < 1 || slot > saves.size()) {
            System.out.println("⚠️ 잘못된 슬롯 번호");
            return null;
        }
        return saves.get(slot - 1);
    }
    
    public void showSaves() {
        System.out.println("\n📚 세이브 목록:");
        for (int i = 0; i < saves.size(); i++) {
            System.out.println("   슬롯 " + (i + 1));
        }
    }
}

/**
 * 사용 예제
 */
public class GameMementoExample {
    public static void main(String[] args) {
        Game game = new Game();
        SaveManager saveManager = new SaveManager();
        
        System.out.println("=== 게임 시작 ===\n");
        game.displayStatus();
        
        // 1단계 클리어
        System.out.println("\n--- 1단계 ---");
        game.play();
        saveManager.save(game.save());
        
        // 2단계 클리어
        System.out.println("\n--- 2단계 ---");
        game.play();
        saveManager.save(game.save());
        
        // 3단계 시작 - 큰 데미지
        System.out.println("\n--- 3단계 ---");
        game.play();
        game.takeDamage(50);
        
        // 게임 오버 직전 - 이전 세이브 로드
        System.out.println("\n=== 세이브 로드 ===");
        saveManager.showSaves();
        GameMemento save = saveManager.load(2);
        if (save != null) {
            game.restore(save);
        }
    }
}
```

---

### 예제 2: 그림판 ⭐⭐⭐

```java
/**
 * Shape 클래스
 */
class Shape {
    private String type;
    private int x, y;
    private String color;
    
    public Shape(String type, int x, int y, String color) {
        this.type = type;
        this.x = x;
        this.y = y;
        this.color = color;
    }
    
    // 깊은 복사
    public Shape(Shape other) {
        this.type = other.type;
        this.x = other.x;
        this.y = other.y;
        this.color = other.color;
    }
    
    @Override
    public String toString() {
        return type + " at (" + x + "," + y + ") - " + color;
    }
}

/**
 * Memento: 캔버스 상태
 */
public class CanvasMemento {
    private final List<Shape> shapes;
    
    public CanvasMemento(List<Shape> shapes) {
        // 깊은 복사
        this.shapes = new ArrayList<>();
        for (Shape shape : shapes) {
            this.shapes.add(new Shape(shape));
        }
    }
    
    public List<Shape> getShapes() {
        // 깊은 복사 반환
        List<Shape> copy = new ArrayList<>();
        for (Shape shape : shapes) {
            copy.add(new Shape(shape));
        }
        return copy;
    }
}

/**
 * Originator: 캔버스
 */
public class Canvas {
    private List<Shape> shapes;
    
    public Canvas() {
        this.shapes = new ArrayList<>();
    }
    
    public void addShape(String type, int x, int y, String color) {
        Shape shape = new Shape(type, x, y, color);
        shapes.add(shape);
        System.out.println("➕ 도형 추가: " + shape);
    }
    
    public void removeLastShape() {
        if (!shapes.isEmpty()) {
            Shape removed = shapes.remove(shapes.size() - 1);
            System.out.println("🗑️ 도형 제거: " + removed);
        }
    }
    
    public CanvasMemento save() {
        System.out.println("💾 캔버스 저장 (" + shapes.size() + "개 도형)");
        return new CanvasMemento(shapes);
    }
    
    public void restore(CanvasMemento memento) {
        this.shapes = memento.getShapes();
        System.out.println("↩️ 캔버스 복원 (" + shapes.size() + "개 도형)");
    }
    
    public void display() {
        System.out.println("\n🎨 캔버스 (" + shapes.size() + "개 도형):");
        for (int i = 0; i < shapes.size(); i++) {
            System.out.println("   " + (i + 1) + ". " + shapes.get(i));
        }
    }
}

/**
 * Caretaker: 히스토리
 */
public class CanvasHistory {
    private Stack<CanvasMemento> undoStack;
    private Stack<CanvasMemento> redoStack;
    
    public CanvasHistory() {
        this.undoStack = new Stack<>();
        this.redoStack = new Stack<>();
    }
    
    public void save(CanvasMemento memento) {
        undoStack.push(memento);
        redoStack.clear(); // 새 작업 시 redo 스택 초기화
    }
    
    public CanvasMemento undo() {
        if (undoStack.isEmpty()) {
            System.out.println("⚠️ 되돌릴 작업이 없습니다");
            return null;
        }
        CanvasMemento memento = undoStack.pop();
        redoStack.push(memento);
        return memento;
    }
    
    public CanvasMemento redo() {
        if (redoStack.isEmpty()) {
            System.out.println("⚠️ 다시 실행할 작업이 없습니다");
            return null;
        }
        CanvasMemento memento = redoStack.pop();
        undoStack.push(memento);
        return memento;
    }
}

/**
 * 사용 예제
 */
public class CanvasMementoExample {
    public static void main(String[] args) {
        Canvas canvas = new Canvas();
        CanvasHistory history = new CanvasHistory();
        
        System.out.println("=== 그림판 ===\n");
        
        // 도형 추가 1
        canvas.addShape("Circle", 10, 10, "Red");
        history.save(canvas.save());
        
        System.out.println();
        
        // 도형 추가 2
        canvas.addShape("Rectangle", 50, 50, "Blue");
        history.save(canvas.save());
        
        System.out.println();
        
        // 도형 추가 3
        canvas.addShape("Triangle", 100, 100, "Green");
        
        canvas.display();
        
        // Undo
        System.out.println("\n=== Undo ===\n");
        CanvasMemento memento = history.undo();
        if (memento != null) {
            canvas.restore(memento);
        }
        canvas.display();
        
        // Redo
        System.out.println("\n=== Redo ===\n");
        memento = history.redo();
        if (memento != null) {
            canvas.restore(memento);
        }
        canvas.display();
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **캡슐화 유지** | 내부 상태 노출 안 함 | Memento |
| **Undo/Redo** | 히스토리 관리 용이 | 에디터 |
| **스냅샷** | 상태 저장 간단 | 게임 세이브 |
| **복원 간편** | 상태 복원 쉬움 | 체크포인트 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **메모리** | Memento 많으면 메모리 | 개수 제한 |
| **복사 비용** | 깊은 복사 비용 | Flyweight |
| **관리 복잡** | 히스토리 관리 | Caretaker |

---

## 7. Command vs Memento

### 🔍 비교표

| 특징 | Command | Memento |
|------|---------|---------|
| **저장** | 명령 저장 | 상태 저장 |
| **복원** | 명령 역실행 | 상태 복원 |
| **복잡도** | 간단 | 복잡한 상태 |
| **예시** | 텍스트 쓰기 | 에디터 전체 상태 |

---

## 8. 핵심 정리

### 📌 Memento 패턴 체크리스트

```
✅ Memento 클래스
✅ Originator 구현
✅ save() 메서드
✅ restore() 메서드
✅ Caretaker 구현
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| Undo/Redo | ⭐⭐⭐ | 상태 복원 |
| 체크포인트 | ⭐⭐⭐ | 게임 세이브 |
| 트랜잭션 | ⭐⭐⭐ | 롤백 |
| 히스토리 | ⭐⭐⭐ | 버전 관리 |

### 💡 핵심 포인트

1. **캡슐화 유지**
2. **스냅샷 저장**
3. **상태 복원**
4. **깊은 복사 주의**

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Mediator](08-Mediator.md) | [다음: Visitor →](10-Visitor.md)**

</div>
