# Composite Pattern (컴포지트 패턴)

> **"부분-전체 계층을 트리 구조로 표현하자"**

[← 이전: Proxy](./03-Proxy.md) | [목차로 돌아가기](../README.md) | [다음: Facade →](05-Facade.md)

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
// 문제 1: 개별 객체와 그룹을 다르게 처리
public class FileSystem {
    public void displayFile(File file) {
        System.out.println("File: " + file.getName());
    }
    
    public void displayFolder(Folder folder) {
        System.out.println("Folder: " + folder.getName());
        for (File file : folder.getFiles()) {
            displayFile(file); // 개별 처리
        }
        for (Folder subFolder : folder.getFolders()) {
            displayFolder(subFolder); // 재귀 처리
        }
    }
    
    // File과 Folder를 다르게 처리해야 함!
}

// 문제 2: 계층 구조 순회가 복잡
public class Organization {
    public int getTotalSalary() {
        int total = 0;
        
        // CEO 급여
        total += ceo.getSalary();
        
        // 부서장들 급여
        for (Manager manager : managers) {
            total += manager.getSalary();
            
            // 팀원들 급여
            for (Employee emp : manager.getTeam()) {
                total += emp.getSalary();
            }
        }
        
        // 계층마다 다른 로직... 복잡!
        return total;
    }
}

// 문제 3: 타입 체크가 필요
public void process(Object item) {
    if (item instanceof File) {
        processFile((File) item);
    } else if (item instanceof Folder) {
        Folder folder = (Folder) item;
        for (Object child : folder.getChildren()) {
            process(child); // 재귀 + 타입 체크
        }
    }
    // instanceof 남발!
}

// 문제 4: 새로운 타입 추가 시 모든 코드 수정
public class GraphicEditor {
    public void draw(Object graphic) {
        if (graphic instanceof Circle) {
            drawCircle((Circle) graphic);
        } else if (graphic instanceof Rectangle) {
            drawRectangle((Rectangle) graphic);
        } else if (graphic instanceof Group) {
            // Group 추가 시 여기도 수정!
            Group group = (Group) graphic;
            for (Object child : group.getChildren()) {
                draw(child);
            }
        }
    }
}
```

### ⚡ 핵심 문제

1. **일관성 없는 처리**: 개별 객체와 그룹을 다르게 처리
2. **복잡한 순회**: 계층 구조 탐색이 어려움
3. **타입 의존성**: instanceof로 타입 체크 필요
4. **확장성 부족**: 새 타입 추가 시 모든 코드 수정

---

## 2. 패턴 정의

### 📖 정의

> **객체들을 트리 구조로 구성하여 부분-전체 계층을 표현하는 패턴. 클라이언트가 개별 객체와 복합 객체를 동일하게 다루도록 한다.**

### 🎯 목적

- **일관된 처리**: 개별 객체와 그룹을 동일하게 처리
- **트리 구조**: 재귀적 계층 구조 표현
- **투명성**: 클라이언트는 타입 구분 불필요
- **확장성**: 새로운 타입 추가 용이

### 💡 핵심 아이디어

```java
// Before: 타입별로 다르게 처리
if (item instanceof File) {
    processFile((File) item);
} else if (item instanceof Folder) {
    processFolder((Folder) item);
}

// After: 통일된 인터페이스로 처리
Component item = ...; // File or Folder
item.display(); // 동일한 방식!
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌─────────────────────┐
│    Component        │  ← 공통 인터페이스
├─────────────────────┤
│ + operation()       │
│ + add(Component)    │
│ + remove(Component) │
│ + getChild(int)     │
└─────────────────────┘
          △
          │
    ┌─────┴─────┐
    │           │
┌───────┐  ┌──────────────┐
│ Leaf  │  │  Composite   │  ← 그룹 객체
├───────┤  ├──────────────┤
│oper...│  │ - children   │───┐
└───────┘  │ + operation()│   │
           │ + add()      │   │ contains
           │ + remove()   │   │
           └──────────────┘   │
                  │           │
                  └───────────┘
                  여러 Component 포함
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **Component** | 공통 인터페이스 | `FileSystemItem` |
| **Leaf** | 단말 노드 (개별 객체) | `File` |
| **Composite** | 복합 노드 (그룹) | `Folder` |
| **Client** | Component 사용 | `FileExplorer` |

