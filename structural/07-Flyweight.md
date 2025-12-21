# Flyweight Pattern (플라이웨이트 패턴)

> **"공유를 통해 메모리 사용을 최소화하자"**

[← 이전: Bridge](06-Bridge.md) | [목차로 돌아가기](../README.md)

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
// 문제 1: 수많은 객체로 메모리 부족
public class Game {
    private List<Tree> trees = new ArrayList<>();
    
    public void createForest() {
        for (int i = 0; i < 1000000; i++) {
            // 100만 그루의 나무 생성!
            Tree tree = new Tree(
                "Oak",           // 나무 종류
                "Green",         // 색상
                "oak_texture.png" // 텍스처
            );
            tree.setPosition(random(), random());
            trees.add(tree);
        }
        // 100만 개 객체 = 수백 MB 메모리!
    }
}

public class Tree {
    private String name;     // 모두 동일
    private String color;    // 모두 동일
    private String texture;  // 모두 동일
    private int x;           // 다름
    private int y;           // 다름
    
    // 같은 데이터를 100만 번 복사!
}

// 문제 2: 텍스트 에디터의 문자 객체
public class TextEditor {
    private List<Character> characters = new ArrayList<>();
    
    public void addText(String text) {
        for (char c : text.toCharArray()) {
            // 각 문자마다 객체 생성
            Character character = new Character(
                c,               // 글자
                "Arial",         // 폰트 (대부분 동일)
                12,              // 크기 (대부분 동일)
                "Black"          // 색상 (대부분 동일)
            );
            characters.add(character);
        }
    }
    
    // "Hello"를 100번 쓰면 500개 'e' 객체!
    // 같은 'e'를 왜 500번 만들까?
}

// 문제 3: 아이콘 캐시 없음
public class FileExplorer {
    public void displayFiles(List<File> files) {
        for (File file : files) {
            Icon icon;
            if (file.getExtension().equals("pdf")) {
                icon = new PDFIcon(); // 매번 생성!
            } else if (file.getExtension().equals("jpg")) {
                icon = new JPGIcon(); // 매번 생성!
            }
            display(icon, file.getName());
        }
        // PDF 100개 = PDF 아이콘 100개 생성!
    }
}
```

### ⚡ 핵심 문제

1. **메모리 낭비**: 동일한 데이터를 반복 저장
2. **성능 저하**: 불필요한 객체 생성
3. **GC 부담**: 많은 객체로 인한 GC 오버헤드
4. **확장성 제약**: 메모리 한계로 객체 수 제한

---

## 2. 패턴 정의

### 📖 정의

> **공유 가능한 객체를 통해 메모리 사용량을 줄이는 패턴. 많은 수의 유사한 객체를 효율적으로 지원한다.**

### 🎯 목적

- **메모리 절약**: 공유로 메모리 사용 최소화
- **성능 향상**: 객체 생성 비용 절감
- **확장성**: 더 많은 객체 생성 가능
- **캐싱**: 재사용 가능한 객체 관리

### 💡 핵심 아이디어

```java
// Before: 100만 개 Tree 객체
for (int i = 0; i < 1000000; i++) {
    new Tree("Oak", "Green", "texture.png", x, y);
}
// 100만 개 x 1KB = 1GB

// After: 1개 TreeType 객체 공유
TreeType oakType = factory.getTreeType("Oak", "Green", "texture.png");
for (int i = 0; i < 1000000; i++) {
    new Tree(oakType, x, y); // 참조만 저장
}
// 1개 x 1KB + (100만 개 x 8bytes) = 8MB
```

### 📊 Intrinsic vs Extrinsic State

| 구분 | 설명 | 공유 가능? | 예시 |
|------|------|-----------|------|
| **Intrinsic State** | 객체 내부 상태 (불변) | ✅ 공유 | 나무 종류, 색상, 텍스처 |
| **Extrinsic State** | 객체 외부 상태 (가변) | ❌ 개별 | 나무 위치 (x, y) |

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌─────────────────┐
│FlyweightFactory │  ← 공유 객체 관리
├─────────────────┤
│ - flyweights    │
│ + getFlyweight()│
└─────────────────┘
         │
         │ creates & manages
         ▼
┌─────────────────┐
│   Flyweight     │  ← 공유 객체
├─────────────────┤
│ - intrinsic     │  ← 내부 상태 (공유)
│ + operation(    │
│   extrinsic)    │  ← 외부 상태 (개별)
└─────────────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **Flyweight** | 공유 가능한 객체 | `TreeType` |
| **FlyweightFactory** | Flyweight 생성/관리 | `TreeFactory` |
| **Client** | 외부 상태 유지 | `Tree` (x, y 좌표) |

---

## 4. 구현 방법

### 기본 구현: 게임 포레스트 ⭐⭐⭐

```java
/**
 * Flyweight: 나무 타입 (공유 가능)
 */
