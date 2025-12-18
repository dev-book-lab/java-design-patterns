# Prototype Pattern (프로토타입 패턴)

> **"기존 객체를 복제하여 새로운 객체를 생성하자"**

[← 이전: Builder](./03-Builder.md) | [목차로 돌아가기](../README.md) | [다음: Abstract Factory →](./05-AbstractFactory.md)

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
// 문제 1: 객체 생성 비용이 너무 큼
public class DatabaseConnection {
    private Connection connection;
    private Map<String, Object> cache;
    
    public DatabaseConnection() {
        // 1. DB 연결 (비용 큼)
        this.connection = DriverManager.getConnection("jdbc:...");
        
        // 2. 캐시 초기화 (비용 큼)
        this.cache = new HashMap<>();
        loadCache(); // 수천 개의 데이터 로드
        
        // 3. 설정 로드 (비용 큼)
        loadConfiguration();
    }
    
    // 똑같은 설정으로 10개 생성하면? → 10번 반복!
}

// 문제 2: 복잡한 초기화 과정
public class GameCharacter {
    private String name;
    private int level;
    private List<Item> inventory;
    private Map<String, Skill> skills;
    private Equipment equipment;
    
    public GameCharacter(String name) {
        this.name = name;
        this.level = 1;
        
        // 복잡한 초기화
        this.inventory = new ArrayList<>();
        initializeStarterItems();      // 시작 아이템 지급
        
        this.skills = new HashMap<>();
        initializeBasicSkills();        // 기본 스킬 설정
        
        this.equipment = new Equipment();
        equipStarterGear();             // 시작 장비 착용
        
        // 매번 이 과정을 반복해야 함!
    }
}

// 문제 3: 동적 타입 생성
public class ShapeFactory {
    public Shape createShape(String type) {
        if (type.equals("CIRCLE")) {
            Circle circle = new Circle();
            // 기본 설정 많음...
            return circle;
        } else if (type.equals("RECTANGLE")) {
            Rectangle rect = new Rectangle();
            // 기본 설정 많음...
            return rect;
        }
        // 타입이 런타임에 결정되는데 매번 new!
    }
}

// 문제 4: 상태를 가진 객체 복사
public class Document {
    private String title;
    private String content;
    private List<String> tags;
    private Author author;
    
    // 이 문서를 복사해서 비슷한 문서 만들고 싶은데...
    public Document copyDocument() {
        Document copy = new Document();
        copy.title = this.title + " (복사본)";
        copy.content = this.content;
        copy.tags = new ArrayList<>(this.tags); // 수동 복사...
        copy.author = this.author; // 얕은 복사? 깊은 복사?
        return copy;
        // 필드 추가되면 여기도 수정 필요!
    }
}
```

### ⚡ 핵심 문제

1. **생성 비용**: 객체 생성이 무거운 경우
2. **복잡한 초기화**: 초기화 로직이 복잡한 경우
3. **상태 보존**: 기존 객체의 상태를 유지하며 복사
4. **동적 타입**: 런타임에 타입이 결정되는 경우

---

## 2. 패턴 정의

### 📖 정의

> **기존 인스턴스를 프로토타입으로 사용하여 새로운 객체를 생성하는 패턴. 원본 객체를 복제(Clone)하여 새 객체를 만든다.**

### 🎯 목적

- **생성 비용 절감**: 무거운 초기화 과정을 한 번만
- **복잡한 객체 복사**: 상태를 그대로 복제
- **동적 타입 생성**: 런타임에 객체 타입 결정
- **독립적인 복사본**: 원본과 별개의 객체

### 💡 핵심 아이디어

```java
// Before: 매번 새로 생성
Shape shape1 = new Circle();
shape1.setRadius(10);
shape1.setColor("red");
// ... 복잡한 설정

Shape shape2 = new Circle(); // 또 처음부터!
shape2.setRadius(10);
shape2.setColor("red");
// ... 동일한 설정 반복

// After: 복제
Shape shape1 = new Circle();
shape1.setRadius(10);
shape1.setColor("red");

