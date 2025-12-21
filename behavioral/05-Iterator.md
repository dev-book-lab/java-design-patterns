# Iterator Pattern (반복자 패턴)

> **"컬렉션 내부 구조를 노출하지 않고 순회하자"**

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
// 문제 1: 내부 구조에 의존
public class BookCollection {
    private Book[] books;
    
    public Book[] getBooks() {
        return books; // 내부 배열 노출!
    }
}

public class Client {
    public void printBooks(BookCollection collection) {
        Book[] books = collection.getBooks();
        for (int i = 0; i < books.length; i++) {
            System.out.println(books[i]);
        }
        // 배열 구조에 의존!
        // List로 바뀌면? 전체 수정!
    }
}

// 문제 2: 다양한 순회 방법
public class Menu {
    private MenuItem[] items;
    
    // 앞에서부터 순회
    public void iterateForward() {
        for (int i = 0; i < items.length; i++) { }
    }
    
    // 뒤에서부터 순회
    public void iterateBackward() {
        for (int i = items.length - 1; i >= 0; i--) { }
    }
    
    // 조건부 순회
    public void iterateFiltered() {
        for (MenuItem item : items) {
            if (item.isActive()) { }
        }
    }
    
    // 순회 로직이 컬렉션에 섞임!
}

// 문제 3: 동시 순회 불가
public class Library {
    private List<Book> books;
    private int currentIndex = 0;
    
    public Book next() {
        return books.get(currentIndex++);
    }
    
    // 한 번에 하나만 순회 가능!
    // 두 곳에서 동시에 순회하면? 충돌!
}

// 문제 4: 서로 다른 컬렉션 순회
public class Restaurant {
    private MenuItem[] breakfastMenu;  // 배열
    private List<MenuItem> lunchMenu;   // List
    private Map<String, MenuItem> dinnerMenu; // Map
    
    public void printAllMenus() {
        // 각각 다른 방식으로 순회
        for (int i = 0; i < breakfastMenu.length; i++) { }
        for (MenuItem item : lunchMenu) { }
        for (Map.Entry<String, MenuItem> entry : dinnerMenu.entrySet()) { }
        
        // 통일된 방법이 없음!
    }
}
```

### ⚡ 핵심 문제

1. **내부 노출**: 컬렉션 내부 구조 노출
2. **결합도**: 클라이언트가 구현에 의존
3. **동시 순회**: 여러 순회 동시 불가
4. **일관성 없음**: 각 컬렉션마다 다른 순회 방법

---

## 2. 패턴 정의

### 📖 정의

> **컬렉션의 내부 표현을 노출하지 않고 순차적으로 요소에 접근할 수 있는 방법을 제공하는 패턴**

### 🎯 목적

- **캡슐화**: 내부 구조 숨김
- **통일된 인터페이스**: 모든 컬렉션 동일한 방법으로 순회
- **다중 순회**: 동시에 여러 순회 가능
- **다양한 순회**: 다양한 순회 방법 지원

### 💡 핵심 아이디어

```java
// Before: 내부 구조 노출
Book[] books = collection.getBooks();
for (int i = 0; i < books.length; i++) {
    System.out.println(books[i]);
}

// After: Iterator로 캡슐화
Iterator<Book> iterator = collection.iterator();
while (iterator.hasNext()) {
    System.out.println(iterator.next());
}
```

---

## 3. 구조와 구성요소

### 📊 UML 다이어그램

```
┌──────────────────┐
│   Aggregate      │  ← 컬렉션 인터페이스
├──────────────────┤
│ + iterator()     │───┐
└──────────────────┘   │ creates
         △             │
         │             ▼
┌──────────────────┐ ┌──────────────────┐
│ConcreteAggregate │ │   Iterator       │
├──────────────────┤ ├──────────────────┤
│ - items          │ │ + hasNext()      │
│ + iterator()     │ │ + next()         │
└──────────────────┘ │ + remove()       │
         │           └──────────────────┘
         │ creates            △
         │                    │ implements
         └───────────────────>│
                    ┌──────────────────┐
                    │ConcreteIterator  │
                    ├──────────────────┤
                    │ - aggregate      │
                    │ - index          │
                    │ + hasNext()      │
                    │ + next()         │
                    └──────────────────┘