public class TreeType {
    private String name;
    private String color;
    private String texture;
    
    public TreeType(String name, String color, String texture) {
        this.name = name;
        this.color = color;
        this.texture = texture;
        System.out.println("🌲 TreeType 생성: " + name + " (" + color + ")");
    }
    
    public void draw(int x, int y) {
        System.out.println("  Drawing " + name + " tree at (" + x + "," + y + 
                ") with " + color + " color");
    }
    
    public String getName() {
        return name;
    }
}

/**
 * FlyweightFactory: 나무 타입 팩토리
 */
public class TreeFactory {
    private static Map<String, TreeType> treeTypes = new HashMap<>();
    
    public static TreeType getTreeType(String name, String color, String texture) {
        // 키 생성
        String key = name + "-" + color + "-" + texture;
        
        // 캐시 확인
        TreeType type = treeTypes.get(key);
        
        if (type == null) {
            // 없으면 생성 후 캐시
            type = new TreeType(name, color, texture);
            treeTypes.put(key, type);
            System.out.println("✅ 캐시에 저장: " + key);
        } else {
            System.out.println("♻️ 캐시에서 재사용: " + key);
        }
        
        return type;
    }
    
    public static int getCacheSize() {
        return treeTypes.size();
    }
}

/**
 * Client: 나무 (외부 상태 포함)
 */
public class Tree {
    private int x;
    private int y;
    private TreeType type; // 공유 객체 참조
    
    public Tree(int x, int y, TreeType type) {
        this.x = x;
        this.y = y;
        this.type = type;
    }
    
    public void draw() {
        type.draw(x, y);
    }
}

/**
 * 숲 시뮬레이션
 */
public class Forest {
    private List<Tree> trees = new ArrayList<>();
    
    public void plantTree(int x, int y, String name, String color, String texture) {
        TreeType type = TreeFactory.getTreeType(name, color, texture);
        Tree tree = new Tree(x, y, type);
        trees.add(tree);
    }
    
    public void draw() {
        System.out.println("\n=== 숲 그리기 ===");
        for (Tree tree : trees) {
            tree.draw();
        }
    }
    
    public void printStats() {
        System.out.println("\n=== 메모리 통계 ===");
        System.out.println("총 나무 수: " + trees.size());
        System.out.println("TreeType 캐시 크기: " + TreeFactory.getCacheSize());
        
        // 메모리 절약 계산
        long withoutFlyweight = trees.size() * 1024; // 1KB per tree
        long withFlyweight = TreeFactory.getCacheSize() * 1024 + trees.size() * 16;
        // 16 bytes per Tree (x, y, reference)
        
        System.out.println("Flyweight 없이: ~" + withoutFlyweight / 1024 + " KB");
        System.out.println("Flyweight 사용: ~" + withFlyweight / 1024 + " KB");
        System.out.println("절약: ~" + (withoutFlyweight - withFlyweight) / 1024 + " KB");
    }
}

/**
 * 사용 예제
 */