Shape shape2 = shape1.clone(); // 복제로 간단히!
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌─────────────────────┐
│   Prototype         │  ← 프로토타입 인터페이스
├─────────────────────┤
│ + clone(): Prototype│
└─────────────────────┘
          △
          │ implements
          │
┌─────────────────────┐
│ ConcretePrototype1  │  ← 구체적인 프로토타입
├─────────────────────┤
│ - field1            │
│ - field2            │
│ + clone()           │  ← 자신을 복제
└─────────────────────┘

┌─────────────────────┐
│      Client         │
├─────────────────────┤
│ - prototype         │
│ + operation()       │
└─────────────────────┘
          │
          │ uses
          ▼
    prototype.clone()
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **Prototype** | clone() 메서드 선언 | `Cloneable` |
| **ConcretePrototype** | clone() 구현 | `Circle`, `Rectangle` |
| **Client** | 복제를 통해 객체 생성 | `ShapeCache` |

---

## 4. 구현 방법

### 방법 1: Cloneable 인터페이스 사용 ⭐⭐⭐

```java
/**
 * 기본 Prototype: Shape
 */
public abstract class Shape implements Cloneable {
    private String id;
    protected String type;
    private int x;
    private int y;
    private String color;
    
    public abstract void draw();
    
    // clone() 메서드 구현
    @Override
    public Shape clone() {
        try {
            return (Shape) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException("Clone not supported", e);
        }
    }
    
    // Getters and Setters
    public String getId() { return id; }
    public void setId(String id) { this.id = id; }
    public String getType() { return type; }
    public int getX() { return x; }
    public void setX(int x) { this.x = x; }
    public int getY() { return y; }
    public void setY(int y) { this.y = y; }
    public String getColor() { return color; }
    public void setColor(String color) { this.color = color; }
}

/**
 * ConcretePrototype 1: Circle
 */
public class Circle extends Shape {
    private int radius;
    
    public Circle() {
        type = "Circle";
    }
    
    @Override
    public void draw() {
        System.out.println("Drawing Circle: " +
                "color=" + getColor() +
                ", x=" + getX() +
                ", y=" + getY() +
                ", radius=" + radius);
    }
    
    public int getRadius() { return radius; }
    public void setRadius(int radius) { this.radius = radius; }
}

/**
 * ConcretePrototype 2: Rectangle
 */
public class Rectangle extends Shape {
    private int width;
    private int height;
    
    public Rectangle() {
        type = "Rectangle";
    }
    
    @Override
    public void draw() {
        System.out.println("Drawing Rectangle: " +
                "color=" + getColor() +
                ", x=" + getX() +
                ", y=" + getY() +
                ", width=" + width +
                ", height=" + height);
    }
    
    public int getWidth() { return width; }
    public void setWidth(int width) { this.width = width; }
    public int getHeight() { return height; }
    public void setHeight(int height) { this.height = height; }
}

/**
 * Client: ShapeCache
 * 프로토타입을 관리하고 복제 제공
 */
public class ShapeCache {
    private static Map<String, Shape> shapeMap = new HashMap<>();
    
    // 초기화: 기본 프로토타입 생성
    public static void loadCache() {
        Circle circle = new Circle();
        circle.setId("1");
        circle.setColor("Red");
        circle.setRadius(10);
        shapeMap.put(circle.getId(), circle);
        
        Rectangle rectangle = new Rectangle();
        rectangle.setId("2");
        rectangle.setColor("Blue");
        rectangle.setWidth(20);
        rectangle.setHeight(10);
        shapeMap.put(rectangle.getId(), rectangle);
        
        System.out.println("프로토타입 캐시 초기화 완료");
    }
    
    // 복제된 객체 반환
    public static Shape getShape(String shapeId) {
        Shape cachedShape = shapeMap.get(shapeId);
        return cachedShape.clone();
    }
}

// 사용 예제
public class PrototypeExample {
    public static void main(String[] args) {
        // 캐시 초기화 (한 번만)
        ShapeCache.loadCache();
        
        // 복제를 통한 객체 생성
        System.out.println("\n=== 복제 1 ===");
        Shape clonedCircle1 = ShapeCache.getShape("1");
        System.out.println("Type: " + clonedCircle1.getType());
        clonedCircle1.draw();
        
        System.out.println("\n=== 복제 2 ===");
        Shape clonedCircle2 = ShapeCache.getShape("1");
        clonedCircle2.setColor("Green"); // 독립적으로 수정
        clonedCircle2.draw();
        
        System.out.println("\n=== 복제 3 ===");
        Shape clonedRectangle = ShapeCache.getShape("2");
        clonedRectangle.draw();
        
        // 원본은 변경되지 않음
        System.out.println("\n=== 원본 확인 ===");
        Shape original = ShapeCache.getShape("1");
        original.draw(); // 여전히 Red
        
        // 객체 독립성 확인
        System.out.println("\n=== 독립성 확인 ===");
        System.out.println("clonedCircle1 == clonedCircle2: " + 
                (clonedCircle1 == clonedCircle2)); // false
    }
}
```

