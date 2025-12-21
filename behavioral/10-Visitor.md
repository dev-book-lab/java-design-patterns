# Visitor Pattern (방문자 패턴)

> **"객체 구조와 알고리즘을 분리하자"**

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
// 문제 1: 새 기능 추가 시 모든 클래스 수정
public interface Shape {
    void draw();
    void calculateArea();
    // 새 기능 추가? 모든 구현체 수정!
}

public class Circle implements Shape {
    public void draw() { }
    public void calculateArea() { }
    // 새 메서드 추가 시 여기도 수정
}

public class Rectangle implements Shape {
    public void draw() { }
    public void calculateArea() { }
    // 여기도 수정
}

// export() 기능 추가하려면?
// → 모든 Shape 구현체 수정!

// 문제 2: 타입별 다른 처리
public class DocumentProcessor {
    public void process(List<Document> docs) {
        for (Document doc : docs) {
            if (doc instanceof PDFDocument) {
                processPDF((PDFDocument) doc);
            } else if (doc instanceof WordDocument) {
                processWord((WordDocument) doc);
            } else if (doc instanceof ExcelDocument) {
                processExcel((ExcelDocument) doc);
            }
            // instanceof 지옥!
        }
    }
}

// 문제 3: 여러 연산이 흩어짐
public class Employee {
    public void calculateSalary() { }
    public void generateReport() { }
    public void exportToXML() { }
    public void sendEmail() { }
    
    // 직원 클래스가 너무 많은 책임!
    // SRP 위반!
}

// 문제 4: 객체 구조 순회
public class FileSystem {
    public void calculateSize(FileSystemNode node) {
        if (node instanceof File) {
            return ((File) node).getSize();
        } else if (node instanceof Directory) {
            Directory dir = (Directory) node;
            int total = 0;
            for (FileSystemNode child : dir.getChildren()) {
                total += calculateSize(child); // 재귀
            }
            return total;
        }
    }
    
    // 새 연산 추가마다 비슷한 코드 반복!
}
```

### ⚡ 핵심 문제

1. **OCP 위반**: 새 연산 추가 시 모든 클래스 수정
2. **SRP 위반**: 클래스가 너무 많은 책임
3. **타입 체크**: instanceof 남발
4. **연산 분산**: 관련 연산이 여러 클래스에 흩어짐

---

## 2. 패턴 정의

### 📖 정의

> **객체 구조를 변경하지 않고 새로운 연산을 추가할 수 있게 하는 패턴. 연산을 수행할 객체(Visitor)를 별도로 만들어 객체 구조의 각 요소를 방문하며 연산을 수행한다.**

### 🎯 목적

- **연산 분리**: 알고리즘을 객체 구조에서 분리
- **OCP 준수**: 새 연산 추가 시 기존 코드 불변
- **SRP 준수**: 각 연산을 별도 클래스로
- **타입 안전**: instanceof 대신 메서드 오버로딩

### 💡 핵심 아이디어

```java
// Before: 객체에 연산 추가
public class Shape {
    void draw() { }
    void export() { } // 새 연산 추가!
}

// After: Visitor로 연산 분리
public interface ShapeVisitor {
    void visit(Circle circle);
    void visit(Rectangle rectangle);
}

public class ExportVisitor implements ShapeVisitor {
    void visit(Circle c) { /* Circle 내보내기 */ }
    void visit(Rectangle r) { /* Rectangle 내보내기 */ }
}
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌──────────────┐
│   Visitor    │  ← 방문자 인터페이스
├──────────────┤
│ + visit(A)   │
│ + visit(B)   │
└──────────────┘
       △
       │ implements
┌───────────────┐
│ConcreteVisitor│
├───────────────┤
│ + visit(A)    │
│ + visit(B)    │
└───────────────┘

┌──────────────┐
│   Element    │  ← 요소 인터페이스
├──────────────┤
│ + accept(v)  │
└──────────────┘
       △
       │ implements
       │
┌──────────────┐
│ElementA      │
├──────────────┤
│ + accept(v)  │────┐
└──────────────┘    │
                    │ visitor.visit(this)