---

## 4. 구현 방법

### 기본 구현: 파일 시스템 ⭐⭐⭐

```java
/**
 * Component: 파일 시스템 항목
 */
public abstract class FileSystemItem {
    protected String name;
    
    public FileSystemItem(String name) {
        this.name = name;
    }
    
    public String getName() {
        return name;
    }
    
    // 모든 항목이 구현해야 하는 메서드
    public abstract void display(int depth);
    public abstract int getSize();
    
    // Composite 메서드 (기본 구현: 예외 발생)
    public void add(FileSystemItem item) {
        throw new UnsupportedOperationException();
    }
    
    public void remove(FileSystemItem item) {
        throw new UnsupportedOperationException();
    }
    
    public List<FileSystemItem> getChildren() {
        throw new UnsupportedOperationException();
    }
}

/**
 * Leaf: 파일
 */
public class File extends FileSystemItem {
    private int size;
    
    public File(String name, int size) {
        super(name);
        this.size = size;
    }
    
    @Override
    public void display(int depth) {
        String indent = "  ".repeat(depth);
        System.out.println(indent + "📄 " + name + " (" + size + " KB)");
    }
    
    @Override
    public int getSize() {
        return size;
    }
}

/**
 * Composite: 폴더
 */
public class Folder extends FileSystemItem {
    private List<FileSystemItem> children;
    
    public Folder(String name) {
        super(name);
        this.children = new ArrayList<>();
    }
    
    @Override
    public void add(FileSystemItem item) {
        children.add(item);
        System.out.println("✅ Added " + item.getName() + " to " + name);
    }
    
    @Override
    public void remove(FileSystemItem item) {
        children.remove(item);
        System.out.println("❌ Removed " + item.getName() + " from " + name);
    }
    
    @Override
    public List<FileSystemItem> getChildren() {
        return new ArrayList<>(children);
    }
    
    @Override
    public void display(int depth) {
        String indent = "  ".repeat(depth);
        System.out.println(indent + "📁 " + name);
        
        // 재귀적으로 자식 항목 표시
        for (FileSystemItem child : children) {
            child.display(depth + 1);
        }
    }
    
    @Override
    public int getSize() {
        // 모든 자식의 크기 합계
        return children.stream()
                .mapToInt(FileSystemItem::getSize)
                .sum();
    }
}

/**
 * 사용 예제
 */
public class CompositeExample {
    public static void main(String[] args) {
        // 파일 생성
        File file1 = new File("document.txt", 10);
        File file2 = new File("image.jpg", 500);
        File file3 = new File("video.mp4", 2000);
        File file4 = new File("music.mp3", 300);
        
        // 폴더 생성
        Folder documents = new Folder("Documents");
        Folder media = new Folder("Media");
        Folder root = new Folder("Root");
        
        // 계층 구조 생성
        System.out.println("=== 파일 시스템 구성 ===");
        documents.add(file1);
        
        media.add(file2);
        media.add(file3);
        media.add(file4);
        
        root.add(documents);
        root.add(media);
        
        // 전체 구조 표시
        System.out.println("\n=== 파일 시스템 구조 ===");
        root.display(0);
        
        // 전체 크기 계산
        System.out.println("\n=== 크기 정보 ===");
        System.out.println("Total size: " + root.getSize() + " KB");
        System.out.println("Documents size: " + documents.getSize() + " KB");
        System.out.println("Media size: " + media.getSize() + " KB");
        
        // 항목 제거
        System.out.println("\n=== 항목 제거 ===");
        media.remove(file3);
        root.display(0);
        System.out.println("New total size: " + root.getSize() + " KB");
    }
}
```