**실행 결과:**
```
프로토타입 캐시 초기화 완료

=== 복제 1 ===
Type: Circle
Drawing Circle: color=Red, x=0, y=0, radius=10

=== 복제 2 ===
Drawing Circle: color=Green, x=0, y=0, radius=10

=== 복제 3 ===
Drawing Rectangle: color=Blue, x=0, y=0, width=20, height=10

=== 원본 확인 ===
Drawing Circle: color=Red, x=0, y=0, radius=10

=== 독립성 확인 ===
clonedCircle1 == clonedCircle2: false
```

---

### 방법 2: 깊은 복사 (Deep Copy) ⭐⭐⭐

```java
/**
 * 참조 타입을 포함한 객체의 깊은 복사
 */
public class Student implements Cloneable {
    private String name;
    private int age;
    private Address address;  // 참조 타입!
    private List<String> courses; // 참조 타입!
    
    public Student(String name, int age, Address address) {
        this.name = name;
        this.age = age;
        this.address = address;
        this.courses = new ArrayList<>();
    }
    
    // 얕은 복사 (Shallow Copy)
    @Override
    public Student clone() {
        try {
            return (Student) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
    
    // 깊은 복사 (Deep Copy)
    public Student deepClone() {
        try {
            Student cloned = (Student) super.clone();
            
            // 참조 타입은 새로 생성
            cloned.address = new Address(
                this.address.getCity(),
                this.address.getStreet()
            );
            
            cloned.courses = new ArrayList<>(this.courses);
            
            return cloned;
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
    
    public void addCourse(String course) {
        this.courses.add(course);
    }
    
    @Override
    public String toString() {
        return "Student{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", address=" + address +
                ", courses=" + courses +
                '}';
    }
    
    // Getters
    public Address getAddress() { return address; }
    public List<String> getCourses() { return courses; }
}

/**
 * 주소 클래스
 */
public class Address {
    private String city;
    private String street;
    
    public Address(String city, String street) {
        this.city = city;
        this.street = street;
    }
    
    @Override
    public String toString() {
        return city + ", " + street;
    }
    
    public String getCity() { return city; }
    public void setCity(String city) { this.city = city; }
    public String getStreet() { return street; }
    public void setStreet(String street) { this.street = street; }
}

// 사용 예제: 얕은 복사 vs 깊은 복사
public class DeepCopyExample {
    public static void main(String[] args) {
        // 원본 객체
        Address address = new Address("Seoul", "Gangnam");
        Student original = new Student("John", 20, address);
        original.addCourse("Math");
        original.addCourse("Physics");
        
        System.out.println("=== 원본 ===");
        System.out.println(original);
        
        // 얕은 복사
        System.out.println("\n=== 얕은 복사 (Shallow Copy) ===");
        Student shallowCopy = original.clone();
        
        // 복사본 수정
        shallowCopy.addCourse("Chemistry");
        shallowCopy.getAddress().setCity("Busan");
        
        System.out.println("복사본: " + shallowCopy);
        System.out.println("원본: " + original);
        System.out.println("⚠️ 주소가 공유됨! 원본도 Busan으로 변경됨!");
        System.out.println("⚠️ courses도 공유됨! 원본에도 Chemistry 추가됨!");
        
        // 깊은 복사
        System.out.println("\n=== 깊은 복사 (Deep Copy) ===");
        Student original2 = new Student("Jane", 22, 
                new Address("Seoul", "Gangnam"));
        original2.addCourse("Math");
        
        Student deepCopy = original2.deepClone();
        
        // 복사본 수정
        deepCopy.addCourse("English");
        deepCopy.getAddress().setCity("Incheon");
        
        System.out.println("복사본: " + deepCopy);
        System.out.println("원본: " + original2);
        System.out.println("✅ 주소가 독립적! 원본은 Seoul 유지!");
        System.out.println("✅ courses도 독립적! 원본은 Math만!");
    }
}
```