```

### 🔧 구성요소

| 요소 | 역할 | 예시 |
|------|------|------|
| **Iterator** | 반복자 인터페이스 | `Iterator<T>` |
| **ConcreteIterator** | 구체적 반복자 | `BookIterator` |
| **Aggregate** | 컬렉션 인터페이스 | `Iterable<T>` |
| **ConcreteAggregate** | 구체적 컬렉션 | `BookCollection` |

---

## 4. 구현 방법

### 기본 구현: 책 컬렉션 ⭐⭐⭐

```java
/**
 * Iterator: 반복자 인터페이스
 */
public interface Iterator<T> {
    boolean hasNext();
    T next();
    void remove();
}

/**
 * Aggregate: 컬렉션 인터페이스
 */
public interface Iterable<T> {
    Iterator<T> iterator();
}

/**
 * 책 클래스
 */
public class Book {
    private String title;
    private String author;
    
    public Book(String title, String author) {
        this.title = title;
        this.author = author;
    }
    
    @Override
    public String toString() {
        return "📚 \"" + title + "\" by " + author;
    }
    
    public String getTitle() {
        return title;
    }
}

/**
 * ConcreteIterator: 책 반복자
 */
public class BookIterator implements Iterator<Book> {
    private Book[] books;
    private int position = 0;
    
    public BookIterator(Book[] books) {
        this.books = books;
    }
    
    @Override
    public boolean hasNext() {
        return position < books.length && books[position] != null;
    }
    
    @Override
    public Book next() {
        if (!hasNext()) {
            throw new NoSuchElementException();
        }
        return books[position++];
    }
    
    @Override
    public void remove() {
        if (position <= 0) {
            throw new IllegalStateException();
        }
        
        if (books[position - 1] != null) {
            // 뒤의 요소들을 앞으로 이동
            for (int i = position - 1; i < books.length - 1; i++) {
                books[i] = books[i + 1];
            }
            books[books.length - 1] = null;
            position--;
        }
    }
}

/**
 * ConcreteAggregate: 책 컬렉션
 */
public class BookCollection implements Iterable<Book> {
    private static final int MAX_ITEMS = 10;
    private Book[] books;
    private int numberOfBooks = 0;
    
    public BookCollection() {
        books = new Book[MAX_ITEMS];
    }
    
    public void addBook(String title, String author) {
        if (numberOfBooks >= MAX_ITEMS) {
            System.out.println("⚠️ 컬렉션이 가득 찼습니다!");
            return;
        }
        
        Book book = new Book(title, author);
        books[numberOfBooks] = book;
        numberOfBooks++;
        System.out.println("➕ 추가: " + book);
    }
    
    @Override
    public Iterator<Book> iterator() {
        return new BookIterator(books);
    }
    
    // 역방향 반복자
    public Iterator<Book> reverseIterator() {
        return new Iterator<Book>() {
            private int position = numberOfBooks - 1;
            
            @Override
            public boolean hasNext() {
                return position >= 0 && books[position] != null;
            }
            
            @Override
            public Book next() {
                if (!hasNext()) {
                    throw new NoSuchElementException();
                }
                return books[position--];
            }
            
            @Override
            public void remove() {
                throw new UnsupportedOperationException();
            }
        };
    }
}

/**
 * 사용 예제
 */