┌──────────────┐    │
│ElementB      │    │
├──────────────┤    │
│ + accept(v)  │────┘
└──────────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **Visitor** | 방문자 인터페이스 | `ShapeVisitor` |
| **ConcreteVisitor** | 구체적 방문자 | `ExportVisitor` |
| **Element** | 요소 인터페이스 | `Shape` |
| **ConcreteElement** | 구체적 요소 | `Circle` |

---

## 4. 구현 방법

### 기본 구현: 도형 시스템 ⭐⭐⭐

```java
/**
 * Visitor: 도형 방문자
 */
public interface ShapeVisitor {
    void visit(Circle circle);
    void visit(Rectangle rectangle);
    void visit(Triangle triangle);
}

/**
 * Element: 도형
 */
public interface Shape {
    void accept(ShapeVisitor visitor);
}

/**
 * ConcreteElement 1: 원
 */
public class Circle implements Shape {
    private double radius;
    
    public Circle(double radius) {
        this.radius = radius;
    }
    
    public double getRadius() {
        return radius;
    }
    
    @Override
    public void accept(ShapeVisitor visitor) {
        visitor.visit(this); // Double Dispatch!
    }
}

/**
 * ConcreteElement 2: 사각형
 */
public class Rectangle implements Shape {
    private double width;
    private double height;
    
    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }
    
    public double getWidth() {
        return width;
    }
    
    public double getHeight() {
        return height;
    }
    
    @Override
    public void accept(ShapeVisitor visitor) {
        visitor.visit(this);
    }
}

/**
 * ConcreteElement 3: 삼각형
 */
public class Triangle implements Shape {
    private double base;
    private double height;
    
    public Triangle(double base, double height) {
        this.base = base;
        this.height = height;
    }
    
    public double getBase() {
        return base;
    }
    
    public double getHeight() {
        return height;
    }
    
    @Override
    public void accept(ShapeVisitor visitor) {
        visitor.visit(this);
    }
}

/**
 * ConcreteVisitor 1: 넓이 계산
 */
public class AreaCalculator implements ShapeVisitor {
    private double totalArea = 0;
    
    @Override
    public void visit(Circle circle) {
        double area = Math.PI * circle.getRadius() * circle.getRadius();
        System.out.println("⭕ 원 넓이: " + String.format("%.2f", area));
        totalArea += area;
    }
    
    @Override
    public void visit(Rectangle rectangle) {
        double area = rectangle.getWidth() * rectangle.getHeight();
        System.out.println("⬜ 사각형 넓이: " + String.format("%.2f", area));
        totalArea += area;
    }
    
    @Override
    public void visit(Triangle triangle) {
        double area = 0.5 * triangle.getBase() * triangle.getHeight();
        System.out.println("🔺 삼각형 넓이: " + String.format("%.2f", area));
        totalArea += area;
    }
    
    public double getTotalArea() {
        return totalArea;
    }
}

/**
 * ConcreteVisitor 2: XML 내보내기
 */
public class XMLExporter implements ShapeVisitor {
    private StringBuilder xml;
    
    public XMLExporter() {
        this.xml = new StringBuilder();
        xml.append("<shapes>\n");
    }
    
    @Override
    public void visit(Circle circle) {
        xml.append("  <circle radius=\"").append(circle.getRadius()).append("\"/>\n");
        System.out.println("📤 원 내보내기");
    }
    
    @Override
    public void visit(Rectangle rectangle) {
        xml.append("  <rectangle width=\"").append(rectangle.getWidth())
           .append("\" height=\"").append(rectangle.getHeight()).append("\"/>\n");
        System.out.println("📤 사각형 내보내기");
    }
    
    @Override
    public void visit(Triangle triangle) {
        xml.append("  <triangle base=\"").append(triangle.getBase())
           .append("\" height=\"").append(triangle.getHeight()).append("\"/>\n");
        System.out.println("📤 삼각형 내보내기");
    }
    
    public String getXML() {
        return xml.toString() + "</shapes>";
    }
}

/**
 * ConcreteVisitor 3: 그리기
 */
public class DrawVisitor implements ShapeVisitor {
    
    @Override
    public void visit(Circle circle) {
        System.out.println("🎨 원 그리기 (반지름: " + circle.getRadius() + ")");
    }
    
    @Override
    public void visit(Rectangle rectangle) {
        System.out.println("🎨 사각형 그리기 (" + rectangle.getWidth() + 
                "x" + rectangle.getHeight() + ")");
    }
    
    @Override
    public void visit(Triangle triangle) {
        System.out.println("🎨 삼각형 그리기 (밑변: " + triangle.getBase() + 
                ", 높이: " + triangle.getHeight() + ")");
    }
}

/**
 * 사용 예제
 */
public class VisitorExample {
    public static void main(String[] args) {
        // 도형 생성
        List<Shape> shapes = Arrays.asList(
            new Circle(5),
            new Rectangle(4, 6),
            new Triangle(3, 8),
            new Circle(3)
        );
        
        // 1. 넓이 계산
        System.out.println("=== 넓이 계산 ===");
        AreaCalculator areaCalc = new AreaCalculator();
        for (Shape shape : shapes) {
            shape.accept(areaCalc);
        }
        System.out.println("총 넓이: " + String.format("%.2f", areaCalc.getTotalArea()));
        
        // 2. XML 내보내기
        System.out.println("\n=== XML 내보내기 ===");
        XMLExporter xmlExporter = new XMLExporter();
        for (Shape shape : shapes) {
            shape.accept(xmlExporter);
        }
        System.out.println("\n" + xmlExporter.getXML());
        
        // 3. 그리기
        System.out.println("\n=== 그리기 ===");
        DrawVisitor drawer = new DrawVisitor();
        for (Shape shape : shapes) {
            shape.accept(drawer);
        }
    }
}
```