**실행 결과:**
```
=== 원본 ===
Student{name='John', age=20, address=Seoul, Gangnam, courses=[Math, Physics]}

=== 얕은 복사 (Shallow Copy) ===
복사본: Student{name='John', age=20, address=Busan, Gangnam, courses=[Math, Physics, Chemistry]}
원본: Student{name='John', age=20, address=Busan, Gangnam, courses=[Math, Physics, Chemistry]}
⚠️ 주소가 공유됨! 원본도 Busan으로 변경됨!
⚠️ courses도 공유됨! 원본에도 Chemistry 추가됨!

=== 깊은 복사 (Deep Copy) ===
복사본: Student{name='Jane', age=22, address=Incheon, Gangnam, courses=[Math, English]}
원본: Student{name='Jane', age=22, address=Seoul, Gangnam, courses=[Math]}
✅ 주소가 독립적! 원본은 Seoul 유지!
✅ courses도 독립적! 원본은 Math만!
```

---

### 방법 3: 복사 생성자 (Copy Constructor) ⭐⭐

```java
/**
 * 복사 생성자를 사용한 구현
 * - Cloneable보다 안전하고 명확
 */
public class Book {
    private String title;
    private String author;
    private int pages;
    private List<String> chapters;
    
    // 일반 생성자
    public Book(String title, String author, int pages) {
        this.title = title;
        this.author = author;
        this.pages = pages;
        this.chapters = new ArrayList<>();
    }
    
    // 복사 생성자
    public Book(Book other) {
        this.title = other.title;
        this.author = other.author;
        this.pages = other.pages;
        this.chapters = new ArrayList<>(other.chapters); // 깊은 복사
    }
    
    public void addChapter(String chapter) {
        this.chapters.add(chapter);
    }
    
    @Override
    public String toString() {
        return "Book{" +
                "title='" + title + '\'' +
                ", author='" + author + '\'' +
                ", pages=" + pages +
                ", chapters=" + chapters +
                '}';
    }
}

// 사용 예제
public class CopyConstructorExample {
    public static void main(String[] args) {
        // 원본
        Book original = new Book("Design Patterns", "GoF", 500);
        original.addChapter("Chapter 1");
        original.addChapter("Chapter 2");
        
        System.out.println("원본: " + original);
        
        // 복사 생성자로 복제
        Book copy = new Book(original);
        copy.addChapter("Chapter 3");
        
        System.out.println("\n복사본: " + copy);
        System.out.println("원본: " + original);
        System.out.println("\n✅ 독립적으로 복제됨!");
    }
}
```

---

## 5. 실전 예제

### 예제 1: 게임 캐릭터 시스템 ⭐⭐⭐