**실행 결과:**
```
=== 파일 시스템 구성 ===
✅ Added document.txt to Documents
✅ Added image.jpg to Media
✅ Added video.mp4 to Media
✅ Added music.mp3 to Media
✅ Added Documents to Root
✅ Added Media to Root

=== 파일 시스템 구조 ===
📁 Root
  📁 Documents
    📄 document.txt (10 KB)
  📁 Media
    📄 image.jpg (500 KB)
    📄 video.mp4 (2000 KB)
    📄 music.mp3 (300 KB)

=== 크기 정보 ===
Total size: 2810 KB
Documents size: 10 KB
Media size: 2800 KB

=== 항목 제거 ===
❌ Removed video.mp4 from Media
📁 Root
  📁 Documents
    📄 document.txt (10 KB)
  📁 Media
    📄 image.jpg (500 KB)
    📄 music.mp3 (300 KB)
New total size: 810 KB
```

---

## 5. 실전 예제

### 예제 1: 조직도 시스템 ⭐⭐⭐

```java
/**
 * Component: 조직 구성원
 */
public abstract class Employee {
    protected String name;
    protected String position;
    protected int salary;
    
    public Employee(String name, String position, int salary) {
        this.name = name;
        this.position = position;
        this.salary = salary;
    }
    
    public String getName() { return name; }
    public int getSalary() { return salary; }
    
    public abstract void showDetails(int depth);
    public abstract int getTotalSalary();
    
    // Composite 메서드
    public void add(Employee employee) {
        throw new UnsupportedOperationException();
    }
    
    public void remove(Employee employee) {
        throw new UnsupportedOperationException();
    }
    
    public List<Employee> getSubordinates() {
        throw new UnsupportedOperationException();
    }
}

/**
 * Leaf: 일반 직원
 */
public class Developer extends Employee {
    private String technology;
    
    public Developer(String name, int salary, String technology) {
        super(name, "Developer", salary);
        this.technology = technology;
    }
    
    @Override
    public void showDetails(int depth) {
        String indent = "  ".repeat(depth);
        System.out.println(indent + "👨‍💻 " + name + " - " + position + 
                " (" + technology + ") - $" + salary);
    }
    
    @Override
    public int getTotalSalary() {
        return salary;
    }
}

public class Designer extends Employee {
    private String tool;
    
    public Designer(String name, int salary, String tool) {
        super(name, "Designer", salary);
        this.tool = tool;
    }
    
    @Override
    public void showDetails(int depth) {
        String indent = "  ".repeat(depth);
        System.out.println(indent + "🎨 " + name + " - " + position + 
                " (" + tool + ") - $" + salary);
    }
    
    @Override
    public int getTotalSalary() {
        return salary;
    }
}

/**
 * Composite: 매니저 (팀 리더)
 */
public class Manager extends Employee {
    private List<Employee> subordinates;
    
    public Manager(String name, int salary) {
        super(name, "Manager", salary);
        this.subordinates = new ArrayList<>();
    }
    
    @Override
    public void add(Employee employee) {
        subordinates.add(employee);
        System.out.println("✅ " + employee.getName() + " added to " + name + "'s team");
    }
    
    @Override
    public void remove(Employee employee) {
        subordinates.remove(employee);
        System.out.println("❌ " + employee.getName() + " removed from " + name + "'s team");
    }
    
    @Override
    public List<Employee> getSubordinates() {
        return new ArrayList<>(subordinates);
    }
    
    @Override
    public void showDetails(int depth) {
        String indent = "  ".repeat(depth);
        System.out.println(indent + "👔 " + name + " - " + position + " - $" + salary);
        
        // 팀원들 표시
        for (Employee employee : subordinates) {
            employee.showDetails(depth + 1);
        }
    }
    
    @Override
    public int getTotalSalary() {
        // 매니저 급여 + 팀원들 급여
        int total = salary;
        for (Employee employee : subordinates) {
            total += employee.getTotalSalary();
        }
        return total;
    }
}

/**
 * 사용 예제
 */
public class OrganizationExample {
    public static void main(String[] args) {
        // CEO
        Manager ceo = new Manager("John (CEO)", 10000);
        
        // 부서장들
        Manager devManager = new Manager("Alice (Dev Manager)", 7000);
        Manager designManager = new Manager("Bob (Design Manager)", 6000);
        
        // 개발팀
        Developer dev1 = new Developer("Charlie", 5000, "Java");
        Developer dev2 = new Developer("David", 5000, "Python");
        Developer dev3 = new Developer("Eve", 4500, "JavaScript");
        
        // 디자인팀
        Designer designer1 = new Designer("Frank", 4000, "Figma");
        Designer designer2 = new Designer("Grace", 4000, "Photoshop");
        
        // 조직 구성
        System.out.println("=== 조직 구성 ===");
        devManager.add(dev1);
        devManager.add(dev2);
        devManager.add(dev3);
        
        designManager.add(designer1);
        designManager.add(designer2);
        
        ceo.add(devManager);
        ceo.add(designManager);
        
        // 조직도 출력
        System.out.println("\n=== 조직도 ===");
        ceo.showDetails(0);
        
        // 급여 통계
        System.out.println("\n=== 급여 통계 ===");
        System.out.println("CEO 팀 총 급여: $" + ceo.getTotalSalary());
        System.out.println("개발팀 총 급여: $" + devManager.getTotalSalary());
        System.out.println("디자인팀 총 급여: $" + designManager.getTotalSalary());
        
        // 직원 재배치
        System.out.println("\n=== 직원 재배치 ===");
        devManager.remove(dev3);
        designManager.add(dev3);
        
        System.out.println("\n=== 재배치 후 조직도 ===");
        ceo.showDetails(0);
    }
}
```