public class IteratorExample {
    public static void main(String[] args) {
        // 컬렉션 생성
        BookCollection collection = new BookCollection();
        
        System.out.println("=== 책 추가 ===");
        collection.addBook("Design Patterns", "GoF");
        collection.addBook("Clean Code", "Robert Martin");
        collection.addBook("Refactoring", "Martin Fowler");
        collection.addBook("Effective Java", "Joshua Bloch");
        
        // 정방향 순회
        System.out.println("\n=== 정방향 순회 ===");
        Iterator<Book> iterator = collection.iterator();
        while (iterator.hasNext()) {
            Book book = iterator.next();
            System.out.println(book);
        }
        
        // 역방향 순회
        System.out.println("\n=== 역방향 순회 ===");
        Iterator<Book> reverseIterator = collection.reverseIterator();
        while (reverseIterator.hasNext()) {
            Book book = reverseIterator.next();
            System.out.println(book);
        }
        
        // 동시 순회
        System.out.println("\n=== 동시 순회 (가능!) ===");
        Iterator<Book> iter1 = collection.iterator();
        Iterator<Book> iter2 = collection.iterator();
        
        System.out.println("Iterator 1: " + iter1.next());
        System.out.println("Iterator 2: " + iter2.next());
        System.out.println("Iterator 1: " + iter1.next());
        System.out.println("Iterator 2: " + iter2.next());
    }
}
```

**실행 결과:**
```
=== 책 추가 ===
➕ 추가: 📚 "Design Patterns" by GoF
➕ 추가: 📚 "Clean Code" by Robert Martin
➕ 추가: 📚 "Refactoring" by Martin Fowler
➕ 추가: 📚 "Effective Java" by Joshua Bloch

=== 정방향 순회 ===
📚 "Design Patterns" by GoF
📚 "Clean Code" by Robert Martin
📚 "Refactoring" by Martin Fowler
📚 "Effective Java" by Joshua Bloch

=== 역방향 순회 ===
📚 "Effective Java" by Joshua Bloch
📚 "Refactoring" by Martin Fowler
📚 "Clean Code" by Robert Martin
📚 "Design Patterns" by GoF

=== 동시 순회 (가능!) ===
Iterator 1: 📚 "Design Patterns" by GoF
Iterator 2: 📚 "Design Patterns" by GoF
Iterator 1: 📚 "Clean Code" by Robert Martin
Iterator 2: 📚 "Clean Code" by Robert Martin
```

---

## 5. 실전 예제

### 예제 1: 다양한 컬렉션 통합 ⭐⭐⭐

```java
/**
 * 메뉴 아이템
 */
public class MenuItem {
    private String name;
    private String description;
    private boolean vegetarian;
    private double price;
    
    public MenuItem(String name, String description, boolean vegetarian, double price) {
        this.name = name;
        this.description = description;
        this.vegetarian = vegetarian;
        this.price = price;
    }
    
    @Override
    public String toString() {
        return String.format("🍽️ %s ($%.2f) - %s %s",
                name, price, description,
                vegetarian ? "🌱" : "🥩");
    }
}

/**
 * 아침 메뉴 (배열 기반)
 */
public class BreakfastMenu implements Iterable<MenuItem> {
    private static final int MAX_ITEMS = 6;
    private MenuItem[] menuItems;
    private int numberOfItems = 0;
    
    public BreakfastMenu() {
        menuItems = new MenuItem[MAX_ITEMS];
        
        addItem("팬케이크", "블루베리 팬케이크", true, 5.99);
        addItem("와플", "벨기에 와플", true, 6.99);
        addItem("베이컨", "베이컨 & 에그", false, 7.99);
    }
    
    public void addItem(String name, String description, boolean vegetarian, double price) {
        MenuItem menuItem = new MenuItem(name, description, vegetarian, price);
        if (numberOfItems >= MAX_ITEMS) {
            System.out.println("메뉴가 가득 찼습니다!");
        } else {
            menuItems[numberOfItems] = menuItem;
            numberOfItems++;
        }
    }
    
    @Override
    public Iterator<MenuItem> iterator() {
        return new Iterator<MenuItem>() {
            private int position = 0;
            
            @Override
            public boolean hasNext() {
                return position < numberOfItems;
            }
            
            @Override
            public MenuItem next() {
                return menuItems[position++];
            }
            
            @Override
            public void remove() {
                throw new UnsupportedOperationException();
            }
        };
    }
}

/**
 * 점심 메뉴 (ArrayList 기반)
 */
public class LunchMenu implements Iterable<MenuItem> {
    private ArrayList<MenuItem> menuItems;
    
    public LunchMenu() {
        menuItems = new ArrayList<>();
        
        addItem("스테이크", "안심 스테이크", false, 15.99);
        addItem("샐러드", "시저 샐러드", true, 8.99);
        addItem("파스타", "크림 파스타", true, 12.99);
    }
    