```java
/**
 * 게임 캐릭터 - 복잡한 초기화가 필요한 객체
 */
public class GameCharacter implements Cloneable {
    private String name;
    private String characterClass;
    private int level;
    private int health;
    private int mana;
    
    // 복잡한 객체들
    private Equipment equipment;
    private List<Skill> skills;
    private Inventory inventory;
    
    // 기본 생성자 (복잡한 초기화)
    public GameCharacter(String name, String characterClass) {
        this.name = name;
        this.characterClass = characterClass;
        this.level = 1;
        
        // 클래스별 초기 스탯 설정
        initializeStats(characterClass);
        
        // 장비 초기화
        this.equipment = new Equipment();
        equipStarterGear(characterClass);
        
        // 스킬 초기화
        this.skills = new ArrayList<>();
        learnBasicSkills(characterClass);
        
        // 인벤토리 초기화
        this.inventory = new Inventory();
        addStarterItems(characterClass);
        
        System.out.println("✨ " + characterClass + " 생성 완료 (복잡한 초기화)");
    }
    
    private void initializeStats(String characterClass) {
        switch (characterClass) {
            case "Warrior":
                this.health = 200;
                this.mana = 50;
                break;
            case "Mage":
                this.health = 100;
                this.mana = 200;
                break;
            case "Archer":
                this.health = 150;
                this.mana = 100;
                break;
        }
    }
    
    private void equipStarterGear(String characterClass) {
        // 시작 장비 착용 (시간 소요)
        equipment.setWeapon(characterClass + " Starter Weapon");
        equipment.setArmor(characterClass + " Starter Armor");
    }
    
    private void learnBasicSkills(String characterClass) {
        // 기본 스킬 학습
        skills.add(new Skill(characterClass + " Basic Attack"));
        skills.add(new Skill(characterClass + " Basic Defense"));
    }
    
    private void addStarterItems(String characterClass) {
        // 시작 아이템 지급
        inventory.addItem("Health Potion", 3);
        inventory.addItem("Mana Potion", 3);
    }
    
    // 깊은 복사 구현
    @Override
    public GameCharacter clone() {
        try {
            GameCharacter cloned = (GameCharacter) super.clone();
            
            // 깊은 복사
            cloned.equipment = new Equipment(this.equipment);
            cloned.skills = new ArrayList<>();
            for (Skill skill : this.skills) {
                cloned.skills.add(new Skill(skill));
            }
            cloned.inventory = new Inventory(this.inventory);
            
            System.out.println("⚡ 캐릭터 복제 (빠름!)");
            return cloned;
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
    
    @Override
    public String toString() {
        return "GameCharacter{" +
                "name='" + name + '\'' +
                ", class='" + characterClass + '\'' +
                ", level=" + level +
                ", HP=" + health +
                ", MP=" + mana +
                ", equipment=" + equipment +
                ", skills=" + skills.size() +
                ", items=" + inventory.getItemCount() +
                '}';
    }
    
    // Setters for customization
    public void setName(String name) { this.name = name; }
    public void levelUp() { this.level++; }
}

// 장비 클래스
class Equipment implements Cloneable {
    private String weapon;
    private String armor;
    
    public Equipment() {}
    
    public Equipment(Equipment other) {
        this.weapon = other.weapon;
        this.armor = other.armor;
    }
    
    public void setWeapon(String weapon) { this.weapon = weapon; }
    public void setArmor(String armor) { this.armor = armor; }
    
    @Override
    public String toString() {
        return weapon + ", " + armor;
    }
}

// 스킬 클래스
class Skill {
    private String name;
    
    public Skill(String name) { this.name = name; }
    public Skill(Skill other) { this.name = other.name; }
    
    @Override
    public String toString() { return name; }
}

// 인벤토리 클래스
class Inventory {
    private Map<String, Integer> items;
    
    public Inventory() {
        this.items = new HashMap<>();
    }
    
    public Inventory(Inventory other) {
        this.items = new HashMap<>(other.items);
    }
    
    public void addItem(String item, int count) {
        items.put(item, items.getOrDefault(item, 0) + count);
    }
    
    public int getItemCount() {
        return items.values().stream().mapToInt(Integer::intValue).sum();
    }
}

// 캐릭터 프로토타입 관리자
public class CharacterPrototypeManager {
    private static Map<String, GameCharacter> prototypes = new HashMap<>();
    
    // 프로토타입 등록 (초기화 한 번만)
    public static void initialize() {
        System.out.println("=== 캐릭터 프로토타입 초기화 ===");
        prototypes.put("Warrior", new GameCharacter("Template", "Warrior"));
        prototypes.put("Mage", new GameCharacter("Template", "Mage"));
        prototypes.put("Archer", new GameCharacter("Template", "Archer"));
        System.out.println("초기화 완료!\n");
    }
    
    // 프로토타입 복제
    public static GameCharacter createCharacter(String characterClass, String name) {
        GameCharacter prototype = prototypes.get(characterClass);
        GameCharacter character = prototype.clone();
        character.setName(name);
        return character;
    }
}

// 사용 예제
public class GameCharacterExample {
    public static void main(String[] args) {
        // 1. 프로토타입 초기화 (한 번만, 시간 소요)
        CharacterPrototypeManager.initialize();
        
        // 2. 복제를 통한 빠른 생성
        System.out.println("=== 캐릭터 생성 (프로토타입 사용) ===");
        
        long start = System.currentTimeMillis();
        GameCharacter player1 = CharacterPrototypeManager.createCharacter("Warrior", "철수");
        GameCharacter player2 = CharacterPrototypeManager.createCharacter("Mage", "영희");
        GameCharacter player3 = CharacterPrototypeManager.createCharacter("Archer", "민수");
        long end = System.currentTimeMillis();
        
        System.out.println("\n=== 생성된 캐릭터 ===");
        System.out.println(player1);
        System.out.println(player2);
        System.out.println(player3);
        
        System.out.println("\n생성 시간: " + (end - start) + "ms");
        
        // 3. 독립성 확인
        System.out.println("\n=== 독립성 확인 ===");
        player1.levelUp();
        System.out.println("철수 레벨업 후:");
        System.out.println("철수: " + player1);
        System.out.println("영희: " + player2);
        System.out.println("✅ 다른 캐릭터는 영향 없음!");
    }
}
```