**실행 결과:**
```
=== 조직 구성 ===
✅ Charlie added to Alice (Dev Manager)'s team
✅ David added to Alice (Dev Manager)'s team
✅ Eve added to Alice (Dev Manager)'s team
✅ Frank added to Bob (Design Manager)'s team
✅ Grace added to Bob (Design Manager)'s team
✅ Alice (Dev Manager) added to John (CEO)'s team
✅ Bob (Design Manager) added to John (CEO)'s team

=== 조직도 ===
👔 John (CEO) - Manager - $10000
  👔 Alice (Dev Manager) - Manager - $7000
    👨‍💻 Charlie - Developer (Java) - $5000
    👨‍💻 David - Developer (Python) - $5000
    👨‍💻 Eve - Developer (JavaScript) - $4500
  👔 Bob (Design Manager) - Manager - $6000
    🎨 Frank - Designer (Figma) - $4000
    🎨 Grace - Designer (Photoshop) - $4000

=== 급여 통계 ===
CEO 팀 총 급여: $45500
개발팀 총 급여: $21500
디자인팀 총 급여: $14000

=== 직원 재배치 ===
❌ Eve removed from Alice (Dev Manager)'s team
✅ Eve added to Bob (Design Manager)'s team

=== 재배치 후 조직도 ===
👔 John (CEO) - Manager - $10000
  👔 Alice (Dev Manager) - Manager - $7000
    👨‍💻 Charlie - Developer (Java) - $5000
    👨‍💻 David - Developer (Python) - $5000
  👔 Bob (Design Manager) - Manager - $6000
    🎨 Frank - Designer (Figma) - $4000
    🎨 Grace - Designer (Photoshop) - $4000
    👨‍💻 Eve - Developer (JavaScript) - $4500
```

---

### 예제 2: 그래픽 에디터 ⭐⭐⭐