    public void addItem(String name, String description, boolean vegetarian, double price) {
        MenuItem menuItem = new MenuItem(name, description, vegetarian, price);
        menuItems.add(menuItem);
    }
    
    @Override
    public Iterator<MenuItem> iterator() {
        return menuItems.iterator(); // ArrayList의 iterator 활용
    }
}

/**
 * 웨이트리스 (여러 메뉴 통합)
 */
public class Waitress {
    private List<Iterable<MenuItem>> menus;
    
    public Waitress(List<Iterable<MenuItem>> menus) {
        this.menus = menus;
    }
    
    public void printMenu() {
        System.out.println("\n=== 전체 메뉴 ===\n");
        
        int menuNumber = 1;
        for (Iterable<MenuItem> menu : menus) {
            System.out.println("메뉴 " + menuNumber++ + ":");
            Iterator<MenuItem> iterator = menu.iterator();
            printMenu(iterator);
            System.out.println();
        }
    }
    
    private void printMenu(Iterator<MenuItem> iterator) {
        while (iterator.hasNext()) {
            MenuItem menuItem = iterator.next();
            System.out.println("  " + menuItem);
        }
    }
    
    public void printVegetarianMenu() {
        System.out.println("\n=== 채식 메뉴 ===\n");
        
        for (Iterable<MenuItem> menu : menus) {
            Iterator<MenuItem> iterator = menu.iterator();
            while (iterator.hasNext()) {
                MenuItem menuItem = iterator.next();
                if (menuItem.toString().contains("🌱")) {
                    System.out.println("  " + menuItem);
                }
            }
        }
    }
}

/**
 * 사용 예제
 */
public class MenuExample {
    public static void main(String[] args) {
        BreakfastMenu breakfastMenu = new BreakfastMenu();
        LunchMenu lunchMenu = new LunchMenu();
        
        List<Iterable<MenuItem>> menus = Arrays.asList(
                breakfastMenu,
                lunchMenu
        );
        
        Waitress waitress = new Waitress(menus);
        
        waitress.printMenu();
        waitress.printVegetarianMenu();
    }
}
```

---

### 예제 2: 트리 구조 순회 ⭐⭐

```java
/**
 * 트리 노드
 */
public class TreeNode<T> {
    private T data;
    private List<TreeNode<T>> children;
    
    public TreeNode(T data) {
        this.data = data;
        this.children = new ArrayList<>();
    }
    
    public void addChild(TreeNode<T> child) {
        children.add(child);
    }
    
    public T getData() {
        return data;
    }
    
    public List<TreeNode<T>> getChildren() {
        return children;
    }
}

/**
 * DFS Iterator (깊이 우선 탐색)
 */
public class DFSIterator<T> implements Iterator<T> {
    private Stack<TreeNode<T>> stack;
    
    public DFSIterator(TreeNode<T> root) {
        stack = new Stack<>();
        if (root != null) {
            stack.push(root);
        }
    }
    
    @Override
    public boolean hasNext() {
        return !stack.isEmpty();
    }
    
    @Override
    public T next() {
        if (!hasNext()) {
            throw new NoSuchElementException();
        }
        
        TreeNode<T> node = stack.pop();
        
        // 자식들을 역순으로 push (왼쪽부터 방문하기 위해)
        List<TreeNode<T>> children = node.getChildren();
        for (int i = children.size() - 1; i >= 0; i--) {
            stack.push(children.get(i));
        }
        
        return node.getData();
    }
    
    @Override
    public void remove() {
        throw new UnsupportedOperationException();
    }
}

/**
 * BFS Iterator (너비 우선 탐색)
 */
public class BFSIterator<T> implements Iterator<T> {
    private Queue<TreeNode<T>> queue;
    
    public BFSIterator(TreeNode<T> root) {
        queue = new LinkedList<>();
        if (root != null) {
            queue.offer(root);
        }
    }
    
    @Override
    public boolean hasNext() {
        return !queue.isEmpty();
    }
    