**실행 결과:**
```
=== 캐릭터 프로토타입 초기화 ===
✨ Warrior 생성 완료 (복잡한 초기화)
✨ Mage 생성 완료 (복잡한 초기화)
✨ Archer 생성 완료 (복잡한 초기화)
초기화 완료!

=== 캐릭터 생성 (프로토타입 사용) ===
⚡ 캐릭터 복제 (빠름!)
⚡ 캐릭터 복제 (빠름!)
⚡ 캐릭터 복제 (빠름!)

=== 생성된 캐릭터 ===
GameCharacter{name='철수', class='Warrior', level=1, HP=200, MP=50, equipment=Warrior Starter Weapon, Warrior Starter Armor, skills=2, items=6}
GameCharacter{name='영희', class='Mage', level=1, HP=100, MP=200, equipment=Mage Starter Weapon, Mage Starter Armor, skills=2, items=6}
GameCharacter{name='민수', class='Archer', level=1, HP=150, MP=100, equipment=Archer Starter Weapon, Archer Starter Armor, skills=2, items=6}

생성 시간: 5ms

=== 독립성 확인 ===
철수 레벨업 후:
철수: GameCharacter{name='철수', class='Warrior', level=2, HP=200, MP=50, equipment=Warrior Starter Weapon, Warrior Starter Armor, skills=2, items=6}
영희: GameCharacter{name='영희', class='Mage', level=1, HP=100, MP=200, equipment=Mage Starter Weapon, Mage Starter Armor, skills=2, items=6}
✅ 다른 캐릭터는 영향 없음!
```

---

### 예제 2: 문서 템플릿 시스템 ⭐⭐⭐