**실행 결과:**
```
=== 넓이 계산 ===
⭕ 원 넓이: 78.54
⬜ 사각형 넓이: 24.00
🔺 삼각형 넓이: 12.00
⭕ 원 넓이: 28.27
총 넓이: 142.81

=== XML 내보내기 ===
📤 원 내보내기
📤 사각형 내보내기
📤 삼각형 내보내기
📤 원 내보내기

<shapes>
  <circle radius="5.0"/>
  <rectangle width="4.0" height="6.0"/>
  <triangle base="3.0" height="8.0"/>
  <circle radius="3.0"/>
</shapes>

=== 그리기 ===
🎨 원 그리기 (반지름: 5.0)
🎨 사각형 그리기 (4.0x6.0)
🎨 삼각형 그리기 (밑변: 3.0, 높이: 8.0)
🎨 원 그리기 (반지름: 3.0)
```

---

## 5. 실전 예제

### 예제 1: 파일 시스템 ⭐⭐⭐

```java
/**
 * Visitor: 파일 시스템 방문자
 */
public interface FileSystemVisitor {
    void visit(File file);
    void visit(Directory directory);
}

/**
 * Element: 파일 시스템 노드
 */
public interface FileSystemNode {
    void accept(FileSystemVisitor visitor);
    String getName();
}

/**
 * ConcreteElement: 파일
 */
public class File implements FileSystemNode {
    private String name;
    private int size;
    
    public File(String name, int size) {
        this.name = name;
        this.size = size;
    }
    
    @Override
    public String getName() {
        return name;
    }
    
    public int getSize() {
        return size;
    }
    
    @Override
    public void accept(FileSystemVisitor visitor) {
        visitor.visit(this);
    }
}

/**
 * ConcreteElement: 디렉토리
 */
public class Directory implements FileSystemNode {
    private String name;
    private List<FileSystemNode> children;
    
    public Directory(String name) {
        this.name = name;
        this.children = new ArrayList<>();
    }
    
    public void add(FileSystemNode node) {
        children.add(node);
    }
    
    @Override
    public String getName() {
        return name;
    }
    
    public List<FileSystemNode> getChildren() {
        return children;
    }
    
    @Override
    public void accept(FileSystemVisitor visitor) {
        visitor.visit(this);
    }
}

/**
 * ConcreteVisitor: 크기 계산
 */
public class SizeCalculator implements FileSystemVisitor {
    private int totalSize = 0;
    
    @Override
    public void visit(File file) {
        totalSize += file.getSize();
        System.out.println("📄 " + file.getName() + ": " + file.getSize() + " KB");
    }
    
    @Override
    public void visit(Directory directory) {
        System.out.println("📁 " + directory.getName());
        for (FileSystemNode child : directory.getChildren()) {
            child.accept(this); // 재귀 방문
        }
    }
    
    public int getTotalSize() {
        return totalSize;
    }
}

/**
 * ConcreteVisitor: 파일 검색
 */
public class FileSearcher implements FileSystemVisitor {
    private String extension;
    private List<String> results;
    
    public FileSearcher(String extension) {
        this.extension = extension;
        this.results = new ArrayList<>();
    }
    
    @Override
    public void visit(File file) {
        if (file.getName().endsWith(extension)) {
            results.add(file.getName());
            System.out.println("🔍 발견: " + file.getName());
        }
    }
    
    @Override
    public void visit(Directory directory) {
        for (FileSystemNode child : directory.getChildren()) {
            child.accept(this);
        }
    }
    
    public List<String> getResults() {
        return results;
    }
}

/**
 * 사용 예제
 */
public class FileSystemVisitorExample {
    public static void main(String[] args) {
        // 파일 시스템 구성
        Directory root = new Directory("root");
        
        Directory docs = new Directory("documents");
        docs.add(new File("report.pdf", 100));
        docs.add(new File("memo.txt", 10));
        
        Directory images = new Directory("images");
        images.add(new File("photo1.jpg", 500));
        images.add(new File("photo2.jpg", 600));
        
        root.add(docs);
        root.add(images);
        root.add(new File("readme.txt", 5));
        
        // 크기 계산
        System.out.println("=== 크기 계산 ===");
        SizeCalculator sizeCalc = new SizeCalculator();
        root.accept(sizeCalc);
        System.out.println("\n총 크기: " + sizeCalc.getTotalSize() + " KB");
        
        // 파일 검색
        System.out.println("\n=== .txt 파일 검색 ===");
        FileSearcher searcher = new FileSearcher(".txt");
        root.accept(searcher);
        System.out.println("\n검색 결과: " + searcher.getResults());
    }
}
```