```java
/**
 * Component: 그래픽 객체
 */
public interface Graphic {
    void draw();
    void move(int x, int y);
    void resize(double scale);
}

/**
 * Leaf: 원
 */
public class Circle implements Graphic {
    private int x, y;
    private int radius;
    
    public Circle(int x, int y, int radius) {
        this.x = x;
        this.y = y;
        this.radius = radius;
    }
    
    @Override
    public void draw() {
        System.out.println("  🔵 Circle at (" + x + "," + y + 
                ") radius=" + radius);
    }
    
    @Override
    public void move(int dx, int dy) {
        x += dx;
        y += dy;
        System.out.println("  ↗️ Circle moved to (" + x + "," + y + ")");
    }
    
    @Override
    public void resize(double scale) {
        radius = (int) (radius * scale);
        System.out.println("  📏 Circle resized to radius=" + radius);
    }
}

/**
 * Leaf: 사각형
 */
public class Rectangle implements Graphic {
    private int x, y;
    private int width, height;
    
    public Rectangle(int x, int y, int width, int height) {
        this.x = x;
        this.y = y;
        this.width = width;
        this.height = height;
    }
    
    @Override
    public void draw() {
        System.out.println("  🟦 Rectangle at (" + x + "," + y + 
                ") size=" + width + "x" + height);
    }
    
    @Override
    public void move(int dx, int dy) {
        x += dx;
        y += dy;
        System.out.println("  ↗️ Rectangle moved to (" + x + "," + y + ")");
    }
    
    @Override
    public void resize(double scale) {
        width = (int) (width * scale);
        height = (int) (height * scale);
        System.out.println("  📏 Rectangle resized to " + width + "x" + height);
    }
}

/**
 * Composite: 그룹
 */
public class GraphicGroup implements Graphic {
    private List<Graphic> graphics;
    private String name;
    
    public GraphicGroup(String name) {
        this.name = name;
        this.graphics = new ArrayList<>();
    }
    
    public void add(Graphic graphic) {
        graphics.add(graphic);
    }
    
    public void remove(Graphic graphic) {
        graphics.remove(graphic);
    }
    
    @Override
    public void draw() {
        System.out.println("📦 Group: " + name);
        for (Graphic graphic : graphics) {
            graphic.draw();
        }
    }
    
    @Override
    public void move(int dx, int dy) {
        System.out.println("🚚 Moving group: " + name);
        for (Graphic graphic : graphics) {
            graphic.move(dx, dy);
        }
    }
    
    @Override
    public void resize(double scale) {
        System.out.println("🔍 Resizing group: " + name);
        for (Graphic graphic : graphics) {
            graphic.resize(scale);
        }
    }
}

/**
 * 사용 예제
 */
public class GraphicEditorExample {
    public static void main(String[] args) {
        // 개별 도형
        Circle circle1 = new Circle(10, 10, 5);
        Circle circle2 = new Circle(30, 30, 8);
        Rectangle rect1 = new Rectangle(50, 50, 20, 10);
        Rectangle rect2 = new Rectangle(100, 100, 30, 15);
        
        // 그룹 1: 원 그룹
        GraphicGroup circleGroup = new GraphicGroup("Circles");
        circleGroup.add(circle1);
        circleGroup.add(circle2);
        
        // 그룹 2: 사각형 그룹
        GraphicGroup rectGroup = new GraphicGroup("Rectangles");
        rectGroup.add(rect1);
        rectGroup.add(rect2);
        
        // 최상위 그룹
        GraphicGroup allShapes = new GraphicGroup("All Shapes");
        allShapes.add(circleGroup);
        allShapes.add(rectGroup);
        
        // 전체 그리기
        System.out.println("=== 전체 그리기 ===");
        allShapes.draw();
        
        // 전체 이동
        System.out.println("\n=== 전체 이동 (10, 20) ===");
        allShapes.move(10, 20);
        
        // 전체 크기 조정
        System.out.println("\n=== 전체 크기 2배 ===");
        allShapes.resize(2.0);
        
        // 결과 확인
        System.out.println("\n=== 변경 후 그리기 ===");
        allShapes.draw();
    }
}
```

---

### 예제 3: HTML 문서 구조 ⭐⭐