```java
/**
 * 문서 템플릿
 */
public abstract class DocumentTemplate implements Cloneable {
    protected String title;
    protected String author;
    protected LocalDateTime createdAt;
    protected List<String> sections;
    protected Map<String, String> metadata;
    
    public DocumentTemplate() {
        this.createdAt = LocalDateTime.now();
        this.sections = new ArrayList<>();
        this.metadata = new HashMap<>();
    }
    
    @Override
    public DocumentTemplate clone() {
        try {
            DocumentTemplate cloned = (DocumentTemplate) super.clone();
            cloned.createdAt = LocalDateTime.now(); // 생성 시간은 새로 
            cloned.sections = new ArrayList<>(this.sections);
            cloned.metadata = new HashMap<>(this.metadata);
            return cloned;
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
    
    public abstract void fillTemplate();
    
    public void display() {
        System.out.println("=== " + title + " ===");
        System.out.println("Author: " + author);
        System.out.println("Created: " + createdAt.format(
                DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));
        System.out.println("Sections:");
        sections.forEach(s -> System.out.println("  - " + s));
        System.out.println("Metadata: " + metadata);
    }
    
    public void setTitle(String title) { this.title = title; }
    public void setAuthor(String author) { this.author = author; }
    public void addSection(String section) { this.sections.add(section); }
    public void addMetadata(String key, String value) { 
        this.metadata.put(key, value); 
    }
}

/**
 * 회의록 템플릿
 */
class MeetingNotesTemplate extends DocumentTemplate {
    public MeetingNotesTemplate() {
        this.title = "회의록";
        this.sections.add("1. 참석자");
        this.sections.add("2. 안건");
        this.sections.add("3. 논의 내용");
        this.sections.add("4. 결정 사항");
        this.sections.add("5. 액션 아이템");
        this.metadata.put("Type", "Meeting Notes");
        this.metadata.put("Template", "Standard");
    }
    
    @Override
    public void fillTemplate() {
        System.out.println("회의록 템플릿 작성 중...");
    }
}

/**
 * 프로젝트 제안서 템플릿
 */
class ProjectProposalTemplate extends DocumentTemplate {
    public ProjectProposalTemplate() {
        this.title = "프로젝트 제안서";
        this.sections.add("1. 프로젝트 개요");
        this.sections.add("2. 목표 및 범위");
        this.sections.add("3. 일정");
        this.sections.add("4. 예산");
        this.sections.add("5. 기대 효과");
        this.metadata.put("Type", "Project Proposal");
        this.metadata.put("Status", "Draft");
    }
    
    @Override
    public void fillTemplate() {
        System.out.println("프로젝트 제안서 작성 중...");
    }
}

/**
 * 문서 관리자
 */
public class DocumentManager {
    private Map<String, DocumentTemplate> templates = new HashMap<>();
    
    public void registerTemplate(String key, DocumentTemplate template) {
        templates.put(key, template);
        System.out.println("✓ 템플릿 등록: " + key);
    }
    
    public DocumentTemplate createDocument(String templateKey) {
        DocumentTemplate template = templates.get(templateKey);
        if (template == null) {
            throw new IllegalArgumentException("Unknown template: " + templateKey);
        }
        return template.clone();
    }
}

// 사용 예제
public class DocumentTemplateExample {
    public static void main(String[] args) {
        DocumentManager manager = new DocumentManager();
        
        // 1. 템플릿 등록 (한 번만)
        System.out.println("=== 템플릿 등록 ===");
        manager.registerTemplate("meeting", new MeetingNotesTemplate());
        manager.registerTemplate("proposal", new ProjectProposalTemplate());
        
        // 2. 템플릿으로부터 문서 생성
        System.out.println("\n=== 문서 생성 1 ===");
        DocumentTemplate doc1 = manager.createDocument("meeting");
        doc1.setAuthor("김철수");
        doc1.setTitle("2024년 1분기 회의록");
        doc1.display();
        
        System.out.println("\n=== 문서 생성 2 ===");
        DocumentTemplate doc2 = manager.createDocument("meeting");
        doc2.setAuthor("이영희");
        doc2.setTitle("프로젝트 킥오프 회의");
        doc2.addSection("6. 다음 회의 일정");
        doc2.display();
        
        System.out.println("\n=== 문서 생성 3 ===");
        DocumentTemplate doc3 = manager.createDocument("proposal");
        doc3.setAuthor("박민수");
        doc3.addMetadata("Department", "IT");
        doc3.display();
        
        System.out.println("\n✅ 각 문서는 독립적입니다!");
    }
}
```

---

### 예제 3: 그래픽 에디터 ⭐⭐