    @Override
    public T next() {
        if (!hasNext()) {
            throw new NoSuchElementException();
        }
        
        TreeNode<T> node = queue.poll();
        
        // 자식들을 큐에 추가
        queue.addAll(node.getChildren());
        
        return node.getData();
    }
    
    @Override
    public void remove() {
        throw new UnsupportedOperationException();
    }
}

/**
 * 트리 (여러 순회 방법 제공)
 */
public class Tree<T> {
    private TreeNode<T> root;
    
    public Tree(TreeNode<T> root) {
        this.root = root;
    }
    
    public Iterator<T> dfsIterator() {
        return new DFSIterator<>(root);
    }
    
    public Iterator<T> bfsIterator() {
        return new BFSIterator<>(root);
    }
}

/**
 * 사용 예제
 */
public class TreeIteratorExample {
    public static void main(String[] args) {
        // 트리 구성
        TreeNode<String> root = new TreeNode<>("1");
        TreeNode<String> child1 = new TreeNode<>("2");
        TreeNode<String> child2 = new TreeNode<>("3");
        TreeNode<String> child3 = new TreeNode<>("4");
        
        root.addChild(child1);
        root.addChild(child2);
        
        child1.addChild(new TreeNode<>("5"));
        child1.addChild(new TreeNode<>("6"));
        child2.addChild(child3);
        
        Tree<String> tree = new Tree<>(root);
        
        // DFS 순회
        System.out.println("=== DFS (깊이 우선) ===");
        Iterator<String> dfs = tree.dfsIterator();
        while (dfs.hasNext()) {
            System.out.print(dfs.next() + " ");
        }
        
        // BFS 순회
        System.out.println("\n\n=== BFS (너비 우선) ===");
        Iterator<String> bfs = tree.bfsIterator();
        while (bfs.hasNext()) {
            System.out.print(bfs.next() + " ");
        }
    }
}
```

---

## 6. 장단점

### ✅ 장점

| 장점 | 설명 | 예시 |
|------|------|------|
| **캡슐화** | 내부 구조 숨김 | 배열/List 몰라도 됨 |
| **통일된 인터페이스** | 모든 컬렉션 동일하게 | Iterator |
| **동시 순회** | 여러 순회 가능 | iter1, iter2 |
| **다양한 순회** | 여러 방법 제공 | DFS, BFS |

### ❌ 단점

| 단점 | 설명 | 해결책 |
|------|------|--------|
| **단순 컬렉션에 과함** | 배열만 있으면 불필요 | 필요시에만 |
| **성능 오버헤드** | 객체 생성 비용 | for-each 사용 |

---

## 7. 안티패턴

### ❌ 안티패턴: Iterator 재사용

```java
// 잘못된 예
Iterator<Book> iterator = collection.iterator();
while (iterator.hasNext()) {
    System.out.println(iterator.next());
}

// 다시 순회하려면?
while (iterator.hasNext()) { // 작동 안 함!
    System.out.println(iterator.next());
}
```

**해결:**
```java
// 새 Iterator 생성
Iterator<Book> iterator2 = collection.iterator();
while (iterator2.hasNext()) {
    System.out.println(iterator2.next());
}
```

---

## 8. 핵심 정리

### 📌 Iterator 패턴 체크리스트

```
✅ Iterator 인터페이스 정의
✅ Iterable 인터페이스 정의
✅ ConcreteIterator 구현
✅ ConcreteAggregate 구현
✅ hasNext(), next() 구현
```

### 🎯 언제 사용할까?

| 상황 | 추천도 | 이유 |
|------|--------|------|
| 내부 구조 숨김 | ⭐⭐⭐ | 캡슐화 |
| 통일된 순회 | ⭐⭐⭐ | 일관성 |
| 다양한 순회 | ⭐⭐⭐ | DFS/BFS |
| 동시 순회 | ⭐⭐⭐ | 독립성 |

### 💡 핵심 포인트

1. **내부 구조 숨김**
2. **통일된 인터페이스**
3. **동시 순회 가능**
4. **Java에 내장**

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[← 이전: Command](04-Command.md) | [다음: State →](06-State.md)**

</div>