---

### 예제 2: 컴파일러 AST ⭐⭐⭐

```java
/**
 * Visitor: 표현식 방문자
 */
public interface ExpressionVisitor {
    int visit(NumberExpression expr);
    int visit(AddExpression expr);
    int visit(MultiplyExpression expr);
}

/**
 * Element: 표현식
 */
public interface Expression {
    int accept(ExpressionVisitor visitor);
}

/**
 * ConcreteElement: 숫자
 */
public class NumberExpression implements Expression {
    private int value;
    
    public NumberExpression(int value) {
        this.value = value;
    }
    
    public int getValue() {
        return value;
    }
    
    @Override
    public int accept(ExpressionVisitor visitor) {
        return visitor.visit(this);
    }
}

/**
 * ConcreteElement: 덧셈
 */
public class AddExpression implements Expression {
    private Expression left;
    private Expression right;
    
    public AddExpression(Expression left, Expression right) {
        this.left = left;
        this.right = right;
    }
    
    public Expression getLeft() {
        return left;
    }
    
    public Expression getRight() {
        return right;
    }
    
    @Override
    public int accept(ExpressionVisitor visitor) {
        return visitor.visit(this);
    }
}

/**
 * ConcreteElement: 곱셈
 */
public class MultiplyExpression implements Expression {
    private Expression left;
    private Expression right;
    
    public MultiplyExpression(Expression left, Expression right) {
        this.left = left;
        this.right = right;
    }
    
    public Expression getLeft() {
        return left;
    }
    
    public Expression getRight() {
        return right;
    }
    
    @Override
    public int accept(ExpressionVisitor visitor) {
        return visitor.visit(this);
    }
}

/**
 * ConcreteVisitor: 계산기
 */
public class Evaluator implements ExpressionVisitor {
    
    @Override
    public int visit(NumberExpression expr) {
        return expr.getValue();
    }
    
    @Override
    public int visit(AddExpression expr) {
        int left = expr.getLeft().accept(this);
        int right = expr.getRight().accept(this);
        System.out.println("  " + left + " + " + right + " = " + (left + right));
        return left + right;
    }
    
    @Override
    public int visit(MultiplyExpression expr) {
        int left = expr.getLeft().accept(this);
        int right = expr.getRight().accept(this);
        System.out.println("  " + left + " * " + right + " = " + (left * right));
        return left * right;
    }
}

/**
 * ConcreteVisitor: 프린터
 */
public class ExpressionPrinter implements ExpressionVisitor {
    
    @Override
    public int visit(NumberExpression expr) {
        System.out.print(expr.getValue());
        return 0; // 사용 안 함
    }
    
    @Override
    public int visit(AddExpression expr) {
        System.out.print("(");
        expr.getLeft().accept(this);
        System.out.print(" + ");
        expr.getRight().accept(this);
        System.out.print(")");
        return 0;
    }
    
    @Override
    public int visit(MultiplyExpression expr) {
        System.out.print("(");
        expr.getLeft().accept(this);
        System.out.print(" * ");
        expr.getRight().accept(this);
        System.out.print(")");
        return 0;
    }
}

/**
 * 사용 예제
 */
public class ExpressionVisitorExample {
    public static void main(String[] args) {
        // (2 + 3) * (4 + 5)
        Expression expr = new MultiplyExpression(
            new AddExpression(
                new NumberExpression(2),
                new NumberExpression(3)
            ),
            new AddExpression(
                new NumberExpression(4),
                new NumberExpression(5)
            )
        );
        
        // 표현식 출력
        System.out.println("=== 표현식 출력 ===");
        ExpressionPrinter printer = new ExpressionPrinter();
        expr.accept(printer);
        System.out.println();
        
        // 계산
        System.out.println("\n=== 계산 ===");
        Evaluator evaluator = new Evaluator();
        int result = expr.accept(evaluator);
        System.out.println("\n결과: " + result);
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **OCP 준수** | 새 연산 추가 용이 | Visitor만 추가 |
| **SRP 준수** | 연산을 별도 클래스로 | AreaCalculator |
| **연산 집중** | 관련 연산이 한 곳에 | XMLExporter |
| **타입 안전** | instanceof 제거 | 메서드 오버로딩 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **Element 변경 어려움** | 새 타입 추가 시 모든 Visitor 수정 | 안정적 구조에만 |
| **캡슐화 약화** | Element 내부 노출 | getter 최소화 |
| **복잡도** | Double Dispatch | 문서화 |

---

## 7. 안티패턴

### ❌ 안티패턴: Element가 자주 변함

```java
// 잘못된 예: Element가 자주 추가/삭제
interface Shape {
    void accept(Visitor v);
}

// Pentagon, Hexagon... 계속 추가
// → 모든 Visitor 수정!
```

**해결:**
```java
// 안정적인 구조에만 Visitor 사용
// 자주 변하는 구조는 Strategy 등 고려
```

---

## 8. 핵심 정리

### 📌 Visitor 패턴 체크리스트

```
✅ Visitor 인터페이스
✅ Element 인터페이스
✅ accept() 메서드
✅ visit() 오버로딩
✅ ConcreteVisitor 구현
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 안정적 구조 | ⭐⭐⭐ | Element 변경 적음 |
| 다양한 연산 | ⭐⭐⭐ | 연산 분리 |
| 복잡한 구조 순회 | ⭐⭐⭐ | AST, 파일 시스템 |
| 타입별 처리 | ⭐⭐⭐ | instanceof 제거 |

### 💡 핵심 포인트

1. **Double Dispatch**
2. **연산과 구조 분리**
3. **안정적 구조에만**
4. **타입 안전**

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Memento](./09-Memento.md) | [다음: Interpreter →](11-Interpreter.md)**

</div>