```java
/**
 * 그래픽 객체
 */
public interface Graphic extends Cloneable {
    void draw();
    void move(int x, int y);
    Graphic clone();
}

/**
 * 복잡한 그래픽 그룹
 */
public class GraphicGroup implements Graphic {
    private List<Graphic> children;
    private String name;
    private int x, y;
    
    public GraphicGroup(String name) {
        this.name = name;
        this.children = new ArrayList<>();
    }
    
    public void add(Graphic graphic) {
        children.add(graphic);
    }
    
    @Override
    public void draw() {
        System.out.println("Drawing Group: " + name + " at (" + x + "," + y + ")");
        children.forEach(Graphic::draw);
    }
    
    @Override
    public void move(int x, int y) {
        this.x = x;
        this.y = y;
    }
    
    @Override
    public GraphicGroup clone() {
        GraphicGroup cloned = new GraphicGroup(this.name + " (Copy)");
        cloned.x = this.x;
        cloned.y = this.y;
        
        // 자식들도 복제
        for (Graphic child : this.children) {
            cloned.add(child.clone());
        }
        
        return cloned;
    }
}

/**
 * 원
 */
class CircleGraphic implements Graphic {
    private int x, y, radius;
    private String color;
    
    public CircleGraphic(int x, int y, int radius, String color) {
        this.x = x;
        this.y = y;
        this.radius = radius;
        this.color = color;
    }
    
    @Override
    public void draw() {
        System.out.println("  Circle: " + color + " at (" + x + "," + y + ") r=" + radius);
    }
    
    @Override
    public void move(int x, int y) {
        this.x = x;
        this.y = y;
    }
    
    @Override
    public CircleGraphic clone() {
        return new CircleGraphic(this.x, this.y, this.radius, this.color);
    }
}

// 사용 예제
public class GraphicEditorExample {
    public static void main(String[] args) {
        // 복잡한 그래픽 그룹 생성
        System.out.println("=== 원본 그룹 생성 ===");
        GraphicGroup group = new GraphicGroup("Logo");
        group.add(new CircleGraphic(0, 0, 50, "Red"));
        group.add(new CircleGraphic(100, 0, 50, "Blue"));
        group.add(new CircleGraphic(50, 100, 50, "Green"));
        group.draw();
        
        // 복제
        System.out.println("\n=== 그룹 복제 ===");
        GraphicGroup clonedGroup = group.clone();
        clonedGroup.move(200, 200);
        clonedGroup.draw();
        
        System.out.println("\n=== 원본 확인 ===");
        group.draw();
        System.out.println("\n✅ 복제본만 이동됨!");
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **성능 향상** | 무거운 초기화 한 번만 | 게임 캐릭터 생성 |
| **복잡한 객체 복사** | 상태 보존하며 복제 | 문서 템플릿 |
| **동적 객체 생성** | 런타임에 타입 결정 | Shape 프로토타입 |
| **객체 독립성** | 복제본은 독립적 | 수정해도 원본 안전 |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **순환 참조** | 객체가 서로 참조 시 복잡 | 복사 로직 신중히 |
| **얕은/깊은 복사** | 참조 타입 처리 주의 | 명시적 깊은 복사 |
| **clone() 복잡성** | 올바른 구현 어려움 | 복사 생성자 고려 |

---

## 7. 안티패턴

### ❌ 안티패턴 1: 얕은 복사 문제

```java
// 잘못된 예: 참조 공유
public class Bad implements Cloneable {
    private List<String> data;
    
    @Override
    public Bad clone() {
        try {
            return (Bad) super.clone(); // data가 공유됨!
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
}
```

**해결:**
```java
@Override
public Good clone() {
    try {
        Good cloned = (Good) super.clone();
        cloned.data = new ArrayList<>(this.data); // 깊은 복사
        return cloned;
    } catch (CloneNotSupportedException e) {
        throw new RuntimeException(e);
    }
}
```

---

## 8. 핵심 정리

### 📌 Prototype 패턴 체크리스트

```
✅ Cloneable 구현 또는 복사 생성자
✅ clone() 메서드 구현
✅ 참조 타입은 깊은 복사
✅ 프로토타입 레지스트리 관리
✅ 독립성 확인
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 객체 생성 비용이 큼 | ⭐⭐⭐ | 복제가 빠름 |
| 복잡한 초기화 | ⭐⭐⭐ | 한 번만 초기화 |
| 동적 타입 생성 | ⭐⭐⭐ | 런타임 결정 |
| 상태 보존 복사 | ⭐⭐⭐ | 기존 상태 유지 |

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Builder](./03-Builder.md) | [다음: Abstract Factory →](./05-AbstractFactory.md)**

</div>