public class FlyweightExample {
    public static void main(String[] args) {
        Forest forest = new Forest();
        
        System.out.println("=== 나무 심기 ===");
        
        // 같은 종류의 나무를 여러 번 심기
        forest.plantTree(10, 20, "Oak", "Green", "oak.png");
        forest.plantTree(15, 25, "Oak", "Green", "oak.png"); // 재사용!
        forest.plantTree(20, 30, "Oak", "Green", "oak.png"); // 재사용!
        
        forest.plantTree(30, 40, "Pine", "Dark Green", "pine.png");
        forest.plantTree(35, 45, "Pine", "Dark Green", "pine.png"); // 재사용!
        
        forest.plantTree(50, 60, "Birch", "White", "birch.png");
        
        // 숲 그리기
        forest.draw();
        
        // 통계 출력
        forest.printStats();
    }
}
```

**실행 결과:**
```
=== 나무 심기 ===
🌲 TreeType 생성: Oak (Green)
✅ 캐시에 저장: Oak-Green-oak.png
🌲 TreeType 생성: Oak (Green)
♻️ 캐시에서 재사용: Oak-Green-oak.png
🌲 TreeType 생성: Oak (Green)
♻️ 캐시에서 재사용: Oak-Green-oak.png
🌲 TreeType 생성: Pine (Dark Green)
✅ 캐시에 저장: Pine-Dark Green-pine.png
🌲 TreeType 생성: Pine (Dark Green)
♻️ 캐시에서 재사용: Pine-Dark Green-pine.png
🌲 TreeType 생성: Birch (White)
✅ 캐시에 저장: Birch-White-birch.png

=== 숲 그리기 ===
  Drawing Oak tree at (10,20) with Green color
  Drawing Oak tree at (15,25) with Green color
  Drawing Oak tree at (20,30) with Green color
  Drawing Pine tree at (30,40) with Dark Green color
  Drawing Pine tree at (35,45) with Dark Green color
  Drawing Birch tree at (50,60) with White color

=== 메모리 통계 ===
총 나무 수: 6
TreeType 캐시 크기: 3
Flyweight 없이: ~6 KB
Flyweight 사용: ~3 KB
절약: ~3 KB
```

---

## 5. 실전 예제

### 예제 1: 텍스트 에디터 ⭐⭐⭐

```java
/**
 * Flyweight: 문자 스타일
 */
public class CharacterStyle {
    private String font;
    private int size;
    private String color;
    
    public CharacterStyle(String font, int size, String color) {
        this.font = font;
        this.size = size;
        this.color = color;
    }
    
    public void render(char character, int x, int y) {
        System.out.println("  '" + character + "' at (" + x + "," + y + 
                ") [" + font + ", " + size + "pt, " + color + "]");
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof CharacterStyle)) return false;
        CharacterStyle that = (CharacterStyle) o;
        return size == that.size && 
               font.equals(that.font) && 
               color.equals(that.color);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(font, size, color);
    }
}

/**
 * FlyweightFactory: 스타일 팩토리
 */
public class StyleFactory {
    private static Map<CharacterStyle, CharacterStyle> styles = new HashMap<>();
    
    public static CharacterStyle getStyle(String font, int size, String color) {
        CharacterStyle style = new CharacterStyle(font, size, color);
        
        return styles.computeIfAbsent(style, s -> {
            System.out.println("🎨 새 스타일 생성: " + font + " " + size + "pt " + color);
            return s;
        });
    }
    
    public static int getStyleCount() {
        return styles.size();
    }
}

/**
 * Client: 문자
 */
public class Character {
    private char value;
    private int x;
    private int y;
    private CharacterStyle style; // 공유
    
    public Character(char value, int x, int y, CharacterStyle style) {
        this.value = value;
        this.x = x;
        this.y = y;
        this.style = style;
    }
    
    public void render() {
        style.render(value, x, y);
    }
}

/**
 * 텍스트 에디터
 */
public class TextEditor {
    private List<Character> characters = new ArrayList<>();
    
    public void addText(String text, String font, int size, String color) {
        CharacterStyle style = StyleFactory.getStyle(font, size, color);
        
        int x = characters.size() * 10; // 간단히 위치 계산
        
        for (char c : text.toCharArray()) {
            characters.add(new Character(c, x, 0, style));
            x += 10;
        }
    }
    
    public void render() {
        System.out.println("\n=== 텍스트 렌더링 ===");
        for (Character c : characters) {
            c.render();
        }
    }
    
    public void printStats() {
        System.out.println("\n=== 메모리 통계 ===");
        System.out.println("총 문자 수: " + characters.size());
        System.out.println("스타일 캐시 크기: " + StyleFactory.getStyleCount());
    }
}

/**
 * 사용 예제
 */
public class TextEditorExample {
    public static void main(String[] args) {
        TextEditor editor = new TextEditor();
        
        System.out.println("=== 텍스트 입력 ===");
        editor.addText("Hello ", "Arial", 12, "Black");
        editor.addText("World", "Arial", 12, "Black"); // 스타일 재사용!
        editor.addText("!", "Arial", 16, "Red");
        
        editor.render();
        editor.printStats();
    }
}
```

---

### 예제 2: 아이콘 캐시 ⭐⭐

```java
/**
 * Flyweight: 아이콘
 */