```java
/**
 * Component: HTML 요소
 */
public abstract class HtmlElement {
    protected String tag;
    
    public HtmlElement(String tag) {
        this.tag = tag;
    }
    
    public abstract String render(int depth);
    
    protected String indent(int depth) {
        return "  ".repeat(depth);
    }
    
    public void add(HtmlElement element) {
        throw new UnsupportedOperationException();
    }
    
    public void remove(HtmlElement element) {
        throw new UnsupportedOperationException();
    }
}

/**
 * Leaf: 텍스트 노드
 */
public class TextElement extends HtmlElement {
    private String text;
    
    public TextElement(String text) {
        super("text");
        this.text = text;
    }
    
    @Override
    public String render(int depth) {
        return indent(depth) + text + "\n";
    }
}

/**
 * Composite: 컨테이너 요소
 */
public class ContainerElement extends HtmlElement {
    private List<HtmlElement> children;
    
    public ContainerElement(String tag) {
        super(tag);
        this.children = new ArrayList<>();
    }
    
    @Override
    public void add(HtmlElement element) {
        children.add(element);
    }
    
    @Override
    public void remove(HtmlElement element) {
        children.remove(element);
    }
    
    @Override
    public String render(int depth) {
        StringBuilder sb = new StringBuilder();
        sb.append(indent(depth)).append("<").append(tag).append(">\n");
        
        for (HtmlElement child : children) {
            sb.append(child.render(depth + 1));
        }
        
        sb.append(indent(depth)).append("</").append(tag).append(">\n");
        return sb.toString();
    }
}

/**
 * 사용 예제
 */
public class HtmlExample {
    public static void main(String[] args) {
        // HTML 문서 구성
        ContainerElement html = new ContainerElement("html");
        
        ContainerElement head = new ContainerElement("head");
        ContainerElement title = new ContainerElement("title");
        title.add(new TextElement("My Page"));
        head.add(title);
        
        ContainerElement body = new ContainerElement("body");
        ContainerElement div = new ContainerElement("div");
        ContainerElement h1 = new ContainerElement("h1");
        h1.add(new TextElement("Welcome!"));
        
        ContainerElement p = new ContainerElement("p");
        p.add(new TextElement("This is a paragraph."));
        
        div.add(h1);
        div.add(p);
        body.add(div);
        
        html.add(head);
        html.add(body);
        
        // 렌더링
        System.out.println("=== HTML 렌더링 ===");
        System.out.println(html.render(0));
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **일관된 처리** | 개별/그룹 구분 없이 처리 | File과 Folder 동일 |
| **계층 구조** | 트리 구조 자연스럽게 표현 | 조직도, 파일 시스템 |
| **확장성** | 새 타입 추가 용이 | 새 도형 추가 |
| **재귀 처리** | 자동으로 재귀 탐색 | getTotalSalary() |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **제한 어려움** | 특정 타입만 추가하기 힘듦 | 타입 체크 추가 |
| **복잡도** | 설계가 복잡해질 수 있음 | 필요시에만 사용 |

---

## 7. 안티패턴

### ❌ 안티패턴 1: Leaf에서 Composite 메서드 노출

```java
// 잘못된 예
public class File extends FileSystemItem {
    @Override
    public void add(FileSystemItem item) {
        // 의미 없음!
        System.out.println("파일에는 추가 불가");
    }
}
```

**해결:**
```java
// Component에서 기본 구현 제공
public void add(FileSystemItem item) {
    throw new UnsupportedOperationException();
}
```

---

## 8. 핵심 정리

### 📌 Composite 패턴 체크리스트

```
✅ Component 인터페이스 정의
✅ Leaf 구현 (단말 노드)
✅ Composite 구현 (복합 노드)
✅ 재귀 구조로 처리
✅ 클라이언트는 타입 구분 불필요
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 계층 구조 표현 | ⭐⭐⭐ | 파일 시스템, 조직도 |
| 부분-전체 관계 | ⭐⭐⭐ | 트리 구조 |
| 일관된 처리 | ⭐⭐⭐ | 개별/그룹 동일 처리 |

### 💡 핵심 포인트

1. **부분과 전체를 동일하게**
2. **재귀적 구조**
3. **트리 패턴의 핵심**
4. **GUI, 파일 시스템에 필수**

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Proxy](./03-Proxy.md) | [다음: Facade →](05-Facade.md)**

</div>