public class Icon {
    private String type;
    private byte[] imageData;
    
    public Icon(String type) {
        this.type = type;
        // 실제로는 파일에서 로드
        this.imageData = new byte[1024]; // 1KB
        System.out.println("📦 아이콘 로드: " + type + ".png");
    }
    
    public void draw(int x, int y, String fileName) {
        System.out.println("  🖼️ " + type + " 아이콘 그리기: " + fileName + 
                " at (" + x + "," + y + ")");
    }
}

/**
 * FlyweightFactory: 아이콘 팩토리
 */
public class IconFactory {
    private static Map<String, Icon> icons = new HashMap<>();
    
    public static Icon getIcon(String type) {
        return icons.computeIfAbsent(type, Icon::new);
    }
    
    public static int getCacheSize() {
        return icons.size();
    }
}

/**
 * Client: 파일
 */
public class File {
    private String name;
    private String extension;
    private int x;
    private int y;
    
    public File(String name, String extension, int x, int y) {
        this.name = name;
        this.extension = extension;
        this.x = x;
        this.y = y;
    }
    
    public void display() {
        Icon icon = IconFactory.getIcon(extension);
        icon.draw(x, y, name);
    }
}

/**
 * 파일 탐색기
 */
public class FileExplorer {
    private List<File> files = new ArrayList<>();
    
    public void addFile(String name, String extension, int x, int y) {
        files.add(new File(name, extension, x, y));
    }
    
    public void displayFiles() {
        System.out.println("\n=== 파일 표시 ===");
        for (File file : files) {
            file.display();
        }
    }
    
    public void printStats() {
        System.out.println("\n=== 메모리 통계 ===");
        System.out.println("총 파일 수: " + files.size());
        System.out.println("아이콘 캐시 크기: " + IconFactory.getCacheSize());
    }
}

/**
 * 사용 예제
 */
public class FileExplorerExample {
    public static void main(String[] args) {
        FileExplorer explorer = new FileExplorer();
        
        System.out.println("=== 파일 추가 ===");
        explorer.addFile("report.pdf", "pdf", 10, 10);
        explorer.addFile("document.pdf", "pdf", 10, 30); // PDF 아이콘 재사용!
        explorer.addFile("photo.jpg", "jpg", 10, 50);
        explorer.addFile("image.jpg", "jpg", 10, 70); // JPG 아이콘 재사용!
        explorer.addFile("data.xlsx", "xlsx", 10, 90);
        
        explorer.displayFiles();
        explorer.printStats();
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **메모리 절약** | 공유로 메모리 사용 최소화 | 100만 나무 → 3개 타입 |
| **성능 향상** | 객체 생성 비용 절감 | 캐시에서 재사용 |
| **확장성** | 더 많은 객체 지원 | 메모리 한계 극복 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **복잡도** | 내부/외부 상태 분리 필요 | 명확한 설계 |
| **스레드 안전** | 공유 객체 동기화 필요 | Immutable 설계 |
| **적용 제약** | 공유 가능한 경우만 | 패턴 선택 신중히 |

---

## 7. 안티패턴

### ❌ 안티패턴: 모든 상태를 공유

```java
// 잘못된 예: 외부 상태도 공유
public class BadTree {
    private String type;
    private int x, y; // 위치도 공유? NO!
}
// 모든 나무가 같은 위치에!
```

---

## 8. 핵심 정리

### 📌 Flyweight 패턴 체크리스트

```
✅ 내부/외부 상태 구분
✅ Flyweight 클래스 정의
✅ FlyweightFactory 구현
✅ 캐싱 메커니즘
✅ 불변 객체 설계
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 많은 유사 객체 | ⭐⭐⭐ | 메모리 절약 |
| 메모리 부족 | ⭐⭐⭐ | 효율성 |
| 공유 가능한 상태 | ⭐⭐⭐ | 재사용 |

### 💡 핵심 포인트

1. **내부 상태 공유**
2. **외부 상태는 개별**
3. **불변 객체 설계**
4. **팩토리로 관리**

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Bridge](06-Bridge.md)**

</div>
